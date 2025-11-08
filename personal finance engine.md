# Smart Personal Finance Engine — Full Project Plan

*Comprehensive, step-by-step plan you can follow to build the project from zero to a portfolio-grade finished product.*

---

## 0. One-line project definition

A console-based personal finance engine that tracks accounts, records transactions, enforces budgets, simulates simple investment strategies, generates reports, and persists everything to files.

---

## 1. Project goals & scope (Clarify the problem)

**Primary goal:** Build a reliable, extensible CLI tool for personal finance that demonstrates full OOP usage in C++ and uses templates, STL, file I/O, and RTTI.

**Who it's for:** learners, hobbyists, and someone wanting a local, private finance manager.

**Must-have features (minimum scope):**

* Create and manage multiple accounts
* Add transactions: deposit, withdrawal, transfer
* Save/load snapshot to file
* Console-based interactive CLI

**Nice-to-have (but optional):**

* Import bank CSVs
* Budget categories and alerts
* Brokerage account with basic trades and positions
* Simple investment strategies (DCA)
* Report generation (CSV/JSON)
* Undo/redo for manual actions

**Out of scope for initial release:**

* Live bank API integration (Plaid, OAuth)
* GUI/web frontend
* Real-time market data

---

## 2. High-level architecture and subsystems (Break it down)

Split the system into independent subsystems so you can develop and test each one.

1. **Core Domain** — classes for `Account`, `Transaction`, `Budget`, `Position`, `Portfolio`.
2. **Persistence** — save/load snapshot, importers (CSV), audit log, backups.
3. **Repository / Storage** — templated in-memory repositories and indexes.
4. **Engine / Business Logic** — posting transactions, reconciling, forecasts.
5. **Strategy / Plugin Layer** — investment strategies, tax rules, report generators.
6. **CLI / UI** — command parser, interactive shell, scripted mode.
7. **Tests & Tools** — unit tests, integration tests, sample data.

---

## 3. Data model (Define core objects)

Write a concise list of classes and main attributes (fields). This is the backbone.

**Entities**

* `User` (id, name, portfolio)
* `Portfolio` (owns accounts and settings)
* `Account` (id, name, type, balance, currency)

  * Derived: `CheckingAccount`, `SavingsAccount`, `BrokerageAccount`
* `Transaction` (id, type, amount, date, accountId, memo)

  * Derived: `Deposit`, `Withdrawal`, `Transfer`, `Trade`
* `Position` (symbol, qty, avg_price) — for brokerage
* `BudgetCategory` (id, name, monthly_limit, spent)
* `Report` (generator output)
* `AuditEvent` (timestamped log entries)

**Supporting types**

* `Money<T>` template — wraps numeric amount and format
* `Id` (string alias)
* `Date` (ISO string or `std::chrono` wrapper)

---

## 4. Key interfaces & patterns

List the interfaces and design patterns you'll use.

**Interfaces (pure virtual classes)**

* `ITransaction` / `Transaction` base (virtual `apply(Account&)`)
* `IInvestmentStrategy` (virtual `execute(Portfolio&)`)
* `IReportGenerator` (virtual `generate(Portfolio&, params...)`)
* `IPersistence` (serialize/deserialize hooks)

**Design patterns**

* Factory: create accounts and transactions from file or CLI
* Strategy: investment and tax strategies
* Repository: templated storage for various entities
* Observer: CLI subscribes to portfolio changes for live updates
* Command: implement undo/redo of user actions

---

## 5. Technology choices & constraints

Keep these fixed so design remains consistent.

