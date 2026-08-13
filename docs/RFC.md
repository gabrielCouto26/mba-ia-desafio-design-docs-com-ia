# RFC - Webhooks de notificacao de pedidos

## 1. Metadados

- **Autor:** Larissa (Tech Lead)
- **Status:** Em revisao
- **Data:** 2026-08-13
- **Revisores:** Marcos (Product Manager), Bruno (Engenheiro Pleno - Pedidos), Diego (Engenheiro Senior - Plataforma), Sofia (Engenheira de Seguranca)

## 2. Resumo executivo (TL;DR)

- Clientes B2B precisam receber mudancas de status de pedidos em ate 10 segundos, sem depender de polling em `GET /orders`.
- Propomos webhooks outbound com captura transacional via outbox no MySQL existente e entrega por worker separado.
- A solucao preserva a consistencia do fluxo de pedidos e evita chamadas HTTP externas dentro da transacao critica de `OrderService.changeStatus`.
- O principal trade-off e aceitar entrega assincrona at-least-once, com possibilidade de duplicidade, em troca de resiliencia e menor acoplamento.
- A primeira fase inclui CRUD de configuracao, entrega assinada, retries, DLQ, replay administrativo e historico de entregas; email, dashboard e rate limiting ficam fora do escopo imediato.
- As proximas validacoes devem cobrir revisao de seguranca, volume real de entregas, politica de retencao da outbox/DLQ e necessidade futura de rate limiting por cliente.

## 3. Contexto e problema

Atlas Comercial, MaxDistribuicao e Nova Cargo solicitaram notificacoes em tempo quase real quando pedidos mudarem de status. Hoje, esses clientes consultam `GET /orders` periodicamente, o que torna a integracao mais lenta, aumenta custo operacional para os integradores e cria risco comercial, especialmente pela expectativa de entrega ate o fim do trimestre.

O sistema atual e uma API Node.js/TypeScript com Express, Prisma, MySQL e modulos por dominio em `src/modules`. O ponto critico da feature esta no modulo de pedidos: a mudanca de status ja atualiza o pedido, registra historico e ajusta estoque dentro de uma transacao. Acoplar esse fluxo a chamadas HTTP para endpoints externos introduziria latencia, falhas de rede e possiveis rollbacks indevidos em uma operacao central de negocio.

A proposta precisa, portanto, resolver duas necessidades ao mesmo tempo: emitir eventos confiaveis quando o status muda e entregar esses eventos a consumidores externos sem comprometer a estabilidade do fluxo de pedidos. A premissa aceita pela reuniao e que "tempo real" significa latencia percebida abaixo de 10 segundos para o primeiro envio.

## 4. Proposta tecnica (visao geral da solucao)

Propomos implementar um sistema de webhooks outbound para eventos de mudanca de status de pedidos, mantendo a API de pedidos como produtora confiavel do evento e delegando a entrega HTTP para um worker separado.

Os componentes principais sao:

- **Modulo de webhooks:** novo dominio em `src/modules/webhooks`, seguindo os padroes existentes de rotas, controller, service, repository e schemas Zod. Ele sera responsavel por configuracoes de endpoints, secrets, filtros de status, historico de entregas e operacoes administrativas de replay.
- **Outbox no MySQL:** tabela persistente para registrar eventos dentro da mesma transacao que altera o pedido e grava `order_status_history`. O evento deve representar um snapshot do pedido no momento da mudanca de status.
- **Worker de entrega:** processo Node separado da API, acionado por entrypoint proprio, que consulta eventos pendentes por polling a cada 2 segundos e envia chamadas HTTP aos endpoints configurados.
- **Entrega assinada:** cada endpoint de cliente tera secret propria. As chamadas outbound serao assinadas com HMAC-SHA256 e headers de identificacao e verificacao, incluindo `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id`.
- **Retry e DLQ:** falhas de entrega serao tratadas com ate 5 tentativas em backoff progressivo. Eventos esgotados serao preservados em dead letter queue separada, com replay manual restrito a `ADMIN`.
- **Observabilidade inicial:** o sistema deve expor historico de entregas para clientes e registrar logs estruturados com Pino para processamento, falhas, latencia de resposta e replays administrativos.

O fluxo de alto nivel e:

1. Uma mudanca valida de status ocorre no modulo de pedidos.
2. Na mesma transacao SQL, o sistema grava o novo status, o historico e o evento na outbox, somente para clientes/endpoints interessados naquele status.
3. O worker busca eventos pendentes, assina o payload e envia a requisicao HTTPS ao consumidor.
4. Em sucesso, a entrega e marcada para consulta posterior. Em falha ou timeout, o evento e reagendado conforme a politica de retry.
5. Apos esgotar tentativas, o evento vai para DLQ e pode ser reprocessado manualmente por endpoint administrativo.

Esta proposta nao resolve, nesta fase, dashboard visual para clientes, notificacao por email quando um webhook falha, rate limiting de saida por cliente, escalabilidade com multiplos workers nem uma garantia forte de ordenacao global. Esses pontos serao revisitados apos uso real e telemetria operacional.

## 5. Alternativas consideradas

### Alternativa: chamada HTTP sincrona no servico de pedidos

- **Descricao:** disparar o webhook diretamente durante a execucao de `changeStatus`, antes de finalizar a operacao de negocio.
- **Vantagem:** menor numero de componentes e latencia minima quando o endpoint do cliente responde rapidamente.
- **Motivo do descarte:** acopla a transacao de pedidos a rede externa. Um cliente lento ou indisponivel poderia travar mudancas de status e criar decisoes ambiguas sobre rollback de negocio.

