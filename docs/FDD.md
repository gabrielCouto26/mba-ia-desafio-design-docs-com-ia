# FDD — Webhooks de notificação de pedidos

Data: 2026-08-13
Autor: Tech Lead (gerado automaticamente)

## 1. Contexto e motivação técnica

Resumo técnico: clientes B2B precisam receber notificações em tempo quase real sobre mudanças de status de pedidos. A proposta (ver [RFC.md](RFC.md)) recomenda padrão outbox no MySQL e worker separado para entrega HTTP assinada (HMAC-SHA256), garantindo que a gravação do evento ocorra dentro da mesma transação do domínio de pedidos e que a entrega externa ocorra assincrona e resiliente.

Referências: [RFC.md](RFC.md), ADR-001..ADR-006 em `adrs/`.

Atores: `OrderService` (produtor), `WebhookWorker` (consumidor/entregador), operadores/admins, consumidores externos (clientes B2B).

Limites: primeira fase entrega at-least-once, single-worker compatível com polling (2s), sem rate limiting por cliente e sem dashboard visual.

## 2. Objetivos técnicos

- Garantir entrega at-least-once dos eventos de mudança de status do pedido.
- Persistir o evento na outbox dentro da transação que atualiza o pedido.
- Assinar callbacks com HMAC-SHA256 por endpoint (header `X-Signature`).
- Expor rastreabilidade por `X-Event-Id` e histórico de entregas.
- Retries com backoff progressivo até 5 tentativas; após isso, mover para DLQ.
- Expor APIs para CRUD de endpoints, consulta de entregas e reprocessamento manual.

Hipótese: SLA de 10s para primeiro envio será atendido com polling a cada 2s e worker único — validar com telemetria pós-implantação.

## 3. Escopo e exclusoes

Incluído:
- Tabela `outbox` no MySQL e migração correspondente.
- Módulo `src/modules/webhooks` com `controller`, `service`, `repository`, `schemas` (Zod) e rotas.
- Worker Node.js autônomo `src/worker.ts` (entrypoint separado), com a lógica de processamento em `src/modules/webhooks`, que faz polling a cada 2s.
- Assinatura HMAC-SHA256, headers de entrega, retries, DLQ, histórico de entregas e endpoints administrativos para replay.

Excluído (fora do MVP):
- Dashboard de clientes, email de notificação de falhas, rate limiting por cliente, escala multi-worker e ordenação global garantida.

## 4. Fluxos detalhados

### 4.1 Criação do evento na outbox

- Origem: chamada do fluxo de mudança de status em `OrderService.changeStatus()`.
- Entrada: objeto `orderId`, `previousStatus`, `newStatus`, `actorId` (quem mudou), `timestamp`.
- Validações:
  - Pedido existe e está em estado esperado.
  - Se `newStatus` gerar webhooks (ver configuração do endpoint / filtros), proceder.
- Processamento:
  1. Iniciar transação SQL via Prisma (usar transações existentes em `order.service.ts`).
  2. Atualizar registro do pedido e `order_status_history` (existente).
  3. Construir payload mínimo do evento (ver esquema abaixo).
  4. Inserir linha na tabela `webhook_outbox` com status `pending`, `attempts = 0`, `next_try_at = NOW()` e `created_at`.
- Payload mínimo (JSON):
  - `event_id` (UUID v4)
  - `event_type` = `order.status_changed`
  - `timestamp` (ISO8601)
  - `order_id`
  - `order_number`
  - `from_status`
  - `to_status`
  - `customer_id`
  - `total_cents`
  - Não incluir `items`; o consumidor pode consultar `GET /orders/:id` para detalhes.
- Limite: rejeitar payloads acima de 64KB; não truncar o evento.
- Persistência: a inserção da outbox deve estar na mesma transação que as alterações do pedido — se a transação der rollback, nada é escrito.
- Falhas: se inserção da outbox falhar (ex: unique constraint), abortar a transação e lançar erro tratável para o chamador (log e retry por chamada externa). Registrar métrica `webhook.outbox.insert_failed`.
- Saída esperada: linha criada em `webhook_outbox` com `status = pending`.

