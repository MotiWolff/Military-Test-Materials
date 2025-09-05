# Military Test Materials - Development Resources

## Overview

This repository contains comprehensive starter code and resources for building a modern military test materials analysis system. The project leverages cutting-edge technologies for processing, analyzing, and managing military test data with advanced natural language processing capabilities.

## Technology Stack

### Backend Framework
- **FastAPI** - High-performance Python web framework for building APIs
- **MongoDB** - Document-oriented NoSQL database for flexible data storage
- **Elasticsearch** - Distributed search and analytics engine

### Infrastructure & DevOps
- **Docker** - Containerization platform for consistent development and deployment
- **Kafka** - Distributed streaming platform for real-time data processing

### Natural Language Processing
- **openai-whisper** - Advanced speech-to-text transcription
- **NLTK** - Natural Language Toolkit for text processing
- **TextBlob** - Simplified text processing library

## Project Architecture

```
military-test-materials/
├── api/
│   ├── main.py              # FastAPI application entry point
│   ├── models/
│   │   ├── __init__.py
│   │   ├── test_material.py # MongoDB document models
│   │   └── analysis.py      # Analysis result models
│   ├── routers/
│   │   ├── __init__.py
│   │   ├── materials.py     # Test materials endpoints
│   │   └── analysis.py      # Analysis endpoints
│   └── services/
│       ├── __init__.py
│       ├── nlp_service.py   # NLP processing service
│       └── search_service.py # Elasticsearch integration
├── processors/
│   ├── audio_processor.py   # Whisper integration
│   ├── text_processor.py    # NLTK/TextBlob processing
│   └── kafka_consumer.py    # Kafka message processing
├── docker/
│   ├── Dockerfile
│   ├── docker-compose.yml
│   └── requirements.txt
├── config/
│   ├── elasticsearch/
│   ├── mongodb/
│   └── kafka/
└── tests/
    ├── unit/
    └── integration/
```

## Getting Started

### Prerequisites
- Python 3.9+
- Docker and Docker Compose
- MongoDB instance
- Elasticsearch cluster
- Kafka cluster

### Installation

1. Clone the repository:
```bash
git clone https://github.com/MotiWolff/Military-Test-Materials.git
cd Military-Test-Materials
```

2. Install dependencies:
```bash
pip install -r docker/requirements.txt
```

3. Set up environment variables:
```bash
cp .env.example .env
# Edit .env with your configuration
```

4. Start services using Docker Compose:
```bash
docker-compose up -d
```

## Core Components

### FastAPI Application

```python
# api/main.py
from fastapi import FastAPI, HTTPException
from motor.motor_asyncio import AsyncIOMotorClient
from elasticsearch import AsyncElasticsearch

app = FastAPI(title="Military Test Materials API")

@app.on_event("startup")
async def startup_db_client():
    app.mongodb_client = AsyncIOMotorClient("mongodb://localhost:27017")
    app.mongodb = app.mongodb_client.military_test_db
    app.elasticsearch = AsyncElasticsearch(["http://localhost:9200"])

@app.get("/health")
async def health_check():
    return {"status": "healthy"}
```

### MongoDB Models

```python
# api/models/test_material.py
from pydantic import BaseModel, Field
from typing import Optional, List
from datetime import datetime
from bson import ObjectId

class TestMaterial(BaseModel):
    id: Optional[str] = Field(alias="_id")
    title: str
    content: str
    material_type: str
    tags: List[str] = []
    created_at: datetime = Field(default_factory=datetime.utcnow)
    processed: bool = False
    
    class Config:
        allow_population_by_field_name = True
        json_encoders = {ObjectId: str}
```

### NLP Processing Service

```python
# api/services/nlp_service.py
import whisper
import nltk
from textblob import TextBlob
from typing import Dict, Any

class NLPService:
    def __init__(self):
        self.whisper_model = whisper.load_model("base")
        nltk.download('punkt')
        nltk.download('vader_lexicon')
    
    async def transcribe_audio(self, audio_path: str) -> str:
        result = self.whisper_model.transcribe(audio_path)
        return result["text"]
    
    async def analyze_sentiment(self, text: str) -> Dict[str, Any]:
        blob = TextBlob(text)
        return {
            "polarity": blob.sentiment.polarity,
            "subjectivity": blob.sentiment.subjectivity
        }
    
    async def extract_entities(self, text: str) -> List[str]:
        blob = TextBlob(text)
        return [str(phrase) for phrase in blob.noun_phrases]
```

### Elasticsearch Integration

```python
# api/services/search_service.py
from elasticsearch import AsyncElasticsearch
from typing import List, Dict, Any

class SearchService:
    def __init__(self, es_client: AsyncElasticsearch):
        self.es = es_client
    
    async def index_document(self, index: str, doc_id: str, document: Dict[str, Any]):
        await self.es.index(index=index, id=doc_id, body=document)
    
    async def search(self, index: str, query: str, size: int = 10) -> List[Dict[str, Any]]:
        body = {
            "query": {
                "multi_match": {
                    "query": query,
                    "fields": ["title^2", "content", "tags"]
                }
            },
            "size": size
        }
        
        result = await self.es.search(index=index, body=body)
        return [hit["_source"] for hit in result["hits"]["hits"]]
```

