# Part 69: Observability — Metrics, Logs, Tracing
## ขั้นตอนที่ 2041-2070: Prometheus, Grafana, Structured Logging, OpenTelemetry

---

## บทนำ

Production observability:
- **Metrics** - Prometheus + Grafana
- **Structured logging** - JSON logs
- **Distributed tracing** - OpenTelemetry
- **Alerting** - thresholds and anomalies
- **Dashboards** - real-time visibility

---

## ขั้นตอนที่ 2041: Prometheus Metrics

```clojure
(ns myapp.metrics
  (:require [iapetos.core :as prometheus]
            [iapetos.collector.ring :as ring-metrics]))

;; Registry
(defonce registry
  (-> (prometheus/collector-registry)
      (prometheus/register
        (prometheus/counter :http/requests-total
          {:labels ["method" "path" "status"]
           :help   "Total HTTP requests"})
        
        (prometheus/histogram :http/request-duration-seconds
          {:labels  ["method" "path"]
           :buckets [0.01 0.05 0.1 0.25 0.5 1.0 2.5 5.0]
           :help    "HTTP request duration"})
        
        (prometheus/gauge :db/pool-active-connections
          {:help "Active DB connections"})
        
        (prometheus/counter :business/orders-created-total
          {:labels ["status"]
           :help   "Orders created"})
        
        (prometheus/histogram :business/order-value-dollars
          {:buckets [10 50 100 500 1000 5000]
           :help    "Order value distribution"}))))

;; Metric helpers
(defn record-request! [method path status duration-ms]
  (prometheus/inc registry :http/requests-total
    {:method method :path path :status (str status)})
  (prometheus/observe registry :http/request-duration-seconds
    {:method method :path path}
    (/ duration-ms 1000.0)))

(defn record-order! [status value]
  (prometheus/inc registry :business/orders-created-total {:status (name status)})
  (prometheus/observe registry :business/order-value-dollars value))

;; Middleware to auto-record HTTP metrics
(defn wrap-metrics [handler]
  (fn [request]
    (let [start    (System/currentTimeMillis)
          response (handler request)
          elapsed  (- (System/currentTimeMillis) start)
          method   (name (:request-method request))
          path     (or (:template (meta request)) (:uri request))
          status   (:status response)]
      (record-request! method path status elapsed)
      response)))

;; Expose metrics endpoint
(defn metrics-handler [_request]
  {:status  200
   :headers {"Content-Type" "text/plain; version=0.0.4"}
   :body    (prometheus/serialize registry)})
```

---

## ขั้นตอนที่ 2042: Structured Logging

```clojure
(ns myapp.logging
  (:require [clojure.tools.logging :as log]))

;; Structured log context (thread-local)
(def ^:dynamic *log-context* {})

(defmacro with-log-context [ctx & body]
  `(binding [*log-context* (merge *log-context* ~ctx)]
     ~@body))

;; Log as JSON
(defn log! [level event-type data]
  (let [entry (merge
                {:timestamp (str (java.time.Instant/now))
                 :level     (name level)
                 :event     (name event-type)}
                *log-context*
                data)]
    (case level
      :info  (log/info  (json/generate-string entry))
      :warn  (log/warn  (json/generate-string entry))
      :error (log/error (json/generate-string entry))
      :debug (log/debug (json/generate-string entry)))))

;; Convenience functions
(defn log-request! [request]
  (log! :info :http/request
    {:method  (name (:request-method request))
     :path    (:uri request)
     :ip      (get-in request [:headers "x-forwarded-for"])
     :user-id (get-in request [:user :sub])}))

(defn log-response! [request response duration-ms]
  (log! :info :http/response
    {:method   (name (:request-method request))
     :path     (:uri request)
     :status   (:status response)
     :duration duration-ms}))

(defn log-error! [event-type error context]
  (log! :error event-type
    (merge context
      {:error   (.getMessage error)
       :class   (.getName (.getClass error))
       :stack   (take 5 (map str (.getStackTrace error)))})))

;; Middleware with structured logging
(defn wrap-request-logging [handler]
  (fn [request]
    (let [request-id (str (java.util.UUID/randomUUID))
          start      (System/currentTimeMillis)]
      (with-log-context {:request-id request-id
                          :trace-id   (get-in request [:headers "x-trace-id"]
                                               request-id)}
        (log-request! request)
        (let [response (try
                         (handler request)
                         (catch Exception e
                           (log-error! :http/error e
                             {:path (:uri request)})
                           (throw e)))]
          (log-response! request response
            (- (System/currentTimeMillis) start))
          (assoc-in response [:headers "X-Request-Id"] request-id))))))
```

---

## ขั้นตอนที่ 2043: OpenTelemetry Tracing

```clojure
(ns myapp.tracing.otel
  (:import [io.opentelemetry.api OpenTelemetry]
           [io.opentelemetry.api.trace Span StatusCode]
           [io.opentelemetry.context Context]))

;; Initialize OpenTelemetry
(defn create-tracer [service-name]
  (-> (io.opentelemetry.sdk.OpenTelemetrySdk/builder)
      (.setTracerProvider
        (-> (io.opentelemetry.sdk.trace.SdkTracerProvider/builder)
            (.addSpanProcessor
              (io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter/builder)
              (.setEndpoint (or (System/getenv "OTEL_EXPORTER_OTLP_ENDPOINT")
                                 "http://localhost:4317")))
            .build))
      .buildAndRegisterGlobal
      (.getTracer service-name)))

