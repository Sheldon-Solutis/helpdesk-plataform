# HelpDesk Platform

Plataforma de gestão de chamados de suporte técnico construída como
desafio de proficiência Full Stack: arquitetura de microsserviços em
Java/Spring Boot, mensageria assíncrona com RabbitMQ, API Gateway como
porta única de entrada e frontend em React consumindo tudo através dele.

Projeto organizado em polirepo — seis repositórios independentes, cada
um com seu próprio pipeline de CI/CD, publicando imagem Docker no GitHub
Container Registry:

| Repositório                       | Responsabilidade                              |
|------------------------------------|------------------------------------------------|
| `helpdesk-api-gateway`             | Porta única de entrada, roteamento e CORS      |
| `helpdesk-user-service`            | Cadastro e gestão de usuários                  |
| `helpdesk-ticket-service`          | Chamados de suporte e suas regras de negócio   |
| `helpdesk-notification-service`    | Consumo de eventos e histórico de notificações |
| `helpdesk-frontend`                | Interface web em React                         |
| `helpdesk-platform`                | Orquestração (Docker Compose, sem código próprio) — **você está aqui** |

---

## 1. Descrição do sistema

Um cliente abre um chamado descrevendo um problema (categoria,
prioridade, título e descrição). O chamado nasce com status `OPEN` e
pode ser atribuído a um técnico, ter sua prioridade/categoria/descrição
alteradas, e evoluir por um fluxo de status controlado até ser encerrado.
Cada mudança relevante no chamado gera um evento assíncrono que o
serviço de notificações consome para manter um histórico consultável —
sem acoplar o fluxo de criação/atualização de chamados à disponibilidade
do serviço de notificações.

Toda a comunicação do frontend com o backend passa por um API Gateway
único; nenhum microsserviço é acessado diretamente pelo navegador.

## 2. Arquitetura

```
                          ┌──────────────┐
   React (5173) ────────► │ API Gateway  │  (8080 · Spring Cloud Gateway/WebMVC)
                          └──────┬───────┘
              ┌───────────────────┼────────────────────┐
              ▼                   ▼                     ▼
       user-service        ticket-service      notification-service
          (8081)               (8082)                 (8083)
              ▲                   │                     ▲
              │  valida customerId│                     │
              └───────────────────┤   publica eventos   │
                                   ▼                     │
                              RabbitMQ ────────────────────┘
                        (helpdesk.exchange, topic)
              │                   │                     │
              ▼                   ▼                     ▼
          users_db            tickets_db          notifications_db
                        (um banco PostgreSQL por serviço)
```

### Responsabilidades por serviço

**`api-gateway`** — único ponto de entrada do frontend. Spring Cloud
Gateway na variante WebMVC (servlet, não reativa): rotas `/api/users/**`,
`/api/tickets/**` e `/api/notifications/**` repassadas como proxy puro,
sem reescrita de path (cada serviço já expõe seus controllers em
`/api/...`). Também concentra a configuração de CORS, liberada apenas
para a origem do frontend.

**`user-service`** — dono dos dados de usuário. Não conhece chamados nem
notificações.

**`ticket-service`** — dono dos dados de chamado e das regras de negócio
em torno deles. É o único serviço que fala com outro serviço via HTTP
direto (fora do gateway): antes de criar ou atribuir um chamado, consulta
o `user-service` para confirmar que `customerId`/`technicianId`
correspondem a um usuário existente e ativo. Publica eventos no RabbitMQ
a cada mudança relevante.

**`notification-service`** — consome os eventos publicados pelo
`ticket-service` e mantém o histórico de notificações. Não tem
conhecimento nenhum sobre `ticket-service` além do formato dos eventos
que recebe — se cair, não impede a criação ou atualização de chamados,
só atrasa o registro do histórico.

**`frontend`** — React consumindo exclusivamente o gateway.

### Isolamento de dados

Cada microsserviço tem seu próprio banco PostgreSQL (`users_db`,
`tickets_db`, `notifications_db`) e sua própria variável de conexão.
Nenhum serviço acessa o banco de outro — referências entre entidades de
serviços diferentes (`customerId`, `technicianId` no chamado) são
`Long` simples, validados via chamada HTTP quando necessário, nunca via
chave estrangeira entre bancos.

## 3. Tecnologias utilizadas

| Camada        | Tecnologias |
|---------------|-------------|
| Backend       | Java 25 · Spring Boot 4.1 · Spring Data JPA · Spring Validation (Bean Validation) · Spring AMQP |
| Gateway       | Spring Cloud Gateway (servidor WebMVC/servlet) |
| Mensageria    | RabbitMQ · exchange topic única · fila roteada por tipo de evento |
| Dados         | PostgreSQL 15 |
| Frontend      | React · JavaScript · Vite |
| Documentação  | springdoc-openapi (Swagger UI) |
| Testes        | JUnit 5 · Mockito · Spring Boot Test (`@WebMvcTest`) |
| Infraestrutura| Docker · Docker Compose · GitHub Actions · GitHub Container Registry (GHCR) |

## 4. Pré-requisitos

