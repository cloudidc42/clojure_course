# Part 44: Data Engineering
## ขั้นตอนที่ 1291-1320: ETL Pipelines, Data Transformation, Apache Spark, BigQuery

---

## บทนำ

Data Engineering ด้วย Clojure:
- **ETL/ELT** - extract, transform, load pipelines
- **Data Transformation** - transduce, reducers
- **Sparkling** - Apache Spark ด้วย Clojure
- **BigQuery** - analytical queries at scale
- **Data Quality** - validation, lineage, profiling

---

## ขั้นตอนที่ 1291: CSV/JSON ETL Pipeline

```clojure
(ns etl.pipeline
  (:require [clojure.data.csv :as csv]
            [clojure.java.io :as io]
            [cheshire.core :as json]
            [next.jdbc :as jdbc]))

;; Extract: Read CSV file lazily
(defn extract-csv [file-path]
  (let [reader (io/reader file-path)
        rows   (csv/read-csv reader)]
    {:reader reader
     :header (first rows)
     :rows   (rest rows)}))

;; Transform: Convert CSV rows to maps with type coercion
(defn transform-row [header row]
  (zipmap
    (map keyword header)
    (map (fn [v]
           (cond
             (re-matches #"\d+" v)          (Long/parseLong v)
             (re-matches #"\d+\.\d+" v)     (Double/parseDouble v)
             (re-matches #"\d{4}-\d{2}-\d{2}" v)
             (java.time.LocalDate/parse v)
             (= "true" v)                   true
             (= "false" v)                  false
             :else                          v))
         row)))

;; Load: Batch insert to PostgreSQL
(defn load-to-postgres [ds table-name rows]
  (let [columns (keys (first rows))
        placeholders (clojure.string/join ","
                        (repeat (count columns) "?"))]
    (jdbc/execute-batch! ds
      [(str "INSERT INTO " (name table-name)
             " (" (clojure.string/join "," (map name columns)) ")"
             " VALUES (" placeholders ")"
             " ON CONFLICT DO NOTHING")]
      (map (fn [row] (map #(get row %) columns)) rows)
      {:batch-size 1000})))

;; Full ETL pipeline
(defn run-etl! [source-file dest-db table]
  (let [{:keys [header rows reader]} (extract-csv source-file)]
    (try
      (println "Starting ETL for" source-file "->>" table)
      (->> rows
           (map #(transform-row header %))
           (filter #(not-any? nil? (vals %)))  ; remove invalid rows
           (partition-all 1000)                  ; batch 1000 at a time
           (map #(load-to-postgres dest-db table %))
           (reduce + 0)
           (println "Loaded rows:"))
      (finally
        (.close reader)))))
```

---

## ขั้นตอนที่ 1292: Transducer-Based Transformation

```clojure
(ns etl.transform
  (:require [clojure.string :as str]))

;; Reusable transformation pieces as transducers
(defn normalize-keys-xf []
  (map #(into {} (map (fn [[k v]] [(-> k name str/lower-case str/trim keyword) v]) %))))

(defn trim-strings-xf []
  (map #(into {} (map (fn [[k v]] [k (if (string? v) (str/trim v) v)]) %))))

(defn remove-empty-xf []
  (remove #(every? (fn [[_ v]] (or (nil? v) (= "" v))) %)))

(defn validate-required-xf [required-keys]
  (filter #(every? (fn [k] (some? (get % k))) required-keys)))

(defn add-metadata-xf [source-file batch-id]
  (map #(assoc %
          :_source source-file
          :_batch  batch-id
          :_loaded-at (java.time.Instant/now))))

;; Compose transformations
(defn customer-transform-xf [source-file batch-id]
  (comp
    (normalize-keys-xf)
    (trim-strings-xf)
    (remove-empty-xf)
    (validate-required-xf [:email :name])
    (map #(update % :email str/lower-case))
    (map #(update % :phone (fnil str/replace "-" "")))
    (add-metadata-xf source-file batch-id)))

;; Apply to any reducible
(defn transform-data [data xf]
  (transduce xf conj [] data))

;; Large file processing with channels
(defn process-large-file! [file-path xf sink-fn]
  (with-open [reader (clojure.java.io/reader file-path)]
    (let [lines (line-seq reader)]
      (transduce
        (comp
          (map #(json/parse-string % true))
          xf
          (partition-all 500))
        (fn
          ([_] nil)
          ([_ batch] (sink-fn batch)))
        nil
        lines))))
```

