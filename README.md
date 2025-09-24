 # Microservices Event Sourcing using Kafka

The transcript describes **Event Sourcing**, a design pattern where all changes to an application's state are stored as a sequence of immutable events rather than just saving the final state. While the transcript mentions "secrets," this is likely a misinterpretation of **CQRS (Command Query Responsibility Segregation)**, a pattern very commonly used with Event Sourcing [, ].[^2]

### Key Concepts and Benefits

Event Sourcing offers several advantages over traditional data storage methods where you would typically update a row in a database.[^7]

* **Complete Audit Trail**: Since every change is saved as an event, you have a complete, unchangeable history of the entity. This is invaluable for auditing and debugging.[^2]
* **Recreate Historical State**: You can determine the state of an object at any point in time by replaying the events up to that point.[^8]
* **Improved Write Performance**: Writing events is a simple append-only operation, which is very fast. There are no slow `UPDATE` or `DELETE` operations on the event log.[^2]
* **Disaster Recovery**: The event log can be used as the definitive source of truth to rebuild read models or other databases in case of failure.


### Beginner-Friendly Code Example: A Bank Account

Let's model a simple bank account using Event Sourcing in C\#. In a traditional approach, you might have a single `Balance` property that you update directly. With Event Sourcing, we record the actions themselves.

#### 1. Define the Events

First, we define the immutable events that can happen to our account. C\# `record` types are perfect for this as they are immutable by design.[^1]

```csharp
// Base interface for all our events
public interface IEvent {}

// Event for when a new account is created
public record AccountCreated(Guid AccountId, decimal InitialBalance) : IEvent;

// Event for when money is deposited
public record MoneyDeposited(decimal Amount) : IEvent;

// Event for when money is withdrawn
public record MoneyWithdrawn(decimal Amount) : IEvent;
```


#### 2. Create the Aggregate

The **Aggregate** is the entity that processes commands and produces events. In our case, it's the `BankAccount`. Notice how its properties have `private set` to prevent direct changes from outside.

```csharp
public class BankAccount
{
    public Guid Id { get; private set; }
    public decimal Balance { get; private set; }
    public int Version { get; private set; } = 0;

    // A private list to track the changes (new events)
    private readonly List<IEvent> _uncommittedEvents = new();

    // Private constructor ensures it's created via events
    private BankAccount() { }

    // Public method to process a deposit command
    public void Deposit(decimal amount)
    {
        if (amount <= 0)
        {
            throw new InvalidOperationException("Deposit amount must be positive.");
        }
        var depositEvent = new MoneyDeposited(amount);
        Apply(depositEvent); // Apply the event to the current state
        _uncommittedEvents.Add(depositEvent); // Add to our list of new events
    }
    
    // A private method to apply an event to the state
    private void Apply(IEvent anEvent)
    {
        switch (anEvent)
        {
            case AccountCreated e:
                Id = e.AccountId;
                Balance = e.InitialBalance;
                break;
            case MoneyDeposited e:
                Balance += e.Amount;
                break;
            case MoneyWithdrawn e:
                Balance -= e.Amount;
                break;
        }
        Version++;
    }

    // A static method to rebuild the account's state from its history
    public static BankAccount RebuildFrom(Guid id, IEnumerable<IEvent> history)
    {
        var account = new BankAccount();
        foreach (var pastEvent in history)
        {
            account.Apply(pastEvent);
        }
        return account;
    }
}
```

In a real application, the `_uncommittedEvents` list would be saved to a specialized **Event Store** database like EventStoreDB or using a library like Marten with PostgreSQL [, ].

### Real-World Use Cases

Event Sourcing is particularly useful in systems where the history of an entity is as important as its current state.

* **E-commerce Platforms**: Tracking an order's lifecycle (`OrderPlaced`, `PaymentProcessed`, `ItemShipped`, `OrderDelivered`). This provides a full history for customer service and analytics.[^6]
* **Financial Systems**: For bank accounts, ledgers, and stock trading, an immutable log of all transactions (`MoneyDeposited`, `TransferSent`) is a legal and business requirement [, ].
* **Inventory Management**: Recording every stock movement (`ItemReceived`, `StockReserved`, `ItemShipped`) ensures a perfect audit trail and helps diagnose discrepancies.
* **Collaborative Applications**: In a tool like Google Docs, every keystroke or formatting change can be an event, allowing for features like viewing version history and collaborative editing.


### Summary and Interview Tips

For a .NET interview, focusing on these key points will demonstrate a strong understanding of Event Sourcing.

* **Explain the Core Idea**: Be ready to explain that Event Sourcing is about persisting state as a sequence of events, not just the current snapshot.
* **Connect it to CQRS**: Mentioning that Event Sourcing is often the "write side" of a CQRS architecture shows a deeper understanding. The event stream is optimized for writes, while separate "read models" are built from these events for fast queries.[^8]
* **Discuss the "Why"**: Emphasize the benefits: full auditability, the ability to debug by replaying history, and high-performance, append-only writes.
* **Acknowledge the Trade-offs**: Show that you have a balanced view. Event Sourcing can introduce complexity, such as "eventual consistency" (where read models might be slightly out of date) and the need to version events over time.
* **Name Key Technologies**: Mentioning popular .NET tools for Event Sourcing like **Marten** (for PostgreSQL) or dedicated databases like **EventStoreDB** shows you are aware of the ecosystem [, ]. You can also mention messaging systems like RabbitMQ or Kafka which are often used to publish events for other services to consume.

# Overall Architecture

This transcript describes a classic implementation of the **Command Query Responsibility Segregation (CQRS)** pattern combined with **Event Sourcing** for a social media application's backend. Here is a breakdown of the main points with code examples and interview advice.

### Summary of the Architecture

The architecture separates the application into two distinct microservices: a **Command API** for handling all data changes (writes) and a **Query API** for handling all data reads. This separation allows each service to be optimized and scaled independently.[^3][^4]

* **Command API (The "Write" Side)**: Manages all state changes, such as creating a new post, adding a comment, or liking a post. It uses Event Sourcing, meaning it doesn't just update data; it records each action as an immutable event in a write-optimized database (**MongoDB**).
* **Query API (The "Read" Side)**: Manages all data retrieval, such as finding all posts or fetching posts by a specific author. It uses a separate, read-optimized database (**SQL Server**) that is specifically designed for efficient querying.
* **Event Bus (The "Glue")**: An **Apache Kafka** message broker acts as the communication channel between the two services. When the Command API writes a new event, it publishes that event to Kafka. The Query API subscribes to these events to update its own read database, a process which leads to "eventual consistency."


### The Flow of Data Explained

The transcript details the lifecycle of a command (a write operation) and a query (a read operation).

#### Command Flow (e.g., Creating a New Post)

1. A client sends a `NewPostCommand` via an HTTP POST request to the **Command API**.
2. The API controller receives the command and uses a **dispatcher** to send it to the correct **command handler**.
3. The handler creates a `PostAggregate` (the business object) and applies the command, which raises a `PostCreatedEvent`.
4. This new event is saved to the **MongoDB Event Store**.
5. After successful persistence, the `PostCreatedEvent` is published to an **Apache Kafka** topic.

#### Query Flow (e.g., Finding All Posts)

1. An **event consumer** in the **Query API** listens to the Kafka topic. It receives the `PostCreatedEvent`.
2. An **event handler** processes this event and updates the **SQL Server read database**. For a `PostCreatedEvent`, it would insert a new record into the `Posts` table.
3. Later, a client sends a `FindAllPostsQuery` via an HTTP GET request to the **Query API**.
4. The controller dispatches this query to a **query handler**.
5. The handler executes a simple, fast query against the SQL Server read database and returns the data.

