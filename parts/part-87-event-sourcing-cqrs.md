# Part 87: Event Sourcing & CQRS
## ขั้นตอนที่ 2581-2610: Event Store, Projections, Snapshots, Sagas

---

## บทนำ

Event Sourcing คือการเก็บ state ในรูปแบบ sequence of events:
- **Event Store** - append-only log of domain events
- **Projections** - build read models from events
- **Snapshots** - optimize replay performance
- **CQRS** - separate Read/Write models
- **Process Managers** - coordinate multi-aggregate workflows

---

## ขั้นตอนที่ 2581: Event Store Foundation

```clojure
(ns myapp.event-sourcing.store
  (:require [next.jdbc :as jdbc]
            [cheshire.core :as json]
            [clojure.spec.alpha :as s]))

;; Event spec
(s/def ::event-id         uuid?)
(s/def ::aggregate-id     string?)
(s/def ::aggregate-type   keyword?)
(s/def ::event-type       keyword?)
(s/def ::payload          map?)
(s/def ::metadata         map?)
(s/def ::sequence-number  pos-int?)
(s/def ::occurred-at      inst?)

(s/def ::event
  (s/keys :req [::event-id ::aggregate-id ::aggregate-type
                ::event-type ::payload ::sequence-number ::occurred-at]
          :opt [::metadata]))

;; Event store schema
;; CREATE TABLE events (
;;   event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
;;   aggregate_id    TEXT NOT NULL,
;;   aggregate_type  TEXT NOT NULL,
;;   event_type      TEXT NOT NULL,
;;   payload         JSONB NOT NULL,
;;   metadata        JSONB DEFAULT '{}',
;;   sequence_number BIGINT NOT NULL,
;;   occurred_at     TIMESTAMPTZ DEFAULT NOW(),
;;   UNIQUE(aggregate_id, sequence_number)
;; );
;; CREATE INDEX events_aggregate_idx ON events(aggregate_id, sequence_number);
;; CREATE INDEX events_type_idx ON events(event_type, occurred_at);

;; Append events (optimistic concurrency control)
(defn append-events! [db aggregate-id aggregate-type events expected-version]
  (jdbc/with-transaction [tx db]
    ;; Check current version
    (let [current-version (or (:max-seq
                                 (jdbc/execute-one! tx
                                   ["SELECT MAX(sequence_number) as max_seq
                                     FROM events WHERE aggregate_id = ?"
                                    aggregate-id]))
                               0)]
      (when (not= current-version expected-version)
        (throw (ex-info "Optimistic concurrency conflict"
                         {:expected  expected-version
                          :actual    current-version
                          :aggregate aggregate-id})))
      
      ;; Append new events
      (doall
        (map-indexed
          (fn [i event]
            (jdbc/execute-one! tx
              ["INSERT INTO events
                  (aggregate_id, aggregate_type, event_type, payload, metadata, sequence_number)
                VALUES (?, ?, ?, ?::jsonb, ?::jsonb, ?)"
               aggregate-id
               (name aggregate-type)
               (name (:event-type event))
               (json/encode (:payload event))
               (json/encode (:metadata event {}))
               (+ current-version i 1)]))
          events)))))

;; Load events for aggregate
(defn load-events [db aggregate-id & [from-version]]
  (let [from (or from-version 0)
        rows (jdbc/execute! db
               ["SELECT event_id, aggregate_id, event_type, payload, metadata,
                         sequence_number, occurred_at
                  FROM events
                  WHERE aggregate_id = ? AND sequence_number > ?
                  ORDER BY sequence_number ASC"
                aggregate-id from])]
    (map (fn [row]
            {:event-id        (:events/event_id row)
             :aggregate-id    (:events/aggregate_id row)
             :event-type      (keyword (:events/event_type row))
             :payload         (json/parse-string (:events/payload row) true)
             :metadata        (json/parse-string (:events/metadata row) true)
             :sequence-number (:events/sequence_number row)
             :occurred-at     (:events/occurred_at row)})
         rows)))
```

---

## ขั้นตอนที่ 2582: Aggregate Rebuild from Events

