# Loan Amortization PDF Upload — Financial Dashboard Liabilities

**Feature:** On the Financial Dashboard → Liabilities tab, RM/FP uploads a bank-issued amortization PDF for a client's loan. The system auto-extracts the loan details and saves them directly to the client's loan record — no manual entry needed.

**Trigger:** Liabilities page — per-loan "Upload Amortization" button.

---

## Sample Document Analysed

File: `L1XXXXXXXXXXXX26.pdf` — ICICI Bank Education Loan amortization schedule

The PDF has two logical sections:

### Section 1 — Header Letter (Page 1)

Plain text key-value pairs:

```
Loan Account Number (LAN)         : L1DEL00100037026
Principal amount of the facility  : ₹ 500,000.00
Tenure                            : 126 months
Rate of Interest                  : 11.55 % p.a.
Product Type                      : Education Loan
Start Date                        : 05-Sep-2023
```

### Section 2 — Amortization Schedule Table (Page 3+)

| Instl No. | Installment Date | Instl. Amount (₹) | Principal (₹) | Interest (₹) | Closing Principal (₹) |
|---|---|---|---|---|---|
| 1 | 05-Sep-2023 | 3,494 | 0.00 | 3,494 | 3,30,000 |
| … | | | | | |
| 31 | 05-Mar-2026 | 5,283 | 2,107 | 3,176 | 3,27,893 |
| 34 | 05-Jun-2026 | 5,283 | 2,168 | 3,115 | 3,21,450 |

> This loan has a moratorium period (rows 1–30) where Principal = ₹0 (interest-only). The real EMI starts from row 31 (₹5,283).

---

## Field Mapping — PDF → `client_loans` table

| Extracted from PDF | `client_loans` column | Example value |
|---|---|---|
| "Principal amount of the facility" | `loan_amount` | ₹5,00,000 |
| "Rate of Interest" | `rate_of_interest` | 11.55 |
| "Product Type" (normalised) | `loan_type` | `education` |
| First row where Principal > 0 → Instl. Amount | `emi_amount` | ₹5,283 |
| Last row where date ≤ today → Closing Principal | `balance_amount` | ₹3,21,450 |
| Total tenure − installments paid so far | `tenure_outstanding` | 92 months |

### How `balance_amount` and `tenure_outstanding` are derived

```
today = current date (e.g. June 15, 2026)

scan amortization rows → find last row where Installment Date ≤ today
→ balance_amount     = that row's "Closing Principal"
→ tenure_outstanding = total_tenure_months − that row's "Instl No."
```

**Example (June 15, 2026):**
- Last paid row = row 34 (05-Jun-2026), Closing Principal = ₹3,21,450
- `balance_amount` = ₹3,21,450
- `tenure_outstanding` = 126 − 34 = **92 months**

### Loan type normalisation

The PDF gives a free-text product name. Map it to the system's `loan_type` values:

| PDF "Product Type" | `loan_type` stored |
|---|---|
| Education Loan | `education` |
| Home Loan / Housing Loan | `home` |
| Car Loan / Auto Loan | `vehicle` |
| Personal Loan | `personal` |
| Anything else | `other` |

---

## API Design

### New Endpoint — Extract & Save

```
POST /api/v1/clients/{client_id}/loans/upload-amortization
Content-Type: multipart/form-data
field: file  (PDF)
```

**Two-step flow — extract first, confirm to save:**

#### Step 1 — Extract preview (no DB write)

```
POST /api/v1/clients/{client_id}/loans/parse-amortization
→ returns LoanExtractPreview (all extracted fields)
```

FE displays the pre-filled loan form. RM can correct any field.

#### Step 2 — Save (existing endpoint, no change needed)

```
POST /api/v1/clients/{client_id}/loans           ← add new loan
PUT  /api/v1/clients/{client_id}/loans/{loan_id} ← update existing loan
```

RM confirms → save using the existing `CreateLoanRequest` / `UpdateLoanRequest` schema. No new save endpoint needed.

