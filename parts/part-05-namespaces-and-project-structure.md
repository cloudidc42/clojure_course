# Part 5: Namespaces และ Project Structure
## ขั้นตอนที่ 121-150: Organization และ Architecture

---

## บทนำ

Namespaces ใน Clojure คือระบบ Module ที่ทำให้:
- จัดการ code ขนาดใหญ่ได้
- ป้องกัน naming conflicts
- สร้าง clean API สำหรับ library
- ทำงานร่วมกันเป็น team ได้ง่าย

---

## ขั้นตอนที่ 121: Namespaces พื้นฐาน

```clojure
;; ประกาศ namespace ที่ต้น file
(ns my-app.core)

;; ชื่อ namespace สัมพันธ์กับ path ของ file:
;; my-app.core → src/my_app/core.clj
;;              (dot → slash, hyphen → underscore)

;; ns ที่ซับซ้อน
(ns my-app.users.service
  (:require [clojure.string :as str]
            [my-app.db :as db]
            [my-app.users.repository :as repo])
  (:import [java.util Date UUID]
           [java.time LocalDateTime])
  (:gen-class))
```

### File Naming Rules

```
Namespace              File Path
-----------            ---------
my-app.core         → src/my_app/core.clj
my.complex-name     → src/my/complex_name.clj
com.example.api     → src/com/example/api.clj

Rules:
  . (dot)     → / (directory separator)
  - (hyphen)  → _ (underscore in filename)
```

---

## ขั้นตอนที่ 122: require - โหลด Namespaces

```clojure
;; วิธีที่ 1: เรียบง่าย (ต้องใส่ qualified name)
(require 'clojure.string)
(clojure.string/upper-case "hello")   ; => "HELLO"

;; วิธีที่ 2: alias (แนะนำมากที่สุด!)
(require '[clojure.string :as str])
(str/upper-case "hello")              ; => "HELLO"

;; วิธีที่ 3: refer (ใช้ชื่อตรงๆ ไม่ต้อง alias)
(require '[clojure.string :refer [upper-case lower-case]])
(upper-case "hello")                  ; => "HELLO"

;; วิธีที่ 4: refer :all (ไม่แนะนำ!)
(require '[clojure.string :refer :all])
;; ปัญหา: ไม่รู้ว่า function มาจากไหน

;; ใน ns form (แนะนำสำหรับ production code)
(ns my-app.core
  (:require [clojure.string :as str]
            [clojure.set :as set]
            [my-app.db :as db]
            [my-app.utils :refer [helper-fn]]))

;; Reload สำหรับ development
(require '[my-app.core :as app] :reload)
(require '[my-app.core :as app] :reload-all)
```

---

## ขั้นตอนที่ 123: import - Java Classes

```clojure
(ns my-app.core
  (:import [java.util Date UUID ArrayList]
           [java.io File InputStream]
           [java.time LocalDateTime ZonedDateTime]
           [java.time.format DateTimeFormatter]))

;; ใช้งาน
(Date.)                          ; new Date()
(UUID/randomUUID)               ; UUID.randomUUID()
(LocalDateTime/now)             ; LocalDateTime.now()

;; Import แบบ inline
(import java.util.Date)
(Date.)

;; คำนวณกับ Java types
(def now (LocalDateTime/now))
(.getYear now)
(.getMonthValue now)
(.getDayOfMonth now)
```

---

## ขั้นตอนที่ 124: Namespace Visibility

```clojure
;; Public (default)
(defn public-fn [x] (* x 2))
(def public-var 42)

;; Private (ใช้ defn- หรือ :private metadata)
(defn- private-fn [x] (* x 3))

(def ^:private private-var 99)

;; private ไม่สามารถเรียกจาก namespace อื่นได้
;; (my.ns/private-fn 5) ; Error!

;; ยังคงสามารถเรียกได้จาก namespace เดิม
;; และจาก REPL ด้วย @#'my.ns/private-fn

;; Dynamic vars
(def ^:dynamic *config* {})

;; Constant
(def ^:const MAX-SIZE 1000)

;; ดู all public vars ใน namespace
(ns-publics 'my-app.core)

;; ดู all vars รวม private
(ns-interns 'my-app.core)
```

