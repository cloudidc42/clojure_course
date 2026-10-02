# Part 85: ClojureScript ขั้นสูง
## ขั้นตอนที่ 2521-2550: shadow-cljs, React Hooks, State Management, SSR

---

## บทนำ

ClojureScript ecosystem ขั้นสูง:
- **shadow-cljs** - modern build tool with hot reload
- **Reagent 3.0** - idiomatic React hooks integration
- **re-frame** - unidirectional data flow
- **Helix** - direct React interop without Reagent
- **Server-Side Rendering** - with Node.js

---

## ขั้นตอนที่ 2521: shadow-cljs Configuration

```clojure
;; shadow-cljs.edn
{:source-paths ["src/main" "src/dev"]
 :dependencies [[reagent "1.2.0"]
                [re-frame "1.3.0"]
                [day8.re-frame/http-fx "0.2.4"]
                [cljs-ajax "0.8.4"]]
 
 :builds
 {:app {:target     :browser
        :output-dir "public/js"
        :asset-path "/js"
        :modules    {:main {:init-fn myapp.core/init
                            :entries [myapp.core]}}
        :devtools   {:http-root "public"
                     :http-port 3000
                     :watch-dir "public"}}
  
  :tests {:target    :browser-test
          :test-dir  "public/test"
          :devtools  {:http-port 3001}}
  
  :node {:target  :node-script
         :output-to "dist/server.js"
         :main   myapp.server/main}}}
```

---

## ขั้นตอนที่ 2522: Reagent with React Hooks

```clojure
(ns myapp.components
  (:require [reagent.core :as r]
            [reagent.dom :as rdom]))

;; Function component (React hooks compatible)
(defn use-local-storage [key default-val]
  (let [state (r/atom (or (js/localStorage.getItem key) default-val))]
    {:value   @state
     :set!    (fn [v]
                (reset! state v)
                (js/localStorage.setItem key v))
     :remove! #(do (swap! state (fn [_] default-val))
                   (js/localStorage.removeItem key))}))

;; Component with custom hook
(defn theme-toggle []
  (let [{:keys [value set!]} (use-local-storage "theme" "light")]
    [:div.theme-toggle
     [:button {:on-click #(set! (if (= value "light") "dark" "light"))
               :class    (str "btn btn-" value)}
      (if (= value "light") "🌙 Dark" "☀️ Light")]]))

;; Use-effect equivalent
(defn timer-component []
  (let [time   (r/atom (js/Date.))
        timer  (r/atom nil)]
    (r/create-class
      {:component-did-mount
       (fn [_]
         (reset! timer (js/setInterval #(reset! time (js/Date.)) 1000)))
       
       :component-will-unmount
       (fn [_]
         (js/clearInterval @timer))
       
       :render
       (fn [_]
         [:div.clock
          [:h2 (.toLocaleTimeString @time)]])})))

;; Functional component with hooks via helix
(require '[helix.core :refer [defnc $]]
         '[helix.hooks :as hooks])

(defnc counter-widget [{:keys [initial-count]}]
  (let [[count set-count] (hooks/use-state (or initial-count 0))
        [active set-active] (hooks/use-state false)]
    (hooks/use-effect
      [count]
      (when (> count 10)
        (js/alert "Count is over 10!")))
    
    [:div {:class "counter"}
     [:p "Count: " count]
     [:button {:on-click #(set-count inc)} "Increment"]
     [:button {:on-click #(set-count dec)} "Decrement"]
     [:button {:on-click #(set-count 0)} "Reset"]]))
```

---

## ขั้นตอนที่ 2523: re-frame Application Architecture

```clojure
(ns myapp.events
  (:require [re-frame.core :as rf]
            [ajax.core :as ajax]))

;; App DB schema
(def default-db
  {:user     nil
   :products []
   :cart     {}
   :ui       {:loading? false
              :error    nil
              :modal    nil}})

;; Initialize
(rf/reg-event-db
  ::initialize-db
  (fn [_ _] default-db))

;; Fetch products
(rf/reg-event-fx
  ::fetch-products
  (fn [{:keys [db]} _]
    {:db         (assoc-in db [:ui :loading?] true)
     :http-xhrio {:method          :get
                  :uri             "/api/products"
                  :response-format (ajax/json-response-format {:keywords? true})
                  :on-success      [::fetch-products-success]
                  :on-failure      [::fetch-products-failure]}}))

(rf/reg-event-db
  ::fetch-products-success
  (fn [db [_ products]]
    (-> db
        (assoc :products products)
        (assoc-in [:ui :loading?] false))))

(rf/reg-event-db
  ::fetch-products-failure
  (fn [db [_ error]]
    (-> db
        (assoc-in [:ui :error] (str "Failed: " (:status error)))
        (assoc-in [:ui :loading?] false))))

;; Cart management
(rf/reg-event-db
  ::add-to-cart
  (fn [db [_ product-id qty]]
    (update-in db [:cart product-id] (fnil + 0) qty)))

(rf/reg-event-db
  ::remove-from-cart
  (fn [db [_ product-id]]
    (update db :cart dissoc product-id)))

;; Subscriptions
(rf/reg-sub ::db identity)
(rf/reg-sub ::products (fn [db _] (:products db)))
(rf/reg-sub ::cart     (fn [db _] (:cart db)))
(rf/reg-sub ::loading? (fn [db _] (get-in db [:ui :loading?])))

(rf/reg-sub
  ::cart-total
  :<- [::cart]
  :<- [::products]
  (fn [[cart products] _]
    (let [products-by-id (into {} (map (juxt :id identity) products))]
      (reduce-kv (fn [total prod-id qty]
                    (+ total (* qty (get-in products-by-id [prod-id :price] 0))))
                  0 cart))))
```

