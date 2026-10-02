# Part 56: ClojureScript ขั้นสูง
## ขั้นตอนที่ 1651-1680: shadow-cljs, Reagent Hooks, Re-frame Interceptors, CLJS Testing

---

## บทนำ

ClojureScript frontend ระดับ advanced:
- **shadow-cljs** - build tool ที่ทรงพลัง
- **Reagent hooks** - React hooks ใน ClojureScript
- **Re-frame interceptors** - middleware สำหรับ events
- **CLJS testing** - test frontend code
- **Code splitting** - lazy loading modules

---

## ขั้นตอนที่ 1651: shadow-cljs Configuration

```clojure
;; shadow-cljs.edn
{:source-paths ["src/cljs" "src/shared"]
 :dependencies [[reagent "1.2.0"]
                [re-frame "1.3.0"]
                [cljs-ajax "0.8.4"]
                [day8.re-frame/http-fx "0.2.4"]
                [com.cognitect/transit-cljs "0.8.280"]]
 
 :dev-http {3000 "public"}
 
 :builds
 {:app
  {:target       :browser
   :output-dir   "public/js"
   :asset-path   "/js"
   :modules      {:main {:init-fn myapp.core/init
                          :entries [myapp.core]}}
   :devtools     {:http-root "public"
                  :http-port 8280
                  :preloads  [reagent.preload
                               re-frame.trace.preload]}}
  
  :test
  {:target :browser-test
   :test-dir "target/test"
   :ns-regexp "-test$"}}}

;; package.json for npm deps
;; "dependencies": {
;;   "react": "18.2.0",
;;   "react-dom": "18.2.0"
;; }
```

---

## ขั้นตอนที่ 1652: Reagent Modern Hooks

```clojure
(ns myapp.components.hooks
  (:require [reagent.core :as r]
            [react :as react]))

;; useEffect hook
(defn use-effect [f deps]
  (react/useEffect
    (fn []
      (let [cleanup (f)]
        (fn [] (when cleanup (cleanup)))))
    (clj->js deps)))

;; Custom hook: useDebounce
(defn use-debounce [value delay-ms]
  (let [debounced (r/atom value)]
    (use-effect
      (fn []
        (let [timer (js/setTimeout #(reset! debounced value) delay-ms)]
          #(js/clearTimeout timer)))
      [value delay-ms])
    @debounced))

;; Custom hook: useLocalStorage
(defn use-local-storage [key default-value]
  (let [stored  (try
                  (-> (js/localStorage.getItem key)
                      js/JSON.parse
                      js->clj)
                  (catch :default _ default-value))
        state   (r/atom (or stored default-value))
        set-fn  (fn [new-val]
                  (reset! state new-val)
                  (js/localStorage.setItem key (js/JSON.stringify (clj->js new-val))))]
    [state set-fn]))

;; Custom hook: useFetch
(defn use-fetch [url]
  (let [state (r/atom {:loading true :data nil :error nil})]
    (use-effect
      (fn []
        (-> (js/fetch url)
            (.then #(.json %))
            (.then #(reset! state {:loading false
                                    :data    (js->clj % :keywordize-keys true)
                                    :error   nil}))
            (.catch #(reset! state {:loading false
                                     :data    nil
                                     :error   (.-message %)}))))
      [url])
    state))

;; Component using hooks
(defn product-search []
  (let [[query set-query!] (use-local-storage "last-query" "")
        debounced   (use-debounce @query 300)
        results     (use-fetch (str "/api/products?q=" debounced))]
    (fn []
      [:div.search
       [:input {:value     @query
                 :on-change #(set-query! (.. % -target -value))
                 :placeholder "Search products..."}]
       (cond
         (:loading @results)  [:div "Loading..."]
         (:error @results)    [:div.error (:error @results)]
         :else                [:ul (map (fn [p] [:li {:key (:id p)} (:name p)])
                                         (:data @results))])])))
```

---

## ขั้นตอนที่ 1653: Re-frame Interceptors

```clojure
(ns myapp.interceptors
  (:require [re-frame.core :as rf]))

;; Interceptors: wrap event handlers
;; :before runs before handler, :after runs after

;; Logging interceptor
(def log-events
  (rf/->interceptor
    :id :log-events
    :before (fn [context]
              (let [event (rf/get-coeffect context :event)]
                (js/console.log "Event:" (clj->js event)))
              context)
    :after  (fn [context]
              (let [effects (rf/get-effect context :db)]
                (when effects
                  (js/console.log "DB changed"))
              context))))

;; Validation interceptor
(defn validate [schema]
  (rf/->interceptor
    :id :validate
    :before (fn [context]
              (let [[event-id & args] (rf/get-coeffect context :event)]
                (if (valid-schema? schema (first args))
                  context
                  (do
                    (rf/dispatch [:validation-error event-id])
                    (assoc-in context [:queue] [])))))))  ; cancel event

;; Undo/redo interceptor
(def undo-stack (rf/->interceptor
  :id :undo-stack
  :before (fn [context]
            (let [db (rf/get-coeffect context :db)]
              (rf/assoc-coeffect context :prev-db db)))
  :after  (fn [context]
            (let [prev-db (rf/get-coeffect context :prev-db)
                  new-db  (rf/get-effect context :db)]
              (when (and prev-db new-db (not= prev-db new-db))
                (rf/dispatch [:push-undo-state prev-db]))
              context))))

;; Optimistic update with rollback
(defn optimistic-update [rollback-key]
  (rf/->interceptor
    :id :optimistic-update
    :before (fn [context]
              (let [db (rf/get-coeffect context :db)]
                (assoc-in context [:coeffects :backup-db] db)))
    :after  (fn [context]
              (let [backup-db (get-in context [:coeffects :backup-db])]
                (update-in context [:effects :http]
                  (fn [req]
                    (assoc req
                      :on-failure [rollback-key backup-db])))))))

;; Event registration with interceptors
(rf/reg-event-db
  :update-product
  [log-events undo-stack (rf/path :products)]
  (fn [products [_ product-id updates]]
    (update products product-id merge updates)))
```

