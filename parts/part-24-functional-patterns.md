# Part 24: Functional Design Patterns
## ขั้นตอนที่ 691-720: Monads, Functors, Lenses, Type Classes

---

## บทนำ

Functional Programming patterns จาก Category Theory:
- **Functor** - map over wrapped values
- **Monad** - chain operations with context (Maybe, Either, IO)
- **Lens** - composable getters/setters
- **Monoid** - combine values
- **Applicative** - apply wrapped functions to wrapped values

ใน Clojure เราใช้ patterns เหล่านี้แบบ pragmatic ไม่ต้องใช้ theoretical terminology

---

## ขั้นตอนที่ 691: Maybe/Option Pattern

```clojure
;; Maybe = ค่าที่อาจ nil ได้
;; ป้องกัน NullPointerException ด้วย composable nil handling

;; Clojure already has some-> and some->>!
(def user {:address {:street "123 Main St"
                      :city {:name "Bangkok"
                              :zip "10100"}}})

;; Without Maybe:
(when-let [address (:address user)]
  (when-let [city (:city address)]
    (:name city)))

;; With some-> (already built-in!):
(some-> user :address :city :name)
;; => "Bangkok"

;; some-> returns nil at first nil value:
(some-> nil :address :city)      ; => nil (no NPE!)
(some-> user :missing :city)     ; => nil

;; Custom maybe monad
(defn maybe [f]
  (fn [val]
    (when (some? val)
      (f val))))

(defn m-chain [& fns]
  (fn [val]
    (reduce (fn [v f]
              (when (some? v) (f v)))
            val
            fns)))

;; Chain nullable operations:
(def get-city
  (m-chain :address :city :name))

(get-city user)    ; => "Bangkok"
(get-city nil)     ; => nil
(get-city {:no-address true})  ; => nil
```

---

## ขั้นตอนที่ 692: Either/Result Pattern

```clojure
;; Either = ผลลัพธ์ที่อาจผิดพลาด
;; Left = error, Right = success

;; Implementation
(defprotocol Either
  (left? [this])
  (right? [this])
  (fmap [this f])
  (bind [this f])
  (value [this]))

(defrecord Left [v]
  Either
  (left? [_] true)
  (right? [_] false)
  (fmap [this _] this)
  (bind [this _] this)
  (value [_] v))

(defrecord Right [v]
  Either
  (left? [_] false)
  (right? [_] true)
  (fmap [_ f] (->Right (f v)))
  (bind [_ f] (f v))
  (value [_] v))

(defn left [v] (->Left v))
(defn right [v] (->Right v))

;; Chain operations
(defn divide [a b]
  (if (zero? b)
    (left "Division by zero")
    (right (/ a b))))

(defn sqrt [x]
  (if (neg? x)
    (left "Cannot sqrt negative")
    (right (Math/sqrt x))))

;; Chain with bind
(-> (right 16)
    (bind #(divide % 4))
    (bind sqrt)
    (fmap str))
;; => Right("2.0")

(-> (right -4)
    (bind #(divide % 2))
    (bind sqrt))
;; => Left("Cannot sqrt negative")
```

---

## ขั้นตอนที่ 693: Lens Pattern

```clojure
;; Lens = composable getter + setter สำหรับ nested data

;; Simple lens
(defrecord Lens [get set])

(defn lens [get-fn set-fn]
  (->Lens get-fn set-fn))

;; Basic operations
(defn view [lens data]
  ((:get lens) data))

(defn set-val [lens data val]
  ((:set lens) data val))

(defn over [lens data f]
  (set-val lens data (f (view lens data))))

;; Lens composition
(defn compose-lenses [outer inner]
  (lens
    (fn [data]
      (view inner (view outer data)))
    (fn [data val]
      (over outer data #(set-val inner % val)))))

;; Common lenses
(defn key-lens [k]
  (lens #(get % k)
        #(assoc %1 k %2)))

(defn nth-lens [n]
  (lens #(nth % n)
        #(assoc %1 n %2)))

;; Usage
(def user {:name "สมชาย"
            :address {:street "123 Main"
                      :city "Bangkok"}})

(def name-lens (key-lens :name))
(def address-lens (key-lens :address))
(def city-lens (key-lens :city))
(def addr-city-lens (compose-lenses address-lens city-lens))

(view name-lens user)              ; => "สมชาย"
(set-val name-lens user "สมหญิง") ; => {...:name "สมหญิง"...}
(view addr-city-lens user)         ; => "Bangkok"
(over addr-city-lens user clojure.string/upper-case)
;; => {:address {:city "BANGKOK" ...} ...}
```

