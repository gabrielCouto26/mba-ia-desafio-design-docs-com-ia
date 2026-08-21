# PRD — Sistema de Webhooks de Notificação de Pedidos

Data: 2026-08-13
Autor: Product Manager Técnico (gerado automaticamente)

### Visão geral
Este documento descreve a proposta de produto para o sistema de Webhooks de Notificação de Pedidos: um mecanismo que entrega notificações outbound a clientes B2B quando um pedido muda de status, com garantias operacionais e administrativas claras. O objetivo é reduzir a dependência de polling por parte dos integradores, fornecer entrega assinada e auditável e dar ao time de operações ferramentas para reprocessamento e investigação.

## 1. Resumo e contexto da feature

Feature-alvo: entrega de notificações outbound (webhooks) para eventos de mudança de status de pedidos.

Resultado esperado: consumidores externos recebem notificações confiáveis e verificáveis de mudanças de status de pedidos sem necessidade de polling intenso; operadores dispõem de visibilidade e controle (histórico, DLQ, reprocessamento).

Encaixe no produto: integra-se ao módulo de Pedidos existente como um canal de notificação adicional — orientado a clientes B2B que precisam de baixa latência percebida para atualizações de estado.

Principais premissas: (i) a alteração de status do pedido continua sendo fonte única da verdade; (ii) operações críticas não devem depender de chamadas externas síncronas que possam introduzir falhas; (iii) detalhes operacionais (filas, polling, esquema) serão tratados como dependência técnica e operacional.

## 2. Problema e motivação

- Problema 1 — Integrações lentas e custosas: clientes B2B dependem de polling em `GET /orders`, gerando latência percebida, tráfego desnecessário e carga no sistema.
  - Afetados: integradores B2B (Atlas Comercial, MaxDistribuição, Nova Cargo — Hipótese: nomes citados na transcrição).
  - Impacto atual: atrasos na reação a mudanças críticas, maior custo de integração e frustração do cliente.
  - Por que resolver agora: requisito comercial prioritário e expectativa de entrega em curto prazo.

- Problema 2 — Falta de visibilidade operacional para falhas de entrega:
  - Afetados: time de Plataforma/Ops e equipes de suporte.
  - Impacto atual: tempo de investigação elevado, necessidade de replays manuais pouco suportados.
  - Por que resolver agora: reduzir MTTR e risco comercial de perda de eventos críticos.

## 3. Público-alvo e cenários de uso

- Público-alvo:
  - Integradores B2B que precisam de notificações de pedido quase em tempo real.
  - Equipe de Plataforma/Ops (monitoramento, reprocessamento, runbooks).
  - Product/CS (validação de integrações) e equipe de segurança.

- Cenários de uso chave:
  - Cenário A — Notificação imediata: quando o status de um pedido muda para `shipped`, o integrador recebe uma notificação assinada contendo `eventId` que permite reconciliation e deduplicação.
    - Gatilho: mudança de status.
    - Ação: sistema envia notificação outbound ao endpoint configurado.
    - Resultado esperado: integrador processa o evento e atualiza seu sistema.

  - Cenário B — Retry e DLQ: se o integrador estiver indisponível, o sistema executa retries; após esgotar tentativas, o evento aparece em DLQ para investigação e reprocessamento manual.
    - Gatilho: falhas sucessivas de entrega.
    - Resultado esperado: alerta/registro em DLQ e possibilidade de reprocessamento via API administrativa.

  - Cenário C — Reconfiguração e desativação de endpoint: time `ADMIN` desativa um endpoint problemático sem impactar a produção de eventos.
    - Gatilho: endpoint inválido ou secret comprometido.
    - Resultado esperado: novas entregas para esse endpoint deixam de ser geradas; eventos não são perdidos para outros endpoints.

## 4. Objetivos e métricas de sucesso

