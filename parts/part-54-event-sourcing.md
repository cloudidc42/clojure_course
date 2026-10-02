# Part 54: Event Sourcing
## ขั้นตอนที่ 1591-1620: Event Store, Projections, Snapshots, Event-Driven Architecture

---

## บทนำ

Event Sourcing - เก็บ state ด้วย events:
- **Event Store** - append-only log ของ events
- **Aggregates** - reconstruct state จาก events
- **Projections** - สร้าง read models จาก events
- **Snapshots** - เพิ่ม performance
- **Event-Driven Architecture** - ระบบที่ react ต่อ events

---

## ขั้นตอนที่ 1591: Event Store Design

```clojure
(ns myapp.event-store
  (:require [next.jdbc :as jdbc]
            [next.jdbc.sql :as sql]
            [cheshire.core :as json]))

;; Schema:
;; CREATE TABLE events (
;;   id          BIGSERIAL PRIMARY KEY,
;;   stream_id   UUID NOT NULL,
;;   stream_type VARCHAR(100) NOT NULL,
;;   version     BIGINT NOT NULL,
;;   event_type  VARCHAR(100) NOT NULL,
;;   data        JSONB NOT NULL,
;;   metadata    JSONB DEFAULT '{}',
;;   created_at  TIMESTAMPTZ DEFAULT NOW(),
;;   UNIQUE (stream_id, version)
;; );
;; CREATE INDEX idx_events_stream ON events(stream_id, version);
;; CREATE INDEX idx_events_type ON events(stream_type, created_at);

(defn append-events!
  "Append events to a stream. Raises on version conflict."
  [db stream-id stream-type expected-version events]
  (jdbc/with-transaction [tx db]
    (let [current-version
          (or (:version
                (jdbc/execute-one! tx
                  ["SELECT MAX(version) as version FROM events WHERE stream_id = ?"
                   stream-id]))
               0)]
      
      (when (not= current-version expected-version)
        (throw (ex-info "Version conflict"
                         {:expected expected-version
                          :actual   current-version
                          :stream   stream-id})))
      
      (doall
        (map-indexed
          (fn [i event]
            (jdbc/execute-one! tx
              ["INSERT INTO events (stream_id, stream_type, version, event_type, data, metadata)
                VALUES (?, ?, ?, ?, ?::jsonb, ?::jsonb)"
               stream-id
               stream-type
               (+ current-version i 1)
               (name (:type event))
               (json/generate-string (:data event))
               (json/generate-string (or (:metadata event) {}))]))
          events)))))

(defn load-events
  "Load all events for a stream, optionally starting from a version."
  [db stream-id & {:keys [from-version] :or {from-version 0}}]
  (jdbc/execute! db
    ["SELECT * FROM events
      WHERE stream_id = ? AND version > ?
      ORDER BY version ASC"
     stream-id from-version]))

(defn load-events-by-type
  "Load all events of a type for projections."
  [db event-type & {:keys [from-id] :or {from-id 0}}]
  (jdbc/execute! db
    ["SELECT * FROM events
      WHERE event_type = ? AND id > ?
      ORDER BY id ASC"
     event-type from-id]))
```

---

## ขั้นตอนที่ 1592: Aggregate Pattern

```clojure
(ns myapp.aggregates)

;; Order aggregate
(defmulti apply-event
  "Apply an event to aggregate state"
  (fn [_state event] (keyword (:events/event_type event))))

(defmethod apply-event :order-created
  [_state event]
  (let [data (cheshire.core/parse-string (:events/data event) true)]
    {:id      (:id data)
     :user-id (:user-id data)
     :items   (:items data)
     :total   (:total data)
     :status  :pending
     :version (:events/version event)}))

(defmethod apply-event :order-confirmed
  [state event]
  (assoc state
    :status  :confirmed
    :version (:events/version event)))

(defmethod apply-event :order-shipped
  [state event]
  (let [data (cheshire.core/parse-string (:events/data event) true)]
    (assoc state
      :status          :shipped
      :tracking-number (:tracking-number data)
      :version         (:events/version event))))

(defmethod apply-event :order-delivered
  [state event]
  (assoc state
    :status     :delivered
    :version    (:events/version event)
    :delivered-at (java.time.Instant/now)))

(defmethod apply-event :order-cancelled
  [state event]
  (let [data (cheshire.core/parse-string (:events/data event) true)]
    (assoc state
      :status        :cancelled
      :cancel-reason (:reason data)
      :version       (:events/version event))))

;; Reconstruct aggregate from events
(defn load-order [db order-id]
  (let [events (event-store/load-events db order-id)]
    (when (seq events)
      (reduce apply-event nil events))))

;; Command handlers - produce events
(defmulti handle-command
  (fn [_state command] (:type command)))

(defmethod handle-command :confirm-order
  [state _command]
  (when-not (= :pending (:status state))
    (throw (ex-info "Can only confirm pending orders"
                     {:status (:status state)})))
  [{:type :order-confirmed
    :data {:confirmed-at (str (java.time.Instant/now))}}])

(defmethod handle-command :ship-order
  [state command]
  (when-not (= :confirmed (:status state))
    (throw (ex-info "Can only ship confirmed orders"
                     {:status (:status state)})))
  [{:type :order-shipped
    :data {:tracking-number (:tracking-number command)
           :shipped-at      (str (java.time.Instant/now))}}])

;; Execute command on aggregate
(defn execute-command! [db aggregate-id command]
  (let [current-state (load-order db aggregate-id)
        new-events    (handle-command current-state command)
        version       (or (:version current-state) 0)]
    (event-store/append-events! db aggregate-id "order" version new-events)
    new-events))
```

