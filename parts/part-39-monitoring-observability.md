# Part 39: Monitoring and Observability
## ขั้นตอนที่ 1141-1170: Prometheus, Grafana, Distributed Tracing, Alerting

---

## บทนำ

Observability สำหรับ Clojure services:
- **Metrics** - Prometheus + Grafana dashboards
- **Tracing** - OpenTelemetry + Jaeger/Zipkin
- **Logging** - Structured logging ด้วย Timbre
- **Alerting** - PagerDuty/OpsGenie integration
- **Health Checks** - Liveness + Readiness probes

---

## ขั้นตอนที่ 1141: Prometheus Metrics

```clojure
;; deps.edn
;; {:deps {io.prometheus/simpleclient            {:mvn/version "0.16.0"}
;;         io.prometheus/simpleclient_hotspot     {:mvn/version "0.16.0"}
;;         io.prometheus/simpleclient_httpserver  {:mvn/version "0.16.0"}}}

(ns myapp.metrics
  (:import [io.prometheus.client Counter Gauge Histogram Summary]
           [io.prometheus.client.hotspot DefaultExports]
           [io.prometheus.client.exporter HTTPServer]))

;; Register default JVM metrics (GC, memory, threads)
(DefaultExports/initialize)

;; Counter: increments only
(def http-requests-total
  (-> (Counter/build)
      (.name "http_requests_total")
      (.help "Total HTTP requests")
      (.labelNames (into-array String ["method" "path" "status"]))
      .register))

;; Gauge: can go up and down
(def active-connections
  (-> (Gauge/build)
      (.name "active_connections")
      (.help "Number of active connections")
      .register))

;; Histogram: distribution of values
(def http-request-duration
  (-> (Histogram/build)
      (.name "http_request_duration_seconds")
      (.help "HTTP request duration in seconds")
      (.labelNames (into-array String ["method" "path"]))
      (.buckets (double-array [0.005 0.01 0.025 0.05 0.1 0.25 0.5 1.0 2.5 5.0]))
      .register))

;; Summary: quantiles
(def db-query-duration
  (-> (Summary/build)
      (.name "db_query_duration_seconds")
      (.help "Database query duration")
      (.quantile 0.5 0.05)
      (.quantile 0.9 0.01)
      (.quantile 0.99 0.001)
      .register))
```

---

## ขั้นตอนที่ 1142: Recording Metrics

