# Part 6: Control Flow และ Error Handling
## ขั้นตอนที่ 151-180: จัดการ Flow และ Errors อย่างมืออาชีพ

---

## บทนำ

Clojure มีแนวทางที่เป็นเอกลักษณ์ในการจัดการ control flow และ errors:
- **Expressions everywhere** - ทุกอย่างมีค่า
- **Immutable-first** - ไม่มี exception ที่เกิดจาก shared mutable state
- **Data as errors** - errors เป็น data ที่จัดการได้
- **Functional error handling** - ใช้ patterns เช่น Either, Maybe

---

## ขั้นตอนที่ 151: if, when, when-not

```clojure
;; if - conditional expression (คืนค่าเสมอ)
(if true "yes" "no")    ; => "yes"
(if false "yes" "no")   ; => "no"
(if nil "yes" "no")     ; => "no"
(if 0 "yes" "no")       ; => "yes" (0 เป็น truthy!)
(if "" "yes" "no")      ; => "yes" ("" เป็น truthy!)

;; if ที่ไม่มี else คืน nil
(if false "yes")         ; => nil

;; when - เมื่อต้องการแค่ branch เดียว
(when true
  (println "a")
  (println "b")
  42)                   ; => 42 (คืนค่าสุดท้าย)

(when false
  (println "never"))    ; => nil

;; when-not - ตรงข้าม when
(when-not (empty? items)
  (process-items items))

;; if กับ multiple expressions ใช้ do
(if condition
  (do
    (log-info "Processing...")
    (process data)
    (send-notification data))
  (log-error "Condition failed"))

;; if-not
(if-not (nil? x)
  (process x)
  (handle-nil))
```

---

## ขั้นตอนที่ 152: cond และ case

```clojure
;; cond - multiple branches
(defn classify-score [score]
  (cond
    (>= score 90) {:grade "A" :label "ดีมาก"}
    (>= score 80) {:grade "B" :label "ดี"}
    (>= score 70) {:grade "C" :label "พอใช้"}
    (>= score 60) {:grade "D" :label "ผ่าน"}
    :else         {:grade "F" :label "ไม่ผ่าน"}))

;; case - match exact values (compile-time lookup table)
(defn http-status [code]
  (case code
    200 "OK"
    201 "Created"
    204 "No Content"
    301 "Moved Permanently"
    400 "Bad Request"
    401 "Unauthorized"
    403 "Forbidden"
    404 "Not Found"
    500 "Internal Server Error"
    (str "Unknown: " code)))

;; case กับ multiple match values
(defn weekend? [day]
  (case day
    (:saturday :sunday) true
    false))

;; condp - condition with shared predicate
(defn classify-bmi [bmi]
  (condp > bmi
    18.5 "Underweight"
    25.0 "Normal"
    30.0 "Overweight"
    "Obese"))

;; condp กับ regex
(defn classify-email [email]
  (condp re-find email
    #"@gmail\.com$"   "Gmail"
    #"@yahoo\.com$"   "Yahoo"
    #"@outlook\.com$" "Outlook"
    "Other"))
```

---

## ขั้นตอนที่ 153: let bindings ขั้นสูง

```clojure
;; let กับ side effects
(let [result (expensive-operation)
      _ (println "Got result:" result)  ; _ เป็น convention สำหรับ discarded value
      processed (process result)]
  processed)

;; if-let
(if-let [user (find-user-by-email email)]
  (process user)
  {:error "User not found"})

;; when-let
(when-let [data (fetch-optional-data)]
  (process data))

;; if-some (Clojure 1.6+)
(if-some [value (possibly-nil-fn)]
  (use-value value)
  (handle-missing))
;; ต่างจาก if-let: if-let ถือว่า false เป็น falsy, if-some ถือว่า false เป็น truthy

;; when-some
(when-some [x (find-value)]
  (println x))

;; let ซ้อนกัน (nested let)
(let [x 10]
  (let [y 20]
    (let [z (+ x y)]
      z)))
;; => 30

;; ดีกว่า: ใช้ let เดียว
(let [x 10
      y 20
      z (+ x y)]
  z)
;; => 30
```

---

## ขั้นตอนที่ 154: Exception Handling

