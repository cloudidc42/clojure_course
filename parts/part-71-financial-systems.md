# Part 71: Financial Systems
## ขั้นตอนที่ 2101-2130: Double-Entry Bookkeeping, Ledger, ACID Transactions, Currency Handling

---

## บทนำ

Financial systems ต้องการความแม่นยำสูง:
- **Double-entry bookkeeping** - ทุก debit ต้องมี credit
- **Immutable ledger** - ไม่แก้ไข ต่อท้ายอย่างเดียว
- **Currency precision** - BigDecimal ไม่ใช่ float
- **Reconciliation** - ตรวจสอบยอดคงเหลือ
- **Audit trail** - บันทึกทุก transaction

---

## ขั้นตอนที่ 2101: Currency และ Money Types

```clojure
(ns myapp.financial.money)

;; NEVER use float/double for money - use BigDecimal
(defrecord Money [amount currency])

(defn money [amount currency]
  (->Money (bigdec amount) (keyword currency)))

(defn money+ [m1 m2]
  (assert (= (:currency m1) (:currency m2)) "Currency mismatch")
  (->Money (+ (:amount m1) (:amount m2)) (:currency m1)))

(defn money- [m1 m2]
  (assert (= (:currency m1) (:currency m2)) "Currency mismatch")
  (->Money (- (:amount m1) (:amount m2)) (:currency m1)))

(defn money* [m factor]
  (->Money (* (:amount m) (bigdec factor)) (:currency m)))

(defn money-zero? [m]
  (zero? (:amount m)))

(defn money-positive? [m]
  (pos? (:amount m)))

;; Format money
(defn format-money [m]
  (let [fmt (java.text.NumberFormat/getCurrencyInstance)
        currency (java.util.Currency/getInstance (name (:currency m)))]
    (.setCurrency fmt currency)
    (.format fmt (:amount m))))

;; Parse from string
(defn parse-money [s currency]
  (->Money (bigdec (clojure.string/replace s #"[^0-9.]" ""))
            (keyword currency)))

;; Round to currency precision
(defn round-money [m]
  (let [currency (java.util.Currency/getInstance (name (:currency m)))
        scale    (.getDefaultFractionDigits currency)]
    (->Money (.setScale (:amount m) scale java.math.RoundingMode/HALF_UP)
              (:currency m))))

;; Currency conversion (needs live rates in production)
(defn convert-currency [m target-currency rates]
  (let [rate (get rates [(:currency m) target-currency])]
    (when-not rate
      (throw (ex-info "No exchange rate available"
                       {:from (:currency m) :to target-currency})))
    (round-money (->Money (* (:amount m) (bigdec rate)) target-currency))))
```

---

## ขั้นตอนที่ 2102: Double-Entry Bookkeeping

