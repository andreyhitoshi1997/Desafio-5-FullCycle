# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
|---|---|
| **Título** | Sistema de Webhooks de Notificação de Pedidos |
| **Autor** | Larissa (Tech Lead) — comprometeu-se a abrir o "doc de design da feature" ao final da reunião (`TRANSCRICAO.md` [09:50]) |
| **Status** | Aceito — decisões fechadas em reunião síncrona; pontos remanescentes listados em [Questões em Aberto](#questões-em-aberto) |
| **Data** | 2026-08-29 |
| **Revisores** | Marcos (PM), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Eng. Segurança) |
| **Reunião de origem** | `TRANSCRICAO.md` — reunião técnica, quinta-feira 09:00–09:53 |

## TL;DR

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) pediram para ser notificados em tempo real
quando o status de um pedido muda, em vez de continuar fazendo *polling* em `GET /orders`
(`TRANSCRICAO.md` [09:00]). Propomos um **sistema de webhooks outbound**: quando `OrderService.changeStatus`
muda o status de um pedido, um evento é gravado atomicamente numa tabela `webhook_outbox` (padrão
Transactional Outbox), e um **worker separado, em polling de 2s**, entrega o evento via HTTP ao(s) endpoint(s)
que o cliente cadastrou, assinado com **HMAC-SHA256**, com **retry exponencial + Dead Letter Queue** e
garantia **at-least-once** deduplicável por `X-Event-Id`. Nenhuma infraestrutura nova é introduzida: tudo roda
sobre o MySQL e a stack (Express, Prisma, Pino) já existentes, seguindo os padrões de módulo e erro já usados
pelo projeto.

## Contexto e Problema

Hoje o OMS não tem nenhum mecanismo de notificação externa, eventos ou filas. Os três clientes B2B batem
repetidamente em `GET /orders` para descobrir mudanças de status, o que os próprios clientes descrevem como
lento e caro de manter — a Atlas chegou a sinalizar risco de migrar para um concorrente se isso não for
resolvido até o fim do trimestre (`TRANSCRICAO.md` [09:00], Marcos). O requisito de latência aceito pelos
clientes é "abaixo de 10 segundos" para ser considerado tempo real [09:02, Marcos].

O ciclo de vida do pedido já é controlado por uma máquina de estados (`src/modules/orders/order.status.ts`) e
por uma transação que atualiza `orders`, `order_status_history` e `stock_quantity`
(`src/modules/orders/order.service.ts`, método `changeStatus`). Qualquer solução de notificação precisa somar
a esse fluxo sem comprometer sua atomicidade nem sua performance, e sem acoplar a mudança de status a um
serviço de terceiros potencialmente lento ou indisponível [09:04, Bruno].

## Proposta Técnica

Em alto nível, a solução tem três partes:

1. **Emissão atômica do evento.** `OrderService.changeStatus` insere, na mesma transação SQL da mudança de
   status, um registro em `webhook_outbox` com o payload do evento já renderizado (snapshot). Isso garante que
   nunca existe mudança de status sem evento correspondente, nem evento "órfão" de uma transação revertida.
   Ver **[ADR-001](adrs/ADR-001-outbox-pattern-mysql.md)** e **[ADR-007](adrs/ADR-007-snapshot-do-payload-no-outbox.md)**.

2. **Entrega assíncrona por worker dedicado.** Um processo Node separado (`src/worker.ts`), com sua própria
   instância de `PrismaClient` apontando para o mesmo banco, faz *polling* da `webhook_outbox` a cada 2
   segundos, busca os eventos pendentes mais antigos e realiza a entrega HTTP para os endpoints de webhook
   ativos do customer, filtrados por status de interesse (o cliente escolhe quais status quer ouvir; o filtro
   é aplicado já na inserção do outbox, para não gerar linhas desnecessárias). Ver
   **[ADR-002](adrs/ADR-002-worker-processo-separado-polling.md)**.

3. **Entrega confiável e verificável.** Cada requisição de entrega é assinada com HMAC-SHA256 (secret única
   por endpoint de webhook, rotacionável com grace period de 24h) e carrega um `X-Event-Id` (UUID) para o
   cliente deduplicar. Falhas de entrega acionam retry com backoff exponencial (5 tentativas, até ~15h de
   janela); esgotadas as tentativas, o evento vai para uma tabela `webhook_dead_letter`, reprocessável
   manualmente via endpoint administrativo. Ver **[ADR-003](adrs/ADR-003-retry-backoff-dead-letter-queue.md)**,
   **[ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)** e **[ADR-005](adrs/ADR-005-at-least-once-x-event-id.md)**.

