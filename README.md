# Sistema de Webhooks de Notificação de Pedidos — Design Docs

> O enunciado original do desafio ("Da Reunião ao Documento: Design Docs Gerados por IA") não faz parte deste repositório — a especificação completa dos critérios de aceite pode ser consultada no repositório base do desafio: https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia. Este README documenta o **processo de produção** desta entrega, não o enunciado em si.

## Sobre o desafio

Este repositório é um fork do desafio "Da Reunião ao Documento", da MBA Full Cycle de Engenharia de IA. O ponto de partida é um Order Management System (OMS) real em produção — Node.js, TypeScript, Express, Prisma/MySQL — e a transcrição literal de uma reunião técnica de ~55 minutos ([TRANSCRICAO.md](TRANSCRICAO.md)) onde tech lead, PM, dois engenheiros e uma engenheira de segurança fecham a decisão de construir um Sistema de Webhooks de Notificação de Pedidos.

A tarefa não foi "resumir a reunião": foi produzir, a partir da transcrição e do código-fonte existente, um pacote de documentação técnica (PRD, RFC, FDD, ADRs, Tracker) acionável o suficiente para um time de engenharia começar a implementar — sem inventar nenhum requisito, decisão ou restrição que não tivesse origem identificável na fala de algum participante ou em um arquivo real do código.

## Ferramentas de IA utilizadas

- **Claude Code (Claude Sonnet 5, via CLI/extensão)** — ferramenta principal de produção de todo o pacote. Usado para: ler a transcrição por completo, explorar o código-fonte do OMS via um subagente de exploração dedicado, redigir os 6 ADRs, o RFC, o FDD, o PRD, o Tracker de rastreabilidade e este README, e para autovalidar o resultado contra o checklist de critérios de aceite do desafio antes da entrega.

## Workflow adotado

A ordem de produção seguiu a recomendação do próprio enunciado do desafio — decisões antes de propostas, propostas antes de detalhamento de implementação, visão de produto por último:

1. **Fork e clone** do repositório base via `gh repo fork` + `git clone`.
2. **Leitura integral da transcrição** (`TRANSCRICAO.md`), identificando decisões fechadas, requisitos explícitos, itens descartados e itens adiados — antes de tocar em qualquer documento.
3. **Exploração do código-fonte** delegada a um subagente de leitura (Explore), com um prompt dirigido pedindo trechos de código reais (não apenas nomes de arquivo) de: estrutura de módulos, transação de `changeStatus` em `order.service.ts`, hierarquia de `AppError`, middleware de erro, logger Pino, autenticação/`requireRole`, schema Prisma e estratégia de testes.
4. **ADRs primeiro** (6 decisões: outbox no MySQL, worker em processo separado com polling, retry + DLQ, HMAC por endpoint, at-least-once com `X-Event-Id`, reuso de padrões existentes) — cada uma com alternativas realmente descartadas na reunião.
5. **RFC**, consolidando a proposta em nível de arquitetura e linkando os ADRs já escritos, com alternativas e questões em aberto.
6. **FDD**, o documento mais extenso, detalhando modelo de dados, os 4 fluxos (criação do evento, processamento do worker, retry, DLQ), 7 contratos HTTP com payload de exemplo, matriz de erros `WEBHOOK_*` e a seção obrigatória de integração citando 9 caminhos de arquivo reais do código.
7. **PRD**, por último entre os documentos grandes, consolidando objetivo de negócio, público-alvo, métricas e riscos a partir do que já estava decidido nos ADRs/RFC/FDD.
8. **Tracker**, montado ao final varrendo os quatro documentos já prontos e mapeando cada item (66 no total) à origem exata na transcrição (`[hh:mm] Nome`) ou a um caminho de arquivo real do código.
9. **Revisão final** item a item contra o checklist de critérios de aceite do enunciado (contagem de requisitos, seções obrigatórias, cobertura do tracker, existência real de cada arquivo de código citado) antes deste README.

## Prompts customizados

Dois prompts centrais guiaram a parte mais sensível a alucinação do processo — a extração de fatos do código e a filtragem do que **não** deveria virar requisito:

**1. Prompt de exploração de código (delegado a subagente), pedindo trechos reais em vez de descrições genéricas:**

```
Preciso de um relatório detalhado com caminhos de arquivo exatos e trechos de
código relevantes (não apenas nomes) sobre:
1. Estrutura de src/modules — mostre o conteúdo real de 1-2 módulos completos.
2. OrderService.changeStatus — como funciona a transação (update em orders,
   insert em order_status_history, decremento de estoque). Mostre o código real.
3. Classes de erro: AppError e subclasses (InsufficientStockError,
   InvalidStatusTransitionError). Estrutura completa e convenção de nomes.
4. Middleware de erro centralizado — como trata AppError, Zod e Prisma.
5. Logger Pino — onde configurado e padrão de uso.
6. Autenticação/JWT e requireRole — código completo.
7. schema.prisma — entidades principais, enum de status, geração de IDs.
[...]
Não precisa opinar nem sugerir nada sobre a feature de webhooks — só reporte
fielmente o que existe hoje no código, com caminhos de arquivo e trechos
copiados.
```

Esse prompt foi desenhado para produzir insumo verificável (trechos reais, caminhos exatos) em vez de um resumo genérico — cada afirmação do FDD sobre "como integrar com o código existente" precisava ser rastreável a algo que esse relatório efetivamente citou.

