# 🛒 E-Commerce API Testing & Security Assessment

> A comprehensive REST API testing and security audit framework for production e-commerce platforms — covering authentication, payment pipelines, PII compliance, and OWASP API Top 10 vulnerability scanning.

---

## 📌 Project Overview

| Field | Details |
|---|---|
| **Target System** | ShopCore API v2.4.1 (`api.shopcore.io`) |
| **Assessment Type** | Black-box + Grey-box API Security Testing |
| **Test Cases** | 201 automated · 187 passed · 14 failed |
| **Vulnerabilities Found** | 9 (1 Critical · 3 High · 3 Medium · 2 Low) |
| **Coverage** | 93% of documented endpoints |
| **Tools Used** | Postman · Burp Suite · OWASP ZAP · Custom Python Harness |

---

## 🗂️ Repository Structure

```
ecommerce-api-security/
├── test-suites/
│   ├── auth/                  # JWT, OAuth2, RBAC tests
│   ├── products/              # CRUD, pagination, search
│   ├── orders/                # Order lifecycle, cancellations
│   ├── payments/              # Stripe, idempotency, webhooks
│   ├── inventory/             # Stock, concurrency, alerts
│   └── users/                 # PII, GDPR, account deletion
├── security/
│   ├── vulnerability-report.md
│   ├── checklist-owasp.md
│   └── findings/
│       ├── VLN-001-BOLA.md
│       ├── VLN-002-mass-assignment.md
│       ├── VLN-003-rate-limiting.md
│       ├── VLN-004-jwt-weak-secret.md
│       └── VLN-005-sql-injection.md
├── collections/
│   └── shopcore-api.postman_collection.json
├── scripts/
│   ├── run_all_tests.sh
│   └── generate_report.py
└── README.md
```

---

## 🧪 Test Suites

### 1. Authentication & Authorization — ✅ 38/38 Passed
- JWT signature validation and expiry enforcement
- OAuth2 authorization code flow with PKCE
- Role-based access control (admin / customer / guest)
- Refresh token rotation and single-use enforcement
- `alg: none` JWT attack vector testing

### 2. Product Catalog API — ✅ 42/44 Passed
- Full CRUD lifecycle testing
- Pagination boundary and off-by-one validation
- SQL injection via `sort` and `filter` query parameters → **VLN-005**
- File upload MIME type spoofing tests
- Search relevance and XSS payload in query strings

### 3. Order Management — ⚠️ 29/34 Passed
- Order creation, status transitions, cancellation flows
- **BOLA/IDOR** on `GET /orders/{id}` — sequential ID enumeration → **VLN-001**
- Concurrent order placement race conditions
- Refund eligibility bypass attempts

### 4. Payment Processing — ❌ 18/25 Passed
- Stripe charge idempotency key validation
- Webhook HMAC signature verification bypass → **VLN-006**
- No rate limiting on `POST /checkout` → **VLN-003**
- PCI-DSS field exposure in order responses
- Double-charge prevention under network retries

### 5. Inventory & Stock — ✅ 28/30 Passed
- Concurrent write race condition (oversell scenario)
- Reserve/release atomicity under load
- Low-stock alert trigger accuracy

### 6. User Profiles & PII — ❌ 32/38 Passed
- Mass assignment of `isAdmin` flag via `PUT /users/{id}` → **VLN-002**
- GDPR right-to-erasure completeness audit
- PII masking in list responses (fixed during assessment)
- Data portability export validation

---

## 🔐 Security Findings

### Critical

| ID | Vulnerability | Endpoint | Status |
|---|---|---|---|
| VLN-001 | **BOLA / IDOR** — Any authenticated user can read any order by ID enumeration | `GET /orders/{id}` | 🔴 Open |

### High

| ID | Vulnerability | Endpoint | Status |
|---|---|---|---|
| VLN-002 | **Mass Assignment** — `isAdmin` injectable via user update body | `PUT /users/{id}` | 🔴 Open |
| VLN-003 | **No Rate Limiting** — Checkout brute-forceable | `POST /checkout` | 🟡 In Progress |
| VLN-004 | **Weak JWT Signing** — Predictable `HS256` secret in staging | `POST /auth/token` | 🟡 In Progress |
| VLN-005 | **SQL Injection** — Unsanitised `sort` param in raw query | `GET /products?sort=` | 🔴 Open |

### Medium

| ID | Vulnerability | Endpoint | Status |
|---|---|---|---|
| VLN-006 | **Webhook Bypass** — HMAC signature not verified | `POST /webhooks/payment` | 🟡 In Progress |
| VLN-007 | **PII Exposure** — Full billing address in order list | `GET /orders` | ✅ Fixed |
| VLN-008 | **CORS Misconfiguration** — Wildcard on credentialed endpoints | `/api/*` | ✅ Fixed |

### Low

| ID | Vulnerability | Status |
|---|---|---|
| VLN-009 | **Verbose Error Leakage** — Stack traces in 500 responses | ✅ Fixed |

---

## 🛡️ Security Checklist (OWASP API Top 10)

