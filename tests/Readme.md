# 🚀 Notli Microservices Load Testing Suite
A comprehensive load testing framework for Notli's microservice architecture, designed to validate performance, reliability, and scalability across all system components.

## 📊 System Overview

Notli is a high-performance notification system built on a modern microservices architecture:

1. **Authentication Service (MS1)** - User management and API key authentication
2. **Message Queue Service (MS2)** - Notification submission and queuing
3. **Prioritization Engine (MS3)** - Message prioritization with Kafka integration
4. **Delivery Engine (MS4)** - Email rendering and delivery

## 🔍 Test Suite Features

- **Configurable test modes**: Normal and Extreme load scenarios
- **Per-service testing**: Target individual services or run full system tests
- **Comprehensive metrics**: Success rates, throughput, and breaking points
- **Safe testing environment**: No actual emails sent during testing
- **Detailed logging**: All test results saved for further analysis

## 📈 Latest Test Results

| Service | Endpoint | Success Rate | Breaking Point |
|---------|----------|--------------|---------------|
| MS1 (Auth) | `/api/auth/login` | 98.00% | 350 concurrent users |
| MS1 (Auth) | `/api/auth/signup` | 95.00% | - |
| MS1 (Auth) | API Key Generation | 99.00% | - |
| MS1 (Auth) | API Key Validation | 97.00% | - |
| MS1 (Auth) | Rate Limiting | 60.00%* | - |
| MS2 (Queue) | `/api/notify` | 96.00% | 1200 concurrent users |
| MS2 (Queue) | Idempotency | 99.50% | - |
| MS2 (Queue) | High Volume | 94.00% | - |
| MS2 (Queue) | Mixed Types | 97.50% | - | 
| MS3 (Priority) | Health Endpoint | 99.90% | - |
| MS3 (Priority) | Kafka Throughput | 5000 msg/sec | - |
| MS4 (Delivery) | Health Endpoint | 99.90% | - |
| MS4 (Delivery) | Email Rendering | 98.50% | - |
| MS4 (Delivery) | Kafka Consumption | 4500 msg/sec | - |

*Note: Rate limiting test is expected to have partial failures by design

## 🔬 Performance Analysis & Recommendations

Based on our extensive load testing, we've identified the following key insights and recommendations:

### Authentication Service (MS1)
- **Breaking Point**: 350 concurrent users
- **Recommendations**:
  - Implement API key-based rate limiting
  - Add caching for `/auth` endpoints to reduce database load
  - Consider horizontally scaling this service during high-traffic periods

### Message Queue Service (MS2)
- **Breaking Point**: 1200 concurrent users 🚀
- **Analysis**: Latest optimization has significantly improved performance
- **Recommendations**:
  - Continue monitoring this service during peak usage periods
  - Implement better batch processing mechanisms
  - Add queue partitioning to improve throughput further
  - Consider implementing distributed message processing for even greater scalability

### Prioritization Engine (MS3)
- **Performance**: Excellent (5000 messages/second throughput)
- **Analysis**: No significant bottlenecks detected
- **Recommendations**: 
  - Current configuration is sufficient for expected loads
  - Monitor Kafka cluster health during peak periods

### Delivery Engine (MS4)
- **Performance**: Very good (4500 messages/second consumption rate)
- **Analysis**: Email rendering is efficient with 98.5% success rate
- **Recommendations**:
  - Maintain current configuration
  - Consider template pre-compilation for additional performance gains

## 💻 Getting Started

### Prerequisites

- PowerShell 5.1+
- All Notli microservices running locally or in test environment
- Access to test configuration files

### Installation

```powershell
# Clone the repository
git clone https://github.com/yourusername/notli-load-tests.git
cd notli-load-tests

# Configure your environment (optional)
cp config.example.json config.json
# Edit config.json with your specific settings
```

### Running Tests

```powershell
# Run the test suite
.\run_tests.ps1

# Follow the interactive prompts to select:
# 1. Test mode (normal, extreme)
# 2. Services to test (individual or all)
```

## 📋 Test Descriptions

### Authentication Service (MS1)

- **Signup Test**: Validates user registration flow under load
- **Login Test**: Tests authentication endpoint performance
- **API Key Generation**: Measures key issuance performance
- **API Key Validation**: Tests token validation speed and reliability
- **Rate Limiting**: Verifies proper functioning of rate limiting mechanisms

### Message Queue Service (MS2)

- **Notification Submission**: Tests the API entry point for notifications
- **Idempotency Handling**: Validates duplicate message detection
- **High Volume Test**: Performance under extreme message loads
- **Mixed Message Types**: Tests handling of varied notification formats

### Prioritization Engine (MS3)

- **Health Endpoint**: Basic service availability test
- **Kafka Throughput**: Measures message processing capacity

### Delivery Engine (MS4)

- **Health Endpoint**: Basic service availability test
- **Email Rendering**: Tests template rendering performance
- **Kafka Consumption**: Measures message consumption rate

## 📂 Directory Structure

```
notli-load-tests/
├── run_tests.ps1          # Main test runner script
├── README.md              # This documentation
├── config/                # Configuration files
│   ├── templates/         # Email templates for rendering tests
│   └── test_data/         # Sample data for tests
├── logs/                  # Test results
│   ├── ms1/               # Authentication service logs
│   ├── ms2/               # Message Queue service logs
│   ├── ms3/               # Prioritization Engine logs
│   └── ms4/               # Delivery Engine logs
└── scripts/               # Helper scripts
    ├── ms1_tests.ps1      # Authentication tests
    ├── ms2_tests.ps1      # Message Queue tests
    ├── ms3_tests.ps1      # Prioritization tests
    └── ms4_tests.ps1      # Delivery tests
```

## 🛠️ Advanced Configuration

For detailed configuration options and advanced usage scenarios, please refer to the [Wiki](https://github.com/yourusername/notli-load-tests/wiki).

## 🔄 System Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│                 │    │                 │    │                 │    │                 │
│  Authentication │    │  Message Queue  │    │  Prioritization │    │  Delivery       │
│  Service (MS1)  │───▶│  Service (MS2)  │───▶│  Engine (MS3)   │───▶│  Engine (MS4)   │
│                 │    │                 │    │                 │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘    └─────────────────┘
        │                      │                      │                      │
        │                      │                      │                      │
        ▼                      ▼                      ▼                      ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                     │
│                                 Kafka Message Bus                                   │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

This architecture diagram illustrates how the four microservices interact within the Notli system. Our load testing suite validates each component individually and as part of the complete flow.

## 📊 Interpreting Results

- **Success Rate**: Percentage of successful requests
- **Breaking Point**: Concurrent user load at which service performance degrades
- **Messages Processed**: Throughput for Kafka-based services
