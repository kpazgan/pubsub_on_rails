# PubSub on Rails

PubSub on Rails is a gem facilitating opinionated approach to leveraging publish/subscribe messaging pattern in Ruby on Rails applications.

There are many programming techniques that are powerful yet complex. The beauty of publish/subscribe patterns is that it is powerful while staying simple.

Instead of using callbacks or directly and explicitly executing series of actions, action execution is requested using an event object combined with event subscription.
This helps in keeping code isolation high, and therefore makes large codebases maintainable and testable.

While it has little to do with event sourcing, it encompasses a couple of ideas related to domain-driven development.
Therefore it is only useful in applications in which domains/bounded-contexts can be identified.
This is especially true for applications covering many side effects, integrations and complex business logic.

## Installation

```ruby
# Gemfile

gem 'pubsub_on_rails', '~> 0.0.7'

# config/initializers/pub_sub.rb

PubSub::Subscriptions.subscriptions_path = Rails.root.join('config/subscriptions.yml')
PubSub::Subscriptions.load!
```

## Entities

There are five entities that are core to PubSub on Rails: domains, events, event publishers, event handlers and subscriptions.

### Domain

Domain is simply a named context in application. You can refer to it as "module", "subsystem", "engine", whatever you like.
Good names for domains are "ordering", "messaging", "logging", "accounts", "logistics" etc.
Your app does not need to have code isolated inside domains, but using Component-Based Rails Applications concept (CBRA) sounds like a nice idea to be combined with PubSub on Rails.

Domain example:

```ruby
# app/domains/messaging.rb

module Messaging
  extend PubSub::Domain
end
```

### Event

Event is basically an object indicating that something has happened (event has occured).
There are two important things that need to be considered when planning an event: its **name** and its **payload** (fields).

Name of event should describe an action that has just happened, also it should be namespaced with the name of the domain it has occurred within.
Some examples of good event names: `Ordering::OrderCancelled`, `Messaging::IncorrectLoginNotificationSent`, `Accounts::UserCreated`, `Bookings::CheckinDateChanged`, `Reporting::MonthlySalesReportGenerationRequested`

Payload of event is just simple set of fields that should convey critical information related to the event.
As the payload is very important for each event (it acts as a contract between publisher and handler), PubSub on Rails leverages `Dry::Struct` and `Dry::Types` to ensure both presence and correct type of attributes events are created with.
It is a good rule of a thumb not to create too many fields for each event and just start with the minimal set. It is easy to add more fields to event's payload later (while it might be cumbersome to remove or change them).

Event example:

```ruby
# app/events/ordering/order_created_event.rb

module Ordering
  class OrderCreatedEvent < PubSub::DomainEvent
    attribute :order_id, Types::Strict::Integer
    attribute :customer_id, Types::Strict::Integer
    attribute :line_items, Types::Strict::Array
    attribute :total_amount, Types::Strict::Float
    attribute :comment, Types::Strict::String.optional
  end
end
```

### Event publisher

Event publisher is any class capable of emitting an event.
Usually a great places to start emitting events are model callbacks, service objects or event handlers.
It is very preferable to emit one specific event from only one place, as in most cases this makes the most sense and makes the whole solution more comprehensible.

Event publisher example:

```ruby
# app/models/order.rb

class Order < ApplicationRecord
  include PubSub::Emit

  belongs_to :customer
  has_many :line_items

  #...

  after_create do
    emit(:ordering__order_created, order_id: id)
  end
end
```

### Event handler

Event handler is a class that encapsulates logic that should be executed in reaction to event being emitted.
One event can be handled by many handlers, but only one unique handler within each domain.
Event handlers can be executed synchronously or asynchronously. The latter is recommended for both performance and error-recovery reasons.

Event handler example:

```ruby
# app/event_handlers/messaging/ordering_order_created_handler.rb

module Messaging
  class OrderingOrderCreatedHandler < PubSub::DomainEventHandler
    def call
      OrderMailer.order_creation_notification(order).deliver_now
    end

    private

    def order
      Order.find(event_data.order_id)
    end
  end
end
```

All fields of event's payload are accessible through `event_data` method, which is a simple struct.