### 4.2 Processamento pelo worker

- Polling: a cada 2s, `WebhookWorker` executa:
  1. Selecionar até N eventos com `status = pending` e `next_try_at <= NOW()` ordenados por `created_at` (N configurável, default 20).
  2. Para cada evento, tentar adquirir lock lógico (UPDATE ... WHERE id = ? AND status = 'pending' AND locked_until < NOW() SET status='processing', locked_until = NOW()+lease).
  3. Carregar endpoints ativos associados ao tenant/order filter na tabela `webhook_endpoints`.
  4. Montar a requisição HTTP:
     - Método `POST` para `endpoint.url`
     - Headers obrigatórios: `Content-Type: application/json`, `X-Event-Id`, `X-Timestamp`, `X-Webhook-Id`, `X-Signature` (HMAC-SHA256)
     - Body: o payload persistido em outbox (ou versão compactada/limitada)
  5. Enviar com timeout de 10s por request e acompanhar código HTTP e latency.
  6. On success (HTTP 2xx): marcar entrega como `delivered`, incrementar `attempts`, salvar `delivered_at` e registro em `webhook_delivery_history`.
  7. On client error (HTTP 4xx): considerar não-recuperável, exceto 429; para 410/404 marcar `permanent_failed` e enviar para DLQ.
  8. On server error (HTTP 5xx) or timeout/network error: incrementar `attempts`, calcular `next_try_at` por backoff, set status back to `pending` e release lock.

- Timeout e cancelamento: usar request timeout configurável (`WEBHOOK_HTTP_TIMEOUT_MS`, default 10000ms). Se exceder, tratar como retriable network error.

- Observabilidade: cada tentativa gera log estruturado com `eventId`, `endpointId`, `attempt`, `statusCode`, `latencyMs`, `error`.

### 4.3 Retry com backoff

- Classificação de erro:
  - Recuperáveis: network errors, timeouts, HTTP 5xx, 429.
  - Não-recoveráveis: HTTP 4xx (exceto 429), invalid URL, signature mismatch at receiver (o receptor deve responder 4xx).
- Política:
  - Max attempts = 5 (configurável `WEBHOOK_MAX_ATTEMPTS`).
  - Backoff fixo entre tentativas: 1m, 5m, 30m, 2h e 12h, conforme a reunião.
  - `next_try_at` deve usar o intervalo correspondente à tentativa atual; não adicionar jitter nem calcular uma fórmula exponencial alternativa.
- Tratamento:
  - Se attempts < max: agendar next_try_at e status = `pending`.
  - Se attempts >= max: mover para DLQ (status `dead_letter`) e registrar em `webhook_dlq`.

### 4.4 DLQ

- Critério: attempts >= max OR resposta 410/404 sem chance de reativação.
- Dados preservados: evento original, endpointId, attempts, last_error, first_attempt_at, last_attempt_at, delivery_history (list), created_at.
- Consulta/Reprocessamento: API administrativa `POST /admin/webhooks/dead-letter/:id/replay` que recoloca o evento na outbox como pendente e reseta `attempts` (autorizada somente por role `ADMIN`). O replay deve preservar o `event_id` para permitir deduplicação pelo consumidor.
- Responsabilidade operacional: equipe de platforma deve monitorar métricas `webhook.dlq.size` e criar runbook para investigar e reprocessar manualmente.

### 4.5 Fluxos alternativos e excecoes

