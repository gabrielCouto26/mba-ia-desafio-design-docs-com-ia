# ADR-004: Entrega at-least-once com X-Event-Id

## Status

Aceito

## Contexto

Em integrações por webhook, falhas de rede, timeouts e respostas ambíguas podem fazer com que o sistema não saiba se o cliente recebeu o evento. Para evitar perda silenciosa, a reunião definiu que eventos podem ser reenviados. Isso implica que o consumidor deve tratar duplicidade.

A alternativa de entrega exactly-once foi considerada complexa porque exigiria coordenação forte entre a plataforma e cada cliente, além de protocolo de confirmação e deduplicação compartilhado. Para os clientes citados, a necessidade central é serem notificados rapidamente sobre mudanças de status, não obter uma garantia transacional distribuída.

## Decisão

Adotar garantia de entrega at-least-once para webhooks de pedidos. O sistema pode entregar o mesmo evento mais de uma vez em cenários de retry, timeout ou falha de confirmação.

Cada evento deve receber um UUID único no momento de inserção na outbox. Esse identificador deve ser enviado no header `X-Event-Id` e também no payload. O consumidor deve usar esse valor para deduplicar eventos recebidos repetidamente.

Os demais headers mínimos definidos para a entrega são `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`.

## Alternativas Consideradas

### Entrega exactly-once

- Vantagem: simplificaria a lógica dos consumidores, que não precisariam lidar com duplicidades.
- Desvantagem: exigiria coordenação distribuída entre sistemas independentes, armazenamento de confirmações e protocolo mais complexo, sem eliminar totalmente ambiguidades de rede.

### Entrega best-effort sem retry

- Vantagem: implementação simples e sem duplicidade causada pela plataforma.
- Desvantagem: eventos seriam perdidos em qualquer falha temporária do cliente ou da rede.

## Consequências

### Positivas

- Reduz risco de perda de evento em falhas temporárias.
- Define contrato claro para idempotência no consumidor.
- Mantém a arquitetura compatível com retry e DLQ.
- Usa UUID, padrão já adotado nos modelos existentes do projeto.

### Negativas

- Clientes precisam persistir ou cachear `X-Event-Id` para deduplicação.
- Reenvios podem causar efeitos duplicados em consumidores que ignorarem o contrato.
- A documentação pública da API precisa destacar a semântica at-least-once.
- Ordering não é garantido globalmente; por enquanto depende de single-worker e ordenação por criação.

## Evidências e rastreabilidade

- Transcrição: [09:24]-[09:26], escolha de at-least-once e deduplicação por `X-Event-Id`; [09:44]-[09:45], lista de headers; [09:12]-[09:14], limitação de ordering enquanto houver single-worker.
- Código-base: `prisma/schema.prisma` usa `@default(uuid())` para entidades principais, reforçando UUID como padrão de identificação.
- Documentos relacionados: `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` existem, mas ainda estão como placeholders.
