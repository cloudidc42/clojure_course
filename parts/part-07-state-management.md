# Part 7: State Management
## ขั้นตอนที่ 181-210: Atoms, Refs, Agents, Vars

---

## บทนำ

ใน Clojure "immutable by default" แต่โลกจริงต้องการ state ที่เปลี่ยนได้ Clojure มี 4 reference types ที่แต่ละอันเหมาะกับสถานการณ์ต่างกัน:

| Reference Type | Use Case | Thread Safety |
|---------------|----------|---------------|
| **Atom**      | Independent state | CAS (Compare-and-Swap) |
| **Ref**       | Coordinated state | STM (Software Transactional Memory) |
| **Agent**     | Async state changes | Actor model |
| **Var**       | Thread-local state | Dynamic binding |

---

## ขั้นตอนที่ 181: Atoms - Independent Mutable State

```clojure
;; สร้าง atom
(def counter (atom 0))
(def user-state (atom {:name "สมชาย" :age 25}))
(def items (atom []))

;; อ่านค่า ด้วย @ หรือ deref
@counter         ; => 0
(deref counter)  ; => 0 (เหมือนกัน)
@user-state      ; => {:name "สมชาย", :age 25}

;; เปลี่ยนค่า
;; swap! - apply function ต่อ current value
(swap! counter inc)        ; => 1
(swap! counter + 5)        ; => 6
(swap! counter * 2)        ; => 12

;; reset! - set ค่าใหม่โดยตรง
(reset! counter 0)         ; => 0
@counter                   ; => 0

;; swap! กับ complex state
(swap! user-state assoc :email "somchai@test.com")
(swap! user-state update :age inc)
(swap! user-state (fn [state]
                    (-> state
                        (assoc :processed true)
                        (update :age inc))))

;; swap-vals! - คืน [old-val new-val]
(swap-vals! counter inc)   ; => [0 1]

;; compare-and-set! - conditional update
(compare-and-set! counter 1 42)  ; true ถ้า current = 1, เปลี่ยนเป็น 42
(compare-and-set! counter 0 99)  ; false ถ้า current ≠ 0
```

---

## ขั้นตอนที่ 182: Atom Watch Functions

```clojure
;; add-watch - ดูการเปลี่ยนแปลง
(def app-state (atom {:users [] :loaded false}))

;; watch function: key, reference, old-val, new-val
(add-watch app-state :logger
  (fn [key ref old-val new-val]
    (println "State changed!")
    (println "Old:" (keys old-val))
    (println "New:" (keys new-val))))

(swap! app-state assoc :loaded true)
;; State changed!
;; Old: (:users :loaded)
;; New: (:users :loaded)

;; Remove watch
(remove-watch app-state :logger)

;; Practical: sync state กับ localStorage (ClojureScript)
(add-watch app-state :persist
  (fn [_ _ _ new-state]
    (js/localStorage.setItem "app-state"
                              (pr-str new-state))))

;; Practical: metrics collection
(def requests (atom 0))
(def errors (atom 0))

(add-watch errors :error-rate-monitor
  (fn [_ _ _ new-val]
    (when (> (/ new-val @requests 1.0) 0.1)
      (alert! "Error rate > 10%!"))))
```

---

## ขั้นตอนที่ 183: Atom Patterns

```clojure
;; Pattern 1: Counter
(def counter (atom 0))
(defn next-id! [] (swap! counter inc))

;; Pattern 2: Cache
(def cache (atom {}))

(defn get-cached [key fetch-fn]
  (or (get @cache key)
      (let [value (fetch-fn)]
        (swap! cache assoc key value)
        value)))

;; Pattern 3: Event log
(def event-log (atom []))

(defn log-event! [event]
  (swap! event-log conj (assoc event :timestamp (System/currentTimeMillis))))

;; Pattern 4: Pub/Sub
(def subscribers (atom {}))

(defn subscribe! [topic handler]
  (swap! subscribers update topic (fnil conj []) handler))

(defn publish! [topic data]
  (doseq [handler (get @subscribers topic [])]
    (handler data)))

;; Pattern 5: State Machine
(def order-state (atom :pending))

(defn transition! [from to]
  (compare-and-set! order-state from to))

(transition! :pending :confirmed)  ; true
(transition! :pending :shipped)    ; false (already :confirmed)
```