```clojure
;; try/catch/finally
(try
  (/ 10 0)
  (catch ArithmeticException e
    (println "ArithmeticException:" (.getMessage e))
    0)
  (catch Exception e
    (println "General Exception:" (.getMessage e))
    -1)
  (finally
    (println "This always runs")))

;; หลาย catch blocks
(defn read-file [filename]
  (try
    (slurp filename)
    (catch java.io.FileNotFoundException e
      {:error "File not found" :file filename})
    (catch java.io.IOException e
      {:error "IO error" :details (.getMessage e)})
    (catch Exception e
      {:error "Unexpected error" :details (.getMessage e)})))

;; throw exception
(defn divide [a b]
  (when (zero? b)
    (throw (ArithmeticException. "Cannot divide by zero")))
  (/ a b))

;; throw ด้วย ex-info (structured exception)
(defn create-user! [data]
  (let [errors (validate data)]
    (when (seq errors)
      (throw (ex-info "Validation failed"
                      {:type :validation-error
                       :errors errors
                       :input data})))))

;; catch ex-info
(try
  (create-user! invalid-data)
  (catch clojure.lang.ExceptionInfo e
    (let [data (ex-data e)]
      (println "Error type:" (:type data))
      (println "Errors:" (:errors data)))))
```

---

## ขั้นตอนที่ 155: ex-info และ Structured Errors

```clojure
;; ex-info เป็น preferred way ใน Clojure
(def e (ex-info "Something went wrong"
                {:type :validation-error
                 :field :email
                 :reason "Invalid format"}))

(ex-message e)  ; => "Something went wrong"
(ex-data e)     ; => {:type :validation-error, :field :email, :reason "Invalid format"}
(ex-cause e)    ; => nil

;; Chaining exceptions
(try
  (some-operation)
  (catch Exception cause
    (throw (ex-info "Higher-level error"
                    {:type :system-error}
                    cause))))

;; Error hierarchy ด้วย keywords
(defn business-error [type message data]
  (ex-info message (merge {:type type} data)))

(defn validation-error [field message]
  (business-error :validation/error
                  "Validation failed"
                  {:field field :message message}))

(defn not-found-error [resource id]
  (business-error :resource/not-found
                  (str resource " not found")
                  {:resource resource :id id}))

;; Pattern matching on error types
(defn handle-error [e]
  (let [{:keys [type]} (ex-data e)]
    (case type
      :validation/error  {:status 400 :body (ex-data e)}
      :resource/not-found {:status 404 :body {:error (ex-message e)}}
      {:status 500 :body {:error "Internal error"}})))
```

---

## ขั้นตอนที่ 156: Assertions และ Pre/Post Conditions

```clojure
;; assert
(assert (pos? x) "x must be positive")

;; Pre/post conditions ใน functions
(defn calculate-percentage [part total]
  {:pre  [(number? part)
          (number? total)
          (pos? total)
          (<= 0 part total)]
   :post [(>= % 0) (<= % 100)]}
  (* 100.0 (/ part total)))

(calculate-percentage 50 100)    ; => 50.0
(calculate-percentage 150 100)   ; => AssertionError
(calculate-percentage 50 0)      ; => AssertionError

;; ปิด assertions ใน production
;; lein uberjar ด้วย :jvm-opts ["-ea"] เปิด, ["-da"] ปิด

;; Validation ด้วย Spec (ดีกว่า assert)
(require '[clojure.spec.alpha :as s])

(s/def ::positive-number (s/and number? pos?))
(s/def ::percentage (s/and number? #(>= % 0) #(<= % 100)))

(s/fdef calculate-percentage
  :args (s/cat :part ::positive-number
               :total ::positive-number)
  :ret ::percentage)
```

---

## ขั้นตอนที่ 157: Functional Error Handling - Either Pattern

```clojure
;; Either/Result pattern สำหรับ functional error handling

;; สร้าง result type
(defn success [value] {:status :success :value value})
(defn failure [error] {:status :failure :error error})
(defn success? [{:keys [status]}] (= status :success))
(defn failure? [{:keys [status]}] (= status :failure))

;; Bind/chain operations
(defn bind [result f]
  (if (success? result)
    (f (:value result))
    result))

(defn fmap [result f]
  (if (success? result)
    (success (f (:value result)))
    result))

;; ใช้งาน
(defn parse-age [s]
  (try
    (let [n (Integer/parseInt s)]
      (if (pos? n)
        (success n)
        (failure "Age must be positive")))
    (catch NumberFormatException _
      (failure (str "Not a number: " s)))))

(defn validate-adult [age]
  (if (>= age 18)
    (success age)
    (failure "Must be 18 or older")))

(defn create-adult-account [age-str]
  (-> (parse-age age-str)
      (bind validate-adult)
      (fmap #(hash-map :age % :account-created true))))

(create-adult-account "25")
;; => {:status :success, :value {:age 25, :account-created true}}

(create-adult-account "15")
;; => {:status :failure, :error "Must be 18 or older"}

(create-adult-account "abc")
;; => {:status :failure, :error "Not a number: abc"}
```