---

## ขั้นตอนที่ 1593: Projections

```clojure
(ns myapp.projections
  (:require [next.jdbc :as jdbc]))

;; Projection: rebuild read model from events
;; Run async, may be behind event store

;; Order summary projection
(defn project-order-summary! [db event]
  (let [data (cheshire.core/parse-string (:events/data event) true)]
    (case (keyword (:events/event_type event))
      :order-created
      (jdbc/execute-one! db
        ["INSERT INTO order_summaries
            (id, user_id, total, status, item_count, created_at)
          VALUES (?, ?, ?, 'pending', ?, NOW())"
         (:id data) (:user-id data) (:total data) (count (:items data))])

      :order-confirmed
      (jdbc/execute-one! db
        ["UPDATE order_summaries SET status = 'confirmed' WHERE id = ?"
         (:events/stream_id event)])

      :order-shipped
      (jdbc/execute-one! db
        ["UPDATE order_summaries
          SET status = 'shipped', tracking_number = ?
          WHERE id = ?"
         (:tracking-number data) (:events/stream_id event)])

      :order-delivered
      (jdbc/execute-one! db
        ["UPDATE order_summaries
          SET status = 'delivered', delivered_at = NOW()
          WHERE id = ?"
         (:events/stream_id event)])

      ;; Unknown event type - ignore
      nil)))

;; Projection runner
(defn run-projection! [db projection-fn projection-name]
  (let [checkpoint (or (:checkpoint
                         (jdbc/execute-one! db
                           ["SELECT checkpoint FROM projection_checkpoints WHERE name = ?"
                            projection-name]))
                       0)]
    (loop [last-id checkpoint]
      (let [events (jdbc/execute! db
                     ["SELECT * FROM events WHERE id > ? ORDER BY id ASC LIMIT 100"
                      last-id])]
        (when (seq events)
          (doseq [event events]
            (projection-fn db event))
          (let [new-checkpoint (:events/id (last events))]
            (jdbc/execute-one! db
              ["INSERT INTO projection_checkpoints (name, checkpoint)
                VALUES (?, ?)
                ON CONFLICT (name) DO UPDATE SET checkpoint = ?"
               projection-name new-checkpoint new-checkpoint])
            (recur new-checkpoint)))))))
```

---

## ขั้นตอนที่ 1594: Snapshots

```clojure
(ns myapp.snapshots
  (:require [next.jdbc :as jdbc]))

;; Schema:
;; CREATE TABLE snapshots (
;;   stream_id  UUID PRIMARY KEY,
;;   version    BIGINT NOT NULL,
;;   data       JSONB NOT NULL,
;;   created_at TIMESTAMPTZ DEFAULT NOW()
;; );

(defn save-snapshot! [db stream-id version state]
  (jdbc/execute-one! db
    ["INSERT INTO snapshots (stream_id, version, data)
      VALUES (?, ?, ?::jsonb)
      ON CONFLICT (stream_id)
      DO UPDATE SET version = ?, data = ?::jsonb, created_at = NOW()"
     stream-id version (cheshire.core/generate-string state)
     version (cheshire.core/generate-string state)]))

(defn load-snapshot [db stream-id]
  (when-let [snap (jdbc/execute-one! db
                    ["SELECT * FROM snapshots WHERE stream_id = ?" stream-id])]
    {:version (:snapshots/version snap)
     :state   (cheshire.core/parse-string (:snapshots/data snap) true)}))

;; Load aggregate with snapshot optimization
(defn load-aggregate-fast [db stream-id]
  (let [snapshot         (load-snapshot db stream-id)
        from-version     (or (:version snapshot) 0)
        remaining-events (event-store/load-events db stream-id
                           :from-version from-version)]
    (if snapshot
      (reduce apply-event (:state snapshot) remaining-events)
      (reduce apply-event nil remaining-events))))

;; Auto-snapshot every N events
(def snapshot-threshold 50)

(defn execute-command-with-snapshot! [db aggregate-id command]
  (let [new-events (execute-command! db aggregate-id command)
        state      (load-aggregate-fast db aggregate-id)]
    
    (when (zero? (mod (:version state) snapshot-threshold))
      (save-snapshot! db aggregate-id (:version state) state))
    
    new-events))
```

