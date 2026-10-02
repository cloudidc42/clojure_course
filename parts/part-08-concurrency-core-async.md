# Part 8: Concurrency ด้วย core.async
## ขั้นตอนที่ 211-240: Channels, Go Blocks, Pipelines

---

## บทนำ

`core.async` นำ Communicating Sequential Processes (CSP) มาสู่ Clojure แทนที่จะแชร์ state ระหว่าง threads เราส่ง **messages ผ่าน channels** ซึ่งทำให้ code ปลอดภัยและเข้าใจง่ายกว่า

```
CSP Philosophy:
"Don't communicate by sharing memory; 
 share memory by communicating."
                    - Rob Pike (Go)
```

---

## ขั้นตอนที่ 211: Channel Fundamentals

```clojure
;; deps.edn
;; {:deps {org.clojure/core.async {:mvn/version "1.6.673"}}}

(require '[clojure.core.async :as async
           :refer [chan go go-loop <! >! <!! >!! close! put! take!
                   alts! alts!! timeout thread]])

;; ===== สร้าง Channels =====

;; 1. Unbuffered channel (rendezvous)
(def ch (chan))
;; ต้องมี sender AND receiver พร้อมกัน

;; 2. Buffered channel
(def buffered-ch (chan 10))   ; รอได้ 10 items

;; 3. Dropping buffer - ทิ้ง new items เมื่อเต็ม
(def dropping-ch (chan (async/dropping-buffer 5)))

;; 4. Sliding buffer - ทิ้ง old items เมื่อเต็ม (queue-like)
(def sliding-ch (chan (async/sliding-buffer 5)))

;; 5. Channel with transducer
(def xf-ch (chan 10 (comp (filter even?) (map #(* % %)))))

;; ===== Put และ Take =====
;; >! และ <! ใช้ใน go blocks (non-blocking)
;; >!! และ <!! ใช้นอก go blocks (blocking)

(go (>! ch "hello"))           ; async put
(go (println "got:" (<! ch)))  ; async take

(>!! buffered-ch "sync put")   ; blocking put
(<!! buffered-ch)              ; blocking take => "sync put"

;; channel ที่ถูก close จะคืน nil เมื่อ take
(close! ch)
(<!! ch)  ; => nil
```

---

## ขั้นตอนที่ 212: Go Blocks - Lightweight Concurrency

```clojure
;; go blocks ทำงานบน thread pool (ไม่ใช่ thread จริง!)
;; ประหยัดหน่วยความจำมาก - ใช้ได้หลายพัน go blocks

;; Simple go block
(go
  (println "Start")
  (<! (timeout 1000))   ; pause 1 second (non-blocking!)
  (println "After 1 second"))

;; go returns channel ที่มีผลลัพธ์
(def result-ch
  (go
    (let [x (<! (fetch-data-ch))
          y (<! (process-x-ch x))]
      {:result y :timestamp (System/currentTimeMillis)})))

(let [result (<!! result-ch)]
  (println "Result:" result))

;; go-loop - loop ที่ทำงานใน go block
(go-loop [count 0]
  (when (< count 5)
    (<! (timeout 500))
    (println "Count:" count)
    (recur (inc count))))

;; equivalent to:
(go
  (loop [count 0]
    (when (< count 5)
      (<! (timeout 500))
      (println "Count:" count)
      (recur (inc count)))))
```

---

## ขั้นตอนที่ 213: alts! - Select ระหว่าง Channels

```clojure
;; alts! เลือก channel แรกที่พร้อม
(defn race [& channels]
  (go
    (let [[value channel] (alts! channels)]
      {:value value :from channel})))

;; Timeout pattern
(defn with-timeout [ch timeout-ms default]
  (go
    (let [[value _] (alts! [ch (timeout timeout-ms)])]
      (or value default))))

;; ตัวอย่าง
(let [fast-ch (go (<! (timeout 100)) "fast result")
      slow-ch (go (<! (timeout 5000)) "slow result")]
  (println (<!! (with-timeout fast-ch 1000 "timeout!"))))
;; => "fast result"

;; Priority select ด้วย :priority true
(go
  (let [[v c] (alts! [high-priority-ch low-priority-ch]
                     :priority true)]
    (process! v)))

;; Default value ถ้าไม่มี channel พร้อม
(go
  (let [[v c] (alts! [ch] :default :nothing)]
    (if (= c :default)
      (println "Nothing ready")
      (println "Got:" v))))
```