---

## ขั้นตอนที่ 125: Project Structure - Leiningen

```
my-app/
├── project.clj                 # Project configuration
├── README.md
├── .gitignore
├── src/
│   └── my_app/                 # main source
│       ├── core.clj            # Entry point
│       ├── config.clj          # Configuration
│       ├── db/
│       │   ├── core.clj        # DB connection
│       │   └── migrations.clj  # DB migrations
│       ├── models/
│       │   ├── user.clj        # User model
│       │   └── product.clj     # Product model
│       ├── api/
│       │   ├── core.clj        # API setup
│       │   ├── users.clj       # Users API
│       │   └── products.clj    # Products API
│       └── utils/
│           ├── string.clj      # String utilities
│           └── date.clj        # Date utilities
├── test/
│   └── my_app/                 # tests mirror src structure
│       ├── core_test.clj
│       ├── models/
│       │   └── user_test.clj
│       └── api/
│           └── users_test.clj
├── dev/
│   └── user.clj               # REPL helpers
└── resources/
    ├── config.edn             # Configuration files
    └── migrations/            # SQL migration files
```

---

## ขั้นตอนที่ 126: project.clj ขั้นสูง

```clojure
;; project.clj
(defproject my-app "1.0.0"
  :description "My Clojure Application"
  :url "https://github.com/me/my-app"
  :license {:name "MIT"}
  
  ;; Dependencies
  :dependencies [[org.clojure/clojure "1.12.0"]
                 [ring/ring-core "1.12.0"]
                 [ring/ring-jetty-adapter "1.12.0"]
                 [metosin/reitit "0.7.2"]
                 [com.github.seancorfield/next.jdbc "1.3.939"]
                 [org.postgresql/postgresql "42.7.4"]
                 [metosin/malli "0.16.4"]
                 [aero/aero "1.1.6"]
                 [com.taoensso/timbre "6.6.1"]]
  
  ;; Source paths
  :source-paths ["src"]
  :test-paths ["test"]
  
  ;; Main entry point
  :main my-app.core
  
  ;; JVM options
  :jvm-opts ["-Xmx512m" "-server"]
  
  ;; Profiles
  :profiles
  {:dev {:dependencies [[ring/ring-mock "0.4.0"]
                        [criterium "0.4.6"]
                        [clj-commons/clj-http-fake "1.0.4"]]
         :source-paths ["dev"]
         :jvm-opts ["-Xmx2g"]}
   
   :test {:dependencies [[io.github.cognitect-labs/test-runner {:sha "..."}]]}
   
   :uberjar {:aot :all
             :main my-app.core
             :jvm-opts ["-Dclojure.compiler.direct-linking=true"]}
   
   :production {:resource-paths ["resources/prod"]}}
  
  ;; Plugins
  :plugins [[lein-ring "0.12.6"]
            [lein-environ "1.2.0"]
            [lein-ancient "0.7.0"]])
```

---

## ขั้นตอนที่ 127: deps.edn ขั้นสูง

