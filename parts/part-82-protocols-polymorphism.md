# Part 82: Protocols และ Polymorphism
## ขั้นตอนที่ 2431-2460: Protocols, Multimethods, Records, Types, Interfaces

---

## บทนำ

Clojure polymorphism:
- **Protocols** - fast, interface-like dispatch
- **Multimethods** - arbitrary dispatch logic
- **Records** - map + protocol implementation
- **Types** - low-level, no map behavior
- **extend-protocol** - add to existing types

---

## ขั้นตอนที่ 2431: Protocols เชิงลึก

```clojure
(ns myapp.protocols)

;; Protocol: defines a set of functions
(defprotocol Serializable
  "Things that can be serialized"
  (serialize   [this format] "Serialize to format (:json :edn :transit)")
  (deserialize [this data format] "Deserialize from format")
  (schema      [this] "Return schema for validation"))

(defprotocol Searchable
  (to-search-doc [this] "Convert to search document")
  (from-search-doc [this doc] "Reconstruct from search document"))

;; Implement protocol on record
(defrecord Product [id name price category]
  Serializable
  (serialize [this format]
    (case format
      :json    (json/generate-string (into {} this))
      :edn     (pr-str (into {} this))
      :transit (transit/write (into {} this))))
  
  (deserialize [_ data format]
    (case format
      :json    (map->Product (json/parse-string data true))
      :edn     (map->Product (edn/read-string data))))
  
  (schema [_]
    {:id       :uuid
     :name     :string
     :price    :decimal
     :category :string})
  
  Searchable
  (to-search-doc [this]
    {:id          (str id)
     :name        name
     :price_cents (int (* price 100))
     :category    category
     :suggest     {:input [name] :weight (int price)}})
  
  (from-search-doc [_ doc]
    (->Product (:id doc) (:name doc) (/ (:price_cents doc) 100.0) (:category doc))))

;; Extend protocol to existing types
(extend-protocol Serializable
  clojure.lang.PersistentHashMap
  (serialize [this format]
    (case format
      :json (json/generate-string this)
      :edn  (pr-str this)))
  
  nil
  (serialize [_ _] "null"))

;; Check protocol implementation
(satisfies? Serializable (->Product "1" "Widget" 9.99 "tools"))  ; => true
(satisfies? Serializable {})  ; => true (extended above)
(satisfies? Serializable 42)  ; => false
```

---

## ขั้นตอนที่ 2432: Multimethods ขั้นสูง

```clojure
;; Multimethods: dispatch on arbitrary function
(defmulti process-event
  (fn [event _context]
    [(:type event) (:source event)]))

;; Specific type+source combination
(defmethod process-event [:order-created :web]
  [event ctx]
  (send-order-confirmation! (:order event))
  (track-analytics! :web-order event))

(defmethod process-event [:order-created :mobile]
  [event ctx]
  (send-push-notification! (:user-id event))
  (track-analytics! :mobile-order event))

;; Default for any :order-created
(defmethod process-event [:order-created :default]
  [event ctx]
  (send-order-confirmation! (:order event)))

;; Catch-all default
(defmethod process-event :default
  [event ctx]
  (log/warn "Unhandled event type" (:type event)))

;; Hierarchy-based dispatch
(derive ::premium-customer ::customer)
(derive ::vip-customer ::premium-customer)

(defmulti discount-rate ::customer-type)

(defmethod discount-rate ::customer        [_] 0)
(defmethod discount-rate ::premium-customer [_] 0.10)
(defmethod discount-rate ::vip-customer    [_] 0.20)

;; ::vip-customer inherits from ::premium-customer
;; but overrides with 20%

;; Prefer one method over another
(prefer-method discount-rate ::vip-customer ::premium-customer)
```

---

## ขั้นตอนที่ 2433: Reify for One-off Implementations

