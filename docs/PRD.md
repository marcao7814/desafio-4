# PRD — Product Requirements Document: Sistema de Webhooks de Notificação de Pedidos

## Resumo e contexto da feature

O Order Management System (OMS) hoje não possui nenhum mecanismo de notificação externa, eventos ou filas. Esta feature introduz um sistema de **webhooks outbound**: quando o status de um pedido muda na plataforma, clientes B2B cadastrados são notificados via HTTP, de forma assíncrona, autenticada e com garantias de entrega. A decisão técnica de como construir foi fechada em reunião entre Tech Lead, PM, engenharia e segurança; este documento consolida o "porquê" e o "o quê" da feature, complementando a proposta técnica no [RFC](RFC.md) e o detalhamento de implementação no [FDD](FDD.md).

## Problema e motivação

Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — fazem hoje polling manual em `GET /orders` para descobrir quando o status de seus pedidos muda ([09:00] Marcos). Isso torna a integração deles lenta e cara, e já gerou risco comercial concreto: a Atlas Comercial sinalizou que pode migrar para um concorrente caso a notificação em tempo real não seja entregue até o fim do trimestre ([09:00] Marcos). Os clientes consideram qualquer latência abaixo de 10 segundos como "tempo real" — não exigem entrega instantânea, apenas que parem de precisar consultar manualmente ([09:02] Marcos).

## Público-alvo e cenários de uso

- **Clientes B2B integradores** (Atlas Comercial, MaxDistribuição, Nova Cargo, e futuros clientes do mesmo perfil): consomem a API para cadastrar endpoints de webhook e recebem notificações HTTP quando o status de seus pedidos muda.
- **Cenário principal:** um pedido do cliente muda de `PROCESSING` para `SHIPPED`; o cliente, que cadastrou um webhook para esse tipo de evento, recebe uma chamada HTTP assinada em até 10 segundos, sem precisar consultar a API ativamente.
- **Operadores internos (usuários autenticados do OMS):** cadastram e gerenciam a configuração de webhooks em nome dos clientes, via API autenticada (não é o cliente final quem acessa a API diretamente com suas próprias credenciais — é feito por um usuário do sistema em nome do cliente, [09:31]-[09:32] Marcos/Bruno/Larissa).
- **Administradores (`role: ADMIN`):** atuam em cenários de exceção, reprocessando manualmente eventos que falharam definitivamente (dead-letter).

## Objetivos e métricas de sucesso

- **Latência de entrega:** notificar mudanças de status em até 10 segundos no pior caso, medida como o tempo entre o commit da mudança de status e a entrega bem-sucedida ao endpoint do cliente — meta que nasce diretamente do critério informado pelos clientes B2B ([09:02] Marcos: "qualquer coisa abaixo de 10 segundos já é tempo real").
- **Eliminar a dependência de polling manual** pelos três clientes B2B piloto (Atlas Comercial, MaxDistribuição, Nova Cargo) como via primária de descoberta de mudança de status, resolvendo o gatilho comercial que motivou a feature.
- **Prazo de entrega:** viabilizar o lançamento em até 3 sprints, incluindo uma janela dedicada de revisão de segurança de pelo menos 2 dias úteis antes do deploy em produção ([09:45]-[09:47] Larissa/Sofia), permitindo à Atlas Comercial confirmar prazo dentro do trimestre.

## Escopo

### Incluso
- CRUD de configuração de webhook por cliente (cadastro, edição, remoção, listagem), com filtro de quais status cada endpoint deseja receber.
- Entrega assíncrona de eventos via outbox transacional e worker dedicado, com retry e dead-letter queue.
- Autenticação HMAC-SHA256 dos payloads, com secret exclusiva por endpoint e suporte a rotação.
- Histórico de entregas por endpoint de webhook.
- Endpoint administrativo de reprocessamento manual de eventos em dead-letter.

### Fora de escopo
- **Notificação por e-mail ao cliente quando o webhook dele está falhando repetidamente** — proposto e explicitamente adiado para uma fase futura, após medição do impacto real ([09:37]-[09:38] Marcos/Larissa).
- **Rate limiting de envio de webhooks por cliente** — levantado como preocupação (ex.: 50 pedidos mudando de status em um minuto), mas decidido observar o comportamento em produção antes de implementar qualquer limitador ([09:38]-[09:39] Diego/Larissa).
- **Dashboard visual para o cliente acompanhar seus webhooks** — descartado para esta fase; painel visual seria um projeto separado do time de frontend ([09:39]-[09:40] Marcos/Larissa).
- **Webhooks inbound (clientes enviando dados para o OMS)** — descartado logo no início da reunião; o escopo é estritamente outbound, do OMS para o cliente ([09:02]-[09:03] Sofia/Marcos).
- **Arquivamento/retenção de eventos entregues** — mencionado como necessidade operacional futura (arquivar após ~30 dias), não faz parte desta entrega ([09:08] Diego).