#### Conditionally processing events

If in any case you would like to control if given handler should be executed or not (maybe using feature flags), you can override `#process_event?` method.

```ruby
# app/event_handlers/messaging/ordering_order_created_handler.rb

module Messaging
  class OrderingOrderCreatedHandler < PubSub::DomainEventHandler
    # ...

    private

    def process_event?
      Features.notifications_enabled?
    end
  end
end
```

### Subscription

Subscription is "the glue", the binds events with their corresponding handlers.
Each subscription binds one or all events with one handler.
Subscription defines if given handler should be executed in synchronous or asynchronous way.

Subscription example:

```yaml
# config/subscriptions.yml

messaging:
  ordering__order_created: async
```

## Testing

Most of entities in Pub/Sub approach should be tested, yet both domain and event classes can be tested implicitly.
It is recommended to start testing from testing subscription itself, then ensure that both event emission and handling are in place. Depending on situation the recommended order may change though.

### RSpec

The recommended RSpec configuration is as follows:

```ruby
# spec/support/pub_sub.rb
 
require 'pub_sub/testing'

RSpec.configure do |config|
  config.include Wisper::RSpec::BroadcastMatcher
  config.include PubSub::Testing::SubscriptionsHelper
  config.include PubSub::Testing::EventDataHelper

  config.before(:suite) do
    PubSub::Testing::SubscriptionsHelper.clear_wisper_subscriptions!
  end

  config.around(:each, subscribers: true) do |example|
    domain_name = example.metadata[:described_class].to_s.underscore
    PubSub.subscriptions.register(domain_name)
    example.run
    clear_wisper_subscriptions!
  end
end
```

### Testing subscription

