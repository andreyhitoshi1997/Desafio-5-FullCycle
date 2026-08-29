# ADR-004: Autenticação HMAC-SHA256 com secret por endpoint e rotação com grace period

## Status

**Aceito** — 2026-08-29

- **Decisores:** Sofia (Eng. Segurança), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos)
- **Fonte:** `TRANSCRICAO.md` [09:19]–[09:22] (Sofia, Bruno, Diego)

## Contexto

O sistema passa a expor dados de pedidos para endpoints HTTP fora da infraestrutura da empresa. O cliente que
recebe o webhook precisa conseguir validar que a requisição realmente veio da plataforma e que o payload não
foi adulterado em trânsito [09:19, Sofia].

## Decisão

Assinar o corpo de cada requisição de webhook com **HMAC-SHA256**, usando uma secret compartilhada entre a
plataforma e o cliente, enviando o resultado no header `X-Signature` para o cliente validar do lado dele
[09:20, Sofia]. Cada **endpoint de webhook cadastrado tem sua própria secret**, gerada pela plataforma e
devolvida na criação — nunca uma secret global da plataforma, para limitar o raio de impacto caso uma secret
vaze [09:21, Sofia]. A secret é **rotacionável via API**: ao rotacionar, a secret antiga permanece válida em
paralelo por **24 horas**, dando tempo ao cliente de migrar seus sistemas, e só depois desse prazo deixa de
ser aceita [09:21, Sofia]. A tabela de configuração do webhook armazena `url`, `secret`, `customer_id` e
estado ativo [09:21, Bruno].

## Alternativas Consideradas

1. **Secret global única para toda a plataforma.** Rejeitada explicitamente: "se vaza uma, vaza tudo"
   [09:21, Sofia] — motivada inclusive por um incidente real em que um cliente vazou secret em log de
   aplicação dele [09:22, Diego].
2. **Rotação sem grace period (troca imediata, sem período de validade dupla).** Descartada implicitamente: a
   decisão registrada já inclui o grace period de 24h como parte do requisito de segurança, justamente para
   evitar quebra de integração no momento da rotação [09:21, Sofia].

## Consequências

**Positivas**
- Cliente consegue verificar autenticidade e integridade do payload usando um algoritmo padrão de mercado
  (HMAC-SHA256), amplamente suportado por bibliotecas [09:20, Sofia].
- Secret por endpoint limita o impacto de um vazamento a um único cliente/integração, não à plataforma
  inteira.
- Rotação com grace period evita downtime de integração para o cliente durante a troca de credencial.

**Negativas**
- Exige guardar duas secrets válidas simultaneamente por até 24h por endpoint durante rotações, aumentando a
  superfície de dados sensíveis a proteger (ver reuso do `redact` do logger em ADR-006/FDD).
- Mais complexidade de modelagem (estado "secret antiga ainda válida até X") comparado a uma secret única e
  estática.