- Endpoint inativo/desativado: se `webhook_endpoints.active = false`, não criar outbox para aquele endpoint.
- Payload inválido: validação do payload antes de inserir; se inválido, não inserir e registrar `WEBHOOK_PAYLOAD_INVALID`.
- Secret: a plataforma gera a secret na criação do endpoint e a devolve uma única vez na resposta. Rotação posterior mantém a secret anterior válida por 24 horas.
- Consumidor indisponível: tratado via retries e DLQ.
- Resposta 4xx do consumidor: se 4xx != 429, tratar como não-recoverável (DLQ) exceto quando cliente solicitar reprocessamento manual.
- Duplicidade: consumidores devem deduplicar por `X-Event-Id`; eventos podem ser re-enviados (at-least-once).

## 5. Contratos públicos (APIs)

Observação: todas as rotas exigem autenticação `Bearer JWT`. O CRUD e a consulta de deliveries aceitam qualquer role autenticada; o replay de DLQ exige role `ADMIN`.

#### [POST] /webhooks

- Objetivo: criar um endpoint de webhook
- Autenticação: `Authorization: Bearer <token>` (role `ADMIN`)
- Headers: `Content-Type: application/json`
- Request:

```json
{
  "name": "orders-status",
  "url": "https://client.example.com/webhooks/orders",
  "statuses": ["SHIPPED", "DELIVERED"],
  "customerId": "customer-uuid",
  "active": true
}
```

- Response:

```json
{
  "id": "uuid",
  "name": "orders-status",
  "url": "https://client.example.com/webhooks/orders",
  "active": true,
  "secret": "generated-by-platform"
}
```

- Status codes:
  - 201 Created
  - 400 Bad Request (WEBHOOK_INVALID_URL)
  - 401 Unauthorized

- Semântica: valida URL (must be https), gera a secret, persiste `webhook_endpoints` e devolve a secret na criação.
- Erros WEBHOOK_ relacionados: `WEBHOOK_INVALID_URL`, `WEBHOOK_ENDPOINT_ALREADY_EXISTS`

#### [GET] /webhooks

- Objetivo: listar endpoints configurados
- Autenticação: `Authorization: Bearer <token>` (role `ADMIN`)
- Headers: opcional `page`, `limit`
- Request: vazio
- Response:

```json
[
  {"id":"uuid","name":"orders-status","url":"https://...","active":true}
]
```

- Status codes: 200 OK
- Erros: `WEBHOOK_ENDPOINT_NOT_FOUND` (se query por id)

#### [PATCH] /webhooks/:id

- Objetivo: atualizar nome/url/secret/active/filters
- Autenticação: `Authorization: Bearer <token>` (role `ADMIN`)
- Headers: `Content-Type: application/json`
- Request exemplo:

```json
{ "url": "https://new.example.com/hook", "active": false }
```

- Response: 200 com entidade atualizada
- Status codes: 200, 400 (WEBHOOK_INVALID_URL), 404 (WEBHOOK_ENDPOINT_NOT_FOUND)

- Erros: `WEBHOOK_ENDPOINT_NOT_FOUND`, `WEBHOOK_INVALID_URL`

#### [POST] /webhooks/:id/rotate-secret

- Objetivo: gerar uma nova secret para o endpoint
- Autenticação: `Authorization: Bearer <token>` (qualquer role autenticada)
- Response: 200 OK com a nova secret; a secret anterior permanece válida por 24 horas
- Erros: `WEBHOOK_ENDPOINT_NOT_FOUND`, `WEBHOOK_SECRET_ROTATION_FAILED`

#### [POST] /webhooks/:id/deactivate

- Objetivo: desativar rapidamente um endpoint (idempotente)
- Autenticação: `Authorization: Bearer <token>` (role `ADMIN`)
- Response: 200 OK
- Erros: `WEBHOOK_ENDPOINT_NOT_FOUND`

#### [GET] /webhooks/:id/deliveries

- Objetivo: consultar histórico de entregas
- Autenticação: `Authorization: Bearer <token>` (qualquer role autenticada)
- Semântica: retorna os últimos 100 deliveries do endpoint, incluindo payload, response, status, erro, attempt e latência.
- Response exemplo:

