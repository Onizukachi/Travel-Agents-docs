# Coding Agent Instructions

## Business Context
This is a travel industry project.

The application allows users to search, book, and purchase tours and hotels.
The project is a **travel aggregator**, not a tour operator.

## Project Overview

We build Rails applications that prioritize **simplicity, clarity, and maintainability**. We trust Rails conventions rather than fighting them. We name things after business domains, not technical patterns. We avoid over-architecture.

Generated code should:
- Follow Rails conventions (not fight the framework)
- Use domain language (Participant, not User; Cloud, not GeneratedImage)
- Keep logic at the right layer (models handle data, controllers handle HTTP, jobs coordinate workflows)
- Normalize data properly (separate concerns into tables, not columns)

### LT CLI in Shell Session

- Always preload `lt` before using LT commands in terminal sessions:
  - `source ./lt.sh`
- Run LT commands through this loaded function (for example, `lt logs rails`, `lt status`).

## Technology Stack & Gems

These are the gems currently used in the project.
If you want to add a new gem, propose it first (with a short rationale and tradeoffs) and wait for approval before adding it.

### Core Rails & Server
- `rails`
- `puma`

### Database & Data
- `activerecord` (included)
- `mysql2` (MySQL)

### Jobs & Background Work
- `sidekiq` (default)

### Testing
- `rspec-rails` (not minitest)
- `factory_bot_rails` (test data)
- `faker` (realistic fake data)

### Code Quality
- `rubocop`
- `prettier` (JavaScript formatting, if you have JS)

### HTTP & API
- `Typhoeus` (clean, readable HTTP requests)
- Don't use: Faraday, RestClient
- Use ExternalRequest as a wrapper Typhoeus

### Admin
- `active_admin` (Rails admin panel)

### Error Tracking
- `sentry-rails` (error reporting—optional)

**Critical rules:**
- Complex operations go in namespaced service classes: `Clouds::CardGenerator` in `services` folder
---

## Debugging and Development

- Use `rails console` for interactive debugging.
- Use `binding.irb` for breakpoints in development.

## Localization (I18n)

- Prefer translations in `config/locales/ru.yml` for validation errors, model attribute names, and model names.
- Use Rails I18n keys under `activerecord.errors.models`, `activerecord.attributes`, and `activerecord.models` instead of hardcoded Russian strings.

---

## Formatting

- For Ruby percent literals, prefer parentheses delimiters: `%w(...)`, `%i(...)`, `%W(...)`, `%I(...)`, `%q(...)`, `%Q(...)`, `%r(...)`, `%x(...)`.
-  Prefer single-quoted strings unless you need double quotes for interpolation, escape sequences, or to avoid extra backslashes.

---

## Code Clarity Guidelines

### Constants Over Magic Numbers

- Replace hard-coded values with named constants.
- Use descriptive constant names that explain the value's purpose.
- Keep constants at the top of the class/module or in a dedicated constants file when shared.

### Meaningful Names

- Variables, methods, and classes should reveal their purpose.
- Names should explain why something exists and how it is used.
- Avoid abbreviations unless they are universally understood (`API`, `URL`, `ID`).

### Smart Comments

- Prefer self-documenting code over comments that repeat what the code already says.
- Use comments to explain why something is implemented a certain way.
- Document API contracts, complex algorithms, and non-obvious side effects.

### Single Responsibility

- Each method should do exactly one thing.
- Keep methods small and focused.
- If a method needs an explanatory comment to describe what it does, split it into smaller methods.

---

## File structure
- `app/admin/`
  ActiveAdmin controllers

- `app/controllers/`
  Application controllers (API and web).

- `app/services/`
  Application and domain services. Contains business logic and workflows

- `app/apis/`
  Used for external service clients and integration classes (including payment gateways and fiscal processors).

- `app/models/`
  ActiveRecord models and domain concerns.

- `app/query/`
  Read-only query objects.
  Used for complex SQL or data-fetching logic.

- `app/serializers/`
  Objects responsible for API response serialization.

- `app/decorators/`
  Presentation-layer decorators.
  Used to format model data for views or serializers.

- `app/helpers/`
  View helpers only.

- `app/mailers/`
  ActionMailer classes and mail delivery logic.

- `app/uploaders/`
  CarrierWave uploaders and related configuration.

## ActiveAdmin Resource Order

For files in `app/admin/*.rb`, follow this baseline order:

1. Base config: `menu`, `actions`, `permit_params`, `includes`, `config.*`
2. `scope`
3. `filter` (including custom filter collections/options)
4. Presentation blocks: `index`, `show`, `form`
5. UI actions: `action_item`, `batch_action`, `sidebar`
6. Custom actions: `member_action`, `collection_action`
7. `controller do ... end` (overrides and private helpers)

