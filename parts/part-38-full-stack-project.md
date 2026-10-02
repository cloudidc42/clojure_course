# Part 38: Full-Stack Project - E-Commerce Platform
## ขั้นตอนที่ 1111-1140: สร้าง Application ครบ Backend + Frontend

---

## บทนำ

สร้าง E-Commerce Platform ที่สมบูรณ์ด้วย Clojure:
- **Backend**: Clojure + Reitit + next.jdbc + Redis
- **Frontend**: ClojureScript + Re-frame + Reagent
- **Database**: PostgreSQL + Redis
- **Auth**: JWT + OAuth2
- **Deploy**: Docker + docker-compose

---

## ขั้นตอนที่ 1111: Project Structure

```
shop/
├── deps.edn
├── shadow-cljs.edn
├── docker-compose.yml
├── Dockerfile
├── resources/
│   ├── public/
│   │   └── index.html
│   ├── migrations/
│   │   ├── V1__create_users.sql
│   │   ├── V2__create_products.sql
│   │   └── V3__create_orders.sql
│   └── config.edn
├── src/
│   ├── clj/
│   │   └── shop/
│   │       ├── core.clj          ; Entry point
│   │       ├── system.clj        ; Integrant system
│   │       ├── config.clj        ; Configuration
│   │       ├── db.clj            ; Database connection
│   │       ├── auth/
│   │       │   ├── jwt.clj
│   │       │   └── handler.clj
│   │       ├── products/
│   │       │   ├── domain.clj
│   │       │   ├── repository.clj
│   │       │   └── handler.clj
│   │       ├── orders/
│   │       │   ├── domain.clj
│   │       │   ├── repository.clj
│   │       │   └── handler.clj
│   │       └── middleware.clj
│   └── cljs/
│       └── shop/
│           ├── core.cljs         ; CLJS entry
│           ├── app_db.cljs       ; Initial state
│           ├── events.cljs       ; Re-frame events
│           ├── subs.cljs         ; Subscriptions
│           └── views/
│               ├── layout.cljs
│               ├── home.cljs
│               ├── products.cljs
│               ├── cart.cljs
│               └── checkout.cljs
└── test/
    ├── clj/shop/
    │   ├── products_test.clj
    │   └── orders_test.clj
    └── cljs/shop/
        └── events_test.cljs
```

---

## ขั้นตอนที่ 1112: deps.edn

```clojure
{:paths ["src/clj" "resources"]
 
 :deps {org.clojure/clojure                {:mvn/version "1.11.1"}
         org.clojure/clojurescript          {:mvn/version "1.11.60"}
         
         ;; Web
         metosin/reitit                     {:mvn/version "0.7.0"}
         metosin/muuntaja                   {:mvn/version "0.6.8"}
         ring/ring-core                     {:mvn/version "1.11.0"}
         http-kit/http-kit                  {:mvn/version "2.8.0"}
         
         ;; Database
         seancorfield/next.jdbc             {:mvn/version "1.3.909"}
         metosin/honeysql                   {:mvn/version "2.5.1103"}
         com.zaxxer/HikariCP                {:mvn/version "5.1.0"}
         org.postgresql/postgresql          {:mvn/version "42.7.1"}
         org.flywaydb/flyway-core           {:mvn/version "10.4.1"}
         
         ;; Redis
         com.taoensso/carmine               {:mvn/version "3.3.2"}
         
         ;; Auth
         buddy/buddy-sign                   {:mvn/version "3.5.351"}
         buddy/buddy-hashers                {:mvn/version "2.0.167"}
         
         ;; Utils
         cheshire/cheshire                  {:mvn/version "5.13.0"}
         aero/aero                          {:mvn/version "1.1.6"}
         integrant/integrant                {:mvn/version "0.8.1"}
         taoensso/timbre                    {:mvn/version "6.3.1"}
         
         ;; Validation
         metosin/malli                      {:mvn/version "0.14.0"}
         
         ;; CLJS
         reagent/reagent                    {:mvn/version "1.2.0"}
         re-frame/re-frame                  {:mvn/version "1.4.2"}
         day8.re-frame/http-fx              {:mvn/version "0.2.4"}
         cljs-http/cljs-http                {:mvn/version "0.1.48"}}
 
 :aliases
 {:dev  {:extra-paths ["src/dev" "test/clj" "test/cljs"]
          :extra-deps  {ring/ring-mock {:mvn/version "0.4.0"}}}
  :test {:main-opts ["-m" "kaocha.runner"]}
  :build {:deps {io.github.clojure/tools.build {:git/tag "v0.9.6" :git/sha "8e78bcc"}}
           :ns-default build}}}
```

---

