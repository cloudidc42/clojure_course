# Part 55: Testing Strategies ขั้นสูง
## ขั้นตอนที่ 1621-1650: Unit Tests, Integration Tests, Property Tests, Test Fixtures, TDD

---

## บทนำ

Testing strategies ระดับ professional:
- **Unit tests** - test ทีละ function
- **Integration tests** - test หลาย components ร่วมกัน
- **Property-based tests** - test with generated data
- **Contract tests** - test API contracts
- **Test fixtures** - setup/teardown patterns

---

## ขั้นตอนที่ 1621: Unit Testing ด้วย clojure.test

```clojure
(ns myapp.products-test
  (:require [clojure.test :refer [deftest is are testing use-fixtures]]
            [myapp.products :as products]))

;; Basic tests
(deftest test-create-product
  (testing "Valid product"
    (let [product (products/create-product
                    {:name "Widget" :price 9.99 :sku "WGT-001"})]
      (is (uuid? (:id product)))
      (is (= "Widget" (:name product)))
      (is (= 9.99 (:price product)))))
  
  (testing "Missing name throws"
    (is (thrown-with-msg? clojure.lang.ExceptionInfo
                           #"Validation failed"
          (products/create-product {:price 9.99 :sku "WGT-001"}))))
  
  (testing "Negative price throws"
    (is (thrown? Exception
          (products/create-product {:name "Widget" :price -1 :sku "WGT-001"})))))

;; Parameterized tests
(deftest test-price-formatting
  (are [input expected]
       (= expected (products/format-price input))
    9.99    "$9.99"
    10.0    "$10.00"
    1000.0  "$1,000.00"
    0.1     "$0.10"))

;; Testing pure functions
(deftest test-calculate-discount
  (let [f products/calculate-discount]
    (is (= 9.0  (f 10.0 0.10)))   ; 10% off
    (is (= 75.0 (f 100.0 0.25)))  ; 25% off
    (is (= 0.0  (f 10.0 1.0)))    ; 100% off
    (is (= 10.0 (f 10.0 0.0)))))  ; no discount

;; Test data builders
(defn make-product [& {:as overrides}]
  (merge {:id    (java.util.UUID/randomUUID)
          :name  "Test Product"
          :price 10.0
          :sku   (str "TEST-" (rand-int 10000))
          :stock 100}
         overrides))

(defn make-order [& {:as overrides}]
  (merge {:id      (java.util.UUID/randomUUID)
          :user-id (java.util.UUID/randomUUID)
          :items   [(make-product)]
          :status  :pending
          :total   10.0}
         overrides))
```

---

## ขั้นตอนที่ 1622: Integration Testing

```clojure
(ns myapp.integration.orders-test
  (:require [clojure.test :refer [deftest is testing use-fixtures]]
            [next.jdbc :as jdbc]
            [myapp.system :as system]
            [myapp.orders :as orders]))

;; Test database fixture
(def test-db
  (delay (system/create-test-db!)))

(defn with-clean-db [test-fn]
  (jdbc/with-transaction [tx @test-db {:rollback-only true}]
    (binding [*db* tx]
      (test-fn))))

(use-fixtures :each with-clean-db)

;; Seed data helpers
(defn seed-user! [& {:as attrs}]
  (jdbc/execute-one! *db*
    ["INSERT INTO users (id, name, email, role)
      VALUES (gen_random_uuid(), ?, ?, 'user')
      RETURNING *"
     (:name attrs "Test User")
     (:email attrs "test@example.com")]))

(defn seed-product! [& {:as attrs}]
  (jdbc/execute-one! *db*
    ["INSERT INTO products (id, name, price, sku, stock)
      VALUES (gen_random_uuid(), ?, ?, ?, ?)
      RETURNING *"
     (:name attrs "Widget")
     (:price attrs 9.99)
     (:sku attrs "WGT-001")
     (:stock attrs 100)]))

;; Integration tests
(deftest test-place-order-integration
  (testing "Successfully places order and updates inventory"
    (let [user    (seed-user! :email "buyer@example.com")
          product (seed-product! :price 25.0 :stock 10)
          user-id (:users/id user)
          prod-id (:products/id product)
          
          order   (orders/place-order! *db*
                    {:user-id user-id
                     :items [{:product-id prod-id :qty 2 :price 25.0}]
                     :total 50.0})]
      
      (is (= :confirmed (:status order)))
      (is (uuid? (:id order)))
      
      ;; Verify inventory decreased
      (let [updated-product (jdbc/execute-one! *db*
                              ["SELECT stock FROM products WHERE id = ?" prod-id])]
        (is (= 8 (:products/stock updated-product))))))
  
  (testing "Fails when insufficient stock"
    (let [user    (seed-user! :email "buyer2@example.com")
          product (seed-product! :stock 1)
          
          result  (try
                    (orders/place-order! *db*
                      {:user-id (:users/id user)
                       :items [{:product-id (:products/id product)
                                :qty 5 :price 9.99}]})
                    :success
                    (catch clojure.lang.ExceptionInfo e
                      {:error (ex-message e)}))]
      (is (map? result))
      (is (= "Insufficient inventory" (:error result))))))
```

