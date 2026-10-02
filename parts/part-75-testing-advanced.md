# Part 75: Advanced Testing Patterns
## ขั้นตอนที่ 2221-2250: Contract Testing, Mutation Testing, Fuzzing, Load Testing

---

## บทนำ

Advanced testing:
- **Contract testing** - API consumer/provider contracts
- **Mutation testing** - ตรวจว่า tests จริงหรือไม่
- **Fuzz testing** - ใส่ random input
- **Load testing** - ทดสอบ performance
- **Chaos testing** - ทดสอบ failure scenarios

---

## ขั้นตอนที่ 2221: Contract Testing

```clojure
(ns myapp.test.contract
  (:require [clj-pact.core :as pact]
            [clojure.test :refer [deftest is]]))

;; Consumer-Driven Contract Testing with Pact
;; Consumer defines what it needs from Provider

;; Consumer side: define contract
(defn define-product-api-contract []
  (pact/consumer "OrderService"
    (pact/provider "ProductService"
      
      ;; Define interaction
      (pact/interaction "get product by id"
        :upon-receiving "a request for product 123"
        :with {:method  "GET"
               :path    "/api/products/123"
               :headers {"Accept" "application/json"}}
        :will-respond-with
        {:status  200
         :headers {"Content-Type" "application/json"}
         :body    {:id    "123"
                   :name  pact/like-string
                   :price pact/like-decimal
                   :stock pact/like-integer}}))))

;; Provider side: verify contract
(deftest verify-product-api-contract
  (pact/verify-provider
    {:provider-name    "ProductService"
     :provider-base-url "http://localhost:3000"
     :pact-files        ["pacts/OrderService-ProductService.json"]
     
     ;; State setup
     :provider-states
     {"product 123 exists"
      (fn [_] (create-test-product! test-db {:id "123" :name "Widget" :price 9.99}))
      
      "product 123 does not exist"
      (fn [_] (delete-test-product! test-db "123"))}}))
```

---

## ขั้นตอนที่ 2222: Property-Based Testing ขั้นสูง

```clojure
(ns myapp.test.property
  (:require [clojure.test.check :as tc]
            [clojure.test.check.generators :as gen]
            [clojure.test.check.properties :as prop]))

;; Generators for domain objects
(def gen-money
  (gen/fmap
    (fn [[amount currency]]
      {:amount (bigdec (/ amount 100.0)) :currency currency})
    (gen/tuple
      (gen/large-integer* {:min 1 :max 1000000})
      (gen/elements ["USD" "EUR" "GBP" "THB"]))))

(def gen-order-item
  (gen/hash-map
    :product-id gen/uuid
    :quantity   (gen/large-integer* {:min 1 :max 100})
    :price      (gen/fmap #(bigdec (/ % 100.0))
                           (gen/large-integer* {:min 1 :max 100000}))))

(def gen-order
  (gen/let [items    (gen/not-empty (gen/vector gen-order-item 1 10))
             discount (gen/double* {:min 0.0 :max 1.0})]
    {:items    items
     :discount discount
     :total    (reduce + (map #(* (:price %) (:quantity %)) items))}))

;; Properties
(defn order-total-property []
  (prop/for-all [order gen-order]
    ;; Total must equal sum of items
    (let [computed-total (reduce + (map #(* (:price %) (:quantity %)) (:items order)))]
      (= (:total order) computed-total))))

(defn discount-never-exceeds-total []
  (prop/for-all [order gen-order]
    (let [discount-amount (* (:total order) (:discount order))
          final-total     (- (:total order) discount-amount)]
      (>= final-total 0))))

(defn order-serialization-roundtrip []
  (prop/for-all [order gen-order]
    (= order
       (-> order
           json/generate-string
           (json/parse-string true)
           (update :total bigdec)
           (update-in [:items] (fn [items]
                                  (map #(update % :price bigdec) items)))))))

;; Run
(tc/quick-check 1000 (order-total-property))
(tc/quick-check 1000 (discount-never-exceeds-total))
```

---

## ขั้นตอนที่ 2223: Integration Tests with Test Containers

