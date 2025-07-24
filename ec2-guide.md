# EC2 Deployment Flow

## 1. EC2 Deployment Workflow

This section outlines how the project was deployed from a local development environment to a live production server hosted on AWS EC2.

---

### Instance Selection

- **Instance Type:** `t4g.large` (2 vCPUs, 8 GiB memory)
- **Storage:** 28 GiB GP2 (General Purpose SSD)
- **Architecture:** ARM64 (Graviton-based)
- **Why t4g.large?**
    - **Cost-efficient burst capacity:** Chosen for its ability to handle short bursts of high CPU usage while maintaining low baseline costs.
    - **Hard compute cap:** Unlike other instance families, T4 instances have a defined performance baseline and will **throttle instead of incurring surprise overage charges**, which helps control cost risk.
    - **Eligible for AWS free tier credits** (depending on usage/region at the time).

---

### Elastic IP Allocation

- **Purpose:**
    - Ensures a **static, public-facing IP address** for the EC2 instance, which remains constant across restarts or reboots.
    - Critical for domain setup and DNS mapping.
- **Steps:**
    - Allocated a new Elastic IP from the EC2 dashboard.
    - Associated the Elastic IP with the running instance via **Actions → Associate Elastic IP**.
- **Verification:**
    - Confirmed association by visiting:
        
        ```bash
        curl http://<elastic-ip>
        
        ```
        

---

### Instance Provisioning

- **Security Groups:**
    - Inbound Rules:
        - Port 22 (SSH) — for remote access
        - Port 80 (HTTP) — for web traffic
        - Port 443 (HTTPS) — for secure traffic
    - Outbound — open (default)

---

## 2. Code Deployment Strategy

- **Git-based Workflow:**
    - Codebase pushed from local to GitHub
    - Pulled into EC2 using:
        
        ```bash
        git clone https://github.com/varunrvkmr/JobTrackr.git
        git pull origin <branch>
        ```
        
- **Environment Files:**
    - `.env` files set up per service (frontend/backend)
    - Sensitive data excluded from Git, added via `scp` or created on EC2
- **Docker Compose Launch:**
    - Used multi-container Docker Compose setup for:
        - Flask backend
        - PostgreSQL (if hosted locally)
        - React frontend
        - Traefik reverse proxy (infrastructure layer)
    - Brought containers up with:
        
        ```bash
        docker-compose up -d --build
        
        ```
        
- **Volumes/Persistence:**
    - Bound Docker volumes used for DB persistence
    - Optional: EBS snapshot or S3 backup for database

---

## 3. Domain and HTTPS Setup

This section details how the EC2-hosted application was connected to a custom domain and secured using HTTPS via Traefik and Let’s Encrypt.

### Domain Configuration

- **Domain Registrar:** GoDaddy
- **DNS Records:**
    - Created an **A Record** pointing to the EC2 instance’s **Elastic IP**
        - **Type:** A
        - **Host:** `@` (root domain) or `www` if subdomain
        - **Value:** `<your-elastic-ip>`
        - **TTL:** Default (e.g., 600 seconds)
- **Verification:**
    - Confirmed domain resolution using:
        
        ```bash
        dig +short yourdomain.com
        
        ```
        
- **Propagation Notes:**
    - DNS changes may take up to 15–30 minutes to fully propagate.

---

### Reverse Proxy Setup with Traefik

- **Why Traefik?**
    - Automatic HTTPS support via Let’s Encrypt
    - Docker-native — detects containers and routes based on labels
    - Lightweight and perfect for containerized projects
- **Traefik Configuration Highlights:**
    - Mounted `traefik.yml` and `acme.json` for configuration and cert storage
    - Docker provider enabled for dynamic service discovery
    - File provider used if additional static routing needed
    
    ```yaml
    entryPoints:
      web:
        address: ":80"
      websecure:
        address: ":443"
    
    providers:
      docker:
        exposedByDefault: false
    
    ```
    
- **Docker Labels (per service):**
    
    Set in your `docker-compose.yml`:
    
    ```yaml
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.frontend.rule=Host(`yourdomain.com`)"
      - "traefik.http.routers.frontend.entrypoints=websecure"
      - "traefik.http.routers.frontend.tls.certresolver=myresolver"
    
    ```
    

