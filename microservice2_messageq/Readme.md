# NOTLI - Microservice 2: Message Queue


This microservice acts as the central message broker for the NOTLI system. It receives notification requests (e.g., email sends) from authenticated sources (like Microservice 1 via API Key), persists them reliably in a message queue, and makes them available for processing by downstream services (Microservice 3: Prioritization).


## Core Features


*   **Reliable Message Ingestion:** Accepts notification requests via a secure HTTP API endpoint.
*   **Message Queue Integration:** Leverages RabbitMQ for durable message storage and queuing.
    *   Uses specific exchanges and queues for routing notification tasks.
    *   Ensures message persistence to survive broker restarts.
*   **API Key Authentication:** Validates incoming requests by checking the provided API key against the Authentication service (Microservice 1).
*   **Decoupling:** Separates the initial request acceptance from the actual processing and delivery, improving system resilience and scalability.
*   **Structured Logging:** Provides logs for message reception, validation success/failure, and queue publishing.


## Tech Stack


*   **Language:** Go (Golang) 1.21+
*   **Web Framework:** Gin (`github.com/gin-gonic/gin`)
*   **Message Broker:** RabbitMQ
*   **RabbitMQ Client:** `github.com/rabbitmq/amqp091-go`
*   **Configuration:** `.env` file (`github.com/joho/godotenv`)
*   **HTTP Client:** Go's standard `net/http` (for API key validation)
*   **JSON Handling:** Go's standard `encoding/json`


## Project Structure


```
.env                   # Environment variables (DO NOT COMMIT)
Readme.md              # This file (should be README.md)
go.mod / go.sum       # Go module files
cmd/
  server/
    main.go          # Application entry point, Gin setup, RabbitMQ connection
internal/
  config/            # Configuration loading (config.go)
  handler/           # HTTP request handlers (message_handler.go)
  messagequeue/      # RabbitMQ connection and publishing logic (rabbitmq.go)
  service/           # Business logic, including API key validation (validation_service.go)
  models/            # Data structures for incoming requests (notification_request.go)
```


## Setup and Configuration


1.  **Prerequisites:**
    *   Go 1.21 or later installed.
    *   RabbitMQ server running and accessible.
    *   Microservice 1 (Authentication) running and accessible (for API key validation).


2.  **RabbitMQ Setup:**
    *   Ensure you have a RabbitMQ instance running.
    *   (Optional but recommended) Create a dedicated user and virtual host (`vhost`) for the NOTLI application within RabbitMQ.
    *   This service will automatically declare the necessary exchange (e.g., `notli_exchange`) and queue (e.g., `notli_notifications_queue`) if they don't exist.


3.  **Environment Variables:**
    *   Create a `.env` file in the `microservice2_messageq` root directory.
    *   Copy the template below and fill in your specific values.


    ```dotenv
    # .env example for Message Queue Service


    # Server Configuration
    SERVER_PORT=3001
    SERVER_HOST=localhost # Or 0.0.0.0 to listen on all interfaces


    # RabbitMQ Configuration
    RABBITMQ_URL=amqp://guest:guest@localhost:5672/ # Format: amqp://user:password@host:port/[vhost]
    RABBITMQ_EXCHANGE=notli_exchange           # Name of the exchange to publish to
    RABBITMQ_QUEUE=notli_notifications_queue   # Name of the queue consumer will listen to
    RABBITMQ_ROUTING_KEY=notification.email    # Routing key for messages


    # Authentication Service Endpoint (for API Key Validation)
    AUTH_SERVICE_URL=http://localhost:3000 # Base URL of Microservice 1
    AUTH_VALIDATE_ENDPOINT=/api/auth/validate # Specific path for the validation API
    ```


4.  **Install Dependencies:**
    *   Navigate to the `microservice2_messageq` directory.
    *   Run: `go mod tidy`


## Running the Service


1.  **Navigate to the `cmd/server` directory:**
    ```bash
    cd cmd/server
    ```
