### PRD: OMS — Sistema de Webhooks de Notificação de Pedidos

Versão: 1.0
Data: 2026-07-06
Responsável: Marcos (Product Manager)

---

### Resumo

O Order Management System (OMS) hoje não possui nenhum mecanismo de notificação externa: clientes B2B precisam fazer polling manual em `GET /orders` para descobrir se o status de um pedido mudou. Esta feature adiciona um sistema de **webhooks outbound**: sempre que o status de um pedido muda, a plataforma notifica ativamente, via HTTP assinado, os endpoints cadastrados pelos clientes, eliminando a necessidade de polling e reduzindo a latência de integração para menos de 10 segundos (`[09:00-09:02] Marcos`).

---

### Contexto e problema

Público-alvo
- Clientes B2B integrados via API que consomem pedidos da plataforma (inicialmente Atlas Comercial, MaxDistribuição e Nova Cargo) (`[09:00] Marcos`)
- Operadores internos da plataforma (roles `OPERATOR`/`ADMIN`) que cadastram e administram webhooks em nome dos clientes (`[09:32] Larissa`)

Cenários de uso chave
- Um cliente B2B cadastra um endpoint HTTPS e passa a receber notificação automática sempre que um pedido dele muda de status (ex.: `PAID` → `PROCESSING` → `SHIPPED`)
- Um cliente escolhe receber notificações apenas para os status que interessam a ele (ex.: só `SHIPPED` e `DELIVERED`) (`[09:33] Marcos`)
- Um cliente consulta o histórico de entregas do seu webhook para auditar sucesso/falha das notificações recebidas (`[09:34] Marcos`)
- Um operador `ADMIN` reprocessa manualmente um evento que falhou definitivamente e foi para a fila de falhas (`[09:18] Diego`, `[09:35-09:36] Larissa`)

Onde essa feature será implantada
- Sistema existente: o OMS, aplicação Node.js/TypeScript com Express, MySQL via Prisma, já em produção com módulos de autenticação, usuários, clientes, produtos e pedidos. A feature adiciona um módulo novo (`src/modules/webhooks/`) e um processo novo (`src/worker.ts`), reaproveitando a infraestrutura e os padrões já estabelecidos no projeto, sem alterar o comportamento dos módulos existentes.

Problemas priorizados
- Clientes fazem polling constante em `GET /orders` para detectar mudanças de status, o que é lento e caro para a integração deles — impacto alto, prioridade alta (`[09:00] Marcos`)
- Risco comercial concreto: a Atlas Comercial sinalizou possível migração para um concorrente caso a solução não seja entregue até o fim do trimestre — impacto alto, prioridade alta (`[09:00] Marcos`)
- Ausência total de mecanismo de notificação externa, evento ou fila no sistema atual, exigindo desenho e construção do zero — impacto médio (esforço de implementação), prioridade alta (contexto do desafio; ausência confirmada no código-fonte)

---

### Objetivos e métricas

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| Eliminar a necessidade de polling manual em `GET /orders` pelos clientes B2B | Latência entre a mudança de status do pedido e a entrega bem-sucedida do webhook | ≤ 10 segundos, no caso de endpoint do cliente disponível (`[09:02] Marcos`, `[09:10] Larissa`) |
| Garantir que nenhuma mudança de status fique sem notificação correspondente | Proporção de mudanças de status commitadas que geram evento registrado na outbox | 100% (atomicidade transacional entre `changeStatus` e a inserção do evento) (`[09:04] Bruno`, ADR-001) |
| Absorver indisponibilidades temporárias dos clientes sem perder eventos | Janela de retry coberta antes do evento ser considerado falha definitiva | ≥ 15 horas (5 tentativas, backoff 1m/5m/30m/2h/12h), cobrindo o caso real de indisponibilidade de 2h já enfrentado (`[09:16-09:17] Diego`) |

---

### Escopo