---

## ขั้นตอนที่ 214: Pipeline Operations

```clojure
;; async/pipeline - parallel processing pipeline

;; pipeline: CPU-bound work (fixed threads)
(let [input  (chan 100)
      output (chan 100)]
  
  ;; 4 parallel workers
  (async/pipeline 4
                  output
                  (map #(* % %))     ; transducer
                  input)
  
  ;; Feed input
  (go (doseq [i (range 10)]
        (>! input i))
      (close! input))
  
  ;; Consume output
  (go-loop []
    (when-let [v (<! output)]
      (println "Result:" v)
      (recur))))

;; pipeline-blocking: I/O-bound work (expandable threads)
(async/pipeline-blocking
  10           ; parallelism
  output-ch
  (map slow-io-operation)
  input-ch)

;; pipeline-async: async operations
(async/pipeline-async
  10
  output-ch
  (fn [input output-ch]
    (go
      (let [result (<! (async-api-call input))]
        (>! output-ch result)
        (close! output-ch))))
  input-ch)
```

---

## ขั้นตอนที่ 215: Merge, Split, Mult, Pub

```clojure
;; ===== merge - รวมหลาย channels =====
(def merged-ch
  (async/merge [(chan1) (chan2) (chan3)]))

;; ===== mult - broadcast ไปยังหลาย channels =====
(def source (chan))
(def m (async/mult source))

;; tap เพื่อรับข้อมูล
(def tap1 (chan 10))
(def tap2 (chan 10))
(async/tap m tap1)
(async/tap m tap2)

;; ทุก message ที่ส่งไป source จะไปถึง tap1 และ tap2
(go (>! source "broadcast message"))

;; untap
(async/untap m tap1)

;; ===== pub/sub - topic-based routing =====
(def events (chan 100))
(def pub (async/pub events :type))  ; route by :type key

;; Subscribe to specific topics
(def user-events (chan 100))
(def order-events (chan 100))

(async/sub pub :user-created user-events)
(async/sub pub :order-placed order-events)

;; Publish
(go (>! events {:type :user-created :name "สมชาย"}))
(go (>! events {:type :order-placed :item "Widget"}))

;; Consume
(go-loop []
  (when-let [event (<! user-events)]
    (println "New user:" (:name event))
    (recur)))
```

---

## ขั้นตอนที่ 216: Real-world Pattern - Worker Pool

```clojure
(require '[clojure.core.async :as async])

(defn worker-pool
  "สร้าง worker pool ที่รับงานจาก work-ch"
  [n work-fn]
  (let [work-ch (async/chan 100)
        results-ch (async/chan 100)]
    
    ;; Start n workers
    (dotimes [worker-id n]
      (async/go-loop []
        (when-let [work (<! work-ch)]
          (let [result (work-fn work)]
            (>! results-ch {:worker worker-id
                             :work work
                             :result result}))
          (recur))))
    
    {:work-ch work-ch
     :results-ch results-ch}))

;; ใช้งาน
(let [{:keys [work-ch results-ch]}
      (worker-pool 4 (fn [x] (* x x)))]
  
  ;; Submit work
  (go (doseq [n (range 20)]
        (>! work-ch n))
      (async/close! work-ch))
  
  ;; Collect results
  (go-loop [collected []]
    (if-let [result (<! results-ch)]
      (recur (conj collected result))
      (println "All done:" (count collected) "results"))))
```

---

## ขั้นตอนที่ 217: Backpressure และ Flow Control