---

## ขั้นตอนที่ 1623: Property-Based Testing

```clojure
(ns myapp.property-test
  (:require [clojure.test :refer [deftest is]]
            [clojure.test.check :as tc]
            [clojure.test.check.generators :as gen]
            [clojure.test.check.properties :as prop]
            [clojure.test.check.clojure-test :refer [defspec]]))

;; Generators
(def gen-product-name
  (gen/such-that #(pos? (count %))
                  (gen/resize 100 gen/string-alphanumeric)))

(def gen-price
  (gen/fmap #(/ % 100.0)
             (gen/choose 1 100000)))  ; 0.01 to 1000.00

(def gen-product
  (gen/hash-map
    :name  gen-product-name
    :price gen-price
    :sku   (gen/fmap #(str "SKU-" %) gen/nat)))

;; Properties
(defspec price-always-positive 100
  (prop/for-all [product gen-product]
    (>= (:price product) 0)))

(defspec discount-never-exceeds-price 100
  (prop/for-all [price  (gen/double* {:min 0.01 :max 1000.0 :NaN? false})
                  discount (gen/double* {:min 0.0  :max 1.0   :NaN? false})]
    (let [result (products/apply-discount price discount)]
      (and (>= result 0)
           (<= result price)))))

(defspec parse-price-roundtrip 100
  (prop/for-all [price gen-price]
    (= price (products/parse-price (products/format-price price)))))

;; Stateful property tests (model-based)
(defspec shopping-cart-invariants 50
  (prop/for-all [ops (gen/vector
                       (gen/one-of
                         [(gen/hash-map :op (gen/return :add)
                                         :item gen-product)
                          (gen/hash-map :op (gen/return :remove)
                                         :idx gen/nat)])
                       0 20)]
    (let [cart (reduce
                 (fn [c op]
                   (case (:op op)
                     :add    (cart/add-item c (:item op))
                     :remove (if (seq (:items c))
                               (cart/remove-item c (:idx op))
                               c)))
                 (cart/create)
                 ops)]
      ;; Invariants
      (and (>= (count (:items cart)) 0)
           (= (:total cart)
              (reduce + (map #(* (:price %) (:qty %)) (:items cart))))))))
```

---

## ขั้นตอนที่ 1624: Mock และ Stub Patterns

```clojure
(ns myapp.mocking-test
  (:require [clojure.test :refer [deftest is testing]]
            [myapp.notifications :as notif]))

;; Simple atom-based mock
(defn make-mock-emailer []
  (let [sent (atom [])]
    {:send!  (fn [to subject body]
               (swap! sent conj {:to to :subject subject :body body}))
     :sent   sent
     :clear! #(reset! sent [])}))

(deftest test-order-confirmation-email
  (let [emailer (make-mock-emailer)
        order   {:id "123" :total 50.0
                  :user {:email "buyer@example.com" :name "Alice"}}]
    
    (notif/send-order-confirmation! order (:send! emailer))
    
    (is (= 1 (count @(:sent emailer))))
    (let [email (first @(:sent emailer))]
      (is (= "buyer@example.com" (:to email)))
      (is (clojure.string/includes? (:subject email) "Order Confirmed"))
      (is (clojure.string/includes? (:body email) "123")))))

;; With-redefs for dynamic binding
(deftest test-payment-failure-handling
  (testing "Retries on network error"
    (let [call-count (atom 0)]
      (with-redefs [payment/charge!
                    (fn [& _]
                      (swap! call-count inc)
                      (if (< @call-count 3)
                        (throw (java.io.IOException. "Network error"))
                        {:id "charge_123" :status "succeeded"}))]
        
        (let [result (orders/place-order-with-retry! test-order)]
          (is (= 3 @call-count))
          (is (= "succeeded" (get-in result [:payment :status]))))))))

;; Protocol-based mocks
(defprotocol PaymentGateway
  (charge! [this amount currency])
  (refund!  [this charge-id amount]))

(defrecord MockPaymentGateway [responses calls]
  PaymentGateway
  (charge! [_ amount _]
    (swap! calls conj {:op :charge :amount amount})
    (or (first (filter #(= :charge (:op %)) @responses))
        {:id "mock_charge_123" :status "succeeded"}))
  (refund!  [_ charge-id _]
    (swap! calls conj {:op :refund :charge-id charge-id})
    {:id "mock_refund_456" :status "succeeded"}))

(defn make-mock-payment-gateway []
  (->MockPaymentGateway (atom []) (atom [])))
```

