# Part 23: DevOps และ CI/CD
## ขั้นตอนที่ 661-690: Docker, Kubernetes, GitHub Actions, Monitoring

---

## บทนำ

Modern Clojure deployment:
- **Docker** - containerize application
- **Kubernetes** - orchestration
- **GitHub Actions** - CI/CD pipeline
- **Prometheus/Grafana** - monitoring
- **ELK Stack** - log aggregation

---

## ขั้นตอนที่ 661: Dockerfile ขั้นสูง

```dockerfile
# Multi-stage build for production
FROM clojure:temurin-21-tools-deps AS builder

WORKDIR /app

# Cache dependencies first (faster rebuilds)
COPY deps.edn build.clj .
RUN clojure -P

# Copy source
COPY src/ src/
COPY resources/ resources/

# Build uberjar
RUN clojure -T:build uber

# ===== Production Stage =====
FROM eclipse-temurin:21-jre-alpine

# Security: non-root user
RUN addgroup -S app && adduser -S app -G app
USER app

WORKDIR /app

# Copy only the jar
COPY --from=builder --chown=app:app /app/target/app.jar .

# JVM settings
ENV JAVA_OPTS="-XX:+UseG1GC \
              -XX:MaxRAMPercentage=75 \
              -XX:+HeapDumpOnOutOfMemoryError \
              -XX:HeapDumpPath=/tmp/heap-dump.hprof \
              -Dfile.encoding=UTF-8"

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --start-period=60s --retries=3 \
  CMD wget -qO- http://localhost:8080/health/live || exit 1

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

---

## ขั้นตอนที่ 662: build.clj สำหรับ CI

```clojure
;; build.clj
(ns build
  (:require [clojure.tools.build.api :as b]))

