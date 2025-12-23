# 🛍️ Catalogue Service

> A Node.js microservice providing the product catalogue REST API for the RoboShop e-commerce platform

[![Version](https://img.shields.io/badge/version-3.0.0-blue.svg)](package.json)
[![Node.js](https://img.shields.io/badge/node.js-runtime-green.svg)](https://nodejs.org/)
[![MongoDB](https://img.shields.io/badge/database-MongoDB-brightgreen.svg)](https://www.mongodb.com/)

---

## 📋 Table of Contents

- [Overview](#overview)
- [CI/CD Pipeline](#cicd-pipeline)
- [Service Architecture](#service-architecture)
- [API Endpoints](#api-endpoints)
- [Deployment](#deployment)
- [Development](#development)
- [Troubleshooting](#troubleshooting)

---

## 🎯 Overview

The **Catalogue Service** is a critical microservice in the RoboShop platform that manages product information, categories, and search functionality. This repository contains both the service source code and the Jenkins CI/CD pipeline configuration for automated build, test, and deployment.

### Key Responsibilities

- **Product Management**: Store and retrieve product information
- **Category Organization**: Manage product categories and hierarchies
- **Search Functionality**: Full-text search across products
- **Health Monitoring**: Provide health check endpoints for orchestration

---

## 🔄 CI/CD Pipeline

### Why This Pipeline Exists

The Jenkins pipeline in this repository automates the entire software delivery lifecycle for the Catalogue service, ensuring:

- ✅ **Consistency**: Every build follows the same process
- ✅ **Quality**: Automated testing and code analysis gates
- ✅ **Speed**: Rapid feedback and deployment cycles
- ✅ **Traceability**: Version-controlled artifacts linked to source code
- ✅ **Reliability**: Automated deployment reduces human error

### Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        CI PIPELINE FLOW                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐  │
│  │ Get Version  │────▶│   Install    │────▶│  Unit Test   │  │
│  │ package.json │     │ Dependencies │     │              │  │
│  └──────────────┘     └──────────────┘     └──────────────┘  │
│         │                                          │           │
│         ▼                                          ▼           │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐  │
│  │  Sonar Scan  │────▶│    Build     │────▶│     SAST     │  │
│  │ Code Quality │     │ catalogue.zip│     │   Security   │  │
│  └──────────────┘     └──────────────┘     └──────────────┘  │
│                               │                                │
│                               ▼                                │
│                    ┌──────────────────┐                       │
│                    │     Publish      │                       │
│                    │   to Nexus       │                       │
│                    │ (Artifact Repo)  │                       │
│                    └──────────────────┘                       │
│                               │                                │
│                               ▼                                │
│                    ┌──────────────────┐                       │
│                    │  Trigger Deploy  │                       │
│                    │ (Separate Job)   │                       │
│                    └──────────────────┘                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Pipeline Stages Explained

#### 1. **Get Version** 📦
- Reads version from `package.json`
- Stores in `packageVersion` variable for use throughout pipeline
- Ensures artifact versioning matches source code

#### 2. **Install Dependencies** 📚
- Runs `npm install` to fetch Node.js dependencies
- Prepares environment for testing and building
- Uses npm's package-lock.json for reproducible builds

#### 3. **Unit Test** 🧪
- Placeholder for automated unit tests
- Future: Will execute `npm test` and fail pipeline on test failures
- Ensures code quality before proceeding

#### 4. **Sonar Scan** 🔍
- Placeholder for SonarQube code quality analysis
- Future: Will run `sonar-scanner` to detect bugs, vulnerabilities, code smells
- Enforces coding standards and security best practices

#### 5. **Build** 🏗️
- Creates `catalogue.zip` artifact containing all service files
- Excludes `.git` directory and existing zip files
- Results in deployable artifact with all necessary code

#### 6. **SAST (Static Application Security Testing)** 🛡️
- Placeholder for security vulnerability scanning
- Future: Will scan code for known security issues
- Catches security problems before deployment

#### 7. **Publish Artifact** 📤
- Uploads `catalogue.zip` to Nexus Repository Manager
- Tags with version from `package.json`
- Creates immutable, versioned artifact for deployment
- **Nexus Coordinates**: `com.roboshop:catalogue:${version}:zip`

#### 8. **Deploy** 🚀
- Triggers downstream deployment job (`catalogue-deploy`)
- Passes version number to deployment pipeline
- Waits for deployment confirmation before completing
- Ensures end-to-end automation from code to production

---

## 🏛️ Service Architecture

### Technology Stack

- **Runtime**: Node.js
- **Framework**: Express.js
- **Database**: MongoDB
- **Logging**: Pino logger
- **Monitoring**: Health check endpoints

### Configuration

#### Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `CATALOGUE_SERVER_PORT` | No | `8080` | HTTP server port |
| `MONGO` | Yes | - | Must be `true` to enable MongoDB |
| `MONGO_URL` | Yes | `mongodb://mongodb:27017/catalogue` | MongoDB connection string |
| `DOCUMENTDB` | No | - | Set to `true` for AWS DocumentDB |
| `GO_SLOW` | No | `0` | Artificial delay in ms for `/product/:sku` |

### Health Check

```bash
GET /health
```

**Response:**
```json
{
  "app": "OK-2",
  "mongo": true
}
```

- `app`: Application status
- `mongo`: MongoDB connection status

---

## 🔌 API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/health` | Health check for monitoring |
| `GET` | `/products` | Get all products |
| `GET` | `/product/:sku` | Get product by SKU |
| `GET` | `/products/:cat` | Get products by category |
| `GET` | `/categories` | Get all categories |
| `GET` | `/search/:text` | Full-text search products |

---

## 🚀 Deployment

### Jenkins Agent Requirements

The build agent (`AGENT-1`) must have:

- ✅ Node.js and npm installed
- ✅ `zip` utility for creating artifacts
- ✅ Jenkins plugins:
  - **Pipeline Utility Steps** (for `readJSON`)
  - **Nexus Artifact Uploader**

### Nexus Configuration

```yaml
Server: http://172.31.1.199:8081
Repository: catalogue
Group ID: com.roboshop
Artifact ID: catalogue
Type: zip
Credentials: nexus-auth (configured in Jenkins)
```

### SystemD Service (VM Deployment)

The service includes a systemd unit file for traditional VM deployments:

```ini
[Service]
User=roboshop
Environment=MONGO=true
Environment=MONGO_URL="mongodb://MONGO_DNSNAME:27017/catalogue"
ExecStart=/bin/node /home/roboshop/catalogue/server.js
```

---

## 💻 Development

### Local Setup

1. **Clone the repository**
   ```bash
   git clone <repository-url>
   cd catalogue
   ```

2. **Install dependencies**
   ```bash
   npm install
   ```

3. **Set environment variables**
   ```bash
   export MONGO=true
   export MONGO_URL="mongodb://localhost:27017/catalogue"
   export CATALOGUE_SERVER_PORT=8080
   ```

4. **Run the service**
   ```bash
   node server.js
   ```

### Release Process

To publish a new version:

1. Update version in `package.json`
2. Commit and push changes
3. Jenkins pipeline automatically:
   - Builds the new version
   - Publishes to Nexus
   - Triggers deployment
4. Verify artifact in Nexus: `com.roboshop:catalogue:<new-version>`

---

## 🔧 Troubleshooting

### Common Issues

#### Pipeline Fails at "Get Version"
**Cause**: Jenkins missing Pipeline Utility Steps plugin  
**Solution**: Install plugin from Jenkins Plugin Manager

#### Pipeline Fails at Build Stage
**Cause**: `zip` command not found on agent  
**Solution**: Install zip utility: `sudo apt-get install zip`

#### Nexus Upload Fails
**Causes**:
- Invalid/missing `nexus-auth` credentials
- Nexus server unreachable
- Repository name mismatch

**Solution**: Verify credentials and network connectivity

#### Health Check Shows `mongo: false`
**Causes**:
- MongoDB not running or unreachable
- Missing `MONGO=true` environment variable
- Incorrect `MONGO_URL`

**Solution**: Verify MongoDB connection and environment configuration

#### Service Won't Start
**Check**:
```bash
# View service logs
journalctl -u catalogue -f

# Check if port is in use
netstat -tulpn | grep 8080
```

---

## 📊 Pipeline Configuration Details

### Jenkinsfile Highlights

```groovy
environment {
    packageVersion = ''  // Global variable for version tracking
}

// Version extracted from package.json
def packageJson = readJSON(file: 'package.json')
packageVersion = packageJson.version

// Artifact coordinates
groupId: 'com.roboshop'
artifactId: 'catalogue'
version: "${packageVersion}"
repository: 'catalogue'

// Downstream deployment trigger
build job: "../catalogue-deploy", 
      wait: true, 
      parameters: [string(name: 'version', value: "$packageVersion")]
```