If you need exceptions, keep them minimal and prefer this order for readability.

## Model Patterns

### Naming: Use Domain Language

**Bad (Technical):**
```ruby
class User < ApplicationRecord
  has_many :generated_images
end

class GeneratedImage < ApplicationRecord
  belongs_to :user
end
```

**Good (Domain-appropriate):**
```ruby
class Participant < ApplicationRecord
  has_many :clouds, dependent: :destroy
  has_many :invitations, dependent: :destroy
end

class Cloud < ApplicationRecord
  belongs_to :participant
end

class Invitation < ApplicationRecord
  belongs_to :participant
end
```

Names should reflect the business domain. "Participant" is what they are at a conference. "Cloud" is what they generate. Not generic terms.

### Model Organization Order

Always follow this order in model files:

```ruby
class Cloud < ApplicationRecord
  # Gems and DSL extensions
  extend FriendlyId
  friendly_id :name, use: [:slugged, :finders]

  # Associations
  belongs_to :participant
  has_many :invitations, dependent: :destroy
  has_one :latest_invitation, -> { order(created_at: :desc) }, class_name: "Invitation"

  # Enums (for state)
  enum :state, %w(uploaded analyzing analyzed generating generated failed).index_with(&:to_s)

  # Validations
  validates :name, :state, presence: true
  validates :participant_id, presence: true

  # Scopes
  scope :generated, -> { where(state: :generated) }
  scope :picked, -> { where(picked: true) }
  scope :recent, -> { order(created_at: :desc) }

  # Callbacks
  before_create do
    self.state ||= :uploaded
  end

  # Delegated methods
  delegate :email, to: :participant, prefix: true

  # Public instance methods
  def ready_to_generate?
    analyzed? && !generating?
  end

  # Private methods
  private

  def generate_filename
    "cloud-#{participant.slug}-#{id}.png"
  end
end
```

### Use Enums for State

**Always use enums for states.** Keep `state`/`status` as string columns in the database and map enum values with `index_with(&:to_s)`.

```ruby
class Cloud < ApplicationRecord
  enum :state, %w(uploaded analyzing analyzed generating generated failed).index_with(&:to_s)
end

# Usage:
cloud.uploaded?          # Predicate method
cloud.generating!        # Bang method (update + save)
Cloud.generated.count    # Scope
```

Why: Type-safe, gives you predicate methods for free, database-efficient.

### Thin Models, Smart Organization

**Model should not be 100+ lines.** If it is, extract to namespaced services.

**Bad (Model too fat):**
```ruby
class Cloud < ApplicationRecord
  def generate_card_image
    # 50 lines of API logic
    # 20 lines of image processing
    # 30 lines of error handling
  end

  def check_nsfw
    # 40 lines of moderation logic
  end

  def upload_to_storage
    # 30 lines of storage logic
  end
end
```

**Good (Extracted to namespaced services):**
```ruby
class Cloud < ApplicationRecord
  # Model: just data and simple methods
  def ready_to_generate?
    analyzed?
  end
end

module Clouds
  class CardGenerator
    def initialize(cloud, api_key: GeminiConfig.api_key)
      @cloud = cloud
      @api_key = api_key
    end

    def generate
      # Complex API logic here, returns IO object
    end
  end
end


module Clouds
  class NSFWDetector
    def initialize(cloud, api_key: GeminiConfig.api_key)
      @cloud = cloud
      @api_key = api_key
    end

    def check
      # Moderation logic, returns true/false
    end
  end
end

```

**When to extract:**
- Any method over 15 lines
- Any method calling external APIs
- Any complex calculation
- Anything reusable

**How to structure:**
```ruby
# app/services/clouds/card_generator.rb
module Clouds
  class CardGenerator
    def initialize(cloud, api_key: GeminiConfig.api_key)
      @cloud = cloud
      @api_key = api_key
    end

    def generate
      # Public method that returns simple value, Dry::Monads[:result] or raises
      prompt = build_prompt
      response = call_api(prompt)
      decode_image(response)
    end

    private

    attr_reader :cloud, :api_key

    def build_prompt
      # ...
    end

    def call_api(prompt)
      # ...
    end

    def decode_image(response)
      # ...
    end
  end
end

```

### Callbacks: Use Sparingly

**Callbacks are okay for simple things. Not for workflows.**

Good use:
```ruby
class Participant < ApplicationRecord
  before_create do
    self.access_token ||= Nanoid.generate(size: 6)
  end

  before_save do
    self.slug = nil if name_changed?  # Friendly ID will regenerate
  end
end
```