| # | Category | Result |
|---|---|---|
| API1 | Broken Object Level Authorization | ❌ FAIL — VLN-001 |
| API2 | Broken Authentication | ⚠️ PARTIAL — VLN-004 |
| API3 | Broken Object Property Level Auth | ❌ FAIL — VLN-002 |
| API4 | Unrestricted Resource Consumption | ❌ FAIL — VLN-003 |
| API5 | Broken Function Level Authorization | ✅ PASS |
| API6 | Unrestricted Access to Sensitive Business Flows | ⚠️ PARTIAL |
| API7 | Server Side Request Forgery | ✅ PASS |
| API8 | Security Misconfiguration | ⚠️ PARTIAL — VLN-008 (fixed) |
| API9 | Improper Inventory Management | ✅ PASS |
| API10 | Unsafe Consumption of APIs | ✅ PASS |

---

## 🚀 Running the Tests

### Prerequisites
```bash
# Python 3.10+
pip install requests pytest pytest-html python-dotenv

# Node.js (for Newman/Postman CLI runner)
npm install -g newman newman-reporter-htmlextra
```

### Run All Test Suites
```bash
# Run full Python test harness
pytest test-suites/ -v --html=reports/results.html

# Run Postman collection via Newman
newman run collections/shopcore-api.postman_collection.json \
  --env-var "baseUrl=https://api.shopcore.io" \
  --reporters cli,htmlextra \
  --reporter-htmlextra-export reports/newman-report.html
```

### Run Specific Suite
```bash
pytest test-suites/auth/ -v
pytest test-suites/payments/ -v -k "webhook"
```

### Security Scan with OWASP ZAP
```bash
docker run -t owasp/zap2docker-stable zap-api-scan.py \
  -t https://api.shopcore.io/openapi.json \
  -f openapi -r zap-report.html
```

---

## 📊 Sample Test Case

```python
# test-suites/orders/test_bola.py

import requests
import pytest

BASE_URL = "https://api.shopcore.io"

def test_order_idor_access(user_a_token, user_b_order_id):
    """
    VLN-001: Verify that User A cannot access User B's order.
    Expected: 403 Forbidden
    Actual (before fix): 200 OK with full order data — BOLA confirmed
    """
    headers = {"Authorization": f"Bearer {user_a_token}"}
    response = requests.get(f"{BASE_URL}/orders/{user_b_order_id}", headers=headers)

    assert response.status_code == 403, (
        f"BOLA VULNERABILITY: Got {response.status_code} instead of 403. "
        f"User A accessed User B's order {user_b_order_id}"
    )

def test_order_idor_sequential_scan(user_token):
    """
    Test that sequential ID enumeration is blocked (rate limited or randomised IDs).
    """
    headers = {"Authorization": f"Bearer {user_token}"}
    accessible = []

    for order_id in range(1000, 1020):
        r = requests.get(f"{BASE_URL}/orders/{order_id}", headers=headers)
        if r.status_code == 200:
            accessible.append(order_id)

    assert len(accessible) <= 1, (
        f"Sequential enumeration returned {len(accessible)} orders. "
        f"Use UUIDs or ownership checks."
    )
```

---

## 🔧 Remediation Recommendations

### VLN-001 — BOLA (Critical)
```python
# Add ownership check before returning order
def get_order(order_id: str, current_user: User):
    order = db.query(Order).filter_by(id=order_id).first()
    if order.user_id != current_user.id and not current_user.is_admin:
        raise HTTPException(status_code=403, detail="Forbidden")
    return order
```

### VLN-002 — Mass Assignment (High)
```python
# Whitelist allowed fields per role — never bind full request body
class UserUpdateSchema(BaseModel):
    name: Optional[str]
    email: Optional[str]
    # isAdmin is NOT here — only settable via admin-specific endpoint
```

### VLN-003 — Rate Limiting (High)
```python
# Redis sliding window rate limiter on checkout
from slowapi import Limiter
limiter = Limiter(key_func=get_remote_address)

@app.post("/checkout")
@limiter.limit("5/minute")
async def checkout(request: Request, ...):
    ...
```

### VLN-004 — Weak JWT Secret (High)
```bash
# Use RS256 asymmetric signing in production
# Generate key pair:
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem
```

### VLN-005 — SQL Injection (High)
```python
# NEVER: raw string concat
query = f"SELECT * FROM products ORDER BY {sort_field}"  # ❌

# ALWAYS: parameterised / ORM whitelist
ALLOWED_SORT = {"price", "name", "created_at"}
if sort_field not in ALLOWED_SORT:
    raise ValueError("Invalid sort field")
products = db.query(Product).order_by(getattr(Product, sort_field))  # ✅
```

---

## 📋 Skills Demonstrated

- **API Security Testing** — OWASP API Top 10, BOLA/IDOR, injection, broken auth
- **Test Automation** — Pytest, Newman/Postman, OWASP ZAP
- **Penetration Testing Methodology** — Reconnaissance, exploitation, reporting
- **PCI-DSS & GDPR Compliance** — Payment data handling, PII audit, right to erasure
- **Vulnerability Reporting** — CVE-style write-ups with PoC and remediation
- **CI/CD Integration** — GitHub Actions pipeline for automated regression security testing
- **Tools** — Burp Suite · Postman · OWASP ZAP · Python · Docker

---

