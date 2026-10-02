# Part 26: WebSockets และ Real-Time Communication
## ขั้นตอนที่ 751-780: HTTP-Kit, SSE, WebSocket Protocol, Real-Time Apps

---

## บทนำ

Real-time communication ใน Clojure:
- **HTTP-Kit** - Clojure web server ที่รองรับ WebSocket และ async HTTP
- **WebSocket** - full-duplex bidirectional communication
- **Server-Sent Events (SSE)** - one-way server → client streaming
- **Long Polling** - compatible fallback

---

## ขั้นตอนที่ 751: HTTP-Kit Setup

```clojure
;; deps.edn
;; {:deps {http-kit/http-kit {:mvn/version "2.8.0"}}}

(ns myapp.server
  (:require [org.httpkit.server :as httpkit]
            [reitit.ring :as ring]))

;; HTTP-Kit สามารถใช้แทน Jetty ได้
(defn start-server! [handler]
  (httpkit/run-server
    handler
    {:port        3000
     :thread      8          ; thread pool size
     :max-body    (* 100 1024 1024)  ; 100MB
     :queue-size  20000}))

;; Stop server
(defonce server (atom nil))

(defn start! []
  (reset! server (start-server! app-handler)))

(defn stop! []
  (when @server
    (@server :timeout 100)
    (reset! server nil)))
```

---

## ขั้นตอนที่ 752: WebSocket Server

```clojure
(ns myapp.websocket
  (:require [org.httpkit.server :as httpkit]))

;; Connected clients registry
(def clients (atom {}))

(defn ws-handler [request]
  (httpkit/as-channel request
    {:on-open    (fn [channel]
                    (let [client-id (str (java.util.UUID/randomUUID))]
                      (swap! clients assoc client-id channel)
                      (println "Client connected:" client-id)
                      (httpkit/send! channel
                        (cheshire.core/generate-string
                          {:type "connected" :id client-id}))))
     
     :on-close   (fn [channel status]
                    ;; Remove from registry
                    (swap! clients
                      (fn [cs]
                        (into {} (filter #(not= channel (val %)) cs))))
                    (println "Client disconnected, status:" status))
     
     :on-receive (fn [channel data]
                    (let [msg (cheshire.core/parse-string data true)]
                      (handle-message! channel msg)))}))

;; Send to specific client
(defn send-to-client! [client-id message]
  (when-let [channel (get @clients client-id)]
    (httpkit/send! channel (cheshire.core/generate-string message))))

;; Broadcast to all clients
(defn broadcast! [message]
  (let [json (cheshire.core/generate-string message)]
    (doseq [[_ channel] @clients]
      (httpkit/send! channel json))))

;; Broadcast to specific group
(defn broadcast-to-group! [group-id message]
  (doseq [[client-id channel] @clients]
    (when (= group-id (get @client-groups client-id))
      (httpkit/send! channel (cheshire.core/generate-string message)))))
```

---

## ขั้นตอนที่ 753: WebSocket Client Handling

```clojure
;; Message routing
(defmulti handle-message! (fn [channel msg] (:type msg)))

(defmethod handle-message! "ping" [channel _]
  (httpkit/send! channel (cheshire.core/generate-string {:type "pong"})))

(defmethod handle-message! "chat" [channel {:keys [text room]}]
  ;; Broadcast to room
  (broadcast-to-room! room {:type "chat"
                              :text text
                              :from (get-client-id channel)
                              :timestamp (System/currentTimeMillis)}))

(defmethod handle-message! "subscribe" [channel {:keys [channel-name]}]
  ;; Add client to channel subscription
  (swap! subscriptions update channel-name (fnil conj #{}) channel)
  (httpkit/send! channel
    (cheshire.core/generate-string
      {:type "subscribed" :channel channel-name})))

(defmethod handle-message! :default [channel msg]
  (httpkit/send! channel
    (cheshire.core/generate-string
      {:type "error" :message "Unknown message type"})))

;; Heartbeat to detect dead connections
(defn start-heartbeat! []
  (future
    (loop []
      (Thread/sleep 30000)  ; every 30 seconds
      (doseq [[id channel] @clients]
        (try
          (httpkit/send! channel
            (cheshire.core/generate-string {:type "ping"}))
          (catch Exception e
            (swap! clients dissoc id))))
      (recur))))
```

---

## ขั้นตอนที่ 754: WebSocket Chat Room

