# main-caddy-proxy

A modern global Caddy proxy server that automatically routes incoming HTTPS requests to Docker services by domain
name.  
This setup replaces legacy nginx-proxy configurations with a simpler, more robust solution supporting HTTP/3 out of the
box.

---

## ✨ Features

- ✅ Automatic reverse proxy for Docker containers
- ✅ Automatic HTTPS certificates via Let's Encrypt
- ✅ HTTP/3 (QUIC) support
- ✅ www to non-www redirects
- ✅ Multi-domain support
- ✅ Minimal configuration via Docker labels
- ✅ No separate ACME companion required

---

## 🚀 Quick Start

### 1️⃣ Clone this repository

```sh
git clone --depth=1 --branch=main https://github.com/jonaaix/main-caddy-proxy.git \
&& rm -rf ./main-caddy-proxy/.git
```

---

### 2️⃣ Create the global Docker network

Before starting any containers, create a shared network:

```sh
docker network create main-proxy
```

All containers that should be accessible via the proxy **must** join this network.

---

### 3️⃣ Configure your email

Edit `compose.yaml` and set your email address in `CADDY_DOCKER_EMAIL`:

```yaml
environment:
   - CADDY_DOCKER_EMAIL=info@yourcompany.com
```

This email is used for Let's Encrypt certificate registration.

---

### 4️⃣ Start the proxy server

Bring up the proxy container:

```sh
docker compose up -d
```

Caddy will start listening on ports 80, 443 (HTTP/1.1 + HTTP/2), and 443/udp (HTTP/3).

---

## 🔗 Connecting Services

Attach your services to the `main-proxy` network and declare domains via Docker labels.

---

### Example: Single domain with www redirect

This example routes `example.com` to the container and redirects `www.example.com` to the non-www
canonical URL:

```yaml
services:
   my-app:
      image: your-app-image
      networks:
         - main-proxy
      labels:
         caddy_0: example.com
         caddy_0.reverse_proxy: "{{upstreams 9000}}"
         caddy_0.encode: zstd gzip
         
         caddy_1: www.example.com
         caddy_1.redir: https://example.com{uri}
         
         caddy_2: mydomain.com
         caddy_2.reverse_proxy: "{{upstreams 8080}}"
         caddy_2.encode: zstd gzip

networks:
   main-proxy:
      external: true
```

---

### Example: Multiple domains routing to the same service

To serve multiple domains pointing to the same app:

```yaml
services:
   my-app:
      image: your-app-image
      networks:
         - main-proxy
      labels:
         # Primary domains
         caddy_0: example.com, second.com
         caddy_0.reverse_proxy: "{{upstreams 9000}}"
         
         # Enable gzip and zstd compression
         caddy_0.encode: zstd gzip

         # www redirect for example.com
         caddy_1: www.example.com
         caddy_1.redir: https://example.com{uri}

         # www redirect for second.com
         caddy_2: www.second.com
         caddy_2.redir: https://second.com{uri}
```

---

### Example: Wildcard (catch-all) routing for SaaS custom domains

If you want to route **any domain** (e.g., users pointing `customer1.com`, `customer2.net`, etc.) to a single container,
use the `*` wildcard:

```yaml
services:
   saas-app:
      image: your-saas-app-image
      networks:
         - main-proxy
      labels:
         # Match any domain
         caddy_0: "*"
         # Reverse proxy to container port 9000
         caddy_0.reverse_proxy: "{{upstreams 9000}}"
         # Enable gzip and zstd compression
         caddy_0.encode: zstd gzip
```

⚠️ **Important Notes:**

* For Let's Encrypt to issue certificates, each custom domain **must** have DNS A/AAAA records pointing to your server.
* Caddy will automatically request and maintain certificates for all discovered hostnames.
* You can optionally restrict accepted domains with Caddyfile configuration or advanced label rules if you don’t want
truly everything proxied.

---

### Start your service

After configuring your service, start it with:

```sh
docker compose up -d
```

Within seconds, Caddy will:

* Discover the container
* Issue certificates
* Configure routing
* Serve HTTPS and HTTP/3

---

## 🛠️ Notes and Tips

* Make sure your DNS records point to the host running Caddy.
* If you want to enable additional headers, security settings, or rate limits, you can add further labels per container.
* `compose.yaml` is **versionless** and uses the modern Compose spec (2025 compatible).

---

## 🗂️ Project Structure

```text
main-caddy-proxy
├── compose.yaml                   # Caddy proxy definition
├── README.md                      # This documentation
└── examples/
    ├── compose.simple-example.yaml
    └── compose.multi-domain-example.yaml
```

---

✅ **Done!** Your Caddy proxy is now ready to serve multiple domains with automatic HTTPS and HTTP/3.

---

## Example: Serving a PHP app running on the host with PHP-FPM

You can route a domain to a local PHP application running on your host machine via PHP-FPM.

#### 1️⃣ Make sure PHP-FPM is listening on TCP

Edit your `www.conf`:

```ini
listen = 127.0.0.1:9000
```

Restart PHP-FPM:

```sh
sudo systemctl restart php8.4-fpm
```

#### 2️⃣ Create a `Caddyfile`

Example:

```caddyfile
app.yourdomain.com {
  root * /srv/myapp
  php_fastcgi host.docker.internal:9000
  file_server
}
```

#### 3️⃣ Mount the PHP directory and the Caddyfile

In your `compose.yaml`:

```yaml
volumes:
  - ./Caddyfile:/etc/caddy/Caddyfile
  - /var/www/myapp:/srv/myapp:ro
```

This ensures Caddy can read the PHP files while PHP-FPM executes them.

#### 4️⃣ Start the Caddy proxy

```sh
docker compose up -d
```

Caddy will automatically issue a certificate and serve your PHP site via HTTPS.
