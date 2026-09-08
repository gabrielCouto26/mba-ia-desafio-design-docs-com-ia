| ID | Documento | Tipo | Conteudo (resumo) | Fonte | Localizacao |
|---|---|---|---|---|---|
| PRD-OBJ-001 | [docs/PRD.md](PRD.md) | Requisito | Clientes B2B precisam receber notificacoes outbound quando pedidos mudam de status, reduzindo polling em `GET /orders` | TRANSCRICAO | TRANSCRICAO.md [09:00] Marcos |
| PRD-OBJ-002 | [docs/PRD.md](PRD.md) | Requisito | Webhooks sao somente outbound; clientes recebem eventos, nao enviam eventos para a plataforma | TRANSCRICAO | TRANSCRICAO.md [09:02] Marcos |
| PRD-MET-001 | [docs/PRD.md](PRD.md) | Criterio de aceite | Notificacao abaixo de 10 segundos e tratada como tempo real pelos clientes | TRANSCRICAO | TRANSCRICAO.md [09:02] Marcos |
| PRD-MET-002 | [docs/PRD.md](PRD.md) | Criterio de aceite | Polling de 2 segundos atende a meta percebida de abaixo de 10 segundos | TRANSCRICAO | TRANSCRICAO.md [09:09] Diego |
| PRD-REQ-001 | [docs/PRD.md](PRD.md) | Requisito | CRUD autenticado para cadastrar, listar, atualizar e desativar endpoints de webhook | TRANSCRICAO | TRANSCRICAO.md [09:33] Bruno |
| PRD-REQ-002 | [docs/PRD.md](PRD.md) | Requisito | Criacao de endpoint recebe `url`, filtros de status, `customerId` e retorna secret gerada pela plataforma | TRANSCRICAO | TRANSCRICAO.md [09:31] Marcos |
| PRD-REQ-003 | [docs/PRD.md](PRD.md) | Requisito | `customer_id` nao vem implicitamente do JWT; deve ser informado pelo contrato da API | TRANSCRICAO | TRANSCRICAO.md [09:32] Larissa |
| PRD-REQ-004 | [docs/PRD.md](PRD.md) | Requisito | Filtros por status definem quais mudancas de pedido geram webhook para cada endpoint | TRANSCRICAO | TRANSCRICAO.md [09:33] Marcos |
| PRD-REQ-005 | [docs/PRD.md](PRD.md) | Requisito | Filtro de endpoints interessados deve ocorrer na insercao da outbox, evitando linhas desnecessarias | TRANSCRICAO | TRANSCRICAO.md [09:34] Bruno |
| PRD-REQ-006 | [docs/PRD.md](PRD.md) | Requisito | Historico de entregas deve retornar os ultimos 100 envios, com sucesso ou falha, payload, response e latencia | TRANSCRICAO | TRANSCRICAO.md [09:34] Marcos |
| PRD-REQ-007 | [docs/PRD.md](PRD.md) | Requisito | Replay manual de DLQ deve existir por endpoint administrativo | TRANSCRICAO | TRANSCRICAO.md [09:18] Diego |
| PRD-REQ-008 | [docs/PRD.md](PRD.md) | Requisito | Replay de DLQ exige role `ADMIN` do JWT e deve registrar quem executou a operacao | TRANSCRICAO | TRANSCRICAO.md [09:36] Sofia |
| PRD-REQ-009 | [docs/PRD.md](PRD.md) | Requisito | CRUD e consulta de webhooks podem ser executados por qualquer role autenticada no MVP | TRANSCRICAO | TRANSCRICAO.md [09:37] Sofia |
| PRD-REQ-010 | [docs/PRD.md](PRD.md) | Requisito | Payloads acima de 64KB devem ser rejeitados, sem truncamento | TRANSCRICAO | TRANSCRICAO.md [09:24] Larissa |
| PRD-NFR-001 | [docs/PRD.md](PRD.md) | Requisito | Entrega externa nao pode bloquear ou causar rollback indevido por indisponibilidade do cliente | TRANSCRICAO | TRANSCRICAO.md [09:04] Bruno |
| PRD-NFR-002 | [docs/PRD.md](PRD.md) | Requisito | URLs de webhook devem usar HTTPS obrigatoriamente | TRANSCRICAO | TRANSCRICAO.md [09:23] Sofia |
| PRD-NFR-003 | [docs/PRD.md](PRD.md) | Requisito | Payload deve permanecer enxuto e nao incluir `items`; detalhes ficam disponiveis por `GET /orders/:id` | TRANSCRICAO | TRANSCRICAO.md [09:43] Diego |
| PRD-NFR-004 | [docs/PRD.md](PRD.md) | Requisito | Secrets de webhook devem ser geradas pela plataforma e associadas a endpoints especificos | TRANSCRICAO | TRANSCRICAO.md [09:21] Sofia |
| PRD-NFR-005 | [docs/PRD.md](PRD.md) | Requisito | Operacoes de entrega devem gerar logs estruturados com dados de tentativa, latencia, status e erro | TRANSCRICAO | TRANSCRICAO.md [09:29] Bruno |
| PRD-ACC-001 | [docs/PRD.md](PRD.md) | Criterio de aceite | CA-001: CRUD de endpoints implementado e protegido por autenticacao JWT | TRANSCRICAO | TRANSCRICAO.md [09:32] Marcos |
| PRD-ACC-002 | [docs/PRD.md](PRD.md) | Criterio de aceite | CA-002: mudanca de status cria evento consultavel por `eventId` dentro da outbox | TRANSCRICAO | TRANSCRICAO.md [09:40] Bruno |
| PRD-ACC-003 | [docs/PRD.md](PRD.md) | Criterio de aceite | CA-003: tentativas de entrega sao registradas e historico e consultavel por `eventId` e endpoint | TRANSCRICAO | TRANSCRICAO.md [09:34] Marcos |
| PRD-ACC-004 | [docs/PRD.md](PRD.md) | Criterio de aceite | CA-004: reprocessamento manual de DLQ cria nova tentativa e atualiza historico | TRANSCRICAO | TRANSCRICAO.md [09:35] Diego |
| PRD-ACC-005 | [docs/PRD.md](PRD.md) | Criterio de aceite | CA-005: 90% dos primeiros envios em teste piloto devem ocorrer em ate 10s | TRANSCRICAO | TRANSCRICAO.md [09:02] Marcos |
| PRD-ACC-006 | [docs/PRD.md](PRD.md) | Criterio de aceite | CA-006: entregas incluem headers de rastreio e assinatura validavel pelo consumidor | TRANSCRICAO | TRANSCRICAO.md [09:44] Diego |
| PRD-RSK-001 | [docs/PRD.md](PRD.md) | Risco | Entregas duplicadas podem ocorrer; consumidores precisam deduplicar por `X-Event-Id` | TRANSCRICAO | TRANSCRICAO.md [09:25] Diego |
| PRD-RSK-002 | [docs/PRD.md](PRD.md) | Risco | Crescimento de outbox e historico exige politica futura de retencao ou arquivamento | TRANSCRICAO | TRANSCRICAO.md [09:08] Diego |
| PRD-RSK-003 | [docs/PRD.md](PRD.md) | Risco | Secret global comprometeria todos os endpoints em caso de vazamento | TRANSCRICAO | TRANSCRICAO.md [09:21] Sofia |
| PRD-RSK-004 | [docs/PRD.md](PRD.md) | Risco | Cliente lento ou fora do ar deve ser tratado por retry e DLQ, nao por rollback de pedido | TRANSCRICAO | TRANSCRICAO.md [09:04] Bruno |
| PRD-DEP-001 | [docs/PRD.md](PRD.md) | Dependencia | Plataforma precisa operar e monitorar worker separado, migrations, DLQ e historico | TRANSCRICAO | TRANSCRICAO.md [09:11] Diego |
| PRD-DEP-002 | [docs/PRD.md](PRD.md) | Dependencia | Revisao de seguranca precisa ocorrer antes do deploy, especialmente HMAC e geracao de secret | TRANSCRICAO | TRANSCRICAO.md [09:46] Sofia |
| PRD-OOS-001 | [docs/PRD.md](PRD.md) | Fora de escopo | Email automatico como fallback para falhas fica fora do MVP | TRANSCRICAO | TRANSCRICAO.md [09:37] Larissa |
| PRD-OOS-002 | [docs/PRD.md](PRD.md) | Fora de escopo | Dashboard visual para clientes fica fora desta fase; apenas endpoints serao entregues | TRANSCRICAO | TRANSCRICAO.md [09:40] Larissa |
| PRD-OOS-003 | [docs/PRD.md](PRD.md) | Fora de escopo | Rate limiting outbound por cliente sera observado antes de virar politica | TRANSCRICAO | TRANSCRICAO.md [09:39] Diego |
| RFC-DEC-001 | [docs/RFC.md](RFC.md) | Decisao | Proposta central: webhooks outbound com captura transacional por outbox no MySQL e entrega por worker separado | TRANSCRICAO | TRANSCRICAO.md [09:06] Diego |
| RFC-DEC-002 | [docs/RFC.md](RFC.md) | Decisao | Evento de webhook deve ser gravado na mesma transacao que altera `orders` e `order_status_history` | TRANSCRICAO | TRANSCRICAO.md [09:06] Diego |
| RFC-DEC-003 | [docs/RFC.md](RFC.md) | Decisao | Falha ao inserir outbox deve abortar a transacao para evitar status alterado sem evento registrado | TRANSCRICAO | TRANSCRICAO.md [09:41] Diego |
| RFC-DEC-004 | [docs/RFC.md](RFC.md) | Decisao | Evento da outbox deve guardar snapshot do payload renderizado no momento da insercao | TRANSCRICAO | TRANSCRICAO.md [09:52] Larissa |
| RFC-DEC-005 | [docs/RFC.md](RFC.md) | Decisao | IDs de eventos devem usar UUID, seguindo o padrao do restante do projeto | TRANSCRICAO | TRANSCRICAO.md [09:51] Larissa |
| RFC-ALT-001 | [docs/RFC.md](RFC.md) | Decisao | Chamada HTTP sincrona dentro de `changeStatus` foi descartada por acoplamento com rede externa | TRANSCRICAO | TRANSCRICAO.md [09:06] Diego |
| RFC-ALT-002 | [docs/RFC.md](RFC.md) | Decisao | Redis Streams ou fila externa foram descartados como overengineering para a fase atual | TRANSCRICAO | TRANSCRICAO.md [09:07] Diego |
| RFC-ALT-003 | [docs/RFC.md](RFC.md) | Decisao | Trigger de banco para acordar worker foi descartada porque MySQL nao oferece notificacao externa nativa | TRANSCRICAO | TRANSCRICAO.md [09:09] Diego |
| RFC-ALT-004 | [docs/RFC.md](RFC.md) | Decisao | Exactly-once foi descartado; at-least-once com deduplicacao por evento e suficiente para o MVP | TRANSCRICAO | TRANSCRICAO.md [09:25] Diego |
| RFC-OPEN-001 | [docs/RFC.md](RFC.md) | Dependencia | Politica de rate limiting por cliente permanece em aberto para decisao apos observabilidade real | TRANSCRICAO | TRANSCRICAO.md [09:39] Diego |
| RFC-OPEN-002 | [docs/RFC.md](RFC.md) | Dependencia | Politica definitiva de retencao e arquivamento permanece em aberto; reuniao citou aproximadamente 30 dias | TRANSCRICAO | TRANSCRICAO.md [09:08] Diego |
| RFC-OPEN-003 | [docs/RFC.md](RFC.md) | Dependencia | Escala com multiplos workers e particionamento por `order_id` fica para decisao futura | TRANSCRICAO | TRANSCRICAO.md [09:13] Diego |
| RFC-OPEN-004 | [docs/RFC.md](RFC.md) | Dependencia | Notificacao ativa de falhas ao cliente, como email, sera reavaliada depois do MVP | TRANSCRICAO | TRANSCRICAO.md [09:37] Larissa |
| ADR-001 | [docs/adrs/ADR-001-outbox-no-mysql.md](adrs/ADR-001-outbox-no-mysql.md) | Decisao | Adotar Outbox no MySQL existente para garantir consistencia entre pedido e evento | TRANSCRICAO | TRANSCRICAO.md [09:06] Diego |
| ADR-002 | [docs/adrs/ADR-002-politica-de-retry-backoff-e-dlq.md](adrs/ADR-002-politica-de-retry-backoff-e-dlq.md) | Decisao | Usar 5 tentativas com backoff fixo de 1m, 5m, 30m, 2h e 12h, depois DLQ separada | TRANSCRICAO | TRANSCRICAO.md [09:17] Diego |
| ADR-003 | [docs/adrs/ADR-003-assinatura-hmac-sha256-por-endpoint.md](adrs/ADR-003-assinatura-hmac-sha256-por-endpoint.md) | Decisao | Assinar payload com HMAC-SHA256 usando secret propria por endpoint | TRANSCRICAO | TRANSCRICAO.md [09:20] Sofia |
| ADR-004 | [docs/adrs/ADR-004-entrega-at-least-once-com-x-event-id.md](adrs/ADR-004-entrega-at-least-once-com-x-event-id.md) | Decisao | Garantia de entrega at-least-once com `X-Event-Id` para deduplicacao pelo consumidor | TRANSCRICAO | TRANSCRICAO.md [09:26] Larissa |
| ADR-005 | [docs/adrs/ADR-005-worker-separado-com-polling.md](adrs/ADR-005-worker-separado-com-polling.md) | Decisao | Worker deve rodar como processo separado e consultar pendentes por polling a cada 2 segundos | TRANSCRICAO | TRANSCRICAO.md [09:10] Larissa |
| ADR-006 | [docs/adrs/ADR-006-reuso-de-padroes-existentes-do-projeto.md](adrs/ADR-006-reuso-de-padroes-existentes-do-projeto.md) | Decisao | Implementar webhooks como `src/modules/webhooks`, reutilizando AppError, Pino, Zod, Prisma e middlewares existentes | TRANSCRICAO | TRANSCRICAO.md [09:30] Larissa |
| FDD-CON-001 | [docs/FDD.md](FDD.md) | Contrato | Insercao da outbox deve ocorrer em `OrderService.changeStatus()` dentro da transacao Prisma existente | TRANSCRICAO | TRANSCRICAO.md [09:40] Bruno |
| FDD-CON-002 | [docs/FDD.md](FDD.md) | Contrato | Funcao de publicacao deve receber o transaction client, evitando injetar o repository inteiro de webhook no pedido | TRANSCRICAO | TRANSCRICAO.md [09:41] Diego |
| FDD-CON-003 | [docs/FDD.md](FDD.md) | Contrato | Payload minimo inclui `event_id`, `event_type`, `timestamp`, dados de pedido, status anterior e novo status | TRANSCRICAO | TRANSCRICAO.md [09:43] Diego |
| FDD-CON-004 | [docs/FDD.md](FDD.md) | Contrato | Worker envia `Content-Type: application/json`, `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id` | TRANSCRICAO | TRANSCRICAO.md [09:44] Sofia |
| FDD-CON-005 | [docs/FDD.md](FDD.md) | Contrato | Request HTTP outbound do worker deve ter timeout de 10 segundos | TRANSCRICAO | TRANSCRICAO.md [09:42] Diego |
| FDD-CON-006 | [docs/FDD.md](FDD.md) | Contrato | `POST /webhooks` cria endpoint, valida HTTPS, gera secret e retorna a secret na criacao | TRANSCRICAO | TRANSCRICAO.md [09:31] Marcos |
| FDD-CON-007 | [docs/FDD.md](FDD.md) | Contrato | `GET /webhooks`, `PATCH /webhooks/:id` e desativacao compoem o CRUD autenticado de configuracao | TRANSCRICAO | TRANSCRICAO.md [09:33] Bruno |
| FDD-CON-008 | [docs/FDD.md](FDD.md) | Contrato | `POST /webhooks/:id/rotate-secret` gera nova secret e mantem a anterior valida por 24 horas | TRANSCRICAO | TRANSCRICAO.md [09:21] Sofia |
| FDD-CON-009 | [docs/FDD.md](FDD.md) | Contrato | `GET /webhooks/:id/deliveries` retorna historico de entregas do endpoint | TRANSCRICAO | TRANSCRICAO.md [09:34] Marcos |
| FDD-CON-010 | [docs/FDD.md](FDD.md) | Contrato | `POST /admin/webhooks/dead-letter/:id/replay` reprocessa DLQ e e restrito a `ADMIN` | TRANSCRICAO | TRANSCRICAO.md [09:35] Larissa |
| FDD-CON-011 | [docs/FDD.md](FDD.md) | Contrato | Worker deve selecionar eventos pendentes mais antigos em batch pequeno, processar e marcar como entregues | TRANSCRICAO | TRANSCRICAO.md [09:08] Diego |
| FDD-CON-012 | [docs/FDD.md](FDD.md) | Contrato | Ordering e limitado por `order_id` enquanto houver single-worker; nao ha garantia global | TRANSCRICAO | TRANSCRICAO.md [09:12] Diego |
| FDD-CON-013 | [docs/FDD.md](FDD.md) | Contrato | Worker deve usar PrismaClient proprio por processo, mesma `DATABASE_URL` e mesmo banco da API | TRANSCRICAO | TRANSCRICAO.md [09:30] Bruno |
| FDD-ERR-001 | [docs/FDD.md](FDD.md) | Contrato | `WEBHOOK_ENDPOINT_NOT_FOUND`: endpoint inexistente deve retornar 404 em chamadas de gestao | TRANSCRICAO | TRANSCRICAO.md [09:28] Bruno |
| FDD-ERR-002 | [docs/FDD.md](FDD.md) | Contrato | `WEBHOOK_ENDPOINT_INACTIVE`: endpoint desativado nao deve gerar outbox para novas entregas | TRANSCRICAO | TRANSCRICAO.md [09:33] Bruno |
| FDD-ERR-003 | [docs/FDD.md](FDD.md) | Contrato | `WEBHOOK_INVALID_URL`: URL nao HTTPS ou invalida deve ser recusada na validacao | TRANSCRICAO | TRANSCRICAO.md [09:23] Sofia |
| FDD-ERR-004 | [docs/FDD.md](FDD.md) | Contrato | `WEBHOOK_SECRET_ROTATION_FAILED`: falha na geracao ou rotacao nao deve alterar endpoint | TRANSCRICAO | TRANSCRICAO.md [09:21] Sofia |
| FDD-ERR-005 | [docs/FDD.md](FDD.md) | Contrato | `WEBHOOK_PAYLOAD_INVALID`: payload invalido nao deve ser inserido na outbox | TRANSCRICAO | TRANSCRICAO.md [09:24] Larissa |
| FDD-ERR-006 | [docs/FDD.md](FDD.md) | Contrato | `WEBHOOK_PAYLOAD_TOO_LARGE`: payload acima de 64KB deve ser rejeitado sem truncamento | TRANSCRICAO | TRANSCRICAO.md [09:24] Larissa |
| FDD-ERR-007 | [docs/FDD.md](FDD.md) | Contrato | `WEBHOOK_SIGNATURE_FAILED`: rejeicao de assinatura pelo consumidor conta como falha de entrega | TRANSCRICAO | TRANSCRICAO.md [09:20] Sofia |
| FDD-ERR-008 | [docs/FDD.md](FDD.md) | Contrato | `WEBHOOK_DELIVERY_TIMEOUT`: timeout de entrega incrementa tentativas e agenda retry | TRANSCRICAO | TRANSCRICAO.md [09:42] Diego |
| FDD-ERR-009 | [docs/FDD.md](FDD.md) | Contrato | `WEBHOOK_DELIVERY_FAILED`: erro de rede ou HTTP 5xx incrementa tentativas e segue backoff | TRANSCRICAO | TRANSCRICAO.md [09:15] Diego |
| FDD-ERR-010 | [docs/FDD.md](FDD.md) | Contrato | `WEBHOOK_MAX_RETRIES_EXCEEDED`: ao esgotar tentativas, evento vai para DLQ | TRANSCRICAO | TRANSCRICAO.md [09:17] Larissa |
| FDD-ERR-011 | [docs/FDD.md](FDD.md) | Contrato | `WEBHOOK_DLQ_REPROCESS_FAILED`: falha de replay deve ser registrada e notificada para operacao | TRANSCRICAO | TRANSCRICAO.md [09:18] Diego |
| FDD-OBS-001 | [docs/FDD.md](FDD.md) | Contrato | Metricas minimas incluem pendentes na outbox, tentativas, tamanho da DLQ, falhas de insert e latencia | TRANSCRICAO | TRANSCRICAO.md [09:29] Bruno |
| FDD-OBS-002 | [docs/FDD.md](FDD.md) | Contrato | Logs nao devem expor secrets; secrets de webhook devem entrar na politica de redaction | TRANSCRICAO | TRANSCRICAO.md [09:22] Sofia |
| FDD-TEST-001 | [docs/FDD.md](FDD.md) | Criterio de aceite | Testes de integracao devem cobrir criacao de outbox na mudanca de status | TRANSCRICAO | TRANSCRICAO.md [09:46] Larissa |
| FDD-TEST-002 | [docs/FDD.md](FDD.md) | Criterio de aceite | Testes de falha devem simular timeout ou 5xx e verificar retry, backoff e DLQ | TRANSCRICAO | TRANSCRICAO.md [09:42] Diego |
| FDD-PLAN-001 | [docs/FDD.md](FDD.md) | Dependencia | Entrega estimada em tres sprints, incluindo modelagem, worker, CRUD, integracao, testes e revisao de seguranca | TRANSCRICAO | TRANSCRICAO.md [09:46] Larissa |
| CODE-001 | [docs/FDD.md](FDD.md) | Evidencia tecnica | `OrderService.changeStatus()` ja usa transacao Prisma e atualiza pedido, historico e estoque; e o ponto de integracao da outbox | CODIGO | src/modules/orders/order.service.ts |
| CODE-002 | [docs/adrs/ADR-001-outbox-no-mysql.md](adrs/ADR-001-outbox-no-mysql.md) | Evidencia tecnica | Schema atual usa MySQL via Prisma e modelos de pedido com UUID, mas ainda nao possui tabelas de webhook | CODIGO | prisma/schema.prisma |
| CODE-003 | [docs/adrs/ADR-006-reuso-de-padroes-existentes-do-projeto.md](adrs/ADR-006-reuso-de-padroes-existentes-do-projeto.md) | Evidencia tecnica | Router raiz registra modulos por dominio; novo router de webhooks deve seguir o mesmo padrao | CODIGO | src/routes/index.ts |
| CODE-004 | [docs/adrs/ADR-006-reuso-de-padroes-existentes-do-projeto.md](adrs/ADR-006-reuso-de-padroes-existentes-do-projeto.md) | Evidencia tecnica | Middleware de autenticacao valida Bearer JWT e expoe `requireRole`, incluindo suporte a `ADMIN` | CODIGO | src/middlewares/auth.middleware.ts |
| CODE-005 | [docs/adrs/ADR-006-reuso-de-padroes-existentes-do-projeto.md](adrs/ADR-006-reuso-de-padroes-existentes-do-projeto.md) | Evidencia tecnica | Middleware centralizado ja transforma `AppError`, Zod e erros Prisma em respostas HTTP padronizadas | CODIGO | src/middlewares/error.middleware.ts |
| CODE-006 | [docs/adrs/ADR-006-reuso-de-padroes-existentes-do-projeto.md](adrs/ADR-006-reuso-de-padroes-existentes-do-projeto.md) | Evidencia tecnica | Logger Pino existente possui redaction para credenciais e deve ser estendido para secrets de webhook | CODIGO | src/shared/logger/index.ts |
| CODE-007 | [docs/FDD.md](FDD.md) | Evidencia tecnica | Helpers de resposta HTTP existentes devem ser reutilizados pelos contratos do modulo de webhooks | CODIGO | src/shared/http/response.ts |
| CODE-008 | [docs/adrs/ADR-006-reuso-de-padroes-existentes-do-projeto.md](adrs/ADR-006-reuso-de-padroes-existentes-do-projeto.md) | Evidencia tecnica | Classes de erro compartilhadas sustentam novos erros `WEBHOOK_` sem criar formato paralelo | CODIGO | src/shared/errors/index.ts |
| CODE-009 | [docs/adrs/ADR-005-worker-separado-com-polling.md](adrs/ADR-005-worker-separado-com-polling.md) | Evidencia tecnica | Entrypoint HTTP atual e separado em `src/server.ts`; worker deve ter entrypoint proprio, nao iniciar pelo boot da API | CODIGO | src/server.ts |
| CODE-010 | [docs/FDD.md](FDD.md) | Evidencia tecnica | Testes atuais de pedidos existem e devem receber cenarios de integracao para criacao de outbox | CODIGO | tests/orders.test.ts |
