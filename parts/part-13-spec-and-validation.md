# Part 13: Spec และ Validation ด้วย Clojure.Spec & Malli
## ขั้นตอนที่ 361-390: Schema Validation, Generative Testing, Instrumentation

---

## บทนำ

Clojure ให้ทางเลือก 2 ตัวหลักสำหรับ validation/schema:
- **clojure.spec** - built-in, powerful, generative testing
- **Malli** - data-driven schemas, fast, flexible

ทั้งสองใช้ **data as specification** ซึ่งเป็น Clojure philosophy

---

## ขั้นตอนที่ 361: clojure.spec พื้นฐาน

```clojure
(require '[clojure.spec.alpha :as s])

;; ===== Define specs =====

;; Predicate spec
(s/def ::name string?)
(s/def ::age  (s/and int? #(>= % 0) #(<= % 150)))
(s/def ::email (s/and string? #(re-matches #".+@.+\..+" %)))
(s/def ::role #{:admin :user :moderator})

;; Map spec
(s/def ::user
  (s/keys :req-un [::name ::email]
          :opt-un [::age ::role]))

;; ===== Validate =====
(s/valid? ::name "สมชาย")    ; => true
(s/valid? ::name 42)          ; => false
(s/valid? ::age 25)           ; => true
(s/valid? ::age -1)           ; => false
(s/valid? ::age 200)          ; => false

;; ===== Explain errors =====
(s/explain ::user {:email "not-valid"})
;; >> val: {:email "not-valid"} fails spec: :myapp/user
;;    at: [:name] predicate: string?
;;    at: [:email] predicate: (re-matches #".+@.+\..+" %)

;; explain-data returns data (not just prints)
(s/explain-data ::user {:email "not-valid"})
```

---

## ขั้นตอนที่ 362: Spec Combinators

```clojure
;; s/and - all predicates must pass
(s/def ::positive-int (s/and int? pos?))

;; s/or - one of predicates
(s/def ::string-or-int
  (s/or :string string?
        :number int?))

(s/valid? ::string-or-int "hello")  ; => true
(s/valid? ::string-or-int 42)       ; => true
(s/valid? ::string-or-int nil)      ; => false

;; s/nilable - value or nil
(s/def ::optional-email (s/nilable ::email))

;; s/coll-of - collection with spec for elements
(s/def ::tags (s/coll-of string? :min-count 1 :max-count 10))

;; s/map-of - map with specs for keys and values
(s/def ::scores (s/map-of string? number?))

;; s/tuple - fixed-length collection with positional specs
(s/def ::point (s/tuple double? double?))
(s/def ::rgb   (s/tuple #(<= 0 % 255) #(<= 0 % 255) #(<= 0 % 255)))

;; s/merge - combine multiple map specs
(s/def ::address (s/keys :req-un [::street ::city ::zip]))
(s/def ::person-with-address (s/merge ::user ::address))
```

---

## ขั้นตอนที่ 363: Function Specs

```clojure
;; fdef - spec สำหรับ function arguments และ return value

(defn divide [a b]
  (/ a b))

(s/fdef divide
  :args (s/cat :a number? :b (s/and number? (complement zero?)))
  :ret  number?
  :fn   #(= (:ret %) (/ (-> % :args :a) (-> % :args :b))))

;; :args = positional arguments spec (s/cat creates named tuple)
;; :ret  = return value spec
;; :fn   = relationship between args and ret

;; More examples
(s/fdef clojure.core/inc
  :args (s/cat :n number?)
  :ret  number?
  :fn   #(= (:ret %) (inc (-> % :args :n))))

(defn create-user [name email]
  {:name name :email email :id (java.util.UUID/randomUUID)})

(s/fdef create-user
  :args (s/cat :name ::name :email ::email)
  :ret  ::user)

;; Instrument functions (ตรวจ args ตอน call จริง)
(require '[clojure.spec.test.alpha :as stest])

(stest/instrument `divide)
;; Now calling (divide 10 0) throws spec error!

;; Check return values too
(stest/check `divide)
;; Generative testing! สร้าง test cases อัตโนมัติ
```

---

## ขั้นตอนที่ 364: Generative Testing ด้วย Spec

```clojure
(require '[clojure.spec.gen.alpha :as gen])
(require '[clojure.test.check.generators :as tgen])

;; Generate data ที่ conform กับ spec!
(gen/sample (s/gen ::name))
;; => ("a" "bc" "def" "hello" ...)

(gen/sample (s/gen ::age))
;; => (0 25 42 100 ...)

(gen/sample (s/gen ::user))
;; => ({:name "abc" :email "a@b.com"} ...)

;; Custom generators
(s/def ::email-address
  (s/with-gen
    (s/and string? #(re-matches #".+@.+\..+" %))
    #(tgen/fmap
       (fn [[user domain]]
         (str user "@" domain ".com"))
       (tgen/tuple
         (tgen/not-empty tgen/string-alphanumeric)
         (tgen/not-empty tgen/string-alphanumeric)))))

;; Property test ด้วย spec
(stest/check `my-function {:clojure.spec.test.check/opts {:num-tests 1000}})
```

---

## ขั้นตอนที่ 365: Malli - Data-driven Schemas

```clojure
;; deps.edn
;; {:deps {metosin/malli {:mvn/version "0.14.0"}}}

