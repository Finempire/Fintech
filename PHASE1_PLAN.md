# Comprehensive TODO list for building a Tally ERP automation SaaS

Building a multi-tenant SaaS for Tally automation in India requires integrating **three complex domains**: accounting automation (competing with Suvit at ₹8,999/year), statutory compliance (GST/TDS), and payroll processing. This guide provides a phased approach prioritizing MVP delivery within 4-6 months, with the full platform achievable in 12-18 months.

The recommended architecture uses **Django with schema-based multi-tenancy** (django-tenants), **PostgreSQL** for data isolation critical to financial applications, and a **desktop connector pattern** for Tally integration since TallyPrime lacks a native REST API—requiring XML over HTTP to localhost:9000.

---

## Phase 1: Foundation and MVP core (Months 1-4)

This phase establishes the multi-tenant infrastructure and delivers the first monetizable feature: bank statement to Tally voucher automation.

### 1.1 Multi-tenant infrastructure setup

| Task | Complexity | Priority |
|------|-----------|----------|
| Set up Django project with django-tenants (PostgreSQL schema-based) | Medium | Critical |
| Create Tenant and Domain models for subdomain routing | Low | Critical |
| Implement JWT authentication with djangorestframework-simplejwt | Low | Critical |
| Build user invitation and role assignment per tenant | Medium | High |
| Set up Celery + Redis for background task processing | Medium | Critical |
| Configure Docker Compose for local development environment | Low | High |

**Database schema considerations:**
```python
# Shared schema (public)
- Tenant (id, name, schema_name, created_on, paid_until)
- Domain (id, tenant_id, domain, is_primary)
- Plan (id, name, razorpay_plan_id, price, features)
- Subscription (id, tenant_id, plan_id, status, current_period_end)

# Tenant-specific schemas
- User (id, email, role, is_active)
- Company (id, name, gstin, pan, address, tally_config)
```

### 1.2 Tally integration layer (critical path)

| Task | Complexity | Priority |
|------|-----------|----------|
| Build Tally XML request/response parser library | High | Critical |
| Create XML templates for all voucher types (Payment, Receipt, Journal, Sales, Purchase) | Medium | Critical |
| Develop desktop connector application (Electron/Python) for Tally bridge | High | Critical |
| Implement connector ↔ SaaS polling mechanism with queue | High | Critical |
| Build ledger/master sync from Tally to SaaS | Medium | Critical |
| Add error handling with LINEERROR parsing and retry logic | Medium | High |

**Tally integration approach:**
The recommended architecture uses a **polling-based desktop connector** since TallyPrime only accepts connections on localhost:9000:

```
SaaS Cloud ←→ Desktop Connector (on Tally PC) ←→ TallyPrime (Port 9000)
     │              │                                    │
   REST API      Polls for          Sends XML         Returns XML
   Queue         pending ops        POST requests      responses
```

**Critical XML structure for voucher creation:**
```xml
<ENVELOPE>
  <HEADER>
    <TALLYREQUEST>Import</TALLYREQUEST>
    <TYPE>Data</TYPE>
    <ID>Vouchers</ID>
  </HEADER>
  <BODY>
    <DATA>
      <TALLYMESSAGE>
        <VOUCHER VCHTYPE="Payment" ACTION="Create">
          <DATE>20251201</DATE>
          <PERSISTEDVIEW>Accounting Voucher View</PERSISTEDVIEW>
          <!-- PERSISTEDVIEW is mandatory or import fails -->
          <LEDGERENTRIES.LIST>...</LEDGERENTRIES.LIST>
        </VOUCHER>
      </TALLYMESSAGE>
    </DATA>
  </BODY>
</ENVELOPE>
```

### 1.3 Bank statement parsing module (MVP feature)

| Task | Complexity | Priority |
|------|-----------|----------|
| Build PDF parser for bank statements (PyPDF2 + tabula-py) | High | Critical |
| Create Excel/CSV parser with column mapping | Low | Critical |
| Implement transaction extraction with date/amount/description parsing | Medium | Critical |
| Build AI-powered ledger suggestion engine (rule-based first, ML later) | High | High |
| Create bulk voucher generation from parsed transactions | Medium | Critical |
| Build ledger mapping UI with "Speedy Recommendations" like Vouchrit | Medium | High |
| Support 50+ Indian bank statement formats (SBI, HDFC, ICICI, Axis) | High | High |

**Database schema for bank parsing:**
```python
# Bank statement processing
- BankAccount (id, company_id, bank_name, account_number, ifsc)
- BankStatement (id, bank_account_id, file, upload_date, status, period_start, period_end)
- ParsedTransaction (id, statement_id, date, description, debit, credit, balance, suggested_ledger_id, confidence_score, status)
- LedgerMappingRule (id, company_id, pattern, ledger_id, priority)
```

