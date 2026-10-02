# Part 14: ClojureScript และ Reagent
## ขั้นตอนที่ 391-420: Frontend Development ด้วย ClojureScript + React

---

## บทนำ

ClojureScript คือ Clojure ที่ compile เป็น JavaScript
- ใช้ code เดียวกันกับ Clojure ได้มาก (cljc files)
- **Reagent** = React wrapper ที่ใช้ Clojure data + atoms
- **Re-frame** = Redux-like state management
- **Shadow-cljs** = build tool ที่นิยมที่สุด

---

## ขั้นตอนที่ 391: Setup ClojureScript Project

```clojure
;; package.json
;; {
;;   "name": "my-cljs-app",
;;   "scripts": {
;;     "dev": "shadow-cljs watch app",
;;     "build": "shadow-cljs release app"
;;   },
;;   "devDependencies": {
;;     "shadow-cljs": "^2.28.0"
;;   }
;; }

;; shadow-cljs.edn
;; {:source-paths ["src"]
;;  :dependencies [[reagent "1.2.0"]
;;                 [re-frame "1.3.0"]]
;;  :builds {:app {:target :browser
;;                 :output-dir "public/js"
;;                 :asset-path "/js"
;;                 :modules {:main {:init-fn myapp.core/init}}}}}

;; src/myapp/core.cljs
(ns myapp.core
  (:require [reagent.dom :as rdom]))

(defn app []
  [:div
   [:h1 "Hello ClojureScript!"]
   [:p "React + Clojure = ❤️"]])

(defn init []
  (rdom/render [app] (.getElementById js/document "app")))
```

---

## ขั้นตอนที่ 392: Reagent Components

```clojure
(ns myapp.components
  (:require [reagent.core :as r]))

;; ===== Component Types =====

;; Type 1: Pure function (simplest)
(defn greeting [name]
  [:div.greeting
   [:h1 "Hello, " name "!"]
   [:p "Welcome"]])

;; Type 2: Function returning function (lifecycle)
(defn auto-focus-input []
  (let [ref (r/atom nil)]
    (r/create-class
      {:component-did-mount #(when @ref (.focus @ref))
       :reagent-render
       (fn []
         [:input {:ref #(reset! ref %)}])})))

;; Type 3: Class component (full lifecycle)
(defn lifecycle-demo []
  (r/create-class
    {:display-name "LifecycleDemo"
     
     :component-did-mount
     (fn [this]
       (println "Mounted!" (r/props this)))
     
     :component-will-unmount
     (fn [this]
       (println "Unmounting..."))
     
     :reagent-render
     (fn [props]
       [:div "Component with lifecycle"])}))
```

---

## ขั้นตอนที่ 393: Reagent Atoms - Reactive State

```clojure
;; ratom = reactive atom - triggers re-render when changed!

(def count (r/atom 0))
(def user (r/atom {:name "สมชาย" :age 25}))
(def items (r/atom []))

;; Component ที่ใช้ atom จะ re-render อัตโนมัติเมื่อ atom เปลี่ยน
(defn counter-component []
  [:div
   [:h2 "Count: " @count]
   [:button {:on-click #(swap! count inc)} "+"]
   [:button {:on-click #(swap! count dec)} "-"]
   [:button {:on-click #(reset! count 0)} "Reset"]])

;; Derived state ด้วย reaction
(require '[reagent.ratom :refer [reaction]])

(def doubled-count
  (reaction (* 2 @count)))

(defn derived-state-demo []
  [:div
   [:p "Count: " @count]
   [:p "Doubled: " @doubled-count]])  ; อัพเดทอัตโนมัติ!

;; cursor - focus ที่ส่วนหนึ่งของ atom
(def user-name (r/cursor user [:name]))

(defn edit-name []
  [:input {:value @user-name
            :on-change #(reset! user-name (.. % -target -value))}])
```

---

## ขั้นตอนที่ 394: Hiccup Syntax

```clojure
;; Hiccup = HTML as Clojure data

;; [:tag attrs children...]
[:div {:class "container"} "Hello"]
;; <div class="container">Hello</div>

;; Shorthand class/id
[:div.container "Hello"]           ; class="container"
[:div#main "Hello"]                 ; id="main"
[:div.foo.bar#baz "Hello"]         ; multiple

;; Attributes
[:input {:type "text"
          :value @state
          :on-change #(reset! state (.. % -target -value))
          :placeholder "Enter text..."}]

;; Event handlers
[:button {:on-click #(do-something)
           :on-mouse-over #(highlight!)
           :on-key-down #(when (= 13 (.-keyCode %)) (submit!))}
 "Click me"]

;; Conditional rendering
[:div
 (when logged-in? [:p "Welcome back!"])
 (if (seq items)
   [:ul (for [item items] [:li {:key (:id item)} (:name item)])]
   [:p "No items"])]

;; Map over collection
[:ul
 (for [{:keys [id name]} @users]
   ^{:key id}  ; React key
   [:li name])]
```

