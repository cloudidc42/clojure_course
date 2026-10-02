# Part 47: clojure.spec และ Generative Testing
## ขั้นตอนที่ 1381-1410: Spec, Instrumentation, fdef, Generators

---

## บทนำ

clojure.spec - specification และ validation:
- **Specs** - describe data shapes
- **Conforming** - validate and transform
- **Instrumentation** - function contracts
- **`fdef`** - function specs with generators
- **`stest/check`** - auto-generated property tests

---

## ขั้นตอนที่ 1381: Basic Specs

```clojure
(ns myapp.specs
  (:require [clojure.spec.alpha :as s]
            [clojure.spec.gen.alpha :as gen]))

;; Primitive specs
(s/def ::name      string?)
(s/def ::email     (s/and string? #(re-matches #".+@.+\..+" %)))
(s/def ::age       (s/and int? #(< 0 % 150)))
(s/def ::price     (s/and number? pos?))
(s/def ::quantity  (s/and int? pos?))

;; Enum spec
(s/def ::status    #{:active :inactive :banned})
(s/def ::role      #{:user :seller :admin})

;; Collection specs
(s/def ::tags      (s/coll-of string? :distinct true))
(s/def ::items     (s/coll-of ::order-item :min-count 1))

;; Map spec
(s/def ::user
  (s/keys :req [::name ::email ::role]
           :opt [::age ::phone]))

;; Strict map spec (no extra keys)
(s/def ::address
  (s/keys :req-un [:address/street :address/city :address/zip-code]
           :opt-un [:address/province :address/country]))

;; Nested spec
(s/def ::product
  (s/keys :req [::name ::price]
           :opt [::description ::category ::stock ::tags]))

;; Validate
(s/valid? ::email "test@example.com")  ; => true
(s/valid? ::email "not-an-email")      ; => false

;; Explain failures
(s/explain ::user {:name "Alice" :email "bad" :role :user})
;; In: [:email] val: "bad" fails spec: :myapp.specs/email

;; Conform: validate and transform
(s/conform ::status :active)    ; => :active
(s/conform ::status :unknown)   ; => :clojure.spec.alpha/invalid
```

---

## ขั้นตอนที่ 1382: Advanced Specs

```clojure
;; OR specs
(s/def ::contact
  (s/or :email   ::email
         :phone   ::phone-number
         :address ::address))

;; Usage with conformed value
(s/conform ::contact "user@example.com")
;; => [:email "user@example.com"]

;; AND specs for refinement
(s/def ::positive-even
  (s/and int? pos? even?))

;; Multi-spec (dispatch on type)
(defmulti event-type :type)
(defmethod event-type :user-created [_]
  (s/keys :req-un [:event/type :event/user-id :event/email]))
(defmethod event-type :order-placed [_]
  (s/keys :req-un [:event/type :event/order-id :event/user-id :event/total]))

(s/def ::event (s/multi-spec event-type :type))

;; Spec for function arguments
(s/def ::divide-args
  (s/and (s/cat :numerator   number?
                 :denominator number?)
          (fn [{:keys [denominator]}]
            (not (zero? denominator)))))

(s/conform ::divide-args [10 2])
;; => {:numerator 10 :denominator 2}

;; Merge specs
(s/def ::timestamped
  (s/keys :opt-un [:entity/created-at :entity/updated-at]))

(s/def ::audited-user
  (s/merge ::user ::timestamped))

;; Spec for sequences
(s/def ::coordinate-pair
  (s/cat :lat (s/and number? #(<= -90 % 90))
          :lng (s/and number? #(<= -180 % 180))))
```

---

## ขั้นตอนที่ 1383: Function Specs ด้วย fdef

```clojure
(ns myapp.functions
  (:require [clojure.spec.alpha :as s]
            [clojure.spec.test.alpha :as stest]))

;; Define function spec
(s/fdef divide
  :args (s/cat :numerator number? :denominator (s/and number? #(not (zero? %))))
  :ret  number?
  :fn   (fn [{:keys [args ret]}]
           (= ret (/ (:numerator args) (:denominator args)))))

(defn divide [numerator denominator]
  (/ numerator denominator))

;; Instrument: check args at runtime in dev
(stest/instrument `divide)

;; Now calling with bad args throws descriptive error
;; (divide 10 0) ; => ExceptionInfo: args ... failed spec

;; Complex function spec
(s/fdef create-order!
  :args (s/cat :db    any?
                :user-id ::user/id
                :items  ::items
                :address ::address)
  :ret  (s/keys :req-un [:order/id :order/status :order/total])
  :fn   (fn [{:keys [args ret]}]
           (= :pending (:status ret))))  ; New orders always pending

