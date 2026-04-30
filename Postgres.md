# Running a Quarkus Maven Project with PostgreSQL (Liquibase)

This document explains how to run a **Quarkus Maven project** locally with a **PostgreSQL database** managed via **Docker Compose**, and how to connect to it using **DBeaver**.

It is especially useful when the application uses **Liquibase**, which requires a running database at startup.

---

## Prerequisites

Make sure you have the following installed and running:

* Java (compatible with your Quarkus version)
* Maven
* Docker & Docker Compose
* IntelliJ IDEA
* DBeaver (or any PostgreSQL client)

---

## 1. Create an IntelliJ Run Configuration for the API

Because the **API module** contains the controllers and is the main entry point, the run configuration must point to it directly.

### Steps

1. Open **Run / Edit Configurations**
2. Create a **new Maven configuration**
3. Set:

   * **Working directory**:

     ```
     <project-root>/application/api
     ```
   * **Goals**:

     ```
     clean compile quarkus:dev
     ```

This configuration will start the Quarkus application in **dev mode**.

---

## 2. Database Requirement (Liquibase)

Since the application uses **Liquibase**, PostgreSQL **must be running before the application starts**.

If PostgreSQL is not available, Quarkus will fail at startup.

---

## 3. Running PostgreSQL (Two Possible Ways)

### Option 1 — Run PostgreSQL via Terminal (Docker Compose)

1. Open a terminal
2. Navigate to the folder containing:

   ```
   docker-compose.dev-env.yml
   ```
3. Run:

   ```bash
   docker compose -f docker-compose.dev-env.yml up -d
   ```

This will start PostgreSQL with all required configuration.

---

### Option 2 — Run PostgreSQL via IntelliJ (Recommended)

This allows PostgreSQL to start **automatically before the API**.

#### Create Docker Compose Configuration

1. Open **Run / Edit Configurations**
2. Add a **Docker Compose** configuration
3. Configure:

   * **Name**: e.g. `postgres-tenant`
   * **Server**: `Docker`
   * **Compose file**: path to `docker-compose.dev-env.yml`
   * **Services**:

     ```yaml
     postgres_service,
     ```

     ⚠️ The service name must match the one defined in `docker-compose.yml`

Example:

```yaml
services:
  postgres_service:
    image: postgres:15
```

---

#### Link PostgreSQL to the API Configuration

1. Open the **API Maven configuration**
2. In **Before launch**, click `+`
3. Select **Run another configuration**
4. Choose the **Docker Compose (PostgreSQL)** configuration

✅ Result: PostgreSQL will always start **before** the API.

---

## 4. Important Port Warning ⚠️

PostgreSQL usually runs on **port 5432**.

If you:

* Run another project
* With another PostgreSQL container
* Using the same port

You will get a **port already in use** error.

### Best Practice

➡️ After finishing a project, **stop the PostgreSQL container**:

```bash
docker stop <container_name>
```

or stop it directly from IntelliJ.

---

## 5. Connecting to PostgreSQL with DBeaver

### Create a New Connection

1. Open **DBeaver**
2. Click **New Connection**
3. Select **PostgreSQL**
4. Fill in:

   * **Connection name**: project name (e.g. `tenant`, `identity`)
   * **Host**: `localhost`
   * **Port**: `5432`
   * **Database**: value from Docker Compose
   * **Username / Password**: from Docker Compose

Example from `docker-compose.yml`:

```yaml
environment:
  POSTGRES_DB: identity
  POSTGRES_USER: postgres
  POSTGRES_PASSWORD: root
volumes:
  - ./infrastructure/src/main/resources/db/init/:/docker-entrypoint-initdb.d/
  - postgres_data:/var/lib/postgresql/data
```

5. Click **Test Connection**
6. Save the connection

At this stage, the database may be **empty** — this is normal.

---

## 6. Start the Application

1. Run the **API Maven configuration**
2. Quarkus starts
3. Liquibase runs automatically
4. Database schema and tables are created

---

## 7. Verify in DBeaver

1. Refresh the database in DBeaver
2. You should now see:

   * Tables
   * Schemas
   * Liquibase metadata tables

✅ The application is now fully connected to PostgreSQL.

---

## Summary

* The API must be run from the **application/api** folder
* Liquibase requires PostgreSQL to be running first
* Docker Compose is the recommended way to manage PostgreSQL
* IntelliJ can automatically start PostgreSQL before the API
* DBeaver is used to inspect and verify the database
* Always stop PostgreSQL containers to avoid port conflicts
