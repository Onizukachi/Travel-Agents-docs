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
- Be readable without comments
- Normalize data properly (separate concerns into tables, not columns)

## Technology Stack & Gems

**Only use these gems.** If you want to add something not listed, ask in a clarifying comment.

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

## File structure
- `app/admin/`
  ActiveAdmin controllers

- `app/controllers/`
  API controllers only (no HTML controllers).

- `app/services/`
  Application and domain services. Contains business logic and workflows

- `app/apis/`
  Used for interaction with payment gateways and fiscal processors only.

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
  # 1. Gems and DSL extensions
  extend FriendlyId
  friendly_id :name, use: [:slugged, :finders]

  # 2. Associations
  belongs_to :participant
  has_many :invitations, dependent: :destroy
  has_one :latest_invitation, -> { order(created_at: :desc) }, class_name: "Invitation"

  # 3. Enums (for state)
  enum :state, %w(uploaded analyzing analyzed generating generated failed).index_by(&:itself)

  # 5. Validations
  validates :name, :state, presence: true
  validates :participant_id, presence: true

  # 6. Scopes
  scope :generated, -> { where(state: :generated) }
  scope :picked, -> { where(picked: true) }
  scope :recent, -> { order(created_at: :desc) }

  # 7. Callbacks
  before_create do
    self.state ||= :uploaded
  end

  # 8. Delegated methods
  delegate :email, to: :participant, prefix: true

  # 9. Public instance methods
  def ready_to_generate?
    analyzed? && !generating?
  end

  # 10. Private methods
  private

  def generate_filename
    "cloud-#{participant.slug}-#{id}.png"
  end
end
```

### Use Enums for State

**Always use enums for states.** No string columns like `status` or `state_string`.

```ruby
class Cloud < ApplicationRecord
  enum :state, %w(uploaded analyzing analyzed generating generated failed).index_by(&:itself)
end

# Usage:
cloud.uploaded?          # Predicate method
cloud.generating!        # Bang method (update + save)
Cloud.generated.count    # Scope
```

Why: Type-safe, gives you predicate methods for free, database-efficient.

### Thin Models, Smart Organization

**Model should not be 100+ lines.** If it is, extract to namespaced services or concerns.

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

Delegates to related objects. Returns simple values, Dry::Monads[:result], or raises exceptions on error.

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

### Don't Use Concerns for Business Logic

**Bad:**
```ruby
module TokenAuthenticated
  extend ActiveSupport::Concern

  included do
    before_action :authenticate_by_token!
  end

  def authenticate_by_token!
    # ...
  end
end

class CloudsController < ApplicationController
  include TokenAuthenticated
end
```

**Good:**
```ruby
class Participant::ApplicationController < ApplicationController
  before_action :set_participant

  private

  def set_participant
    @participant = Participant.find_by!(access_token: params[:access_token])
  end
end

class Participant::CloudsController < Participant::ApplicationController
  # Inheritance handles scoping, no magic
end
```

Inheritance is clearer than concerns.

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
  t.integer :participant_id, null: false, foreign_key: true
  t.enum :status, enum_type: :invitation_status, default: "sent"
  t.datetime :opened_at
  t.string :bounce_type
  t.datetime :bounced_at
  t.timestamps
end

create_table :clouds do |t|
  t.integer :participant_id, null: false, foreign_key: true
  t.enum :state, enum_type: :cloud_state, default: "uploaded"
  t.boolean :picked, default: false
  t.string :failure_reason
  t.timestamps
end
```

## Job Patterns

### Jobs Orchestrate, Models Execute

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
rescue => err
  Sentry.capture_exception(e)
  cloud.update!(state: :failed, failure_reason: err.message)
end
```

Always:
- Catch errors with `rescue => err`
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
<% @clouds.select { |c| c.participant.premium? && c.state.in?(%w[generated]) } %>
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

### Pattern: Guard Clauses Over Nested Conditionals

**Bad:**
```ruby
def create
  if admin?
    if params[:valid]
      if check_limit
        create_object
      end
    end
  end
end
```

**Good:**
```ruby
def create
  return head 401 unless admin?
  return head 422 unless params[:valid]
  return head 429 unless check_limit

  create_object
end
```

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
  t.integer :participant_id
  t.enum :state
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

If yes to all: your code is ready to ship.

---

## Orders, Payments, and Receipts flow

This section describes the end-to-end flow of order creation, payment processing, and receipt generation.

### 1. Order prebuild (draft state)

During order formation (traveler details input, coupon application, price recalculation), the frontend sends incremental updates to:

- `Papi::V3::OrdersController#prebuild`