```clojure
(ns myapp.financial.ledger
  (:require [next.jdbc :as jdbc]))

;; Chart of Accounts
;; 1xxx = Assets
;; 2xxx = Liabilities
;; 3xxx = Equity
;; 4xxx = Revenue
;; 5xxx = Expenses

(defn create-accounts-table! [db]
  (jdbc/execute! db ["
    CREATE TABLE IF NOT EXISTS accounts (
      id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
      code        VARCHAR(10) UNIQUE NOT NULL,
      name        VARCHAR(255) NOT NULL,
      type        VARCHAR(20) NOT NULL,  -- asset|liability|equity|revenue|expense
      currency    VARCHAR(3)  NOT NULL DEFAULT 'USD',
      parent_code VARCHAR(10) REFERENCES accounts(code),
      created_at  TIMESTAMPTZ DEFAULT NOW()
    )"]))

(defn create-journal-table! [db]
  (jdbc/execute! db ["
    CREATE TABLE IF NOT EXISTS journal_entries (
      id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
      reference       VARCHAR(100) NOT NULL,
      description     TEXT NOT NULL,
      transaction_at  TIMESTAMPTZ NOT NULL,
      created_at      TIMESTAMPTZ DEFAULT NOW(),
      created_by      UUID NOT NULL,
      metadata        JSONB
    )"]))

(defn create-journal-lines-table! [db]
  (jdbc/execute! db ["
    CREATE TABLE IF NOT EXISTS journal_lines (
      id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
      journal_entry_id UUID NOT NULL REFERENCES journal_entries(id),
      account_code    VARCHAR(10) NOT NULL REFERENCES accounts(code),
      debit           NUMERIC(20, 4) NOT NULL DEFAULT 0,
      credit          NUMERIC(20, 4) NOT NULL DEFAULT 0,
      currency        VARCHAR(3) NOT NULL DEFAULT 'USD',
      description     TEXT,
      CHECK (debit >= 0 AND credit >= 0),
      CHECK (debit = 0 OR credit = 0)  -- can't be both
    )"]))

;; Double-entry: debits MUST equal credits
(defn post-journal-entry! [db entry]
  (let [lines      (:lines entry)
        total-debit  (reduce + (map :debit lines))
        total-credit (reduce + (map :credit lines))]
    
    ;; Validate: balanced entry
    (when (not= total-debit total-credit)
      (throw (ex-info "Journal entry not balanced"
                       {:debits total-debit :credits total-credit})))
    
    (jdbc/with-transaction [tx db]
      (let [entry-id (-> (jdbc/execute-one! tx
                           ["INSERT INTO journal_entries
                             (reference, description, transaction_at, created_by, metadata)
                             VALUES (?, ?, ?, ?, ?::jsonb)
                             RETURNING id"
                            (:reference entry)
                            (:description entry)
                            (:transaction-at entry)
                            (:created-by entry)
                            (json/generate-string (dissoc entry :lines))]
                           jdbc/unqualified-snake-kebab-column-names)
                         :id)]
        
        (doseq [line lines]
          (jdbc/execute-one! tx
            ["INSERT INTO journal_lines
              (journal_entry_id, account_code, debit, credit, currency, description)
              VALUES (?, ?, ?, ?, ?, ?)"
             entry-id
             (:account line)
             (or (:debit line) 0)
             (or (:credit line) 0)
             (or (:currency line) "USD")
             (:description line)]))
        
        entry-id))))
```

---

## ขั้นตอนที่ 2103: Account Balance Queries

```clojure
(defn get-account-balance
  "Calculate balance for account code (and its children)"
  [db account-code as-of-date]
  (let [result (jdbc/execute-one! db
                 ["WITH RECURSIVE account_tree AS (
                     SELECT code FROM accounts WHERE code = ?
                     UNION ALL
                     SELECT a.code FROM accounts a
                     JOIN account_tree at ON a.parent_code = at.code
                   )
                   SELECT
                     acc.type,
                     COALESCE(SUM(jl.debit), 0)  AS total_debit,
                     COALESCE(SUM(jl.credit), 0) AS total_credit
                   FROM account_tree
                   JOIN accounts acc ON acc.code = account_tree.code
                   LEFT JOIN journal_lines jl ON jl.account_code = account_tree.code
                   LEFT JOIN journal_entries je ON je.id = jl.journal_entry_id
                   WHERE je.transaction_at <= ?
                   GROUP BY acc.type"
                  account-code as-of-date])]
    
    ;; Balance depends on account type:
    ;; Asset/Expense: balance = debits - credits
    ;; Liability/Equity/Revenue: balance = credits - debits
    (let [debit  (:total_debit result 0)
          credit (:total_credit result 0)
          type   (:type result)]
      (if (#{:asset :expense} (keyword type))
        (- debit credit)
        (- credit debit)))))

;; Trial balance
(defn trial-balance [db as-of-date]
  (jdbc/execute! db
    ["SELECT
        a.code, a.name, a.type,
        COALESCE(SUM(jl.debit), 0)  AS total_debit,
        COALESCE(SUM(jl.credit), 0) AS total_credit
      FROM accounts a
      LEFT JOIN journal_lines jl ON jl.account_code = a.code
      LEFT JOIN journal_entries je ON je.id = jl.journal_entry_id
        AND je.transaction_at <= ?
      GROUP BY a.code, a.name, a.type
      ORDER BY a.code"
     as-of-date]))
```

---

## ขั้นตอนที่ 2104: Financial Transactions