O módulo é implementado como `src/modules/webhooks/`, seguindo a mesma estrutura
controller/service/repository/routes/schemas dos módulos existentes, reaproveitando `AppError`, o middleware
de erro centralizado, `requireRole`, Zod e o logger Pino já presentes no projeto — ver
**[ADR-006](adrs/ADR-006-reuso-de-padroes-existentes-do-projeto.md)**. O detalhamento de contratos HTTP,
fluxos e modelagem de dados está no **[FDD](FDD.md)**; o racional produto/negócio (escopo, métricas,
critérios de aceite) está no **[PRD](PRD.md)**.

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Disparo HTTP síncrono dentro de `changeStatus`** | Um cliente lento ou fora do ar travaria a transação de mudança de status de *outros* pedidos, e não há rollback sensato para uma falha de rede em terceiro externo (`TRANSCRICAO.md` [09:04], Bruno). |
| **Fila dedicada (ex.: Redis Streams)** | Exigiria subir e operar infraestrutura nova para um time pequeno; considerado overengineering frente ao MySQL já disponível ([09:06–09:07], Diego e Larissa). |
| **Worker reativo via trigger de banco (estilo `LISTEN`/`NOTIFY`)** | MySQL não tem notificação nativa de processo externo; emular isso é mais complexo que um polling de 2s, que já atende ao requisito de latência ([09:09], Diego). |
| **Garantia exactly-once** | Exigiria coordenação de duas fases entre plataforma e cliente; at-least-once + dedup por `X-Event-Id` resolve 99% dos casos com muito menos complexidade e é o padrão adotado por Stripe/GitHub ([09:25], Diego). |

Detalhamento completo de cada decisão, incluindo consequências positivas e negativas, está nos ADRs
correspondentes (seção "Decisões Relacionadas" abaixo).

## Questões em Aberto

1. **Rate limiting de envio por cliente.** Se um customer tiver dezenas de pedidos mudando de status no mesmo
   minuto, o worker pode "bombardear" o endpoint dele com chamadas simultâneas. O time decidiu **não**
   implementar rate limiting nesta fase; a decisão é observar o volume real em produção e revisitar se virar
   problema (`TRANSCRICAO.md` [09:38–09:39], Diego e Larissa).
2. **Ordenação global ao escalar para múltiplos workers.** A garantia de ordenação por `order_id` depende de
   operar com um único worker. Se o sistema precisar escalar horizontalmente, será necessário particionamento
   por `order_id` ou lock pessimista — problema explicitamente adiado, sem desenho definido
   (`TRANSCRICAO.md` [09:12–09:13], Diego, Bruno e Larissa).
3. **Endurecimento de autorização do CRUD de configuração.** Hoje qualquer usuário autenticado (`ADMIN` ou
   `OPERATOR`) pode gerenciar webhooks; só o replay de DLQ exige `ADMIN`. Sofia sinalizou que isso pode ser
   revisto mais à frente, sem decisão fechada (`TRANSCRICAO.md` [09:36–09:37], Sofia e Marcos).

## Impacto e Riscos

- **Impacto no fluxo crítico de pedidos:** `OrderService.changeStatus` ganha uma escrita adicional dentro da
  mesma transação. Uma falha ao inserir no outbox deve fazer rollback de toda a mudança de status — não pode
  existir cenário de status mudado sem evento registrado (`TRANSCRICAO.md` [09:40–09:41], Bruno e Diego).
- **Risco de acúmulo de dados:** sem uma rotina de arquivamento (fora do escopo desta fase), a
  `webhook_outbox` e a `webhook_dead_letter` crescem indefinidamente. Mitigado por índices dedicados e por um
  plano futuro de arquivamento após 30 dias ([09:08], Diego).
- **Risco de segurança:** exposição de dados de pedidos para fora da infraestrutura da empresa. Mitigado por
  HMAC-SHA256, secret por endpoint, TLS obrigatório e rotação de secret — Sofia reservou pelo menos dois dias
  úteis de revisão de segurança dedicada antes do deploy, com foco em HMAC e geração de secret
  (`TRANSCRICAO.md` [09:46], Sofia).
- **Estimativa:** ~3 sprints, incluindo a revisão de segurança da Sofia no fechamento (`TRANSCRICAO.md`
  [09:45–09:47], Larissa).

## Decisões Relacionadas

- [ADR-001 — Padrão Outbox no MySQL](adrs/ADR-001-outbox-pattern-mysql.md)
- [ADR-002 — Worker em processo separado com polling de 2s](adrs/ADR-002-worker-processo-separado-polling.md)
- [ADR-003 — Retry com backoff exponencial e Dead Letter Queue](adrs/ADR-003-retry-backoff-dead-letter-queue.md)
- [ADR-004 — HMAC-SHA256 com secret por endpoint](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)
- [ADR-005 — At-least-once com dedup via X-Event-Id](adrs/ADR-005-at-least-once-x-event-id.md)
- [ADR-006 — Reuso dos padrões existentes do projeto](adrs/ADR-006-reuso-de-padroes-existentes-do-projeto.md)
- [ADR-007 — Snapshot do payload no outbox, id UUID](adrs/ADR-007-snapshot-do-payload-no-outbox.md)

Detalhamento de implementação (contratos HTTP, matriz de erros, fluxos passo a passo): **[FDD.md](FDD.md)**.
Racional de produto (escopo, métricas de sucesso, riscos de negócio): **[PRD.md](PRD.md)**.
