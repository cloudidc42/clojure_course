# Part 91: Advanced Macros & Metaprogramming
## ขั้นตอนที่ 2701-2730: Macro Design, Code Walking, Syntax Quoting, Compiler Hooks

---

## บทนำ

Macro programming ขั้นสูง:
- **Hygienic macros** - avoid symbol capture
- **Code walking** - transform arbitrary code
- **Compiler hooks** - custom transformation passes
- **DSL design** - domain-specific languages
- **Macro testing** - verify expansion

---

## ขั้นตอนที่ 2701: Hygienic Macros

```clojure
(ns myapp.macros)

;; Symbol capture problem
(defmacro bad-swap! [a b]
  ;; Bug: 'tmp' might conflict with user's 'tmp'
  `(let [tmp ~a]
     (set! ~a ~b)
     (set! ~b tmp)))

;; Fix with gensym
(defmacro safe-swap! [a b]
  (let [tmp (gensym "tmp")]
    `(let [~tmp ~a]
       (set! ~a ~b)
       (set! ~b ~tmp))))

;; Auto-gensym in syntax-quote: use # suffix
(defmacro with-timing [label & body]
  `(let [start# (System/nanoTime)
         result# (do ~@body)
         elapsed# (/ (- (System/nanoTime) start#) 1e6)]
     (println ~label "took" elapsed# "ms")
     result#))

;; Macro that expands to multiple forms
(defmacro def-validator [name & rules]
  `(defn ~name [data#]
     (reduce (fn [errors# [pred# msg#]]
               (if (pred# data#) errors# (conj errors# msg#)))
             []
             ~(vec (map (fn [[pred msg]] `[~pred ~msg]) rules)))))

;; Usage
(def-validator validate-user
  [(fn [u] (:name u))      "Name is required"]
  [(fn [u] (:email u))     "Email is required"]
  [(fn [u] (> (:age u) 18)) "Must be 18+"])
```

---

## ขั้นตอนที่ 2702: Code Walking ด้วย tools.analyzer

```clojure
(ns myapp.macros.walker
  (:require [clojure.walk :as walk]))

;; Walk and transform code
(defn replace-symbol [form old-sym new-sym]
  (walk/postwalk
    (fn [x]
      (if (= x old-sym) new-sym x))
    form))

