# Part 16: Microservices Architecture
## ขั้นตอนที่ 451-480: Service Design, gRPC, Kafka, Docker

---

## บทนำ

Clojure เหมาะกับ Microservices เพราะ:
- Immutable data → ง่ายต่อการ serialize/deserialize
- Pure functions → ง่ายต่อการ test แต่ละ service
- core.async → async communication
- EDN/Transit → efficient data serialization
- JVM → ใช้ Java libraries ได้ทั้งหมด

---

## ขั้นตอนที่ 451: Service Architecture

```
Microservices Pattern:

┌─────────┐    HTTP/gRPC    ┌─────────────┐
│ Gateway  │───────────────▶│ User Service │
│  (Ring)  │                └─────────────┘
│          │    Events       ┌─────────────┐
│          │◀──────────────▶│ Order Service│
│          │    (Kafka)     └─────────────┘
└─────────┘                 ┌─────────────┐
                            │ Email Service│
                            └─────────────┘

Each service:
- Own database
- Own deployment
- Communicates via APIs/Events
- Independent scaling
```

---

## ขั้นตอนที่ 452: Service Template

```clojure
;; src/user_service/system.clj
(ns user-service.system
  (:require [integrant.core :as ig]
            [ring.adapter.jetty :as jetty]
            [next.jdbc :as jdbc]
            [user-service.api :as api]
            [user-service.db :as db]))

;; System configuration (integrant pattern)
(def config
  {:adapter/jetty {:port 8080
                    :join? false
                    :handler (ig/ref :handler/app)}
   
   :db/pool {:host (or (System/getenv "DB_HOST") "localhost")
              :port 5432
              :dbname "users_db"
              :user "postgres"
              :password (System/getenv "DB_PASSWORD")}
   
   :handler/app {:db (ig/ref :db/pool)}
   
   :service/health-check {}})

;; Integrant lifecycle
(defmethod ig/init-key :adapter/jetty [_ {:keys [port handler join?]}]
  (jetty/run-jetty handler {:port port :join? join?}))

(defmethod ig/halt-key! :adapter/jetty [_ server]
  (.stop server))

(defmethod ig/init-key :db/pool [_ opts]
  (db/create-pool opts))

(defmethod ig/halt-key! :db/pool [_ pool]
  (.close pool))

(defmethod ig/init-key :handler/app [_ {:keys [db]}]
  (api/create-app db))

;; Start/stop
(defn start []
  (ig/init config))

(defn stop [system]
  (ig/halt! system))

;; REPL
(defonce system (atom nil))

(defn restart []
  (when @system (stop @system))
  (reset! system (start)))
```

---

## ขั้นตอนที่ 453: Service Discovery และ Health Checks

```clojure
(ns user-service.health
  (:require [ring.util.response :as resp]))

;; Health check endpoint
(defn health-check-handler [{:keys [db kafka]}]
  (fn [_request]
    (let [db-ok?    (try (jdbc/execute-one! db ["SELECT 1"]) true
                         (catch Exception _ false))
          kafka-ok? (try (kafka/connected? kafka) true
                         (catch Exception _ false))
          healthy?  (and db-ok? kafka-ok?)]
      {:status (if healthy? 200 503)
       :body {:status (if healthy? "healthy" "unhealthy")
               :checks {:database db-ok?
                         :kafka kafka-ok?}
               :timestamp (System/currentTimeMillis)}})))

;; Kubernetes probes
;; /health/live  - is the process alive?
;; /health/ready - is it ready to receive traffic?
(def routes
  [["/health"
    ["/live"  {:get (fn [_] {:status 200 :body {:status "alive"}})}]
    ["/ready" {:get (health-check-handler deps)}]]])
```

---

## ขั้นตอนที่ 454: Kafka Integration