---

## ขั้นตอนที่ 184: Refs และ STM

```clojure
;; Refs สำหรับ Coordinated State Changes
;; (หลาย states ต้องเปลี่ยนพร้อมกัน atomic)

;; สร้าง refs
(def account-a (ref {:id 1 :balance 1000}))
(def account-b (ref {:id 2 :balance 500}))
(def transaction-log (ref []))

;; อ่านค่า
@account-a  ; => {:id 1, :balance 1000}

;; เปลี่ยนค่าต้องอยู่ใน dosync!
(dosync
  ;; alter - เหมือน swap! สำหรับ refs
  (alter account-a update :balance - 200)
  (alter account-b update :balance + 200)
  (alter transaction-log conj {:from 1 :to 2 :amount 200 :time (System/currentTimeMillis)}))

@account-a  ; => {:id 1, :balance 800}
@account-b  ; => {:id 2, :balance 700}
@transaction-log  ; => [{:from 1, :to 2, :amount 200, ...}]

;; ref-set - set ค่าโดยตรง (ใน dosync)
(dosync
  (ref-set account-a {:id 1 :balance 0}))

;; commute - สำหรับ operations ที่ commutative (เร็วกว่า alter)
;; ใช้เมื่อ order ไม่สำคัญ
(dosync
  (commute transaction-log conj {:event "audit"}))
```

---

## ขั้นตอนที่ 185: STM ขั้นสูง

```clojure
;; ensure - prevent write skew
(def seats-available (ref 5))
(def bookings (ref []))

;; Without ensure (buggy):
;; Thread A reads seats-available = 5, creates booking
;; Thread B reads seats-available = 5, creates booking
;; Both commit → overbooking!

;; With ensure:
(defn book-seat! [passenger-name]
  (dosync
    (ensure seats-available)  ; กัน write skew
    (when (> @seats-available 0)
      (alter seats-available dec)
      (alter bookings conj {:passenger passenger-name
                            :booked-at (System/currentTimeMillis)})
      true)))

;; io! - ระบุว่า side effect ไม่ควรเกิดใน transaction
;; dosync อาจ retry หลายครั้ง! อย่าทำ IO ใน dosync!
(dosync
  (io! (println "This will warn - don't do IO in dosync!"))
  (alter counter inc))

;; Correct: เก็บผลลัพธ์แล้วทำ IO ข้างนอก
(let [result (dosync (alter counter inc))]
  (println "Counter is now:" result))
```

---

## ขั้นตอนที่ 186: Agents - Async State

```clojure
;; Agent สำหรับ asynchronous state updates

(def async-counter (agent 0))
(def email-queue (agent []))

;; อ่านค่า
@async-counter   ; => 0

;; send - async dispatch (uses pool of threads)
(send async-counter inc)
(send async-counter + 10)
(send async-counter * 2)

;; ค่าจะอัพเดทแบบ async!
@async-counter   ; อาจยังเป็น 0 หรือค่า intermediate

;; await - รอให้ทุก pending actions เสร็จ
(await async-counter)
@async-counter   ; => 22 (0 + 1 + 10 = 11, * 2 = 22)

;; send-off - สำหรับ blocking operations (เช่น IO)
(def file-writer (agent []))

(defn write-to-file! [agent-state line]
  (spit "output.txt" (str line "\n") :append true)
  (conj agent-state line))

(send-off file-writer write-to-file! "Line 1")
(send-off file-writer write-to-file! "Line 2")

(await file-writer)
```

---

## ขั้นตอนที่ 187: Agent Error Handling

```clojure
;; Agent errors
(def broken-agent (agent 0))

(send broken-agent (fn [_] (throw (Exception. "Oops!"))))

;; Agent จะเข้า error state
(agent-error broken-agent)
;; => #<Exception java.lang.Exception: Oops!>

;; ต้อง restart agent
(restart-agent broken-agent 0)
@broken-agent   ; => 0

;; Error handler
(def safe-agent (agent 0
  :error-handler (fn [agent exception]
                   (println "Agent error:" (.getMessage exception))
                   ;; ไม่ต้อง restart เพราะ error-handler จัดการแล้ว
                   )))

(send safe-agent (fn [_] (throw (Exception. "Test"))))
;; prints: Agent error: Test
;; agent ยังทำงานได้ปกติ!

;; Error mode
(def safe-agent2 (agent 0
  :error-mode :continue))
;; :continue = ข้ามเมื่อ error (default: :fail)
```

