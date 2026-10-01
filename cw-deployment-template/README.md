# CW Deployment Template Setup Guide

This deployment template was originally created by copying the `service-template` folder from the parent directory (`cp -r ../service-template ./cw-deployment-template`). It has been specifically customized to run Metaphactory with an external Jena Fuseki database and the Ontopic module. 

This guide consolidates all the setup steps, fixes, and configurations required to recreate this environment from scratch.

## 1. Initial Setup & Environment

**Location**: Run these commands on your host machine.

1. **Clone the deployment template**: 
   ```bash
   cp -r ../service-template ./cw-deployment-template
   cd cw-deployment-template
   ```
2. **Configure `.env`**: Modify the `.env` file to set `COMPOSE_PROJECT_NAME` and append the Ontopic compose files to `COMPOSE_FILE` (before the trailing `docker-compose.overwrite.yml`). 
   * **Note 1:** Ensure your `COMPOSE_FILE` uses **relative paths** (e.g., `../ontopic/docker-compose.yml`) rather than absolute paths, otherwise the stack will fail to boot on other team members' machines.
   * **Note 2:** Ontopic and Metaphactory are designed as a single, unified stack sharing the same networks. **Do not** attempt to start them separately (e.g. by `cd`-ing into the `ontopic` folder). They must always be started together from this deployment folder.

## 2. Install & Run  postgres db

      ```brew install postgresql@18```

      ``` '/usr/local/Cellar/postgresql@18/18.6/bin/pg_ctl' -D '/usr/local/var/postgresql@18' -l logfile start```

## 3. Ontopic Module Preparation

**Location**: Run these commands on your host machine inside the `cw-deployment-template` directory.

To successfully boot the Ontopic mapping/virtualization containers, several prerequisites must be fulfilled before running docker-compose:

### Missing Secret Mounts
Docker Compose fails if bind-mounted files don't exist on the host. Create the required secret placeholders in `cw-deployment-template`:
```bash
mkdir -p secrets/azure-blob-storage secrets/ai secrets/s3 secrets/store
touch secrets/azure-blob-storage/account-key \
      secrets/azure-blob-storage/account-name \
      secrets/azure-blob-storage/sas-token \
      secrets/ai/llm-additional-headers \
      secrets/ai/openai-key \
      secrets/ai/anthropic-key \
      secrets/s3/access-key-id \
      secrets/s3/access-key-secret
```

### Database Password Formulation
The isolated PostgreSQL container (`store-server-db`) requires a password. **Crucially, this password must not contain a trailing newline**, otherwise the Node.js backend authentication will fail while Postgres succeeds:
```bash
printf "secret" > secrets/store/db-password
```

### JDBC Drivers
The `ontopic-server` requires JDBC drivers mapped into the `./jdbc` (to be created in the `cw-deployment-template` folder) volume to connect to databases (including its fallback H2 database):
```bash
mkdir -p jdbc
# Download the drivers into the jdbc folder
wget -P ./jdbc https://repo1.maven.org/maven2/com/h2database/h2/2.2.224/h2-2.2.224.jar
wget -P ./jdbc https://jdbc.postgresql.org/download/postgresql-42.7.2.jar
```

## 4. Starting the Stack

**Location**: Run on your host machine inside the `cw-deployment-template` directory.

Once all configuration steps and prerequisites are met, bring up the stack:
```bash
docker compose up -d
```
Access Metaphactory at `http://localhost:10214`.


*Note: If containers boot up without joining all networks properly, force recreate them on the host machine using: `docker compose up --force-recreate -d store-server`.*
---

## 5. Metaphactory Repository Setup 

**Location**: Modify the `default.ttl` file on your host machine (if mapped) or inside the Metaphactory container.

By default, Metaphactory's `default.ttl` repository configuration might be misconfigured for external SPARQL endpoints like Jena Fuseki. We must apply two major fixes.

### Authentication and Update Endpoint
To allow Metaphactory to write to Fuseki (e.g., when uploading an ontology in the UI), it requires an explicit `config:sparql.updateEndpoint` and proper Basic Auth configuration. 

