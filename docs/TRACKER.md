# Tracker de Rastreabilidade

Mapeia cada item registrado em `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` e `docs/adrs/*.md` à sua origem —
um trecho com timestamp de `TRANSCRICAO.md`, ou um caminho real de arquivo em `CODIGO`. Itens sem origem
identificável foram removidos ou ajustados durante a produção dos documentos (ver `README.md`, seção
"Iterações e ajustes").

## PRD.md

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro de webhook (url, customerId, events, secret gerada pela plataforma) | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Edição de webhook (PATCH) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Remoção de webhook (DELETE) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Listagem de webhooks de um customer (GET) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro de eventos por status, aplicado na inserção no outbox | TRANSCRICAO | [09:34] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas por webhook | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Endpoint admin de replay manual de DLQ, role ADMIN | TRANSCRICAO | [09:35] Larissa |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Rotação de secret via API com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Emissão do evento dentro da mesma transação de changeStatus | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Assinatura HMAC-SHA256 do payload, header X-Signature | TRANSCRICAO | [09:20] Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Identificador único por evento (X-Event-Id) para dedup | TRANSCRICAO | [09:25] Diego |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Retry automático com backoff exponencial (5 tentativas) | TRANSCRICAO | [09:15] Diego |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência de entrega ≤ 10s da mudança de status | TRANSCRICAO | [09:10] Larissa |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | URL de webhook deve ser https, senão rejeita | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Limite de payload de 64KB, erro se ultrapassar | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s por chamada HTTP do worker | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Garantia mínima at-least-once (não exactly-once) | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Secret por endpoint, nunca secret global | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Replay de DLQ deve ser auditável (logar quem executou) | TRANSCRICAO | [09:36] Sofia |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Reuso da stack e padrões existentes, sem infra nova | TRANSCRICAO | [09:30] Larissa |
| PRD-OBJ-01 | docs/PRD.md | Objetivo/Métrica | Latência ≤10s como meta quantitativa de sucesso | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo/Métrica | Redução de polling client-side em GET /orders | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-03 | docs/PRD.md | Objetivo/Métrica | Retenção da Atlas via entrega dentro do prazo (fim de novembro) | TRANSCRICAO | [09:45] Marcos |
| PRD-SCOPE-OUT-01 | docs/PRD.md | Restrição (fora de escopo) | Notificação por e-mail em falha recorrente adiada para fase futura | TRANSCRICAO | [09:37] Larissa |
| PRD-SCOPE-OUT-02 | docs/PRD.md | Restrição (fora de escopo) | Dashboard visual para cliente, fora de escopo | TRANSCRICAO | [09:39] Larissa |
| PRD-SCOPE-OUT-03 | docs/PRD.md | Restrição (fora de escopo) | Webhooks inbound descartados, só outbound | TRANSCRICAO | [09:02] Marcos |
| PRD-SCOPE-OUT-04 | docs/PRD.md | Restrição (fora de escopo) | Rate limiting de envio adiado, "observar e decidir depois" | TRANSCRICAO | [09:38] Diego |
| PRD-SCOPE-OUT-05 | docs/PRD.md | Restrição (fora de escopo) | Arquivamento automático de eventos entregues, fora de escopo | TRANSCRICAO | [09:08] Diego |
| PRD-SCOPE-OUT-06 | docs/PRD.md | Restrição (fora de escopo) | Escalonamento para múltiplos workers, "problema do futuro" | TRANSCRICAO | [09:13] Diego |
| PRD-RISK-01 | docs/PRD.md | Risco | Cliente indisponível por período prolongado esgota retries | TRANSCRICAO | [09:16] Diego |
| PRD-RISK-02 | docs/PRD.md | Risco | Vazamento de secret (incidente real já ocorrido) | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-03 | docs/PRD.md | Risco | Acúmulo de eventos na tabela de outbox sob alto volume | TRANSCRICAO | [09:07] Bruno |
| PRD-RISK-04 | docs/PRD.md | Risco | Perda de ordenação ao escalar para múltiplos workers | TRANSCRICAO | [09:12] Diego |
| PRD-RISK-05 | docs/PRD.md | Risco | Sobrecarga do endpoint do cliente por rajada de eventos | TRANSCRICAO | [09:38] Diego |
| PRD-DEC-01 | docs/PRD.md | Trade-off | At-least-once + dedup no cliente em vez de exactly-once | TRANSCRICAO | [09:25] Diego |
| PRD-DEP-01 | docs/PRD.md | Dependência | Reuso de AppError/hierarquia de erros existente | CODIGO | src/shared/errors/app-error.ts |
| PRD-DEP-02 | docs/PRD.md | Dependência | Banco MySQL já provisionado via Prisma | CODIGO | prisma/schema.prisma |
| PRD-DEP-03 | docs/PRD.md | Dependência | Disponibilidade da Sofia para revisão de segurança (2 dias úteis) | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-04 | docs/PRD.md | Dependência | Confirmação de prazo com a Atlas pelo Marcos | TRANSCRICAO | [09:45] Marcos |
| PRD-AC-01 | docs/PRD.md | Critério de Aceitação | Aprovação de segurança da Sofia antes do deploy | TRANSCRICAO | [09:46] Sofia |

