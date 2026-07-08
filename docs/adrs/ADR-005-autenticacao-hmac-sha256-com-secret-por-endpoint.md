# ADR-XXX — Autenticação HMAC-SHA256 com Secret por Endpoint e Grace Period de Rotação

- **Status:** Aceito
- **Data:** 2026-07-06
- **Autores:** Sofia (Engenheira de Segurança), Diego (Engenheiro Sênior, time de Plataforma)
- **Revisores:** Larissa (Tech Lead), Bruno (Engenheiro Pleno, time de Pedidos), Marcos (Product Manager)

---

## Relações

- **Relacionado a:** [ADR-008 — Módulo de Webhook Segue os Padrões Existentes de Controller→Service→Repository e Reutiliza a Infraestrutura Compartilhada](./ADR-008-modulo-webhook-segue-padroes-existentes.md)

---

## Contexto

O sistema de webhooks de notificação de pedidos expõe eventos com dados sensíveis de clientes B2B — como mudanças de status de pedidos — para endpoints HTTP externos fora da infraestrutura da plataforma. Conforme destacado por Sofia em `[09:19]`, qualquer ator malicioso que intercepte ou forje requisições HTTP para o endpoint de um cliente pode injetar eventos falsos de mudança de status, comprometendo a integridade das integrações. A necessidade de autenticação e verificação de integridade do payload é, portanto, um requisito de segurança não negociável para o MOD-008.

O contexto B2B da plataforma amplifica o risco: os três clientes iniciais (Atlas Comercial, MaxDistribuição e Nova Cargo) são parceiros que tomam decisões operacionais automatizadas com base nos eventos recebidos. Um payload adulterado ou forjado pode causar ações incorretas em sistemas externos sem qualquer intervenção humana. Além disso, como cada cliente cadastra seu próprio endpoint de recebimento, é necessário um mecanismo de autenticação que isolie o risco por cliente, e não de forma global.

A discussão de segurança ocorreu entre `[09:19]` e `[09:22]`, conduzida por Sofia, com participação de Diego, Larissa e Bruno. O resultado foi a adoção do padrão HMAC-SHA256 com secret individual por endpoint, seguindo o mesmo modelo amplamente adotado pela indústria (Stripe, GitHub Webhooks). A decisão também contempla o ciclo de vida das secrets, incluindo um mecanismo de rotação com período de carência (grace period) para evitar falhas de entrega durante a transição.

---

## Decisão

O worker de webhooks assina o corpo completo do payload JSON com HMAC-SHA256 usando a secret exclusiva do endpoint de destino, e inclui a assinatura resultante no header `X-Signature` de cada requisição HTTP de saída. Conforme definido por Sofia em `[09:20]` e `[09:21]`, cada endpoint de webhook registrado na tabela `webhook_endpoints` possui seu próprio campo `secret` — não existe uma secret global da plataforma. Dessa forma, o vazamento da secret de um único cliente não compromete os demais. O receptor, ao receber a notificação, recalcula o HMAC sobre o body usando a secret que ele conhece e compara com o valor do header `X-Signature` para verificar autenticidade e integridade do payload.

Para o ciclo de vida das secrets, foi decidida em `[09:21]` (Sofia) e confirmada em `[09:22]` (Diego e Larissa) a implementação de rotação com grace period de 24 horas. Quando um cliente solicita a rotação da sua secret pela API, a secret anterior permanece válida em paralelo com a nova secret durante 24 horas. Nesse período de carência, o worker tenta verificar a assinatura com ambas as secrets (antiga e nova) antes de rejeitar uma entrega. Após o encerramento do grace period, a secret antiga é invalidada definitivamente. Esse mecanismo, conforme argumentado por Diego em `[09:22]`, evita a janela de falha que ocorreria se a troca fosse atômica — situação que já causou problemas reais na plataforma quando um cliente vazou uma secret em logs de aplicação.

---

## Alternativas Consideradas

### Secret global única da plataforma

Uma única secret compartilhada entre a plataforma e todos os clientes receptores foi considerada como opção mais simples de implementação. Essa alternativa foi rejeitada por Sofia em `[09:21]` porque, caso a secret global vazasse — seja por acidente nos logs de um cliente, seja por ataque direcionado —, todos os consumidores de webhook da plataforma estariam comprometidos simultaneamente. O modelo de secret por endpoint limita o raio de impacto (blast radius) a um único cliente.

### API Key no header sem assinatura de payload