```clojure
;; deps.edn
;; {:deps {org.apache.kafka/kafka-clients {:mvn/version "3.6.0"}}}

(ns user-service.events
  (:require [clojure.data.json :as json])
  (:import [org.apache.kafka.clients.producer
            KafkaProducer ProducerRecord]
           [org.apache.kafka.clients.consumer
            KafkaConsumer]))

;; ===== Producer =====
(defn create-producer [bootstrap-servers]
  (KafkaProducer.
    {"bootstrap.servers" bootstrap-servers
     "key.serializer"   "org.apache.kafka.common.serialization.StringSerializer"
     "value.serializer" "org.apache.kafka.common.serialization.StringSerializer"
     "acks"             "all"
     "retries"          "3"}))

(defn publish-event! [producer topic event]
  (let [record (ProducerRecord.
                  topic
                  (:id event)        ; key
                  (json/write-str event))] ; value
    (.send producer record)
    (println "Published:" (:type event) "to" topic)))

;; ===== Consumer =====
(defn create-consumer [bootstrap-servers group-id]
  (KafkaConsumer.
    {"bootstrap.servers" bootstrap-servers
     "group.id"          group-id
     "key.deserializer"  "org.apache.kafka.common.serialization.StringDeserializer"
     "value.deserializer" "org.apache.kafka.common.serialization.StringDeserializer"
     "auto.offset.reset"  "earliest"
     "enable.auto.commit" "true"}))

(defn start-consumer! [consumer topics handler-fn]
  (.subscribe consumer topics)
  (future
    (try
      (loop []
        (let [records (.poll consumer 100)]
          (doseq [record records]
            (handler-fn {:key   (.key record)
                          :value (json/read-str (.value record) :key-fn keyword)
                          :topic (.topic record)
                          :partition (.partition record)
                          :offset    (.offset record)}))
          (recur)))
      (catch Exception e
        (println "Consumer error:" (.getMessage e))))))
```

---

## ขั้นตอนที่ 455: Event-driven Microservices

```clojure
;; User Service publishes events
(defn create-user! [producer db user-data]
  (let [user (db/insert-user! db user-data)]
    ;; Publish domain event
    (publish-event! producer "user.events"
      {:type      :user/created
       :id        (str (:id user))
       :timestamp (System/currentTimeMillis)
       :data      {:user-id (:id user)
                   :email   (:email user)
                   :name    (:name user)}})
    user))

;; Email Service consumes events
(defn handle-user-event [event]
  (case (keyword (:type (:value event)))
    :user/created
    (email/send-welcome!
      {:to   (get-in event [:value :data :email])
       :name (get-in event [:value :data :name])})
    
    :user/password-reset
    (email/send-reset-link!
      {:to    (get-in event [:value :data :email])
       :token (get-in event [:value :data :token])})
    
    ;; Unknown event - log and skip
    (println "Unknown event type:" (:type (:value event)))))

;; Start consuming
(let [consumer (create-consumer "localhost:9092" "email-service")]
  (start-consumer! consumer ["user.events"] handle-user-event))
```

---

## ขั้นตอนที่ 456: API Gateway Pattern

```clojure
(ns gateway.core
  (:require [reitit.ring :as ring]
            [ring.adapter.jetty :as jetty]
            [clj-http.client :as http]))

;; Service registry
(def services
  {:user-service  "http://user-service:8080"
   :order-service "http://order-service:8080"
   :auth-service  "http://auth-service:8080"})

;; Proxy request to downstream service
(defn proxy-request [service path request]
  (let [base-url (get services service)
        url      (str base-url path)
        response (http/request
                   {:method  (:request-method request)
                    :url     url
                    :headers (select-keys (:headers request)
                                          ["authorization" "content-type"])
                    :body    (when (:body request)
                               (slurp (:body request)))
                    :throw-exceptions false})]
    {:status  (:status response)
     :headers (:headers response)
     :body    (:body response)}))

;; Gateway routes
(def routes
  [["/api/v1"
    ["/auth/*path"  {:handler #(proxy-request :auth-service  (str "/" (:path (:path-params %))) %)}]
    ["/users/*path" {:handler #(proxy-request :user-service  (str "/" (:path (:path-params %))) %)}]
    ["/orders/*path" {:handler #(proxy-request :order-service (str "/" (:path (:path-params %))) %)}]]])

;; Rate limiting per service
;; Authentication at gateway level
;; Request logging
;; Circuit breaking
```

---

## ขั้นตอนที่ 457: Circuit Breaker สำหรับ Microservices

```clojure
(ns gateway.circuit-breaker)

;; States: :closed → :open → :half-open → :closed
(def circuit-state
  (atom {:state       :closed
          :failures    0
          :last-failure nil
          :timeout-ms  30000}))

(defn circuit-open? [state]
  (and (= :open (:state state))
       (< (- (System/currentTimeMillis) (:last-failure state))
          (:timeout-ms state))))

(defn record-failure! []
  (swap! circuit-state
         (fn [state]
           (let [new-failures (inc (:failures state))]
             (if (>= new-failures 5)  ; threshold
               (assoc state :state :open
                             :failures new-failures
                             :last-failure (System/currentTimeMillis))
               (assoc state :failures new-failures))))))

(defn record-success! []
  (reset! circuit-state
    {:state :closed :failures 0 :last-failure nil :timeout-ms 30000}))

(defn call-with-circuit-breaker [service-fn]
  (let [state @circuit-state]
    (if (circuit-open? state)
      ;; Circuit is open - fail fast
      (throw (ex-info "Circuit breaker open"
                       {:service "downstream"
                        :retry-after (+ (:last-failure state) (:timeout-ms state))}))
      ;; Try the call
      (try
        (let [result (service-fn)]
          (record-success!)
          result)
        (catch Exception e
          (record-failure!)
          (throw e))))))
```

