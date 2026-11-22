# Design Docs com IA: Sistema de Webhooks de Notificação de Pedidos

Este README documenta o processo de produção do pacote de design docs do desafio. O enunciado original está em [`INSTRUCTIONS.MD`](INSTRUCTIONS.MD) e a fonte primária em [`TRANSCRICAO.md`](TRANSCRICAO.md).

## Sobre o desafio

O desafio parte de uma transcrição de reunião técnica de ~55 minutos (5 participantes) que decide uma feature de webhooks de notificação de pedidos para um OMS existente em Node/Express/Prisma/MySQL, e do código-fonte desse OMS. A tarefa é transformar essas duas fontes em um pacote de documentação de design: PRD, RFC, FDD, 7 ADRs, um tracker de rastreabilidade e este README de processo. A regra central é anti-alucinação: todo requisito, decisão, restrição ou contrato registrado precisa ser rastreável a um timestamp da transcrição (formato `[hh:mm] Nome`) ou a um caminho real do código. O código-fonte e a transcrição não são alterados; a entrega é 100% documental.

Esse desafio foi bem interessante de se implementar, pois se trata de um cenário muito próximo do que eu trabalho no dia a dia. Como eu já uso ferramentas de IA há bastante tempo no trabalho para fazer documentações técnicas, aproveitei para testar os prompts disponibilizados pelo professor, aprender novas técnicas com base nesses prompts e comparar os resultados (observando o que eu posso levar para meu dia a dia de trabalho para melhorar a qualidade das minhas documentações). Uma questão interessante também é que eu costumo trabalhar com a geração de documentações em inglês e nesse desafio trabalhei com elas em português (por conta disso, precisei alterar alguns termos durante as revisões do que foi gerado pela IA).

## Ferramentas de IA utilizadas

- **Claude Code (CLI)**: ferramenta principal, agente com acesso direto ao repositório, usada para ler o código e mapear as referências de integração, escrever e revisar os documentos e rodar as verificações manuais via bash.
- **Prompts do curso** (pasta [`prompts/`](prompts/)): base de formato para os geradores de PRD, FDD, ADR e diagramas. Não usados como estão: cada um foi adaptado ao contexto do desafio (detalhado na seção de prompts customizados).

## Workflow adotado

A produção foi organizada em 7 etapas com checkpoint de revisão ao final de cada uma, no papel de maestro: a IA gera, eu reviso e aprovo antes de seguir (indicando ajustes caso necessário).

1. **Análise prévia**: leitura completa do enunciado, da transcrição e do código. Consolidação em um arquivo de trabalho, `WORK_ANALYSIS.md` (não faz parte do pacote entregável): mapa decisor com timestamps, referências de código, requisitos, fora de escopo, alternativas descartadas, questões em aberto e cuidados de geração. Após a análise prévia, foi gerado um plano de implementação `IMPLEMENTATION_PLAN.md` e foi usado para acompanhar a implementação de cada etapa com detalhes específicos.
2. **ADRs** (7 arquivos): primeiro porque as decisões isoladas alimentam todos os outros documentos.
3. **RFC**: proposta de arquitetura consumindo os ADRs.
4. **FDD**: detalhamento de implementação, contratos e integração com o código real.
5. **PRD**: por último entre os grandes documentos; com RFC, FDD e ADRs prontos, vira praticamente uma consolidação em nível de produto (ordem também sugerida pelo enunciado).
6. **TRACKER**: varredura dos documentos prontos e montagem da matriz de rastreabilidade.
7. **README** (este arquivo) e revisão final contra a checklist de critérios de aceite.

Prática transversal: ao final de cada etapa, verificação manual via bash (timestamps citados existem na transcrição, caminhos de código citados existem no repositório, contagens mínimas, ausência do travessão) seguida de uma passada de revisão crítica do próprio conteúdo antes de apresentar o checkpoint. As correções encontradas nessas passadas ficaram registradas no log de iterações da `WORK_ANALYSIS.md`.

## Prompts customizados

O prompt inicial de análise foi montado puramente por mim com base na forma que trabalho no dia a dia. Após essa primeira análise, foram gerados os documentos `WORK_ANALYSIS.md` e `IMPLEMENTATION_PLAN.md`.

**1. Análise Inicial**