Incluso
- Cadastro, edição, remoção e listagem de webhooks por cliente (URL, filtro de status desejado)
- Entrega automática de eventos de mudança de status via HTTP, assinada com HMAC-SHA256
- Retry automático com backoff exponencial e Dead Letter Queue (DLQ) para falhas persistentes
- Endpoint administrativo de reprocessamento manual de eventos em DLQ, restrito a role `ADMIN`
- Histórico consultável de entregas por webhook (sucesso/falha, payload, resposta, tempo de resposta)
- Rotação de secret por endpoint com período de carência (grace period) de 24 horas
- Identificador estável de evento (`X-Event-Id`) para deduplicação do lado do cliente (garantia at-least-once)

Fora de escopo
- Notificação proativa (ex.: e-mail) ao cliente quando o webhook dele acumula falhas — adiado para fase futura, após medição de impacto (`[09:37] Marcos, Larissa`)
- Rate limiting de envio de webhooks por cliente — equipe optou por observar o comportamento em produção antes de decidir se é necessário (`[09:38-09:39] Diego, Larissa`)
- Dashboard visual para o cliente acompanhar seus webhooks — fica para projeto separado do time de frontend (`[09:39-09:40] Larissa`)
- Recebimento de webhooks de terceiros (inbound) — escopo é exclusivamente outbound, da plataforma para o cliente (`[09:02-09:03] Marcos, Sofia`)
- Escalonamento para múltiplos workers em paralelo com garantia de ordering global — arquitetura atual é single-worker; escalar exigiria decisão de particionamento futura, fora desta entrega (`[09:12-09:13] Diego`)

---

### Requisitos funcionais

#### FR-01 Notificação automática de mudança de status
Sempre que o status de um pedido muda com sucesso, o sistema deve notificar automaticamente todos os webhooks do cliente inscritos naquele status.

**Fluxo principal**
- Operador/sistema altera o status de um pedido via `PATCH /orders/:id/status`
- Dentro da mesma transação da mudança de status, o sistema identifica os webhooks do cliente inscritos no novo status
- Um evento é registrado de forma atômica, com o snapshot do pedido no momento da transição
- O worker de entrega, em ciclo de polling, envia o evento via HTTP assinado ao(s) endpoint(s) do cliente

**Fluxos alternativos e exceções**
- Nenhum webhook do cliente está inscrito no status alcançado: nenhum evento é gerado (`[09:34] Bruno, Diego`)
- A transação de mudança de status falha: nenhum evento é registrado (rollback atômico)

**Erros previstos**
- Payload do evento excede 64 KB: evento não é enviado, erro registrado (`[09:23-09:24] Sofia, Diego`)

**Prioridade:** alta

---

#### FR-02 Cadastro de webhook
O cliente (via API autenticada) deve poder cadastrar um novo webhook informando URL de destino e os status de pedido que deseja acompanhar.

**Fluxo principal**
- Requisição autenticada com `customerId`, `url` (HTTPS) e lista de status desejados
- Sistema gera uma secret exclusiva para o endpoint e a retorna apenas nesta resposta
- Webhook é persistido como ativo

**Fluxos alternativos e exceções**
- `customerId` informado não existe

**Erros previstos**
- URL cadastrada não é HTTPS: requisição rejeitada (`[09:23] Sofia`)

**Prioridade:** alta

---

#### FR-03 Edição de webhook
O cliente deve poder atualizar URL, lista de status inscritos ou status ativo/inativo de um webhook já cadastrado.

**Fluxo principal**
- Requisição autenticada informa o id do webhook e os campos a alterar
- Sistema valida e persiste as alterações

**Fluxos alternativos e exceções**
- Webhook informado não pertence a nenhum cadastro existente

**Erros previstos**
- Nova URL não é HTTPS: requisição rejeitada

**Prioridade:** média

---

#### FR-04 Remoção de webhook
O cliente deve poder remover um webhook cadastrado, interrompendo o recebimento de novas notificações.