---

### Let’s Encrypt SSL (HTTPS)

- **TLS Certificate Automation:**
    - Enabled Traefik’s built-in ACME resolver
    - Used HTTP-01 challenge to validate domain ownership
    - Persisted certs in `acme.json`:
        
        ```yaml
        certificatesResolvers:
          myresolver:
            acme:
              email: your-email@example.com
              storage: /letsencrypt/acme.json
              httpChallenge:
                entryPoint: web
        
        ```
        
- **Permissions:**
    - Ensured `acme.json` exists and is chmod 600:
        
        ```bash
        touch acme.json
        chmod 600 acme.json
        
        ```
        
- **Certificate Renewal:**
    - Handled automatically by Traefik
    - Logs available via:
        
        ```bash
        docker logs <traefik-container-name>
        
        ```
        

---

### Post-Setup Checks

- **SSL Validation:**
    - Verified HTTPS status via:
        
        ```bash
        curl -I https://yourdomain.com
        
        ```
        
        or browser padlock
        
- **Cross-Origin Considerations:**
    - Updated `.env` and CORS rules to reflect the new domain name

---

## 4. CORS and API Endpoint Configuration

This section documents how Cross-Origin Resource Sharing (CORS) and API endpoint handling were configured and debugged during deployment. It includes `.env` setups, CORS policies, and lessons learned while bridging the frontend and backend across domains and ports.

---

### 🌉 Frontend ↔ Backend Integration

- **Frontend:** React (or Next.js with TypeScript)
- **Backend:** Flask API with JWT-based authentication
- **Connection Strategy:**
    - The frontend sends requests to the backend through a Traefik-managed HTTPS endpoint (e.g., `https://jobtrackr.ai/api`).
- **Environment File Setup:**
    
    In `.env`:
    
    ```
    REACT_APP_API_URL=https://jobtrackr.ai/api
    NEXT_PUBLIC_API_URL=https://jobtrackr.ai/api
    
    ```
    
    - These environment variables control where the frontend sends API requests.
    - Ensured **no trailing slashes** to avoid malformed URLs when concatenating.

---

### Flask CORS Setup

- **Library Used:** `flask-cors`
- **CORS Initialization in `create_app()`:**
    
    ```python
    from flask_cors import CORS
    
    CORS(app, supports_credentials=True, origins=["https://jobtrackr.ai"])
    
    ```
    
- **Key Considerations:**
    - `supports_credentials=True` was necessary for JWT cookie-based auth.
    - Wildcard  origins were avoided in production to maintain security.
    - Allowed origins were explicitly listed and updated as needed.

---

### Common Issues Encountered

1. **Double Slashes in Endpoint URLs**
    - Example: `https://jobtrackr.ai/api//auth/login`
    - **Cause:** Combining `REACT_APP_API_URL=https://jobtrackr.ai/api` with `fetch(BASE_URL + "/auth/login")`
    - **Fix:** Stripped trailing slash from the base URL or used a helper to normalize paths.
2. **Malformed URLs from `.env`**
    - Some `.env` values were interpreted incorrectly due to copy-paste or concatenation.
    - **Fix:** Validated with `console.log(BASE_URL)` and double-checked `.env` syntax.
3. **CORS Rejections in Browser**
    - CORS preflight `OPTIONS` requests were failing due to missing headers.
    - **Fixes:**
        - Ensured backend explicitly allowed `Content-Type`, `Authorization`, etc.
        - Verified browser request modes (`cors`, `include`, etc.) matched backend setup.
4. **HTTPS + Cookies Not Persisting**
    - Cookies not sent in requests because `credentials: "include"` was missing.
    - **Fix:** Ensured frontend requests used:
        
        ```
        fetch(BASE_URL + "/auth/check", {
          method: "GET",
          credentials: "include"
        })
        
        ```
        

---

### Final Setup Recap

- Backend CORS configured to:
    - Allow only the production domain
    - Support credentials (JWT cookies)
    - Handle preflight `OPTIONS` requests
- Frontend environment files pointed cleanly to the `/api` path
- All fetch calls included `credentials: "include"` and were verified in browser dev tools
- Logs (`curl`, browser console, Flask logs) were key to debugging

---