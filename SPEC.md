**Database design (SQLite)**



CREATE TABLE companies (
  id INTEGER PRIMARY KEY,
  code TEXT NOT NULL UNIQUE,          -- AKHI-EP, AKHI-PP
  name TEXT NOT NULL,
  address TEXT, phone TEXT, email TEXT,
  tax_number TEXT,
  logo_path TEXT,
  doc_template TEXT NOT NULL DEFAULT 'default',
  is_active INTEGER NOT NULL DEFAULT 1
);

CREATE TABLE parties (
  id INTEGER PRIMARY KEY,
  type TEXT NOT NULL CHECK (type IN ('customer','vendor')),
  code TEXT NOT NULL UNIQUE,
  name TEXT NOT NULL,
  contact_person TEXT, designation TEXT,
  phone TEXT, email TEXT,
  address1 TEXT, address2 TEXT,
  is_active INTEGER NOT NULL DEFAULT 1
);

CREATE TABLE payment_methods (
  id INTEGER PRIMARY KEY,
  name TEXT NOT NULL UNIQUE,           -- Cash, Cheque, Bank Transfer, bKash, Nagad, Rocket, Card
  is_active INTEGER NOT NULL DEFAULT 1
);

CREATE TABLE number_sequences (
  company_id INTEGER NOT NULL,
  doc_type TEXT NOT NULL,              -- QTN, CH, INV, PAY, BILL
  seq_date TEXT NOT NULL,              -- YYYY-MM-DD
  last_number INTEGER NOT NULL DEFAULT 0,
  PRIMARY KEY (company_id, doc_type, seq_date)
);

CREATE TABLE settings (key TEXT PRIMARY KEY, value TEXT);

