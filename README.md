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

```
┌─────────────────────┐
│  indexProcessed.csv │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│     Producer        │ (Samples data and publishes to Kafka)
│  kakfaProducer.ipynb│
└──────────┬──────────┘
           │
           ▼
    ┌──────────────┐
    │ Kafka Broker │
    │  test-topic  │
    └──────────┬───┘
           │
           ▼
┌─────────────────────┐
│     Consumer        │ (Subscribes and processes data)
│ kafkaConsumer.ipynb │
└─────────────────────┘
```

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
- **s3fs & boto3**: AWS S3 integration (optional)

### Data Files
- **indexProcessed.csv**: Preprocessed stock market index data

### Notebooks
- **kakfaProducer.ipynb**: Produces stock market data to Kafka
- **kafkaConsumer.ipynb**: Consumes and processes data from Kafka

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

## 📊 Performance Notes

- Current setup uses 1-second intervals between messages for demonstration
- Can be adjusted in the producer cell: `sleep(1)` 
- Single partition configuration suitable for development/testing
- For production, increase replication factor and optimize partitioning

## 🔐 Security Considerations

- Currently uses PLAINTEXT authentication (development only)
- For production, implement SASL/SSL authentication
- Restrict EC2 security group to trusted IP ranges
- Store credentials securely using environment variables

## 📝 License

This project is part of a data streaming learning exercise.

## 👨‍💻 Author

**Aaditya Khanal** - ak-pydev

## 🤝 Contributing

Feel free to fork this project and submit pull requests for improvements.

**Last Updated:** November 2025
