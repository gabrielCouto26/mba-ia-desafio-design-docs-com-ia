# ADR-003: Assinatura HMAC-SHA256 por endpoint

## Status

Aceito

## Contexto

Webhooks de pedidos são chamadas outbound para endpoints fora da infraestrutura do projeto. Os clientes precisam validar que a requisição veio da plataforma e que o payload não foi adulterado em trânsito. A reunião também registrou o risco de secrets vazadas em logs de clientes, tornando inadequada uma credencial global compartilhada por todos os endpoints.

O projeto já usa autenticação JWT para APIs internas e de gestão, mas esse mecanismo não resolve a autenticação da chamada outbound enviada ao servidor do cliente. Para webhooks, a verificação precisa ser feita pelo consumidor com base no corpo recebido e em uma secret compartilhada.

Hipótese: a geração, armazenamento e rotação de secrets serão adicionados ao novo módulo `src/modules/webhooks`, seguindo os padrões de schemas e serviços já presentes em outros módulos.

## Decisão

Assinar cada entrega de webhook com HMAC-SHA256 calculado sobre o corpo JSON enviado. A assinatura deve ser enviada no header `X-Signature`.

Cada endpoint cadastrado pelo cliente deve possuir uma secret própria, não global. A secret deve ser gerada pela plataforma, associada ao endpoint e rotacionável. Durante rotação, a secret antiga deve permanecer válida por 24 horas para permitir migração segura pelo cliente.

Também será obrigatório cadastrar URLs HTTPS, e cada envio deve incluir `X-Timestamp` para permitir mitigação de replay attack pelo consumidor.

## Alternativas Consideradas

### Secret global da plataforma

- Vantagem: reduz complexidade de armazenamento e rotação.
- Desvantagem: vazamento de uma única secret comprometeria todos os clientes e endpoints.

### JWT assinado para cada webhook

- Vantagem: aproveita conceito já familiar no projeto e permitiria claims estruturadas.
- Desvantagem: aumenta complexidade para clientes e não é necessário para validar integridade do payload; HMAC é mais simples e amplamente suportado para webhooks.

## Consequências

### Positivas

- Permite ao cliente verificar origem e integridade da mensagem.
- Limita o impacto de vazamento de credencial a um endpoint específico.
- A rotação com grace period reduz risco operacional durante troca de secrets.
- Alinha a integração ao padrão comum de mercado para webhooks.

### Negativas

- Exige armazenamento seguro e política de rotação para secrets.
- O cliente passa a ser responsável por implementar verificação da assinatura.
- A janela de 24 horas com duas secrets válidas aumenta temporariamente a superfície de aceitação.
- Mudanças no formato exato do corpo alteram a assinatura, exigindo disciplina de serialização.

## Evidências e rastreabilidade

- Transcrição: [09:19]-[09:22], decisão por HMAC-SHA256, secret por endpoint e rotação com grace period de 24h; [09:23], URL HTTPS obrigatória; [09:44], headers `X-Signature` e `X-Timestamp`.
- Código-base: `src/modules/auth/auth.service.ts` e `src/middlewares/auth.middleware.ts` cobrem autenticação da API, mas não chamadas outbound; `src/shared/logger/index.ts` redige tokens e senhas, e deverá ser estendido para secrets de webhook.
- Documentos relacionados: `docs/PRD.md`, `docs/RFC.md` e `docs/FDD.md` detalham os requisitos e a proposta de implementação.