```clojure
;; deps.edn
{:deps
 {org.clojure/clojure {:mvn/version "1.12.0"}
  ring/ring-core {:mvn/version "1.12.0"}
  metosin/reitit {:mvn/version "0.7.2"}
  com.github.seancorfield/next.jdbc {:mvn/version "1.3.939"}
  metosin/malli {:mvn/version "0.16.4"}}
 
 :paths ["src" "resources"]
 
 :aliases
 {;; Development
  :dev
  {:extra-paths ["dev" "test"]
   :extra-deps
   {nrepl/nrepl {:mvn/version "1.3.0"}
    cider/cider-nrepl {:mvn/version "0.50.2"}
    com.bhauman/rebel-readline {:mvn/version "0.1.4"}
    djblue/portal {:mvn/version "0.58.0"}}}
  
  ;; Testing
  :test
  {:extra-paths ["test"]
   :extra-deps
   {io.github.cognitect-labs/test-runner
    {:git/url "https://github.com/cognitect-labs/test-runner.git"
     :sha "9e25b8e8fb659e9d5d96e56e0b95b45c98ce8e33"}}}
  
  ;; Build uberjar
  :build
  {:deps {io.github.clojure/tools.build {:mvn/version "0.10.5"}}
   :ns-default build}
  
  ;; Run
  :run
  {:main-opts ["-m" "my-app.core"]}
  
  ;; Start REPL
  :repl
  {:main-opts ["-m" "nrepl.cmdline"
               "--middleware" "[cider.nrepl/cider-middleware]"]}}}
```

---

## ขั้นตอนที่ 128: build.clj - Build Script

```clojure
;; build.clj (ใช้กับ deps.edn)
(ns build
  (:require [clojure.tools.build.api :as b]))

(def lib 'my-org/my-app)
(def version "1.0.0")
(def class-dir "target/classes")
(def basis (b/create-basis {:project "deps.edn"}))
(def uber-file (format "target/%s-%s-standalone.jar" (name lib) version))

(defn clean [_]
  (b/delete {:path "target"}))

(defn compile-clj [_]
  (b/compile-clj {:basis basis
                  :src-dirs ["src"]
                  :class-dir class-dir}))

(defn uber [_]
  (clean nil)
  (b/copy-dir {:src-dirs ["src" "resources"]
               :target-dir class-dir})
  (b/compile-clj {:basis basis
                  :src-dirs ["src"]
                  :class-dir class-dir})
  (b/uber {:class-dir class-dir
           :uber-file uber-file
           :basis basis
           :main 'my-app.core}))

;; รัน: clj -T:build uber
```

---

## ขั้นตอนที่ 129: Configuration Management

```clojure
;; resources/config.edn
{:server {:port 8080
          :host "0.0.0.0"}
 :db {:url "jdbc:postgresql://localhost/mydb"
      :user "postgres"
      :password "secret"
      :pool-size 10}
 :redis {:host "localhost"
         :port 6379}
 :app {:secret "change-me-in-production"
       :name "My App"}}

;; src/my_app/config.clj
(ns my-app.config
  (:require [clojure.edn :as edn]
            [clojure.java.io :as io]))

(defn load-config
  ([]
   (load-config "config.edn"))
  ([filename]
   (-> (io/resource filename)
       slurp
       edn/read-string)))

;; Environment-aware config ด้วย Aero
;; project.clj: [aero/aero "1.1.6"]

(require '[aero.core :as aero])

;; resources/config.edn ด้วย Aero
{:server {:port #or [#env PORT 8080]}
 :db {:url #or [#env DATABASE_URL "jdbc:postgresql://localhost/mydb"]
      :password #env DB_PASSWORD}}

;; โหลด
(defn load-config []
  (aero/read-config (io/resource "config.edn")
                    {:profile (keyword (or (System/getenv "APP_ENV") "dev"))}))
```

---

## ขั้นตอนที่ 130: Dependency Injection Pattern

```clojure
;; ไม่มี global state - ส่ง dependencies ผ่าน function
;; ทำให้ test ได้ง่ายกว่า

;; ❌ Global state (hard to test)
(def db (create-db-connection config))

(defn get-user [id]
  (query db "SELECT * FROM users WHERE id = ?" [id]))

;; ✅ Injected dependencies (easy to test)
(defn get-user [db id]
  (query db "SELECT * FROM users WHERE id = ?" [id]))

;; ✅ Component pattern ด้วย map
(defn create-user-service [deps]
  {:get-user     (fn [id] (get-user (:db deps) id))
   :create-user  (fn [data] (create-user (:db deps) data))
   :update-user  (fn [id data] (update-user (:db deps) id data))})

;; bootstrap
(def system
  (let [config (load-config)
        db (create-db (:db config))
        user-service (create-user-service {:db db})]
    {:db db
     :user-service user-service
     :config config}))

;; ใช้งาน
((:get-user (:user-service system)) 1)
```