| Objetivo | Métrica | Meta | Fonte/hipótese |
|---|---:|---:|---|
| Entregar notificações com baixa latência percebida | % de primeiros envios entregues em <=10s | >= 90% dos eventos (Hipótese baseada em RFC/FDD) | RFC/FDD — SLA objetivo 10s |
| Confiabilidade de entrega operacional | % de eventos processados com sucesso ou movidos para DLQ após tentativas | >= 99% dos eventos são resolvidos (delivered ou DLQ) | Hipótese operacional |
| Reduzir carga por polling nos clientes | Redução no número médio de chamadas `GET /orders` por cliente integrado | -30% de polling em 30 dias com clientes piloto (Hipótese) | Expectativa de adoção pelos clientes |
| Reduzir tickets de suporte relacionados a sincronização de pedidos | Número de tickets mensais atribuíveis a atraso de eventos | -30% em 60 dias pós-rollout com pilotos (Hipótese) | Métrica de suporte |

## 5. Escopo

Incluso: (requisitos funcionais principais abaixo)

#### RF-001 CRUD de Endpoints de Webhook
- Descricao: permitir que usuários autenticados criem, listem, atualizem e desativem endpoints de webhook (nome, URL, secret gerada pela plataforma, filtros por status, active).
- Valor: integrações configuráveis por cliente e controle administrativo sobre canais de entrega.
- Fluxo principal: usuário autenticado envia criação; sistema valida URL, gera a secret e persiste a entidade.
- Fluxos alternativos e excecoes: validação de URL inválida; falha na geração/rotação da secret; desativação idempotente.
- Erros previstos: `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_ROTATION_FAILED`, `WEBHOOK_ENDPOINT_ALREADY_EXISTS`.
- Prioridade: alta

#### RF-002 Emissão de evento no fluxo de pedidos
- Descricao: ao mudar status de pedido, registrar um evento para entrega a endpoints interessados (somente quando aplicável).
- Valor: garante que mudanças de estado sejam capturadas como evento entregue ao integrador.
- Fluxo principal: mudança de status -> evento criado para endpoints com filtros compatíveis.
- Fluxos alternativos e excecoes: endpoint inativo => não criar evento; payload inválido => não criar evento e registrar erro.
- Erros previstos: falha na criação do evento, duplicidade.
- Prioridade: alta

#### RF-003 Entregas assinadas e cabeçalhos de rastreio
- Descricao: cada entrega inclui cabeçalhos que permitem verificação e rastreio (`X-Event-Id`, `X-Timestamp`, `X-Signature`, `X-Webhook-Id`).
- Valor: consumidores validam origem e evitaram fraudes/duplicidade.
- Fluxo principal: worker envia payload + headers; consumidor valida e responde 2xx.
- Fluxos alternativos e excecoes: consumidor rejeita por assinatura inválida -> contabilizar como failed.
- Erros previstos: `WEBHOOK_SIGNATURE_FAILED`, 4xx/5xx responses.
- Prioridade: alta

#### RF-004 Política de retries e DLQ
- Descricao: mecanismo de retries com backoff e, após máximo de tentativas, mover evento para DLQ com dados para reprocessamento manual.
- Valor: resiliencia frente a falhas temporarias do consumidor.
- Fluxo principal: tentativa -> falha -> agendamento de novo try -> se esgotado -> DLQ.
- Fluxos alternativos e excecoes: resposta 4xx não-recoverable -> mover para DLQ imediatamente (exceto 429).
- Erros previstos: acúmulo de DLQ sem reprocessamento.
- Prioridade: alta

#### RF-005 Histórico de entregas e consulta
- Descricao: expor `GET /webhooks/:id/deliveries` para consultar os últimos 100 registros por endpoint, incluindo tentativas, payload, response, erros e latência.
- Valor: permite investigação e auditoria por Ops e suporte.
- Fluxo principal: usuário autenticado consulta entregas -> recebe os últimos 100 registros com timestamps, status e detalhes operacionais.
- Fluxos alternativos e excecoes: ausência de registro -> 404.
- Erros previstos: inconsistência de registros por falhas de gravação.
- Prioridade: alta