Pra rodar via Docker (caminho recomendado): **Docker** e **Docker
Compose** (plugin v2 — confirme com `docker compose version`).

Pra rodar sem Docker (desenvolvimento local de um serviço específico):
**JDK 25** e **Node.js 20+** (para o frontend), além de um PostgreSQL e
um RabbitMQ acessíveis (locais ou via `docker compose up postgres
rabbitmq`).

## 5. Instruções de execução

### Caminho A — imagens publicadas (produção/demo)

```bash
git clone <url-do-helpdesk-platform>
cd helpdesk-platform
cp .env.example .env
# edite .env: GITHUB_OWNER=<seu usuário/org do GitHub>
docker compose up -d
```

Isso sobe os 7 containers (Postgres, RabbitMQ e os 5 serviços de
aplicação) puxando as imagens já publicadas no GHCR — nenhum build
acontece nesta etapa. Respeitando os healthchecks configurados, a ordem
de subida é: Postgres/RabbitMQ → `user-service` → `ticket-service` /
`notification-service` → `api-gateway` → `frontend`.

### Caminho B — build local a partir do código-fonte

Com os 5 repositórios de serviço clonados como irmãos deste:

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml up --build
```

### Caminho C — sem Docker, serviço a serviço

```bash
docker compose up postgres rabbitmq -d   # só a infra
cd helpdesk-user-service && ./mvnw spring-boot:run
cd helpdesk-ticket-service && ./mvnw spring-boot:run
cd helpdesk-notification-service && ./mvnw spring-boot:run
cd helpdesk-api-gateway && ./mvnw spring-boot:run
cd helpdesk-frontend && npm install && npm run dev -- --port 5173
```

### Portas e URLs

| Serviço                | Porta | URL |
|-------------------------|-------|-----|
| Frontend (React)        | 5173  | http://localhost:5173 |
| API Gateway             | 8080  | http://localhost:8080/api |
| user-service            | 8081  | http://localhost:8081/swagger-ui.html |
| ticket-service          | 8082  | http://localhost:8082/swagger-ui.html |
| notification-service    | 8083  | http://localhost:8083/swagger-ui.html |
| PostgreSQL               | 5432  | — |
| RabbitMQ (management)   | 15672 | http://localhost:15672 |

Pra derrubar: `docker compose down` (mantém os dados) ou `docker compose
down -v` (apaga os volumes — necessário se o banco tiver sido inicializado
de forma incompleta numa tentativa anterior).

## 6. Principais endpoints

Todas as rotas abaixo são acessadas através do gateway, na forma
`http://localhost:8080/api/<recurso>/...`.

### Usuários (`user-service`)

| Método | Rota | Descrição |
|--------|------|-----------|
| `POST`   | `/api/users` | Cria usuário (`name`, `email`, `role`) — `201` |
| `GET`    | `/api/users` | Lista usuários **ativos** — `200` |
| `GET`    | `/api/users/{id}` | Busca por id — `200` / `404` |
| `PATCH`  | `/api/users/{id}` | Atualiza `name`/`email` — `200` / `404` / `409` (e-mail já em uso) |
| `DELETE` | `/api/users/{id}` | Inativa (não remove o registro) — `204` |
| `POST`   | `/api/users/{id}` | Reativa um usuário inativo — `200` |

### Chamados (`ticket-service`)

| Método | Rota | Descrição |
|--------|------|-----------|
| `POST`   | `/api/tickets` | Cria chamado. **`customerId` vem do header `Customer-Id`, não do corpo.** Nasce sempre `OPEN` — `201` |
| `GET`    | `/api/tickets` | Lista todos |
| `GET`    | `/api/tickets/{id}` | Detalha um chamado — `404` se não existir |
| `GET`    | `/api/tickets/filter?status=&category=&priority=` | Filtra chamados |
| `GET`    | `/api/tickets/search?word=` | Busca por título/descrição |
| `GET`    | `/api/tickets/customer?id=` | Chamados de um cliente |
| `PUT`    | `/api/tickets/{id}` | Atualiza `description`/`category`/`priority`/`status` |
| `PATCH`  | `/api/tickets/{id}/status={status}` | Atalho dedicado pra mudança de status |
| `PATCH`  | `/api/tickets/{id}/technician={id}` | Atribui/troca técnico |
| `DELETE` | `/api/tickets/{id}` | Encerra o chamado |

### Notificações (`notification-service`)

| Método | Rota | Descrição |
|--------|------|-----------|
| `GET` | `/api/notifications` | Histórico completo |
| `GET` | `/api/notifications/{id}` | Uma notificação — `404` se não existir |

Contrato completo, testável interativamente, em `/swagger-ui.html` de
cada serviço.

## 7. Eventos RabbitMQ

`ticket-service` publica no exchange **`helpdesk.exchange`** (topic).
`notification-service` consome de uma **fila única** (`ticket.queue`),
roteada internamente por tipo de evento através de `@RabbitHandler` — um
método por tipo de payload, o Spring decide qual chamar pelo tipo Java
inferido.