**Note for Team Members:** The `default.ttl` file being read by the system lives **inside the running container** at `/runtime-data/config/repositories/default.ttl`. To safely edit it, use Docker to copy it out and back in:

```bash
# 1. Copy it out to your host (replace container name if needed)
docker cp cw-deployment-1-metaphactory:/runtime-data/config/repositories/default.ttl ./default.ttl

# 2. Modify it locally to use SPARQLBasicAuthRepository and add the updateEndpoint:
```
```turtle
config:rep.impl [
      config:rep.type "metaphactory:SPARQLBasicAuthRepository";
      config:sparql.queryEndpoint <http://host.docker.internal:3030/process-graph/sparql>;
      config:sparql.updateEndpoint <http://host.docker.internal:3030/process-graph/update>;
      mph:username "admin";
      mph:password "your-password";
      mph:quadMode true
]
```
```bash
# 3. Copy it back and restart the container
docker cp ./default.ttl cw-deployment-1-metaphactory:/runtime-data/config/repositories/default.ttl
docker restart cw-deployment-1-metaphactory
```

## 6. Independent Setup: Jena Fuseki Configuration

**Important Note:** The Jena Fuseki database runs **independently** from this Docker Compose stack. It runs directly on the host machine in a separate Docker container on port `3030`.

### Starting Fuseki and Creating the Database

To start Jena Fuseki instance, run the following command on host machine:
```bash
docker run -d --name jena-fuseki -p 3030:3030 stain/jena-fuseki
```

Once the container is up and running, create the database. Open the Fuseki web interface at `http://localhost:3030` and create a **permanent TDB2 database** named `process-graph`. This step creates the dataset configuration file that we will modify below.

### Fixing Ontology Visibility (Named Graphs vs Default Graph)

When ontologies are uploaded via Metaphactory, they are stored in **Named Graphs** in Fuseki. However, Metaphactory's Ontology Catalog uses standard SPARQL queries (which only search the **Default Graph**) to discover them. This results in the UI showing "0 ontologies" despite successful uploads.

To fix this, you must enable **Union Default Graph** in Jena Fuseki container.

**Location**: Run these commands on your host machine to modify the Fuseki container.

1. Copy out the dataset configuration file from your running Fuseki container to your host machine (assuming the container name is `jena-fuseki`):
   ```bash
   docker cp jena-fuseki:/fuseki/configuration/process-graph.ttl ./process-graph.ttl
   ```
2. Edit `process-graph.ttl` on your host machine to add `tdb2:unionDefaultGraph true ;` inside the `:tdb_dataset_readwrite` block:
   ```turtle
   :tdb_dataset_readwrite
           rdf:type       tdb2:DatasetTDB2;
           tdb2:unionDefaultGraph true ;
           tdb2:location  "/fuseki/databases/process-graph" .
   ```
3. Copy the modified file back to the Fuseki container and restart it:
   ```bash
   docker cp ./process-graph.ttl jena-fuseki:/fuseki/configuration/process-graph.ttl
   docker restart jena-fuseki
   ```
This setting forces Fuseki to treat the union of all named graphs as the default graph, making all uploaded ontologies instantly visible in the Metaphactory UI.

---

## 7. Optional: Tutorial Setup (Destination Database)

**Location**: Modify `docker-compose.overwrite.yml` inside the `cw-deployment-template` directory on your host machine.

For certain tutorials, the system requires access to a local "Destination database". To run this local instance, we add the `destination-tutorial-db` container and connect it to `ontopic-server` via a shared `databases` network. 

This configuration has been added to your `docker-compose.overwrite.yml`:
```yaml
services:
  # ... existing services ...

  destination-tutorial-db:
    image: ontopicvkg/destination-tutorial-db
    shm_size: 1g
    restart: unless-stopped
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres2
    networks:
      - databases

  ontopic-server:
    networks:
      - databases

networks:
  databases:
```

This ensures the Ontopic mapping/virtualization engine can reach the tutorial database to fetch data. Once you run `docker compose up -d`, this database will automatically provision and connect alongside the rest of the stack.