## ขั้นตอนที่ 1113: Database Migrations

```sql
-- V1__create_users.sql
CREATE TABLE users (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name       VARCHAR(200) NOT NULL,
    email      VARCHAR(200) NOT NULL UNIQUE,
    password   VARCHAR(200),
    provider   VARCHAR(50)  DEFAULT 'local',
    provider_id VARCHAR(200),
    role       VARCHAR(50)  DEFAULT 'user',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);

-- V2__create_products.sql
CREATE TABLE products (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(500) NOT NULL,
    description TEXT,
    price       DECIMAL(12,2) NOT NULL,
    category    VARCHAR(100),
    image_url   VARCHAR(500),
    stock       INTEGER DEFAULT 0,
    status      VARCHAR(50) DEFAULT 'active',
    created_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_products_category ON products(category);
CREATE INDEX idx_products_status ON products(status);
CREATE INDEX idx_products_name ON products USING gin(to_tsvector('english', name));

-- V3__create_orders.sql
CREATE TABLE orders (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id    UUID REFERENCES users(id),
    status     VARCHAR(50) DEFAULT 'pending',
    total      DECIMAL(12,2),
    items      JSONB,
    address    JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_orders_user ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
```

---

## ขั้นตอนที่ 1114: Backend System

```clojure
(ns shop.system
  (:require [integrant.core :as ig]
            [shop.config :as config]
            [shop.db :as db]
            [shop.routes :as routes]))

(def system-config
  {:shop/config  {}
   :shop/db      {:config (ig/ref :shop/config)}
   :shop/redis   {:config (ig/ref :shop/config)}
   :shop/handler {:db    (ig/ref :shop/db)
                   :redis (ig/ref :shop/redis)
                   :config (ig/ref :shop/config)}
   :shop/server  {:handler (ig/ref :shop/handler)
                   :config  (ig/ref :shop/config)}})

(defmethod ig/init-key :shop/db [_ {:keys [config]}]
  (let [ds (db/create-datasource (get-in config [:database]))]
    (db/run-migrations! ds)
    ds))

(defmethod ig/init-key :shop/handler [_ {:keys [db redis config]}]
  (routes/create-handler {:db db :redis redis :config config}))

(defmethod ig/init-key :shop/server [_ {:keys [handler config]}]
  (let [port (get-in config [:server :port] 8080)]
    (println "Server starting on port" port)
    (org.httpkit.server/run-server handler {:port port})))
```

---

## ขั้นตอนที่ 1115: API Routes

```clojure
(ns shop.routes
  (:require [reitit.ring :as ring]
            [reitit.coercion.malli :as malli-coercion]
            [muuntaja.core :as m]
            [reitit.ring.coercion :as coercion]
            [reitit.ring.middleware.muuntaja :as muuntaja]
            [reitit.ring.middleware.parameters :as parameters]))

(defn create-handler [{:keys [db redis config]}]
  (ring/ring-handler
    (ring/router
      [["/api"
        ["/auth"
         ["/login"    {:post  (auth-handler/login-handler db config)}]
         ["/register" {:post  (auth-handler/register-handler db)}]
         ["/refresh"  {:post  (auth-handler/refresh-handler config)}]]
        
        ["/products"
         [""         {:get  (product-handler/list-products db redis)
                       :post (product-handler/create-product db)}]
         ["/:id"     {:get    (product-handler/get-product db redis)
                       :put    (product-handler/update-product db)
                       :delete (product-handler/delete-product db)}]
         ["/search"  {:get  (product-handler/search-products db)}]]
        
        ["/cart"
         [""         {:get    (cart-handler/get-cart redis)
                       :delete (cart-handler/clear-cart redis)}]
         ["/items"   {:post   (cart-handler/add-item redis)}]
         ["/items/:product-id"
          {:delete (cart-handler/remove-item redis)
           :put    (cart-handler/update-qty redis)}]]
        
        ["/orders"
         [""         {:get  (order-handler/list-orders db)
                       :post (order-handler/create-order db redis)}]
         ["/:id"     {:get (order-handler/get-order db)}]]
        
        ["/users/me" {:get    (user-handler/get-profile db)
                       :put    (user-handler/update-profile db)}]]
       
       ["/health"    {:get health-handler}]]
      
      {:data {:coercion   (malli-coercion/create)
               :muuntaja   m/instance
               :middleware [parameters/parameters-middleware
                             muuntaja/format-middleware
                             coercion/coerce-request-middleware
                             coercion/coerce-response-middleware
                             (middleware/wrap-auth config)
                             middleware/wrap-cors
                             middleware/wrap-errors]}})
    
    (ring/create-default-handler)))
```

---

