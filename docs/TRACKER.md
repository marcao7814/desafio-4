# Tracker de Rastreabilidade

Mapeia cada item registrado em [PRD](PRD.md), [RFC](RFC.md), [FDD](FDD.md) e [ADRs](adrs/) à sua origem — na transcrição da reunião técnica (`TRANSCRICAO.md`) ou no código-fonte existente.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro de webhook (POST) com url, customerId, eventTypes; secret gerada pelo sistema | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Edição (PATCH) de webhook cadastrado | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Remoção (DELETE) de webhook cadastrado | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Listagem (GET) dos webhooks de um cliente | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro de eventos por lista de status desejados por webhook | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Histórico de entregas via GET /webhooks/:id/deliveries | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Endpoint administrativo de replay de dead-letter restrito a role ADMIN | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Rotação de secret via API com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Consistência transacional entre mudança de status e criação do evento | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | X-Event-Id único por evento para deduplicação do cliente | TRANSCRICAO | [09:25] Diego |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Assinatura HMAC-SHA256 do payload | TRANSCRICAO | [09:20] Sofia |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Preservação de eventos definitivamente falhos em dead-letter | TRANSCRICAO | [09:18] Diego |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência de entrega abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 segundos por chamada HTTP de entrega | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | TLS obrigatório (URLs devem ser https) | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Limite de 64KB de payload, erro sem truncamento | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Garantia de entrega at-least-once | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Auditoria (log de usuário) em ações de replay administrativo | TRANSCRICAO | [09:36] Sofia |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Reuso dos padrões existentes de erro, logging e autenticação | TRANSCRICAO | [09:30] Larissa |
| PRD-OBJ-01 | docs/PRD.md | Requisito Não Funcional | Meta quantitativa: latência de entrega < 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Decisão | Objetivo: eliminar dependência de polling manual dos clientes B2B | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-03 | docs/PRD.md | Decisão | Meta de prazo: entrega em até 3 sprints | TRANSCRICAO | [09:47] Larissa |
| PRD-SCOPE-OUT-01 | docs/PRD.md | Restrição | Fora de escopo: notificação por e-mail em falhas repetidas | TRANSCRICAO | [09:37] Larissa |
| PRD-SCOPE-OUT-02 | docs/PRD.md | Restrição | Fora de escopo: rate limiting de envio por cliente | TRANSCRICAO | [09:39] Larissa |
| PRD-SCOPE-OUT-03 | docs/PRD.md | Restrição | Fora de escopo: dashboard visual para o cliente | TRANSCRICAO | [09:40] Larissa |
| PRD-SCOPE-OUT-04 | docs/PRD.md | Restrição | Fora de escopo: webhooks inbound (só outbound) | TRANSCRICAO | [09:02] Marcos |
| PRD-SCOPE-OUT-05 | docs/PRD.md | Restrição | Fora de escopo: arquivamento/retenção de eventos entregues | TRANSCRICAO | [09:08] Diego |
| PRD-RISK-01 | docs/PRD.md | Trade-off | Risco de vazamento de secret (incidente prévio já ocorrido) | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-02 | docs/PRD.md | Trade-off | Risco de indisponibilidade prolongada de cliente (~2h já observado) | TRANSCRICAO | [09:16] Diego |
| PRD-RISK-03 | docs/PRD.md | Trade-off | Risco de atraso por dependência da revisão de segurança | TRANSCRICAO | [09:46] Sofia |
| PRD-RISK-04 | docs/PRD.md | Trade-off | Risco de crescimento não controlado da tabela outbox | TRANSCRICAO | [09:07] Bruno |
| RFC-ALT-01 | docs/RFC.md | Trade-off | Alternativa descartada: disparo síncrono na transação de changeStatus | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Alternativa descartada: fila dedicada (Redis Streams) | TRANSCRICAO | [09:07] Larissa |
| RFC-OPEN-01 | docs/RFC.md | Restrição | Questão em aberto: rate limiting de envio por cliente | TRANSCRICAO | [09:38] Diego |
| RFC-OPEN-02 | docs/RFC.md | Restrição | Questão em aberto: escalonamento para múltiplos workers e ordering | TRANSCRICAO | [09:13] Bruno |
| RFC-OPEN-03 | docs/RFC.md | Restrição | Questão em aberto: nível de autorização do CRUD de configuração | TRANSCRICAO | [09:37] Sofia |
| FDD-CONTRATO-01 | docs/FDD.md | Requisito Funcional | Contrato: POST /api/v1/webhooks | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Requisito Funcional | Contrato: GET /api/v1/webhooks | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Requisito Funcional | Contrato: PATCH /api/v1/webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Requisito Funcional | Contrato: DELETE /api/v1/webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Requisito Funcional | Contrato: POST /api/v1/webhooks/:id/rotate-secret | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Requisito Funcional | Contrato: GET /api/v1/webhooks/:id/deliveries | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Requisito Funcional | Contrato: POST /api/v1/admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:18] Diego |
| FDD-ERR-01 | docs/FDD.md | Restrição | Código de erro WEBHOOK_NOT_FOUND | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Restrição | Código de erro WEBHOOK_INVALID_URL | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-03 | docs/FDD.md | Restrição | Código de erro WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-04 | docs/FDD.md | Restrição | Convenção de prefixo WEBHOOK_ para todos os códigos de erro do módulo | TRANSCRICAO | [09:29] Larissa |
| FDD-FLOW-01 | docs/FDD.md | Decisão | Payload do evento é snapshot renderizado no momento da inserção na outbox | TRANSCRICAO | [09:52] Larissa |
| FDD-FLOW-02 | docs/FDD.md | Decisão | Worker processa a outbox em polling a cada 2 segundos | TRANSCRICAO | [09:09] Diego |
| FDD-FLOW-03 | docs/FDD.md | Decisão | Progressão de backoff: 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| FDD-FLOW-04 | docs/FDD.md | Decisão | Evento esgotado move para webhook_dead_letter, replay manual disponível | TRANSCRICAO | [09:18] Diego |
| FDD-INTEG-01 | docs/FDD.md | Restrição | Extensão do método changeStatus para publicar evento na mesma transação | CODIGO | src/modules/orders/order.service.ts |
| FDD-INTEG-02 | docs/FDD.md | Restrição | Reuso de AppError e subclasses HTTP para erros WEBHOOK_* | CODIGO | src/shared/errors/app-error.ts |
| FDD-INTEG-03 | docs/FDD.md | Restrição | Nenhuma alteração necessária no middleware de erro centralizado | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INTEG-04 | docs/FDD.md | Restrição | Reuso de authenticate/requireRole para proteger rotas do módulo | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INTEG-05 | docs/FDD.md | Restrição | Reuso da instância singleton do logger Pino | CODIGO | src/shared/logger/index.ts |
| FDD-INTEG-06 | docs/FDD.md | Restrição | Registro do novo router de webhooks na composição da aplicação | CODIGO | src/app.ts |
| FDD-INTEG-07 | docs/FDD.md | Restrição | src/server.ts como modelo de bootstrap para o novo src/worker.ts | CODIGO | src/server.ts |
| FDD-INTEG-08 | docs/FDD.md | Restrição | Novas tabelas seguindo convenções de uuid e @@map do schema Prisma | CODIGO | prisma/schema.prisma |
| FDD-INTEG-09 | docs/FDD.md | Restrição | Reuso do middleware validate() para os schemas Zod do módulo | CODIGO | src/middlewares/validate.middleware.ts |
| ADR-001 | docs/adrs/ADR-001-outbox-pattern-mysql.md | Decisão | Padrão Outbox no MySQL para captura consistente de eventos | TRANSCRICAO | [09:06] Diego |
| ADR-002 | docs/adrs/ADR-002-worker-processo-separado-polling.md | Decisão | Worker em processo separado, polling de 2 segundos | TRANSCRICAO | [09:11] Diego |
| ADR-003 | docs/adrs/ADR-003-retry-backoff-dead-letter-queue.md | Decisão | Retry com backoff exponencial (5 tentativas) e DLQ em tabela dedicada | TRANSCRICAO | [09:15] Diego |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256 com secret exclusiva por endpoint | TRANSCRICAO | [09:20] Sofia |
| ADR-005 | docs/adrs/ADR-005-at-least-once-x-event-id.md | Decisão | Garantia at-least-once com deduplicação via X-Event-Id | TRANSCRICAO | [09:24] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | Reuso máximo dos padrões arquiteturais existentes do projeto | CODIGO | src/modules/orders/order.service.ts |

## Cobertura

- Total de itens rastreados: 66.
- Itens com Fonte = TRANSCRICAO: 56 (≈85%).
- Itens com Fonte = CODIGO: 10.
- Decisões técnicas secundárias (formato exato de payload, headers adicionais, timeouts internos de rotação, códigos de erro auxiliares não citados literalmente na reunião) foram deliberadamente **deixadas fora deste tracker**: por não terem uma citação direta e inequívoca na transcrição ou um arquivo de código já existente, incluí-las aqui seria mascarar uma inferência de engenharia como se fosse uma decisão registrada — o próprio critério que este tracker existe para proteger. Essas decisões estão descritas no [FDD](FDD.md) como detalhamento de implementação, não como fatos rastreáveis.
