## Processo: da transcrição à suite de design docs

Este repositório contém a entrega do desafio: transformar a transcrição da reunião técnica e o código existente numa suite coerente de documentação (PRD, RFC, FDD, ADRs e TRACKER). O objetivo do README é explicar, de forma prática e verificável, como a transcrição foi convertida em documentos acionáveis, quais evidências foram usadas e quais prompts e iterações guiaram o trabalho.

O ponto de partida foi a transcrição (`TRANSCRICAO.md`) e o código em `src/` (inspeção pontual para identificar pontos de integração), além dos documentos já gerados em `docs/`. O resultado buscado é rastreabilidade: cada requisito ou decisão citada nos docs tem origem na transcrição ou em um arquivo do código.

**1. Sobre o desafio**

- Missão: extrair requisitos e decisões da reunião e produzir PRD, RFC, FDD, ADRs e um TRACKER que ligue cada item à sua fonte.
- Papel da transcrição: fonte primária de requisitos, decisões e questões em aberto; usada para construir PRD/RFC/ADRs e para preencher o TRACKER.
- Papel do código-base: evidência e ponto de integração (ex.: `src/modules/orders/order.service.ts`, `src/app.ts`, `prisma/schema.prisma`, `src/shared/http/response.ts`) para garantir que o FDD aponte caminhos reais.

**2. Ferramentas de IA utilizadas**

- Geração de skills e documentos: Gemini/Antigravity, Codex e GitHub Copilot.
- Inspeção do repositório local (leitura de `TRANSCRICAO.md`, `docs/*`, `src/*`, `prisma/*`) para validar paths e evitar referências inexistentes.

**3. Workflow adotado**

1. Ler e anotar a transcrição para identificar requisitos, decisões e perguntas em aberto.
2. Inspecionar o código para localizar pontos de integração e caminhos reais que serão citados no FDD/ADRs.
3. Produzir ADRs (decisões), em seguida RFC, depois FDD (detalhes de implementação) e PRD (consolidação de produto).  
4. Gerar o TRACKER para mapear cada item à transcrição ou ao código.  
5. Iterar: ajustar prompts e refinar documentos até atingir consistência e rastreabilidade.

Essa ordem minimiza retrabalho porque ADRs fixam decisões que guiam o FDD, e o TRACKER funciona como controle de integridade.

**4. Prompts customizados (exemplos práticos)**

- PRD / RFC prompt (produto → proposta):

```
Você é um redator técnico. Entrada: transcrição da reunião e listagem de arquivos do repositório. Saída: um RFC conciso (máx 4 páginas) com: metadados (autor, revisores), resumo executivo, proposta técnica, pelo menos 2 alternativas reais descartadas com trade-offs e 2 questões em aberto. Critérios obrigatórios: referencie ao menos 2 ADRs e cite caminhos do código quando usados como evidência.
```

- FDD / ADR prompt (decisão → implementação):

```
Você é um engenheiro escrevendo um FDD/ADR. Entrada: RFC + paths relevantes do código. Saída: FDD acionável com: fluxos (outbox insert, worker processing), contratos HTTP com exemplos (request/response e headers), matriz de erros com prefixo `WEBHOOK_*`, variáveis de ambiente, migrations necessárias e seção "Integração com o sistema existente" que nomeie ao menos 4 caminhos reais do repositório. Critérios obrigatórios: incluir exemplos concretos e pelo menos 1 teste de integração sugerido (arquivo alvo em `tests/`).
```

Usei versões iteradas desses prompts, exigindo sempre que arquivos citados existissem no repositório.

**5. Iterações e ajustes (principais correções)**

- Alucinação de paths: inicialmente a IA citou arquivos inexistentes. Ajuste: forneci listagem de paths reais e passei a validar automaticamente cada referência. Resultado: `docs/FDD.md` e `docs/TRACKER.md` agora citam caminhos reais como `src/modules/orders/order.service.ts`.

- Contratos vagos no FDD: os primeiros rascunhos não tinham exemplos de payload/headers. Ajuste: exigi exemplos de request/response e headers (`X-Event-Id`, `X-Signature`, `X-Webhook-Id`) no prompt. Resultado: `docs/FDD.md` contém contratos concretos e matriz de erros com `WEBHOOK_`.

- ADRs sem alternativas: alguns ADRs não listavam alternativas reais. Ajuste: passei a exigir pelo menos 1 alternativa prática por ADR e justificativa. Resultado: conjunto de ADRs cobre as decisões principais (outbox, retry/DLQ, HMAC, at-least-once, worker polling, reuso de padrões) com trade-offs.

Resumo de iterações: 3 ciclos principais de geração/revisão/refino para consolidar documentos consistentes e rastreáveis.

**6. Como navegar a entrega (ordem sugerida)**

- [docs/PRD.md](docs/PRD.md) — PRD: problema, público, objetivos e critérios de aceitação.
- [docs/RFC.md](docs/RFC.md) — RFC: proposta técnica, alternativas e questões em aberto.
- [docs/adrs/](docs/adrs/) — ADRs: decisões arquiteturais (ADR-001..ADR-006).
- [docs/FDD.md](docs/FDD.md) — FDD: especificação de implementação, fluxos, contratos e integração com o código.
- [docs/TRACKER.md](docs/TRACKER.md) — TRACKER: matriz de rastreabilidade ligando cada item à `TRANSCRICAO.md` ou a arquivos reais do código.

Ordem prática de leitura: RFC → ADRs → FDD → PRD → TRACKER.

Próximos passos que posso executar para ajudar:
- validar automaticamente que todos os caminhos referenciados em `docs/` existem;
- gerar um checklist de testes unitários e de integração com base no `docs/FDD.md`;
- scaffolder o módulo `src/modules/webhooks` e a migration Prisma sugerida pelo FDD.

Arquivo atualizado: [README.md](README.md)
