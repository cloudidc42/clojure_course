# Part 11: Testing ใน Clojure
## ขั้นตอนที่ 301-330: Unit Tests, Integration Tests, Property-Based Testing

---

## บทนำ

Clojure ทำให้ testing ง่ายมากเพราะ:
1. **Pure functions** - ง่ายต่อการ test (input → output)
2. **Immutable data** - ไม่มี shared mutable state
3. **REPL** - test ได้ทันทีขณะพัฒนา
4. **clojure.test** - built-in testing framework

---

## ขั้นตอนที่ 301: clojure.test พื้นฐาน

```clojure
(ns myapp.core-test
  (:require [clojure.test :refer [deftest is are testing run-tests]]))

;; Simple test
(deftest test-addition
  (is (= 4 (+ 2 2))))

;; Multiple assertions
(deftest test-string-functions
  (is (= "HELLO" (clojure.string/upper-case "hello")))
  (is (= "hello" (clojure.string/lower-case "HELLO")))
  (is (clojure.string/starts-with? "hello" "hell"))
  (is (not (clojure.string/blank? "  hello  "))))

;; Group assertions
(deftest test-math
  (testing "basic arithmetic"
    (is (= 4 (+ 2 2)))
    (is (= 0 (- 5 5))))
  
  (testing "division"
    (is (= 2.0 (/ 4.0 2.0)))
    (is (thrown? ArithmeticException (/ 1 0)))))

;; are - multiple cases with same pattern
(deftest test-even
  (are [x] (even? x)
    2 4 6 8 100))

(deftest test-comparison
  (are [result a b] (= result (compare a b))
    -1 1 2
     0 2 2
     1 3 2))

;; Run tests
(run-tests 'myapp.core-test)
```

---

## ขั้นตอนที่ 302: Fixtures - Setup และ Teardown

```clojure
(ns myapp.db-test
  (:require [clojure.test :refer :all]
            [next.jdbc :as jdbc]
            [myapp.db :as db]))

;; Once fixture (เรียกครั้งเดียวตอนเริ่ม/จบ)
(defn setup-db []
  (def test-ds (db/create-test-pool))
  (db/run-migrations! test-ds))

(defn teardown-db []
  (db/drop-test-schema! test-ds)
  (.close test-ds))

(use-fixtures :once
  (fn [test-fn]
    (setup-db)
    (test-fn)
    (teardown-db)))

;; Each fixture (เรียกก่อน/หลังทุก test)
(use-fixtures :each
  (fn [test-fn]
    ;; Setup: clear data before each test
    (jdbc/execute! test-ds ["TRUNCATE TABLE users RESTART IDENTITY CASCADE"])
    (test-fn)
    ;; No teardown needed - will clear before next test
    ))

;; Test ที่ใช้ database
(deftest test-create-user
  (let [user (db/create-user! test-ds {:name "สมชาย" :email "test@test.com"})
        found (db/find-user-by-id test-ds (:id user))]
    (is (= "สมชาย" (:name found)))
    (is (= "test@test.com" (:email found)))))
```

---

## ขั้นตอนที่ 303: Mocking และ Stubbing

```clojure
(ns myapp.service-test
  (:require [clojure.test :refer :all]
            [myapp.service :as svc]
            [myapp.email :as email]))

;; Pattern 1: with-redefs (global redef for test duration)
(deftest test-send-welcome-email
  (let [sent-emails (atom [])]
    (with-redefs [email/send! (fn [to subject body]
                                (swap! sent-emails conj {:to to :subject subject}))]
      (svc/register-user! {:email "test@test.com" :name "สมชาย"})
      (is (= 1 (count @sent-emails)))
      (is (= "test@test.com" (-> @sent-emails first :to))))))

;; Pattern 2: Dependency injection (better!)
(defn make-user-service [user-repo email-service]
  {:register (fn [{:keys [email name]}]
               (let [user (user-repo :create {:email email :name name})]
                 (email-service :send-welcome {:to email :name name})
                 user))})

;; Test with fake implementations
(deftest test-user-service-register
  (let [created-users (atom [])
        sent-emails (atom [])
        
        fake-repo (fn [op data]
                    (case op
                      :create (let [user (assoc data :id 1)]
                                (swap! created-users conj user)
                                user)))
        
        fake-email (fn [op data]
                     (swap! sent-emails conj data))
        
        service (make-user-service fake-repo fake-email)]
    
    ((:register service) {:email "test@test.com" :name "สมชาย"})
    
    (is (= 1 (count @created-users)))
    (is (= 1 (count @sent-emails)))
    (is (= "test@test.com" (-> @sent-emails first :to)))))
```

