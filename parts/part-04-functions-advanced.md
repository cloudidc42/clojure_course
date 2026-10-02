# Part 4: Functions ขั้นสูง
## ขั้นตอนที่ 91-120: Mastering Functional Programming

---

## บทนำ

Functions ใน Clojure ไม่ใช่แค่ "block of code" - มันเป็น **building blocks** ของทุกอย่าง หลักการ Functional Programming ที่เราจะเรียนรู้ใน Part นี้จะทำให้คุณเขียนโค้ดที่:
- เชื่อถือได้มากขึ้น
- ทดสอบง่ายขึ้น
- แก้ไขได้ง่ายขึ้น
- อ่านได้ง่ายขึ้น

---

## ขั้นตอนที่ 91: defn vs fn

```clojure
;; defn = def + fn (syntactic sugar)
(defn add [a b]
  (+ a b))

;; เท่ากับ:
(def add
  (fn [a b]
    (+ a b)))

;; fn - anonymous function
(fn [x] (* x x))

;; เรียกใช้ทันที (IIFE - Immediately Invoked Function Expression)
((fn [x] (* x x)) 5)   ; => 25

;; Short form #() - สำหรับ function สั้นๆ
#(* % %)                ; anonymous function กับ 1 argument
#(+ %1 %2)             ; 2 arguments
#(str %1 " " %2 " " %3) ; 3 arguments

(#(* % %) 5)            ; => 25
(#(+ %1 %2) 3 4)        ; => 7

;; ข้อจำกัดของ #()
;; - ไม่สามารถ nest ได้
;; - ควรใช้เมื่อ expression สั้น (ไม่เกิน 1-2 operations)
;; - ถ้า logic ซับซ้อน ใช้ fn หรือ defn แทน
```

---

## ขั้นตอนที่ 92: Multiple Arities

```clojure
;; defn รองรับหลาย arities (จำนวน arguments)
(defn greet
  ([]      "สวัสดี!")
  ([name]  (str "สวัสดี " name "!"))
  ([name greeting] (str greeting " " name "!")))

(greet)                    ; => "สวัสดี!"
(greet "สมชาย")            ; => "สวัสดี สมชาย!"
(greet "สมชาย" "หวัดดี")   ; => "หวัดดี สมชาย!"

;; Variadic functions (& rest args)
(defn sum [& numbers]
  (reduce + numbers))

(sum 1 2 3)               ; => 6
(sum 1 2 3 4 5)           ; => 15

;; Mix fixed and variadic
(defn log [level message & more]
  (apply println (str "[" level "]" message) more))

(log "INFO" "Starting server" "port=8080" "env=prod")
;; [INFO]Starting server port=8080 env=prod

;; Default arguments ด้วย multi-arity
(defn create-user
  ([name] (create-user name "user"))
  ([name role] (create-user name role nil))
  ([name role email]
   {:name name
    :role role
    :email email
    :created-at (System/currentTimeMillis)}))

(create-user "สมชาย")
(create-user "สมชาย" "admin")
(create-user "สมชาย" "admin" "somchai@test.com")
```

---

## ขั้นตอนที่ 93: Higher-Order Functions

```clojure
;; Functions ที่รับ function เป็น argument
;; และ/หรือ คืน function

;; === Functions ที่รับ functions ===

;; map, filter, reduce (เรียนแล้ว)
(map inc [1 2 3])

;; apply - ใช้ list เป็น arguments
(apply + [1 2 3 4 5])        ; => 15
(apply str ["a" "b" "c"])    ; => "abc"
(apply max [3 1 4 1 5 9])   ; => 9

;; ต่างจาก map:
;; map:   (map + [1 2 3])  → (+1) (+2) (+3) → (1 2 3)
;; apply: (apply + [1 2 3]) → (+ 1 2 3) → 6

;; every?, some, not-any?, not-every?
(every? pos? [1 2 3])      ; => true
(some neg? [1 -2 3])       ; => true
(some #{:admin} roles)      ; => :admin หรือ nil

;; === Functions ที่คืน functions ===

;; partial - apply บาง arguments ก่อน
(def double (partial * 2))
(double 5)                  ; => 10
(map double [1 2 3])        ; => (2 4 6)

(def add-10 (partial + 10))
(add-10 5)                  ; => 15
(map add-10 [1 2 3])        ; => (11 12 13)

;; partial ด้วย multiple args
(def greet-admin (partial str "Welcome, admin: "))
(greet-admin "John")        ; => "Welcome, admin: John"

;; comp - compose functions
(def process (comp str/upper-case str/trim))
(process "  hello  ")       ; => "HELLO"

(def analyze (comp frequencies str/lower-case))
(analyze "Hello World")
;; => {\h 1, \e 1, \l 3, \o 2, \  1, \w 1, \r 1, \d 1}

;; Pipeline ด้วย comp
(def transform
  (comp
   (partial filter #(> (:score %) 80))
   (partial map #(assoc % :grade "A"))))

(transform students)
```

---

## ขั้นตอนที่ 94: Function Composition