---

## ขั้นตอนที่ 395: Re-frame - State Management

```clojure
;; deps: re-frame "1.3.0"

(ns myapp.events
  (:require [re-frame.core :as rf]))

;; ===== Events (actions) =====

(rf/reg-event-db
  :initialize
  (fn [_ _]
    {:users []
     :loading false
     :error nil}))

(rf/reg-event-db
  :set-loading
  (fn [db [_ loading?]]
    (assoc db :loading loading?)))

(rf/reg-event-fx
  :load-users
  (fn [{:keys [db]} _]
    {:db (assoc db :loading true)
     :http-xhrio {:method :get
                   :uri "/api/users"
                   :on-success [:load-users-success]
                   :on-failure [:load-users-error]}}))

(rf/reg-event-db
  :load-users-success
  (fn [db [_ users]]
    (assoc db :users users :loading false)))

(rf/reg-event-db
  :load-users-error
  (fn [db [_ error]]
    (assoc db :error error :loading false)))
```

---

## ขั้นตอนที่ 396: Re-frame Subscriptions

```clojure
(ns myapp.subs
  (:require [re-frame.core :as rf]))

;; ===== Subscriptions (derived state) =====

(rf/reg-sub
  :users
  (fn [db _] (:users db)))

(rf/reg-sub
  :loading?
  (fn [db _] (:loading db)))

(rf/reg-sub
  :error
  (fn [db _] (:error db)))

;; Derived subscription (computed from other subs)
(rf/reg-sub
  :filtered-users
  :<- [:users]           ; depends on :users
  (fn [users [_ query]]
    (if (seq query)
      (filter #(clojure.string/includes? (:name %) query) users)
      users)))

(rf/reg-sub
  :user-count
  :<- [:users]
  (fn [users _] (count users)))

;; ===== Use in components =====
(ns myapp.views
  (:require [re-frame.core :as rf]))

(defn user-list []
  (let [users   @(rf/subscribe [:users])
        loading @(rf/subscribe [:loading?])
        error   @(rf/subscribe [:error])]
    [:div
     (cond
       loading [:p "Loading..."]
       error   [:p.error "Error: " error]
       :else   [:ul (for [u users]
                     ^{:key (:id u)} [:li (:name u)])])]))
```

---

## ขั้นตอนที่ 397: Interop กับ JavaScript

```clojure
;; ===== ClojureScript <-> JavaScript =====

;; JS ออบเจ็กต์
(def obj (js-obj "name" "สมชาย" "age" 25))
(.-name obj)          ; => "สมชาย"
(set! (.-name obj) "สมหญิง")

;; JS Array
(def arr (js/Array. 1 2 3))
(.push arr 4)
(aget arr 0)          ; => 1

;; Convert ClojureScript ↔ JavaScript
(clj->js {:name "สมชาย" :tags ["a" "b"]})
;; => #js {:name "สมชาย", :tags #js ["a" "b"]}

(js->clj #js {:name "สมชาย" :age 25} :keywordize-keys true)
;; => {:name "สมชาย", :age 25}

;; Call JS functions
(.log js/console "Hello from ClojureScript!")
(.getElementById js/document "app")
(js/fetch "/api/users")

;; localStorage
(.setItem js/localStorage "key" "value")
(.getItem js/localStorage "key")

;; setTimeout/setInterval
(js/setTimeout #(println "Hello!") 1000)
(def timer (js/setInterval #(println "Tick!") 500))
(js/clearInterval timer)
```

---

## ขั้นตอนที่ 398: HTTP กับ cljs-ajax

```clojure
(ns myapp.api
  (:require [ajax.core :refer [GET POST PUT DELETE]]
            [re-frame.core :as rf]))

;; Basic GET
(GET "/api/users"
  {:handler       (fn [response] (println "Users:" response))
   :error-handler (fn [error]   (println "Error:" error))
   :response-format :json
   :keywords? true})

;; POST with JSON body
(POST "/api/users"
  {:params {:name "สมชาย" :email "test@test.com"}
   :format :json
   :response-format :json
   :keywords? true
   :handler #(println "Created:" %)
   :error-handler #(println "Error:" %)})

;; With authentication
(defn auth-header []
  {"Authorization" (str "Bearer " (get-token))})

(GET "/api/profile"
  {:headers (auth-header)
   :handler profile-handler
   :response-format :json
   :keywords? true})

;; Re-frame effect handler for HTTP
(rf/reg-fx
  :api-call
  (fn [{:keys [method url body on-success on-failure]}]
    (let [handler (fn [response] (rf/dispatch (conj on-success response)))
          error   (fn [error]    (rf/dispatch (conj on-failure error)))]
      (case method
        :get    (GET url {:handler handler :error-handler error :response-format :json :keywords? true})
        :post   (POST url {:params body :format :json :response-format :json :keywords? true :handler handler :error-handler error})
        :put    (PUT url {:params body :format :json :response-format :json :keywords? true :handler handler :error-handler error})
        :delete (DELETE url {:handler handler :error-handler error})))))
```

