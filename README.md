# Da Reunião ao Documento: Design Docs Gerados por IA

Pacote de design docs do **Sistema de Webhooks de Notificação de Pedidos**, produzido a partir da transcrição
de uma reunião técnica (`TRANSCRICAO.md`) e do código-fonte real do Order Management System deste repositório.
O enunciado completo do desafio está no repositório base:
[devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).

Este README documenta o **processo de produção** dos documentos, não a aplicação em si (para isso, ver o
código em `src/`, `prisma/` e `tests/`, que não foram alterados).

## Sobre o desafio

O desafio propõe transformar a transcrição literal de uma reunião de decisão técnica — cinco pessoas (Tech
Lead, PM, dois engenheiros e uma engenheira de segurança) discutindo por ~55 minutos como construir um sistema
de webhooks — num pacote de documentação de engenharia acionável: um PRD que justifica a feature para o
negócio, um RFC que propõe a arquitetura e expõe alternativas descartadas, sete ADRs que registram cada
decisão isolada, um FDD que detalha como implementar (contratos HTTP, erros, fluxos, integração com o código
existente), e um Tracker que amarra cada afirmação dos documentos a um trecho específico da transcrição ou a
um arquivo real do código.

A regra central do desafio é a **rastreabilidade**: nada nos documentos pode ser inventado. Toda decisão,
requisito ou restrição precisa apontar para um `[hh:mm] Nome` da transcrição ou para um caminho de arquivo do
código-fonte. Identificar o que foi **descartado ou adiado** na reunião (rate limiting, e-mail de alerta,
dashboard, webhooks inbound) é tão parte da tarefa quanto identificar o que entrou.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
|---|---|
| **Claude Code** (Sonnet 5) | Única ferramenta de IA usada, de ponta a ponta: leitura integral da transcrição e do código-fonte real (não apenas trechos colados), exploração da estrutura de módulos/erros/schemas existente, extração e filtragem dos requisitos/decisões/restrições, redação dos sete documentos, montagem do Tracker de rastreabilidade linha a linha, e importação/versionamento do repositório base via Git. |

Nenhuma outra ferramenta (ChatGPT, Cursor, Gemini) foi usada nesta produção — optou-se por manter tudo dentro
de uma única sessão de Claude Code para preservar consistência de contexto entre os sete documentos, evitando
o risco de um documento divergir do outro por terem sido gerados em ferramentas/sessões diferentes.

## Workflow adotado

1. **Setup do repositório.** O repositório de destino estava vazio (sem commits). Antes de qualquer documento,
   foi feita a importação do repositório base (`devfullcycle/mba-ia-desafio-design-docs-com-ia`) via
   mirror-clone + push, preservando o histórico original de commits, e o trabalho passou a ocorrer na branch
   `main`.
2. **Leitura integral da transcrição** (`TRANSCRICAO.md`, as 324 linhas, do início ao fim) e **exploração do
   código real** (não resumos): `order.service.ts`, `order.status.ts`, a hierarquia de erros em
   `src/shared/errors/`, `error.middleware.ts`, `auth.middleware.ts`, `validate.middleware.ts`, o logger Pino,
   `server.ts`, `routes/index.ts` e `prisma/schema.prisma` — para que toda referência a "reuso de padrão
   existente" nos documentos apontasse para código que de fato existe.
3. **ADRs primeiro** (7 arquivos): cada uma das 6 decisões arquiteturais principais discutidas na reunião virou
   um ADR isolado, mais um sétimo cobrindo a modelagem do payload do outbox (snapshot vs. referência, UUID vs.
   auto-incremento) — uma decisão real da reunião, mas secundária o suficiente para não caber nas 6
   categorias principais.
4. **RFC**, consolidando a proposta em cima dos ADRs já escritos, com as alternativas descartadas (chamada
   síncrona, Redis Streams, trigger de banco, exactly-once) e as questões que a própria reunião deixou em
   aberto (rate limiting, ordenação ao escalar, endurecimento de autorização).
5. **FDD**, detalhando contratos HTTP, matriz de erros `WEBHOOK_*`, fluxos passo a passo e a seção obrigatória
   de integração com pelo menos 4 arquivos reais do código.
6. **PRD**, por último entre os documentos grandes — com RFC, FDD e ADRs prontos, consolidar o racional de
   produto (escopo, métricas, riscos) foi majoritariamente um exercício de síntese e checagem de rastreabilidade.