```clojure
;; comp - compose สร้าง pipeline ของ functions
(defn process-name [s]
  ((comp str/capitalize str/trim str/lower-case) s))

(process-name "  HELLO WORLD  ")
;; => "Hello world"

;; ใช้ comp อย่างเต็มที่
(def pipeline
  (comp
   (partial take 5)       ; take first 5
   (partial filter even?) ; keep evens
   (partial map #(* % 2)) ; double each
   ))

(pipeline (range 20))
;; => (0 4 8 12 16)

;; ลำดับ comp: ขวาไปซ้าย (ตรงข้ามกับ ->)
;; (comp f g h) = f(g(h(x)))
;; (-> x h g f) = f(g(h(x)))  ; เหมือนกัน!

;; เปรียบเทียบ
(def v [5 1 8 3 2 9 4 7 6])

;; ด้วย comp
((comp (partial take 3) sort) v)
;; => (1 2 3)

;; ด้วย ->
(-> v sort (take 3))
;; => (1 2 3)

;; Memoize - cache function results
(defn slow-fib [n]
  (Thread/sleep 100)  ; simulate slow calculation
  (if (<= n 1) n
      (+ (slow-fib (- n 1)) (slow-fib (- n 2)))))

(def fast-fib (memoize slow-fib))
(time (fast-fib 30))  ; slow first time
(time (fast-fib 30))  ; instant second time!
```

---

## ขั้นตอนที่ 95: Closures

```clojure
;; Closure: function ที่ "จำ" environment ที่สร้างมา

;; ตัวอย่างง่าย
(defn make-adder [n]
  (fn [x] (+ n x)))  ; fn ที่ "จำ" n

(def add-5 (make-adder 5))
(def add-10 (make-adder 10))

(add-5 3)    ; => 8
(add-10 3)   ; => 13

;; Closure กับ state (ระวัง! มี side effect)
(defn make-counter []
  (let [count (atom 0)]
    {:increment #(swap! count inc)
     :get #(deref count)
     :reset #(reset! count 0)}))

(def counter (make-counter))
((:increment counter))
((:increment counter))
((:increment counter))
((:get counter))    ; => 3
((:reset counter))
((:get counter))    ; => 0

;; Closure สำหรับ configuration
(defn make-db-query [db-spec]
  (fn [sql params]
    ;; db-spec ถูก "จำ" ใน closure
    (jdbc/query db-spec [sql params])))

(def query (make-db-query production-db))
(query "SELECT * FROM users WHERE id = ?" [1])
```

---

## ขั้นตอนที่ 96: Currying และ Partial Application

```clojure
;; Partial Application - fix บาง arguments
(defn multiply [a b c]
  (* a b c))

;; Fix a = 2
(def double-then-scale (partial multiply 2))
(double-then-scale 3 4)    ; => (* 2 3 4) = 24

;; Manual currying ใน Clojure
(defn curry-add [a]
  (fn [b]
    (fn [c]
      (+ a b c))))

(((curry-add 1) 2) 3)     ; => 6
((curry-add 1) 2)          ; => fn
(curry-add 1)              ; => fn

;; Practical partial application
(def users [{:id 1 :name "A" :role :admin}
            {:id 2 :name "B" :role :user}
            {:id 3 :name "C" :role :admin}])

;; Filter by role
(def admin? (comp #{:admin} :role))
(def admins (partial filter admin?))

(admins users)
;; => ({:id 1, :name "A", :role :admin} {:id 3, ...})

;; Database query builder
(defn query-users [db filters]
  (filter (fn [user]
            (every? (fn [[k v]]
                      (= (get user k) v))
                    filters))
          db))

(def find-admins (partial query-users users {:role :admin}))
(find-admins)
;; หรือ
(query-users users {:role :admin})
```

---

## ขั้นตอนที่ 97: Recursion

```clojure
;; Recursion พื้นฐาน
(defn factorial [n]
  (if (<= n 1)
    1
    (* n (factorial (- n 1)))))

(factorial 5)   ; => 120
(factorial 10)  ; => 3628800

;; ปัญหา: stack overflow กับ n ขนาดใหญ่
;; (factorial 10000)  ; => StackOverflowError!

;; วิธีแก้: Tail Recursion ด้วย recur
(defn factorial-tail [n]
  (loop [n n acc 1]
    (if (<= n 1)
      acc
      (recur (dec n) (* acc n)))))

(factorial-tail 10000)  ; ✓ ไม่ stack overflow!

;; recur = จาก tail position เท่านั้น
(defn sum-to [n]
  (loop [i 1 acc 0]
    (if (> i n)
      acc
      (recur (inc i) (+ acc i)))))

(sum-to 100)   ; => 5050
(sum-to 1000000)  ; => ทำงานได้!

;; Mutual recursion
(declare odd?)

(defn even?' [n]
  (if (= n 0) true (odd? (dec n))))

(defn odd? [n]
  (if (= n 0) false (even?' (dec n))))

;; ใช้ trampoline สำหรับ mutual recursion ที่ large n
(defn even-trampoline [n]
  (if (= n 0) true #(odd-trampoline (dec n))))

(defn odd-trampoline [n]
  (if (= n 0) false #(even-trampoline (dec n))))

(trampoline even-trampoline 1000000)  ; => true
```