(def lib 'com.example/myapp)
(def version (or (System/getenv "APP_VERSION")
                 (str "0.1." (b/git-count-revs nil))))
(def class-dir "target/classes")
(def basis (b/create-basis {:project "deps.edn"}))
(def uber-file (format "target/%s-%s-standalone.jar"
                        (name lib) version))

(defn clean [_]
  (b/delete {:path "target"}))

(defn compile-clj [_]
  (b/compile-clj {:basis basis
                   :src-dirs ["src"]
                   :class-dir class-dir}))

(defn uber [_]
  (clean nil)
  (b/copy-dir {:src-dirs ["src" "resources"]
                :target-dir class-dir})
  (compile-clj nil)
  (b/uber {:class-dir class-dir
            :uber-file uber-file
            :basis     basis
            :main      'myapp.core}))

(defn test [_]
  (let [basis (b/create-basis {:project "deps.edn" :aliases [:test]})]
    (b/process {:command-args (into ["java" "-cp" (b/classpath-string {:basis basis})
                                     "clojure.main" "-m" "kaocha.runner"]
                                    ["unit"])})))
```

---

## ขั้นตอนที่ 663: GitHub Actions CI/CD

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_DB: myapp_test
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: test-password
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '21'
      
      - name: Cache Clojure dependencies
        uses: actions/cache@v3
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-clojure-${{ hashFiles('deps.edn') }}
          restore-keys: ${{ runner.os }}-clojure-
      
      - name: Install Clojure tools
        uses: DeLaGuardo/setup-clojure@12
        with:
          cli: latest
      
      - name: Run tests
        env:
          DATABASE_URL: jdbc:postgresql://localhost/myapp_test
          DATABASE_USER: postgres
          DATABASE_PASSWORD: test-password
        run: clojure -T:build test
      
      - name: Upload test results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: test-results
          path: target/test-results/

  build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix={{branch}}-
            type=raw,value=latest,enable={{is_default_branch}}
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache
          cache-to: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache,mode=max

  deploy:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: build
    environment: production
    
    steps:
      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/myapp \
            myapp=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest \
            --namespace production
```

---

## ขั้นตอนที่ 664: Kubernetes Deployment

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: myapp
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      containers:
        - name: myapp
          image: ghcr.io/myorg/myapp:latest
          ports:
            - containerPort: 8080
          
          env:
            - name: APP_ENV
              value: "production"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: database-url
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: jwt-secret
          
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "2Gi"
              cpu: "1000m"
          
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 30
          
          volumeMounts:
            - name: app-config
              mountPath: /app/config
              readOnly: true
      
      volumes:
        - name: app-config
          configMap:
            name: app-config

---
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: production
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

## ขั้นตอนที่ 665: Prometheus Metrics

```clojure
;; deps.edn
;; {:deps {io.prometheus.client/simpleclient {:mvn/version "0.16.0"}
;;         io.prometheus.client/simpleclient_hotspot {:mvn/version "0.16.0"}}}

(ns myapp.metrics
  (:import [io.prometheus.client Counter Histogram Gauge CollectorRegistry]
           [io.prometheus.client.hotspot DefaultExports]))

;; Register default JVM metrics
(DefaultExports/initialize)

;; Custom metrics
(def request-counter
  (-> (Counter/build)
      (.name "http_requests_total")
      (.labelNames (into-array String ["method" "path" "status"]))
      (.help "Total HTTP requests")
      (.register)))

(def request-duration
  (-> (Histogram/build)
      (.name "http_request_duration_seconds")
      (.labelNames (into-array String ["method" "path"]))
      (.buckets (double-array [0.001 0.005 0.01 0.05 0.1 0.5 1.0 5.0]))
      (.help "HTTP request duration in seconds")
      (.register)))

(def active-users
  (-> (Gauge/build)
      (.name "active_users")
      (.help "Currently active users")
      (.register)))

;; Middleware to record metrics
(defn wrap-metrics [handler]
  (fn [request]
    (let [start    (System/nanoTime)
          response (handler request)
          elapsed  (/ (- (System/nanoTime) start) 1e9)
          method   (name (:request-method request))
          path     (:uri request)
          status   (str (:status response))]
      
      (-> request-counter (.labels (into-array String [method path status])) .inc)
      (-> request-duration (.labels (into-array String [method path])) (.observe elapsed))
      
      response)))

;; Expose metrics endpoint
(defn metrics-handler [_]
  {:status  200
   :headers {"Content-Type" "text/plain; version=0.0.4"}
   :body    (let [writer (java.io.StringWriter.)]
              (io.prometheus.client.exporter.common.TextFormat/write004
                writer CollectorRegistry/defaultRegistry)
              (.toString writer))})
```

---

## ขั้นตอนที่ 666: Structured Logging

```clojure
(ns myapp.logging
  (:require [taoensso.timbre :as log]))

;; Configure structured JSON logging
(def json-appender
  {:enabled?   true
   :async?     false
   :fn         (fn [{:keys [timestamp_ level ?ns-str msg_]}]
                 (println
                   (cheshire.core/generate-string
                     {:timestamp  @timestamp_
                      :level      (name level)
                      :namespace  ?ns-str
                      :message    @msg_})))})

(log/merge-config!
  {:appenders {:json json-appender}
   :output-fn :inherit
   :min-level :info})

;; Log with structured context
(defn log-request! [request response duration-ms]
  (log/info
    {:event     "http.request"
     :method    (name (:request-method request))
     :path      (:uri request)
     :status    (:status response)
     :duration  duration-ms
     :request-id (:request-id request)
     :user-id   (get-in request [:auth :sub])}))

(defn log-error! [e context]
  (log/error
    {:event   "application.error"
     :error   (.getMessage e)
     :type    (.getName (.getClass e))
     :context context}))

;; Use in application
(defn handler [request]
  (let [start (System/currentTimeMillis)]
    (try
      (let [response (process-request request)]
        (log-request! request response (- (System/currentTimeMillis) start))
        response)
      (catch Exception e
        (log-error! e {:path (:uri request)})
        {:status 500 :body {:error "Internal error"}}))))
```

---

*Part 23 จาก 100+ | ขั้นตอน 661-690 จาก 1000+*
