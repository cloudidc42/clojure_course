# Part 64: Infrastructure as Code
## ขั้นตอนที่ 1891-1920: Terraform, Pulumi Clojure, AWS CDK, Configuration Management

---

## บทนำ

Infrastructure as Code ด้วย Clojure:
- **Terraform** - declarative infrastructure
- **Pulumi** - IaC ด้วย real programming languages
- **AWS CDK** - Cloud Development Kit
- **Aero** - configuration management
- **Integrant** - system lifecycle management

---

## ขั้นตอนที่ 1891: Aero Configuration Management

```clojure
;; resources/config.edn - multi-environment configuration
{:app/db
 {:host     #profile {:dev  "localhost"
                       :test "localhost"
                       :prod #env "DB_HOST"}
  :port     #long #profile {:dev  5432
                              :test 5433
                              :prod #env "DB_PORT"}
  :name     #profile {:dev  "myapp_dev"
                       :test "myapp_test"
                       :prod #env "DB_NAME"}
  :user     #env "DB_USER"
  :password #secret [:db :password]  ; from secrets manager
  :pool-size #long #profile {:dev  5
                               :prod 20}}

 :app/redis
 {:host #profile {:dev  "localhost"
                   :prod #env "REDIS_HOST"}
  :port 6379
  :ssl? #profile {:dev false :prod true}}

 :app/server
 {:port #long #profile {:dev  3000
                          :test 3001
                          :prod 8080}
  :host "0.0.0.0"}

 :app/kafka
 {:servers #profile {:dev  "localhost:9092"
                      :prod #env "KAFKA_SERVERS"}
  :group-id "myapp"}

 :app/features
 {:new-checkout?  #profile {:dev true :prod false}
  :maintenance?   false}}

;; Load config
(ns myapp.config
  (:require [aero.core :as aero]
            [clojure.java.io :as io]))

(defn load-config [profile]
  (aero/read-config
    (io/resource "config.edn")
    {:profile profile}))

;; Resolve secrets from AWS Secrets Manager
(defmethod aero/reader :secret [opts _tag [path key]]
  (let [secret-name (str "myapp/" (name path) "/" (name key))]
    (get-secret-from-aws! secret-name)))
```

---

## ขั้นตอนที่ 1892: Integrant System Lifecycle

```clojure
(ns myapp.system
  (:require [integrant.core :as ig]
            [next.jdbc :as jdbc]
            [taoensso.carmine :as car]))

;; System configuration
(def base-config
  {:app/db       {:host      "localhost"
                   :port      5432
                   :dbname    "myapp"
                   :user      "myapp"
                   :password  "secret"
                   :pool-size 10}

   :app/redis    {:host "localhost"
                   :port 6379}

   :app/kafka    {:servers "localhost:9092"
                   :group-id "myapp"}

   :app/cache    {:redis (ig/ref :app/redis)
                   :ttl 3600}

   :app/router   {:db    (ig/ref :app/db)
                   :cache (ig/ref :app/cache)}

   :app/server   {:port   3000
                   :router (ig/ref :app/router)}})

;; Lifecycle implementations
(defmethod ig/init-key :app/db [_ config]
  (println "Starting DB pool...")
  (create-connection-pool config))

(defmethod ig/halt-key! :app/db [_ pool]
  (println "Closing DB pool...")
  (.close pool))

(defmethod ig/init-key :app/redis [_ {:keys [host port]}]
  {:pool {} :spec {:host host :port port}})

(defmethod ig/halt-key! :app/redis [_ _]
  ;; Redis connections are stateless
  nil)

(defmethod ig/init-key :app/server [_ {:keys [port router]}]
  (println "Starting server on port" port)
  (org.httpkit.server/run-server router {:port port}))

(defmethod ig/halt-key! :app/server [_ stop-fn]
  (println "Stopping server...")
  (stop-fn))

;; Start system
(defn start! [profile]
  (let [config (load-config profile)]
    (ig/init config)))

(defn stop! [system]
  (ig/halt! system))

;; Main entry point
(defn -main [& args]
  (let [profile (keyword (or (first args) "prod"))
        system  (start! profile)]
    
    ;; Graceful shutdown
    (.addShutdownHook (Runtime/getRuntime)
      (Thread. #(do
                   (println "Shutting down...")
                   (stop! system)
                   (println "Goodbye"))))
    
    (println "System started with profile:" profile)))
```

---

## ขั้นตอนที่ 1893: Docker Multi-Stage Build