---

## ขั้นตอนที่ 304: Testing HTTP Handlers

```clojure
(ns myapp.api-test
  (:require [clojure.test :refer :all]
            [ring.mock.request :as mock]
            [cheshire.core :as json]
            [myapp.app :refer [app]]))

;; ring-mock สร้าง request maps สำหรับ test

(defn parse-json-body [response]
  (some-> response :body (json/parse-string true)))

(deftest test-health-endpoint
  (let [response (app (mock/request :get "/api/health"))]
    (is (= 200 (:status response)))
    (is (= "ok" (:status (parse-json-body response))))))

(deftest test-get-users
  (let [response (app (-> (mock/request :get "/api/users")
                          (mock/header "Authorization" "Bearer valid-token")))]
    (is (= 200 (:status response)))
    (is (vector? (:users (parse-json-body response))))))

(deftest test-create-user
  (let [user-data {:name "สมชาย" :email "test@test.com"}
        response (app (-> (mock/request :post "/api/users")
                          (mock/content-type "application/json")
                          (mock/json-body user-data)))]
    (is (= 201 (:status response)))
    (let [created (parse-json-body response)]
      (is (= "สมชาย" (:name created)))
      (is (integer? (:id created))))))

(deftest test-unauthorized-access
  (let [response (app (mock/request :get "/api/protected"))]
    (is (= 401 (:status response)))))
```

---

## ขั้นตอนที่ 305: Property-Based Testing ด้วย test.check

```clojure
;; deps.edn
;; {:deps {org.clojure/test.check {:mvn/version "1.1.1"}}}

(ns myapp.prop-test
  (:require [clojure.test :refer [deftest is]]
            [clojure.test.check :as tc]
            [clojure.test.check.generators :as gen]
            [clojure.test.check.properties :as prop]
            [clojure.test.check.clojure-test :refer [defspec]]))

;; Generators สร้าง test data โดยอัตโนมัติ!
(gen/sample gen/int)        ; => (0 1 -1 2 -2 ...)
(gen/sample gen/string-ascii) ; => ("" "a" "bc" ...)
(gen/sample gen/boolean)    ; => (true false ...)

;; Property: sort ของ sort ควรเท่ากับ sort ครั้งเดียว
(defspec idempotent-sort
  100  ; จำนวน test cases
  (prop/for-all [xs (gen/vector gen/int)]
    (= (sort xs) (sort (sort xs)))))

;; Property: reverse ของ reverse ควรได้ original
(defspec reverse-twice-is-identity
  100
  (prop/for-all [xs (gen/list gen/int)]
    (= xs (reverse (reverse xs)))))

;; Property: ใส่ map แล้วดึงออกได้ค่าเดิม
(defspec map-get-round-trip
  100
  (prop/for-all [k gen/keyword
                 v gen/int]
    (= v (get (assoc {} k v) k))))
```

---

## ขั้นตอนที่ 306: Custom Generators