#### RF-006 Reprocessamento manual de DLQ
- Descricao: API administrativa para reprocessar um evento em DLQ (opção para reusar `eventId` ou gerar novo), restrita a roles `ADMIN`.
- Valor: dá controle humano para recuperar eventos críticos.
- Fluxo principal: `ADMIN` requisita reprocessamento -> evento criado novamente na fila de entrega.
- Fluxos alternativos e excecoes: reprocessamento falha -> registrar `WEBHOOK_DLQ_REPROCESS_FAILED`.
- Erros previstos: permissões insuficientes.
- Prioridade: alta

#### RF-007 Timeouts e monitoração de entregas
- Descricao: expor métricas e logs estruturados por entrega (latency, statusCode, attempt, error) para integração com monitoramento e alertas.
- Valor: permite SRE acompanhar saúde do sistema e criar alertas (ex.: DLQ > 0).
- Fluxo principal: cada tentativa gera métricas; dashboards/alerts configuráveis por Ops.
- Fluxos alternativos e excecoes: métricas ausentes por falha de telemetria.
- Erros previstos: alertas falsos-positivos por flutuações curtas.
- Prioridade: alta

#### RF-008 Validação e política de ativação de endpoints
- Descricao: validar URL (ex: https), gerar secret na criação e permitir ativar/desativar endpoint sem perder eventos históricos.
- Valor: reduz riscos de vazamento e falhas por endpoints mal configurados.
- Fluxo principal: criação -> validação -> ativação.
- Fluxos alternativos e excecoes: falha na geração/rotação da secret; se URL inválida -> recusa.
- Erros previstos: `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_ROTATION_FAILED`.
- Prioridade: alta

#### RF-009 (Opcional) Filtros por tipo de evento e tenant
- Descricao: configurar filtros para enviar somente certos tipos de eventos (ex: status específicos) e suportar multi-tenant (Hipótese).
- Valor: reduz ruído para consumidores e permite personalização por cliente.
- Fluxo principal: criar endpoint com filtros -> somente eventos compatíveis geram entrega.
- Fluxos alternativos e excecoes: filtro mal definido -> nenhum evento enviado.
- Erros previstos: configuração incorreta causando perda de eventos.
- Prioridade: média

## 6. Fora de escopo

- Dashboard visual para clientes (monitor, métricas por cliente): adiado para fases posteriores por complexidade e necessidade de priorização; por enquanto exposição via métricas/APIs para Ops.
- Notificações automáticas por e-mail para falhas de webhook: fora do MVP — reprocessamento manual via API e alertas operacionais são suficientes inicialmente.
- Rate limiting por cliente e escala multi-worker com particionamento: adiado — MVP usa abordagem simples para validar hipóteses de carga.

## 7. Requisitos não funcionais

- Confiabilidade percebida: 99% dos eventos devem ser resolvidos (delivered ou DLQ) e a fuga para DLQ deve ser rastreável.
- Segurança e privacidade: entregas devem ser verificáveis por assinatura e transmitidas via HTTPS (Hipótese: clientes exigem TLS); secrets de endpoint devem ser gerenciados por Ops.
- Performance esperada: objetivo de primeira entrega observável <=10s para 90% dos eventos (ver métricas). Latência alvo de produção será validada com pilotos.
- Auditabilidade: todas as tentativas de entrega, erros e reprocessamentos devem ser auditáveis e consultáveis por `eventId` e `endpointId`.
- Usabilidade operacional: APIs administrativas para CRUD de endpoints, consulta de histórico e reprocessamento DLQ; logs estruturados para investigação.
- Compatibilidade: integrar com o `Order` producer existente e autenticação/roles do sistema; (Hipótese: uso de role `ADMIN` para operações sensíveis).

## 8. Decisões e trade-offs principais

- Decisão: adotar entrega at-least-once com `eventId` para deduplicação pelo consumidor.
  - Motivo: reduz complexidade operacional e evita acoplamento forte entre produção e entrega.
  - Alternativa descartada: tentar garantir exactly-once (muito complexo e arriscado nesta fase).
  - Impacto: consumidores precisam deduplicar; risco de entregas duplicadas mitigado por campos de rastreio.