;; Collect all symbols in a form
(defn collect-symbols [form]
  (let [result (atom #{})]
    (walk/postwalk
      (fn [x]
        (when (symbol? x) (swap! result conj x))
        x)
      form)
    @result))

;; Instrument function calls for tracing
(defmacro trace-calls [& body]
  (walk/postwalk
    (fn [form]
      (if (and (seq? form)
               (symbol? (first form))
               (not ('#{quote def defn let fn} (first form))))
        (let [fname (first form)
              gargs (gensym "args")]
          `(let [~gargs (list ~@(rest form))]
             (println "Calling" '~fname "with" ~gargs)
             (apply ~fname ~gargs)))
        form))
    `(do ~@body)))

;; Macro that generates multiple defs
(defmacro defentity [entity-name fields & specs]
  (let [ns-sym  (symbol (name entity-name))
        make-fn (symbol (str "make-" (name entity-name)))
        valid-fn (symbol (str "valid-" (name entity-name) "?"))]
    `(do
       (defrecord ~entity-name ~fields)
       
       (defn ~make-fn [~@fields]
         (~(symbol (str "->" (name entity-name))) ~@fields))
       
       (defn ~valid-fn [x#]
         (instance? ~entity-name x#))
       
       (defmethod print-method ~entity-name [v# w#]
         (.write w# (str "#" ~(str entity-name) " " (into {} v#)))))))

;; Usage
(defentity Product [id name price category])
;; Creates: ->Product map->Product, make-product, valid-product?, print-method
```

---

## ขั้นตอนที่ 2703: DSL Design Patterns

```clojure
;; Build DSLs that read like natural language

;; SQL-like query DSL
(defmacro query [& clauses]
  (let [parts (partition-by keyword? (cons :select clauses))
        clause-map (into {} (map (fn [[k & v]] [k v]) parts))]
    `(build-query ~clause-map)))

;; Usage:
;; (query :from orders :where (= status :paid) :order-by created-at :limit 10)

;; State machine DSL
(defmacro defstatemachine [name & transitions]
  `(def ~name
     {:transitions
      ~(into {}
             (map (fn [[from arrows to]]
                     (assert (= arrows '->))
                     [from (if (vector? to) (set to) #{to})])
                  (partition 3 transitions)))}))

(defstatemachine order-machine
  :pending    -> [:confirmed :cancelled]
  :confirmed  -> [:paid :cancelled]
  :paid       -> :shipped
  :shipped    -> :delivered)

;; HTTP routing DSL
(defmacro defroutes [& routes]
  `(fn [request#]
     (some (fn [[method# pattern# handler#]]
              (when (and (= method# (:request-method request#))
                         (matches-pattern? pattern# (:uri request#)))
                (handler# request#)))
            ~(mapv (fn [[method path handler]]
                      [method path handler])
                    (partition 3 routes)))))
```

---

## ขั้นตอนที่ 2704: Compile-Time Computation

```clojure
;; Macros can run arbitrary code at compile time

;; Generate a dispatch table at compile time
(defmacro def-op-table [name & ops]
  (let [op-map (into {} (map (fn [[sym op]]
                               [(name sym) op])
                              (partition 2 ops)))]
    `(def ~name ~op-map)))

(def-op-table math-ops
  + "add" - "subtract" * "multiply" / "divide")

;; Inline constants from files at compile time
(defmacro resource-string [filename]
  (slurp filename))  ; Runs at macro expansion time

;; Generate repetitive code from data
(defmacro gen-getters [record-name & fields]
  `(do
     ~@(map (fn [field]
               `(defn ~(symbol (str "get-" (name field)))
                  [~(symbol (name record-name))]
                  (~(keyword field) ~(symbol (name record-name)))))
             fields)))

(gen-getters user :id :name :email :role)
;; Generates: get-id, get-name, get-email, get-role

;; Reader literal macros
;; data_readers.clj: {env myapp.macros/env-reader}
(defn env-reader [var-name]
  (or (System/getenv (str var-name))
      (throw (ex-info "Missing env var" {:var var-name}))))
;; Usage in EDN: #env "DATABASE_URL"
```

---

## ขั้นตอนที่ 2705: Macro Testing

```clojure
;; Test macros by examining expansions

(require '[clojure.test :refer [deftest is testing]])

(deftest test-with-timing-expansion
  (testing "with-timing expands correctly"
    (let [expansion (macroexpand-1 '(with-timing "test" (+ 1 2)))]
      (is (= 'let (first expansion)))
      (is (some #(= 'System/nanoTime %) (flatten expansion))))))

(deftest test-with-timing-behavior
  (testing "with-timing returns result"
    (let [result (with-timing "test" (* 6 7))]
      (is (= 42 result)))))

;; Check that no reflection warnings in macro output
(deftest test-no-reflection
  (binding [*warn-on-reflection* true]
    (with-out-str  ; Capture warnings
      (eval '(with-timing "t" (str "hello"))))))

;; Macro expansion comparison
(defn expansions-equal? [macro-form expected-form]
  (= (macroexpand-all macro-form)
     (macroexpand-all expected-form)))

;; Verify hygiene: gensyms are unique
(deftest test-hygiene
  (let [exp1 (macroexpand-1 '(safe-swap! a b))
        exp2 (macroexpand-1 '(safe-swap! c d))]
    (is (not= (ffirst (rest exp1))  ; First gensym from exp1
              (ffirst (rest exp2))   ; First gensym from exp2
              ))))
```

---

*Part 91 จาก 100+ | ขั้นตอน 2701-2730 จาก 1000+*