### Beginner-Friendly .NET Code Examples

The transcript mentions several interfaces like `ICommandDispatcher` and `ICommandHandler`. In modern .NET, this pattern is most easily implemented using the popular library **MediatR**, which handles the dispatching logic for you.[^2]

#### 1. Defining a Command

A command is a simple object that represents an intent to change something. A C\# `record` is a good choice as it's typically just a data carrier.

```csharp
// The command object, which implements MediatR's IRequest interface.
// It represents the "New Post" action from the transcript.
public record NewPostCommand(Guid PostId, string Author, string Message) : IRequest<Guid>;
```


#### 2. Creating a Command Handler

The handler contains the business logic for a specific command. It takes the command, interacts with the aggregate/event store, and publishes events.

```csharp
// This handler processes the NewPostCommand.
public class NewPostCommandHandler : IRequestHandler<NewPostCommand, Guid>
{
    // In a real app, these would be injected dependencies.
    private readonly IEventStoreRepository _eventStoreRepo;
    private readonly IEventPublisher _eventPublisher;

    public NewPostCommandHandler(IEventStoreRepository eventStoreRepo, IEventPublisher eventPublisher)
    {
        _eventStoreRepo = eventStoreRepo;
        _eventPublisher = eventPublisher;
    }

    public async Task<Guid> Handle(NewPostCommand command, CancellationToken cancellationToken)
    {
        // 1. Create the aggregate and raise the event.
        var aggregate = new PostAggregate(command.PostId, command.Author, command.Message);

        // 2. Persist the event to the write database (e.g., MongoDB).
        await _eventStoreRepo.SaveAsync(aggregate);

        // 3. Publish the event to the event bus (e.g., Kafka).
        await _eventPublisher.PublishAsync(aggregate.GetUncommittedEvents());
        
        return aggregate.Id;
    }
}
```


#### 3. Using the Dispatcher in the Controller

The API controller becomes very simple. It just needs to create the command object and send it via MediatR's `IMediator` interface.[^5]

```csharp
[ApiController]
[Route("api/posts")]
public class PostCommandController : ControllerBase
{
    private readonly IMediator _mediator;

    public PostCommandController(IMediator mediator)
    {
        _mediator = mediator;
    }

    [HttpPost]
    public async Task<IActionResult> NewPost([FromBody] NewPostCommand command)
    {
        var postId = await _mediator.Send(command);
        return StatusCode(201, new { Id = postId });
    }
}
```


### Real-World Use Cases

This architecture is powerful for complex systems where read and write needs differ significantly.[^4]

* **Social Media**: As described, handling a high volume of writes (posts, likes, comments) separately from the high volume of reads (loading feeds).
* **E-commerce**: Write-heavy operations like placing an order, processing payment, and updating inventory are separated from read-heavy operations like browsing products and searching.
* **Banking**: A transaction (`MoneyTransferredCommand`) is a write operation that must be perfectly audited (Event Sourcing), while checking your balance is a simple read query.
* **Booking Systems**: Handling flight or hotel reservations (writes) is a critical, complex process, while searching for available rooms/flights (reads) is a different performance challenge.


### Summary and Interview Tips

When discussing this architecture in an interview, focus on the "why" behind the design choices.

* **Clearly Define CQRS**: Start by stating that CQRS separates the models for reading and writing data. The write model is optimized for command processing and validation, while the read model is optimized for fast queries.[^3]
* **Explain the Scalability Benefit**: Emphasize that you can scale the read and write services independently. If your application gets millions of reads but only thousands of writes, you can add more instances of the Query API without touching the Command API.
* **Connect CQRS and Event Sourcing**: Explain that while they are separate patterns, they work very well together. Event Sourcing provides a perfect, auditable log for the write-side of a CQRS system.
* **Be Ready to Draw It**: An interviewer may ask you to whiteboard this architecture. Practice drawing the boxes: Client -> Command API -> Kafka -> Query API -> Read DB. Show the command and query flows clearly.
* **Discuss the Main Trade-off: Eventual Consistency**: This is the most important concept to show you have a balanced understanding. Because the read database is updated *after* an event is published, there is a small delay. The read model is "eventually consistent" with the write model. Discuss how this is acceptable for most social media features (e.g., a "like" appearing a second later is fine) but might not be for others (e.g., a bank transfer).

# All about Kafka

The provided transcript gives a high-level introduction to Apache Kafka, explaining its origin and purpose as a real-time event streaming platform. Here is a summary with more details on the key components mentioned, along with beginner-friendly .NET examples and interview tips.

### Summary of Apache Kafka

**Apache Kafka** is an open-source, distributed event streaming platform initially created at LinkedIn in 2011 to handle high-throughput data streams [, ]. It has since become the industry standard for building real-time, event-driven applications, capable of processing trillions of events per day. The transcript notes that it will be used as the **event bus** in the course's architecture, connecting the write side (Command API) and the read side (Query API).[^1]

### Key Architectural Components

The transcript mentions several core components of Kafka that are essential to understand [, ].


| Component | Description |
| :-- | :-- |
| **Broker** | A single Kafka server. A group of brokers forms a **Kafka cluster**, which provides scalability and fault tolerance [, ]. |
| **Topic** | A named category or channel to which records (events) are published. For example, you might have a `social-media-posts` topic [, ]. |
| **Partition** | A topic is split into one or more partitions. Each partition is an ordered, immutable sequence of records called a **commit log** [, ]. Partitions are the key to Kafka's parallelism and scalability. |
| **Producer** | A client application that writes or publishes records to a Kafka topic [, ]. In the CQRS architecture, the Command API acts as a producer. |
| **Consumer** | A client application that subscribes to one or more topics and reads and processes records [, ]. The Query API acts as a consumer. Consumers track their progress using an **offset**, which is a unique ID for each record in a partition [^7]. |
| **ZooKeeper** | An external service that was traditionally used to manage and coordinate the Kafka cluster, handling tasks like tracking broker status and electing leaders for partitions. Newer versions of Kafka are replacing ZooKeeper with an internal component called KRaft [, ]. |

### Beginner-Friendly .NET Code Examples

To interact with Kafka from a .NET application, you typically use a client library. The most popular one is `Confluent.Kafka`, provided by Confluent, the company founded by the creators of Kafka.

#### 1. Creating a Producer

This example shows how a .NET application (like the Command API) can produce a message to a Kafka topic.

```csharp
using Confluent.Kafka;
using System.Threading.Tasks;

public class KafkaProducer
{
    private readonly IProducer<string, string> _producer;

    public KafkaProducer(string bootstrapServers)
    {
        var config = new ProducerConfig { BootstrapServers = bootstrapServers };
        _producer = new ProducerBuilder<string, string>(config).Build();
    }

    public async Task ProduceMessageAsync(string topic, string key, string value)
    {
        var message = new Message<string, string> { Key = key, Value = value };
        
        // Asynchronously send the message to the specified topic
        var deliveryResult = await _producer.ProduceAsync(topic, message);
        
        Console.WriteLine($"Delivered '{deliveryResult.Value}' to '{deliveryResult.TopicPartitionOffset}'");
    }
}

// How to use it:
// var producer = new KafkaProducer("localhost:9092");
// await producer.ProduceMessageAsync("social-media-posts", "post-123", "{'event':'PostCreated', 'author':'John'}");
```

* **BootstrapServers**: The address of one or more Kafka brokers. The client only needs one to connect to the entire cluster.
* **Key**: The message key (`post-123`). All messages with the same key are guaranteed to go to the same partition, which ensures their order of processing.[^5]


#### 2. Creating a Consumer