---

## ขั้นตอนที่ 131: Component / Integrant / Mount

```clojure
;; === Integrant - Lifecycle Management ===
;; [integrant/integrant "0.11.0"]

(require '[integrant.core :as ig])

;; config
(def config
  {:db/connection {:url "jdbc:postgresql://localhost/myapp"
                   :pool-size 10}
   
   :server/http {:port 8080
                 :handler (ig/ref :app/routes)}
   
   :app/routes {:db (ig/ref :db/connection)}})

;; Define how to start each component
(defmethod ig/init-key :db/connection [_ {:keys [url pool-size]}]
  (println "Starting DB connection...")
  (create-connection-pool url pool-size))

(defmethod ig/init-key :server/http [_ {:keys [port handler]}]
  (println "Starting HTTP server on port" port)
  (start-server handler port))

(defmethod ig/init-key :app/routes [_ {:keys [db]}]
  (create-routes db))

;; Define how to stop each component
(defmethod ig/halt-key! :db/connection [_ connection]
  (println "Closing DB connection...")
  (.close connection))

(defmethod ig/halt-key! :server/http [_ server]
  (println "Stopping HTTP server...")
  (.stop server))

;; Start system
(def system (ig/init config))

;; Stop system
(ig/halt! system)
```

---

## ขั้นตอนที่ 132: Mount - Simple Lifecycle

```clojure
;; [mount/mount "0.1.18"]
(require '[mount.core :as mount :refer [defstate]])

;; Define stateful components
(defstate config
  :start (load-config)
  :stop (println "Config stopped"))

(defstate database
  :start (do
           (println "Connecting to DB...")
           (create-connection (:db-url config)))
  :stop (do
          (println "Disconnecting from DB...")
          (.close database)))

(defstate web-server
  :start (start-server database)
  :stop (stop-server web-server))

;; Start/Stop
(mount/start)
(mount/stop)

;; Start specific states
(mount/start #'config #'database)

;; For testing
(mount/start-with {#'database test-db})
```

---

## ขั้นตอนที่ 133: Namespace Conventions ใน Projects ขนาดใหญ่

```clojure
;; === Convention สำหรับ Web Application ===

;; my-app.core         - Entry point, system initialization
;; my-app.config       - Configuration loading
;; my-app.db           - Database connection
;; my-app.db.migrations - DB migrations

;; Domain layers:
;; my-app.users.model      - User data model (defrecord, specs)
;; my-app.users.repository - DB operations for users
;; my-app.users.service    - Business logic for users
;; my-app.users.handler    - HTTP handlers for users API

;; my-app.products.model
;; my-app.products.repository
;; my-app.products.service
;; my-app.products.handler

;; Shared utilities:
;; my-app.utils.string
;; my-app.utils.date
;; my-app.utils.validation

;; API
;; my-app.api.core     - API setup, middleware
;; my-app.api.router   - Route definitions

;; ตัวอย่าง users namespace
(ns my-app.users.service
  (:require [my-app.users.repository :as repo]
            [my-app.utils.validation :as validate]
            [clojure.tools.logging :as log]))

(defn create-user! [db user-data]
  (log/info "Creating user" (:email user-data))
  (if-let [error (validate/validate-user user-data)]
    {:error error}
    (let [user (repo/insert-user! db user-data)]
      {:success true :user user})))
```

---

## ขั้นตอนที่ 134: Circular Dependencies

