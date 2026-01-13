# Getting Started

<!-- Sealed Intelligence can be run in two different modes, depending on your needs:

- **Portable Mode (Windows)**: Ideal for quickly testing Sealed Intelligence on your local machine without setting up any external dependencies. No installation is required—just download the bundle, and launch the app.

- **Docker Mode**: Suitable for both advanced local testing and production deployments. This mode gives you full access to all features, but requires Docker, a PostgreSQL database, and some additional configuration.


## Portable Mode

### Launch
- Download the latest portable version <a href="https://github.com/sealed-intelligence/sealed-intelligence/releases/latest" target="_blank">here</a>.
- Unzip the file.
- Click on `launch_sealed_intelligence.bat` to run Sealed Intelligence.
- Browse `http://127.0.0.1:5000` to access the app.

### Configuration

At the login screen, click on the Sign up button to create an account. After logging in, you can access the admin panel by clicking on the gear icon at the top-right of the app and selecting "Admin Panel".

- **Add a Database Connection**. You can either manually add a connection or sync with your BI tool.
    - To manually add a database connection, click on 'Add Database Connection'.
    - To connect your BI tool, see [here](./bi_integration.md).
- **Managing Access to Database Tables/Views**
    - Click on [DB Name] > Manage Permissions
    - Select tables/views that you want to query on.
    !!! note
        It is recommended to use views instead of tables for Sealed Intelligence, as they provide greater flexibility. In the views:

        - Include only the columns that are relevant for answering user queries. The fewer irrelevant columns you have, the better the response quality will be.
        - Filter out rows that are redundant.
        - Perform renaming, type conversion, or any other transformations that can help make the data cleaner.

- **Managing Semantic Layer and Metadata**
    - You can provide business metrics, domain-knowledge and descriptions at database, table and column-level (Admin Panel > Database Connections > [DB Name] > Semantic Layer and Metadata Management).
    - It’s recommended to add descriptions only when it's actually helpful. For example, the column sale_date does not need a description, but a column like abc23 would. 
    - The `db schema` command shows you what metadata is shared with the LLM when you ask a question.


## Docker Mode -->
## Deployment (Test)

Below we provide instructions for deploying Sealed Intelligence on a VM with public IP over HTTP for testing. For **production deployment**, please see [here](./production_deployment.md).

### Step 1: Launch a Linux server with a public IP 
For serving the LLM (gpt-oss-20b) the VM should have a GPU with at least 24GB VRAM. NVIDIA L4 GPUs are a good choice for LLM inference for testing (and light production).

=== "AWS"

    - Launch new EC2 instance
    - Select ubuntu as OS type
    - For Amazon Machine Image (AMI), choose *Deep Learning Base AMI with Single CUDA* as it has the required libraries and packages already installed
    - For instance type, select g6.xlarge
    - For storage, allow 100 GB
    - For security group
        - Allow inbound **TCP 5000**

=== "Google Cloud"

    - Go to Compute Engine → VM instances → Create instance
    - "Machine Configuration" tab
        - Select "GPUs"
        - GPU Type: L4
        - Machine Type: n1-standard-4
    - "OS and storage" tab
        - Click 'Change' button in the middle of page
        - Operating system: Deep Learning on Linux
        - Version: Deep Learning VM with CUDA + Pytorch M131 
        - Size: 100
    - Create
    - After Creation:
        - Click on the instance, in Network interface click 'View details' under 'Network details'.
        - Create VPC firewall rule
        - Allow inbound **TCP 5000**

### Step 2: Create a PostgreSQL DB
This will be used as the internal database for the app. There is no need to add any tables; Sealed Intelligence will create those on startup. Note down the following info: **host name**, **database name**, **port**, **username** and **password**.  
In the DB security group allow **5432/tcp** from the VM to database host

### Step 3: Create a directory for Sealed Intelligence
SSH into the server and run the following command:
```
mkdir sealed-intelligence
cd sealed-intelligence
```

### Step 4: Check if Docker is already installed
```sh
docker --version
```
If Docker is installed, you’ll see something like `Docker version 24.x.x, build …`.  In this case **skip step 5**.

