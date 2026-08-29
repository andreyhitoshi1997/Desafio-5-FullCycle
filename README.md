# Da Reunião ao Documento: Design Docs Gerados por IA

Pacote de design docs do **Sistema de Webhooks de Notificação de Pedidos**, produzido a partir de
`TRANSCRICAO.md` e do código-fonte real deste OMS. Enunciado completo do desafio:
[devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).

## Sobre o desafio

Transformar a transcrição literal de uma reunião técnica (Tech Lead, PM, dois engenheiros e uma engenheira de
segurança, ~55 minutos) num pacote de documentação acionável: PRD (por quê/o quê), RFC (proposta de arquitetura
e alternativas descartadas), 7 ADRs (cada decisão isolada), FDD (como implementar) e um Tracker que rastreia
cada afirmação até a transcrição ou o código. A regra central é rastreabilidade: nada pode ser inventado, e
identificar o que foi **descartado/adiado** (rate limiting, e-mail de alerta, dashboard, webhooks inbound) é
tão parte da tarefa quanto identificar o que entrou.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
|---|---|
| **Claude Code** (Sonnet 5) | Única ferramenta usada: leitura integral da transcrição e do código real, extração e filtragem de requisitos/decisões, redação dos documentos, montagem do Tracker linha a linha, e auditoria final de consistência contra o código. |

## Workflow adotado

1. Leitura integral de `TRANSCRICAO.md` e exploração do código real (`order.service.ts`, `order.status.ts`,
   hierarquia de erros, middlewares, logger, `server.ts`, `routes/index.ts`, `prisma/schema.prisma`).
2. **ADRs primeiro** (7 arquivos) — as 6 decisões arquiteturais principais + 1 decisão secundária de modelagem
   (snapshot de payload / UUID no outbox).
3. **RFC**, consolidando a proposta em cima dos ADRs, com alternativas descartadas e questões deixadas em aberto.
4. **FDD**, com fluxos, contratos HTTP, matriz de erros `WEBHOOK_*` e a seção de integração com o código existente.
5. **PRD**, por último entre os documentos grandes — com RFC/FDD/ADRs prontos, virou síntese e checagem de escopo.
6. **Tracker**, varrendo os documentos prontos e mapeando cada item a uma origem.
7. Auditoria final: releitura de cada afirmação técnica do FDD/ADRs contra o código-fonte real, arquivo por
   arquivo, para confirmar que nenhum método, classe ou caminho citado é inexistente ou está descrito errado.

## Prompts customizados

```
Releia TRANSCRICAO.md do início ao fim. Para cada tópico discutido, classifique em exatamente uma categoria:

1. DECIDIDO — vira requisito/decisão nos documentos, com timestamp+nome de quem fechou a decisão.
2. DESCARTADO — foi proposto e explicitamente rejeitado. Cite a fala que rejeita e o motivo dado.
3. ADIADO — reconhecido como válido, mas empurrado para fase futura ou "não é agora".
4. DETALHE SECUNDÁRIO — mencionado, mas não é decisão arquitetural (ex.: validação de schema).

Não crie uma quarta opção "implícito" ou "provável". Se não houver uma fala que sustente a categoria,
o item não entra em nenhum documento. Liste separadamente os itens DESCARTADO/ADIADO — eles vão para a
seção "Fora de Escopo" do PRD e "Questões em Aberto" do RFC, não para os requisitos funcionais.
```

```
Para cada linha que você adicionar ao Tracker, preencha a coluna Localização ANTES de decidir se a linha
existe. Se você não conseguir apontar um [hh:mm] Nome específico em TRANSCRICAO.md ou um caminho de
arquivo real em src//prisma//tests/ que sustente o item, não crie a linha — e volte ao documento de
origem (PRD/RFC/FDD/ADR) e remova ou reformule a frase que gerou esse item, porque ela provavelmente não
tem origem identificável. Não aceite "inferido do contexto geral da reunião" como localização válida.
```

## Iterações e ajustes

1. **Objetivo de métrica quase inflado.** Um rascunho do PRD tentou expressar a meta como "≥95% das entregas
   em até 10s" — a transcrição só define um teto absoluto de 10s, sem nenhuma porcentagem. Reescrito usando
   apenas os números que a reunião de fato definiu.
2. **Matriz de erros superficial.** A transcrição só nomeia 3 códigos `WEBHOOK_*` literalmente. Uma matriz de
   3 linhas não cobria os cenários que o próprio FDD descreve (payload grande, timeout, replay inexistente).
   Estendida para 9 códigos, com cada linha estendida marcada no Tracker como "extensão do padrão" em vez de
   parecer citação literal da reunião.
3. **Cobertura do Tracker escrita antes da tabela fechar.** O resumo no rodapé tinha números arredondados
   escritos antes da tabela estar completa. Recontado após a tabela final (107 linhas: 91 TRANSCRICAO, 16
   CODIGO) para não contradizer a própria tabela que resume.
4. **Auditoria final linha a linha.** Depois dos documentos prontos, cada referência técnica do FDD e dos
   ADRs (nomes de método, classes de erro, arquivos citados na seção de integração) foi conferida de novo
   contra o código-fonte real, um arquivo por vez, em vez de confiar na primeira leitura.

## Como navegar a entrega

1. **[docs/PRD.md](docs/PRD.md)** — problema, público, escopo, métricas de sucesso.
2. **[docs/RFC.md](docs/RFC.md)** — proposta de arquitetura, alternativas descartadas, questões em aberto.
3. **[docs/adrs/](docs/adrs/)** — as 7 decisões isoladas (`ADR-001` a `ADR-007`).
4. **[docs/FDD.md](docs/FDD.md)** — fluxos, contratos HTTP, matriz de erros, integração com o código existente.
5. **[docs/TRACKER.md](docs/TRACKER.md)** — rastreabilidade de cada afirmação até a transcrição ou o código.
6. **`TRANSCRICAO.md`** — fonte primária.
