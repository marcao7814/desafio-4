# ADR-005: Garantia at-least-once com deduplicação via X-Event-Id

## Status

Aceito

## Contexto

Dada a combinação de outbox assíncrona (ADR-001), worker com retry (ADR-003) e possibilidade de falha de rede após o cliente já ter processado a requisição mas antes da confirmação chegar ao worker, existe a possibilidade real de o mesmo evento ser entregue mais de uma vez ao cliente. É preciso decidir qual garantia de entrega o sistema oferece e como o cliente consegue lidar com entregas duplicadas.

Garantir entrega exatamente uma vez (exactly-once) entre dois sistemas distribuídos independentes exigiria coordenação bilateral (ex.: protocolo de confirmação transacional entre as duas partes), que aumenta significativamente a complexidade de implementação tanto do lado do sistema quanto do lado de cada cliente integrador.

## Decisão

O sistema garante **at-least-once delivery**: um evento pode, em cenários de falha, ser entregue mais de uma vez ao mesmo endpoint. Cada evento carrega um identificador único (`event_id`, UUID gerado no momento em que o evento entra na outbox) enviado no header **`X-Event-Id`**. O cliente é responsável por deduplicar do lado dele, usando esse identificador como chave de idempotência.

Essa responsabilidade de deduplicação do lado do cliente será documentada de forma destacada no portal de desenvolvedor / documentação de integração para os clientes B2B.

## Alternativas Consideradas

**Garantia exactly-once.** Eliminaria a necessidade de deduplicação do lado do cliente, mas exigiria protocolo de confirmação coordenado entre os dois sistemas (ex.: two-phase commit distribuído ou reconciliação ativa), adicionando complexidade desproporcional ao ganho. É o mesmo trade-off adotado por padrão de mercado em provedores de webhook consolidados (ex.: Stripe, GitHub), que também optam por at-least-once com deduplicação no cliente.

## Consequências

**Positivas**
- Simplicidade de implementação no worker: no pior caso (falha após envio, antes da confirmação), o evento simplesmente é reentregue na próxima tentativa, sem necessidade de coordenação distribuída.
- Alinhado a um padrão de mercado já validado e conhecido pelos times de integração dos clientes B2B, reduzindo o atrito de adoção.
- `X-Event-Id` também serve, de forma incidental, como identificador de rastreamento para debug e para o histórico de entregas (`GET /webhooks/:id/deliveries`).

**Negativas**
- Transfere para o cliente a responsabilidade de implementar deduplicação; um cliente que não implementar isso corretamente pode processar o mesmo evento de negócio mais de uma vez do lado dele.
- Resolve cerca de 99% dos casos práticos, mas não oferece uma garantia absoluta de entrega única — trade-off aceito conscientemente pelo time em favor de simplicidade operacional.
