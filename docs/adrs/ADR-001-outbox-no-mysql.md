# ADR-001: Outbox no MySQL

## Status

Aceito

## Contexto

Clientes B2B precisam receber notificações de mudança de status de pedidos com atraso inferior a 10 segundos. A transcrição registra que a mudança de status atual já executa operações críticas na mesma transação: atualização de `orders`, criação de histórico e movimentação de estoque. Adicionar chamadas HTTP síncronas nesse fluxo tornaria a alteração de status dependente da disponibilidade e latência dos endpoints externos.

O código existente confirma esse ponto de integração em `src/modules/orders/order.service.ts`: o método `changeStatus` usa `this.prisma.$transaction`, atualiza o pedido, grava `orderStatusHistory` e debita ou repõe estoque. O projeto usa MySQL via Prisma, conforme `prisma/schema.prisma`, com `datasource db provider = "mysql"`.

Hipótese: a nova tabela de outbox ainda será criada, pois o estado atual do projeto não implementa webhooks nem outbox.

## Decisão

Adotar o padrão Outbox no MySQL existente. Sempre que o status de um pedido mudar, o sistema deve inserir um evento em uma tabela `webhook_outbox` dentro da mesma transação SQL que altera o pedido e grava o histórico. O evento deve conter um snapshot do payload no momento da mudança de status.

Se a transação principal fizer commit, o evento estará persistido para entrega posterior. Se a transação fizer rollback, o evento também será descartado.

## Alternativas Consideradas

### Chamada HTTP síncrona no serviço de pedidos

- Vantagem: implementação inicial mais simples e menor latência quando o endpoint do cliente responde rapidamente.
- Desvantagem: acopla a transação de negócio à rede externa; clientes lentos ou indisponíveis bloqueariam mudanças de status e poderiam provocar rollback indevido.

### Redis Streams ou fila externa

- Vantagem: oferece primitivas específicas de fila, consumo e escalabilidade.
- Desvantagem: exige nova infraestrutura operacional para uma equipe pequena e para um volume que pode ser atendido pelo MySQL já existente.

## Consequências

### Positivas

- Preserva consistência entre mudança de status e registro do evento.
- Evita chamadas HTTP externas dentro da transação crítica de pedidos.
- Reaproveita a infraestrutura MySQL e Prisma já adotada pelo projeto.
- Permite recuperação posterior pelo worker mesmo após falhas temporárias.

### Negativas

- Aumenta o modelo de dados com tabelas, índices e políticas de retenção para outbox.
- Exige worker assíncrono e monitoramento operacional próprios.
- Introduz latência mínima de entrega associada ao ciclo de polling.
- O crescimento da tabela precisa ser controlado por arquivamento ou expurgo de eventos entregues.

## Evidências e rastreabilidade

- Transcrição: [09:06]-[09:08], decisão por outbox em MySQL com índices por status e `created_at`; [09:40]-[09:41], inserção do evento dentro da transação de `changeStatus`; [09:51]-[09:52], UUID e snapshot do payload.
- Código-base: `src/modules/orders/order.service.ts`, `prisma/schema.prisma`.
- Documentos relacionados: `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` existem, mas ainda estão como placeholders.
