# Part 98: Production Excellence
## ขั้นตอนที่ 2911-2940: Chaos Engineering, Load Testing, SRE Practices, Runbooks

---

## บทนำ

Production excellence ระดับ SRE:
- **Chaos engineering** - inject failures to build resilience
- **Load testing** - gatling/k6 patterns
- **SLI/SLO/SLA** - reliability measurements
- **Runbooks as code** - executable incident response
- **Cost optimization** - efficient resource usage

---

## ขั้นตอนที่ 2911: Chaos Engineering Framework

```clojure
(ns myapp.chaos)

;; Chaos experiments: deliberately inject failures

(defprotocol ChaosExperiment
  (experiment-name [this])
  (hypothesis [this])
  (inject-fault! [this target])
  (verify-steady-state [this metrics])
  (rollback! [this]))

;; Latency injection
(defrecord LatencyInjector [delay-ms target-pct]
  ChaosExperiment
  (experiment-name [_] "latency-injection")
  (hypothesis [_] "System remains responsive under 200ms added latency")
  
  (inject-fault! [_ handler]
    (fn [request]
      (when (< (rand) (/ target-pct 100))
        (Thread/sleep delay-ms))
      (handler request)))
  
  (verify-steady-state [_ metrics]
    (and (< (:error-rate metrics) 0.01)
         (< (:p99-latency metrics) 5000)))
  
  (rollback! [_]
    (println "Latency injection removed")))

;; Error injection
(defrecord ErrorInjector [error-pct error-code]
  ChaosExperiment
  (experiment-name [_] "error-injection")
  (hypothesis [_] "Circuit breaker opens when 50% errors injected")
  
  (inject-fault! [_ handler]
    (fn [request]
      (if (< (rand) (/ error-pct 100))
        {:status error-code :body {:error "Chaos injected"}}
        (handler request))))
  
  (verify-steady-state [_ metrics]
    (< (:error-rate metrics) (/ error-pct 100 2)))  ; CB should absorb some
  
  (rollback! [_]
    (println "Error injection removed")))

;; Chaos runner
(defn run-experiment! [experiment target observe-fn duration-ms]
  (let [original target
        start    (System/currentTimeMillis)]
    (println "Starting chaos experiment:" (experiment-name experiment))
    (println "Hypothesis:" (hypothesis experiment))
    
    ;; Inject fault
    (inject-fault! experiment target)
    
    ;; Observe for duration
    (Thread/sleep duration-ms)
    (let [metrics (observe-fn)]
      
      ;; Verify
      (let [passed? (verify-steady-state experiment metrics)]
        (println "Hypothesis" (if passed? "CONFIRMED" "REJECTED"))
        (println "Metrics:" metrics)
        
        ;; Always rollback
        (rollback! experiment)
        
        {:passed?  passed?
         :metrics  metrics
         :duration (- (System/currentTimeMillis) start)}))))
```

---

## ขั้นตอนที่ 2912: SLI/SLO Tracking

```clojure
;; Service Level Indicators and Objectives

(def slos
  {:availability {:target  99.9  ; 99.9%
                   :window  :rolling-30d
                   :measure :success-rate}
   
   :latency-p99  {:target  500    ; 500ms
                   :window  :rolling-7d
                   :measure :p99-request-latency}
   
   :error-rate   {:target  0.1   ; 0.1%
                   :window  :rolling-24h
                   :measure :http-error-rate}})

;; Error budget calculation
(defn error-budget [slo current-window-stats]
  (let [target  (:target slo)
        actual  (get current-window-stats (:measure slo))
        allowed-errors (* (- 1 (/ target 100)) (:window-size-requests current-window-stats))]
    {:target         target
     :actual         actual
     :budget-total   allowed-errors
     :budget-used    (:error-count current-window-stats)
     :budget-remaining (- allowed-errors (:error-count current-window-stats))
     :burn-rate      (/ (:error-count current-window-stats) allowed-errors)}))

;; Alert when burn rate too high
(defn check-burn-rate-alert! [db notification-service]
  (let [current-stats (get-current-slo-stats db)
        budgets (map (fn [[slo-name slo]]
                       {:name   slo-name
                        :budget (error-budget slo current-stats)})
                     slos)]
    (doseq [{:keys [name budget]} budgets]
      (when (> (:burn-rate budget) 2.0)
        (notify! notification-service
          {:severity :page
           :summary  (str "SLO burn rate critical: " name)
           :details  budget})))))

;; SLO dashboard metrics
(defn slo-compliance-report [db from-date to-date]
  (for [[slo-name slo] slos]
    {:slo        slo-name
     :target     (:target slo)
     :actual     (measure-slo db slo from-date to-date)
     :compliant? (>= (measure-slo db slo from-date to-date) (:target slo))}))
```

