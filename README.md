# Sistema de Webhooks de Notificação de Pedidos — Pacote de Design Docs

> Este README documenta o **processo de produção** do pacote de design docs deste desafio. O enunciado original do desafio foi movido para [`README-DESAFIO.md`](./README-DESAFIO.md).

## Sobre o desafio

O desafio parte de uma transcrição literal (`TRANSCRICAO.md`) de uma reunião técnica de ~55 minutos, na qual Tech Lead, PM, dois engenheiros e uma engenheira de segurança fecham a arquitetura de um sistema de webhooks outbound para o Order Management System (OMS) já existente neste repositório. Nenhuma decisão foi registrada em lugar algum além dessa conversa. A tarefa é transformar essa reunião — mais o código-fonte real da aplicação — em um pacote completo de design docs (PRD, RFC, FDD, ADRs e um Tracker de rastreabilidade), acionável o suficiente para o time começar a implementar sem ambiguidade.

O ponto central do exercício não é "gerar documentos", é **não inventar nada**: toda decisão, requisito ou restrição registrada em qualquer documento precisa apontar de volta para um timestamp específico da transcrição (`[hh:mm] Nome`) ou para um caminho real de arquivo no código-fonte. Identificar o que foi explicitamente descartado ou adiado na reunião é tão importante quanto capturar o que foi decidido.

## Ferramentas de IA utilizadas

- **Claude Code (Claude Sonnet 5)** — ferramenta principal de produção de todo o pacote. Leu `TRANSCRICAO.md` e o código-fonte diretamente do repositório (via ferramentas de leitura de arquivo, não colagem manual), gerou cada documento em Markdown e foi usado interativamente para revisão e correção de cada entrega antes de avançar para a próxima.
- **Skills customizadas de ADR** (`adr-map`, `adr-identify`, `adr-generate`, do plugin do curso) — usadas em uma sessão anterior para mapear a base de código em módulos (`docs/adrs/mapping.md`) e identificar/gerar as 8 ADRs a partir desse mapeamento, antes do trabalho de RFC/FDD/PRD/Tracker documentado aqui.
- **Prompts de entrevista customizados para RFC/FDD/PRD** (adaptados dos prompts disponibilizados no curso) — usados como referência de estrutura de saída (esqueleto de documento e regras de conteúdo mínimo), mas executados em **modo direto** em vez do fluxo de entrevista pergunta-a-pergunta original, já que a transcrição e o código já continham todas as respostas necessárias (ver seção "Iterações e ajustes").

## Workflow adotado

O trabalho seguiu a ordem sugerida no enunciado do desafio, com um ajuste de ferramenta importante:

1. **ADRs primeiro.** Antes do início desta conversa, as 8 ADRs em `docs/adrs/` já haviam sido produzidas em sessão separada, usando as skills `adr-map` (mapeamento modular do código-fonte) → `adr-identify` (identificação de decisões candidatas por módulo) → `adr-generate` (redação formal de cada ADR com citações de transcrição e código). Essas ADRs formaram o "esqueleto de decisões fechadas" sobre o qual todo o resto foi construído.
2. **RFC.** Consolidação das 8 ADRs em uma proposta de arquitetura única, no nível "o que propomos e por quê" — sem redigir o "como implementar" (isso ficou reservado para o FDD). Nesta etapa, a IA foi instruída a ler `TRANSCRICAO.md`, `README.md` (enunciado) e todas as ADRs antes de escrever, e a perguntar em caso de ambiguidade.
3. **FDD.** O usuário colou um prompt de skill de entrevista para FDD (formato pergunta-a-pergunta). Em vez de seguir a entrevista literalmente, foi feita uma pergunta de esclarecimento (via `AskUserQuestion`) confirmando se o documento deveria ser gerado direto, reaproveitando tudo o que já havia sido lido (transcrição, código, RFC, ADRs), ou via entrevista passo a passo. A resposta foi "direto", e esse padrão se repetiu no PRD.
4. **PRD.** Mesma dinâmica do FDD: prompt de entrevista colado, mas geração direta, priorizando a estrutura de seções exigida pelo `README.md` (enunciado) sobre o esqueleto genérico do prompt de entrevista, já que a primeira é a que efetivamente é avaliada.
5. **Tracker.** Compilação final: cada documento já produzido (PRD, RFC, FDD, 8 ADRs) foi varrido item a item para gerar uma linha correspondente na tabela de rastreabilidade, sem introduzir nenhuma informação nova.
6. **README do processo** (este arquivo) — escrito por último, documentando a jornada real, incluindo os ajustes de abordagem que aconteceram no meio do caminho.

Em cada etapa, a instrução explícita para a IA foi a mesma: nada de inventar, tudo rastreável à transcrição ou ao código, e perguntar em caso de dúvida real em vez de assumir.

## Prompts customizados

**1. Prompt de geração direta do RFC**, escrito para forçar leitura das fontes antes de qualquer geração e para tornar explícita a regra de "não inventar":

```
Com base no arquivo @TRANSCRICAO.md , @README.md e em @docs/adrs/*.md,
gere em @docs/RFC.md a documentação RFC, não invente nada, tudo tenha
base nos documentos citados, em caso de ambiguidade, me pergunte.
```

