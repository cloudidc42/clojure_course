# Part 49: Protocols และ Polymorphism ขั้นสูง
## ขั้นตอนที่ 1441-1470: Protocols, Records, Types, Multimethods, Open Dispatch

---

## บทนำ

Polymorphism ใน Clojure:
- **Protocols** - เหมือน interfaces แต่ extensible
- **Records** - fast maps with protocol support
- **Types (deftype)** - low-level, mutable fields
- **Multimethods** - dispatch on arbitrary function
- **Extension ด้วย extend-protocol**

---

## ขั้นตอนที่ 1441: Protocols ขั้นสูง

```clojure
(ns myapp.protocols)

;; Protocol as abstraction boundary
(defprotocol Storage
  "Abstract storage interface"
  (store!   [this key value opts])
  (fetch    [this key])
  (delete!  [this key])
  (exists?  [this key])
  (list-keys [this prefix]))

(defprotocol Cacheable
  "Things that can be cached"
  (cache-key    [this])
  (cache-ttl    [this])
  (serialize    [this])
  (deserialize  [_ data]))

(defprotocol Validatable
  "Data that can validate itself"
  (valid?   [this])
  (errors   [this])
  (validate [this]))

;; Extend protocol to existing types
(extend-protocol Validatable
  clojure.lang.IPersistentMap
  (valid? [m] (empty? (errors m)))
  (errors [m] (cond-> []
                (nil? (:email m))  (conj "Email required")
                (nil? (:name m))   (conj "Name required")))
  (validate [m] (if (valid? m) m (throw (ex-info "Invalid" {:errors (errors m)}))))
  
  String
  (valid? [s] (not (clojure.string/blank? s)))
  (errors [s] (when (clojure.string/blank? s) ["String cannot be blank"]))
  (validate [s] (if (valid? s) s (throw (ex-info "Blank string" {})))))

;; Extend to nil safely
(extend-protocol Validatable
  nil
  (valid? [_] false)
  (errors [_] ["Value is nil"])
  (validate [_] (throw (ex-info "Nil value" {}))))
```

---

## ขั้นตอนที่ 1442: Storage Implementations

```clojure
;; In-Memory Storage
(defrecord InMemoryStorage [store]
  Storage
  
  (store! [_ key value {:keys [ttl]}]
    (let [entry {:value value
                  :expires-at (when ttl
                                (+ (System/currentTimeMillis) (* ttl 1000)))}]
      (swap! store assoc key entry)
      value))
  
  (fetch [_ key]
    (when-let [{:keys [value expires-at]} (get @store key)]
      (if (and expires-at (> (System/currentTimeMillis) expires-at))
        (do (swap! store dissoc key) nil)
        value)))
  
  (delete! [_ key]
    (swap! store dissoc key)
    nil)
  
  (exists? [_ key]
    (boolean (fetch _ key)))
  
  (list-keys [_ prefix]
    (filter #(clojure.string/starts-with? (str %) prefix)
             (keys @store))))

(defn create-in-memory-storage []
  (->InMemoryStorage (atom {})))

;; Redis Storage
(defrecord RedisStorage [conn]
  Storage
  
  (store! [_ key value {:keys [ttl]}]
    (taoensso.carmine/wcar conn
      (if ttl
        (taoensso.carmine/setex (str key) ttl (cheshire.core/generate-string value))
        (taoensso.carmine/set   (str key) (cheshire.core/generate-string value))))
    value)
  
  (fetch [_ key]
    (when-let [data (taoensso.carmine/wcar conn
                      (taoensso.carmine/get (str key)))]
      (cheshire.core/parse-string data true)))
  
  (delete! [_ key]
    (taoensso.carmine/wcar conn (taoensso.carmine/del (str key)))
    nil)
  
  (exists? [_ key]
    (= 1 (taoensso.carmine/wcar conn (taoensso.carmine/exists (str key)))))
  
  (list-keys [_ prefix]
    (taoensso.carmine/wcar conn (taoensso.carmine/keys (str prefix "*")))))

;; S3 Storage
(defrecord S3Storage [client bucket-name]
  Storage
  
  (store! [_ key value {:keys [content-type ttl]}]
    (cognitect.aws.client.api/invoke client
      {:op      :PutObject
       :request {:Bucket      bucket-name
                  :Key         (str key)
                  :Body        (.getBytes (cheshire.core/generate-string value))
                  :ContentType (or content-type "application/json")}})
    value)
  
  (fetch [_ key]
    (try
      (let [result (cognitect.aws.client.api/invoke client
                     {:op      :GetObject
                      :request {:Bucket bucket-name :Key (str key)}})]
        (cheshire.core/parse-string
          (slurp (:Body result)) true))
      (catch Exception _ nil)))
  
  (exists? [_ key]
    (try
      (cognitect.aws.client.api/invoke client
        {:op      :HeadObject
         :request {:Bucket bucket-name :Key (str key)}})
      true
      (catch Exception _ false))))
```