**Fluxo principal**
- Requisição autenticada informa o id do webhook a remover
- Sistema remove o cadastro; eventos já enfileirados para esse webhook não são mais processados

**Fluxos alternativos e exceções**
- Webhook informado não existe

**Erros previstos**
- N/A (operação idempotente do ponto de vista de resultado final)

**Prioridade:** média

---

#### FR-05 Listagem de webhooks por cliente
O sistema deve permitir listar todos os webhooks cadastrados de um determinado cliente.

**Fluxo principal**
- Requisição autenticada informa o `customerId`
- Sistema retorna a lista paginada de webhooks daquele cliente

**Fluxos alternativos e exceções**
- Cliente sem nenhum webhook cadastrado: lista vazia

**Erros previstos**
- N/A

**Prioridade:** média

---

#### FR-06 Filtro de eventos por status
Cada webhook deve poder escolher quais status de pedido deseja acompanhar, evitando notificações irrelevantes.

**Fluxo principal**
- No cadastro ou edição do webhook, o cliente informa a lista de status de interesse
- O sistema filtra a geração do evento já no momento da inserção na fila de entrega, não enviando eventos para status não inscritos (`[09:33-09:34] Marcos, Bruno`)

**Fluxos alternativos e exceções**
- Lista de status vazia: nenhum evento é gerado para esse webhook até que ao menos um status seja configurado

**Erros previstos**
- N/A

**Prioridade:** alta

---

#### FR-07 Histórico de entregas
O cliente deve poder consultar o histórico das últimas entregas realizadas para um dos seus webhooks, incluindo sucesso/falha, payload, resposta recebida e tempo de resposta.

**Fluxo principal**
- Requisição autenticada informa o id do webhook
- Sistema retorna a lista paginada das últimas entregas realizadas (`[09:34] Marcos`)

**Fluxos alternativos e exceções**
- Webhook sem nenhuma entrega registrada ainda: lista vazia

**Erros previstos**
- Webhook informado não existe

**Prioridade:** média

---

#### FR-08 Retry automático e Dead Letter Queue
Quando uma entrega falha, o sistema deve tentar reentregar automaticamente com intervalos crescentes, movendo o evento para uma fila de falhas definitivas após esgotar as tentativas.

**Fluxo principal**
- Entrega falha (timeout de 10s ou erro HTTP)
- Sistema agenda nova tentativa seguindo backoff exponencial (1m, 5m, 30m, 2h, 12h)
- Após a 5ª falha consecutiva, o evento é movido para a Dead Letter Queue (DLQ), com motivo da falha preservado (`[09:15-09:18] Diego`)

**Fluxos alternativos e exceções**
- Cliente volta a responder com sucesso em qualquer tentativa intermediária: entrega é concluída e o ciclo de retry é encerrado

**Erros previstos**
- Timeout de 10 segundos sem resposta do cliente é tratado como falha (`[09:42] Diego`)

**Prioridade:** alta

---

#### FR-09 Replay manual de eventos em DLQ
Um operador `ADMIN` deve poder reprocessar manualmente um evento que foi movido para a DLQ.

**Fluxo principal**
- Operador `ADMIN` autenticado aciona o reprocessamento informando o id do evento na DLQ
- Sistema reinsere o evento na fila de entrega ativa, reiniciando o ciclo de tentativas
- Ação é registrada em log de auditoria com identificação do operador (`[09:18] Diego`, `[09:35-09:36] Larissa, Sofia`)

**Fluxos alternativos e exceções**
- Evento informado não existe na DLQ

**Erros previstos**
- Operador autenticado sem role `ADMIN`: acesso negado (`[09:36] Larissa, Sofia`)

**Prioridade:** média

---

#### FR-10 Rotação de secret com grace period
O cliente deve poder rotacionar a secret usada para validar a assinatura dos webhooks recebidos, sem período de indisponibilidade na verificação.

