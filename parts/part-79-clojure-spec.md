# Part 79: clojure.spec และ Malli Schema
## ขั้นตอนที่ 2341-2370: Runtime Validation, Generative Testing, Instrumentation

---

## บทนำ

Data validation ระดับ production:
- **clojure.spec** - built-in spec system
- **Malli** - modern schema library
- **Instrumentation** - validate at function boundaries
- **Generative testing** - auto-generate test cases
- **Coercion** - parse and transform data

---

## ขั้นตอนที่ 2341: clojure.spec Basics

```clojure
(ns myapp.spec
  (:require [clojure.spec.alpha :as s]
            [clojure.spec.gen.alpha :as gen]))

;; Define specs
(s/def ::email
  (s/and string?
         #(re-matches #"^[^\s@]+@[^\s@]+\.[^\s@]+$" %)))

(s/def ::price
  (s/and (s/or :decimal decimal? :double double? :int int?)
         #(>= % 0)))

(s/def ::quantity
  (s/and int? #(> % 0)))

(s/def ::product-id uuid?)

;; Composite specs
(s/def ::product
  (s/keys :req-un [::product-id ::name ::price]
           :opt-un [::description ::category-id ::stock]))

(s/def ::order-item
  (s/keys :req-un [::product-id ::quantity]))

(s/def ::items
  (s/coll-of ::order-item :min-count 1))

(s/def ::order
  (s/keys :req-un [::customer-id ::items]))

;; Validate
(s/valid? ::email "user@example.com")   ; => true
(s/valid? ::email "not-an-email")       ; => false

;; Explain invalid data
(s/explain ::product {:name "Widget" :price -5})
;; Prints: val: -5 fails spec: :myapp.spec/price

;; Get explanation as data
(s/explain-data ::product {:name "Widget" :price -5})
```

---

## ขั้นตอนที่ 2342: Function Specs and Instrumentation

```clojure
;; Spec for functions: args, return value, relationship
(s/def ::create-order-args
  (s/cat :customer-id ::customer-id
          :items        ::items))

(s/def ::create-order-ret
  (s/keys :req-un [::order-id ::status ::created-at]))

(s/fdef create-order!
  :args (s/cat :db any? :customer-id ::customer-id :items ::items)
  :ret  ::create-order-ret
  :fn   (fn [{{:keys [customer-id items]} :args, ret :ret}]
           ;; Postcondition: returned order has correct customer
           (= customer-id (:customer-id ret))))

;; Instrument: check args at call time
(s/check-asserts true)  ; Enable in dev

;; In development: instrument all specs
(require '[clojure.spec.test.alpha :as stest])
(stest/instrument)  ; Instrument all speced functions

;; Or specific functions
(stest/instrument `create-order!)

;; Generate test cases from specs
(gen/sample (s/gen ::product) 5)
;; Generates 5 random valid products

;; Run generative tests on function
(stest/check `create-order!
  {:clojure.spec.test.check/opts {:num-tests 100}})
```

---

## ขั้นตอนที่ 2343: Malli - Modern Schema Library

```clojure
(ns myapp.schemas
  (:require [malli.core :as m]
            [malli.transform :as mt]
            [malli.error :as me]
            [malli.generator :as mg]))

;; Malli schemas (more expressive than spec)
(def Money
  [:map
   [:amount [:and :decimal [:>= 0M]]]
   [:currency [:enum :USD :EUR :GBP :THB]]])

(def Product
  [:map
   [:id    [:re #"[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}"]]
   [:name  [:string {:min 1 :max 255}]]
   [:price Money]
   [:stock [:int {:min 0}]]
   [:tags  {:optional true} [:set :keyword]]])

(def CreateProductRequest
  [:map
   [:name  [:string {:min 1 :max 255}]]
   [:price [:double {:min 0.01 :max 999999.99}]]
   [:stock {:optional true} [:int {:min 0}]]])

;; Validate
(m/validate Product
  {:id    "123e4567-e89b-12d3-a456-426614174000"
   :name  "Widget"
   :price {:amount 9.99M :currency :USD}
   :stock 100})

;; Get human-readable errors
(me/humanize (m/explain Product {:name "" :price {:amount -1M}}))
;; => {:name ["should be at least 1 characters"],
;;     :price {:amount ["should be at least 0"]}}

;; Coerce (parse and transform)
(def coerce-product
  (m/coercer CreateProductRequest mt/string-transformer))

(coerce-product {"name" "Widget" "price" "9.99" "stock" "100"})
;; => {:name "Widget" :price 9.99 :stock 100}

;; Generate test data
(mg/generate Product {:size 5})
```

---

## ขั้นตอนที่ 2344: Schema-Driven Validation Middleware

```clojure
;; Validate HTTP requests using Malli schemas
(defn validate-request-body [schema handler]
  (fn [request]
    (let [body (:body-params request)]
      (if-let [error (m/explain schema body)]
        {:status 400
         :body   {:error  "Validation failed"
                   :fields (me/humanize error)}}
        (handler request)))))

;; Route-level validation
(def routes
  [["/api/products"
    {:post {:parameters {:body CreateProductRequest}
            :responses  {201 {:body Product}}
            :handler    create-product-handler}}]])

;; Validation with coercion (parse strings from form data)
(defn coerce-and-validate [schema data]
  (let [coerced (m/decode schema data (mt/transformer
                                        mt/string-transformer
                                        mt/default-value-transformer))]
    (if (m/validate schema coerced)
      {:ok coerced}
      {:error (me/humanize (m/explain schema coerced))})))

;; Usage in handler
(defn create-product-handler [request]
  (let [result (coerce-and-validate
                 CreateProductRequest
                 (:body-params request))]
    (if (:error result)
      {:status 422 :body {:errors (:error result)}}
      (let [product (create-product! db (:ok result))]
        {:status 201 :body product}))))
```

---

## ขั้นตอนที่ 2345: Advanced Spec Patterns

```clojure
;; Multi-spec for polymorphic data
(defmulti event-type :type)

(defmethod event-type :order-created [_]
  (s/keys :req-un [::order-id ::customer-id ::items]))

(defmethod event-type :order-shipped [_]
  (s/keys :req-un [::order-id ::tracking-number ::carrier]))

(defmethod event-type :order-cancelled [_]
  (s/keys :req-un [::order-id ::reason]))

(s/def ::event
  (s/multi-spec event-type :type))

;; Conform: destructure complex data
(s/def ::op #{:add :remove :update})
(s/def ::patch-op (s/cat :op ::op :path string? :value any?))

(s/conform ::patch-op [:add "/name" "Widget"])
;; => {:op :add :path "/name" :value "Widget"}

;; Specify allowed keys precisely
(s/def ::strict-product
  (s/and
    (s/keys :req-un [::id ::name ::price])
    (fn [m] (= #{:id :name :price} (set (keys m))))))

;; Spec for dependent fields
(s/def ::promo-code
  (s/and
    (s/keys :req-un [::discount-type ::discount-value])
    (fn [{:keys [discount-type discount-value]}]
      (case discount-type
        :percent (and (number? discount-value) (<= 0 discount-value 100))
        :fixed   (and (number? discount-value) (>= discount-value 0))
        false))))
```

---

*Part 79 จาก 100+ | ขั้นตอน 2341-2370 จาก 1000+*
