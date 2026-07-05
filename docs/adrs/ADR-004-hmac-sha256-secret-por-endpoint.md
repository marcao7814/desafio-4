# ADR-004: Autenticação de webhooks via HMAC-SHA256 com secret por endpoint

## Status

Aceito

## Contexto

O sistema passará a enviar dados de pedidos (potencialmente sensíveis do ponto de vista comercial) para endpoints HTTP fora da infraestrutura da empresa, controlados pelos próprios clientes. O cliente que recebe o webhook precisa de uma forma de verificar que a requisição realmente partiu do sistema e que o payload não foi adulterado em trânsito.

Além disso, é preciso decidir se todos os endpoints de um cliente (ou de todos os clientes) compartilham uma única credencial de assinatura, ou se cada endpoint cadastrado tem a sua própria.

## Decisão

Cada requisição de webhook é assinada com **HMAC-SHA256** sobre o corpo (body) da requisição, usando uma **secret exclusiva por endpoint cadastrado** (não uma secret global da plataforma nem compartilhada entre clientes). A assinatura resultante vai no header `X-Signature`; o cliente recalcula o HMAC do lado dele com a mesma secret e compara.

A secret é gerada pelo sistema no momento do cadastro do endpoint e devolvida uma única vez na resposta de criação. Ela é **rotacionável**: o cliente pode solicitar uma nova secret via API, e durante um **grace period de 24 horas** a secret antiga continua válida em paralelo à nova, dando tempo para o cliente atualizar seus sistemas antes que a antiga seja definitivamente invalidada.

## Alternativas Consideradas

**Secret única global da plataforma (compartilhada entre todos os endpoints/clientes).** Mais simples de gerenciar, mas concentra risco: o vazamento de uma única secret (já ocorreu um caso de cliente vazando secret em log de aplicação) comprometeria a autenticidade de webhooks para todos os clientes simultaneamente, não apenas o endpoint afetado. Descartada por ampliar demais o raio de impacto de um vazamento.

## Consequências

**Positivas**
- HMAC-SHA256 é padrão de mercado amplamente suportado (praticamente toda linguagem/plataforma tem biblioteca pronta), reduzindo o atrito de integração para os clientes B2B.
- Secret por endpoint limita o raio de impacto de um vazamento a um único cadastro, não à plataforma inteira nem a outros endpoints do mesmo cliente.
- Rotação com grace period de 24h permite resposta a um vazamento sem quebrar a integração do cliente em produção.

**Negativas**
- Exige armazenar e gerenciar o ciclo de vida de duas secrets simultâneas por endpoint durante o período de rotação (atual e anterior), aumentando a complexidade do modelo de dados e da lógica de verificação no worker.
- A geração e o armazenamento seguro da secret (nunca logada, nunca reexibida após a criação) exigem revisão de segurança dedicada antes do deploy, conforme já sinalizado pela responsável por segurança na reunião.
- Não protege contra replay attack por si só — depende de o cliente validar o timestamp da requisição (`X-Timestamp`) do lado dele; o sistema fornece o dado, mas não impõe a validação.