---

## ขั้นตอนที่ 1443: deftype สำหรับ Performance

```clojure
;; deftype: mutable fields, implements Java interfaces
;; ใช้เมื่อต้องการ performance สูงสุด

(deftype PriorityQueue [^:volatile-mutable heap]
  clojure.lang.IPersistentCollection
  
  (count [_] (count heap))
  
  (cons [this item]
    (set! heap (conj heap item))
    this)
  
  (empty [_] (PriorityQueue. []))
  
  (equiv [_ other] (= heap (.-heap other)))
  
  clojure.lang.ISeq
  (first [_] (first (sort heap)))
  (next  [_] (next  (sort heap)))
  (more  [_] (rest  (sort heap)))
  
  Object
  (toString [_] (str "PriorityQueue" heap)))

;; Ring buffer implementation
(deftype RingBuffer [^objects buf
                     ^:volatile-mutable head
                     ^:volatile-mutable tail
                     ^:volatile-mutable size]
  
  clojure.lang.IPersistentCollection
  (count [_] size)
  
  clojure.lang.IPersistentStack
  (peek [_]
    (when (pos? size)
      (aget buf head)))
  
  (pop [this]
    (when (pos? size)
      (let [item (aget buf head)]
        (aset buf head nil)
        (set! head (mod (inc head) (alength buf)))
        (set! size (dec size))
        item)
    this))
  
  Object
  (toString [_]
    (str "RingBuffer[" size "/" (alength buf) "]")))

(defn create-ring-buffer [capacity]
  (->RingBuffer (object-array capacity) 0 0 0))
```

---

## ขั้นตอนที่ 1444: Multimethods ขั้นสูง

```clojure
(ns myapp.dispatch)

;; Hierarchy-based dispatch
(derive ::circle    ::shape)
(derive ::rectangle ::shape)
(derive ::triangle  ::shape)
(derive ::square    ::rectangle)  ; square is a rectangle

(defmulti area :type)

(defmethod area ::circle [{:keys [radius]}]
  (* Math/PI radius radius))

(defmethod area ::rectangle [{:keys [width height]}]
  (* width height))

(defmethod area ::triangle [{:keys [base height]}]
  (* 0.5 base height))

;; Square inherits rectangle method!
(area {:type ::square :width 5 :height 5})  ; => 25

;; Multi-dispatch on two args
(defmulti intersects?
  (fn [a b] [(:type a) (:type b)]))

(defmethod intersects? [::circle ::circle] [a b]
  (let [d (Math/sqrt (+ (Math/pow (- (:x a) (:x b)) 2)
                         (Math/pow (- (:y a) (:y b)) 2)))]
    (< d (+ (:radius a) (:radius b)))))

;; Default for any combination
(defmethod intersects? :default [a b]
  ;; Fallback using bounding boxes
  (bounding-box-intersect? a b))

;; Prefer specificity
(prefer-method intersects? [::square ::square] [::rectangle ::rectangle])

;; Dynamic dispatch based on runtime state
(defmulti payment-handler
  (fn [order]
    (cond
      (:crypto-payment? order) :crypto
      (> (:total order) 100000) :large-amount
      :else :standard)))

(defmethod payment-handler :crypto [order]
  (crypto-payment/process! order))

(defmethod payment-handler :large-amount [order]
  (manual-review/submit! order))

(defmethod payment-handler :standard [order]
  (stripe/charge! order))
```

