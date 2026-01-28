# LiteLLM + Open WebUI Docker Deployment


[🇨🇳 中文 ](README_ZH.md)
This project uses Docker Compose to deploy a complete LLM proxy service stack, including LiteLLM proxy, PostgreSQL database, and Open WebUI interface.

## Service Architecture

### LiteLLM Proxy Service
- **Port**: 4000
- **Function**: Provides LLM model proxy services, supporting multiple model management and API calls
- **Configuration**: Configured via external configuration file `litellmconfig.yaml`
- **Features**:
  - Supports adding models via UI interface
  - Health check mechanism

### PostgreSQL Database
- **Port**: 5432
- **Function**: Stores LiteLLM configurations and model data
- **Data Persistence**: Uses named volume `litellm_postgres_data` to persist data
- **Connection Information**:
  - Database name: litellm
  - Username: llmproxy
  - Password: dbpassword9090

### Open WebUI Interface
- **Port**: 9036
- **Function**: Provides a user-friendly web interface integrated with LiteLLM proxy
- **Auto Connection**: Automatically connects to LiteLLM via Docker internal network (http://litellm:4000/v1)
- **Data Persistence**: Uses `open-webui` volume to persist user data
- **Dependencies**: Automatically waits for LiteLLM service to start before launching

## Usage Instructions

### Start All Services
```bash
docker compose up -d
```

**Note**: The `-d` parameter runs in detached (background) mode, allowing services to run in the background and freeing the terminal. Without `-d`, services run in the foreground with real-time logs; press `Ctrl+C` to stop.

### Check Service Status
```bash
docker compose ps
```

### Stop All Services
```bash
docker compose down

### Restart All Services
```bash
docker compose down && docker compose up -d
```

### View Service Logs
```bash
# View all service logs
docker compose logs

# View specific service logs
docker compose logs litellm
docker compose logs open-webui
```

## Access Addresses

### Local Access
- **LiteLLM Dashboard**: http://localhost:4000/ui/
- **Open WebUI**: http://localhost:9036/

### LAN Access

#### Get Server IP Address

First, obtain the LAN IP address of the computer running the Docker services:

**Windows**:
```bash
ipconfig
# Look for "IPv4 Address", usually in 192.168.x.x format
```

**Linux/macOS**:
```bash
ip addr show
# or
ifconfig
# Look for inet address, usually in 192.168.x.x format
```

#### LAN Access Addresses

Assuming the server IP is `192.168.1.111`, the LAN access addresses are:

- **LiteLLM Dashboard**: http://192.168.1.111:4000/ui/
- **Open WebUI**: http://192.168.1.111:9036/

#### Port Explanation

According to [`docker-compose.yml`](docker-compose.yml) configuration:
- **9036 port**: Open WebUI service (container internal 8080 port mapped to host 9036 port)
- **4000 port**: LiteLLM service

#### Troubleshooting

If unable to access from other LAN devices, check the following:

1. **Firewall Settings**:
   - **Windows**: Create inbound rules in Windows Firewall to allow ports 9036, 4000
     - Open Windows Firewall Advanced Settings
     - Select "Inbound Rules" > "New Rule"
     - Select "Port", click "Next"
     - Select "TCP", enter specific ports: 9036,4000
     - Select "Allow the connection", click "Next"
     - Select applicable network types (usually all checked), click "Next"
     - Enter rule name (e.g., "LiteLLM Service"), complete creation
   - **Linux**: Use `ufw allow 9036`, `ufw allow 4000` (inbound rules by default)
   - **macOS**: Configure inbound connections in System Preferences > Security & Privacy > Firewall

2. **Docker Service Status**:
   ```bash
   docker compose ps
   # Ensure all services status is "running"
   ```

3. **Port Binding Check**:
   - Confirm port bindings in [`docker-compose.yml`](docker-compose.yml) use `0.0.0.0` (correctly configured)
   - This binds services to all network interfaces, not just localhost

4. **Network Connectivity Test**:
   ```bash
   # Test on server
   curl http://192.168.1.111:9036
   
   # Test on other devices
   ping 192.168.1.111
   ```

5. **Router Settings**:
   - Ensure LAN devices can access each other
   - Check for AP isolation or guest network restrictions

#### Security Recommendations

1. **Change Default Password**: Upon first access to Open WebUI, immediately change the default admin password
2. **Network Isolation**: If possible, deploy services in an isolated network segment
3. **VPN Access**: For external access, use VPN instead of exposing ports to the public internet
4. **Regular Updates**: Keep Docker images and system updated

## Configuration Instructions

### Configuring Proxy on Cloud VMs with ShellCrash Installed

If your cloud VM has [ShellCrash](https://github.com/juewuy/ShellCrash) installed for bypassing restrictions, you need to configure proxy in `docker-compose.yml` for the LiteLLM service to access external APIs.

#### Configuration Steps

1. **Get Server IP Address**:
   ```bash
   hostname -I | awk '{print $1}'
   ```
   Assuming the returned IP is `172.31.219.189`

2. **Check ShellCrash Configuration**:
   ```bash
   cat /tmp/ShellCrash/config.yaml | grep "mixed-port"
   ```
   Confirm the mixed port, usually 8964

3. **Modify docker-compose.yml**:

   Add the following three lines to the `environment` section of the `litellm` service:

   ```yaml
   services:
     litellm:
       # ... other configs ...
       environment:
         DATABASE_URL: "postgresql://llmproxy:dbpassword9090@db:5432/litellm"
         STORE_MODEL_IN_DB: "True"
         LITELLM_ENABLE_PROMETHEUS: "True"
         HTTP_PROXY: "http://172.31.219.189:8964"  # Use ShellCrash proxy
         HTTPS_PROXY: "http://172.31.219.189:8964"  # Use ShellCrash proxy
         NO_PROXY: "localhost,127.0.0.1,db,open-webui"  # Don't proxy local services
         # ... other environment variables ...
   ```

4. **Restart Services**:
   ```bash
   docker compose down && docker compose up -d
   ```

#### Notes

- Replace `172.31.219.189` with your actual server IP
- Replace `8964` with your ShellCrash mixed port
- `NO_PROXY` ensures local Docker network communication doesn't go through proxy
- Services must be restarted after configuration changes

### Environment Variables
- Loaded via `.env` file
- Important items include API keys, database connection info, etc.

### Configuration Files
- **LiteLLM Config**: `litellmconfig.yaml`

### Data Volumes
- `postgres_data`: PostgreSQL data persistence
- `open-webui`: Open WebUI user data persistence

## Docker Image Version Management

### View Latest Version

LiteLLM Docker image is hosted on GitHub Packages; check latest version at:
https://github.com/berriai/litellm/pkgs/container/litellm

### Update Docker Image Version

Current version: `ghcr.io/berriai/litellm-database:main-v1.80.0-nightly`

To update to the latest version:

1. **Visit GitHub Packages page** to view the latest available version tag
2. **Edit [`docker-compose.yml`](docker-compose.yml) file**, modify the `image` field for the `litellm` service:

```yaml
services:
  litellm:
    image: ghcr.io/berriai/litellm:main-v1.80.0.dev2  # Replace with latest version
    # ... other configurations unchanged
```

3. **Pull new image and restart services**:

```bash
# Pull the latest image
docker compose pull

# Restart services to use new version
docker compose up -d

# Verify version update
docker compose ps
```

### Version Tag Explanation

- `main-v1.80.0-nightly`: Nightly version with latest features but may be unstable
- `main-v1.80.0.dev2`: Development version with specific updates
- `main-stable`: Stable version, suitable for production

**Recommendation**: Use stable version (`main-stable`) for production; use latest for development/testing to get new features.

## Configuration File Management

### Configuration File Location Notes

This project places the [`litellmconfig.yaml`](litellmconfig.yaml) configuration file in the **project directory** by default, which is recommended.

#### Default Configuration (Recommended)
```yaml
volumes:
  # Mount config file from project folder (recommended)
  - ./litellmconfig.yaml:/app/config.yaml
```

### Using Config File in Project Directory (Default Method)

1. Ensure [`litellmconfig.yaml`](litellmconfig.yaml) exists in project root
2. Use default docker-compose.yml (no changes needed)
3. Start services:
   ```bash
   docker compose up -d
   ```

### Using Config File from Other Locations on Computer

If you want to place the config file elsewhere (e.g., Documents folder), follow these steps:

#### 1. Determine Config File Path

**Windows Example**:
```yaml
volumes:
  # Absolute path example (C drive user Documents folder)
  - /c/Users/your_username/Documents/litellmconfig.yaml:/app/config.yaml
  
  # Or Windows path format (extra quotes needed)
  - "C:\Users\your_username\Documents\litellmconfig.yaml:/app/config.yaml"
```

**Linux/macOS Example**:
```yaml
volumes:
  # Absolute path example
  - /home/your_username/documents/litellmconfig.yaml:/app/config.yaml
  
  # Or use ~ for home directory
  - ~/documents/litellmconfig.yaml:/app/config.yaml
```

#### 2. Modify docker-compose.yml

Edit [`docker-compose.yml`](docker-compose.yml) file, locate `litellm` service `volumes` section, change left-side path to your actual config path:

```yaml
services:
  litellm:
    # ... other configs ...
    volumes:
      # Change left path to your actual config file path
      - /your/full/path/to/config/litellmconfig.yaml:/app/config.yaml
    # ... other configs ...
```

**Important Notes**:
- Right side `/app/config.yaml` must remain unchanged
- Left path must be full host path to your config file
- Windows users: Use forward slashes `/` or double backslashes `\\`

#### 3. Restart Services

After changes, restart for effect:

```bash
# Stop current services
docker compose down

# Restart services
docker compose up -d

# Verify config loaded
docker compose logs litellm
```

### Configuration File Priority

1. **External config file** (via volumes mount) - highest priority
2. **Environment variables** - next
3. **Default config** - lowest

### Configuration File Structure

[`litellmconfig.yaml`](litellmconfig.yaml) includes main sections:

- **general_settings**: General settings like master_key, database connection, etc.
- **litellm_settings**: LiteLLM specific settings like retry count, timeout, etc.
- **model_list**: Model list defining available AI models and configs

### Quick Start Recommendations

**For new users**, strongly recommend default method:
1. Place [`litellmconfig.yaml`](litellmconfig.yaml) in project directory
2. Modify API keys and model settings as needed
3. Run `docker compose up -d` directly

**For advanced users**, if sharing config across projects, place in central location and modify path as above.

## Notes

1. Before first start, ensure API keys in `.env` are correctly configured
2. Windows users: Confirm `litellmconfig.yaml` path is correct
3. Service startup order ensured via `depends_on`
4. All services have health checks for availability
5. Restart required after config changes