```clojure
(ns myapp.event-sourcing.aggregate)

;; Order aggregate
(defmulti apply-event
  (fn [_state event] (:event-type event)))

(defmethod apply-event :order-created
  [_state {:keys [payload]}]
  {:id          (:order-id payload)
   :customer-id (:customer-id payload)
   :status      :pending
   :items       (:items payload)
   :total       (reduce + (map :total (:items payload)))
   :created-at  (:occurred-at _state)})

(defmethod apply-event :order-confirmed
  [state _event]
  (assoc state :status :confirmed))

(defmethod apply-event :payment-received
  [state {:keys [payload]}]
  (assoc state
    :status     :paid
    :payment-id (:payment-id payload)))

(defmethod apply-event :order-shipped
  [state {:keys [payload]}]
  (assoc state
    :status         :shipped
    :tracking-number (:tracking-number payload)))

(defmethod apply-event :order-cancelled
  [state {:keys [payload]}]
  (assoc state
    :status         :cancelled
    :cancel-reason  (:reason payload)))

;; Rebuild aggregate from events
(defn rebuild-aggregate [events]
  (reduce apply-event nil events))

;; Command handler
(defn confirm-order! [db order-id]
  (let [events  (load-events db order-id)
        state   (rebuild-aggregate events)
        version (count events)]
    
    (when (not= :pending (:status state))
      (throw (ex-info "Order cannot be confirmed"
                       {:order-id order-id :status (:status state)})))
    
    (append-events! db order-id :order
      [{:event-type :order-confirmed
        :payload    {:order-id   order-id
                     :confirmed-at (java.time.Instant/now)}}]
      version)))
```

---

## ขั้นตอนที่ 2583: Projections (Read Models)

```clojure
(ns myapp.event-sourcing.projections)

;; Projection: maintain a read-optimized view

;; Order summary projection
;; CREATE TABLE order_summary (
;;   order_id    TEXT PRIMARY KEY,
;;   customer_id TEXT NOT NULL,
;;   status      TEXT NOT NULL,
;;   total       DECIMAL(10,2),
;;   item_count  INT,
;;   created_at  TIMESTAMPTZ,
;;   updated_at  TIMESTAMPTZ
;; );

(defmulti handle-projection-event
  (fn [_db event] (:event-type event)))

(defmethod handle-projection-event :order-created
  [db {:keys [payload occurred-at]}]
  (jdbc/execute! db
    ["INSERT INTO order_summary
        (order_id, customer_id, status, total, item_count, created_at, updated_at)
      VALUES (?, ?, 'pending', ?, ?, ?, ?)"
     (:order-id payload)
     (:customer-id payload)
     (:total payload)
     (count (:items payload))
     occurred-at
     occurred-at]))

(defmethod handle-projection-event :order-confirmed
  [db {:keys [payload occurred-at]}]
  (jdbc/execute! db
    ["UPDATE order_summary
      SET status = 'confirmed', updated_at = ?
      WHERE order_id = ?"
     occurred-at (:order-id payload)]))

(defmethod handle-projection-event :order-cancelled
  [db {:keys [payload occurred-at]}]
  (jdbc/execute! db
    ["UPDATE order_summary
      SET status = 'cancelled', updated_at = ?
      WHERE order_id = ?"
     occurred-at (:order-id payload)]))

(defmethod handle-projection-event :default
  [_db event]
  ;; Ignore unrecognized events
  nil)

;; Projection runner: replay all events
(defn rebuild-projection! [db]
  (jdbc/execute! db ["TRUNCATE TABLE order_summary"])
  (let [all-events (jdbc/execute! db
                     ["SELECT event_type, payload::text, occurred_at
                       FROM events
                       WHERE aggregate_type = 'order'
                       ORDER BY occurred_at, sequence_number"])]
    (doseq [row all-events]
      (handle-projection-event db
        {:event-type (keyword (:events/event_type row))
         :payload    (json/parse-string (:events/payload row) true)
         :occurred-at (:events/occurred_at row)}))))

;; Incremental projection with checkpoint
(defn run-projection-incrementally! [db last-event-id]
  (let [new-events (jdbc/execute! db
                     ["SELECT event_id, event_type, payload::text, occurred_at
                       FROM events
                       WHERE event_id > ? AND aggregate_type = 'order'
                       ORDER BY occurred_at, sequence_number
                       LIMIT 1000"
                      last-event-id])]
    (doseq [row new-events]
      (handle-projection-event db
        {:event-type  (keyword (:events/event_type row))
          :payload     (json/parse-string (:events/payload row) true)
          :occurred-at (:events/occurred_at row)}))
    (when (seq new-events)
      (:events/event_id (last new-events)))))
```