---

## ขั้นตอนที่ 1445: Protocol Composition

```clojure
;; Build complex behaviors from simple protocols

(defprotocol Identifiable
  (id [this]))

(defprotocol Timestamped
  (created-at [this])
  (updated-at [this]))

(defprotocol Auditable
  (audit-log    [this])
  (with-audit   [this entry]))

(defprotocol Searchable
  (to-index-doc [this])
  (search-fields [this]))

;; Compose into a complete entity
(defrecord Product [data audit-entries]
  Identifiable
  (id [_] (:id data))
  
  Timestamped
  (created-at [_] (:created-at data))
  (updated-at [_] (:updated-at data))
  
  Auditable
  (audit-log  [_] audit-entries)
  (with-audit [this entry]
    (update this :audit-entries conj
            (assoc entry :timestamp (java.time.Instant/now))))
  
  Searchable
  (to-index-doc [_]
    {:id          (:id data)
     :title       (:name data)
     :description (:description data)
     :category    (:category data)
     :price       (:price data)})
  (search-fields [_] [:name :description :category])
  
  clojure.lang.ILookup
  (valAt [_ k] (get data k))
  (valAt [_ k not-found] (get data k not-found))
  
  Object
  (toString [_] (str "Product[" (:id data) ":" (:name data) "]")))

(defn create-product [data]
  (->Product (assoc data :id (java.util.UUID/randomUUID)
                         :created-at (java.time.Instant/now))
              []))
```

---

## Project: Plugin System ด้วย Protocols

```clojure
(ns myapp.plugin-system)

;; Plugin protocol
(defprotocol Plugin
  (plugin-name    [this])
  (plugin-version [this])
  (initialize!    [this config])
  (on-event       [this event-type data])
  (shutdown!      [this]))

;; Plugin registry
(def registry (atom {}))

(defn register-plugin! [plugin config]
  (initialize! plugin config)
  (swap! registry assoc (plugin-name plugin) plugin)
  (println "Plugin registered:" (plugin-name plugin) (plugin-version plugin)))

(defn dispatch-event! [event-type data]
  (doseq [[_ plugin] @registry]
    (try
      (on-event plugin event-type data)
      (catch Exception e
        (println "Plugin error:" (plugin-name plugin) (.getMessage e))))))

;; Example plugins
(defrecord EmailPlugin [config mailer]
  Plugin
  (plugin-name    [_] "email")
  (plugin-version [_] "1.0.0")
  (initialize!    [_ cfg] (reset! config cfg))
  (on-event       [_ :order-placed data]
    (mailer/send! (:user-email data)
                   "Order Confirmed"
                   (email-template :order-confirmed data)))
  (on-event       [_ _ _] nil)  ; ignore other events
  (shutdown!      [_] nil))

(defrecord AnalyticsPlugin [buffer]
  Plugin
  (plugin-name    [_] "analytics")
  (plugin-version [_] "2.1.0")
  (initialize!    [_ _] (reset! buffer []))
  (on-event       [_ event-type data]
    (swap! buffer conj {:event event-type :data data :at (java.time.Instant/now)}))
  (shutdown!      [_]
    (analytics/flush! @buffer)))
```

---

*Part 49 จาก 100+ | ขั้นตอน 1441-1470 จาก 1000+*
