# EndGame V2 - Project Documentation

## Overview

This is **EndGame V2**, a DDOS prevention and security system for Tor onion services (darknet websites). It acts as a protective front-end layer that:

- Filters malicious traffic using NGINX with custom Lua scripts
- Provides captcha challenges to verify legitimate users
- Implements rate limiting based on Tor circuit IDs
- Protects backend onion services from distributed denial-of-service attacks

## Important Notice

**This project CANNOT run on Replit** because it:

1. **Requires root/system access**: Modifies system files in `/etc/`, installs system packages, and manages daemons
2. **Needs custom NGINX compilation**: Compiles NGINX from source with specialized modules
3. **Requires Tor daemon**: Runs multiple Tor processes for onion service functionality
4. **System-specific**: Designed exclusively for Debian 10 bare metal servers
5. **Network configuration**: Needs low-level network and firewall control

## Advanced Security Features

In addition to V3 Onion Management, EndGame V2 includes several other critical security components:

### 1. Web Application Firewall (NAXSI)
- **Rulesets**: Includes `naxsi_core.rules` for blocking common web attacks (SQLi, XSS, RFI).
- **Custom Whitelisting**: `naxsi_whitelist.rules` allows for application-specific tuning.
- **Circuit Killing**: Malicious requests triggered via WAF can automatically trigger a Tor circuit reset.

### 2. System Hardening (Sysctl)
- **Network Optimization**: The included `sysctl.conf` tunes `net.core.somaxconn` and `net.core.netdev_max_backlog` to handle massive concurrent connection surges.
- **SYN Protection**: Enhanced SYN cookie and spoof protection configuration.

### 3. Vanguards Protection
- Protects against guard discovery and traffic analysis attacks.
- Integrated into the `startup.sh` and `onionbalance.sh` workflows.

### 4. Lua-based Rate Limiting
- **Dual-Layer Protection**: Limits are applied to both the Tor Circuit ID (`$proxy_protocol_addr`) and the custom captcha cookie (`$cookie_dcap`).
- **Dynamic Blacklisting**: Cookies that fail validation are temporarily blacklisted across the NGINX shared dictionary.


