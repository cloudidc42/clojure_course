# Part 28: Advanced Macros และ Metaprogramming
## ขั้นตอนที่ 811-840: Compiler API, Syntax Quote, Code Generation, DSL Advanced

---

## บทนำ

Macros คือหนึ่งในความสามารถที่ทรงพลังที่สุดของ Clojure:
- รันที่ **compile time** ไม่ใช่ runtime
- รับ **code as data** ส่งคืน **code as data** (Homoiconicity)
- ใช้สร้าง **DSL**, **code generation**, **metaprogramming**

---

## ขั้นตอนที่ 811: ทบทวน Syntax Quote

```clojure
;; Backtick (`) = syntax quote: fully qualify all symbols
`(println "hello")
;; => (clojure.core/println "hello")

;; Unquote (~) = evaluate inside syntax quote
(def name "World")
`(println ~name)
;; => (clojure.core/println "World")

;; Unquote-splicing (~@) = splice list into expression
(def items [1 2 3])
`(list ~@items)
;; => (clojure.core/list 1 2 3)

;; Gensym (#) = auto-generate unique symbol
`(let [x# 10] x#)
;; x# expands to something like x__123__auto__

;; Why gensym? Prevent variable capture
(defmacro bad-double [x]
  `(let [result (* ~x 2)]
     result))

;; Problem: if caller has 'result' in scope, can conflict!
;; Solution: use gensym
(defmacro safe-double [x]
  `(let [result# (* ~x 2)]
     result#))
```

---

## ขั้นตอนที่ 812: Macro Debugging

```clojure
;; macroexpand-1 = expand one level
(macroexpand-1 '(when true (println "yes")))
;; => (if true (do (println "yes")))

;; macroexpand = expand completely
(macroexpand '(when true (println "yes")))

;; clojure.walk/macroexpand-all = expand ALL nested macros
(require '[clojure.walk :as walk])
(walk/macroexpand-all '(-> 5 (* 3) (+ 2)))
;; => (+ (* 5 3) 2)

;; trace macro calls
(require '[clojure.pprint :as pp])

(defmacro debug-macro [form]
  `(do
     (println "Expanding:" '~form)
     ~form))

;; Check if something is a macro
(meta #'when)  ; => {:macro true ...}
```

---

## ขั้นตอนที่ 813: Pattern Matching Macro

```clojure
;; สร้าง simple pattern matching ด้วย macro

(defmacro match [expr & clauses]
  (let [e (gensym "expr")]
    `(let [~e ~expr]
       (cond
         ~@(mapcat
             (fn [[pattern body]]
               (cond
                 (= pattern :else)
                 [true body]
                 
                 (vector? pattern)
                 [`(and (vector? ~e)
                         (= (count ~e) ~(count pattern)))
                  (let [bindings (mapcat (fn [p i]
                                           [p `(nth ~e ~i)])
                                         pattern
                                         (range))]
                    `(let [~@bindings] ~body))]
                 
                 (map? pattern)
                 [`(and (map? ~e)
                         ~@(map (fn [[k v]] `(= ~v (get ~e ~k)))
                                pattern))
                  body]
                 
                 :else
                 [`(= ~e ~pattern) body]))
             (partition 2 clauses))))))

;; ใช้งาน
(match [:point 3 4]
  [:circle r]  (str "Circle with r=" r)
  [:point x y] (str "Point at " x "," y)
  :else         "Unknown shape")
;; => "Point at 3,4"

(match {:type :user :name "สมชาย"}
  {:type :admin} "Admin user"
  {:type :user :name n} (str "Regular user: " n)
  :else "Unknown")
;; => "Regular user: สมชาย"
```

---

## ขั้นตอนที่ 814: Code Generation Macros

```clojure
;; Generate code at compile time

;; Generate enum-like constants
(defmacro defenum [name & values]
  `(do
     ;; Define individual constants
     ~@(map-indexed (fn [i v]
                      `(def ~(symbol (str (name name) "-" v)) ~i))
                    values)
     ;; Define lookup map
     (def ~name
       {:values ~(mapv str values)
        :->int  ~(into {} (map-indexed (fn [i v] [(keyword v) i]) values))
        :->name ~(into {} (map-indexed (fn [i v] [i (str v)]) values))})))

(defenum Status PENDING ACTIVE INACTIVE DELETED)
;; Generates:
;; (def Status-PENDING 0)
;; (def Status-ACTIVE  1)
;; (def Status-INACTIVE 2)
;; (def Status-DELETED  3)
;; (def Status {:values [...] :->int {:PENDING 0...} :->name {0 "PENDING"...}})

;; Generate CRUD operations
(defmacro defentity [name table-name & fields]
  (let [table (keyword table-name)
        create-name (symbol (str "create-" name "!"))
        find-name   (symbol (str "find-" name))
        update-name (symbol (str "update-" name "!"))
        delete-name (symbol (str "delete-" name "!"))]
    `(do
       (defn ~create-name [data#]
         (sql/insert! db ~table data#))
       
       (defn ~find-name [id#]
         (sql/get-by-id db ~table id#))
       
       (defn ~update-name [id# updates#]
         (sql/update! db ~table updates# {:id id#}))
       
       (defn ~delete-name [id#]
         (sql/delete! db ~table {:id id#})))))

(defentity user :users :id :name :email :created-at)
;; Generates: create-user!, find-user, update-user!, delete-user!
```

---

## ขั้นตอนที่ 815: State Machine Macro

```clojure
;; DSL สำหรับ State Machine

(defmacro defstatemachine [name & transitions]
  (let [transition-map
        (reduce (fn [m [from event to action]]
                  (assoc-in m [from event] {:to to :action action}))
                {}
                (partition 4 transitions))]
    `(defn ~name [current-state event data]
       (if-let [transition# (get-in ~transition-map [current-state event])]
         (let [new-state# (:to transition#)]
           (when-let [action# (:action transition#)]
             (action# data))
           {:state new-state#
            :data  data})
         (throw (ex-info (str "No transition from " current-state " via " event)
                          {:state current-state :event event}))))))

;; ใช้งาน
(defstatemachine order-fsm
  :pending  :confirm  :confirmed  #(println "Order confirmed!" %)
  :pending  :cancel   :cancelled  #(println "Order cancelled!")
  :confirmed :ship    :shipped    #(println "Order shipped!" %)
  :shipped  :deliver  :delivered  #(println "Order delivered!" %))

(order-fsm :pending :confirm {:order-id 123})
;; => {:state :confirmed, :data {:order-id 123}}
```

---

## ขั้นตอนที่ 816: Reader Macros และ Tagged Literals

```clojure
;; Tagged Literals = custom reader syntax
;; Clojure built-ins: #inst, #uuid

#inst "2024-01-01"    ; => java.util.Date
#uuid "550e8400-e29b-41d4-a716-446655440000"  ; => java.util.UUID

;; Define custom tagged literal
;; File: src/myapp/tags.clj
(ns myapp.tags)

(defn currency [s]
  (let [[amount currency] (clojure.string/split s #"\s+")]
    {:amount   (BigDecimal. amount)
     :currency (keyword currency)}))

(defn env [var-name]
  (or (System/getenv var-name)
      (throw (RuntimeException. (str "Missing env var: " var-name)))))

;; File: data_readers.clj (in classpath root)
;; {my/currency myapp.tags/currency
;;  my/env      myapp.tags/env}

;; Usage in code:
;; #my/currency "100.00 THB"
;; => {:amount 100.00M :currency :THB}

;; #my/env "DATABASE_URL"
;; => "jdbc:postgresql://..."
```

---

## ขั้นตอนที่ 817: Compile-Time Computation

```clojure
;; Macro ทำงานที่ compile time → no runtime cost!

;; Computed at compile time
(defmacro compile-time-sqrt [n]
  (Math/sqrt n))

;; This is evaluated ONCE at compile time:
(def sqrt-of-2 (compile-time-sqrt 2))
;; In bytecode: (def sqrt-of-2 1.4142135623730951)

;; Pre-compute lookup tables at compile time
(defmacro precompute-table [f max-n]
  `(vector ~@(map f (range max-n))))

(def sin-table
  (precompute-table #(Math/sin (/ (* % Math/PI) 180.0)) 360))
;; Sine table computed once at startup, O(1) lookup

;; Conditional compilation
(defmacro when-dev [& body]
  (when (= "dev" (System/getenv "APP_ENV"))
    `(do ~@body)))

(when-dev
  (println "Debug mode on!")
  (reset! debug-mode true))
;; Code completely absent in production builds!
```

---

## ขั้นตอนที่ 818: Advanced DSL Patterns

```clojure
;; Query DSL สำหรับ HoneySQL

(defmacro defquery [name & clauses]
  (let [parts (partition 2 clauses)
        honeysql-map (reduce (fn [m [k v]]
                               (case k
                                 :from   (assoc m :from [v])
                                 :select (assoc m :select (if (vector? v) v [v]))
                                 :where  (assoc m :where v)
                                 :join   (assoc m :join (flatten [(:join m []) [v]]))
                                 :limit  (assoc m :limit v)
                                 :order  (assoc m :order-by (if (vector? v) v [v]))
                                 m))
                             {}
                             parts)]
    `(def ~name (quote ~honeysql-map))))

(defquery active-users-query
  :from   :users
  :select [:id :name :email]
  :where  [:= :status "active"]
  :order  [[:created-at :desc]]
  :limit  100)

;; SQL DSL for permissions
(defmacro with-permissions [permissions & body]
  `(let [user# *current-user*]
     (doseq [perm# ~permissions]
       (when-not (has-permission? user# perm#)
         (throw (ex-info (str "Missing permission: " perm#)
                          {:status 403 :permission perm#}))))
     ~@body))

(with-permissions [:user/read :user/write]
  (update-user! user-id {:name "New Name"}))
```

---

## Project: Test Framework DSL

```clojure
;; สร้าง mini test framework ด้วย macros

(def ^:dynamic *test-context* nil)
(def test-results (atom []))

(defmacro describe [desc & body]
  `(binding [*test-context* ~desc]
     ~@body))

(defmacro it [desc & body]
  `(let [context# *test-context*]
     (try
       ~@body
       (swap! test-results conj
         {:desc    (str context# " " ~desc)
          :status  :pass})
       (println (str "✓ " context# " " ~desc))
       (catch AssertionError e#
         (swap! test-results conj
           {:desc    (str context# " " ~desc)
            :status  :fail
            :error   (.getMessage e#)})
         (println (str "✗ " context# " " ~desc ": " (.getMessage e#))))
       (catch Exception e#
         (swap! test-results conj
           {:desc    (str context# " " ~desc)
            :status  :error
            :error   (.getMessage e#)})
         (println (str "✗ ERROR: " context# " " ~desc))))))

(defmacro expect [actual expected]
  `(when-not (= ~actual ~expected)
     (throw (AssertionError.
              (str "Expected: " ~expected
                   "\n  Got: " ~actual)))))

;; ใช้งาน
(describe "Calculator"
  (it "adds two numbers"
    (expect (+ 1 2) 3))
  
  (it "multiplies numbers"
    (expect (* 3 4) 12))
  
  (it "throws on division by zero"
    (try
      (/ 1 0)
      (expect :no-exception :should-throw)
      (catch ArithmeticException _
        :ok))))

;; Run results summary
(defn summary []
  (let [results @test-results
        passed  (count (filter #(= :pass (:status %)) results))
        failed  (count (filter #(= :fail (:status %)) results))]
    (println (format "\n%d passed, %d failed" passed failed))))
```

---

*Part 28 จาก 100+ | ขั้นตอน 811-840 จาก 1000+*
