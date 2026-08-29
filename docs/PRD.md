# PRD — Sistema de Webhooks de Notificação de Pedidos

## Resumo e Contexto da Feature

O OMS hoje não tem nenhum mecanismo de notificação externa: clientes que integram com a plataforma só
descobrem mudanças em seus pedidos consultando repetidamente `GET /orders`. Esta feature introduz um sistema
de **webhooks outbound**, que notifica automaticamente sistemas de clientes B2B quando o status de um pedido
muda, eliminando a necessidade de polling. A decisão técnica de como construir foi fechada em reunião entre
Tech Lead, PM, engenharia e segurança (`TRANSCRICAO.md`); este PRD consolida o racional de produto por trás
dessa decisão.

## Problema e Motivação

Três clientes B2B — **Atlas Comercial**, **MaxDistribuição** e **Nova Cargo** — solicitaram formalmente ser
notificados em tempo real sobre mudanças de status de seus pedidos (`TRANSCRICAO.md` [09:00], Marcos). Hoje
eles fazem polling periódico em `GET /orders`, o que os próprios clientes descrevem como "lento e caro" de
manter do lado deles. O risco de negócio é concreto: a Atlas sinalizou que pode migrar para um concorrente se
a integração não for entregue até o fim do trimestre ([09:00], Marcos).

## Público-Alvo e Cenários de Uso

- **Clientes B2B integrados via API** (inicialmente Atlas Comercial, MaxDistribuição, Nova Cargo) que
  consomem o OMS programaticamente e precisam reagir a mudanças de status de pedidos sem polling.
- **Cenário principal:** o pedido de um cliente muda de `PROCESSING` para `SHIPPED`; o sistema de e-commerce
  do cliente recebe um webhook em poucos segundos e atualiza a UI dele automaticamente, sem que o cliente
  precise consultar a API.
- **Cenário de operação interna:** um operador da plataforma com role `ADMIN` precisa reprocessar
  manualmente uma notificação que falhou definitivamente (endpoint do cliente ficou fora do ar por um período
  prolongado) — `TRANSCRICAO.md` [09:18], [09:35].

## Objetivos e Métricas de Sucesso

| Objetivo | Métrica | Meta |
|---|---|---|
| Eliminar a percepção de "lentidão" reportada pelos clientes B2B ao descobrir mudanças de status | Latência entre a mudança de status do pedido e a primeira tentativa de entrega do webhook | **≤ 10 segundos**, o teto que os próprios clientes definem como "tempo real" (`TRANSCRICAO.md` [09:02], considerando o pior caso de 2s de polling do worker aceito em [09:10]) |
| Reduzir a dependência de polling client-side em `GET /orders` | Volume de chamadas `GET /orders` originadas pelos três clientes B2B após adoção do webhook (qualitativo — sem baseline numérico registrado na reunião) | Tendência de queda observável após rollout para os três clientes piloto |
| Reter os clientes B2B em risco de churn | Confirmação de prazo e adoção pela Atlas Comercial | Entrega dentro do prazo comunicado ao cliente (fim de novembro, `TRANSCRICAO.md` [09:45]) |

## Escopo

### Incluso

- Cadastro (CRUD) de endpoints de webhook por customer, com filtro de quais status disparam notificação.
- Emissão atômica de evento a cada mudança de status elegível, via padrão outbox.
- Entrega assíncrona por worker dedicado, com autenticação HMAC-SHA256, retry com backoff e Dead Letter
  Queue.
- Garantia at-least-once com deduplicação pelo cliente via `X-Event-Id`.
- Histórico de entregas consultável pelo cliente (`GET /webhooks/:id/deliveries`).
- Endpoint administrativo (`role: ADMIN`) para reprocessar manualmente itens da DLQ.
- Rotação de secret por endpoint, com grace period de 24h.

### Fora de Escopo

1. **Notificação por e-mail em caso de falha recorrente do webhook.** Explicitamente descartado para esta
   fase: "Email tá fora de escopo dessa fase. Talvez próxima fase, depois que a gente medir o impacto."
   (`TRANSCRICAO.md` [09:37]–[09:38], Larissa e Marcos).
2. **Dashboard visual para o cliente acompanhar seus webhooks.** Descartado: "Painel é projeto separado do
   time de frontend." (`TRANSCRICAO.md` [09:39]–[09:40], Larissa e Marcos). Nesta fase, só há endpoints de
   API.
3. **Webhooks inbound** (clientes enviando eventos para a plataforma). Fora de escopo desde a definição do
   problema: "Só saindo da gente pra eles. Eles querem receber, não mandar." (`TRANSCRICAO.md` [09:02],
   Marcos).