---

## ขั้นตอนที่ 694: Monoid Pattern

```clojure
;; Monoid = combine-able values with identity element
;; Laws:
;;   (combine identity x) = x
;;   (combine x identity) = x
;;   (combine a (combine b c)) = (combine (combine a b) c)

(defprotocol Monoid
  (combine [this other])
  (identity-val [this]))

;; Sum monoid
(defrecord Sum [value]
  Monoid
  (combine [_ other] (->Sum (+ value (:value other))))
  (identity-val [_] (->Sum 0)))

;; Product monoid
(defrecord Product [value]
  Monoid
  (combine [_ other] (->Product (* value (:value other))))
  (identity-val [_] (->Product 1)))

;; String monoid
(defrecord StringMonoid [value]
  Monoid
  (combine [_ other] (->StringMonoid (str value (:value other))))
  (identity-val [_] (->StringMonoid "")))

;; Fold over collection using monoid
(defn mconcat [monoid-vals]
  (reduce combine monoid-vals))

;; More practical: Clojure's own monoids
;; Numbers: (reduce + 0 [1 2 3])      identity=0,  op=+
;; Strings: (reduce str "" ["a" "b"]) identity="", op=str
;; Lists:   (reduce into [] [[1] [2]]) identity=[], op=into
;; Maps:    (reduce merge {} [{:a 1}]) identity={}, op=merge
```

---

## ขั้นตอนที่ 695: Free Monad Pattern

```clojure
;; Free Monad: separate description from interpretation
;; Describe WHAT to do, not HOW to do it
;; Useful for: testable effects, multiple interpreters

;; DSL for database operations
(defprotocol DBOp
  (interpret [this db]))

(defrecord FindUser [id]
  DBOp
  (interpret [_ db]
    (db/find-by-id db :users id)))

(defrecord CreateUser [user]
  DBOp
  (interpret [_ db]
    (db/insert! db :users user)))

(defrecord UpdateUser [id updates]
  DBOp
  (interpret [_ db]
    (db/update! db :users updates {:id id})))

;; Program as sequence of operations
(defn register-user-program [name email]
  [(->FindUser [:email email])
   (fn [existing]
     (if existing
       {:error "Email already exists"}
       (->CreateUser {:name name :email email})))])

;; Real interpreter
(defn run-program [program db]
  (reduce (fn [result op]
            (if (fn? op)
              (let [next-op (op result)]
                (when next-op (interpret next-op db)))
              (interpret op db)))
          nil
          program))

;; Test interpreter (no database!)
(defn test-interpreter [program]
  (let [test-db (atom {})]
    (reduce (fn [result op]
              (if (fn? op)
                (op result)
                {:description (str "Execute: " (type op))}))
            nil
            program)))
```

---

## ขั้นตอนที่ 696: Clojure Protocols as Type Classes

```clojure
;; Protocols เหมือน type classes ใน Haskell
;; กำหนด behavior ที่ type ต้องมี

;; "Showable" type class
(defprotocol Showable
  (show [x]))

;; "Comparable" type class
(defprotocol Orderable
  (compare-to [this other]))

;; "Serializable" type class
(defprotocol Serializable
  (serialize [x format])
  (deserialize [data format expected-type]))

;; Implement for custom types
(defrecord Temperature [value unit]
  Showable
  (show [_]
    (str value "°" (name unit)))
  
  Orderable
  (compare-to [this other]
    (let [to-celsius #(case (:unit %)
                        :C (:value %)
                        :F (/ (* (- (:value %) 32) 5) 9)
                        :K (- (:value %) 273.15))]
      (compare (to-celsius this) (to-celsius other))))
  
  Serializable
  (serialize [_ :json]
    {:value value :unit (name unit)}))

;; Extend existing types (without modifying source!)
(extend-protocol Showable
  Long   (show [n] (str n))
  Double (show [d] (format "%.2f" d))
  String (show [s] (str "\"" s "\"")))

;; Functions using type class
(defn print-all [showables]
  (doseq [x showables]
    (println (show x))))
```

