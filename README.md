# SonarQube with PostgreSQL + Community Branch Plugin (Docker Compose)

This repository provides a persistent Docker Compose setup to run **SonarQube Community Edition** with the **Community Branch Plugin** and a dedicated **PostgreSQL** database.

This configuration extends the standard SonarQube Community Edition by adding multi-branch analysis support through the [SonarQube Community Branch Plugin](https://github.com/mc1arke/sonarqube-community-branch-plugin).

## Features

* **Multi-Branch Support:** Includes the Community Branch Plugin, enabling branch and pull request analysis in the free Community Edition.
* **Stable Database:** Uses a dedicated PostgreSQL 15 database, as recommended by SonarSource for persistent data storage.
* **Persistent Storage:** All data (database, SonarQube data, extensions, logs) is stored in named Docker volumes, so your data survives container restarts and image updates.
* **Isolated Networking:** Both services run on a custom bridge network, allowing them to communicate securely by name.
* **Ready-to-Use:** Just run `docker-compose up` and start analyzing your branches!

## What is the Community Branch Plugin?

The [SonarQube Community Branch Plugin](https://github.com/mc1arke/sonarqube-community-branch-plugin) is an open-source extension that adds branch and pull request analysis capabilities to SonarQube Community Edition (which usually only supports the `master`/`main` branch).

**Key capabilities:**
- Analyze multiple branches in a single project
- View pull request analysis and decoration
- Compare branches with the main branch
- Track code quality across different branches

## Prerequisites

* [Docker](https://docs.docker.com/get-docker/)
* [Docker Compose](https://docs.docker.com/compose/install/)

---

## How to Use

1.  **Download:**
    Clone this repository or download the `docker-compose.yml` file into a new directory (for example, `~/docker/sonarqube-branch`).

2.  **Start the Services:**
    From inside that directory, run:
    ```bash
    docker-compose up -d
    ```
    This will pull the images (if you don't have them) and start the containers.

3.  **Access SonarQube:**
    Wait 1-2 minutes for the server to initialize. You can check the logs with:
    ```bash
    docker-compose logs -f sonarqube
    ```

    Once it's running, access the dashboard in your browser at:
    **[http://localhost:9000](http://localhost:9000)**

4.  **Login:**
    * **Default Username:** `admin`
    * **Default Password:** `admin`
    
    (You will be prompted to change the password on first login).

That's it! Your SonarQube instance with branch analysis support is ready to use.

---

## Customization & Versioning

This `docker-compose.yml` is designed to be a flexible template. Here is how you can customize it.

### 1. `sonarqube-db` (PostgreSQL Service)

* **`image: postgres:15-alpine`**
    This configuration uses PostgreSQL 15 (Alpine version), which is lightweight and stable. You can change this to other versions if needed.
    * **Example:** `image: postgres:16-alpine`

* **`environment:`**
    You can change the `POSTGRES_USER`, `POSTGRES_PASSWORD`, or `POSTGRES_DB` values.
    **If you change these**, you **must** update the `SONAR_JDBC_...` variables in the `sonarqube` service to match.

* **`ports: - "5433:5432"`**
    This line is **optional**. It exposes the database to your host PC on port `5433`. This is only useful if you want to connect to the database with an external tool (like DBeaver or pgAdmin).

### 2. `sonarqube` (SonarQube Application)

* **`image: mc1arke/sonarqube-with-community-branch-plugin:25.9.0.112764-community`**
    This is the most important setting. This image comes **pre-bundled** with the Community Branch Plugin. 
    
    **Important:** Do NOT use the official `sonarqube:community` image, or you will lose branch support.
    
    **Available options:**
    * **Check the [mc1arke Docker Hub tags](https://hub.docker.com/r/mc1arke/sonarqube-with-community-branch-plugin/tags)** to find newer versions of SonarQube packaged with the plugin.

* **`ports: - "9000:9000"`**
    If you already have a service on your host using port `9000`, you can change the *host* port (the left side).
    * **Example:** `ports: - "9001:9000"` will make SonarQube available at `http://localhost:9001`.

---

## Configuring Branch Analysis

Once your SonarQube instance is running, you can analyze branches and pull requests using your preferred scanner. Here are examples for common scenarios:

### Branch Analysis

**Maven:**
```bash
mvn sonar:sonar \
  -Dsonar.projectKey=my-project \
  -Dsonar.host.url=http://localhost:9000 \
  -Dsonar.token=YOUR_TOKEN \
  -Dsonar.branch.name=feature/my-branch
```

**Gradle:**
```bash
./gradlew sonar \
  -Dsonar.projectKey=my-project \
  -Dsonar.host.url=http://localhost:9000 \
  -Dsonar.token=YOUR_TOKEN \
  -Dsonar.branch.name=feature/my-branch
```

### Pull Request Analysis

**Maven:**
```bash
mvn sonar:sonar \
  -Dsonar.projectKey=my-project \
  -Dsonar.host.url=http://localhost:9000 \
  -Dsonar.token=YOUR_TOKEN \
  -Dsonar.pullrequest.key=123 \
  -Dsonar.pullrequest.branch=feature/my-branch \
  -Dsonar.pullrequest.base=main
```

**Gradle:**
```bash
./gradlew sonar \
  -Dsonar.projectKey=my-project \
  -Dsonar.host.url=http://localhost:9000 \
  -Dsonar.token=YOUR_TOKEN \
  -Dsonar.pullrequest.key=123 \
  -Dsonar.pullrequest.branch=feature/my-branch \
  -Dsonar.pullrequest.base=main
```

### CI/CD Integration

**GitHub Actions Example:**
```yaml
- name: SonarQube Scan
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
    SONAR_HOST_URL: http://localhost:9000
  run: |
    mvn sonar:sonar \
      -Dsonar.projectKey=my-project \
      -Dsonar.branch.name=${{ github.ref_name }}
```

For detailed configuration instructions for different CI/CD platforms, refer to the [plugin documentation](https://github.com/mc1arke/sonarqube-community-branch-plugin#usage).

---

## Updating Your Stack

Because this setup uses a custom image, you update by modifying the version tag in `docker-compose.yml`:

```bash
# 1. Stop the services
docker-compose down

# 2. Edit docker-compose.yml and update the image tag to a newer version from mc1arke
# For example: mc1arke/...:25.9.0... → mc1arke/...:26.0.0...

# 3. Pull the new images
docker-compose pull

# 4. Start the services with new images
docker-compose up -d
```

The new containers will start and connect to your existing data volumes.

---

## Troubleshooting

### "The version of SonarQube you are trying to upgrade from is too old"

This error can happen when jumping between major versions. SonarQube has upgrade path limitations and cannot always jump from very old versions (like 9.x) directly to newer ones (like 11.x+) without intermediate steps.

**Fix (This will delete all your SonarQube analysis data):**
```bash
# Stop and remove all containers AND volumes
docker-compose down -v

# Start fresh with the new version
docker-compose up -d
```

### Branch Analysis Not Working

If branch analysis features are not available, check if you are using the correct image.

```bash
docker-compose images
```

Make sure the image listed for SonarQube is `mc1arke/sonarqube-with-community-branch-plugin`, NOT the official `sonarqube` image.

### Container Fails to Start

Check the logs to identify the issue:
```bash
docker-compose logs -f sonarqube
docker-compose logs -f sonarqube-db
```

Common issues:
- **Insufficient memory:** SonarQube requires at least 2GB of RAM
- **Port conflicts:** Ensure ports 9000 and 5433 are not in use by other services
- **Database connection:** Verify PostgreSQL is running and credentials match between the two services

---

## Useful Links & Documentation

### SonarQube Community Branch Plugin
* **Plugin GitHub Repository:** [github.com/mc1arke/sonarqube-community-branch-plugin](https://github.com/mc1arke/sonarqube-community-branch-plugin)
* **Pre-built Docker Image (Alternative):** [hub.docker.com/r/mc1arke/sonarqube-with-community-branch-plugin](https://hub.docker.com/r/mc1arke/sonarqube-with-community-branch-plugin)
* **Plugin Usage Guide:** [github.com/mc1arke/sonarqube-community-branch-plugin#usage](https://github.com/mc1arke/sonarqube-community-branch-plugin#usage)

### SonarQube Official Resources
* **SonarQube Downloads:** [sonarsource.com/products/sonarqube/downloads/](https://www.sonarsource.com/products/sonarqube/downloads/)
* **SonarQube Docker Installation Guide:** [docs.sonarsource.com/sonarqube-server/latest/server-installation/from-docker-image](https://docs.sonarsource.com/sonarqube-server/latest/server-installation/from-docker-image/)

### Database & Infrastructure
* **PostgreSQL on Docker Hub:** [hub.docker.com/_/postgres](https://hub.docker.com/_/postgres)
* **Docker Compose File Reference:** [docs.docker.com/compose/compose-file/](https://docs.docker.com/compose/compose-file/)

---

## Differences from Standard SonarQube Setup

This repository differs from the standard [SonarQube-with-PostgreSQL](https://github.com/MathCunha16/SonarQube-with-PostgreSQL) setup in the following ways:

| Feature | Standard Setup | This Setup (with Branch Plugin) |
|---------|---------------|----------------------------------|
| **Branch Analysis** | ❌ Not supported | ✅ Supported |
| **PR Analysis** | ❌ Not supported | ✅ Supported |
| **Best For** | Single branch analysis | Multi-branch projects, CI/CD pipelines |

**When to use each:**
- **Use the [standard setup](https://github.com/MathCunha16/SonarQube-with-PostgreSQL)** if you only need to analyze a single branch (typically `main` or `master`)
- **Use this setup** if you need to analyze multiple branches, compare them, or analyze pull requests

---

## License

This project is licensed under the **MIT License**.

The SonarQube Community Branch Plugin is licensed under the **LGPL-3.0 License**. See the [plugin repository](https://github.com/mc1arke/sonarqube-community-branch-plugin/blob/master/LICENSE) for details.