Bad use:
```ruby
# DON'T: Complex workflow in callback
class Cloud < ApplicationRecord
  after_create do
    CloudGenerationJob.perform_later(self)
    Mailer.notify_created(self).deliver_later
    Metrics.record_cloud_created(self)
  end
end
```

**Instead use:** A job or service object for workflows.

---

## Controller Patterns

### Keep Controllers Extremely Thin

**Target: 5-15 lines per action.** No business logic.

```ruby
class Participant::CloudsController < Participant::ApplicationController
  def new
    redirect_to home_path unless @participant.can_generate_cloud?
  end

  def create
    return head 422 unless @participant.can_generate_cloud?

    blob = ActiveStorage::Blob.find_signed(params[:cloud][:blob_signed_id])
    return head 422 unless blob

    cloud = @participant.clouds.create do
      it.image.attach(blob)
    end

    CloudGenerationJob.perform_later(cloud)

    redirect_to cloud_path(cloud)
  end

  def update
    cloud = @participant.clouds.find(params[:id])
    Cloud.transaction do
      @participant.clouds.update_all(picked: false)
      cloud.update_column(:picked, true)
    end

    redirect_to home_path
  end
end
```

**Action breakdown:**
- Guard clauses (early returns)
- Simple model operations (create, update)
- Job enqueueing
- Redirect/render

**No:**
- Business logic
- Complex conditionals
- Data transformation

### Use Namespace Controllers for Authentication/Scoping

**Pattern:**
```ruby
# app/controllers/participant/application_controller.rb
class Participant::ApplicationController < ::ApplicationController
  before_action :set_participant

  private

  def set_participant
    @participant = ::Participant.find_by!(access_token: params[:access_token])
  end
end

# app/controllers/participant/clouds_controller.rb
class Participant::CloudsController < Participant::ApplicationController
  # @participant is automatically set
  def index
    @clouds = @participant.clouds.recent
  end
end
```

All routes under `Participant::` are automatically scoped. No need for concerns or custom modules.

### Return Early, Use Guard Clauses

**Bad:**
```ruby
def create
  if user.premium?
    if params[:name].present?
      if validate_input
        cloud = create_cloud
        return redirect_to cloud
      end
    end
  end
  head 422
end
```

**Good:**
```ruby
def create
  return head 401 unless @participant.premium?
  return head 422 unless params[:name].present?
  return head 422 unless validate_input

  cloud = @participant.clouds.create!(params.permit(:name))
  redirect_to cloud
end
```

Guard clauses make the happy path obvious.

---

## Database Design & Migrations

### Normalize Data: One Concern Per Table

**Bad (Denormalized):**
```ruby
create_table :participants do |t|
  t.string :email
  t.string :full_name
  t.datetime :invitation_sent_at
  t.datetime :invitation_opened_at
  t.string :bounce_type
  t.datetime :bounced_at
  t.boolean :invitation_resend_requested
  # Everything crammed together
end
```

**Good (Normalized):**
```ruby
create_table :participants do |t|
  t.string :email, null: false
  t.string :full_name, null: false
  t.string :access_token
  t.integer :cloud_generations_quota, default: 5
  t.integer :cloud_generations_count, default: 0
  t.integer :invitations_count, default: 0
  t.timestamps
end

create_table :invitations do |t|
  t.references :participant, null: false, foreign_key: true
  t.string :status, null: false, default: "sent"
  t.datetime :opened_at
  t.string :bounce_type
  t.datetime :bounced_at
  t.timestamps
end

create_table :clouds do |t|
  t.references :participant, null: false, foreign_key: true
  t.string :state, null: false, default: "uploaded"
  t.boolean :picked, default: false
  t.string :failure_reason
  t.timestamps
end
```

Use ActiveRecord enums in models for these string state columns.

### Migration & Schema Workflow

- After adding or changing migrations, run `bundle exec rails db:migrate`.
- After both migrate and rollback, `db/schema.rb` may include unrelated local drift changes.
- Remove unrelated schema changes and keep only changes relevant to your migration task.
- Keep `ActiveRecord::Schema.define(version: ...)` aligned with the currently latest applied migration (after both migrate and rollback).

## Job Patterns

### Jobs Orchestrate, Services Execute

