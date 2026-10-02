# Part 51: REPL-Driven Development
## ขั้นตอนที่ 1501-1530: Workflow, Debugging, Hot Reload, nREPL, CIDER

---

## บทนำ

REPL-Driven Development - หัวใจของ Clojure:
- **nREPL** - network REPL protocol
- **CIDER** - Emacs IDE integration
- **Calva** - VS Code integration  
- **Hot reload** - เปลี่ยน code ขณะ run
- **Debug patterns** - tap>, reveal, portal

---

## ขั้นตอนที่ 1501: nREPL Setup

```clojure
;; deps.edn
(ns user
  (:require [clojure.repl :refer :all]
            [clojure.pprint :refer [pprint]]
            [clojure.tools.namespace.repl :as repl-tools]))

;; Start nREPL server manually
(require '[nrepl.server :as nrepl])

(defonce nrepl-server
  (nrepl/start-server :port 7888
                       :bind "127.0.0.1"))

(println "nREPL server started on port 7888")

;; dev/user.clj - auto-loaded dev namespace
(defn start  [] (integrant.repl/go))
(defn stop   [] (integrant.repl/halt))
(defn restart [] (integrant.repl/reset))
(defn refresh [] (repl-tools/refresh))

;; Auto-refresh on file change
(repl-tools/set-refresh-dirs "src" "test" "dev")
```

---

## ขั้นตอนที่ 1502: Debugging ด้วย tap>

```clojure
;; tap> - send values to registered taps
;; Non-blocking, doesn't break program flow

(add-tap (fn [v]
           (println "TAP:" (pr-str v))))

;; Use in production-like code
(defn process-order [order]
  (tap> {:event :process-start :order-id (:id order)})
  (let [validated (validate order)]
    (tap> {:event :validated :result validated})
    (if (:valid? validated)
      (let [result (save-order! validated)]
        (tap> {:event :saved :result result})
        result)
      (do
        (tap> {:event :invalid :errors (:errors validated)})
        (throw (ex-info "Invalid order" validated))))))

;; Portal: visual tap explorer
;; deps.edn: {:deps {djblue/portal {:mvn/version "0.55.0"}}}
(require '[portal.api :as p])

(def portal (p/open))
(add-tap #'p/submit)

;; Now all tap> values appear in Portal UI
(tap> {:test "Hello Portal!"})
(tap> [1 2 3 {:a :b}])

;; Remove tap when done
(remove-tap #'p/submit)
```

---

## ขั้นตอนที่ 1503: REPL Utilities

