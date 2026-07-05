# ADR-002: Worker em processo separado com polling de 2 segundos

## Status

Aceito

## Contexto

Uma vez que eventos são gravados na tabela `webhook_outbox` (ver [ADR-001](ADR-001-outbox-pattern-mysql.md)), algo precisa lê-los e efetivamente enviar as chamadas HTTP para os endpoints dos clientes. Os clientes B2B definiram que qualquer entrega abaixo de 10 segundos é aceitável como "tempo real" — não há requisito de latência sub-segundo.

MySQL não possui um mecanismo nativo de notificação de processo externo equivalente ao `NOTIFY`/`LISTEN` do PostgreSQL. Triggers de banco existem, mas só executam SQL; não conseguem acionar um processo Node.js fora do banco sem soluções improvisadas (escrever em arquivo, chamar um endpoint a partir do trigger), consideradas frágeis e fora do padrão do projeto.

Também é necessário decidir onde esse processo de consumo roda: dentro da mesma instância da API HTTP (`src/server.ts`) ou como processo independente.

## Decisão

O consumo da outbox é feito por um **worker em processo Node.js separado** (`src/worker.ts`, novo entry point análogo a `src/server.ts`, iniciado via `npm run worker`), que roda em **polling a cada 2 segundos**: busca o lote mais antigo de eventos pendentes (ordenados por `created_at`), processa e marca como entregues ou agenda retry.

O worker conecta-se ao mesmo banco (`DATABASE_URL`) e usa o mesmo Prisma Client, porém com uma instância própria de `PrismaClient` — cada processo Node.js deve ter sua própria instância, já que `PrismaClient` gerencia seu próprio pool de conexões por processo.

Rodar como processo separado (e não dentro da instância da API) evita que um restart de deploy da API interrompa o processamento de eventos pendentes, e permite escalar ou reiniciar cada componente de forma independente.

## Alternativas Consideradas

**Reagir a triggers do banco (equivalente a `LISTEN`/`NOTIFY` do Postgres).** Inviável no MySQL: triggers só executam SQL dentro do próprio banco, não conseguem notificar um processo externo sem mecanismos improvisados (escrita em arquivo, chamada HTTP disparada de dentro do trigger), que fogem do padrão de infraestrutura do projeto e adicionam uma superfície de falha desnecessária.

**Worker rodando dentro do mesmo processo da API (`src/server.ts`).** Mais simples de fazer deploy (um único processo), mas um restart da API (deploy, crash, autoscaling) interromperia o processamento de eventos pendentes até a próxima subida. Descartada para preservar a independência operacional entre API e processamento assíncrono de webhooks.

## Consequências

**Positivas**
- Atende com folga o requisito de latência dos clientes (abaixo de 10 segundos): no pior caso, o atraso é de até 2 segundos (intervalo de polling) mais o tempo de processamento do lote.
- Desacopla o ciclo de vida do worker do ciclo de vida da API — reiniciar/deployar um não afeta o outro.
- Implementação simples (loop de polling), sem exigir infraestrutura de mensageria adicional.

**Negativas**
- Introduz latência mínima de até 2 segundos mesmo no caso ideal (não é notificação instantânea).
- Com um único worker, a leitura em lote e o polling constante geram carga de leitura contínua sobre a tabela `webhook_outbox`, mitigada por índices em `status` e `created_at`.
- Enquanto a operação for single-worker, a ordenação de entrega por `order_id` é preservada apenas porque há um único consumidor processando em ordem de `created_at`; paralelizar workers no futuro (fora de escopo desta feature) quebraria essa garantia implícita sem particionamento por `order_id` ou lock pessimista.