```clojure
;; ❌ Circular dependency = error!
;; a.clj requires b.clj
;; b.clj requires a.clj
;; → CompilerException: Circular dependency

;; วิธีแก้ที่ 1: Refactor - หา common ground
;; แยก shared code ออกไปใน namespace ใหม่

;; my-app.users.model - shared types/specs (no dependencies)
;; my-app.users.repository - depends on model
;; my-app.users.service - depends on repository

;; วิธีแก้ที่ 2: Protocols
;; ประกาศ protocol ใน shared namespace
;; แต่ละ namespace implement protocol

;; วิธีแก้ที่ 3: Dependency injection
;; แทนที่จะ require กัน ส่งเป็น parameter

;; วิธีแก้ที่ 4: declare
(declare helper-fn)  ; forward declaration

(defn main-fn [x]
  (+ x (helper-fn x)))

(defn helper-fn [x]
  (* x 2))
```

---

## ขั้นตอนที่ 135: clojure.tools.namespace

```clojure
;; Development tool สำหรับ reload namespaces

;; project.clj
;; [org.clojure/tools.namespace "1.4.4"]

;; dev/user.clj
(ns user
  (:require [clojure.tools.namespace.repl :as repl]))

;; Reload namespaces ที่เปลี่ยนแปลง
(defn refresh []
  (repl/refresh))

;; Refresh ทุกอย่างตั้งแต่ต้น
(defn reset []
  (repl/clear)
  (repl/refresh))

;; ใน REPL
(refresh)
;; :reloading (my-app.utils my-app.db my-app.core)
;; :ok

;; Stop system ก่อน refresh (สำหรับ stateful apps)
(defn go []
  (mount/start))

(defn halt []
  (mount/stop))

(defn reset []
  (halt)
  (repl/refresh :after 'user/go))
```

---

## ขั้นตอนที่ 136: Public API Design

```clojure
;; src/my_library/core.clj
(ns my-library.core
  "Public API สำหรับ my-library
  
  ตัวอย่างการใช้งาน:
  
  (require '[my-library.core :as lib])
  
  (lib/create-client {:url \"http://api.example.com\"})
  (lib/fetch-data client {:query \"test\"})"
  (:require [my-library.internal.http :as http]
            [my-library.internal.auth :as auth]))

;; Public functions เท่านั้น!
(defn create-client
  "สร้าง client สำหรับ API
  
  Options:
  - :url     - base URL (required)
  - :timeout - timeout ใน milliseconds (default: 5000)
  - :api-key - API key สำหรับ authentication"
  [{:keys [url timeout api-key]
    :or {timeout 5000}}]
  {:pre [(string? url)]}
  {:url url
   :timeout timeout
   :headers (when api-key {"X-API-Key" api-key})})

(defn fetch-data [client params]
  "ดึงข้อมูลจาก API"
  (http/get client "/data" params))

;; Internal functions ใน my-library.internal.*
;; User ไม่ควรเรียกตรงๆ
```

---

## ขั้นตอนที่ 137: Macros ใน Namespaces

```clojure
;; ตั้งชื่อ macro ชัดเจน และ document
(defmacro defroute
  "ประกาศ route handler
  
  ตัวอย่าง:
  (defroute :get \"/users\" [request]
    {:status 200 :body (get-all-users)})"
  [method path params & body]
  `(register-route! ~method ~path
                    (fn ~params ~@body)))

;; ให้ users ใช้งาน macro จาก library
(ns my-app.api.routes
  (:require [my-lib.routing :refer [defroute]]))

(defroute :get "/users" [req]
  {:status 200 :body (get-all-users req)})
```

---

## ขั้นตอนที่ 138: Namespace Aliases Best Practices

```clojure
;; Convention ทั่วไปสำหรับ aliases
(ns my-app.core
  (:require
   ;; Standard library
   [clojure.string :as str]
   [clojure.set :as set]
   [clojure.walk :as walk]
   [clojure.zip :as zip]
   [clojure.pprint :as pp]
   [clojure.java.io :as io]
   [clojure.data.json :as json]
   
   ;; Database
   [next.jdbc :as jdbc]
   [honey.sql :as sql]
   
   ;; HTTP
   [ring.util.response :as resp]
   [reitit.ring :as ring]
   
   ;; Logging
   [clojure.tools.logging :as log]
   [taoensso.timbre :as timbre]
   
   ;; Time
   [java-time.api :as jt]
   
   ;; Validation
   [malli.core :as m]
   [clojure.spec.alpha :as s]
   
   ;; Internal
   [my-app.db :as db]
   [my-app.utils :as utils]))
