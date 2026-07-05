# ADR-006: Reuso máximo dos padrões arquiteturais existentes do projeto

## Status

Aceito

## Contexto

O módulo de webhooks é uma feature nova dentro de uma base de código já estabelecida, com convenções consistentes de estrutura de módulo, tratamento de erros, logging e autenticação. A introdução de uma feature nova é uma oportunidade recorrente para "reinventar a roda" com uma abordagem própria (nova convenção de erros, novo logger, nova estrutura de pastas), o que fragmentaria a base de código e aumentaria o custo de manutenção a longo prazo.

## Decisão

O módulo de webhooks segue estritamente os padrões já estabelecidos no projeto, sem introduzir novas convenções:

- **Estrutura de módulo**: nova pasta `src/modules/webhooks/` com `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`, `webhook.schemas.ts`, no mesmo padrão de `src/modules/orders/` e `src/modules/products/`. A lógica de processamento assíncrono fica em `webhook.worker.ts` (ou `webhook.processor.ts`) dentro do mesmo módulo, com um novo entry point `src/worker.ts` análogo a `src/server.ts`.
- **Erros**: reaproveita a classe base `AppError` (`src/shared/errors/app-error.ts`) e as subclasses HTTP genéricas que aceitam código customizado (`ConflictError`, `UnprocessableEntityError`, `BadRequestError` em `src/shared/errors/http-errors.ts`), seguindo o mesmo padrão de `InsufficientStockError` e `InvalidStatusTransitionError` (`src/shared/errors/`). Novos códigos de erro usam o prefixo `WEBHOOK_` (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`), no mesmo estilo `SCREAMING_SNAKE_CASE` já usado em `INSUFFICIENT_STOCK` e `INVALID_STATUS_TRANSITION`. Exceção: `NotFoundError` hoje fixa o código em `'NOT_FOUND'` e não aceita customização (diferente das demais), então `WEBHOOK_NOT_FOUND` exige uma classe nova estendendo `AppError` diretamente — não uma modificação de `NotFoundError` (ver detalhe em [FDD.md § Matriz de erros](../FDD.md#matriz-de-erros-previstos-prefixo-webhook_)).
- **Middleware de erro**: nenhuma mudança necessária em `src/middlewares/error.middleware.ts` — por herdar de `AppError`, os novos erros do módulo webhook já são tratados automaticamente pelo handler centralizado existente.
- **Logging**: reaproveita a instância singleton de Pino (`src/shared/logger/index.ts`), seguindo o mesmo padrão de mensagem estruturada (objeto de contexto + mensagem curta em `snake_case`, ex.: `webhook_delivery_failed`, `webhook_retry_scheduled`).
- **Validação**: schemas Zod em `webhook.schemas.ts`, aplicados via o middleware genérico `validate()` (`src/middlewares/validate.middleware.ts`), no mesmo padrão de `order.schemas.ts` e `product.schemas.ts`.
- **Autenticação/autorização**: reaproveita `authenticate` e `requireRole('ADMIN')` (`src/middlewares/auth.middleware.ts`), no mesmo padrão hoje usado em `src/modules/users/user.routes.ts`, para proteger o endpoint de replay de dead-letter.
- **Persistência**: reaproveita o mesmo `PrismaClient`/`DATABASE_URL`, com IDs `uuid()` como em todas as demais entidades do `prisma/schema.prisma`, e a mesma convenção de tabelas em `snake_case` via `@@map`.

## Alternativas Consideradas

**Criar convenções próprias para o módulo de webhooks** (ex.: uma biblioteca de erros específica, um logger dedicado, ou uma estrutura de pastas diferente por ser um módulo "mais assíncrono" que os demais). Foi considerada implicitamente e descartada: nenhum requisito da feature justifica divergir dos padrões já validados em produção pelos demais módulos, e a divergência aumentaria o custo cognitivo para qualquer desenvolvedor transitar entre módulos do sistema.

## Consequências

**Positivas**
- Curva de aprendizado mínima para qualquer desenvolvedor da equipe já familiarizado com `orders`, `products` ou `customers` — o módulo de webhooks "parece" com o resto do sistema.
- O middleware de erro centralizado, o logger e o middleware de autenticação não precisam de nenhuma alteração para suportar a nova feature.
- Reduz superfície de revisão de segurança: a revisora de segurança já conhece o comportamento de `AppError`, `authenticate` e `requireRole`, podendo focar a revisão nos pontos realmente novos (HMAC, geração de secret).

**Negativas**
- Herda eventuais limitações dos padrões existentes (ex.: testes hoje são majoritariamente de integração via HTTP contra banco real, sem suíte unitária isolada — ver `tests/orders.test.ts`); o módulo de webhooks tende a repetir essa mesma limitação em vez de introduzir uma estratégia de teste diferente.
- Acopla a evolução do módulo de webhooks à evolução dos padrões compartilhados (`AppError`, logger, middleware de auth): uma mudança futura nesses componentes compartilhados impacta também o módulo de webhooks.
- O reuso não é 100% uniforme entre as subclasses de erro: `NotFoundError` não aceita código customizado (diferente de `ConflictError`/`UnprocessableEntityError`), então o módulo precisa de uma classe nova para os casos 404 em vez de reaproveitar `NotFoundError` diretamente — uma limitação pré-existente do padrão de erros, não algo introduzido por esta feature.
