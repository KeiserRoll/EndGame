# EndGame V2 - Deployment Guide for Nexus Market

Complete step-by-step instructions to deploy EndGame V2 as a DDOS prevention layer for Nexus Market.

---

## Architecture Overview

```
Users (Tor Browser)
       |
       v
OnionBalance Master Onion  (your public .onion address)
       |
       v
+------+------+------+
|      |      |      |
v      v      v      v
Front1 Front2 Front3 Front4  (EndGame V2 instances)
       |
       v
Nexus Market Backend  (your core application server)
```

---

## Server Requirements

| Role | CPU | RAM | OS | Quantity |
|------|-----|-----|----|----------|
| OnionBalance Manager | 2 cores | 2 GB | Debian 10 | 1 |
| EndGame Front | 4 cores @ 3GHz+ | 4 GB | Debian 10 | 3+ (minimum) |
| Nexus Market Backend | Your existing server | - | - | 1 |

---

## Step 1: Set Up OnionBalance Manager (Separate Server)

This server manages the master .onion address and distributes traffic across your fronts.

```bash
# Transfer the onionbalance files to your management server
scp -r LOOKHERE-scripts/onionbalance/* root@MANAGER_IP:/root/

# SSH into the manager
ssh root@MANAGER_IP

# Run the setup script
chmod +x onionbalance.sh
bash onionbalance.sh
```

After setup completes, you will get a **master onion address**. This is your public-facing Nexus Market .onion URL.

**Important**: Edit the OnionBalance params before running:
```bash
cd /root/onionbalance
nano onionbalance/hs_v3/params.py
# Change: N_INTROS_PER_INSTANCE = 2  ->  N_INTROS_PER_INSTANCE = 1
python3 setup.py install
```

Generate your config:
```bash
onionbalance-config --hs-version v3 -n 3
```

Save the generated master .onion address. You will need it for Step 2.

---

## Step 2: Configure EndGame for Nexus Market

Before deploying to front servers, edit `setup.sh` with your specific values:

```bash
# === REQUIRED: Your OnionBalance master address ===
MASTERONION="YOUR_MASTER_ONION_ADDRESS.onion"

# === REQUIRED: Tor control port password ===
TORAUTHPASSWORD="generate_a_strong_password_here"

# === NEXUS MARKET BACKEND CONNECTION ===
# Option A: Backend is on a DIFFERENT server (connected over Tor)
LOCALPROXY=false
BACKENDONIONURL="YOUR_NEXUS_MARKET_BACKEND.onion"

# Option B: Backend is on the SAME server (local connection - faster)
# LOCALPROXY=true
# PROXYPASSURL="127.0.0.1:8080"   # or whatever port Nexus Market runs on

# === REQUIRED: Shared encryption keys (same across ALL fronts) ===
# Generate with: cat /dev/urandom | tr -dc 'a-zA-Z0-9' | head -c 64
KEY="GENERATE_A_64_CHARACTER_RANDOM_KEY_HERE_USING_COMMAND_ABOVE"
# Generate with: cat /dev/urandom | tr -dc 'a-zA-Z0-9' | head -c 8
SALT="GEN8CHAR"
SESSION_LENGTH=3600

# === NEXUS MARKET BRANDING ===
HEXCOLOR="1a1a2e"        # Dark theme primary color
HEXCOLORDARK="0f0f1a"    # Dark theme secondary color
SITENAME="Nexus Market"
```

### Generate Your Keys
Run these commands to generate secure random keys:

```bash
# Generate 64-character encryption key
echo "KEY: $(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | head -c 64)"

# Generate 8-character salt
echo "SALT: $(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | head -c 8)"

# Generate Tor control password
echo "PASS: $(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | head -c 32)"
```

**Write these down. The KEY and SALT must be identical across all front servers.**

---

## Step 3: Customize Branding

### Captcha Page Logo & Favicon
Edit `resty/caphtml_d.lua`:
- Line 143: Replace the base64 favicon with Nexus Market's favicon
- Line 162: Replace the base64 logo with Nexus Market's logo

Convert your images:
- Favicon: https://base64.guru/converter/encode/image/ico
- Logo: https://base64.guru/converter/encode/image

### Queue Page
Edit `queue.html`:
- Search for `<link href="` to find and replace the favicon
- Search for `.logobgimg` to find and replace the logo
- The site name and colors are auto-replaced by setup.sh

---

## Step 4: Get Dependencies (Run Once)

On the first front server (or a build machine):

```bash
chmod +x getdependencies.sh
./getdependencies.sh
```

This creates a `dependencies/` folder. Copy this folder to all other front servers to avoid re-downloading.

---

## Step 5: Deploy to Front Servers

Transfer all project files to each blank Debian 10 front server:

```bash
# From your local machine
scp -r ./* root@FRONT_SERVER_IP:/root/endgame/
```

SSH into each front server and run:

```bash
ssh root@FRONT_SERVER_IP
cd /root/endgame
chmod +x setup.sh
bash setup.sh
```

The script will:
1. Install all dependencies (Tor, NGINX, build tools)
2. Compile custom NGINX with all security modules
3. Configure Tor hidden service
4. Set up NAXSI WAF rules
5. Start all services
6. Output the front's .onion address

**Save each front's .onion address** - you need them for OnionBalance.

---

## Step 6: Link Fronts to OnionBalance

After deploying all fronts, go back to your OnionBalance manager server.

