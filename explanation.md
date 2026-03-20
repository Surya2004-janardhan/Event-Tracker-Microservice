# Project Documentation: User Activity Ingestion System

This document provides a comprehensive, end-to-end explanation of the User Activity Ingestion System.

---

## 🏗️ Architecture Overview

The system is designed as a decoupled, message-driven architecture to handle high volumes of user activity data asynchronously.

```mermaid
graph TD
    Client[User/Client] -- "POST /api/v1/activities" --> API[API Service (Express.js)]
    API -- "Rate Limiting & Validation" --> API
    API -- "Publish Message" --> RMQ[RabbitMQ (Message Queue)]
    RMQ -- "Consume Message" --> Consumer[Consumer Service (Node.js)]
    Consumer -- "Process & Transform" --> Consumer
    Consumer -- "Save Activity" --> DB[(MongoDB)]
```

### Main Components
1.  **API Service**: The entry point for all incoming activity data. It handles authentication (rate limiting), validation, and queuing.
2.  **RabbitMQ**: Acts as a buffer and communication bridge between the API and Consumer services, ensuring reliability and scalability.
3.  **Consumer Service**: A background worker that processes activities from the queue and persists them to the database.
4.  **MongoDB**: The final data store for all processed user activities.

---

## 🚀 Workflow: Life of a Request

1.  **Ingestion**: A client sends a POST request with activity data (e.g., page views, clicks) to the API.
2.  **Throttling**: The `rateLimiter` middleware checks the client's IP address to prevent abuse.
3.  **Validation**: The `activityController` uses a Joi schema to ensure the payload contains all required fields (`userId`, `eventType`, `timestamp`, `payload`).
4.  **Queuing**: If valid, the activity is sent to the `user_activities` queue in RabbitMQ. The API returns a `202 Accepted` status immediately.
5.  **Consumption**: The Consumer service picks up the message from RabbitMQ.
6.  **Persistence**: The `activityProcessor` creates a new Mongoose document and saves it to MongoDB.
7.  **Acknowledgment**: Once saved, the Consumer acknowledges the message in RabbitMQ, removing it from the queue.

---

## 📂 Component Deep Dive

### 1. API Service (`/api/src`)

The API service is built with Express.js and focuses on fast, reliable ingestion.

#### `server.js`
- **Purpose**: Main entry point that initializes the Express app and establishes the initial RabbitMQ connection.
- **Key Functions**:
  - `startServer()`: Initializes the queue connection and starts the HTTP server.

#### `routes/activityRoutes.js`
- **Purpose**: Defines the API endpoints.
- **Endpoints**:
  - `POST /activities`: Primary endpoint for activity ingestion, protected by the `rateLimiter`.

#### `controllers/activityController.js`
- **Purpose**: Orchestrates the request handling logic.
- **Key Functions**:
  - `ingestActivity(req, res)`: The core handler. It validates the request body using Joi and calls `publishToQueue`.

#### `services/queueService.js`
- **Purpose**: Abstracts the interaction with RabbitMQ.
- **Key Functions**:
  - `connectQueue()`: Singleton-pattern function to manage the RabbitMQ connection and channel.
  - `publishToQueue(data)`: Sends stringified data to the `user_activities` queue with persistence enabled.

#### `middlewares/rateLimiter.js`
- **Purpose**: Protects the API from being overwhelmed.
- **Logic**: Uses an in-memory Map to track requests per IP within a sliding window.

### 2. Consumer Service (`/consumer/src`)

The Consumer service is a standalone Node.js application dedicated to processing and storing data.

#### `worker.js`
- **Purpose**: The main engine of the consumer. It connects to both MongoDB and RabbitMQ.
- **Logic**: It consumes messages one by one (`prefetch(1)`) to avoid overwhelming the system. If processing fails, it handles requeueing (for transient errors) or discarding (for malformed data).

#### `services/activityProcessor.js`
- **Purpose**: Handles the business logic of processing an activity.
- **Key Functions**:
  - `processActivity(data)`: Converts the raw queue data into a Mongoose model, generates a unique ID, and saves it to the database.

#### `models/Activity.js`
- **Purpose**: Defines the data structure in MongoDB.
- **Schema**:
  - `userId`: Indexed for fast querying.
  - `id`: A unique UUID for each activity instance.
  - `eventType`, `timestamp`, `payload`: The core activity data.
  - `processedAt`: Timestamp of when the system finished processing.

---

## 🛠️ Infrastructure & Setup

- **Docker Compose**: Orchestrates the entire environment (`api`, `consumer`, `rabbitmq`, `mongodb`).
- **Dependencies**:
  - `amqplib`: For RabbitMQ communication.
  - `mongoose`: For MongoDB object modeling.
  - `joi`: For robust input validation.
  - `express`: Web framework for the API.

---

## 🧪 Error Handling & Reliability

- **Graceful Shutdown**: Both services are designed to handle connection closures gracefully.
- **Message Persistence**: RabbitMQ messages are marked as `persistent: true`, and the queue is `durable: true`, ensuring data isn't lost if RabbitMQ restarts.
- **Retry Logic**: The Consumer service implements basic error handling with RabbitMQ `nack` and requeueing.