```text
Fale em português durante essa sessão. Primeiramente, faça a análise do documento INSTRUCTIONS.MD para entender o objetivo desse projeto (basicamente, esse
projeto faz parte da resolução de um exercício de um curso de pós-graduação). Após isso, analise a estrutura do projeto por completo
para entender como ele funciona (a pasta prompts foi adicionada por mim para disponibilizar alguns prompts que foram usados durante as aulas). Após essa análise, crie um arquivo com a arquitetura do projeto, informações e observações importantes (esse arquivo é para ser complementar ao INSTRUCTIONS.MD, pois o INSTRUCTIONS.MD já tem quase todas as infos necessárias). Com isso, sua primeira tarefa é fazer toda essa
análise inicial do projeto. Se notar qualquer ambiguidade ou ficar alguma dúvida me avise antes de seguir para a próxima etapa (que será relacionada a criar um plano de implementação para esse exercício) 
```

Os prompts do curso (disponibilizados pelo professor) foram adaptados em vez de usados como estão. As duas adaptações principais:

**2. Geração de ADRs** (adaptado de `prompts/adr-generator.md`). O prompt original busca "ADRs potenciais" do histórico git de um repositório com histórico longo, classifica em tiers e marca lacunas com `[NEEDS INPUT]`. Aqui a fonte não é o git, é a transcrição, então o pipeline original do prompt foi removido e o foco passou a ser o mapa decisor:

```text
A partir do mapa decisor da WORK_ANALYSIS.md (seção 3), produza 7 ADRs no
formato MADR adaptado, um por decisão principal + snapshot do payload:
ADR-001 outbox no MySQL, ADR-002 retry com backoff e DLQ, ADR-003 HMAC-SHA256
com secret por endpoint, ADR-004 at-least-once com X-Event-Id, ADR-005 worker
separado em polling, ADR-006 reuso dos padrões existentes, ADR-007 snapshot do
payload na inserção.

Regras:
- Header: Status (Aceito), Data, ADRs relacionados.
- 7 seções: Contexto, Direcionadores da Decisão,
  Alternativas Consideradas, Decisão, Prós e Contras das Opções,
  Consequências, Referências.
- Máximo 3 opções por ADR, máximo 5 referências de arquivo, sem code snippets.
- Alternativas reais da transcrição, com o timestamp de quem as levantou.
- Todo fato citado precisa de timestamp [hh:mm] Nome ou caminho real de arquivo.
- Arquivos em docs/adrs/ADR-NNN-titulo-em-kebab-case.md.
- Sem travessão como conectivo; termos técnicos em inglês.
```

**3. Geração do FDD** (adaptado de `prompts/fdd-generator.md`). O prompt original conduz uma entrevista para levantar o conteúdo; aqui a fonte já existe (transcrição + código), então a entrevista foi substituída pela leitura das fontes, mantendo o esqueleto de saída:

```text
Produza docs/FDD.md para a feature de webhooks usando o esqueleto de saída do
prompts/fdd-generator.md (10 seções numeradas), sem a entrevista: a fonte é a
TRANSCRICAO.md e o código existente.

Acrescente a seção obrigatória do desafio "Integração com o sistema existente"
com pelo menos 4 caminhos reais do repositório.

Na seção de contratos: 7 endpoints da API com request/response JSON e status
codes, o contrato do envio outbound (payload [09:43] e headers [09:44]-[09:45])
e a função publishWebhookEvent(tx, order, fromStatus, toStatus) [09:41].
Na seção de erros: matriz WEBHOOK_* (os 3 códigos citados em [09:28] + derivados
no mesmo padrão, com a derivação declarada).
Diagramas mermaid embutidos aplicando as regras do
prompts/mermaid-diagram-generator.md inline (sequência, fluxograma do worker,
ER das tabelas), sem arquivos extras.

Todo item precisa de referência: timestamp ou path. Sem travessão.
```

A mesma lógica se aplicou ao PRD (esqueleto do `prompts/prd-generator.md` sem a entrevista) e ao RFC (sem prompt do curso; estrutura pelas seções obrigatórias do enunciado).

**4. Verificação manual** (prompt customizado do processo, rodado ao final de cada etapa):

