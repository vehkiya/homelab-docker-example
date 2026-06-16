# 🏡 Core Homelab Starter Kit

Welcome! If you are looking to self-host services at home securely, you are in the right place. This setup is a streamlined, core version of the architecture found in **"vehkiya/homelab"**, designed to get you up and running without overwhelming configuration bloat.

---

## 🎯 Repository Goals

The goal of this project is to establish a rock-solid foundation for your homelab using just three essential components:

1. **The Guard (Tailscale VPN):** Keeps your connection secure and allows you to access your home services from anywhere in the world.
2. **The Traffic Director (Traefik Reverse Proxy):** Directs incoming traffic to the right app and automatically hooks up secure HTTPS (SSL) certificates so your data is encrypted.
3. **The Proof of Concept (Whoami):** A tiny, lightweight test application to prove everything is working perfectly before you start adding your own apps (like Plex, Nextcloud, or Home Assistant).

---

## 🛠️ Prerequisites & Preparation

Before spinning things up, you will need a few basic ingredients:

- **Docker & Docker Compose** installed on your machine.
- **A registered domain name** (e.g., `yourname.com`). You can get one cheaply via Cloudflare, Namecheap, or any major registrar.
- **An external Docker network.** Traefik needs a dedicated highway to talk to your other containers. Create it by running this command in your terminal:
  ```bash
  docker network create traefik_default
  ```

---

## 🚀 Step-by-Step Setup Guide

### Step 1: Configure your Environment (`.env`)

Create a copy of `.env.example` file named `.env` in the same folder as your `compose.yml`. This file holds your personal settings so you don't have to hardcode them.

Fill it out like this:

```env
BASE_DIR=/home/user/homelab       # Change to the actual path of your folder
DOMAIN=yourdomain.com             # Your registered domain
ACME_EMAIL=you@example.com        # Your email for SSL expiration warnings
TAILSCALE_AUTH_KEY=tskey-auth...  # Your Tailscale auth key (from your Tailscale dashboard)
CERT_RESOLVER=cloudflare          # The name of your certificate provider block
DNS_PROVIDER=cloudflare           # Your DNS provider
DNS_RESOLVERS=1.1.1.1:53,8.8.8.8:53

# Port configurations
WEB_PORT=80
WEBSECURE_PORT=443
```

More certificate resolves supported by Traefik and details can be found on [the official Traefik documentation](https://doc.traefik.io/traefik/reference/install-configuration/tls/certificate-resolvers/overview/)

### Step 2: Launch the Stack (Staging Mode)

By default, your `compose.yml` is configured to use Let's Encrypt **Staging** certificates. This ensures everything is configured properly without risk of hitting strict Let's Encrypt rate limits.

Open your terminal in the directory containing your files and run:

```bash
docker compose up -d
```

> 💡 **What does this do?** The `-d` flag stands for "detached" mode. It tells Docker to run the containers quietly in the background.

### Step 3: Verify with Staging Certificates

Check the Traefik logs to ensure it is communicating with the staging servers correctly:

```bash
docker logs reverse-proxy -f
```

Once finished, try navigating to `https://whoami.yourdomain.com` in your browser. Because we are using staging certificates, your browser will show an "Untrusted Certificate Authority" warning. Click through the warning to confirm the `whoami` page successfully displays your connection information.

### Step 4: Switch to Production Certificates

Once you have confirmed that the staging setup works, you can manually activate real, trusted certificates:

1. Open your `compose.yml` file.
2. Find the Let's Encrypt Staging configuration line under the `reverse-proxy` arguments:
   ```yaml
   - "--certificatesresolvers.cloudflare.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory"
   ```
3. **Comment out** or delete that line to allow Traefik to point to the production endpoints by default.
4. Restart your stack to fetch the real certificate:
   ```bash
   docker compose up -d --force-recreate
   ```
   Now your browser will display a beautiful, secure padlock without warnings! _(It can take a few minutes for the certificates to be generated and provisioned)_

---

## 🔐 Understanding SSL: HTTP01 vs. DNS01 Challenges

To get a secure `https://` web address, Let's Encrypt (our certificate authority) needs to prove that you actually own your domain name. It does this using one of two "challenges":

### 1. The DNS01 Challenge (The "Secret Handshake" Method) — ⭐ PREFERRED

- **How it works:** Let's Encrypt says, _"Go to your domain provider (like Cloudflare) and create a hidden text record."_ Traefik logs into your DNS account using an API token, automatically places the secret text record there, Let's Encrypt checks it, and your certificate is granted.
- **Why it's preferred & required for Private Services:** Because verification happens entirely on the internet's public DNS servers, **your home server never has to open a single port to the public web.** \* **Security Benefit:** This is the safest way to run a homelab. You can keep your apps completely hidden inside your local network or Tailscale VPN, entirely invisible to malicious internet scanners, while still enjoying valid, trusted HTTPS encryption.

### 2. The HTTP01 Challenge (The "Front Door" Method)

- **How it works:** Let's Encrypt says, _"Put this secret file on your website."_ It then tries to visit your domain on Port 80 from the public internet to read that file. If it can see it, you get your SSL certificate.
- **How to use it:** In the `compose.yml` file, you would comment out the DNS challenge lines and uncomment the `.httpchallenge` configuration lines.
- **The Downside:** Your homelab **must be fully exposed to the public internet** on Port 80 for validation to pass, opening up a potential entry point for attackers.

---

## 💡 Suggested Future Improvements

Now that your core stack is running, consider these next steps to harden your setup:

- **Use a Docker Socket Proxy:** Right now, Traefik talks directly to `/var/run/docker.sock` to see when new containers launch. If someone somehow hacked your Traefik container, they could control your entire Docker system. Using a container like `docker-socket-proxy` acts as a firewall, letting Traefik only see the bare minimum metadata it needs to route traffic safely.
