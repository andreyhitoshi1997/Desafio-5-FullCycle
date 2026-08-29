# FDD — Feature Design Document: Sistema de Webhooks de Notificação de Pedidos

> Documento de implementação. Para o racional de produto ver [PRD.md](PRD.md); para a proposta de arquitetura
> e alternativas descartadas ver [RFC.md](RFC.md); para cada decisão isolada com trade-offs ver `docs/adrs/`.

## 1. Contexto e Motivação Técnica

O OMS não possui hoje nenhum mecanismo de notificação externa, eventos ou filas — esse vácuo é proposital e é
exatamente o que esta feature preenche. O ciclo de vida do pedido é controlado por
`OrderService.changeStatus` (`src/modules/orders/order.service.ts`), que roda dentro de uma transação Prisma
que já atualiza `orders`, insere em `order_status_history` e ajusta `stock_quantity`. A feature precisa somar
a esse fluxo a emissão de um evento de notificação, sem comprometer a atomicidade nem a performance dessa
transação, e sem acoplar o ciclo de vida do pedido à disponibilidade de sistemas de terceiros
(`TRANSCRICAO.md` [09:03]–[09:04]).

## 2. Objetivos Técnicos

- Garantir que **toda** mudança de status elegível gere um evento de webhook, de forma atômica com a própria
  mudança (nunca status mudado sem evento, nunca evento sem status mudado de fato).
- Entregar o evento ao cliente em até 10 segundos no caminho feliz, respeitando o requisito de "tempo real"
  dos clientes B2B (`TRANSCRICAO.md` [09:02]).
- Garantir autenticidade e integridade do payload (HMAC-SHA256) e uma garantia de entrega at-least-once,
  deduplicável pelo cliente.
- Não introduzir infraestrutura nova: reaproveitar MySQL, Prisma, Express, Pino e as convenções de módulo,
  erro e autenticação já existentes (ver [ADR-006](adrs/ADR-006-reuso-de-padroes-existentes-do-projeto.md)).

## 3. Escopo e Exclusões

**Dentro do escopo:** CRUD de configuração de webhook, filtro de eventos por status, emissão atômica via
outbox, worker de entrega com retry/backoff/DLQ, autenticação HMAC-SHA256 com rotação de secret, histórico de
entregas, endpoint administrativo de replay de DLQ.

