# Loan Amortization Report Upload — Feature Design

**Feature:** RM/FP uploads a bank-issued amortization PDF → system auto-extracts loan details and pre-fills the loan entry for the client.

**Replaces:** Manual data entry for `loan_type`, `loan_amount`, `rate_of_interest`, `tenure`, `emi_amount`, `balance_amount`.

---

## Sample Document Analysed

File: `L1XXXXXXXXXXXX26.pdf` — ICICI Bank Education Loan amortization schedule

The PDF has two logical sections:

### Section 1 — Header Letter (Page 1)

Plain text key-value pairs. Example from the ICICI format:

```
Loan Account Number (LAN)         : L1DEL00100037026
Principal amount of the facility  : ₹ 500,000.00
Tenure                            : 126 months
Equated Monthly Instalment (EMI)  : As per amortization schedule
EMI Date                          : 5th of every month
Rate of Interest                  : 11.55 % p.a.
Product Type                      : Education Loan
```

### Section 2 — Amortization Schedule Table (Page 3 onwards)

A table with one row per installment:

| Instl No. | Installment Date | Instl. Amount (₹) | Principal (₹) | Interest (₹) | Closing Principal (₹) |
|---|---|---|---|---|---|
| 1 | 05-Sep-2023 | 3,494 | 0.00 | 3,494 | 330,000 |
| … | … | … | … | … | … |
| 31 | 05-Mar-2026 | 5,283 | 2,107 | 3,176 | 327,893 |
| 34 | 05-Jun-2026 | 5,283 | 2,168 | 3,115 | 321,450 |

> Note: This loan has a moratorium period (rows 1–30) where principal = 0 and only interest is charged. The standard EMI kicks in from row 31 (₹5,283).

---

## What Can Be Extracted

| Field | Mapped to `client_loans` column | Source in PDF |
|---|---|---|
| Loan type | `loan_type` | "Product Type" in header text |
| Original principal | `loan_amount` | "Principal amount of the facility" in header |
| Rate of interest | `rate_of_interest` | "Rate of Interest" in header |
| Full tenure | — (reference only) | "Tenure (Months)" in amortization header |
| EMI amount | `emi_amount` | First row in schedule where `Principal > 0` → `Instl. Amount` |
| Current outstanding balance | `balance_amount` | Find the row whose `Installment Date` ≤ today → `Closing Principal` |
| Remaining tenure (months) | `tenure_outstanding` | Total tenure − installments already paid |
| Loan account number | — (for display/reference) | LAN from header |
| Start date | — (reference) | "Start Date" in amortization header |

### How current outstanding balance is derived

```
today = current date

find the last row where Installment Date <= today
→ outstanding_balance = that row's "Closing Principal"

remaining_months = total_tenure - installment_number_of_that_row
```

**Example (as of June 15, 2026):**
- Last paid installment = row 34 (05-Jun-2026)
- `balance_amount` = ₹3,21,450
- `tenure_outstanding` = 126 − 34 = **92 months remaining**

---

## Extraction Approach (Backend)

### Endpoint

```
POST /api/v1/clients/{client_id}/loans/extract-from-pdf
Content-Type: multipart/form-data
file: <amortization_pdf>
```

Returns extracted fields as a preview (no DB write). RM reviews and confirms before saving.

### Parser Steps

1. **Open PDF with `pdfplumber`** (already a dependency in the project)
2. **Extract header fields** — scan all pages for key-value patterns:
   ```
   "Principal amount.*?:\s*₹?\s*([\d,]+)"
   "Rate of Interest.*?:\s*([\d.]+)"
   "Tenure.*?:\s*(\d+)"
   "Product Type.*?:\s*(.+)"
   ```
3. **Find the amortization table** — look for a table whose first row contains headers: `Instl No.`, `Installment Date`, `Principal`, `Closing Principal`
4. **Find current row** — iterate rows, parse `Installment Date` (format: `DD-MMM-YYYY`), find last row ≤ today
5. **Derive EMI** — first row where `Principal (₹) > 0` gives the true post-moratorium EMI amount
6. **Return preview** — all extracted fields for RM to review before confirming

### Known Variations to Handle

| Bank | Difference |
|---|---|
| ICICI | Moratorium rows (Principal = 0 for initial months); two EMI amounts in schedule |
| HDFC | Different key names ("Loan Amount" vs "Principal amount of the facility") |
| SBI | Schedule may span multiple tables across pages |

Parser should treat all extracted values as **best-effort** — RM can correct any field before saving.

---

## Fields Not Extractable from This PDF

These must still be entered manually by the RM:

- `loan_type` — needs normalisation (e.g. "Education Loan" → `education`). Map known strings; fall back to `other`.
- Any prepayment or top-up history

---

## Flow

```
RM uploads amortization PDF
        ↓
POST /loans/extract-from-pdf
        ↓
Parser extracts: loan_type, loan_amount, rate, EMI, balance, tenure_outstanding
        ↓
Return preview to FE (no DB write)
        ↓
RM reviews pre-filled form, corrects if needed
        ↓
RM confirms → POST /onboarding/:id/loans  (existing endpoint, no change)
```

---

## Files to Create / Modify

| File | Change |
|---|---|
| `app/core/loan_pdf_extractor.py` | New — PDF parsing logic |
| `app/api/v1/routes/clients.py` | New endpoint `POST /{client_id}/loans/extract-from-pdf` |
| `app/schemas/client.py` | New response schema `LoanExtractPreview` |
