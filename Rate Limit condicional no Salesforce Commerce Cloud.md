1. Introdução
Esta implementação visa criar um mecanismo de "Circuit Breaker" ou "Soft Ban" para usuários que realizam ações que resultam em uma resposta específica de uma API externa (ex: reject_id: 5).

Objetivos:
- Bloqueio Condicional: Bloquear o usuário por 10 minutos apenas se a API retornar o código de rejeição específico.
- Performance: Utilizar a Sessão para verificação rápida, evitando chamadas desnecessárias ao banco de dados.
- Rastreabilidade (Tracking): Utilizar Custom Objects para persistir o bloqueio e permitir auditoria via Business Manager.

2. Arquitetura

O diagrama abaixo ilustra o fluxo de decisão. Utilizamos o padrão de "Cache-First" (Sessão) para leitura e "Write-Through" (Sessão + DB) para escrita do bloqueio.

```
@startuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

Person(user, "Usuário", "Cliente acessando o site")
Container(controller, "Controller SFCC", "JavaScript", "Recebe a requisição HTTP/AJAX")
Container(helper, "RateLimitHelper", "Script Module", "Lógica de verificação e bloqueio")
ContainerDb(session, "SFCC Session", "Memory", "Cache temporário de leitura rápida")
ContainerDb(customObj, "Custom Object", "Database", "Persistência e Tracking (RateLimitTracking)")
System_Ext(api, "API Externa", "Sistema Terceiro", "Processa a requisição de negócio")

Rel(user, controller, "Faz Requisição")
Rel(controller, helper, "1. Verifica se está bloqueado")

Rel(helper, session, "2. Check Session (Fast)")
Rel(helper, customObj, "3. Check DB (Fallback/Sync)")

alt Bloqueado
    Rel(helper, controller, "Retorna Status: Bloqueado")
    Rel(controller, user, "Retorna Erro 429")
else Permitido
    Rel(controller, api, "4. Chama API")
    
    alt Resposta API == reject_id: 5
        Rel(api, controller, "Retorna Rejeição")
        Rel(controller, helper, "5. Solicita Bloqueio (10 min)")
        Rel(helper, session, "6. Grava na Sessão")
        Rel(helper, customObj, "7. Grava no Custom Object (Tracking)")
        Rel(controller, user, "Retorna Erro de Bloqueio")
    else Sucesso
        Rel(controller, user, "Retorna Sucesso")
    end
end
@enduml
``` 

3. Business Manager (Configuração)
Para que o tracking funcione, é necessário criar a estrutura de dados no Business Manager.

Criação do Custom Object
Acesse: Administration > Site Development > Custom Object Types
1. Novo Tipo:
- ID: RateLimitTracking
- Key Attribute: blockedID (String) — Armazenará o CustomerNo ou ID da Sessão.
- Storage Scope: Site

2. Definição de Atributos (Attribute Definitions):
- Crie os atributos abaixo e adicione-os ao "Attribute Group" do objeto.

ID do Atributo,Tipo de Dado,Obrigatório,Descrição
unblockTime,Date+Time,Sim,Data/Hora exata em que o bloqueio expira.
reason,String,Não,"Motivo do bloqueio (ex: ""API Reject 5""). Útil para auditoria."


4. Códigos (Backend SFCC)
4.1. Helper Script (RateLimitHelper.js)
Este é o módulo central. Salve em: */cartridge/scripts/helpers/RateLimitHelper.js