This example shows how a background service in another application (like the Query API) can consume messages from a topic.

```csharp
using Confluent.Kafka;
using System;
using System.Threading;

public class KafkaConsumer
{
    private readonly IConsumer<string, string> _consumer;

    public KafkaConsumer(string bootstrapServers, string groupId)
    {
        var config = new ConsumerConfig
        {
            BootstrapServers = bootstrapServers,
            GroupId = groupId, // Identifies the consumer group
            AutoOffsetReset = AutoOffsetReset.Earliest // Start reading from the beginning of the topic
        };
        _consumer = new ConsumerBuilder<string, string>(config).Build();
    }

    public void StartConsuming(string topic, CancellationToken cancellationToken)
    {
        _consumer.Subscribe(topic);

        while (!cancellationToken.IsCancellationRequested)
        {
            try
            {
                // Poll for new messages
                var consumeResult = _consumer.Consume(cancellationToken);
                
                Console.WriteLine($"Consumed message '{consumeResult.Message.Value}' from topic '{consumeResult.Topic}'");
                
                // In a real app, you would process the message here (e.g., update the read database)
            }
            catch (ConsumeException e)
            {
                Console.WriteLine($"Error consuming: {e.Error.Reason}");
            }
        }
        _consumer.Close();
    }
}

// How to use it:
// var consumer = new KafkaConsumer("localhost:9092", "query-api-group");
// consumer.StartConsuming("social-media-posts", new CancellationToken());
```

* **GroupId**: This is a crucial concept. All consumers with the same `GroupId` work together to process messages from a topic. Kafka ensures that each partition is only consumed by one consumer within the group at any given time, which is how it achieves parallel processing.[^1]


### Summary and Interview Tips

When discussing Kafka in a .NET interview, focus on its role in a distributed system.

* **Explain Its Purpose**: Describe Kafka as a "distributed commit log" used for building real-time, event-driven architectures. Emphasize that it decouples services. The producer doesn't need to know who is consuming the message, and vice-versa.[^5]
* **Know the Core Components**: Be able to confidently define Topic, Partition, Producer, Consumer, and Broker. Explaining that partitions are the unit of parallelism is a key point.
* **Mention "At-Least-Once" Delivery**: By default, Kafka provides "at-least-once" message delivery guarantees. This means a message will never be lost but could, in rare failure scenarios, be delivered more than once. Mention that consumers must be designed to handle this (i.e., be idempotent).
* **Discuss Key vs. No Key**: Explain the importance of message keys. A key ensures all events for a specific entity (like a single social media post) go to the same partition and are processed in order. If no key is provided, Kafka distributes messages in a round-robin fashion for load balancing [, ].
* **Acknowledge ZooKeeper's Role (and its Decline)**: Mentioning that ZooKeeper handles cluster coordination shows historical knowledge. Adding that newer versions are replacing it with KRaft to simplify operations demonstrates that your knowledge is up-to-date.

### Summary of Prerequisites and Tools

The transcript lists all the tools needed for the course, emphasizing a cross-platform approach (Windows, macOS, Linux) and the use of Docker to simplify the setup of infrastructure services.


| Tool / Technology | Version Mentioned | Purpose |
| :-- | :-- | :-- |
| **.NET SDK** | .NET 6 | The core software development kit needed to build and run .NET applications. The transcript notes to use the latest stable version if .NET 6 is outdated [, ]. |
| **IDE / Code Editor** | N/A | For writing C\# code. The recommended options are **Visual Studio 2022**, **VS Code**, or **JetBrains Rider**. |
| **Postman** | N/A | A client tool for making HTTP requests to test the web APIs. |
| **Docker** | 20.10.7+ | A containerization platform used to easily run **Apache Kafka**, **MongoDB**, and **SQL Server** in isolated containers, avoiding complex local installations. |

### .NET Development Environment Setup

The transcript provides a step-by-step guide for setting up the development environment, including how to verify each installation.

#### 1. Install the .NET SDK

The first step is to install the .NET Software Development Kit. The SDK includes everything needed to build and run .NET applications, including the runtime and command-line tools [, ].[^2]

* **To Install**: Download the installer for your operating system from the official .NET website.
* **To Verify**: After installation, open a terminal or command prompt and run the following command. It will display the installed SDK version.

```bash
dotnet --version
```


#### 2. Choose and Configure a Code Editor

You need an editor to write code. The transcript highlights VS Code as a lightweight option that becomes powerful with extensions.[^10]

* **Visual Studio 2022**: A full-featured IDE popular among .NET developers.
* **JetBrains Rider**: A cross-platform .NET IDE known for its intelligent features.
* **Visual Studio Code (VS Code)**: A free, lightweight editor. If using VS Code, the transcript recommends installing these essential extensions from the marketplace:
    * **C\# for Visual Studio Code**: Provides core C\# support like syntax highlighting and IntelliSense.
    * **NuGet Package Manager**: A GUI for managing project dependencies (packages).
    * **SQL Server (mssql)**: A client for connecting to and querying a Microsoft SQL Server database directly from the editor.


#### 3. Install API Testing and Containerization Tools

* **Postman**: Download and install from the official Postman website to create and send HTTP requests to the APIs you will build.
* **Docker Desktop**: Download and install for your OS (Windows/Mac) from the Docker website. This tool allows you to run applications and databases in containers.
* **To Verify Docker**: Run the following command in a terminal to check the installed version.

```bash
docker --version
```


### Docker Configuration and Commands

The transcript introduces Docker as a way to manage the project's infrastructure (databases and message broker). It explains the basic commands to get started.

#### 1. Creating a Docker Network

A custom Docker network is created so that all the containers (Kafka, MongoDB, SQL Server) and the microservices can communicate with each other by name.

* **Command to Create Network**:

```bash
docker network create --driver bridge --attachable my-docker-network
```

    * `--driver bridge`: Creates a standard isolated network on the host machine.
    * `--attachable`: Allows you to attach other containers to this network after it's created.
    * `my-docker-network`: The custom name given to the network.
* **Command to List Networks**:

```bash
docker network ls
```


#### 2. Running Commands with Privileges (`sudo`)

The transcript correctly notes that on Linux, Docker commands often require administrative privileges and must be prefixed with `sudo`. On Windows, the equivalent is running Command Prompt or PowerShell "As Administrator."

* **Example (Linux)**:

```bash
sudo docker ps
```

This command lists all currently running Docker containers.


### Summary and Interview Tips

While a setup guide is not a typical interview topic, understanding the "why" behind these tools demonstrates practical experience.

