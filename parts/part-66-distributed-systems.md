# Part 66: Distributed Systems
## ขั้นตอนที่ 1951-1980: CAP Theorem, Consensus, Leader Election, Distributed Locks

---

## บทนำ

Distributed Systems concepts ในทางปฏิบัติ:
- **CAP Theorem** - Consistency, Availability, Partition Tolerance
- **Consensus** - ทุก node เห็นตรงกัน
- **Leader election** - เลือก primary node
- **Distributed locks** - coordination ข้าม services
- **CRDT** - Conflict-free Replicated Data Types

---

## ขั้นตอนที่ 1951: Distributed Locking ด้วย Redis

```clojure
(ns myapp.distributed-lock
  (:require [taoensso.carmine :as car]))

;; Redlock algorithm (simplified single-instance)
(defn acquire-lock!
  "Acquire distributed lock. Returns lock token or nil."
  [redis-pool lock-name ttl-ms]
  (let [token (str (java.util.UUID/randomUUID))]
    (when (= "OK"
              (car/wcar redis-pool
                (car/set lock-name token "NX" "PX" ttl-ms)))
      token)))

(defn release-lock!
  "Release lock only if we own it (using Lua script for atomicity)."
  [redis-pool lock-name token]
  (let [lua-script
        "if redis.call('get', KEYS[1]) == ARGV[1] then
           return redis.call('del', KEYS[1])
         else
           return 0
         end"]
    (= 1 (car/wcar redis-pool
           (car/eval lua-script 1 lock-name token)))))

(defmacro with-distributed-lock
  [redis-pool lock-name ttl-ms & body]
  `(let [token# (acquire-lock! ~redis-pool ~lock-name ~ttl-ms)]
     (if token#
       (try
         (do ~@body)
         (finally
           (release-lock! ~redis-pool ~lock-name token#)))
       (throw (ex-info "Could not acquire lock"
                        {:lock ~lock-name})))))

;; Usage: ensure only one instance processes daily batch
(defn run-daily-report! [db redis-pool]
  (with-distributed-lock redis-pool "daily-report-lock" 60000
    (println "Running daily report...")
    (generate-daily-report! db)))

;; Lock with retry
(defn acquire-lock-with-retry! [redis-pool lock-name ttl-ms
                                  max-wait-ms retry-interval-ms]
  (loop [waited 0]
    (or (acquire-lock! redis-pool lock-name ttl-ms)
        (when (< waited max-wait-ms)
          (Thread/sleep retry-interval-ms)
          (recur (+ waited retry-interval-ms))))))
```

---

## ขั้นตอนที่ 1952: Optimistic Concurrency Control

```clojure
;; Version-based optimistic locking
(defn update-with-occ!
  "Update entity only if version matches. Retries on conflict."
  [db entity-id expected-version update-fn max-retries]
  (loop [retries 0]
    (if (> retries max-retries)
      (throw (ex-info "Too many concurrent updates"
                       {:entity-id entity-id}))
      (let [current (db/get-by-id db entity-id)
            _ (when (not= (:version current) expected-version)
                (throw (ex-info "Version mismatch - read stale data"
                                 {:expected expected-version
                                  :actual   (:version current)})))
            updates    (update-fn current)
            result     (jdbc/execute-one! db
                          ["UPDATE entities
                            SET data = ?::jsonb, version = version + 1
                            WHERE id = ? AND version = ?
                            RETURNING *"
                           (json/generate-string updates)
                           entity-id expected-version])]
        (if result
          result
          ;; Someone else updated - retry
          (do
            (Thread/sleep (rand-int 100))
            (recur (inc retries))))))))

;; Compare-and-swap on atoms (in-memory OCC)
(defn cas-update!
  "Compare-and-swap style update."
  [state-atom key expected-val new-val]
  (loop []
    (let [current @state-atom]
      (if (= (get current key) expected-val)
        (if (compare-and-set! state-atom current
                               (assoc current key new-val))
          true
          (recur))
        false))))
```

---

## ขั้นตอนที่ 1953: Leader Election

```clojure
(ns myapp.leader-election
  (:require [taoensso.carmine :as car]))

;; Simple leader election using Redis
(defonce leader-state
  (atom {:is-leader? false
          :leader-id  nil}))

(defn try-become-leader! [redis-pool node-id ttl-seconds]
  (let [result (car/wcar redis-pool
                  (car/set "app:leader" node-id "NX" "EX" ttl-seconds))]
    (= "OK" result)))

(defn renew-leadership! [redis-pool node-id ttl-seconds]
  (let [current-leader (car/wcar redis-pool (car/get "app:leader"))]
    (when (= current-leader node-id)
      (car/wcar redis-pool (car/expire "app:leader" ttl-seconds))
      true)))

(defn start-leader-election! [redis-pool node-id]
  (let [stop-ch (clojure.core.async/chan)]
    (clojure.core.async/go-loop []
      (clojure.core.async/alt!
        (clojure.core.async/timeout 5000)
        ([_]
         (if (:is-leader? @leader-state)
           ;; Already leader: renew
           (if (renew-leadership! redis-pool node-id 15)
             (swap! leader-state assoc :is-leader? true)
             (swap! leader-state assoc :is-leader? false))
           
           ;; Not leader: try to become one
           (if (try-become-leader! redis-pool node-id 15)
             (do
               (println "Became leader:" node-id)
               (swap! leader-state assoc :is-leader? true :leader-id node-id)
               (on-become-leader!))
             (do
               (let [leader (car/wcar redis-pool (car/get "app:leader"))]
                 (swap! leader-state assoc :is-leader? false :leader-id leader)))))
         (recur))
        
        stop-ch ([_] (println "Leader election stopped"))))
    
    stop-ch))

;; Execute only on leader
(defn on-leader-only [f]
  (when (:is-leader? @leader-state)
    (f)))
```

---

## ขั้นตอนที่ 1954: CRDT (Conflict-free Replicated Data Types)

```clojure
(ns myapp.crdt)

;; G-Counter: grow-only counter (only increment)
;; Good for: page views, event counts, distributed counters
(defrecord GCounter [counts node-id]
  Object
  (toString [_] (str "GCounter{" counts "}")))

(defn g-counter [node-id]
  (->GCounter {node-id 0} node-id))

(defn g-counter-increment [counter]
  (update-in counter [:counts (:node-id counter)] inc))

(defn g-counter-value [counter]
  (reduce + (vals (:counts counter))))

(defn g-counter-merge [c1 c2]
  (->GCounter
    (merge-with max (:counts c1) (:counts c2))
    (:node-id c1)))

;; PN-Counter: positive-negative (increment and decrement)
(defrecord PNCounter [pos neg node-id])

(defn pn-counter [node-id]
  (->PNCounter (g-counter node-id) (g-counter node-id) node-id))

(defn pn-counter-increment [c]
  (update c :pos g-counter-increment))

(defn pn-counter-decrement [c]
  (update c :neg g-counter-increment))

(defn pn-counter-value [c]
  (- (g-counter-value (:pos c))
     (g-counter-value (:neg c))))

(defn pn-counter-merge [c1 c2]
  (->PNCounter
    (g-counter-merge (:pos c1) (:pos c2))
    (g-counter-merge (:neg c1) (:neg c2))
    (:node-id c1)))

;; LWW-Register: Last-Write-Wins Register
(defrecord LWWRegister [value timestamp node-id])

(defn lww-set [reg value]
  (->LWWRegister value (System/currentTimeMillis) (:node-id reg)))

(defn lww-merge [r1 r2]
  (if (> (:timestamp r1) (:timestamp r2)) r1 r2))

;; OR-Set: Observed-Remove Set (add and remove elements)
(defrecord ORSet [elements tombstones])

(defn or-set-add [s element]
  (update s :elements assoc element (str (java.util.UUID/randomUUID))))

(defn or-set-remove [s element]
  (if-let [tag (get (:elements s) element)]
    (-> s
        (update :elements dissoc element)
        (update :tombstones conj tag))
    s))

(defn or-set-contains? [s element]
  (contains? (:elements s) element))

(defn or-set-merge [s1 s2]
  (let [merged-tombstones (into (:tombstones s1) (:tombstones s2))
        merged-elements   (merge (:elements s1) (:elements s2))
        cleaned-elements  (into {}
                             (remove (fn [[_ tag]]
                                       (contains? merged-tombstones tag))
                                     merged-elements))]
    (->ORSet cleaned-elements merged-tombstones)))
```

---

## ขั้นตอนที่ 1955: Distributed Tracing Correlation

```clojure
;; Trace context propagation
(ns myapp.tracing)

;; Thread-local trace context
(def ^:dynamic *trace-context* nil)

(defn with-trace [trace-id span-id f]
  (binding [*trace-context* {:trace-id trace-id
                               :span-id  span-id}]
    (f)))

;; Inject trace headers into HTTP requests
(defn inject-trace-headers [headers]
  (if *trace-context*
    (assoc headers
      "X-Trace-Id" (:trace-id *trace-context*)
      "X-Span-Id"  (str (java.util.UUID/randomUUID))
      "X-Parent-Span-Id" (:span-id *trace-context*))
    headers))

;; Extract trace from incoming request
(defn extract-trace [headers]
  {:trace-id (get headers "x-trace-id" (str (java.util.UUID/randomUUID)))
   :span-id  (get headers "x-span-id"  (str (java.util.UUID/randomUUID)))
   :parent   (get headers "x-parent-span-id")})

;; Middleware to propagate trace context
(defn wrap-trace-context [handler]
  (fn [request]
    (let [trace (extract-trace (:headers request))]
      (with-trace (:trace-id trace) (:span-id trace)
        #(handler (assoc request :trace-context trace))))))
```

---

## Project: Distributed Rate Limiter

```clojure
(ns myapp.distributed-rate-limiter
  (:require [taoensso.carmine :as car]))

;; Sliding window rate limiter using Redis sorted sets
(defn check-rate-limit!
  "Returns {:allowed? bool :remaining int :reset-at long}"
  [redis-pool key limit window-seconds]
  (let [now       (System/currentTimeMillis)
        window-ms (* window-seconds 1000)
        window-start (- now window-ms)
        
        lua-script
        "local key = KEYS[1]
         local now = tonumber(ARGV[1])
         local window_start = tonumber(ARGV[2])
         local limit = tonumber(ARGV[3])
         local window_ms = tonumber(ARGV[4])

         -- Remove old entries
         redis.call('ZREMRANGEBYSCORE', key, '-inf', window_start)

         -- Count current entries
         local count = redis.call('ZCARD', key)

         if count < limit then
           -- Add new entry
           redis.call('ZADD', key, now, now)
           redis.call('PEXPIRE', key, window_ms)
           return {1, limit - count - 1, now + window_ms}
         else
           -- Get oldest entry to know when window resets
           local oldest = redis.call('ZRANGE', key, 0, 0, 'WITHSCORES')
           local reset_at = tonumber(oldest[2]) + window_ms
           return {0, 0, reset_at}
         end"
        
        [allowed remaining reset-at]
        (car/wcar redis-pool
          (car/eval lua-script 1 key now window-start limit window-ms))]
    
    {:allowed?  (= 1 allowed)
     :remaining remaining
     :reset-at  reset-at
     :retry-after (when (= 0 allowed)
                    (/ (- reset-at now) 1000.0))}))

;; Middleware
(defn wrap-distributed-rate-limit [handler redis-pool rate-fn]
  (fn [request]
    (let [{:keys [limit window-seconds key]} (rate-fn request)
          result (check-rate-limit! redis-pool key limit window-seconds)]
      (if (:allowed? result)
        (-> (handler request)
            (assoc-in [:headers "X-RateLimit-Remaining"]
                       (str (:remaining result)))
            (assoc-in [:headers "X-RateLimit-Reset"]
                       (str (:reset-at result))))
        {:status  429
         :headers {"Retry-After"          (str (:retry-after result))
                    "X-RateLimit-Remaining" "0"}
         :body    {:error "Rate limit exceeded"}}))))
```

---

*Part 66 จาก 100+ | ขั้นตอน 1951-1980 จาก 1000+*