---

## ขั้นตอนที่ 1654: Re-frame Advanced Subscriptions

```clojure
(ns myapp.subs
  (:require [re-frame.core :as rf]
            [reagent.ratom :refer [reaction]]))

;; Layer 2 subscription: extract from db
(rf/reg-sub :products
  (fn [db _] (:products db)))

(rf/reg-sub :cart
  (fn [db _] (:cart db)))

;; Layer 3 subscription: derived data
(rf/reg-sub :cart-total
  :<- [:cart]
  :<- [:products]
  (fn [[cart products] _]
    (reduce (fn [total [product-id qty]]
               (+ total (* qty (get-in products [product-id :price] 0))))
             0
             cart)))

(rf/reg-sub :cart-item-count
  :<- [:cart]
  (fn [cart _]
    (reduce + (vals cart))))

;; Filtered/sorted subscriptions
(rf/reg-sub :filtered-products
  :<- [:products]
  :<- [:product-filter]
  (fn [[products filter-opts] _]
    (cond->> (vals products)
      (:category filter-opts)
      (filter #(= (:category filter-opts) (:category %)))
      
      (:min-price filter-opts)
      (filter #(>= (:price %) (:min-price filter-opts)))
      
      (:search filter-opts)
      (filter #(clojure.string/includes?
                  (clojure.string/lower-case (:name %))
                  (clojure.string/lower-case (:search filter-opts))))
      
      true
      (sort-by (:sort-by filter-opts :name)))))

;; Parameterized subscription
(rf/reg-sub :product-by-id
  (fn [db [_ product-id]]
    (get-in db [:products product-id])))

;; Usage in component:
;; (rf/subscribe [:product-by-id "prod-123"])
```

---

## ขั้นตอนที่ 1655: ClojureScript Testing

```clojure
(ns myapp.components-test
  (:require [cljs.test :refer [deftest is testing async]]
            [reagent.core :as r]
            [re-frame.core :as rf]
            [myapp.events :as events]
            [myapp.subs :as subs]))

;; Initialize re-frame for tests
(defn setup! []
  (rf/dispatch-sync [:initialize-db]))

;; Test subscriptions
(deftest test-cart-total
  (setup!)
  (rf/dispatch-sync [:set-products {:p1 {:id "p1" :price 10.0}
                                     :p2 {:id "p2" :price 25.0}}])
  (rf/dispatch-sync [:add-to-cart "p1" 2])
  (rf/dispatch-sync [:add-to-cart "p2" 1])
  
  (is (= 45.0 @(rf/subscribe [:cart-total]))))

;; Test events
(deftest test-add-remove-cart
  (setup!)
  (rf/dispatch-sync [:add-to-cart "p1" 3])
  (is (= 3 (get @(rf/subscribe [:cart]) "p1")))
  
  (rf/dispatch-sync [:remove-from-cart "p1"])
  (is (nil? (get @(rf/subscribe [:cart]) "p1"))))

;; Async test
(deftest test-fetch-products
  (async done
    (rf/dispatch [:fetch-products])
    (js/setTimeout
      (fn []
        (is (pos? (count @(rf/subscribe [:products]))))
        (done))
      1000)))

;; Component rendering test with reagent
(deftest test-product-card-renders
  (let [product {:id "p1" :name "Widget" :price 9.99}
        component (r/as-element [:div.product-card
                                   [:h2 (:name product)]
                                   [:span (str "$" (:price product))]])]
    (is (some? component))))
```

---

## Project: Dashboard Application

```clojure
(ns myapp.dashboard
  (:require [re-frame.core :as rf]
            [reagent.core :as r]))

;; Dashboard with multiple panels
(defn metric-card [{:keys [title value unit trend]}]
  [:div.metric-card
   [:h3.metric-title title]
   [:div.metric-value
    [:span.number value]
    [:span.unit unit]]
   [:div.metric-trend {:class (if (pos? trend) "up" "down")}
    (if (pos? trend) "▲" "▼")
    (str (js/Math.abs trend) "%")]])

(defn orders-chart []
  (let [data @(rf/subscribe [:orders-by-day])]
    [:div.chart
     ;; Chart implementation with D3 or similar
     (map (fn [{:keys [date count]}]
            [:div.bar {:key   date
                        :style {:height (str (* count 2) "px")
                                 :width  "30px"}}
             [:span.label date]])
          data)]))

(defn dashboard []
  (let [metrics   @(rf/subscribe [:dashboard-metrics])
        loading?  @(rf/subscribe [:loading?])]
    [:div.dashboard
     (if loading?
       [:div.loading "Loading..."]
       [:<>
        [:div.metrics-row
         [metric-card {:title "Revenue" :value "$12,450" :unit "today" :trend 5.2}]
         [metric-card {:title "Orders" :value 148 :unit "today" :trend -2.1}]
         [metric-card {:title "Users" :value 1823 :unit "active" :trend 12.5}]]
        [:div.charts
         [:div.chart-container
          [:h2 "Orders This Week"]
          [orders-chart]]]])]))

;; Init
(defn init []
  (rf/dispatch-sync [:initialize-db])
  (rf/dispatch [:fetch-dashboard-data])
  (r/render [dashboard] (.getElementById js/document "app")))
```

---

*Part 56 จาก 100+ | ขั้นตอน 1651-1680 จาก 1000+*
