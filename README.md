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

 

#### Obtaining the Docker Image

```bash
# Pull the specific Elasticsearch version
sudo docker pull docker.elastic.co/elasticsearch/elasticsearch:7.6.2

# Verify the image was downloaded
sudo docker images | grep elasticsearch
```

#### Running Elasticsearch in Background

You have two options for running Elasticsearch in the background:

**Option A: Using Docker Detached Mode**
```bash
# Run container in detached mode (-d flag)
sudo docker run -d \
  --name es01 \
  --net elastic \
  -p 9200:9200 \
  -p 9300:9300 \
  -m 1GB \
  -e "xpack.ml.max_machine_memory_percent=20" \
  -e "discovery.type=single-node" \
  docker.elastic.co/elasticsearch/elasticsearch:7.6.2

# Check if container is running
sudo docker ps | grep es01

# View logs
sudo docker logs -f es01
```

**Option B: Using Screen**
```bash
# Install screen if not available
sudo apt-get install screen

# Start a screen session
screen -S elasticsearch

# Within the screen session, run START_ES or the docker command
./START_ES
# OR
sudo docker run --name es01 --net elastic -p 9200:9200 -it -m 1GB \
  -e "xpack.ml.max_machine_memory_percent=20" \
  -e "discovery.type=single-node" \
  docker.elastic.co/elasticsearch/elasticsearch:7.6.2

# Detach from screen: Press Ctrl+A, then D

# Reattach to screen session later
screen -r elasticsearch

# List all screen sessions
screen -ls

# Kill screen session when done
screen -X -S elasticsearch quit
```

**What you'll see**:
- Container download progress (first time only)
- Elasticsearch startup logs
- Messages about template loading and index creation
- Eventually: "Node started" message

**For background mode**: Use `docker logs -f es01` to view logs anytime.

 

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

### Understanding the Deployment Structure