| Evento                     | Routing key           | Quando |
|------------------------------|------------------------|--------|
| `TicketCreatedEvent`         | `ticket.created`       | Ao abrir um chamado |
| `TicketAssignedEvent`        | `ticket.assigned`      | Ao atribuir/trocar o técnico |
| `TicketStatusChangedEvent`   | `ticket.changed`       | Ao transicionar o status |
| `TicketDeletedEvent`         | `ticket.deleted`       | Ao encerrar o chamado |

Cada evento é publicado **depois do commit da transação** no banco
(`TransactionSynchronizationManager.registerSynchronization(...).afterCommit(...)`)
— garante que nunca existe uma notificação referente a uma mudança que
não foi persistida de verdade.

## 8. Estratégia de persistência

- JPA/Hibernate com `ddl-auto=update` em todos os serviços — o schema é
  criado/atualizado automaticamente a partir das entidades no boot.
- Um banco PostgreSQL por serviço, cada um com sua própria variável de
  conexão (`DATABASE_URL`/`DATABASE_USER`/`DATABASE_PASSWORD`) — sem
  acesso cruzado.
- Relacionamentos entre entidades de serviços diferentes (`customerId`,
  `technicianId`, `ticketId` na notificação) são armazenados como `Long`
  simples, **não** como chave estrangeira — não é possível ter FK
  cruzando bancos diferentes numa arquitetura de microsserviços, então a
  consistência é garantida na camada de aplicação (validação via HTTP
  contra o `user-service` antes de gravar).
- Bean Validation (`@NotBlank`, `@Email`, `@NotNull`) nos DTOs de entrada
  garante que dados obrigatórios/malformados nunca cheguem à camada de
  persistência.

## 9. Principais decisões arquiteturais

- **Spring Cloud Gateway na variante WebMVC (servlet), não reativa** —
  simplifica o stack ao não introduzir Project Reactor/WebFlux só no
  gateway enquanto todo o resto do backend é blocking/servlet.
- **Fila única + `@RabbitHandler` por tipo, em vez de uma fila por
  evento** — menos infraestrutura de mensageria pra manter, com o
  roteamento por tipo resolvido pelo próprio Spring AMQP a partir do
  payload.
- **`customerId` via header (`Customer-Id`), não no corpo da
  requisição** — evita que o cliente possa "forjar" um chamado em nome
  de outro usuário só editando o JSON enviado.
- **Transição de status controlada no `TicketService`** — uma vez
  `CLOSED`, o chamado não aceita mais mudança de status; `RESOLVED` só
  pode avançar para `CLOSED`. A regra fica centralizada num único método
  (`validateStatusTransition`), reaproveitado tanto pelo `PATCH .../status={x}`
  quanto pelo `PUT` genérico.
- **Validação cruzada síncrona via HTTP, com falha graciosa** — o
  `ticket-service` chama o `user-service` diretamente (fora do gateway)
  pra validar `customerId`/`technicianId`; se o `user-service` estiver
  fora do ar, o `ticket-service` responde `503` de forma controlada em
  vez de propagar uma exceção de conexão crua.
- **Inativação lógica de usuário, não remoção física** — `DELETE
  /api/users/{id}` marca `active=false` e preserva o histórico, em vez
  de apagar o registro.
- **Multi-repo com imagem Docker publicada por serviço** — cada um dos 5
  repositórios de código tem seu próprio pipeline (GitHub Actions →
  GHCR), publicado a cada push na `main`. O `helpdesk-platform` não
  builda nada: só orquestra `docker compose pull && up` contra as
  imagens já publicadas, com um `docker-compose.dev.yml` separado para
  quando se quer buildar a partir do código-fonte local.

## Testes

```bash
cd helpdesk-user-service && ./mvnw test
cd helpdesk-ticket-service && ./mvnw test
cd helpdesk-notification-service && ./mvnw test
cd helpdesk-api-gateway && ./mvnw test
```

Cobertura atual: regras de negócio do `TicketService` (status sempre
nasce `OPEN`, transições controladas, validação cruzada com o
`user-service`, publicação de evento por mudança relevante), do
`UserService` (unicidade de e-mail, inativar/reativar) e do
`NotificationService` (os 4 handlers de evento); testes dos endpoints
principais via `@WebMvcTest`/MockMvc nos 3 serviços com API própria; e um
teste de configuração do CORS no gateway. `Postgres`/`RabbitMQ` reais
**não** são necessários pra rodar esses testes — só o `contextLoads()`
padrão do Spring Boot (que já vinha do Initializr) exige a infra de pé.

## Decisões conhecidas em aberto

Documentado aqui de propósito, em vez de escondido: dois comportamentos
atuais são decisões conscientes, não bugs, e valem discussão em uma
eventual apresentação técnica.

- **`GET /api/tickets/filter`** aplica um único critério por vez, na
  ordem `status` → `category` → `priority`, em vez de combinar os três
  com `AND`. Evolução natural: reescrever com `Specification`
  (`JpaSpecificationExecutor`).
- **`DELETE /api/tickets/{id}`** hoje remove o registro fisicamente. Uma
  leitura alternativa (e mais consistente com o `DELETE /api/users/{id}`,
  que inativa em vez de apagar) seria tratá-lo como encerramento lógico
  (`status = CLOSED`, registro preservado).