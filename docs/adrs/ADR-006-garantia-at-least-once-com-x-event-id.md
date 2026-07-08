# ADR-XXX — Garantia de Entrega At-Least-Once com Deduplicação via X-Event-Id

- **Status:** Aceito
- **Data:** 2026-07-06
- **Autores:** Diego (Engenheiro Sênior, time de Plataforma), Larissa (Tech Lead)
- **Revisores:** Bruno (Engenheiro Pleno, time de Pedidos), Sofia (Engenheira de Segurança), Marcos (Product Manager)

---

## Relações

- **Depende de:** [ADR-004 — Política de Retry com Backoff Exponencial e Dead Letter Queue para Webhooks](./ADR-004-politica-de-retry-com-backoff-exponencial-e-dlq.md)
- **Relacionado a:** [ADR-007 — Payload da Outbox Armazenado como Snapshot no Momento da Inserção](./ADR-007-payload-snapshot-no-momento-da-insercao.md)

---

## Contexto

Sistemas distribuídos que realizam entregas de eventos por HTTP enfrentam um desafio fundamental: a rede é inerentemente não confiável. Quando o worker de webhooks realiza uma chamada HTTP ao endpoint do cliente e não recebe resposta — seja por timeout, queda de rede ou reinicialização do processo receptor —, ele não tem como saber se o evento foi processado ou não pelo destinatário. Nesse cenário, a única opção segura do ponto de vista do remetente é tentar novamente, o que inevitavelmente pode resultar em o cliente receber o mesmo evento mais de uma vez.

O sistema de webhooks desta plataforma foi projetado para notificar clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — sobre mudanças de status de pedidos em tempo real (conforme decisão de negócio registrada em `[09:00] Marcos`). A política de retry adotada (backoff exponencial com 5 tentativas: 1m/5m/30m/2h/12h, registrada em `[09:17] Diego` e `[09:08] Larissa`) garante que eventos não sejam perdidos em janelas de indisponibilidade dos clientes, mas introduz explicitamente a possibilidade de reentrega. Um cliente que ficou fora do ar durante a primeira tentativa e voltou a tempo da segunda receberá o evento duas vezes caso a primeira entrega tenha de fato chegado antes da queda.

A equipe avaliou as garantias de entrega possíveis — best-effort, at-least-once e exactly-once — e decidiu adotar **at-least-once com identificador estável de evento**, transferindo a responsabilidade de deduplicação para o lado do cliente. Esse modelo é amplamente adotado na indústria e é suficiente para atender ao caso de uso: os clientes precisam reagir a transições de status de pedidos (ex.: decrementar estoque, disparar logística), operações que devem ser idempotentes por natureza e que já requerem controle de reprocessamento em qualquer integração robusta.

## Decisão

O sistema de webhooks garante entrega **at-least-once**: todo evento será entregue ao menos uma vez, podendo ser entregue mais de uma vez em cenários de retry. Cada evento recebe um identificador único (UUID v4) gerado no momento em que a linha é inserida na tabela `webhook_outbox` — ou seja, no instante da transição de status do pedido, dentro da mesma transação atômica. Esse identificador é propagado como o header HTTP `X-Event-Id` em todas as tentativas de entrega do mesmo evento lógico, garantindo estabilidade: independentemente de quantas vezes o worker tentar entregar o evento, o `X-Event-Id` permanece o mesmo. A responsabilidade de deduplicar eventos recebidos com o mesmo `X-Event-Id` é do cliente integrador, conforme acordado em `[09:25] Diego` e confirmado em `[09:26] Larissa`.

O campo `eventId` da tabela `webhook_outbox` possui restrição `@unique` no schema Prisma, o que impede a inserção de duas linhas de outbox para o mesmo evento lógico — protegendo a consistência do lado do remetente. A função `publishWebhookEvent(tx, order, fromStatus, toStatus)`, chamada pelo `OrderService` dentro da transação de mudança de status (conforme proposto em `[09:41] Bruno`), é responsável por gerar o UUID do evento no momento da inserção. O payload do evento é serializado na inserção (snapshot), refletindo o estado do pedido no instante da transição, e não no momento do envio — decisão registrada em `[09:52] Larissa` e `[09:52] Diego`.

## Alternativas Consideradas

### Exactly-once delivery

A entrega exactly-once garantiria que cada evento fosse processado exatamente uma vez pelo receptor, eliminando a necessidade de deduplicação no lado do cliente. Porém, implementar essa garantia em sistemas distribuídos requer coordenação bidirecional: o receptor precisaria confirmar atomicamente o recebimento e o processamento do evento de forma que o remetente pudesse, ao mesmo tempo, marcar o evento como consumido de forma irrevogável. Isso introduz acoplamento forte entre a confiabilidade do remetente e a disponibilidade e comportamento do receptor — qualquer falha do lado do cliente impacta diretamente a integridade do processo no lado da plataforma.