---

## ขั้นตอนที่ 697: Continuation-Passing Style (CPS)

```clojure
;; CPS: ส่ง continuation (callback) แทนที่จะ return
;; ช่วยแก้ stack overflow สำหรับ deep recursion

;; Normal style
(defn factorial [n]
  (if (zero? n) 1 (* n (factorial (dec n)))))

;; CPS style
(defn factorial-cps [n k]
  (if (zero? n)
    (k 1)
    (factorial-cps (dec n) (fn [result]
                              (k (* n result))))))

;; Call it with identity continuation
(factorial-cps 5 identity)  ; => 120

;; Trampoline + CPS สำหรับ deep recursion
(defn factorial-trampoline [n k]
  (if (zero? n)
    #(k 1)
    #(factorial-trampoline (dec n) (fn [result]
                                     (fn [] (k (* n result)))))))

(trampoline (factorial-trampoline 100000 identity))

;; Async CPS (callback hell, but explicit)
(defn fetch-user [id callback]
  (future (callback (db/find-user id))))

(defn get-user-orders [user callback]
  (future (callback (db/find-orders (:id user)))))

(defn calculate-total [orders callback]
  (callback (reduce + (map :total orders))))

;; Chain
(fetch-user 1
  (fn [user]
    (get-user-orders user
      (fn [orders]
        (calculate-total orders
          (fn [total]
            (println "Total:" total)))))))
```

---

## ขั้นตอนที่ 698: Algebraic Data Types (ADT)

```clojure
;; Clojure เลียนแบบ Haskell's ADT ด้วย tagged maps + multimethods

;; Shape ADT
(defn circle [radius]
  {:type :circle :radius radius})

(defn rectangle [width height]
  {:type :rectangle :width width :height height})

(defn triangle [base height]
  {:type :triangle :base base :height height})

;; Pattern matching ด้วย multimethod
(defmulti area :type)

(defmethod area :circle [{:keys [radius]}]
  (* Math/PI radius radius))

(defmethod area :rectangle [{:keys [width height]}]
  (* width height))

(defmethod area :triangle [{:keys [base height]}]
  (/ (* base height) 2))

;; Or more elegant: ใช้ defrecord + protocol
(defprotocol Shape
  (area [this])
  (perimeter [this]))

(defrecord Circle [radius]
  Shape
  (area [_] (* Math/PI radius radius))
  (perimeter [_] (* 2 Math/PI radius)))

(defrecord Rectangle [width height]
  Shape
  (area [_] (* width height))
  (perimeter [_] (* 2 (+ width height))))

;; ใช้งาน
(let [shapes [(->Circle 5) (->Rectangle 4 6)]]
  (map area shapes))
;; => (78.54... 24)
```

---

## Project: Pipeline สำหรับ Data Validation

```clojure
;; Composable validation pipeline ด้วย Either monad

(defn validate [value & validators]
  (reduce (fn [result validator]
            (if (left? result)
              result   ; short-circuit on first error
              (validator (value-of result))))
          (right value)
          validators))

;; Validators
(defn non-empty [s]
  (if (seq s) (right s) (left "Cannot be empty")))

(defn min-length [n]
  (fn [s]
    (if (>= (count s) n)
      (right s)
      (left (str "Must be at least " n " characters")))))

(defn max-length [n]
  (fn [s]
    (if (<= (count s) n)
      (right s)
      (left (str "Must be at most " n " characters")))))

(defn matches [pattern msg]
  (fn [s]
    (if (re-matches pattern s)
      (right s)
      (left msg))))

;; Use
(validate "test@email.com"
  non-empty
  (min-length 5)
  (matches #".+@.+\..+" "Invalid email"))
;; => Right("test@email.com")

(validate ""
  non-empty
  (min-length 5))
;; => Left("Cannot be empty")

(validate "ab"
  non-empty
  (min-length 5))
;; => Left("Must be at least 5 characters")
```

---

*Part 24 จาก 100+ | ขั้นตอน 691-720 จาก 1000+*
