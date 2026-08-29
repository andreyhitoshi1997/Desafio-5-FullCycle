# ADR-005: Garantia at-least-once com dedup via X-Event-Id

- **Status:** Aceito
- **Data:** 2026-08-29
- **Decisores:** Diego (Eng. Sênior, Plataforma), Sofia (Eng. Segurança), Marcos (PM)
- **Fonte:** `TRANSCRICAO.md` [09:24]–[09:26] (Diego, Bruno, Sofia, Marcos)

## Contexto

Com retry automático (ADR-003), é possível que o cliente receba o mesmo evento mais de uma vez — por exemplo,
se a entrega teve sucesso mas a confirmação (resposta HTTP) se perdeu antes do worker registrar o sucesso. É
preciso decidir qual garantia de entrega o sistema oferece e como o cliente lida com duplicatas.

## Decisão

O sistema garante **at-least-once delivery**: o cliente pode, eventualmente, receber o mesmo evento mais de
uma vez, e deve estar preparado para isso [09:24, Diego]. Para viabilizar a deduplicação do lado do cliente,
cada evento carrega um `event_id` — um **UUID gerado no momento em que o evento entra na outbox**, único por
evento — enviado no header **`X-Event-Id`**. O cliente que receber o mesmo `event_id` duas vezes é responsável
por dedicar do lado dele [09:25, Diego].

## Alternativas Consideradas

1. **Garantia exactly-once.** Rejeitada: exigiria coordenação de duas fases entre plataforma e cliente,
   elevando muito a complexidade de implementação e operação para resolver um problema que o padrão de
   mercado (Stripe, GitHub) já resolve de forma mais simples com at-least-once + dedup no cliente
   [09:25, Diego].

## Consequências

**Positivas**
- Simplicidade de implementação: o worker não precisa de coordenação distribuída para garantir entrega única,
  apenas reenviar em caso de dúvida sobre sucesso.
- Alinhado a um padrão de mercado já conhecido pelos integradores (Stripe, GitHub), reduzindo a curva de
  aprendizado do cliente [09:25, Diego].
- Marcos se compromete a documentar o comportamento de forma destacada no portal do desenvolvedor, mitigando
  o risco de surpresa para o cliente [09:26, Marcos].

**Negativas**
- Transfere parte da responsabilidade de correção para o cliente: ele precisa implementar dedup por
  `event_id` do lado dele — Sofia registra essa ressalva explicitamente [09:25: "Isso joga responsabilidade
  pro cliente"].
- Sem uma garantia de exactly-once, cenários de bug no cliente que ignorem a dedicação podem gerar
  processamento duplicado de efeitos colaterais do lado dele — risco aceito e delegado à documentação da API.