---

## ขั้นตอนที่ 158: Error Accumulation

```clojure
;; การสะสม errors แทนที่จะหยุดที่ error แรก

(defn validate-user-form [form]
  (let [errors (cond-> []
                 (empty? (:name form))
                 (conj {:field :name :error "Name is required"})
                 
                 (not (re-matches #".+@.+\..+" (:email form "")))
                 (conj {:field :email :error "Invalid email format"})
                 
                 (< (count (:password form "")) 8)
                 (conj {:field :password :error "Password too short"})
                 
                 (not= (:password form) (:confirm-password form))
                 (conj {:field :confirm-password :error "Passwords don't match"}))]
    (if (empty? errors)
      {:valid true}
      {:valid false :errors errors})))

(validate-user-form {:name "" :email "invalid" :password "123"})
;; => {:valid false
;;     :errors [{:field :name :error "Name is required"}
;;              {:field :email :error "Invalid email format"}
;;              {:field :password :error "Password too short"}
;;              {:field :confirm-password :error "Passwords don't match"}]}
```

---

## ขั้นตอนที่ 159: Retry Logic

```clojure
;; Retry ด้วย exponential backoff
(defn retry
  "เรียก f ซ้ำถ้า exception เกิด
  Options:
  - :max-retries  (default: 3)
  - :initial-delay (default: 1000ms)
  - :backoff-factor (default: 2)
  - :retry-on  - predicate บน exception (default: always retry)"
  [f & {:keys [max-retries initial-delay backoff-factor retry-on]
        :or {max-retries 3
             initial-delay 1000
             backoff-factor 2
             retry-on (constantly true)}}]
  (loop [attempts 0
         delay initial-delay]
    (let [result (try
                   {:success (f)}
                   (catch Exception e
                     {:error e}))]
      (if (:success result)
        (:success result)
        (let [e (:error result)]
          (if (and (< attempts max-retries)
                   (retry-on e))
            (do
              (Thread/sleep delay)
              (recur (inc attempts) (* delay backoff-factor)))
            (throw e)))))))

;; ใช้งาน
(retry
  (fn [] (api-call))
  :max-retries 5
  :initial-delay 500
  :retry-on #(instance? java.net.ConnectException %))
```

---

## ขั้นตอนที่ 160: Circuit Breaker Pattern

```clojure
;; Circuit Breaker ป้องกันการเรียก service ที่ล้มเหลวซ้ำๆ

(defn create-circuit-breaker
  [{:keys [failure-threshold reset-timeout]
    :or {failure-threshold 5
         reset-timeout 60000}}]
  (atom {:state :closed      ; :closed, :open, :half-open
         :failures 0
         :last-failure-time nil}))

(defn call-with-circuit-breaker [cb f]
  (let [{:keys [state failures last-failure-time]} @cb
        now (System/currentTimeMillis)]
    (cond
      ;; Open state: check if should reset
      (= state :open)
      (if (> (- now last-failure-time) 60000)
        (do
          (swap! cb assoc :state :half-open)
          (call-with-circuit-breaker cb f))
        (throw (ex-info "Circuit breaker is OPEN" {:state :open})))
      
      ;; Half-open or closed: try the call
      :else
      (try
        (let [result (f)]
          (swap! cb assoc :state :closed :failures 0)
          result)
        (catch Exception e
          (swap! cb (fn [cb-state]
                      (let [new-failures (inc (:failures cb-state))]
                        (assoc cb-state
                               :failures new-failures
                               :last-failure-time now
                               :state (if (>= new-failures 5) :open :closed)))))
          (throw e))))))

;; ใช้งาน
(def payment-cb (create-circuit-breaker {:failure-threshold 3}))

(defn process-payment [payment]
  (call-with-circuit-breaker payment-cb
    #(call-payment-service payment)))
```

---

## ขั้นตอนที่ 161: Timeout และ Deadline

```clojure
;; Future กับ timeout
(defn with-timeout [timeout-ms f]
  (let [fut (future (f))]
    (try
      (deref fut timeout-ms ::timeout)
      (catch Exception e
        (future-cancel fut)
        (throw e)))))

(def result
  (with-timeout 5000
    (fn []
      (Thread/sleep 3000)
      "Done!")))

;; ถ้าเกิน timeout
(def result
  (with-timeout 1000
    (fn []
      (Thread/sleep 3000)
      "Done!")))
;; result = ::timeout

;; Deadline pattern
(defn with-deadline [deadline-epoch f]
  (let [remaining (- deadline-epoch (System/currentTimeMillis))]
    (if (pos? remaining)
      (with-timeout remaining f)
      (throw (ex-info "Deadline exceeded" {:deadline deadline-epoch})))))
```