---

## ขั้นตอนที่ 2584: Snapshots for Performance

```clojure
;; Snapshots: avoid replaying all events every time
;; CREATE TABLE snapshots (
;;   aggregate_id      TEXT NOT NULL,
;;   aggregate_type    TEXT NOT NULL,
;;   snapshot_version  BIGINT NOT NULL,
;;   state             JSONB NOT NULL,
;;   created_at        TIMESTAMPTZ DEFAULT NOW(),
;;   PRIMARY KEY (aggregate_id, snapshot_version)
;; );

(def snapshot-frequency 50) ; Snapshot every N events

(defn save-snapshot! [db aggregate-id aggregate-type state version]
  (jdbc/execute! db
    ["INSERT INTO snapshots (aggregate_id, aggregate_type, snapshot_version, state)
      VALUES (?, ?, ?, ?::jsonb)
      ON CONFLICT (aggregate_id, snapshot_version) DO NOTHING"
     aggregate-id (name aggregate-type) version (json/encode state)]))

(defn load-latest-snapshot [db aggregate-id]
  (when-let [row (jdbc/execute-one! db
                   ["SELECT state, snapshot_version
                     FROM snapshots
                     WHERE aggregate_id = ?
                     ORDER BY snapshot_version DESC
                     LIMIT 1"
                    aggregate-id])]
    {:state   (json/parse-string (:snapshots/state row) true)
     :version (:snapshots/snapshot_version row)}))

;; Load aggregate with snapshot optimization
(defn load-aggregate-with-snapshot [db aggregate-id]
  (let [snapshot (load-latest-snapshot db aggregate-id)
        from-ver (or (:version snapshot) 0)
        events   (load-events db aggregate-id from-ver)
        state    (if snapshot
                   (reduce apply-event (:state snapshot) events)
                   (rebuild-aggregate events))
        version  (+ from-ver (count events))]
    
    ;; Create new snapshot if needed
    (when (and (>= (count events) snapshot-frequency)
               (seq events))
      (save-snapshot! db aggregate-id :order state version))
    
    {:state   state
     :version version}))
```

---

## ขั้นตอนที่ 2585: CQRS Query Side

```clojure
;; CQRS: Separate command and query handling

;; Query models (denormalized for performance)
(defn get-customer-orders [db customer-id status page page-size]
  (jdbc/execute! db
    [(str "SELECT o.order_id, o.status, o.total, o.item_count,
                  o.created_at, c.name as customer_name
           FROM order_summary o
           JOIN customers c ON c.id = o.customer_id
           WHERE o.customer_id = ?"
          (when status " AND o.status = ?")
          " ORDER BY o.created_at DESC"
          " LIMIT ? OFFSET ?")
     customer-id
     (when status (name status))
     page-size (* page page-size)]))

;; Revenue report (read model updated by projections)
(defn revenue-by-day [db from-date to-date]
  (jdbc/execute! db
    ["SELECT DATE(created_at) as day,
              COUNT(*) as order_count,
              SUM(total) as revenue
       FROM order_summary
       WHERE status = 'paid'
         AND created_at BETWEEN ? AND ?
       GROUP BY DATE(created_at)
       ORDER BY day"
     from-date to-date]))

;; Search with full-text
(defn search-orders [db query]
  (jdbc/execute! db
    ["SELECT order_id, status, total, created_at
       FROM order_summary
       WHERE to_tsvector('english', order_id || ' ' || customer_id)
             @@ plainto_tsquery('english', ?)
       LIMIT 20"
     query]))
```

---

*Part 87 จาก 100+ | ขั้นตอน 2581-2610 จาก 1000+*