### Alternativa: Redis Streams ou fila externa

- **Descricao:** publicar eventos de mudanca de status em infraestrutura dedicada de fila/stream e consumir por workers.
- **Vantagem:** oferece primitivas especializadas de mensageria, consumo e escalabilidade.
- **Motivo do descarte:** introduz nova infraestrutura para uma equipe pequena e para um requisito que pode ser atendido pelo MySQL ja operado pelo projeto. A reuniao classificou a abordagem como overengineering para a fase atual.

### Alternativa: trigger de banco para avisar o worker

- **Descricao:** usar mecanismos no banco para tentar acordar o worker quando uma linha nova for inserida.
- **Vantagem:** poderia reduzir consultas periodicas e tornar a entrega mais reativa.
- **Motivo do descarte:** MySQL nao oferece notificacao externa nativa equivalente ao `LISTEN/NOTIFY` do Postgres; improvisar esse comportamento aumentaria fragilidade operacional. Polling de 2 segundos atende ao SLA percebido.

### Alternativa: entrega exactly-once

- **Descricao:** garantir que cada consumidor receba cada evento uma unica vez.
- **Vantagem:** simplificaria a vida dos integradores ao eliminar deduplicacao no lado cliente.
- **Motivo do descarte:** exigiria coordenacao distribuida entre sistemas independentes e nao eliminaria completamente ambiguidades de rede. A equipe escolheu at-least-once com `X-Event-Id`, alinhado a padroes de mercado.

## 6. Questoes em aberto

### Rate limiting por cliente

- **Pergunta:** devemos limitar a taxa de entregas outbound por cliente ou endpoint?
- **Impacto de nao decidir:** clientes com alto volume podem receber rajadas de chamadas e responder com falhas, ampliando retries e ruido operacional.
- **Dono sugerido:** Diego (Plataforma), com apoio de Marcos para validar expectativa dos clientes.
- **Momento recomendado para decisao:** apos observar volume e taxa de erro nas primeiras integracoes B2B.

### Retencao e arquivamento de eventos entregues

- **Pergunta:** qual politica definitiva de retencao para outbox entregue, historico de entregas e DLQ?
- **Impacto de nao decidir:** crescimento indefinido das tabelas pode degradar consultas do worker, aumentar custo de armazenamento e dificultar suporte.
- **Dono sugerido:** Larissa e Diego.
- **Momento recomendado para decisao:** antes do go-live ou no desenho detalhado do FDD, validando requisitos de suporte e auditoria.

### Notificacao ativa de falhas ao cliente

- **Pergunta:** a plataforma deve enviar email ou outro alerta quando um webhook falhar repetidamente?
- **Impacto de nao decidir:** clientes podem descobrir problemas apenas consultando historico ou percebendo ausencia de eventos.
- **Dono sugerido:** Marcos.
- **Momento recomendado para decisao:** pos-MVP, depois de medir frequencia de falhas e necessidade de comunicacao proativa.

### Escala com multiplos workers e ordenacao

- **Pergunta:** quando e como particionar processamento para preservar ordenacao por pedido ao escalar workers?
- **Impacto de nao decidir:** a fase inicial fica limitada a single-worker para preservar comportamento previsivel; escala futura pode quebrar ordenacao se for feita sem estrategia.
- **Dono sugerido:** Diego.
- **Momento recomendado para decisao:** quando throughput real exigir mais de um worker ou batches maiores.

## 7. Impacto e riscos

**Produto:** a feature reduz dependencia de polling e atende uma demanda explicita de clientes B2B estrategicos. O risco principal e vender "tempo real" como garantia absoluta; a comunicacao deve deixar claro que o primeiro envio mira ate 10 segundos e que retries podem atrasar eventos durante indisponibilidade do consumidor.

**Engenharia:** a solucao adiciona um dominio de webhooks e um processo assincromo, mas preserva a arquitetura atual do projeto. O maior cuidado e manter a insercao da outbox dentro da transacao de pedidos sem espalhar responsabilidades de entrega pelo modulo de orders.

**Operacao:** a plataforma passa a depender de um worker saudavel, politicas de retry, DLQ e retencao. A mitigacao e tratar o worker como componente operacional proprio, com logs estruturados, historico consultavel e processo manual de replay.

**Seguranca:** payloads de pedidos passam a sair da infraestrutura para endpoints externos. A mitigacao inclui HTTPS obrigatorio, HMAC-SHA256 por endpoint, secret rotacionavel com grace period, limite de payload e revisao de Sofia antes do deploy.

**Clientes integradores:** consumidores passam a receber eventos de forma mais eficiente, mas precisam validar assinatura, tolerar duplicidade e deduplicar por `X-Event-Id`. A mitigacao e documentar esse contrato no portal de desenvolvedor e manter payload enxuto, com consulta posterior a `GET /orders/:id` quando precisarem de detalhes.

## 8. Decisoes relacionadas

- [ADR-001: Outbox no MySQL](./adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002: Politica de retry com backoff e DLQ](./adrs/ADR-002-politica-de-retry-backoff-e-dlq.md)
- [ADR-003: Assinatura HMAC-SHA256 por endpoint](./adrs/ADR-003-assinatura-hmac-sha256-por-endpoint.md)
- [ADR-004: Entrega at-least-once com X-Event-Id](./adrs/ADR-004-entrega-at-least-once-com-x-event-id.md)
- [ADR-005: Worker separado com polling](./adrs/ADR-005-worker-separado-com-polling.md)
- [ADR-006: Reuso de padroes existentes do projeto](./adrs/ADR-006-reuso-de-padroes-existentes-do-projeto.md)
