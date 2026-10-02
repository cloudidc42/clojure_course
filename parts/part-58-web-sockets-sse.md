# Part 58: WebSockets และ Server-Sent Events
## ขั้นตอนที่ 1711-1740: Real-time Communication, Chat, Live Updates, Heartbeat

---

## บทนำ

Real-time communication ใน Clojure:
- **WebSockets** - full-duplex, bidirectional
- **Server-Sent Events (SSE)** - server push, unidirectional
- **Long polling** - polling ที่ efficient
- **Chat system** - real-time messaging
- **Live dashboard** - push metrics updates

---

## ขั้นตอนที่ 1711: WebSocket ด้วย http-kit

```clojure
(ns myapp.websocket
  (:require [org.httpkit.server :as http]
            [clojure.core.async :as async]
            [cheshire.core :as json]))

;; Connection registry
(defonce connections (atom {}))

(defn connect! [client-id channel]
  (swap! connections assoc client-id {:channel channel
                                        :connected-at (java.time.Instant/now)
                                        :rooms #{}})
  (println "Client connected:" client-id))

(defn disconnect! [client-id]
  (swap! connections dissoc client-id)
  (println "Client disconnected:" client-id))

(defn send! [client-id message]
  (when-let [conn (get @connections client-id)]
    (http/send! (:channel conn)
                 (json/generate-string message))))

(defn broadcast! [message]
  (doseq [[_ conn] @connections]
    (http/send! (:channel conn)
                 (json/generate-string message))))

(defn broadcast-to-room! [room-id message]
  (doseq [[_ conn] @connections
          :when (contains? (:rooms conn) room-id)]
    (http/send! (:channel conn)
                 (json/generate-string message))))

;; WebSocket handler
(defn ws-handler [request]
  (let [client-id (str (java.util.UUID/randomUUID))]
    (http/with-channel request channel
      (connect! client-id channel)
      
      (http/on-close channel
        (fn [status]
          (disconnect! client-id)))
      
      (http/on-receive channel
        (fn [data]
          (let [msg (json/parse-string data true)]
            (handle-message! client-id msg)))))))

;; Message router
(defmulti handle-message!
  (fn [_ msg] (:type msg)))

(defmethod handle-message! :join-room
  [client-id {:keys [room-id]}]
  (swap! connections update-in [client-id :rooms] conj room-id)
  (broadcast-to-room! room-id
    {:type :user-joined
     :user-id client-id
     :room-id room-id}))

(defmethod handle-message! :chat-message
  [client-id {:keys [room-id text]}]
  (let [msg {:type    :chat-message
              :from    client-id
              :room-id room-id
              :text    text
              :at      (str (java.time.Instant/now))}]
    (broadcast-to-room! room-id msg)
    (save-message! msg)))

(defmethod handle-message! :ping
  [client-id _]
  (send! client-id {:type :pong}))
```

---

## ขั้นตอนที่ 1712: Server-Sent Events (SSE)

```clojure
(ns myapp.sse
  (:require [ring.core.protocols :as ring-protocols]))

;; SSE format: "data: {json}\n\n"
(defn sse-format [event-type data & {:keys [id retry]}]
  (str
    (when id    (str "id: " id "\n"))
    (when retry (str "retry: " retry "\n"))
    "event: " (name event-type) "\n"
    "data: " (cheshire.core/generate-string data) "\n\n"))

;; SSE handler using async body
(defn sse-handler [request event-ch]
  {:status 200
   :headers {"Content-Type"  "text/event-stream"
              "Cache-Control" "no-cache"
              "Connection"    "keep-alive"
              "Access-Control-Allow-Origin" "*"}
   :body
   (reify ring-protocols/StreamableResponseBody
     (write-body-to-stream [_ response output-stream]
       (let [writer (java.io.OutputStreamWriter. output-stream)]
         ;; Send initial connection event
         (.write writer (sse-format :connected {:ok true}))
         (.flush writer)
         
         ;; Stream events
         (loop []
           (when-let [event (async/<!! event-ch)]
             (try
               (.write writer (sse-format (:type event) (:data event)))
               (.flush writer)
               (recur)
               (catch java.io.IOException _
                 ;; Client disconnected
                 (async/close! event-ch)))))
         
         (.close writer))))})

;; SSE endpoint
(defn events-endpoint [request]
  (let [user-id  (get-in request [:session :user-id])
        event-ch (async/chan 100)]
    
    ;; Register client
    (swap! sse-clients assoc user-id event-ch)
    
    ;; Cleanup on disconnect
    (async/go
      (async/<! (async/timeout 30000))  ; 30s timeout
      (async/close! event-ch)
      (swap! sse-clients dissoc user-id))
    
    (sse-handler request event-ch)))

;; Push to specific user
(defn push-to-user! [user-id event]
  (when-let [ch (get @sse-clients user-id)]
    (async/put! ch event)))

;; Push to all users
(defn push-to-all! [event]
  (doseq [[_ ch] @sse-clients]
    (async/put! ch event)))
```

---

## ขั้นตอนที่ 1713: Heartbeat และ Reconnection