**2. Prompt de filtragem de escopo (usado ao redigir a seção "Fora de escopo" do PRD e as "Questões em aberto" do RFC), forçando a IA a distinguir decisão fechada de item descartado/adiado:**

```
Releia a transcrição entre [09:36] e [09:40]. Para cada tópico levantado
(notificação por e-mail em falha, rate limiting de envio, dashboard visual,
nível de autorização do CRUD), classifique explicitamente como:
(a) decidido e fechado nesta reunião,
(b) explicitamente descartado/adiado para fase futura, ou
(c) levantado mas deixado sem decisão.
Cite o timestamp e o nome de quem fala. Não liste nada como requisito se a
classificação for (b) ou (c) — isso vai para "Fora de escopo" ou "Questões
em aberto", nunca para a lista de requisitos funcionais.
```

Esse prompt foi o que efetivamente evitou que "rate limiting" e "notificação por e-mail" — ambos mencionados na call, nenhum dos dois decidido como requisito desta fase — vazassem para dentro dos Requisitos Funcionais do PRD.

## Iterações e ajustes

O processo não foi de geração única — os pontos de correção mais relevantes:

1. **Separação RFC vs. FDD.** A primeira tentativa de esboço do RFC tendia a descer para detalhe de contrato HTTP (payloads, headers) que é papel do FDD. Foi necessário reescrever a seção "Proposta técnica" do RFC para ficar em nível de arquitetura (outbox + worker + segurança, sem payload de exemplo), movendo todo o detalhamento de contratos e matriz de erros exclusivamente para o FDD — a distinção de "altura" entre os documentos, como o próprio enunciado do desafio adverte, não é automática.
2. **Erro de idioma detectado em revisão.** Um ADR (HMAC/secret por endpoint) foi gerado com uma frase misturando inglês e português ("como already sinalizado pela responsável por segurança"). Identificado numa releitura linha a linha do ADR e corrigido antes da entrega.
3. **Risco de tracker inflado.** A primeira lista de "matriz de erros" do FDD incluía códigos como `WEBHOOK_PAYLOAD_TOO_LARGE` e `WEBHOOK_DEAD_LETTER_NOT_FOUND`, que são elaborações técnicas razoáveis mas **não têm citação literal na transcrição nem no código**. Em vez de forçar uma linha de tracker "quase verdadeira" para cada um, a decisão foi deixá-los apenas no FDD como detalhamento de implementação e não incluí-los no Tracker — e documentar essa exclusão deliberada na própria seção de cobertura do `TRACKER.md`, para deixar explícito o que foi inferência de engenharia vs. fato rastreável.
4. **Verificação final de caminhos de arquivo.** Antes de fechar a entrega, todos os caminhos de arquivo citados no FDD e nos ADRs (`src/modules/orders/order.service.ts`, `src/shared/errors/app-error.ts`, `prisma/schema.prisma`, etc. — 15 no total) foram checados um a um contra o repositório real, para garantir que nenhum arquivo mencionado fosse inexistente.
5. **Reuso incorreto de `NotFoundError` detectado numa auditoria posterior.** O FDD e o ADR-006 afirmavam que `WEBHOOK_NOT_FOUND` seria produzido por reuso direto de `NotFoundError`, mas essa classe (`src/shared/errors/http-errors.ts`) fixa o código em `NOT_FOUND` e não aceita customização — diferente de `ConflictError`/`UnprocessableEntityError`. Corrigido para exigir uma classe nova estendendo `AppError` diretamente.

No total, o processo passou por 4 ciclos principais: (i) geração inicial de ADRs → RFC → FDD → PRD → Tracker; (ii) revisão de separação de altura entre RFC e FDD e correção do erro de idioma; (iii) revisão final item a item contra o checklist de critérios de aceite do desafio, incluindo a verificação de existência de arquivos citados; (iv) auditoria independente posterior que encontrou e corrigiu o reuso incorreto de `NotFoundError`.

## Como navegar a entrega

Ordem sugerida de leitura, do mais alto nível ao mais detalhado:

1. [docs/PRD.md](docs/PRD.md) — por que a feature existe, para quem, e o que define sucesso.
2. [docs/RFC.md](docs/RFC.md) — a proposta técnica em nível de arquitetura, alternativas descartadas e questões em aberto.
3. [docs/adrs/](docs/adrs/) — as 6 decisões arquiteturais isoladas (`ADR-001` a `ADR-006`), cada uma com contexto, alternativas e consequências.
4. [docs/FDD.md](docs/FDD.md) — o "como construir": modelo de dados, fluxos, contratos HTTP, matriz de erros, resiliência, observabilidade e integração com o código existente.
5. [docs/TRACKER.md](docs/TRACKER.md) — rastreabilidade de cada item dos documentos acima até a transcrição ou o código.
6. [TRANSCRICAO.md](TRANSCRICAO.md) — fonte primária, útil para conferir qualquer citação `[hh:mm] Nome` do Tracker.

## Critérios de Aceite

A entrega foi revisada item a item contra o checklist completo do enunciado original (seção "Critérios de Aceite" do desafio base) antes do push final — incluindo contagem mínima de requisitos funcionais, presença de todas as seções obrigatórias em cada documento, cobertura mínima do Tracker (≥80% dos itens, ≥70% com fonte TRANSCRICAO, ≥5 com fonte CODIGO) e verificação de que nenhum arquivo de código citado é inexistente no repositório.