---

## ขั้นตอนที่ 1293: Data Validation และ Quality

```clojure
(ns etl.quality
  (:require [malli.core :as m]))

;; Data quality rules
(defn check-completeness [data required-columns]
  (let [total     (count data)
        complete  (count (filter
                           (fn [row]
                             (every? #(some? (get row %)) required-columns))
                           data))]
    {:total     total
     :complete  complete
     :missing   (- total complete)
     :pct-complete (if (pos? total)
                     (* 100.0 (/ complete total))
                     0)}))

(defn check-duplicates [data key-fn]
  (let [grouped (group-by key-fn data)
        dupes   (filter #(> (count (val %)) 1) grouped)]
    {:total      (count data)
     :duplicates (count dupes)
     :dup-keys   (map first dupes)}))

(defn check-distribution [data column]
  (let [values (map #(get % column) data)
        freq   (frequencies values)]
    {:column  column
     :distinct (count freq)
     :top-10  (take 10 (sort-by val > freq))
     :nulls   (count (filter nil? values))}))

(defn profile-dataset [data]
  {:row-count  (count data)
   :columns    (keys (first data))
   :sample     (take 3 data)
   :stats      (map #(check-distribution data %) (keys (first data)))})

;; Data lineage tracking
(defn create-lineage-record [job-name sources targets]
  {:job-id    (java.util.UUID/randomUUID)
   :job-name  job-name
   :started-at (java.time.Instant/now)
   :sources   sources
   :targets   targets})

(defn finish-lineage! [record rows-processed errors]
  (assoc record
    :finished-at   (java.time.Instant/now)
    :rows-processed rows-processed
    :errors        errors
    :status        (if (empty? errors) :success :failed)))
```

---

## ขั้นตอนที่ 1294: Incremental Loading

```clojure
(ns etl.incremental
  (:require [next.jdbc :as jdbc]
            [java-time.api :as jt]))

;; Watermark tracking
(defn get-watermark [ds table]
  (-> (jdbc/execute-one! ds
        ["SELECT max_processed_at FROM etl_watermarks WHERE table_name = ?"
         table])
      :max_processed_at))

(defn update-watermark! [ds table timestamp]
  (jdbc/execute! ds
    ["INSERT INTO etl_watermarks (table_name, max_processed_at, updated_at)
      VALUES (?, ?, NOW())
      ON CONFLICT (table_name)
      DO UPDATE SET max_processed_at = ?, updated_at = NOW()"
     table timestamp timestamp]))

;; Incremental extract
(defn extract-since [ds table watermark]
  (jdbc/execute! ds
    [(str "SELECT * FROM " table
           " WHERE updated_at > ?"
           " ORDER BY updated_at ASC"
           " LIMIT 50000")
     (or watermark (java.time.Instant/EPOCH))]))

;; CDC (Change Data Capture) simulation
(defn extract-changes [ds table since]
  (jdbc/execute! ds
    [(str "SELECT *, 'insert' as change_type FROM " table
           " WHERE created_at > ?"
           " UNION ALL "
           "SELECT *, 'update' as change_type FROM " table
           " WHERE updated_at > ? AND created_at <= ?")
     since since since]))

;; Run incremental ETL
(defn run-incremental! [source-ds dest-ds source-table dest-table transform-fn]
  (let [watermark (get-watermark dest-ds dest-table)
        records   (extract-since source-ds source-table watermark)]
    
    (when (seq records)
      (let [transformed (map transform-fn records)
            max-ts      (apply max (map :updated_at records))]
        (jdbc/execute-batch! dest-ds
          [(str "INSERT INTO " dest-table
                 " SELECT * FROM json_populate_recordset(null::" dest-table ", ?)")]
          [[(json/generate-string transformed)]])
        (update-watermark! dest-ds dest-table max-ts)
        (println "Loaded" (count records) "records, watermark:" max-ts)))))
```

