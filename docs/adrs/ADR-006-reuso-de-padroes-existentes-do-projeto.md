# ADR-006: Reuso máximo dos padrões existentes do projeto para o módulo de webhooks

- **Status:** Aceito
- **Data:** 2026-08-29
- **Decisores:** Larissa (Tech Lead), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma)
- **Fonte:** `TRANSCRICAO.md` [09:27]–[09:30] (Bruno, Diego, Larissa)

## Contexto

O OMS já tem convenções estabelecidas de estrutura de módulo, tratamento de erro e logging, usadas de forma
consistente pelos módulos `auth`, `users`, `customers`, `products` e `orders`. Introduzir o módulo de webhooks
é uma oportunidade de decidir explicitamente entre reaproveitar essas convenções ou criar um padrão novo
específico para a feature.

Levantamento do que já existe na base de código:
- Estrutura de módulo por domínio: `src/modules/<dominio>/{<dominio>.controller,service,repository,routes,schemas}.ts`
  (ex.: `src/modules/orders/order.service.ts`, `order.repository.ts`, `order.routes.ts`, `order.schemas.ts`).
- Hierarquia de erros: `AppError` (`src/shared/errors/app-error.ts`) como classe-base com `statusCode`,
  `errorCode` e `details`; subclasses HTTP genéricas em `src/shared/errors/http-errors.ts`
  (`NotFoundError`, `ConflictError`, `UnprocessableEntityError`, `ValidationError`, `BadRequestError`,
  `UnauthorizedError`, `ForbiddenError`); e classes específicas de domínio que estendem essas, como
  `InvalidStatusTransitionError extends ConflictError` e `InsufficientStockError extends UnprocessableEntityError`.
- Middleware de erro centralizado (`src/middlewares/error.middleware.ts`), que já trata qualquer `AppError`,
  `ZodError` e erros conhecidos do Prisma, sem precisar de código específico por módulo.
- Autenticação e autorização (`src/middlewares/auth.middleware.ts`): `authenticate` (JWT Bearer) e
  `requireRole(...roles)` para restringir rotas por papel (`ADMIN` | `OPERATOR`).
- Validação de entrada com Zod, aplicada via `src/middlewares/validate.middleware.ts` (`validate({ body, params, query })`).
- Logger estruturado Pino (`src/shared/logger/index.ts`), já com `redact` configurado para campos sensíveis
  (`*.password`, `*.token`, `req.headers.authorization`, etc.).

## Decisão

O módulo de webhooks **reaproveita integralmente** esses padrões, sem introduzir bibliotecas ou convenções
novas [09:30, Larissa]:

- Estrutura: `src/modules/webhooks/` seguindo o mesmo layout controller/service/repository/routes/schemas dos
  demais módulos [09:27, Bruno].
- Erros: novas classes de erro do domínio de webhooks estendem `AppError`/as subclasses HTTP existentes em
  `src/shared/errors/`, com códigos prefixados por **`WEBHOOK_`** (ex.: `WEBHOOK_NOT_FOUND`,
  `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`), no mesmo espírito de `INSUFFICIENT_STOCK` e
  `INVALID_STATUS_TRANSITION` já usados no módulo de pedidos [09:28, Bruno].
- `src/middlewares/error.middleware.ts` **não é alterado**: por já tratar qualquer subclasse de `AppError`
  genericamente, absorve os novos erros de webhook sem modificação [09:29, Bruno].
- Autorização do endpoint administrativo de replay de DLQ reaproveita `requireRole('ADMIN')` sem alterações
  [09:36, Larissa e Sofia].
- Logging usa a instância `logger` já existente (Pino) — nenhum logger novo é introduzido [09:29, Bruno].
- O worker (`src/worker.ts`) abre sua **própria instância de `PrismaClient`**, mas aponta para a mesma
  `DATABASE_URL`/mesmo banco, já que `PrismaClient` é por processo [09:30, Bruno].

## Alternativas Consideradas

1. **Criar convenções próprias para o módulo de webhooks** (ex.: uma hierarquia de erros paralela, um logger
   dedicado, ou uma estrutura de pastas diferente por ser uma feature "de infraestrutura"). Rejeitada
   implicitamente pelo time: a decisão registrada foi de reuso máximo justamente para evitar essa
   fragmentação [09:30, Larissa: "reuso máximo do que já existe"].

## Consequências

**Positivas**
- Nenhuma curva de aprendizado adicional para o time: o módulo de webhooks é imediatamente reconhecível por
  quem já trabalha nos módulos existentes.
- `error.middleware.ts` não precisa de nenhuma alteração — reduz a superfície de mudança e o risco de
  regressão em código compartilhado por todos os módulos.
- Auditoria e observabilidade (logs, formato de erro HTTP) permanecem consistentes em toda a API.

**Negativas**
- Convenções que eventualmente não sejam ideais para um fluxo assíncrono/outbox (pensado originalmente para
  fluxo request/response síncrono) são herdadas mesmo assim — ex.: a hierarquia de `AppError` foi desenhada
  para respostas HTTP, e seu uso dentro do worker (que não responde a um request) é uma adaptação, não o caso
  de uso original dessas classes.