---

## ขั้นตอนที่ 1625: Test Namespaces Organization

```clojure
;; test/myapp/
;; ├── unit/
;; │   ├── products_test.clj
;; │   ├── orders_test.clj
;; │   └── cart_test.clj
;; ├── integration/
;; │   ├── orders_api_test.clj
;; │   └── payments_test.clj
;; └── e2e/
;;     └── checkout_flow_test.clj

;; deps.edn test aliases
;; {:aliases
;;   {:test {:extra-paths ["test"]
;;           :extra-deps  {lambdaisland/kaocha {:mvn/version "1.87.1366"}}
;;           :main-opts   ["-m" "kaocha.runner"]}
;;    :test-unit {:extra-paths ["test"]
;;                :extra-deps  {lambdaisland/kaocha {:mvn/version "1.87.1366"}}
;;                :main-opts   ["-m" "kaocha.runner" "--focus-metadata" ":unit"]}}}

;; tests.edn for Kaocha
{:kaocha/tests
 [{:kaocha.testable/id :unit
   :kaocha.testable/type :kaocha.type/clojure.test
   :kaocha/source-paths ["src"]
   :kaocha/test-paths   ["test/unit"]
   :kaocha/ns-patterns  [#"-test$"]}
  {:kaocha.testable/id :integration
   :kaocha.testable/type :kaocha.type/clojure.test
   :kaocha/source-paths ["src"]
   :kaocha/test-paths   ["test/integration"]
   :kaocha/ns-patterns  [#"-test$"]}]}

;; Run unit tests only
;; clj -M:test --focus :unit

;; Mark tests with metadata
(deftest ^:unit test-calculate-total
  (is (= 10.0 (calculate-total [{:price 5.0 :qty 2}]))))

(deftest ^:integration test-create-order-db
  (is (some? (create-order! test-db order-data))))

(deftest ^:slow test-send-email
  (is (= :sent (send-email! real-emailer "test@example.com" "Hello"))))
```

---

## Project: Test Utility Library

```clojure
(ns test.utils
  (:require [next.jdbc :as jdbc]
            [clojure.test :refer [is]]))

;; Database test helpers
(defmacro with-tx-rollback [db & body]
  `(jdbc/with-transaction [tx# ~db {:rollback-only true}]
     (binding [*db* tx#]
       ~@body)))

;; Assert helpers
(defn assert-contains [expected actual]
  (doseq [[k v] expected]
    (is (= v (get actual k))
        (str "Expected " k " = " v " but got " (get actual k)))))

(defn assert-error [expected-msg f & args]
  (try
    (apply f args)
    (is false (str "Expected exception with message: " expected-msg))
    (catch clojure.lang.ExceptionInfo e
      (is (clojure.string/includes? (ex-message e) expected-msg)
          (str "Expected '" expected-msg "' in '" (ex-message e) "'")))))

;; Time helpers for tests
(defmacro freeze-time [instant & body]
  `(with-redefs [java.time.Instant/now (constantly ~instant)]
     ~@body))

;; Async test helpers
(defn wait-for [pred timeout-ms]
  (loop [elapsed 0]
    (if (pred)
      true
      (if (> elapsed timeout-ms)
        false
        (do (Thread/sleep 10)
            (recur (+ elapsed 10)))))))

(defmacro eventually [& body]
  `(is (wait-for #(try ~@body (catch Exception _# false)) 5000)
       "Condition never became true within 5 seconds"))
```

---

*Part 55 จาก 100+ | ขั้นตอน 1621-1650 จาก 1000+*