```clojure
(ns myapp.chat
  (:require [org.httpkit.server :as httpkit]
            [clojure.core.async :as async]))

;; Room state
(def rooms (atom {}))  ; {room-id #{channel}}

(defn join-room! [room-id channel user-info]
  (swap! rooms update room-id (fnil conj #{}) channel)
  ;; Notify room members
  (broadcast-to-room! room-id
    {:type    "user-joined"
     :user    user-info
     :members (count (get @rooms room-id))}))

(defn leave-room! [room-id channel]
  (swap! rooms update room-id disj channel)
  (when (empty? (get @rooms room-id))
    (swap! rooms dissoc room-id)))

(defn broadcast-to-room! [room-id message]
  (doseq [channel (get @rooms room-id)]
    (httpkit/send! channel (cheshire.core/generate-string message))))

;; Message history (last 50 messages per room)
(def message-history (atom {}))

(defn add-message! [room-id message]
  (swap! message-history
    (fn [history]
      (update history room-id
        (fn [msgs]
          (take 50 (cons message (or msgs '()))))))))

(defn get-history [room-id]
  (reverse (get @message-history room-id [])))

;; Chat WebSocket handler
(defn chat-handler [request]
  (let [room-id (get-in request [:path-params :room-id])
        user    (:current-user request)]
    (httpkit/as-channel request
      {:on-open    (fn [channel]
                      (join-room! room-id channel user)
                      ;; Send history
                      (httpkit/send! channel
                        (cheshire.core/generate-string
                          {:type    "history"
                           :messages (get-history room-id)})))
       
       :on-close   (fn [channel _]
                      (leave-room! room-id channel))
       
       :on-receive (fn [channel data]
                      (let [msg (cheshire.core/parse-string data true)
                            full-msg {:type      "message"
                                       :text      (:text msg)
                                       :user      user
                                       :timestamp (System/currentTimeMillis)}]
                        (add-message! room-id full-msg)
                        (broadcast-to-room! room-id full-msg)))})))
```

---

## ขั้นตอนที่ 755: Server-Sent Events (SSE)

```clojure
;; SSE: one-way streaming from server to client
;; ดีสำหรับ: live feeds, notifications, progress updates
;; ไม่ต้องการ full-duplex

(defn sse-headers []
  {"Content-Type"  "text/event-stream"
   "Cache-Control" "no-cache"
   "Connection"    "keep-alive"
   "X-Accel-Buffering" "no"})  ; nginx buffering off

(defn sse-event [& {:keys [id event data retry]}]
  (str (when id    (str "id: " id "\n"))
       (when event (str "event: " event "\n"))
       "data: " data "\n"
       (when retry (str "retry: " retry "\n"))
       "\n"))

;; SSE endpoint
(defn sse-handler [request]
  (let [user-id (get-in request [:auth :sub])]
    (httpkit/as-channel request
      {:on-open (fn [channel]
                    ;; Register channel for notifications
                    (swap! sse-clients assoc user-id channel)
                    
                    ;; Send initial event
                    (httpkit/send! channel
                      (sse-event :event "connected"
                                  :data (cheshire.core/generate-string
                                          {:userId user-id}))))
       
       :on-close (fn [channel _]
                    (swap! sse-clients dissoc user-id))}
      ;; Response headers
      {:status  200
       :headers (sse-headers)})))

;; Send notification to user
(defn notify-user! [user-id event-type data]
  (when-let [channel (get @sse-clients user-id)]
    (httpkit/send! channel
      (sse-event :event event-type
                  :data  (cheshire.core/generate-string data)))))

;; Push notifications
(defn notify-order-update! [user-id order]
  (notify-user! user-id "order-update"
    {:orderId (:id order)
     :status  (name (:status order))
     :message (str "Order " (:id order) " is now " (name (:status order)))}))
```

---

## ขั้นตอนที่ 756: Real-Time Dashboard