### Step 5: Install Docker 
```sh
sudo apt-get update
sudo apt-get install -y ca-certificates curl
curl -fsSL https://get.docker.com | sudo sh
```
Verify installation:
```sh
docker --version
docker compose version
```
Allow running Docker without sudo:
```sh
sudo usermod -aG docker $USER
newgrp docker
```
### Step 6: Create `si.env` and `docker-compose.yaml` files
`si.env` (replace `***` with actual values)
```
SI_DB_HOST=***
SI_DB_DATABASE=***
SI_DB_PORT=***
SI_DB_USER=***
SI_DB_PASS=***

SI_LICENCE_KEY=***  # The License key provided to you by the Sealed Intelligence team.
SI_AUTH_KEY=***     # The key used for password hashing and token generation.
                    # Select a random string. Don't change it when upgrading Sealed Intelligence.

SI_ALLOW_HTTP=true  # remove in production
```
`docker-compose.yaml`
```yaml
services:
  llm:
    image: vllm/vllm-openai:v0.13.0
    container_name: llm
    gpus: all
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
    ipc: host
    volumes:
      - ./hf-cache:/root/.cache/huggingface
    command:
      - --model=openai/gpt-oss-20b
      - --enable-auto-tool-choice
      - --tool-call-parser=openai
      - --reasoning-parser=openai_gptoss
      - --host=0.0.0.0
      - --port=8000
    restart: unless-stopped

  sealed-intelligence:
    # it's recommended to use a specific image tag instead of 'latest'
    image: intellimenta/sealed-intelligence:latest
    container_name: sealed-intelligence
    env_file:
      - ./si.env
    ports:
      - "5000:5000"
    restart: unless-stopped
```

### Step 7: Deploy
- `docker compose up -d`
- Check Status: `docker compose ps -a`
- If the Sealed Intelligence container status is "Exited", you can use `docker compose logs sealed-intelligence` to see the logs and troubleshoot the issue. After troubleshooting you can restart that container using `docker compose restart sealed-intelligence`
- Follow the LLM container logs: `docker compose logs -f llm`. Once it shows "Application startup complete", the LLM container is ready.
- Verify deployment is working by browsing `http://<VM-Public-IP>:5000`


## Configuration

After creating an account and logging in as admin, it's time for configuring the app. You can access the admin panel by clicking on the gear icon at the top-right of the app and selecting "Admin Panel".

- **Add a Database Connection**. You can either manually add a connection or sync with your BI tool.
    - To manually add a database connection, click on 'Add Database Connection'.
    - To connect your BI tool, see [here](./bi_integration.md).
- **Manage Users Access to Database Tables/Views**
    - In Sealed Intelligence, groups are used to manage table-level access; Users are assigned to groups, and groups are given access to tables. 
    - There are two built-in groups: Admins and Default. All new users are automatically added to the Default group. You can add and manage groups in Admin Panel > User Management > User Groups.
    - To specify which user groups should have access to which tables from a database, go to Admin Panel > Database Connections > [DB Name] > Manage Permissions.
    !!! note
        It is recommended to use views instead of tables for Sealed Intelligence, as they provide greater flexibility. In the views:

        - Include only the columns that are relevant for answering user queries. The fewer irrelevant columns you have, the better the response quality will be.
        - Filter out rows that are redundant.
        - Perform renaming, type conversion, or any other transformations that can help make the data cleaner.

- **Manage Semantic Layer and Metadata**
    - You can provide business metrics, domain-knowledge and descriptions at database, table and column-level (Admin Panel > Database Connections > [DB Name] > Semantic Layer and Metadata Management).
    - You should add descriptions only when it's actually helpful. For example, the column sale_date does not need a description, but a column like abc23 would. 
    - The `db schema` command shows you what data is shared with the LLM when you ask a question.
- **Email Setup**
Setup email (Settings > Admin Panel > Email Setup) so new users can verify their email address. It's also used for password-reset functionality.
- **Query Suggestions**
New users usually don't know what type of questions they can ask. Adding query suggestions (Settings > Admin Panel > Query Suggestions) can greatly help with that. Users would see the query suggestions in the main page when they login.
- **Add Text-to-SQL Translation Tests**  
When Sealed Intelligence answers users' analytical questions, it goes through a series of steps. One of the most critical steps is the text-to-SQL translation. If this step is not performed correctly, the insights provided to users may be unreliable.  
For this reason, it is crucial to ensure that the text-to-SQL translation is working correctly. See [here](./admin_panel_overview.md#text-to-sql-translation-tests) for more details.