---

## ขั้นตอนที่ 1595: Event Bus

```clojure
(ns myapp.event-bus
  (:require [clojure.core.async :as async]))

;; In-process event bus for domain events
(defonce event-handlers (atom {}))

(defn subscribe! [event-type handler-fn]
  (swap! event-handlers update event-type
         (fnil conj []) handler-fn))

(defn publish! [event]
  (let [handlers (get @event-handlers (:type event) [])
        errors   (atom [])]
    (doseq [handler handlers]
      (try
        (handler event)
        (catch Exception e
          (swap! errors conj {:handler handler :error e}))))
    (when (seq @errors)
      (println "Event handler errors:" @errors))))

;; Async event bus
(defonce event-ch (async/chan (async/sliding-buffer 10000)))

(defn start-event-processor! []
  (async/go-loop []
    (when-let [event (async/<! event-ch)]
      (publish! event)
      (recur))))

(defn publish-async! [event]
  (async/put! event-ch event))

;; Outbox pattern: ensure events are published even if process crashes
(defn save-with-outbox! [db tx-fn events]
  (jdbc/with-transaction [tx db]
    (tx-fn tx)
    (doseq [event events]
      (jdbc/execute-one! tx
        ["INSERT INTO outbox (event_type, data, status)
          VALUES (?, ?::jsonb, 'pending')"
         (name (:type event))
         (cheshire.core/generate-string event)]))))

(defn process-outbox! [db]
  (let [events (jdbc/execute! db
                 ["SELECT * FROM outbox WHERE status = 'pending'
                   ORDER BY id ASC LIMIT 100 FOR UPDATE SKIP LOCKED"])]
    (doseq [event events]
      (try
        (publish! (cheshire.core/parse-string (:outbox/data event) true))
        (jdbc/execute-one! db
          ["UPDATE outbox SET status = 'processed', processed_at = NOW()
            WHERE id = ?"
           (:outbox/id event)])
        (catch Exception e
          (jdbc/execute-one! db
            ["UPDATE outbox SET status = 'failed', error = ? WHERE id = ?"
             (.getMessage e) (:outbox/id event)]))))))
```

---

## Project: E-Commerce Event-Sourced System

```clojure
(ns myapp.ecommerce.system)

;; Complete event-sourced order system

(def order-events
  #{:order-created :order-confirmed :order-shipped
    :order-delivered :order-cancelled :item-added :item-removed
    :payment-captured :payment-refunded})

(defn place-order! [db user-id items]
  (let [order-id (java.util.UUID/randomUUID)
        total    (reduce + (map #(* (:price %) (:qty %)) items))
        event    {:type :order-created
                  :data {:id       order-id
                         :user-id  user-id
                         :items    items
                         :total    total}}]
    (event-store/append-events! db order-id "order" 0 [event])
    (publish-async! (assoc event :stream-id order-id))
    order-id))

;; Event handlers
(subscribe! :order-created
  (fn [event]
    (project-order-summary! (:db @system) (:data event))
    (email/send-order-confirmation! (:user-email (:data event)))))

(subscribe! :order-shipped
  (fn [event]
    (project-order-summary! (:db @system) event)
    (email/send-shipping-notification!
      (:user-email (:data event))
      (:tracking-number (:data event)))))

;; Query side: fast reads from projections
(defn get-user-orders [db user-id status]
  (jdbc/execute! db
    ["SELECT * FROM order_summaries
      WHERE user_id = ?
      AND (? IS NULL OR status = ?)
      ORDER BY created_at DESC
      LIMIT 50"
     user-id status status]))

;; Rebuild projection from scratch
(defn rebuild-projection! [db]
  (jdbc/execute! db ["TRUNCATE order_summaries"])
  (jdbc/execute! db
    ["DELETE FROM projection_checkpoints WHERE name = 'order-summary'"])
  (run-projection! db project-order-summary! "order-summary"))
```

---

*Part 54 จาก 100+ | ขั้นตอน 1591-1620 จาก 1000+*
