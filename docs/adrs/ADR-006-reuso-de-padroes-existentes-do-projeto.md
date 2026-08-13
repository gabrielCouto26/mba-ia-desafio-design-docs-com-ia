# ADR-006: Reuso de padrões existentes do projeto

## Status

Aceito

## Contexto

A reunião decidiu que a feature de webhooks deve entrar no projeto sem criar uma arquitetura paralela. O código-base já apresenta convenções consistentes: módulos por domínio em `src/modules`, rotas Express, schemas Zod, controllers, services, repositories, Prisma, autenticação JWT, `requireRole`, `AppError`, middleware centralizado de erro e logger Pino.

Esses padrões aparecem, por exemplo, em `src/modules/orders/order.routes.ts`, `src/modules/orders/order.service.ts`, `src/modules/orders/order.repository.ts`, `src/middlewares/validate.middleware.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/auth.middleware.ts` e `src/shared/logger/index.ts`.

## Decisão

Implementar webhooks como um novo módulo `src/modules/webhooks`, seguindo a estrutura existente de controller, service, repository, routes e schemas. As rotas de gestão devem ser registradas em `src/routes/index.ts` sob `/webhooks`, reutilizando autenticação JWT e validação Zod.

Erros específicos devem seguir o padrão `AppError` e usar prefixo `WEBHOOK_` nos códigos. Logs devem usar Pino por meio de `src/shared/logger/index.ts`. O replay manual de DLQ deve reutilizar `requireRole('ADMIN')`.

O processamento assíncrono deve ficar em entrypoint separado, mas a lógica de domínio deve permanecer no módulo de webhooks para preservar coesão.

## Alternativas Consideradas

### Criar uma subaplicação isolada para webhooks

- Vantagem: isolamento forte e liberdade para escolher estrutura própria.
- Desvantagem: duplicaria padrões de autenticação, erro, validação, logs e acesso a dados, aumentando custo de manutenção.

### Implementar tudo dentro do módulo de pedidos

- Vantagem: reduz a quantidade inicial de arquivos e mantém perto do ponto de emissão dos eventos.
- Desvantagem: mistura gestão de endpoints, segurança, entregas, DLQ e processamento HTTP com regras de negócio de pedidos.

## Consequências

### Positivas

- Mantém consistência para manutenção futura.
- Reduz curva de aprendizado para o time que já conhece a estrutura atual.
- Reaproveita middlewares e padrões de erro testados.
- Evita introdução desnecessária de frameworks ou bibliotecas estruturais.

### Negativas

- O módulo de webhooks precisará integrar-se cuidadosamente com `OrderService.changeStatus` sem acoplar excessivamente os domínios.
- O padrão atual de injeção manual de dependências em `src/app.ts` pode crescer e ficar mais verboso.
- O worker separado exigirá um bootstrap próprio além do `src/server.ts`.
- O reaproveitamento dos padrões atuais pode limitar otimizações específicas de mensageria até que haja necessidade real.

## Evidências e rastreabilidade

- Transcrição: [09:27]-[09:30], decisão por `src/modules/webhooks`, reuso de `AppError`, Pino, middleware de erro, schemas Zod, códigos `WEBHOOK_` e Prisma; [09:35]-[09:36], replay com role `ADMIN`.
- Código-base: `src/app.ts`, `src/routes/index.ts`, `src/modules/orders/order.routes.ts`, `src/modules/orders/order.service.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/error.middleware.ts`, `src/shared/errors/index.ts`, `src/shared/logger/index.ts`.
- Documentos relacionados: `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` existem, mas ainda estão como placeholders.