## Requisitos funcionais

1. Cliente (via usuário operador autenticado) pode cadastrar um webhook informando `url`, `customerId` e a lista de status de pedido que deseja receber; a `secret` é gerada pelo sistema e devolvida apenas na criação ([09:31]-[09:32] Marcos/Bruno).
2. Sistema permite editar (`PATCH`) um webhook cadastrado, incluindo a URL, a lista de status e o estado ativo/inativo ([09:33] Bruno).
3. Sistema permite remover (`DELETE`) um webhook cadastrado ([09:33] Bruno).
4. Sistema permite listar (`GET`) os webhooks cadastrados de um cliente ([09:33] Bruno).
5. Cada webhook filtra por lista de status de interesse; o evento só é gerado se algum webhook do cliente estiver interessado naquele status específico ([09:33]-[09:34] Marcos/Bruno/Diego).
6. Cliente pode consultar o histórico das últimas entregas de um webhook (sucesso/falha, payload, resposta, tempo de resposta) via `GET /webhooks/:id/deliveries` ([09:34] Marcos).
7. Sistema permite reprocessamento manual de um evento em dead-letter via endpoint administrativo `POST /admin/webhooks/dead-letter/:id/replay`, restrito a usuários com `role: ADMIN` ([09:18]-[09:19] Diego, [09:35]-[09:36] Larissa/Sofia).
8. Sistema permite ao cliente rotacionar a secret de um webhook via API, mantendo a secret anterior válida por 24 horas em paralelo à nova ([09:21] Sofia).
9. Toda mudança de status de pedido elegível gera um evento de notificação de forma consistente com a transação que efetivou a mudança — nunca há mudança de status sem evento correspondente nem evento sem mudança real ([09:40]-[09:41] Bruno/Diego).
10. Cada evento entregue carrega um identificador único (`X-Event-Id`) que permite ao cliente deduplicar entregas repetidas, já que a garantia de entrega é at-least-once ([09:25] Diego).
11. Requisições de webhook são assinadas com HMAC-SHA256 sobre o corpo, permitindo ao cliente verificar autenticidade e integridade ([09:20] Sofia).
12. Eventos que falham em todas as tentativas de reenvio são preservados em uma tabela de dead-letter (não descartados), com o payload original, motivo da falha e timestamp ([09:18] Diego).

## Requisitos não funcionais

- **Latência de entrega:** até 10 segundos no pior caso, incluindo o intervalo de polling do worker (2 segundos) e o tempo de resposta do cliente ([09:02] Marcos, [09:09]-[09:10] Diego).
- **Timeout de chamada HTTP:** 10 segundos por tentativa de entrega; excedido, é tratado como falha ([09:42] Diego/Sofia).
- **TLS obrigatório:** URLs de webhook cadastradas devem usar `https://`; URLs `http://` são recusadas na validação ([09:23] Sofia).
- **Limite de tamanho de payload:** eventos que ultrapassam 64KB não são enviados — erro explícito, sem truncamento ([09:23]-[09:24] Sofia/Diego).
- **Garantia de entrega:** at-least-once (não exactly-once); duplicidade é responsabilidade de deduplicação do cliente via `X-Event-Id` ([09:24]-[09:26] Diego/Sofia).
- **Auditoria:** ações administrativas de replay de dead-letter devem registrar o usuário que executou a ação ([09:36] Sofia).
- **Consistência com o restante do sistema:** a feature deve reutilizar os padrões já estabelecidos de tratamento de erro, logging e autenticação, sem introduzir novas convenções ([09:29]-[09:30] Bruno/Diego/Larissa).

## Decisões e trade-offs principais

As decisões arquiteturais centrais estão documentadas individualmente nos ADRs, com contexto, alternativas descartadas e consequências:

- Padrão Outbox no MySQL em vez de fila dedicada — [ADR-001](adrs/ADR-001-outbox-pattern-mysql.md).
- Worker em processo separado com polling de 2s em vez de reagir a triggers de banco ou rodar embutido na API — [ADR-002](adrs/ADR-002-worker-processo-separado-polling.md).
- Retry com backoff exponencial (5 tentativas) e DLQ em tabela dedicada — [ADR-003](adrs/ADR-003-retry-backoff-dead-letter-queue.md).
- HMAC-SHA256 com secret por endpoint (não secret global) — [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md).
- Garantia at-least-once com deduplicação via `X-Event-Id` em vez de exactly-once — [ADR-005](adrs/ADR-005-at-least-once-x-event-id.md).
- Reuso máximo dos padrões já existentes no projeto (erros, logging, autenticação, estrutura de módulo) — [ADR-006](adrs/ADR-006-reuso-padroes-existentes.md).

