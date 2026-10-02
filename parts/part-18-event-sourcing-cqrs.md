# Part 18: Event Sourcing และ CQRS
## ขั้นตอนที่ 511-540: Event Store, Projections, Command Handlers

---

## บทนำ

**Event Sourcing**: แทนที่จะเก็บ current state → เก็บ sequence of events
**CQRS**: แยก Command (write) และ Query (read) models

ประโยชน์:
- Full audit log ทุก state change
- สามารถ replay events เพื่อ rebuild state
- ง่ายต่อการ debug และ trace
- Scalable read models

---

## ขั้นตอนที่ 511: Event Store

```clojure
(ns eventsource.store)

;; Event ทุกตัวมี structure เดียวกัน
(defn make-event [aggregate-id event-type data]
  {:event/id            (java.util.UUID/randomUUID)
   :event/timestamp     (java.time.Instant/now)
   :event/type          event-type
   :event/version       1
   :aggregate/id        aggregate-id
   :aggregate/seq       nil   ; set by store
   :event/data          data})

;; Event Store protocol
(defprotocol EventStore
  (append-events! [this aggregate-id expected-version events])
  (load-events [this aggregate-id])
  (load-events-from [this aggregate-id from-seq])
  (subscribe [this event-type handler]))

;; In-memory event store (for development/testing)
(defrecord InMemoryEventStore [events-atom subscribers-atom]
  EventStore
  
  (append-events! [_ aggregate-id expected-version new-events]
    (swap! events-atom
           (fn [all-events]
             (let [existing  (filter #(= aggregate-id (:aggregate/id %)) all-events)
                   current-v (count existing)]
               (when (not= expected-version current-v)
                 (throw (ex-info "Optimistic concurrency conflict"
                                  {:expected expected-version
                                   :actual   current-v})))
               (let [versioned (map-indexed
                                  (fn [i event]
                                    (assoc event :aggregate/seq (+ current-v i 1)))
                                  new-events)]
                 (concat all-events versioned))))))
  
  (load-events [_ aggregate-id]
    (filter #(= aggregate-id (:aggregate/id %)) @events-atom))
  
  (load-events-from [_ aggregate-id from-seq]
    (->> @events-atom
         (filter #(= aggregate-id (:aggregate/id %)))
         (filter #(>= (:aggregate/seq %) from-seq)))))

(defn create-in-memory-store []
  (->InMemoryEventStore (atom []) (atom {})))
```

---

## ขั้นตอนที่ 512: Aggregate Reconstruction

```clojure
(ns eventsource.aggregate)

;; Rebuild state from events (event reduction)
(defmulti apply-event
  "Apply a single event to state, return new state"
  (fn [state event] (:event/type event)))

;; Default: skip unknown events (for forward compatibility)
(defmethod apply-event :default [state _] state)

;; BankAccount aggregate
(defmethod apply-event :account/opened
  [_ {:keys [event/data]}]
  {:id       (:account-id data)
   :owner    (:owner data)
   :balance  0M
   :status   :active
   :version  0})

(defmethod apply-event :account/credited
  [state {:keys [event/data aggregate/seq]}]
  (-> state
      (update :balance + (:amount data))
      (assoc :version seq)))

(defmethod apply-event :account/debited
  [state {:keys [event/data aggregate/seq]}]
  (-> state
      (update :balance - (:amount data))
      (assoc :version seq)))

(defmethod apply-event :account/closed
  [state {:keys [aggregate/seq]}]
  (assoc state :status :closed :version seq))

;; Rebuild aggregate from events
(defn rebuild [events]
  (reduce apply-event nil events))

;; Load aggregate from store
(defn load-aggregate [store aggregate-id]
  (let [events (load-events store aggregate-id)]
    (when (seq events)
      (rebuild events))))
```

---

## ขั้นตอนที่ 513: Command Handlers

