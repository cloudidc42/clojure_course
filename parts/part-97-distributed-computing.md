# Part 97: Distributed Computing ขั้นสูง
## ขั้นตอนที่ 2881-2910: Consensus, Consistent Hashing, Gossip, Service Mesh

---

## บทนำ

Distributed systems ระดับ production:
- **Raft consensus** - leader election และ log replication
- **Consistent hashing** - distribute load evenly
- **Gossip protocol** - decentralized state propagation
- **Vector clocks** - causal ordering
- **Service mesh** - observability and traffic management

---

## ขั้นตอนที่ 2881: Consistent Hashing Ring

```clojure
(ns myapp.distributed.hashing)

;; Consistent hashing: distribute keys across nodes
;; with minimal remapping when nodes join/leave

(defn fnv1a-hash [s]
  (let [offset-basis 2166136261
        prime        16777619]
    (reduce (fn [hash b]
              (bit-and 0xFFFFFFFF
                (unchecked-multiply prime (bit-xor hash b))))
            offset-basis
            (.getBytes s "UTF-8"))))

(defrecord HashRing [virtual-nodes ring])

(defn make-ring [nodes replicas]
  (let [virtual-nodes
        (into (sorted-map)
              (for [node   nodes
                    replica (range replicas)]
                [(fnv1a-hash (str node ":" replica)) node]))]
    (->HashRing (* (count nodes) replicas) virtual-nodes)))

(defn find-node [^HashRing ring key]
  (let [hash   (fnv1a-hash key)
        ring   (:ring ring)
        entry  (first (subseq ring >= hash))]
    (if entry
      (val entry)
      (val (first ring)))))  ; Wrap around

(defn add-node [ring node replicas]
  (let [new-entries (into {} (for [r (range replicas)]
                               [(fnv1a-hash (str node ":" r)) node]))]
    (update ring :ring merge new-entries)))

(defn remove-node [ring node replicas]
  (let [to-remove (set (map #(fnv1a-hash (str node ":" %))
                             (range replicas)))]
    (update ring :ring #(apply dissoc % to-remove))))

;; Find N nodes for replication
(defn find-n-nodes [ring key n]
  (let [hash  (fnv1a-hash key)
        ring  (:ring ring)
        all   (concat (subseq ring >= hash)
                       ring)
        unique-nodes (distinct (map val all))]
    (take n unique-nodes)))
```

---

## ขั้นตอนที่ 2882: Gossip Protocol

```clojure
(ns myapp.distributed.gossip
  (:require [clojure.core.async :as async]))

;; Gossip: eventually consistent state propagation
;; Each node periodically shares its state with random peers

(defrecord GossipNode [node-id state peers heartbeat])

(defn make-gossip-node [node-id initial-state known-peers]
  (->GossipNode
    node-id
    (atom (merge initial-state
                  {node-id {:alive?    true
                              :heartbeat 0
                              :updated   (System/currentTimeMillis)}}))
    (atom known-peers)
    (atom 0)))

;; Merge state (CRDT-like: take latest by heartbeat)
(defn merge-states [local remote]
  (merge-with
    (fn [l r]
      (if (> (:heartbeat r) (:heartbeat l)) r l))
    local remote))

;; Gossip to random peer
(defn gossip! [node transport]
  (when-let [peer (rand-nth (seq @(:peers node)))]
    (transport/send! transport peer
      {:type  :gossip
       :from  (:node-id node)
       :state @(:state node)})))

;; Receive gossip
(defn receive-gossip! [node {:keys [from state]}]
  (swap! (:state node) merge-states state)
  ;; Add sender to known peers
  (swap! (:peers node) conj from))

;; Heartbeat: mark self as alive
(defn heartbeat! [node]
  (swap! (:heartbeat node) inc)
  (swap! (:state node) update (:node-id node)
    #(assoc % :heartbeat @(:heartbeat node)
              :updated   (System/currentTimeMillis))))

;; Failure detection: nodes not heard from > threshold are suspect
(defn detect-failures [node failure-threshold-ms]
  (let [now    (System/currentTimeMillis)
        states @(:state node)]
    (->> states
         (filter (fn [[id info]]
                    (and (not= id (:node-id node))
                         (> (- now (:updated info)) failure-threshold-ms))))
         (map first))))

;; Start gossip loop
(defn start-gossip-loop! [node transport interval-ms]
  (let [stop! (atom false)]
    (async/go-loop []
      (when-not @stop!
        (heartbeat! node)
        (gossip! node transport)
        (async/<! (async/timeout interval-ms))
        (recur)))
    #(reset! stop! true)))
```

---