**Fluxo principal**
- Cliente solicita rotação de secret de um webhook via API
- Sistema gera nova secret e mantém a secret anterior válida em paralelo por 24 horas
- Ao final do grace period, a secret anterior é invalidada definitivamente (`[09:21-09:22] Sofia, Diego`)

**Fluxos alternativos e exceções**
- Cliente solicita nova rotação antes do grace period anterior expirar: comportamento a ser definido em detalhe técnico (ver FDD)

**Erros previstos**
- N/A

**Prioridade:** alta

---

### Requisitos não funcionais

Performance
- Latência de entrega de webhook: pior caso de até 2 segundos de espera pelo ciclo de polling do worker, mais tempo de rede, respeitando o teto de 10 segundos combinado com os clientes (`[09:02] Marcos`, `[09:09-09:10] Diego`)
- Timeout de 10 segundos por chamada HTTP de entrega ao cliente (`[09:42] Diego`)

Disponibilidade
- O worker de entrega roda como processo separado da API, de forma que reinícios/deploys da API não interrompam entregas em andamento (`[09:11] Diego, Larissa`)

Segurança e autorização
- Toda entrega de webhook é assinada com HMAC-SHA256 usando secret exclusiva por endpoint (`[09:19-09:21] Sofia`)
- URL de destino do webhook deve ser obrigatoriamente HTTPS (`[09:23] Sofia`)
- Endpoint de replay de DLQ exige autenticação e role `ADMIN`, com registro de auditoria de quem executou a ação (`[09:36] Sofia, Larissa`)
- Demais operações de CRUD de configuração de webhook exigem apenas autenticação (qualquer role), podendo ser restringidas futuramente (`[09:36-09:37] Sofia, Marcos`)

Observabilidade
- Logs estruturados (Pino, já padrão do projeto) para inserção de eventos, ciclos do worker, falhas de entrega e ações administrativas de replay (ver FDD, seção 8)

Confiabilidade e integridade de dados
- A captura do evento de notificação deve ser atômica com a mudança de status do pedido: nunca existe status alterado sem evento registrado, nem evento registrado sem a mudança de status ter sido commitada (`[09:04] Bruno`, `[09:06-09:08] Diego`, ADR-001)
- Garantia de entrega at-least-once, com `X-Event-Id` estável para deduplicação do lado do cliente (`[09:25] Diego`)

Compatibilidade e portabilidade
- Payload do evento limitado a 64 KB (`[09:23-09:24] Sofia, Diego`)
- Segue o padrão de API REST/JSON já usado no restante do OMS, sem introdução de novos formatos

Compliance
- Ação de replay manual de DLQ deve ficar registrada para fins de auditoria, identificando o operador responsável (`[09:36] Sofia`)

---

### Arquitetura e abordagem

Abordagem
- Padrão Transactional Outbox sobre o MySQL já utilizado pelo projeto: o evento é gravado dentro da mesma transação SQL da mudança de status, sem introdução de infraestrutura nova (fila externa, broker de mensagens) (`[09:06-09:08] Diego, Larissa`)
- Entrega assíncrona feita por um processo worker Node.js separado, em polling periódico sobre a tabela de eventos pendentes (`[09:09-09:11] Diego, Larissa`)

Componentes
- Módulo `webhooks` na API (cadastro, edição, remoção, listagem, histórico de entregas), seguindo a mesma estrutura `controller/service/repository` dos módulos existentes
- Tabela de outbox (eventos pendentes de entrega) e tabela de Dead Letter Queue (eventos com falha definitiva)
- Worker dedicado (processo Node.js separado) responsável pelo envio, retry e movimentação para DLQ

Integrações
- Chamadas HTTP de saída (outbound) para os endpoints HTTPS cadastrados pelos clientes B2B, autenticadas via HMAC-SHA256
- Nenhuma integração externa de infraestrutura nova (sem Redis, sem broker de mensagens) — decisão explícita para manter a operação simples para um time pequeno (`[09:07] Diego`)