```json
{
  "eventId":"uuid",
  "deliveries":[
    {"attempt":1,"statusCode":500,"timestamp":"...","error":"timeout"},
    {"attempt":2,"statusCode":200,"timestamp":"..."}
  ]
}
```

- Status codes: 200, 404 (WEBHOOK_EVENT_NOT_FOUND)

#### [POST] /admin/webhooks/dead-letter/:id/replay

- Objetivo: reprocessar evento na DLQ, recolocando-o na outbox como pendente
- Autenticação: `Authorization: Bearer <token>` (role `ADMIN`)
- Response: 202 Accepted com `outboxId`
- Erros: `WEBHOOK_DLQ_REPLAY_FAILED`, `WEBHOOK_DLQ_NOT_FOUND`

Observação: para cada endpoint, o worker envia cabeçalhos de entrega:
- `X-Event-Id: <uuid>`
- `X-Timestamp: <ISO8601>`
- `X-Webhook-Id: <uuid>`
- `X-Signature: sha256=<hex-hmac>` (HMAC-SHA256 sobre body usando endpoint.secret)

Idempotencia: consumidores serão instruídos a deduplicar por `X-Event-Id`.

Versionamento: adicionar `X-Webhook-Version` header no futuro para breaking changes.

Rate limits: por agora, não implementado; observar volume e taxa de erro antes de decidir uma política por cliente.

## 6. Matriz de erros previstos

| Codigo | Condição | Tratamento | Retry | Resposta/efeito |
|---|---|---:|---:|---|
| WEBHOOK_ENDPOINT_NOT_FOUND | Endpoint id não existe | 400/404 para chamadas admin | no | registrar e retornar 404 |
| WEBHOOK_ENDPOINT_INACTIVE | Endpoint desativado | rejeitar criação de outbox para esse endpoint | no | evento não criado para esse endpoint |
| WEBHOOK_INVALID_URL | URL inválida (não-https ou parse fail) | validar e recusar criação | no | 400 |
| WEBHOOK_SECRET_ROTATION_FAILED | Falha na geração ou rotação da secret | não alterar endpoint | no | 500 |
| WEBHOOK_PAYLOAD_INVALID | Payload do evento inválido | não inserir outbox, notificar | no | 400 |
| WEBHOOK_PAYLOAD_TOO_LARGE | Payload excede 64KB | rejeitar evento sem truncar | no | 400 |
| WEBHOOK_SIGNATURE_FAILED | assinatura recebida inválida pelo cliente | considerar falha de entrega | no | registrar e contar como delivery failed |
| WEBHOOK_DELIVERY_TIMEOUT | Timeout de entrega | incrementar attempts e agendar retry | yes | retry/backoff |
| WEBHOOK_DELIVERY_FAILED | Erro de rede/5xx | incrementar attempts e agendar retry | yes | retry/backoff |
| WEBHOOK_MAX_RETRIES_EXCEEDED | Max attempts alcançado | mover para DLQ | no | alarmar e registrar |
| WEBHOOK_DLQ_REPROCESS_FAILED | Reprocess de DLQ falhou | registrar e notificar ops | no | 500/202 conforme caso |

## 7. Estratégias de resiliencia

- Timeouts: default HTTP timeout = 10000ms (10s); configurável `WEBHOOK_HTTP_TIMEOUT_MS`.
- Retries: max 5; backoff fixo de 1m/5m/30m/2h/12h, conforme decisão da reunião.
- Fallback: mover para DLQ; reprocessamento manual por `ADMIN`.
- Concorrência/locks: lease lock via `locked_until` column; UPDATE ... WHERE ... to acquire.
- Idempotência: deduplicação pelo consumidor usando `X-Event-Id`.
- Degradação segura: se DB indisponível, operação de mudança de status NÃO escreve outbox e falha a operação — prefer consistency. Hipótese: alternativa seria write-ahead log local, mas fica fora do MVP.
- Proteção contra consumidor lento: timeout + retries; consider rate limiting por cliente em fase 2.

