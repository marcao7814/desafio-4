# FDD — Feature Design Document: Sistema de Webhooks de Notificação de Pedidos

## Contexto e motivação técnica

O `OrderService` (`src/modules/orders/order.service.ts`) já controla o ciclo de vida do pedido através de uma máquina de estados (`src/modules/orders/order.status.ts`) e uma transação que atualiza `orders`, insere em `order_status_history` e ajusta `stock_quantity`. Esta feature estende esse fluxo para também publicar um evento de notificação, sem adicionar chamadas de rede síncronas a essa transação e sem exigir nova infraestrutura de mensageria — ver [RFC](RFC.md) e [ADR-001](adrs/ADR-001-outbox-pattern-mysql.md).

Este documento assume as decisões já fechadas nos ADRs como dadas e detalha como implementá-las: modelo de dados, fluxos passo a passo, contratos HTTP, matriz de erros, resiliência e observabilidade.

## Objetivos técnicos

- Garantir que todo evento de notificação seja consistente com a mudança de status que o originou (nunca um sem o outro), sem exigir coordenação distribuída.
- Entregar eventos aos clientes com latência de pior caso abaixo de 10 segundos.
- Garantir autenticidade e integridade do payload entregue, com possibilidade de rotação de credencial sem downtime de integração.
- Sobreviver a indisponibilidade temporária de um cliente (horas) sem perda silenciosa de eventos.
- Não introduzir nenhuma convenção nova de erro, logging, autenticação ou estrutura de módulo além das já usadas no projeto.

## Escopo e exclusões

**Dentro do escopo:** CRUD de configuração de webhook por cliente, captura de evento via outbox transacional, worker de entrega com retry e DLQ, autenticação HMAC com rotação de secret, histórico de entregas, endpoint administrativo de replay de dead-letter.

**Fora de escopo (explicitamente descartado ou adiado na reunião):**
- Notificação de falha ao cliente (ex.: e-mail quando o webhook dele falha repetidamente) — adiado para fase futura ([09:37] Marcos, [09:37] Larissa).
- Rate limiting de envio por cliente — decisão de "observar e decidir depois", não implementado nesta fase ([09:38]-[09:39] Diego/Larissa).
- Dashboard visual para o cliente acompanhar webhooks — fora de escopo, projeto separado do time de frontend ([09:39]-[09:40] Larissa).
- Arquivamento/retenção de eventos entregues após 30 dias — mencionado como necessidade futura, não implementado nesta feature ([09:08] Diego).
- Suporte a múltiplos workers em paralelo / particionamento de ordering — arquitetura atual é single-worker ([09:12]-[09:13] Diego/Bruno).

## Modelo de dados (novas tabelas)

Todas as novas tabelas usam `id` como `String @id @default(uuid())` (`@db.Char(36)`), seguindo a convenção já usada em todo `prisma/schema.prisma` ([09:51] Larissa: "UUID, segue o padrão do resto do projeto. Tudo é uuid.").

- **`webhook_endpoint`**: `id`, `customerId` (FK `Customer`), `url` (`VARCHAR`, validado como HTTPS), `eventTypes` (JSON com lista de `OrderStatus` que o endpoint quer receber), `secret` (secret ativa), `previousSecret` (nullable, usada durante rotação), `previousSecretExpiresAt` (nullable, `DateTime`), `active` (`Boolean`), `createdAt`, `updatedAt`.
- **`webhook_outbox`**: `id`, `webhookEndpointId` (FK), `eventId` (UUID, é o valor enviado em `X-Event-Id`), `eventType`, `payload` (JSON, snapshot renderizado no momento da inserção — [09:52] Larissa/Diego/Bruno: "snapshot na inserção"), `status` (`PENDING | PROCESSING | DELIVERED | FAILED`), `attempts` (`Int`, default 0), `nextAttemptAt` (`DateTime`), `lastError` (`Text`, nullable), `createdAt`, `updatedAt`. Índices em `status` e `createdAt` ([09:08] Diego).
- **`webhook_delivery`**: `id`, `webhookEndpointId` (FK), `eventId`, `success` (`Boolean`), `responseStatus` (`Int`, nullable), `responseBodySnippet` (`Text`, nullable), `durationMs` (`Int`), `attemptedAt` (`DateTime`). Alimenta `GET /webhooks/:id/deliveries` ([09:34] Marcos).
- **`webhook_dead_letter`**: `id`, `webhookEndpointId` (FK), `eventId`, `payload` (JSON), `reason` (`Text`), `failedAt` (`DateTime`) ([09:18] Diego: "tabela `webhook_dead_letter` separada, com a payload, motivo da falha e timestamp").

