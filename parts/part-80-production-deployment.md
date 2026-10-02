# Part 80: Production Deployment
## ขั้นตอนที่ 2371-2400: Zero-Downtime Deploy, Blue-Green, Canary, Rollback

---

## บทนำ

Production deployment strategies:
- **Blue-Green** - สลับ traffic ระหว่างสอง environments
- **Canary releases** - rollout ทีละน้อย
- **Rolling updates** - update ทีละ pod
- **Feature flags** - decouple deploy จาก release
- **Database migrations** - zero-downtime schema changes

---

## ขั้นตอนที่ 2371: Feature Flags

```clojure
(ns myapp.feature-flags)

;; Feature flag system
(defprotocol FeatureFlags
  (enabled? [this flag-name context])
  (get-variant [this flag-name context]))

;; Simple in-memory flags (for development)
(defrecord InMemoryFlags [flags]
  FeatureFlags
  (enabled? [_ flag-name _ctx]
    (boolean (get @flags flag-name)))
  (get-variant [_ flag-name _ctx]
    (get @flags flag-name)))

;; Redis-backed flags (for production)
(defrecord RedisFlags [redis-pool]
  FeatureFlags
  (enabled? [_ flag-name context]
    (let [flag  (json/parse-string
                   (car/wcar redis-pool
                     (car/get (str "feature:" flag-name)))
                   true)
          rules (:rules flag [])]
      
      (and (:enabled flag)
           (or (empty? rules)
               (some #(match-rule? % context) rules)))))
  
  (get-variant [_ flag-name context]
    (let [flag     (json/parse-string
                     (car/wcar redis-pool
                       (car/get (str "feature:" flag-name)))
                     true)
          variants (:variants flag)
          user-id  (:user-id context)]
      
      (when variants
        ;; Deterministic assignment based on user-id
        (let [hash    (Math/abs (hash user-id))
              bucket  (mod hash 100)
              variant (first (filter #(< bucket (:threshold %)) variants))]
          (:name variant))))))

;; Match rule
(defn match-rule? [rule context]
  (case (:type rule)
    :user-id    (contains? (set (:users rule)) (:user-id context))
    :percentage (< (mod (Math/abs (hash (:user-id context))) 100)
                    (:percentage rule))
    :email-domain (= (email-domain (:email context)) (:domain rule))
    false))

;; Middleware to inject flags
(defn wrap-feature-flags [handler flags]
  (fn [request]
    (handler (assoc request :flags flags))))

;; Usage in handler
(defn checkout-handler [request]
  (let [flags (:flags request)
        user  (:user request)]
    (if (enabled? flags :new-checkout-flow {:user-id (:id user)})
      (new-checkout-flow request)
      (old-checkout-flow request))))
```

---

## ขั้นตอนที่ 2372: Zero-Downtime Database Migrations

```clojure
(ns myapp.migrations)

;; Database migration principles for zero-downtime:
;; 1. Never drop columns immediately
;; 2. Add columns as nullable first
;; 3. Backfill data
;; 4. Add NOT NULL constraint
;; 5. Remove old columns in next deploy

;; Migration state machine
(def migration-phases
  {:add-nullable-column    "Step 1: Add column (nullable, has default)"
   :backfill-data          "Step 2: Backfill existing rows"
   :add-not-null-constraint "Step 3: Add NOT NULL (DB validates all rows filled)"
   :remove-old-column      "Step 4: Remove old column (next deploy)"})

;; Example: rename products.description to products.long_description
;; Phase 1 (Deploy A): Add new column alongside old
(defn migration-001-phase-1 [db]
  (jdbc/execute! db
    ["ALTER TABLE products
      ADD COLUMN IF NOT EXISTS long_description TEXT"]))

;; Phase 2 (Deploy A): Backfill new column from old
(defn migration-001-phase-2 [db]
  ;; Process in batches to avoid locking
  (loop [offset 0]
    (let [updated (jdbc/execute-one! db
                    ["UPDATE products
                      SET long_description = description
                      WHERE id IN (
                        SELECT id FROM products
                        WHERE long_description IS NULL
                        LIMIT 1000
                      )
                      RETURNING COUNT(*)"])]
      (when (pos? (:count updated))
        (println "Backfilled" (:count updated) "rows")
        (Thread/sleep 100)  ; Don't hammer DB
        (recur (+ offset 1000))))))

;; Phase 3 (Deploy B): App writes to BOTH columns
;; (done in application code, not migration)

;; Phase 4 (Deploy C): Remove old column
(defn migration-001-phase-4 [db]
  (jdbc/execute! db
    ["ALTER TABLE products DROP COLUMN IF EXISTS description"]))

;; Migration runner with locking
(defn run-migration! [db migration-fn migration-id]
  (jdbc/with-transaction [tx db]
    ;; Advisory lock to prevent concurrent migrations
    (jdbc/execute! tx ["SELECT pg_advisory_xact_lock(12345)"])
    
    ;; Check if already run
    (let [already-run? (jdbc/execute-one! tx
                         ["SELECT 1 FROM schema_migrations WHERE id = ?"
                          migration-id])]
      (when-not already-run?
        (migration-fn tx)
        (jdbc/execute! tx
          ["INSERT INTO schema_migrations (id, ran_at) VALUES (?, NOW())"
           migration-id])
        (println "Migration" migration-id "completed")))))
```

