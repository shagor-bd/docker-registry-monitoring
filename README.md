Creating a private Docker registry with a local DNS resolver and certificate on an Ubuntu server involves several steps. Here's a comprehensive guide:

---

### **Prerequisites**
1. An Ubuntu server with root or sudo access.
2. Docker and Docker Compose installed.
3. OpenSSL installed for generating certificates.
4. A domain name or subdomain pointing to the server's IP address (for local DNS configuration).

---

### **Steps to Set Up**

#### **1. Install Docker and Docker Compose**
If not already installed:
```bash
sudo apt update
sudo apt install -y docker.io docker-compose
sudo systemctl enable docker --now
```

#### **2. Set Up Local DNS Resolver**
If you're using `dnsmasq` for local DNS:
```bash
sudo apt install -y dnsmasq
```

Edit the configuration to map your registry domain to the server's IP address:
```bash
echo "address=/myregistry.local/127.0.0.1" | sudo tee /etc/dnsmasq.d/docker-registry
```

Restart `dnsmasq`:
```bash
sudo systemctl restart dnsmasq
```

Ensure `dnsmasq` is resolving DNS locally:
```bash
nslookup myregistry.local
```

#### **3. Generate Self-Signed Certificates**
Generate an SSL certificate for your private registry:
```bash
sudo mkdir -p /etc/docker/certs.d/myregistry.local
cd /etc/docker/certs.d/myregistry.local

openssl req -newkey rsa:4096 -nodes -sha256 -keyout myregistry.local.key -x509 -days 365 -out myregistry.local.crt \
    -subj "/C=US/ST=State/L=City/O=Organization/OU=Org Unit/CN=myregistry.local"
```

#### **4. Start the Private Docker Registry**
Create a `docker-compose.yml` file:
```yaml
version: '3'

services:
  registry:
    image: registry:2
    container_name: private_registry
    ports:
      - "443:443"
    environment:
      REGISTRY_HTTP_ADDR: "0.0.0.0:443"
      REGISTRY_HTTP_TLS_CERTIFICATE: "/certs/myregistry.local.crt"
      REGISTRY_HTTP_TLS_KEY: "/certs/myregistry.local.key"
    volumes:
      - /etc/docker/certs.d/myregistry.local:/certs
      - /var/lib/registry:/var/lib/registry
```

Start the registry:
```bash
sudo docker-compose up -d
```

#### **5. Configure Docker to Trust the Registry**
On the same server or any Docker client, configure Docker to trust the self-signed certificate:
```bash
sudo mkdir -p /etc/docker/certs.d/myregistry.local
sudo cp /etc/docker/certs.d/myregistry.local/myregistry.local.crt /etc/docker/certs.d/myregistry.local/ca.crt
```

Restart Docker:
```bash
sudo systemctl restart docker
```

#### **6. Test the Private Registry**
Tag an image and push it to the private registry:
```bash
docker pull alpine
docker tag alpine myregistry.local/alpine
docker push myregistry.local/alpine
```

Verify the image is listed:
```bash
curl -k https://myregistry.local/v2/_catalog
```

---

### **Optional Enhancements**
1. **Secure Access**: Use a real domain and certificates from Let's Encrypt for added security.
2. **Authentication**: Configure basic authentication by setting the `REGISTRY_AUTH` environment variable.
3. **Monitoring**: Add monitoring tools like `Prometheus` or `Grafana`.

Let me know if you'd like to customize or troubleshoot any of these steps!

To add user authentication to your private Docker registry, you'll use **basic authentication**. This requires creating a file with user credentials and configuring the registry to use it. Below are the detailed steps:

---

### **Steps to Enable Authentication**

#### **1. Install `htpasswd`**
Install the `apache2-utils` package, which includes the `htpasswd` tool:
```bash
sudo apt update
sudo apt install -y apache2-utils
```

---

#### **2. Create a Credentials File**
Use `htpasswd` to create a file with the username and password:
```bash
sudo mkdir -p /etc/docker/registry
sudo htpasswd -c /etc/docker/registry/htpasswd your-username
```
- Replace `your-username` with the desired username.
- You'll be prompted to set a password.

To add more users later:
```bash
sudo htpasswd /etc/docker/registry/htpasswd another-username
```

---

#### **3. Update the `docker-compose.yml` File**
Modify the `docker-compose.yml` file to include the authentication configuration:
```yaml
version: '3'

services:
  registry:
    image: registry:2
    container_name: private_registry
    ports:
      - "443:443"
    environment:
      REGISTRY_HTTP_ADDR: "0.0.0.0:443"
      REGISTRY_HTTP_TLS_CERTIFICATE: "/certs/myregistry.local.crt"
      REGISTRY_HTTP_TLS_KEY: "/certs/myregistry.local.key"
      REGISTRY_AUTH: "htpasswd"
      REGISTRY_AUTH_HTPASSWD_REALM: "Registry Realm"
      REGISTRY_AUTH_HTPASSWD_PATH: "/auth/htpasswd"
    volumes:
      - /etc/docker/certs.d/myregistry.local:/certs
      - /etc/docker/registry:/auth
      - /var/lib/registry:/var/lib/registry
```

---

#### **4. Restart the Docker Registry**
Bring down the existing container (if running) and restart it with the updated configuration:
```bash
sudo docker-compose down
sudo docker-compose up -d
```

---

#### **5. Test the Authentication**

1. **Log in to the registry**:
   ```bash
   docker login myregistry.local
   ```
   - Enter the username and password created earlier.

2. **Tag and push an image**:
   ```bash
   docker pull alpine
   docker tag alpine myregistry.local/alpine
   docker push myregistry.local/alpine
   ```