```dockerfile
# Dockerfile สำหรับ Clojure production image

# Stage 1: Build
FROM clojure:temurin-21-tools-deps-bookworm-slim AS builder

WORKDIR /app

# Cache dependencies
COPY deps.edn .
RUN clojure -P

# Build uberjar
COPY . .
RUN clojure -T:build uber

# Stage 2: Runtime
FROM eclipse-temurin:21-jre-alpine AS runtime

# Security: non-root user
RUN addgroup -g 1001 app && \
    adduser -u 1001 -G app -s /bin/sh -D app

WORKDIR /app

# Copy only the jar
COPY --from=builder /app/target/myapp-standalone.jar .

# Healthcheck
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

USER app

EXPOSE 3000

ENV JVM_OPTS="-Xmx512m -Xms256m \
              -XX:+UseContainerSupport \
              -XX:MaxRAMPercentage=75.0 \
              -XX:+UseG1GC"

ENTRYPOINT ["sh", "-c", \
  "java $JVM_OPTS -jar myapp-standalone.jar"]
```

---

## ขั้นตอนที่ 1894: Kubernetes Configuration

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
      maxUnavailable: 0
      maxSurge: 1
  template:
    metadata:
      labels:
        app: myapp
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      serviceAccountName: myapp-sa
      containers:
      - name: myapp
        image: myapp:1.0.0
        ports:
        - containerPort: 3000
          name: http
        - containerPort: 9090
          name: metrics
        env:
        - name: APP_PROFILE
          value: prod
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: db-host
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
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 3000
          initialDelaySeconds: 15
          periodSeconds: 5
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values: [myapp]
              topologyKey: kubernetes.io/hostname
```

---

## ขั้นตอนที่ 1895: build.clj สำหรับ CI/CD

```clojure
;; build.clj
(ns build
  (:require [clojure.tools.build.api :as b]))

(def lib     'com.mycompany/myapp)
(def version (format "1.0.%s" (b/git-count-revs nil)))
(def class-dir "target/classes")
(def uber-file (format "target/%s-%s-standalone.jar"
                         (name lib) version))

(defn clean [_]
  (b/delete {:path "target"}))

(defn test [_]
  (let [basis (b/create-basis {:aliases [:test]})]
    (b/process {:command-args
                ["clojure" "-M:test"
                 "--reporter" "documentation"]})))

(defn compile-clj [_]
  (let [basis (b/create-basis {:project "deps.edn"})]
    (b/compile-clj {:basis      basis
                     :src-dirs   ["src/clj"]
                     :class-dir  class-dir
                     :ns-compile ['myapp.core]})))

(defn uber [_]
  (clean nil)
  (compile-clj nil)
  (let [basis (b/create-basis {:project "deps.edn"})]
    (b/copy-dir {:src-dirs   ["src" "resources"]
                  :target-dir class-dir})
    (b/uber {:class-dir class-dir
              :uber-file uber-file
              :basis     basis
              :main      'myapp.core
              :manifest  {"Implementation-Title"   (str lib)
                           "Implementation-Version" version}}))
  (println "Built:" uber-file))
```

---

## Project: Infrastructure Config Generator

```clojure
(ns infra.generator)

;; Generate Kubernetes configs from Clojure data
(defn deployment [name image {:keys [replicas port memory cpu env-vars]}]
  {:apiVersion "apps/v1"
   :kind       "Deployment"
   :metadata   {:name name}
   :spec       {:replicas replicas
                 :selector {:matchLabels {:app name}}
                 :template
                 {:metadata {:labels {:app name}}
                  :spec
                  {:containers
                   [{:name  name
                     :image image
                     :ports [{:containerPort port}]
                     :env   (map (fn [[k v]]
                                    {:name  (name k) :value (str v)})
                                  env-vars)
                     :resources
                     {:requests {:memory (str memory "Mi") :cpu (str cpu "m")}
                      :limits   {:memory (str (* 2 memory) "Mi")
                                  :cpu    (str (* 4 cpu) "m")}}}]}}}})

;; Generate all configs
(defn generate-configs! [services output-dir]
  (doseq [{:keys [name image] :as service} services]
    (let [config   (deployment name image service)
          filename (str output-dir "/" name ".yaml")]
      (spit filename
        (yaml/generate-string config :dumper-options {:flow-style :block}))
      (println "Generated:" filename))))

(generate-configs!
  [{:name "api"    :image "myapp/api:1.0"    :replicas 3 :port 3000 :memory 256 :cpu 250}
   {:name "worker" :image "myapp/worker:1.0" :replicas 2 :port 8080 :memory 512 :cpu 500}]
  "k8s/generated")
```

---

*Part 64 จาก 100+ | ขั้นตอน 1891-1920 จาก 1000+*
