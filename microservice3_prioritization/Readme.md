# NOTLI - Microservice 3: Prioritization Engine


This microservice acts as the intelligent routing and prioritization hub within the NOTLI system. It consumes notification messages from the central message queue (populated by Microservice 2), applies defined prioritization rules, and republishes the messages to appropriate downstream queues, typically categorized by priority level, for consumption by the Delivery service (Microservice 4).


## Core Features


*   **Message Consumption:** Reliably consumes messages from the main RabbitMQ notification queue (e.g., `notli_notifications_queue`).
*   **Prioritization Logic:** Applies rules to assign a priority level (e.g., `high`, `medium`, `low`) to each message. Rules can be based on:
    *   Explicit `priority` field in the message.
    *   Message type (`transactional` vs. `promotional`).
    *   User account tier or attributes (requires lookup, potentially via API call or cached data).
    *   Keywords in the message content (future enhancement).
*   **Message Routing:** Publishes messages to specific downstream queues based on their assigned priority (e.g., `delivery_queue_high`, `delivery_queue_medium`, `delivery_queue_low`).
*   **Acknowledgement Handling:** Properly acknowledges messages with RabbitMQ only after successful processing and republishing, ensuring reliability.
*   **Concurrency:** Can run multiple concurrent consumers (goroutines) to process messages in parallel.


## Tech Stack


*   **Language:** Go (Golang) 1.21+
*   **Message Broker:** RabbitMQ
*   **RabbitMQ Client:** `github.com/rabbitmq/amqp091-go`
*   **Configuration:** `.env` file (`github.com/joho/godotenv`)
*   **JSON Handling:** Go's standard `encoding/json`
*   **(Optional) API Endpoint for User Tier Lookup**


## Project Structure


```
.env                   # Environment variables (DO NOT COMMIT)
Readme.md              # This file (should be README.md)
go.mod / go.sum       # Go module files
cmd/
  consumer/
    main.go          # Application entry point, RabbitMQ consumer setup
internal/
  config/            # Configuration loading (config.go)
  consumer/          # RabbitMQ consumer logic (consumer.go)
  prioritizer/       # Logic for assigning priority (rules.go)
  publisher/         # Logic for publishing to downstream queues (publisher.go)
  models/            # Data structures for messages (notification.go)
```


## Setup and Configuration


1.  **Prerequisites:**
    *   Go 1.21 or later installed.
    *   RabbitMQ server running and accessible.
    *   The upstream queue (`notli_notifications_queue` from Microservice 2) exists and is receiving messages.


2.  **RabbitMQ Setup:**
    *   Ensure the RabbitMQ instance is running.
    *   This service will declare the downstream priority queues (e.g., `delivery_queue_high`, `delivery_queue_medium`, `delivery_queue_low`) and potentially an exchange to route to them if they don't exist.
    *   Ensure the user configured in `.env` has permissions to declare/bind/consume/publish to the relevant queues/exchanges.


3.  **Environment Variables:**
    *   Create a `.env` file in the `microservice3_prioritization` root directory.
    *   Copy the template below and fill in your specific values.


    ```dotenv
    # .env example for Prioritization Service


    # RabbitMQ Configuration
    RABBITMQ_URL=amqp://guest:guest@localhost:5672/ # Format: amqp://user:password@host:port/[vhost]


    # Upstream Queue (Consumed From)
    RABBITMQ_CONSUME_QUEUE=notli_notifications_queue


    # Downstream Queues/Exchange (Published To)
    # Option 1: Direct to Queues
    RABBITMQ_PUBLISH_QUEUE_HIGH=delivery_queue_high
    RABBITMQ_PUBLISH_QUEUE_MEDIUM=delivery_queue_medium
    RABBITMQ_PUBLISH_QUEUE_LOW=delivery_queue_low
    # Option 2: Publish to an Exchange with Routing Keys
    # RABBITMQ_PUBLISH_EXCHANGE=delivery_exchange
    # RABBITMQ_ROUTING_KEY_HIGH=delivery.high
    # RABBITMQ_ROUTING_KEY_MEDIUM=delivery.medium
    # RABBITMQ_ROUTING_KEY_LOW=delivery.low


    # Consumer Configuration
    CONSUMER_CONCURRENCY=10 # Number of concurrent message processing goroutines
    RABBITMQ_PREFETCH_COUNT=20 # How many messages a single consumer can hold unacknowledged


    # (Optional) API Endpoint for User Tier Lookup
    # USER_SERVICE_URL=http://localhost:3000/api/users # Example
    ```


4.  **Install Dependencies:**
    *   Navigate to the `microservice3_prioritization` directory.
    *   Run: `go mod tidy`


## Running the Service


1.  **Navigate to the `cmd/consumer` directory:**
    ```bash
    cd cmd/consumer
    ```
2.  **Run the application:**
    ```bash
    go run main.go
    ```
    *   The service will start, connect to RabbitMQ, and begin consuming messages from `RABBITMQ_CONSUME_QUEUE`.
    *   It does **not** typically expose HTTP endpoints; its primary function is message processing.


## Prioritization Logic


1.  A message is consumed from `RABBITMQ_CONSUME_QUEUE`.
2.  The `prioritizer` component analyzes the message content.
3.  **Default Priority:** If a `priority` field exists in the message (e.g., "high", "medium", "low"), it's used directly.
4.  **Fallback Rules (Example):**
    *   If `message_type` is `transactional`, assign `high` priority.
    *   If `message_type` is `promotional`, assign `low` priority.
    *   *(Future)* If user tier is `premium`, upgrade priority to `high`.
    *   *(Future)* If keywords like "urgent" or "important" are found, upgrade priority.
    *   If no rules match, assign `medium` priority.
5.  The `publisher` component takes the original message and the determined priority.
6.  It publishes the message to the corresponding downstream queue (e.g., a message determined as `high` priority goes to `RABBITMQ_PUBLISH_QUEUE_HIGH`).
7.  Only after successful publishing, the original message is acknowledged (`ack`) with RabbitMQ.
8.  If processing or publishing fails, the message can be negatively acknowledged (`nack`) to be requeued or sent to a dead-letter queue (if configured).


## Performance & Scalability Considerations


*   **Consumer Concurrency:** The `CONSUMER_CONCURRENCY` setting allows scaling message processing horizontally *within* a single instance by using multiple goroutines.
*   **Prefetch Count:** `RABBITMQ_PREFETCH_COUNT` controls how many messages RabbitMQ sends to a consumer before receiving acknowledgements. Tuning this helps balance throughput and resource usage.
*   **Horizontal Scaling:** You can run multiple instances of this microservice. RabbitMQ will distribute messages from the `RABBITMQ_CONSUME_QUEUE` across all running instances/consumers, allowing for significant scaling.
*   **Rule Complexity:** Complex prioritization rules involving external API calls (like user tier lookups) can become bottlenecks. Consider caching or optimizing these lookups.
*   **Downstream Queue Performance:** Ensure the Delivery service (Microservice 4) can consume messages from the priority queues at a sufficient rate to prevent backlog.


## Future Considerations


*   Implementing dynamic rule loading from a database or configuration service instead of hardcoding.
*   Adding keyword analysis for prioritization.
*   Integrating with a user information service for tier-based prioritization (with caching).
*   Implementing more sophisticated retry and dead-lettering strategies for processing failures.
*   Adding distributed tracing for observability across the message flow.
*   Using a dedicated exchange for publishing instead of directly publishing to queues, allowing more flexible routing.

