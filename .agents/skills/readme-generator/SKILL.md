---
name: readme-generator
description: Gera README detalhado sobre o processo de engenharia de prompt, investigacao do codigo-base e geracao da suite de design docs.
---

Voce e o Desenvolvedor responsavel por documentar o processo de engenharia de prompt, investigacao do codigo-base e geracao da suite de design docs. Gere `README.md` na raiz do projeto, descrevendo de forma clara e profissional como o desafio foi conduzido, quais ferramentas foram utilizadas e como a entrega foi organizada.

O README deve responder: "Como a transcricao virou um conjunto coerente de documentos de produto, arquitetura, implementacao e rastreabilidade?". O texto deve ser compreensivel para alguem que queira revisar, reutilizar ou evoluir o fluxo.

## Modo de execucao

1. Leia `TRANSCRICAO.md`.
2. Leia as skills/prompts usados para gerar PRD, RFC, ADRs, FDD e TRACKER, se estiverem disponiveis.
3. Leia os documentos gerados em `docs/`.
4. Inspecione o codigo-base apenas o suficiente para explicar como ele foi usado como evidencia.
5. Gere o README diretamente, sem entrevistar o usuario, exceto se nao houver documentos gerados nem informacao minima sobre o processo.
6. Se alguma ferramenta ou IA nao estiver comprovada no historico local, marque como `Nao evidenciado` ou descreva como hipotese.

## Regras obrigatorias

- Gere o arquivo `README.md` na raiz do projeto.
- O texto deve ser objetivo, mas suficientemente detalhado para demonstrar metodo e aprendizado.
- Evite linguagem excessivamente tecnica sem contexto.
- A secao de prompts deve apresentar exemplos praticos, nao apenas descricoes genericas.
- A secao de iteracoes deve mostrar maturacao do processo e evidenciar melhoria continua.

## Estrutura obrigatoria do README

### 1. Sobre o desafio

Escreva 1 a 2 paragrafos resumindo:

- Missao do desafio.
- Papel da transcricao.
- Papel do codigo-base.
- Resultado esperado da entrega.

### 2. Ferramentas de IA utilizadas

Liste as IAs ou assistentes usados e o papel de cada um. Exemplos:

- ChatGPT: estruturacao inicial, refinamento de PRD/FDD, revisao de consistencia.
- Claude: revisao critica de clareza ou deteccao de lacunas.
- Gemini: comparacao alternativa ou validacao textual.

Use apenas ferramentas realmente usadas quando houver evidencia. Se nao houver evidencia, escreva que a entrega foi preparada com o assistente disponivel no ambiente e evite inventar ferramentas.

### 3. Workflow adotado

Descreva a ordem cronologica da geracao dos documentos e a logica de organizacao do raciocinio:

1. Extracao da transcricao.
2. Leitura exploratoria do codigo-base.
3. PRD para alinhar produto e escopo.
4. RFC para consolidar proposta tecnica de alto nivel.
5. ADRs para registrar decisoes arquiteturais.
6. FDD para detalhar implementacao.
7. TRACKER para consolidar rastreabilidade.

Explique brevemente por que essa ordem reduz retrabalho.

### 4. Prompts customizados

Escolha 2 prompts principais que foram vitais para o sucesso e insira-os em blocos de codigo Markdown.

Os prompts devem ser concretos e reutilizaveis. Prefira:

- Um prompt de PRD ou RFC, focado em produto/proposta.
- Um prompt de FDD ou ADR, focado em decisao e implementacao.

Cada prompt pode ser resumido, desde que mantenha papel, entradas, saida esperada e criterios obrigatorios.

### 5. Iteracoes e ajustes

Relate pelo menos 2 momentos em que a IA cometeu ou poderia cometer erros, por exemplo:

- Alucinacao de arquivos inexistentes.
- Detalhamento tecnico dentro do PRD.
- Falta de metricas quantitativas.
- Contratos incompletos no FDD.
- ADRs sem alternativas reais.
- Tracker sem timestamps ou sem caminhos reais.

Para cada iteracao, descreva:

- Problema percebido.
- Ajuste feito no prompt ou no processo.
- Resultado apos o ajuste.

### 6. Como navegar a entrega

Forneca um guia com caminhos dos arquivos e ordem sugerida de leitura:

- `docs/PRD.md`
- `docs/RFC.md`
- `docs/adrs/`
- `docs/FDD.md`
- `docs/TRACKER.md`
- `README.md`

Explique em uma frase o papel de cada documento.

## Criterios de qualidade

- O README deve demonstrar processo, nao apenas listar arquivos.
- A narrativa deve ser honesta sobre limites, hipoteses e ajustes.
- A secao de ferramentas nao deve inventar IAs ou validacoes nao realizadas.
- Os prompts devem evidenciar como a qualidade final foi direcionada.
- A navegacao deve permitir que outra pessoa revise a entrega na ordem correta.