**Good separation:**
```ruby
# Job orchestrates workflow
class CloudGenerationJob < ApplicationJob
  def perform(cloud)
    generate(cloud)
  end

  private

  def generate(cloud)
    generator = Clouds::CardGenerator.new(cloud)
    io = generator.generate  # Delegates to service class
    cloud.generated_image.attach(io:, filename: "...")
  end
end

# Service class executes business logic
module Clouds
  class CardGenerator
    def initialize(cloud)
      @cloud = cloud
    end

    def generate
      # Complex API/processing logic here
      StringIO.new(decoded_image_data)
    end

    private

    def build_prompt
      # ...
    end

    def call_api(prompt)
      # ...
    end
  end
end

```

**Don't put complex logic in jobs.** Jobs are for orchestration. Service classes handle complexity.

### Error Handling in Jobs

```ruby
def moderate(_step)
  # Do work
  cloud.update!(state: :analyzed)
rescue => e
  Sentry.capture_exception(e)
  cloud.update!(state: :failed, failure_reason: e.message)
end
```

Always:
- Catch errors with `rescue => e`
- Report to error tracking
- Update model state to reflect failure
- Don't re-raise unless you want the entire job to fail

---

## Query Objects

### Use Query Objects for Complex Queries

**When:**
- Query has multiple conditions
- Query is reused across controllers
- Query is easier to test in isolation

---

### Database Queries from Views

**Good:** Simple associations and scopes
```erb
<% @participant.clouds.recent.each do |cloud| %>
  <%= render "cloud", cloud: %>
<% end %>
```

**Bad:** N+1 queries, complex logic in views
```erb
<!-- DON'T: complex query logic -->
<% @clouds.select { |c| c.participant.premium? && c.state.in?(%w(generated)) } %>
```

**Instead:** Query object or scope
```ruby
# Model
scope :recent, -> { order(created_at: :desc) }

# Controller
@clouds = @participant.clouds.recent

# View
<% @clouds.each do |cloud| %>
  <%= render "cloud", cloud: %>
<% end %>
```

---

## Testing Patterns

### RSpec > Minitest

Use RSpec for better DSL and readability.

### Testing Commands

Run individual test files using:

```bash
# Run specific spec file
bin/rspec spec/models/user_spec.rb

# Run specific test
bin/rspec spec/models/user_spec.rb:25
```

### Test Data Performance

- Prefer `let_it_be` where possible to speed up test suites.
- If an object is modified in examples, use `let_it_be_with_reload`.

### Test Organization

```
spec/
├── models/          # Models
├── services/        # Business logic
├── controllers/     # Controller/HTTP responses
├── workers/         # Jobs
├── factories/
├── fixtures/
├── mailer/
├── support/
└── spec_helper.rb
```

### Model Tests: Logic & Validations

```ruby
describe Cloud do
  describe "#ready_to_generate?" do
    it "returns true when analyzed" do
      cloud = create(:cloud, state: :analyzed)
      expect(cloud.ready_to_generate?).to be true
    end

    it "returns false when generating" do
      cloud = create(:cloud, state: :generating)
      expect(cloud.ready_to_generate?).to be false
    end
  end

  describe "validations" do
    it "validates presence of participant_id" do
      cloud = build(:cloud, participant_id: nil)
      expect(cloud).not_to be_valid
    end
  end
end
```

### Controller Tests: HTTP Behavior

```ruby
describe "Participant::CloudsController" do
  describe "POST /participant/:access_token/clouds" do
    it "creates a cloud when user can generate" do
      participant = create(:participant)
      blob = create(:active_storage_blob)

      expect do
        post participant_clouds_path(access_token: participant.access_token),
          params: { cloud: { blob_signed_id: blob.signed_id } }
      end.to change(Cloud, :count).by(1)

      expect(response).to redirect_to(participant_cloud_path(participant.access_token, Cloud.last))
    end
  end
end
```

### Use FactoryBot for Test Data

```ruby
# spec/factories/participants.rb
FactoryBot.define do
  factory :participant do
    full_name { Faker::Name.name }
    email { Faker::Internet.email }

    trait :with_cloud do
      after(:create) do |participant|
        create(:cloud, participant:)
      end
    end
  end
end

# In tests
participant = create(:participant, :with_cloud)
```

### Don't Test Framework Behavior

