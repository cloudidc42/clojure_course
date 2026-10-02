# Part 30: Advanced ClojureScript
## ขั้นตอนที่ 871-900: Re-frame Advanced, ClojureScript Interop, CLJS Testing, Code Splitting

---

## บทนำ

ClojureScript (CLJS) compiles to JavaScript:
- **Re-frame** - Elm-inspired state management
- **Reagent** - React wrapper ที่เป็น idiomatic
- **Shadow-cljs** - modern build toolchain
- **cljs-http** - HTTP client
- **CLJS Interop** - ใช้ JS libraries

---

## ขั้นตอนที่ 871: Re-frame Architecture

```clojure
;; Re-frame สร้างบน:
;; app-db = single atom ที่เก็บ state ทั้งหมด
;; events = describe changes
;; effects = side effects
;; subscriptions = derived data

;; ===== app_db.cljs =====
(ns myapp.app-db)

;; Initial state
(def default-db
  {:user       nil
   :loading    {}
   :errors     {}
   :products   {:items  []
                 :total  0
                 :page   1
                 :query  ""}
   :cart       {:items {} :checkout-step :idle}
   :ui         {:sidebar-open? false
                 :theme         :light
                 :modal         nil}})
```

---

## ขั้นตอนที่ 872: Events และ Effects

```clojure
(ns myapp.events
  (:require [re-frame.core :as rf]
            [myapp.app-db :as db]))

;; ===== Initialization =====
(rf/reg-event-db
  ::initialize-db
  (fn [_ _]
    db/default-db))

;; ===== Products =====

;; Pure event: update local state only
(rf/reg-event-db
  ::set-product-query
  (fn [db [_ query]]
    (-> db
        (assoc-in [:products :query] query)
        (assoc-in [:products :page] 1))))

;; Side effect event: fetch from API
(rf/reg-event-fx
  ::fetch-products
  (fn [{:keys [db]} [_ params]]
    {:db    (assoc-in db [:loading :products] true)
     :http  {:url     "/api/v1/products"
              :method  :get
              :params  params
              :success ::products-fetched
              :error   ::products-fetch-failed}}))

(rf/reg-event-db
  ::products-fetched
  (fn [db [_ {:keys [items total]}]]
    (-> db
        (assoc-in [:products :items]  items)
        (assoc-in [:products :total]  total)
        (assoc-in [:loading :products] false))))

;; ===== Cart =====

(rf/reg-event-db
  ::add-to-cart
  (fn [db [_ product-id qty]]
    (update-in db [:cart :items product-id]
               (fnil + 0) qty)))

(rf/reg-event-db
  ::remove-from-cart
  (fn [db [_ product-id]]
    (update-in db [:cart :items] dissoc product-id)))

;; Checkout flow
(rf/reg-event-fx
  ::start-checkout
  (fn [{:keys [db]} _]
    (if (nil? (:user db))
      {:dispatch [::redirect "/login?next=/checkout"]}
      {:db       (assoc-in db [:cart :checkout-step] :shipping)
       :navigate "/checkout/shipping"})))
```

---

## ขั้นตอนที่ 873: Subscriptions

```clojure
(ns myapp.subs
  (:require [re-frame.core :as rf]))

;; Level 1: direct db access
(rf/reg-sub
  ::user
  (fn [db _] (:user db)))

(rf/reg-sub
  ::products
  (fn [db _] (:products db)))

;; Level 2: derived from other subs
(rf/reg-sub
  ::cart-items
  (fn [db _] (get-in db [:cart :items])))

(rf/reg-sub
  ::cart-count
  :<- [::cart-items]
  (fn [cart-items _]
    (reduce + (vals cart-items))))

(rf/reg-sub
  ::cart-total
  :<- [::cart-items]
  :<- [::products-map]
  (fn [[cart-items products-map] _]
    (->> cart-items
         (map (fn [[product-id qty]]
                (let [product (get products-map product-id)]
                  (* (:price product) qty))))
         (reduce + 0))))

;; Products as map (for O(1) lookup)
(rf/reg-sub
  ::products-map
  :<- [::products]
  (fn [{:keys [items]} _]
    (into {} (map #(vector (:id %) %) items))))

;; Loading state
(rf/reg-sub
  ::loading?
  (fn [db [_ key]]
    (get-in db [:loading key] false)))

;; Parameterized subscription
(rf/reg-sub
  ::product-by-id
  :<- [::products-map]
  (fn [products-map [_ id]]
    (get products-map id)))
```