3. **Verify the image is pushed**:
   ```bash
   curl -k -u your-username:your-password https://myregistry.local/v2/_catalog
   ```

---

### **Optional Enhancements**
- **Force HTTPS**: Use real certificates from Let's Encrypt.
- **Secure Credential Storage**: Use Docker credential helpers to avoid storing passwords in plain text.
- **Monitoring and Logs**: Add logging configurations to track authentication and registry usage.

Let me know if you'd like further assistance!

Adding monitoring tools like **Prometheus** and **Grafana** to monitor your private Docker registry involves setting up the tools and configuring the Docker Registry to expose metrics. Here's how you can do it:

---

### **Step-by-Step Guide**

#### **1. Configure Docker Registry for Metrics**
Modify your `docker-compose.yml` file to expose Prometheus-compatible metrics:
```yaml
version: '3'

services:
  registry:
    image: registry:2
    container_name: private_registry
    ports:
      - "443:443"
      - "5000:5000" # Optional, for Prometheus metrics
    environment:
      REGISTRY_HTTP_ADDR: "0.0.0.0:443"
      REGISTRY_HTTP_TLS_CERTIFICATE: "/certs/myregistry.local.crt"
      REGISTRY_HTTP_TLS_KEY: "/certs/myregistry.local.key"
      REGISTRY_AUTH: "htpasswd"
      REGISTRY_AUTH_HTPASSWD_REALM: "Registry Realm"
      REGISTRY_AUTH_HTPASSWD_PATH: "/auth/htpasswd"
      REGISTRY_METRICS_ENABLED: "true"  # Enable metrics
    volumes:
      - /etc/docker/certs.d/myregistry.local:/certs
      - /etc/docker/registry:/auth
      - /var/lib/registry:/var/lib/registry
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
volumes:
  grafana-data:
```

---

#### **2. Create the Prometheus Configuration File**
Create a `prometheus.yml` file in the same directory as your `docker-compose.yml`:
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "docker_registry"
    static_configs:
      - targets: ['localhost:5000']
```

This configuration tells Prometheus to scrape metrics from the Docker Registry at port `5000`.

---

#### **3. Start the Stack**
Run the stack with Docker Compose:
```bash
sudo docker-compose up -d
```

---

#### **4. Access Prometheus and Grafana**
- **Prometheus**: Open your browser and navigate to `http://<server-ip>:9090`.
- **Grafana**: Open your browser and navigate to `http://<server-ip>:3000`.
  - Default login: `admin` / `admin`.

---

#### **5. Add Prometheus as a Data Source in Grafana**
1. Log in to Grafana.
2. Go to **Configuration > Data Sources > Add data source**.
3. Select **Prometheus**.
4. Enter the URL: `http://prometheus:9090` (internal Docker network).
5. Save and test.

---

#### **6. Import a Grafana Dashboard for Docker Registry**
1. Go to **Create > Import** in Grafana.
2. Use the Docker Registry dashboard JSON file or an existing dashboard ID (e.g., ID `9935` for a Docker Registry-specific dashboard).
3. Configure the data source as Prometheus and import it.

---

### **Metrics Available**
The Docker Registry exposes metrics like:
- **HTTP requests**: Counts of registry HTTP requests by response code.
- **Storage metrics**: Disk space usage for blobs and manifests.
- **Latency**: Request latencies for the registry.

---

Let me know if you want help customizing the dashboards or troubleshooting!



To enable your private Docker registry (along with Prometheus and Grafana) to start automatically at system boot, you can create a custom **systemd service file**. Here's how to do it for your `docker-compose` stack:

---

### **Step-by-Step Guide**

#### **1. Create a Systemd Service File**
Create a new service file for your Docker Compose stack, for example, `docker-registry.service`:

```bash
sudo nano /etc/systemd/system/docker-registry.service
```

Add the following content:

```ini
[Unit]
Description=Private Docker Registry with Prometheus and Grafana
Requires=docker.service
After=docker.service

[Service]
Restart=always
WorkingDirectory=/path/to/your/docker-compose/directory
ExecStart=/usr/bin/docker-compose up -d
ExecStop=/usr/bin/docker-compose down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```

Replace `/path/to/your/docker-compose/directory` with the actual path where your `docker-compose.yml` file is located.

---

#### **2. Reload Systemd**
After creating the service file, reload the systemd manager to recognize the new service:
```bash
sudo systemctl daemon-reload
```

---

#### **3. Enable the Service at Startup**
Enable the service to start on boot:
```bash
sudo systemctl enable docker-registry.service
```

---

#### **4. Start the Service**
Start the service manually to test it:
```bash
sudo systemctl start docker-registry.service
```

---

#### **5. Check the Service Status**
Verify that the service is running without issues:
```bash
sudo systemctl status docker-registry.service
```

---

#### **6. Logs and Troubleshooting**
To view logs for the service:
```bash
journalctl -u docker-registry.service
```

---

### **Explanation of Key Sections**
- **`[Unit]`**: Specifies dependencies and ordering. Ensures Docker starts before the registry.
- **`[Service]`**:
  - `WorkingDirectory`: Specifies the directory containing `docker-compose.yml`.
  - `ExecStart`: Command to start the stack.
  - `ExecStop`: Command to stop the stack gracefully.
  - `Restart=always`: Ensures the service restarts on failure.
- **`[Install]`**: Configures the service to run at boot.

---

With this setup, your private Docker registry and its associated tools (Prometheus and Grafana) will start automatically after a reboot. Let me know if you encounter any issues!