---

## ขั้นตอนที่ 162: Logging ด้วย timbre

```clojure
;; [com.taoensso/timbre "6.6.1"]
(require '[taoensso.timbre :as log])

;; Basic logging
(log/info "Server started on port" 8080)
(log/debug "Processing request" {:path "/api/users" :method :GET})
(log/warn "High memory usage" {:used-mb 450 :max-mb 512})
(log/error "Database error" {:query "SELECT..." :error "timeout"})

;; Log with exception
(try
  (risky-operation)
  (catch Exception e
    (log/error e "Operation failed" {:context "processing-payment"})))

;; Log levels: trace, debug, info, warn, error, fatal

;; Configuration
(log/set-config!
  {:min-level :info
   :appenders {:spit (log/spit-appender {:fname "app.log"})
               :println (log/println-appender)}})

;; Structured logging
(log/info ::user-login {:user-id 1 :ip "192.168.1.1" :timestamp (System/currentTimeMillis)})

;; Context-aware logging
(log/with-context {:request-id "abc123" :user-id 42}
  (log/info "Processing order")
  (log/info "Order processed" {:order-id 789}))
```

---

## ขั้นตอนที่ 163: Control Flow กับ Atoms

```clojure
;; Atoms สำหรับ state transitions
(def application-state (atom :starting))

(defn valid-transition? [from to]
  (case [from to]
    ([:starting :running]
     [:running :stopping]
     [:stopping :stopped]
     [:running :errored]
     [:errored :stopping]) true
    false))

(defn transition-state! [new-state]
  (let [success? (atom false)]
    (swap! application-state
           (fn [current]
             (if (valid-transition? current new-state)
               (do (reset! success? true)
                   new-state)
               current)))
    @success?))

(transition-state! :running)  ; => true
(transition-state! :stopped)  ; => false (invalid transition)
@application-state            ; => :running
```

---

## ขั้นตอนที่ 164: Conditional Threading

```clojure
;; cond-> (conditional threading)
(defn build-query [filters]
  (cond-> {:query "SELECT * FROM users"}
    (:active filters)
    (update :query str " WHERE active = true")
    
    (:email filters)
    (-> (update :query str " AND email = ?")
        (update :params (fnil conj []) (:email filters)))
    
    (:limit filters)
    (update :query str " LIMIT " (:limit filters))))

(build-query {:active true :email "test@test.com" :limit 10})
;; => {:query "SELECT * FROM users WHERE active = true AND email = ? LIMIT 10"
;;     :params ["test@test.com"]}

;; cond->> 
(defn build-pipeline [opts]
  (cond->> data
    (:filter opts) (filter (:filter opts))
    (:transform opts) (map (:transform opts))
    (:sort opts) (sort-by (:sort opts))
    (:limit opts) (take (:limit opts))))

;; some->  (stop on nil)
(defn get-user-city [user-id]
  (some-> (find-user user-id)
          :address
          :city
          str/upper-case))
;; คืน nil ถ้า user ไม่มี หรือ ไม่มี address
```

---

## ขั้นตอนที่ 165: Dynamic Control Flow ด้วย Macros

```clojure
;; เขียน control flow macros เอง

;; repeat-until
(defmacro repeat-until [condition & body]
  `(loop []
     ~@body
     (when-not ~condition
       (recur))))

