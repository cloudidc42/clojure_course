# Part 33: Advanced Testing Strategies
## ขั้นตอนที่ 961-990: Property-Based Testing, Mutation Testing, Contract Testing, BDD

---

## บทนำ

Testing strategies ขั้นสูง:
- **Property-Based Testing** - test.check เพื่อค้นหา edge cases
- **Mutation Testing** - ทดสอบว่า tests detect bugs จริง
- **Contract Testing** - verify API contracts
- **BDD** - Behavior-Driven Development
- **Kaocha** - test runner ที่ flexible

---

## ขั้นตอนที่ 961: test.check - Property-Based Testing

```clojure
;; deps.edn
;; {:deps {org.clojure/test.check {:mvn/version "1.1.1"}}}

(ns myapp.property-test
  (:require [clojure.test :refer :all]
            [clojure.test.check :as tc]
            [clojure.test.check.generators :as gen]
            [clojure.test.check.properties :as prop]
            [clojure.test.check.clojure-test :refer [defspec]]))

;; Property: reverse of reverse is original
(defspec reverse-prop 100
  (prop/for-all [v (gen/vector gen/int)]
    (= v (reverse (reverse v)))))

;; Property: sort result is ordered
(defspec sort-prop 200
  (prop/for-all [v (gen/vector gen/int)]
    (let [sorted (sort v)]
      (every? (fn [[a b]] (<= a b))
              (partition 2 1 sorted)))))

;; Property: addition is commutative
(defspec add-commutative 100
  (prop/for-all [a gen/int
                  b gen/int]
    (= (+ a b) (+ b a))))

;; Property: json roundtrip
(defspec json-roundtrip 100
  (prop/for-all [m (gen/map
                     gen/keyword
                     (gen/one-of [gen/string gen/int gen/boolean]))]
    (= m (cheshire.core/parse-string
           (cheshire.core/generate-string m)
           true))))
```

---

## ขั้นตอนที่ 962: Custom Generators

```clojure
;; Custom generators สำหรับ domain types

;; Email generator
(def email-gen
  (gen/fmap
    (fn [[local domain]]
      (str local "@" domain ".com"))
    (gen/tuple
      (gen/such-that #(seq %) gen/string-alphanumeric)
      (gen/such-that #(seq %) gen/string-alphanumeric))))

;; User generator
(def user-gen
  (gen/hash-map
    :id    gen/pos-int
    :name  (gen/such-that #(seq %) gen/string-alphanumeric)
    :email email-gen
    :age   (gen/choose 18 120)
    :role  (gen/elements [:user :admin :moderator])))

;; Order generator
(def product-gen
  (gen/hash-map
    :id    gen/pos-int
    :price (gen/fmap #(/ % 100.0) (gen/choose 1 100000))
    :name  gen/string-alphanumeric))

(def order-item-gen
  (gen/hash-map
    :product product-gen
    :qty     (gen/choose 1 10)))

(def order-gen
  (gen/hash-map
    :id     gen/pos-int
    :user   user-gen
    :items  (gen/vector order-item-gen 1 10)
    :status (gen/elements [:pending :confirmed :shipped :delivered])))

;; Test with domain generators
(defspec order-total-non-negative 100
  (prop/for-all [order order-gen]
    (let [total (calculate-order-total order)]
      (>= total 0))))

(defspec order-discount-reduces-total 100
  (prop/for-all [order order-gen
                  discount (gen/choose 1 50)]
    (let [original-total    (calculate-order-total order)
          discounted-total  (apply-discount order discount)]
      (<= discounted-total original-total))))
```

---

## ขั้นตอนที่ 963: Stateful Property Testing