* **Articulate the Role of the .NET SDK vs. Runtime**: Be able to explain that the **SDK (Software Development Kit)** is for developers (it contains compilers and tools to build apps), while the **Runtime** is for end-users (it's only needed to run already-built apps). Installing the SDK includes the runtime.[^6]
* **Explain Why Docker is Used**: State that Docker simplifies development and deployment by packaging applications and their dependencies into standardized units called **containers**. This ensures the application runs the same way everywhere (developer's laptop, testing, production) and avoids the "it works on my machine" problem.
* **Describe a Docker Network**: Explain that Docker networks allow isolated containers to communicate with each other. A **bridge network** is a private, internal network for containers on a single host. Using a custom bridge network allows containers to resolve each other's addresses by their container names, which is simpler and more reliable than using IP addresses.
* **Show You're Current**: The course uses .NET 6. Mentioning that the current LTS (Long-Term Support) version is .NET 8 would show that your knowledge is up-to-date. LTS versions are important for enterprise applications because they are supported by Microsoft for a longer period.


# Setting up Docker Compose with Kafka and Zookeeper Containers

 This transcript explains how to use **Docker Compose** to set up and run Apache Kafka and its dependency, Apache ZooKeeper. This approach simplifies the management of multi-container applications.

### Summary of Docker Compose for Kafka

**Docker Compose** is a tool for defining and running multi-container Docker applications [, ]. It uses a YAML file (by default, `docker-compose.yml`) to configure an application's services, networks, and volumes. The transcript outlines using Docker Compose to deploy Kafka and ZooKeeper, which are required for the event-driven architecture of the course.

* **Installation**: The transcript directs users to the official Docker documentation to install Docker Compose. It is important to note that for modern versions of **Docker Desktop** on Windows and Mac, Docker Compose is already included and can be run as `docker compose` (with a space) [, ]. The older standalone versions used `docker-compose` (with a hyphen).[^1]
* **Verification**: Once installed, you can verify the version by running `docker compose --version` in the terminal.


### The `docker-compose.yml` File Explained

The core of the setup is the `docker-compose.yml` file. The transcript breaks down a sample file that defines two main **services**: `zookeeper` and `kafka`.

#### 1. The `zookeeper` Service

Apache ZooKeeper is described as a mandatory dependency for older Kafka clusters. It manages the state of the Kafka brokers, such as tracking which brokers are online and electing a "leader" for partitions [, ].

```yaml
services:
  zookeeper:
    image: 'bitnami/zookeeper:latest' # The Docker image to use
    ports:
      - '2181:2181' # Maps host port 2181 to container port 2181
    environment:
      - ALLOW_ANONYMOUS_LOGIN=yes # A simple configuration for local development
    volumes:
      - 'zookeeper_data:/bitnami' # Persists Zookeeper data
```


#### 2. The `kafka` Service

This service defines the main Kafka broker.

```yaml
  kafka:
    image: 'bitnami/kafka:latest' # The Kafka Docker image from Bitnami
    ports:
      - '9092:9092' # Exposes the main Kafka port for clients to connect
    environment:
      # Tells Kafka where to find Zookeeper
      - KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181 
      # Advertises its address to external clients (e.g., your .NET app)
      - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092 
      - ALLOW_PLAINTEXT_LISTENER=yes
    volumes:
      - 'kafka_data:/bitnami/kafka' # Persists Kafka topics and messages
    depends_on:
      - zookeeper # Ensures Zookeeper starts before Kafka
```


#### 3. Top-Level `volumes` and `networks`

The file also defines persistent volumes and connects the services to the network created in the previous step.

```yaml
volumes:
  zookeeper_data:
  kafka_data:

networks:
  default:
    external:
      name: my-docker-network
```

* **Persistent Volumes**: The `volumes` section is crucial. It ensures that if the containers are destroyed, the data (like Kafka messages and ZooKeeper state) is not lost because it is stored on the host machine.[^1]


### Running Kafka with Docker Compose

The process to launch the services is straightforward:

1. Create a file named `docker-compose.yml` with the content described above.
2. Open a terminal in the same directory as the file.
3. Run the command:

```bash
docker compose up -d
```

    * `up`: This command builds, (re)creates, starts, and attaches to containers for a service.
    * `-d` or `--detach`: This flag is very important for running the containers in the background. Without it, the containers would run in the foreground, and closing the terminal would stop them.

After running the command, Docker will download the specified images if they are not already on your machine and then start the containers. You can verify that they are running with `docker ps`.

### Summary and Interview Tips

Understanding Docker Compose is a valuable skill for any .NET developer working with microservices.

* **Explain the Purpose of Docker Compose**: Describe it as an orchestration tool for defining and managing multi-container applications on a single host. It simplifies development by allowing you to spin up an entire environment (app, database, message broker) with one command (`docker compose up`) [, ].
* **Contrast `docker run` vs. `docker compose up`**: `docker run` is used to start a single container. `docker compose up` reads a YAML file to start a complete set of services with their networks and volumes already configured.[^2]
* **Know Key YAML Properties**: Be able to explain the basic structure of a `docker-compose.yml` file, including `services`, `image`, `ports`, `environment`, `volumes`, and `depends_on`.
* **Discuss the Importance of Volumes**: Explain that containers are ephemeral. If you store data directly inside a container's filesystem, it will be lost when the container is removed. Volumes solve this by mapping a directory on the host machine to a directory inside the container, ensuring data persistence.[^1]
* **Acknowledge the Shift Away from ZooKeeper**: Mentioning that ZooKeeper was essential for Kafka cluster management is correct. However, showing you are up-to-date by adding that modern Kafka versions can run in **KRaft mode** (without ZooKeeper) demonstrates deeper knowledge. This simplifies the architecture significantly.

# Setting up MongoDB

This transcript covers the process of deploying MongoDB, the write database for the CQRS architecture, as a Docker container. Here is a summary of the process, a breakdown of the commands, and the relevant expert advice.

### Summary of Deploying MongoDB with Docker

The transcript explains how to run a MongoDB instance using a single `docker run` command instead of using Docker Compose. This method is straightforward for deploying a single service. The key steps are:[^1]

1. Executing the `docker run` command with specific flags to configure the container.
2. Using a **Docker Volume** to ensure the database's data persists even if the container is removed or recreated.
3. Connecting to the running MongoDB container using a GUI client like **Robo 3T** (now part of Studio 3T) to verify the setup [, ].

### The `docker run` Command Explained

The transcript provides a specific command to launch the MongoDB container. Let's break down what each part does [, ].

```bash
docker run \
    -d \
    --name mongo-container \
    -p 27017:27017 \
    --network my-docker-network \
    --restart always \
    -v mongo-data:/data/db \
    mongo:latest
```

| Flag | Purpose |
| :-- | :-- |
| **`docker run`** | The fundamental command to create and start a new container from an image. |
| **`-d`** | Runs the container in **detached** mode (in the background). |
| **`--name mongo-container`** | Assigns a memorable name to the container for easy reference. |
| **`-p 27017:27017`** | **Publishes** a port. It maps port 27017 on the host machine to port 27017 inside the container, allowing external applications (like your .NET app or Robo 3T) to connect to it [^1]. |
| **`--network my-docker-network`** | Attaches the container to the previously created user-defined bridge network, allowing it to communicate with the Kafka container and your microservices. |
| **`--restart always`** | A policy that automatically restarts the container if it stops for any reason (e.g., a crash or a system reboot). |
| **`-v mongo-data:/data/db`** | Creates and mounts a **volume**. This is the most critical part for a database [, ]. |
| **`mongo:latest`** | The name of the Docker **image** to use (`mongo`) and its tag (`latest`) [^3]. |

#### The Importance of Docker Volumes

As the transcript emphasizes, containers are ephemeral. If you write data directly into a container's filesystem and that container is deleted, the data is gone forever. **Volumes** solve this by storing data on the host machine in an area managed by Docker [, ].

The flag `-v mongo-data:/data/db` means:

* **`mongo-data`**: The name of the Docker volume on your host machine. If it doesn't exist, Docker creates it.[^2]
* **`:`**: The separator.
* **`/data/db`**: The path inside the MongoDB container where MongoDB stores its database files.

This command links the persistent volume on the host to the data directory inside the container, ensuring your data is safe.[^3]

### Beginner-Friendly .NET Code Example: Connecting to MongoDB

After running the container, you can connect to it from a .NET application using the official `MongoDB.Driver` NuGet package.

```csharp
using MongoDB.Driver;
using MongoDB.Bson;

public class MongoDbConnector
{
    public void ConnectAndListDatabases()
    {
        // 1. Connection String for the local Docker container
        const string connectionString = "mongodb://localhost:27017";

        // 2. Create a client
        var client = new MongoClient(connectionString);

        // 3. List all databases to verify the connection
        Console.WriteLine("Successfully connected to MongoDB. Databases:");
        using (var cursor = client.ListDatabases())
        {
            foreach (var dbDocument in cursor.ToEnumerable())
            {
                Console.WriteLine($"- {dbDocument["name"]}");
            }
        }
    }
}

// How to use it:
// var connector = new MongoDbConnector();
// connector.ConnectAndListDatabases();
```

This simple example demonstrates how your .NET microservice would connect to the `mongo-container` you deployed. It uses the standard `localhost:27017` address because the port was published to your host machine [, ].

### Real-World Use Cases

Running databases like MongoDB in Docker is extremely common in modern software development for several reasons:

* **Local Development**: It provides every developer on a team with an identical, isolated database instance without needing to install and configure MongoDB manually on their machine.
* **Integration Testing**: In automated CI/CD pipelines, a temporary MongoDB container can be spun up, tests can be run against it, and then it can be torn down. This ensures tests are clean and repeatable.
* **Prototyping**: It's a fast and easy way to get a database running for a new project or proof-of-concept.


### Summary and Interview Tips

When discussing running databases in Docker during an interview, focusing on the practical implications is key.

* **Explain the "Why"**: Start by explaining that containerizing a database standardizes the development environment and simplifies setup.
* **Prioritize Data Persistence**: Immediately bring up **volumes**. Explain that they are the correct mechanism for persisting database data outside of the container's lifecycle. Contrasting this with a bind mount (mapping a specific host folder) can also show deeper knowledge.
* **Distinguish `docker run` from `docker compose`**: Explain that `docker run` is great for single containers, but for a multi-service application (like one with a database, a cache, and an app), **Docker Compose** is superior as it defines the entire stack in one file.
* **Talk About Configuration**: Mention that configuration (like usernames and passwords for a production setup) should be passed in via **environment variables** (`-e` flag), not hardcoded in the image. The transcript's example omits authentication for simplicity, but you should acknowledge this isn't for production.
* **Mention Networking**: Briefly explain that custom Docker networks are used to provide a stable communication channel between containers, allowing them to resolve each other by their container names.

# Setting up Microsoft SQL Server

This transcript explains how to deploy Microsoft SQL Server, the read database for the application, as a Docker container. It covers the `docker run` command, important environment variables, and how to connect to the database using client tools.

### Summary of Deploying SQL Server with Docker

The guide demonstrates how to launch a Microsoft SQL Server container using a single `docker run` command. This is similar to the MongoDB setup, but with different configuration options specific to SQL Server. Key steps include:[^1]

1. Running the `docker run` command with mandatory environment variables for accepting the license agreement and setting the system administrator password.
2. Publishing the default SQL Server port (1433) to the host machine.
3. Connecting to the running database instance using either SQL Server Management Studio (SSMS) or the SQL Server extension in Visual Studio Code to verify the deployment.

### The `docker run` Command for SQL Server Explained

The transcript provides a detailed command to deploy the SQL Server container. Here's a breakdown of the components:

```bash
docker run \
    -d \
    --name sql-container \
    --network my-docker-network \
    --restart always \
    -e "ACCEPT_EULA=Y" \
    -e "SA_PASSWORD=YourStrong!Passw0rd" \
    -e "MSSQL_PID=Express" \
    -p 1433:1433 \
    mcr.microsoft.com/mssql/server:2017-latest
```

| Flag / Variable | Purpose |
| :-- | :-- |
| `-d`, `--name`, `--network`, `--restart` | These flags function the same as in the MongoDB example: run in detached mode, assign a name, connect to a network, and set a restart policy. |
| **`-e "ACCEPT_EULA=Y"`** | This environment variable is **mandatory**. You must explicitly accept the End-User License Agreement for the container to start [, ]. |
| **`-e "SA_PASSWORD=...`** | This sets the password for the **`sa` (system administrator)** user. You must provide a strong password. Note: The official documentation now recommends using `MSSQL_SA_PASSWORD` as `SA_PASSWORD` is deprecated [^2]. |
| **`-e "MSSQL_PID=Express"`** | This specifies the SQL Server edition to run. `Express` is a free edition suitable for development and small applications [^5]. Other options include `Developer` (free, full-featured) and `Standard`. |
| **`-p 1433:1433`** | This publishes the default TCP port for SQL Server, allowing client tools and applications to connect [^6]. |
| **Image Name** | `mcr.microsoft.com/mssql/server:2017-latest` points to Microsoft's official container registry for the 2017 version of SQL Server [^3]. |

**Important Note on Data Persistence:** The command in the transcript is missing a `-v` flag for a **volume**. Without a volume, all data created in the database will be lost if the container is removed. For any real use, you must add a volume to persist the database files, similar to the MongoDB example. A corrected command would include:[^2]
`-v sql-data:/var/opt/mssql`

### Beginner-Friendly .NET Code Example: Connecting to SQL Server

Once the container is running, you can connect to it from a .NET application using libraries like Dapper or Entity Framework Core. Here is a simple example using `System.Data.SqlClient`.

```csharp
using System.Data.SqlClient;

public class SqlServerConnector
{
    public void ConnectAndQuery()
    {
        // 1. Connection String for the local Docker container
        // Use the password you set in the docker run command.
        var connectionString = "Server=localhost,1433;User ID=sa;Password=YourStrong!Passw0rd;";

        // 2. Create and open a connection
        using (var connection = new SqlConnection(connectionString))
        {
            try
            {
                connection.Open();
                Console.WriteLine("Successfully connected to SQL Server.");

                // 3. Run a simple query to get the server version
                using (var command = new SqlCommand("SELECT @@VERSION", connection))
                {
                    var version = command.ExecuteScalar();
                    Console.WriteLine(version);
                }
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Connection failed: {ex.Message}");
            }
        }
    }
}

// How to use it:
// var connector = new SqlServerConnector();
// connector.ConnectAndQuery();
```

This code demonstrates how your Query API would connect to the SQL Server container, using the credentials and port defined during deployment.

### Real-World Use Cases

Running SQL Server in Docker is now a standard practice for:

* **Cross-Platform Development**: It allows developers on macOS and Linux to easily run SQL Server, which was traditionally a Windows-only database.
* **CI/CD Pipelines**: Automated testing pipelines can spin up a fresh SQL Server instance for each test run, ensuring a clean and isolated environment.
* **Microservices**: In a microservices architecture, it allows a service to own its database and have it deployed alongside the service code, simplifying dependency management.


### Summary and Interview Tips

When discussing SQL Server on Docker in an interview, here are the key points to highlight:

* **Explain the Key Environment Variables**: Be ready to name the two mandatory variables: `ACCEPT_EULA` and `MSSQL_SA_PASSWORD` (or the older `SA_PASSWORD`). Mentioning `MSSQL_PID` to select the edition also shows strong knowledge.[^5]
* **Emphasize Data Persistence**: Proactively point out that the provided command is incomplete for real-world use because it lacks a **volume** (`-v` flag). Explain that without it, the database is ephemeral, and all data would be lost when the container stops. This demonstrates a practical, production-oriented mindset.
* **Discuss Connection Methods**: Talk about connecting from client tools (like SSMS or Azure Data Studio) and from a .NET application (using a standard connection string pointing to `localhost` and the mapped port).[^4]
* **Mention Client Tools**: Being familiar with popular client tools shows hands-on experience. **SQL Server Management Studio (SSMS)** is the classic Windows tool, while **Azure Data Studio** is a modern, cross-platform alternative that works well with VS Code.


# Setting up the Project Structure

This transcript details the initial project setup for the microservices solution using the .NET Command-Line Interface (CLI). It establishes a clean folder structure and creates the necessary projects for both the Command and Query APIs, reflecting a layered architectural approach.

### Summary of Project Structure

The setup organizes the code into a logical structure that separates the Command and Query responsibilities at the highest level, with each service further divided into layers.

**Top-Level Folder Structure:**

* `CQRS.Core/`: A shared class library for common CQRS components.
* `SM-Post/`: The main solution folder for the social media post microservices.
    * `Post.Cmd/`: Contains all projects for the **Command API**.
    * `Post.Query/`: Contains all projects for the **Query API**.

**Projects within each Service (`Cmd` and `Query`):**

* **`.Api` (ASP.NET Core Web API):** The entry point for the service, responsible for handling HTTP requests and responses.
* **`.Domain` (Class Library):** Contains the core business logic, entities, and aggregates. This layer should have no dependencies on other layers.
* **`.Infrastructure` (Class Library):** Handles external concerns like database access, messaging, and interacting with other services.


### Key .NET CLI Commands Used

The entire setup is performed using the `.NET CLI`, a powerful tool for managing .NET projects.


| Command | Purpose | Example from Transcript |
| :-- | :-- | :-- |
| **`dotnet new <TEMPLATE>`** | Creates a new project from a template [^1]. | `dotnet new webapi` or `dotnet new classlib` |
| **`-o <OUTPUT_DIR>`** | Specifies the output directory and project name. | `dotnet new classlib -o CQRS.Core` |
| **`dotnet new sln`** | Creates a new, empty solution file in the current directory. | `dotnet new sln` (creates `SM-Post.sln`) |
| **`dotnet sln add <PROJECT_PATH>`** | Adds one or more existing projects to the solution file. | `dotnet sln add Post.Cmd/Post.Cmd.Api/Post.Cmd.Api.csproj` |
| **`cd <DIRECTORY>`** | Changes the current directory in the terminal. | `cd Post.Cmd` |

### Summary and Interview Tips

Understanding how to structure a microservices solution is a critical skill for a .NET developer.

* **Explain the Layered Architecture**: Be ready to describe the purpose of each project (`Api`, `Domain`, `Infrastructure`). This setup is a form of **Clean Architecture** or **Onion Architecture**, where dependencies flow inward toward the `Domain` layer. The `Domain` is the heart of the service and should be independent of technical details like databases or APIs.[^7]
* **Advocate for Separation**: The structure in the transcript is excellent for demonstrating the separation of concerns. The Command and Query services are completely independent, each with its own layers. This allows them to be developed, deployed, and scaled separately.[^2]
* **Single Solution vs. Multiple Solutions**: The transcript uses a single solution file (`.sln`) to manage all the projects. This is common for smaller microservice systems or during initial development. For larger systems with many independent teams, a best practice is to have a **separate solution file for each microservice**. This improves performance (faster load times in Visual Studio) and enforces stronger boundaries between services.[^5][^6]
* **Mention the .NET CLI**: Highlighting your comfort with the `.NET CLI` shows you are proficient outside of the Visual Studio GUI. It's essential for automation, scripting, and working in non-Windows environments.


# Adding Project References

This transcript details how to connect the previously created projects by adding project references using the .NET CLI. It establishes the dependency hierarchy for each microservice, which is fundamental to a layered architecture.

### Summary of Project References

The process involves using the `dotnet add reference` command to create links between the projects. This ensures that code in one project (e.g., the API layer) can access the public types and methods of another (e.g., the Domain layer).

A key takeaway from the transcript is the creation of a new, shared project:

* **`Post.Common`**: A new class library was created to hold the event objects (`PostCreatedEvent`, `CommentAddedEvent`, etc.). This is a crucial architectural decision. Since both the **Command API** (which creates the events) and the **Query API** (which consumes them) need to understand these event contracts, they must be in a shared location.


### The Dependency Flow

The project references create a clear, one-way dependency flow, which is a hallmark of Clean Architecture.

**For both Command and Query services:**

* **API Layer** -> depends on -> **Infrastructure, Domain, Common, and Core**
    * The API layer is the entry point and orchestrates calls to the other layers.
* **Infrastructure Layer** -> depends on -> **Domain and Core**
    * The Infrastructure layer implements interfaces defined in the Domain (e.g., repositories) and needs the Domain objects.
* **Domain Layer** -> depends on -> **Common and Core**
    * The Domain contains the core business logic and should have the fewest dependencies. It depends on `Post.Common` for the event definitions.
* **Common Layer** -> depends on -> **Core**
    * The shared events project depends only on the core CQRS abstractions.

This structure ensures that the core business logic in the `Domain` is not dependent on technical details like databases or APIs.

### The `dotnet add reference` Command

The transcript extensively uses this command to wire up the projects.

**Command Structure:**
`dotnet add <PROJECT_TO_MODIFY> reference <PROJECT_TO_REFERENCE>` [, ]

**Example from Transcript:**

```bash
dotnet add Post.Cmd/Post.Cmd.Api/Post.Cmd.Api.csproj reference Post.Cmd/Post.Cmd.Domain/Post.Cmd.Domain.csproj
```

* This command modifies the `Post.Cmd.Api.csproj` file to add a reference to the `Post.Cmd.Domain` project.


### Summary and Interview Tips

Understanding project dependencies is fundamental to building maintainable .NET applications.

* **Explain the Dependency Rule**: A key principle of Clean Architecture is that **dependencies should always point inwards**. The outer layers (like API and Infrastructure) depend on the inner layers (Domain), but the Domain should never depend on the outer layers. The structure in the transcript correctly follows this rule.
* **The Importance of Shared "Contracts"**: The creation of the `Post.Common` project is a great point to discuss. In microservices, when services communicate (e.g., via events), they must agree on a "contract." Placing these event classes in a shared library is a common and effective way to manage these contracts.
* **CLI vs. IDE**: The transcript notes that doing this via the CLI can be tedious. In an interview, you can acknowledge this and mention that in practice, many developers use the Visual Studio or Rider GUI to manage references. However, knowing the CLI command is essential for scripting and automation in CI/CD pipelines.
* **Project Reference vs. NuGet Package**: Be able to explain the difference. A **project reference** links to another project *within the same solution*. A **NuGet package reference** pulls in a compiled library (a `.dll` file) from an external source like NuGet.org. The shared `Post.Common` project could also have been distributed as a private NuGet package, which is a common practice in larger organizations.

# Adding Nuget package references

his transcript explains how to add external dependencies, known as **NuGet packages**, to the projects. It highlights that dependencies are added only where they are needed, which is a key principle of a well-structured application.

### Summary of NuGet Packages Added

NuGet is the package manager for .NET, allowing developers to consume and share compiled libraries. The transcript demonstrates adding packages using the NuGet Package Manager GUI in VS Code.

**Packages are added only to specific layers:**

* **`CQRS.Core` Project**:
    * `MongoDB.Driver`: The official .NET driver for MongoDB [, ]. This suggests that some core event sourcing logic, likely related to the event store repository, resides in this shared project.
* **`Post.Cmd.Infrastructure` Project**:
    * `Confluent.Kafka`: The client library for producing messages to Apache Kafka.
    * `Microsoft.Extensions.Options`: Used for handling configuration, often for dependency injection.
    * `MongoDB.Driver`: Added again here, likely to implement the specific event store repository for the Command API.
* **`Post.Query.Infrastructure` Project**:
    * `Microsoft.EntityFrameworkCore.SqlServer`: The Object-Relational Mapper (ORM) for interacting with the SQL Server read database.
    * `Microsoft.Extensions.Hosting`: Provides hosting and lifetime management for background services, perfect for the Kafka consumer.
    * `Confluent.Kafka`: The same library, but used here to consume messages from Kafka.


### Important Architectural Points

* **No Packages in the Domain Layer**: A crucial observation is that **no NuGet packages were added to the `Domain` projects**. This is intentional and a core tenet of Clean Architecture. The domain should be free of external dependencies and technical details, containing only pure business logic.
* **Infrastructure is the "Plugin" Layer**: The `Infrastructure` projects are where all external dependencies live. This layer acts as a "plugin" to the core application, handling communication with databases (MongoDB, SQL Server) and message brokers (Kafka). This makes it easy to swap out technologies later if needed. For example, you could replace SQL Server with PostgreSQL by changing only the `Post.Query.Infrastructure` project.
* **Version Matching**: The transcript correctly notes the importance of matching the versions of `Microsoft.*` packages (like `Microsoft.Extensions.Options`) to the project's target framework (.NET 6). This helps avoid compatibility issues.


### The `dotnet restore` Command

After adding packages, the transcript shows the use of the `dotnet restore` command.

* **Purpose**: This command downloads and installs all the packages listed in the project files (`.csproj`) for the entire solution. It ensures that all dependencies are present before building the application. While commands like `dotnet build` and `dotnet run` often trigger a restore automatically, it's good practice to run it manually after changing dependencies.


### Summary and Interview Tips

Knowledge of NuGet and dependency management is essential for any .NET developer.

* **Explain NuGet's Role**: Describe NuGet as the package manager for .NET, used to add third-party libraries and frameworks to a project.
* **CLI vs. IDE for Packages**: Similar to project references, you can manage packages via the GUI (in Visual Studio or VS Code) or the CLI (`dotnet add package <PACKAGE_NAME>`). Mentioning both shows versatility. The CLI is vital for automated builds.
* **Articulate the Dependency Strategy**: Be prepared to explain *why* packages are only in the `Infrastructure` layers. The goal is to isolate the core business logic (`Domain`) from external concerns. This makes the domain easier to test and more resilient to changes in technology.
* **Discuss Transitive Dependencies**: NuGet packages can have their own dependencies. This is called a **transitive dependency**. `dotnet restore` handles resolving this entire dependency graph. Understanding this concept is a sign of a more experienced developer.


# Understanding Different Message Types

This transcript introduces the fundamental message types in CQRS and Event Sourcing, focusing on **Commands** and **Events**. It defines what a command is and establishes a clear naming convention.

### Key Message Types

In CQRS and Event Sourcing, the application communicates through distinct types of messages. While the transcript focuses on two, there are three primary types:[^1]

* **Command**: A request to perform an action or change the state of the system. It is an instruction that the system can either accept or reject.[^2]
* **Event**: A statement of fact that something has already happened in the past. Events are immutable and represent a state change.[^3]
* **Query**: A request for data that does not change the state of the system.


### Understanding Commands

As the transcript explains, a command is a message that encapsulates two things:

1. **Intent**: What the user wants to do.
2. **Data**: All the information required to perform that action.

**Key Characteristics of a Command:**

* **Imperative Naming**: Commands should always be named with a verb in the present tense, clearly stating the desired action (e.g., `CreatePostCommand`, `AddCommentCommand`).[^4]
* **Targets a Single Aggregate**: A command is typically directed at a single business object or aggregate (e.g., a specific social media post).
* **Can Be Rejected**: The system can validate a command and reject it if it violates business rules (e.g., attempting to add a comment to a deleted post).


### Beginner-Friendly .NET Code Example

A command is a simple data-transfer object (DTO). In C\#, a `record` is a perfect way to define a command because it's lightweight and primarily intended to carry data.

```csharp
// The base for all commands, often used for message routing
public abstract class BaseCommand
{
    // Every message should have a unique ID for traceability
    public Guid Id { get; protected set; } = Guid.NewGuid();
}

// Command to create a new social media post
public record NewPostCommand : BaseCommand
{
    public string Author { get; init; }
    public string Message { get; init; }
}

// Command to add a comment to an existing post
public record AddCommentCommand : BaseCommand
{
    // The ID of the post to which the comment is being added
    public Guid PostId { get; init; } 
    public string Comment { get; init; }
    public string Username { get; init; }
}
```

* The `init` keyword makes the properties "init-only," meaning they can only be set when the object is created, making the command object itself immutable after creation.


### Summary and Interview Tips

When discussing commands and events in an interview, clarity and precision are key.

* **Distinguish Command vs. Event**: This is a fundamental concept.
    * **Command**: "Please do this." (e.g., `ApproveOrderCommand`)
    * **Event**: "This happened." (e.g., `OrderApprovedEvent`)
A command is an *intent* that can be rejected; an event is a *fact* that cannot be changed.[^5]
* **Use Correct Naming Conventions**: Always use imperative, present-tense verbs for commands (`Create...`, `Add...`, `Update...`) and past-tense verbs for events (`...Created`, `...Added`, `...Updated`). This immediately communicates the purpose of the message.
* **Explain the Flow**: Describe how a **Command** is sent to a command handler, which validates it, processes it, and then produces one or more **Events**. These events are what get persisted in an Event Sourcing system.[^6]
* **Connect to CQRS**: Explain that commands belong to the "write side" of a CQRS architecture, as their purpose is to change state. Queries belong to the "read side."

# Creating Command Objects(Simple Data Transfer Objects (DTOs). Their only job is to carry data and express intent from the client to the command handler. They should not contain any business logic.)

This transcript details the process of creating the **Command** objects for the social media application. It establishes a clear class hierarchy to ensure all commands share a common structure and then defines the specific commands needed for the Command API.

### Summary of Command Object Creation

The approach follows two main steps:

1. **Create Base Classes**: An abstract `Message` class is created to provide a unique `Id` for all message types (Commands and Events). A `BaseCommand` class then inherits from `Message` to act as the foundation for all command objects. These are placed in the shared `CQRS.Core` project.[^1]
2. **Create Concrete Commands**: Specific command classes are created for each action the Command API must handle (e.g., `NewPostCommand`). These are placed in a `Commands` folder within the `Post.Cmd.Api` project.

### Code Examples of the Command Hierarchy

This structure ensures that every command is a message and automatically has an `Id` property for tracking and logging. Using C\# `record` types is a modern, beginner-friendly way to define these data-carrying objects.

#### 1. Base Classes (in `CQRS.Core` project)

```csharp
// The root of all messages, ensuring each has a unique identifier.
public abstract class Message
{
    public Guid Id { get; protected set; } = Guid.NewGuid();
}

// The base class for all commands, inheriting the Id from Message.
public abstract class BaseCommand : Message
{
}
```


#### 2. Concrete Commands (in `Post.Cmd.Api` project)

These `record` classes inherit from `BaseCommand` and contain the specific data needed for each action.

```csharp
// Creates a new post
public record NewPostCommand(string Author, string Message) : BaseCommand;

// Edits the text of an existing post
public record EditMessageCommand(string Message) : BaseCommand;

// Likes a post (no extra data needed)
public record LikePostCommand() : BaseCommand;

// Adds a new comment to a post
public record AddCommentCommand(string Comment, string Username) : BaseCommand;

// Edits an existing comment
public record EditCommentCommand(Guid CommentId, string Comment, string Username) : BaseCommand;

// Removes a comment from a post
public record RemoveCommentCommand(Guid CommentId, string Username) : BaseCommand;

// Deletes a post entirely
public record DeletePostCommand(string Username) : BaseCommand;
```

*Notice the inclusion of `Username` in commands like `EditCommentCommand` and `DeletePostCommand`. This is important data that will be used by the command handler to perform validation (e.g., "Is this user allowed to delete this post?").*

### Summary and Interview Tips

Understanding how to model commands is fundamental to CQRS.

* **Explain the Base Class Pattern**: Using a base `Message` or `BaseCommand` class is a common pattern to enforce consistency. It ensures all commands have common properties like an `Id` for traceability or a `Timestamp` without duplicating code.
* **Commands as Data Carriers**: Emphasize that commands are simple Data Transfer Objects (DTOs). Their only job is to carry data and express intent from the client to the command handler. They should not contain any business logic.
* **Immutability is Key**: Using C\# `record` types makes the command objects immutable by default. This is a best practice because once a command is created, its intent and data should not change as it moves through the system.
* **The "Why" Behind Command Properties**: When looking at a command's properties, you should be able to infer the business rule. For example, `DeletePostCommand` having a `Username` property implies a business rule that only the post's original author can delete it.


# Understanding Events

This transcript defines the second fundamental message type in Event Sourcing: the **Event**. It establishes what an event is, where it comes from, and the standard naming convention to follow.

### Understanding Events

An event is a message that describes something that has *already occurred* within the application. It is a statement of fact and, once created, is immutable.[^1]

**Key Characteristics of an Event:**

* **A Record of the Past**: Unlike a command (which is a request), an event is a notification that a state change has happened.
* **Originate from an Aggregate**: Events are typically raised by a business object (the aggregate) in response to a successfully processed command.[^2]
* **Past-Tense Naming**: Events should always be named with a verb in the past tense (e.g., `PostCreatedEvent`, `CommentAddedEvent`, `OrderShippedEvent`) to clearly indicate that the action is complete [, ].


### Beginner-Friendly .NET Code Example

Just like commands, events are simple data-carrying objects. They should contain all the relevant information about what happened so that other parts of the system can react to them without needing to query for more data.

C\# `record` types are an excellent choice for defining events due to their inherent immutability.[^3]

```csharp
// The base for all events, ensuring they are a type of Message
public abstract class BaseEvent : Message
{
    // A version number is crucial for replaying events in the correct order
    public int Version { get; set; }
}

// Event indicating a new post was successfully created
public record PostCreatedEvent : BaseEvent
{
    public Guid PostId { get; init; }
    public string Author { get; init; }
    public string Message { get; init; }
    public DateTime DatePosted { get; init; }
}

// Event indicating a comment was successfully added to a post
public record CommentAddedEvent : BaseEvent
{
    public Guid CommentId { get; init; }
    public string Comment { get; init; }
    public string Username { get; init; }
    public DateTime CommentDate { get; init; }
}
```


### Summary and Interview Tips

Distinguishing between commands and events is a frequent topic in system design interviews.

* **Command vs. Event is Key**: Be ready to clearly articulate the difference.
    * **Command**: An *intent* to do something. It can be rejected. (e.g., `CreatePostCommand`)
    * **Event**: A *record* of something that has happened. It is a fact and cannot be changed. (e.g., `PostCreatedEvent`)
* **Emphasize Naming Conventions**: Mentioning the "imperative vs. past-tense" naming convention for commands and events is a simple way to demonstrate a solid understanding of the pattern [, ].
* **Events as the Source of Truth**: In Event Sourcing, the sequence of events *is* the state of the application. Explain that you can rebuild the current state of any business object by replaying all of its historical events in order.[^1]
* **"Rich" Events**: Good events contain all the necessary data about the change. This is important because it allows other microservices that consume these events to be fully autonomous; they don't need to call back to the original service to get more information.


# Understanding Events and Types

This transcript details the creation of the **Event** objects that correspond to the previously defined commands. It establishes a clear class hierarchy and places the event classes in the `Post.Common` project, making them accessible to both the Command and Query microservices.

### Summary of Event Object Creation

The implementation follows a logical pattern that mirrors the command setup:

1. **Create a Base Class**: An abstract `BaseEvent` class is created in the `CQRS.Core` project. This class inherits from the `Message` class (providing a unique `Id`) and adds two important properties:
    * **`Version` (int)**: Crucial for Event Sourcing. When reconstructing the state of an object, events must be replayed in the exact order they occurred. The version number ensures this ordering.
    * **`Type` (string)**: Acts as a **discriminator**. When events are serialized (e.g., to JSON) and sent over Kafka, this field allows the consumer to know which specific event class to deserialize the message into.
2. **Create Concrete Events**: For each command, a corresponding event class is created in the `Post.Common` project. These classes inherit from `BaseEvent` and contain all the data representing the state change.

### Code Examples of the Event Hierarchy

Using modern C\# features makes the event definitions clean and robust.

#### 1. Base Class (in `CQRS.Core` project)

```csharp
// The root of all messages, ensuring each has a unique identifier.
public abstract class Message
{
    public Guid Id { get; protected set; } = Guid.NewGuid();
}

// The base for all events, adding Version and Type properties.
public abstract class BaseEvent : Message
{
    protected BaseEvent(string type)
    {
        Type = type;
    }

    public int Version { get; set; }
    public string Type { get; private set; }
}
```


#### 2. Concrete Events (in `Post.Common` project)

The transcript shows using a constructor to set the `Type` property. Using C\# `record` types simplifies this further.

```csharp
// Using a record with a constructor to set the base properties
public record PostCreatedEvent(Guid Id, string Author, string Message, DateTime DatePosted) 
    : BaseEvent(nameof(PostCreatedEvent));

public record MessageUpdatedEvent(Guid Id, string Message) 
    : BaseEvent(nameof(MessageUpdatedEvent));

public record PostLikedEvent(Guid Id) 
    : BaseEvent(nameof(PostLikedEvent));

public record CommentAddedEvent(Guid Id, Guid CommentId, string Comment, string Username, DateTime CommentDate) 
    : BaseEvent(nameof(CommentAddedEvent));
    
public record CommentUpdatedEvent(Guid Id, Guid CommentId, string Comment, string Username, DateTime EditDate) 
    : BaseEvent(nameof(CommentUpdatedEvent));

public record CommentRemovedEvent(Guid Id, Guid CommentId) 
    : BaseEvent(nameof(CommentRemovedEvent));
    
public record PostRemovedEvent(Guid Id) 
    : BaseEvent(nameof(PostRemovedEvent));
```

* The `nameof()` operator is a robust way to set the `Type` discriminator, as it automatically updates if the class name is refactored.


### Summary and Interview Tips

When discussing event objects in an interview, demonstrating knowledge of their role in a distributed system is crucial.

* **Explain the `Version` Property**: State that the `Version` is essential for optimistic concurrency control and for replaying events to rebuild an aggregate's state. Each new event for a specific aggregate instance will have an incrementing version number (1, 2, 3, etc.).
* **Explain the `Type` Discriminator**: The `Type` property is vital for **polymorphism** during deserialization [, ]. When a consumer receives a JSON message from Kafka, it's just text. The `Type` field tells the deserializer, "This message is a `PostCreatedEvent`, so you should use that class to parse it." This allows a single Kafka topic to carry multiple types of events.
* **Events Should Be "Fat"**: Good events are "fat" or "rich," meaning they contain all the data related to the change. This enables consuming services to be autonomous. For example, the `PostCreatedEvent` contains the `Author` and `Message`, so the Query API's consumer doesn't need to call back to the Command API to get those details.
* **Events in a Shared Location**: Placing the event classes in a shared library (`Post.Common`) is a critical design choice. Both the producer (Command API) and the consumer (Query API) need to agree on the event "contract." A shared library is the simplest way to enforce this agreement.


# Command Dispatching