7. **Tracker**, montado varrendo os seis documentos já prontos e mapeando cada requisito/decisão/restrição
   identificável a uma linha com fonte e localização.
8. **Este README**, por último, já com o processo completo para descrever com precisão.

## Prompts customizados

Dois prompts guiaram as etapas mais sensíveis a alucinação — filtragem de escopo e montagem do tracker — em
vez de pedidos genéricos do tipo "gere o PRD a partir da transcrição":

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

Cinco correções concretas ao longo da produção, não geração de primeira tentativa sem revisão:

1. **Comando de execução incompatível com a tarefa.** A automação inicialmente disparada (`dev-loop`, com
   ciclo dev → QA → security e critério de saída baseado em "build compila e testes verdes") foi desenhada
   para mudança de código, não para produção documental — e o desafio proíbe explicitamente alterar
   `src/`/`prisma/`/`tests/`. Em vez de forçar o encaixe (rodar testes que não existem para os documentos), o
   ciclo foi adaptado: autoria dos documentos → verificação cruzada de consistência com a transcrição/código →
   revisão de segurança do conteúdo relativo a HMAC/secret, sem inventar uma etapa de "build" que não se
   aplica.
2. **Estado do repositório assumido incorretamente no início.** A suposição inicial era que o diretório de
   trabalho já continha o repositório base clonado. A checagem (`git log`, `git ls-remote`) revelou um
   repositório vazio, sem nenhum commit — foi preciso importar o repositório base antes de qualquer documento
   poder ser escrito, evitando presumir uma estrutura de arquivos que ainda não existia.
3. **Objetivo de métrica quase inflado com um número não sourceável.** Um primeiro rascunho do PRD tentou
   expressar a meta de latência como "≥95% das entregas em até 10s" — mas a transcrição não contém nenhuma
   porcentagem, só o teto absoluto de 10 segundos citado pelo PM. A meta foi reescrita usando apenas os
   números que a reunião de fato definiu (10s de teto, 2s de polling como pior caso de latência mínima), em
   vez de inventar uma taxa de sucesso plausível, mas não dita por ninguém.
4. **Matriz de erros do FDD superficial na primeira passada.** A transcrição só nomeia três códigos
   `WEBHOOK_*` literalmente (`WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`). Uma
   matriz de erros com só três linhas não cobre os cenários que o próprio FDD descreve (payload grande demais,
   timeout, replay de DLQ inexistente). A matriz foi estendida para 9 códigos seguindo o prefixo `WEBHOOK_`
   já decidido, mas cada linha estendida foi marcada no Tracker como "extensão do padrão" em vez de aparentar
   ser uma citação literal — para não passar uma falsa certeza de que aquele código específico foi dito em
   voz alta na reunião.
5. **Tracker com contagem de cobertura incorreta na primeira escrita.** O resumo de cobertura no rodapé do
   Tracker foi escrito antes da tabela estar fechada, com números arredondados ("100 linhas, 85 TRANSCRICAO").
   Após a tabela completa (107 linhas), os números foram recontados e corrigidos (91 TRANSCRICAO, 16 CODIGO)
   para o rodapé não contradizer a própria tabela que ele resume.

## Como navegar a entrega

Ordem sugerida de leitura, da mais alta para a mais baixa altitude:

1. **[docs/PRD.md](docs/PRD.md)** — por quê e o quê: problema, público, escopo, métricas de sucesso.
2. **[docs/RFC.md](docs/RFC.md)** — como propomos resolver, alternativas descartadas, questões em aberto.
3. **[docs/adrs/](docs/adrs/)** — as 7 decisões arquiteturais isoladas (`ADR-001` a `ADR-007`), cada uma com
   contexto, alternativas e consequências.
4. **[docs/FDD.md](docs/FDD.md)** — como construir: fluxos, contratos HTTP, matriz de erros, integração com o
   código existente (`src/modules/orders/order.service.ts` e outros).
5. **[docs/TRACKER.md](docs/TRACKER.md)** — para auditar qualquer afirmação dos documentos acima até sua
   origem na transcrição ou no código.
6. **`TRANSCRICAO.md`** — a fonte primária, para quem quiser conferir qualquer citação no contexto original.