A abordagem de enviar apenas uma chave de API no header de autorização, sem assinar o conteúdo do payload, não foi formalmente debatida na reunião, mas o HMAC foi escolhido diretamente porque ele garante não apenas autenticidade (quem enviou) mas também integridade (o conteúdo não foi alterado em trânsito). Uma API Key simples não impede ataques de adulteração de payload por intermediários.

### Sem autenticação — controle apenas por allowlist de IPs

O modelo de não assinar payloads e confiar exclusivamente em regras de firewall ou allowlist de IPs de saída não chegou a ser discutido, pois o time, liderado por Sofia, adotou HMAC como ponto de partida padrão da indústria. Essa alternativa seria insuficiente em ambientes cloud onde os IPs de saída podem mudar, e não protege contra adulteração de payload em trânsito.

---

## Consequências

### Positivas

- Isolamento de secret por endpoint: o comprometimento da credencial de um único cliente não afeta os demais cadastros de webhook na plataforma.
- HMAC-SHA256 garante simultaneamente autenticidade (o payload veio da plataforma) e integridade (nenhum byte foi alterado em trânsito), pois assina o corpo completo da requisição.
- O grace period de 24 horas durante a rotação de secrets elimina a janela de falha de entrega que existiria em uma troca atômica, permitindo que o cliente migre seus sistemas sem interrupção.
- Segue o padrão consolidado da indústria (Stripe, GitHub Webhooks), reduzindo a curva de aprendizado dos times de integração dos clientes B2B.
- A revisão do código de HMAC e de geração de secrets pela Sofia antes do deploy (prevista no planejamento de sprints em `[09:46]`) adiciona uma camada formal de auditoria de segurança.

### Negativas / Trade-offs

- A tabela `webhook_endpoints` passa a armazenar secrets criptográficas que precisam ser protegidas em repouso (encryption at rest recomendada), aumentando os requisitos de segurança da camada de persistência.
- Durante o grace period de rotação, o worker precisa tentar a verificação com duas secrets diferentes antes de rejeitar uma entrega, adicionando complexidade à lógica de processamento do `WebhookProcessor`.
- Os clientes receptores precisam implementar a verificação HMAC no lado deles, o que aumenta a complexidade de integração em comparação com um modelo sem autenticação. Esse custo precisa estar bem documentado no portal de desenvolvedores (responsabilidade do Marcos, conforme `[09:26]`).
- A secret gerada pela plataforma é devolvida ao cliente apenas no momento da criação do endpoint (conforme modelo descrito por Marcos em `[09:31]`); se o cliente perder a secret, precisará rotacioná-la, passando pelo fluxo de grace period.

---

## Referências

### Transcrição

- `[09:19] Sofia` — motivação para autenticação: proteção contra injeção de eventos falsos em endpoints externos
- `[09:20] Sofia` — escolha de HMAC-SHA256, header `X-Signature`, assinatura sobre o payload completo
- `[09:20] Bruno` — confirmação do algoritmo SHA-256
- `[09:21] Sofia` — secret exclusiva por endpoint (não global), campo na tabela de configuração; URL + secret + customer_id + estado ativo
- `[09:21] Sofia` — secret rotacionável via API, grace period de 24 horas com dois secrets válidos em paralelo
- `[09:22] Diego` — confirmação do grace period; referência ao incidente real de vazamento de secret em log de aplicação de cliente
- `[09:22] Sofia` — resumo da decisão: HMAC-SHA256 sobre corpo do request, secret por endpoint, rotação com grace period de 24h
- `[09:22] Larissa` — aceite formal da decisão
- `[09:44] Diego` — headers de saída: `X-Event-Id`, `X-Signature`, `X-Timestamp`, `Content-Type: application/json`
- `[09:44] Sofia` — adição do header `X-Webhook-Id` para identificação do endpoint de origem
- `[09:46] Sofia` — reserva de revisão de segurança do código de HMAC e geração de secrets antes do deploy

### Código

- `prisma/schema.prisma` — modelos existentes `Customer`, `Order`, `OrderStatusHistory`; a tabela `webhook_endpoints` (a ser criada no MOD-008) incluirá o campo `secret` por endpoint, seguindo o padrão `@id @default(uuid())` e `@@map()` dos demais modelos
- `src/modules/webhooks/webhook.processor.ts` (a ser criado) — responsável pela lógica de assinatura HMAC e verificação dual de secrets durante grace period
- `src/worker.ts` (a ser criado) — entry-point do worker separado que executa o `WebhookProcessor`