---

## ขั้นตอนที่ 188: Vars และ Dynamic Binding

```clojure
;; Dynamic vars สำหรับ thread-local state

(def ^:dynamic *database* nil)
(def ^:dynamic *current-user* nil)
(def ^:dynamic *request-id* nil)

;; Functions ใช้ dynamic vars
(defn get-user [id]
  (query *database* "SELECT * FROM users WHERE id = ?" [id]))

(defn get-current-user []
  @*current-user*)

;; ตั้งค่าด้วย binding
(binding [*database* production-db
          *current-user* {:id 1 :name "Admin"}
          *request-id* "req-abc123"]
  (do
    (get-user 1)
    (get-current-user)))

;; binding ใน threads
(def result (atom nil))

(binding [*current-user* {:id 42}]
  (future
    ;; binding propagates to child threads (Clojure 1.3+)
    (reset! result @*current-user*)))

(Thread/sleep 100)
@result  ; => {:id 42}

;; with-bindings - programmatic binding
(with-bindings {#'*database* test-db
                #'*current-user* test-user}
  (run-tests))
```

---

## ขั้นตอนที่ 189: State Patterns ขั้นสูง

```clojure
;; === Pattern: Immutable State with History ===

(def history (atom []))
(def current-state (atom {}))

(defn update-state! [f]
  (let [old-state @current-state
        new-state (swap! current-state f)]
    (swap! history conj {:old old-state
                         :new new-state
                         :timestamp (System/currentTimeMillis)})
    new-state))

(defn undo! []
  (when-let [last-entry (last @history)]
    (reset! current-state (:old last-entry))
    (swap! history (comp vec butlast))
    @current-state))

;; ใช้งาน
(update-state! #(assoc % :name "สมชาย"))
(update-state! #(assoc % :age 25))
(update-state! #(assoc % :age 26))

@current-state  ; => {:name "สมชาย", :age 26}
(undo!)         ; => {:name "สมชาย", :age 25}
(undo!)         ; => {:name "สมชาย"}
```

---

## ขั้นตอนที่ 190: Redux-like State Management

```clojure
;; Redux pattern สำหรับ UI state (คล้าย ClojureScript + Re-frame)

;; State
(def app-db
  (atom {:users []
         :loading false
         :error nil
         :selected-user nil}))

;; Actions (events)
(defn fetch-users-start []
  {:type :fetch-users/start})

(defn fetch-users-success [users]
  {:type :fetch-users/success :users users})

(defn fetch-users-error [error]
  {:type :fetch-users/error :error error})

(defn select-user [user-id]
  {:type :select-user :id user-id})

;; Reducers (pure functions)
(defmulti reducer (fn [state event] (:type event)))

(defmethod reducer :fetch-users/start [state _]
  (assoc state :loading true :error nil))

(defmethod reducer :fetch-users/success [state {:keys [users]}]
  (assoc state :loading false :users users))

(defmethod reducer :fetch-users/error [state {:keys [error]}]
  (assoc state :loading false :error error))

(defmethod reducer :select-user [state {:keys [id]}]
  (assoc state :selected-user (first (filter #(= id (:id %)) (:users state)))))

(defmethod reducer :default [state _] state)

;; Dispatch
(defn dispatch! [event]
  (swap! app-db reducer event))

;; ใช้งาน
(dispatch! (fetch-users-start))
(dispatch! (fetch-users-success [{:id 1 :name "สมชาย"} {:id 2 :name "สมหญิง"}]))
(dispatch! (select-user 1))

@app-db
;; => {:users [{...} {...}] :loading false :error nil :selected-user {:id 1 ...}}
```

---

## ขั้นตอนที่ 191: Concurrency Patterns