Characteristics:
- The order is not persisted yet.
- The endpoint is used for validation, pricing, and preview purposes.
- Multiple requests may be sent as the user modifies order data.
- No payment objects are created at this stage.

---

### 2. Order creation

When the user selects a payment method and confirms the purchase, the frontend sends a request to:

- `Papi::V3::OrdersController#prebuild`

Characteristics:
- The order is persisted in the database.
- The order transitions from a draft/prebuild state to a created state.
- Further payment actions depend on the selected payment method.

---

### 3. Payment initiation

Available payment methods are determined by:

- `Order#payment_methods`

Based on the selected payment method, the frontend performs one of the following actions:

- **Card payment**:
  - Sends a request to:
    - `Papi::V3::PaymentsController#pay_card`

- **Non-card / redirect-based payment methods**:
  - Sends a request to:
    - `Papi::V3::PaymentsController#payment_params`
  - The response typically contains a payment URL to which the user must be redirected.

---

### 4. Payment creation and processors

Payments are created via:

- `PaymentBuilder`

Responsibilities of `PaymentBuilder`:
- Creates a `Payment` record.
- Assigns a `processor` to the payment.

The `processor`:
- Is a string representing a Ruby constant name.
- Points to a class responsible for interacting with a specific payment gateway API.
- All payment processors are located in:
  - `app/apis/payment_processor/`

Each processor encapsulates:
- API request building
- Authentication
- Gateway-specific logic

---

### 5. Asynchronous payment callbacks

Most payment gateways operate asynchronously.

After payment actions (authorization, capture, refund), external providers send callbacks (webhooks) to the system.

Callback handling:
- Implemented in classes with the `_callback_processor` suffix.
- Located in `app/apis/`
- Example:
  - `app/apis/alfa_pay_callback_processor.rb`

Responsibilities of callback processors:
- Validate incoming notifications.
- Update payment and order states.
- Trigger post-payment logic when applicable.

---

### 6. Receipts and line items generation

After successful payments or refunds:

- Receipts are generated.
- Line items are created and attached to receipts.

Domain models:
- `Receipt` — represents a fiscal receipt.
- `LineItemV2` — represents individual product or service items within a receipt.

Receipt generation logic:
- Implemented in:
  - `Receipts::Builder`

Responsibilities of `Receipts::Builder`:
- Create receipts for payments and refunds.
- Generate corresponding `LineItemV2` records.
- Ensure consistency between payments, receipts, and line items.

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
| String state | Type-unsafe | Enum |
| Fat models | Hard to maintain | Extract to namespaced services |
| Fat controllers | Hard to test | Thin controller + model method |
| Complex conditionals | Hard to read | Guard clauses |

---

## Promo Campaigns & Coupons Notes

These are project-specific rules discovered while working with promo campaign coupon generation and cleanup.

### Coupon Generation Flow

- Entry point is `PromoCampaign#start_generation!(initiator:)`.
- `start_generation!` creates a `PromoCampaignOperation` with `operation_type: :generate` and `status: :processing`.
- `PromoCampaigns::CouponGeneratorWorker` processes this operation and creates coupons one-by-one via `Coupons::CouponCreator`.
- Do not re-run generation guard checks inside the worker that block `processing` state operations.

### Partial Generation Policy

- Partial generation is allowed by business logic.
- If generation fails in the middle, already created coupons must remain.
- In failure case:
  - operation status becomes `failed`
  - `error_message` is filled
  - generated count is still stored in operation metadata

### Operation Metadata Convention

- `PromoCampaignOperation` keeps runtime generation details in `metadata`.
- Use string keys in metadata consistently.
- For generated count always use:
  - key: `'generated_coupons_count'`
- Accessor should read the same string key:
  - `metadata.fetch('generated_coupons_count', 0)`

### Cleanup Constraints

- `PromoCampaigns::ArchivedCleanupWorker` deletes archived campaigns only if `promo_campaign.can_delete_coupons?` is true.
- `PromoCampaign#can_delete_coupons?` depends on:
  - no `processing` operations
  - no used coupons (`contains_used_coupons?`)
- `contains_used_coupons?` treats coupons as used if `uses_left` differs from campaign `uses`.
- In cleanup specs, coupons intended for deletion should be created with:
  - `uses_left: promo_campaign.uses`

### Testing Expectations

- Worker specs should validate:
  - coupon count change
  - operation final status
  - `generated_coupons_count` for success/failure/partial failure
  - success/failure mailer call
- If a spec around metadata behaves unexpectedly after `reload`, first verify test DB schema is up to date for `promo_campaign_operations.metadata`.
