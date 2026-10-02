# Part 12: Macros ขั้นสูง
## ขั้นตอนที่ 331-360: Code Generation, DSLs, Metaprogramming

---

## บทนำ

Macros คือ **code that writes code** ใน Clojure เราสามารถ extend ภาษาได้!
- Macro ทำงานที่ **compile time** (ก่อน runtime)
- Input: unevaluated forms (S-expressions)
- Output: new Clojure code
- ทำให้สร้าง **DSL** (Domain-Specific Language) ได้

```clojure
;; ตัวอย่าง: when macro
(macroexpand '(when (> x 0) (println x) x))
;; => (if (> x 0) (do (println x) x) nil)
;;    ^^^ Macro เปลี่ยน code ก่อน compile!
```

---

## ขั้นตอนที่ 331: Macro Basics

```clojure
;; defmacro = define macro
;; arguments = unevaluated S-expressions

(defmacro my-when [condition & body]
  `(if ~condition
     (do ~@body)
     nil))

;; Backtick (`) = syntax-quote: ป้องกัน evaluation + ใส่ namespace
;; ~ (tilde) = unquote: evaluate expression
;; ~@ (splicing) = unquote-splicing: splice list

;; ทดสอบ
(my-when true
  (println "Yes!")
  42)
;; => "Yes!" 42

;; macroexpand ดู code ที่ macro สร้าง
(macroexpand '(my-when true (println "Yes!") 42))
;; => (if true (do (println "Yes!") 42) nil)

;; macroexpand-all ดู recursive expansion
(clojure.walk/macroexpand-all '(my-when (> 1 0) (do-something)))
```

---

## ขั้นตอนที่ 332: Quote และ Unquote

```clojure
;; ' (quote) - ป้องกัน evaluation
(quote (+ 1 2))       ; => (+ 1 2)  (a list, not evaluated!)
'(+ 1 2)              ; shorthand

;; ` (backtick/syntax-quote) - คล้าย quote แต่ใส่ namespace
`(+ 1 2)              ; => (clojure.core/+ 1 2)
`foo                  ; => myapp.core/foo (current namespace)

;; ~ (tilde) - evaluate ใน syntax-quoted form
(let [x 42]
  `(println ~x))       ; => (clojure.core/println 42)

;; ~@ (splicing) - splice list ใน syntax-quoted form
(let [args [1 2 3]]
  `(+ ~@args))         ; => (clojure.core/+ 1 2 3)

;; ตัวอย่าง macro ใช้ทั้งหมด
(defmacro apply-fn [f & args]
  `(~f ~@args))

(apply-fn + 1 2 3)   ; => 6
(apply-fn println "hello" "world")  ; prints "hello world"
```

---

## ขั้นตอนที่ 333: Gensym - Avoiding Variable Capture

```clojure
;; Variable capture = macro variable ไป conflict กับ user variable

;; ❌ Buggy macro: x ซ้ำกับ user's x
(defmacro bad-double [x]
  `(let [x (* ~x 2)]   ; captures user's x!
     x))

;; ✅ Fixed: ใช้ gensym สร้างชื่อ unique
(defmacro good-double [x]
  (let [temp (gensym "temp")]
    `(let [~temp (* ~x 2)]
       ~temp)))

;; หรือ ใช้ # ใน syntax-quote (auto-gensym)
(defmacro good-double-2 [x]
  `(let [temp# (* ~x 2)]
     temp#))

;; ตัวอย่าง: timing macro
(defmacro time-it [& body]
  `(let [start# (System/currentTimeMillis)
         result# (do ~@body)
         elapsed# (- (System/currentTimeMillis) start#)]
     (println "Elapsed:" elapsed# "ms")
     result#))

(time-it
  (Thread/sleep 1000)
  (+ 1 2))
;; Elapsed: 1001 ms
;; => 3
```

---

## ขั้นตอนที่ 334: Practical Macros

```clojure
;; unless = opposite of when
(defmacro unless [condition & body]
  `(when (not ~condition)
     ~@body))

(unless false
  (println "This runs!"))

;; while loop (Clojure ไม่มี built-in while)
(defmacro while [condition & body]
  `(loop []
     (when ~condition
       ~@body
       (recur))))

;; swap! ด้วย value
(defmacro update-in! [atom-ref path & update-form]
  `(swap! ~atom-ref update-in ~path (fn [v#] ~@update-form)))

;; -> with debug printing
(defmacro ->debug [x & forms]
  (reduce
    (fn [expr form]
      `(let [result# (~(first form) ~expr ~@(rest form))]
         (println "After" '~(first form) ":" result#)
         result#))
    x forms))

(->debug [1 2 3 4 5]
  (filter odd?)
  (map #(* % 2))
  (reduce +))
;; After filter: (1 3 5)
;; After map: (2 6 10)
;; After reduce: 18
```

---

## ขั้นตอนที่ 335: DSL - Domain-Specific Language

```clojure
;; สร้าง DSL สำหรับ HTML generation

(defmacro html [& elements]
  `(clojure.string/join "\n" (list ~@elements)))

(defmacro tag [name & content]
  `(str "<" ~(clojure.core/name name) ">"
        (clojure.string/join "" (list ~@content))
        "</" ~(clojure.core/name name) ">"))

(defmacro tag-attr [tag-name attrs & content]
  (let [attr-str (clojure.string/join " "
                   (map (fn [[k v]] (str (name k) "=\"" v "\"")) attrs))]
    `(str "<" ~(clojure.core/name tag-name) " " ~attr-str ">"
          (clojure.string/join "" (list ~@content))
          "</" ~(clojure.core/name tag-name) ">")))

;; ใช้งาน
(html
  (tag :html
    (tag :head (tag :title "My Page"))
    (tag :body
      (tag-attr :h1 {:class "title"} "Hello World")
      (tag :p "This is a paragraph"))))

;; DSL สำหรับ SQL
(defmacro select-query [table & clauses]
  (let [parts (partition 2 clauses)
        where-clause (first (filter #(= :where (first %)) parts))
        order-clause (first (filter #(= :order-by (first %)) parts))
        limit-clause (first (filter #(= :limit (first %)) parts))]
    `(str "SELECT * FROM " ~(name table)
          ~(when where-clause (str " WHERE " (second where-clause)))
          ~(when order-clause (str " ORDER BY " (second order-clause)))
          ~(when limit-clause (str " LIMIT " (second limit-clause))))))

(select-query users
  :where "active = true"
  :order-by "name"
  :limit 20)
;; => "SELECT * FROM users WHERE active = true ORDER BY name LIMIT 20"
```

---

## ขั้นตอนที่ 336: Compile-time Validation

```clojure
;; Macros สามารถ validate ที่ compile time!

(defmacro def-enum [name & values]
  (when (empty? values)
    (throw (IllegalArgumentException. (str "Enum " name " must have values"))))
  `(do
     (def ~name #{~@values})
     (defn ~(symbol (str "valid-" name "?")) [v#]
       (contains? ~name v#))))

(def-enum status :active :inactive :pending :deleted)

;; Validates at compile time:
;; (def-enum empty-enum)  ; throws!

;; Runtime:
status  ; => #{:active :inactive :pending :deleted}
(valid-status? :active)   ; => true
(valid-status? :unknown)  ; => false

;; Type-safe configuration macro
(defmacro defconfig [name & {:as opts}]
  (let [required-keys [:host :port]
        missing (filter #(not (contains? opts %)) required-keys)]
    (when (seq missing)
      (throw (IllegalArgumentException.
               (str "Config " name " missing required keys: " missing))))
    `(def ~name ~opts)))

;; ✅ Compiles
(defconfig db-config :host "localhost" :port 5432)

;; ❌ Compile error!
;; (defconfig broken-config :host "localhost")
;; IllegalArgumentException: Config broken-config missing required keys: [:port]
```

---

## ขั้นตอนที่ 337: Macro for Retry Logic

```clojure
;; Retry macro - elegantly retry on failure
(defmacro with-retry
  "Retry body up to max-retries times with exponential backoff"
  [{:keys [max-retries base-delay-ms on-retry]
    :or   {max-retries 3 base-delay-ms 1000}}
   & body]
  `(loop [attempt# 1]
     (let [result# (try
                     {:ok (do ~@body)}
                     (catch Exception e#
                       {:err e#}))]
       (cond
         (:ok result#) (:ok result#)
         
         (>= attempt# ~max-retries)
         (throw (:err result#))
         
         :else
         (let [delay# (* ~base-delay-ms (Math/pow 2 (dec attempt#)))]
           ~(when on-retry
              `(~on-retry {:attempt attempt# :error (:err result#)}))
           (Thread/sleep (long delay#))
           (recur (inc attempt#)))))))

;; ใช้งาน
(with-retry {:max-retries 3
             :base-delay-ms 500
             :on-retry (fn [{:keys [attempt]}]
                         (println "Retry attempt:" attempt))}
  (fetch-from-unreliable-api))
```

---

## ขั้นตอนที่ 338: Recursive Macros

```clojure
;; Macro ที่ call ตัวเอง (ระวัง - ต้องมี base case)

;; cond-> แบบ custom
(defmacro cond-update
  "Conditionally update a map"
  [m & clauses]
  (if (empty? clauses)
    m
    (let [[test-expr update-fn & rest] clauses]
      `(cond-update
        (if ~test-expr
          (~update-fn ~m)
          ~m)
        ~@rest))))

;; Builder pattern macro
(defmacro build
  "Build a value through a series of transformations"
  [initial & steps]
  (reduce (fn [form [condition transform]]
            (if (true? condition)
              `(-> ~form ~transform)
              `(if ~condition (-> ~form ~transform) ~form)))
          initial
          (partition 2 steps)))

;; ใช้งาน
(build {}
  :always    (assoc :created-at (System/currentTimeMillis))
  name       (assoc :name name)
  email      (assoc :email email)
  admin?     (assoc :role :admin))
```

---

## ขั้นตอนที่ 339: Reader Macros และ Tagged Literals

```clojure
;; Clojure built-in reader macros:
;; ' → quote
;; ` → syntax-quote
;; ~ → unquote
;; ~@ → unquote-splicing
;; @ → deref
;; # → function/set/tagged literal
;; ^ → metadata

;; Tagged literals (data readers)
;; สร้าง custom tagged literal

;; data_readers.clj (ใน classpath root)
;; {myapp/date myapp.reader/read-date}

;; myapp/reader.clj
(defn read-date [s]
  (java.time.LocalDate/parse s))

;; ใช้งาน
#myapp/date "2024-01-15"
;; => #object[java.time.LocalDate ... 2024-01-15]

;; Built-in tagged literals:
#inst "2024-01-15T00:00:00Z"  ; java.util.Date
#uuid "a0be1234-..."           ; java.util.UUID

;; Metadata
(def ^:private my-var 42)
(def ^{:doc "Description" :added "1.0"} another-var 42)
(meta #'another-var)
;; => {:doc "Description", :added "1.0", :name another-var, ...}
```

---

## ขั้นตอนที่ 340: Testing Macros

```clojure
;; Test macros ด้วย macroexpand
(deftest test-my-when-macro
  ;; Test expansion
  (is (= '(if true (do 1 2) nil)
         (macroexpand '(my-when true 1 2))))
  
  ;; Test behavior
  (is (= 2 (my-when true 1 2)))
  (is (nil? (my-when false 1 2)))
  
  ;; Test with side effects
  (let [called? (atom false)]
    (my-when false (reset! called? true))
    (is (not @called?)))
  
  (let [called? (atom false)]
    (my-when true (reset! called? true))
    (is @called?)))

;; Test that macro doesn't capture variables
(deftest test-no-variable-capture
  (let [x 5]
    ;; x should still be 5, not affected by macro internals
    (is (= 10 (good-double x)))
    (is (= 5 x))))
```

---

## Project Exercise: Test Framework DSL

```clojure
;; สร้าง test framework ขนาดเล็ก ด้วย macros

(ns minitest.core)

;; State
(def ^:dynamic *test-results* nil)

;; Core assertion
(defmacro check [description form]
  `(let [result# (try (do ~form true) (catch Exception e# false))
         msg#    (str (if result# "✓" "✗") " " ~description)]
     (when *test-results*
       (swap! *test-results* conj {:desc ~description :pass result# :form '~form}))
     (println msg#)
     result#))

;; describe/it DSL
(defmacro describe [name & tests]
  `(do
     (println "\n" ~name)
     ~@tests))

(defmacro it [name & assertions]
  `(do
     (println "\n  " ~name)
     ~@(map (fn [assertion]
              `(check ~(str "    " assertion) ~assertion))
            assertions)))

;; expect/to DSL  
(defmacro expect [actual & [matcher expected]]
  (case matcher
    :to-equal    `(= ~actual ~expected)
    :to-be-nil   `(nil? ~actual)
    :to-contain  `(some #(= ~expected %) ~actual)
    :to-throw    `(try ~actual false (catch Exception _# true))
    (throw (IllegalArgumentException. (str "Unknown matcher: " matcher)))))

;; ใช้งาน
(describe "Calculator"
  (it "adds numbers"
    (expect (+ 1 2) :to-equal 3)
    (expect (+ 0 0) :to-equal 0))
  
  (it "handles edge cases"
    (expect (/ 1 0) :to-throw)
    (expect nil :to-be-nil))
  
  (it "works with collections"
    (expect [1 2 3] :to-contain 2)
    (expect (filter even? [1 2 3 4]) :to-equal '(2 4))))
```

---

### สรุป Macros

```
เมื่อไหร่ควรใช้ Macros:
========================
✓ Control flow ใหม่ (unless, while, with-*)
✓ DSL สำหรับ configuration/routing/testing
✓ Compile-time validation/optimization  
✓ Code elimination (ไม่ต้องประมวลผลตอน runtime)
✓ Boilerplate reduction

เมื่อไหร่ไม่ควรใช้:
====================
✗ ถ้า function ทำได้ → ใช้ function แทน!
✗ เพื่อความเท่ → ทำให้ code อ่านยาก
✗ สำหรับ abstraction ที่ไม่จำเป็น

กฎสำคัญ:
==========
1. ใช้ ` (backtick) แทน ' (quote) ใน macro body
2. ใช้ gensym หรือ # เพื่อ avoid variable capture
3. ทดสอบด้วย macroexpand ก่อน
4. ถ้า function ทำได้ → ใช้ function!
```

---

*Part 12 จาก 100+ | ขั้นตอน 331-360 จาก 1000+*