## 8. Observabilidade

- Métricas (Prometheus style):
  - `webhook_outbox_pending_total{endpointId}` gauge
  - `webhook_delivery_attempts_total{endpointId,result=success|failure|timeout}` counter
  - `webhook_dlq_size_total` gauge
  - `webhook_outbox_insert_failures_total` counter
  - `webhook_delivery_latency_ms_bucket` histogram

- Labels: `endpointId`, `tenantId`, `eventType`, `result` (low cardinality)
- Logs (Pino structured): cada log inclui: `ts`, `level`, `service=webhook-worker|api`, `eventId`, `endpointId`, `attempt`, `statusCode`, `latencyMs`, `error`.
  - Não logar secrets ou body completo em logs de erro padrão; adicionar `webhook.secret` à configuração de redaction do logger.
- Tracing (opentelemetry): spans:
  - `OrderService.changeStatus` (existing) — adicionar attribute `eventId` quando outbox inserido.
  - `WebhookWorker.processEvent` — attributes: `eventId`, `endpointId`, `attempt`.
- Dashboards e alertas mínimos:
  - Alert: `webhook_dlq_size_total > 0` por mais de 10m -> P1
  - Alert: `rate(webhook_delivery_attempts_total{result="failure"}[5m]) > threshold` -> P2

## 9. Dependências e compatibilidade

- Banco: MySQL atual usado via Prisma — adicionar migration para `webhook_outbox`, `webhook_endpoints`, `webhook_delivery_history`, `webhook_dlq`.
- Variáveis de ambiente:
  - `WEBHOOK_POLL_INTERVAL_MS` (default 2000)
  - `WEBHOOK_HTTP_TIMEOUT_MS` (default 10000)
  - `WEBHOOK_MAX_ATTEMPTS` (default 5)
  - `WEBHOOK_WORKER_CONCURRENCY` (default 1, single-worker no MVP)
- Bibliotecas: `undici`, `crypto` (nativo) para HMAC, `pino` para logs, `opentelemetry` para traces.

Compatibilidade: não altera contratos existentes de `GET /orders`; adiciona colunas/tabelas e novo módulo `src/modules/webhooks`.

## 10. Integração com o sistema existente

Usar caminhos reais do repositório para integrar implementações:

- `Arquivo`: src/modules/orders/order.service.ts
  - Estado atual observado: contém a lógica de mudança de status e transações do pedido.
  - Mudança proposta: ao final da transação de mudança de status (antes do commit), inserir chamada ao `WebhookRepository.createOutboxEvent(tx, payload)` — usando a mesma transação Prisma.
  - Risco: alteração na transação pode introduzir regressões; cobrir com testes de integração.
  - Testes afetados: `tests/orders.test.ts`, adicionar casos que verificam criação de outbox.

- `Arquivo`: src/modules/orders/order.routes.ts
  - Estado atual observado: rota que expõe endpoints de pedidos (create, patch status).
  - Mudança proposta: garantir que erros de outbox lancem erros bem formatados e mapeáveis (HTTP 500) — preferir não expor detalhes.
  - Risco: alterar payloads de erro; revisar consumidores.
  - Testes afetados: `tests/orders.test.ts`.

- `Arquivo`: src/app.ts
  - Estado atual observado: inicialização do Express, middlewares (auth, error handler, request logger).
  - Mudança proposta: registrar o novo router de webhooks; o worker deve iniciar pelo entrypoint separado `src/worker.ts`, não pelo boot da API.
  - Risco: aumentar tempo de boot; usar feature flag.
  - Testes afetados: `tests/setup.ts` boot mocks; atualizar para suportar worker flag.