### 1.4 Basic React dashboard

| Task | Complexity | Priority |
|------|-----------|----------|
| Set up React 18 + TypeScript + Vite project | Low | Critical |
| Implement Ant Design component library integration | Low | High |
| Build authentication flow (login, JWT storage, refresh) | Medium | Critical |
| Create file upload component for bank statements | Low | Critical |
| Build transaction review and approval interface | Medium | Critical |
| Implement ledger mapping drag-and-drop UI | Medium | High |
| Add Tally sync status dashboard | Low | High |

---

## Phase 2: GST compliance module (Months 4-7)

### 2.1 E-invoicing integration

| Task | Complexity | Priority |
|------|-----------|----------|
| Partner with GSP (ClearTax/Masters India) for API access | Medium | Critical |
| Implement OAuth 2.0 authentication with encrypted credential storage | Medium | Critical |
| Build e-invoice JSON generation per GST INV-1 schema (30 mandatory fields) | High | Critical |
| Integrate with IRP for IRN generation | High | Critical |
| Implement QR code generation and embedding | Low | High |
| Build auto-push to GSTR-1 tracking | Medium | High |
| Add e-invoice cancellation within 24-hour window | Low | High |

**E-invoice JSON schema essentials:**
```json
{
  "Version": "1.1",
  "TranDtls": {"TaxSch": "GST", "SupTyp": "B2B"},
  "DocDtls": {"Typ": "INV", "No": "INV001", "Dt": "03/12/2025"},
  "SellerDtls": {"Gstin": "29AAACM1234D1Z5", "LglNm": "..."},
  "BuyerDtls": {"Gstin": "...", "Pos": "29"},
  "ItemList": [{"HsnCd": "8471", "GstRt": 18, ...}],
  "ValDtls": {"TotInvVal": 11800}
}
```

### 2.2 GST return preparation

| Task | Complexity | Priority |
|------|-----------|----------|
| Build GSTR-1 data aggregation from sales vouchers | High | Critical |
| Implement B2B, B2C, CDN, HSN summary table generation | High | Critical |
| Create GSTR-3B liability computation engine | High | Critical |
| Build ITC calculation from GSTR-2B data | High | High |
| Add return filing submission via GSP API | Medium | High |
| Implement filing status tracking and notifications | Low | High |

### 2.3 GST reconciliation engine

| Task | Complexity | Priority |
|------|-----------|----------|
| Build GSTR-2B download and parsing from GSTN | Medium | Critical |
| Implement purchase register vs 2B matching algorithm | High | Critical |
| Create fuzzy matching for invoice number variations | Medium | High |
| Build mismatch categorization (exact, suggested, missing in 2B, missing in PR) | Medium | High |
| Calculate Rule 36(4) ITC eligibility (≤105% of 2B) | Medium | High |
| Add vendor compliance scoring dashboard | Medium | Medium |

**Reconciliation database schema:**
```python
- GSTR2BRecord (id, company_id, period, supplier_gstin, invoice_no, invoice_date, taxable_value, igst, cgst, sgst, status)
- ReconciliationResult (id, gstr2b_record_id, purchase_voucher_id, match_type, mismatch_reasons, action_taken)
```

### 2.4 E-way bill integration

| Task | Complexity | Priority |
|------|-----------|----------|
| Implement e-way bill generation API for consignments >₹50,000 | Medium | High |
| Build Part A auto-population from e-invoice | Low | High |
| Add Part B vehicle update functionality | Low | Medium |
| Implement validity tracking and extension | Low | Medium |
| Create consolidated e-way bill for multiple consignments | Medium | Medium |

---

## Phase 3: Invoice OCR and automation (Months 5-8)

### 3.1 Invoice digitization

| Task | Complexity | Priority |
|------|-----------|----------|
| Integrate OCR service (Google Vision API or AWS Textract) | Medium | High |
| Build invoice field extraction (vendor, date, amounts, line items, GST) | High | High |
| Implement GSTIN validation against GSTN master | Low | High |
| Create HSN code extraction and validation | Medium | High |
| Build confidence scoring for extracted data | Medium | Medium |
| Add manual correction interface with learning feedback loop | Medium | Medium |

### 3.2 Auto-voucher creation

| Task | Complexity | Priority |
|------|-----------|----------|
| Map extracted invoice data to Tally voucher XML | Medium | High |
| Implement automatic ledger suggestion from vendor history | Medium | High |
| Build batch processing for bulk invoice upload | Medium | High |
| Add duplicate invoice detection | Medium | High |
| Create approval workflow before Tally push | Medium | Medium |

---

## Phase 4: Payroll module (Months 7-10)

### 4.1 Employee and salary structure management

