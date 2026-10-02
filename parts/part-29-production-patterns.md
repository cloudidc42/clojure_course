# Part 29: Production Patterns และ Best Practices
## ขั้นตอนที่ 841-870: Configuration, Error Handling, Observability, Graceful Shutdown

---

## บทนำ

Production-ready Clojure application ต้องมี:
- **12-Factor App** principles
- **Graceful Shutdown** ไม่ drop requests
- **Health Checks** สำหรับ orchestration
- **Circuit Breaker** ป้องกัน cascading failures
- **Structured Logging** ค้นหาได้ง่าย
- **Distributed Tracing** ติดตาม request path

---

## ขั้นตอนที่ 841: Configuration Management

```clojure
;; 12-Factor: Config from environment
;; deps.edn
;; {:deps {aero/aero {:mvn/version "1.1.6"}}}

;; config/config.edn
;; {:app {:env     #or [#env APP_ENV "development"]
;;         :port    #or [#env PORT    8080]
;;         :secret  #env JWT_SECRET}
;;  :db  {:url     #env DATABASE_URL
;;         :pool    {:max-size  #or [#env DB_POOL_SIZE 10]
;;                   :min-size  2}}
;;  :redis {:host  #or [#env REDIS_HOST "localhost"]
;;           :port  #or [#env REDIS_PORT 6379]}}

(ns myapp.config
  (:require [aero.core :as aero]
            [clojure.java.io :as io]))

(defn load-config []
  (let [env (keyword (or (System/getenv "APP_ENV") "development"))
        config (aero/read-config
                 (io/resource "config/config.edn")
                 {:profile env})]
    (validate-config! config)
    config))

(defn validate-config! [config]
  (let [required [[:app :secret] [:db :url]]]
    (doseq [path required]
      (when (nil? (get-in config path))
        (throw (ex-info (str "Missing required config: " (clojure.string/join "." (map name path)))
                         {:config-path path}))))))

;; Feature flags
(def features
  {:new-checkout-flow   (= "true" (System/getenv "FEATURE_NEW_CHECKOUT"))
   :multi-currency      (= "true" (System/getenv "FEATURE_MULTI_CURRENCY"))
   :ai-recommendations  (= "true" (System/getenv "FEATURE_AI_RECS"))})

(defn feature-enabled? [feature]
  (get features feature false))
```

---

## ขั้นตอนที่ 842: Integrant Lifecycle Management

```clojure
;; deps.edn
;; {:deps {integrant/integrant {:mvn/version "0.8.1"}}}

(ns myapp.system
  (:require [integrant.core :as ig]))

;; System map = components and their dependencies
(def config
  {:myapp/config  {}
   
   :myapp/db      {:config     (ig/ref :myapp/config)
                   :migrations true}
   
   :myapp/redis   {:config (ig/ref :myapp/config)}
   
   :myapp/cache   {:redis (ig/ref :myapp/redis)}
   
   :myapp/handler {:db    (ig/ref :myapp/db)
                   :cache (ig/ref :myapp/cache)}
   
   :myapp/server  {:handler (ig/ref :myapp/handler)
                   :config  (ig/ref :myapp/config)}})

;; Initialize components
(defmethod ig/init-key :myapp/config [_ _]
  (load-config))

(defmethod ig/init-key :myapp/db [_ {:keys [config migrations]}]
  (let [ds (create-datasource (:db config))]
    (when migrations
      (run-migrations! ds))
    ds))

(defmethod ig/init-key :myapp/server [_ {:keys [handler config]}]
  (let [port (get-in config [:app :port])]
    (println (str "Starting server on port " port))
    (httpkit/run-server handler {:port port})))

;; Halt (shutdown)
(defmethod ig/halt-key! :myapp/server [_ stop-fn]
  (println "Stopping server...")
  (stop-fn :timeout 5000))

(defmethod ig/halt-key! :myapp/db [_ ds]
  (.close ds))

;; Start/stop system
(defonce system (atom nil))

(defn start! []
  (reset! system (ig/init config)))

(defn stop! []
  (when @system
    (ig/halt! @system)
    (reset! system nil)))

;; Graceful shutdown
(defn register-shutdown-hook! []
  (.addShutdownHook (Runtime/getRuntime)
    (Thread. (fn []
               (println "Shutdown signal received...")
               (stop!)
               (println "Shutdown complete.")))))
```