- `Arquivo`: prisma/schema.prisma
  - Estado atual observado: schema atual com modelos `Order`, etc.
  - Mudança proposta: adicionar modelos `WebhookEndpoint`, `WebhookOutbox`, `WebhookDeliveryHistory`, `WebhookDLQ`, com índices em `next_try_at` e `status`.
  - Risco: migrations e necessidade de manutenção de dados em produção.
  - Testes afetados: seed/migrations/seed.ts.

- `Arquivo`: src/shared/http/response.ts
  - Estado atual observado: utilitários para respostas HTTP padronizadas.
  - Mudança proposta: adicionar helpers para resposta de erro padronizados com códigos `WEBHOOK_*` nos corpos de erro e mapear para HTTP codes.
  - Risco: padronização do formato de erro; revisar uso global.

Hipótese: nomes e caminhos exatos conferidos via inspeção do workspace (existem `src/modules/orders/order.service.ts`, `src/app.ts`, `prisma/schema.prisma`, `src/shared/http/response.ts`).

## 11. Critérios de aceite técnicos

- Unitários:
  - `WebhookRepository.createOutboxEvent` tem testes unitários que simulam falha no insert.
  - `WebhookWorker` tem testes de retries e backoff (mocks de fetch retornando 500/timeout).
- Integração:
  - Ao executar mudança de status via endpoint de orders, existe uma linha em `webhook_outbox` criada dentro da mesma transação.
  - Worker processa evento com sucesso em cenário happy path, registro em `webhook_delivery_history` e `webhook_outbox.status = delivered`.
- Segurança:
  - Assinatura HMAC validada por exemplos nos docs; secret não logado.
- Observabilidade:
  - Métricas básicas expostas e alertas configurados (ex.: DLQ > 0).
- Migrations:
  - Migration SQL criada e aplicada com sucesso (local e CI).
- Testes automatizados:
  - Cobertura mínima: testes unitários para repository/service e testes de integração para worker (mock HTTP).

## 12. Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação | Plano de contingência |
|---|---:|---:|---|---|
| Duplicidade de entrega | Alta | Médio | Instruir consumidores a deduplicar por `X-Event-Id`; doc e exemplos | Retrocompat: reprocess manual via DLQ |
| Crescimento indefinido da outbox | Médio | Alto | Definir política de retenção antes do go-live; a reunião mencionou arquivamento após aproximadamente 30 dias, sem fixar prazo final | Arquivar e remover rows antigas após decisão operacional |
| Secret comprometido | Baixa | Alto | Rotação de secret com grace period; permitir revogação | Rotacionar e invalidar entregas antigas, notificar clientes |
| Consumidor instável gerando DLQ | Médio | Médio | Alertas em DLQ, limitar retries | Operação reprocessa manualmente; criar backoff mais agressivo |
| DB transaction regressions | Baixa | Alto | Cobertura de testes, revisão de PR e canary deploy | Rollback release e usar feature flag para outbox insertion |

---

## Anexos técnicos rápidos

- Exemplo SQL (migração simplificada):

```sql
CREATE TABLE webhook_endpoints (
  id CHAR(36) PRIMARY KEY,
  name VARCHAR(255),
  url TEXT NOT NULL,
  secret TEXT,
  filters JSON,
  active BOOLEAN DEFAULT TRUE,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE webhook_outbox (
  id CHAR(36) PRIMARY KEY,
  event_id CHAR(36) NOT NULL,
  event_type VARCHAR(100),
  payload JSON,
  status VARCHAR(32) DEFAULT 'pending',
  attempts INT DEFAULT 0,
  next_try_at DATETIME,
  locked_until DATETIME NULL,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_outbox_next_try ON webhook_outbox(next_try_at, status);
```

---

Se desejar, posso agora:
- Implementar a migration Prisma e os modelos em `prisma/schema.prisma`.
- Scaffolder o módulo `src/modules/webhooks` com arquivos iniciais.
# FDD — Feature Design Document

<!-- documento a ser elaborado -->