---

## ขั้นตอนที่ 874: Reagent Components

```clojure
(ns myapp.views
  (:require [re-frame.core :as rf]
            [reagent.core :as r]))

;; Product Card Component
(defn product-card [{:keys [id name price image-url]}]
  [:div.product-card {:key id}
   [:img {:src image-url :alt name}]
   [:div.product-info
    [:h3.product-name name]
    [:span.product-price (str "฿" (/ price 100.0))]]
   [:button.add-to-cart
    {:on-click #(rf/dispatch [::events/add-to-cart id 1])}
    "เพิ่มในตะกร้า"]])

;; Product List with loading state
(defn product-list []
  (let [products  @(rf/subscribe [::subs/products])
        loading?  @(rf/subscribe [::subs/loading? :products])]
    [:div.product-list
     (if loading?
       [:div.loading "กำลังโหลด..."]
       (if (empty? (:items products))
         [:div.empty "ไม่พบสินค้า"]
         [:div.products-grid
          (for [p (:items products)]
            ^{:key (:id p)} [product-card p])]))]))

;; Cart Sidebar
(defn cart-sidebar []
  (let [items @(rf/subscribe [::subs/cart-items])
        total @(rf/subscribe [::subs/cart-total])]
    [:div.cart
     [:h2 "ตะกร้าสินค้า"]
     (for [[product-id qty] items]
       (let [product @(rf/subscribe [::subs/product-by-id product-id])]
         ^{:key product-id}
         [:div.cart-item
          [:span (:name product)]
          [:span " x " qty]
          [:span " = ฿" (* (:price product) qty 0.01)]
          [:button {:on-click #(rf/dispatch [::events/remove-from-cart product-id])} "✕"]]))
     [:div.cart-total "รวม: ฿" (* total 0.01)]
     [:button.checkout-btn
      {:on-click #(rf/dispatch [::events/start-checkout])}
      "สั่งซื้อ"]]))

;; Form component with local state
(defn search-form []
  (let [query (r/atom "")]
    (fn []
      [:form.search-form
       {:on-submit (fn [e]
                     (.preventDefault e)
                     (rf/dispatch [::events/fetch-products {:query @query}]))}
       [:input
        {:type        "text"
         :value       @query
         :placeholder "ค้นหาสินค้า..."
         :on-change   #(reset! query (.. % -target -value))}]
       [:button {:type "submit"} "ค้นหา"]])))
```

---

## ขั้นตอนที่ 875: JavaScript Interop

```clojure
(ns myapp.interop)

;; Access JS objects
(.-title js/document)  ; document.title
(set! (.-title js/document) "New Title")

;; Call JS functions
(.log js/console "Hello from CLJS")
(.alert js/window "Alert!")

;; Create JS objects
(js-obj "name" "สมชาย" "age" 25)
;; or
#js {:name "สมชาย" :age 25}

;; Convert between CLJS and JS
(clj->js {:name "สมชาย" :items [1 2 3]})
;; => JS object

(js->clj #js {:name "สมชาย" :age 25} :keywordize-keys true)
;; => {:name "สมชาย" :age 25}

;; Use NPM packages (with shadow-cljs)
;; shadow-cljs.edn:
;; {:dependencies [[day8.re-frame/http-fx "0.2.4"]]
;;  :builds {:app {:modules {:main {:entries [myapp.core]}}}}}

;; Import ES module
(ns myapp.charts
  (:require ["chart.js" :as Chart]
            ["date-fns" :as date-fns]))

(defn create-line-chart [canvas-id data]
  (Chart/Chart. (.getElementById js/document canvas-id)
    (clj->js {:type    "line"
               :data    {:labels   (:labels data)
                          :datasets [{:label "Revenue"
                                      :data  (:values data)}]}
               :options {:responsive true}})))
```

---

## ขั้นตอนที่ 876: HTTP Effects