**Fora do escopo desta feature** (ver detalhamento de origem no [PRD](PRD.md#escopo)):
notificação por e-mail em caso de falha recorrente, rate limiting de envio, dashboard visual para o cliente,
webhooks inbound, arquivamento automático de eventos entregues, escalonamento para múltiplos workers.

## 4. Fluxos Detalhados

### 4.1 Criação do evento na outbox

1. Um caller autenticado chama `PATCH /api/v1/orders/:id/status` (endpoint já existente).
2. `OrderService.changeStatus` abre a transação Prisma já existente, valida a transição
   (`canTransition` em `src/modules/orders/order.status.ts`), debita/repõe estoque quando aplicável, atualiza
   `orders` e insere em `order_status_history` — tudo como hoje.
3. **Novo passo, dentro da mesma transação:** o service chama
   `publishWebhookEvent(tx, order, fromStatus, toStatus)` (nova função pura em
   `src/modules/webhooks/webhook.outbox.ts`, recebendo o `tx` da transação corrente — sem injeção de
   repository inteiro, conforme decidido em reunião: `TRANSCRICAO.md` [09:41], Diego e Bruno).
4. `publishWebhookEvent`:
   a. Busca (via `tx`) os `WebhookEndpoint` ativos do `customer_id` do pedido cujo array `events` contém o
      `toStatus`. Se nenhum endpoint corresponder, não insere nada (`TRANSCRICAO.md` [09:34], Bruno/Diego).
   b. Para cada endpoint correspondente, monta o payload (snapshot — ver
      [ADR-007](adrs/ADR-007-snapshot-do-payload-no-outbox.md)) e insere uma linha em `webhook_outbox` com
      `status = PENDING`.
   c. Se o payload serializado exceder 64KB, lança `WebhookPayloadTooLargeError` (`WEBHOOK_PAYLOAD_TOO_LARGE`)
      **dentro da transação** — isso reverte a mudança de status inteira, não só a emissão do evento
      (`TRANSCRICAO.md` [09:23]–[09:24], [09:40]–[09:41]).
5. A transação é commitada. Se qualquer passo acima falhar, tudo é revertido — não existe estado intermediário
   onde o status mudou mas o evento não foi registrado, ou vice-versa.

### 4.2 Processamento pelo worker

1. `src/worker.ts` inicializa sua própria instância de `PrismaClient` (mesmo `DATABASE_URL`,
   processo separado — [ADR-002](adrs/ADR-002-worker-processo-separado-polling.md)) e entra num loop de
   polling de 2 segundos.
2. A cada ciclo, busca um lote pequeno (ex.: 20) de linhas de `webhook_outbox` com `status = PENDING` e
   `next_attempt_at <= now()`, ordenadas por `created_at` (garante ordering por `order_id` em cenário
   single-worker — `TRANSCRICAO.md` [09:12]).
3. Para cada linha, marca `status = PROCESSING` (evita disputa se o worker escalar no futuro) e monta a
   requisição HTTP:
   - `POST` para a `url` do `WebhookEndpoint`.
   - Headers: `Content-Type: application/json`, `X-Event-Id` (o próprio id da linha do outbox, UUID),
     `X-Signature` (HMAC-SHA256 do corpo, usando a secret ativa do endpoint), `X-Timestamp` (ISO 8601, hora do
     envio) e `X-Webhook-Id` (id do `WebhookEndpoint`) — todos definidos em `TRANSCRICAO.md` [09:44].
   - Timeout de 10 segundos (`TRANSCRICAO.md` [09:42]).
4. Registra uma linha em `webhook_deliveries` para a tentativa (sucesso ou falha, status HTTP, corpo de
   resposta truncado, duração em ms).
5. Se a resposta for 2xx: marca a linha do outbox como `status = SENT`.
6. Se falhar (timeout, erro de rede, status != 2xx): segue para o fluxo de retry (4.3).

### 4.3 Retry com backoff exponencial

1. Ao falhar uma tentativa, o worker incrementa `attempts` na linha do outbox e calcula `next_attempt_at`
   segundo a progressão fixa
   **1 min → 5 min → 30 min → 2h → 12h** (tentativas 1 a 5 — [ADR-003](adrs/ADR-003-retry-backoff-dead-letter-queue.md)).
2. A linha volta para `status = PENDING` com o novo `next_attempt_at`; o worker só volta a pegá-la quando esse
   horário chegar.
3. Se a 5ª tentativa falhar, segue para o fluxo de DLQ (4.4) em vez de reagendar novamente.

### 4.4 Dead Letter Queue e replay manual

1. Esgotadas as 5 tentativas, o worker move o evento: insere uma linha em `webhook_dead_letter` (payload,
   motivo da última falha, timestamp) e remove/marca a linha original da `webhook_outbox` como finalizada
   (`TRANSCRICAO.md` [09:18]).
2. Um operador com role `ADMIN` chama `POST /api/v1/admin/webhooks/dead-letter/:id/replay`
   (`TRANSCRICAO.md` [09:18], [09:35]).
3. O endpoint recoloca o evento na `webhook_outbox` como `status = PENDING`, `attempts = 0`,
   `next_attempt_at = now()`, e **loga o `userId` de quem executou o replay**, para auditoria
   (`TRANSCRICAO.md` [09:36], Sofia).
4. O worker pega o evento no próximo ciclo de polling, como qualquer evento pendente novo.

## 5. Contratos Públicos

Prefixo comum: `/api/v1` (`src/app.ts`). Todas as rotas exigem `Authorization: Bearer <jwt>` via
`authenticate` (`src/middlewares/auth.middleware.ts`), exceto onde indicado. Erros seguem o formato padrão do
projeto: `{ "error": { "code": "...", "message": "...", "details"?: ... } }` (`error.middleware.ts`).

### 5.1 `POST /api/v1/webhooks` — cadastrar webhook

Requisitado por qualquer usuário autenticado (`TRANSCRICAO.md` [09:36]: CRUD de configuração não exige
`ADMIN`, só o replay de DLQ exige).

Request:
```json
{
  "customerId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "url": "https://api.atlas-comercial.com/webhooks/oms",
  "events": ["SHIPPED", "DELIVERED"]
}
```

Response `201 Created`:
```json
{
  "id": "b6e1a2c4-1234-4a11-9c2d-0e5f6a7b8c9d",
  "customerId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "url": "https://api.atlas-comercial.com/webhooks/oms",
  "events": ["SHIPPED", "DELIVERED"],
  "secret": "whsec_9f1c2a7e4b3d4c5e8a9f0b1c2d3e4f5a",
  "active": true,
  "createdAt": "2026-08-29T12:00:00.000Z",
  "updatedAt": "2026-08-29T12:00:00.000Z"
}
```
A `secret` só é retornada neste momento e no de rotação (5.6) — nunca em `GET`/`PATCH`. `secret` é gerada pela
plataforma, não enviada pelo cliente (`TRANSCRICAO.md` [09:31]).

### 5.2 `GET /api/v1/webhooks?customerId=...` — listar webhooks de um customer

Response `200 OK` (segue o padrão `paginated()` de `src/shared/http/response.ts`, já usado por
`GET /orders`):
```json
{
  "data": [
    {
      "id": "b6e1a2c4-1234-4a11-9c2d-0e5f6a7b8c9d",
      "customerId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "url": "https://api.atlas-comercial.com/webhooks/oms",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-08-29T12:00:00.000Z",
      "updatedAt": "2026-08-29T12:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

### 5.3 `PATCH /api/v1/webhooks/:id` — editar webhook

Request:
```json
{ "events": ["PAID", "SHIPPED", "DELIVERED"], "active": true }
```

Response `200 OK`: mesmo formato do item de 5.2 (sem `secret`).

### 5.4 `DELETE /api/v1/webhooks/:id` — remover webhook

Response `204 No Content`, sem corpo — mesmo padrão de `DELETE /orders/:id`.

### 5.5 `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas

Últimas 100 entregas do endpoint, sucesso ou falha (`TRANSCRICAO.md` [09:34]).

Response `200 OK`:
```json
{
  "data": [
    {
      "id": "d4e5f6a7-...",
      "outboxEventId": "b6e1a2c4-...",
      "attemptNumber": 1,
      "success": true,
      "httpStatus": 200,
      "durationMs": 184,
      "createdAt": "2026-08-29T12:00:02.100Z"
    },
    {
      "id": "d4e5f6a8-...",
      "outboxEventId": "c7f2b3d5-...",
      "attemptNumber": 3,
      "success": false,
      "httpStatus": null,
      "responseExcerpt": "ETIMEDOUT",
      "durationMs": 10000,
      "createdAt": "2026-08-29T11:35:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 2, "totalPages": 1 }
}
```

### 5.6 `POST /api/v1/webhooks/:id/secret/rotate` — rotacionar secret

Response `200 OK`:
```json
{
  "secret": "whsec_5c8e7f2a1b3d9c4e6a8f0b2c4d6e8f0a",
  "previousSecretValidUntil": "2026-08-30T12:00:00.000Z"
}
```
A secret anterior permanece válida por 24h (`TRANSCRICAO.md` [09:21]).

### 5.7 `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — reprocessar item da DLQ

Exige `requireRole('ADMIN')` (`TRANSCRICAO.md` [09:35]–[09:36], reaproveitando
`src/middlewares/auth.middleware.ts`).

Response `202 Accepted`:
```json
{
  "id": "e8f9a0b1-...",
  "status": "PENDING",
  "requeuedAt": "2026-08-29T13:10:00.000Z",
  "requeuedBy": "5a1b2c3d-userid-do-admin"
}
```

## 6. Matriz de Erros (`WEBHOOK_*`)

Todos estendem a hierarquia existente em `src/shared/errors/` (ver seção 9) e fluem pelo
`error.middleware.ts` sem tratamento especial.

| Código | HTTP | Cenário | Onde ocorre | Fonte |
|---|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | Webhook endpoint não encontrado para o `id` informado | GET/PATCH/DELETE/deliveries/rotate | `TRANSCRICAO.md` [09:28] |
| `WEBHOOK_INVALID_URL` | 400 | URL cadastrada não é `https://` ou é malformada (validação Zod) | POST/PATCH | `TRANSCRICAO.md` [09:23], [09:28] |
| `WEBHOOK_SECRET_REQUIRED` | 400 | Operação depende de secret ativa e nenhuma foi encontrada (estado inconsistente) | rotate | `TRANSCRICAO.md` [09:28] |
| `WEBHOOK_INVALID_EVENT_FILTER` | 400 | Lista `events` contém valor fora do enum `OrderStatus` | POST/PATCH | `TRANSCRICAO.md` [09:33], [09:28] (extensão do padrão) |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Payload do evento excede 64KB no momento da inserção no outbox | `changeStatus` (interno) | `TRANSCRICAO.md` [09:23]–[09:24] |
| `WEBHOOK_DELIVERY_TIMEOUT` | — (interno, registrado em `webhook_deliveries`) | Chamada HTTP ao cliente não respondeu em 10s | worker | `TRANSCRICAO.md` [09:42] |
| `WEBHOOK_DELIVERY_FAILED` | — (interno) | Endpoint do cliente respondeu com status fora de 2xx | worker | `TRANSCRICAO.md` [09:14]–[09:15] |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `id` informado no replay não existe em `webhook_dead_letter` | replay | `TRANSCRICAO.md` [09:18], [09:35] (extensão do padrão) |
| `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | Replay solicitado para um item já reenfileirado | replay | `TRANSCRICAO.md` [09:18] (extensão do padrão) |

## 7. Estratégias de Resiliência

- **Timeout:** 10 segundos por chamada HTTP de entrega (`TRANSCRICAO.md` [09:42]).
- **Retry:** backoff exponencial fixo, 5 tentativas, progressão 1m/5m/30m/2h/12h
  ([ADR-003](adrs/ADR-003-retry-backoff-dead-letter-queue.md)).
- **Fallback:** esgotadas as tentativas, o evento vai para `webhook_dead_letter`; recuperação é manual via
  endpoint administrativo — não há retry automático indefinido (decisão explícita contra essa alternativa,
  `TRANSCRICAO.md` [09:15]).
- **Isolamento de falha:** o worker roda em processo separado da API — uma falha ou lentidão no processamento
  de webhooks nunca impacta a disponibilidade da API de pedidos
  ([ADR-002](adrs/ADR-002-worker-processo-separado-polling.md)).
- **Idempotência do lado do cliente:** `X-Event-Id` único por evento permite dedup no cliente, dado que a
  garantia de entrega é at-least-once, não exactly-once
  ([ADR-005](adrs/ADR-005-at-least-once-x-event-id.md)).
- **Limite de payload:** 64KB; eventos maiores falham a inserção no outbox (erram, não truncam —
  `TRANSCRICAO.md` [09:23]).

## 8. Observabilidade

- **Logs:** reaproveita o logger Pino já configurado em `src/shared/logger/index.ts` (mesmo `redact` de
  campos sensíveis — a `secret` do webhook deve ser adicionada à lista de `redactPaths`, análoga a
  `*.password`/`*.token`). Eventos estruturados sugeridos: `webhook_outbox_event_created`,
  `webhook_delivery_attempt`, `webhook_delivery_succeeded`, `webhook_delivery_failed`,
  `webhook_dead_letter_created`, `webhook_dead_letter_replayed` (este último inclui o `userId` do admin, por
  exigência de auditoria — `TRANSCRICAO.md` [09:36]).
- **Métricas (propostas, a validar com o time de plataforma na implementação):**
  `webhook_outbox_pending_count` (gauge, por idade do evento mais antigo pendente — sinaliza atraso do
  worker), `webhook_delivery_attempts_total{result="success|failure"}` (contador), `webhook_delivery_latency_seconds`
  (histograma, tempo entre criação do evento e entrega bem-sucedida — mede o cumprimento da meta de 10s do
  PRD), `webhook_dead_letter_total` (contador, entradas movidas para DLQ).
- **Tracing/correlação:** o `X-Event-Id` funciona como identificador de correlação ponta a ponta (outbox →
  tentativas em `webhook_deliveries` → eventual DLQ), no mesmo espírito do `requestId` já usado em
  `error.middleware.ts` para correlacionar erros HTTP a um request.

## 9. Integração com o Sistema Existente

- **`src/modules/orders/order.service.ts`:** o método `changeStatus` é estendido para, dentro da mesma
  transação Prisma (`this.prisma.$transaction(async (tx) => { ... })`) que já atualiza `orders` e insere em
  `order_status_history`, chamar `publishWebhookEvent(tx, order, from, to)`. Nenhum outro método do serviço é
  alterado; `create` e `delete` não emitem eventos (fora do escopo — a reunião só tratou mudança de status).

- **`src/modules/orders/order.status.ts`:** não é alterado, mas é a fonte de verdade dos valores possíveis de
  `fromStatus`/`toStatus` (`canTransition`, enum `OrderStatus` do Prisma) usados tanto para popular o payload
  do evento quanto para validar o array `events` de um `WebhookEndpoint` (`WEBHOOK_INVALID_EVENT_FILTER`
  reaproveita o mesmo enum `OrderStatus`, não um enum próprio do módulo de webhooks).

- **`src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts`:** as novas classes de erro do
  módulo (`WebhookNotFoundError`, `WebhookInvalidUrlError`, `WebhookPayloadTooLargeError`, etc., em
  `src/modules/webhooks/webhook.errors.ts`) estendem as classes HTTP genéricas já existentes — por exemplo
  `class WebhookNotFoundError extends NotFoundError` e
  `class WebhookPayloadTooLargeError extends UnprocessableEntityError`, seguindo exatamente o padrão de
  `InvalidStatusTransitionError extends ConflictError` já usado no módulo de pedidos. Nenhuma classe nova é
  adicionada a `app-error.ts`/`http-errors.ts`; eles permanecem genéricos.

- **`src/middlewares/error.middleware.ts`:** não recebe nenhuma alteração. Por já despachar qualquer instância
  de `AppError` para a resposta HTTP correta usando `err.statusCode`/`err.errorCode`/`err.details`, os novos
  erros `WEBHOOK_*` são tratados automaticamente.

- **`src/middlewares/auth.middleware.ts`:** `authenticate` protege todas as rotas do módulo; `requireRole`
  é reaproveitado sem modificação para restringir `POST /api/v1/admin/webhooks/dead-letter/:id/replay` a
  usuários com `role: 'ADMIN'` (`TRANSCRICAO.md` [09:36]).

- **`src/server.ts`:** serve de modelo estrutural para o novo `src/worker.ts` — mesmo padrão de bootstrap
  assíncrono, log de início (`logger.info`) e *graceful shutdown* em `SIGINT`/`SIGTERM` (`prisma.$disconnect()`
  antes de sair), mas iniciando um loop de polling em vez de um `app.listen`.

- **`src/routes/index.ts`:** ganha um novo `router.use('/webhooks', buildWebhookRouter(controllers.webhooks))`
  e um `router.use('/admin/webhooks', buildAdminWebhookRouter(controllers.webhooks))`, seguindo o mesmo padrão
  de composição usado para `orders`, `customers`, `products` etc. `Controllers` (tipo em
  `src/routes/index.ts`) ganha a chave `webhooks`.

- **`prisma/schema.prisma`:** ganham 3 novos models — `WebhookEndpoint`, `WebhookOutboxEvent` e
  `WebhookDelivery`/`WebhookDeadLetter` — seguindo a convenção já usada em todo o schema: `id String @id
  @default(uuid()) @db.Char(36)` ([ADR-007](adrs/ADR-007-snapshot-do-payload-no-outbox.md)), `@map` para nome
  de tabela em `snake_case` (ex.: `@@map("webhook_outbox")`), e índices explícitos nos campos usados para
  filtro do worker (`status`, `created_at`, `next_attempt_at`), no mesmo espírito de `@@index([status])` em
  `Order`.

- **`src/shared/http/response.ts`:** `GET /webhooks` e `GET /webhooks/:id/deliveries` reaproveitam
  `paginated()`, o mesmo helper usado por `OrderService.list`, mantendo o formato `{ data, pagination }`
  consistente em toda a API.

## 10. Dependências e Compatibilidade

- Nenhuma dependência nova de runtime é necessária: HMAC-SHA256 é coberto pelo módulo nativo `crypto` do
  Node.js; UUID já é gerado com o pacote `uuid` já presente no `package.json`.
- Requer uma migration Prisma nova (`prisma/migrations/`) para os 3 models listados na seção 9 — não altera
  nenhuma tabela existente.
- `src/worker.ts` precisa de um novo script no `package.json` (`"worker": "tsx --env-file=.env
  src/worker.ts"`, espelhando o script `dev` já existente para `server.ts`) e de uma entrada equivalente para
  produção (build via `tsc -p tsconfig.build.json`, já existente).

## 11. Critérios de Aceite Técnicos

- Uma mudança de status que insira com sucesso em `webhook_outbox` só é commitada se a transação inteira
  (order + history + estoque + outbox) for bem-sucedida; um teste de integração deve provar que uma falha
  forçada na inserção do outbox reverte a mudança de status (não deixa o pedido em estado "mudou mas não
  notificou").
- Um teste de integração deve provar que um evento sem nenhum `WebhookEndpoint` correspondente (status fora do
  filtro, ou customer sem webhook cadastrado) não gera linha nenhuma em `webhook_outbox`.
- Testes do worker devem cobrir: sucesso em 1ª tentativa; falha com reagendamento correto de
  `next_attempt_at` na progressão 1m/5m/30m/2h/12h; movimentação para `webhook_dead_letter` após a 5ª falha;
  replay via endpoint administrativo recolocando o evento como `PENDING`.
- Um teste deve validar que `X-Signature` é uma HMAC-SHA256 válida do corpo exato enviado, calculável pelo
  cliente com a `secret` retornada na criação.
- Um teste deve validar que, durante o grace period de 24h após rotação, tanto a secret antiga quanto a nova
  validam corretamente uma assinatura.
- Um teste deve validar que `POST /api/v1/admin/webhooks/dead-letter/:id/replay` retorna `403` (via
  `ForbiddenError` existente) para um usuário `OPERATOR`.

## 12. Riscos Técnicos e Mitigação

| Risco | Mitigação |
|---|---|
| Acúmulo de linhas em `webhook_outbox`/`webhook_deliveries` sob alto volume, degradando o polling | Índices em `status`/`next_attempt_at`/`created_at`; leitura em lotes pequenos; arquivamento após 30 dias fica como trabalho futuro já identificado (`TRANSCRICAO.md` [09:08]) |
| Vazamento de `secret` em log de aplicação (já ocorreu com um cliente, `TRANSCRICAO.md` [09:22]) | `secret` nunca retornada fora da criação/rotação; adicionar `*.secret` ao `redactPaths` do logger (`src/shared/logger/index.ts`) |
| Perda de ordenação ao escalar para múltiplos workers no futuro | Documentado como limitação conhecida; exigir particionamento por `order_id` ou lock pessimista antes de qualquer escala horizontal (questão em aberto no [RFC](RFC.md#questões-em-aberto)) |
| Transação de `changeStatus` mais lenta pela escrita adicional no outbox | Escrita é um único `insert` por endpoint correspondente (geralmente 0 ou 1), sem chamada de rede dentro da transação — impacto de latência esperado é mínimo |