### Preview Response Schema (`LoanExtractPreview`)

```python
class LoanExtractPreview(BaseModel):
    loan_type: str | None          # normalised value e.g. "education"
    loan_amount: Decimal | None    # original principal
    rate_of_interest: Decimal | None
    emi_amount: Decimal | None     # post-moratorium EMI
    balance_amount: Decimal | None # current outstanding as of today
    tenure_outstanding: int | None # remaining months
    # reference fields shown to RM but not saved
    lan_number: str | None         # Loan Account Number from PDF
    bank_name: str | None          # inferred from PDF metadata / filename
    confidence: str                # "high" | "low" — low if any field could not be parsed
```

---

## Where It Fits in the Liabilities Page

The liabilities tab already fetches and displays loans from `client_loans`. This feature adds a new way to **populate** a loan record — instead of the RM typing in the values, they upload the PDF.

**Current loan display fields on liabilities page:**

| Dashboard field | `client_loans` column | Populated by PDF? |
|---|---|---|
| Loan type | `loan_type` | Yes |
| Original loan amount | `loan_amount` | Yes |
| EMI | `emi_amount` | Yes |
| Rate of interest | `rate_of_interest` | Yes |
| Outstanding balance | `balance_amount` | Yes — auto-calculated to today |
| Remaining tenure | `tenure_outstanding` | Yes — auto-calculated to today |
| Balance override | `balance_override` | No — planner sets manually |

After upload and confirm, the liabilities tab reloads and shows the loan with the amortization analysis (prepay vs invest, annual amortization schedule) already populated — no extra steps.

---

## Files to Create / Modify

| File | Change |
|---|---|
| `app/core/loan_pdf_extractor.py` | **New** — PDF parsing logic using `pdfplumber` |
| `app/schemas/client.py` | **New schema** — `LoanExtractPreview` |
| `app/api/v1/routes/clients.py` | **New endpoint** — `POST /{client_id}/loans/parse-amortization` |

No DB changes needed — uses existing `client_loans` table and existing save endpoints.

---

## Parser Logic (for `loan_pdf_extractor.py`)

```
1. Open PDF with pdfplumber

2. Extract header fields — scan all pages for patterns:
   - "Principal amount.*?₹\s*([\d,]+)"     → loan_amount
   - "Rate of Interest.*?([\d.]+)"          → rate_of_interest
   - "Tenure.*?(\d+)\s*months"              → total_tenure
   - "Product Type\s*:\s*(.+)"             → loan_type (raw)
   - "LAN.*?:\s*(\S+)"                     → lan_number

3. Find amortization table — table where headers include
   "Instl No." and "Closing Principal"

4. Parse rows:
   - Find first row where Principal > 0  → emi_amount
   - Find last row where date ≤ today    → balance_amount, installments_paid
   - tenure_outstanding = total_tenure - installments_paid

5. Normalise loan_type string → system enum value

6. Return LoanExtractPreview
```

### Known bank format variations

| Bank | Known difference |
|---|---|
| ICICI | Moratorium rows (Principal = 0); two separate EMI amounts in schedule |
| HDFC | Key label "Loan Amount" instead of "Principal amount of the facility" |
| SBI | Schedule may span multiple separate tables across pages |
| Axis | Header fields may be in a table rather than plain text |

Parser should treat all values as best-effort — `confidence: "low"` if any required field could not be parsed. RM can always correct before saving.

---

## Flow Diagram

```
RM opens Liabilities tab
        ↓
Clicks "Upload Amortization PDF" on a loan row (or on Add Loan)
        ↓
FE sends PDF → POST /loans/parse-amortization
        ↓
Backend extracts: loan_type, loan_amount, rate, EMI, balance, tenure
        ↓
FE shows pre-filled loan form (RM can edit any field)
        ↓
RM confirms → POST /loans  or  PUT /loans/{id}  (existing endpoints)
        ↓
Liabilities tab reloads — loan appears with full amortization analysis
```