## RFC.md

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| RFC-META-01 | docs/RFC.md | Decisão | Larissa se compromete a redigir o doc de design da feature | TRANSCRICAO | [09:50] Larissa |
| RFC-ALT-01 | docs/RFC.md | Trade-off | Disparo síncrono descartado (trava transação de outros pedidos) | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Redis Streams descartado (overengineering p/ time pequeno) | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Trade-off | Trigger de banco (LISTEN/NOTIFY) descartado, MySQL não suporta | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Trade-off | Exactly-once descartado por complexidade de coordenação | TRANSCRICAO | [09:25] Diego |
| RFC-OPEN-01 | docs/RFC.md | Questão em Aberto | Rate limiting de envio por cliente | TRANSCRICAO | [09:39] Diego |
| RFC-OPEN-02 | docs/RFC.md | Questão em Aberto | Ordenação global ao escalar para múltiplos workers | TRANSCRICAO | [09:13] Diego |
| RFC-OPEN-03 | docs/RFC.md | Questão em Aberto | Endurecimento futuro da autorização do CRUD de configuração | TRANSCRICAO | [09:37] Sofia |
| RFC-IMPACT-01 | docs/RFC.md | Risco | Impacto da escrita adicional na transação de changeStatus | TRANSCRICAO | [09:41] Diego |
| RFC-IMPACT-02 | docs/RFC.md | Risco | Acúmulo de dados sem rotina de arquivamento | TRANSCRICAO | [09:08] Diego |
| RFC-IMPACT-03 | docs/RFC.md | Risco | Exposição de dados a terceiros, mitigada por revisão de segurança dedicada | TRANSCRICAO | [09:46] Sofia |
| RFC-IMPACT-04 | docs/RFC.md | Estimativa | ~3 sprints incluindo revisão de segurança | TRANSCRICAO | [09:46] Larissa |