---

## ขั้นตอนที่ 98: loop/recur Pattern

```clojure
;; loop/recur เป็น idiomatic Clojure สำหรับ iteration

;; Pattern พื้นฐาน
(loop [bindings initial-values]
  (if termination-condition
    result
    (recur new-values)))

;; ตัวอย่าง: หา max ใน list
(defn my-max [numbers]
  (loop [remaining (rest numbers)
         current-max (first numbers)]
    (if (empty? remaining)
      current-max
      (recur (rest remaining)
             (if (> (first remaining) current-max)
               (first remaining)
               current-max)))))

(my-max [3 1 4 1 5 9 2 6])   ; => 9

;; ตัวอย่าง: accumulate ด้วย loop
(defn process-items [items]
  (loop [remaining items
         result []
         errors []]
    (if (empty? remaining)
      {:results result :errors errors}
      (let [item (first remaining)
            [new-result new-errors]
            (try
              [(conj result (process-item item)) errors]
              (catch Exception e
                [result (conj errors {:item item :error (.getMessage e)})]))]
        (recur (rest remaining) new-result new-errors)))))

;; ตัวอย่าง: countdown
(defn countdown [n]
  (loop [i n]
    (when (> i 0)
      (println i)
      (recur (dec i))))
  (println "Go!"))

(countdown 5)
;; 5
;; 4
;; 3
;; 2
;; 1
;; Go!
```

---

## ขั้นตอนที่ 99: Trampolines

```clojure
;; trampoline แก้ปัญหา mutual recursion + stack overflow

;; แทนที่จะ call function ตรงๆ ให้คืน thunk (zero-arg fn)
;; trampoline จะ keep calling จนไม่ใช่ fn

(defn even? [n]
  (if (= n 0)
    true
    #(odd? (dec n))))   ; คืน thunk แทน recursive call!

(defn odd? [n]
  (if (= n 0)
    false
    #(even? (dec n))))

(trampoline even? 100000)   ; => true (ไม่ stack overflow!)
(trampoline odd? 100001)    ; => true

;; สร้าง trampolining function
(defmacro trampoline-fn [& body]
  `(trampoline (fn [] ~@body)))
```

---

## ขั้นตอนที่ 100: Memoization

```clojure
;; memoize - cache function results
(def expensive-fn
  (memoize
   (fn [n]
     (Thread/sleep 1000)  ; simulate slow work
     (* n n))))

(time (expensive-fn 5))   ; "Elapsed time: 1000ms"
(time (expensive-fn 5))   ; "Elapsed time: 0ms" (cached!)
(time (expensive-fn 6))   ; "Elapsed time: 1000ms" (different arg)

;; Fibonacci กับ memoize (classic example)
(def fib
  (memoize
   (fn fib [n]
     (if (<= n 1)
       n
       (+ (fib (- n 1)) (fib (- n 2)))))))

(fib 50)    ; => 12586269025 (instant!)
(fib 100)   ; => instant!

;; Manual memoize ที่ flexible กว่า
(defn memoize-with-ttl
  "Memoize ที่มี time-to-live (seconds)"
  [f ttl-seconds]
  (let [cache (atom {})]
    (fn [& args]
      (let [now (System/currentTimeMillis)
            ttl-ms (* ttl-seconds 1000)
            cached (get @cache args)]
        (if (and cached (< (- now (:timestamp cached)) ttl-ms))
          (:value cached)
          (let [result (apply f args)]
            (swap! cache assoc args {:value result :timestamp now})
            result))))))

(def get-user-cached
  (memoize-with-ttl
   (fn [id]
     (fetch-user-from-db id))  ; expensive DB call
   300))  ; cache 5 minutes