```clojure
;; Useful REPL helpers

;; Spy: print and return value
(defmacro spy [form]
  `(let [result# ~form]
     (println "SPY" '~form "=>" (pr-str result#))
     result#))

;; Usage
(spy (+ 1 2))     ; prints "SPY (+ 1 2) => 3", returns 3
(spy (:name user)) ; prints "SPY (:name user) => \"Alice\"", returns "Alice"

;; Time: print elapsed time
(defmacro time-it [label form]
  `(let [start# (System/nanoTime)
          result# ~form
          ms# (/ (- (System/nanoTime) start#) 1e6)]
     (printf "%s took %.2f ms%n" ~label ms#)
     result#))

;; Inspect data structures
(defn keys-at-depth
  "Show all keys at every level of a nested map"
  [m depth]
  (when (and (map? m) (> depth 0))
    (println (str (apply str (repeat (- 3 depth) "  "))
                   (keys m)))
    (doseq [v (vals m)]
      (keys-at-depth v (dec depth)))))

;; Find which namespace defines a symbol
(defn find-ns-for [sym]
  (filter #(ns-resolve % sym) (all-ns)))

;; Show a function's source
(source clojure.string/join)

;; Show docstring
(doc clojure.string/join)

;; Find all functions matching a pattern
(apropos #"string")
```

---

## ขั้นตอนที่ 1504: Hot Reload Workflow

```clojure
;; dev/user.clj - development workflow

(ns user
  (:require [integrant.repl :as ig-repl]
            [integrant.repl.state :as state]
            [clojure.tools.namespace.repl :as tools-ns]
            [myapp.system :as system]))

;; Configure system for dev
(ig-repl/set-prep!
  (fn []
    (-> (aero/read-config (io/resource "config.edn")
                           {:profile :dev})
        system/prepare-config)))

(defn dev-system [] state/system)
(defn dev-db     [] (:app/db (dev-system)))

;; Useful shortcuts
(def go    ig-repl/go)
(def halt  ig-repl/halt)
(def reset ig-repl/reset)  ; stop + refresh namespaces + start

;; REPL workflow:
;; 1. Start: (go)
;; 2. Edit code in editor
;; 3. Reload: (reset)
;; 4. Test in REPL: (find-user (dev-db) "alice@example.com")
;; 5. Edit again...

;; Namespace reload without restarting system
(defn reload-ns [ns-sym]
  (tools-ns/refresh-dirs "src/clj")
  (require ns-sym :reload))

;; Watch files and auto-reload
(require '[hawk.core :as hawk])

(hawk/watch! [{:paths   ["src/clj"]
               :handler (fn [ctx e]
                          (when (= :modify (:kind e))
                            (println "Reloading..." (:file e))
                            (tools-ns/refresh))
                          ctx)}])
```

---

## ขั้นตอนที่ 1505: REPL-Driven Testing

```clojure
;; Test from REPL without test runner

;; Quick inline tests
(assert (= 4 (+ 2 2)))
(assert (= "hello" (clojure.string/lower-case "HELLO")))

;; Test a specific function
(require '[clojure.test :refer [is]])

(is (= {:a 1 :b 2}
       (merge {:a 0} {:a 1 :b 2})))

;; Run tests for specific namespace
(require '[clojure.test :as test])
(test/run-tests 'myapp.products-test)

;; Run just one test
(test/test-var #'myapp.products-test/test-create-product)

;; Interactive debugging helper
(defn debug-call [f & args]
  (println "Calling:" f)
  (println "Args:" args)
  (let [result (try
                 (apply f args)
                 (catch Exception e
                   {:error (.getMessage e) :type (type e)}))]
    (println "Result:")
    (pprint result)
    result))

;; Explore live system state
(defn inspect-db []
  (let [db (dev-db)]
    {:tables     (jdbc/execute! db ["SELECT table_name FROM information_schema.tables
                                     WHERE table_schema = 'public'"])
     :row-counts (jdbc/execute! db ["SELECT relname, n_live_tup FROM pg_stat_user_tables
                                     ORDER BY n_live_tup DESC LIMIT 10"])}))
```

---

## ขั้นตอนที่ 1506: Reveal และ Flowstorm Debugging

```clojure
;; Reveal: interactive data browser
;; deps.edn: {:deps {vlaaad/reveal {:mvn/version "1.3.282"}}}

(require '[vlaaad.reveal :as reveal])

;; Open Reveal window
(reveal/repl)

;; All evaluated values appear in Reveal
;; Can explore nested structures visually

;; FlowStorm: time-travel debugger
;; deps.edn: {:deps {com.github.flow-storm/flow-storm-dbg {:mvn/version "3.7.6"}}}

(require '[flow-storm.api :as fs-api])

;; Start FlowStorm
(fs-api/local-connect)

;; Instrument namespaces to trace
(fs-api/instrument-namespaces-clj
  #{"myapp.products" "myapp.orders"}
  {:disable-events? false})

;; Now run your code and see every step in FlowStorm UI
(place-order! db test-order)

;; See all function calls, values at each step
;; Time-travel: go back to any point
```

---

## Project: Dev Utilities Namespace

```clojure
;; dev/user.clj - complete development utilities

(ns user
  (:require [clojure.repl :refer :all]
            [clojure.pprint :refer [pprint pp]]
            [clojure.reflect :as reflect]
            [clojure.string :as str]
            [next.jdbc :as jdbc]
            [taoensso.carmine :as car]))

;; ===== System Management =====
(defonce system (atom nil))

(defn start! []
  (when @system (stop!))
  (reset! system (myapp.system/start! :dev))
  (println "System started"))

(defn stop! []
  (when @system
    (myapp.system/stop! @system)
    (reset! system nil)
    (println "System stopped")))

(defn restart! [] (stop!) (start!))

;; ===== Quick DB Access =====
(defn db [] (:db @system))
(defn redis [] (:redis @system))

(defn q [sql & params]
  (jdbc/execute! (db) (into [sql] params)))

(defn q1 [sql & params]
  (jdbc/execute-one! (db) (into [sql] params)))

;; ===== Fixtures =====
(defn create-test-user! []
  (q1 "INSERT INTO users (id, name, email, role)
       VALUES (gen_random_uuid(), 'Test User', 'test@example.com', 'admin')
       RETURNING *"))

(defn clear-test-data! []
  (q "DELETE FROM orders WHERE user_id IN (SELECT id FROM users WHERE email LIKE '%test%')")
  (q "DELETE FROM users WHERE email LIKE '%test%'"))

;; ===== Inspection =====
(defn show-routes []
  (doseq [r (myapp.routes/all-routes)]
    (println (:method r) (:path r))))

(defn show-ns-publics [ns-sym]
  (->> (ns-publics ns-sym)
       (sort-by first)
       (map (fn [[sym var]]
               {:name sym :doc (:doc (meta var))}))
       pprint))
```

---

*Part 51 จาก 100+ | ขั้นตอน 1501-1530 จาก 1000+*