```clojure
(ns myapp.effects
  (:require [re-frame.core :as rf]
            [cljs-http.client :as http]
            [cljs.core.async :refer [<!] :refer-macros [go]]))

;; Register HTTP effect handler
(rf/reg-fx
  :http
  (fn [{:keys [url method params success error headers]}]
    (go
      (let [response (<! (case method
                           :get    (http/get url {:query-params params
                                                   :headers (merge {"Accept" "application/json"} headers)})
                           :post   (http/post url {:json-params params :headers headers})
                           :put    (http/put url {:json-params params :headers headers})
                           :delete (http/delete url {:headers headers})))]
        (if (http/unexceptional-status? (:status response))
          (rf/dispatch [success (:body response)])
          (rf/dispatch [error   {:status  (:status response)
                                  :message (get-in response [:body :message])}]))))))

;; Interceptor: add auth token
(defn auth-interceptor []
  (rf/->interceptor
    :id :auth
    :before (fn [context]
               (let [token @(rf/subscribe [::subs/auth-token])
                     effect (get-in context [:effects :http])]
                 (if (and token effect)
                   (assoc-in context [:effects :http :headers "Authorization"]
                              (str "Bearer " token))
                   context)))))

;; Use interceptors
(rf/reg-event-fx
  ::fetch-profile
  [(auth-interceptor)]
  (fn [_ _]
    {:http {:url     "/api/v1/profile"
             :method  :get
             :success ::profile-loaded
             :error   ::profile-error}}))
```

---

## ขั้นตอนที่ 877: ClojureScript Testing

```clojure
;; shadow-cljs.edn
;; {:builds {:test {:target :node-test
;;                  :output-to "out/test.js"
;;                  :entries [myapp.core-test myapp.events-test]}}}

(ns myapp.events-test
  (:require [cljs.test :refer [deftest testing is] :refer-macros [async]]
            [re-frame.core :as rf]
            [day8.re-frame.test :as rf-test]
            [myapp.events :as events]
            [myapp.subs :as subs]))

;; Test synchronous events
(deftest test-add-to-cart
  (rf-test/run-test-sync
    (rf/dispatch [::events/initialize-db])
    (rf/dispatch [::events/add-to-cart 1 2])
    (rf/dispatch [::events/add-to-cart 2 3])
    (is (= {1 2 2 3} @(rf/subscribe [::subs/cart-items])))
    (is (= 5 @(rf/subscribe [::subs/cart-count])))))

;; Test async events
(deftest test-fetch-products
  (async done
    (rf-test/run-test-async
      (rf/dispatch [::events/initialize-db])
      (rf/dispatch [::events/fetch-products {}])
      (rf-test/wait-for [::events/products-fetched
                          ::events/products-fetch-failed]
        (is (false? @(rf/subscribe [::subs/loading? :products])))
        (done)))))

;; Test subscriptions
(deftest test-cart-total
  (rf-test/run-test-sync
    (rf/dispatch [::events/initialize-db])
    ;; Set up products
    (rf/dispatch [::events/products-fetched
                  {:items [{:id 1 :price 100}
                             {:id 2 :price 200}]}])
    ;; Add to cart
    (rf/dispatch [::events/add-to-cart 1 2])
    (rf/dispatch [::events/add-to-cart 2 1])
    ;; Verify total: (100*2) + (200*1) = 400
    (is (= 400 @(rf/subscribe [::subs/cart-total])))))
```

---

## Project: E-commerce SPA

```clojure
;; main.cljs - Entry point
(ns myapp.core
  (:require [reagent.dom :as dom]
            [re-frame.core :as rf]
            [myapp.router :as router]
            [myapp.events :as events]))

(defn app []
  (let [route @(rf/subscribe [::subs/current-route])]
    [:div#app
     [header]
     [:main.content
      (case (:name route)
        :home      [views/home-page]
        :products  [views/products-page]
        :product   [views/product-detail-page (:params route)]
        :cart      [views/cart-page]
        :checkout  [views/checkout-page]
        :not-found [views/not-found-page])]
     [cart-sidebar]
     [notification-area]]))

;; Initialize
(defn init! []
  (rf/dispatch-sync [::events/initialize-db])
  (router/start!)
  (dom/render [app] (.getElementById js/document "app")))

;; Hot reload
(defn ^:dev/after-load mount-app []
  (rf/clear-subscription-cache!)
  (dom/unmount-component-at-node (.getElementById js/document "app"))
  (init!))
```

---

*Part 30 จาก 100+ | ขั้นตอน 871-900 จาก 1000+*