---

## ขั้นตอนที่ 843: Health Checks

```clojure
(ns myapp.health
  (:require [ring.util.response :as resp]))

;; Health check interface
(defprotocol HealthCheck
  (check [this])
  (name [this]))

;; DB health check
(defrecord DBHealthCheck [ds]
  HealthCheck
  (check [_]
    (try
      (jdbc/execute-one! ds ["SELECT 1"])
      {:status :up :latency (measure-latency #(jdbc/execute-one! ds ["SELECT 1"]))}
    (catch Exception e
      {:status :down :error (.getMessage e)})))
  (name [_] "database"))

;; Redis health check
(defrecord RedisHealthCheck []
  HealthCheck
  (check [_]
    (try
      (let [start (System/nanoTime)
            _ (redis (car/ping))
            latency-ms (/ (- (System/nanoTime) start) 1e6)]
        {:status :up :latency latency-ms})
    (catch Exception e
      {:status :down :error (.getMessage e)})))
  (name [_] "redis"))

;; Aggregate health
(defn check-all [checks]
  (let [results (map (fn [check]
                       [(name check) (myapp.health/check check)])
                     checks)
        all-up? (every? #(= :up (:status (second %))) results)]
    {:status    (if all-up? :up :degraded)
     :checks    (into {} results)
     :timestamp (System/currentTimeMillis)}))

;; Endpoints
(defn liveness-handler [checks]
  (fn [_]
    {:status  200
     :headers {"Content-Type" "application/json"}
     :body    (cheshire.core/generate-string {:status "alive"})}))

(defn readiness-handler [checks]
  (fn [_]
    (let [health (check-all checks)
          status (if (= :up (:status health)) 200 503)]
      {:status  status
       :headers {"Content-Type" "application/json"}
       :body    (cheshire.core/generate-string health)})))
```

---

## ขั้นตอนที่ 844: Circuit Breaker

```clojure
(ns myapp.circuit-breaker)

;; States: :closed (normal), :open (failing), :half-open (testing)

(defrecord CircuitBreaker
  [state failure-count last-failure-time
   failure-threshold timeout-ms success-threshold])

(defn create-breaker
  "Create a circuit breaker with given thresholds"
  [& {:keys [failure-threshold timeout-ms success-threshold]
       :or   {failure-threshold 5
               timeout-ms        60000
               success-threshold 2}}]
  (atom {:state             :closed
          :failure-count     0
          :success-count     0
          :last-failure-time nil
          :failure-threshold failure-threshold
          :timeout-ms        timeout-ms
          :success-threshold success-threshold}))

(defn should-allow-request? [breaker]
  (let [{:keys [state last-failure-time timeout-ms]} @breaker
        now (System/currentTimeMillis)]
    (case state
      :closed    true
      :open      (when (and last-failure-time
                             (> (- now last-failure-time) timeout-ms))
                    (swap! breaker assoc :state :half-open)
                    true)
      :half-open true)))

(defn record-success! [breaker]
  (swap! breaker
    (fn [{:keys [state success-threshold] :as b}]
      (if (= :half-open state)
        (let [new-count (inc (:success-count b))]
          (if (>= new-count success-threshold)
            (assoc b :state :closed :failure-count 0 :success-count 0)
            (assoc b :success-count new-count)))
        (assoc b :failure-count 0)))))

(defn record-failure! [breaker]
  (swap! breaker
    (fn [{:keys [failure-threshold] :as b}]
      (let [new-count (inc (:failure-count b))]
        (if (>= new-count failure-threshold)
          (assoc b :state :open
                   :failure-count new-count
                   :last-failure-time (System/currentTimeMillis))
          (assoc b :failure-count new-count))))))

(defn with-circuit-breaker
  "Execute fn with circuit breaker protection"
  [breaker f & args]
  (if (should-allow-request? breaker)
    (try
      (let [result (apply f args)]
        (record-success! breaker)
        result)
      (catch Exception e
        (record-failure! breaker)
        (throw e)))
    (throw (ex-info "Circuit breaker is open"
                     {:status 503 :retry-after (:timeout-ms @breaker)}))))
```

