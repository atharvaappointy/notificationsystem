# NOTLI - Microservice 4: Delivery Engine

This microservice is the final stage in the NOTLI notification pipeline. It consumes prioritized messages from dedicated delivery queues (populated by Microservice 3), renders templates, interacts with external delivery providers (initially SMTP for email), handles delivery attempts and retries, and potentially tracks final delivery status.

## Core Features

*   **Prioritized Message Consumption:** Consumes messages from multiple RabbitMQ queues (e.g., `delivery_queue_high`, `delivery_queue_medium`, `delivery_queue_low`), prioritizing consumption from higher-priority queues.
*   **Template Rendering:** (Future Enhancement) Uses a templating engine to combine message data with pre-defined templates (HTML/Text) before delivery.
*   **Email Delivery:** Sends emails using configured providers (currently standard SMTP).
*   **Retry Logic:** Implements basic retry mechanisms (e.g., retry on transient SMTP errors) with potential backoff.
*   **Acknowledgement Handling:** Acknowledges messages with RabbitMQ upon successful delivery attempt or after exhausting retries.
*   **Concurrency:** Can run multiple concurrent consumers to process messages from different priority queues.
*   **Provider Abstraction:** Designed to support multiple delivery providers (e.g., SendGrid, Mailgun) in the future via interfaces.

## Tech Stack

*   **Language:** Go (Golang) 1.21+
*   **Message Broker:** RabbitMQ
*   **RabbitMQ Client:** `github.com/rabbitmq/amqp091-go`
*   **Email (SMTP):** Go's standard `net/smtp`
*   **Configuration:** `.env` file (`github.com/joho/godotenv`)
*   **JSON Handling:** Go's standard `encoding/json`
*   **(Future):** Go template engine (`html/template`, `text/template`), additional email provider SDKs.

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
  delivery/
    email/           # Email specific logic
      smtp_provider.go # Logic for sending via SMTP
      provider.go    # Interface for email providers
    handler.go       # Handles message processing and delivery orchestration
  models/            # Data structures for messages (notification.go)
  # (Future) template/ # Template rendering logic
```

## Setup and Configuration

1.  **Prerequisites:**
    *   Go 1.21 or later installed.
    *   RabbitMQ server running and accessible.
    *   The downstream priority queues (`delivery_queue_high`, etc., from Microservice 3) exist and are receiving messages.
    *   Access to an SMTP server (e.g., Gmail, SendGrid SMTP, Mailgun SMTP, or a local server like MailHog for testing).

2.  **RabbitMQ Setup:**
    *   Ensure the RabbitMQ instance is running.
    *   Ensure the user configured in `.env` has permissions to consume from the priority queues.

3.  **Environment Variables:**
    *   Create a `.env` file in the `microservice4_delivery` root directory.
    *   Copy the template below and fill in your specific SMTP server details.

    ```dotenv
    # .env example for Delivery Service

    # RabbitMQ Configuration
    RABBITMQ_URL=amqp://guest:guest@localhost:5672/ # Format: amqp://user:password@host:port/[vhost]

    # Delivery Queues (Consumed From - Comma-separated in order of priority)
    RABBITMQ_CONSUME_QUEUES=delivery_queue_high,delivery_queue_medium,delivery_queue_low

    # Consumer Configuration
    CONSUMER_CONCURRENCY_PER_QUEUE=5 # Number of concurrent goroutines per queue
    RABBITMQ_PREFETCH_COUNT=10      # Prefetch count per consumer

    # Email Provider Configuration (SMTP Example)
    EMAIL_PROVIDER=smtp
    SMTP_HOST=smtp.example.com
    SMTP_PORT=587
    SMTP_USERNAME=your_smtp_username
    SMTP_PASSWORD=your_smtp_password
    SMTP_FROM_ADDRESS=notifications@yourdomain.com # Default 'From' address
    # Optional: Specify authentication mechanism if needed (e.g., PLAIN, LOGIN)
    # SMTP_AUTH_MECHANISM=PLAIN

    # (Future) Other Provider Config (e.g., SendGrid)
    # EMAIL_PROVIDER=sendgrid
    # SENDGRID_API_KEY=YOUR_SENDGRID_KEY
    # SENDGRID_FROM_ADDRESS=notifications@yourdomain.com
    ```

4.  **Install Dependencies:**
    *   Navigate to the `microservice4_delivery` directory.
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
    *   The service will start, connect to RabbitMQ, and begin consuming messages from the configured `RABBITMQ_CONSUME_QUEUES`, respecting priority.
    *   It primarily processes messages and interacts with external delivery services; it typically does not expose HTTP endpoints.

## Delivery Logic

1.  The `consumer` component listens on the specified priority queues.
2.  It prioritizes fetching messages from queues earlier in the `RABBITMQ_CONSUME_QUEUES` list (e.g., `delivery_queue_high` first).
3.  When a message is received, it's passed to the `delivery.Handler`.
4.  The handler determines the message type (e.g., `email`).
5.  **(Future):** If templates are involved, the handler calls the `template` engine to render the final `subject`, `body_html`, and `body_text`.
6.  The handler selects the appropriate `delivery.Provider` based on configuration (`EMAIL_PROVIDER`) and message details.
7.  It calls the provider's `Send` method with the necessary details (`to`, `cc`, `bcc`, `subject`, `body`, etc.).
8.  **SMTP Example:** The `smtp_provider` constructs the email MIME message and uses `net/smtp` to connect and send.
9.  If the `Send` operation is successful (e.g., SMTP command sequence completes without error), the message is acknowledged (`ack`) with RabbitMQ.
10. If the `Send` operation fails with a potentially transient error (e.g., temporary network issue, SMTP server busy), the handler might implement a limited retry logic (e.g., wait and try again a few times).
11. If the `Send` fails permanently (e.g., invalid recipient address, authentication failure) or retries are exhausted, the message might be negatively acknowledged (`nack`) to be requeued or dead-lettered, or simply acknowledged if no further action is possible.
12. **(Future):** Implement status tracking by storing delivery attempts and final status (e.g., provider's message ID, success/failure) in a database.

## Performance & Reliability Considerations

*   **Consumer Priority:** Ensure the logic correctly prioritizes consuming from high-priority queues.
*   **Connection Pooling:** For providers involving network connections (like SMTP or HTTP APIs), use connection pooling to improve performance and reduce overhead.
*   **Provider Rate Limits:** Be mindful of rate limits imposed by external providers (SMTP servers, APIs like SendGrid). Implement client-side rate limiting or throttling if necessary.
*   **Retry Strategy:** Define a clear retry strategy for transient errors. Avoid indefinite retries. Exponential backoff is common.
*   **Dead Lettering:** Configure RabbitMQ dead-letter exchanges for messages that consistently fail delivery, allowing for later inspection and manual intervention.
*   **Scalability:** Run multiple instances of the service. RabbitMQ will distribute messages across consumers listening to the same queue.
*   **Error Handling:** Gracefully handle errors from providers and RabbitMQ operations.

## Future Considerations

*   Implementing the template rendering engine.
*   Adding support for more email providers (SendGrid, Mailgun, SES) behind the `Provider` interface.
*   Implementing robust delivery status tracking (database persistence, webhooks for status updates from providers).
*   Supporting other notification channels (SMS, Push Notifications) with their own providers.
*   Implementing more sophisticated rate limiting strategies based on provider feedback.
*   Adding circuit breakers for external provider interactions.
*   Integrating distributed tracing for end-to-end visibility.