## FDD.md

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| FDD-FLOW-01 | docs/FDD.md | Fluxo | Criação do evento na outbox dentro da transação | TRANSCRICAO | [09:06] Diego |
| FDD-FLOW-02 | docs/FDD.md | Fluxo | Processamento pelo worker e headers de entrega | TRANSCRICAO | [09:44] Diego |
| FDD-FLOW-03 | docs/FDD.md | Fluxo | Retry com progressão de backoff | TRANSCRICAO | [09:17] Diego |
| FDD-FLOW-04 | docs/FDD.md | Fluxo | Movimentação para DLQ e replay manual | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato Público | POST /api/v1/webhooks (criação, secret retornada) | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato Público | GET /api/v1/webhooks (listagem) | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato Público | PATCH /api/v1/webhooks/:id (edição) | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato Público | DELETE /api/v1/webhooks/:id (remoção) | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato Público | GET /api/v1/webhooks/:id/deliveries (histórico) | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato Público | POST /api/v1/webhooks/:id/secret/rotate | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato Público | POST /api/v1/admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:35] Diego |
| FDD-ERR-01 | docs/FDD.md | Restrição (erro) | WEBHOOK_NOT_FOUND | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Restrição (erro) | WEBHOOK_INVALID_URL | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-03 | docs/FDD.md | Restrição (erro) | WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-04 | docs/FDD.md | Restrição (erro) | WEBHOOK_INVALID_EVENT_FILTER (extensão do padrão WEBHOOK_*) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-05 | docs/FDD.md | Restrição (erro) | WEBHOOK_PAYLOAD_TOO_LARGE | TRANSCRICAO | [09:23] Sofia |
| FDD-ERR-06 | docs/FDD.md | Restrição (erro) | WEBHOOK_DELIVERY_TIMEOUT | TRANSCRICAO | [09:42] Diego |
| FDD-ERR-07 | docs/FDD.md | Restrição (erro) | WEBHOOK_DELIVERY_FAILED | TRANSCRICAO | [09:15] Diego |
| FDD-ERR-08 | docs/FDD.md | Restrição (erro) | WEBHOOK_DEAD_LETTER_NOT_FOUND (extensão do padrão WEBHOOK_*) | TRANSCRICAO | [09:18] Diego |
| FDD-ERR-09 | docs/FDD.md | Restrição (erro) | WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED (extensão do padrão WEBHOOK_*) | TRANSCRICAO | [09:18] Diego |
| FDD-RESIL-01 | docs/FDD.md | Decisão | Timeout de 10s por chamada de entrega | TRANSCRICAO | [09:42] Diego |
| FDD-RESIL-02 | docs/FDD.md | Decisão | Retry com backoff exponencial fixo | TRANSCRICAO | [09:17] Diego |
| FDD-RESIL-03 | docs/FDD.md | Decisão | Fallback para DLQ, sem retry indefinido | TRANSCRICAO | [09:15] Diego |
| FDD-RESIL-04 | docs/FDD.md | Decisão | Isolamento de falha via processo separado | TRANSCRICAO | [09:11] Diego |
| FDD-RESIL-05 | docs/FDD.md | Decisão | Idempotência via X-Event-Id no cliente | TRANSCRICAO | [09:25] Diego |
| FDD-RESIL-06 | docs/FDD.md | Restrição | Limite de payload de 64KB | TRANSCRICAO | [09:24] Diego |
| FDD-OBS-01 | docs/FDD.md | Decisão | Reuso do logger Pino existente para eventos estruturados | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-02 | docs/FDD.md | Decisão | Métricas propostas para medir a meta de latência de 10s | TRANSCRICAO | [09:02] Marcos |
| FDD-OBS-03 | docs/FDD.md | Decisão | X-Event-Id como correlação, no espírito do requestId existente | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INTEG-01 | docs/FDD.md | Restrição (integração) | Extensão de OrderService.changeStatus para publicar evento na transação | CODIGO | src/modules/orders/order.service.ts |
| FDD-INTEG-02 | docs/FDD.md | Restrição (integração) | Reuso do enum/máquina de estados de order.status.ts | CODIGO | src/modules/orders/order.status.ts |
| FDD-INTEG-03 | docs/FDD.md | Restrição (integração) | Novas classes de erro estendem AppError/http-errors existentes | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INTEG-04 | docs/FDD.md | Restrição (integração) | error.middleware.ts não é alterado, trata WEBHOOK_* automaticamente | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INTEG-05 | docs/FDD.md | Restrição (integração) | Reuso de requireRole('ADMIN') no endpoint de replay | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INTEG-06 | docs/FDD.md | Restrição (integração) | src/worker.ts espelha o padrão de bootstrap de server.ts | CODIGO | src/server.ts |
| FDD-INTEG-07 | docs/FDD.md | Restrição (integração) | Novas rotas registradas em routes/index.ts | CODIGO | src/routes/index.ts |
| FDD-INTEG-08 | docs/FDD.md | Restrição (integração) | Novos models seguem convenção uuid/@map do schema Prisma | CODIGO | prisma/schema.prisma |
| FDD-INTEG-09 | docs/FDD.md | Restrição (integração) | Listagens reaproveitam o helper paginated() | CODIGO | src/shared/http/response.ts |
| FDD-DEP-01 | docs/FDD.md | Dependência | HMAC via módulo nativo crypto, UUID via pacote já existente | CODIGO | package.json |
| FDD-DEP-02 | docs/FDD.md | Dependência | Novo script npm run worker, mesmo padrão do script dev | CODIGO | package.json |
| FDD-AC-01 | docs/FDD.md | Critério de Aceitação | Teste de atomicidade da transação changeStatus + outbox | CODIGO | src/modules/orders/order.service.ts |
| FDD-AC-02 | docs/FDD.md | Critério de Aceitação | Testes do worker cobrindo sucesso, retry e DLQ | TRANSCRICAO | [09:15] Diego |
| FDD-AC-03 | docs/FDD.md | Critério de Aceitação | Teste de assinatura HMAC-SHA256 válida | TRANSCRICAO | [09:20] Sofia |
| FDD-AC-04 | docs/FDD.md | Critério de Aceitação | Teste de grace period de 24h na rotação de secret | TRANSCRICAO | [09:21] Sofia |
| FDD-RISK-01 | docs/FDD.md | Risco | Acúmulo de linhas degradando o polling do worker | TRANSCRICAO | [09:08] Diego |
| FDD-RISK-02 | docs/FDD.md | Risco | Vazamento de secret em log de aplicação | TRANSCRICAO | [09:22] Diego |
| FDD-RISK-03 | docs/FDD.md | Risco | Perda de ordenação ao escalar workers | TRANSCRICAO | [09:13] Diego |
| FDD-RISK-04 | docs/FDD.md | Risco | Latência adicional na transação de changeStatus | TRANSCRICAO | [09:04] Bruno |

