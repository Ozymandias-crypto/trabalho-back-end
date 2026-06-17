# Eventus API

API REST em FastAPI para uma plataforma de eventos com locais, eventos, tipos de ingresso, participantes, inscricoes, pagamentos e auditoria.

O dominio foi escolhido porque combina ciclo de vida, capacidade limitada, regras dependentes de estado e calculos derivados. A API evita que um evento venda mais vagas que sua capacidade, controla inscricoes duplicadas, calcula o total da inscricao a partir do tipo de ingresso, registra pagamento e aplica cancelamento com reembolso.

## Como rodar

```bash
docker compose up --build
```

A API sobe em `http://localhost:8000`.

Documentacao interativa:

```text
http://localhost:8000/docs
```

Rodar testes dentro do container:

```bash
docker compose run api pytest
```

Rodar migrations manualmente:

```bash
docker compose run api alembic upgrade head
docker compose run api alembic downgrade 202606170002
docker compose run api alembic upgrade head
```

## Diagrama ER

```text
VENUES 1 --- N EVENTS 1 --- N TICKET_TYPES
                  |
                  | 1
                  |
                  N
             REGISTRATIONS N --- 1 ATTENDEES
                  |
                  | 1
                  |
                  1
              PAYMENTS

AUDIT_LOGS registra historico por entity_type + entity_id.
```

## Entidades

### Venue

Local fisico onde eventos acontecem.

| Campo | Tipo | Obrigatorio | Constraints |
| --- | --- | --- | --- |
| id | int | sim | chave primaria |
| name | string | sim | 3 a 120 caracteres |
| address | string | sim | 5 a 255 caracteres |
| capacity | int | sim | maior que zero |
| is_active | bool | sim | padrao true |
| created_at | datetime | sim | gerado pelo banco |

Relacionamento: um local possui muitos eventos. A capacidade do local limita a capacidade maxima de cada evento.

### Event

Evento publicado na plataforma.

| Campo | Tipo | Obrigatorio | Constraints |
| --- | --- | --- | --- |
| id | int | sim | chave primaria |
| venue_id | int | sim | FK para Venue |
| title | string | sim | 3 a 160 caracteres |
| description | text | nao | ate 3000 caracteres no schema |
| starts_at | datetime | sim | deve ser futuro na criacao |
| ends_at | datetime | sim | maior que starts_at |
| capacity_override | int | nao | maior que zero e menor/igual ao local |
| status | enum | sim | draft, published, canceled, finished |
| version | int | sim | maior que zero, usado para concorrencia |
| created_at | datetime | sim | gerado pelo banco |

Relacionamentos: pertence a um local, possui muitos tipos de ingresso e muitas inscricoes.

### TicketType

Tipo de ingresso vendido para um evento.

| Campo | Tipo | Obrigatorio | Constraints |
| --- | --- | --- | --- |
| id | int | sim | chave primaria |
| event_id | int | sim | FK para Event |
| name | string | sim | unico por evento |
| price_cents | int | sim | maior ou igual a zero |
| capacity | int | sim | maior que zero |
| sales_start | datetime | sim | inicio da janela de venda |
| sales_end | datetime | sim | maior que sales_start e antes do evento |
| is_active | bool | sim | padrao true |
| created_at | datetime | sim | gerado pelo banco |

Relacionamento: um evento possui varios tipos de ingresso. A soma das capacidades dos ingressos ativos nao pode passar a capacidade do evento.

### Attendee

Pessoa que pode se inscrever em eventos.

| Campo | Tipo | Obrigatorio | Constraints |
| --- | --- | --- | --- |
| id | int | sim | chave primaria |
| name | string | sim | 3 a 120 caracteres |
| email | email | sim | unico, normalizado para minusculo |
| document | string | nao | 5 a 40 caracteres |
| created_at | datetime | sim | gerado pelo banco |

Relacionamento: um participante pode ter inscricoes em eventos diferentes, mas nao pode ter duas inscricoes para o mesmo evento.

### Registration

Inscricao de um participante em um evento usando um tipo de ingresso.

| Campo | Tipo | Obrigatorio | Constraints |
| --- | --- | --- | --- |
| id | int | sim | chave primaria |
| event_id | int | sim | FK para Event |
| attendee_id | int | sim | FK para Attendee |
| ticket_type_id | int | sim | FK para TicketType |
| status | enum | sim | pending_payment, confirmed, checked_in, canceled |
| total_price_cents | int | sim | calculado a partir do ingresso |
| checked_in_at | datetime | nao | preenchido no check-in |
| canceled_at | datetime | nao | preenchido no cancelamento |
| created_at | datetime | sim | gerado pelo banco |
| updated_at | datetime | sim | atualizado pelo banco |

