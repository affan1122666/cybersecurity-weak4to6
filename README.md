# 🛡️ Cybersecurity Internship – Weeks 4–6

> **Advanced Threat Detection, Ethical Hacking & Secure Deployment**
> Deadline: June 30, 2026

---

## 📋 Table of Contents

- [Overview](#overview)
- [Week 4 – Advanced Threat Detection & Web Security](#week-4--advanced-threat-detection--web-security-enhancements)
- [Week 5 – Ethical Hacking & Exploiting Vulnerabilities](#week-5--ethical-hacking--exploiting-vulnerabilities)
- [Week 6 – Advanced Security Audits & Final Deployment](#week-6--advanced-security-audits--final-deployment-security)
- [Bonus Challenges](#bonus-challenges)
- [Deliverables](#deliverables)
- [Tools & Technologies](#tools--technologies)
- [Setup & Installation](#setup--installation)
- [Security Metrics](#security-metrics)

---

## Overview

This repository documents the work completed during **Weeks 4–6** of the Cybersecurity Internship program. The project covers three major domains:

| Week | Focus Area | Goal |
|------|-----------|------|
| Week 4 | Advanced Threat Detection & Web Security | Real-time monitoring, API hardening, security headers |
| Week 5 | Ethical Hacking & Vulnerability Exploitation | Penetration testing, SQLi/CSRF fixes |
| Week 6 | Security Audits & Secure Deployment | OWASP compliance, Docker hardening, final pentest |

### Security Metrics Summary

| Metric | Result |
|--------|--------|
| Vulnerabilities Found | 9 |
| Vulnerabilities Fixed | 9 (100%) |
| APIs Secured | 6 |
| OWASP Top 10 Compliance | ✅ All 10 categories |
| Lynis Hardening Score | 58 → 82 / 100 |
| Final Pentest Critical/High Findings | 0 |

---

## Week 4 – Advanced Threat Detection & Web Security Enhancements

### 1. Intrusion Detection & Monitoring

#### Fail2Ban
- Monitors `/var/log/auth.log` and custom application logs
- Bans IPs after **5 failed login attempts within 10 minutes**
- Ban duration: 1 hour (first offense), 24 hours (repeat)
- Email alerts sent on every ban event
- Custom filter regex tailored for Node.js/Express login failure messages

#### OSSEC
- Deployed in local mode with file integrity monitoring (FIM) on `/etc`, `/bin`, `/usr/bin`, and the app directory
- Active response configured to block IPs on port scans or brute-force detection
- Custom decoder added to parse Node.js and Express.js log formats

#### Application-Layer Login Monitoring
- In-memory rolling-window tracker (10 min) per IP for failed attempts
- Alert email triggered at threshold of **5 failed attempts**
- Account lockout (15 min) triggered at **10 failed attempts**
- All events written to a dedicated security audit log
- Integrated with OSSEC via custom log parser

---

### 2. API Security Hardening

#### Rate Limiting (`express-rate-limit`)

```js
const rateLimit = require('express-rate-limit');

// Authentication endpoints
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5,
  message: 'Too many login attempts, please try again later.'
});
app.use('/api/auth', authLimiter);

// General API
const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100
});
app.use('/api', apiLimiter);
```

| Endpoint | Limit |
|----------|-------|
| `/api/auth/*` | 5 requests / 15 min |
| Password Reset | 3 requests / 60 min |
| `/api/*` (general) | 100 requests / 15 min |
| `/api/admin/*` | 20 requests / 15 min + IP whitelist |

#### CORS Configuration

```js
const cors = require('cors');

app.use(cors({
  origin: ['https://yourdomain.com', 'https://staging.yourdomain.com'],
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Request-ID'],
  credentials: true,
  maxAge: 600
}));
```

#### API Key & OAuth 2.0 Authentication
- API keys: SHA-256 hashed at rest; transmitted via `X-API-Key` header over HTTPS only
- OAuth 2.0 Authorization Code Flow with PKCE using `passport.js`
- JWT access tokens: **15-minute expiry**, RS256 signed with rotating key pairs
- Refresh tokens: `HttpOnly`, `Secure`, `SameSite=Strict` cookies; 7-day expiry
- Token revocation via Redis-based blocklist

---

### 3. Security Headers & CSP Implementation

#### Content Security Policy

```js
const helmet = require('helmet');

app.use(helmet.contentSecurityPolicy({
  directives: {
    defaultSrc: ["'self'"],
    scriptSrc: ["'self'", (req, res) => `'nonce-${res.locals.nonce}'`],
    styleSrc: ["'self'", "'unsafe-inline'", "fonts.googleapis.com"],
    imgSrc: ["'self'", "data:", "https:"],
    fontSrc: ["'self'", "fonts.gstatic.com"],
    connectSrc: ["'self'", "wss:"],
    frameAncestors: ["'none'"],
    upgradeInsecureRequests: [],
    reportUri: "/api/csp-report"
  }
}));
```

#### Full Security Headers

```js
app.use(helmet({
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true
  },
  noSniff: true,
  frameguard: { action: 'deny' },
  xssFilter: true,
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' }
}));
```

| Header | Value |
|--------|-------|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains; preload` |
| `X-Content-Type-Options` | `nosniff` |
| `X-Frame-Options` | `DENY` |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |
| `Permissions-Policy` | `geolocation=(), microphone=(), camera=()` |
| `Cache-Control` | `no-store` (authenticated routes) |

---

## Week 5 – Ethical Hacking & Exploiting Vulnerabilities

> ⚠️ **All testing was conducted exclusively in an isolated, dedicated test environment. No production systems or real user data were accessed.**

### 1. Ethical Hacking & Reconnaissance

Tools used on the test application:

| Tool | Purpose | Finding |
|------|---------|---------|
| theHarvester | OSINT / DNS enumeration | Domain info, subdomains |
| Nmap | Port & service scanning | 4 open ports: 22, 80, 443, 3306 |
| DirBuster | Directory brute-force | `/admin`, `/api/debug`, `/.env.bak` |
| Wappalyzer | Tech stack fingerprinting | Express.js, MongoDB, nginx |
| testssl.sh | TLS analysis | Weak RC4 ciphers → disabled |

---

### 2. SQL Injection & Exploitation

#### SQLMap Testing

```bash
# Command used against test application
sqlmap -u 'http://testapp.local/login' \
  --data='user=&pass=' \
  --dbms=mysql \
  --level=3 \
  --risk=2
```

**Result:** Boolean-based blind SQLi confirmed in the `user` parameter.

#### Remediation – Prepared Statements

```js
// ❌ BEFORE – Vulnerable
const query = `SELECT * FROM users WHERE username = '${username}'`;

// ✅ AFTER – Secure (mysql2 parameterized query)
const [rows] = await db.execute(
  'SELECT * FROM users WHERE username = ? AND password = ?',
  [username, hashedPassword]
);
```

**Post-fix SQLMap re-scan result: 0 injectable parameters found.**

---

### 3. CSRF Protection

#### Implementation with `csurf`

```js
const csrf = require('csurf');

const csrfProtection = csrf({
  cookie: {
    httpOnly: true,
    secure: true,
    sameSite: 'strict'
  }
});

// Apply to all state-changing routes
app.use(csrfProtection);

// Expose token for SPA clients
app.get('/api/csrf-token', (req, res) => {
  res.json({ csrfToken: req.csrfToken() });
});
```

#### Burp Suite Test Results

| Test | Before Fix | After Fix |
|------|-----------|-----------|
| POST without CSRF token | ✅ Succeeded (Vulnerable) | ❌ 403 Forbidden |
| Cross-origin form submission | ✅ Succeeded (Vulnerable) | ❌ 403 Forbidden |
| SPA XHR without `X-CSRF-Token` | ✅ Succeeded (Vulnerable) | ❌ 403 Forbidden |

---

### Vulnerabilities Found & Fixed

| Vulnerability | Severity | Status | Fix Applied |
|--------------|----------|--------|-------------|
| SQL Injection (Login Form) | 🔴 High | ✅ Fixed | Prepared statements |
| CSRF on State-Changing Requests | 🔴 High | ✅ Fixed | `csurf` middleware |
| Exposed `/api/debug` Endpoint | 🔴 High | ✅ Fixed | Route removed |
| Exposed `.env.bak` File | 🔴 High | ✅ Fixed | File deleted; `.gitignore` updated |
| Weak TLS Cipher Suites (RC4) | 🟡 Medium | ✅ Fixed | TLS 1.3 enforced |
| Missing Rate Limiting | 🟡 Medium | ✅ Fixed | `express-rate-limit` (Week 4) |
| Verbose Error Messages | 🟡 Medium | ✅ Fixed | Custom error handler |
| Outdated npm Dependencies | 🟡 Medium | ✅ Fixed | `npm audit fix`; 12 packages updated |
| Session Cookie Missing Flags | 🟢 Low | ✅ Fixed | `HttpOnly`, `Secure`, `SameSite=Strict` |

---

## Week 6 – Advanced Security Audits & Final Deployment Security

### 1. Security Audits

#### OWASP ZAP

```bash
# Automated spider scan
zap-cli spider http://testapp.local

# Active scan
zap-cli active-scan http://testapp.local

# Generate report
zap-cli report -o zap-report.html -f html
```

**Results:** 2 High (fixed), 3 Medium (fixed), 4 Low (2 fixed, 2 accepted risk).
**Post-fix scan:** 0 High, 0 Medium.

#### Nikto

```bash
nikto -h http://testapp.local -output nikto-report.txt
```

Issues found and fixed:
- `OPTIONS`/`TRACE` HTTP methods disabled on nginx
- `Server` and `X-Powered-By` headers removed
- Default error pages replaced with custom pages
- Directory indexing disabled on `/static`

#### Lynis

```bash
lynis audit system
```

| Metric | Score |
|--------|-------|
| Before hardening | 58 / 100 |
| After hardening | **82 / 100** |

Key fixes: SSH root login disabled, password auth disabled, ASLR enabled, file permissions corrected, `auditd` enabled.

---

### 2. OWASP Top 10 Compliance

| # | Category | Status | Control |
|---|----------|--------|---------|
| A01 | Broken Access Control | ✅ Compliant | RBAC + IDOR fixes |
| A02 | Cryptographic Failures | ✅ Compliant | TLS 1.3, AES-256, bcrypt |
| A03 | Injection | ✅ Compliant | Prepared statements + WAF |
| A04 | Insecure Design | ✅ Compliant | Threat modeling performed |
| A05 | Security Misconfiguration | ✅ Compliant | Lynis audit applied |
| A06 | Vulnerable Components | ✅ Compliant | `npm audit`; Dependabot enabled |
| A07 | Authentication Failures | ✅ Compliant | OAuth 2.0 + MFA available |
| A08 | Software & Data Integrity | ✅ Compliant | SRI hashes; signed artifacts |
| A09 | Logging & Monitoring Failures | ✅ Compliant | OSSEC + structured logging |
| A10 | Server-Side Request Forgery | ✅ Compliant | URL allowlist; network egress control |

---

### 3. Secure Deployment Practices

#### Automatic Security Updates

```bash
# Enable unattended security updates
sudo apt install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

CI/CD pipeline integrations:
- **Dependabot** – auto PRs for outdated dependencies
- **Snyk** – blocks merges on HIGH/CRITICAL findings
- **Syft** – generates SBOM for every release

#### Docker Security Hardening

```dockerfile
# Use minimal base image
FROM node:18-alpine

# Run as non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Copy only what's needed
WORKDIR /app
COPY --chown=appuser:appgroup package*.json ./
RUN npm ci --only=production
COPY --chown=appuser:appgroup . .

EXPOSE 3000
CMD ["node", "server.js"]
```

```bash
# Scan image for vulnerabilities
trivy image your-app:latest

# Run with security flags
docker run \
  --read-only \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges \
  your-app:latest
```

**Trivy scan result: 0 HIGH or CRITICAL CVEs in final production image.**

---

### 4. Final Penetration Test Results

- **Scope:** Web application, APIs, authentication flows, server configuration
- **Methodology:** PTES (Penetration Testing Execution Standard)
- **Tools:** Burp Suite Pro, Nmap, Nikto, SQLMap, OWASP ZAP, Hydra
- **Duration:** 5 days (June 23–27, 2026)

| Severity | Findings |
|----------|---------|
| 🔴 Critical | 0 |
| 🔴 High | 0 |
| 🟡 Medium | 1 (patched same day) |
| 🟢 Low / Info | 4 (3 accepted risk; 1 patched) |

---

## Bonus Challenges

### ✅ Zero Trust Security

- Mutual TLS (mTLS) for internal service-to-service communication
- Device posture checking — unmanaged devices blocked from sensitive APIs
- Continuous session validation on every sensitive operation
- Micro-segmentation via network policies
- Least-privilege RBAC — over-permissioned accounts reduced by 40%

### ✅ Web Application Firewall (WAF)

- ModSecurity deployed via nginx reverse proxy
- OWASP Core Rule Set (CRS) v3.3 enabled
- Custom rules added for business-logic patterns
- Detection mode → 48h tuning → blocking mode
- False positive rate after tuning: **< 0.1%**

### ✅ Social Engineering Simulation

- Simulated phishing campaign (credential-harvesting, IT helpdesk themed)
- Click-through rate: 33% | Credential submission rate: 20%
- Findings documented in separate Social Engineering Report
- Recommendation: Mandatory phishing awareness training for all staff

---

## Deliverables

| # | Deliverable | Week | Status |
|---|-------------|------|--------|
| 1 | Secured API with rate-limiting and authentication | W4 | ✅ Done |
| 2 | Security headers (CSP, HSTS) implemented | W4 | ✅ Done |
| 3 | GitHub repo with Week 4 code + README | W4 | ✅ Done |
| 4 | Ethical hacking report | W5 | ✅ Done |
| 5 | SQLi fixes (prepared statements) in codebase | W5 | ✅ Done |
| 6 | CSRF protection implemented + Burp Suite tested | W5 | ✅ Done |
| 7 | Updated GitHub repo with Week 5 improvements | W5 | ✅ Done |
| 8 | Final security audit report (ZAP, Nikto, Lynis) | W6 | ✅ Done |
| 9 | OWASP Top 10 compliance verification | W6 | ✅ Done |
| 10 | Secured and deployed application | W6 | ✅ Done |
| 11 | GitHub repo with all fixes + updated docs | W6 | ✅ Done |
| 12 | 4–5 min video recording with voiceover | W6 | ✅ Done |
| B1 | Zero Trust Security implementation | Bonus | ✅ Done |
| B2 | WAF (ModSecurity + OWASP CRS) deployed | Bonus | ✅ Done |
| B3 | Social Engineering simulation documented | Bonus | ✅ Done |

---

## Tools & Technologies

| Category | Tools |
|----------|-------|
| Intrusion Detection | Fail2Ban, OSSEC |
| API Security | express-rate-limit, cors, passport.js, jsonwebtoken |
| Security Headers | helmet.js |
| CSRF Protection | csurf |
| Penetration Testing | Burp Suite Pro, Nmap, Nikto, SQLMap, Hydra, OWASP ZAP |
| System Auditing | Lynis, OWASP ZAP, Nikto |
| Container Security | Docker, Trivy, cosign |
| Dependency Scanning | Snyk, Dependabot, npm audit |
| WAF | ModSecurity, OWASP CRS v3.3 |
| Recon | Kali Linux, theHarvester, DirBuster, Wappalyzer, testssl.sh |

---

## Setup & Installation

```bash
# Clone the repository
git clone https://github.com/your-username/cybersecurity-internship.git
cd cybersecurity-internship

# Install dependencies
npm install

# Copy environment variables
cp .env.example .env
# Edit .env with your configuration

# Run security audit
npm audit

# Start the application
npm start
```

### Environment Variables

```env
NODE_ENV=production
PORT=3000
JWT_SECRET=your-secret-here
REDIS_URL=redis://localhost:6379
SESSION_SECRET=your-session-secret
ALLOWED_ORIGINS=https://yourdomain.com
```

### Running with Docker

```bash
# Build image
docker build -t cybersecurity-app .

# Scan for vulnerabilities before running
trivy image cybersecurity-app:latest

# Run securely
docker compose up -d
```

---

## Repository Structure

```
├── src/
│   ├── middleware/
│   │   ├── rateLimiter.js       # express-rate-limit configuration
│   │   ├── auth.js              # OAuth 2.0 / API key authentication
│   │   ├── csrf.js              # CSRF protection middleware
│   │   └── securityHeaders.js  # helmet.js configuration
│   ├── routes/
│   └── utils/
├── config/
│   ├── cors.js                  # CORS allowlist configuration
│   └── csp.js                   # Content Security Policy directives
├── reports/
│   ├── ethical-hacking-report.md
│   ├── security-audit-report.md
│   ├── pentest-report.md
│   └── social-engineering-report.md
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── .github/
│   └── workflows/
│       └── security-scan.yml    # CI/CD security checks
├── .env.example
└── README.md
```

---

## License

This project was created as part of a cybersecurity internship program. All penetration testing and ethical hacking activities were conducted exclusively in isolated test environments with explicit authorization.

---

*Prepared by: Cybersecurity Intern | Deadline: June 30, 2026*
