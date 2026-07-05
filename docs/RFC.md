# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
|---|---|
| Autor | Larissa (Tech Lead) |
| Status | Em revisão |
| Data | Quinta-feira, reunião de definição técnica (09:00–09:53) |
| Revisores | Marcos (PM), Bruno (Eng. Pleno, time de Pedidos), Diego (Eng. Sênior, time de Plataforma), Sofia (Eng. de Segurança) |
| Documentos relacionados | [PRD](PRD.md) · [FDD](FDD.md) · [ADRs](adrs/) |

## Resumo executivo (TL;DR)

Vamos notificar clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) em tempo quase real quando o status de um pedido mudar, via webhooks HTTP assinados. A mudança de status já ocorre em transação SQL no `OrderService`; propomos gravar o evento numa tabela `webhook_outbox` dentro dessa mesma transação (padrão Transactional Outbox) e processá-lo de forma assíncrona por um worker dedicado, com retry, dead-letter queue e autenticação HMAC por endpoint. Não há mudança de comportamento perceptível para o fluxo síncrono de pedidos hoje existente.

## Contexto e problema

Três clientes B2B fazem hoje polling manual em `GET /orders` para descobrir mudanças de status, o que é lento, caro para eles e já gerou risco comercial explícito (ameaça de migração para concorrente por parte de um dos clientes caso a entrega não saia até o fim do trimestre). Eles aceitam qualquer latência abaixo de 10 segundos como "tempo real" — não há requisito de latência sub-segundo.

O sistema hoje não tem nenhum mecanismo de notificação externa, eventos ou filas — este vácuo é o que a feature preenche. A restrição mais importante levantada tecnicamente é que a mudança de status já roda dentro de uma transação sensível (atualiza `orders`, insere em `order_status_history`, decrementa `stock_quantity`); qualquer solução não pode arriscar travar ou tornar inconsistente essa transação por causa de um cliente de webhook lento ou indisponível.

## Proposta técnica

A abordagem combina três elementos, cada um com sua decisão registrada em ADR dedicado:

1. **Captura do evento via Transactional Outbox.** Dentro da mesma transação que já muda o status do pedido, uma linha é inserida em `webhook_outbox` com o payload do evento já renderizado (snapshot do estado no momento da mudança). Isso garante, pelas próprias garantias ACID do MySQL já utilizado pelo projeto, que o evento só existe se a mudança de status foi de fato commitada — sem exigir infraestrutura de mensageria adicional. Ver [ADR-001](adrs/ADR-001-outbox-pattern-mysql.md).

2. **Processamento assíncrono por worker dedicado.** Um novo processo Node.js (`src/worker.ts`), independente da API HTTP, faz polling da tabela outbox a cada 2 segundos, envia as chamadas HTTP aos endpoints cadastrados pelos clientes, e trata retry com backoff exponencial (5 tentativas, 1m/5m/30m/2h/12h) até mover o evento para uma tabela `webhook_dead_letter` em caso de falha definitiva, reprocessável manualmente por um administrador. Ver [ADR-002](adrs/ADR-002-worker-processo-separado-polling.md) e [ADR-003](adrs/ADR-003-retry-backoff-dead-letter-queue.md).

3. **Segurança e semântica de entrega.** Cada requisição de webhook é assinada com HMAC-SHA256 usando uma secret exclusiva por endpoint cadastrado (rotacionável, com grace period de 24h), e cada evento carrega um `X-Event-Id` único que permite ao cliente deduplicar entregas — o sistema garante at-least-once, não exactly-once. Ver [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) e [ADR-005](adrs/ADR-005-at-least-once-x-event-id.md).

A feature é implementada como um novo módulo `src/modules/webhooks/`, seguindo exatamente a estrutura de módulo, tratamento de erros, logging e autenticação já usados pelo restante do projeto — sem introduzir nenhuma convenção nova. Ver [ADR-006](adrs/ADR-006-reuso-padroes-existentes.md). O detalhamento de contratos HTTP, matriz de erros e fluxos passo a passo está no [FDD](FDD.md), não neste documento.