2.  **Run the application:**
    ```bash
    go run main.go
    ```
    *   The server will start, connect to RabbitMQ, and listen for HTTP requests on the configured port (e.g., `http://localhost:3001`).


## API Endpoints


*   **Health Check:**
    *   `GET /health`: Returns `{"status": "UP", "rabbitmq_status": "connected/disconnected"}`.
*   **Submit Notification Request (Requires API Key Auth):**
    *   `POST /api/messages/submit`: Accepts a notification request.
        *   **Requires Header:** `X-API-Key: YOUR_VALID_API_KEY`
        *   **Request Body (JSON):** Varies based on notification type (e.g., email), but should include details like `to`, `subject`, `body`, `type`, `priority` (optional), etc. Example for email:
            ```json
            {
              "type": "email",
              "to": ["recipient1@example.com", "recipient2@example.com"],
              "cc": ["cc@example.com"],
              "bcc": ["bcc@example.com"],
              "subject": "Your Subject Here",
              "body_html": "<h1>Hello!</h1><p>This is the HTML body.</p>",
              "body_text": "Hello! This is the text body.",
              "priority": "medium" // Optional: e.g., low, medium, high
            }
            ```
        *   **Success Response (HTTP 202 Accepted):** `{"status": "message accepted", "message_id": "<unique_internal_id>"}` (Message ID is generated internally, not the RabbitMQ message ID).
        *   **Error Responses:**
            *   `HTTP 400 Bad Request`: Invalid JSON body or missing required fields.
            *   `HTTP 401 Unauthorized`: Missing or invalid `X-API-Key` header.
            *   `HTTP 403 Forbidden`: API Key is valid but might lack permissions (if implemented).
            *   `HTTP 500 Internal Server Error`: Failed to publish to RabbitMQ or other unexpected server errors.
            *   `HTTP 503 Service Unavailable`: Cannot connect to RabbitMQ or Authentication Service.


## Message Flow


1.  Microservice 1 (or an external client) sends a `POST /api/messages/submit` request with an `X-API-Key` header and JSON payload.
2.  The `message_handler` receives the request.
3.  It calls the `validation_service` to validate the `X-API-Key` by making an HTTP request to Microservice 1's `/api/auth/validate` endpoint.
4.  If the key is valid, the handler parses the JSON payload.
5.  The payload (notification request) is marshaled into JSON format suitable for the queue.
6.  The `messagequeue` service publishes the message to the configured RabbitMQ exchange (`notli_exchange`) with the routing key (`notification.email`).
7.  RabbitMQ routes the message to the bound queue (`notli_notifications_queue`).
8.  The handler returns `HTTP 202 Accepted` to the client.
9.  Microservice 3 (Prioritization) consumes messages from `notli_notifications_queue`.


## Performance & Reliability Considerations


*   **RabbitMQ Durability:** Messages are published as 'persistent', and the queue is declared as 'durable' to survive broker restarts. Ensure your RabbitMQ cluster is configured for high availability in production.
*   **Publisher Confirms:** Consider implementing RabbitMQ Publisher Confirms for higher assurance that messages have been successfully received by the broker.
*   **API Key Validation Caching:** To reduce latency and load on Microservice 1, consider caching API key validation results for a short duration.
*   **Scalability:** This service can be scaled horizontally (running multiple instances) as RabbitMQ handles message distribution. Ensure your RabbitMQ cluster can handle the load.
*   **Dead Letter Queue:** Configure a Dead Letter Exchange/Queue in RabbitMQ to handle messages that cannot be processed successfully by consumers.


## Future Considerations


*   Support for different notification types (SMS, Push Notifications) with distinct routing keys/queues.
*   Schema validation for incoming request bodies.
*   Implementing Publisher Confirms and Return Handling.
*   Adding request tracing (e.g., OpenTelemetry) for better observability.
*   Implementing a Dead Letter Queue mechanism.

