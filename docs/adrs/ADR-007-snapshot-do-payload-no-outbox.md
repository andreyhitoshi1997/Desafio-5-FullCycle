# ADR-007: Snapshot do payload no momento da inserção no outbox, com id UUID

- **Status:** Aceito
- **Data:** 2026-08-29
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos)
- **Fonte:** `TRANSCRICAO.md` [09:51]–[09:52] (Diego, Larissa, Bruno)

## Contexto

Ao modelar a tabela `webhook_outbox` (ADR-001), ficaram duas perguntas de modelagem em aberto ao final da
reunião: (1) o identificador do registro do outbox deve ser auto-incremental ou UUID; e (2) o evento deve
guardar o payload já renderizado no momento da inserção, ou apenas o `order_id`, renderizando o payload só na
hora do envio pelo worker.

## Decisão

- **Id do outbox em UUID**, seguindo o padrão já usado em todo o resto do projeto — todas as tabelas do
  Prisma schema usam `@id @default(uuid()) @db.Char(36)` [09:51, Larissa: "UUID, segue o padrão do resto do
  projeto. Tudo é uuid."].
- **Snapshot do payload no momento da inserção**: o evento grava o payload já montado (os campos do pedido no
  estado daquele momento) dentro da própria linha do outbox, em vez de guardar só uma referência (`order_id`)
  e remontar o payload na hora do envio [09:52, Larissa e Diego].

## Alternativas Consideradas

1. **Guardar apenas `order_id` e renderizar o payload no momento do envio.** Rejeitada: se o pedido mudar de
   novo antes do worker processar o evento mais antigo, o payload enviado refletiria um estado diferente do
   que motivou o evento original, criando uma inconsistência entre "o que mudou" e "o que foi notificado"
   [09:52, Larissa: "Se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou.
   Senão tem caso esquisito."].
2. **Id auto-incremental para o outbox.** Rejeitada por quebrar a convenção uniforme de identificadores já
   adotada em todas as tabelas existentes do projeto (`prisma/schema.prisma`).

## Consequências

**Positivas**
- Cada evento é imutável e autocontido: reflete fielmente o estado do pedido no instante da transição de
  status, independentemente do que acontecer com o pedido depois.
- Consistente com a convenção de identificadores (UUID) já usada em `users`, `customers`, `products`,
  `orders`, `order_items` e `order_status_history`.

**Negativas**
- A linha do outbox fica maior (guarda o payload inteiro, não só uma referência), o que reforça a necessidade
  do limite de 64KB por evento e da rotina futura de arquivamento de eventos entregues (ver ADR-001 e FDD).
- Qualquer enriquecimento futuro do payload precisa ser decidido no momento da inserção — não é possível
  "completar" um evento já gravado com dados calculados depois.