---

## ขั้นตอนที่ 2913: Runbooks as Code

```clojure
;; Executable runbooks for incident response

(defprotocol RunbookStep
  (step-description [this])
  (execute-step! [this context])
  (can-automate? [this]))

;; Restart a pod in Kubernetes
(defrecord RestartPodStep [namespace deployment]
  RunbookStep
  (step-description [_]
    (str "Restart deployment " deployment " in " namespace))
  (can-automate? [_] true)
  (execute-step! [_ _]
    (kubectl/rollout-restart! namespace deployment)))

;; Scale up deployment
(defrecord ScaleDeploymentStep [namespace deployment replicas]
  RunbookStep
  (step-description [_]
    (str "Scale " deployment " to " replicas " replicas"))
  (can-automate? [_] true)
  (execute-step! [_ _]
    (kubectl/scale! namespace deployment replicas)))

;; Check database connections
(defrecord CheckDbConnectionsStep [db max-connections]
  RunbookStep
  (step-description [_] "Check database connection pool")
  (can-automate? [_] true)
  (execute-step! [_ _]
    (let [stats (db-pool-stats db)]
      {:active  (:active stats)
       :idle    (:idle stats)
       :pending (:pending stats)
       :ok?     (< (:active stats) max-connections)})))

;; Runbook: Handle High Latency
(def high-latency-runbook
  {:name        "Handle High API Latency"
   :triggers    [{:alert "p99_latency > 2000ms"}]
   :steps       [(->CheckDbConnectionsStep (current-db) 90)
                 (->RestartPodStep "production" "api-gateway")
                 (->ScaleDeploymentStep "production" "api-service" 5)]})

;; Execute runbook
(defn execute-runbook! [runbook context]
  (println "Executing runbook:" (:name runbook))
  (reduce
    (fn [results step]
      (println "Step:" (step-description step))
      (if (can-automate? step)
        (do
          (let [result (execute-step! step context)]
            (println "Result:" result)
            (conj results {:step (step-description step) :result result})))
        (do
          (println "MANUAL STEP REQUIRED:" (step-description step))
          (println "Press Enter when complete...")
          (read-line)
          (conj results {:step (step-description step) :result :manual}))))
    [] (:steps runbook)))
```

---

## ขั้นตอนที่ 2914: Cost Optimization Patterns

```clojure
;; Measure and optimize resource costs

;; Track per-request cost
(defn wrap-cost-tracking [handler cost-per-ms]
  (fn [request]
    (let [start   (System/nanoTime)
          result  (handler request)
          elapsed (/ (- (System/nanoTime) start) 1e6)
          cost    (* elapsed cost-per-ms)]
      (record-cost! {:endpoint (:uri request)
                     :method   (name (:request-method request))
                     :latency  elapsed
                     :cost     cost})
      result)))

;; Lazy evaluation: don't compute until needed
(defn expensive-report [db from to]
  ;; Return lazy seq instead of computing all at once
  (lazy-seq
    (let [batch (fetch-report-batch db from to 100)]
      (when (seq batch)
        (concat batch
                (lazy-seq (expensive-report db (:id (last batch)) to)))))))

;; Batch reads to reduce DB calls
(defn batch-load [db ids]
  (let [results (jdbc/execute! db
                  ["SELECT * FROM items WHERE id = ANY(?)"
                   (into-array String ids)])]
    (index-by :id results)))

;; Cache expensive computations
(def expensive-calc
  (timed-memoize
    (fn [params] (actually-expensive-computation params))
    (* 5 60 1000)))  ; Cache 5 minutes

;; Connection pooling tuning
(defn optimal-pool-size
  "Calculate optimal pool size using Little's Law"
  [target-rps avg-latency-ms]
  (let [throughput    (/ target-rps 1000)  ; requests/ms
        concurrency   (* throughput avg-latency-ms)  ; Little's Law
        safety-factor 1.1]
    (int (* concurrency safety-factor))))
```

---

*Part 98 จาก 100 | ขั้นตอน 2911-2940 จาก 3000*