```clojure
(ns myapp.ws-heartbeat
  (:require [clojure.core.async :as async]))

;; Heartbeat to detect stale connections
(defn start-heartbeat! [connections interval-ms]
  (let [stop-ch (async/chan)]
    (async/go-loop []
      (async/alt!
        (async/timeout interval-ms)
        ([_]
         (let [now       (System/currentTimeMillis)
               stale-ids (keep (fn [[id conn]]
                                  (when (> (- now (:last-seen conn 0)) (* 2 interval-ms))
                                    id))
                                @connections)]
           ;; Close stale connections
           (doseq [id stale-ids]
             (when-let [ch (:channel (get @connections id))]
               (org.httpkit.server/close ch))
             (swap! connections dissoc id))
           
           ;; Ping all remaining
           (doseq [[id conn] @connections]
             (try
               (org.httpkit.server/send!
                 (:channel conn)
                 (cheshire.core/generate-string {:type :ping}))
               (catch Exception _
                 (swap! connections dissoc id)))))
         (recur))
        
        stop-ch ([_] (println "Heartbeat stopped"))))
    
    stop-ch))

;; Update last-seen on message
(defn on-message-with-heartbeat [client-id msg]
  (swap! connections update-in [client-id :last-seen]
         (constantly (System/currentTimeMillis)))
  (handle-message! client-id msg))

;; Client-side reconnection (ClojureScript)
;; (defn connect-with-retry! [url on-message]
;;   (let [ws (atom nil)
;;         connect! (fn connect! []
;;                    (let [socket (js/WebSocket. url)]
;;                      (reset! ws socket)
;;                      (set! (.-onmessage socket) #(on-message (.-data %)))
;;                      (set! (.-onclose socket)
;;                        (fn [_]
;;                          (js/setTimeout connect! 3000)))))]  ; retry in 3s
;;     (connect!)))
```

---

## ขั้นตอนที่ 1714: Real-time Chat System

```clojure
(ns myapp.chat
  (:require [next.jdbc :as jdbc]
            [clojure.core.async :as async]))

;; Chat room management
(defonce rooms (atom {}))

(defn create-room! [db room-id name]
  (jdbc/execute-one! db
    ["INSERT INTO rooms (id, name, created_at)
      VALUES (?, ?, NOW())"
     room-id name])
  (swap! rooms assoc room-id {:id      room-id
                                :name    name
                                :members #{}}))

(defn join-room! [room-id user-id channel]
  (swap! rooms update-in [room-id :members] conj user-id)
  (swap! connections update user-id
         #(update % :rooms conj room-id))
  
  ;; Notify others
  (broadcast-to-room! room-id
    {:type    :member-joined
     :room-id room-id
     :user-id user-id
     :count   (count (get-in @rooms [room-id :members]))}))

(defn leave-room! [room-id user-id]
  (swap! rooms update-in [room-id :members] disj user-id)
  (broadcast-to-room! room-id
    {:type    :member-left
     :room-id room-id
     :user-id user-id}))

;; Message history
(defn get-recent-messages [db room-id limit]
  (jdbc/execute! db
    ["SELECT m.*, u.name as user_name
      FROM messages m
      JOIN users u ON m.user_id = u.id
      WHERE m.room_id = ?
      ORDER BY m.created_at DESC
      LIMIT ?"
     room-id limit]))

;; Typing indicators
(defonce typing-status (atom {}))

(defn user-typing! [room-id user-id]
  (swap! typing-status assoc-in [room-id user-id]
         (System/currentTimeMillis))
  (broadcast-to-room! room-id
    {:type    :typing
     :room-id room-id
     :user-id user-id}))

(defn clear-stale-typing! []
  (let [now     (System/currentTimeMillis)
        stale-ms 3000]
    (swap! typing-status
           (fn [status]
             (into {}
               (for [[room users] status]
                 [room (into {}
                          (filter #(< (- now (val %)) stale-ms) users))]))))))
```

---

## Project: Live Order Tracking

```clojure
(ns myapp.live-tracking
  (:require [clojure.core.async :as async]))

;; Real-time order status updates via SSE
(defonce order-subscribers (atom {}))

(defn subscribe-to-order! [order-id user-id event-ch]
  (swap! order-subscribers
         update order-id
         (fnil conj #{})
         {:user-id user-id :channel event-ch}))

(defn unsubscribe-from-order! [order-id user-id]
  (swap! order-subscribers
         update order-id
         (fn [subs]
           (remove #(= user-id (:user-id %)) subs))))

;; Push status update when order changes
(defn push-order-update! [order-id status details]
  (let [event {:type    :order-update
                :order-id order-id
                :status  status
                :details details
                :at      (str (java.time.Instant/now))}]
    
    (doseq [sub (get @order-subscribers order-id [])]
      (async/put! (:channel sub) event))))

;; Hook into order events
(defn on-order-status-change [order]
  (push-order-update!
    (:id order)
    (:status order)
    (case (:status order)
      :shipped  {:tracking (:tracking-number order)}
      :delivered {:at (str (java.time.Instant/now))}
      {})))

;; HTTP endpoint
(defn track-order-endpoint [request]
  (let [order-id (get-in request [:path-params :order-id])
        user-id  (get-in request [:session :user-id])
        event-ch (async/chan 50)]
    
    (subscribe-to-order! order-id user-id event-ch)
    
    ;; Send current status immediately
    (async/put! event-ch {:type     :current-status
                            :order-id order-id
                            :status   (db/get-order-status order-id)})
    
    (sse-handler request event-ch)))
```

---

*Part 58 จาก 100+ | ขั้นตอน 1711-1740 จาก 1000+*
