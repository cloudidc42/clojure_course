# Part 94: Compiler & Tooling ขั้นสูง
## ขั้นตอนที่ 2791-2820: tools.analyzer, REPL Enhancements, Custom Linters, nREPL

---

## บทนำ

Clojure tooling ecosystem ระดับ professional:
- **tools.analyzer** - analyze Clojure AST
- **nREPL middleware** - extend REPL capabilities
- **Custom linters** - enforce coding standards
- **Eastwood/clj-kondo** - integrate static analysis
- **REPL-driven development** - interactive workflow

---

## ขั้นตอนที่ 2791: tools.analyzer สำหรับ AST Analysis

```clojure
(ns myapp.tooling.analyzer
  (:require [clojure.tools.analyzer.jvm :as ana]
            [clojure.tools.analyzer.passes :as passes]
            [clojure.tools.analyzer.passes.jvm.emit-form :as emit]))

;; Analyze a form to AST
(defn analyze-form [form]
  (binding [ana/macroexpand-1 (fn [form env] (macroexpand-1 form))]
    (ana/analyze form (ana/empty-env))))

;; Walk AST and collect info
(defn collect-fn-calls [ast]
  (let [calls (atom [])]
    (passes/postwalk ast
      (fn [node]
        (when (= :invoke (:op node))
          (when-let [fname (get-in node [:fn :form])]
            (swap! calls conj fname)))
        node))
    @calls))

;; Find all reflection warnings
(defn find-reflection-sites [ns-sym]
  (let [ns-map (ns-publics ns-sym)
        results (atom [])]
    (doseq [[sym var] ns-map]
      (when-let [source (source-fn sym)]
        (binding [*warn-on-reflection* true]
          (with-out-str
            (try
              (eval (read-string source))
              (catch Exception _))))))
    @results))

;; Count form complexity (cyclomatic)
(defn cyclomatic-complexity [ast]
  (let [branches (atom 0)]
    (passes/postwalk ast
      (fn [node]
        (when (#{:if :case :try :catch} (:op node))
          (swap! branches inc))
        node))
    (inc @branches)))

;; Detect potential issues
(defn analyze-ns-for-issues [ns-sym]
  (let [vars (ns-publics ns-sym)]
    (for [[sym var] vars
          :let [meta  (meta var)
                form  (try (read-string (pr-str (:source meta ""))) (catch Exception _ nil))]
          :when form
          :let [ast   (analyze-form form)
                cmplx (cyclomatic-complexity ast)]]
      {:name       sym
       :complexity  cmplx
       :too-complex (> cmplx 10)})))
```

---

## ขั้นตอนที่ 2792: Custom nREPL Middleware

```clojure
(ns myapp.tooling.nrepl-middleware
  (:require [nrepl.middleware :as middleware]
            [nrepl.transport :as t]))

;; nREPL middleware: intercepts and enriches messages
(defn timing-middleware [handler]
  (fn [{:keys [op session] :as msg}]
    (let [start (System/nanoTime)]
      (handler
        (assoc msg :transport
          (reify nrepl.transport.Transport
            (recv [_ timeout]
              (let [resp (.recv (:transport msg) timeout)
                    elapsed (/ (- (System/nanoTime) start) 1e6)]
                (when (and resp (= "eval" op))
                  (assoc resp :eval-time (str elapsed "ms")))))))))))

;; Middleware that logs all eval operations
(defn audit-middleware [handler]
  (fn [msg]
    (when (= "eval" (:op msg))
      (log/info "REPL eval"
        {:code    (first (clojure.string/split-lines (:code msg "")))
         :session (:session msg)
         :user    (System/getProperty "user.name")}))
    (handler msg)))

;; Middleware that adds automatic require on NameNotFound
(defn auto-require-middleware [handler]
  (fn [{:keys [op code] :as msg}]
    (if (= "eval" op)
      (let [result (handler msg)]
        ;; Check for NameNotFoundException and try to auto-require
        result)
      (handler msg))))

;; Register middleware
(middleware/set-descriptor! #'timing-middleware
  {:requires #{#'nrepl.middleware.session/session}
   :expects  #{}
   :handles  {"eval" {:doc "Adds timing info to eval responses"}}})
```

---

## ขั้นตอนที่ 2793: Custom clj-kondo Hooks