```clojure
(ns eventsource.commands)

;; Command = intent to change state
(defn open-account-cmd [owner initial-deposit]
  {:command/type :open-account
   :owner        owner
   :amount       initial-deposit})

(defn deposit-cmd [account-id amount description]
  {:command/type :deposit
   :account-id   account-id
   :amount       amount
   :description  description})

(defn withdraw-cmd [account-id amount]
  {:command/type :withdraw
   :account-id   account-id
   :amount       amount})

;; Command handler: validate → generate events
(defmulti handle-command
  (fn [store command] (:command/type command)))

(defmethod handle-command :open-account
  [store {:keys [owner amount]}]
  (let [account-id (java.util.UUID/randomUUID)
        events     [(make-event account-id :account/opened
                      {:account-id account-id :owner owner})
                    (when (pos? amount)
                      (make-event account-id :account/credited
                        {:amount amount :description "Initial deposit"}))]]
    (append-events! store account-id 0 (remove nil? events))
    account-id))

(defmethod handle-command :deposit
  [store {:keys [account-id amount description]}]
  (let [account (or (load-aggregate store account-id)
                    (throw (ex-info "Account not found" {:id account-id})))]
    (when (= :closed (:status account))
      (throw (ex-info "Account is closed" {:id account-id})))
    (when (<= amount 0)
      (throw (ex-info "Amount must be positive" {:amount amount})))
    (let [event (make-event account-id :account/credited
                   {:amount amount :description description})]
      (append-events! store account-id (:version account) [event]))))

(defmethod handle-command :withdraw
  [store {:keys [account-id amount]}]
  (let [account (or (load-aggregate store account-id)
                    (throw (ex-info "Account not found" {:id account-id})))]
    (when (< (:balance account) amount)
      (throw (ex-info "Insufficient funds"
                       {:balance  (:balance account)
                        :requested amount})))
    (let [event (make-event account-id :account/debited
                   {:amount amount})]
      (append-events! store account-id (:version account) [event]))))
```

---

## ขั้นตอนที่ 514: CQRS - Read Models (Projections)

```clojure
(ns eventsource.projections)

;; Projections: derived read models from events

;; Account balance projection
(def account-balances (atom {}))

(defn update-balance-projection! [event]
  (case (:event/type event)
    :account/opened
    (swap! account-balances assoc
           (-> event :event/data :account-id)
           {:id      (-> event :event/data :account-id)
            :owner   (-> event :event/data :owner)
            :balance 0M
            :status  :active})
    
    :account/credited
    (swap! account-balances update-in
           [(-> event :aggregate/id) :balance]
           + (-> event :event/data :amount))
    
    :account/debited
    (swap! account-balances update-in
           [(-> event :aggregate/id) :balance]
           - (-> event :event/data :amount))
    
    :account/closed
    (swap! account-balances assoc-in
           [(-> event :aggregate/id) :status] :closed)
    
    nil))  ; unknown events - skip

;; Transaction history projection
(def transaction-history (atom {}))

(defn update-transaction-projection! [event]
  (when (#{:account/credited :account/debited} (:event/type event))
    (let [account-id (:aggregate/id event)
          tx {:type      (:event/type event)
               :amount    (-> event :event/data :amount)
               :description (-> event :event/data :description)
               :timestamp (:event/timestamp event)}]
      (swap! transaction-history update account-id (fnil conj []) tx))))

;; Rebuild all projections from scratch
(defn rebuild-projections! [store]
  (reset! account-balances {})
  (reset! transaction-history {})
  (let [all-events @(:events-atom store)]
    (doseq [event (sort-by :event/timestamp all-events)]
      (update-balance-projection! event)
      (update-transaction-projection! event))))
```

---

## ขั้นตอนที่ 515: Event Replay

```clojure
;; Event replay: rebuild state at any point in time!

(defn rebuild-at-time [events timestamp]
  (->> events
       (filter #(.isBefore (:event/timestamp %) timestamp))
       (reduce apply-event nil)))

;; Time travel debugging!
(defn account-state-at [store account-id timestamp]
  (let [events (load-events store account-id)]
    (rebuild-at-time events timestamp)))

;; Snapshot optimization: don't replay ALL events every time
(defn create-snapshot [aggregate]
  {:snapshot/id         (java.util.UUID/randomUUID)
   :snapshot/aggregate-id (:id aggregate)
   :snapshot/timestamp  (java.time.Instant/now)
   :snapshot/version    (:version aggregate)
   :snapshot/state      aggregate})

(defn load-with-snapshot [store snapshot-store aggregate-id]
  (let [snapshot (snapshot-store/latest snapshot-store aggregate-id)]
    (if snapshot
      ;; Replay only events after snapshot
      (let [events (load-events-from store aggregate-id (inc (:snapshot/version snapshot)))]
        (reduce apply-event (:snapshot/state snapshot) events))
      ;; Replay all events
      (load-aggregate store aggregate-id))))
```

---

## ขั้นตอนที่ 516: PostgreSQL Event Store

