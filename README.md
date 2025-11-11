# ClaimLinker Setup Guide for Internal Server

 

## Overview

 

ClaimLinker is a Web service and API that links arbitrary text to fact-checked claims from ClaimsKG database. This guide provides step-by-step instructions for setting up ClaimLinker on our internal server environment.

 

**Paper Reference**: Maliaroudakis E., Boland K., Dietze S., Todorov K., Tzitzikas Y. and Fafalios P., "ClaimLinker: Linking Text to a Knowledge Graph of Fact-checked Claims", In Companion Proceedings of the Web Conference 2021. [Slides](https://users.ics.forth.gr/~fafalios/files/ppts/ClaimLinker_TheWebConf2021_Slides.pdf)

 

## System Architecture

 

ClaimLinker consists of three main components:

 

1. **ClaimLinker_commons**: Core library with NLP processing and similarity algorithms

2. **ClaimLinker_web**: Web services and user interfaces  

3. **ElasticSearch_Tools**: Data indexing and search functionality

 

The system requires:

- Elasticsearch 7.6.2 for storing and searching claims

- Java 17+ for running the application

- Tomcat for web service deployment

- Docker for containerized services

 

## Prerequisites and System Setup

 

### Install Required Software

 

```bash

# Update system

sudo apt-get update

 

# Install Docker

sudo apt-get install docker.io docker-compose

sudo usermod -aG docker $USER

newgrp docker

 

# Install Java 17 and Maven

sudo apt-get install openjdk-17-jdk maven

 

# Install Tomcat (adjust version as needed)

sudo apt-get install tomcat9

 

# Set system parameters for Elasticsearch

sudo sysctl -w vm.max_map_count=262144

echo 'vm.max_map_count=262144' | sudo tee -a /etc/sysctl.conf

```

 

### Verify Installations

 

```bash

# Check Docker

docker --version

docker ps

 

# Check Java

java -version

 

# Check Maven

mvn -version

 

# Check Tomcat

sudo systemctl status tomcat9

```

 

## Project Setup

 

### Clone and Build ClaimLinker

 

```bash

# Clone the repository

git clone https://github.com/malvag/ClaimLinker.git

cd ClaimLinker

 

# Install FEL library dependency (required for entity linking)

cd ClaimLinker_commons

mvn install:install-file \

  -Dfile=./lib/FEL-0.1.0-fat.jar \

  -DgroupId=com.yahoo.semsearch \

  -DartifactId=FEL \

  -Dversion=0.1.0 \

  -Dpackaging=jar \

  -DgeneratePom=true

 

# Build entire project from root directory

cd ..

mvn clean compile package

```

 

### Verify Build

 

```bash

# Check that JAR files were created

ls -la ClaimLinker_commons/target/ClaimLinker_commons-1.0-jar-with-dependencies.jar

ls -la ClaimLinker_web/target/ClaimLinker_web-1.0.jar

ls -la ElasticSearch_Tools/target/ElasticSearch_Tools-1.0.jar

 

# All three files should exist and have recent timestamps

```

 

## Data Preparation

 

Ensure you have these required files in the `data/` directory:

 

```bash

# Check required data files

ls -la data/claim_extraction_18_10_2019_annotated.csv  # Claims dataset

ls -la data/english-20200420.hash                      # FEL hash file

ls -la data/stopwords.txt                              # Stopwords for NLP

ls -la data/puncs.txt                                  # Punctuation patterns

```

 

If any files are missing, obtain them from the original ClaimLinker repository or data sources.

 

## Startup Scripts Explanation

 

The repository includes four startup scripts that automate the setup process:

 

### START_ES - Elasticsearch Container

**Purpose**: Starts Elasticsearch 7.6.2 in Docker container

 

**What it does**:

```bash

sudo docker run --name es01 --net elastic -p 9200:9200 -it -m 1GB \

  -e "xpack.ml.max_machine_memory_percent=20" \

  -e "discovery.type=single-node" \

  docker.elastic.co/elasticsearch/elasticsearch:7.6.2

```

 

**Explanation**:

- Creates container named `es01` on `elastic` network

- Maps port 9200 for API access

- Limits memory to 1GB with ML features using max 20%

- Runs in single-node mode (no clustering)

- Runs in foreground (you'll see logs continuously)

 

### START_INDEX - Data Loading

**Purpose**: Loads fact-checked claims into Elasticsearch

 

**What it does**:

```bash

java -Xmx2048m -cp \

.:ClaimLinker_commons/target/ClaimLinker_commons-1.0-jar-with-dependencies.jar:\

ClaimLinker_web/target/ClaimLinker_web-1.0.jar:\

ElasticSearch_Tools/target/ElasticSearch_Tools-1.0.jar: \

csd.claimlinker.es.ElasticInitializer \

-f data/claim_extraction_18_10_2019_annotated.csv \

-h localhost

```

 

**Explanation**:

- Allocates 2GB heap memory for processing large CSV

- Sets up Java classpath with all required JARs

- Runs ElasticInitializer to parse CSV and index claims

- Connects to Elasticsearch on localhost:9200

- Creates `claims` index with proper mappings

 

### START_DEMO - Test Application

**Purpose**: Runs demonstration of ClaimLinker functionality

 

**What it does**:

```bash

java -Xmx2048m -cp \

.:ClaimLinker_commons/target/ClaimLinker_commons-1.0-jar-with-dependencies.jar:\

ClaimLinker_web/target/ClaimLinker_web-1.0.jar:\

ElasticSearch_Tools/target/ElasticSearch_Tools-1.0.jar: \

csd.claimlinker.ClaimLinkerTest

```

 

**Explanation**:

- Runs ClaimLinkerTest class with sample text

- Demonstrates all similarity measures working together

- Shows example output format and scoring

- Useful for testing that everything is working

 

### START_KIBANA - Data Visualization

**Purpose**: Starts Kibana for exploring indexed claims

 

**What it does**:

```bash

sudo docker run --name kib01 --net elastic -p 5601:5601 \

  -e "ELASTICSEARCH_HOSTS=http://es01:9200" \

  docker.elastic.co/kibana/kibana:7.6.2

```

 

**Explanation**:

- Creates Kibana container connected to Elasticsearch

- Maps port 5601 for web interface access

- Automatically connects to the es01 container

- Provides web UI for data exploration and visualization

 

## Complete Setup Process

 

### Step 1: Prepare Docker Network

 

```bash

# Create Docker network for containers to communicate

docker network create elastic

 

# Verify network was created

docker network ls | grep elastic

```

 

### Step 2: Start Elasticsearch

 

```bash

# Make script executable (if needed)

chmod +x START_ES

 

# Start Elasticsearch (runs in foreground - keep terminal open)

./START_ES

```

 

**What you'll see**:

- Container download progress (first time only)

- Elasticsearch startup logs

- Messages about template loading and index creation

- Eventually: "Node started" message

 

**Leave this terminal running** - Elasticsearch logs will continue to show.

 

### Step 3: Load Claims Data

 

**Open a new terminal** and navigate to ClaimLinker directory:

 

```bash

cd /home/methodshub/ClaimLinker

 

# Wait for Elasticsearch to fully start (30-60 seconds)

# Test if ES is ready

curl http://localhost:9200/_cluster/health?pretty

 

# Should show status: "green" or "yellow"

# If connection refused, wait longer and retry

 

# Load the claims data

chmod +x START_INDEX

./START_INDEX

```

 

**What you'll see**:

- Connection establishment messages

- CSV file reading progress

- Bulk indexing operations

- Success message when complete

 

**Verify data loaded**:

```bash

# Check number of indexed claims

curl "localhost:9200/claims/_count?pretty"

 

# Should show count > 0, typically several thousand claims

```

 

### Step 4: Test with Demo

 

```bash

# Run the demonstration

chmod +x START_DEMO

./START_DEMO

```

 

**What you'll see**:

- "Demo pipeline started!" message

- Processing of sample text: "Of course, we are 5 percent of the world's population;"

- List of similar claims found with similarity scores

- Execution time

 

### Step 5: Start Kibana (Optional)

 

```bash

# Start Kibana for data exploration

chmod +x START_KIBANA

./START_KIBANA

```

 

**Access Kibana**:

- Wait 2-3 minutes for Kibana to start

- Open browser: http://localhost:5601

- Create index pattern: `claims*`

- Explore data in Discover tab

 

## Setting Up Web Service with Tomcat

 

### Deploy ClaimLinker Web Application

 

```bash

# Start Tomcat using the correct path and user

sudo -u tomcat /opt/tomcat/bin/startup.sh

 

# Check Tomcat status

curl -I http://localhost:8080

 

# Build WAR file (if not already built)

mvn clean package

 

# Deploy to Tomcat

sudo cp ClaimLinker_web/target/ClaimLinker_web-1.0.war /opt/tomcat/webapps/claimlinker.war

 

# Restart Tomcat to deploy

sudo -u tomcat /opt/tomcat/bin/shutdown.sh

sleep 5

sudo -u tomcat /opt/tomcat/bin/startup.sh

 

# Wait for deployment (30-60 seconds)

sleep 60

```

 

### Test Web Service

 

```bash

# Test basic endpoint

curl "http://localhost:8080/claimlinker"

 

# Test with sample query

curl "http://localhost:8080/claimlinker?app=service&text=vaccine%20safety"

 

# Test with political claim

curl "http://localhost:8080/claimlinker?app=service&text=Interest%20on%20debt%20will%20exceed%20secur…

```

 

**Expected Response Format**:

```json

{

  "_results": [{

    "text": "vaccine safety",

    "sentencePosition": 0,

    "association_type": "same_as",

    "linkedClaims": [

      {

        "claimReview_claimReviewed": "Vaccines are safe and effective",

        "_score": 45.2,

        "extra_title": "Fact check title",

        "rating_alternateName": "true",

        "creativeWork_author_name": "Fact checker name",

        "claimReview_url": "https://factcheck.url",

        "claim_uri": "http://data.gesis.org/claimskg/creative_work/id"

      }

    ]

  }],

  "timeElapsed": 1200

}

```

 

## Making Data Persistent

 

The default setup loses data when containers restart. To make data persistent:

 

### Create Persistent Elasticsearch