Edit the OnionBalance config to include all front addresses:

```yaml
# /root/onionbalance/config/config.yaml
services:
  - key: master_key_file.key
    instances:
      - address: FRONT1_ONION_ADDRESS
      - address: FRONT2_ONION_ADDRESS
      - address: FRONT3_ONION_ADDRESS
```

Start OnionBalance:

```bash
cd /root/onionbalance
nohup onionbalance -v info -c config/config.yaml &
```

Verify it's working:

```bash
tail -f nohup.out
# You should see "distinct descriptors" being pushed
```

---

## Step 7: Verify Deployment

### Quick Check
Visit your master .onion address in Tor Browser. You should see the Nexus Market captcha/queue page.

### Full Security Validation
Run the pentest toolkit from any machine with Python 3:

```bash
# Full security audit
python3 pentest/endgame_pentest.py -t YOUR_MASTER_ONION.onion

# Quick WAF check
python3 pentest/endgame_pentest.py -t YOUR_MASTER_ONION.onion --test-waf

# Rate limit check
python3 pentest/endgame_pentest.py -t YOUR_MASTER_ONION.onion --test-rate
```

Target: Security Grade A (90%+ tests passing).

---

## Step 8: Tuning for Nexus Market

### Rate Limits
Edit `site.conf` lines 114-115 to match Nexus Market's page load behavior:

```nginx
# Adjust based on how many resources (CSS, JS, images) a typical page loads
limit_req_zone $proxy_protocol_addr zone=circuits:50m rate=6r/s;
limit_req_zone $cookie_dcap zone=capcookie:50m rate=6r/s;
```

- If Nexus Market pages load 10+ resources, increase to `rate=10r/s`
- Burst values on lines 263/269: adjust `burst=10` to match

### WAF Whitelist
Edit `naxsi_whitelist.rules` for Nexus Market's specific needs:
- If the market uses search with special characters, whitelist those rules
- If users submit PGP keys or messages with special chars, whitelist accordingly

### Session Length
In `setup.sh`, adjust `SESSION_LENGTH`:
- `3600` = 1 hour (recommended for active browsing)
- `7200` = 2 hours (for longer sessions)

### Tor Streams
In `torrc`, adjust `HiddenServiceMaxStreams` to your burst value + 2:
```
HiddenServiceMaxStreams 12    # if burst=10, set to 12
```

---

## Connecting to Nexus Market Backend

### Option A: Remote Backend (Over Tor) - More Secure

Your Nexus Market backend runs on a separate server with its own private .onion address.

In `setup.sh`:
```bash
LOCALPROXY=false
BACKENDONIONURL="your_nexus_backend_private.onion"
```

The EndGame fronts will proxy filtered traffic to Nexus Market through Tor using load-balanced SOCKS connections (ports 9060/9070).

**Backend Requirements:**
- Nexus Market must have its own hidden service running
- The backend .onion should NOT be public (only EndGame fronts know it)
- No rate limiting on the backend - EndGame handles all filtering

### Option B: Local Backend (Same Server) - Faster

If Nexus Market runs on the same server as an EndGame front:

In `setup.sh`:
```bash
LOCALPROXY=true
PROXYPASSURL="127.0.0.1:8080"   # Nexus Market's local port
```

**Pros:** Lower latency, more reliable connection
**Cons:** If the front goes down, the backend goes with it

### Backend Host Header
EndGame passes the original `Host` header to the backend. Make sure Nexus Market accepts requests with your master .onion as the Host header.

---

## Scaling Guide

### When to Add More Fronts
- Under attack: Add fronts when existing ones hit CPU limits
- Proactive: Start with 3, scale to 5-10 for high-traffic markets

### Adding a New Front
1. Set up a new Debian 10 server
2. Copy the project files (with same KEY/SALT/config)
3. Run `setup.sh`
4. Add the new front's .onion to OnionBalance config
5. Restart OnionBalance

### Monitoring
On each front server:
```bash
# Check NGINX status
nginx -t
systemctl status nginx

# Monitor Tor
nyx

# Check error logs
tail -f /var/log/nginx/front_error.log

# Check WAF triggers
tail -f /etc/nginx/naxsi.log
```

---

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| 502 errors | Backend unreachable | Check Tor connectivity, verify backend .onion is running |
| Captcha not loading | Lua error | Check `/var/log/nginx/front_error.log` |
| Rate limiting too aggressive | Burst too low | Increase `burst=` values in site.conf |
| WAF blocking legitimate requests | Rules too strict | Add rules to `naxsi_whitelist.rules` |
| OnionBalance not publishing | Config error | Check `nohup.out` on manager server |
| "Killed path" message | Cookie blacklisted | User needs new Tor identity |

---

## Security Checklist Before Going Live

- [ ] Changed default KEY and SALT in setup.sh
- [ ] Changed TORAUTHPASSWORD from default
- [ ] Updated branding (logo, favicon, colors) for Nexus Market
- [ ] Backend .onion address is NOT publicly known
- [ ] At least 3 front servers deployed
- [ ] OnionBalance publishing distinct descriptors
- [ ] Pentest toolkit shows Grade A
- [ ] WAF whitelist tuned for Nexus Market's forms
- [ ] Rate limits tested against normal browsing patterns
- [ ] NAXSI logs reviewed for false positives
- [ ] Backup of KEY/SALT stored securely (if lost, all sessions invalidate)
