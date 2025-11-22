# Kafka Stock Market Data Streaming

A real-time data streaming application that demonstrates Apache Kafka producer and consumer patterns for processing stock market data. This project streams stock data from a CSV file through Kafka topics and enables real-time data consumption and processing.

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage](#usage)
  - [Setting up Kafka](#setting-up-kafka)
  - [Running the Producer](#running-the-producer)
  - [Running the Consumer](#running-the-consumer)
- [Project Structure](#project-structure)
- [Key Components](#key-components)
- [Troubleshooting](#troubleshooting)
- [Dependencies](#dependencies)

## 🎯 Overview

This project implements a Kafka-based data pipeline for streaming stock market data. It includes:
- **Producer**: Reads stock data from `indexProcessed.csv` and publishes it to a Kafka topic
- **Consumer**: Subscribes to the Kafka topic and processes incoming stock data in real-time
- Real-time data streaming with 1-second intervals between messages
- JSON serialization for efficient data transmission

## 🏗️ Architecture

### End-to-End Data Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Dataset (indexProcessed.csv)                                   │
│           │                                                     │
│           ▼                                                     │
│  ┌──────────────────────┐                                       │
│  │ Stock Market App     │                                       │
│  │ Simulation           │                                       │
│  │ (SDK Boto3)          │                                       │
│  └──────────┬───────────┘                                       │
│             │                                                   │
└─────────────┼───────────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Apache Kafka (EC2)                             │
│                    test-topic                                   │
└─────────────────────┬───────────────────────────────────────────┘
                      │
          ┌───────────┴───────────┐
          │                       │
          ▼                       ▼
    ┌──────────────┐      ┌─────────────────┐
    │  Consumer    │      │  AWS Services   │
    │  (Processing)│      │  Integration    │
    └──────────────┘      └────────┬────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
              ▼                    ▼                    ▼
        ┌──────────┐        ┌────────────┐      ┌─────────────┐
        │ S3       │        │ AWS Glue   │      │ Amazon      │
        │ (Data    │        │ Data       │      │ Athena      │
        │ Storage) │        │ Catalog    │      │ (SQL        │
        └──────────┘        └─────┬──────┘      │  Queries)   │
                                  │             └─────────────┘
                                  ▼
                          ┌────────────────┐
                          │ Crawler        │
                          │ (Auto-discover │
                          │  & catalog)    │
                          └────────────────┘
```

### Data Flow Components

1. **Source**: Stock market dataset (`indexProcessed.csv`)
2. **Producer**: Reads and streams data to Kafka topic
3. **Kafka Broker**: Message queue (running on EC2)
4. **Consumer**: Processes streaming data
5. **AWS Integration**:
   - **Amazon S3**: Store processed data
   - **AWS Glue**: Data catalog and ETL
   - **Amazon Athena**: Query processed data using SQL
   - **Crawler**: Automatically discover and catalog data schema

## 📦 Prerequisites

### System Requirements
- **macOS** or Linux/Windows with appropriate shell environment
- **Python 3.7+**
- **Java 17+** (required for Kafka)
- **AWS EC2 Instance** (t2.micro recommended for testing)
- **SSH access** to your EC2 instance

### Software
- Apache Kafka 4.0.1 or higher
- Python package manager: `uv` or `pip`

## 🚀 Installation

### 1. Clone the Repository

```bash
git clone https://github.com/ak-pydev/kafka_stock_market_data_streaming.git
cd kafka_stock_market_data_streaming
```

### 2. Install Python Dependencies

```bash
# Using uv (recommended)
uv pip install -r requirements.txt

# Or using pip
pip install -r requirements.txt
```

### 3. Install and Configure Kafka on EC2

#### Step 1: Launch an AWS EC2 Instance
- Create a t2.micro instance (eligible for free tier)
- Configure security group to allow SSH (port 22) and Kafka (port 9092)

#### Step 2: Connect to Your Instance
```bash
# Set proper permissions for your key pair (macOS)
chmod 400 your-key-pair.pem

# Connect via SSH
ssh -i your-key-pair.pem ec2-user@your-ec2-public-ip
```

#### Step 3: Download and Extract Kafka
```bash
wget https://downloads.apache.org/kafka/4.0.1/kafka_2.13-4.0.1.tgz
tar -xzf kafka_2.13-4.0.1.tgz
cd kafka_2.13-4.0.1
```

#### Step 4: Install Java
```bash
# For Amazon Linux 2
sudo yum install -y java-17-amazon-corretto
```

## ⚙️ Configuration

### Kafka Cluster Setup with KRaft

Kafka 4.0.1 uses KRaft (Kafka Raft) instead of ZooKeeper. Follow these steps:

#### 1. Generate Cluster ID
```bash
KAFKA_CLUSTER_ID="$(bin/kafka-storage.sh random-uuid)"
echo $KAFKA_CLUSTER_ID
```

#### 2. Initialize KRaft Metadata
```bash
bin/kafka-storage.sh format -t $KAFKA_CLUSTER_ID -c config/kraft/server.properties
```

#### 3. Edit Server Configuration

If Kafka fails to start, configure heap memory and update `config/server.properties`:

**Update heap memory** in `bin/kafka-server-start.sh` (line 29):
```bash
export KAFKA_HEAP_OPTS="-Xmx256M -Xms128M"
```

**Update** `config/server.properties`:
```properties
process.roles=broker,controller
node.id=1
controller.quorum.voters=1@localhost:9093

controller.listener.names=CONTROLLER
listeners=PLAINTEXT://:9092,CONTROLLER://:9093
```

#### 4. Format Storage and Start
```bash
KAFKA_CLUSTER_ID="$(bin/kafka-storage.sh random-uuid)"
bin/kafka-storage.sh format -t $KAFKA_CLUSTER_ID -c config/server.properties --standalone

# Start Kafka broker
bin/kafka-server-start.sh config/server.properties
```

### 3. Create Kafka Topic
```bash
bin/kafka-topics.sh --create \
  --topic test-topic \
  --bootstrap-server localhost:9092 \
  --partitions 1 \
  --replication-factor 1
```

### 4. Update Producer Configuration

In `kakfaProducer.ipynb`, update the broker address:
```python
producer = KafkaProducer(
    bootstrap_servers=['YOUR_EC2_PUBLIC_IP:9092'], 
    value_serializer=lambda x: dumps(x).encode('utf-8')
)
```

Replace `YOUR_EC2_PUBLIC_IP` with your actual EC2 instance public IP address.

## ⚡ Quick Start

### Minimal Setup (Development)

```bash
# 1. Clone and setup
git clone https://github.com/ak-pydev/kafka_stock_market_data_streaming.git
cd kafka_stock_market_data_streaming
uv pip install -r requirements.txt

# 2. On EC2: Start Kafka
ssh -i key.pem ec2-user@YOUR_EC2_IP
cd kafka_2.13-4.0.1
bin/kafka-server-start.sh config/server.properties

# 3. On EC2: Create topic
bin/kafka-topics.sh --create \
  --topic test-topic \
  --bootstrap-server localhost:9092 \
  --partitions 1 \
  --replication-factor 1

# 4. Locally: Update and run producer (kakfaProducer.ipynb)
# 5. Locally: Run consumer (kafkaConsumer.ipynb)
```

## 📖 Usage

### Running the Producer

The producer reads random samples from the stock market dataset and publishes them to Kafka:

1. Open `kakfaProducer.ipynb` in Jupyter
2. Execute cells sequentially:
   - **Cell 1**: Import required libraries
   - **Cell 2**: Initialize Kafka producer with your broker address
   - **Cell 3**: Test connection (optional)
   - **Cell 4**: Load CSV data
   - **Cell 5**: Start streaming (publishes one message per second)
   - **Cell 6**: Flush producer to ensure all messages are sent

**Key Features:**
- Streams data at 1-second intervals
- Randomly samples rows from the dataset
- Automatically serializes data to JSON format
- Can be terminated with keyboard interrupt (Ctrl+C)

### Running the Consumer

The consumer subscribes to the Kafka topic and processes incoming messages:

1. Open `kafkaConsumer.ipynb` in Jupyter
2. Execute cells to:
   - Import required libraries
   - Connect to Kafka broker
   - Subscribe to the topic
   - Process and display incoming messages in real-time

**Expected Output:**
- Messages will be displayed as they arrive
- Each message contains a complete stock record from the dataset

## 📁 Project Structure

```
kafka_stock_market_data_streaming/
├── README.md                    # This file
├── kakfaProducer.ipynb         # Kafka producer implementation
├── kafkaConsumer.ipynb         # Kafka consumer implementation
├── indexProcessed.csv          # Stock market dataset
├── requirements.txt            # Python dependencies
├── starter_guide.txt           # Quick reference guide
└── .git/                       # Git repository
```

## 🔑 Key Components

### Dependencies (from `requirements.txt`)
- **kafka-python**: Apache Kafka client library
- **pandas**: Data manipulation and CSV handling
- **numpy**: Numerical computing
- **notebook**: Jupyter notebook environment
- **s3fs**: S3 filesystem interface
- **boto3**: AWS SDK for Python

### Data Files
- **indexProcessed.csv**: Preprocessed stock market index data

### Notebooks
- **kakfaProducer.ipynb**: Produces stock market data to Kafka
- **kafkaConsumer.ipynb**: Consumes and processes data from Kafka

## 🌐 AWS Integration Setup

### Prerequisites for AWS Services

1. **AWS Account** with appropriate permissions
2. **IAM User** with permissions for S3, Glue, and Athena
3. **AWS CLI** configured on your local machine
4. **boto3** and **s3fs** installed (included in requirements)

### 1. Create S3 Bucket for Data Storage

```bash
# Create a new S3 bucket
aws s3 mb s3://kafka-stock-market-data-$(date +%s)

# Set bucket name as environment variable
export AWS_BUCKET_NAME="your-bucket-name"
```

### 2. Configure IAM Permissions

Create an IAM policy for your user with the following permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::kafka-stock-market-data-*",
        "arn:aws:s3:::kafka-stock-market-data-*/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "glue:CreateDatabase",
        "glue:CreateTable",
        "glue:UpdateTable",
        "glue:GetDatabase",
        "glue:GetTable",
        "glue:BatchCreatePartition"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "athena:StartQueryExecution",
        "athena:GetQueryExecution",
        "athena:GetQueryResults",
        "athena:StopQueryExecution"
      ],
      "Resource": "*"
    }
  ]
}
```

### 3. Configure AWS Credentials

```bash
# Configure AWS CLI with your credentials
aws configure

# Or set environment variables
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_DEFAULT_REGION="us-east-1"
```

### 4. Enhanced Consumer with S3 Integration

The consumer can be extended to save processed data to S3:

```python
import boto3
import pandas as pd
from kafka import KafkaConsumer
from json import loads

# Initialize S3 client
s3_client = boto3.client('s3')
bucket_name = 'your-bucket-name'

# Initialize Kafka consumer
consumer = KafkaConsumer(
    'test-topic',
    bootstrap_servers=['YOUR_EC2_IP:9092'],
    value_deserializer=lambda x: loads(x.decode('utf-8')),
    group_id='stock-market-consumer',
    auto_offset_reset='earliest'
)

# Collect messages
messages = []
for message in consumer:
    messages.append(message.value)
    
    # Save to S3 every 100 messages
    if len(messages) >= 100:
        df = pd.DataFrame(messages)
        timestamp = pd.Timestamp.now().strftime('%Y%m%d_%H%M%S')
        
        s3_key = f'stock-data/processed_{timestamp}.parquet'
        df.to_parquet(f's3://{bucket_name}/{s3_key}')
        messages = []
```

### 5. Set Up AWS Glue Crawler

```bash
# Create Glue database
aws glue create-database \
  --database-input Name=stock_market_db,Description="Stock market data database"

# Create Glue crawler
aws glue create-crawler \
  --name stock-market-crawler \
  --role arn:aws:iam::YOUR_ACCOUNT_ID:role/GlueRole \
  --database-target DatabaseName=stock_market_db \
  --s3-targets Path=s3://your-bucket-name/stock-data/
```

### 6. Query Data with Amazon Athena

```python
import boto3
import pandas as pd

athena_client = boto3.client('athena')

# Run a query
response = athena_client.start_query_execution(
    QueryString='SELECT * FROM stock_market_db.stock_data LIMIT 10;',
    QueryExecutionContext={'Database': 'stock_market_db'},
    ResultConfiguration={'OutputLocation': 's3://your-bucket-name/athena-results/'}
)

# Fetch results
query_execution_id = response['QueryExecutionId']
results = athena_client.get_query_results(QueryExecutionId=query_execution_id)

# Convert to DataFrame
df = pd.DataFrame(results['ResultSet']['Rows'])
print(df)
```

## 📊 Data Schema

The stock market data contains the following fields:

| Field | Type | Description |
|-------|------|-------------|
| Date | String | Trading date |
| Open | Float | Opening price |
| High | Float | Highest price of the day |
| Low | Float | Lowest price of the day |
| Close | Float | Closing price |
| Volume | Integer | Trading volume |
| Adj Close | Float | Adjusted closing price |

Example message:
```json
{
  "Date": "2024-01-15",
  "Open": 150.25,
  "High": 152.80,
  "Low": 149.50,
  "Close": 151.75,
  "Volume": 2500000,
  "Adj Close": 151.75
}
```

## 🐛 Troubleshooting

### Issue: Kafka server won't start

**Solution:**
1. Check Java installation: `java -version`
2. Reduce heap memory in `bin/kafka-server-start.sh`:
   ```bash
   export KAFKA_HEAP_OPTS="-Xmx256M -Xms128M"
   ```
3. Ensure proper KRaft initialization with correct storage format

### Issue: Connection refused (producer/consumer)

**Solution:**
1. Verify Kafka is running on EC2
2. Check security group allows traffic on port 9092
3. Confirm bootstrap server address is correct
4. Test connection: `bin/kafka-console-consumer.sh --bootstrap-server YOUR_IP:9092 --topic test-topic`

### Issue: Permission denied when connecting via SSH

**Solution:**
```bash
chmod 400 your-key-pair.pem
```

### Issue: Serialization errors

**Solution:**
Ensure JSON serializer is configured in producer:
```python
value_serializer=lambda x: dumps(x).encode('utf-8')
```

### Issue: AWS S3 Access Denied

**Solution:**
1. Verify IAM credentials are correctly configured
2. Check bucket name is correct
3. Ensure IAM user has S3 permissions
4. Test credentials: `aws s3 ls`

### Issue: Athena query returns empty results

**Solution:**
1. Verify data was successfully written to S3
2. Check Glue crawler has run and discovered the table
3. Confirm the database and table names in the query
4. Check S3 path matches crawler configuration

## ✅ Best Practices

### Producer Best Practices
- **Batch Messages**: Increase throughput by batching multiple messages
  ```python
  producer = KafkaProducer(
      bootstrap_servers=['YOUR_IP:9092'],
      value_serializer=lambda x: dumps(x).encode('utf-8'),
      batch_size=16384,  # 16KB batches
      linger_ms=10  # Wait up to 10ms for batching
  )
  ```

- **Error Handling**: Implement proper error callbacks
  ```python
  def on_send_error(exc):
      print(f"Error sending message: {exc}")
  
  producer.send('test-topic', value=data, on_error=on_send_error)
  ```

- **Compression**: Enable compression to reduce bandwidth
  ```python
  producer = KafkaProducer(
      bootstrap_servers=['YOUR_IP:9092'],
      compression_type='snappy'  # or 'gzip'
  )
  ```

### Consumer Best Practices
- **Consumer Groups**: Use consumer groups for scalability
  ```python
  consumer = KafkaConsumer(
      'test-topic',
      bootstrap_servers=['YOUR_IP:9092'],
      group_id='stock-market-group',
      auto_offset_reset='earliest'
  )
  ```

- **Offset Management**: Commit offsets periodically
  ```python
  consumer = KafkaConsumer(
      'test-topic',
      bootstrap_servers=['YOUR_IP:9092'],
      enable_auto_commit=True,
      auto_commit_interval_ms=1000  # Commit every 1 second
  )
  ```

### Data Pipeline Best Practices
- **Monitoring**: Implement monitoring and alerting
- **Backups**: Regularly backup data in S3
- **Partitioning**: Use appropriate S3 partitioning for efficient querying
  ```python
  # Example: s3://bucket/stock-data/year=2024/month=01/day=15/
  ```
- **Data Validation**: Validate incoming data before storing
- **Documentation**: Keep data schema documentation updated in Glue Catalog

### Security Best Practices
- **Use VPC**: Deploy Kafka within a VPC for isolation
- **Enable SASL/SSL**: Use authentication and encryption for production
- **IAM Roles**: Use IAM roles instead of hardcoded credentials
- **Encryption**: Enable S3 encryption at rest
- **Network**: Restrict security group to trusted IPs only

## 📈 Monitoring and Metrics

### Kafka Metrics to Monitor
- **Throughput**: Messages per second
- **Latency**: End-to-end message delivery time
- **Consumer Lag**: Difference between latest and consumed offset
- **Broker Disk Usage**: Monitor to prevent running out of space

### Check Consumer Lag
```bash
bin/kafka-consumer-groups.sh \
  --bootstrap-server YOUR_IP:9092 \
  --group stock-market-group \
  --describe
```

### AWS CloudWatch Metrics
- S3 object count and size
- Glue crawler execution time
- Athena query execution time and scanned bytes

## 🚀 Advanced Deployment

### Production Deployment Considerations

#### 1. Containerization with Docker

Create a `Dockerfile` for containerized deployment:

```dockerfile
FROM python:3.10-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["jupyter", "notebook", "--ip=0.0.0.0", "--allow-root"]
```

#### 2. Multi-Broker Kafka Cluster

For production, set up multiple Kafka brokers:

```properties
# config/server.properties
process.roles=broker,controller
node.id=1
controller.quorum.voters=1@broker1:9093,2@broker2:9093,3@broker3:9093
listeners=PLAINTEXT://broker1:9092,CONTROLLER://broker1:9093
```

#### 3. Enable SASL/SSL Authentication

```python
from kafka import KafkaProducer
from kafka.errors import KafkaError

producer = KafkaProducer(
    bootstrap_servers=['broker1:9092', 'broker2:9092', 'broker3:9092'],
    security_protocol='SASL_SSL',
    sasl_mechanism='PLAIN',
    sasl_plain_username='username',
    sasl_plain_password='password',
    ssl_cafile='/path/to/ca-cert',
    ssl_certfile='/path/to/client-cert',
    ssl_keyfile='/path/to/client-key'
)
```

#### 4. Infrastructure as Code (Terraform)

Use Terraform to provision AWS resources:

```hcl
# main.tf
resource "aws_s3_bucket" "kafka_data" {
  bucket = "kafka-stock-market-data"
}

resource "aws_glue_catalog_database" "stock_market" {
  name = "stock_market_db"
}

resource "aws_ec2_instance" "kafka_broker" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.medium"
  key_name      = aws_key_pair.deployer.key_name
}
```

#### 5. Kubernetes Deployment

Deploy Kafka and consumer using Kubernetes:

```yaml
# kafka-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-consumer
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kafka-consumer
  template:
    metadata:
      labels:
        app: kafka-consumer
    spec:
      containers:
      - name: consumer
        image: kafka-consumer:latest
        env:
        - name: KAFKA_BROKERS
          value: "kafka-0.kafka-headless:9092"
        - name: AWS_BUCKET
          valueFrom:
            configMapKeyRef:
              name: aws-config
              key: bucket-name
```

## 📚 Additional Resources

- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [AWS Glue Documentation](https://docs.aws.amazon.com/glue/)
- [Amazon Athena Documentation](https://docs.aws.amazon.com/athena/)
- [boto3 Documentation](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html)
- [kafka-python Documentation](https://kafka-python.readthedocs.io/)

## 📊 Performance Notes

- Current setup uses 1-second intervals between messages for demonstration
- Can be adjusted in the producer: `sleep(1)` 
- Single partition configuration suitable for development/testing
- For production, increase replication factor and optimize partitioning
- Expected throughput: ~1,000 messages/second on t2.micro (with batching: 5,000-10,000 msg/s)

## 🔐 Security Considerations

- Currently uses PLAINTEXT authentication (development only)
- For production, implement SASL/SSL authentication
- Restrict EC2 security group to trusted IP ranges
- Store credentials securely using AWS Secrets Manager or environment variables
- Enable encryption in transit (TLS/SSL) and at rest (S3, EBS)
- Use IAM roles instead of access keys when possible
- Enable VPC Flow Logs for network monitoring
- Implement CloudTrail for audit logging

## 📝 License

This project is part of a data streaming learning exercise.

## 👨‍💻 Author

**Aaditya Khanal** - ak-pydev

## 🤝 Contributing

Feel free to fork this project and submit pull requests for improvements.

**Last Updated:** November 2025