```clojure
(ns myapp.dashboard
  (:require [org.httpkit.server :as httpkit]
            [clojure.core.async :as async]))

;; Dashboard metrics stream
(def metrics-ch (async/chan (async/sliding-buffer 100)))

;; Background metrics producer
(defn start-metrics-collector! []
  (future
    (loop []
      (let [metrics {:timestamp (System/currentTimeMillis)
                      :cpu-usage  (get-cpu-usage)
                      :memory-mb  (get-memory-usage)
                      :req-per-sec (get-request-rate)
                      :active-users (count @clients)}]
        (async/>!! metrics-ch metrics))
      (Thread/sleep 1000)  ; every second
      (recur))))

;; WebSocket endpoint for dashboard
(defn dashboard-ws-handler [request]
  (httpkit/as-channel request
    {:on-open  (fn [channel]
                  ;; Subscribe to metrics
                  (let [sub-ch (async/chan 100)]
                    (async/tap metrics-mult sub-ch)
                    (swap! dashboard-clients assoc channel sub-ch)
                    ;; Start forwarding
                    (async/go-loop []
                      (when-let [m (async/<! sub-ch)]
                        (when (httpkit/open? channel)
                          (httpkit/send! channel
                            (cheshire.core/generate-string m))
                          (recur))))))
     
     :on-close (fn [channel _]
                  (when-let [sub-ch (get @dashboard-clients channel)]
                    (async/close! sub-ch))
                  (swap! dashboard-clients dissoc channel))}))

;; Initialize multiplexer
(def metrics-mult (async/mult metrics-ch))
```

---

## ขั้นตอนที่ 757: WebSocket Rooms ด้วย core.async

```clojure
(ns myapp.rooms
  (:require [clojure.core.async :as async]))

;; Each room has its own channel
(def room-channels (atom {}))

(defn get-or-create-room! [room-id]
  (or (get @room-channels room-id)
      (let [ch (async/chan (async/sliding-buffer 1000))]
        (swap! room-channels assoc room-id ch)
        ;; Room lifecycle management
        (async/go-loop []
          (when-let [msg (async/<! ch)]
            ;; Fanout to all room subscribers
            (doseq [sub-ch (get @room-subscribers room-id [])]
              (async/>! sub-ch msg))
            (recur)))
        ch)))

(defn join-room-async! [room-id ws-channel user]
  (let [sub-ch (async/chan 100)]
    (swap! room-subscribers update room-id (fnil conj []) sub-ch)
    
    ;; Forward messages from sub-ch to WebSocket
    (async/go-loop []
      (when-let [msg (async/<! sub-ch)]
        (httpkit/send! ws-channel (cheshire.core/generate-string msg))
        (recur)))
    
    sub-ch))

(defn send-to-room! [room-id message]
  (when-let [room-ch (get @room-channels room-id)]
    (async/>!! room-ch message)))
```

---

## Project: Live Collaboration Editor

```clojure
;; Real-time collaborative text editor

(ns collab.editor
  (:require [org.httpkit.server :as httpkit]
            [clojure.core.async :as async]))

;; Operational Transform: represent changes
(defrecord Op [type position text length])

(defn insert-op [pos text] (->Op :insert pos text nil))
(defn delete-op [pos len]  (->Op :delete pos nil len))

;; Document state
(def documents (atom {}))  ; {doc-id {:content "..." :version 0}}

;; Apply operation to document
(defn apply-op! [doc-id op]
  (swap! documents
    (fn [docs]
      (let [doc (get docs doc-id {:content "" :version 0})]
        (assoc docs doc-id
          (case (:type op)
            :insert {:content (str (subs (:content doc) 0 (:position op))
                                    (:text op)
                                    (subs (:content doc) (:position op)))
                     :version (inc (:version doc))}
            :delete {:content (str (subs (:content doc) 0 (:position op))
                                    (subs (:content doc) (+ (:position op) (:length op))))
                     :version (inc (:version doc))}))))))

;; WebSocket handler for collaborative editing
(defn editor-ws-handler [request]
  (let [doc-id  (get-in request [:path-params :doc-id])
        user    (:current-user request)]
    (httpkit/as-channel request
      {:on-open    (fn [channel]
                      (swap! doc-clients update doc-id (fnil conj #{}) channel)
                      ;; Send current document state
                      (httpkit/send! channel
                        (cheshire.core/generate-string
                          {:type     "state"
                           :document (get @documents doc-id {:content "" :version 0})})))
       
       :on-close   (fn [channel _]
                      (swap! doc-clients update doc-id disj channel))
       
       :on-receive (fn [channel data]
                      (let [{:keys [type position text length]} (cheshire.core/parse-string data true)]
                        (let [op (if (= "insert" type)
                                   (insert-op position text)
                                   (delete-op position length))]
                          (apply-op! doc-id op)
                          ;; Broadcast to all other clients in document
                          (doseq [other-channel (disj (get @doc-clients doc-id #{}) channel)]
                            (httpkit/send! other-channel
                              (cheshire.core/generate-string
                                {:type     "op"
                                 :op       (into {} op)
                                 :version  (:version (get @documents doc-id))
                                 :user     (:name user)})))))})))
```

---

*Part 26 จาก 100+ | ขั้นตอน 751-780 จาก 1000+*
