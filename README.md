---

## Tech Stack

| Component | Role |
|---|---|
| **Keycloak 26.x** | Identity Provider — users, roles, groups |
| **n8n** | Workflow orchestration — policy engine |
| **Keycloak REST API** | Bridge between n8n and Keycloak |
| **Podman** | Container runtime for Keycloak |
| **Docker** | Container runtime for n8n |

---

## Lab Environment

| Component | Details |
|---|---|
| Realm | TechCorp |
| Users | Abhijit, Anurag, Mayur |
| Roles | admin, manager, employee, security-analyst |
| Groups | Engineering, Finance, HR, Security |

---

## Workflows

### Workflow 1 — Access Request & Entitlement Assignment

Automates the end-to-end access request lifecycle with real-time SoD enforcement.

**Flow:**
1. Webhook receives access request
2. SoD-Check identifies conflicting roles
3. Keycloak API fetches user's existing roles
4. If conflict detected → REQUEST DENIED
5. If clear → Role assigned via Keycloak API
6. Audit log entry created

**SoD Rule enforced:**
> `manager` + `security-analyst` cannot coexist on the same user

**Test — Approved request:**
```bash
curl -X POST http://localhost:5678/webhook/access-request \
  -H "Content-Type: application/json" \
  -d '{
    "username": "anurag",
    "requested_role": "employee",
    "requested_by": "hr-system",
    "justification": "New hire onboarding"
  }'
```

**Test — Denied request (SoD violation):**
```bash
curl -X POST http://localhost:5678/webhook/access-request \
  -H "Content-Type: application/json" \
  -d '{
    "username": "mayur",
    "requested_role": "manager",
    "requested_by": "hr-system",
    "justification": "Promotion"
  }'
```

---

### Workflow 2 — Access Certification Campaign

Scheduled weekly certification campaign that reviews all user entitlements and generates a compliance report.

**Schedule:** Every Sunday at 8:00 AM

**What it does:**
1. Fetches all users from Keycloak
2. Fetches role assignments for each user
3. Flags users with high privilege roles
4. Detects SoD violations
5. Generates certification report with recommendations

**Sample output:**
```json
{
  "reportId": "CERT-1778232132366",
  "summary": {
    "totalUsers": 3,
    "flagged": 3,
    "sodViolations": 0,
    "clean": 0
  },
  "recommendation": "Review flagged users with their managers"
}
```

---

### Workflow 3 — Auto Revocation & Lifecycle Control

Time-bound access management — roles are automatically revoked after a defined period.

**Flow:**
1. Webhook receives revocation request with expiry time
2. Workflow pauses until expiry time
3. On expiry — role automatically removed from Keycloak
4. Audit log entry created

**Test:**
```bash
curl -X POST http://localhost:5678/webhook/revocation-request \
  -H "Content-Type: application/json" \
  -d '{
    "username": "abhijit",
    "role": "manager",
    "requested_by": "hr-system",
    "reason": "Contract ended",
    "expiry_minutes": 1
  }'
```

---

## Audit Trail

Every action across all 3 workflows generates a structured audit log:

```json
{
  "timestamp": "2026-05-08T10:11:48.118Z",
  "action": "ROLE_REVOKED",
  "username": "abhijit",
  "roleRevoked": "manager",
  "requestedBy": "hr-system",
  "reason": "Contract ended",
  "decidedBy": "Auto-Revocation-Engine",
  "status": "REVOKED"
}
```

---

## Setup Guide

### Prerequisites
- Podman installed
- Docker installed
- n8n running on port 5678

### 1. Start Keycloak
```bash
podman run -d \
  --name keycloak \
  -p 8080:8080 \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=admin123 \
  quay.io/keycloak/keycloak:latest \
  start-dev
```

### 2. Start n8n
```bash
docker run -d \
  --name n8n \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  n8nio/n8n
```

### 3. Import Workflows
1. Open n8n at `http://localhost:5678`
2. Go to Workflows → Import from File
3. Import all 3 JSON files from the `workflows/` folder

### 4. Configure Keycloak
Create a realm called `TechCorp` with:
- Users: abhijit, anurag, mayur
- Roles: admin, manager, employee, security-analyst
- Groups: Engineering, Finance, HR, Security

---

## Known Limitations & Production Improvements

This is a proof of concept. Production deployment would require:

| Gap | Production Solution |
|---|---|
| Hardcoded credentials | HashiCorp Vault or environment variables |
| No retry logic | Exponential backoff with dead-letter queues |
| No correlation IDs | Add requestId to all audit log entries |
| No input validation | Schema validation on webhook payloads |
| No monitoring | Prometheus metrics + Grafana dashboard |
| Single instance n8n | Horizontal scaling with message queue |

---

## Key Concepts Demonstrated

- **Identity Governance & Administration (IGA)**
- **Segregation of Duties (SoD) enforcement**
- **Access Certification Campaigns**
- **Time-bound access & Auto revocation**
- **Audit trail & Compliance logging**
- **Keycloak REST API integration**
- **Workflow automation with n8n**
- **Zero Trust principles**

---

## Author

Nikunj — IAM & Identity Automation Engineer  
Self-taught | Keycloak | n8n | Microsoft Entra | Zero Trust  
[LinkedIn](www.linkedin.com/in/nikunj-a-3a5bb72ab) | [GitHub](https://github.com/nik-cybersec)