```clojure
(ns myapp.metrics)

;; Increment counter
(defn record-request! [method path status]
  (.inc (.labels http-requests-total
          (into-array String [method path (str status)]))))

;; Gauge operations
(defn connection-opened! []
  (.inc active-connections))

(defn connection-closed! []
  (.dec active-connections))

;; Time an operation
(defmacro with-histogram [histogram labels & body]
  `(let [timer# (.startTimer (.labels ~histogram (into-array String ~labels)))]
     (try
       (let [result# (do ~@body)]
         (.observeDuration timer#)
         result#)
       (catch Exception e#
         (.observeDuration timer#)
         (throw e#)))))

(defn time-db-query! [query-fn]
  (let [start (System/nanoTime)
        result (query-fn)
        elapsed (/ (- (System/nanoTime) start) 1e9)]
    (.observe db-query-duration elapsed)
    result))

;; Ring middleware for automatic HTTP metrics
(defn wrap-metrics [handler]
  (fn [request]
    (let [method (name (:request-method request))
          path   (:uri request)
          timer  (.startTimer (.labels http-request-duration
                                 (into-array String [method path])))]
      (try
        (let [response (handler request)
              status   (:status response)]
          (.observeDuration timer)
          (record-request! method path status)
          response)
        (catch Exception e
          (.observeDuration timer)
          (record-request! method path "500")
          (throw e))))))

;; Start metrics HTTP server (scrape endpoint)
(defn start-metrics-server! [port]
  (HTTPServer. port))
```

---

## ขั้นตอนที่ 1143: Custom Business Metrics

```clojure
(ns myapp.business-metrics
  (:import [io.prometheus.client Counter Gauge Histogram]))

;; Orders placed
(def orders-placed
  (-> (Counter/build)
      (.name "orders_placed_total")
      (.help "Total orders placed")
      (.labelNames (into-array String ["payment_method" "user_tier"]))
      .register))

;; Revenue
(def revenue-total
  (-> (Counter/build)
      (.name "revenue_total_baht")
      (.help "Total revenue in Baht")
      .register))

;; Cart abandonment
(def cart-abandoned
  (-> (Counter/build)
      (.name "cart_abandoned_total")
      (.help "Number of abandoned carts")
      .register))

;; Active users (sliding window)
(def active-users-gauge
  (-> (Gauge/build)
      (.name "active_users_current")
      (.help "Currently active users")
      .register))

;; Order value distribution
(def order-value-histogram
  (-> (Histogram/build)
      (.name "order_value_baht")
      (.help "Distribution of order values")
      (.buckets (double-array [100 500 1000 2000 5000 10000 50000]))
      .register))

;; Record business events
(defn order-placed! [order]
  (.inc (.labels orders-placed
          (into-array String [(:payment-method order)
                               (:user-tier order)])))
  (.inc revenue-total (double (:total order)))
  (.observe order-value-histogram (double (:total order))))
```

---

## ขั้นตอนที่ 1144: Structured Logging ด้วย Timbre

```clojure
(ns myapp.logging
  (:require [taoensso.timbre :as log]
            [cheshire.core :as json]))

;; JSON appender สำหรับ structured logging
(defn json-appender []
  {:enabled?   true
   :async?     false
   :min-level  nil
   :rate-limit nil
   :output-fn  (fn [{:keys [level ?ns-str ?line ?err msg_]}]
                 (json/generate-string
                   {:timestamp (str (java.time.Instant/now))
                    :level     (name level)
                    :namespace ?ns-str
                    :line      ?line
                    :message   (force msg_)
                    :error     (when ?err (.getMessage ?err))}))
   :fn         (fn [data]
                 (let [output ((:output-fn data) data)]
                   (println output)
                   (flush)))})

;; Configure Timbre
(log/merge-config!
  {:appenders {:json (json-appender)}
   :level     (keyword (System/getenv "LOG_LEVEL") "info")
   :min-level [["*" :info]]})

;; Add context to all logs in scope
(defmacro with-log-context [ctx & body]
  `(log/with-context+ ~ctx
     ~@body))

;; Usage
(with-log-context {:request-id "abc123" :user-id "user-456"}
  (log/info "Processing request" {:action "place-order" :items 3})
  (log/warn "High latency detected" {:db-ms 450}))

;; Correlation ID middleware
(defn wrap-request-id [handler]
  (fn [request]
    (let [req-id (or (get-in request [:headers "x-request-id"])
                     (str (java.util.UUID/randomUUID)))]
      (with-log-context {:request-id req-id}
        (let [response (handler request)]
          (assoc-in response [:headers "x-request-id"] req-id))))))
```

---

## ขั้นตอนที่ 1145: OpenTelemetry Distributed Tracing

```clojure
;; deps.edn
;; {:deps {io.opentelemetry.instrumentation/opentelemetry-instrumentation-clojure
;;          {:mvn/version "1.32.0"}
;;         io.opentelemetry/opentelemetry-exporter-otlp
;;          {:mvn/version "1.32.0"}}}

(ns myapp.tracing
  (:import [io.opentelemetry.api GlobalOpenTelemetry]
           [io.opentelemetry.api.trace Span SpanKind]
           [io.opentelemetry.api.trace.propagation W3CTraceContextPropagator]
           [io.opentelemetry.context Context]))

(defn tracer []
  (.getTracer (GlobalOpenTelemetry/get) "myapp"))

(defmacro with-span [name {:keys [kind attributes]} & body]
  `(let [span# (-> (tracer)
                    .spanBuilder
                    (. ~(symbol (str "." "startSpan"))
                       ~name))]
     (let [scope# (.makeCurrent span#)]
       (try
         ~@body
         (catch Exception e#
           (.recordException span# e#)
           (.setStatus span# io.opentelemetry.api.trace.StatusCode/ERROR
                        (.getMessage e#))
           (throw e#))
         (finally
           (.end span#)
           (.close scope#))))))

;; Simpler version using spans directly
(defn start-span!
  ([name] (start-span! name :internal))
  ([name kind]
   (-> (.spanBuilder (tracer) name)
       (.setSpanKind (case kind
                        :server   SpanKind/SERVER
                        :client   SpanKind/CLIENT
                        :producer SpanKind/PRODUCER
                        :consumer SpanKind/CONSUMER
                        SpanKind/INTERNAL))
       (.startSpan))))

(defn end-span! [span]
  (.end span))

;; Add attributes to current span
(defn add-attribute! [key value]
  (when-let [span (Span/current)]
    (.setAttribute span key (str value))))

;; Trace HTTP requests
(defn wrap-tracing [handler]
  (fn [request]
    (let [span (start-span! (str (:request-method request) " " (:uri request))
                             :server)]
      (add-attribute! "http.method" (name (:request-method request)))
      (add-attribute! "http.url" (:uri request))
      (try
        (let [response (handler request)]
          (add-attribute! "http.status_code" (:status response))
          (.end span)
          response)
        (catch Exception e
          (.recordException span e)
          (.end span)
          (throw e))))))
```

---

## ขั้นตอนที่ 1146: Health Checks

```clojure
(ns myapp.health
  (:require [next.jdbc :as jdbc]
            [taoensso.carmine :as car]))

;; Health check protocol
(defprotocol HealthCheck
  (check-health [this]))

;; Database health
(defrecord DatabaseHealth [ds]
  HealthCheck
  (check-health [_]
    (try
      (jdbc/execute-one! ds ["SELECT 1"])
      {:status :healthy :component "database"}
      (catch Exception e
        {:status :unhealthy :component "database"
         :error  (.getMessage e)}))))

;; Redis health
(defrecord RedisHealth [conn]
  HealthCheck
  (check-health [_]
    (try
      (let [pong (car/wcar conn (car/ping))]
        (if (= "PONG" pong)
          {:status :healthy :component "redis"}
          {:status :degraded :component "redis" :response pong}))
      (catch Exception e
        {:status :unhealthy :component "redis" :error (.getMessage e)}))))

;; External API health
(defrecord ExternalApiHealth [url timeout-ms]
  HealthCheck
  (check-health [_]
    (try
      (let [resp @(org.httpkit.client/get url {:timeout timeout-ms})]
        (if (< (:status resp) 500)
          {:status :healthy :component url}
          {:status :degraded :component url :http-status (:status resp)}))
      (catch Exception e
        {:status :unhealthy :component url :error (.getMessage e)}))))

;; Aggregate health handler
(defn health-handler [checks]
  (fn [_request]
    (let [results  (mapv check-health checks)
          all-ok?  (every? #(= :healthy (:status %)) results)
          any-down? (some #(= :unhealthy (:status %)) results)]
      {:status (cond all-ok?   200
                      any-down? 503
                      :else     207)
       :body   {:status     (cond all-ok?   "healthy"
                                   any-down? "unhealthy"
                                   :else     "degraded")
                 :checks     results
                 :timestamp  (str (java.time.Instant/now))}})))

;; Kubernetes readiness vs liveness
(defn liveness-handler [_req]
  {:status 200 :body {:status "alive"}})

(defn readiness-handler [checks]
  (fn [req]
    ((health-handler checks) req)))
```

---

## ขั้นตอนที่ 1147: Grafana Dashboard Configuration

```json
{
  "dashboard": {
    "title": "Clojure App Dashboard",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [{
          "expr": "rate(http_requests_total[5m])",
          "legendFormat": "{{method}} {{path}} {{status}}"
        }]
      },
      {
        "title": "Request Duration P99",
        "type": "stat",
        "targets": [{
          "expr": "histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))",
          "legendFormat": "p99 latency"
        }]
      },
      {
        "title": "Error Rate",
        "type": "stat",
        "targets": [{
          "expr": "rate(http_requests_total{status=~'5..'}[5m]) / rate(http_requests_total[5m])",
          "legendFormat": "Error %"
        }]
      },
      {
        "title": "Active Connections",
        "type": "gauge",
        "targets": [{
          "expr": "active_connections",
          "legendFormat": "Connections"
        }]
      },
      {
        "title": "JVM Heap Usage",
        "type": "graph",
        "targets": [{
          "expr": "jvm_memory_bytes_used{area='heap'}",
          "legendFormat": "Heap Used"
        }, {
          "expr": "jvm_memory_bytes_max{area='heap'}",
          "legendFormat": "Heap Max"
        }]
      },
      {
        "title": "GC Pause Time",
        "type": "graph",
        "targets": [{
          "expr": "rate(jvm_gc_collection_seconds_sum[5m])",
          "legendFormat": "{{gc}}"
        }]
      }
    ]
  }
}
```

---

## ขั้นตอนที่ 1148: Alerting Rules (Prometheus)

```yaml
# alerting-rules.yml
groups:
  - name: clojure-app
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          rate(http_requests_total{status=~"5.."}[5m]) 
          / rate(http_requests_total[5m]) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate: {{ $value | humanizePercentage }}"
          description: "Error rate above 5% for 2 minutes"
      
      # High latency
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99, 
            rate(http_request_duration_seconds_bucket[5m])) > 2.0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P99 latency {{ $value }}s"
      
      # Service down
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.instance }} is down"
      
      # High JVM heap
      - alert: HighJvmHeap
        expr: |
          jvm_memory_bytes_used{area="heap"} 
          / jvm_memory_bytes_max{area="heap"} > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "JVM heap {{ $value | humanizePercentage }} full"
      
      # Database connection pool exhausted
      - alert: DBPoolExhausted
        expr: hikaricp_connections_pending > 5
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "DB connection pool pending {{ $value }} requests"
```

---

## ขั้นตอนที่ 1149: Log Aggregation (ELK Stack)

```clojure
;; Logstash/ELK integration via structured JSON logs

(ns myapp.logging.elk
  (:require [taoensso.timbre :as log]
            [cheshire.core :as json]))

;; Enhanced JSON appender with ECS (Elastic Common Schema) format
(defn ecs-json-appender []
  {:enabled? true
   :fn       (fn [{:keys [level ?ns-str ?file ?line ?err msg_ context]}]
               (let [output
                     (json/generate-string
                       (cond->
                         {"@timestamp" (str (java.time.Instant/now))
                          "log.level"  (name level)
                          "log.logger" ?ns-str
                          "log.origin" {:file {:name ?file :line ?line}}
                          "message"    (force msg_)
                          "service.name" (System/getenv "SERVICE_NAME")
                          "service.version" (System/getenv "APP_VERSION")
                          "host.name"  (.. java.net.InetAddress getLocalHost getHostName)}
                         
                         ;; Add error if present
                         ?err
                         (assoc "error.message"    (.getMessage ?err)
                                 "error.stack_trace" (with-out-str
                                                       (.printStackTrace ?err)))
                         
                         ;; Add context (request-id, user-id, etc.)
                         context
                         (merge (reduce-kv (fn [m k v]
                                              (assoc m (name k) (str v)))
                                            {} context))))]
                 (println output)
                 (flush)))})

;; Usage with MDC (Mapped Diagnostic Context)
(defn handle-request [request]
  (log/with-context+ {:request-id (java.util.UUID/randomUUID)
                       :user-id    (get-in request [:auth :user-id])
                       :path       (:uri request)
                       :method     (name (:request-method request))}
    (log/info "Request received")
    (let [result (process request)]
      (log/info "Request completed" {:duration-ms (:ms result)})
      result)))
```

---

## ขั้นตอนที่ 1150: Jaeger Distributed Tracing Setup

```yaml
# docker-compose addition for tracing
services:
  jaeger:
    image: jaegertracing/all-in-one:1.52
    ports:
      - "6831:6831/udp"  # UDP compact thrift
      - "6832:6832/udp"  # UDP binary thrift
      - "5778:5778"      # Jaeger configs
      - "16686:16686"    # UI
      - "14268:14268"    # HTTP collector
      - "4317:4317"      # OTLP gRPC
      - "4318:4318"      # OTLP HTTP
    environment:
      - COLLECTOR_OTLP_ENABLED=true
```

```clojure
;; Configure OTEL to send to Jaeger
;; -Dotel.exporter.otlp.endpoint=http://jaeger:4317
;; -Dotel.service.name=myapp
;; -Dotel.traces.exporter=otlp

;; Or programmatic setup
(ns myapp.tracing-config
  (:import [io.opentelemetry.sdk OpenTelemetrySdk]
           [io.opentelemetry.sdk.trace SdkTracerProvider]
           [io.opentelemetry.exporter.otlp.trace OtlpGrpcSpanExporter]
           [io.opentelemetry.sdk.trace.export BatchSpanProcessor]))

(defn configure-tracing! [jaeger-endpoint service-name]
  (let [exporter  (-> (OtlpGrpcSpanExporter/builder)
                       (.setEndpoint jaeger-endpoint)
                       .build)
        processor (BatchSpanProcessor/create exporter)
        provider  (-> (SdkTracerProvider/builder)
                       (.addSpanProcessor processor)
                       .build)
        sdk       (-> (OpenTelemetrySdk/builder)
                       (.setTracerProvider provider)
                       .buildAndRegisterGlobal)]
    sdk))
```

---

## Project: Observability Stack

```clojure
;; Complete observability setup for production

(ns myapp.observability
  (:require [myapp.metrics :as metrics]
            [myapp.health :as health]
            [myapp.logging :as logging]))

(defn setup-observability! [system]
  ;; 1. Configure structured logging
  (logging/configure-json-logging!
    {:level   (keyword (or (System/getenv "LOG_LEVEL") "info"))
     :service (System/getenv "SERVICE_NAME")})
  
  ;; 2. Start metrics scrape endpoint on port 9090
  (metrics/start-metrics-server! 9090)
  
  ;; 3. Configure tracing
  (when-let [jaeger (System/getenv "JAEGER_ENDPOINT")]
    (tracing/configure-tracing! jaeger (System/getenv "SERVICE_NAME")))
  
  ;; 4. Register all middleware
  (-> (:handler system)
      (metrics/wrap-metrics)
      (tracing/wrap-tracing)
      (logging/wrap-request-id)))

;; Run Prometheus metrics server
(defonce metrics-server (atom nil))

(defn start-metrics-server! [port]
  (reset! metrics-server
    (org.httpkit.server/run-server
      (fn [_] {:status 200
                :headers {"Content-Type" "text/plain; version=0.0.4"}
                :body (io.prometheus.client.exporter.common.TextFormat/write004)})
      {:port port})))
```

---

*Part 39 จาก 100+ | ขั้นตอน 1141-1170 จาก 1000+*