;; while
(defmacro while [condition & body]
  `(loop []
     (when ~condition
       ~@body
       (recur))))

;; with-retry
(defmacro with-retry [n & body]
  `(loop [attempts# ~n]
     (try
       ~@body
       (catch Exception e#
         (if (pos? attempts#)
           (recur (dec attempts#))
           (throw e#))))))

;; ใช้งาน
(with-retry 3
  (potentially-failing-operation))
```

---

## ขั้นตอนที่ 166: Resource Management

```clojure
;; with-open - auto close resources
(with-open [reader (io/reader "file.txt")]
  (doseq [line (line-seq reader)]
    (println line)))
;; reader ถูกปิดอัตโนมัติ

;; with-open หลาย resources
(with-open [in (io/reader "input.txt")
            out (io/writer "output.txt")]
  (doseq [line (line-seq in)]
    (.write out (str/upper-case line))
    (.newLine out)))

;; JDBC connection management
(with-open [conn (jdbc/get-connection db-spec)]
  (jdbc/execute! conn ["SELECT * FROM users"]))

;; สร้าง macro สำหรับ resource management
(defmacro with-resource [binding & body]
  `(let [~(first binding) ~(second binding)]
     (try
       ~@body
       (finally
         (.close ~(first binding))))))

(with-resource [resource (create-resource)]
  (use-resource resource))
```

---

## ขั้นตอนที่ 167: Error Recovery Strategies

```clojure
;; Strategy 1: Default Values
(defn safe-divide [a b]
  (if (zero? b) 0 (/ a b)))

;; Strategy 2: Optional/Maybe
(defn find-user [id]
  (try
    (some-> (query-db id) first)
    (catch Exception _ nil)))

;; Strategy 3: Fallback Chain
(defn get-config-value [key]
  (or (System/getenv (name key))
      (get env-config key)
      (get default-config key)
      (throw (ex-info "Config not found" {:key key}))))

;; Strategy 4: Supervisor pattern
(defn supervised-task [task-fn supervisor-fn]
  (try
    (task-fn)
    (catch Exception e
      (supervisor-fn e))))

;; Strategy 5: Compensating transactions
(defn create-order! [order]
  (let [saved-order (save-order! order)]
    (try
      (charge-payment! (:payment order))
      (reduce-inventory! (:items order))
      saved-order
      (catch Exception e
        ;; Compensate: undo previous steps
        (delete-order! (:id saved-order))
        (throw (ex-info "Order creation failed"
                        {:reason "Payment or inventory failed"}
                        e))))))
```

---

## ขั้นตอนที่ 168: Short-Circuit Evaluation

```clojure
;; and/or ใน Clojure short-circuit!

;; ใช้ and สำหรับ nil-safe chain
(and user
     (:address user)
     (:city (:address user)))
;; คืน city หรือ nil/false ที่ first falsy

;; ใช้ or สำหรับ fallback
(or cached-value
    (fetch-from-db key)
    default-value)
;; คืนค่าแรกที่ truthy

;; Guard clauses ด้วย short-circuit
(defn process [user data]
  (or
   (when-not user {:error "No user"})
   (when-not data {:error "No data"})
   (when-not (valid? data) {:error "Invalid data"})
   (do-actual-processing user data)))

;; some-> เป็น nil-safe chain
(some-> user
        :address
        :city
        str/upper-case)
```

---

## ขั้นตอนที่ 169: Transaction-like Patterns

```clojure
;; Clojure STM (Software Transactional Memory)
;; ใช้ ref สำหรับ coordinated state changes

(def account-a (ref 1000))
(def account-b (ref 500))

;; Transfer ใน transaction
(defn transfer! [from to amount]
  (dosync
   (when (< @from amount)
     (throw (ex-info "Insufficient funds"
                     {:balance @from :requested amount})))
   (alter from - amount)
   (alter to + amount)))

;; Transaction รับประกัน:
;; 1. Atomic: ทั้งหมดสำเร็จหรือไม่สำเร็จเลย
;; 2. Consistent: state valid เสมอ
;; 3. Isolated: transactions ไม่เห็น partial updates ของกัน

(transfer! account-a account-b 200)
@account-a  ; => 800
@account-b  ; => 700

;; Retry อัตโนมัติ
;; ถ้า transaction conflict, STM จะ retry อัตโนมัติ!
```

---

## ขั้นตอนที่ 170: Async Error Handling

```clojure
;; future กับ error handling
(def f (future
         (Thread/sleep 1000)
         (/ 1 0)))  ; จะ throw!

;; future-cancel
(future-cancel f)

;; deref กับ timeout
(deref f 2000 :timeout)

;; catch exceptions จาก futures
(try
  (deref f 2000 :timeout)
  (catch ExecutionException e
    (println "Future threw:" (.getCause e))))

;; promise กับ error handling
(def p (promise))

(future
  (try
    (deliver p {:result (risky-operation)})
    (catch Exception e
      (deliver p {:error (.getMessage e)}))))

(let [{:keys [result error]} @p]
  (if error
    (handle-error error)
    (handle-result result)))
```

---

## ขั้นตอนที่ 171: Spec-based Validation

```clojure
(require '[clojure.spec.alpha :as s])

;; Define specs
(s/def ::email (s/and string? #(re-matches #".+@.+\..+" %)))
(s/def ::age (s/and int? #(> % 0) #(< % 150)))
(s/def ::name (s/and string? #(> (count %) 0)))
(s/def ::role #{:admin :user :moderator})

(s/def ::user
  (s/keys :req-un [::name ::email ::age]
          :opt-un [::role]))

;; Validate
(s/valid? ::user {:name "สมชาย" :email "test@test.com" :age 25})
;; => true

(s/valid? ::user {:name "" :email "invalid" :age -1})
;; => false

;; Explain errors
(s/explain ::user {:name "" :email "invalid" :age -1})
;; In: [:name] val: "" fails spec: ...
;; In: [:email] val: "invalid" fails spec: ...
;; In: [:age] val: -1 fails spec: ...

;; Get structured error data
(s/explain-data ::user {:name "" :email "invalid"})
;; => {:clojure.spec.alpha/problems [{...} {...}]}

;; สร้าง validator function
(defn validate [spec data]
  (if (s/valid? spec data)
    {:valid true :data data}
    {:valid false
     :errors (::s/problems (s/explain-data spec data))}))
```

---

## ขั้นตอนที่ 172: Malli Validation

```clojure
;; [metosin/malli "0.16.4"]
(require '[malli.core :as m]
         '[malli.error :as me])

;; Define schema
(def UserSchema
  [:map
   [:name [:string {:min 1}]]
   [:email [:re #".+@.+\..+"]]
   [:age [:int {:min 0 :max 150}]]
   [:role {:optional true}
    [:enum :admin :user :moderator]]])

;; Validate
(m/validate UserSchema {:name "สมชาย" :email "test@test.com" :age 25})
;; => true

(m/validate UserSchema {:name "" :email "invalid" :age -1})
;; => false

;; Human-readable errors
(-> UserSchema
    (m/explain {:name "" :email "invalid" :age -1})
    me/humanize)
;; => {:name ["should be at least 1 characters"]
;;     :email ["should match pattern \".+@.+\\.+\""]
;;     :age ["should be at least 0"]}

;; Transform and validate
(require '[malli.transform :as mt])

(def CoerceUserSchema
  [:map
   [:name :string]
   [:age [:int {:decode/string #(Integer/parseInt %)
                :decode/json #(int %)}]]])

(m/decode CoerceUserSchema
          {:name "สมชาย" :age "25"}
          mt/string-transformer)
;; => {:name "สมชาย" :age 25}
```

---

## ขั้นตอนที่ 173: Guard Clauses Pattern

```clojure
;; Guard clauses: ตรวจ conditions ที่ top ก่อน
;; แทนที่จะ nest ลึก

;; ❌ Deep nesting
(defn process-order [order]
  (if (valid? order)
    (if (user-exists? (:user-id order))
      (if (items-available? (:items order))
        (if (payment-valid? (:payment order))
          (create-order! order)
          {:error "Payment failed"})
        {:error "Items not available"})
      {:error "User not found"})
    {:error "Invalid order"}))

;; ✅ Guard clauses
(defn process-order [order]
  (cond
    (not (valid? order))
    {:error "Invalid order"}
    
    (not (user-exists? (:user-id order)))
    {:error "User not found"}
    
    (not (items-available? (:items order)))
    {:error "Items not available"}
    
    (not (payment-valid? (:payment order)))
    {:error "Payment failed"}
    
    :else
    (create-order! order)))

;; หรือ ด้วย or
(defn process-order [order]
  (or
   (when-not (valid? order) {:error "Invalid order"})
   (when-not (user-exists? (:user-id order)) {:error "User not found"})
   (when-not (items-available? (:items order)) {:error "Items not available"})
   (create-order! order)))
```

---

## ขั้นตอนที่ 174: Finite State Machines

```clojure
;; FSM ด้วย Clojure

(def order-transitions
  {:pending   {:confirm :confirmed
               :cancel  :cancelled}
   :confirmed {:ship    :shipped
               :cancel  :cancelled}
   :shipped   {:deliver :delivered}
   :delivered {}
   :cancelled {}})

(defn transition [fsm state event]
  (if-let [next-state (get-in fsm [state event])]
    {:ok true :state next-state}
    {:ok false :error (str "Invalid transition: " state " + " event)}))

;; ใช้งาน
(def order (atom {:id 1 :state :pending}))

(defn order-event! [event]
  (let [result (transition order-transitions (:state @order) event)]
    (if (:ok result)
      (swap! order assoc :state (:state result))
      (println "Error:" (:error result)))))

(order-event! :confirm)
;; @order => {:id 1, :state :confirmed}

(order-event! :cancel)
;; @order => {:id 1, :state :cancelled}

(order-event! :ship)
;; Error: Invalid transition: cancelled + ship
```

---

## ขั้นตอนที่ 175: Error Middleware Pattern

```clojure
;; HTTP middleware สำหรับ error handling

(defn wrap-exception-handler [handler]
  (fn [request]
    (try
      (handler request)
      (catch clojure.lang.ExceptionInfo e
        (let [{:keys [type]} (ex-data e)]
          (case type
            :validation/error
            {:status 400
             :headers {"Content-Type" "application/json"}
             :body (json/write-str {:error (ex-message e)
                                   :details (ex-data e)})}
            
            :auth/unauthorized
            {:status 401
             :body (json/write-str {:error "Unauthorized"})}
            
            :resource/not-found
            {:status 404
             :body (json/write-str {:error (ex-message e)})}
            
            {:status 500
             :body (json/write-str {:error "Internal Server Error"})})))
      
      (catch Exception e
        (log/error e "Unhandled exception")
        {:status 500
         :body (json/write-str {:error "Internal Server Error"})}))))
```

---

## ขั้นตอนที่ 176: Defensive Programming

```clojure
;; Input validation สำหรับ external data

(defn safe-parse-int [s]
  (when (string? s)
    (try
      (Integer/parseInt (str/trim s))
      (catch NumberFormatException _ nil))))

(defn safe-get-in [m ks & [default]]
  (let [v (get-in m ks ::not-found)]
    (if (= v ::not-found) default v)))

;; Defensive map access
(defn get-user-name [response]
  (or (get-in response [:data :user :name])
      (get-in response [:user :name])
      (get response :name)
      "Unknown"))

;; Sanitize input
(defn sanitize-string [s max-length]
  (when (string? s)
    (-> s
        str/trim
        (subs 0 (min (count s) max-length)))))

(defn sanitize-user-input [data]
  {:name (sanitize-string (:name data) 100)
   :email (some-> (:email data) str/trim str/lower-case)
   :age (safe-parse-int (:age data))})
```

---

## ขั้นตอนที่ 177: Error Reporting

```clojure
;; Sentry integration
;; [io.sentry/sentry-clj "7.14.0"]

(require '[sentry-clj.core :as sentry])

;; Initialize
(sentry/init! {:dsn (System/getenv "SENTRY_DSN")
               :environment "production"
               :release "v1.0.0"})

;; Capture exception
(try
  (risky-operation)
  (catch Exception e
    (sentry/send-event
     {:throwable e
      :extra {:user-id (:id current-user)
              :operation "risky-operation"}})))

;; Capture message
(sentry/send-event
 {:message "Payment timeout"
  :level :warning
  :extra {:payment-id "abc123"
          :amount 1000}})

;; Breadcrumbs
(sentry/add-breadcrumb!
 {:type "info"
  :category "order"
  :message "Processing order"
  :data {:order-id 123}})
```

---

## ขั้นตอนที่ 178: Testing Error Paths

```clojure
(require '[clojure.test :refer :all])

;; Test exception throwing
(deftest test-divide
  (testing "normal division"
    (is (= 5 (divide 10 2))))
  
  (testing "division by zero throws"
    (is (thrown? ArithmeticException
                 (divide 10 0))))
  
  (testing "division by zero has right message"
    (is (thrown-with-msg? ArithmeticException
                          #"Cannot divide by zero"
                          (divide 10 0)))))

;; Test ex-info
(deftest test-validation
  (testing "invalid user throws with data"
    (try
      (create-user! {:name "" :email "invalid"})
      (is false "Should have thrown")
      (catch clojure.lang.ExceptionInfo e
        (is (= :validation-error (:type (ex-data e))))
        (is (seq (:errors (ex-data e))))))))

;; Matchers สำหรับ exception data
(defn thrown-with-data? [expected-data thunk]
  (try
    (thunk)
    false
    (catch clojure.lang.ExceptionInfo e
      (= expected-data (ex-data e)))))

(is (thrown-with-data? {:type :not-found}
      #(find-user! 99999)))
```

---

## ขั้นตอนที่ 179: Monitoring และ Alerting

```clojure
;; Health check endpoints
(defn health-check [db redis]
  {:status :ok
   :checks
   {:database
    (try
      (jdbc/execute-one! db ["SELECT 1"])
      {:status :healthy}
      (catch Exception e
        {:status :unhealthy :error (.getMessage e)}))
    
    :redis
    (try
      (redis/ping redis)
      {:status :healthy}
      (catch Exception e
        {:status :unhealthy :error (.getMessage e)}))
    
    :memory
    (let [runtime (Runtime/getRuntime)
          used (- (.totalMemory runtime) (.freeMemory runtime))
          max (.maxMemory runtime)
          percent (* 100 (/ used max))]
      (if (< percent 90)
        {:status :healthy :used-percent percent}
        {:status :warning :used-percent percent}))}})

;; Metrics
(defn record-metric! [metric-name value tags]
  (statsd/gauge metric-name value tags))

(defn timed [metric-name f]
  (let [start (System/currentTimeMillis)
        result (f)
        elapsed (- (System/currentTimeMillis) start)]
    (record-metric! metric-name elapsed {})
    result))
```

---

## ขั้นตอนที่ 180: Summary - Error Handling Best Practices

```clojure
;; === Best Practices ===

;; 1. ใช้ ex-info สำหรับ business errors
(throw (ex-info "User not found" {:type :not-found :id user-id}))

;; 2. แยก error types ด้วย keywords
(defn ->error [type message & [data]]
  (ex-info message (merge {:type type} data)))

;; 3. Handle errors ที่ edges (handlers/routes) ไม่ใช่ deep ใน logic
(defn api-handler [request]
  (try
    (process-business-logic request)
    (catch clojure.lang.ExceptionInfo e
      (error-response e))))

;; 4. ใช้ Either/Result pattern สำหรับ expected failures
(defn parse-user-input [data]
  (if (valid? data)
    {:ok data}
    {:error "Invalid input" :details (validate data)}))

;; 5. Log ก่อน rethrow
(catch Exception e
  (log/error e "Unexpected error" {:context "..."})
  (throw e))

;; 6. Never swallow exceptions
;; ❌ (catch Exception e nil)
;; ✅ (catch Exception e (log/error e "...") (handle-gracefully))

;; 7. Test error paths ด้วย thrown? และ thrown-with-msg?
;; 8. Use circuit breakers สำหรับ external services
;; 9. Retry กับ exponential backoff สำหรับ transient errors
;; 10. Monitor และ alert บน critical errors
```

---

### Project Exercise: Transaction Processing System

```clojure
;; สร้าง robust transaction processing system

(ns transaction.core
  (:require [clojure.spec.alpha :as s]))

;; Specs
(s/def ::amount (s/and number? pos?))
(s/def ::account-id pos-int?)
(s/def ::transaction-type #{:deposit :withdrawal :transfer})

(s/def ::transaction
  (s/keys :req-un [::amount ::account-id ::transaction-type]))

;; State
(def accounts (atom {1 {:id 1 :name "สมชาย" :balance 10000}
                     2 {:id 2 :name "สมหญิง" :balance 5000}}))

;; Operations
(defn deposit! [account-id amount]
  (swap! accounts update-in [account-id :balance] + amount)
  {:success true :new-balance (get-in @accounts [account-id :balance])})

(defn withdrawal! [account-id amount]
  (let [balance (get-in @accounts [account-id :balance])]
    (if (>= balance amount)
      (do
        (swap! accounts update-in [account-id :balance] - amount)
        {:success true :new-balance (get-in @accounts [account-id :balance])})
      {:success false :error "Insufficient funds" :balance balance})))

(defn transfer! [from-id to-id amount]
  (let [result (withdrawal! from-id amount)]
    (if (:success result)
      (do
        (deposit! to-id amount)
        {:success true
         :from-balance (get-in @accounts [from-id :balance])
         :to-balance (get-in @accounts [to-id :balance])})
      result)))

(defn process-transaction [tx]
  (if (s/valid? ::transaction tx)
    (case (:transaction-type tx)
      :deposit    (deposit! (:account-id tx) (:amount tx))
      :withdrawal (withdrawal! (:account-id tx) (:amount tx))
      :transfer   (transfer! (:account-id tx) (:to-account-id tx) (:amount tx)))
    {:success false :error "Invalid transaction" 
     :details (s/explain-data ::transaction tx)}))

;; Test
(comment
  (process-transaction {:account-id 1 :amount 500 :transaction-type :deposit})
  (process-transaction {:account-id 2 :amount 10000 :transaction-type :withdrawal})
  (process-transaction {:account-id 1 :amount 1000 :transaction-type :transfer
                        :to-account-id 2}))
```

---

### อ่านต่อใน Part 7: Sequences และ Lazy Evaluation ขั้นสูง →

---

*Part 6 จาก 100+ | ขั้นตอน 151-180 จาก 1000+*