---

## ขั้นตอนที่ 1295: Aggregate Queries และ Analytics

```clojure
(ns analytics.queries
  (:require [honey.sql :as sql]
            [next.jdbc :as jdbc]))

;; Funnel analysis
(defn funnel-analysis [ds date-range]
  (jdbc/execute! ds
    ["WITH funnel AS (
        SELECT
          user_id,
          MAX(CASE WHEN event_type = 'page_view' THEN 1 ELSE 0 END) as viewed,
          MAX(CASE WHEN event_type = 'add_to_cart' THEN 1 ELSE 0 END) as added_to_cart,
          MAX(CASE WHEN event_type = 'checkout_start' THEN 1 ELSE 0 END) as started_checkout,
          MAX(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) as purchased
        FROM events
        WHERE created_at BETWEEN ? AND ?
        GROUP BY user_id
      )
      SELECT
        SUM(viewed)           as visitors,
        SUM(added_to_cart)    as added_to_cart,
        SUM(started_checkout) as started_checkout,
        SUM(purchased)        as purchasers,
        ROUND(100.0 * SUM(purchased) / NULLIF(SUM(viewed), 0), 2) as conversion_rate
      FROM funnel"
     (:from date-range) (:to date-range)]))

;; Cohort analysis
(defn cohort-retention [ds]
  (jdbc/execute! ds
    ["WITH cohorts AS (
        SELECT
          user_id,
          DATE_TRUNC('month', first_purchase_at) as cohort_month
        FROM users
        WHERE first_purchase_at IS NOT NULL
      ),
      activities AS (
        SELECT
          o.user_id,
          DATE_TRUNC('month', o.created_at) as activity_month
        FROM orders o
      )
      SELECT
        c.cohort_month,
        DATE_PART('month', AGE(a.activity_month, c.cohort_month)) as months_since_first,
        COUNT(DISTINCT a.user_id) as retained_users
      FROM cohorts c
      JOIN activities a ON c.user_id = a.user_id
      WHERE a.activity_month >= c.cohort_month
      GROUP BY c.cohort_month, months_since_first
      ORDER BY c.cohort_month, months_since_first"]))
```

---

## Project: Data Warehouse ETL

```clojure
(ns warehouse.etl
  (:require [etl.pipeline :as pipeline]
            [etl.incremental :as incr]))

;; Dimension tables (slowly changing)
(defn load-dim-customers! [ds]
  (let [records (pipeline/extract-csv "data/customers.csv")]
    (->> (:rows records)
         (map #(pipeline/transform-row (:header records) %))
         (map (fn [r]
                (-> r
                    (assoc :valid_from (java.time.LocalDate/now)
                            :valid_to   (java.time.LocalDate/parse "9999-12-31")
                            :is_current true)
                    (dissoc :password))))
         (pipeline/load-to-postgres ds :dim_customers))))

;; Fact table (events)
(defn load-fact-orders! [source-ds dest-ds since]
  (incr/run-incremental! source-ds dest-ds :orders :fact_orders
    (fn [order]
      {:order_id     (:id order)
       :customer_key (lookup-customer-key dest-ds (:user_id order))
       :product_key  (lookup-product-key dest-ds (:product_id order))
       :date_key     (date-to-key (:created_at order))
       :amount       (:total order)
       :quantity     (:quantity order)
       :channel      (:channel order)})))

;; Orchestrate
(defn run-daily-etl! []
  (println "Starting daily ETL:" (java.time.LocalDateTime/now))
  (load-dim-customers! warehouse-ds)
  (load-fact-orders! oltp-ds warehouse-ds (yesterday))
  (refresh-materialized-views! warehouse-ds)
  (println "ETL complete"))
```

---

*Part 44 จาก 100+ | ขั้นตอน 1291-1320 จาก 1000+*