| Task | Complexity | Priority |
|------|-----------|----------|
| Build employee master with all statutory fields (PAN, Aadhaar, UAN, ESIC IP) | Medium | Critical |
| Create flexible salary structure templates (Basic 40-50%, HRA, DA, Special) | Medium | Critical |
| Implement CTC to take-home calculator | Medium | High |
| Build attendance and leave management integration | High | High |
| Add loan and advance tracking with EMI deduction | Medium | Medium |

**Payroll database schema:**
```python
- Employee (id, company_id, name, pan, aadhaar, uan, esic_ip, bank_account, date_of_joining, salary_structure_id)
- SalaryStructure (id, name, components_json)  # {basic: 40%, hra: 50% of basic, ...}
- SalaryComponent (id, name, type, calculation_type, taxability)
- Attendance (id, employee_id, date, status, hours_worked)
- Leave (id, employee_id, leave_type, start_date, end_date, status)
```

### 4.2 Statutory deduction calculations

| Task | Complexity | Priority |
|------|-----------|----------|
| Implement PF calculation (12% employee, 13.61% employer including EPS/EDLI) | Medium | Critical |
| Build ESI calculation (0.75% employee, 3.25% employer for salary ≤₹21,000) | Low | Critical |
| Create state-wise Professional Tax engine (Maharashtra, Karnataka, Gujarat, etc.) | Medium | Critical |
| Implement TDS computation with old/new regime support | High | Critical |
| Calculate gratuity provision (4.81% of Basic) | Low | High |
| Build bonus calculation per Payment of Bonus Act (8.33%-20%) | Medium | High |

**Key formulas to implement:**
```python
# PF Calculation (capped at ₹15,000 PF wage)
employee_pf = min(basic + da, 15000) * 0.12
employer_eps = min(basic + da, 15000) * 0.0833
employer_epf = employee_pf - employer_eps
edli = min(basic + da, 15000) * 0.005

# ESI (if gross ≤ ₹21,000)
employee_esi = gross_salary * 0.0075
employer_esi = gross_salary * 0.0325

# Net Salary
net = gross - employee_pf - employee_esi - pt - tds
```

### 4.3 Payroll processing and Tally export

| Task | Complexity | Priority |
|------|-----------|----------|
| Build monthly payroll processing workflow | High | Critical |
| Generate salary slips in PDF format | Medium | High |
| Create Tally payroll voucher XML export | Medium | Critical |
| Build PF ECR file generation for EPFO portal | Medium | High |
| Generate ESI contribution challan data | Medium | High |
| Create Form 24Q quarterly TDS return data | High | High |
| Build Form 16 generation | High | High |

### 4.4 Statutory filing integrations

| Task | Complexity | Priority |
|------|-----------|----------|
| Integrate EPFO API for ECR filing (if API available) | High | Medium |
| Build ESIC portal data export | Medium | Medium |
| Create state PT return data generation | Medium | Medium |
| Implement TRACES integration for Form 24Q | High | Medium |

---

## Phase 5: Advanced features and scale (Months 10-14)

### 5.1 Practice management (for CA firms)

| Task | Complexity | Priority |
|------|-----------|----------|
| Build client management dashboard | Medium | Medium |
| Implement WhatsApp document collection (like Suvit Chat) | High | Medium |
| Add automatic follow-up reminders | Medium | Medium |
| Create team member assignment and tracking | Medium | Medium |
| Build client portal for document upload | Medium | Medium |

### 5.2 Advanced automation

| Task | Complexity | Priority |
|------|-----------|----------|
| Implement ML-based ledger prediction (train on user corrections) | High | Medium |
| Build anomaly detection for fraud prevention | High | Low |
| Add predictive cash flow forecasting (like Vouchrit) | High | Low |
| Create natural language query interface for reports | High | Low |
| Implement e-commerce integrations (Amazon, Meesho, Flipkart) | High | Medium |

### 5.3 Mobile application

| Task | Complexity | Priority |
|------|-----------|----------|
| Build React Native mobile app for invoice capture | High | Medium |
| Implement camera-based document scanning | Medium | Medium |
| Add approval workflow on mobile | Medium | Medium |
| Create push notifications for compliance deadlines | Low | Medium |

---

## API endpoints structure

### Authentication and tenant management
```
POST   /api/auth/login/              # JWT token obtain
POST   /api/auth/refresh/            # JWT token refresh
POST   /api/auth/register/           # New tenant registration
GET    /api/tenant/profile/          # Current tenant info
PUT    /api/tenant/settings/         # Update settings
```

### Company and masters
```
GET    /api/v1/companies/            # List companies
POST   /api/v1/companies/            # Create company
GET    /api/v1/ledgers/              # Chart of accounts
POST   /api/v1/ledgers/              # Create ledger
POST   /api/v1/ledgers/sync-tally/   # Sync from Tally
```