* Language: **C++17 or C++20** (pick what's allowed in your environment)
* Build: **CMake**
* No GUI — **console-only**
* Persistence formats: **JSON** snapshot (using header-only `nlohmann/json` if allowed) and **CSV** for imports/exports.
* Money: use `int64_t` cents in `Money<int64_t>` for fiat to avoid floating rounding errors.
* Testing: header-only framework like **doctest** or **Catch2** (single header).
* Logging: simple append-only text file (ISO 8601 timestamps).

---

## 6. Minimal Viable Product (MVP)

Define the smallest product that is useful and demonstrable.

**MVP features**

1. Create, list, delete accounts
2. Post deposit and withdrawal transactions
3. Transfer between accounts
4. Print account balances
5. Save/load snapshot to/from JSON file
6. Simple CLI with commands for the above

**Why MVP first?**

* Confirms core data model and persistence work. All other features build on top of this.

---

## 7. Roadmap & sprint plan (detailed tasks)

Break work into 9 sprints (2–6 days each depending on your pace). Each sprint ends with a working demo.

**Sprint 0 — Setup (1 day)**

* Initialize Git repo
* Create `CMakeLists.txt`
* Add `.gitignore`, `README.md`
* Add test framework header

**Sprint 1 — Money & IDs (1–2 days)**

* Implement `Money<T>` template
* Implement `Id` type & simple utilities
* Unit tests for `Money` arithmetic and formatting

**Sprint 2 — Accounts (2–3 days)**

* Implement `Account` base and `CheckingAccount`/`SavingsAccount`
* Methods: `post(Transaction)`, `getBalance()`
* Unit tests for posting transactions

**Sprint 3 — Transactions (3 days)**

* Implement `Transaction` base and `Deposit`, `Withdrawal`, `Transfer`
* Implement `apply(Account&)` semantics
* CLI commands: `add-transaction`, `list-transactions`
* Unit tests for transaction posting and transfers

**Sprint 4 — Repository & In-memory store (2 days)**

* Implement templated `Repository<T>` for accounts and transactions
* Add index by `Id` for quick lookup
* Tests for repository operations

**Sprint 5 — Persistence (3 days)**

* JSON snapshot save/load (atomic write tmp+rename)
* Simple CSV importer stub (map columns)
* Audit log write on state changes
* Test: save -> clear memory -> load -> assert equality

**Sprint 6 — CLI polish & user flows (2–3 days)**

* Build interactive shell or command parser
* Commands: `create-account`, `post`, `transfer`, `save`, `load`, `list-accounts`
* Help text and command history (optional)

**Sprint 7 — Budgets & Reports (3 days)**

* Implement `BudgetCategory` and monthly budget checks
* Implement a simple cashflow report (CSV export)
* Tests for budget enforcement and report generation

**Sprint 8 — Brokerage & Trades (4 days)**

* Add `BrokerageAccount`, `Trade` transaction, `Position` tracking
* Compute simple realized/unrealized P&L
* Unit tests for position updates

**Sprint 9 — Strategies, RTTI, undo/redo, polish (4–6 days)**

* Implement `IInvestmentStrategy` (DCA example)
* Use RTTI (`dynamic_cast`) to detect trades for tax-reporting
* Implement undo/redo command pattern for manual edits
* Clean docs and final README

---

## 8. Deliverables at each sprint (so you can demo)

Provide a demo checklist so every sprint produces a demonstrable artifact.

* S0: repo with CMake and README
* S1: `money_test` passing
* S2: `account_demo` (create account & deposit)
* S3: `transactions_demo` (post and transfer)
* S4: `repo_demo` (search & list)
* S5: `persist_demo` (save & load snapshot)
* S6: `cli_demo` (interactive session script)
* S7: `budget_report.csv` sample
* S8: `broker_demo` (buy/sell and position list)
* S9: `strategy_demo` (schedule DCA, run it), final docs

---

## 9. Example file formats & layout

**Snapshot (JSON)** — store the authoritative state (accounts, transactions, positions)

```json
{ "user": "Fahim", "accounts": [ {"type":"Checking","id":"CHK-001","name":"Everyday","balance":123456} ], "transactions": [ {"type":"Deposit","id":"T1","account":"CHK-001","amount":50000,"date":"2025-11-01T10:00:00Z"} ] }
```

**CSV import (bank statement)** – columns: `Date,Amount,Credit/Debit,Payee,Memo`

**Audit log** – text lines: `2025-11-01T10:00:00Z | INFO | Deposit T1 applied to CHK-001 | balance 1234.56`

**Repo layout** (suggested)

```
/ (project root)
  /src
    main.cpp
    cli.cpp
    core/ money.hpp, account.hpp, transaction.hpp
    persistence/ json_persist.cpp
    repo/ repository.hpp
  /tests
  CMakeLists.txt
  README.md
```

---

## 10. Command-line UX (sample session)

Example user session to show flow (you should be able to run these once CLI is implemented):

```
$ finance-cli create-account --type checking --id CHK-001 --name "Everyday"
Account CHK-001 created.

$ finance-cli post --type deposit --account CHK-001 --amount 500.00 --memo "paycheck"
Transaction T1 posted.

$ finance-cli transfer --from CHK-001 --to SAV-001 --amount 200.00
Transfer T2 posted.

$ finance-cli save --file snapshot.json
Saved snapshot.json
```

---

## 11. Testing strategy & important edge-cases

**Unit tests**: Money arithmetic, Account posting, Transaction types, Repository operations.

**Integration tests**: Save/load roundtrip, import CSV and reconcile, CLI command flow.

**Edge cases to test**

* Rounding and precision errors (Money template)
* Applying the same transaction twice (idempotency)
* Partial failure during import (roll back)
* Loading a corrupted snapshot (graceful error)
* Negative balances and overdraft behavior

---

## 12. Development practices & coding conventions

* Use `clang-format` or `clang-format` style; maintain consistent indentation
* Prefer `std::unique_ptr`/`std::shared_ptr` for ownership; avoid raw pointers across modules
* Use RAII for file handles and locks
* Keep functions small and single-purpose
* Write tests before/while implementing (TDD lightweight)
* Commit often with clear messages (small atomic commits)

---

## 13. Risk & mitigation

* **Scope creep**: avoid adding online banking or GUI until core is done. Follow MVP!
* **Precision bugs**: use integer cents for fiat amounts.
* **Serialization complexity**: keep a stable JSON schema and version it in files.

---

## 14. Final checklist before coding

* [ ] Repo created and CMake ready
* [ ] MVP features documented and agreed
* [ ] Money type decided (`int64_t` cents)
* [ ] Decide JSON library or write simple serializer
* [ ] Test framework added
* [ ] Sprint schedule pinned

---

## 15. Next recommended immediate actions (first 24–48 hours)

1. Create the Git repo and push an initial `README` describing the project goal and MVP.
2. Add `CMakeLists.txt` and skeleton directories.
3. Implement `Money<int64_t>` and unit tests (this will give early confidence).
4. Sketch `Account` and `Transaction` classes on paper (5–10 minutes).

---

## Appendix A — Minimal class skeletons (to paste in .hpp quickly)

```cpp
// money.hpp
#pragma once
#include <cstdint>
#include <string>

using Cents = int64_t;

struct Money {
  Cents cents;
  explicit Money(Cents c=0): cents(c){}
  Money operator+(const Money&o) const { return Money(cents + o.cents); }
  Money operator-(const Money&o) const { return Money(cents - o.cents); }
  std::string toString() const; // implement formatting
};

// account.hpp (very small)
#pragma once
#include "money.hpp"
#include <string>
#include <vector>

struct Transaction; // forward

class Account {
public:
  Account(std::string id, std::string name);
  virtual ~Account();
  virtual void post(const Transaction& t);
  Money balance() const;
  std::string id() const;
private:
  std::string id_;
  std::string name_;
  Money balance_;
};
```

---

If you want, I will now **generate the actual Git repo skeleton and the first sprint code** (Money template, tests, and CMake) so you can run it locally and start building — say the word and I will produce those files next.
