# ADR-XXX — Política de Retry com Backoff Exponencial e Dead Letter Queue para Webhooks

- **Status:** Aceito
- **Data:** 2026-07-06
- **Autores:** Diego (Engenheiro Sênior, time de Plataforma), Larissa (Tech Lead)
- **Revisores:** Bruno (Engenheiro Pleno, time de Pedidos), Marcos (Product Manager), Sofia (Engenheira de Segurança)

---

## Contexto

O sistema de webhooks desenvolvido no módulo MOD-008 é responsável por notificar clientes B2B — como Atlas Comercial, MaxDistribuição e Nova Cargo — sobre mudanças de status nos seus pedidos. Quando o status de um pedido é alterado no serviço de orders, um evento é gravado atomicamente na tabela `webhook_outbox` dentro da mesma transação SQL que atualiza `orders` e `order_status_history` (conforme definido no schema em `prisma/schema.prisma`). Um worker separado, rodando em polling de 2 segundos, lê esses eventos pendentes e realiza chamadas HTTP para os endpoints cadastrados pelos clientes.

O problema central que motivou esta decisão é: o que fazer quando a chamada HTTP ao endpoint do cliente falha? Clientes B2B frequentemente possuem janelas de manutenção planejadas, indisponibilidades de rede ou incidentes que deixam seus sistemas fora do ar por períodos que vão de minutos a várias horas. Como registrado na transcrição da reunião técnica (`[09:16] Diego` — "já tinha cliente nosso com indisponibilidade de duas horas em manutenção planejada"), uma política de retry muito agressiva descartaria eventos válidos antes que o cliente tivesse chance de recuperar seu sistema. Por outro lado, tentar indefinidamente gera acúmulo de eventos potencialmente obsoletos na `webhook_outbox`, polui o worker e torna o comportamento do sistema difícil de raciocinar.

Existe também uma restrição de integridade operacional: eventos que ficam permanentemente presos no outbox bloqueiam o crescimento da fila e dificultam o monitoramento. É necessário um mecanismo de escape que isole definitivamente os eventos irrecuperáveis sem descartá-los silenciosamente, preservando a possibilidade de intervenção manual por operadores administrativos. O timeout por chamada foi definido em 10 segundos (`[09:42] Diego` — "10 segundos. Cliente lento que não responde em 10s a gente trata como falha e marca pra retry"), o que delimita claramente o que constitui uma tentativa falha.

---

## Decisão

Foi decidido adotar uma política de **5 tentativas de retry com backoff exponencial** para falhas na entrega de webhooks. Os intervalos entre tentativas seguem a progressão: **1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas**, cobrindo uma janela total de aproximadamente 15 horas entre a primeira falha e o encerramento da última tentativa (`[09:17] Diego` — "1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas. Total de quase 15 horas entre primeira falha e última tentativa"). Conforme destacado por Larissa (`[09:16] Larissa` — "cobre uma janela de aproximadamente 15 horas de indisponibilidade do cliente"), esse intervalo foi considerado suficiente para cobrir situações realistas de manutenção planejada ou incidentes de maior duração que os clientes B2B tipicamente enfrentam. O campo `nextRetryAt` na tabela `webhook_outbox` será utilizado pelo worker para filtrar apenas os eventos cujo próximo retry já é devido, evitando processamento desnecessário.

Após o esgotamento das 5 tentativas, o evento é movido para uma tabela separada denominada `webhook_dead_letter`, que armazena o payload original, o motivo da falha e o timestamp de expiração (`[09:18] Diego` — "tabela webhook_dead_letter separada, com a payload, motivo da falha e timestamp"). O registro correspondente é removido da `webhook_outbox`, mantendo-a limpa e eficiente para o worker em polling. Operadores com role `ADMIN` podem acionar o replay manual de um evento na DLQ via `POST /admin/webhooks/dead-letter/:id/replay`, que reinsere o evento na `webhook_outbox` com status pendente (`[09:35] Diego`; `[09:36] Larissa` — "role ADMIN obrigatório no replay e a gente reaproveita o requireRole que já existe"). O middleware `requireRole` já existente em `src/middlewares/auth.middleware.ts` é reutilizado para proteger esse endpoint sem necessidade de nova infraestrutura de autorização.

---

## Alternativas Consideradas

### 3 tentativas com intervalos mais curtos

Bruno sugeriu reduzir para 3 tentativas, argumentando que seria uma abordagem mais agressiva e com menor acúmulo de estados intermediários (`[09:16] Bruno` — "3 não é melhor? Mais agressivo"). Diego rejeitou essa alternativa porque 3 tentativas com intervalos razoáveis cobririam no máximo 30 a 40 minutos de indisponibilidade (`[09:16] Diego` — "3 é pouco. Se o cliente teve indisponibilidade de manhã, a gente retentaria três vezes em 30 minutos e mataria"). Para o contexto B2B, onde manutenções planejadas de 2 horas ou mais são comuns, esta política resultaria em perda definitiva de eventos que ainda poderiam ser entregues com sucesso em uma janela ligeiramente maior. A alternativa foi descartada por insuficiência de cobertura temporal para o perfil de clientes da plataforma.

### Retry indefinido com backoff exponencial