(defonce tracer (create-tracer "myapp"))

;; Create span
(defmacro with-span [name attrs & body]
  `(let [span# (-> ~tracer
                    (.spanBuilder ~name)
                    (.startSpan))]
     (try
       (doseq [[k# v#] ~attrs]
         (.setAttribute span# (name k#) (str v#)))
       (let [result# (with-bindings {#'current-span span#}
                        ~@body)]
         (.setStatus span# StatusCode/OK)
         result#)
       (catch Exception e#
         (.setStatus span# StatusCode/ERROR (.getMessage e#))
         (.recordException span# e#)
         (throw e#))
       (finally
         (.end span#)))))

;; Trace DB queries
(defmacro with-db-span [operation table & body]
  `(with-span ~(str "db." operation)
     {:db.operation ~operation
      :db.table     ~table
      :db.system    "postgresql"}
     ~@body))

;; Trace HTTP calls
(defmacro with-http-span [method url & body]
  `(with-span ~(str "http." (clojure.string/lower-case ~method))
     {:http.method ~method
      :http.url    ~url}
     ~@body))

;; Usage
(defn get-order [db order-id]
  (with-db-span "SELECT" "orders"
    (jdbc/execute-one! db
      ["SELECT * FROM orders WHERE id = ?" order-id])))
```

---

## ขั้นตอนที่ 2044: Health Checks

```clojure
(ns myapp.health)

;; Health check registry
(defonce health-checks (atom {}))

(defn register-health-check! [name check-fn]
  (swap! health-checks assoc name check-fn))

(defn run-health-check [name check-fn]
  (try
    (let [start  (System/currentTimeMillis)
          result (check-fn)
          elapsed (- (System/currentTimeMillis) start)]
      {:status   :healthy
       :duration elapsed
       :detail   result})
    (catch Exception e
      {:status  :unhealthy
       :error   (.getMessage e)})))

(defn run-all-checks []
  (let [results (into {}
                  (map (fn [[name check-fn]]
                          [name (run-health-check name check-fn)])
                       @health-checks))
        healthy? (every? #(= :healthy (:status %)) (vals results))]
    {:healthy? healthy?
     :checks   results}))

;; Register standard checks
(defn setup-health-checks! [db redis-pool]
  (register-health-check! :database
    (fn []
      (jdbc/execute-one! db ["SELECT 1 AS ok"])
      "OK"))
  
  (register-health-check! :redis
    (fn []
      (car/wcar redis-pool (car/ping))
      "OK"))
  
  (register-health-check! :disk-space
    (fn []
      (let [root  (java.io.File. "/")
            free-pct (* 100 (/ (.getFreeSpace root) (.getTotalSpace root)))]
        (when (< free-pct 10)
          (throw (ex-info "Low disk space" {:free-pct free-pct})))
        (format "%.1f%% free" free-pct)))))

;; Health endpoints
(defn liveness-handler [_request]
  ;; Just check the app is running
  {:status 200 :body {:status "alive"}})

(defn readiness-handler [_request]
  ;; Check all dependencies
  (let [result (run-all-checks)]
    {:status (if (:healthy? result) 200 503)
     :body   result}))
```

---

## ขั้นตอนที่ 2045: Alerting Rules

```clojure
;; Prometheus alerting rules (alerting_rules.yml)
;; Generated programmatically from Clojure

(defn generate-alert-rules [service-name thresholds]
  {:groups
   [{:name  (str service-name ".alerts")
     :rules
     [{:alert  "HighErrorRate"
       :expr   (format "rate(http_requests_total{status=~\"5..\"}[5m]) /
                       rate(http_requests_total[5m]) > %.2f"
                        (:error-rate thresholds 0.05))
       :for    "5m"
       :labels {:severity "critical"}
       :annotations
       {:summary     "High error rate detected"
        :description "{{ $value | printf \"%.2f\" }}% error rate"}}
      
      {:alert  "SlowResponses"
       :expr   (format "histogram_quantile(0.95, http_request_duration_seconds) > %.1f"
                        (:p95-latency thresholds 1.0))
       :for    "5m"
       :labels {:severity "warning"}
       :annotations {:summary "P95 latency too high"}}
      
      {:alert  "DatabaseConnectionsHigh"
       :expr   (format "db_pool_active_connections > %d"
                        (:max-db-connections thresholds 18))
       :for    "2m"
       :labels {:severity "warning"}
       :annotations {:summary "DB connection pool nearly exhausted"}}]}]})
```

---

## Project: Observability Dashboard

```clojure
(ns myapp.observability)

;; Complete observability setup
(defn setup-observability! [config db redis-pool]
  ;; Metrics
  (setup-prometheus-metrics! registry)
  
  ;; Structured logging config
  (configure-logback!
    {:format  :json
     :output  :stdout
     :level   (get config :log-level "INFO")
     :context {:service (get config :service-name "myapp")
               :env     (get config :environment "production")}})
  
  ;; Distributed tracing
  (setup-opentelemetry!
    {:service-name (get config :service-name "myapp")
     :otlp-endpoint (get config :otlp-endpoint "http://jaeger:4317")})
  
  ;; Health checks
  (setup-health-checks! db redis-pool)
  
  (println "Observability configured"))

;; Wrap app with all observability middleware
(defn wrap-observability [handler]
  (-> handler
      wrap-request-logging
      wrap-metrics
      wrap-trace-context))
```

---

*Part 69 จาก 100+ | ขั้นตอน 2041-2070 จาก 1000+*