```text
Verifique manualmente o documento desta etapa via bash:
1. Nenhum travessão em-dash no arquivo.
2. Todo timestamp citado [hh:mm] existe como início de linha na TRANSCRICAO.md.
3. Todo caminho de código citado existe no repositório (test -f).
4. Contagens mínimas do enunciado (FRs, endpoints, alternativas, etc.).
5. Links internos entre documentos resolvem.
Se qualquer check falhar, mostre a linha exata do problema antes de corrigir.
```

## Iterações e ajustes

Foram 6 iterações principais, todas capturadas pelas passadas de revisão ou pela verificação manual antes do checkpoint da etapa. A primeira está registrada na seção 12 da `WORK_ANALYSIS.md`; as demais ficam documentadas apenas aqui.

1. **ADR-003 com citação errada**: a justificativa do HMAC dizia "modelo de provedores consolidados como Stripe e GitHub" citando `[09:25] Diego`, mas nesse timestamp a transcrição fala de at-least-once com event_id. A fala correta sobre HMAC como padrão de mercado é da Sofia em `[09:20]`. Causa: generalização de "padrão de mercado" sem conferir a fonte exata.
2. **RFC com citação malformada**: `[09:31]-09:34]` sem o colchete de abertura do segundo timestamp, pego na revisão antes do checkpoint.
3. **FDD com três desvios nas regras de mermaid**: participantes com espaço no nome sem aspas (`API Express`, `Endpoint cliente`), labels de nó acima do limite de 3 palavras e entidades do ER em MAIÚSCULAS divergindo dos nomes de tabela minúsculos usados nos ADRs (`webhook_outbox`, `webhook_dead_letter`).
4. **PRD com prazo impreciso na fonte**: a meta dizia "fim do trimestre" citando o trecho `[09:45]-[09:47]`, onde o prazo concreto é "fim de novembro"; "fim do trimestre" é a formulação de `[09:00]`. Cada afirmação ficou com a referência que de fato a sustenta.
5. **TRACKER com referências misturadas e bug no script de verificação**: a linha do modelo de dados misturava dois fatos com timestamps diferentes (nome da tabela em `[09:06]`, índice em `[09:08]`); e o próprio script de verificação tinha um bug de regex que gerava falso negativo em todos os 87 pares de timestamp + falante, e foi reescrito com prefixo fixo via awk. Lição: quando a verificação falha de forma suspeita (tudo errado de uma vez), a verificação também precisa ser verificada.
6. **Termos em português ajustados**: como eu usei o glm (modelo chinês) para a realização desse exercício, alguns termos em português ficaram "estranhos" ou errados, o que é um problema comumente conhecido de modelos chineses quando trabalham usando português. Alguns exemplos simples foram o uso do termo "mecânico" ao invés de "manual" e "âncora" ao invés de "referência".

O padrão das iterações 1 e 4 (as mais importantes) é o mesmo: a IA produz texto plausível demais e só a conferência da referência exata expõe o desvio. É exatamente o problema que o TRACKER existe para minimizar.

## Como navegar a entrega

Ordem de leitura sugerida, do porquê ao como:

| Ordem | Arquivo | O que contém |
| --- | --- | --- |
| 0 | [`INSTRUCTIONS.MD`](INSTRUCTIONS.MD) | Enunciado original do desafio |
| 1 | [`TRANSCRICAO.md`](TRANSCRICAO.md) | Fonte primária: a reunião que decide a feature |
| 2 | [`docs/PRD.md`](docs/PRD.md) | O quê e por quê: problema, público, escopo, 10 FRs, NFRs e riscos |
| 3 | [`docs/RFC.md`](docs/RFC.md) | O que propomos: proposta técnica, alternativas descartadas e questões em aberto |
| 4 | [`docs/adrs/`](docs/adrs/README.md) | As 7 decisões isoladas (índice em `docs/adrs/README.md`) |
| 5 | [`docs/FDD.md`](docs/FDD.md) | Como construir: fluxos, contratos, matriz de erros, integração com o código |
| 6 | [`docs/TRACKER.md`](docs/TRACKER.md) | Rastreabilidade: 99 itens com referências na transcrição ou no código |

Arquivos do processo (não fazem parte do pacote avaliado, mas documentam o caminho): `WORK_ANALYSIS.md` (mapa decisor e log de iterações), `IMPLEMENTATION_PLAN.md` (plano das etapas) e [`prompts/`](prompts/) (prompts do curso usados como base). O código-fonte do OMS (`src/`, `prisma/`, `tests/`) permanece intocado.