- Decisão: MVP com mecanismo interno simples (polling/single-worker como hipótese operacional).
  - Motivo: velocidade de entrega e menor infra adicional para validar demanda.
  - Alternativa descartada: introduzir infra de mensageria externa imediatamente (ex.: Kafka/Redis Streams).
  - Impacto: limita escala inicial e requer plano de evolução para particionamento e ordenação.

## 9. Dependências

- Dependências internas:
  - `OrderService` / módulo de pedidos: fonte dos eventos (implementação existente em `src/modules/orders`) — obrigatório.
  - Time de Plataforma/Ops para rodar e monitorar worker e aplicar migrations.
  - Autenticação e roles existentes para restrição de APIs (Hipótese: `ADMIN` role existe).

- Dependências externas/decisões técnicas (Hipótese quando inferido):
  - Gestão de secrets para endpoints (rota de rotação/armazenamento seguro).
  - Migrations no banco de dados para armazenar histórico de entregas/DLQ.
  - Aprovação de segurança para sair de payloads de pedido para endpoints externos.

## 10. Riscos e mitigação

- Risco: Duplicidade de entregas gerando inconsistência nos integradores.
  - Probabilidade: alta
  - Impacto: médio
  - Estratégia de mitigação: exigir `X-Event-Id` e orientar consumidores a deduplicar; documentar contrato claramente.
  - Sinal de alerta: aumento de chamadas repetidas para o mesmo `eventId` nos logs.

- Risco: Crescimento indefinido de outbox/DLQ degradando performance do DB.
  - Probabilidade: média
  - Impacto: alto
  - Estratégia de mitigação: definir política de retenção/archiving e criar alertas em `webhook_dlq_size` e métricas de backlog.
  - Sinal de alerta: aumento contínuo do backlog pendente e crescimento de tabela beyond threshold.

- Risco: Endpoints externos mal configurados (invalid URL/secret rotation) causando falhas e ruído.
  - Probabilidade: média
  - Impacto: baixo/medio
  - Estratégia de mitigação: validação na criação, geração/rotação controlada da secret, permitir desativação rápida e exposição de erros específicos (`WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_ROTATION_FAILED`).
  - Sinal de alerta: pico de erros de validação e tentativas falhas imediatamente após criação.

## 11. Critérios de aceitação

- CA-001: CRUD de endpoints implementado e protegido por autenticação JWT. (verificável via API test)
- CA-002: Ao alterar status de pedido em cenário de integração, existe registro para entrega consultável por `eventId` (integração testada em ambiente de staging).
- CA-003: Sistema registra tentativas de entrega e expõe histórico consultável por `eventId` e `endpointId`.
- CA-004: Reprocessamento manual de DLQ cria nova tentativa e atualiza histórico (test automático/manual).
- CA-005: 90% dos primeiros envios registrados em teste de carga piloto ocorrem em <=10s (metrica observavel em staging).
- CA-006: Entregas incluem cabeçalhos de rastreio e assinatura; consumidor de teste valida assinatura com sucesso.

## 12. Estratégia de testes e validação

- Validação funcional:
  - Unit e integration tests cobrindo criação de endpoint, validação, criação de evento na mudança de status, métricas e histórico de entregas.

- Validação com usuários/times consumidores:
  - Pilot com 1–3 clientes B2B: verificar integração, medir latência e correções de contrato.

- Validação operacional:
  - Testes de falha do consumidor (simular 5xx/timeout) e verificar retries, backoff e movimento para DLQ.
  - Testes de carga limitados para validar crescimento do backlog e comportamento de retenção.

- Evidências esperadas para aceite:
  - Logs e métricas mostrando taxa de sucesso, latencias e tamanho de DLQ dentro das metas.
  - Demonstração de reprocessamento manual por `ADMIN` e consulta de histórico.

---

Notas e hipoteses: informações extraídas do RFC, FDD e ADRs fornecidos; onde aplicável marquei `Hipótese` (ex.: números de meta, uso de role `ADMIN`, retenção). Para dúvidas ou ajustes, recomenda-se validar metas e dependências com Times de Plataforma e Security antes do desenvolvimento.
# PRD — Product Requirements Document

<!-- documento a ser elaborado -->