**2. Trecho adaptado do prompt de entrevista de FDD** (originalmente pensado para rodar como entrevista pergunta-a-pergunta), reaproveitado como especificação de estrutura de saída obrigatória em modo de geração direta:

```
Garanta capturar, no mínimo, as seguintes seções do FDD:
- Contexto e motivação técnica
- Objetivos técnicos
- Escopo e exclusões
- Fluxos detalhados e diagramas
- Contratos públicos (assinaturas, endpoints, headers, exemplos)
- Erros, exceções e fallback
- Observabilidade
- Dependências e compatibilidade
- Critérios de aceite técnicos
- Riscos e mitigação

Regras:
- Não invente detalhes técnicos sem rotular como hipótese.
- Para cada contrato público, forneça exemplos mínimos e semântica
  de campos/headers.
- Em "Observabilidade", especifique métricas, logs e tracing que
  validam o comportamento da feature.
```

Esse mesmo padrão (esqueleto do prompt de entrevista + regra de não invenção) foi reaproveitado na geração direta do PRD, adaptando as seções para o formato de "Esqueleto de PRD" fornecido pelo prompt original do curso.

## Iterações e ajustes

1. **Entrevista interativa vs. geração direta (FDD e PRD).** Os prompts de skill colados pelo usuário para FDD e PRD foram desenhados como entrevistas de uma pergunta por vez. Rodar isso literalmente teria ignorado o fato de que a transcrição e o código já continham praticamente todas as respostas necessárias, e teria produzido um processo mais lento sem ganho de qualidade. Em vez de assumir a decisão, foi usada a ferramenta de pergunta ao usuário (`AskUserQuestion`) para confirmar explicitamente: gerar direto reaproveitando o contexto já levantado, ou seguir a entrevista passo a passo. O usuário escolheu geração direta nas duas ocasiões — esse foi o ajuste de processo mais significativo da sessão.
2. **Esqueleto do prompt de entrevista vs. estrutura exigida pelo `README.md`.** O prompt de FDD colado usa um formato de saída (`### FDD: [nome]`, seções numeradas 1-10) que não é idêntico à estrutura mínima obrigatória listada no enunciado do desafio (que exige, por exemplo, a seção específica "Integração com o sistema existente" com no mínimo 4 caminhos de arquivo reais, algo que o prompt genérico de entrevista não previa). A decisão foi usar o esqueleto do prompt como inspiração de formatação, mas dar prioridade à estrutura de seções do `README.md`, já que é o que efetivamente é avaliado pelos critérios de aceite.
3. **Correção de erro de escrita de arquivo.** Ao gerar `docs/RFC.md` e `docs/FDD.md` pela primeira vez, o `Write` falhou porque o arquivo alvo (ainda com o placeholder `<!-- documento a ser elaborado -->`) não havia sido lido explicitamente nessa etapa da conversa antes da escrita. Corrigido lendo o arquivo primeiro e reexecutando a escrita — sem impacto no conteúdo final, mas um lembrete de que edição de arquivo existente sempre exige leitura prévia.
4. **Grounding explícito de cada afirmação técnica no FDD.** Antes de escrever a seção obrigatória "Integração com o sistema existente", os arquivos reais do código (`order.service.ts`, `app-error.ts`, `http-errors.ts`, `auth.middleware.ts`, `error.middleware.ts`, `validate.middleware.ts`, `app.ts`, `server.ts`, `database.ts`, `env.ts`, `logger/index.ts`, `schema.prisma`, `order.controller.ts`, `order.routes.ts`, `order.schemas.ts`, `routes/index.ts`) foram lidos diretamente, em vez de reaproveitar apenas os trechos já citados nas ADRs. Isso permitiu citar números de linha exatos (ex.: `order.service.ts:126-179`) e evitar qualquer menção a arquivo ou método inexistente.

No total, foram 3 ciclos principais de geração e ajuste: (1) confirmação de abordagem via `AskUserQuestion` antes do FDD e do PRD, (2) correções mecânicas de leitura-antes-de-escrita, e (3) verificação cruzada de cada caminho de código citado contra o código-fonte real antes de fechar o FDD e o Tracker.

## Como navegar a entrega

Ordem sugerida de leitura, do mais alto nível ao mais detalhado:

1. [`docs/PRD.md`](./docs/PRD.md) — por que a feature existe, para quem, e o que define "pronto"
2. [`docs/RFC.md`](./docs/RFC.md) — proposta técnica de arquitetura, alternativas descartadas e questões em aberto
3. [`docs/adrs/`](./docs/adrs/) — as 8 decisões arquiteturais fechadas, uma por arquivo (`ADR-001` a `ADR-008`)
4. [`docs/FDD.md`](./docs/FDD.md) — especificação de implementação: fluxos, contratos HTTP, matriz de erros, integração com o código existente
5. [`docs/TRACKER.md`](./docs/TRACKER.md) — tabela de rastreabilidade cruzando cada item dos documentos acima com `TRANSCRICAO.md` ou com o código-fonte
6. [`TRANSCRICAO.md`](./TRANSCRICAO.md) e o código em `src/`/`prisma/schema.prisma` — fontes primárias, para conferência
7. [`README-DESAFIO.md`](./README-DESAFIO.md) — enunciado original do desafio e critérios de aceite