## Alternativas consideradas

**Disparo síncrono da chamada HTTP dentro da própria transação de `changeStatus`.** Foi a primeira abordagem discutida. Descartada porque a transação de mudança de status já é pesada (atualiza `orders`, `order_status_history` e estoque) e um cliente de webhook lento ou indisponível travaria a mudança de status de outros pedidos; além disso, não há como fazer rollback de uma chamada HTTP já enviada caso a transação falhe depois.

**Fila de mensageria dedicada (ex.: Redis Streams) em vez de outbox no MySQL existente.** Resolveria o desacoplamento de forma mais "clássica", mas exigiria subir, operar e monitorar um componente de infraestrutura adicional para um time pequeno que hoje só mantém MySQL. Descartada por ser desproporcional ao volume e à urgência do problema — outbox no MySQL já existente entrega a mesma garantia de consistência sem custo operacional novo.

## Questões em aberto

- **Rate limiting de envio por cliente.** Se um cliente tiver, por exemplo, 50 pedidos mudando de status no mesmo minuto, o sistema hoje enviaria as 50 chamadas sem limitação. Ficou definido observar o comportamento em produção e decidir depois se um limitador é necessário — não faz parte do escopo desta entrega.
- **Escalonamento para múltiplos workers.** A garantia de ordenação de entrega por `order_id` hoje depende de haver um único worker consumindo a outbox em ordem de `created_at`. Se no futuro for necessário paralelizar o processamento, será preciso decidir entre particionamento por `order_id` ou lock pessimista — matéria deixada para quando (e se) a necessidade de escala aparecer.
- **Nível de autorização do CRUD de configuração de webhook.** Hoje qualquer usuário autenticado (qualquer role) pode cadastrar/editar/remover endpoints de webhook de um cliente; apenas o replay de dead-letter exige role `ADMIN`. Foi sinalizado que essa permissão pode ser "endurecida" mais adiante, sem que isso tenha sido decidido nesta reunião.

## Impacto e riscos

- **Impacto na transação de `changeStatus`:** a inserção na outbox passa a fazer parte da mesma transação SQL — uma falha na inserção do evento deve causar rollback de toda a mudança de status, para não haver mudança de status sem notificação correspondente. Isso é impacto direto sobre um caminho crítico já existente e exige testes de regressão cuidadosos.
- **Risco de vazamento de secret:** já houve um incidente prévio de cliente vazando secret em log de aplicação. Mitigado por secret por endpoint (não global) e por rotação com grace period, mas exige revisão de segurança dedicada antes do deploy (ver seção de riscos do [PRD](PRD.md)).
- **Risco de crescimento descontrolado da tabela outbox:** sem rotina de arquivamento (fora de escopo desta feature, mas necessária operacionalmente), o volume de eventos entregues pode crescer indefinidamente. Mitigado por índices em `status`/`created_at` e pela expectativa de arquivamento futuro após 30 dias.
- **Impacto de prazo:** a Tech Lead estimou 3 sprints, incluindo uma janela reservada de pelo menos 2 dias úteis para revisão de segurança dedicada (HMAC e geração de secret) antes do deploy em produção.

## Decisões relacionadas

- [ADR-001 — Padrão Outbox no MySQL](adrs/ADR-001-outbox-pattern-mysql.md)
- [ADR-002 — Worker em processo separado com polling](adrs/ADR-002-worker-processo-separado-polling.md)
- [ADR-003 — Retry com backoff exponencial e Dead Letter Queue](adrs/ADR-003-retry-backoff-dead-letter-queue.md)
- [ADR-004 — Autenticação HMAC-SHA256 com secret por endpoint](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)
- [ADR-005 — Garantia at-least-once com X-Event-Id](adrs/ADR-005-at-least-once-x-event-id.md)
- [ADR-006 — Reuso máximo dos padrões existentes do projeto](adrs/ADR-006-reuso-padroes-existentes.md)