## Fluxos detalhados

### 1. Criação do evento na outbox

1. Um caller (ex.: rota `PATCH /orders/:id/status`) invoca `OrderService.changeStatus`, que abre `this.prisma.$transaction(async (tx) => {...})` como já ocorre hoje.
2. Após validar a transição (`canTransition`) e aplicar débito/reposição de estoque, e **antes de commitar**, o service chama uma nova função `publishWebhookEvent(tx, order, fromStatus, toStatus)` (proposta em [09:41] Bruno/Diego), localizada em `src/modules/webhooks/webhook.service.ts` ou módulo equivalente.
3. `publishWebhookEvent` busca (via `tx`) os `webhook_endpoint` ativos do `customerId` do pedido cujo `eventTypes` contém `toStatus`. Se nenhum endpoint estiver interessado, nada é inserido — o filtro é aplicado na inserção, não no envio ([09:34] Bruno/Diego: "Filtra na inserção... Economiza linha na tabela").
4. Para cada endpoint interessado, insere uma linha em `webhook_outbox` com `eventId = uuid()`, `payload` já renderizado (`event_id, event_type: "order.status_changed", timestamp ISO 8601, order_id, order_number, from_status, to_status, customer_id, total_cents` — sem `items`, [09:43] Diego), `status: PENDING`, `nextAttemptAt: now()`.
5. A função retorna dentro da mesma `tx`; se qualquer insert falhar, toda a transação (incluindo a mudança de status) sofre rollback — é essencial que a inserção esteja dentro da transação principal ([09:40]-[09:41] Bruno/Diego: "Se ficar fora da transação, perde a garantia toda").
6. A transação principal commita: `orders`, `order_status_history`, `stock` e `webhook_outbox` são persistidos atomicamente.

### 2. Processamento pelo worker

