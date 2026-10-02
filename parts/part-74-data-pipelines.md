# Part 74: Data Pipelines และ ETL
## ขั้นตอนที่ 2191-2220: Stream Processing, Batch ETL, Data Validation, Transformation

---

## บทนำ

Data pipeline patterns:
- **Stream processing** - process events in real-time
- **Batch ETL** - Extract, Transform, Load
- **Data validation** - schema enforcement
- **Dead letter queue** - handle failures gracefully
- **Backpressure** - flow control

---

## ขั้นตอนที่ 2191: Generic Pipeline Framework

```clojure
(ns myapp.pipeline.core
  (:require [clojure.core.async :as async]))

;; Pipeline step protocol
(defprotocol PipelineStep
  (process [this record context])
  (step-name [this]))

;; Pipeline execution
(defrecord Pipeline [steps error-handler metrics])

(defn run-pipeline! [pipeline records context]
  (reduce
    (fn [acc-records step]
      (let [step-name   (step-name step)
            start-time  (System/currentTimeMillis)]
        (mapcat
          (fn [record]
            (try
              (let [result (process step record context)]
                (if (sequential? result) result [result]))
              (catch Exception e
                ((:error-handler pipeline)
                 {:step   step-name
                  :record record
                  :error  e})
                [])))  ; Skip failed records
          acc-records)))
    records
    (:steps pipeline)))

;; Async pipeline using channels
(defn async-pipeline! [steps input-ch error-ch concurrency]
  (let [channels (reduce
                   (fn [prev-ch step]
                     (let [next-ch (async/chan 1000)]
                       (async/pipeline-blocking
                         concurrency
                         next-ch
                         (map (fn [record]
                                (try
                                  (process step record {})
                                  (catch Exception e
                                    (async/put! error-ch
                                      {:step   (step-name step)
                                       :record record
                                       :error  e})
                                    ::skip))))
                         prev-ch)
                       next-ch))
                   input-ch
                   steps)]
    channels))
```

---

## ขั้นตอนที่ 2192: ETL Steps

```clojure
;; Extract step
(defrecord ExtractFromDB [db query]
  PipelineStep
  (process [_ _record _ctx]
    (jdbc/execute! db [query]))
  (step-name [_] "extract-from-db"))

;; Transform step: normalize data
(defrecord NormalizeRecord [field-mappings]
  PipelineStep
  (process [_ record _ctx]
    (into {}
      (map (fn [[from to]]
               [to (get record from)])
           field-mappings)))
  (step-name [_] "normalize"))

;; Validate step with schema
(defrecord ValidateRecord [schema]
  PipelineStep
  (process [_ record _ctx]
    (let [result (validate-schema schema record)]
      (if (:valid? result)
        record
        (throw (ex-info "Validation failed"
                         {:record record
                          :errors (:errors result)})))))
  (step-name [_] "validate"))

;; Enrich step: add computed fields
(defrecord EnrichRecord [enrichments]
  PipelineStep
  (process [_ record ctx]
    (reduce
      (fn [r [field f]]
        (assoc r field (f r ctx)))
      record
      enrichments))
  (step-name [_] "enrich"))

;; Load step
(defrecord LoadToWarehouse [db table-name]
  PipelineStep
  (process [_ record _ctx]
    (jdbc/execute-one! db
      (hsql/format
        {:insert-into table-name
         :values      [record]
         :on-conflict {:do-update-set (keys record)}}))
    record)
  (step-name [_] (str "load-to-" (name table-name))))

;; Filter step
(defrecord FilterRecords [pred]
  PipelineStep
  (process [_ record _ctx]
    (when (pred record) record))
  (step-name [_] "filter"))
```

---

## ขั้นตอนที่ 2193: Batch Processing

```clojure
(ns myapp.pipeline.batch)

;; Process records in batches for efficiency
(defn batch-process!
  [records batch-size process-batch-fn error-fn]
  (let [batches   (partition-all batch-size records)
        total     (count records)
        processed (atom 0)
        errors    (atom [])]
    
    (doseq [batch batches]
      (try
        (process-batch-fn batch)
        (swap! processed + (count batch))
        (when (= 0 (mod @processed 1000))
          (println (format "Processed %d/%d records (%.1f%%)"
                            @processed total
                            (* 100.0 (/ @processed total)))))
        (catch Exception e
          (swap! errors into
            (map (fn [record]
                    {:record record :error (.getMessage e)})
                 batch))
          (when error-fn
            (error-fn e batch)))))
    
    {:processed @processed
     :errors    @errors
     :success?  (empty? @errors)}))

;; Bulk upsert for efficiency
(defn bulk-upsert! [db table-name records key-columns]
  (let [columns    (keys (first records))
        value-rows (map (fn [r] (map #(get r %) columns)) records)
        conflict-target (map name key-columns)
        update-cols (remove #(contains? (set key-columns) %) columns)]
    
    (jdbc/execute! db
      (into [(str "INSERT INTO " (name table-name)
                  " (" (clojure.string/join ", " (map name columns)) ")"
                  " VALUES "
                  (clojure.string/join ", "
                    (repeat (count records)
                             (str "(" (clojure.string/join ", "
                                        (repeat (count columns) "?")) ")")))
                  " ON CONFLICT (" (clojure.string/join ", " conflict-target) ")"
                  " DO UPDATE SET "
                  (clojure.string/join ", "
                    (map #(str (name %) " = EXCLUDED." (name %)) update-cols)))]
             (mapcat identity value-rows)))))
```