---

## ขั้นตอนที่ 2373: Canary Deployment

```clojure
;; Traffic splitting for canary releases
(ns myapp.deployment.canary)

;; Request routing based on canary percentage
(defn canary-router [canary-percentage canary-handler stable-handler]
  (fn [request]
    (let [user-id  (get-in request [:user :id] "anonymous")
          ;; Consistent hashing: same user always goes to same version
          hash-val (mod (Math/abs (hash user-id)) 100)]
      (if (< hash-val canary-percentage)
        (canary-handler request)
        (stable-handler request)))))

;; Gradual rollout: increase canary % over time
(defonce rollout-config
  (atom {:percentage 0 :status :stopped}))

(defn start-gradual-rollout! [target-percentage step-pct interval-minutes]
  (reset! rollout-config {:percentage 0 :status :running})
  (future
    (loop [current 0]
      (when (and (< current target-percentage)
                 (= :running (:status @rollout-config)))
        (let [next-pct (min target-percentage (+ current step-pct))]
          (swap! rollout-config assoc :percentage next-pct)
          (println "Canary percentage:" next-pct "%")
          
          ;; Monitor error rates before proceeding
          (let [error-rate (get-canary-error-rate)]
            (if (> error-rate 0.01)  ; >1% errors: rollback
              (do
                (println "High error rate, rolling back canary!")
                (rollback-canary!))
              (do
                (Thread/sleep (* interval-minutes 60000))
                (recur next-pct)))))))))

(defn rollback-canary! []
  (reset! rollout-config {:percentage 0 :status :rolled-back})
  (println "Canary rolled back to 0%"))
```

---

## ขั้นตอนที่ 2374: Health Checks and Graceful Shutdown

```clojure
;; Production-grade graceful shutdown
(ns myapp.lifecycle)

(defonce system-state (atom :starting))

(defn setup-graceful-shutdown! [system]
  (.addShutdownHook
    (Runtime/getRuntime)
    (Thread.
      (fn []
        (println "Shutdown signal received")
        
        ;; 1. Stop accepting new requests
        (reset! system-state :stopping)
        
        ;; 2. Wait for in-flight requests to complete
        (println "Waiting for in-flight requests...")
        (Thread/sleep 5000)
        
        ;; 3. Stop background workers
        (when-let [workers (:workers system)]
          (doseq [worker workers]
            (.interrupt worker)))
        
        ;; 4. Flush any pending data
        (when-let [cache (:cache system)]
          (flush-cache! cache))
        
        ;; 5. Close connections
        (println "Closing connections...")
        (ig/halt! system)
        
        (println "Shutdown complete")))))

;; Middleware to reject requests during shutdown
(defn wrap-shutdown-check [handler]
  (fn [request]
    (if (= :stopping @system-state)
      {:status  503
       :headers {"Retry-After" "30"}
       :body    {:error "Service shutting down"}}
      (handler request))))

;; Kubernetes-aware readiness probe
(defn readiness-check []
  (and (= :running @system-state)
       (db-connected?)
       (cache-connected?)))
```

---

## ขั้นตอนที่ 2375: Production Monitoring Setup

```clojure
;; Complete production monitoring config
(ns myapp.production)

(defn production-config []
  {:metrics
   {:prometheus-port 9090
    :custom-metrics  [:orders-per-second
                       :revenue-per-minute
                       :cart-abandonment-rate]}
   
   :logging
   {:format  :json
    :level   :info
    :outputs [:stdout]
    :fields  {:service "myapp"
              :version (get-app-version)
              :env     "production"}}
   
   :tracing
   {:enabled?     true
    :sample-rate  0.1  ; 10% sampling
    :exporter     :jaeger
    :endpoint     "http://jaeger:14268/api/traces"}
   
   :alerting
   {:slack-webhook (System/getenv "SLACK_ALERT_WEBHOOK")
    :pagerduty-key (System/getenv "PAGERDUTY_KEY")
    :rules         [{:name       "High error rate"
                     :query      "rate(errors[5m]) > 0.05"
                     :severity   :critical
                     :notify     [:pagerduty :slack]}
                    {:name       "High latency"
                     :query      "p99(latency) > 1000"
                     :severity   :warning
                     :notify     [:slack]}]}})
```

---

*Part 80 จาก 100+ | ขั้นตอน 2371-2400 จาก 1000+*
