# Part 20: Data Processing และ Analytics
## ขั้นตอนที่ 571-600: Large Data, Pipeline Processing, Spark Integration

---

## บทนำ

Clojure เหมาะมากกับ Data Engineering:
- Immutable data → thread-safe
- Lazy sequences → infinite streams
- Transducers → efficient pipeline
- Reducers → parallel processing
- Spark/Hadoop integration ผ่าน JVM

---

## ขั้นตอนที่ 571: Processing Large CSV Files

```clojure
(ns dataproc.csv
  (:require [clojure.data.csv :as csv]
            [clojure.java.io :as io]))

;; Read large CSV lazily
(defn read-csv-lazy [filename]
  (let [reader (io/reader filename)]
    (csv/read-csv reader)))

;; ❌ Don't load all into memory
(defn bad-approach [filename]
  (slurp filename))  ; loads entire file!

;; ✅ Stream processing
(defn process-large-csv [input-file output-file process-fn]
  (with-open [reader (io/reader input-file)
              writer (io/writer output-file)]
    (let [rows (csv/read-csv reader)
          header (first rows)
          data-rows (rest rows)]
      
      ;; Write header
      (csv/write-csv writer [header])
      
      ;; Process and write rows lazily
      (doseq [row data-rows]
        (let [record (zipmap (map keyword header) row)
              result (process-fn record)]
          (when result
            (csv/write-csv writer [(map str (vals result))])))))))

;; ใช้งาน
(process-large-csv
  "sales.csv"
  "sales-processed.csv"
  (fn [{:keys [amount date status] :as record}]
    (when (= "completed" status)
      (assoc record :amount-thb (* (parse-double amount) 35)))))
```

---

## ขั้นตอนที่ 572: Data Transformation Pipelines

```clojure
;; Composable data transformations ด้วย transducers

;; Define transforms
(def normalize-name
  (map #(update % :name clojure.string/trim)))

(def filter-active
  (filter #(= "active" (:status %))))

(def enrich-with-tax
  (map #(assoc % :total-with-tax (* (:total %) 1.07))))

(def format-currency
  (map #(update % :total-with-tax (fn [v] (format "%.2f" v)))))

;; Compose pipeline
(def sales-pipeline
  (comp normalize-name
        filter-active
        enrich-with-tax
        format-currency))

;; Apply to data
(into [] sales-pipeline sales-data)

;; Streaming with channel
(require '[clojure.core.async :as async])

(defn stream-transform [xf input-ch]
  (let [output-ch (async/chan 1000 xf)]
    (async/pipe input-ch output-ch)
    output-ch))

(defn start-pipeline [source-file]
  (let [input-ch  (async/chan 1000)
        output-ch (stream-transform sales-pipeline input-ch)]
    
    ;; Feed data
    (async/go
      (with-open [reader (io/reader source-file)]
        (doseq [line (line-seq reader)]
          (async/>! input-ch (parse-line line))))
      (async/close! input-ch))
    
    output-ch))
```

---

## ขั้นตอนที่ 573: Statistical Analysis

```clojure
(ns dataproc.stats)

;; Basic statistics functions

(defn mean [xs]
  (/ (reduce + xs) (count xs)))

(defn variance [xs]
  (let [m (mean xs)
        n (count xs)]
    (/ (reduce + (map #(Math/pow (- % m) 2) xs)) n)))

(defn std-dev [xs]
  (Math/sqrt (variance xs)))

(defn median [xs]
  (let [sorted (sort xs)
        n (count sorted)
        mid (quot n 2)]
    (if (odd? n)
      (nth sorted mid)
      (/ (+ (nth sorted (dec mid)) (nth sorted mid)) 2.0))))

(defn percentile [xs p]
  (let [sorted (sort xs)
        n (count sorted)
        idx (int (* p n))]
    (nth sorted (min idx (dec n)))))

(defn summary-stats [xs]
  {:count  (count xs)
   :min    (apply min xs)
   :max    (apply max xs)
   :mean   (mean xs)
   :median (median xs)
   :std    (std-dev xs)
   :p25    (percentile xs 0.25)
   :p75    (percentile xs 0.75)
   :p95    (percentile xs 0.95)
   :p99    (percentile xs 0.99)})

;; Moving statistics
(defn moving-avg [xs n]
  (->> xs
       (partition n 1)
       (map mean)))

;; Frequency distribution
(defn frequency-distribution [xs bins]
  (let [min-val (apply min xs)
        max-val (apply max xs)
        range   (- max-val min-val)
        bin-size (/ range bins)]
    (->> xs
         (group-by #(min (int (/ (- % min-val) bin-size)) (dec bins)))
         (sort-by first)
         (map (fn [[bin items]]
                {:bin   (+ min-val (* bin bin-size))
                 :count (count items)})))))
```

---

## ขั้นตอนที่ 574: Time Series Analysis

```clojure
(ns dataproc.timeseries)

;; Time series data processing

(defn resample [data interval-ms agg-fn]
  "Group data into time buckets"
  (->> data
       (group-by #(- (:timestamp %)
                      (mod (:timestamp %) interval-ms)))
       (sort-by first)
       (map (fn [[bucket-start records]]
              {:timestamp bucket-start
               :value     (agg-fn (map :value records))
               :count     (count records)}))))

;; 1-minute buckets
(def minutely (partial resample data (* 60 1000) mean))
;; 1-hour buckets
(def hourly   (partial resample data (* 3600 1000) mean))

;; Detect anomalies (simple z-score method)
(defn detect-anomalies [ts threshold]
  (let [values (map :value ts)
        m      (mean values)
        s      (std-dev values)]
    (filter (fn [{:keys [value]}]
              (> (Math/abs (/ (- value m) s)) threshold))
            ts)))

;; Trend detection (linear regression)
(defn linear-regression [xs ys]
  (let [n  (count xs)
        sx  (reduce + xs)
        sy  (reduce + ys)
        sxy (reduce + (map * xs ys))
        sxx (reduce + (map #(* % %) xs))
        slope     (/ (- (* n sxy) (* sx sy))
                      (- (* n sxx) (* sx sx)))
        intercept (/ (- sy (* slope sx)) n)]
    {:slope slope :intercept intercept
     :predict (fn [x] (+ intercept (* slope x)))}))
```