Relacionamentos: une Event, Attendee e TicketType. Possui um Payment.

### Payment

Registro de pagamento da inscricao.

| Campo | Tipo | Obrigatorio | Constraints |
| --- | --- | --- | --- |
| id | int | sim | chave primaria |
| registration_id | int | sim | unico, FK para Registration |
| amount_cents | int | sim | maior ou igual a zero |
| status | enum | sim | pending, paid, failed, refunded |
| provider_reference | string | nao | unico quando presente |
| paid_at | datetime | nao | preenchido ao confirmar |
| refunded_at | datetime | nao | preenchido ao reembolsar |
| created_at | datetime | sim | gerado pelo banco |

Relacionamento: pagamento 1-para-1 com inscricao.

### AuditLog

Historico de alteracoes relevantes.

| Campo | Tipo | Obrigatorio | Constraints |
| --- | --- | --- | --- |
| id | int | sim | chave primaria |
| entity_type | string | sim | indexado |
| entity_id | int | sim | indexado |
| action | string | sim | nome da acao |
| metadata | json | sim | detalhes contextuais |
| created_at | datetime | sim | gerado pelo banco |

## Maquinas de estado

Evento:

```text
draft -> published -> finished
   |         |
   v         v
canceled  canceled
```

Estados terminais: `canceled` e `finished`. Nao faz sentido voltar de `canceled`, porque ingressos e inscricoes podem ter sido cancelados e reembolsados. Nao faz sentido voltar de `finished`, porque o evento ja aconteceu.

Inscricao:

```text
pending_payment -> confirmed -> checked_in
       |              |
       v              v
    canceled       canceled
```

Estados terminais: `checked_in` e `canceled`. Uma inscricao com check-in representa presenca efetivada; uma cancelada libera a vaga e encerra o fluxo.

Pagamento:

```text
pending -> paid -> refunded
   |
   v
 failed
```

Estados terminais: `failed` e `refunded`. `paid` deixa de ser terminal porque pode ser reembolsado em cancelamentos.

## Regras de negocio implementadas

| ID | Regra | Onde esta | Erro |
| --- | --- | --- | --- |
| RN01 | Evento so pode ser criado em local ativo, no futuro, e com capacidade menor ou igual a do local. | `EventService.create` | `VENUE_INACTIVE`, `EVENT_START_IN_PAST`, `EVENT_CAPACITY_EXCEEDS_VENUE` |
| RN02 | Evento so pode ser publicado se estiver em rascunho, possuir local ativo e pelo menos um ingresso ativo. | `EventService.publish` | `INVALID_EVENT_TRANSITION`, `EVENT_WITHOUT_TICKETS` |
| RN03 | Tipos de ingresso so podem ser criados enquanto o evento esta em rascunho; a venda termina antes do inicio do evento. | `TicketTypeService.create` | `TICKET_EVENT_LOCKED`, `INVALID_SALES_WINDOW` |
| RN04 | A soma da capacidade dos ingressos ativos nao pode ultrapassar a capacidade do evento. | `TicketTypeService.create` e `EventService.publish` | `TICKET_CAPACITY_EXCEEDS_EVENT` |
| RN05 | Inscricao so e permitida em evento publicado, com ingresso ativo, dentro da janela de venda e com vagas disponiveis. | `RegistrationService.create` | `EVENT_NOT_AVAILABLE`, `OUTSIDE_SALES_WINDOW`, `EVENT_CAPACITY_EXHAUSTED`, `TICKET_TYPE_SOLD_OUT` |
| RN06 | Um participante nao pode ter duas inscricoes para o mesmo evento. | `RegistrationService.create` | `DUPLICATE_REGISTRATION` |
| RN07 | O total da inscricao e derivado do preco atual do tipo de ingresso no momento da inscricao. | `RegistrationService.create` | calculo `total_price_cents` |
| RN08 | Confirmar pagamento so e permitido para inscricao em `pending_payment`, com referencia de provedor unica. | `RegistrationService.confirm_payment` | `REGISTRATION_NOT_PENDING_PAYMENT`, `PAYMENT_REFERENCE_ALREADY_USED` |
| RN09 | Cancelar inscricao confirmada reembolsa pagamento pago; cancelar inscricao pendente falha o pagamento pendente. | `RegistrationService.cancel` | `REGISTRATION_TERMINAL_STATE` |
| RN10 | Check-in so e permitido para inscricoes confirmadas de eventos publicados. | `RegistrationService.check_in` | `REGISTRATION_NOT_CONFIRMED`, `EVENT_NOT_AVAILABLE_FOR_CHECKIN` |
| RN11 | Cancelar evento cancela inscricoes ativas e reembolsa pagamentos pagos, mas recusa cancelamento se ja houver check-in. | `EventService.cancel` | `EVENT_TERMINAL_STATE`, `EVENT_HAS_CHECKED_IN_REGISTRATIONS` |