---

## ขั้นตอนที่ 845: Distributed Tracing

```clojure
;; OpenTelemetry ด้วย Java SDK

;; deps.edn
;; {:deps {io.opentelemetry/opentelemetry-api {:mvn/version "1.32.0"}
;;         io.opentelemetry/opentelemetry-sdk {:mvn/version "1.32.0"}
;;         io.opentelemetry.instrumentation/opentelemetry-ring {:mvn/version "1.32.0"}}}

(ns myapp.tracing
  (:import [io.opentelemetry.api GlobalOpenTelemetry]
           [io.opentelemetry.api.trace Span StatusCode]
           [io.opentelemetry.context Context]))

(def tracer
  (-> (GlobalOpenTelemetry/get)
      (.getTracer "myapp" "1.0.0")))

(defmacro with-span [name attributes & body]
  `(let [span# (-> ~tracer
                    (.spanBuilder ~name)
                    .startSpan)
          scope# (.makeCurrent span#)]
     (try
       ~@(when (seq attributes)
           (map (fn [[k v]]
                  `(.setAttribute span# ~k (str ~v)))
                (partition 2 attributes)))
       (let [result# (do ~@body)]
         (.setStatus span# StatusCode/OK)
         result#)
       (catch Exception e#
         (.setStatus span# StatusCode/ERROR (.getMessage e#))
         (.recordException span# e#)
         (throw e#))
       (finally
         (.end span#)
         (.close scope#)))))

;; Ring middleware
(defn wrap-tracing [handler]
  (fn [request]
    (with-span "http.request"
      ["http.method" (name (:request-method request))
       "http.url"    (:uri request)]
      (let [response (handler request)]
        (when-let [span (Span/current)]
          (.setAttribute span "http.status_code" (str (:status response))))
        response))))

;; ใช้งาน
(defn get-product-handler [request]
  (with-span "get.product"
    ["product.id" (get-in request [:path-params :id])]
    (let [id       (get-in request [:path-params :id])
          product  (with-span "db.find-product" []
                      (db/find-product id))
          enriched (with-span "enrich-product" []
                      (enrich-product product))]
      {:status 200 :body enriched})))
```

---

## ขั้นตอนที่ 846: Error Handling Strategy

```clojure
(ns myapp.errors)

;; Error hierarchy
(def error-codes
  {:not-found          {:http-status 404 :message "Resource not found"}
   :unauthorized       {:http-status 401 :message "Authentication required"}
   :forbidden          {:http-status 403 :message "Insufficient permissions"}
   :validation         {:http-status 422 :message "Validation failed"}
   :conflict           {:http-status 409 :message "Resource conflict"}
   :rate-limit         {:http-status 429 :message "Too many requests"}
   :service-unavailable {:http-status 503 :message "Service unavailable"}
   :internal           {:http-status 500 :message "Internal server error"}})

;; Error factory functions
(defn not-found! [resource-type id]
  (throw (ex-info (str resource-type " not found")
                   {:code :not-found
                    :resource-type resource-type
                    :id id})))

(defn validation-error! [errors]
  (throw (ex-info "Validation failed"
                   {:code :validation :errors errors})))

(defn forbidden! [action resource]
  (throw (ex-info "Forbidden"
                   {:code :forbidden :action action :resource resource})))

;; Global error handler middleware
(defn wrap-error-handler [handler]
  (fn [request]
    (try
      (handler request)
      (catch clojure.lang.ExceptionInfo e
        (let [{:keys [code] :as data} (ex-data e)
              {:keys [http-status message]} (get error-codes code (:internal error-codes))]
          (log/warn {:event   "application.error"
                      :code    code
                      :message (.getMessage e)
                      :data    data})
          {:status  http-status
           :headers {"Content-Type" "application/json"}
           :body    (cheshire.core/generate-string
                      {:error   (name code)
                       :message (.getMessage e)
                       :details data})}))
      
      (catch Exception e
        (log/error {:event   "unexpected.error"
                     :message (.getMessage e)
                     :class   (.getName (.getClass e))})
        {:status  500
         :headers {"Content-Type" "application/json"}
         :body    (cheshire.core/generate-string
                    {:error   "internal_error"
                     :message "An unexpected error occurred"})}))))
```

---

## ขั้นตอนที่ 847: Graceful Shutdown

```clojure
(ns myapp.shutdown
  (:require [clojure.core.async :as async]))

;; Track in-flight requests
(def in-flight (atom 0))
(def shutting-down? (atom false))

;; Middleware to track requests
(defn wrap-in-flight [handler]
  (fn [request]
    (if @shutting-down?
      {:status  503
       :headers {"Retry-After" "10"}
       :body    "Service shutting down, please retry"}
      (do
        (swap! in-flight inc)
        (try
          (handler request)
          (finally
            (swap! in-flight dec)))))))

;; Wait for in-flight requests to complete
(defn wait-for-requests! [timeout-ms]
  (let [deadline (+ (System/currentTimeMillis) timeout-ms)]
    (loop []
      (when (and (pos? @in-flight)
                 (< (System/currentTimeMillis) deadline))
        (Thread/sleep 100)
        (recur)))))

;; Graceful shutdown sequence
(defn graceful-shutdown! []
  (println "Starting graceful shutdown...")
  
  ;; Stop accepting new requests
  (reset! shutting-down? true)
  
  ;; Wait for in-flight requests (max 30 seconds)
  (println "Waiting for in-flight requests...")
  (wait-for-requests! 30000)
  (println (str "In-flight requests remaining: " @in-flight))
  
  ;; Stop components
  (stop-system!)
  
  (println "Shutdown complete."))

;; JVM shutdown hook
(defn register-shutdown! []
  (.addShutdownHook
    (Runtime/getRuntime)
    (Thread. graceful-shutdown!)))
```

---

## Project: Production-Ready API Server

```clojure
(ns myapp.core
  (:require [integrant.core :as ig]
            [reitit.ring :as ring]
            [myapp.config :as config]
            [myapp.health :as health]))

;; Full production server setup
(def system-config
  {:myapp/config {}
   
   :myapp/db {:config (ig/ref :myapp/config)}
   
   :myapp/redis {:config (ig/ref :myapp/config)}
   
   :myapp/circuit-breakers
   {:payment (circuit-breaker/create-breaker :failure-threshold 3)
    :email   (circuit-breaker/create-breaker :failure-threshold 5)}
   
   :myapp/health-checks
   {:db    (ig/ref :myapp/db)
    :redis (ig/ref :myapp/redis)}
   
   :myapp/handler
   {:db               (ig/ref :myapp/db)
    :redis            (ig/ref :myapp/redis)
    :circuit-breakers (ig/ref :myapp/circuit-breakers)
    :health-checks    (ig/ref :myapp/health-checks)}
   
   :myapp/server
   {:handler (ig/ref :myapp/handler)
    :config  (ig/ref :myapp/config)}})

;; Build full middleware stack
(defn build-handler [deps]
  (ring/ring-handler
    (ring/router
      [["/health"
        ["/live"  {:get (health/liveness-handler  (:health-checks deps))}]
        ["/ready" {:get (health/readiness-handler (:health-checks deps))}]]
       ["/api/v1" api-routes]])
    (ring/routes
      (ring/redirect-trailing-slash-handler)
      (ring/create-default-handler))
    {:middleware
     [parameters/parameters-middleware
      muuntaja/format-middleware
      wrap-error-handler
      wrap-in-flight
      wrap-tracing
      wrap-metrics
      (wrap-auth (:config deps))]}))

(defn -main []
  (register-shutdown!)
  (start!)
  (println "Server started!"))
```

---

*Part 29 จาก 100+ | ขั้นตอน 841-870 จาก 1000+*