;; Run generative tests (check)
(stest/check `divide {:clojure.spec.test.check/opts {:num-tests 1000}})
```

---

## ขั้นตอนที่ 1384: Custom Generators

```clojure
(ns myapp.generators
  (:require [clojure.spec.alpha :as s]
            [clojure.spec.gen.alpha :as gen]
            [clojure.test.check.generators :as tgen]))

;; Override generator for email
(s/def ::email
  (s/with-gen
    (s/and string? #(re-matches #".+@.+\..+" %))
    (fn []
      (gen/let [name     (gen/such-that #(re-matches #"[a-z]+" %) tgen/string-alphanumeric)
                domain   (gen/elements ["gmail.com" "hotmail.com" "yahoo.com" "example.com"])]
        (str name "@" domain)))))

;; UUID generator
(s/def ::id
  (s/with-gen uuid?
    (fn [] tgen/uuid)))

;; Price generator (sensible range)
(s/def ::price
  (s/with-gen
    (s/and number? pos?)
    (fn []
      (gen/fmap #(/ % 100.0) (gen/choose 100 100000)))))

;; Product generator
(s/def ::product
  (s/with-gen
    (s/keys :req [::name ::price])
    (fn []
      (gen/let [name     (gen/elements ["Widget" "Gadget" "Doohickey" "Thingamajig"])
                suffix   (gen/choose 1000 9999)
                price    (gen/fmap #(/ % 100.0) (gen/choose 100 50000))
                category (gen/elements ["electronics" "clothing" "food" "books"])]
        {:name     (str name " " suffix)
         :price    price
         :category category}))))

;; Generate 10 sample products
(gen/sample (s/gen ::product) 10)
```

---

## ขั้นตอนที่ 1385: Spec-based Test Data Generation

```clojure
(ns myapp.test-data
  (:require [clojure.spec.alpha :as s]
            [clojure.spec.gen.alpha :as gen]))

;; Generate complete test scenarios
(defn generate-test-order []
  (gen/generate
    (gen/let [user-id  (s/gen ::user/id)
              items    (gen/vector (s/gen ::order-item) 1 10)
              address  (s/gen ::address)]
      {:user-id user-id
       :items   items
       :address address
       :total   (reduce + (map #(* (:price %) (:qty %)) items))})))

;; Generate entire database state
(defn generate-test-db [n-users n-products n-orders]
  (let [users    (gen/sample (s/gen ::user) n-users)
        products (gen/sample (s/gen ::product) n-products)
        orders   (gen/sample
                   (gen/let [user    (gen/elements users)
                              items   (gen/vector
                                        (gen/let [product (gen/elements products)
                                                   qty     (gen/choose 1 5)]
                                          {:product-id (:id product)
                                           :qty        qty
                                           :price      (:price product)})
                                        1 5)]
                     {:user-id (:id user) :items items})
                   n-orders)]
    {:users users :products products :orders orders}))

;; Property-based test using specs
(require '[clojure.test.check.properties :as prop])
(require '[clojure.test.check :as tc])

(def order-total-property
  (prop/for-all [order (s/gen ::order)]
    (= (:total order)
       (reduce + (map #(* (:price %) (:qty %)) (:items order))))))

(tc/quick-check 1000 order-total-property)
;; {:result true, :num-tests 1000, :seed ...}
```

---

## Project: Spec-driven API

```clojure
(ns api.spec-driven
  (:require [clojure.spec.alpha :as s]
            [clojure.spec.test.alpha :as stest]))

;; Define all API specs
(s/def :api/user-id  ::user/id)
(s/def :api/product-id ::product/id)

(s/def :api/create-user-request
  (s/keys :req-un [:api/name :api/email :api/password]))

(s/def :api/create-user-response
  (s/keys :req-un [:api/user-id :api/email :api/created-at]))

;; Spec-driven handler
(defn validated-handler [request-spec response-spec handler]
  (fn [request]
    (let [body (:body-params request)]
      (if-not (s/valid? request-spec body)
        {:status 422
         :body   {:error   "Invalid request"
                   :details (s/explain-data request-spec body)}}
        (let [response (handler request)]
          (when-not (s/valid? response-spec (:body response))
            (println "WARNING: Response doesn't match spec!"
                      (s/explain-data response-spec (:body response))))
          response)))))

;; Instrument all functions in production build
(defn instrument-all! []
  (stest/instrument))

;; In tests: check functions with generated data
(defn check-all-functions! []
  (stest/check (stest/enumerate-namespace 'myapp.core)))
```

---

*Part 47 จาก 100+ | ขั้นตอน 1381-1410 จาก 1000+*