---

## ขั้นตอนที่ 458: Docker สำหรับ Clojure

```dockerfile
# Dockerfile สำหรับ Clojure service
FROM clojure:temurin-21-tools-deps AS builder

WORKDIR /app
COPY deps.edn .
RUN clojure -P   # Download dependencies

COPY . .
RUN clojure -T:build uber  # Build uberjar

# Production image (smaller)
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app
COPY --from=builder /app/target/app.jar .

# Non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 8080

ENTRYPOINT ["java", 
            "-XX:+UseG1GC",
            "-XX:MaxRAMPercentage=75",
            "-jar", "app.jar"]
```

```yaml
# docker-compose.yml
version: '3.9'

services:
  user-service:
    build: ./user-service
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=postgres
      - DB_PASSWORD=secret
      - KAFKA_BROKERS=kafka:9092
    depends_on:
      - postgres
      - kafka
    healthcheck:
      test: ["CMD", "wget", "-q", "-O-", "http://localhost:8080/health/live"]
      interval: 30s
      timeout: 10s
      retries: 3

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: users_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres-data:/var/lib/postgresql/data

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
    depends_on:
      - zookeeper

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

volumes:
  postgres-data:
```

---

## ขั้นตอนที่ 459: Service Mesh และ Observability

```clojure
;; Distributed tracing ด้วย OpenTelemetry
(ns gateway.tracing
  (:require [ring.util.request :as req]))

(defn trace-middleware [handler]
  (fn [request]
    (let [trace-id (or (get-in request [:headers "x-trace-id"])
                       (str (java.util.UUID/randomUUID)))
          span-id  (str (java.util.UUID/randomUUID))
          request  (assoc request :trace-id trace-id :span-id span-id)]
      
      ;; Log structured trace data
      (println (format "{\"trace_id\":\"%s\",\"span_id\":\"%s\",\"method\":\"%s\",\"path\":\"%s\"}"
                       trace-id span-id
                       (name (:request-method request))
                       (:uri request)))
      
      (let [response (handler request)]
        ;; Propagate trace headers downstream
        (update response :headers merge
                {"x-trace-id" trace-id
                 "x-span-id"  span-id})))))

;; Metrics ด้วย Prometheus
(defn record-request-metric! [method path status duration-ms]
  ;; Push to metrics store / Prometheus
  (println (format "http_requests_total{method=\"%s\",path=\"%s\",status=\"%d\"} 1"
                   method path status))
  (println (format "http_request_duration_ms{method=\"%s\",path=\"%s\"} %d"
                   method path duration-ms)))

(defn metrics-middleware [handler]
  (fn [request]
    (let [start    (System/currentTimeMillis)
          response (handler request)
          duration (- (System/currentTimeMillis) start)]
      (record-request-metric!
        (name (:request-method request))
        (:uri request)
        (:status response)
        duration)
      response)))
```

---

## ขั้นตอนที่ 460: Configuration Management

```clojure
;; 12-factor app: config ใน environment variables
(ns myapp.config)

(defn load-config []
  {:server {:port  (Integer/parseInt (or (System/getenv "PORT") "8080"))
             :host  (or (System/getenv "HOST") "0.0.0.0")}
   
   :database {:url      (or (System/getenv "DATABASE_URL")
                             "jdbc:postgresql://localhost/myapp_dev")
               :pool-size (Integer/parseInt (or (System/getenv "DB_POOL_SIZE") "10"))}
   
   :kafka {:brokers (or (System/getenv "KAFKA_BROKERS") "localhost:9092")
            :group-id (or (System/getenv "KAFKA_GROUP_ID") "myapp")}
   
   :auth {:secret     (System/getenv "JWT_SECRET")   ; required!
           :expiry-sec (Integer/parseInt (or (System/getenv "JWT_EXPIRY") "86400"))}
   
   :redis {:url (or (System/getenv "REDIS_URL") "redis://localhost:6379")}
   
   :env (keyword (or (System/getenv "APP_ENV") "development"))})

;; Validate required config
(defn validate-config! [config]
  (let [required {:auth [:secret]}
        missing  (for [[ns keys] required
                       k keys
                       :when (nil? (get-in config [ns k]))]
                   [ns k])]
    (when (seq missing)
      (throw (ex-info "Missing required environment variables"
                       {:missing missing}))))
  config)

;; Load once at startup
(def config (delay (-> (load-config) (validate-config!))))
```

---

*Part 16 จาก 100+ | ขั้นตอน 451-480 จาก 1000+*
