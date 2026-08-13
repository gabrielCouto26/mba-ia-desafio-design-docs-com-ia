# ADR-005: Worker separado com polling

## Status

Aceito

## Contexto

A outbox exige um processo responsável por buscar eventos pendentes e enviá-los para endpoints externos. A reunião definiu que o primeiro envio deve atender ao requisito de percepção de tempo real dos clientes, aceitando atraso abaixo de 10 segundos. Também foi apontado que MySQL não oferece mecanismo nativo equivalente a `LISTEN/NOTIFY` para acordar um processo externo.

O projeto atual possui `src/server.ts` como entrypoint HTTP, inicializando Express, Prisma e logger. Não existe worker de background implementado. A reunião propôs criar uma entrypoint separada, por exemplo `src/worker.ts`, reutilizando a mesma stack e a mesma `DATABASE_URL`, mas em processo Node independente.

## Decisão

Executar o processamento de webhooks em um worker separado do processo da API. O worker deve consultar a tabela de outbox por polling a cada 2 segundos, buscar eventos pendentes em batches pequenos, enviar as requisições HTTP e atualizar o estado de entrega.

O worker deve usar sua própria instância de `PrismaClient`, pois roda em outro processo, mas deve apontar para o mesmo banco da API.

## Alternativas Consideradas

### Executar o worker dentro do processo da API

- Vantagem: implantação inicial mais simples, com menos processos para gerenciar.
- Desvantagem: mistura responsabilidades, acopla ciclo de vida da API e processamento assíncrono, e dificulta escalar ou reiniciar cada parte separadamente.

### Trigger de banco para notificar o worker

- Vantagem: reduziria latência e evitaria consultas periódicas sem trabalho.
- Desvantagem: MySQL não fornece notificação externa nativa para esse caso; improvisar com triggers aumentaria complexidade e fragilidade.

## Consequências

### Positivas

- Isola chamadas HTTP externas do processo da API.
- Permite operar, reiniciar e escalar o worker separadamente no futuro.
- Polling de 2 segundos atende ao requisito de entrega percebida como tempo real.
- Mantém implementação simples sem introduzir nova infraestrutura.

### Negativas

- A operação passa a depender de um segundo processo ativo.
- Polling gera consultas periódicas mesmo quando não há eventos.
- A latência mínima prática passa a depender do intervalo de polling e do tamanho dos batches.
- Múltiplos workers no futuro podem afetar ordering se não houver particionamento ou locking por pedido.

## Evidências e rastreabilidade

- Transcrição: [09:08]-[09:11], decisão por polling a cada 2 segundos e processo separado; [09:11], proposta de `src/worker.ts` e script `npm run worker`; [09:29]-[09:30], PrismaClient separado por processo.
- Código-base: `src/server.ts` é o entrypoint HTTP atual; `src/config/database.ts` fornece conexão Prisma; `src/shared/logger/index.ts` fornece logger reutilizável.
- Documentos relacionados: `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` existem, mas ainda estão como placeholders.