---

## ขั้นตอนที่ 2524: Advanced Reagent Components

```clojure
(ns myapp.ui.table)

;; Sortable, filterable table component
(defn use-table-state [initial-data]
  (let [sort-key   (r/atom nil)
        sort-dir   (r/atom :asc)
        filter-txt (r/atom "")
        page       (r/atom 0)
        page-size  20]
    {:sorted-data
     (r/reaction
       (let [data    initial-data
             filtered (if (empty? @filter-txt)
                         data
                         (filter #(clojure.string/includes?
                                    (str (vals %))
                                    @filter-txt)
                                  data))
             sorted  (if @sort-key
                       (sort-by @sort-key filtered)
                       filtered)
             ordered (if (= :desc @sort-dir) (reverse sorted) sorted)
             start   (* @page page-size)]
         (vec (take page-size (drop start ordered)))))
     :sort-key   sort-key
     :sort-dir   sort-dir
     :filter-txt filter-txt
     :page       page
     :page-size  page-size}))

(defn data-table [columns data-source]
  (let [table (use-table-state @data-source)]
    [:div.table-container
     [:input {:type        "text"
              :placeholder "Filter..."
              :value       @(:filter-txt table)
              :on-change   #(reset! (:filter-txt table) (.. % -target -value))}]
     [:table
      [:thead
       [:tr (for [{:keys [key label]} columns]
              ^{:key key}
              [:th {:on-click #(do
                                 (if (= @(:sort-key table) key)
                                   (swap! (:sort-dir table) {:asc :desc :desc :asc})
                                   (do (reset! (:sort-key table) key)
                                       (reset! (:sort-dir table) :asc))))}
               label
               (when (= @(:sort-key table) key)
                 (if (= @(:sort-dir table) :asc) " ↑" " ↓"))])]]
      [:tbody
       (for [row @(:sorted-data table)]
         ^{:key (:id row)}
         [:tr (for [{:keys [key render]} columns]
                ^{:key key}
                [:td (if render (render (get row key) row) (get row key))])])]]]))
```

---

## ขั้นตอนที่ 2525: ClojureScript Node.js Integration

```clojure
(ns myapp.node-server
  (:require ["express" :as express]
            ["fs" :as fs]
            [cljs.core.async :refer [<!]]
            [cljs.core.async :refer-macros [go]]))

;; Express server in ClojureScript
(defn create-server []
  (let [app (express)]
    
    ;; Middleware
    (.use app (.json express))
    (.use app (.urlencoded express #js {:extended true}))
    
    ;; Routes
    (.get app "/health"
      (fn [_req res]
        (.json res #js {:status "ok" :time (js/Date.now)})))
    
    (.get app "/api/data"
      (fn [req res]
        (go
          (let [data (<! (fetch-data))]
            (.json res (clj->js data))))))
    
    (.post app "/api/process"
      (fn [req res]
        (let [body (js->clj (.-body req) :keywordize-keys true)]
          (try
            (let [result (process-data body)]
              (.json res (clj->js result)))
            (catch js/Error e
              (.status res 400)
              (.json res #js {:error (.-message e)}))))))
    
    app))

;; Server-side rendering with React
(defn ssr-render [component props]
  (let [ReactDOM (js/require "react-dom/server")]
    (.renderToString ReactDOM
      (r/as-element [component props]))))

(defn main []
  (let [app    (create-server)
        port   (or (.-PORT js/process.env) 3000)]
    (.listen app port
      (fn []
        (println "Server running on port" port)))))
```

---

*Part 85 จาก 100+ | ขั้นตอน 2521-2550 จาก 1000+*