** Important Note**: The standard Maven WAR packaging does not work correctly for this project as described in the [original GitHub documentation](https://github.com/malvag/ClaimLinker). Based on our working setup, here's how to properly deploy ClaimLinker.

### Actual Working Deployment Structure

The ClaimLinker application should be deployed to `/opt/tomcat/webapps/claims/` with the following structure:
```
/opt/tomcat/webapps/claims/
├── ClaimLinker.jsp
├── ClaimLinker_web/
│   ├── ClaimLinker_web.iml
│   ├── META-INF/
│   ├── loading.gif
│   ├── pom.xml
│   ├── resource/
│   ├── src/
│   ├── target/
│   └── web/
├── ClaimLinker_web-1.0.jar
├── META-INF/
│   └── application.xml
├── WEB-INF/
│   ├── classes/
│   ├── data/
│   ├── lib/
│   └── web.xml
├── bookmarklet.js
├── index.html
├── loading.gif
└── resource/
    ├── css/
    └── js/
```

### Deploying ClaimLinker

#### Step 1: Build the Project
```bash
# Navigate to ClaimLinker directory
cd /home/methodshub/ClaimLinker

# Build the project
mvn clean package

# Verify build was successful
ls -la ClaimLinker_web/target/
```

#### Step 2: Deploy to Tomcat
```bash
# Create the deployment directory
sudo mkdir -p /opt/tomcat/webapps/claims

# Copy all necessary files to the deployment directory
# Copy JSP files and web resources from project root
sudo cp ClaimLinker_web/web/ClaimLinker.jsp /opt/tomcat/webapps/claims/
sudo cp ClaimLinker_web/web/index.html /opt/tomcat/webapps/claims/
sudo cp ClaimLinker_web/web/bookmarklet.js /opt/tomcat/webapps/claims/
sudo cp ClaimLinker_web/web/loading.gif /opt/tomcat/webapps/claims/

# Copy the entire ClaimLinker_web directory
sudo cp -r ClaimLinker_web /opt/tomcat/webapps/claims/

# Copy the built JAR file
sudo cp ClaimLinker_web/target/ClaimLinker_web-1.0.jar /opt/tomcat/webapps/claims/

# Copy WEB-INF directory with compiled classes and configuration
sudo cp -r ClaimLinker_web/target/ClaimLinker_web-1.0/WEB-INF /opt/tomcat/webapps/claims/

# Copy META-INF directory
sudo cp -r ClaimLinker_web/target/ClaimLinker_web-1.0/META-INF /opt/tomcat/webapps/claims/

# Copy resource directory (CSS, JS, etc.)
sudo cp -r ClaimLinker_web/web/resource /opt/tomcat/webapps/claims/
```

#### Step 3: Set Proper Permissions
```bash
# Set ownership to tomcat user
sudo chown -R tomcat:tomcat /opt/tomcat/webapps/claims

# Verify the structure
ls -la /opt/tomcat/webapps/claims/
ls -la /opt/tomcat/webapps/claims/WEB-INF/
ls -la /opt/tomcat/webapps/claims/META-INF/
ls -la /opt/tomcat/webapps/claims/resource/
```

#### Step 4: Start/Restart Tomcat
```bash
# If Tomcat is running, restart it
sudo -u tomcat /opt/tomcat/bin/shutdown.sh
sleep 5
sudo -u tomcat /opt/tomcat/bin/startup.sh

# If Tomcat is not running, just start it
sudo -u tomcat /opt/tomcat/bin/startup.sh

# Wait for deployment
sleep 60

# Check Tomcat logs for any errors
sudo tail -f /opt/tomcat/logs/catalina.out
```

### Verify Deployment
```bash
# Check if all files are in place
ls -la /opt/tomcat/webapps/claims/ClaimLinker.jsp
ls -la /opt/tomcat/webapps/claims/WEB-INF/web.xml
ls -la /opt/tomcat/webapps/claims/WEB-INF/classes/
ls -la /opt/tomcat/webapps/claims/WEB-INF/lib/

# Verify data directory exists in WEB-INF
ls -la /opt/tomcat/webapps/claims/WEB-INF/data/
```

### Test the Web Application
```bash
# Test the main page
curl -I "http://localhost:8080/claims/"

# Test the JSP page
curl -I "http://localhost:8080/claims/ClaimLinker.jsp"

# Test the API endpoint
curl "http://localhost:8080/claims/claimlinker?app=service&text=vaccine%20safety"

# Alternative: Test using the ClaimLinker servlet directly
curl "http://localhost:8080/claims/ClaimLinker?app=service&text=vaccine%20safety"
```

**Note**: The application is accessible at `/claims/` not `/claimlinker/` as the directory name is `claims`.

### Access the Web Interface

Open your browser and navigate to:
- **Main page**: http://localhost:8080/claims/
- **ClaimLinker form**: http://localhost:8080/claims/ClaimLinker.jsp
- **API endpoint**: http://localhost:8080/claims/claimlinker?app=service&text=YOUR_TEXT

### Expected Response Format
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

### Common Deployment Issues

#### Issue 1: Files Not Found
```bash
# If you get 404 errors, verify file locations in the source
find ClaimLinker_web -name "ClaimLinker.jsp"
find ClaimLinker_web -name "index.html"
find ClaimLinker_web -name "web.xml"

# Adjust copy commands based on actual locations
```

#### Issue 2: ClassNotFoundException
```bash
# Ensure all JARs are in WEB-INF/lib
ls -la /opt/tomcat/webapps/claims/WEB-INF/lib/

# Check if classes are compiled
ls -la /opt/tomcat/webapps/claims/WEB-INF/classes/csd/claimlinker/
```

#### Issue 3: Data Files Not Found
```bash
# Verify data files are accessible to the application
ls -la /opt/tomcat/webapps/claims/WEB-INF/data/

# If missing, copy them from the project
sudo cp -r data/* /opt/tomcat/webapps/claims/WEB-INF/data/
sudo chown -R tomcat:tomcat /opt/tomcat/webapps/claims/WEB-INF/data/
```

### Troubleshooting Web Service
```bash
# Check Tomcat logs for errors
sudo tail -100 /opt/tomcat/logs/catalina.out

# Check for any deployment errors
sudo grep -i error /opt/tomcat/logs/catalina.out

# Check application-specific logs
sudo tail -100 /opt/tomcat/logs/localhost.*.log

# Verify Tomcat is running
ps aux | grep tomcat

# Check if port 8080 is listening
sudo netstat -tlnp | grep 8080

# Test Tomcat manager (if available)
curl http://localhost:8080/

# Restart Tomcat if needed
sudo -u tomcat /opt/tomcat/bin/shutdown.sh
sleep 5
sudo -u tomcat /opt/tomcat/bin/startup.sh
```

### Simplified Deployment Script

For easier redeployment, you can create a script:
```bash
# Create deploy script
cat > deploy_claimlinker.sh << 'EOF'
#!/bin/bash

echo "Building ClaimLinker..."
cd /home/methodshub/ClaimLinker
mvn clean package

echo "Stopping Tomcat..."
sudo -u tomcat /opt/tomcat/bin/shutdown.sh
sleep 5

echo "Cleaning old deployment..."
sudo rm -rf /opt/tomcat/webapps/claims/*

echo "Deploying new version..."
sudo mkdir -p /opt/tomcat/webapps/claims

# Copy web files
sudo cp ClaimLinker_web/web/ClaimLinker.jsp /opt/tomcat/webapps/claims/
sudo cp ClaimLinker_web/web/index.html /opt/tomcat/webapps/claims/
sudo cp ClaimLinker_web/web/bookmarklet.js /opt/tomcat/webapps/claims/
sudo cp ClaimLinker_web/web/loading.gif /opt/tomcat/webapps/claims/

# Copy directories
sudo cp -r ClaimLinker_web /opt/tomcat/webapps/claims/
sudo cp ClaimLinker_web/target/ClaimLinker_web-1.0.jar /opt/tomcat/webapps/claims/
sudo cp -r ClaimLinker_web/target/ClaimLinker_web-1.0/WEB-INF /opt/tomcat/webapps/claims/
sudo cp -r ClaimLinker_web/target/ClaimLinker_web-1.0/META-INF /opt/tomcat/webapps/claims/
sudo cp -r ClaimLinker_web/web/resource /opt/tomcat/webapps/claims/

# Set permissions
sudo chown -R tomcat:tomcat /opt/tomcat/webapps/claims

echo "Starting Tomcat..."
sudo -u tomcat /opt/tomcat/bin/startup.sh

echo "Deployment complete! Waiting for startup..."
sleep 60

echo "Testing deployment..."
curl -I "http://localhost:8080/claims/"

echo "Done!"
EOF

chmod +x deploy_claimlinker.sh
```

Run the script with:
```bash
./deploy_claimlinker.sh
```
## Making Data Persistent

 

The default setup loses data when containers restart. To make data persistent:

 

### Create Persistent Elasticsearch

Section to be added soon 