```clojure
;; Future - one-off async computation
(def result
  (future
    (Thread/sleep 2000)
    (+ 1 2 3)))

;; ทำงานอื่นไปก่อน...
(println "Doing other work...")

;; รอผลลัพธ์
@result   ; blocks จนได้ผล
;; => 6

;; realized? - check ว่าเสร็จหรือยัง
(realized? result)  ; => false/true

;; Promise - one-time delivery channel
(def p (promise))

(future
  (Thread/sleep 1000)
  (deliver p "Hello from future!"))

;; รอรับ
@p  ; => "Hello from future!"

;; Promise as callback
(defn async-operation [callback]
  (future
    (Thread/sleep 1000)
    (callback "result")))

;; Better: return promise
(defn async-operation-promise []
  (let [p (promise)]
    (future
      (Thread/sleep 1000)
      (deliver p "result"))
    p))

(let [result (async-operation-promise)]
  ;; ทำงานอื่น...
  @result)  ; รับค่า
```

---

## ขั้นตอนที่ 192: Concurrent Data Processing

```clojure
;; pmap - parallel map
(defn slow-process [x]
  (Thread/sleep 100)
  (* x x))

;; Sequential
(time (doall (map slow-process (range 10))))
;; ~1000ms

;; Parallel!
(time (doall (pmap slow-process (range 10))))
;; ~100ms (10x faster!)

;; pmap ใช้ ForkJoinPool threads
;; เหมาะกับ CPU-intensive tasks

;; pcalls - call หลาย functions พร้อมกัน
(pcalls
  #(fetch-from-api-1)
  #(fetch-from-api-2)
  #(fetch-from-db))
;; ทุกอย่างทำพร้อมกัน!

;; pour - parallel into
;; ไม่มีใน core แต่ทำได้ด้วย:
(defn pmap-into [to xf from]
  (into to xf (pmap identity from)))

;; clojure.core.reducers (r/)
(require '[clojure.core.reducers :as r])

(time
  (->> (range 1000000)
       (r/filter even?)
       (r/map #(* % %))
       (r/fold +)))
;; ใช้ fork/join framework
```

---

## ขั้นตอนที่ 193: core.async เบื้องต้น

```clojure
;; [org.clojure/core.async "0.7.559"]
(require '[clojure.core.async :as async
           :refer [chan go go-loop <! >! <!! >!! close! put! take!]])

;; Channel - ท่อส่งข้อมูลระหว่าง goroutines
(def ch (chan))          ; unbuffered
(def ch (chan 10))       ; buffered (10 items)
(def ch (chan (async/dropping-buffer 10)))   ; drop new if full
(def ch (chan (async/sliding-buffer 10)))    ; drop old if full

;; go block - lightweight "goroutine" (ไม่ใช่ thread จริง!)
(go
  (println "Hello from go block!")
  (>! ch "message from go"))    ; non-blocking put

;; อ่านจาก channel
(go
  (let [msg (<! ch)]
    (println "Received:" msg)))

;; Blocking operations (ใช้นอก go block)
(>!! ch "sync put")    ; blocking put
(<!! ch)               ; blocking take

;; close channel
(close! ch)
;; อ่านจาก closed channel คืน nil
```

---

## ขั้นตอนที่ 194: core.async Patterns

```clojure
;; Pipeline pattern
(defn pipeline [input-ch process-fn output-ch]
  (go-loop []
    (when-let [item (<! input-ch)]
      (>! output-ch (process-fn item))
      (recur))))

;; Fan-out
(defn fan-out [source channels]
  (go-loop []
    (when-let [item (<! source)]
      (doseq [ch channels]
        (>! ch item))
      (recur))))

;; Fan-in (merge)
(defn fan-in [channels]
  (let [out (chan)]
    (doseq [ch channels]
      (go-loop []
        (when-let [val (<! ch)]
          (>! out val)
          (recur))))
    out))

;; timeout
(let [result (async/alt!
               (async/timeout 5000) "timeout"
               some-channel ([v] v))]
  (println result))

;; Practical: rate limiting
(defn rate-limited-channel [n ms]
  (let [ch (chan)]
    (go-loop []
      (>! ch :tick)
      (<! (async/timeout (/ ms n)))
      (recur))
    ch))
```

---

## ขั้นตอนที่ 195: Thread Safety ใน Clojure

