# Banking Mini Core System

Spring Boot backend for customer accounts, deposits/withdrawals, fund transfers,
audit logging, and statement/balance reporting.

## Tech stack
- Java 17, Spring Boot 3.5.6
- Spring Data JPA + Hibernate, H2 (in-memory)
- Spring AOP (centralized logging via `LoggingAspect`)
- springdoc-openapi (Swagger UI)
- Lombok

## Run it

```bash
mvn spring-boot:run
```

App starts on **`http://localhost:8082`**.

| Tool | URL |
|---|---|
| Swagger UI | `http://localhost:8082/swagger-ui.html` |
| OpenAPI spec (JSON) | `http://localhost:8082/v3/api-docs` |
| H2 console | `http://localhost:8082/h2-console` |
| H2 JDBC URL | `jdbc:h2:mem:bankingdb` (user: `bank_db`, no password) |

### Seeding sample data
To start with pre-populated test data instead of an empty database, add `data.sql`
to `src/main/resources` and set in `application.properties`:
```properties
spring.jpa.defer-datasource-initialization=true
spring.sql.init.mode=always
```

## Layers

```
entity/                 -> Customer, Account, Transaction, AuditLog + BaseEntity (createdAt/updatedAt)
dto/request_dto/         -> CustomerRequestDTO, AccountRequestDTO, DepositRequestDTO, WithdrawRequestDTO, TransferRequestDTO
dto/response_dto/        -> CustomerResponseDTO, AccountResponseDTO, TransactionResponseDTO, StatementEntryDTO, BalanceSummaryDTO
repository/               -> Spring Data JPA interfaces (pessimistic-lock query used for transfers)
service_implementation/  -> business logic: deposit, withdraw, transfer, statements, balance summary
controller/               -> REST endpoints
exception/                -> ResourceNotFoundException, InsufficientBalanceException, InvalidTransactionException, GlobalExceptionHandler
aspect/                   -> LoggingAspect (AOP) - logs entry/exit/exceptions for every service + controller call
```

## API reference

### Customers
| Method | Path | Body |
|---|---|---|
| POST | `/api/customers` | `{ "firstName", "lastName", "email", "phone" }` |
| GET | `/api/customers` | - |
| GET | `/api/customers/{customerId}` | - |
| DELETE | `/api/customers/{customerId}` | - |

### Accounts
| Method | Path | Body |
|---|---|---|
| POST | `/api/accounts` | `{ "customerId", "accountType": "SAVINGS\|CURRENT", "openingBalance" }` |
| GET | `/api/accounts/{accountId}` | - |
| GET | `/api/accounts/customer/{customerId}` | - |
| DELETE | `/api/accounts/{accountId}` | - |

### Transactions
| Method | Path | Body |
|---|---|---|
| POST | `/api/transactions/deposit` | `{ "accountId", "amount" }` |
| POST | `/api/transactions/withdraw` | `{ "accountId", "amount" }` |
| POST | `/api/transactions/transfer` | `{ "fromAccountId", "toAccountId", "amount" }` |

### Reporting
| Method | Path | Query params |
|---|---|---|
| GET | `/api/accounts/{accountId}/statement` | `from=YYYY-MM-DD&to=YYYY-MM-DD` |
| GET | `/api/accounts/{accountId}/balance-summary` | `from=YYYY-MM-DD&to=YYYY-MM-DD` |

## Sample flow (curl)

```bash
# 1. Create a customer
curl -X POST localhost:8082/api/customers -H "Content-Type: application/json" \
  -d '{"firstName":"Asha","lastName":"Rao","email":"asha.rao@example.com","phone":"9000000001"}'

# 2. Open a savings account for that customer (use the returned customerId)
curl -X POST localhost:8082/api/accounts -H "Content-Type: application/json" \
  -d '{"customerId":1,"accountType":"SAVINGS","openingBalance":5000}'

# 3. Deposit
curl -X POST localhost:8082/api/transactions/deposit -H "Content-Type: application/json" \
  -d '{"accountId":1,"amount":2000}'

# 4. Withdraw
curl -X POST localhost:8082/api/transactions/withdraw -H "Content-Type: application/json" \
  -d '{"accountId":1,"amount":1500}'

# 5. Transfer between two accounts
curl -X POST localhost:8082/api/transactions/transfer -H "Content-Type: application/json" \
  -d '{"fromAccountId":1,"toAccountId":2,"amount":1000}'

# 6. Statement
curl "localhost:8082/api/accounts/1/statement?from=2026-06-01&to=2026-06-30"

# 7. Balance summary
curl "localhost:8082/api/accounts/1/balance-summary?from=2026-06-01&to=2026-06-30"
```

## Design notes
- **Thread-safe transfers** — accounts are locked with a pessimistic DB lock
  (`SELECT ... FOR UPDATE`) inside `transfer()`, always acquiring the lock on the
  *lower* account id first so two concurrent transfers between the same pair of
  accounts can never deadlock each other.
- **Overdraft prevention** — balance is checked before every withdrawal/transfer
  debit; insufficient funds throw `InsufficientBalanceException` (HTTP 400).
- **ACID compliance** — every money-moving method is `@Transactional`; a
  transfer's debit, credit, and both transaction/audit-log inserts commit
  together or roll back together.
- **Transfer pairing** — a transfer writes two `Transaction` rows
  (`TRANSFER_OUT` / `TRANSFER_IN`), tagged with a shared `transfer_ref_id` so
  they can always be matched back up, since each transaction row only belongs
  to one account.
- **Centralized logging** — `LoggingAspect` wraps every service- and
  controller-layer method with entry/exit/exception logging via AOP, instead
  of scattering log statements through each class.
- **Resource-not-found handling** — a single `ResourceNotFoundException`
  covers both "customer not found" and "account not found" cases, since both
  represent the same kind of failure (a lookup by id returned nothing).