```clojure
;; Test stateful systems: sequence of operations

(defn gen-cart-commands []
  (gen/vector
    (gen/one-of
      [(gen/fmap #(vector :add % (inc (rand-int 5))) (gen/choose 1 100))
       (gen/fmap #(vector :remove %) (gen/choose 1 100))
       (gen/return [:clear])])
    0 20))

;; Reference implementation (simple, correct but slow)
(defn reference-cart []
  (atom {}))

(defn apply-command-reference [cart cmd]
  (case (first cmd)
    :add    (swap! cart update (second cmd) (fnil + 0) (nth cmd 2))
    :remove (swap! cart dissoc (second cmd))
    :clear  (reset! cart {})))

;; System under test
(defn apply-command-sut [cart cmd]
  (case (first cmd)
    :add    (cart-service/add-item! cart (second cmd) (nth cmd 2))
    :remove (cart-service/remove-item! cart (second cmd))
    :clear  (cart-service/clear! cart)))

;; Property: SUT matches reference
(defspec cart-matches-reference 50
  (prop/for-all [commands (gen-cart-commands)]
    (let [ref-cart (reference-cart)
          sut-cart (cart-service/new-cart)]
      (doseq [cmd commands]
        (apply-command-reference ref-cart cmd)
        (apply-command-sut sut-cart cmd))
      (= @ref-cart (cart-service/get-items sut-cart)))))
```

---

## ขั้นตอนที่ 964: Contract Testing

```clojure
;; Contract testing: verify API contracts between services
;; Consumer-driven contracts with Pact

;; deps.edn
;; {:deps {au.com.dius.pact.consumer/junit5 {:mvn/version "4.6.7"}}}

(ns myapp.contract-test
  (:require [clojure.test :refer :all]
            [clj-http.client :as http]))

;; Define expected contract
(def user-api-contract
  {:provider "UserService"
   :consumer "OrderService"
   :interactions
   [{:description "Get user by ID"
     :request     {:method "GET"
                    :path   "/api/users/1"
                    :headers {"Accept" "application/json"}}
     :response    {:status  200
                    :headers {"Content-Type" "application/json"}
                    :body    {:id    1
                               :name  (fn [v] (string? v))
                               :email (fn [v] (re-matches #".+@.+\..+" v))}}}]})

;; Verify contract against real provider
(deftest verify-user-api-contract
  (doseq [{:keys [description request response]} (:interactions user-api-contract)]
    (testing description
      (let [actual (http/request
                     {:method  (keyword (clojure.string/lower-case (:method request)))
                      :url     (str "http://user-service" (:path request))
                      :headers (:headers request)
                      :as      :json})]
        (is (= (:status response) (:status actual)))
        ;; Verify response body matches contract
        (doseq [[k validator] (:body response)]
          (let [actual-val (get-in actual [:body k])]
            (if (fn? validator)
              (is (validator actual-val) (str "Field " k " failed validation"))
              (is (= validator actual-val)))))))))
```

---

## ขั้นตอนที่ 965: Kaocha Test Runner

```clojure
;; tests.edn - Kaocha configuration
;; {:kaocha/tests [{:kaocha.testable/id :unit
;;                   :kaocha/test-paths ["test/unit"]}
;;                  {:kaocha.testable/id :integration
;;                   :kaocha/test-paths ["test/integration"]
;;                   :kaocha.filter/tags #{:integration}}
;;                  {:kaocha.testable/id :property
;;                   :kaocha/test-paths ["test/property"]
;;                   :kaocha.filter/tags #{:property}}]
;;  :kaocha/plugins [:kaocha.plugin/profiling
;;                    :kaocha.plugin/cloverage]
;;  :kaocha.plugin.cloverage/opts {:codecov? true
;;                                   :html?    true}}

;; deps.edn
;; {:aliases {:test {:extra-deps {lambdaisland/kaocha {:mvn/version "1.87.1380"}
;;                                 cloverage/cloverage  {:mvn/version "1.2.4"}}
;;                    :main-opts  ["-m" "kaocha.runner"]}}}

;; Run: clojure -M:test unit
;; Run all: clojure -M:test
;; Watch mode: clojure -M:test --watch

;; Tag tests
(deftest ^:unit test-calculation
  (is (= 42 (calculate 6 7))))

(deftest ^:integration test-database-connection
  (is (db/connected? ds)))

;; Focus tests
(deftest ^:focus test-important-feature
  (testing "specific behavior"
    (is (= :expected (complex-operation)))))
```

