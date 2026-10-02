# Part 43: Deployment and CI/CD
## ขั้นตอนที่ 1261-1290: Docker, Kubernetes, GitHub Actions, Helm

---

## บทนำ

Deployment pipeline สำหรับ Clojure applications:
- **Docker** - containerization สำหรับ uberjar
- **Kubernetes** - orchestration, scaling, rollouts
- **GitHub Actions** - CI/CD automation
- **Helm** - Kubernetes package management
- **Zero-downtime deployment** - blue/green, canary

---

## ขั้นตอนที่ 1261: Dockerfile สำหรับ Clojure

```dockerfile
# Multi-stage build
# Stage 1: Build
FROM clojure:temurin-21-tools-deps AS builder

WORKDIR /build

# Copy dependencies first (cache layer)
COPY deps.edn ./
RUN clojure -P  # Pre-download dependencies

# Copy source
COPY src/ ./src/
COPY resources/ ./resources/

# Build uberjar
RUN clojure -T:build uber

# Stage 2: Runtime (minimal image)
FROM eclipse-temurin:21-jre-alpine

# Security: run as non-root
RUN addgroup -S app && adduser -S app -G app
USER app

WORKDIR /app

# Copy only the jar
COPY --from=builder /build/target/app-standalone.jar ./app.jar

# JVM flags for containers
ENV JVM_OPTS="-Xms256m -Xmx512m \
              -XX:+UseZGC \
              -XX:MaxRAMPercentage=75.0 \
              -XX:+ExitOnOutOfMemoryError \
              -Djava.security.egd=file:/dev/./urandom"

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
  CMD wget -qO- http://localhost:8080/health || exit 1

EXPOSE 8080

CMD ["sh", "-c", "exec java $JVM_OPTS -jar app.jar"]
```

---

## ขั้นตอนที่ 1262: build.clj สำหรับ Uberjar

```clojure
(ns build
  (:require [clojure.tools.build.api :as b]))

(def lib      'myapp/myapp)
(def version  (or (System/getenv "APP_VERSION") "0.1.0-SNAPSHOT"))
(def class-dir "target/classes")
(def basis     (b/create-basis {:project "deps.edn"}))
(def jar-file  (format "target/%s-%s-standalone.jar"
                         (name lib) version))

(defn clean [_]
  (b/delete {:path "target"}))

(defn compile-clj [_]
  (b/compile-clj
    {:basis     basis
     :src-dirs  ["src"]
     :class-dir class-dir}))

(defn uber [opts]
  (clean nil)
  (compile-clj nil)
  (b/copy-dir {:src-dirs   ["src" "resources"]
                :target-dir class-dir})
  (b/uber {:class-dir class-dir
            :uber-file  jar-file
            :basis      basis
            :main       'myapp.core})
  (println "Built:" jar-file))
```

---

## ขั้นตอนที่ 1263: Kubernetes Deployment

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
  labels:
    app: myapp
    version: "1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # Zero-downtime
  template:
    metadata:
      labels:
        app: myapp
        version: "1.0.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port:   "9090"
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: myapp
          image: registry.example.com/myapp:1.0.0
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
              name: http
            - containerPort: 9090
              name: metrics
          
          env:
            - name: APP_ENV
              value: "production"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: myapp-secrets
                  key: database-url
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: myapp-secrets
                  key: jwt-secret
          
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "1000m"
          
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 10
            failureThreshold: 3
          
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 5
            failureThreshold: 3
          
          lifecycle:
            preStop:
              exec:
                # Allow graceful shutdown
                command: ["/bin/sh", "-c", "sleep 5"]
```

---

## ขั้นตอนที่ 1264: GitHub Actions CI/CD

```yaml
# .github/workflows/ci-cd.yml
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
    name: Test
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: DeLaGuardo/setup-clojure@12
        with:
          tools-deps: '1.11.1.1413'
      
      - name: Cache dependencies
        uses: actions/cache@v3
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-clojure-${{ hashFiles('deps.edn') }}
      
      - name: Run tests
        env:
          DATABASE_URL: "jdbc:postgresql://localhost:5432/test_db"
          DATABASE_PASSWORD: test
        run: clojure -M:test
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3

  build:
    name: Build and Push Docker Image
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    permissions:
      contents: read
      packages: write
    
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to Container Registry
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
            type=sha,prefix=sha-
            type=ref,event=branch
            type=semver,pattern={{version}}
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to:   type=gha,mode=max
          build-args: |
            APP_VERSION=${{ github.sha }}

  deploy:
    name: Deploy to Production
    needs: build
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: azure/setup-kubectl@v3
      
      - name: Configure kubectl
        run: |
          echo "${{ secrets.KUBE_CONFIG }}" | base64 -d > $HOME/.kube/config
      
      - name: Deploy with Helm
        run: |
          helm upgrade --install myapp ./helm/myapp \
            --namespace production \
            --set image.tag=${{ github.sha }} \
            --set replicaCount=3 \
            --atomic \
            --timeout 5m
