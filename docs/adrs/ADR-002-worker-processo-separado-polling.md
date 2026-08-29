# ADR-002: Worker de entrega em processo separado, com polling de 2 segundos

## Status

**Aceito** — 2026-08-29

- **Decisores:** Diego (Eng. Sênior, Plataforma), Larissa (Tech Lead), Bruno (Eng. Pleno, Pedidos)
- **Fonte:** `TRANSCRICAO.md` [09:08]–[09:11] (Diego, Bruno, Larissa, Marcos)

## Contexto

Com o padrão Outbox decidido (ADR-001), é preciso definir como os eventos pendentes em `webhook_outbox` são
lidos e entregues, e onde esse processo roda. O requisito de negócio é que a notificação chegue em menos de
10 segundos, o teto que os clientes B2B consideram "tempo real" [09:02, Marcos]. Bruno levantou se seria
possível usar um *trigger* de banco para ser mais reativo [09:09].

## Decisão

Um **worker dedicado, rodando como processo separado** da API HTTP, lê os eventos pendentes mais antigos da
`webhook_outbox` em lotes pequenos, a cada **2 segundos**, processa e marca como entregues [09:09, Diego]. O
worker roda como um novo entry-point do projeto (`src/worker.ts`, análogo ao `src/server.ts` existente),
disparado por um script `npm run worker` [09:11, Larissa], conectando ao mesmo `DATABASE_URL` mas com sua
própria instância de `PrismaClient` — cada processo Node precisa da sua, mesmo apontando para o mesmo banco
[09:29–09:30, Bruno e Diego]. Rodar em processo separado é deliberado: se a API reiniciar, o worker continua
processando o backlog [09:11, Diego].

Ordenação: com um único worker processando em ordem de `created_at`, a entrega por `order_id` respeita a
ordem de mudança de status. Não há garantia de ordenação global entre pedidos diferentes, e essa garantia se
perde caso o sistema evolua para múltiplos workers em paralelo — tratado como limitação conhecida e
documentada, não como bloqueio atual [09:12–09:13, Diego, Bruno e Larissa].

## Alternativas Consideradas

1. **Reação via trigger de banco (estilo `LISTEN`/`NOTIFY` do Postgres).** Rejeitada: MySQL não tem um
   mecanismo nativo de notificação de processo externo — um trigger só executa SQL, não avisa um processo
   fora do banco. Emular isso (escrever em arquivo, bater num endpoint) foi considerado desnecessariamente
   complexo frente ao requisito de latência já satisfeito por polling [09:09, Diego].
2. **Worker embutido no mesmo processo da API.** Rejeitada implicitamente: perderia o processamento em caso
   de reinício da API, e acoplaria o ciclo de vida de dois componentes com responsabilidades diferentes
   [09:11, Diego].

## Consequências

**Positivas**
- Simplicidade operacional: sem infraestrutura de mensageria, só um segundo processo Node.
- Isolamento de falhas: reinício/deploy da API não interrompe o processamento de eventos pendentes.
- Latência de pickup previsível e dentro do requisito de negócio (< 10s, ver ADR-001 e PRD).

**Negativas**
- Latência mínima de até ~2 segundos antes mesmo da primeira tentativa de entrega, no pior caso
  [09:10, Larissa: "a latência mínima vai ser 2 segundos no pior caso... aceitamos"].
- Ordenação de entrega só é garantida por `order_id` e enquanto o sistema operar com um único worker; escalar
  horizontalmente exigirá particionamento por `order_id` ou lock pessimista — problema explicitamente adiado
  [09:13, Diego].