4. **Rate limiting de envio por cliente.** Adiado: "Eu acho que não [faz parte do escopo]. A gente observa e
   implementa se virar problema." (`TRANSCRICAO.md` [09:38]–[09:39], Diego e Larissa). Registrado como questão
   em aberto no [RFC](RFC.md#questões-em-aberto).
5. **Arquivamento automático de eventos entregues.** Adiado: "Linhas entregues a gente arquiva depois de 30
   dias ou assim, fora do escopo dessa feature." (`TRANSCRICAO.md` [09:08], Diego).
6. **Escalonamento para múltiplos workers em paralelo.** Adiado: "isso é problema do futuro, não agora."
   (`TRANSCRICAO.md` [09:13], Diego).

## Requisitos Funcionais

| ID | Requisito | Fonte |
|---|---|---|
| RF-01 | Cliente (via API autenticada) cadastra um webhook informando `url`, `customerId` e lista de status de interesse; a `secret` é gerada pela plataforma e devolvida na criação | [09:31]–[09:32], [09:21] |
| RF-02 | Cliente edita um webhook existente (`PATCH`) | [09:33] |
| RF-03 | Cliente remove um webhook (`DELETE`) | [09:33] |
| RF-04 | Cliente lista os webhooks cadastrados de um customer (`GET`) | [09:33] |
| RF-05 | Cada webhook filtra quais status de pedido disparam notificação para ele; o filtro é aplicado no momento da inserção no outbox, não no envio | [09:33]–[09:34] |
| RF-06 | Cliente consulta o histórico das últimas 100 entregas de um webhook (sucesso/falha, payload, resposta, tempo de resposta) | [09:34]–[09:35] |
| RF-07 | Operador `ADMIN` reprocessa manualmente um item da Dead Letter Queue via endpoint administrativo | [09:18], [09:35]–[09:36] |
| RF-08 | Cliente rotaciona a `secret` de um webhook via API; a secret antiga permanece válida por 24h em paralelo | [09:21] |
| RF-09 | Toda mudança de status do pedido gera, dentro da mesma transação, um evento de webhook para os endpoints correspondentes | [09:03]–[09:08], [09:40]–[09:41] |
| RF-10 | Cada requisição de entrega é assinada com HMAC-SHA256, validável pelo cliente via header `X-Signature` | [09:19]–[09:22] |
| RF-11 | Cada evento carrega um identificador único (`X-Event-Id`) para o cliente deduplicar entregas repetidas | [09:24]–[09:26] |
| RF-12 | Entregas com falha são reautomaticamente retentadas com backoff exponencial (5 tentativas) antes de ir para a DLQ | [09:14]–[09:19] |

*(12 requisitos funcionais identificados, acima do mínimo de 8 exigido.)*

## Requisitos Não Funcionais

| ID | Requisito | Fonte |
|---|---|---|
| RNF-01 | Latência: tentativa de entrega em até 10 segundos da mudança de status | [09:02], [09:10] |
| RNF-02 | URLs de webhook cadastradas devem ser `https`; URLs `http` são rejeitadas na validação | [09:23] |
| RNF-03 | Payload de evento limitado a 64KB; eventos maiores falham em vez de serem truncados | [09:23]–[09:24] |
| RNF-04 | Timeout de 10 segundos por chamada HTTP de entrega | [09:42] |
| RNF-05 | Garantia de entrega at-least-once (não exactly-once) | [09:24]–[09:26] |
| RNF-06 | Secret de assinatura é única por endpoint de webhook, nunca global | [09:21] |
| RNF-07 | Toda execução de replay manual de DLQ deve ser auditável (registrar quem executou) | [09:36] |
| RNF-08 | A feature deve reaproveitar a stack e os padrões já existentes (módulos, erros, logger, autenticação), sem introduzir infraestrutura nova | [09:29]–[09:30] |

## Decisões e Trade-offs Principais

As decisões arquiteturais estão detalhadas, com alternativas e consequências, nos ADRs correspondentes:
[padrão outbox](adrs/ADR-001-outbox-pattern-mysql.md),
[worker em polling](adrs/ADR-002-worker-processo-separado-polling.md),
[retry e DLQ](adrs/ADR-003-retry-backoff-dead-letter-queue.md),
[HMAC-SHA256](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md),
[at-least-once](adrs/ADR-005-at-least-once-x-event-id.md),
[reuso de padrões](adrs/ADR-006-reuso-de-padroes-existentes-do-projeto.md) e
[snapshot de payload](adrs/ADR-007-snapshot-do-payload-no-outbox.md).

Trade-off central de produto: o time optou por **at-least-once com dedup no cliente** em vez de
exactly-once, priorizando velocidade de entrega da feature (3 sprints) sobre uma garantia mais forte que
exigiria coordenação distribuída (`TRANSCRICAO.md` [09:25], Diego). Isso transfere parte da responsabilidade
de correção para o cliente, mitigado por documentação clara no portal do desenvolvedor (`TRANSCRICAO.md`
[09:26], Marcos).

## Dependências

- Estrutura de módulos, hierarquia de erros (`AppError`), middleware de erro, autenticação JWT/`requireRole`
  e logger Pino já existentes no OMS (`src/shared/`, `src/middlewares/`).
- Banco MySQL já provisionado via Prisma (`prisma/schema.prisma`) — três novas tabelas, sem infraestrutura
  adicional.
- Disponibilidade da Sofia (Eng. Segurança) para revisão dedicada de segurança antes do deploy — pelo menos 2
  dias úteis reservados, com foco em HMAC e geração de secret (`TRANSCRICAO.md` [09:46]).
- Confirmação de prazo com a Atlas por parte do Marcos (PM) — fim de novembro (`TRANSCRICAO.md` [09:45],
  [09:47]).

## Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Cliente fica indisponível por período prolongado (já ocorreu: ~2h de manutenção planejada de um cliente) e esgota as 5 tentativas de retry | Média (já ocorreu antes) | Médio — cliente perde a notificação até replay manual | Backoff generoso (~15h de janela), DLQ dedicada + endpoint de replay manual, histórico de entregas consultável (`TRANSCRICAO.md` [09:16]) |
| Vazamento de secret (já ocorreu: cliente vazou secret em log de aplicação dele) | Baixa/Média (incidente já registrado) | Alto — permitiria falsificação de webhooks assinados | Secret por endpoint (limita o raio de impacto) + rotação suportada via API com grace period de 24h (`TRANSCRICAO.md` [09:21]–[09:22]) |
| Acúmulo de eventos na tabela de outbox sob alto volume, degradando o worker | Baixa (mitigada por design) | Médio — atraso na entrega de todos os clientes | Índices dedicados em status/created_at, leitura em lotes pequenos, plano futuro de arquivamento após 30 dias (`TRANSCRICAO.md` [09:07]–[09:08]) |
| Perda de garantia de ordenação ao escalar para múltiplos workers no futuro | Baixa no curto prazo (permanece single-worker) | Médio, se ocorrer sem planejamento | Documentado como limitação conhecida agora; exigir particionamento por `order_id` antes de qualquer escala horizontal (`TRANSCRICAO.md` [09:12]–[09:13]) |
| Sobrecarga do endpoint do cliente por rajada de mudanças de status (sem rate limiting nesta fase) | Baixa/desconhecida (sem dados de produção ainda) | Baixo-Médio | Observar volume real por customer após lançamento; revisitar rate limiting se necessário (`TRANSCRICAO.md` [09:38]–[09:39]) |

## Critérios de Aceitação

- Os 12 requisitos funcionais (RF-01 a RF-12) estão implementados e cobertos por teste automatizado.
- Uma mudança de status nunca resulta em estado inconsistente entre `orders` e `webhook_outbox` (garantido
  por teste de transação — ver [FDD, seção 11](FDD.md#11-critérios-de-aceite-técnicos)).
- Uma notificação a um endpoint saudável chega em até 10 segundos da mudança de status, em ambiente de teste.
- Um webhook mal configurado (URL `http`, payload acima de 64KB) é rejeitado com o código de erro
  `WEBHOOK_*` correspondente, sem afetar a mudança de status quando a rejeição ocorre na validação do
  cadastro (RNF-02), e revertendo a transação quando ocorre na emissão do evento (RNF-03).
- Sofia (Eng. Segurança) aprova a implementação de HMAC e geração/rotação de secret antes do deploy em
  produção (`TRANSCRICAO.md` [09:46]).

## Estratégia de Testes e Validação

- **Testes de integração** (padrão já usado pelo projeto: Vitest + Supertest, ver `tests/orders.test.ts`)
  cobrindo os endpoints de CRUD de webhook, rotação de secret e histórico de entregas, subindo a API real
  contra um banco de teste.
- **Teste de atomicidade:** forçar falha na inserção do outbox (ex.: mock de erro do Prisma) e validar que a
  mudança de status inteira sofre rollback — não pode existir "status mudou, evento não" nem o inverso.
- **Testes do worker** isolados da API HTTP: simular endpoints de cliente com respostas de sucesso, timeout e
  erro 5xx, validando a progressão de backoff, o envio correto dos headers (`X-Event-Id`, `X-Signature`,
  `X-Timestamp`, `X-Webhook-Id`) e a movimentação para DLQ após a 5ª falha.
- **Teste de assinatura:** validar que `X-Signature` é uma HMAC-SHA256 correta e verificável com a `secret`
  devolvida na criação, e que a secret antiga continua válida durante o grace period de rotação.
- **Revisão de segurança dedicada** pela Sofia antes do deploy, com foco em HMAC, geração e rotação de secret
  (`TRANSCRICAO.md` [09:46]) — gate manual, não automatizável.
