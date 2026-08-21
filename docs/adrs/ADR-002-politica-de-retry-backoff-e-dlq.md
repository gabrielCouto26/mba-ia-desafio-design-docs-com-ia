# ADR-002: Política de retry com backoff e DLQ

## Status

Aceito

## Contexto

Endpoints de clientes podem ficar temporariamente indisponíveis, responder com erro ou exceder timeout. A reunião descartou falha imediata como comportamento final, pois clientes podem ter manutenções de algumas horas. Também foi considerado que retentativas infinitas deixariam eventos pendurados indefinidamente e dificultariam operação.

O requisito de negócio aceita atraso inferior a 10 segundos para o primeiro envio, mas falhas de entrega precisam de uma estratégia explícita de recuperação. A transcrição também define que eventos com falha permanente devem ser preservados para diagnóstico e replay manual.

Hipótese: a DLQ será implementada como nova tabela Prisma/MySQL, pois ainda não há modelos de webhook em `prisma/schema.prisma`.

## Decisão

Implementar retry com intervalos fixos para entregas de webhook. Cada evento deve ter até 5 tentativas, com a agenda definida na reunião: 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas. Não aplicar jitter nem uma fórmula exponencial alternativa.

Após esgotar as tentativas, o evento deve ser movido para uma tabela separada `webhook_dead_letter`, preservando payload, motivo da falha, timestamp e dados úteis para diagnóstico. O reprocessamento deve ser manual por endpoint administrativo, exigindo role `ADMIN`.

## Alternativas Consideradas

### Retry indefinido com backoff

- Vantagem: maximiza a chance de entrega futura se o cliente voltar depois de um longo período.
- Desvantagem: mantém eventos vivos sem limite, dificulta limpeza operacional e pode mascarar integrações quebradas permanentemente.

### Apenas 3 tentativas

- Vantagem: reduz volume de retries e acelera classificação de falha permanente.
- Desvantagem: janela curta demais para indisponibilidades de manutenção já observadas em clientes.

## Consequências

### Positivas

- Dá resiliência a falhas temporárias de clientes sem bloquear o fluxo de pedidos.
- Cria limite operacional claro para eventos que não puderam ser entregues.
- Mantém evidência persistida para suporte, auditoria e replay manual.
- Reduz ruído na outbox principal ao separar falhas permanentes em DLQ.

### Negativas

- Eventos podem demorar muitas horas para serem classificados como falha final.
- A DLQ exige endpoints administrativos, autorização, logs e processo operacional de replay.
- O worker precisa controlar agenda de próximas tentativas, contador de retries e motivo da última falha.
- Clientes podem receber eventos antigos após recuperação, exigindo deduplicação e tratamento de ordem temporal.

## Evidências e rastreabilidade

- Transcrição: [09:15]-[09:18], escolha de 5 tentativas com backoff 1m/5m/30m/2h/12h e DLQ separada; [09:18]-[09:19], replay manual por endpoint admin; [09:35]-[09:36], replay exige role `ADMIN`.
- Código-base: `src/middlewares/auth.middleware.ts` já expõe `requireRole`; `src/shared/errors/index.ts` e `src/middlewares/error.middleware.ts` já oferecem padrão para erros administrativos e validação.
- Documentos relacionados: `docs/PRD.md`, `docs/RFC.md` e `docs/FDD.md` detalham os requisitos e a proposta de implementação.