1. `src/worker.ts` inicializa um `PrismaClient` próprio (nova instância de processo, mesma `DATABASE_URL`) e entra em loop de polling a cada 2 segundos ([09:09] Diego).
2. A cada iteração, busca um lote de `webhook_outbox` com `status IN (PENDING, FAILED)` e `nextAttemptAt <= now()`, ordenado por `createdAt` (ordem de entrega por `order_id` preservada apenas em regime single-worker, [09:12] Diego).
3. Para cada evento do lote, marca `status = PROCESSING`, monta os headers (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type: application/json`) e envia `POST` ao `url` do `webhook_endpoint`, com timeout de 10 segundos ([09:42] Diego/Sofia).
4. **Sucesso** (2xx dentro do timeout): grava `webhook_delivery` com `success: true`, atualiza `webhook_outbox.status = DELIVERED`.
5. **Falha** (timeout, erro de rede, status não-2xx): grava `webhook_delivery` com `success: false` e segue para o fluxo de retry (seção 3).

### 3. Retry com backoff exponencial

1. Em caso de falha, incrementa `attempts` do registro em `webhook_outbox`.
2. Se `attempts < 5`: calcula `nextAttemptAt = now() + backoff[attempts]`, onde `backoff = [1min, 5min, 30min, 2h, 12h]` ([09:17] Diego), define `status = FAILED` (aguardando próxima tentativa) e grava `lastError`.
3. Se `attempts >= 5`: segue para o fluxo de dead-letter (seção 4) em vez de reagendar.

### 4. Dead Letter Queue e replay

1. Ao esgotar as 5 tentativas, o worker cria uma linha em `webhook_dead_letter` com o `payload` original, `reason` (motivo da última falha) e `failedAt`, e remove (ou marca como encerrado) o registro correspondente em `webhook_outbox` ([09:18] Diego).
2. Um administrador (role `ADMIN`) pode reprocessar manualmente via `POST /admin/webhooks/dead-letter/:id/replay` ([09:18]-[09:19] Diego/Larissa), que recria uma linha em `webhook_outbox` com `status: PENDING`, `attempts: 0`, a partir do `payload` armazenado na dead-letter, e loga o `userId` do administrador que executou a ação para fins de auditoria ([09:36] Sofia: "o endpoint de admin tem que logar quem fez o replay").

## Contratos públicos

Todos os endpoints ficam sob `/api/v1/webhooks` (CRUD e histórico) e `/api/v1/admin/webhooks` (replay), autenticados via `authenticate` (JWT Bearer). Segue o envelope de erro padrão do projeto (`src/middlewares/error.middleware.ts`): `{ "error": { "code", "message", "details"? } }`.

### 1. `POST /api/v1/webhooks` — cadastrar endpoint de webhook

Requisição:
```json
{
  "customerId": "8f14e45f-ceea-467e-b76a-9daf6a3ee453",
  "url": "https://integracoes.atlas-comercial.com/webhooks/oms",
  "eventTypes": ["PAID", "SHIPPED", "DELIVERED"]
}
```

Resposta `201 Created` (secret devolvida uma única vez, na criação):
```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "customerId": "8f14e45f-ceea-467e-b76a-9daf6a3ee453",
  "url": "https://integracoes.atlas-comercial.com/webhooks/oms",
  "eventTypes": ["PAID", "SHIPPED", "DELIVERED"],
  "secret": "whsec_7f9c3a1e4b2d4f6a8c0e1b3d5f7a9c1e",
  "active": true,
  "createdAt": "2026-07-05T13:00:00.000Z"
}
```

Erros possíveis: `400 WEBHOOK_INVALID_URL` (URL não é HTTPS), `400 WEBHOOK_EVENT_TYPES_EMPTY`, `404 NOT_FOUND` (reuso direto de `new NotFoundError('Customer')` — gera o código genérico `NOT_FOUND`, não um código específico `CUSTOMER_NOT_FOUND`, já que a validação do cliente não foi um ponto discutido na reunião e não há necessidade de um código dedicado para este caso).

### 2. `GET /api/v1/webhooks?customerId=...` — listar webhooks de um cliente

Requisição: sem corpo — filtros via query string, ex.: `GET /api/v1/webhooks?customerId=8f14e45f-ceea-467e-b76a-9daf6a3ee453&page=1&pageSize=20`.

Resposta `200 OK` (envelope de paginação padrão de `src/shared/http/response.ts`):
```json
{
  "data": [
    {
      "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "customerId": "8f14e45f-ceea-467e-b76a-9daf6a3ee453",
      "url": "https://integracoes.atlas-comercial.com/webhooks/oms",
      "eventTypes": ["PAID", "SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-07-05T13:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```
Nota: `secret` nunca é retornada em respostas de leitura, apenas no momento da criação ou rotação.

### 3. `PATCH /api/v1/webhooks/:id` — editar endpoint

Requisição:
```json
{ "eventTypes": ["SHIPPED", "DELIVERED"], "active": true }
```

Resposta `200 OK`:
```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "customerId": "8f14e45f-ceea-467e-b76a-9daf6a3ee453",
  "url": "https://integracoes.atlas-comercial.com/webhooks/oms",
  "eventTypes": ["SHIPPED", "DELIVERED"],
  "active": true,
  "updatedAt": "2026-07-05T14:00:00.000Z"
}
```

Erros possíveis: `404 WEBHOOK_NOT_FOUND`, `400 WEBHOOK_INVALID_URL`.

### 4. `DELETE /api/v1/webhooks/:id` — remover endpoint

Requisição: sem corpo — `DELETE /api/v1/webhooks/3fa85f64-5717-4562-b3fc-2c963f66afa6`.

Resposta: `204 No Content`. Erros possíveis: `404 WEBHOOK_NOT_FOUND`.

### 5. `POST /api/v1/webhooks/:id/rotate-secret` — rotacionar secret

Requisição: sem corpo — `POST /api/v1/webhooks/3fa85f64-5717-4562-b3fc-2c963f66afa6/rotate-secret`.

Resposta `200 OK`:
```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "secret": "whsec_1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d",
  "previousSecretValidUntil": "2026-07-06T14:00:00.000Z"
}
```
Semântica: a secret anterior permanece válida por 24 horas em paralelo à nova ([09:21] Sofia). Erros possíveis: `404 WEBHOOK_NOT_FOUND`.

### 6. `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas

Requisição: sem corpo — `GET /api/v1/webhooks/3fa85f64-5717-4562-b3fc-2c963f66afa6/deliveries?page=1&pageSize=100`.

Resposta `200 OK` (últimas 100 entregas, [09:34] Marcos):
```json
{
  "data": [
    {
      "eventId": "b3c1e2d4-...",
      "eventType": "order.status_changed",
      "success": true,
      "responseStatus": 200,
      "durationMs": 184,
      "attemptedAt": "2026-07-05T13:05:02.000Z"
    },
    {
      "eventId": "a1b2c3d4-...",
      "eventType": "order.status_changed",
      "success": false,
      "responseStatus": 503,
      "durationMs": 10000,
      "attemptedAt": "2026-07-05T12:50:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 2, "totalPages": 1 }
}
```
Erros possíveis: `404 WEBHOOK_NOT_FOUND`.

### 7. `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — reprocessar evento morto (role `ADMIN`)

Requisição: sem corpo — `POST /api/v1/admin/webhooks/dead-letter/d1e2f3a4-b5c6-7d8e-9f0a-1b2c3d4e5f6a/replay`, exige `authenticate` + `requireRole('ADMIN')` ([09:35]-[09:36] Larissa/Sofia). Resposta `202 Accepted`:
```json
{
  "outboxId": "9c8b7a6f-5e4d-3c2b-1a0f-9e8d7c6b5a4f",
  "originalDeadLetterId": "d1e2f3a4-b5c6-7d8e-9f0a-1b2c3d4e5f6a",
  "status": "PENDING"
}
```
Erros possíveis: `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`, `403 FORBIDDEN` (role diferente de `ADMIN`, herdado do middleware `requireRole` já existente).

## Matriz de erros previstos (prefixo `WEBHOOK_`)

| Código | HTTP Status | Quando ocorre |
|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | Endpoint de webhook (`webhook_endpoint`) inexistente para o `id` informado |
| `WEBHOOK_INVALID_URL` | 400 | URL cadastrada/atualizada não usa esquema `https://` |
| `WEBHOOK_EVENT_TYPES_EMPTY` | 400 | Lista `eventTypes` vazia no cadastro ou edição |
| `WEBHOOK_SECRET_REQUIRED` | 500 (interno, não exposto ao chamador HTTP) | Invariante violada: worker encontra `webhook_endpoint` sem `secret` no momento do envio — falha defensiva, evento vai para retry/DLQ |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Payload do evento renderizado excede 64KB ([09:23]-[09:24] Sofia/Diego) — evento não é inserido na outbox, erro registrado no log da chamada de `changeStatus` |
| `WEBHOOK_DELIVERY_TIMEOUT` | — (código interno, gravado em `webhook_delivery.responseBodySnippet`/`webhook_outbox.lastError`) | Chamada HTTP ao endpoint do cliente excede 10 segundos ([09:42] Diego) |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `id` de dead-letter inexistente no replay administrativo |
| `WEBHOOK_SECRET_ROTATION_ALREADY_IN_PROGRESS` | 409 | Nova rotação solicitada enquanto a `previousSecret` do ciclo anterior ainda está no grace period de 24h |
| `WEBHOOK_INACTIVE_ENDPOINT` | 409 | Tentativa de rotacionar secret ou consultar deliveries de um endpoint com `active: false` |

A maioria herda diretamente de `ConflictError`, `UnprocessableEntityError` ou `BadRequestError` (`src/shared/errors/http-errors.ts`), que já aceitam um código de erro customizado via parâmetro, seguindo exatamente o padrão de `InsufficientStockError`/`InvalidStatusTransitionError` (que estendem essas classes, não `AppError` diretamente).

**Exceção a registrar:** `WEBHOOK_NOT_FOUND` e `WEBHOOK_DEAD_LETTER_NOT_FOUND` não podem reaproveitar `NotFoundError` como está — diferente de `ConflictError`/`UnprocessableEntityError`, a classe `NotFoundError` hoje fixa o código em `'NOT_FOUND'` (`constructor(resource = 'Resource') { super(\`${resource} not found\`, 404, 'NOT_FOUND'); }`, `src/shared/errors/http-errors.ts:27-31`) e não aceita um código customizado. Para emitir os códigos específicos exigidos pela matriz acima, o módulo de webhooks precisa de uma nova classe (`WebhookNotFoundError`, `WebhookDeadLetterNotFoundError`) estendendo `AppError` diretamente com `statusCode: 404` e o código específico — o mesmo princípio de `InvalidStatusTransitionError`, só que a partir de `AppError` em vez de `NotFoundError`. Isso é uma classe nova dentro do próprio módulo novo (`src/modules/webhooks/`), não uma alteração em `src/shared/errors/http-errors.ts` nem em `NotFoundError` em si — e nenhuma mudança é necessária em `src/middlewares/error.middleware.ts`, que já serializa qualquer `AppError` pelo `errorCode`/`statusCode` que ela carregar.

## Estratégias de resiliência

- **Timeout de entrega:** 10 segundos por chamada HTTP ao endpoint do cliente; excedido o timeout, o worker trata como falha e segue o fluxo de retry ([09:42] Diego/Sofia).
- **Retry com backoff exponencial:** 5 tentativas, intervalos 1m/5m/30m/2h/12h ([09:15]-[09:17] Diego).
- **Fallback para dead-letter:** após esgotar as tentativas, o evento não é descartado — vai para `webhook_dead_letter`, com reprocessamento manual disponível via endpoint administrativo ([09:18] Diego).
- **Isolamento de falhas por endpoint:** a falha de entrega para um `webhook_endpoint` não afeta o processamento de eventos de outros endpoints/clientes — cada linha da outbox é processada e avaliada independentemente.
- **Não bloqueio do caminho crítico:** a inserção na outbox é uma escrita local (mesma transação SQL), não uma chamada de rede; a indisponibilidade de um cliente de webhook nunca impacta a latência de `changeStatus` ([09:04] Bruno).
- **Limite de tamanho de payload:** eventos que ultrapassam 64KB não são enviados (erro, sem truncamento) ([09:23]-[09:24] Sofia/Diego).

## Observabilidade

- **Métricas:** contadores `webhook_delivery_success_total` e `webhook_delivery_failure_total` (com label `event_type`), gauge `webhook_outbox_pending` (tamanho da fila de eventos pendentes), contador `webhook_dead_letter_total`, histograma `webhook_delivery_duration_ms`.
- **Logs estruturados (Pino, `src/shared/logger/index.ts`):** eventos-chave logados no padrão já usado no projeto (objeto de contexto + mensagem curta em `snake_case`): `webhook_event_enqueued`, `webhook_delivery_attempted`, `webhook_delivery_succeeded`, `webhook_delivery_failed`, `webhook_retry_scheduled`, `webhook_moved_to_dead_letter`, `webhook_dead_letter_replayed` (incluindo `adminUserId` para auditoria, [09:36] Sofia).
- **Tracing/correlação:** o projeto não possui infraestrutura de tracing distribuído; a correlação entre a mudança de status, o evento na outbox e as tentativas de entrega é feita via `event_id` (mesmo valor usado no header `X-Event-Id`), presente em todos os logs relacionados ao ciclo de vida do evento.

## Dependências e compatibilidade

- Depende do MySQL e do `PrismaClient` já usados pelo projeto — nenhuma nova infraestrutura de dados.
- Depende de `authenticate`/`requireRole` (`src/middlewares/auth.middleware.ts`) e do middleware `validate` (`src/middlewares/validate.middleware.ts`) já existentes.
- Introduz um novo processo de runtime (`src/worker.ts`), com seu próprio script `npm run worker`, que precisa estar rodando em paralelo à API para que eventos sejam efetivamente entregues — sem o worker ativo, eventos se acumulam em `PENDING` na outbox sem erro aparente para o usuário da API.
- Compatível com o schema de autenticação atual (JWT com `role: ADMIN | OPERATOR`); nenhuma mudança na estrutura do JWT é necessária.

## Critérios de aceite técnicos

- Uma mudança de status que dispararia um webhook, mas cuja inserção na outbox falha, resulta em rollback completo da transação (pedido não muda de status).
- Um evento inserido na outbox é entregue ao endpoint correto em até 10 segundos no cenário de cliente saudável (2s de polling + tempo de resposta).
- Um endpoint que falha 5 vezes é movido para `webhook_dead_letter` e some da tabela `webhook_outbox` ativa.
- Um replay administrativo de dead-letter só é aceito para usuários com `role: ADMIN` e é auditado (log com `adminUserId`).
- A assinatura `X-Signature` gerada pelo worker é verificável pelo cliente usando a `secret` retornada na criação/rotação, com HMAC-SHA256 sobre o corpo exato enviado.
- Durante o grace period de 24h após rotação, requisições assinadas tanto com a secret nova quanto com a antiga são consideradas válidas pelo cliente (o sistema aceita ambas na verificação, se aplicável a endpoints que também recebem callbacks — nesta feature, unidirecional, a garantia relevante é o worker sempre assinar com a secret mais recente).

## Riscos e mitigação

| Risco | Mitigação |
|---|---|
| Falha na inserção da outbox derruba a transação inteira de mudança de status | Testes de integração cobrindo o caminho de falha, seguindo o padrão já usado em `tests/orders.test.ts` para atomicidade de estoque |
| Crescimento não controlado de `webhook_outbox` | Índices em `status`/`createdAt`; arquivamento após 30 dias fica como item de operação futura, fora desta entrega |
| Vazamento de secret | Secret por endpoint (não global), rotação com grace period, revisão de segurança dedicada antes do deploy ([09:46] Sofia) |
| Worker parado silenciosamente (processo caído) sem alarme | Métrica `webhook_outbox_pending` como sinal indireto; alarme operacional fica fora do escopo desta feature, mas a métrica é pré-requisito para ele |

## Integração com o sistema existente

- **`src/modules/orders/order.service.ts`** (método `changeStatus`): estendido para, dentro da mesma `this.prisma.$transaction(async (tx) => {...})` já existente, chamar `publishWebhookEvent(tx, order, fromStatus, toStatus)` após a validação de transição e antes do commit — reaproveitando o mesmo `tx: Prisma.TransactionClient` já usado para `tx.order.update` e `tx.orderStatusHistory.create`.
- **`src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts`**: as novas classes de erro do módulo de webhooks estendem `ConflictError`/`UnprocessableEntityError`/`BadRequestError` quando o código precisa ser customizado (mesmo padrão de `InsufficientStockError`/`InvalidStatusTransitionError`); `WebhookNotFoundError` e `WebhookDeadLetterNotFoundError` são exceção e estendem `AppError` diretamente, já que `NotFoundError` não aceita código customizado (ver [Matriz de erros](#matriz-de-erros-previstos-prefixo-webhook_)). Nenhum arquivo existente em `src/shared/errors/` precisa ser modificado.
- **`src/middlewares/error.middleware.ts`**: nenhuma alteração de código é necessária — por herdarem de `AppError`, os erros `WEBHOOK_*` já são serializados corretamente pelo handler centralizado existente (`{ error: { code, message, details } }`).
- **`src/middlewares/auth.middleware.ts`**: `authenticate` protege todas as rotas de `src/modules/webhooks/webhook.routes.ts`; `requireRole('ADMIN')` protege especificamente `POST /admin/webhooks/dead-letter/:id/replay`, seguindo o mesmo padrão hoje aplicado em `src/modules/users/user.routes.ts` (`GET /:id`).
- **`src/shared/logger/index.ts`**: `webhook.service.ts` e `webhook.worker.ts` importam a instância singleton `logger` já configurada (redaction, formato JSON em produção), sem criar uma nova instância de Pino.
- **`src/app.ts`**: `buildControllers` ganha a construção de `WebhookRepository`/`WebhookService`/`WebhookController` (mesmo padrão de injeção manual via construtor já usado para `orders`/`products`), e `buildApiRouter` passa a montar `buildWebhookRouter(controllers.webhooks)` sob `/api/v1/webhooks` e `/api/v1/admin/webhooks`.
- **`src/server.ts`**: serve de modelo direto para o novo `src/worker.ts` — mesmo padrão de bootstrap assíncrono, shutdown gracioso em `SIGINT`/`SIGTERM` (aqui, parando o loop de polling e chamando `prisma.$disconnect()`), e log de `worker_started`/`worker_shutdown_initiated` via `logger.fatal`/`logger.info` no mesmo estilo de `server_started`/`shutdown_initiated`.
- **`prisma/schema.prisma`**: adição dos modelos `WebhookEndpoint`, `WebhookOutbox`, `WebhookDelivery`, `WebhookDeadLetter`, seguindo as convenções já usadas (`id String @id @default(uuid()) @db.Char(36)`, `@@map` para nomes de tabela em `snake_case`, índices em colunas de filtro frequente).
- **`src/middlewares/validate.middleware.ts`**: `webhook.schemas.ts` usa o mesmo helper `validate({ body, params, query })` já usado em `order.routes.ts`/`product.routes.ts`, incluindo `z.string().uuid()` para IDs e `z.coerce.number()` para paginação.
