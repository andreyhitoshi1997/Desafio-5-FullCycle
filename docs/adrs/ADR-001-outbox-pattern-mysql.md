# ADR-001: Padrão Outbox no MySQL para emissão de eventos de webhook

## Status

**Aceito** — 2026-08-29

- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos)
- **Fonte:** `TRANSCRICAO.md` [09:03]–[09:08] (Bruno, Diego, Larissa)

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) pediram para ser notificados quando o
status de um pedido muda, em vez de continuar fazendo *polling* em `GET /orders` [09:00–09:02, Marcos].
O ciclo de vida do pedido é controlado por `OrderService.changeStatus` (`src/modules/orders/order.service.ts`),
que hoje já executa, numa única transação Prisma: `update` em `orders`, `insert` em `order_status_history` e
ajuste de `stock_quantity` em `products`. Essa transação é considerada "pesada" pelo time [09:04, Bruno], e
qualquer novo efeito colateral precisa entrar sem comprometer sua atomicidade nem sua performance.

A pergunta de abertura da reunião foi: o disparo do webhook deve ser síncrono dentro do `changeStatus`, ou
passar por algum mecanismo assíncrono (fila/outbox)? [09:03, Larissa]

## Decisão

Adotar o **padrão Transactional Outbox sobre o MySQL já existente**: dentro da mesma transação SQL que já
atualiza `orders` e `order_status_history`, o `OrderService` também insere um registro numa nova tabela
`webhook_outbox` contendo o evento já renderizado (ver ADR-007). Um worker separado, em outro processo, lê
essa tabela de forma assíncrona e realiza a entrega HTTP (ver ADR-002). Se a transação principal falhar e
sofrer rollback, o evento nunca chega a existir; se ela commitar, o evento está garantidamente registrado
[09:06, Diego]. Não há infraestrutura nova: a tabela vive no mesmo MySQL já usado pelo Prisma.

## Alternativas Consideradas

1. **Disparo HTTP síncrono dentro de `changeStatus`.** Rejeitada: um cliente lento ou indisponível travaria a
   transação de mudança de status de *outros* pedidos, e não existe uma ação de rollback sensata para uma
   falha de rede num sistema de terceiros [09:04, Bruno].
2. **Fila dedicada (ex.: Redis Streams).** Rejeitada: exigiria subir e operar infraestrutura nova (ex.: Redis
   Cluster) para um time pequeno — "overengineering" segundo o time, quando o MySQL já disponível resolve
   [09:06–09:07, Diego e Larissa].

## Consequências

**Positivas**
- Garantia forte de atomicidade entre a mudança de status e o registro do evento — impossível existir
  mudança de status sem evento correspondente, ou evento "órfão" de uma transação que sofreu rollback.
- Nenhuma infraestrutura nova a operar; reaproveita o MySQL e o `PrismaClient` já usados pelo time.

**Negativas**
- Introduz uma tabela de fila "artesanal" (`webhook_outbox`) que precisa de índices dedicados (`status`,
  `created_at` [09:08, Diego]) e de uma rotina futura de arquivamento das linhas entregues após ~30 dias,
  explicitamente fora do escopo desta feature [09:08, Diego].
- A entrega não é instantânea: depende de um worker de *polling* assíncrono (latência mínima associada,
  tratada na ADR-002).