```clojure
;; Backpressure - ชะลอ producer เมื่อ consumer ช้า

;; ❌ No backpressure - producer อาจ overwhelm consumer
(go-loop []
  (let [data (generate-data)]
    (put! ch data)    ; fire-and-forget!
    (recur)))

;; ✅ With backpressure - producer รอ consumer
(go-loop []
  (let [data (generate-data)]
    (when (>! ch data)    ; blocks ถ้า channel เต็ม
      (recur))))

;; Throttling ด้วย timeout
(defn throttled-producer [ch rate-ms items]
  (go
    (doseq [item items]
      (>! ch item)
      (<! (timeout rate-ms)))))   ; rate limit!

;; Debouncing
(defn debounced-channel [source-ch delay-ms]
  (let [out (chan)]
    (go-loop [pending nil]
      (let [[v c] (alts! (remove nil? [source-ch
                                        (when pending (timeout delay-ms))]))]
        (cond
          (= c source-ch) (recur v)           ; got new value, reset timer
          (= c (timeout delay-ms)) (do (>! out pending) (recur nil)))))
    out))
```

---

## ขั้นตอนที่ 218: Async HTTP Requests

```clojure
;; HTTP requests แบบ async ด้วย core.async

;; Concept: ทำ N requests พร้อมกัน
(defn fetch-all [urls]
  (let [result-ch (chan (count urls))]
    (doseq [url urls]
      (go
        (try
          (let [response (http/get url)]   ; blocking call in go = uses thread!
            (>! result-ch {:url url :status (:status response)}))
          (catch Exception e
            (>! result-ch {:url url :error (.getMessage e)})))))
    result-ch))

;; ✅ Better: ใช้ thread สำหรับ blocking I/O
(defn fetch-async [url result-ch]
  (async/thread    ; real thread, not go-block
    (try
      (let [response (http/get url)]
        (>!! result-ch {:url url :status (:status response)}))
      (catch Exception e
        (>!! result-ch {:url url :error (.getMessage e)})))))

;; Collect N results
(defn collect-n [ch n]
  (go-loop [results [] remaining n]
    (if (zero? remaining)
      results
      (recur (conj results (<! ch))
             (dec remaining)))))

;; ใช้งาน
(let [urls ["http://api1.example.com" "http://api2.example.com" "http://api3.example.com"]
      result-ch (chan (count urls))]
  (doseq [url urls]
    (fetch-async url result-ch))
  (<!! (collect-n result-ch (count urls))))
```

---

## ขั้นตอนที่ 219: Event-driven Architecture

```clojure
;; Event Bus ด้วย core.async

(defrecord EventBus [pub-ch pub])

(defn create-event-bus []
  (let [pub-ch (chan 1000)
        pub (async/pub pub-ch :event-type)]
    (->EventBus pub-ch pub)))

(defn publish! [bus event]
  (async/put! (:pub-ch bus) event))

(defn subscribe! [bus event-type handler]
  (let [sub-ch (chan 100)]
    (async/sub (:pub bus) event-type sub-ch)
    (async/go-loop []
      (when-let [event (async/<! sub-ch)]
        (handler event)
        (recur)))
    sub-ch))  ; return for unsubscribe

(defn unsubscribe! [bus event-type sub-ch]
  (async/unsub (:pub bus) event-type sub-ch)
  (async/close! sub-ch))

;; ใช้งาน
(def bus (create-event-bus))

(subscribe! bus :user-login
  (fn [event]
    (println "User logged in:" (:user-id event))))

(subscribe! bus :user-login
  (fn [event]
    (audit-log! event)))

(publish! bus {:event-type :user-login
               :user-id 42
               :timestamp (System/currentTimeMillis)})
```

---

## ขั้นตอนที่ 220: Stateful Actors

```clojure
;; Actor pattern ด้วย go-loop

(defn create-actor
  "สร้าง stateful actor"
  [initial-state handler]
  (let [mailbox (chan 100)]
    (async/go-loop [state initial-state]
      (when-let [message (<! mailbox)]
        (let [new-state (handler state message)]
          (recur new-state))))
    mailbox))

;; Counter actor
(def counter-actor
  (create-actor 0
    (fn [state {:keys [type payload reply-ch]}]
      (case type
        :increment (let [new-state (+ state payload)]
                     (when reply-ch (async/put! reply-ch new-state))
                     new-state)
        :get       (do (when reply-ch (async/put! reply-ch state))
                       state)
        :reset     0
        state))))

;; Send message ไปยัง actor
(defn send-actor! [actor msg]
  (async/put! actor msg))

(defn ask-actor! [actor msg]
  (let [reply-ch (chan 1)]
    (async/put! actor (assoc msg :reply-ch reply-ch))
    (<!! reply-ch)))

;; ใช้งาน
(send-actor! counter-actor {:type :increment :payload 1})
(send-actor! counter-actor {:type :increment :payload 5})
(ask-actor! counter-actor {:type :get})  ; => 6
```