- **nginx.conf** - Main NGINX configuration with custom modules
- **site.conf** - Virtual host configuration with Lua integration
- **setup.sh** - Automated installation script (requires root)
- **lua/cap.lua** - Captcha and access control logic
- **resty/** - Lua libraries for NGINX (cookies, sessions, encryption)
- **getdependencies.sh** - Downloads required NGINX modules and libraries

### Technology Stack

**Web Server:**
- NGINX with custom compiled modules:
  - lua-nginx-module (Lua scripting in NGINX)
  - naxsi (Web Application Firewall)
  - socks-nginx-module (Tor proxy support)
  - headers-more-nginx-module
  - echo-nginx-module

**Security & Privacy:**
- Tor (onion service hosting)
- Vanguards (circuit protection)
- OnionBalance (load balancing across multiple fronts)
- NAXSI WAF (SQL injection, XSS protection)

**Programming:**
- Lua/LuaJIT for request filtering and captcha generation
- Bash scripts for deployment automation
- Python (STEM library for Tor control)

## How This Should Be Deployed

### Prerequisites
- Dedicated Debian 10 server (4 CPU cores @ 3GHz+, 4GB RAM recommended)
- Root/sudo access
- Understanding of Tor onion services and DDOS mitigation

### Deployment Steps

1. **Set up OnionBalance** (see LOOKHERE-scripts/onionbalance/)
2. **Configure setup.sh** with your settings:
   - MASTERONION: Your OnionBalance v3 address
   - TORAUTHPASSWORD: Tor control authentication
   - KEY/SALT: Session encryption keys
   - BACKENDONIONURL: Your backend onion service
   - HEXCOLOR, SITENAME: Branding customization

3. **Get dependencies**: Run `./getdependencies.sh`
4. **Run setup**: Execute `./setup.sh` as root
5. **Scale out**: Deploy multiple fronts and add to OnionBalance

### Configuration Files to Customize

- `resty/caphtml_d.lua` - Base64 encoded favicon and logo images
- `queue.html` - Queue page branding
- `naxsi_whitelist.rules` - WAF whitelist for your specific application
- `site.conf` - Rate limiting values (lines 114-115, 263, 269)

## Onion Management (V3)

The project includes built-in support for **V3 Onion Management** via OnionBalance. 

### Location
- Scripts and documentation are in `LOOKHERE-scripts/onionbalance/`.

### Key Features
- **Scalability**: Overcomes the Tor introduction point limit by load-balancing across multiple front instances.
- **OnionBalance V3**: Specialized support for version 3 onion services with distinct descriptors.
- **Automated Setup**: The `onionbalance.sh` script handles the installation of OnionBalance, Tor, and Vanguards on a management server.

### Configuration
1. Install on a separate 2 CPU / 2GB RAM server.
2. Modify `params.py` to set `N_INTROS_PER_INSTANCE = 1` as recommended in the onionbalance README.
3. Use `onionbalance-config` to generate your master onion and instance keys.


Each EndGame front:
1. Receives Tor connections
2. Challenges visitors with captchas
3. Rate limits based on circuit ID and cookies
4. Filters malicious traffic with NAXSI WAF
5. Proxies legitimate requests to backend via Tor or local connection

## Use Case

This is designed for **darknet marketplace operators** and **high-value onion services** that face:
- Large-scale DDOS attacks
- Extortion attempts
- Need for anonymity preservation
- High availability requirements

## Penetration Testing Toolkit

A comprehensive Python-based pentest suite is included at `pentest/endgame_pentest.py` to validate all security layers after deployment.

### Test Categories (15 categories, 40+ checks)
1. Connectivity
2. Host Header Validation
3. HTTP Method Filtering (PUT/DELETE/PATCH/OPTIONS/TRACE)
4. User-Agent Enforcement
5. POST Referer Validation
6. Cookie/Session System (queue cookie, HttpOnly, SameSite)
7. Forged Cookie Rejection
8. Rate Limiting (circuit + cookie)
9. WAF - SQL Injection (6 vectors)
10. WAF - XSS (6 vectors)
11. WAF - Directory Traversal (4 vectors)
12. WAF - RFI (4 vectors)
13. Security Headers
14. Tor2Web Proxy Blocking
15. Kill Endpoint

### Usage
```bash
# Full suite against onion target
python3 pentest/endgame_pentest.py -t YOUR_ONION.onion

# Against IP target
python3 pentest/endgame_pentest.py -t 10.10.10.10 -p 80 --no-tor

# Selective testing
python3 pentest/endgame_pentest.py -t TARGET --test-waf
python3 pentest/endgame_pentest.py -t TARGET --test-rate
python3 pentest/endgame_pentest.py -t TARGET --test-auth
python3 pentest/endgame_pentest.py -t TARGET --test-headers
```

### Output
- Color-coded terminal results with PASS/FAIL
- Security grade (A-F)
- JSON report file auto-saved

## Deployment Guide

A complete step-by-step deployment guide is available in `DEPLOY.md`, covering:
- OnionBalance manager setup
- Front server configuration and deployment
- Backend linking (remote over Tor or local proxy)
- Branding customization
- Security validation with the pentest toolkit
- Scaling and monitoring
- Troubleshooting guide
- Pre-launch security checklist

## Recent Changes

- 2025-01-08: Imported to Replit (documentation created)
- Added penetration testing toolkit (pentest/)
- Added comprehensive deployment guide (DEPLOY.md)

## User Preferences

N/A - This is infrastructure software, not a development project suitable for Replit.

## Why This Documentation Exists

Since this project cannot run on Replit, this documentation serves to:
1. Explain what the project is
2. Document its intended deployment environment
3. Provide guidance for proper deployment
4. Prevent confusion about why it won't work in a containerized environment

## Alternative Approach for Replit

If you want to develop/test components of this system on Replit, you could:
- Extract and test the Lua scripts in isolation
- Develop the captcha generation logic
- Test the session/cookie management separately
- Create a mock frontend for the captcha interface

However, the full integrated system requires a dedicated Debian server with root access.