---

## ขั้นตอนที่ 399: Full Re-frame App

```clojure
;; ===== Complete Mini App =====

;; events.cljs
(ns myapp.events
  (:require [re-frame.core :as rf]))

(rf/reg-event-db :init
  (fn [_ _] {:count 0 :todos [] :input ""}))

(rf/reg-event-db :increment-count
  (fn [db _] (update db :count inc)))

(rf/reg-event-db :set-input
  (fn [db [_ val]] (assoc db :input val)))

(rf/reg-event-db :add-todo
  (fn [db _]
    (let [text (:input db)]
      (if (seq text)
        (-> db
            (update :todos conj {:id (rand-int 9999) :text text :done false})
            (assoc :input ""))
        db))))

(rf/reg-event-db :toggle-todo
  (fn [db [_ id]]
    (update db :todos
            (fn [todos]
              (map #(if (= id (:id %))
                      (update % :done not)
                      %) todos)))))

;; subs.cljs
(ns myapp.subs
  (:require [re-frame.core :as rf]))

(rf/reg-sub :count (fn [db _] (:count db)))
(rf/reg-sub :todos (fn [db _] (:todos db)))
(rf/reg-sub :input (fn [db _] (:input db)))

(rf/reg-sub :pending-todos
  :<- [:todos]
  (fn [todos _] (count (remove :done todos))))

;; views.cljs
(ns myapp.views
  (:require [re-frame.core :as rf]))

(defn todo-item [{:keys [id text done]}]
  [:li {:class (when done "done")}
   [:input {:type "checkbox"
             :checked done
             :on-change #(rf/dispatch [:toggle-todo id])}]
   [:span text]])

(defn main-panel []
  (let [count   @(rf/subscribe [:count])
        todos   @(rf/subscribe [:todos])
        input   @(rf/subscribe [:input])
        pending @(rf/subscribe [:pending-todos])]
    [:div.app
     [:h1 "My App"]
     
     ;; Counter
     [:section
      [:h2 "Counter: " count]
      [:button {:on-click #(rf/dispatch [:increment-count])} "+"]]
     
     ;; Todos
     [:section
      [:h2 "Todos (" pending " pending)"]
      [:input {:value input
                :on-change #(rf/dispatch [:set-input (.. % -target -value)])
                :on-key-press #(when (= 13 (.-charCode %))
                                 (rf/dispatch [:add-todo]))}]
      [:button {:on-click #(rf/dispatch [:add-todo])} "Add"]
      [:ul (for [todo todos]
             ^{:key (:id todo)} [todo-item todo])]]]))

;; core.cljs
(ns myapp.core
  (:require [reagent.dom :as rdom]
            [re-frame.core :as rf]
            [myapp.views :refer [main-panel]]))

(defn init []
  (rf/dispatch-sync [:init])
  (rdom/render [main-panel] (.getElementById js/document "app")))
```

---

## ขั้นตอนที่ 400: Code Sharing (cljc files)

```clojure
;; .cljc = ทำงานได้ทั้ง Clojure และ ClojureScript!

;; shared/validation.cljc
(ns myapp.shared.validation)

(defn valid-email? [email]
  (boolean (re-matches #".+@.+\..+" email)))

(defn valid-name? [name]
  (and (string? name) (>= (count name) 1) (<= (count name) 100)))

(defn validate-user [{:keys [name email]}]
  (let [errors (cond-> {}
                 (not (valid-name? name))
                 (assoc :name "Name must be 1-100 characters")
                 
                 (not (valid-email? email))
                 (assoc :email "Invalid email format"))]
    {:valid (empty? errors) :errors errors}))

;; Reader conditionals: platform-specific code
(defn current-time []
  #?(:clj  (System/currentTimeMillis)  ; Clojure
     :cljs (.now js/Date)))             ; ClojureScript

(defn parse-int [s]
  #?(:clj  (Integer/parseInt s)
     :cljs (js/parseInt s)))
```

---

*Part 14 จาก 100+ | ขั้นตอน 391-420 จาก 1000+*
