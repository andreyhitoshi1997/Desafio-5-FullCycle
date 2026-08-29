# ADR-003: Retry com backoff exponencial (5 tentativas) e Dead Letter Queue em tabela separada

- **Status:** Aceito
- **Data:** 2026-08-29
- **Decisores:** Diego (Eng. Sênior, Plataforma), Larissa (Tech Lead), Bruno (Eng. Pleno, Pedidos)
- **Fonte:** `TRANSCRICAO.md` [09:14]–[09:19] (Diego, Bruno, Larissa, Marcos)

## Contexto

Endpoints de clientes ficarão indisponíveis eventualmente (o time já presenciou um caso real de ~2 horas de
indisponibilidade por manutenção planejada de um cliente [09:16, Diego]). É preciso decidir quantas vezes
retentar a entrega, com qual progressão de espera, e o que fazer quando as tentativas se esgotam.

## Decisão

**Backoff exponencial com 5 tentativas** no total, com a progressão **1 minuto, 5 minutos, 30 minutos, 2
horas, 12 horas** entre tentativas — uma janela total de quase 15 horas entre a primeira falha e a última
tentativa [09:15–09:17, Diego]. Esgotadas as 5 tentativas, o evento é considerado falha permanente e movido
para uma **tabela separada `webhook_dead_letter`**, contendo payload, motivo da falha e timestamp — mantendo
a `webhook_outbox` principal "limpa" e servindo de evidência para debug e reprocessamento [09:18, Diego].
Reprocessamento de itens da DLQ é **manual**, via endpoint administrativo
`POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente [09:18, Diego] e
exige role `ADMIN` (ver ADR-006 sobre reuso de `requireRole`).

## Alternativas Consideradas

1. **3 tentativas, mais agressivo.** Proposta por Bruno [09:16]. Rejeitada: uma indisponibilidade de manhã
   esgotaria as 3 tentativas em ~30 minutos, matando o evento antes que o cliente tivesse chance de voltar —
   cenário já observado na prática (indisponibilidade de 2h de um cliente) [09:16, Diego].
2. **Retry indefinido com backoff, sem teto de tentativas.** Descartada verbalmente por Diego [09:15]: deixa
   eventos "pendurados para sempre" quando o cliente efetivamente desapareceu, sem sinalizar falha permanente
   nem permitir intervenção humana.
3. **Marcar falha permanente como status na própria tabela `webhook_outbox`** (`status = 'failed'`), em vez de
   tabela separada. Rejeitada: polui a leitura da fila principal de pendentes e mistura duas responsabilidades
   (fila operacional vs. registro de auditoria/debug de falhas) [09:18, Diego e Bruno].

## Consequências

**Positivas**
- Cobre janelas realistas de indisponibilidade de cliente (até ~15h) sem exigir intervenção manual imediata.
- DLQ separada mantém a fila operacional enxuta e fornece um registro dedicado, auditável, para investigação.
- Endpoint de replay dá ao time (e não ao sistema) o controle final sobre reenviar eventos definitivamente
  falhos, evitando reprocessamento automático de payloads potencialmente problemáticos.

**Negativas**
- Um cliente fora do ar por mais de ~15h perde a notificação até uma ação manual de replay — aceito
  explicitamente pelo PM como cenário de "cliente já com problema sério dele" [09:17, Marcos].
- Reprocessamento manual introduz dependência de operação humana (e de auditoria de quem faz o replay, ver
  ADR-006) em vez de recuperação totalmente automática.