**Skip these tests:**
- Model associations (too simple to break)
- Generated code (don't test the generator output)

**Test these:**
- Validations
- Business logic methods
- Controller responses

---

## Routing

### RESTful Routes with Namespaces

```ruby
Rails.application.routes.draw do
  root "clouds#index"

  # Public routes
  resources :clouds, only: [:index, :show]

  # Participant-scoped routes
  scope "/c/:access_token", as: :participant, module: :participant do
    resource :home, only: [:show]
    resources :clouds
  end

  # Webhooks
  namespace :webhooks do
    resource :mandrill, only: [:create]
  end
end
```

**Route patterns:**
- RESTful resources (index, show, create, update, delete)
- Namespaces for logical grouping (admin, webhooks, participant)
- Scopes for parameter injection (:access_token available to all routes)
---

## Common Patterns & Anti-Patterns

### Pattern: Delegation Over Inheritance for Small Helpers

**Bad:**
```ruby
module EmailHelper
  def participant_email
    "#{@participant.slug}@example.com"
  end
end

class CloudsController < ApplicationController
  include EmailHelper
end
```

**Good:**
```ruby
class CloudsController < ApplicationController
  def email_address
    @participant.participant_email  # Delegate to model
  end
end

class Participant
  def participant_email
    "#{slug}@example.com"
  end
end
```

### Anti-Pattern: Passing Around Hashes

**Bad:**
```ruby
def create_cloud(params)
  { state: "generating", id: cloud.id, user_id: cloud.participant_id }
end

result = create_cloud(name: "test")
puts result[:state]
```

**Good:**
```ruby
def create_cloud(params)
  Participant.create!(**params)
end

cloud = create_cloud(name: "test")
puts cloud.state
```

Return objects, not hashes. Type-safe and IDE-friendly.

---

### Scopes: Don't Use Complex Calculations

**Bad:**
```ruby
scope :active, -> {
  where("clouds.created_at > ?", Time.current - 30.days)
    .where("clouds.state = ? OR (clouds.state = ? AND clouds.updated_at > ?)",
      "generated", "generating", Time.current - 1.hour)
}
```

**Good:**
```ruby
scope :recent, -> { where("created_at > ?", 30.days.ago) }
scope :active, -> { where(state: [:generated, :generating]) }
scope :recently_active, -> { where("updated_at > ?", 1.hour.ago) }

# Compose scopes
Cloud.recent.active.recently_active
```

### N+1 Prevention

**Use eager loading:**
```ruby
# Bad: N+1 queries
Participant.all.each { |p| p.clouds.count }

# Good: 2 queries total
Participant.all.includes(:clouds)
```

### Indexes

```ruby
create_table :clouds do |t|
  t.references :participant, null: false, foreign_key: true
  t.string :state, null: false, default: "uploaded"
  t.boolean :picked

  t.index [:participant_id, :state]  # Composite index
  t.index :picked
end
```

Index on:
- Foreign keys
- Frequently queried columns
- Enum state columns
---

## Summary: The Checklist

Before writing/generating code, ask:

- [ ] Is this named after a business domain concept (not technical)?
- [ ] Is the model organized in the right order (gems → associations → enums → validations → scopes)?
- [ ] Are states in enums, not string columns?
- [ ] Is the controller action under 15 lines?
- [ ] Is complex logic extracted to namespaced service classes (e.g., `Clouds::CardGenerator`)?
- [ ] Is the database normalized (one concern per table)?
- [ ] Are tests in RSpec, not minitest?
- [ ] Are views simple (scopes/associations, no complex logic)?
- [ ] Are validation errors, model attributes, and model names translated through `config/locales/ru.yml`?
- [ ] For `app/admin/*.rb`, are blocks organized in the agreed ActiveAdmin order?

If yes to all: your code is ready to ship.

---

## Payments Playbook

For order/payment/callback/receipt flows, use:
- `.docs/payments.md`

---

## Quick Reference

| Pattern | Location | When |
|---------|----------|------|
| Model method | `Cloud#ready_to_generate?` | Simple query or check |
| Service class | `Clouds::CardGenerator` | Complex operation (>15 lines) |
| Scope | `Cloud.generated` | Reusable query |
| Job | `CloudGenerationJob` | Async workflow or steps |
| Query Object | `Participant::PendingQuery` | Complex query |
| Controller action | `Participant::CloudsController#create` | HTTP handling only |

| Anti-Pattern | Why | Alternative |
|---|---|---|
| Concern for logic | Magic, hard to trace | Inheritance or delegation |
| String state without enum mapping | No predicates/scopes and easy to break | String-backed enum (`index_with(&:to_s)`) |
| Fat models | Hard to maintain | Extract to namespaced services |
| Fat controllers | Hard to test | Thin controller + model method or services |
| Complex conditionals | Hard to read | Guard clauses |

---

## ActiveAdmin UI Testing (MCP)

- Before opening any UI page, start server logs:
  - `lt logs rails`
- Open the target page directly (for example, `https://leveltravel.dev/admin/payment_logs`).
- If authentication is required, click `Войти` (credentials are already prefilled in the browser session).
- If the page is unavailable or returns `502 Bad Gateway`, reload and retry.
- If UI errors happen, use Rails logs to identify and fix the issue.
