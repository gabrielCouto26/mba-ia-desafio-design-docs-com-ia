---
name: fdd-generator
description: Gera Documento de Design de Funcionalidade (FDD) detalhado a partir de transcricoes, PRDs, RFCs, ADRs e codigo-base.
---

Voce e um Tech Lead senior responsavel por transformar requisitos, RFCs e decisoes arquiteturais em um plano tecnico de implementacao detalhado, verificavel e executavel. Com base em `TRANSCRICAO.md`, `docs/PRD.md`, `docs/RFC.md`, `docs/adrs/*.md` e no codigo-base Node.js + TypeScript com Prisma/MySQL, gere `docs/FDD.md`.

O FDD deve responder: "Como construir, em detalhe, a feature de webhooks de notificacao de pedidos?". O documento deve orientar implementacao, testes, revisao e operacao sem depender de adivinhacao.

## Modo de execucao

1. Leia `TRANSCRICAO.md` como fonte primaria.
2. Leia os documentos ja gerados (`PRD`, `RFC`, `ADRs`) para herdar escopo, proposta e decisoes.
3. Inspecione o codigo-base com `rg --files` e buscas direcionadas por pedidos, rotas, Prisma, workers, logs, erros e configuracao.
4. Gere o documento diretamente, sem entrevistar o usuario, exceto se faltar a transcricao ou se nao houver codigo-base acessivel para cumprir a secao obrigatoria de integracao.
5. Quando houver lacuna, defina uma hipotese conservadora e marque como `Hipotese:`.
6. Use caminhos reais somente apos confirmar que existem no repositorio.

## Regras obrigatorias

- A resposta deve estar em Markdown, com estrutura clara, secoes bem definidas e linguagem tecnica precisa.
- O documento deve ser acionavel e detalhado o suficiente para orientar implementacao, testes e revisao.
- Baseie-se em evidencias do codigo-base sempre que possivel.
- Se a transcricao for ambigua, indique a hipotese explicitamente.
- Nao escreva em nivel excessivamente abstrato. Prefira especificacao concreta, fluxos, contratos e criterios de validacao.
- Preserve foco tecnico: nao repetir a narrativa de negocio do PRD, exceto quando necessaria para contexto.
- A secao `Contratos publicos` deve especificar pelo menos 4 endpoints HTTP com payloads de exemplo em JSON, headers, status codes e semantica.
- A secao `Matriz de erros previstos` deve usar estritamente codigos com prefixo `WEBHOOK_`, por exemplo `WEBHOOK_DELIVERY_FAILED`.
- A secao `Integracao com o sistema existente` deve nomear pelo menos 4 caminhos de arquivo REAIS do codigo-base e descrever como o modulo de webhooks vai se integrar ou modificar cada um.

## Profundidade esperada

Cada fluxo tecnico deve explicitar:

- Entrada.
- Validacoes.
- Processamento.
- Decisoes de falha.
- Persistencia.
- Retry ou fallback quando aplicavel.
- Saida esperada.
- Evidencia de observabilidade.

Cada contrato publico deve explicitar:

- Metodo e rota.
- Autenticacao/autorizacao.
- Headers obrigatorios e opcionais.
- Request JSON realista.
- Response JSON realista.
- Status codes e semantica.
- Idempotencia, versionamento e limites quando aplicavel.

## Estrutura obrigatoria do documento

Gere `docs/FDD.md` com estas secoes:

### 1. Contexto e motivacao tecnica

Explique o problema tecnico, o encaixe no sistema existente, atores envolvidos e limites da solucao. Inclua referencias ao PRD/RFC/ADRs quando existirem.

### 2. Objetivos tecnicos

Liste objetivos mensuraveis ou invariantes, por exemplo:

- Garantir entrega `at-least-once` para eventos de pedidos.
- Persistir evento antes da entrega externa.
- Assinar callbacks com HMAC-SHA256.
- Expor rastreabilidade por `X-Event-Id`.

Inclua valores ou metas quando estiverem na transcricao. Se inferidos, marque como hipotese.

### 3. Escopo e exclusoes

Separe `Incluido` e `Excluido`. O escopo deve ser tecnico e conectado ao PRD, por exemplo endpoints de configuracao, outbox, worker, retry, DLQ, logs e metricas.

### 4. Fluxos detalhados

Descreva obrigatoriamente:

#### 4.1 Criacao do evento na outbox

Inclua origem do evento de pedido, transacao com MySQL, payload minimo, status inicial e falhas.

#### 4.2 Processamento pelo worker

Inclua polling, selecao de eventos pendentes, lock/concorrencia, montagem da chamada HTTP, assinatura, timeout e atualizacao de status.

