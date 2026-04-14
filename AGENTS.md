# Coding Agent Instructions

## 1) Business Context

- Product domain: travel.
- The app lets participants search, book, and purchase tours and hotels.
- Business model: travel aggregator, not a tour operator.

## 2) Core Principles

- Prefer Rails conventions over custom architecture.
- Keep code simple, clear, and maintainable.
- Use business/domain naming, not generic technical naming.
- Keep responsibilities by layer:
  - Models: data + small domain logic.
  - Controllers: HTTP only.
  - Jobs: orchestration only.
  - Services: complex business operations.

## 3) Operating Workflow

### LT CLI in shell session

- Before any LT command, run: `source ./lt.sh`.
- Run LT commands through the loaded function (`lt logs rails`, `lt status`, `lt console`, etc.).

### Debugging

- Use `lt console` for interactive debugging.
- Use `binding.irb` for breakpoints in development.

### Documentation sync (mandatory)

After every change to any of:
- `AGENTS.md`
- `.tasks/**/*`
- `.docs/**/*`

Do all of the following:
1. Mirror updated files into `../agents_md/`.
2. Run `git pull` inside `../agents_md` before push.
3. Commit and push mirrored changes in `../agents_md`.

## 4) Tech Stack and Dependency Rules

Current stack/gems:
- Rails + Puma
- ActiveRecord + MySQL (`mysql2`)
- Sidekiq
- RSpec (`rspec-rails`), `factory_bot_rails`, `faker`
- `rubocop`, `prettier` (for JS if present)
- `active_admin`
- `sentry-rails` (optional)
- HTTP: `Typhoeus` via `ExternalRequest`

Dependency policy:
- Do not add new gems without explicit approval.
- If a gem is needed, propose short rationale + tradeoffs first.

HTTP policy:
- Use `ExternalRequest` as wrapper around `Typhoeus`.
- Do not introduce `Faraday` or `RestClient`.

## 5) File Placement

- `app/admin/`: ActiveAdmin resources/controllers
- `app/controllers/`: web/API controllers
- `app/services/`: domain/application services
- `app/apis/`: external integrations (payments/fiscal/etc.)
- `app/models/`: ActiveRecord models
- `app/query/`: read-only query objects
- `app/serializers/`: API serialization
- `app/decorators/`: presentation formatting
- `app/helpers/`: view helpers only
- `app/mailers/`: mail delivery logic
- `app/uploaders/`: CarrierWave uploaders

## 6) Naming and Organization Conventions

### Naming

- Use domain names (`Participant`, `Cloud`, etc.), not technical placeholders (`User`, `GeneratedImage`, etc.).

### Model structure order

Keep this order in model files:
1. DSL/gem extensions
2. associations
3. enums
4. validations
5. scopes
6. callbacks
7. delegations
8. public methods
9. private methods

### State fields

- Use enums for states/statuses.
- Keep DB columns as strings.
- Prefer string-backed enum mapping:

```ruby
enum :state, %w(uploaded analyzing analyzed generating generated failed).index_with(&:to_s)
```

### Controller constraints

- Target: 5-15 lines per action.
- Use guard clauses and early returns.
- Keep business logic out of controllers.
- Use namespaced base controllers for shared scoping/auth.

### Service extraction rules

Extract to namespaced services (e.g. `Clouds::CardGenerator`) when logic is:
- complex,
- calling external APIs,
- reused,
- or too large for a model/controller method.

## 7) Data and Migrations

### Data modeling

- Normalize tables: one concern per table.
- Index foreign keys and frequently filtered columns.
- Add composite indexes for common query patterns.

### Migration workflow

After migration edits:
1. Run `bundle exec rails db:migrate`.
2. If needed, verify rollback path.
3. Keep `db/schema.rb` limited to relevant changes.
4. Keep `ActiveRecord::Schema.define(version: ...)` aligned with latest applied migration.

## 8) Job Rules

- Jobs orchestrate workflow; heavy logic belongs in services.
- Handle errors explicitly:
  - rescue,
  - report (Sentry where used),
  - persist failed state/reason.

## 9) Query and View Rules

- Use query objects for complex/reused data fetching.
- Keep views simple: associations/scopes only.
- Avoid complex filtering logic in templates.
- Prevent N+1 with eager loading (`includes`, etc.).

## 10) Localization (I18n)

- Prefer translations in `config/locales/ru.yml`.
- Use keys under:
  - `activerecord.errors.models`
  - `activerecord.attributes`
  - `activerecord.models`
- Avoid hardcoded Russian strings in validations and model labels.

## 11) Formatting Rules

- Prefer single-quoted strings unless interpolation/escaping requires double quotes.
- Use percent literal parentheses delimiters:
  - `%w(...)`, `%i(...)`, `%W(...)`, `%I(...)`, `%q(...)`, `%Q(...)`, `%r(...)`, `%x(...)`.

## 12) ActiveAdmin Rules

### Resource section order (`app/admin/*.rb`)

1. Base config: `menu`, `actions`, `permit_params`, `includes`, `config.*`
2. `scope`
3. `filter`
4. presentation: `index`, `show`, `form`
5. UI actions: `action_item`, `batch_action`, `sidebar`
6. custom actions: `member_action`, `collection_action`
7. `controller do ... end`

### UI testing flow (MCP)

1. Start logs: `lt logs rails`.
2. Open target page directly (example: `https://leveltravel.dev/admin/payment_logs`).
3. If auth required, click `Войти` (credentials are prefilled).
4. If `502 Bad Gateway`, reload and retry.
5. Use Rails logs to diagnose/fix UI errors.

## 13) Testing Rules

- Use RSpec (not minitest).
- Prefer focused runs:

```bash
bin/rspec spec/models/some_model_spec.rb
bin/rspec spec/models/some_model_spec.rb:25
```

- Use `let_it_be` / `let_it_be_with_reload` when it improves suite speed.
- Test:
  - validations,
  - business logic,
  - controller HTTP behavior.
- Do not spend effort on testing trivial framework behavior.

## 14) Routing

- Prefer RESTful routes.
- Use namespaces/scopes for logical and auth boundaries (participant/admin/webhooks).

## 15) Payments

- For order/payment/callback/receipt flows, follow: `.docs/payments.md`.

## 16) Definition of Done

### MUST

- Domain naming is consistent with business terms.
- Layering is respected (controller/job/model/service responsibilities).
- State columns use enums with string-backed mapping.
- Complex logic is extracted to namespaced services.
- I18n keys are used for AR errors/attributes/model names.
- `app/admin/*.rb` follows the agreed block order.
- Required migrations are applied and schema changes are clean/relevant.
- Relevant tests pass locally for changed behavior.

### SHOULD

- Controllers stay thin with clear guard clauses.
- Queries remain composable/readable; no view-level complex filtering.
- N+1 risks are handled with eager loading.
- New indexes are added where query patterns require them.

If all MUST items are satisfied, the change is ready for review.