### Kafka Consumer

```python
# processors/kafka_consumer.py
from kafka import KafkaConsumer
import json
import asyncio
from api.services.nlp_service import NLPService

class MaterialProcessor:
    def __init__(self):
        self.consumer = KafkaConsumer(
            'test-materials',
            bootstrap_servers=['localhost:9092'],
            value_deserializer=lambda x: json.loads(x.decode('utf-8'))
        )
        self.nlp_service = NLPService()
    
    async def process_messages(self):
        for message in self.consumer:
            material = message.value
            
            # Process with NLP
            if material.get('type') == 'audio':
                content = await self.nlp_service.transcribe_audio(material['path'])
            else:
                content = material['content']
            
            sentiment = await self.nlp_service.analyze_sentiment(content)
            entities = await self.nlp_service.extract_entities(content)
            
            # Update database with processed data
            # Index in Elasticsearch
```

## Docker Configuration

### Dockerfile

```dockerfile
# docker/Dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY ../api ./api
COPY ../processors ./processors

EXPOSE 8000

CMD ["uvicorn", "api.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Docker Compose

```yaml
# docker/docker-compose.yml
version: '3.8'

services:
  api:
    build:
      context: ..
      dockerfile: docker/Dockerfile
    ports:
      - "8000:8000"
    environment:
      - MONGODB_URL=mongodb://mongodb:27017
      - ELASTICSEARCH_URL=http://elasticsearch:9200
      - KAFKA_BROKERS=kafka:9092
    depends_on:
      - mongodb
      - elasticsearch
      - kafka

  mongodb:
    image: mongo:5.0
    ports:
      - "27017:27017"
    volumes:
      - mongodb_data:/data/db

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.14.0
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data

  kafka:
    image: confluentinc/cp-kafka:latest
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    ports:
      - "9092:9092"
    depends_on:
      - zookeeper

  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"

volumes:
  mongodb_data:
  elasticsearch_data:
```

## Requirements

```txt
# docker/requirements.txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
motor==3.3.2
pymongo==4.6.0
elasticsearch[async]==7.17.9
kafka-python==2.0.2
openai-whisper==20231117
nltk==3.8.1
textblob==0.17.1
pydantic==2.5.0
python-multipart==0.0.6
aiofiles==23.2.1
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
pytest==7.4.3
pytest-asyncio==0.21.1
httpx==0.25.2
```

## API Endpoints

### Test Materials
- `POST /materials/` - Create new test material
- `GET /materials/` - List all materials with pagination
- `GET /materials/{id}` - Get specific material
- `PUT /materials/{id}` - Update material
- `DELETE /materials/{id}` - Delete material

### Analysis
- `POST /analysis/transcribe` - Transcribe audio file
- `POST /analysis/sentiment` - Analyze text sentiment
- `POST /analysis/entities` - Extract named entities
- `GET /analysis/search` - Search materials using Elasticsearch

### Health & Monitoring
- `GET /health` - Health check endpoint
- `GET /metrics` - Application metrics

## Testing

Run the test suite:

```bash
# Unit tests
pytest tests/unit/

# Integration tests
pytest tests/integration/

# All tests with coverage
pytest --cov=api tests/
```

## Configuration

### Environment Variables

```bash
# .env
MONGODB_URL=mongodb://localhost:27017
ELASTICSEARCH_URL=http://localhost:9200
KAFKA_BROKERS=localhost:9092
SECRET_KEY=your-secret-key-here
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30
WHISPER_MODEL=base
```

## Contributing

### Development Setup

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/amazing-feature`
3. Make your changes and add tests
4. Run the test suite: `pytest`
5. Commit your changes: `git commit -m 'Add amazing feature'`
6. Push to the branch: `git push origin feature/amazing-feature`
7. Open a Pull Request

### Code Style

- Follow PEP 8 guidelines
- Use type hints for all function parameters and return values
- Write comprehensive docstrings for classes and functions
- Maintain test coverage above 80%

### Commit Message Format

```
type(scope): short description

Longer description if needed

- List any breaking changes
- Reference issues: Closes #123
```

Types: feat, fix, docs, style, refactor, test, chore

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Security

For security concerns, please email security@example.com instead of using the issue tracker.

## Deployment

### Production Deployment

1. Set up production environment variables
2. Build Docker images: `docker-compose build`
3. Deploy using: `docker-compose up -d`
4. Monitor logs: `docker-compose logs -f`

### Scaling

- Use Docker Swarm or Kubernetes for orchestration
- Set up load balancing for the API service
- Configure MongoDB replica sets for high availability
- Use Elasticsearch clustering for better performance