---

## ขั้นตอนที่ 575: Apache Spark ด้วย Sparkling

```clojure
;; deps.edn
;; {:deps {gorillalabs/sparkling {:mvn/version "3.0.0"}}}

(ns dataproc.spark
  (:require [sparkling.core :as spark]
            [sparkling.conf :as conf]))

;; Create Spark context
(def sc
  (-> (conf/spark-conf)
      (conf/master "local[*]")
      (conf/app-name "Clojure Spark")
      spark/spark-context))

;; Basic RDD operations
(def data-rdd
  (spark/parallelize sc [1 2 3 4 5 6 7 8 9 10]))

(-> data-rdd
    (spark/filter even?)
    (spark/map #(* % %))
    (spark/reduce +))
;; => 220 (4+16+36+64+100)

;; Process text file
(def word-count
  (-> (spark/text-file sc "hdfs://large-text-file.txt")
      (spark/flat-map #(clojure.string/split % #"\s+"))
      (spark/map #(vector % 1))
      (spark/reduce-by-key +)
      spark/collect))

;; DataFrame-style with Spark SQL
(def df (spark/read-json sc "data.json"))
(spark/sql "SELECT category, SUM(amount) FROM sales GROUP BY category")
```

---

## ขั้นตอนที่ 576: Data Validation Pipeline

```clojure
(ns dataproc.validation
  (:require [malli.core :as m]))

;; Schema for CSV records
(def SalesRecord
  [:map
   [:order-id  [:re #"ORD-\d+"]]
   [:date      [:re #"\d{4}-\d{2}-\d{2}"]]
   [:amount    [:double {:min 0}]]
   [:currency  [:enum "THB" "USD" "EUR"]]
   [:customer  [:string {:min 1}]]
   [:status    [:enum "pending" "completed" "cancelled"]]])

;; Validation pipeline
(defn validate-record [record]
  (if (m/validate SalesRecord record)
    {:valid true  :data record}
    {:valid false :data record
     :errors (m/explain SalesRecord record)}))

(defn categorize-records [records]
  (let [validated (map validate-record records)]
    {:valid   (map :data (filter :valid validated))
     :invalid (filter (complement :valid) validated)}))

;; Process with error reporting
(defn process-with-validation [records process-fn]
  (let [{:keys [valid invalid]} (categorize-records records)]
    (println (format "Valid: %d, Invalid: %d" (count valid) (count invalid)))
    (when (seq invalid)
      (println "Sample errors:")
      (doseq [err (take 5 invalid)]
        (println "  " (:data err) "→" (:errors err))))
    (map process-fn valid)))
```

---

## Project: Sales Analytics Dashboard

```clojure
(ns analytics.dashboard
  (:require [dataproc.stats :as stats]
            [clojure.data.csv :as csv]
            [clojure.java.io :as io]))

;; Load and analyze sales data
(defn analyze-sales [csv-file]
  (with-open [reader (io/reader csv-file)]
    (let [rows (csv/read-csv reader)
          header (first rows)
          data   (map (fn [row]
                        (let [record (zipmap (map keyword header) row)]
                          (update record :amount parse-double)))
                      (rest rows))
          completed (filter #(= "completed" (:status %)) data)]
      
      {:total-orders    (count data)
       :completed       (count completed)
       :completion-rate (/ (count completed) (count data) 1.0)
       
       :revenue-stats
       (stats/summary-stats (map :amount completed))
       
       :by-category
       (->> completed
            (group-by :category)
            (map (fn [[cat records]]
                   {:category cat
                    :count    (count records)
                    :revenue  (reduce + (map :amount records))
                    :avg-order (stats/mean (map :amount records))}))
            (sort-by :revenue >))
       
       :daily-revenue
       (->> completed
            (group-by :date)
            (map (fn [[date records]]
                   {:date    date
                    :revenue (reduce + (map :amount records))
                    :orders  (count records)}))
            (sort-by :date))
       
       :top-customers
       (->> completed
            (group-by :customer)
            (map (fn [[customer records]]
                   {:customer customer
                    :orders   (count records)
                    :revenue  (reduce + (map :amount records))}))
            (sort-by :revenue >)
            (take 10))})))

;; Generate report
(defn print-report [analysis]
  (println "=== SALES ANALYTICS REPORT ===\n")
  (println (format "Total Orders: %d" (:total-orders analysis)))
  (println (format "Completed: %d (%.1f%%)"
                   (:completed analysis)
                   (* 100 (:completion-rate analysis))))
  (println "\n--- Revenue Summary ---")
  (let [{:keys [mean median p95]} (:revenue-stats analysis)]
    (println (format "Mean Order: %.2f" mean))
    (println (format "Median Order: %.2f" median))
    (println (format "95th Percentile: %.2f" p95)))
  (println "\n--- Top Categories ---")
  (doseq [{:keys [category count revenue]} (take 5 (:by-category analysis))]
    (println (format "  %s: %d orders, %.0f revenue" category count revenue))))
```

---

*Part 20 จาก 100+ | ขั้นตอน 571-600 จาก 1000+*