```

---

## ขั้นตอนที่ 139: Resources และ Static Files

```clojure
;; โหลด resource จาก classpath
(require '[clojure.java.io :as io])

;; โหลด file
(slurp (io/resource "config.edn"))
(slurp (io/resource "sql/queries.sql"))

;; โหลด EDN
(require '[clojure.edn :as edn])
(def config
  (-> (io/resource "config.edn")
      slurp
      edn/read-string))

;; โหลด JSON
(require '[clojure.data.json :as json])
(def data
  (-> (io/resource "data/seed.json")
      slurp
      json/read-str))

;; โหลดหลายภาษา (i18n)
(def translations
  (-> (io/resource "i18n/th.edn")
      slurp
      edn/read-string))

(defn t [key & args]
  (if-let [template (get translations key)]
    (apply format template args)
    (str "MISSING:" (name key))))

(t :greeting "สมชาย")  ; => "สวัสดี สมชาย!"
```

---

## ขั้นตอนที่ 140: Dev-only Utilities

```clojure
;; dev/user.clj - พื้นที่สำหรับ development tools
(ns user
  (:require [clojure.tools.namespace.repl :as repl]
            [clojure.pprint :as pp]
            [my-app.core :as app]
            [my-app.db :as db]))

;; Quick helpers สำหรับ REPL
(defn start! []
  (app/start!)
  (println "System started"))

(defn stop! []
  (app/stop!)
  (println "System stopped"))

(defn reset! []
  (stop!)
  (repl/refresh :after 'user/start!))

;; Database helpers
(defn query [sql & params]
  (apply db/query app/db sql params))

(defn exec! [sql & params]
  (apply db/execute! app/db sql params))

;; Inspection helpers
(defn pp [x]
  (pp/pprint x))

(defn routes []
  (pp app/all-routes))

;; Seeding test data
(defn seed-users! []
  (doseq [user [{:name "สมชาย" :email "a@test.com"}
                {:name "สมหญิง" :email "b@test.com"}]]
    (exec! "INSERT INTO users (name, email) VALUES (?, ?)"
           (:name user) (:email user))))
```

---

## ขั้นตอนที่ 141: Modular Architecture

```clojure
;; Pattern: แยก system เป็น modules

;; my-app/
;; ├── modules/
;; │   ├── auth/       ← Auth module
;; │   │   ├── model.clj
;; │   │   ├── service.clj
;; │   │   ├── routes.clj
;; │   │   └── module.clj   ← Module entry point
;; │   ├── users/
;; │   │   ├── model.clj
;; │   │   ├── repository.clj
;; │   │   ├── service.clj
;; │   │   ├── routes.clj
;; │   │   └── module.clj
;; │   └── products/
;; │       └── ...
;; └── core.clj         ← Combines all modules

;; modules/users/module.clj
(ns my-app.modules.users.module
  (:require [my-app.modules.users.routes :as routes]
            [my-app.modules.users.service :as service]))

(defn create [deps]
  {:routes (routes/create deps)
   :service (service/create deps)})

;; core.clj
(ns my-app.core
  (:require [my-app.modules.users.module :as users]
            [my-app.modules.auth.module :as auth]))

(defn create-app [config db]
  (let [deps {:db db :config config}
        users-module (users/create deps)
        auth-module (auth/create deps)]
    {:routes (concat (:routes users-module)
                     (:routes auth-module))
     :services {:users (:service users-module)
                :auth (:service auth-module)}}))
```

---

## ขั้นตอนที่ 142: Error Boundaries ใน Namespaces

```clojure
;; Pattern: กำหนด error types สำหรับแต่ละ namespace

;; ใช้ ex-info สำหรับ structured errors
(defn create-user! [db user-data]
  (let [errors (validate-user user-data)]
    (if (seq errors)
      (throw (ex-info "Validation failed"
                      {:type :validation-error
                       :errors errors
                       :data user-data}))
      (try
        (insert-user! db user-data)
        (catch Exception e
          (throw (ex-info "Database error"
                          {:type :db-error
                           :cause e
                           :data user-data}
                          e)))))))

;; Handle errors
(defn handle-user-creation [db data]
  (try
    {:success true :user (create-user! db data)}
    (catch clojure.lang.ExceptionInfo e
      (let [info (ex-data e)]
        (case (:type info)
          :validation-error {:error "Invalid data" :details (:errors info)}
          :db-error {:error "Database error"}
          {:error "Unknown error"})))))
```

---

## ขั้นตอนที่ 143: Namespace Interoperability

```clojure
;; ใช้ resolve สำหรับ dynamic namespace lookup
(defn call-fn [ns-name fn-name & args]
  (let [sym (symbol (str ns-name "/" fn-name))
        f (resolve sym)]
    (if f
      (apply f args)
      (throw (Exception. (str "Function not found: " sym))))))

;; Dynamic require
(defn load-plugin [plugin-name]
  (require (symbol plugin-name))
  (ns-publics (find-ns (symbol plugin-name))))

;; Aliasing ที่ dynamic
(alias 'utils 'my-app.utils)
(utils/format-date (java.util.Date.))
```

---

## ขั้นตอนที่ 144: Testing Namespaces

```clojure
;; test/my_app/users/service_test.clj
(ns my-app.users.service-test
  (:require [clojure.test :refer :all]
            [my-app.users.service :as svc]
            [my-app.test-utils :refer [with-test-db]]))

;; Test fixtures
(use-fixtures :each
  (fn [test-fn]
    ;; Setup before each test
    (println "Setting up test...")
    (test-fn)
    ;; Teardown after each test
    (println "Tearing down test...")))

(deftest test-create-user
  (with-test-db [db]
    (testing "valid user creation"
      (let [result (svc/create-user! db {:name "สมชาย" :email "test@test.com"})]
        (is (true? (:success result)))
        (is (some? (get-in result [:user :id])))))
    
    (testing "duplicate email"
      (svc/create-user! db {:name "A" :email "dup@test.com"})
      (let [result (svc/create-user! db {:name "B" :email "dup@test.com"})]
        (is (some? (:error result)))))))

;; test/my_app/test_utils.clj
(ns my-app.test-utils
  (:require [next.jdbc :as jdbc]))

(defmacro with-test-db [binding & body]
  `(let [~(first binding) (jdbc/get-datasource test-db-spec)]
     (try
       ~@body
       (finally
         ;; cleanup
         ))))
```

---

## ขั้นตอนที่ 145: Namespace Documentation

```clojure
;; เขียน docstring สำหรับ namespace
(ns my-app.users.service
  "User service - business logic สำหรับ user management
  
  ฟังก์ชันหลัก:
  - create-user!   : สร้าง user ใหม่
  - update-user!   : อัพเดท user
  - delete-user!   : ลบ user
  - get-user       : ดึง user ตาม id
  - find-users     : ค้นหา users
  
  Dependencies:
  - my-app.users.repository : DB operations
  - my-app.email.service    : Email notifications"
  (:require [my-app.users.repository :as repo]
            [my-app.email.service :as email]))

;; ดู namespace docstring
(find-ns 'my-app.users.service)

;; Generate documentation ด้วย codox
;; [lein-codox "0.10.8"]
;; lein codox
```

---

## ขั้นตอนที่ 146: Versioning และ Backwards Compatibility

```clojure
;; Deprecate functions gracefully
(defn ^:deprecated old-get-user
  "DEPRECATED: ใช้ get-user-by-id แทน"
  [id]
  (get-user-by-id id))

(defn get-user-by-id [id]
  ;; new implementation
  )

;; Feature flags ด้วย dynamic vars
(def ^:dynamic *enable-new-feature* false)

(defn do-something []
  (if *enable-new-feature*
    (new-implementation)
    (old-implementation)))

;; เปิดใช้ feature ใน test
(binding [*enable-new-feature* true]
  (do-something))
```

---

## ขั้นตอนที่ 147: Performance ใน Namespaces

```clojure
;; AOT (Ahead-of-Time) Compilation
;; ใน project.clj
;; :aot [my-app.core]
;; หรือ :aot :all

;; Direct linking (faster function calls)
;; ใน project.clj
;; :jvm-opts ["-Dclojure.compiler.direct-linking=true"]

;; ลด startup time ด้วย GraalVM Native Image
;; ต้องระวัง reflection - ใส่ type hints
(defn process [^String s ^Long n]
  (str/repeat s n))

;; Check reflection warnings
;; ใน project.clj: :warn-on-reflection true
;; หรือ set! *warn-on-reflection* true ใน REPL
(set! *warn-on-reflection* true)
(defn fast-string [s n]
  (.repeat ^String s ^int n))
```

---

## ขั้นตอนที่ 148: Classpath และ Dependencies

```clojure
;; ดู classpath
(require '[clojure.java.classpath :as cp])
(cp/classpath)

;; ดู loaded namespaces
(loaded-libs)

;; ตรวจสอบว่า namespace exists
(find-ns 'clojure.string)
;; => #object[clojure.lang.Namespace...]

;; ดู all loaded namespaces
(all-ns)

;; Dependency conflicts
;; ใช้ lein deps :tree เพื่อดู dependency tree
;; lein deps :tree

;; แก้ conflict ด้วย :exclusions
;; project.clj:
;; [some-lib "1.0" :exclusions [org.clojure/clojure]]
```

---

## ขั้นตอนที่ 149: Integration Tests

```clojure
;; test/integration/api_test.clj
(ns integration.api-test
  (:require [clojure.test :refer :all]
            [ring.mock.request :as mock]
            [my-app.core :as app]))

;; Test HTTP endpoints
(deftest test-users-api
  (let [app-handler (app/create-handler)]
    
    (testing "GET /api/users"
      (let [response (app-handler (mock/request :get "/api/users"))]
        (is (= 200 (:status response)))
        (is (vector? (json/read-str (:body response))))))
    
    (testing "POST /api/users"
      (let [body (json/write-str {:name "สมชาย" :email "test@test.com"})
            request (-> (mock/request :post "/api/users")
                        (mock/content-type "application/json")
                        (mock/body body))
            response (app-handler request)]
        (is (= 201 (:status response)))))))
```

---

## ขั้นตอนที่ 150: Summary - Project Structure Best Practices

```
✅ DO:
  - ใช้ kebab-case สำหรับ namespace names
  - จัดโครงสร้างตาม domain (users, products, orders)
  - แยก model, repository, service, handler
  - ใช้ :require กับ alias เสมอ
  - ประกาศ private functions ด้วย defn-
  - เขียน docstring สำหรับ public API
  - ทดสอบผ่าน public API เท่านั้น

❌ DON'T:
  - ใช้ :refer :all (ยกเว้น test files)
  - สร้าง circular dependencies
  - ใส่ business logic ใน handler layer
  - ใช้ global mutable state
  - ประกาศ namespace ที่ยาวเกินไป
  - Mix IO กับ pure functions

📁 Recommended Structure:
  src/
    my_app/
      core.clj          ← system bootstrap
      config.clj         ← configuration
      db/
        core.clj          ← db connection
      [domain]/
        model.clj         ← data specs/types
        repository.clj    ← db operations
        service.clj       ← business logic
        handler.clj       ← HTTP handlers
      api/
        core.clj          ← API setup
        routes.clj        ← route definitions
      utils/
        ...               ← shared utilities
```

---

### อ่านต่อใน Part 6: Control Flow และ Error Handling →

---

*Part 5 จาก 100+ | ขั้นตอน 121-150 จาก 1000+*