## ขั้นตอนที่ 1116: Product Handler

```clojure
(ns shop.products.handler
  (:require [shop.products.repository :as repo]
            [malli.core :as m]))

(def ProductSchema
  [:map
   [:name        [:string {:min 1 :max 500}]]
   [:price       [:double {:min 0}]]
   [:description {:optional true} :string]
   [:category    {:optional true} :string]
   [:stock       {:optional true} [:int {:min 0}]]])

(defn list-products [db redis]
  (fn [request]
    (let [{:keys [page limit category q]}
          (:query-params request)
          
          page  (Integer/parseInt (or page "1"))
          limit (Integer/parseInt (or limit "20"))]
      
      {:status 200
       :body   (repo/find-products db
                 {:page     page
                  :limit    limit
                  :category category
                  :query    q})})))

(defn get-product [db redis]
  (fn [request]
    (let [id      (get-in request [:path-params :id])
          cached  (redis-cache/get redis (str "product:" id))]
      
      (if cached
        {:status 200 :body cached}
        (if-let [p (repo/find-by-id db id)]
          (do (redis-cache/set redis (str "product:" id) p 3600)
              {:status 200 :body p})
          {:status 404 :body {:error "Product not found"}})))))

(defn create-product [db]
  (fn [request]
    (let [data (:body-params request)]
      (if (m/validate ProductSchema data)
        (let [product (repo/create! db data)]
          {:status 201 :body product})
        {:status 422
         :body   {:error "Validation failed"
                   :errors (m/explain ProductSchema data)}}))))
```

---

## ขั้นตอนที่ 1117: ClojureScript Frontend

```clojure
;; app_db.cljs
(ns shop.app-db)

(def default-db
  {:user      nil
   :cart      {:items {}}
   :products  {:items [] :total 0 :page 1 :loading? false}
   :orders    {:items [] :loading? false}
   :ui        {:page :home :modal nil :error nil}})

;; events.cljs
(ns shop.events
  (:require [re-frame.core :as rf]))

(rf/reg-event-db :initialize (fn [_ _] shop.app-db/default-db))

;; Products
(rf/reg-event-fx
  :fetch-products
  (fn [{:keys [db]} [_ params]]
    {:db   (assoc-in db [:products :loading?] true)
     :http {:url     "/api/products"
             :params  params
             :success :products-loaded
             :error   :api-error}}))

(rf/reg-event-db
  :products-loaded
  (fn [db [_ {:keys [items total]}]]
    (-> db
        (assoc-in [:products :items]    items)
        (assoc-in [:products :total]    total)
        (assoc-in [:products :loading?] false))))

;; Cart
(rf/reg-event-fx
  :add-to-cart
  (fn [{:keys [db]} [_ product-id qty]]
    {:db         (update-in db [:cart :items product-id] (fnil + 0) qty)
     :local-storage {:action :save :key "cart" :data (:cart db)}}))

;; Checkout
(rf/reg-event-fx
  :checkout
  (fn [{:keys [db]} _]
    {:http {:url     "/api/orders"
             :method  :post
             :body    {:items (get-in db [:cart :items])}
             :success :order-placed
             :error   :checkout-error}}))

;; views/products.cljs
(ns shop.views.products
  (:require [re-frame.core :as rf]))

(defn product-card [{:keys [id name price image-url]}]
  [:div.card.product-card {:key id}
   [:img.product-image {:src image-url :alt name}]
   [:div.card-body
    [:h5.card-title name]
    [:p.price (str "฿" (/ price 100.0))]
    [:button.btn.btn-primary
     {:on-click #(rf/dispatch [:add-to-cart id 1])}
     "เพิ่มในตะกร้า"]]])

(defn products-page []
  (let [products  @(rf/subscribe [:products])
        loading?  (get-in products [:loading?])]
    [:div.products-page
     [:h1 "สินค้าทั้งหมด"]
     (if loading?
       [:div.loading "กำลังโหลด..."]
       [:div.row
        (for [p (:items products)]
          ^{:key (:id p)} [product-card p])])]))
```

---

## ขั้นตอนที่ 1118: docker-compose.yml

```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - APP_ENV=production
      - DATABASE_URL=jdbc:postgresql://postgres:5432/shop
      - DATABASE_USER=shop
      - DATABASE_PASSWORD=${DB_PASSWORD}
      - REDIS_HOST=redis
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped

  postgres:
    image: postgres:16-alpine
    environment:
      - POSTGRES_DB=shop
      - POSTGRES_USER=shop
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U shop"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - app

volumes:
  postgres_data:
  redis_data:
```

---

*Part 38 จาก 100+ | ขั้นตอน 1111-1140 จาก 1000+*