## Dependências

- Banco de dados MySQL e `PrismaClient` já operados pelo projeto (nenhuma infraestrutura de mensageria nova).
- Middleware de autenticação/autorização existente (`authenticate`, `requireRole`) para proteger endpoints de configuração e o endpoint administrativo de replay.
- Disponibilidade de um processo worker rodando continuamente em produção — sem ele, eventos se acumulam sem serem entregues.
- Revisão de segurança dedicada (pelo menos 2 dias úteis) antes do deploy, com foco em HMAC e geração/armazenamento de secret ([09:46] Sofia).
- Comunicação do time de produto com os clientes B2B para documentação de integração (deduplicação via `X-Event-Id`, verificação de assinatura) no portal de desenvolvedor ([09:26] Marcos).

## Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Vazamento de secret de um cliente (já ocorreu um caso de secret vazada em log de aplicação do cliente) | Média | Alto — compromete autenticidade das notificações daquele endpoint | Secret exclusiva por endpoint (não global), suporte a rotação com grace period de 24h, revisão de segurança dedicada antes do deploy |
| Cliente fica indisponível por período prolongado (já houve caso de ~2h de indisponibilidade em manutenção planejada) e esgota as tentativas de retry (~15h) | Baixa a média | Médio — cliente perde a notificação automática daquele evento específico | Retry com backoff estendido (até 12h na última tentativa) cobrindo janelas de manutenção comuns; evento preservado em dead-letter com replay manual disponível |
| Atraso na revisão de segurança compromete o prazo de 3 sprints prometido à Atlas Comercial | Baixa | Médio — risco comercial de a Atlas Comercial migrar para concorrente | Janela de revisão de segurança (2 dias úteis) já reservada dentro da estimativa de 3 sprints, não como item posterior |
| Crescimento não controlado da tabela `webhook_outbox` ao longo do tempo | Média (a médio/longo prazo) | Médio — degradação de performance de leitura do worker | Índices dedicados em `status`/`created_at`; arquivamento de eventos entregues após 30 dias identificado como necessidade futura (fora desta entrega, mas já planejado) |

## Critérios de aceitação

- Um cliente consegue cadastrar, editar, listar e remover webhooks via API autenticada.
- Uma mudança de status de pedido para a qual existe webhook cadastrado gera uma entrega HTTP assinada com HMAC-SHA256, contendo `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id`.
- Uma falha de entrega é reprocessada automaticamente até 5 vezes, com os intervalos de backoff definidos, antes de ser movida para dead-letter.
- Um administrador consegue reprocessar manualmente um evento em dead-letter, e essa ação fica registrada em log de auditoria.
- Um cliente consegue rotacionar sua secret sem perder a validade da secret anterior durante 24 horas.
- Nenhuma mudança de status de pedido ocorre sem o evento de notificação correspondente ser gravado (quando há webhook interessado), e vice-versa.

## Estratégia de testes e validação

Seguindo o padrão já estabelecido no projeto (`tests/orders.test.ts`, `tests/auth.test.ts`), a validação é feita majoritariamente por **testes de integração via HTTP** (`supertest`) contra um banco MySQL real, cobrindo:

- Fluxo completo de CRUD de webhook (criação com devolução de secret, edição, remoção, listagem com paginação).
- Atomicidade da transação estendida de `changeStatus`: uma falha simulada na inserção da outbox deve reverter também a mudança de status e o ajuste de estoque, no mesmo espírito do teste existente que verifica que o estoque não é decrementado quando uma transição falha por estoque insuficiente.
- Verificação de assinatura HMAC: dado um payload e uma secret conhecidos, o `X-Signature` gerado deve ser reproduzível de forma determinística.
- Cenário de retry: uma falha simulada de entrega deve resultar em `attempts` incrementado e `nextAttemptAt` calculado conforme a progressão de backoff.
- Cenário de dead-letter: um evento que esgota as 5 tentativas deve aparecer em `webhook_dead_letter` e desaparecer da `webhook_outbox` ativa.
- Controle de acesso: o endpoint de replay administrativo deve retornar `403` para usuários sem `role: ADMIN`, reaproveitando a mesma verificação já validada em testes para outros endpoints restritos a admin.