```clojure
;; High-level business transactions using double-entry

;; Record customer payment
(defn record-payment! [db payment]
  (post-journal-entry! db
    {:reference    (str "PMT-" (:invoice-id payment))
     :description  (str "Payment for invoice " (:invoice-id payment))
     :transaction-at (:paid-at payment)
     :created-by   (:processed-by payment)
     :lines
     [{:account "1010"  ; Cash/Bank
       :debit   (:amount payment)
       :description "Payment received"}
      {:account "1200"  ; Accounts Receivable
       :credit  (:amount payment)
       :description (str "Invoice " (:invoice-id payment))}]}))

;; Record sale
(defn record-sale! [db sale]
  (let [tax (* (:subtotal sale) 0.07)]
    (post-journal-entry! db
      {:reference    (str "SALE-" (:order-id sale))
       :description  (str "Sale order " (:order-id sale))
       :transaction-at (:sold-at sale)
       :created-by   (:seller-id sale)
       :lines
       [{:account "1200"                   ; Accounts Receivable
         :debit   (+ (:subtotal sale) tax)
         :description "Amount due from customer"}
        {:account "4000"                   ; Revenue
         :credit  (:subtotal sale)
         :description "Product sales"}
        {:account "2300"                   ; Sales Tax Payable
         :credit  tax
         :description "Sales tax collected"}]})))

;; Record COGS
(defn record-cogs! [db sale items]
  (let [total-cogs (reduce + (map #(* (:quantity %) (:cost %)) items))]
    (post-journal-entry! db
      {:reference    (str "COGS-" (:order-id sale))
       :description  "Cost of goods sold"
       :transaction-at (:sold-at sale)
       :created-by   (:seller-id sale)
       :lines
       [{:account "5000"  ; Cost of Goods Sold
         :debit   total-cogs}
        {:account "1300"  ; Inventory
         :credit  total-cogs}]})))
```

---

## ขั้นตอนที่ 2105: Reconciliation

```clojure
;; Bank reconciliation
(defn reconcile-account! [db account-code bank-statement-date bank-balance]
  (let [ledger-balance (get-account-balance db account-code bank-statement-date)
        difference     (- ledger-balance bank-balance)]
    
    {:account         account-code
     :as-of           bank-statement-date
     :ledger-balance  ledger-balance
     :bank-balance    bank-balance
     :difference      difference
     :reconciled?     (zero? difference)
     :unreconciled-items
     (when (not (zero? difference))
       (find-unreconciled-items db account-code bank-statement-date))}))

;; Find transactions not matched to bank statement
(defn find-unreconciled-items [db account-code as-of-date]
  (jdbc/execute! db
    ["SELECT je.reference, je.description, je.transaction_at,
             jl.debit, jl.credit
      FROM journal_lines jl
      JOIN journal_entries je ON je.id = jl.journal_entry_id
      WHERE jl.account_code = ?
        AND je.transaction_at <= ?
        AND NOT EXISTS (
          SELECT 1 FROM bank_statement_lines bsl
          WHERE bsl.matched_journal_line_id = jl.id
        )
      ORDER BY je.transaction_at"
     account-code as-of-date]))
```

---

## Project: Invoice Payment System

```clojure
(ns myapp.financial.invoicing)

(defn create-invoice! [db order]
  (let [subtotal (reduce + (map #(* (:price %) (:qty %)) (:items order)))
        tax      (* subtotal 0.07)
        total    (+ subtotal tax)
        due-date (.plusDays (java.time.LocalDate/now) 30)]
    
    (jdbc/with-transaction [tx db]
      ;; Create invoice record
      (let [invoice (jdbc/execute-one! tx
                      ["INSERT INTO invoices
                        (order_id, customer_id, subtotal, tax, total, due_date, status)
                        VALUES (?, ?, ?, ?, ?, ?, 'pending')
                        RETURNING *"
                       (:id order) (:customer-id order)
                       subtotal tax total due-date])]
        
        ;; Record in ledger
        (record-sale! tx {:order-id (:id order)
                           :subtotal subtotal
                           :sold-at  (java.time.Instant/now)
                           :seller-id "system"})
        
        invoice))))

(defn process-payment! [db invoice-id payment-details]
  (jdbc/with-transaction [tx db]
    (let [invoice (jdbc/execute-one! tx
                    ["SELECT * FROM invoices WHERE id = ? FOR UPDATE"
                     invoice-id])]
      
      (when (not= "pending" (:invoices/status invoice))
        (throw (ex-info "Invoice already paid" {:invoice-id invoice-id})))
      
      ;; Record payment in ledger
      (record-payment! tx
        {:invoice-id invoice-id
         :amount     (:invoices/total invoice)
         :paid-at    (java.time.Instant/now)
         :processed-by "payment-system"})
      
      ;; Update invoice status
      (jdbc/execute-one! tx
        ["UPDATE invoices SET status = 'paid', paid_at = NOW() WHERE id = ?"
         invoice-id]))))
```

---

*Part 71 จาก 100+ | ขั้นตอน 2101-2130 จาก 1000+*