Uma segunda alternativa discutida foi continuar tentando indefinidamente, sem limite de tentativas, mas com intervalos crescentes de backoff. Essa abordagem garantiria que nenhum evento fosse descartado enquanto o endpoint do cliente eventualmente se recuperasse. A proposta foi rejeitada porque eventos podem tornar-se semanticamente obsoletos ao longo do tempo — por exemplo, um pedido cancelado pelo cliente antes que seu sistema se recupere (`[09:16] Larissa` — contexto implícito de que "retry indefinido polui o outbox e torna o worker mais difícil de raciocinar"). Além disso, uma `webhook_outbox` com eventos acumulados indefinidamente degradaria a performance do worker em polling, exigiria estratégias de arquivamento progressivamente mais complexas e tornaria o comportamento do sistema imprevisível para a equipe de operações.

---

## Consequências

### Positivas

- A janela de ~15 horas cobre com folga manutenções planejadas e a grande maioria dos incidentes de disponibilidade que clientes B2B tipicamente enfrentam, conforme observado historicamente pela equipe.
- A `webhook_outbox` permanece enxuta: eventos entregues com sucesso ou movidos para a DLQ são removidos da tabela principal, preservando a eficiência do índice composto em `(status, nextRetryAt)` necessário para o worker.
- O campo `nextRetryAt` permite que o worker ignore eventos ainda não vencidos em cada ciclo de polling, sem varredura completa da tabela.
- A tabela `webhook_dead_letter` funciona como evidência auditável de falhas permanentes, com payload e motivo preservados para diagnóstico.
- O endpoint `POST /admin/webhooks/dead-letter/:id/replay` permite intervenção manual por operadores ADMIN sem necessidade de alterações de código ou acesso direto ao banco de dados, reutilizando o `requireRole('ADMIN')` já implementado em `src/middlewares/auth.middleware.ts`.
- A progressão de backoff distribui a carga de retries ao longo do tempo, evitando picos de chamadas simultâneas a endpoints que podem estar se recuperando.

### Negativas / Trade-offs

- Eventos em backoff máximo (12h) podem ser entregues com até 15 horas de atraso após a primeira falha, o que pode ser inaceitável para eventos sensíveis ao tempo em alguns fluxos de negócio futuros.
- A tabela `webhook_dead_letter` requer monitoramento ativo; eventos acumulados sem replay por longos períodos podem confundir operadores ou criar dívida operacional invisível.
- A `webhook_outbox` precisa de índice composto em `(status, nextRetryAt)` para que o worker em polling mantenha performance aceitável à medida que o volume de eventos cresce.
- O replay manual via endpoint admin exige intervenção humana; não há mecanismo automático de re-enfileiramento após um período de quarentena na DLQ, o que pode gerar acúmulo em cenários de falha prolongada sem atenção da equipe de operações.
- Notificação proativa ao cliente quando seu webhook acumula falhas consecutivas foi explicitamente descartada para esta fase (`[09:37] Larissa` — "Email tá fora de escopo dessa fase"), o que significa que clientes podem não perceber que seus eventos estão falhando sem consultarem os logs de entrega.

---

## Referências

### Transcrição

- `[09:15] Diego` — proposta de 5 tentativas com backoff exponencial e DLQ
- `[09:15] Bruno` — questionamento sobre 3 tentativas como alternativa mais agressiva
- `[09:16] Diego` — justificativa de rejeição das 3 tentativas (indisponibilidades de 2h em manutenção planejada)
- `[09:16] Bruno` — "3 não é melhor? Mais agressivo."
- `[09:16] Larissa` — "cobre uma janela de aproximadamente 15 horas de indisponibilidade do cliente"
- `[09:16] Diego` — "se passar das cinco, vai pra dead letter"
- `[09:17] Diego` — definição dos intervalos exatos: 1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas
- `[09:17] Marcos` — validação de negócio: "Se um cliente meu cair por 15 horas, ele já tá com problema sério dele. Acho aceitável."
- `[09:17] Larissa` — "Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h."
- `[09:18] Diego` — definição da tabela `webhook_dead_letter` separada com payload, motivo e timestamp
- `[09:18] Bruno` — solicitação de endpoint para reprocessamento manual
- `[09:18] Diego` — `POST /admin/webhooks/dead-letter/:id/replay`
- `[09:35] Diego` — confirmação do endpoint de replay da DLQ
- `[09:36] Larissa` — "role ADMIN obrigatório no replay e a gente reaproveita o requireRole que já existe"
- `[09:36] Sofia` — "Mexer em fila de entrega de notificação não é coisa de operador. E o endpoint de admin tem que logar quem fez o replay, pra auditoria."
- `[09:42] Diego` — timeout de 10 segundos por chamada HTTP antes de considerar falha
- `[09:48] Larissa` — resumo final confirmado por todos os participantes

### Código

- `prisma/schema.prisma` — schema atual com modelos `Order`, `OrderStatus` e `OrderStatusHistory`; as tabelas `webhook_outbox` e `webhook_dead_letter` serão adicionadas neste schema seguindo o mesmo padrão (UUID como `@id @default(uuid())`, campos `createdAt`/`updatedAt`, `@@map` com snake_case)
- `src/middlewares/auth.middleware.ts` — implementação de `requireRole(...roles)` que será reutilizada para proteger o endpoint `POST /admin/webhooks/dead-letter/:id/replay` com `requireRole('ADMIN')`