```clojure
;; สร้าง custom generators สำหรับ domain types

;; Generator for email addresses
(def email-gen
  (gen/fmap (fn [[name domain ext]]
              (str name "@" domain "." ext))
            (gen/tuple
              (gen/not-empty gen/string-alphanumeric)
              (gen/not-empty gen/string-alphanumeric)
              (gen/elements ["com" "org" "net" "th"]))))

;; Generator for user
(def user-gen
  (gen/hash-map
    :name    (gen/not-empty gen/string-ascii)
    :email   email-gen
    :age     (gen/choose 18 100)
    :active  gen/boolean))

;; Generator for money (positive decimal)
(def money-gen
  (gen/fmap #(/ % 100.0)
            (gen/choose 1 100000)))

;; Property test with custom generators
(defspec user-email-is-valid
  200
  (prop/for-all [user user-gen]
    (clojure.string/includes? (:email user) "@")))

;; Stateful property test
(defspec counter-always-increments
  100
  (prop/for-all [increments (gen/vector (gen/choose 1 100) 1 20)]
    (let [counter (atom 0)]
      (doseq [n increments]
        (swap! counter + n))
      (= @counter (reduce + increments)))))
```

---

## ขั้นตอนที่ 307: Integration Tests

```clojure
(ns myapp.integration-test
  (:require [clojure.test :refer :all]
            [ring.mock.request :as mock]
            [myapp.system :as system]))

;; Start real system for integration tests
(def ^:dynamic *system* nil)

(defn start-test-system []
  (system/start-system {:db-url "jdbc:postgresql://localhost/myapp_test"
                         :port 0}))  ; port 0 = random available port

(use-fixtures :once
  (fn [test-fn]
    (binding [*system* (start-test-system)]
      (try
        (test-fn)
        (finally
          (system/stop-system *system*))))))

(use-fixtures :each
  (fn [test-fn]
    ;; Clear test data
    (system/clear-test-data! *system*)
    (test-fn)))

;; Integration test
(deftest test-user-registration-flow
  (let [app (:app *system*)
        
        ;; Register
        reg-response (app (-> (mock/request :post "/api/auth/register")
                              (mock/json-body {:name "สมชาย"
                                               :email "test@test.com"
                                               :password "password123"})))
        _ (is (= 201 (:status reg-response)))
        
        ;; Login
        login-response (app (-> (mock/request :post "/api/auth/login")
                                (mock/json-body {:email "test@test.com"
                                                 :password "password123"})))
        _ (is (= 200 (:status login-response)))
        token (-> login-response :body json/parse-string (get "token"))
        
        ;; Access protected resource
        profile-response (app (-> (mock/request :get "/api/me")
                                  (mock/header "Authorization" (str "Bearer " token))))]
    
    (is (= 200 (:status profile-response)))
    (is (= "สมชาย" (-> profile-response :body json/parse-string (get "name"))))))
```

---

## ขั้นตอนที่ 308: Test Utilities

```clojure
(ns myapp.test-utils
  (:require [next.jdbc :as jdbc]
            [next.jdbc.sql :as sql]
            [myapp.auth :as auth]))

;; Factory functions สร้าง test data

(defn make-user [& {:keys [name email role active]
                     :or {name "Test User"
                          email (str "test+" (rand-int 10000) "@test.com")
                          role "user"
                          active true}}]
  {:name name :email email :role role :active active
   :password-hash (auth/hash-password "password123")})

(defn create-user! [ds & opts]
  (sql/insert! ds :users (apply make-user opts)
    {:return-keys true}))

(defn create-admin! [ds & opts]
  (apply create-user! ds :role "admin" opts))

(defn make-token [user]
  (auth/create-token (:id user) [(:role user)]))

;; DB test helpers
(defn truncate-all! [ds]
  (jdbc/execute! ds
    ["TRUNCATE TABLE users, orders, products RESTART IDENTITY CASCADE"]))

;; Response helpers
(defn ok? [response]
  (< (:status response) 400))

(defn body-json [response]
  (some-> response :body (cheshire.core/parse-string true)))

;; Assertion helpers
(defn assert-status [expected-status response]
  (when (not= expected-status (:status response))
    (throw (ex-info "Unexpected status"
                    {:expected expected-status
                     :actual (:status response)
                     :body (:body response)})))
  response)
```