```clojure
;; Clojure data structures เป็น thread-safe ทุกอย่าง!
;; ไม่ต้องทำ synchronization สำหรับ persistent collections

;; แต่ต้องระวังกับ reference types
;; Atom: CAS (thread-safe)
(def counter (atom 0))

;; concurrent increments
(let [futures (repeatedly 100 #(future (swap! counter inc)))]
  (run! deref futures))

@counter  ; => 100 (เสมอ! ไม่มี race condition)

;; ❌ ไม่ safe: read-modify-write ด้วย plain vars
;; (def counter 0)
;; (def counter (inc counter))  ; race condition!

;; ✅ Safe: ใช้ atom
(swap! counter inc)  ; atomic CAS

;; ❌ ไม่ safe: หลาย swap! ที่ต้องทำพร้อมกัน
;; (swap! counter1 inc)
;; (swap! counter2 inc)
;; ทำทีละอัน อาจ inconsistent

;; ✅ Safe: ใช้ refs + dosync
(dosync
  (alter ref1 inc)
  (alter ref2 inc))
```

---

## ขั้นตอนที่ 196: Volatile - Fast Mutable Value

```clojure
;; volatile! เร็วกว่า atom แต่ไม่ thread-safe!
;; ใช้เฉพาะใน single-thread context

(def v (volatile! 0))

@v           ; => 0
(vreset! v 42)   ; set value
(vswap! v inc)   ; apply function

;; Use case: local accumulator ใน transducer
(defn counting-transducer []
  (fn [rf]
    (let [count (volatile! 0)]
      (fn
        ([] (rf))
        ([result] (rf result))
        ([result input]
         (vswap! count inc)
         (rf result input))))))

;; Use case: loop counter ที่ไม่ต้องการ thread safety
(defn process-items [items]
  (let [errors (volatile! 0)]
    (doseq [item items]
      (try
        (process! item)
        (catch Exception _
          (vswap! errors inc))))
    @errors))
```

---

## ขั้นตอนที่ 197: Producer-Consumer Pattern

```clojure
;; Classic Producer-Consumer ด้วย core.async

(require '[clojure.core.async :as async])

(defn producer [ch items]
  (async/go
    (doseq [item items]
      (async/>! ch item)
      (println "Produced:" item))
    (async/close! ch)
    (println "Producer done")))

(defn consumer [ch process-fn]
  (async/go-loop []
    (when-let [item (async/<! ch)]
      (println "Consuming:" item)
      (process-fn item)
      (recur))
    (println "Consumer done")))

;; ใช้งาน
(let [ch (async/chan 5)]
  (producer ch (range 10))
  (consumer ch println)
  (Thread/sleep 1000))

;; Multiple consumers (load balancing)
(let [ch (async/chan 5)]
  (producer ch (range 100))
  (dotimes [i 3]
    (consumer ch #(println "Worker" i "processing:" %)))
  (Thread/sleep 2000))
```

---

## ขั้นตอนที่ 198: Observable State Pattern

```clojure
;; Observer pattern ด้วย atom watches

(defprotocol Observable
  (subscribe [this key handler])
  (unsubscribe [this key])
  (get-value [this]))

(defrecord ObservableAtom [state]
  Observable
  (subscribe [this key handler]
    (add-watch state key
      (fn [k ref old new]
        (handler {:key k :old old :new new})))
    this)
  
  (unsubscribe [this key]
    (remove-watch state key)
    this)
  
  (get-value [this]
    @state))

(defn observable [initial-value]
  (->ObservableAtom (atom initial-value)))

;; ใช้งาน
(def user (observable {:name "สมชาย" :age 25}))

(subscribe user :logger
  (fn [{:keys [old new]}]
    (println "Changed from" old "to" new)))

(swap! (:state user) update :age inc)
;; Changed from {:name "สมชาย", :age 25} to {:name "สมชาย", :age 26}
```

---

## ขั้นตอนที่ 199: Real-world Example - Session Management