```
'use strict';

/**
 * RateLimitHelper.js
 * Gerencia a lógica de bloqueio temporário de usuários baseada em respostas de API.
 */

var CustomObjectMgr = require('dw/object/CustomObjectMgr');
var Transaction = require('dw/system/Transaction');
var Logger = require('dw/system/Logger');

// Configuração: 10 minutos em milissegundos
var BLOCK_DURATION_MS = 10 * 60 * 1000; 

/**
 * Verifica se o ID fornecido está atualmente bloqueado.
 * Prioriza a sessão para performance.
 * @param {string} id - Identificador do usuário (CustomerNo ou SessionID)
 * @returns {boolean} True se bloqueado, False caso contrário.
 */
function isBlocked(id) {
    var now = new Date();

    // 1. Verificação Rápida (Sessão)
    if (session.privacy.isRateLimited === true) {
        if (session.privacy.rateLimitExpire > now.getTime()) {
            return true;
        }
        // Se expirou na sessão, limpamos a flag localmente
        session.privacy.isRateLimited = false;
    }

    // 2. Verificação Robusta (Banco de Dados - Custom Object)
    // Só consulta o banco se não achou na sessão (ou para garantir sincronia em nova sessão)
    var blockRecord = CustomObjectMgr.getCustomObject('RateLimitTracking', id);

    if (blockRecord) {
        if (blockRecord.custom.unblockTime > now) {
            // Sincroniza a sessão para evitar próximos hits no DB
            session.privacy.isRateLimited = true;
            session.privacy.rateLimitExpire = blockRecord.custom.unblockTime.getTime();
            return true;
        } 
        
        // Se o bloqueio no DB já expirou, removemos o registro para limpar o tracking
        try {
            Transaction.wrap(function () {
                CustomObjectMgr.remove(blockRecord);
            });
        } catch (e) {
            Logger.error('RateLimitHelper: Erro ao limpar registro expirado: {0}', e.message);
        }
    }

    return false;
}

/**
 * Aplica o bloqueio ao usuário devido a uma resposta negativa da API.
 * @param {string} id - Identificador do usuário
 * @param {string} reason - Motivo para o log/tracking
 */
function applyBlock(id, reason) {
    var unlockDate = new Date(new Date().getTime() + BLOCK_DURATION_MS);

    // 1. Aplica na Sessão (Bloqueio Imediato)
    session.privacy.isRateLimited = true;
    session.privacy.rateLimitExpire = unlockDate.getTime();

    // 2. Persiste no Banco (Tracking e Segurança entre sessões)
    try {
        Transaction.wrap(function () {
            var blockRecord = CustomObjectMgr.getCustomObject('RateLimitTracking', id);
            
            if (!blockRecord) {
                blockRecord = CustomObjectMgr.createCustomObject('RateLimitTracking', id);
            }

            blockRecord.custom.unblockTime = unlockDate;
            blockRecord.custom.reason = reason || 'API Rejection';
        });
        
        Logger.warn('RateLimitHelper: Usuário {0} bloqueado até {1}. Motivo: {2}', id, unlockDate, reason);
        
    } catch (e) {
        Logger.error('RateLimitHelper: Falha ao gravar no DB: {0}', e.message);
    }
}

module.exports = {
    isBlocked: isBlocked,
    applyBlock: applyBlock
};
```

4.2. Exemplo de Uso no Controller (CheckoutServices.js ou similar)

Exemplo de implementação dentro de uma rota de controller.

```
var server = require('server');

var RateLimitHelper = require('*/cartridge/scripts/helpers/RateLimitHelper');
var ServiceMgr = require('dw/svc/LocalServiceRegistry'); // Exemplo genérico

server.post('SubmitPayment', server.middleware.https, function (req, res, next) {
    
    // Identifica o usuário: Se logado usa CustomerNo, senão usa SessionID
    var currentUserId = req.currentCustomer.profile 
        ? req.currentCustomer.profile.customerNo 
        : session.sessionID;

    // --- PASSO 1: Verificação (Guarda) ---
    if (RateLimitHelper.isBlocked(currentUserId)) {
        res.setStatusCode(429); // Too Many Requests
        res.json({
            success: false,
            error: true,
            errorMessage: 'Muitas tentativas falhas. Por favor, aguarde 10 minutos.'
        });
        return next();
    }

    // --- PASSO 2: Chamada do Serviço ---
    // (Exemplo simplificado de chamada de serviço)
    var result = { ok: true, object: { reject_id: 5 } }; // SIMULAÇÃO DA API

    // --- PASSO 3: Análise da Resposta ---
    if (result.ok) {
        var apiResponse = result.object;

        // Verifica a condição de bloqueio (reject_id: 5)
        if (apiResponse && apiResponse.reject_id === 5) {
            
            // APLICA O RATE LIMIT
            RateLimitHelper.applyBlock(currentUserId, 'Payment API reject_id: 5');

            res.setStatusCode(400); // Bad Request (ou 429 dependendo da regra de negócio)
            res.json({
                success: false,
                error: true,
                errorMessage: 'Transação recusada. Aguarde 10 minutos para tentar novamente.'
            });
            return next();
        }
        
        // ... Lógica de sucesso normal ...
    }

    res.json({ success: true });
    return next();
});

module.exports = server.exports;
```

5. Storefront (Frontend)
O frontend deve estar preparado para lidar com o status HTTP 429 ou com a mensagem de erro JSON retornada.
Exemplo Ajax (jQuery/Zepto padrão SFCC)

```
$.ajax({
    url: 'CheckoutServices-SubmitPayment',
    method: 'POST',
    data: formData,
    success: function (data) {
        // Lógica de sucesso
    },
    error: function (xhr) {
        if (xhr.status === 429 || (xhr.responseJSON && xhr.responseJSON.errorMessage)) {
            // Exibe mensagem amigável ao usuário
            var msg = xhr.responseJSON.errorMessage || "Sistema temporariamente indisponível.";
            displayErrorMessage(msg);
            
            // Opcional: Desabilitar botão de submit visualmente
            $('.submit-button').prop('disabled', true).text('Aguarde...');
        }
    }
});
```

6. Como Visualizar o Tracking (Operacional)
Após a implementação, a equipe de suporte ou operações pode verificar os bloqueios:

1. No Business Manager, vá em Merchant Tools > Custom Objects > Custom Object Editor.
2. Selecione o tipo RateLimitTracking.
3. Clique em Find.
4. A lista exibirá:
- blockedID: Quem está bloqueado.
- unblockTime: Quando o bloqueio acaba.
- reason: Por que foi bloqueado (ex: "Payment API reject_id: 5").