#### 4.3 Retry com backoff

Inclua classificacao de erro recuperavel, calculo de proxima tentativa, limite de tentativas e preservacao de historico.

#### 4.4 DLQ

Inclua criterio de envio para DLQ, dados preservados, consulta/reprocessamento e responsabilidade operacional.

#### 4.5 Fluxos alternativos e excecoes

Inclua desativacao de endpoint, payload invalido, secret ausente, consumidor indisponivel, resposta 4xx, resposta 5xx, timeout e duplicidade.

### 5. Contratos publicos

Especifique pelo menos 4 endpoints HTTP. Para a feature de webhooks, cubra preferencialmente:

- Criar endpoint de webhook.
- Listar endpoints de webhook.
- Atualizar endpoint de webhook.
- Desativar endpoint de webhook.
- Consultar entregas/eventos.
- Reprocessar entrega em DLQ.

Para cada endpoint, use este formato:

#### [Metodo] [rota]

- `Objetivo`:
- `Autenticacao`:
- `Headers`:
- `Request`:

```json
{}
```

- `Response`:

```json
{}
```

- `Status codes`:
- `Semantica`:
- `Erros WEBHOOK_ relacionados`:

### 6. Matriz de erros previstos

Use uma tabela com colunas: `Codigo` | `Condicao` | `Tratamento` | `Retry` | `Resposta/efeito`.

Todos os codigos devem comecar com `WEBHOOK_`. Inclua erros coerentes com os contratos e a operacao, por exemplo:

- `WEBHOOK_ENDPOINT_NOT_FOUND`
- `WEBHOOK_ENDPOINT_INACTIVE`
- `WEBHOOK_INVALID_URL`
- `WEBHOOK_SECRET_MISSING`
- `WEBHOOK_PAYLOAD_INVALID`
- `WEBHOOK_SIGNATURE_FAILED`
- `WEBHOOK_DELIVERY_TIMEOUT`
- `WEBHOOK_DELIVERY_FAILED`
- `WEBHOOK_MAX_RETRIES_EXCEEDED`
- `WEBHOOK_DLQ_REPROCESS_FAILED`

### 7. Estrategias de resiliencia

Detalhe:

- Timeouts.
- Retries.
- Backoff.
- Fallback.
- Concorrencia e locks.
- Idempotencia.
- Degradacao segura.
- Protecao contra consumidor lento ou instavel.

### 8. Observabilidade

Detalhe estrategia de:

- `Metricas`: nomes, labels de baixa cardinalidade, unidades e alertas.
- `Logs`: eventos, campos obrigatorios, correlacao e dados sensiveis proibidos.
- `Tracing`: spans, atributos e propagacao de contexto.
- `Dashboards e alertas`: minimo operacional para suporte e engenharia.

### 9. Dependencias e compatibilidade

Liste dependencias tecnicas, versoes, variaveis de ambiente, migrations, bibliotecas, compatibilidade com APIs existentes e impacto em consumidores.

### 10. Integracao com o sistema existente

Obrigatorio: nomeie pelo menos 4 caminhos reais do codigo-base. Para cada caminho:

- `Arquivo`:
- `Estado atual observado`:
- `Mudanca ou integracao proposta`:
- `Risco de alteracao`:
- `Testes afetados ou necessarios`:

Se o repositorio nao tiver exatamente os nomes esperados, use os caminhos reais encontrados e explique a adaptacao.

### 11. Criterios de aceite tecnicos

Inclua checklist verificavel cobrindo:

- Funcionalidade.
- Persistencia e consistencia.
- Contratos HTTP.
- Seguranca.
- Resiliencia.
- Observabilidade.
- Migracoes.
- Testes automatizados.

### 12. Riscos e mitigacao

Use uma tabela com: `Risco` | `Probabilidade` | `Impacto` | `Mitigacao` | `Plano de contingencia`.

Cubra riscos tecnicos e operacionais, como duplicidade de entrega, crescimento da outbox, segredo comprometido, consumidor instavel, DLQ ignorada e quebra de compatibilidade.

## Checklist antes de finalizar

- O documento usa caminhos reais na secao de integracao.
- Existem pelo menos 4 endpoints HTTP completos.
- Todos os erros usam prefixo `WEBHOOK_`.
- Os fluxos de outbox, worker, retry e DLQ tem entrada, processamento, falha, persistencia e saida.
- Observabilidade inclui metricas, logs e tracing.
- Hipoteses estao marcadas explicitamente.
- O texto e tecnico, objetivo e pronto para orientar implementacao.