CREATE TABLE quotations (
  id INTEGER PRIMARY KEY,
  number TEXT NOT NULL UNIQUE,
  company_id INTEGER NOT NULL REFERENCES companies(id),
  party_id INTEGER NOT NULL REFERENCES parties(id),
  quote_date TEXT NOT NULL,
  valid_until TEXT,
  reference TEXT, subject TEXT,
  discount INTEGER NOT NULL DEFAULT 0,       -- paisa
  vat_rate_bp INTEGER NOT NULL DEFAULT 0,    -- basis points: 15% = 1500
  terms TEXT,
  status TEXT NOT NULL DEFAULT 'draft'
    CHECK (status IN ('draft','sent','accepted','rejected','cancelled')),
  converted_invoice_id INTEGER REFERENCES invoices(id),
  created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE TABLE quotation_items (
  id INTEGER PRIMARY KEY,
  quotation_id INTEGER NOT NULL REFERENCES quotations(id) ON DELETE CASCADE,
  sort_order INTEGER NOT NULL DEFAULT 0,
  description TEXT NOT NULL,
  unit TEXT,
  qty_milli INTEGER NOT NULL,                -- quantity x 1000 (2.5 = 2500)
  rate INTEGER NOT NULL                      -- paisa
);

CREATE TABLE challans (
  id INTEGER PRIMARY KEY,
  number TEXT NOT NULL UNIQUE,
  company_id INTEGER NOT NULL REFERENCES companies(id),
  party_id INTEGER NOT NULL REFERENCES parties(id),
  challan_date TEXT NOT NULL,
  reference TEXT, subject TEXT, remarks TEXT,
  status TEXT NOT NULL DEFAULT 'issued' CHECK (status IN ('draft','issued','cancelled')),
  created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE TABLE challan_items (
  id INTEGER PRIMARY KEY,
  challan_id INTEGER NOT NULL REFERENCES challans(id) ON DELETE CASCADE,
  sort_order INTEGER NOT NULL DEFAULT 0,
  description TEXT NOT NULL,
  unit TEXT,
  qty_milli INTEGER NOT NULL
);

CREATE TABLE invoices (
  id INTEGER PRIMARY KEY,
  number TEXT NOT NULL UNIQUE,
  company_id INTEGER NOT NULL REFERENCES companies(id),
  party_id INTEGER NOT NULL REFERENCES parties(id),
  quotation_id INTEGER REFERENCES quotations(id),
  invoice_date TEXT NOT NULL,
  due_date TEXT,
  reference TEXT, subject TEXT, remarks TEXT,
  discount INTEGER NOT NULL DEFAULT 0,
  vat_rate_bp INTEGER NOT NULL DEFAULT 0,
  advance INTEGER NOT NULL DEFAULT 0,
  status TEXT NOT NULL DEFAULT 'draft' CHECK (status IN ('draft','issued','cancelled')),
  created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE TABLE invoice_items (
  id INTEGER PRIMARY KEY,
  invoice_id INTEGER NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
  sort_order INTEGER NOT NULL DEFAULT 0,
  description TEXT NOT NULL,
  unit TEXT,
  qty_milli INTEGER NOT NULL,
  rate INTEGER NOT NULL
);

CREATE TABLE vendor_bills (
  id INTEGER PRIMARY KEY,
  company_id INTEGER NOT NULL REFERENCES companies(id),
  party_id INTEGER NOT NULL REFERENCES parties(id),
  bill_number TEXT NOT NULL,                 -- vendor's own number, free text
  bill_date TEXT NOT NULL,
  due_date TEXT,
  subject TEXT,
  amount INTEGER NOT NULL,
  status TEXT NOT NULL DEFAULT 'open' CHECK (status IN ('open','cancelled')),
  UNIQUE (company_id, party_id, bill_number)
);

CREATE TABLE ledger_entries (
  id INTEGER PRIMARY KEY,
  company_id INTEGER NOT NULL REFERENCES companies(id),
  party_id INTEGER NOT NULL REFERENCES parties(id),
  entry_date TEXT NOT NULL,
  entry_type TEXT NOT NULL
    CHECK (entry_type IN ('invoice','vendor_bill','payment','opening','adjustment')),
  ref_no TEXT,
  description TEXT,
  debit INTEGER NOT NULL DEFAULT 0,          -- paisa
  credit INTEGER NOT NULL DEFAULT 0,         -- paisa
  payment_method_id INTEGER REFERENCES payment_methods(id),
  invoice_id INTEGER REFERENCES invoices(id),    -- optional: payment tied to an invoice
  vendor_bill_id INTEGER REFERENCES vendor_bills(id),
  is_void INTEGER NOT NULL DEFAULT 0,
  created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
  CHECK (debit >= 0 AND credit >= 0 AND NOT (debit > 0 AND credit > 0))
);
CREATE INDEX idx_ledger_party_date ON ledger_entries(party_id, entry_date);
CREATE INDEX idx_invoices_party ON invoices(party_id);



**How the logic works**



line_amount   = round(qty_milli x rate / 1000)
subtotal      = sum(line_amount)
taxable       = subtotal - discount
vat_amount    = round(taxable x vat_rate_bp / 10000)
grand_total   = taxable + vat_amount
current_due   = grand_total - advance

Ledger sign convention.
Party
Increases balance
Decreases balance
Balance means
Customer
Invoice, opening, debit adjustment (debit)
Payment, credit adjustment (credit)
They owe you = debits minus credits
Vendor
Vendor bill, opening (credit)
Payment (debit)
You owe them = credits minus debits
Creating an invoice (one transaction): get next number, insert invoice, insert items, insert one ledger_entries row (debit = current_due, or the grand total if you record the advance as a separate payment row; decide once and keep it consistent). If anything fails, nothing is saved.
Recording a payment: insert a ledger_entries row with entry_type='payment', credit = amount, optional invoice_id.
Invoice status (derived, never stored):
paid_total   = sum of non-void payments linked to the invoice
outstanding  = grand_total - advance - paid_total
outstanding <= 0                     -> PAID
paid_total > 0 and outstanding > 0   -> PARTIALLY PAID
outstanding > 0 and today > due_date -> OVERDUE
otherwise                            -> ISSUED
Cancelling an invoice: set status='cancelled' and set is_void=1 on its ledger rows. Never delete.
Statement: all non-void ledger rows for one party in a date range. Opening balance = everything before the start date. The running balance is calculated while displaying.
Numbering: PREFIX-YYMMDD-NNN, e.g. INV-260918-001, with the counter in number_sequences per company, per type, per day. Increment inside the same transaction as the save.
Simplification to remember: one payment can point to one invoice (or none, as an on-account payment). You can add multi-invoice allocation later if needed.


**A4 printing rules**


A4 printing
- Page size: A4 portrait, 210mm x 297mm. Use webContents.printToPDF with
  pageSize: 'A4', printBackground: true, preferCSSPageSize: true.
- CSS: @page { size: A4 portrait; margin: 12mm 14mm; }
  Use mm and pt units for print layout, not px or vh.
- Use black text on white. Keep colours light; office printers waste ink on big fills.
- Header block: company logo, name, address, phone, email, tax number.
- Quotation title, number, date, valid-until, customer name and address, reference, subject.
- Items table: columns SL, Description, Unit, Qty, Rate, Amount. Right-align numbers.
- Table header row repeats on each page (thead { display: table-header-group; }).
- Rows never split across pages (tr { page-break-inside: avoid; }).
- Totals block (subtotal, discount, VAT, grand total, amount in words) and terms
  must stay together (break-inside: avoid) and never be orphaned on a page alone.
- Footer with "Page X of Y" and company contact line.
- Signature area at the bottom of the last page: "Prepared by" and "Authorised signature".
- Fonts bundled locally in the app (no Google Fonts). Include a Bengali-capable font
  (for example Noto Sans Bengali) if Bangla text is used.
- Logo is read from the local file and embedded as base64 in the HTML.
- Amount in words: Taka and Paisa, e.g. "Taka Twelve Thousand Five Hundred Only".
- Long documents (30+ items) must paginate correctly. Test with 5, 25 and 60 items.