### Bank statement processing
```
POST   /api/v1/bank-statements/upload/           # Upload statement
GET    /api/v1/bank-statements/{id}/transactions/ # Get parsed transactions
POST   /api/v1/bank-statements/{id}/map-ledgers/  # Apply ledger mapping
POST   /api/v1/bank-statements/{id}/generate-vouchers/ # Create vouchers
```

### Voucher management
```
GET    /api/v1/vouchers/             # List vouchers
POST   /api/v1/vouchers/             # Create voucher
POST   /api/v1/vouchers/bulk/        # Bulk create
POST   /api/v1/vouchers/{id}/push-tally/ # Push to Tally
GET    /api/v1/vouchers/sync-status/ # Tally sync status
```

### GST compliance
```
GET    /api/v1/gst/gstr1/            # GSTR-1 draft
POST   /api/v1/gst/gstr1/file/       # File GSTR-1
GET    /api/v1/gst/gstr3b/           # GSTR-3B computation
GET    /api/v1/gst/gstr2b/           # Download GSTR-2B
POST   /api/v1/gst/reconcile/        # Run reconciliation
POST   /api/v1/gst/einvoice/generate/ # Generate e-invoice
POST   /api/v1/gst/ewaybill/generate/ # Generate e-way bill
```

### Payroll
```
GET    /api/v1/employees/            # List employees
POST   /api/v1/employees/            # Create employee
GET    /api/v1/payroll/run/{month}/  # Get payroll for month
POST   /api/v1/payroll/process/      # Process payroll
GET    /api/v1/payroll/{id}/slip/    # Download salary slip
POST   /api/v1/payroll/{id}/export-tally/ # Export to Tally
```

---

## Third-party integrations required

| Integration | Purpose | Phase | Complexity |
|------------|---------|-------|------------|
| **Razorpay** | Subscription billing | 1 | Low |
| **AWS S3 / DigitalOcean Spaces** | File storage | 1 | Low |
| **SendGrid / AWS SES** | Email notifications | 1 | Low |
| **Google Vision API / AWS Textract** | Invoice OCR | 3 | Medium |
| **GSP Partner (ClearTax/Masters India)** | GSTN API access | 2 | High |
| **WhatsApp Business API** | Document collection | 5 | High |
| **Sentry** | Error monitoring | 1 | Low |
| **Twilio** | OTP verification | 1 | Low |

---

## Deployment infrastructure

**Recommended stack for launch:**
- **DigitalOcean** for simpler operations and predictable costs
- **Managed PostgreSQL** ($15/month starting)
- **App Platform** or Kubernetes cluster
- **Spaces** for file storage (S3-compatible)
- **Managed Redis** for caching and Celery broker

**Production architecture:**
```
Load Balancer
    ↓
Django App (3 replicas) ←→ PostgreSQL (multi-tenant schemas)
    ↓                          ↓
Celery Workers ←→ Redis (broker + cache)
    ↓
Celery Beat (scheduled tasks: GST sync, report generation)
```

---

## MVP vs future features summary

### MVP (Months 1-4) — Target launch features
- Multi-tenant infrastructure with subdomain routing
- Bank statement parsing (PDF/Excel) for 20+ major banks
- AI-powered ledger suggestion with 80%+ accuracy
- Tally desktop connector with voucher push
- Basic React dashboard with file upload and review
- Razorpay subscription integration

### Post-MVP Phase 1 (Months 4-7) — GST compliance
- E-invoicing with IRN generation
- GSTR-1 and GSTR-3B preparation
- GSTR-2B reconciliation
- E-way bill generation

### Post-MVP Phase 2 (Months 7-10) — Payroll
- Employee management with statutory fields
- PF, ESI, PT, TDS calculations
- Payroll processing and salary slips
- Tally payroll voucher export

### Future features (Months 10+)
- Invoice OCR with ML enhancement
- WhatsApp document collection
- Mobile application
- E-commerce integrations
- Practice management for CA firms
- Predictive analytics and cash flow forecasting

---

## Competitive pricing recommendation

Based on Suvit's ₹8,999/year unlimited plan and market analysis:

| Plan | Price/Year | Features |
|------|-----------|----------|
| **Starter** | ₹4,999 | 1 company, 500 transactions/month, Bank parsing, Tally sync |
| **Professional** | ₹8,999 | 3 companies, Unlimited transactions, GST compliance, E-invoicing |
| **Business** | ₹14,999 | 10 companies, Payroll module, Priority support |
| **Enterprise** | Custom | Unlimited companies, Dedicated support, API access, On-premise option |

This phased approach enables launching a monetizable MVP within 4 months while building toward a comprehensive Tally automation platform that addresses the key gaps in existing solutions: **transparent pricing**, **mobile experience**, **broader bank format support**, and **superior customer support** — areas where Suvit and Vouchrit receive negative feedback.