## docs/adrs/*.md

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| ADR-001 | docs/adrs/ADR-001-outbox-pattern-mysql.md | Decisão | Padrão Outbox no MySQL para emissão de eventos | TRANSCRICAO | [09:06] Diego |
| ADR-002 | docs/adrs/ADR-002-worker-processo-separado-polling.md | Decisão | Worker em processo separado, polling de 2s | TRANSCRICAO | [09:09] Diego |
| ADR-003 | docs/adrs/ADR-003-retry-backoff-dead-letter-queue.md | Decisão | Retry com backoff (5 tentativas) e DLQ em tabela separada | TRANSCRICAO | [09:15] Diego |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256 com secret por endpoint e rotação | TRANSCRICAO | [09:20] Sofia |
| ADR-005 | docs/adrs/ADR-005-at-least-once-x-event-id.md | Decisão | At-least-once com dedup via X-Event-Id | TRANSCRICAO | [09:24] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-de-padroes-existentes-do-projeto.md | Decisão | Reuso máximo dos padrões existentes do projeto | TRANSCRICAO | [09:30] Larissa |
| ADR-007 | docs/adrs/ADR-007-snapshot-do-payload-no-outbox.md | Decisão | Snapshot do payload na inserção, id UUID | TRANSCRICAO | [09:52] Larissa |

## Cobertura

- **Total de linhas:** 107 (40 em PRD.md, 12 em RFC.md, 48 em FDD.md, 7 em ADRs).
- **Fonte TRANSCRICAO:** 91 linhas (~85%), todas com timestamp no formato `[hh:mm] Nome` — acima do mínimo de 70% exigido.
- **Fonte CODIGO:** 16 linhas (~15%), todas com caminho de arquivo real do repositório — acima do mínimo de 5 exigido.
- Itens de granularidade muito fina dentro de um mesmo endpoint/erro/ADR (ex.: cada campo de um payload de
  exemplo) não ganham linha própria — são cobertos pela linha do contrato/erro/decisão a que pertencem, para
  manter o tracker legível sem duplicar informação já rastreável pelo próprio corpo do documento.