```clojure
(ns myapp.test.integration
  (:require [clojure.test :refer [use-fixtures deftest is testing]]
            [testcontainers-clj.core :as tc]))

;; Start real services for integration tests
(def postgres-container
  (tc/start! {:image      "postgres:15"
              :env        {"POSTGRES_DB"       "test"
                           "POSTGRES_USER"     "test"
                           "POSTGRES_PASSWORD" "test"}
              :wait-for   {:log-message #"database system is ready"}
              :exposed-ports [5432]}))

(def redis-container
  (tc/start! {:image        "redis:7-alpine"
              :exposed-ports [6379]}))

;; Create test DB connection
(defonce test-db
  (atom nil))

(defn setup-integration-tests! []
  (let [pg-port    (tc/mapped-port postgres-container 5432)
        redis-port (tc/mapped-port redis-container 6379)]
    
    (reset! test-db
      (jdbc/get-datasource
        {:dbtype   "postgresql"
         :host     "localhost"
         :port     pg-port
         :dbname   "test"
         :user     "test"
         :password "test"}))
    
    (run-migrations! @test-db)))

(use-fixtures :once
  (fn [f]
    (setup-integration-tests!)
    (f)
    (tc/stop! postgres-container)
    (tc/stop! redis-container)))

(use-fixtures :each
  (fn [f]
    ;; Rollback each test
    (jdbc/with-transaction [tx @test-db {:rollback-only true}]
      (binding [*db* tx]
        (f)))))

;; Real integration test
(deftest test-order-creation-integration
  (testing "creating an order updates inventory"
    (let [product (create-product! *db*
                    {:name "Widget" :price 9.99 :stock 100})
          order   (create-order! *db*
                    {:items [{:product-id (:id product)
                              :quantity   5}]})]
      
      (is (= "pending" (:status order)))
      
      (let [updated-product (get-product *db* (:id product))]
        (is (= 95 (:stock updated-product)))))))
```

---

## ขั้นตอนที่ 2224: Load Testing

```clojure
(ns myapp.test.load)

;; Simple load test framework
(defn run-load-test!
  [{:keys [target-fn concurrency duration-seconds ramp-up-seconds]}]
  (let [start-time (System/currentTimeMillis)
        end-time   (+ start-time (* duration-seconds 1000))
        results    (java.util.concurrent.ConcurrentLinkedQueue.)
        errors     (atom 0)
        
        worker (fn []
                 (loop []
                   (when (< (System/currentTimeMillis) end-time)
                     (let [req-start (System/currentTimeMillis)]
                       (try
                         (target-fn)
                         (.add results (- (System/currentTimeMillis) req-start))
                         (catch Exception e
                           (swap! errors inc))))
                     (recur))))
        
        threads (mapv (fn [i]
                        (let [t (Thread. worker)]
                          (when ramp-up-seconds
                            (Thread/sleep (long (* 1000 (/ ramp-up-seconds concurrency)))))
                          (.start t)
                          t))
                       (range concurrency))]
    
    (doseq [t threads] (.join t))
    
    (let [latencies (sort (vec results))
          n         (count latencies)
          p50-idx   (int (* n 0.50))
          p95-idx   (int (* n 0.95))
          p99-idx   (int (* n 0.99))]
      
      {:total-requests    n
       :errors            @errors
       :error-rate        (/ @errors (double (+ n @errors)))
       :rps               (/ n duration-seconds)
       :latency-ms
       {:mean (when (pos? n) (/ (reduce + latencies) n))
        :p50  (nth latencies p50-idx)
        :p95  (nth latencies p95-idx)
        :p99  (nth latencies p99-idx)
        :max  (last latencies)}})))

;; Usage
(def result
  (run-load-test!
    {:target-fn       #(http/get "http://localhost:3000/api/products")
     :concurrency     50
     :duration-seconds 60
     :ramp-up-seconds 10}))

(println "Load test results:")
(println "  RPS:" (:rps result))
(println "  P95 latency:" (get-in result [:latency-ms :p95]) "ms")
(println "  Error rate:" (* 100 (:error-rate result)) "%")
```

---

## ขั้นตอนที่ 2225: Chaos Testing

```clojure
;; Chaos engineering: test system resilience
(ns myapp.test.chaos)

;; Fault injection middleware
(defn wrap-chaos [handler chaos-config]
  (fn [request]
    (let [{:keys [latency-probability latency-ms
                   error-probability]} chaos-config]
      
      ;; Random latency injection
      (when (and latency-probability
                  (< (rand) latency-probability))
        (Thread/sleep (or latency-ms 500)))
      
      ;; Random error injection
      (if (and error-probability
                (< (rand) error-probability))
        {:status 500 :body {:error "Chaos injected failure"}}
        (handler request)))))

;; DB chaos: randomly fail queries
(defn chaos-db-wrapper [db {:keys [error-probability]}]
  (reify javax.sql.DataSource
    (getConnection [_]
      (if (< (rand) (or error-probability 0))
        (throw (java.sql.SQLException "Chaos: DB connection refused"))
        (.getConnection db)))))

;; Network partition simulation
(defn simulate-network-partition!
  [service-name duration-seconds registry]
  (println "Simulating partition for" service-name "for" duration-seconds "seconds")
  
  ;; Remove service instances
  (let [instances (discover registry service-name)]
    (doseq [inst instances]
      (deregister! registry service-name (:id inst)))
    
    ;; Re-register after duration
    (future
      (Thread/sleep (* duration-seconds 1000))
      (doseq [inst instances]
        (register! registry service-name inst))
      (println "Partition healed for" service-name))))
```

---

*Part 75 จาก 100+ | ขั้นตอน 2221-2250 จาก 1000+*