---

## ขั้นตอนที่ 2194: Change Data Capture (CDC)

```clojure
;; PostgreSQL CDC using logical replication
(ns myapp.pipeline.cdc)

;; Listen to DB changes via LISTEN/NOTIFY
(defn start-cdc-listener! [db event-handler]
  (let [conn    (jdbc/get-connection db)
        stmt    (.createStatement conn)]
    
    ;; Create trigger on target table
    (.execute stmt
      "CREATE OR REPLACE FUNCTION notify_change()
       RETURNS trigger AS $$
       BEGIN
         PERFORM pg_notify(
           'table_changes',
           json_build_object(
             'table',  TG_TABLE_NAME,
             'op',     TG_OP,
             'old',    row_to_json(OLD),
             'new',    row_to_json(NEW)
           )::text
         );
         RETURN NEW;
       END;
       $$ LANGUAGE plpgsql")
    
    ;; Start listening
    (.execute stmt "LISTEN table_changes")
    
    ;; Poll for notifications
    (future
      (loop []
        (Thread/sleep 100)
        (let [notification (.getNotification conn)]
          (when notification
            (let [payload (json/parse-string (.getParameter notification) true)]
              (event-handler payload))))
        (recur)))))

;; Handle CDC events
(defn handle-change-event! [event search-engine cache]
  (let [{:keys [table op new old]} event]
    (case table
      "products"
      (case op
        ("INSERT" "UPDATE")
        (do
          (search/index-product! search-engine new)
          (cache/invalidate! cache (str "product:" (:id new))))
        "DELETE"
        (do
          (search/remove-product! search-engine (:id old))
          (cache/invalidate! cache (str "product:" (:id old)))))
      
      nil)))  ; Ignore other tables
```

---

## Project: Analytics ETL Pipeline

```clojure
(ns myapp.analytics.etl)

;; Daily analytics pipeline
(defn run-daily-analytics! [source-db warehouse-db]
  (let [yesterday (java.time.LocalDate/now .minusDays 1)
        
        pipeline  (->Pipeline
                    [(->ExtractFromDB source-db
                       (str "SELECT o.*, c.segment, c.region
                             FROM orders o
                             JOIN customers c ON c.id = o.customer_id
                             WHERE o.created_at::date = '" yesterday "'"))
                     
                     (->ValidateRecord order-analytics-schema)
                     
                     (->EnrichRecord
                       {:day-of-week   #(-> % :created_at parse-date .getDayOfWeek .getValue)
                        :hour-of-day   #(-> % :created_at parse-date .getHour)
                        :order-size    #(count (:items %))
                        :has-discount? #(pos? (or (:discount %) 0))})
                     
                     (->LoadToWarehouse warehouse-db :fact_orders)]
                    
                    (fn [error-info]
                      (log/error "Pipeline error"
                                  (select-keys error-info [:step :error])))
                    {})]
    
    (println "Starting analytics ETL for" yesterday)
    (let [result (run-pipeline! pipeline [{}] {})]
      (println "ETL complete:" (count result) "records processed"))))

;; Schedule daily at 1 AM
(defn schedule-analytics! [source-db warehouse-db]
  (future
    (loop []
      (let [next-run (-> (java.time.LocalDate/now .plusDays 1)
                         (.atTime 1 0)
                         (.atZone (java.time.ZoneId/systemDefault))
                         .toInstant
                         .toEpochMilli)
            sleep-ms (- next-run (System/currentTimeMillis))]
        (Thread/sleep sleep-ms)
        (try
          (run-daily-analytics! source-db warehouse-db)
          (catch Exception e
            (log/error "Analytics ETL failed" {:error (.getMessage e)})))
        (recur)))))
```

---

*Part 74 จาก 100+ | ขั้นตอน 2191-2220 จาก 1000+*