## ขั้นตอนที่ 2883: Vector Clocks

```clojure
;; Vector clocks: track causality in distributed systems

(defn make-vclock [node-id]
  {node-id 0})

(defn increment [vclock node-id]
  (update vclock node-id (fnil inc 0)))

(defn merge-vclocks [vc1 vc2]
  (merge-with max vc1 vc2))

(defn happens-before? [vc1 vc2]
  "Returns true if vc1 causally precedes vc2"
  (and
    ;; All entries in vc1 are <= corresponding entries in vc2
    (every? (fn [[node tick]]
               (<= tick (get vc2 node 0)))
             vc1)
    ;; At least one entry is strictly less
    (some (fn [[node tick]]
             (< tick (get vc2 node 0)))
          vc1)))

(defn concurrent? [vc1 vc2]
  "Events are concurrent if neither happened before the other"
  (not (or (happens-before? vc1 vc2)
            (happens-before? vc2 vc1))))

;; Versioned value with vector clock
(defrecord VersionedValue [value vclock node-id])

(defn write-value [state key value node-id]
  (let [old-vc   (get-in state [key :vclock] {})
        new-vc   (increment (merge-vclocks old-vc {node-id 0}) node-id)]
    (assoc state key (->VersionedValue value new-vc node-id))))

(defn resolve-conflict [v1 v2]
  "Resolve concurrent writes (last-write-wins by node-id)"
  (cond
    (happens-before? (:vclock v1) (:vclock v2)) v2
    (happens-before? (:vclock v2) (:vclock v1)) v1
    ;; Concurrent: application-specific resolution
    :else (if (> (hash (:node-id v1)) (hash (:node-id v2))) v1 v2)))
```

---

## ขั้นตอนที่ 2884: Distributed Transactions (Two-Phase Commit)

```clojure
;; 2PC: coordinator ensures all-or-nothing across services

(defprotocol TwoPhaseParticipant
  (prepare! [this tx-id data])
  (commit!  [this tx-id])
  (rollback! [this tx-id]))

;; Coordinator
(defn execute-2pc! [coordinator-db participants data]
  (let [tx-id (str (java.util.UUID/randomUUID))]
    
    ;; Phase 1: Prepare
    (let [votes (mapv (fn [p]
                          (try
                            (prepare! p tx-id data)
                            :yes
                            (catch Exception _
                              :no)))
                        participants)]
      
      (if (every? #{:yes} votes)
        ;; Phase 2a: All voted yes → commit
        (do
          (record-decision! coordinator-db tx-id :commit)
          (doseq [p participants]
            (try
              (commit! p tx-id)
              (catch Exception e
                ;; Log but continue - participant will recover
                (log/error "Commit failed" {:participant p :error (.getMessage e)}))))
          {:status :committed :tx-id tx-id})
        
        ;; Phase 2b: Some voted no → abort
        (do
          (record-decision! coordinator-db tx-id :abort)
          (doseq [p participants]
            (try (rollback! p tx-id) (catch Exception _)))
          {:status :aborted :tx-id tx-id})))))
```

---

## ขั้นตอนที่ 2885: Service Mesh Integration

```clojure
;; Istio/Envoy integration: handle retries, circuit breaking at mesh level

;; Propagate distributed trace headers
(defn wrap-mesh-headers [handler]
  (fn [request]
    (let [trace-headers (select-keys (:headers request)
                          ["x-request-id" "x-b3-traceid" "x-b3-spanid"
                           "x-b3-parentspanid" "x-b3-sampled" "x-b3-flags"])]
      (binding [*trace-headers* trace-headers]
        (handler request)))))

;; Add tracing headers to outgoing requests
(defn with-trace-headers [http-opts]
  (update http-opts :headers merge *trace-headers*))

;; Health check for Kubernetes liveness/readiness probes
(defn liveness-handler [_]
  {:status 200 :body {:status "alive"}})

(defn readiness-handler [system]
  (fn [_]
    (let [db-ok?    (db-healthy? (:db system))
          cache-ok? (redis-healthy? (:cache system))]
      (if (and db-ok? cache-ok?)
        {:status 200 :body {:status "ready"}}
        {:status 503 :body {:status "not-ready"
                             :checks {:db    db-ok?
                                      :cache cache-ok?}}}))))

;; Graceful shutdown for pod termination
(defn wrap-shutdown-probe [handler]
  (fn [request]
    (if @shutting-down?
      {:status  503
       :headers {"Retry-After" "10"}
       :body    "Service shutting down"}
      (handler request))))
```

---

*Part 97 จาก 100+ | ขั้นตอน 2881-2910 จาก 1000+*