---

## ขั้นตอนที่ 966: BDD with Speclj

```clojure
;; deps.edn
;; {:deps {speclj/speclj {:mvn/version "3.4.3"}}}

(ns myapp.cart-spec
  (:require [speclj.core :refer :all]
            [myapp.cart :as cart]))

(describe "Shopping Cart"
  
  (with cart-store (atom {}))
  
  (describe "adding items"
    
    (it "adds new item with quantity 1 by default"
      (cart/add-item! @cart-store 1)
      (should= 1 (cart/item-count @cart-store 1)))
    
    (it "increments quantity when adding existing item"
      (cart/add-item! @cart-store 1 2)
      (cart/add-item! @cart-store 1 3)
      (should= 5 (cart/item-count @cart-store 1)))
    
    (it "allows adding multiple different items"
      (cart/add-item! @cart-store 1)
      (cart/add-item! @cart-store 2)
      (should= 2 (count (cart/all-items @cart-store)))))
  
  (describe "removing items"
    
    (before (cart/add-item! @cart-store 1 5))
    
    (it "removes item completely"
      (cart/remove-item! @cart-store 1)
      (should-be-nil (cart/item-count @cart-store 1)))
    
    (it "does nothing for non-existent item"
      (cart/remove-item! @cart-store 999)
      (should= 5 (cart/item-count @cart-store 1))))
  
  (describe "calculating total"
    
    (context "with products at various prices"
      
      (with products {1 {:price 100.0 :name "Widget"}
                        2 {:price 200.0 :name "Gadget"}})
      
      (before
        (cart/add-item! @cart-store 1 2)
        (cart/add-item! @cart-store 2 1))
      
      (it "calculates correct total"
        (should= 400.0 (cart/total @cart-store @products))))))
```

---

## Project: Full Test Suite

```clojure
(ns myapp.test-suite
  (:require [clojure.test :refer :all]
            [clojure.test.check.clojure-test :refer [defspec]]
            [clojure.test.check.properties :as prop]
            [clojure.test.check.generators :as gen]))

;; Test pyramid:
;; 70% unit, 20% integration, 10% E2E

;; ===== Unit Tests =====

(deftest test-user-validation
  (testing "valid user"
    (is (valid? {:name "สมชาย" :email "a@b.com" :age 25})))
  
  (testing "invalid email"
    (is (not (valid? {:name "สมชาย" :email "invalid" :age 25}))))
  
  (testing "negative age"
    (is (not (valid? {:name "สมชาย" :email "a@b.com" :age -1})))))

;; ===== Property Tests =====

(defspec user-serialization-roundtrip 100
  (prop/for-all [user user-gen]
    (= user (deserialize (serialize user)))))

;; ===== Integration Tests =====

(deftest ^:integration test-user-crud
  (with-db-fixture
    (let [id   (:id (create-user! {:name "Test" :email "test@test.com"}))
          user (find-user id)]
      (is (= "Test" (:name user)))
      (update-user! id {:name "Updated"})
      (is (= "Updated" (:name (find-user id))))
      (delete-user! id)
      (is (nil? (find-user id))))))

;; ===== API Tests =====

(deftest ^:integration test-api-endpoints
  (let [token (create-test-token {:id 1 :role :admin})]
    (testing "GET /api/users"
      (let [response (test-request :get "/api/users"
                                    :headers {"Authorization" (str "Bearer " token)})]
        (is (= 200 (:status response)))
        (is (vector? (get-in response [:body :items])))))
    
    (testing "POST /api/users with invalid data"
      (let [response (test-request :post "/api/users"
                                    :headers {"Authorization" (str "Bearer " token)}
                                    :body {:name "" :email "invalid"})]
        (is (= 422 (:status response)))
        (is (contains? (:body response) :errors))))))
```

---

*Part 33 จาก 100+ | ขั้นตอน 961-990 จาก 1000+*
