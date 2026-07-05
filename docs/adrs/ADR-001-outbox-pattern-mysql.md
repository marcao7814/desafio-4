# ADR-001: Padrão Outbox no MySQL para entrega de eventos de webhook

## Status

Aceito

## Contexto

O sistema de webhooks precisa notificar clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) quando o status de um pedido muda. A mudança de status já ocorre dentro de uma transação SQL que atualiza `orders`, insere em `order_status_history` e ajusta `stock_quantity` dos produtos (`src/modules/orders/order.service.ts`, método `changeStatus`).

Disparar a chamada HTTP de webhook de forma síncrona, dentro dessa mesma transação, foi descartado: qualquer cliente lento ou fora do ar travaria a mudança de status de outros pedidos, e não há como fazer rollback de uma chamada HTTP já enviada. Ao mesmo tempo, o evento de notificação só pode existir se a transação de mudança de status realmente commitar — não pode haver evento "órfão" nem mudança de status sem evento correspondente.

O time é pequeno e já opera MySQL como único banco de dados. Subir uma infraestrutura de mensageria dedicada (ex.: Redis Streams, um message broker) para resolver isso foi considerado desproporcional ao problema.

## Decisão

Adotar o padrão **Transactional Outbox** usando o próprio MySQL: dentro da mesma transação que atualiza `orders` e insere em `order_status_history`, uma linha é inserida em uma nova tabela `webhook_outbox` contendo o evento a ser entregue (payload já renderizado — ver [ADR-006](ADR-006-reuso-padroes-existentes.md) para o motivo de reaproveitar a transação existente). Um worker separado (ver [ADR-002](ADR-002-worker-processo-separado-polling.md)) lê essa tabela de forma assíncrona e realiza as chamadas HTTP.

Como a inserção na outbox ocorre na mesma transação Prisma (`tx.$transaction`) que já existe em `changeStatus`, a consistência é garantida pelas próprias garantias ACID do MySQL: se a transação principal commita, o evento foi persistido; se sofre rollback, o evento desaparece junto.

## Alternativas Consideradas

**Fila dedicada (Redis Streams ou similar).** Resolveria o desacoplamento entre a geração do evento e o envio, mas exigiria subir e operar mais um componente de infraestrutura (cluster, monitoramento, backup) para um time pequeno que já mantém apenas MySQL. Descartada por ser overengineering neste estágio — não há requisito de throughput que justifique o custo operacional adicional.

**Disparo síncrono no `OrderService.changeStatus`.** Mais simples de implementar, mas acopla a disponibilidade de clientes externos à latência da transação interna de mudança de status, e não permite rollback de uma chamada HTTP já feita. Descartada por risco de indisponibilidade em cascata.

## Consequências

**Positivas**
- Consistência forte entre mudança de status e criação do evento: nunca há status mudado sem evento e nunca há evento sem status efetivamente mudado, sem exigir um segundo sistema de armazenamento nem two-phase commit.
- Reaproveita a infraestrutura de banco já operada pelo time, sem novo componente para provisionar, monitorar ou dar patch de segurança.
- Simples de auditar e depurar: a tabela outbox é consultável via SQL comum.

**Negativas**
- Introduz acoplamento entre a carga de escrita de eventos e o MySQL de produção (mais uma tabela de alta escrita no mesmo banco transacional da aplicação).
- Requer rotina de manutenção (arquivamento de eventos entregues após 30 dias, fora do escopo desta feature) para a tabela não crescer indefinidamente.
- Não escala horizontalmente por si só — depende da decisão de worker único descrita em [ADR-002](ADR-002-worker-processo-separado-polling.md); paralelizar workers no futuro exigirá particionamento ou lock pessimista, hoje fora de escopo.