---

## ขั้นตอนที่ 309: Test Organization

```
src/
  myapp/
    core.clj
    service/
      user_service.clj
    api/
      users.clj
    db/
      users.clj

test/
  myapp/
    core_test.clj          ; Unit tests
    service/
      user_service_test.clj
    api/
      users_test.clj        ; Handler tests
    db/
      users_test.clj        ; Repository tests
    integration/
      user_flow_test.clj    ; Integration tests
```

```clojure
;; project.clj test configuration
;; :test-paths ["test"]
;; :profiles {:test {:dependencies [[ring/ring-mock "0.4.0"]]}}

;; Run specific namespace
;; lein test myapp.service.user-service-test

;; Run all tests
;; lein test

;; Run with test selectors
;; lein test :unit
;; lein test :integration
;; lein test :all

;; Tag tests
(deftest ^:unit test-pure-function ...)
(deftest ^:integration test-with-db ...)
(deftest ^:slow test-performance ...)
```

---

## Project Exercise: TDD สร้าง Calculator Service

```clojure
;; Test-first approach!

(ns calculator.core-test
  (:require [clojure.test :refer :all]
            [calculator.core :as calc]))

;; ===== Write tests FIRST =====

(deftest test-basic-operations
  (testing "addition"
    (is (= 5  (calc/calculate "2 + 3")))
    (is (= 0  (calc/calculate "0 + 0")))
    (is (= -1 (calc/calculate "-3 + 2"))))
  
  (testing "subtraction"
    (is (= 1 (calc/calculate "3 - 2")))
    (is (= -5 (calc/calculate "0 - 5"))))
  
  (testing "multiplication"
    (is (= 6 (calc/calculate "2 * 3")))
    (is (= 0 (calc/calculate "5 * 0"))))
  
  (testing "division"
    (is (= 2.0 (calc/calculate "6 / 3")))
    (is (thrown? ArithmeticException (calc/calculate "1 / 0")))))

(deftest test-expressions
  (is (= 7  (calc/calculate "1 + 2 * 3")))   ; precedence
  (is (= 9  (calc/calculate "(1 + 2) * 3")))  ; parentheses
  (is (= 14 (calc/calculate "2 + 3 * 4 - 2 * 1"))))

(deftest test-history
  (let [calc (calc/create-calculator)]
    (calc/evaluate! calc "2 + 3")
    (calc/evaluate! calc "10 - 5")
    (is (= 2 (count (calc/history calc))))
    (is (= 5 (-> calc calc/history last :result)))))

;; ===== Now implement to pass tests =====
(ns calculator.core)

(defn calculate [expr]
  ;; Implementation...
  (let [parts (clojure.string/split expr #"\s+")
        [a op b] parts
        a (parse-double a)
        b (parse-double b)]
    (case op
      "+" (+ a b)
      "-" (- a b)
      "*" (* a b)
      "/" (if (zero? b)
            (throw (ArithmeticException. "Division by zero"))
            (/ a b)))))

(defn create-calculator []
  (atom {:history []}))

(defn evaluate! [calc expr]
  (let [result (calculate expr)]
    (swap! calc update :history conj {:expr expr :result result})
    result))

(defn history [calc]
  (:history @calc))
```

---

### สรุป Testing

```
Testing Pyramid:
===============
Unit Tests (fast, many)
  - Pure functions
  - Business logic
  - No I/O
  
Integration Tests (medium)
  - Database queries
  - External services (with stubs)
  
E2E Tests (slow, few)
  - Full HTTP flow
  - Real database
  - Acceptance tests

Tools:
======
clojure.test      - built-in testing
ring/ring-mock    - HTTP testing
test.check        - property-based testing
mockery/stub      - mock libraries
kaocha            - test runner with plugins
```

---

*Part 11 จาก 100+ | ขั้นตอน 301-330 จาก 1000+*