```clojure
;; reify: create an anonymous type implementing protocols/interfaces

;; One-off event handler
(defn make-one-time-listener [f]
  (let [fired? (atom false)]
    (reify EventListener
      (on-event [_ event]
        (when (compare-and-set! fired? false true)
          (f event))))))

;; One-off Comparator
(defn make-comparator [key-fn]
  (reify java.util.Comparator
    (compare [_ a b]
      (compare (key-fn a) (key-fn b)))))

(sort (make-comparator :price) products)

;; One-off Iterator
(defn paginated-iterator [fetch-page-fn page-size]
  (let [page   (atom 0)
        buffer (atom [])]
    (reify java.util.Iterator
      (hasNext [_]
        (or (seq @buffer)
            (let [next-page (fetch-page-fn @page page-size)]
              (when (seq next-page)
                (reset! buffer next-page)
                (swap! page inc)
                true))))
      (next [_]
        (let [item (first @buffer)]
          (swap! buffer rest)
          item)))))
```

---

## ขั้นตอนที่ 2434: deftype สำหรับ Performance

```clojure
;; deftype: bare-bones, no map behavior
;; Use for performance-critical code

(deftype FastCounter [^:volatile-mutable ^long count]
  Object
  (toString [_] (str "Counter(" count ")"))
  
  clojure.lang.IDeref
  (deref [_] count)
  
  clojure.lang.IFn
  (invoke [this]
    (set! count (inc count))
    this))

;; Much faster than atom for single-threaded counting
(def c (->FastCounter 0))
(c)  ; increment
@c   ; read

;; Ring buffer (circular buffer)
(deftype RingBuffer [^objects buf ^int capacity
                     ^:volatile-mutable ^int head
                     ^:volatile-mutable ^int tail
                     ^:volatile-mutable ^int size]
  clojure.lang.IPersistentCollection
  (count [_] size)
  (seq [_]
    (when (pos? size)
      (for [i (range size)]
        (aget buf (mod (+ head i) capacity)))))
  
  clojure.lang.IPersistentStack
  (peek [_]
    (when (pos? size)
      (aget buf (mod (dec (+ head size)) capacity)))))

(defn ring-buffer [capacity]
  (->RingBuffer (object-array capacity) capacity 0 0 0))

(defn rb-push! [^RingBuffer rb item]
  (let [idx (mod (.tail rb) (.capacity rb))]
    (aset (.buf rb) idx item)
    (set! (.tail rb) (inc (.tail rb)))
    (if (= (.size rb) (.capacity rb))
      (set! (.head rb) (inc (.head rb)))
      (set! (.size rb) (inc (.size rb))))))
```

---

## Project: Plugin System ด้วย Protocols

```clojure
(ns myapp.plugins)

;; Plugin system using protocols
(defprotocol Plugin
  (plugin-name    [this])
  (plugin-version [this])
  (initialize!    [this config])
  (shutdown!      [this])
  (handle-event   [this event]))

;; Plugin registry
(defonce plugin-registry (atom {}))

(defn register-plugin! [plugin]
  (swap! plugin-registry assoc (plugin-name plugin) plugin))

(defn load-plugins! [config]
  (doseq [[name plugin] @plugin-registry]
    (println "Loading plugin:" name)
    (initialize! plugin (get config name {}))))

;; Example plugin
(defrecord SlackNotificationPlugin [webhook-url client]
  Plugin
  (plugin-name    [_] "slack-notifications")
  (plugin-version [_] "1.0.0")
  
  (initialize! [this config]
    (reset! client (slack/make-client (:webhook-url config))))
  
  (shutdown! [_]
    (when @client (slack/close! @client)))
  
  (handle-event [_ event]
    (when (= :order-created (:type event))
      (slack/send! @client
        (str "New order: " (get-in event [:order :id]))))))

;; Broadcast to all plugins
(defn broadcast-event! [event]
  (doseq [[_ plugin] @plugin-registry]
    (try
      (handle-event plugin event)
      (catch Exception e
        (log/error "Plugin error" {:plugin (plugin-name plugin)
                                    :error  (.getMessage e)})))))
```

---

*Part 82 จาก 100+ | ขั้นตอน 2431-2460 จาก 1000+*