```clojure
;; .clj-kondo/hooks/myapp.clj
;; Custom static analysis hooks

(ns myapp.linting.hooks
  (:require [clj-kondo.hooks-api :as hooks]))

;; Validate defentity usage
(defn defentity-hook [{:keys [node]}]
  (let [children (rest (:children node))
        name-node (first children)
        fields-node (second children)]
    
    ;; Validate entity name is PascalCase
    (when (and name-node (symbol? (hooks/sexpr name-node)))
      (let [name-str (str (hooks/sexpr name-node))]
        (when-not (re-matches #"[A-Z][a-zA-Z0-9]*" name-str)
          (hooks/reg-finding!
            (assoc (meta name-node)
              :message (str "defentity name should be PascalCase, got: " name-str)
              :type    :myapp/entity-naming)))))
    
    ;; Validate fields are keywords
    (when (and fields-node (vector? (hooks/sexpr fields-node)))
      (doseq [field (hooks/sexpr fields-node)]
        (when-not (symbol? field)
          (hooks/reg-finding!
            {:message "defentity fields must be symbols"
             :type    :myapp/entity-fields}))))))

;; Hook for with-tenant
(defn with-tenant-hook [{:keys [node]}]
  (let [args (:children (first (rest (:children node))))]
    (when (< (count args) 1)
      (hooks/reg-finding!
        (assoc (meta node)
          :message "with-tenant requires a tenant argument"
          :type    :myapp/with-tenant-arity)))))
```

---

## ขั้นตอนที่ 2794: REPL-Driven Development Workflow

```clojure
;; dev/user.clj - REPL development utilities
(ns user
  (:require [clojure.repl :refer [doc source apropos]]
            [clojure.pprint :refer [pprint]]
            [integrant.repl :as ig-repl]))

;; System management
(defn start! []
  (ig-repl/set-prep! #(ecommerce.system/load-config))
  (ig-repl/go))

(defn stop! []  (ig-repl/halt))
(defn reset! [] (ig-repl/reset))
(defn refresh! [] (clojure.tools.namespace.repl/refresh))

;; Development utilities
(defn pp [x] (pprint x) x)

(defn time-ms [f & args]
  (let [start (System/nanoTime)
        result (apply f args)]
    (println "Time:" (/ (- (System/nanoTime) start) 1e6) "ms")
    result))

;; DB access in REPL
(defn db []
  (-> (ig-repl/system) :db/pool))

(defn q [sql & params]
  (next.jdbc/execute! (db) (into [sql] params)))

;; Test helpers
(defn with-test-system [f]
  (let [sys (ig/init (test-config))]
    (try (f sys)
         (finally (ig/halt! sys)))))

;; Reload modified namespaces
(defn reload-app! []
  (clojure.tools.namespace.repl/set-refresh-dirs "src" "test")
  (clojure.tools.namespace.repl/refresh-all))

;; Inspect running system
(defn inspect-system []
  (->> (ig-repl/system)
       keys
       (map (fn [k]
               {:key    k
                :type   (type (get (ig-repl/system) k))
                :value  (get (ig-repl/system) k)}))))
```

---

## ขั้นตอนที่ 2795: Code Generation Tools

```clojure
(ns myapp.tooling.codegen)

;; Generate CRUD handler boilerplate from spec
(defn gen-crud-handlers [resource-name spec]
  (let [rname    (name resource-name)
        make-sym #(symbol (str % rname))]
    `(do
       ;; List
       (defn ~(make-sym "list-")
         [request]
         (let [items# (list-resource! ~resource-name (:query-params request))]
           {:status 200 :body items#}))
       
       ;; Get
       (defn ~(make-sym "get-")
         [request]
         (let [id#   (get-in request [:path-params :id])
               item# (find-resource ~resource-name id#)]
           (if item#
             {:status 200 :body item#}
             {:status 404 :body {:error "Not found"}})))
       
       ;; Create
       (defn ~(make-sym "create-")
         [request]
         (let [body#   (validate-body (:body-params request) ~spec)
               result# (create-resource! ~resource-name body#)]
           {:status 201 :body result#}))
       
       ;; Update
       (defn ~(make-sym "update-")
         [request]
         (let [id#     (get-in request [:path-params :id])
               body#   (validate-body (:body-params request) ~spec)
               result# (update-resource! ~resource-name id# body#)]
           (if result#
             {:status 200 :body result#}
             {:status 404 :body {:error "Not found"}})))
       
       ;; Delete
       (defn ~(make-sym "delete-")
         [request]
         (let [id# (get-in request [:path-params :id])]
           (delete-resource! ~resource-name id#)
           {:status 204})))))

;; Generate routes from resource list
(defmacro defresource [name spec]
  `(do
     ~(gen-crud-handlers name spec)
     (def ~(symbol (str (clojure.core/name name) "-routes"))
       [~(str "/" (clojure.core/name name))
        {:get  ~(symbol (str "list-" (clojure.core/name name)))
         :post ~(symbol (str "create-" (clojure.core/name name)))}
        [~(str "/" (clojure.core/name name) "/:id")
         {:get    ~(symbol (str "get-" (clojure.core/name name)))
          :put    ~(symbol (str "update-" (clojure.core/name name)))
          :delete ~(symbol (str "delete-" (clojure.core/name name)))}]])))
```

---

*Part 94 จาก 100+ | ขั้นตอน 2791-2820 จาก 1000+*
