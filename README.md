# TRACE Survey Processor

![Python](https://img.shields.io/badge/Python-3776AB.svg?style=for-the-badge&logo=python&logoColor=white)
![Apache Kafka](https://img.shields.io/badge/Apache_Kafka-231F20.svg?style=for-the-badge&logo=apache-kafka&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED.svg?style=for-the-badge&logo=docker&logoColor=white)
![Jenkins](https://img.shields.io/badge/Jenkins-D24939.svg?style=for-the-badge&logo=jenkins&logoColor=white)
![Semantic Release](https://img.shields.io/badge/Semantic_Release-494949.svg?style=for-the-badge&logo=semantic-release&logoColor=white)

This service consumes messages from a Kafka topic, downloads PDF files from Google Cloud Storage, extracts data from these PDFs, and publishes the processed data to another Kafka topic.

## Overview

The TRACE Survey Processor application is responsible for:

1. Consuming trace survey upload notifications from the `trace-survey-uploaded` Kafka topic (published by [api-server](https://github.com/cyse7125-sp25-team03/api-server.git))
2. Downloading the corresponding PDF files from Google Cloud Storage (GCS)
3. Extracting structured data from the PDFs (course info, instructor, ratings, and comments)
4. Publishing the processed data to the `trace-survey-processed` Kafka topic (consumed by [trace-consumer](https://github.com/cyse7125-sp25-team03/trace-consumer.git))

## Architecture

The service consists of the following components:

- **Kafka Consumer**: Consumes messages from the `trace-survey-uploaded` topic
- **GCS Client**: Downloads PDF files from Google Cloud Storage
- **PDF Extractor**: Extracts structured data from the survey PDFs
- **Kafka Producer**: Publishes processed data to the `trace-survey-processed` topic
- **Health Check Server**: Provides endpoints for Kubernetes liveness and readiness probes

## Data Flow

1. A message is received from the `trace-survey-uploaded` Kafka topic (from [api-server](https://github.com/cyse7125-sp25-team03/api-server.git)) containing the trace ID and GCS file location
2. The PDF file is downloaded from GCS
3. The PDF contents are parsed to extract:
   - Course information (ID, name, subject, etc.)
   - Instructor information
   - Ratings (question text, means, medians, etc.)
   - Student comments
4. The extracted data is published to the `trace-survey-processed` Kafka topic, where it's consumed by the [trace-consumer](https://github.com/cyse7125-sp25-team03/trace-consumer.git) service

## Configuration

The service is configured using environment variables:

| Variable | Description | Default |
|----------|-------------|---------|
| `SERVER_PORT` | HTTP server port | `8081` |
| `KAFKA_BROKERS` | Comma-separated list of Kafka brokers | `kafka-controller-0...` |
| `KAFKA_INPUT_TOPIC` | Topic to consume messages from | `trace-survey-uploaded` |
| `KAFKA_OUTPUT_TOPIC` | Topic to publish processed messages to | `trace-survey-processed` |
| `KAFKA_CONSUMER_GROUP` | Consumer group ID | `trace-processor` |
| `KAFKA_USERNAME` | Kafka username for SASL auth | `""` |
| `KAFKA_PASSWORD` | Kafka password for SASL auth | `""` |
| `MAX_RETRIES` | Maximum number of processing retries | `3` |
| `RETRY_BACKOFF_MS` | Backoff time in ms between retries | `1000` |
| `MAX_CONCURRENT_JOBS` | Maximum concurrent processing jobs | `5` |
| `PROCESSING_TIMEOUT` | Processing timeout in seconds | `300` |
| `LOG_LEVEL` | Logging level (debug, info, warn, error, fatal) | `info` |

## Health Checks

The application provides the following health check endpoints:

- `/healthz/live`: Liveness probe to check if the application is running
- `/healthz/ready`: Readiness probe to check if the application is ready to process requests, including connectivity to Kafka and GCS

## Development

### Prerequisites

- Python 3.10+
- pipenv (optional)

### Setup

1. Clone the repository
2. Install dependencies:
   ```
   pip install -r requirements.txt
   ```

### Running Locally

To run the service locally:

```
python app.py
```

## Deployment

### Building the Docker Image

```
docker build -t trace-processor:latest .
```

### Running with Docker

```
docker run -p 8081:8081 \
  -e KAFKA_BROKERS=kafka:9092 \
  -e KAFKA_CONSUMER_GROUP=trace-processor \
  trace-processor:latest
```

### Kubernetes Deployment

The service can be deployed to Kubernetes using Helm charts from the [helm-charts](https://github.com/cyse7125-sp25-team03/helm-charts.git) repository:

```bash
# Clone the helm-charts repository
git clone https://github.com/cyse7125-sp25-team03/helm-charts.git

# Install the trace-processor chart
helm install trace-processor ./trace-processor -n trace-processor
```

Alternatively, you can add the Helm repository and install directly:

```bash
# Add the Helm repository
helm repo add team03 https://github.com/cyse7125-sp25-team03/helm-charts

# Install the trace-processor chart
helm install trace-processor team03/trace-processor -n trace-processor
```

## Message Format

### Input Message (trace-survey-uploaded)

```json
{
  "traceId": "string",
  "courseId": "string",
  "fileName": "string",
  "gcsBucket": "string",
  "gcsPath": "string",
  "instructorId": "string",
  "semesterTerm": "string",
  "section": "string",
  "uploadedBy": "string",
  "uploadedAt": "2023-01-01T00:00:00Z"
}
```

### Output Message (trace-survey-processed)

```json
{
  "traceId": "string",
  "course": {
    "courseId": "string",
    "courseName": "string",
    "subject": "string",
    "catalogSection": "string",
    "semester": "string",
    "year": 2023,
    "enrollment": 25,
    "responses": 20,
    "declines": 1,
    "processedAt": "2023-01-01T00:00:00Z",
    "originalFileName": "string",
    "gcsBucket": "string",
    "gcsPath": "string"
  },
  "instructor": {
    "name": "string"
  },
  "ratings": [
    {
      "questionText": "string",
      "category": "string",
      "responses": 20,
      "responseRate": 0.8,
      "courseMean": 4.5,
      "deptMean": 4.2,
      "univMean": 4.0,
      "courseMedian": 5.0,
      "deptMedian": 4.0,
      "univMedian": 4.0
    }
  ],
  "comments": [
    {
      "category": "string",
      "questionText": "string",
      "responseNumber": 1,
      "commentText": "string"
    }
  ],
  "processedAt": "2023-01-01T00:00:00Z",
  "error": "string"
}
```

## CI/CD and Releases

This project uses Jenkins for continuous integration and Semantic Release for versioning:

- When a pull request is successfully merged, a Docker image is built
- The Semantic Versioning bot creates a release on GitHub with a tag
- The tagged release is used for the Docker image, which is then pushed to Docker Hub

## License

This project is licensed under the GNU General Public License v3.0. See the [LICENSE](LICENSE) file for details.