### Decisões e trade-offs

#### Decisão: Outbox transacional no MySQL existente, em vez de fila externa (Redis Streams)
-**Justificativa:** garante atomicidade entre a mudança de status e o registro do evento sem exigir infraestrutura nova; time pequeno sem experiência operacional em Redis Cluster (`[09:06-09:08] Diego, Larissa`)
-**Trade-off:** o MySQL não é otimizado como broker de eventos de alto volume; pode exigir revisão se o volume de eventos crescer muito (ADR-001)

#### Decisão: Worker em processo separado, com polling de 2 segundos, em vez de reagir via trigger de banco
-**Justificativa:** MySQL não possui mecanismo nativo de notificação a processos externos; processo separado isola falhas entre API e entrega de webhooks (`[09:09-09:11] Diego, Larissa`)
-**Trade-off:** latência mínima de até 2 segundos entre a mudança de status e o início do envio; ordenação de entrega só é garantida com um único worker (`[09:12-09:13] Diego`)

#### Decisão: Retry com 5 tentativas e backoff exponencial (até 15h), com Dead Letter Queue
-**Justificativa:** cobre com folga indisponibilidades reais já enfrentadas por clientes (ex.: manutenção planejada de 2 horas), sem reter eventos indefinidamente (`[09:15-09:18] Diego`)
-**Trade-off:** eventos em backoff máximo podem chegar com até 15 horas de atraso; eventos que esgotam as tentativas exigem intervenção manual via replay (ADR-004)

#### Decisão: Autenticação HMAC-SHA256 com secret exclusiva por endpoint e rotação com grace period de 24h
-**Justificativa:** limita o impacto de vazamento de secret a um único cliente; evita janela de indisponibilidade durante troca de secret, já uma dor real vivida pela equipe (`[09:19-09:22] Sofia, Diego`)
-**Trade-off:** aumenta a complexidade de verificação no worker durante o grace period (checagem contra duas secrets) (ADR-005)

#### Decisão: Garantia de entrega at-least-once com deduplicação via `X-Event-Id`, em vez de exactly-once
-**Justificativa:** exactly-once exigiria coordenação bidirecional entre plataforma e cliente, inviável sem acoplamento forte; padrão amplamente adotado no mercado (Stripe, GitHub) (`[09:24-09:26] Diego`)
-**Trade-off:** transfere a responsabilidade de deduplicação para o lado do cliente, exigindo documentação clara no portal de desenvolvedor (`[09:26] Marcos`, ADR-006)

---

### Dependências

#### organizational: Documentação no portal de desenvolvedor
O time de produto (Marcos) precisa publicar documentação clara sobre a semântica at-least-once, a obrigatoriedade de deduplicação via `X-Event-Id` e o processo de integração com HMAC-SHA256, para orientar corretamente os clientes B2B (`[09:26] Marcos`)

#### organizational: Revisão de segurança dedicada antes do deploy
Sofia (Engenharia de Segurança) reservou ao menos dois dias úteis para revisar especificamente o código de geração/verificação de HMAC e de geração de secrets antes de qualquer deploy em produção (`[09:46] Sofia`)

#### technical: Confirmação de prazo com os clientes B2B
Marcos é responsável por confirmar o prazo (fim de novembro, meta da Atlas Comercial) com os três clientes após o planejamento de sprints ser fechado pela equipe técnica (`[09:45-09:47] Marcos, Larissa`)

---

### Riscos e mitigação

#### Cliente com alto volume de mudanças de status recebe muitas chamadas simultâneas do worker
-**Probabilidade:** média
-**Impacto:** sobrecarga do endpoint do cliente, aumento de falhas de entrega e retries desnecessários
-**Mitigação:**
  -Nenhuma ação de rate limiting nesta fase, decisão consciente da equipe (`[09:38-09:39] Diego, Larissa`)
  -Observar métricas de volume de entregas por cliente após o lançamento
