# ADR-003: Retry com backoff exponencial e Dead Letter Queue

## Status

Aceito

## Contexto

Clientes de webhook podem estar temporariamente indisponíveis (manutenção planejada, incidentes, deploys). O sistema já teve um caso de cliente com indisponibilidade de cerca de duas horas em manutenção planejada. É necessário decidir quantas vezes tentar reenviar um evento que falhou, com qual espaçamento entre tentativas, e o que fazer quando todas as tentativas se esgotam.

Também é preciso decidir onde registrar eventos que falharam definitivamente: manter apenas um campo de status "failed" na própria tabela `webhook_outbox`, ou usar uma tabela dedicada.

## Decisão

Adotar **retry com backoff exponencial, limitado a 5 tentativas**, com a seguinte progressão de intervalos entre tentativas: 1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas — totalizando uma janela de aproximadamente 15 horas entre a primeira falha e a última tentativa.

Esgotadas as 5 tentativas, o evento é movido para uma tabela dedicada **`webhook_dead_letter`**, contendo o payload original, o motivo da última falha e o timestamp. Isso mantém a tabela `webhook_outbox` enxuta (só eventos ativos/pendentes) e preserva o histórico de falhas como evidência para debug e reprocessamento.

Eventos em dead-letter podem ser reprocessados manualmente por um operador com role `ADMIN`, via endpoint `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente. O endpoint deve registrar (log de auditoria) qual usuário admin executou o replay.

## Alternativas Consideradas

**3 tentativas.** Mais agressivo, mas insuficiente: um cliente com indisponibilidade de poucas horas (caso já observado na prática) esgotaria as tentativas em poucos minutos, perdendo a notificação definitivamente antes que o cliente voltasse ao ar.

**Retry indefinido com backoff crescente, sem limite de tentativas.** Evita perder eventos, mas deixa eventos "pendurados" indefinidamente se o cliente desativou o endpoint ou saiu do ar permanentemente, poluindo a fila ativa com lixo operacional sem sinalização clara de falha permanente.

**Marcar falha definitiva como status na própria tabela `webhook_outbox`, sem tabela separada.** Mais simples de implementar, mas mistura eventos ativos (pendentes/processando) com eventos mortos na mesma tabela consultada pelo worker em todo ciclo de polling, dificultando a leitura operacional e exigindo filtros adicionais em toda query do worker.

## Consequências

**Positivas**
- Cobre cenários reais de indisponibilidade de cliente (até a ordem de meio dia) sem exigir intervenção manual.
- Separação clara entre eventos ativos (`webhook_outbox`) e eventos definitivamente falhos (`webhook_dead_letter`) simplifica tanto o worker quanto a auditoria de falhas.
- Reprocessamento manual e controlado (via endpoint restrito a `ADMIN`) dá ao time uma via de recuperação sem reprocessar automaticamente eventos que podem estar definitivamente inválidos (ex.: URL descadastrada).

**Negativas**
- Um cliente fora do ar por mais de ~15 horas perde a notificação automática daquele evento específico e depende de replay manual a partir da dead-letter.
- Mantém duas tabelas relacionadas ao ciclo de vida do mesmo evento, exigindo cuidado para não duplicar processamento entre elas.
- O endpoint de replay manual é uma nova superfície administrativa que precisa de controle de acesso e auditoria (ver [ADR-006](ADR-006-reuso-padroes-existentes.md) para reaproveitamento de `requireRole`).