(require '[malli.core :as m]
         '[malli.error :as me]
         '[malli.generator :as mg])

;; ===== Schema definitions (เป็น data!) =====

(def UserSchema
  [:map
   [:id     :uuid]
   [:name   [:string {:min 1 :max 100}]]
   [:email  [:re #".+@.+\..+"]]
   [:age    {:optional true} [:int {:min 0 :max 150}]]
   [:role   [:enum :admin :user :moderator]]
   [:active :boolean]])

(def CreateUserRequest
  [:map
   [:name  [:string {:min 1 :max 100}]]
   [:email [:re #".+@.+\..+"]]
   [:password [:string {:min 8}]]])

;; Validate
(m/validate UserSchema {:id (java.util.UUID/randomUUID)
                         :name "สมชาย"
                         :email "test@test.com"
                         :role :user
                         :active true})
;; => true

;; Explain errors
(m/explain UserSchema {:name ""})
;; returns detailed error data

;; Human-readable errors
(-> UserSchema
    (m/explain {:name ""})
    (me/humanize))
;; => {:name ["should be at least 1 characters"]}
```

---

## ขั้นตอนที่ 366: Malli Advanced

```clojure
;; ===== Malli Schema Types =====

;; Primitive types
[:string {:min 1 :max 100}]
[:int {:min 0 :max 100}]
[:double]
[:boolean]
[:keyword]
[:uuid]
[:inst]    ; java.util.Date

;; Collection types
[:vector :string]             ; vector of strings
[:set :keyword]               ; set of keywords
[:sequential :any]            ; any sequential
[:map-of :keyword :string]    ; map from keyword to string

;; Optional and nullable
[:maybe :string]              ; nil or string
[:or :string :int]            ; string or int
[:and :string [:min-count 1]] ; string with min length

;; Nested schemas
[:map
 [:address [:map
            [:street :string]
            [:city :string]
            [:zip [:re #"\d{5}"]]]]]

;; Enum
[:enum :pending :confirmed :shipped :delivered]

;; Function schema
[:=> [:cat :int :int] :int]  ; (int, int) -> int

;; ===== Coercion (transform + validate) =====
(require '[malli.transform :as mt])

(def decode-user
  (m/decoder UserSchema mt/json-transformer))

;; Transform JSON (all strings) → proper types
(decode-user {"id" "a0be1234-..."
              "name" "สมชาย"
              "active" "true"})
;; => {:id #uuid "a0be1234-...", :name "สมชาย", :active true}
```

---

## ขั้นตอนที่ 367: Malli with HTTP

```clojure
(ns myapp.api.users
  (:require [malli.core :as m]
            [malli.error :as me]
            [malli.transform :as mt]))

;; Request/Response schemas
(def CreateUserRequest
  [:map
   [:name     [:string {:min 1 :max 100}]]
   [:email    [:re {:error/message "Invalid email"} #".+@.+\..+"]]
   [:password [:string {:min 8 :error/message "Password must be at least 8 characters"}]]
   [:age      {:optional true} [:int {:min 0 :max 150}]]])

(def UserResponse
  [:map
   [:id      :uuid]
   [:name    :string]
   [:email   :string]
   [:role    [:enum :admin :user]]
   [:active  :boolean]
   [:created-at :inst]])

;; Validation helper
(defn validate-request [schema data]
  (if (m/validate schema data)
    {:ok data}
    {:err (me/humanize (m/explain schema data))}))

;; Handler with validation
(defn create-user-handler [{:keys [body-params]}]
  (let [{:keys [ok err]} (validate-request CreateUserRequest body-params)]
    (if err
      {:status 422
       :body {:error "Validation failed"
               :details err}}
      {:status 201
       :body (db/create-user! ok)})))

;; Middleware: auto-validate with schema
(defn wrap-validate [handler schema]
  (fn [request]
    (let [{:keys [ok err]} (validate-request schema (:body-params request))]
      (if err
        {:status 422 :body {:errors err}}
        (handler (assoc request :validated-body ok))))))
```

---

## ขั้นตอนที่ 368: Schema Registry

```clojure
(require '[malli.registry :as mr])

;; Register schemas globally
(mr/set-default-registry!
  (merge (m/default-schemas)
         (mt/js-undefined-transformer)
         {:user/id       :uuid
          :user/name     [:string {:min 1 :max 100}]
          :user/email    [:re #".+@.+\..+"]
          :user/role     [:enum :admin :user :moderator]
          :user/active   :boolean
          :user/profile  [:map
                          [:id    :user/id]
                          [:name  :user/name]
                          [:email :user/email]]
          :product/id    :uuid
          :product/price [:double {:min 0}]
          :product/stock [:int {:min 0}]}))

;; Use registered schemas
(m/validate :user/profile
  {:id (java.util.UUID/randomUUID)
   :name "สมชาย"
   :email "test@test.com"})
```

---

## ขั้นตอนที่ 369: Spec vs Malli

```
Feature Comparison:
===================

                    clojure.spec    Malli
Schemas as data         ✗            ✅
Runtime performance     OK           Fast
Generative testing      ✅           ✅
Error messages          OK           Excellent
JSON Schema export      ✗            ✅
Coercion                ✗            ✅
Open specs (extend)     ✅           ✅
Built-in                ✅           ✗

When to use:
============
clojure.spec:
  ✓ Heavy use of generative testing
  ✓ Function instrumentation/checking
  ✓ Core Clojure libraries integration
  ✓ Don't want external deps

Malli:
  ✓ HTTP API validation
  ✓ Need great error messages  
  ✓ JSON Schema compatibility
  ✓ Data transformation/coercion
  ✓ Registry of named schemas
  ✓ Modern Clojure projects
```

---

## ขั้นตอนที่ 370: Validation สำหรับ Production

```clojure
(ns myapp.validation
  (:require [malli.core :as m]
            [malli.error :as me]
            [malli.transform :as mt]))

;; Schemas
(def schemas
  {:user/create [:map
                 [:name     [:string {:min 1 :max 100}]]
                 [:email    [:re #".+@.+\..+"]]
                 [:password [:string {:min 8 :max 100}]]]
   
   :user/update [:map {:closed false}
                 [:name  {:optional true} [:string {:min 1 :max 100}]]
                 [:email {:optional true} [:re #".+@.+\..+"]]]
   
   :order/create [:map
                  [:user-id  :uuid]
                  [:items    [:vector [:map
                                       [:product-id :uuid]
                                       [:qty        [:int {:min 1}]]]]]
                  [:address  [:map
                               [:street [:string {:min 5}]]
                               [:city   [:string {:min 2}]]
                               [:zip    [:re #"\d{5}"]]]]]})

;; Validation result
(defn validate [schema-key data]
  (let [schema (get schemas schema-key)]
    (if (m/validate schema data)
      {:valid true :data data}
      {:valid false
       :errors (me/humanize (m/explain schema data))})))

;; Throw on invalid
(defn validate! [schema-key data]
  (let [{:keys [valid errors]} (validate schema-key data)]
    (when-not valid
      (throw (ex-info "Validation failed"
                      {:status 422
                       :errors errors})))
    data))

;; Usage
(validate! :user/create {:name "สมชาย" :email "test@test.com" :password "secure123"})
```

---

## Project: API Request Validation Middleware

```clojure
(ns myapp.middleware.validation
  (:require [malli.core :as m]
            [malli.error :as me]
            [malli.transform :as mt]))

;; Schema registry
(def registry (atom {}))

(defn register-schema! [k schema]
  (swap! registry assoc k schema))

;; Request validator
(defn validate-body [schema-key body]
  (when-let [schema (get @registry schema-key)]
    (let [decoder (m/decoder schema mt/json-transformer)]
      (try
        (let [decoded (decoder body)]
          (if (m/validate schema decoded)
            {:ok decoded}
            {:err (me/humanize (m/explain schema decoded))}))
        (catch Exception e
          {:err {:general ["Invalid request body"]}})))))

;; Reitit middleware
(defn validation-middleware [schema-key]
  {:name ::body-validation
   :wrap (fn [handler]
           (fn [request]
             (let [body (:body-params request)
                   {:keys [ok err]} (validate-body schema-key body)]
               (if err
                 {:status 422
                  :body {:errors err
                          :message "Validation failed"}}
                 (handler (assoc request :valid-body ok))))))})

;; Register schemas at startup
(register-schema! :user/create
  [:map
   [:name     [:string {:min 1 :max 100}]]
   [:email    [:re #".+@.+\..+"]]
   [:password [:string {:min 8}]]])

;; Use in routes
(def routes
  [["/api/users"
    ["" {:post {:middleware [(validation-middleware :user/create)]
                :handler    create-user-handler}}]]])
```

---

*Part 13 จาก 100+ | ขั้นตอน 361-390 จาก 1000+*