```clojure
(ns session-manager.core)

;; In-memory session store
(def sessions (atom {}))

(defn generate-session-id []
  (str (java.util.UUID/randomUUID)))

(defn create-session! [user-id]
  (let [session-id (generate-session-id)
        session {:id session-id
                 :user-id user-id
                 :created-at (System/currentTimeMillis)
                 :last-active (System/currentTimeMillis)
                 :data {}}]
    (swap! sessions assoc session-id session)
    session-id))

(defn get-session [session-id]
  (when-let [session (get @sessions session-id)]
    ;; Update last-active
    (swap! sessions update-in [session-id :last-active]
           (constantly (System/currentTimeMillis)))
    session))

(defn update-session! [session-id key value]
  (swap! sessions assoc-in [session-id :data key] value))

(defn delete-session! [session-id]
  (swap! sessions dissoc session-id))

;; Session cleanup
(defn cleanup-expired-sessions! [max-age-ms]
  (let [now (System/currentTimeMillis)]
    (swap! sessions
           #(into {}
                  (filter (fn [[_ session]]
                            (< (- now (:last-active session)) max-age-ms))
                          %)))))

;; Schedule cleanup every 5 minutes
(defn start-cleanup-job! []
  (future
    (loop []
      (Thread/sleep (* 5 60 1000))
      (cleanup-expired-sessions! (* 30 60 1000))  ; 30 min timeout
      (recur))))
```

---

## ขั้นตอนที่ 200: Summary - State Management Guidelines

```
Reference Type Selection Guide:
================================
Use Atom when:
  ✓ Single state that changes independently
  ✓ Need thread-safe but simple updates
  ✓ Most common choice!
  
  Examples: counter, cache, application config, 
           user session, request state

Use Ref when:
  ✓ Multiple states that MUST change together atomically
  ✓ Financial transactions
  ✓ Booking systems (prevent overbooking)
  
  Examples: bank transfers, inventory management,
           ticket booking, database-like consistency

Use Agent when:
  ✓ Async state that doesn't need immediate feedback
  ✓ IO-bound state updates
  ✓ Background processing
  
  Examples: file writing, email queue, logging,
           metrics collection

Use Var (dynamic) when:
  ✓ Thread-local context
  ✓ Testing (mock dependencies)
  ✓ Request-scoped configuration
  
  Examples: database connection per request,
           current user context, test fixtures

===
Immutability Reminder:
- Default: use plain data (immutable)
- Need to change: use atoms (usually)
- Need coordination: use refs
- Need async: use agents
- Never mutate data structures directly!
```

---

### Project Exercise: Multi-user Chat System

```clojure
(ns chat.core
  (:require [clojure.core.async :as async]))

;; State
(def users (atom {}))      ; {session-id -> user-info}
(def rooms (atom {}))      ; {room-id -> #{session-ids}}
(def messages (atom {}))   ; {room-id -> [messages]}

;; User management
(defn join! [session-id username]
  (swap! users assoc session-id {:id session-id :username username :joined-at (System/currentTimeMillis)}))

(defn leave! [session-id]
  ;; Remove from all rooms
  (swap! rooms update-vals #(disj % session-id))
  (swap! users dissoc session-id))

;; Room management
(defn join-room! [session-id room-id]
  (swap! rooms update room-id (fnil conj #{}) session-id))

(defn leave-room! [session-id room-id]
  (swap! rooms update room-id disj session-id))

;; Messaging
(defn send-message! [session-id room-id content]
  (let [user (get @users session-id)
        message {:id (str (java.util.UUID/randomUUID))
                 :from (:username user)
                 :content content
                 :room room-id
                 :timestamp (System/currentTimeMillis)}]
    (swap! messages update room-id (fnil conj []) message)
    message))

(defn get-room-messages [room-id n]
  (take-last n (get @messages room-id [])))

;; ทดสอบ
(comment
  (join! "s1" "สมชาย")
  (join! "s2" "สมหญิง")
  
  (join-room! "s1" "general")
  (join-room! "s2" "general")
  
  (send-message! "s1" "general" "สวัสดีทุกคน!")
  (send-message! "s2" "general" "หวัดดีครับ!")
  
  (get-room-messages "general" 10))
```

---

### อ่านต่อใน Part 8: Concurrency ด้วย core.async →

---

*Part 7 จาก 100+ | ขั้นตอน 181-210 จาก 1000+*