```clojure
(ns eventsource.postgres-store
  (:require [next.jdbc :as jdbc]))

;; DDL
;; CREATE TABLE events (
;;   id UUID PRIMARY KEY,
;;   aggregate_id UUID NOT NULL,
;;   aggregate_seq BIGINT NOT NULL,
;;   event_type VARCHAR(200) NOT NULL,
;;   event_version INT NOT NULL DEFAULT 1,
;;   data JSONB NOT NULL,
;;   metadata JSONB,
;;   created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
;;   UNIQUE(aggregate_id, aggregate_seq)
;; );
;; CREATE INDEX idx_events_aggregate ON events(aggregate_id, aggregate_seq);
;; CREATE INDEX idx_events_type ON events(event_type);

(defrecord PostgresEventStore [ds]
  EventStore
  
  (append-events! [_ aggregate-id expected-version new-events]
    (jdbc/with-transaction [tx ds]
      ;; Optimistic locking check
      (let [{:keys [max_seq]} (jdbc/execute-one! tx
                                ["SELECT MAX(aggregate_seq) as max_seq
                                  FROM events WHERE aggregate_id = ?"
                                 (str aggregate-id)])]
        (when (not= expected-version (or max_seq 0))
          (throw (ex-info "Concurrency conflict"
                           {:expected expected-version
                            :actual   (or max_seq 0)})))
        
        ;; Append events
        (doseq [[i event] (map-indexed vector new-events)]
          (jdbc/execute! tx
            ["INSERT INTO events (id, aggregate_id, aggregate_seq, event_type, data)
              VALUES (?, ?, ?, ?, ?::jsonb)"
             (str (:event/id event))
             (str aggregate-id)
             (+ expected-version i 1)
             (name (:event/type event))
             (cheshire.core/generate-string (:event/data event))])))))
  
  (load-events [_ aggregate-id]
    (->> (jdbc/execute! ds
           ["SELECT * FROM events WHERE aggregate_id = ? ORDER BY aggregate_seq"
            (str aggregate-id)])
         (map deserialize-event))))
```

---

## ขั้นตอนที่ 517: CQRS API

```clojure
(ns eventsource.api
  (:require [reitit.ring :as ring]))

;; Command API (write side)
(defn post-command-handler [store]
  (fn [{{:keys [command-type]} :path-params
        body :body-params}]
    (try
      (let [command (assoc body :command/type (keyword command-type))
            result  (handle-command store command)]
        {:status 200 :body {:success true :id (str result)}})
      (catch clojure.lang.ExceptionInfo e
        (let [data (ex-data e)]
          {:status (or (:status data) 400)
           :body   {:error   (.getMessage e)
                    :details (dissoc data :status)}})))))

;; Query API (read side)
(defn get-account-handler [_]
  (fn [{{:keys [id]} :path-params}]
    (if-let [account (get @account-balances (java.util.UUID/fromString id))]
      {:status 200 :body account}
      {:status 404 :body {:error "Account not found"}})))

(defn get-transactions-handler [_]
  (fn [{{:keys [id]} :path-params}]
    {:status 200
     :body {:transactions (get @transaction-history (java.util.UUID/fromString id) [])}}))

;; Routes
(def routes
  [["/commands/:command-type" {:post (post-command-handler event-store)}]
   ["/accounts/:id"
    ["" {:get (get-account-handler nil)}]
    ["/transactions" {:get (get-transactions-handler nil)}]]])
```

---

## Project Exercise: Bank System ด้วย Event Sourcing

```clojure
;; Complete bank system demo

(comment
  (def store (create-in-memory-store))
  
  ;; Open accounts
  (def alice-id (handle-command store (open-account-cmd "Alice" 1000M)))
  (def bob-id   (handle-command store (open-account-cmd "Bob" 500M)))
  
  ;; Transactions
  (handle-command store (deposit-cmd alice-id 500M "Salary"))
  (handle-command store (withdraw-cmd alice-id 200M))
  (handle-command store (deposit-cmd bob-id 300M "Birthday gift"))
  
  ;; Rebuild projections
  (rebuild-projections! store)
  
  ;; Query current state
  (get @account-balances alice-id)
  ;; => {:id #uuid "...", :owner "Alice", :balance 1300M, :status :active}
  
  (get @transaction-history alice-id)
  ;; => [{:type :account/credited, :amount 1000M, ...}
  ;;     {:type :account/credited, :amount 500M, ...}
  ;;     {:type :account/debited, :amount 200M, ...}]
  
  ;; Time travel!
  (account-state-at store alice-id (java.time.Instant/parse "2024-01-01T12:00:00Z"))
  ;; => state at that exact point in time
  )
```

---

*Part 18 จาก 100+ | ขั้นตอน 511-540 จาก 1000+*