```

---

## ขั้นตอนที่ 101: Transducers - เร็วและ Composable

```clojure
;; ปัญหากับ map + filter ปกติ: สร้าง intermediate collections
(->> (range 1000000)
     (filter even?)    ; สร้าง list ชั่วคราว 500000 elements
     (map #(* % %))    ; สร้าง list ชั่วคราวอีก 500000 elements
     (take 5))
;; ช้า + ใช้ memory มาก

;; Transducers: ไม่มี intermediate collections!
(sequence
 (comp
  (filter even?)
  (map #(* % %))
  (take 5))
 (range 1000000))
;; เร็วกว่า + ใช้ memory น้อยกว่า

;; Transducer = function ที่ transform reducing function
;; (filter even?) = transducer
;; (map #(* % %)) = transducer
;; (comp ...) = composed transducer

;; วิธีใช้ transducers:
;; 1. sequence - สร้าง lazy seq
(sequence (map inc) [1 2 3])
;; => (2 3 4)

;; 2. into - สร้าง collection
(into [] (map inc) [1 2 3])
;; => [2 3 4]

(into #{} (filter even?) [1 2 3 4 5])
;; => #{2 4}

;; 3. transduce - reduce ด้วย transducer
(transduce (comp (filter even?) (map #(* % %))) + (range 10))
;; => 0 + 4 + 16 + 36 + 64 = 120

;; 4. eduction - lazy evaluation
(def xf (comp (filter even?) (map #(* % %))))
(into [] xf (range 10))
;; => [0 4 16 36 64]
```

---

## ขั้นตอนที่ 102: Transducer ขั้นสูง

```clojure
;; Stateful transducers
;; dedupe transducer
(sequence (dedupe) [1 1 2 2 3 1 1])
;; => (1 2 3 1)

;; take transducer
(sequence (take 3) (range 100))
;; => (0 1 2)

;; partition-all transducer
(sequence (partition-all 3) (range 10))
;; => ((0 1 2) (3 4 5) (6 7 8) (9))

;; เขียน custom transducer
(defn my-map [f]
  (fn [rf]  ; rf = reducing function
    (fn
      ([] (rf))                  ; init
      ([result] (rf result))     ; completion
      ([result input]            ; step
       (rf result (f input))))))

(into [] (my-map inc) [1 2 3])
;; => [2 3 4]

;; Custom stateful transducer
(defn my-take [n]
  (fn [rf]
    (let [remaining (volatile! n)]
      (fn
        ([] (rf))
        ([result] (rf result))
        ([result input]
         (let [rem @remaining]
           (vswap! remaining dec)
           (if (pos? rem)
             (rf result input)
             (reduced result))))))))  ; reduced = stop!

(into [] (my-take 3) (range 100))
;; => [0 1 2]
```

---

## ขั้นตอนที่ 103: Protocols - Polymorphism

```clojure
;; Protocol = interface ของ Clojure
(defprotocol Shape
  (area [s] "คำนวณพื้นที่")
  (perimeter [s] "คำนวณเส้นรอบ")
  (describe [s] "อธิบาย shape"))

;; Implement กับ Record
(defrecord Circle [radius]
  Shape
  (area [_] (* Math/PI radius radius))
  (perimeter [_] (* 2 Math/PI radius))
  (describe [_] (str "วงกลมรัศมี " radius)))

(defrecord Rectangle [width height]
  Shape
  (area [_] (* width height))
  (perimeter [_] (* 2 (+ width height)))
  (describe [_] (str "สี่เหลี่ยม " width "x" height)))

(defrecord Triangle [a b c]  ; sides
  Shape
  (area [_]
    (let [s (/ (+ a b c) 2)]  ; Heron's formula
      (Math/sqrt (* s (- s a) (- s b) (- s c)))))
  (perimeter [_] (+ a b c))
  (describe [_] (str "สามเหลี่ยม " a "+" b "+" c)))

;; ใช้งาน
(def shapes
  [(->Circle 5)
   (->Rectangle 4 6)
   (->Triangle 3 4 5)])

(doseq [s shapes]
  (println (describe s))
  (printf "  Area: %.2f  Perimeter: %.2f%n"
          (double (area s))
          (double (perimeter s))))

;; Polymorphic ทั้งหมด!
(map area shapes)
(map perimeter shapes)
(apply max (map area shapes))  ; หา shape ที่ใหญ่สุด
```

---

## ขั้นตอนที่ 104: Multimethods - Open Polymorphism

```clojure
;; Multimethod = dispatch บน arbitrary value (ไม่ใช่แค่ type)

;; ประกาศ multimethod
(defmulti notify
  "ส่ง notification ตาม channel"
  :channel)

;; Implement สำหรับแต่ละ dispatch value
(defmethod notify :email [{:keys [to subject body]}]
  (println (str "Email to: " to))
  (println (str "Subject: " subject))
  (println (str "Body: " body)))

(defmethod notify :sms [{:keys [phone message]}]
  (println (str "SMS to: " phone))
  (println (str "Message: " message)))

(defmethod notify :push [{:keys [device title body]}]
  (println (str "Push to device: " device))
  (println (str "Title: " title)))

;; Default method
(defmethod notify :default [msg]
  (println "Unknown channel:" (:channel msg)))

;; ใช้งาน
(notify {:channel :email
         :to "user@test.com"
         :subject "Test"
         :body "Hello"})

(notify {:channel :sms
         :phone "+66891234567"
         :message "Your OTP is 123456"})

;; Dispatch บน type
(defmulti serialize class)

(defmethod serialize String [s]
  (str "\"" s "\""))

(defmethod serialize Number [n]
  (str n))

(defmethod serialize clojure.lang.IPersistentMap [m]
  (str "{" (str/join ", " (map #(str (serialize (key %)) ": " (serialize (val %))) m)) "}"))

;; Hierarchies
(derive ::cat ::animal)
(derive ::dog ::animal)

(defmulti make-sound ::animal-type)
(defmethod make-sound ::animal [_] "...")
(defmethod make-sound ::cat [_] "Meow")
(defmethod make-sound ::dog [_] "Woof")
```

---

## ขั้นตอนที่ 105: Functional Design Patterns

```clojure
;; === Pattern 1: Data Transformation Pipeline ===

(defn process-order [order]
  (-> order
      validate-order
      calculate-prices
      apply-discounts
      generate-invoice
      send-confirmation))

;; === Pattern 2: Strategy Pattern ===

;; Strategy as function parameter
(defn process-payment [payment-fn order]
  (payment-fn order))

(defn credit-card-payment [order]
  {:method :credit-card
   :amount (:total order)
   :status :processed})

(defn crypto-payment [order]
  {:method :crypto
   :amount (:total order)
   :currency "BTC"
   :status :pending})

(process-payment credit-card-payment order)
(process-payment crypto-payment order)

;; === Pattern 3: Observer Pattern ===

(def event-handlers (atom {}))

(defn subscribe [event handler]
  (swap! event-handlers update event (fnil conj []) handler))

(defn publish [event data]
  (doseq [handler (get @event-handlers event [])]
    (handler data)))

(subscribe :user-created
           (fn [user] (println "Send welcome email to" (:email user))))

(subscribe :user-created
           (fn [user] (println "Add to analytics" (:id user))))

(publish :user-created {:id 1 :email "test@test.com"})
```

---

## ขั้นตอนที่ 106: Function Composition Patterns

```clojure
;; === Middleware Pattern ===

(defn wrap-logging [handler]
  (fn [request]
    (println "Request:" request)
    (let [response (handler request)]
      (println "Response:" response)
      response)))

(defn wrap-auth [handler]
  (fn [request]
    (if (:token request)
      (handler request)
      {:status 401 :body "Unauthorized"})))

(defn wrap-timing [handler]
  (fn [request]
    (let [start (System/currentTimeMillis)
          response (handler request)
          elapsed (- (System/currentTimeMillis) start)]
      (assoc response :timing elapsed))))

;; Base handler
(defn my-handler [request]
  {:status 200 :body "Hello World"})

;; Compose middleware
(def app
  (-> my-handler
      wrap-auth
      wrap-logging
      wrap-timing))

(app {:path "/" :token "abc123"})

;; === Pipe Pattern ===
(defn pipe [& fns]
  (fn [x]
    (reduce (fn [acc f] (f acc)) x fns)))

(def process
  (pipe
   str/lower-case
   str/trim
   #(str/replace % #"\s+" "-")
   #(str "slug-" %)))

(process "  Hello World  ")
;; => "slug-hello-world"
```

---

## ขั้นตอนที่ 107: Error Handling Patterns

```clojure
;; === Either/Result Pattern ===

;; ไม่มีใน core แต่สามารถสร้างเองได้

(defn ok [value] {:ok value})
(defn error [msg] {:error msg})
(defn ok? [{:keys [ok]}] (some? ok))

(defn parse-int [s]
  (try
    (ok (Integer/parseInt s))
    (catch NumberFormatException _
      (error (str "ไม่ใช่ตัวเลข: " s)))))

(defn divide [a b]
  (if (zero? b)
    (error "ไม่สามารถหารด้วยศูนย์")
    (ok (/ a b))))

;; Chain operations
(defn chain [result f]
  (if (ok? result)
    (f (:ok result))
    result))

(-> (parse-int "10")
    (chain #(divide % 2))
    (chain #(ok (str "Result: " %))))
;; => {:ok "Result: 5"}

(-> (parse-int "abc")
    (chain #(divide % 2)))
;; => {:error "ไม่ใช่ตัวเลข: abc"}

;; ใช้ cats library สำหรับ proper monadic operations
;; [funcool/cats "2.4.2"]
;; (require '[cats.monad.either :as either])
```

---

## ขั้นตอนที่ 108: Function Utilities

```clojure
;; identity - คืน argument เดิม (useful ใน higher-order functions)
(identity 42)              ; => 42
(filter identity [1 nil 2 false 3])
;; => (1 2 3)

;; constantly - สร้าง function ที่คืนค่าเดิมเสมอ
(def always-42 (constantly 42))
(always-42)                ; => 42
(always-42 "anything")     ; => 42
(map (constantly 0) [1 2 3]) ; => (0 0 0)

;; juxt - apply หลาย functions พร้อมกัน
(def stats (juxt count #(apply + %) #(apply max %) #(apply min %)))
(stats [3 1 4 1 5 9 2 6])
;; => [8 31 9 1]  ; [count sum max min]

((juxt :name :age :email) {:name "สมชาย" :age 25 :email "test@test.com"})
;; => ["สมชาย" 25 "test@test.com"]

;; complement - negate predicate
(def not-empty? (complement empty?))
(not-empty? [])     ; => false
(not-empty? [1 2])  ; => true

(filter (complement nil?) [1 nil 2 nil 3])
;; => (1 2 3)

;; every-pred / some-fn - combine predicates
(def adult-male? (every-pred #(= :male (:gender %)) #(>= (:age %) 18)))
(adult-male? {:gender :male :age 25})   ; => true
(adult-male? {:gender :female :age 25}) ; => false
(adult-male? {:gender :male :age 15})   ; => false

(def young-or-admin? (some-fn #(< (:age %) 25) #(= :admin (:role %))))
(young-or-admin? {:age 20 :role :user})  ; => true
(young-or-admin? {:age 30 :role :admin}) ; => true
(young-or-admin? {:age 30 :role :user})  ; => nil (false)
```

---

## ขั้นตอนที่ 109: Macros พื้นฐาน

```clojure
;; Macro = code ที่สร้าง code (compile time!)

;; ตัวอย่างง่าย: unless (ตรงข้าม when)
(defmacro unless [condition & body]
  `(when (not ~condition)
     ~@body))

(unless false
  (println "This runs!"))
;; => This runs!

(unless true
  (println "This doesn't run"))
;; => nil

;; ดูว่า macro expand เป็นอะไร
(macroexpand '(unless false (println "hello")))
;; => (clojure.core/when (clojure.core/not false) (println "hello"))

;; swap macro
(defmacro swap-vals! [a b]
  `(let [tmp# ~a]
     (def ~a ~b)
     (def ~b tmp#)))

;; while loop
(defmacro while [condition & body]
  `(loop []
     (when ~condition
       ~@body
       (recur))))

;; log macro ที่ include file/line info
(defmacro log-info [& msg]
  `(println (str "[INFO] " ~*file* ":" ~(:line (meta &form)) " - " (str ~@msg))))
```

---

## ขั้นตอนที่ 110: let และ letfn

```clojure
;; let - local bindings
(let [x 10
      y 20
      z (+ x y)]  ; สามารถใช้ x และ y ได้ทันที!
  (println x y z))
;; => 10 20 30

;; let ซ้อน
(let [base 100
      with-tax (* base 1.07)
      rounded (Math/round with-tax)]
  {:original base :with-tax with-tax :final rounded})
;; => {:original 100, :with-tax 107.0, :final 107}

;; letfn - local functions (mutually recursive ได้)
(letfn [(even? [n] (if (= n 0) true (odd? (dec n))))
        (odd? [n] (if (= n 0) false (even? (dec n))))]
  (even? 4))
;; => true

;; Destructuring ใน let
(let [{:keys [name age] :or {age 0}} {:name "สมชาย"}
      [first-hobby & hobbies] ["อ่านหนังสือ" "เขียนโค้ด" "ออกกำลังกาย"]]
  (println name "อายุ" age)
  (println "งานอดิเรกหลัก:" first-hobby)
  (println "งานอดิเรกอื่น:" hobbies))
```

---

## ขั้นตอนที่ 111: do, doto, doseq, dotimes

```clojure
;; do - group expressions, คืนค่าสุดท้าย
(do
  (println "first")
  (println "second")
  42)           ; คืน 42

;; doto - chain method calls บน object เดียว
(doto (StringBuilder.)
  (.append "Hello")
  (.append " ")
  (.append "World")
  (.toString))
;; => "Hello World"

;; doto กับ map
(doto {}
  (println))  ; ไม่ได้ใช้บ่อย แต่เป็นไปได้

;; doseq - loop สำหรับ side effects
(doseq [x [1 2 3]]
  (println x))
;; 1
;; 2
;; 3

;; doseq กับ multiple bindings (nested loop)
(doseq [x [1 2]
        y [3 4]]
  (println x y))
;; 1 3
;; 1 4
;; 2 3
;; 2 4

;; doseq กับ :when
(doseq [x (range 10)
        :when (even? x)]
  (print x " "))
;; 0 2 4 6 8

;; doseq กับ :let
(doseq [item [{:name "A" :price 10}
              {:name "B" :price 20}]
        :let [total (* (:price item) 1.07)]]
  (printf "%s: %.2f%n" (:name item) total))

;; dotimes - loop n ครั้ง
(dotimes [i 5]
  (println "Iteration:" i))
;; Iteration: 0
;; Iteration: 1
;; Iteration: 2
;; Iteration: 3
;; Iteration: 4
```

---

## ขั้นตอนที่ 112: cond, case, condp

```clojure
;; cond - multiple conditions
(defn classify-age [age]
  (cond
    (< age 0)  "Invalid"
    (< age 13) "Child"
    (< age 18) "Teenager"
    (< age 65) "Adult"
    :else       "Senior"))

(classify-age 25)   ; => "Adult"
(classify-age 10)   ; => "Child"

;; case - match exact values (เร็วกว่า cond)
(defn day-name [n]
  (case n
    1 "จันทร์"
    2 "อังคาร"
    3 "พุธ"
    4 "พฤหัสบดี"
    5 "ศุกร์"
    6 "เสาร์"
    7 "อาทิตย์"
    "ไม่รู้จัก"))

(day-name 3)   ; => "พุธ"
(day-name 8)   ; => "ไม่รู้จัก"

;; condp - match กับ predicate
(defn classify-number [n]
  (condp < n      ; predicate: #(< % n)
    0   "Positive"
    -10 "Slightly negative"
    "Very negative"))

;; if-let / when-let
(defn greet-user [user-id]
  (if-let [user (find-user user-id)]
    (str "Welcome, " (:name user))
    "User not found"))

(defn process-if-exists [data key]
  (when-let [value (get data key)]
    (process value)))
```

---

## ขั้นตอนที่ 113: for Comprehension

```clojure
;; for - list comprehension (lazy)
(for [x [1 2 3]]
  (* x x))
;; => (1 4 9)

;; Multiple bindings (nested)
(for [x [1 2 3]
      y [4 5 6]]
  (* x y))
;; => (4 5 6 8 10 12 12 15 18)

;; :when - filter
(for [x (range 10)
      :when (even? x)]
  x)
;; => (0 2 4 6 8)

;; :let - local binding
(for [x (range 5)
      :let [sq (* x x)]
      :when (> sq 5)]
  {:x x :square sq})
;; => ({:x 3, :square 9} {:x 4, :square 16})

;; :while - stop condition
(for [x (range 10)
      :while (< x 5)]
  x)
;; => (0 1 2 3 4)

;; Practical: Combinations
(for [suit [:hearts :diamonds :clubs :spades]
      rank [:ace 2 3 4 5 6 7 8 9 10 :jack :queen :king]]
  {:suit suit :rank rank})
;; สร้างไพ่ 52 ใบ!

;; Matrix operations
(def matrix [[1 2 3] [4 5 6] [7 8 9]])

(for [row (range 3)
      col (range 3)
      :let [val (get-in matrix [row col])]
      :when (odd? val)]
  {:row row :col col :val val})
```

---

## ขั้นตอนที่ 114: Binding Forms

```clojure
;; binding - dynamic binding (thread-local)
(def ^:dynamic *db* nil)
(def ^:dynamic *current-user* nil)

(defn get-data []
  ;; ใช้ *db* โดยไม่ต้อง pass as argument
  (query *db* "SELECT * FROM data"))

;; ตั้งค่า dynamic vars ชั่วคราว
(binding [*db* test-database
          *current-user* {:id 1 :name "Test"}]
  (get-data))  ; ใช้ test-database ใน scope นี้

;; หลัง binding block: กลับเป็นค่าเดิม!

;; set! - เปลี่ยน dynamic var
(binding [*db* nil]
  (set! *db* production-db)  ; เปลี่ยนใน binding scope
  (get-data))

;; with-bindings
(with-bindings {#'*db* test-db
                #'*current-user* admin}
  (do-something))
```

---

## ขั้นตอนที่ 115: Function Best Practices

```clojure
;; 1. Functions ควรเล็กและทำ 1 อย่าง
;; ❌ ฟังก์ชันใหญ่ที่ทำหลายอย่าง
(defn process-user-and-send-email [data]
  (let [validated (validate data)
        saved (save-to-db validated)
        email (format-email saved)]
    (send-email email)))

;; ✅ แยกเป็น small functions
(defn validate-user [data] ...)
(defn save-user [data] ...)
(defn send-welcome-email [user] ...)

(defn create-user [data]
  (-> data validate-user save-user send-welcome-email))

;; 2. ใช้ meaningful names
;; ❌ (fn [x] (* x x x))
;; ✅ (fn [n] (* n n n))  ; หรือ
;; ✅ (defn cube [n] (* n n n))

;; 3. ให้ docstring
(defn calculate-bmi
  "คำนวณ BMI จากน้ำหนัก (kg) และส่วนสูง (meters)
  คืนค่า BMI และ category"
  [weight-kg height-m]
  (let [bmi (/ weight-kg (* height-m height-m))]
    {:bmi (double bmi)
     :category (cond
                 (< bmi 18.5) "Underweight"
                 (< bmi 25.0) "Normal"
                 (< bmi 30.0) "Overweight"
                 :else "Obese")}))

;; 4. ใช้ pre/post conditions สำหรับ validation
(defn divide [a b]
  {:pre [(number? a) (number? b) (not (zero? b))]
   :post [(number? %)]}
  (/ a b))
```

---

## ขั้นตอนที่ 116: Advanced Destructuring

```clojure
;; Destructuring ลึกซึ้ง

;; :as - ดึง whole value ด้วย
(let [[first & rest :as all] [1 2 3 4 5]]
  {:first first :rest rest :all all})
;; => {:first 1, :rest (2 3 4 5), :all [1 2 3 4 5]}

;; Nested vector
(let [[[a b] [c d]] [[1 2] [3 4]]]
  (+ a b c d))  ; => 10

;; Map กับ rename
(let [{name-val :name age-val :age} {:name "สมชาย" :age 25}]
  (println name-val age-val))
;; => สมชาย 25

;; String keys ใน map
(let [{"name" n "age" a} {"name" "John" "age" 30}]
  (println n a))
;; => John 30

;; Mixed
(defn analyze-response
  [{status :status
    {users :items total :count} :data
    [first-error & more-errors] :errors
    :or {errors []}}]
  {:ok (= status "success")
   :user-count total
   :first-user (first users)
   :has-errors (some? first-error)})
```

---

## ขั้นตอนที่ 117: Namespaced Maps

```clojure
;; ตั้งแต่ Clojure 1.9+

;; Literal namespaced map
#:user{:name "สมชาย" :age 25 :email "test@test.com"}
;; = {:user/name "สมชาย" :user/age 25 :user/email "test@test.com"}

;; Destructuring
(let [{:user/keys [name age email]}
      {:user/name "สมชาย" :user/age 25}]
  (println name age))
;; => สมชาย 25

;; ใช้ใน Spec
(s/def :user/name string?)
(s/def :user/age pos-int?)
(s/def ::user (s/keys :req [:user/name :user/age]))

;; ใน Datomic
{:db/id 1
 :person/name "สมชาย"
 :person/email "test@test.com"}
```

---

## ขั้นตอนที่ 118: Function Testing Strategies

```clojure
;; Unit Testing กับ clojure.test
(require '[clojure.test :refer :all])

;; Test pure functions
(deftest test-calculate-bmi
  (testing "Normal BMI"
    (let [result (calculate-bmi 70 1.75)]
      (is (= "Normal" (:category result)))
      (is (< 24.0 (:bmi result) 24.1))))
  
  (testing "Underweight"
    (is (= "Underweight" (:category (calculate-bmi 50 1.75)))))
  
  (testing "Obese"
    (is (= "Obese" (:category (calculate-bmi 100 1.70))))))

;; Property-based testing กับ test.check
;; [org.clojure/test.check "1.1.1"]

(require '[clojure.test.check.generators :as gen]
         '[clojure.test.check.properties :as prop]
         '[clojure.test.check :as tc])

;; Property: reverse ของ reverse = original
(def prop-double-reverse
  (prop/for-all [v (gen/vector gen/int)]
    (= v (vec (reverse (reverse v))))))

(tc/quick-check 1000 prop-double-reverse)
;; => {:result true, :num-tests 1000, :seed ...}

;; Property: sort idempotent
(def prop-sort-idempotent
  (prop/for-all [v (gen/vector gen/int)]
    (= (sort v) (sort (sort v)))))

(tc/quick-check 1000 prop-sort-idempotent)
```

---

## ขั้นตอนที่ 119: Functional Patterns Summary

```clojure
;; === Design สำหรับ Functional Code ===

;; 1. Data In, Data Out
;; Functions รับ data ธรรมดา คืน data ธรรมดา
;; ไม่มี hidden state, ไม่มี side effects

;; 2. Small Functions, Composed Together
;; ฟังก์ชันเล็กๆ compose เป็นระบบใหญ่

;; 3. Transform, Don't Mutate
;; สร้าง data ใหม่แทนที่จะแก้ data เดิม

;; 4. Separate IO from Logic
;; Logic ใน pure functions
;; IO ที่ edges เท่านั้น

;; Bad: Mixed IO and logic
(defn save-processed-user [user-id]
  (let [user (fetch-from-db user-id)  ; IO
        processed (process user)       ; logic
        _ (save-to-db processed)]      ; IO
    processed))

;; Better: Separate concerns
(defn process-user [user]              ; pure!
  (-> user
      validate
      normalize
      enrich))

(defn save-user! [user]                ; IO!
  (jdbc/execute! db ...))

;; Usage: IO at the edges
(let [user (fetch-user! user-id)]      ; IO
  (-> user
      process-user                      ; pure
      save-user!))                      ; IO
```

---

## ขั้นตอนที่ 120: Project Exercise - Functional Pipeline

```clojure
;; สร้าง ETL (Extract-Transform-Load) Pipeline แบบ Functional

(ns etl.core
  (:require [clojure.string :as str]))

;; Extract
(defn parse-csv-line [line]
  (str/split line #","))

(defn parse-csv [content]
  (let [lines (str/split-lines content)
        headers (map keyword (parse-csv-line (first lines)))
        rows (rest lines)]
    (map #(zipmap headers (parse-csv-line %)) rows)))

;; Transform
(defn normalize-name [record]
  (update record :name str/trim))

(defn parse-age [record]
  (update record :age #(try (Integer/parseInt %) (catch Exception _ nil))))

(defn parse-salary [record]
  (update record :salary #(try (Double/parseDouble %) (catch Exception _ 0.0))))

(defn validate-record [record]
  (when (and (:name record)
             (:age record)
             (pos? (:age record)))
    record))

(defn enrich-record [record]
  (assoc record
         :salary-band (cond
                        (< (:salary record 0) 30000) :low
                        (< (:salary record 0) 60000) :medium
                        :else :high)
         :processed-at (System/currentTimeMillis)))

;; Pipeline
(defn transform-pipeline [records]
  (->> records
       (map normalize-name)
       (map parse-age)
       (map parse-salary)
       (keep validate-record)
       (map enrich-record)))

;; Load
(defn generate-report [records]
  {:total-records (count records)
   :by-band (frequencies (map :salary-band records))
   :avg-age (double (/ (reduce + (map :age records))
                       (count records)))
   :max-salary (apply max (map :salary records))})

;; ทดสอบ
(comment
  (def csv-data
    "name,age,salary
สมชาย,25,55000
สมหญิง,30,75000
มานะ,22,25000
มาลี,35,90000")

  (def records (parse-csv csv-data))
  (def processed (transform-pipeline records))
  (def report (generate-report processed))

  ;; ดูผลลัพธ์
  (clojure.pprint/pprint report))
```

---

### อ่านต่อใน Part 5: Namespaces และ Project Structure →

---

*Part 4 จาก 100+ | ขั้นตอน 91-120 จาก 1000+*