-**Plano de contingência:** se o problema se materializar, avaliar rate limiting de saída por endpoint em fase futura

#### Vazamento de secret de um cliente
-**Probabilidade:** baixa, mas já ocorreu um incidente real similar reportado pela equipe (`[09:22] Diego`)
-**Impacto:** possibilidade de um ator malicioso forjar entregas para o endpoint daquele cliente específico até a rotação da secret
-**Mitigação:**
  -Secret exclusiva por endpoint, limitando o raio de impacto a um único cliente (ADR-005)
  -Rotação self-service com grace period de 24h, sem downtime de verificação
  -Revisão de segurança dedicada da Sofia antes do deploy (`[09:46] Sofia`)
-**Plano de contingência:** cliente aciona rotação de secret assim que suspeitar de vazamento

#### Crescimento não controlado da tabela de outbox ao longo do tempo
-**Probabilidade:** média
-**Impacto:** degradação progressiva de performance do polling do worker
-**Mitigação:**
  -Índices adequados nos campos de status e data de criação desde o início (ADR-003)
-**Plano de contingência:** implementar rotina de arquivamento de eventos entregues após 30 dias em iteração futura, fora do escopo desta entrega (`[09:08] Diego`)

#### Cliente não implementa corretamente a deduplicação de eventos
-**Probabilidade:** média
-**Impacto:** processamento duplicado de eventos no sistema do cliente, com possíveis consequências operacionais do lado dele
-**Mitigação:**
  -Documentação clara e destacada no portal de desenvolvedor sobre a semântica at-least-once (`[09:26] Marcos`)
  -Alinhamento com padrão de mercado já conhecido por integradores B2B (Stripe, GitHub) (`[09:25] Diego`)
-**Plano de contingência:** suporte ao cliente durante o onboarding para validar a implementação da deduplicação

---

### Critérios de aceitação

- Uma mudança de status de pedido gera um evento de webhook para todo cliente inscrito naquele status, e nenhum evento é gerado quando a transação de mudança de status falha
- Um evento é entregue ao endpoint do cliente em até 10 segundos quando o endpoint está disponível
- Uma falha de entrega é retentada seguindo os intervalos de 1m/5m/30m/2h/12h antes de ser movida para a Dead Letter Queue
- Toda entrega inclui assinatura HMAC-SHA256 válida, verificável pelo cliente com a secret do seu endpoint
- Durante o grace period de 24h após rotação de secret, entregas assinadas com a secret antiga e com a nova são ambas aceitas
- Todo evento entregue, inclusive em reentregas, carrega o mesmo `X-Event-Id`
- Um operador sem role `ADMIN` não consegue acionar o replay de eventos em DLQ
- Um cliente cadastrando um webhook com URL não-HTTPS recebe erro de validação
- Um cliente consegue consultar o histórico das últimas entregas do seu webhook, incluindo status de sucesso/falha e tempo de resposta

---

### Testes e validação

Tipos de teste obrigatórios
- Testes unitários para a lógica de cálculo de backoff, filtragem de eventos por status e geração/verificação de assinatura HMAC
- Testes de integração para o fluxo completo de `changeStatus` → inserção na outbox → processamento pelo worker → entrega
- Testes de integração para o CRUD de configuração de webhook e para o endpoint de replay de DLQ, incluindo controle de acesso por role
- Testes de segurança dedicados para geração de secret, verificação de HMAC e comportamento durante o grace period de rotação (revisão de Sofia, `[09:46] Sofia`)

Estratégia de validação
- Testes automatizados (unitários e de integração) cobrindo os fluxos principal e de exceção antes do deploy
- Revisão de segurança manual da Sofia sobre o código de HMAC e geração de secrets, reservada como etapa formal antes do deploy em produção (`[09:46] Sofia`)
