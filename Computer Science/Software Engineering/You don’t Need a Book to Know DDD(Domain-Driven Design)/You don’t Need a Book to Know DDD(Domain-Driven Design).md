#software-engineering #domain-modeling-driven 

![[Pasted image 20241115123905.png]]

It took me a while to figure out the patterns behind DDD, though the most important thing are ubiquitous language and bounded context. In this article, I don’t want to address the philosophical part of DDD, instead I want to dive into the practical implementation of some parts to connect with the philosophy behind it. If you are not familiar with the two most important concepts behind DDD, there are countless articles out there explaining them, just go and search them.

Somewhere I read this conclusion of DDD patterns and I found it quite good:

> You model your business using Entities (the ID matters) and Value Objects (the values matter). You use Repositories to retrieve and store them. You create them with the help of Factories. If an object is too complex for a single class, you’ll create Aggregates that will bind Entities & Value Objects under the same root. If a business logic doesn’t belong to a given object, you’ll define Services that will manipulate the involved elements. Eventually, when the state of the business changes (a change that matters to business experts), you’ll publish Domain Events to communicate the change.

Breaking down each concept will help piece up the whole picture of DDD.

I just use some Rust example, they are very readable I promise!

## 1. Entities

Entities are objects that have a distinct identity that runs through time and different states. The identity is usually represented by an ID.

```rust
// src/entities/user.rs
pub struct User {
    pub id: u32,
    pub username: String,
    pub email: String,
    pub age: u8,
}

impl User {
    pub fn new(id: u32, username: String, email: String, age: u8) -> Self {
        User { id, username, email, age }
    }
}
```

## 2. Value Objects

Value objects are objects that are defined by their attributes. They do not have a distinct identity.

```rust
// src/value_objects/address.rs
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Address {
    pub street: String,
    pub city: String,
    pub zip_code: String,
}

impl Address {
    pub fn new(street: String, city: String, zip_code: String) -> Self {
        Address { street, city, zip_code }
    }
}
```

## 3. Repositories

Repositories are used to retrieve and store entities. They act as a collection of entities.

```rust
// src/repositories/user_repository.rs
use crate::entities::user::User;

pub struct UserRepository {
    users: Vec<User>,
}

impl UserRepository {
    pub fn new() -> Self {
        UserRepository { users: Vec::new() }
    }

    pub fn add(&mut self, user: User) {
        self.users.push(user);
    }

    pub fn find_by_id(&self, id: u32) -> Option<&User> {
        self.users.iter().find(|&user| user.id == id)
    }
}

```
## 4. Factories

Factories are used to create complex objects and aggregates.

```rust
// src/factories/user_factory.rs
use crate::entities::user::User;

pub struct UserFactory;

impl UserFactory {
    pub fn create_user(id: u32, username: String, email: String, age: u8) -> User {
        User::new(id, username, email, age)
    }
}

```
## 5. Aggregates

Aggregates are clusters of entities and value objects that are treated as a single unit.

```rust
// src/aggregates/order.rs
use crate::entities::user::User;
use crate::value_objects::address::Address;

pub struct Order {
    pub id: u32,
    pub user: User,
    pub shipping_address: Address,
}

impl Order {
    pub fn new(id: u32, user: User, shipping_address: Address) -> Self {
        Order { id, user, shipping_address }
    }
}
```

## 6. Services

Services contain business logic that doesn’t naturally fit within an entity or value object.

```rust
// src/services/order_service.rs
use crate::aggregates::order::Order;
use crate::entities::user::User;
use crate::value_objects::address::Address;

pub struct OrderService;

impl OrderService {
    pub fn create_order(user: User, shipping_address: Address) -> Order {
        let order_id = 1; // In a real application, this would be generated
        Order::new(order_id, user, shipping_address)
    }
}

```
Key Points about Services are:

**1.Stateless**: Services are typically stateless. They do not hold any state themselves but operate on the state of entities and value objects.

**2.Encapsulation of Business Logic**: They encapsulate business logic that spans multiple entities or value objects or that doesn’t fit neatly within a single entity or value object.

**3.Coordination**: They often coordinate interactions between multiple entities and value objects.

## 7. Domain Events

Domain events are used to communicate changes in the state of the business.

```rust
// src/events/user_registered.rs
pub struct UserRegistered {
    pub user_id: u32,
    pub username: String,
}

impl UserRegistered {
    pub fn new(user_id: u32, username: String) -> Self {
        UserRegistered { user_id, username }
    }
}
```

They are typically published to an event bus or event store, which can then be used to notify other parts of the system or external systems about these events.

Where Domain Events Are Published:

**1.Event Bus**: Domain events are often published to an in-memory event bus within the application. This allows other parts of the application to subscribe to and handle these events. Since it is in-memory, the events are ephemeral and will be lost if the application restarts or crashes. It is suitable for scenarios where events need to be processed quickly and do not require persistence, such as inter-component communication within a single application instance.

**2.Event Store**: For more complex scenarios, domain events can be published to an event store, such as AWS EventBridge, Kafka, or a custom event store. This allows for durable storage and replay of events. Think of it as just:

```rust
pub struct EventStore {
    file_path: String,
}

```

An event store can serve as the single source of truth in an event-driven architecture, particularly in the context of event sourcing.

In event sourcing, every change to the state of an application is captured as an event and stored in the event store. The current state of the application can be reconstructed by replaying these events.

A quick take-away is events are immutable and represent facts that have occurred. Once an event is stored, it is never changed or deleted.

![[Pasted image 20241115124216.png]]

**3.Message Brokers**: Domain events can also be published to message brokers like RabbitMQ or AWS SNS/SQS for asynchronous processing and integration with other systems.

# Putting It All Together

```rust
mod entities {
    pub mod user;
}

mod value_objects {
    pub mod address;
}

mod repositories {
    pub mod user_repository;
}

mod factories {
    pub mod user_factory;
}

mod aggregates {
    pub mod order;
}

mod services {
    pub mod order_service;
}

mod events {
    pub mod user_registered;
}

use entities::user::User;
use value_objects::address::Address;
use repositories::user_repository::UserRepository;
use factories::user_factory::UserFactory;
use services::order_service::OrderService;
use events::user_registered::UserRegistered;

fn main() {
    // Create a user using the factory
    let user = UserFactory::create_user(1, String::from("Alice"), String::from("alice@example.com"), 30);

    // Create a repository and add the user
    let mut user_repo = UserRepository::new();
    user_repo.add(user);

    // Find the user by ID
    if let Some(user) = user_repo.find_by_id(1) {
        println!("User found: {}", user.username);
    }

    // Create an address
    let address = Address::new(String::from("123 Main St"), String::from("Hometown"), String::from("12345"));

    // Create an order using the service
    let order = OrderService::create_order(user.clone(), address);

    // Publish a domain event, typically published to an event bus or event store, 
   // which can then be used to notify other parts of the system or external systems about these events.
    let event = UserRegistered::new(user.id, user.username.clone());
    println!("Domain event: UserRegistered for user {}", event.username);
}
```

A key point take-away is that factories focus on object creation, while services focus on business logic and operations.