Inscricoes em `pending_payment` tambem reservam vaga. Essa decisao evita venda duplicada da ultima vaga enquanto o pagamento esta em processamento. Em uma versao de producao, seria natural adicionar expiracao automatica para reservas pendentes.

## Cenários de borda tratados

1. Recurso limitado chega a zero: `EVENT_CAPACITY_EXHAUSTED` ou `TICKET_TYPE_SOLD_OUT`.
2. Modificar entidade em estado terminal: inscricoes `checked_in` ou `canceled` nao podem ser canceladas novamente; eventos `canceled` ou `finished` nao mudam de estado.
3. Datas invalidas ou sobrepostas: evento exige `ends_at > starts_at`; ingresso exige `sales_end > sales_start` e venda encerrando antes do evento.
4. Entidade pai com filhos ativos: cancelar evento percorre inscricoes ativas, cancela cada uma e reembolsa pagamentos pagos.
5. Calculo derivado invalido: totais e valores monetarios possuem constraint de nao-negatividade.

## Decisoes de design

### Relacionamentos

`Registration` foi modelada como entidade propria, e nao apenas uma tabela associativa, porque possui estado, preco congelado, datas de check-in/cancelamento e pagamento. Isso permite regras como "inscricao confirmada pode fazer check-in" sem misturar responsabilidades com `Attendee` ou `Event`.

`Payment` ficou separado de `Registration` porque pagamento tem ciclo de vida proprio. Uma inscricao pode estar cancelada enquanto o pagamento registra se foi falhado ou reembolsado.

`AuditLog` usa `entity_type` e `entity_id` para auditar varias entidades sem criar uma tabela de historico para cada uma. A migration 3 adiciona essa tabela quando o dominio passa a exigir rastreabilidade de transicoes.

### Validators vs services

Validators Pydantic cuidam de regras locais ao payload: formato de email, normalizacao de datas e comparacoes dentro do proprio corpo da requisicao, como `ends_at > starts_at`.

Services cuidam de regras que dependem do banco ou do estado de outras entidades: capacidade disponivel, evento publicado, local ativo, duplicidade de inscricao, referencia de pagamento ja usada e reembolso.

### Migration 2

A segunda migration adiciona `version` em `events` e indices de consulta. A coluna `version` permite argumentar e evoluir para controle otimista de concorrencia quando dois usuarios tentam alterar o mesmo evento. Os indices atendem consultas frequentes: listar eventos por status/data e buscar ingressos ativos de um evento.

### Concorrencia

As operacoes sensiveis usam `SELECT ... FOR UPDATE` via SQLAlchemy em evento, tipo de ingresso e inscricao. Em PostgreSQL isso reduz race conditions no consumo de vagas: duas inscricoes simultaneas para a ultima vaga precisam esperar a validacao uma da outra. Em uma evolucao com maior escala, a API poderia combinar isso com controle otimista usando `events.version` e retry transacional.

## Migrations

1. `202606170001_initial_structure`: cria entidades principais e constraints iniciais.
2. `202606170002_add_version_and_indexes`: adiciona `events.version` e indices motivados por concorrencia e consultas.
3. `202606170003_add_audit_log`: adiciona historico/auditoria das transicoes importantes.

Cada migration possui `downgrade`.

## Exemplos rapidos

Criar local:

```bash
curl -X POST http://localhost:8000/venues \
  -H "Content-Type: application/json" \
  -d '{"name":"Centro de Convencoes","address":"Av. Central, 100","capacity":100}'
```

Criar evento:

```bash
curl -X POST http://localhost:8000/events \
  -H "Content-Type: application/json" \
  -d '{"venue_id":1,"title":"FastAPI Summit","starts_at":"2030-05-10T10:00:00Z","ends_at":"2030-05-10T18:00:00Z","capacity_override":80}'
```

Ver disponibilidade:

```bash
curl http://localhost:8000/events/1/availability
```