---

## ขั้นตอนที่ 221: Stream Processing

```clojure
;; Real-time stream processing

(defn stream-processor
  "Process infinite stream with windowing"
  [source-ch window-size]
  (let [output-ch (chan 100)]
    (async/go-loop [window (java.util.ArrayDeque.)]
      (when-let [item (<! source-ch)]
        (.addLast window item)
        (when (> (.size window) window-size)
          (.removeFirst window))
        (when (= (.size window) window-size)
          (>! output-ch (vec window)))
        (recur window)))
    output-ch))

;; Moving average
(defn moving-average [source-ch window-size]
  (let [windowed (stream-processor source-ch window-size)]
    (async/pipe
      windowed
      (chan 100 (map #(/ (reduce + %) (count %)))))))

;; ตัวอย่าง: stock price moving average
(let [prices (chan 100)
      avg-5min (moving-average prices 5)]
  
  ;; Feed prices
  (go (doseq [price [100 102 98 105 103 101 107 104]]
        (>! prices price)
        (<! (timeout 100))))
  
  ;; Monitor
  (go-loop []
    (when-let [avg (<! avg-5min)]
      (printf "5-min MA: %.2f%n" (double avg))
      (recur))))
```

---

## ขั้นตอนที่ 222: Error Handling ใน core.async

```clojure
;; Error handling ใน go blocks

;; Pattern 1: Try/catch ใน go block
(defn safe-go [f]
  (go
    (try
      (f)
      (catch Exception e
        {:error (.getMessage e)}))))

;; Pattern 2: Result channel ที่ส่ง errors ด้วย
(defn fetch-user [id]
  (go
    (try
      {:ok (db/find-user id)}
      (catch Exception e
        {:err (.getMessage e)}))))

(go
  (let [{:keys [ok err]} (<! (fetch-user 42))]
    (if err
      (println "Error:" err)
      (println "User:" ok))))

;; Pattern 3: Error channel แยก
(defn pipeline-with-errors [input-ch]
  (let [success-ch (chan 100)
        error-ch   (chan 100)]
    (go-loop []
      (when-let [item (<! input-ch)]
        (try
          (>! success-ch (process item))
          (catch Exception e
            (>! error-ch {:item item :error (.getMessage e)})))
        (recur)))
    [success-ch error-ch]))

;; Monitor errors separately
(let [[results errors] (pipeline-with-errors input-ch)]
  (go-loop []
    (when-let [err (<! errors)]
      (log/error "Processing failed:" err)
      (recur))))
```

---

## ขั้นตอนที่ 223: Testing Async Code

```clojure
;; Testing core.async code

(require '[clojure.test :refer :all]
         '[clojure.core.async :as async])

;; Pattern 1: <!! ใน tests
(deftest test-channel-value
  (let [ch (chan 1)]
    (go (>! ch 42))
    (is (= 42 (<!! ch)))))

;; Pattern 2: timeout guard
(defn <??
  "Take from channel with timeout, throw if timeout"
  [ch timeout-ms]
  (let [[v c] (alts!! [ch (timeout timeout-ms)])]
    (if (= c ch) v
        (throw (ex-info "Timeout" {:timeout-ms timeout-ms})))))

(deftest test-async-operation
  (let [result-ch (go
                    (<! (timeout 100))
                    "done")]
    (is (= "done" (<?? result-ch 1000)))))

;; Pattern 3: collect all results
(defn drain! [ch timeout-ms]
  (loop [results []]
    (let [[v c] (alts!! [ch (timeout timeout-ms)])]
      (if (= c ch)
        (recur (conj results v))
        results))))

(deftest test-pipeline
  (let [in (chan 5)
        out (chan 5)]
    (pipeline 2 out (map inc) in)
    (go (doseq [i (range 5)] (>! in i)) (close! in))
    (is (= [1 2 3 4 5] (sort (drain! out 1000))))))
```

---

## ขั้นตอนที่ 224: thread vs go