Testing subscription is as easy as telling what domains should subscribe to what event in what way.
Example of `subscribe_to` matcher can be found [here](https://gist.github.com/stevo/bd11c8dae812c919de6a61d1292dbfe1).
Example:

```ruby
RSpec.describe Messaging, subscribers: true do
  it { is_expected.to subscribe_to(:ordering__order_created).asynchronously }
end
```

### Testing publishers

To test publisher it is crucial to test if event was emitted (broadcasted) under certain conditions (if any).

Example:

```ruby
RSpec.describe Order do
  describe 'after_create' do
    it 'emits ordering__order_created' do
      customer = create(:customer)
      line_items = create_list(:line_item, 2)

      expect {
        Order.create(
          customer: customer,
          total_amount: 100.99,
          comment: 'Small order',
          line_items: line_items
        )
      }.to broadcast(
             :ordering__order_created,
             order_id: fetch_next_id_for(Order),
             total_amount: 100.99,
             comment: 'Small order',
             line_items: line_items
           )
    end
  end
end
```

### Testing handlers

Handlers can be tested by testing their `call!` method, that calls `call` behind the scenes.
To ensure event payload contract is met, please use `event_data_for` helper to build event payload hash.
It will instantiate event object behind the scenes to ensure it exists and its payload requirements are met.

Example:

```ruby
module Messaging
  RSpec.describe OrderingOrderCreatedHandler do
    describe '#call!' do
      it 'delivers order creation notification' do
        order = create(:order)
        event_data = event_data_for(
          'ordering__order_created',
          order_id: order.id,
          total_amount: 100.99,
          comment: 'Small order',
          line_items: [build(:line_item)]
        )
        order_creation_notification = double(:order_creation_notification, deliver_now: true)
        allow(OrderMailer).to receive(:order_creation_notification).
          with(order).and_return(order_creation_notification)

        OrderingOrderCreatedHandler.new(event_data).call!

        expect(order_creation_notification).to have_received(:deliver_now)
      end
    end
  end
end
```
### Integration testing

By default all subscriptions are cleared in testing environment to enforce testing in isolation.
To enable integration testing, a test can be wrapped in block provided by `with_subscription_to` helper.
This way events emitted by code executed within this block will be handled by handlers from domains provided as arguments to `with_subscription_to` helper.

```ruby
it 'does something' do
    with_subscription_to('messaging', 'logging') do
      # setup, subject, assertions etc.
    end
end
```

### Subscriptions linting

It is a common problem to implement a publisher and handler and forget about implementing subscription.
Without proper integration testing the problem might stay undetected before identified (hopefully) during manual testing.
This is where subscriptions linting comes into play. All existing event handlers will be verified against registered subscriptions during linting process.
In case of any mismatch, exception will be raised.

To lint subscriptions, place `PubSub::Subscriptions.lint!` for instance in your `rails_helper.rb` or some initializer of choice.

## Logger

Even though default domain always routes event subscriptions to correspondingly named event handlers, it is possible to implement domains that will route subscriptions in the different way.
The simplest way is to define it manually:

```ruby
# app/domains/logging.rb

module Messaging
  def self.ordering__order_created(event_payload)
    # whatever you need goes here
  end
end
```

This technique can be useful for instance for logging

```ruby
# app/domains/logging.rb

module Logging
  def self.event_logger
    @event_logger ||= Logger.new("#{Rails.root}/log/#{Rails.env}_event_logger.log")
  end

  def self.method_missing(method_name, *event_data)
    event_logger.info("Evt: #{method_name}: \n#{event_data.map(&:to_json).join(', ')}\n\n")
  end

  def self.respond_to_missing?(method_name, include_private = false)
    method_name.to_s.start_with?(/[a-z_]+__/) || super
  end
end
```

```yaml
# config/subscriptions.yml

logging:
  all_events: sync
```

# `emit` vs `broadcast`

PubSub on Rails leverages `wisper` and `wisper-sidekiq` under the hood.
This is why instead of using `emit`, you can broadcast events by using wisper's `broadcast` method.

```ruby
# app/models/order.rb

class Order < ApplicationRecord
  include Wisper::Publisher

  #...

  after_create do
    broadcast(
      :ordering__guest_checked_in,
      order_id: id,
      line_items: line_items,
      total_amount: total_amount,
      comment: comment
    )
  end
end
```

Why `emit` then? `emit` provides a couple of simplifications that make emitting events easier and more reliable.

## Payload verification

Every time event is emitted, its payload is supplied to corresponding event class and is verified.
This ensures that whenever we emit event we can be sure its payload is matching specification.

Example:

```ruby
module Accounts
  class PersonCreatedEvent < DomainEvent
    attribute :person_id, Types::Strict::Integer
  end
end
```

* `emit(:accounts__person_created, person_id: 1)` is ok
* `emit(:accounts__person_created)` will result in ```PubSub::EventEmission::EventPayloadArgumentMissing: Event [Accounts::PersonCreatedEvent] expects [person_id] payload attribute to be either exposed as [person] method in emitting object or provided as argument```
* `emit(:accounts__person_created, person_id: 'abc')` will result in ```Dry::Struct::Error: [Accounts::PersonCreatedEvent.new] "abc" (String) has invalid type for :person_id violates constraints (type?(Integer, "abc") failed)```

## Automatic event name prefixing

When you namespace your code to match your domain names, you can skip prefixing an event name with domain name when emitting it.

```ruby
# app/models/oriering/order.rb

module Ordering
  class Order < ApplicationRecord
    include PubSub::Emit

    after_create do
      emit(:order_created, order_id: id)
      # emit(:ordering__order_created, order_id: id) # this will work as well
    end
  end
end
```

## Automatic event payload population

Whenever you emit an event, it will try to populate its payload with data using public interface of object it is emitted from within. 

```ruby
# app/models/oriering/order.rb

module Ordering
  class Order < ApplicationRecord
    include PubSub::Emit

    after_create do
      emit(:order_created, order_id: id)
      # emit(
      #   :ordering__order_created,
      #   order_id: id, # `self` does not implement `order_id`, therefore value has to be provided explicitly here
      #   total_amount: total_amount, # attribute matches the name of method on `self`, therefore it can be skipped
      #   comment: comment # same here
      # )
    end
  end
end
```

## Manually broadcasting events

If for some reason you need to safely broadcast some event, the best way is to instantiate its class and call `#broadcast!` on it.

Example:

```ruby
Ordering::OrderCreatedEvent.new(order_id: 1, total_amount: 25, line_items: []).broadcast!
```

# TODO

- Dynamic event classes
- TYPES declaration