Conforme explicitado por Diego em `[09:24]` e `[09:25]`: *"exactly-once é impossível sem coordenação dos dois lados. A gente entrega at-least-once"* e *"Stripe faz isso, GitHub faz isso. É o padrão da indústria"*. A complexidade adicional não se justifica para o caso de uso: os clientes B2B integrados já são esperados a implementar idempotência em suas operações críticas, e a documentação no portal de desenvolvedor (mencionada por Marcos em `[09:26]`) é suficiente para orientar a implementação correta.

### Best-effort sem identificador de evento

Uma abordagem ainda mais simples seria entregar eventos sem qualquer mecanismo de identificação, tornando impossível a deduplicação do lado do cliente em caso de reentrega. Essa alternativa foi descartada de imediato pela equipe: todos os participantes concordaram que um identificador estável de evento é necessário desde o início, pois as consequências de processamento duplicado em operações de negócio (como duplo decremento de estoque ou disparo redundante de logística) são inaceitáveis para os clientes B2B. Não há registro de discussão favorável a essa opção na transcrição.

## Consequências

### Positivas

- Implementação simples do lado do remetente: gerar um UUID na inserção do outbox, incluí-lo no header `X-Event-Id` em todas as tentativas de envio e retentar livremente sem preocupação com estado do receptor.
- Alinhamento com o padrão de mercado amplamente adotado: Stripe, GitHub e Shopify utilizam o mesmo modelo, o que reduz a curva de aprendizado para clientes que já integraram essas plataformas.
- A restrição `@unique` em `webhook_outbox.eventId` previne duplicatas de outbox para o mesmo evento lógico, garantindo consistência do lado da plataforma.
- Ausência de acoplamento entre a lógica de retry do worker e o comportamento ou disponibilidade do receptor: o worker pode retentar de forma completamente autônoma.
- O `X-Event-Id` estável permite auditoria e rastreabilidade: é possível correlacionar todas as tentativas de entrega de um mesmo evento lógico na tabela de histórico (`webhook_delivery_log` ou similar).
- Compatível com o header `X-Timestamp` (registrado em `[09:44] Diego`) que permite ao cliente detectar replay attacks se desejar implementar proteção adicional.

### Negativas / Trade-offs

- O cliente integrador é obrigado a implementar lógica de idempotência usando o `X-Event-Id`, o que aumenta a complexidade da integração do lado dele.
- Se um cliente não implementar deduplicação corretamente, poderá processar o mesmo evento múltiplas vezes — com consequências potencialmente graves dependendo da operação (ex.: duplo decremento de estoque, dupla notificação ao destinatário final do pedido).
- A documentação no portal de desenvolvedor (responsabilidade de Marcos, conforme `[09:26]`) precisa explicar de forma clara e destacada a semântica at-least-once e a obrigatoriedade de deduplicação pelo `X-Event-Id` — falha nessa comunicação resultará em integrações incorretas.
- O onboarding de novos clientes B2B precisará incluir verificação de que a implementação do lado deles contempla deduplicação, o que pode adicionar fricção no processo de integração.

## Referências

### Transcrição

- `[09:24] Diego` — descarte de exactly-once por impossibilidade sem coordenação bidirecional
- `[09:25] Diego` — definição de at-least-once, X-Event-Id como UUID gerado na entrada da outbox, responsabilidade de dedup no cliente
- `[09:25] Sofia` — observação sobre transferência de responsabilidade para o cliente
- `[09:25] Diego` — referência explícita ao padrão Stripe e GitHub como validação da abordagem
- `[09:26] Marcos` — comprometimento com documentação clara no portal de desenvolvedores
- `[09:26] Larissa` — registro formal da decisão: at-least-once com X-Event-Id para deduplicação no cliente
- `[09:41] Bruno` — proposta da função `publishWebhookEvent(tx, order, fromStatus, toStatus)` como ponto de geração do eventId dentro da transação
- `[09:44] Diego` — listagem dos headers HTTP do request de webhook: X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id, Content-Type
- `[09:52] Larissa` / `[09:52] Diego` — decisão de payload como snapshot no momento da inserção (não renderizado no envio)

### Código

- `C:\www\full-cycle\mba\mba-ia-desafio-design-docs-com-ia\prisma\schema.prisma` — schema Prisma com o padrão UUID (`@id @default(uuid())`) adotado em todos os modelos; campo `webhook_outbox.eventId` deve ter restrição `@unique`
- `C:\www\full-cycle\mba\mba-ia-desafio-design-docs-com-ia\docs\adrs\potential-adrs\must-document\MOD-008\at-least-once-delivery-with-x-event-id-dedup.md` — potential ADR de origem com evidências e pontuação
