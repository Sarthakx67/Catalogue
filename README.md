## 06-catalogue (Catalogue service)

This repository contains the **Catalogue** Node.js service code and its **CI pipeline** that packages the service and publishes it to **Nexus**.

### Repo responsibility (DevOps view)
- Build a versioned artifact (`catalogue.zip`) from the source.
- Publish the artifact to Nexus (`com.roboshop:catalogue:<version>`).
- Trigger the deployment job (separate repo/job) with the same `version`.

### Service runtime contract (what ops needs to know)
- **Port:** `CATALOGUE_SERVER_PORT` (default `8080`) ([server.js](server.js))
- **Healthcheck:** `GET /health` returns `{ app: "OK-2", mongo: <bool> }` ([server.js](server.js))
- **MongoDB is required:** the app defines `mongoConnect()` only when `MONGO=true` (or `DOCUMENTDB=true`). Standard runtime is **Mongo**.

#### Required environment variables (Mongo)
- `MONGO=true`
- `MONGO_URL=mongodb://<mongo-host>:27017/catalogue`
- Optional: `CATALOGUE_SERVER_PORT=8080`
- Optional: `GO_SLOW=<milliseconds>` (adds artificial delay for `GET /product/:sku`)

#### systemd unit
A systemd unit is provided for VM-based installs ([systemd.service](systemd.service)):
- Runs as `roboshop`
- Sets `MONGO=true` and `MONGO_URL=...`
- Starts `node /home/roboshop/catalogue/server.js`

### CI/CD (Jenkins) — build + publish + trigger deploy
Pipeline: [Jenkinsfile](Jenkinsfile)

#### Jenkins agent requirements
The Jenkins node labeled `AGENT-1` must have:
- `node` + `npm`
- `zip` (used to create `catalogue.zip`)
- Jenkins plugin support for:
	- `readJSON` (Pipeline Utility Steps)
	- `nexusArtifactUploader` (Nexus Artifact Uploader)

#### Pipeline stages (as implemented)
- **Get version**: reads `version` from [package.json](package.json) and stores it in `packageVersion`
- **Install dependencies**: `npm install`
- **Unit test / Sonar Scan / SAST**: placeholders (currently `echo` only)
- **Build**: creates `catalogue.zip` from the workspace (excludes `.git` and existing `.zip`)
- **Publish Artifact**: uploads `catalogue.zip` to Nexus
- **Deploy**: triggers downstream job `../catalogue-deploy` and passes `version=<packageVersion>`

### Nexus publishing (implementation details)
Configured in [Jenkinsfile](Jenkinsfile) `Publish Artifact` stage:

- **Nexus version:** `nexus3`
- **Protocol:** `http`
- **Nexus URL:** `172.31.1.199:8081`
- **Repository:** `catalogue`
- **Coordinates:**
	- `groupId`: `com.roboshop`
	- `artifactId`: `catalogue`
	- `version`: taken from [package.json](package.json) (`packageVersion`)
	- `type`: `zip`
- **Uploaded file name:** `catalogue.zip`
- **Jenkins credentials:** `credentialsId: nexus-auth`

Operationally, the deploy pipeline consumes the **version string** and the configuration tooling (via Ansible roles in the deploy repo) is expected to download the matching artifact from Nexus.

### How to publish a new version (release flow)
1. Update `version` in [package.json](package.json).
2. Run the Jenkins job for this repo.
3. Validate that Nexus contains `com.roboshop:catalogue:<version>` (type `zip`).
4. Confirm the downstream deploy job ran with the same `version` parameter.

### Troubleshooting (common)
- **Pipeline fails at “Get version”**: Jenkins missing `readJSON` support (Pipeline Utility Steps plugin).
- **Pipeline fails at zip stage**: `zip` not installed on `AGENT-1`.
- **Nexus upload fails**: `nexus-auth` missing/invalid, Nexus not reachable, or repository name mismatch.
- **/health shows `mongo:false`**: Mongo unreachable or `MONGO/MONGO_URL` not set correctly.