```

---

## ขั้นตอนที่ 1265: Helm Chart

```yaml
# helm/myapp/values.yaml
replicaCount: 3

image:
  repository: ghcr.io/org/myapp
  pullPolicy: Always
  tag: "latest"

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"
  hosts:
    - host: api.myapp.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: myapp-tls
      hosts:
        - api.myapp.com

resources:
  requests:
    memory: 256Mi
    cpu: 250m
  limits:
    memory: 512Mi
    cpu: 1000m

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

postgresql:
  enabled: true
  auth:
    existingSecret: myapp-db-secret
  primary:
    persistence:
      size: 20Gi

redis:
  enabled: true
  auth:
    enabled: false
  master:
    persistence:
      size: 5Gi
```

---

## ขั้นตอนที่ 1266: Blue/Green Deployment

```clojure
;; Rolling deployment with zero downtime in Clojure

(ns myapp.shutdown
  (:require [clojure.core.async :as async]))

;; Graceful shutdown handler
(def in-flight-requests (atom 0))
(def shutdown-requested (atom false))

(defn wrap-graceful-shutdown [handler]
  (fn [request]
    (if @shutdown-requested
      {:status 503 :body {:error "Service shutting down"}}
      (do
        (swap! in-flight-requests inc)
        (try
          (handler request)
          (finally
            (swap! in-flight-requests dec)))))))

(defn graceful-shutdown! [server timeout-ms]
  (println "Graceful shutdown initiated...")
  (reset! shutdown-requested true)
  
  ;; Wait for in-flight requests
  (let [deadline (+ (System/currentTimeMillis) timeout-ms)]
    (loop []
      (when (and (pos? @in-flight-requests)
                  (< (System/currentTimeMillis) deadline))
        (Thread/sleep 100)
        (recur))))
  
  (println "Stopping server...")
  (server)  ; stop function from http-kit
  (println "Server stopped."))

;; Register shutdown hook
(defn register-shutdown-hook! [server]
  (.addShutdownHook (Runtime/getRuntime)
    (Thread.
      (fn []
        (graceful-shutdown! server 30000)))))
```

---

## Project: Complete CI/CD Pipeline

```yaml
# Complete workflow combining test, security scan, build, deploy

name: Full Pipeline

on:
  push:
    branches: [main]

jobs:
  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run SAST scan
        uses: returntocorp/semgrep-action@v1
        with:
          config: "p/clojure p/owasp-top-ten"
      - name: Dependency audit
        run: |
          clojure -Sdeps '{:deps {jonase/kibit {:mvn/version "0.1.8"}}}' \
            -m kibit.driver src/

  # Parallel test suites
  unit-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: clojure -M:test :unit

  integration-test:
    runs-on: ubuntu-latest
    services:
      postgres: {image: postgres:16, ...}
    steps:
      - uses: actions/checkout@v4
      - run: clojure -M:test :integration

  build-and-deploy:
    needs: [security-scan, unit-test, integration-test]
    runs-on: ubuntu-latest
    steps:
      - name: Build Docker image
        run: docker build -t myapp:${{ github.sha }} .
      - name: Push to registry
        run: docker push ghcr.io/org/myapp:${{ github.sha }}
      - name: Deploy Canary (10%)
        run: |
          kubectl set image deployment/myapp-canary \
            myapp=ghcr.io/org/myapp:${{ github.sha }}
      - name: Monitor canary (5 minutes)
        run: sleep 300 && check-canary-health.sh
      - name: Full rollout
        run: |
          kubectl set image deployment/myapp \
            myapp=ghcr.io/org/myapp:${{ github.sha }}
```

---

*Part 43 จาก 100+ | ขั้นตอน 1261-1290 จาก 1000+*