```clojure
;; go block: ใช้ park แทน block (non-blocking waiting)
;; - สำหรับ I/O-bound, fast operations
;; - ใช้ fixed thread pool (default: 8 threads)
;; - ❌ อย่าใช้ blocking operations ใน go block!

;; thread: สร้าง real Java thread
;; - สำหรับ blocking I/O (database, HTTP, file)
;; - ใช้ CPU thread จริง

;; ❌ Wrong: blocking I/O ใน go block
(go
  (let [data (slurp "large-file.txt")]  ; blocks thread!
    (>! output-ch data)))

;; ✅ Right: blocking I/O ใน thread
(async/thread
  (let [data (slurp "large-file.txt")]
    (>!! output-ch data)))

;; หรือ wrap ใน future ใน go
(go
  (let [data (<! (async/thread (slurp "large-file.txt")))]
    (>! output-ch data)))

;; Rule of thumb:
;; go = fast, non-blocking, channel operations
;; thread = slow, blocking, I/O operations
```

---

## Project Exercise: Real-time Log Processor

```clojure
(ns log-processor.core
  (:require [clojure.core.async :as async :refer [chan go go-loop <! >! close!]]))

;; ===== Log Entry =====
(defn parse-log-line [line]
  (let [[timestamp level message] (clojure.string/split line #"\s+" 3)]
    {:timestamp timestamp
     :level (keyword (clojure.string/lower-case level))
     :message message
     :raw line}))

;; ===== Processing Pipeline =====
(defn start-log-processor []
  (let [raw-ch     (chan 1000)
        parsed-ch  (chan 1000)
        filtered-ch (chan 100)
        alert-ch   (chan 50)]
    
    ;; Stage 1: Parse raw log lines
    (async/pipeline 4
                    parsed-ch
                    (map parse-log-line)
                    raw-ch)
    
    ;; Stage 2: Filter errors and warnings
    (async/pipeline 2
                    filtered-ch
                    (filter #(#{:error :warn} (:level %)))
                    parsed-ch)
    
    ;; Stage 3: Detect patterns → alert
    (go-loop [recent-errors (java.util.ArrayDeque.)]
      (when-let [entry (<! filtered-ch)]
        (when (= :error (:level entry))
          (.addLast recent-errors (System/currentTimeMillis))
          ;; Keep only last 60 seconds
          (let [one-min-ago (- (System/currentTimeMillis) 60000)]
            (while (and (.peekFirst recent-errors)
                        (< (.peekFirst recent-errors) one-min-ago))
              (.removeFirst recent-errors)))
          ;; Alert if > 10 errors in last minute
          (when (> (.size recent-errors) 10)
            (>! alert-ch {:type :high-error-rate
                           :count (.size recent-errors)
                           :entry entry})))
        (recur recent-errors)))
    
    {:raw-ch raw-ch
     :alert-ch alert-ch}))

;; ===== Main =====
(defn -main []
  (let [{:keys [raw-ch alert-ch]} (start-log-processor)]
    
    ;; Alert handler
    (go-loop []
      (when-let [alert (<! alert-ch)]
        (println "🚨 ALERT:" (:type alert) "- count:" (:count alert))
        (recur)))
    
    ;; Simulate log input
    (go
      (doseq [i (range 100)]
        (let [level (rand-nth ["INFO" "WARN" "ERROR" "ERROR" "ERROR"])]
          (>! raw-ch (str "2024-01-01T00:00:00 " level " Message " i)))
        (<! (async/timeout 50))))
    
    (Thread/sleep 10000)
    (println "Done")))
```

---

### สรุป core.async

```
Core Concepts:
==============
chan     - สร้าง channel
>!/<! - put/take ใน go block (non-blocking)
>!!/<!  - put/take นอก go block (blocking)
go      - lightweight goroutine
go-loop - looping goroutine
thread  - real thread สำหรับ blocking I/O
alts!   - select ระหว่าง channels
timeout - channel ที่ close หลัง N ms

Higher-level:
=============
pipeline         - parallel transducer pipeline
pipeline-blocking - blocking I/O pipeline
merge            - รวม channels
mult/tap         - broadcast
pub/sub          - topic routing
```

---

*Part 8 จาก 100+ | ขั้นตอน 211-240 จาก 1000+*
