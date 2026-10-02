# Part 50: Error Handling ขั้นสูง
## ขั้นตอนที่ 1471-1500: Exception Hierarchies, Railway Programming, Error Recovery

---

## บทนำ

Error handling ระดับ production:
- **Exception hierarchy** - สร้าง domain exceptions
- **Railway programming** - either-based error flow
- **Retry patterns** - backoff, circuit breaker
- **Error aggregation** - collect all errors
- **Structured error responses** - RFC 7807 Problem Details

---

## ขั้นตอนที่ 1471: Exception Hierarchy

```clojure
(ns myapp.errors)

;; Define application error hierarchy
(defmulti error-type :error/type)

;; Error records with context
(defrecord AppError [type message code context cause]
  Object
  (toString [_] (str "[" code "] " message)))

(defn app-error
  ([type message] (app-error type message nil nil nil))
  ([type message code] (app-error type message code nil nil))
  ([type message code context] (app-error type message code context nil))
  ([type message code context cause]
   (->AppError type message code context cause)))

;; Domain errors
(defn not-found [entity-type id]
  (app-error :not-found
    (str entity-type " not found: " id)
    "NOT_FOUND"
    {:entity-type entity-type :id id}))

(defn unauthorized [reason]
  (app-error :unauthorized reason "UNAUTHORIZED" {:reason reason}))

(defn forbidden [user-id resource action]
  (app-error :forbidden
    (str "User " user-id " cannot " action " " resource)
    "FORBIDDEN"
    {:user-id user-id :resource resource :action action}))

(defn validation-error [errors]
  (app-error :validation-error
    "Validation failed"
    "VALIDATION_ERROR"
    {:errors errors}))

(defn conflict [message context]
  (app-error :conflict message "CONFLICT" context))

(defn rate-limited [limit window]
  (app-error :rate-limited
    (str "Rate limit: " limit " requests per " window "s")
    "RATE_LIMITED"
    {:limit limit :window window}))

;; HTTP status mapping
(defn error->status [error]
  (case (:type error)
    :not-found        404
    :unauthorized     401
    :forbidden        403
    :validation-error 422
    :conflict         409
    :rate-limited     429
    :not-implemented  501
    500))
```

---

## ขั้นตอนที่ 1472: Railway-Oriented Programming

```clojure
(ns myapp.railway)

;; Result type: {:ok value} or {:error error}
(defn ok [v]    {:ok v})
(defn fail [e]  {:error e})
(defn ok?    [r] (contains? r :ok))
(defn error? [r] (contains? r :error))

;; Bind: chain operations that might fail
(defn >>= [result f]
  (if (ok? result)
    (try
      (f (:ok result))
      (catch AppError e (fail e))
      (catch Exception e (fail (app-error :internal (.getMessage e)))))
    result))

;; Map over ok value
(defn fmap [result f]
  (if (ok? result)
    (ok (f (:ok result)))
    result))

;; Threading macro for railway
(defmacro >>
  "Thread value through railway pipeline"
  [initial & steps]
  (reduce (fn [acc step]
             `(>>= ~acc ~step))
           initial
           steps))

;; Example: order placement pipeline
(defn validate-order [order]
  (if (empty? (:items order))
    (fail (validation-error ["Order must have items"]))
    (ok order)))

(defn check-inventory [order]
  (let [unavailable (filter (fn [item]
                               (< (inventory/get-stock (:product-id item))
                                  (:qty item)))
                             (:items order))]
    (if (seq unavailable)
      (fail (conflict "Items out of stock" {:items unavailable}))
      (ok order))))

(defn charge-payment [order]
  (try
    (let [charge (payment/charge! (:total order) (:user-id order))]
      (ok (assoc order :payment-id (:id charge))))
    (catch Exception e
      (fail (app-error :payment-failed (.getMessage e))))))

(defn create-order-record [order]
  (ok (db/create-order! order)))

;; Compose the pipeline
(defn place-order! [order]
  (>> (ok order)
      validate-order
      check-inventory
      charge-payment
      create-order-record))

;; Handle result
(defn handle-place-order [request]
  (let [result (place-order! (:body-params request))]
    (if (ok? result)
      {:status 201 :body (:ok result)}
      {:status (error->status (:error result))
       :body   {:error   (:message (:error result))
                 :code    (:code (:error result))
                 :details (:context (:error result))}})))
```

---

## ขั้นตอนที่ 1473: Error Accumulation

```clojure
;; Collect ALL errors, not just first one
(defn validate-all [& validators]
  (fn [value]
    (let [errors (keep (fn [v]
                          (let [result (v value)]
                            (when (error? result) (:error result))))
                        validators)]
      (if (seq errors)
        (fail {:errors errors})
        (ok value)))))

;; Field-level validation
(defn validate-field [field-name validators]
  (fn [record]
    (let [value  (get record field-name)
          errors (keep (fn [v]
                          (when-not (v value)
                            (str (name field-name) ": " (meta v))))
                        validators)]
      (if (seq errors)
        {:error errors}
        nil))))

;; Full form validation
(defn validate-registration [data]
  (let [validators
        [(validate-field :email    [#(re-matches #".+@.+\..+" (str %))  ; validator with meta
                                     ^:must-have-at #(clojure.string/includes? (str %) "@")])
         (validate-field :password [#(>= (count (str %)) 8)
                                     #(re-matches #".*[0-9].*" (str %))])
         (validate-field :name     [#(not (clojure.string/blank? (str %)))])]
        
        errors (->> validators
                    (keep #(% data))
                    (mapcat :error))]
    
    (if (seq errors)
      (fail (validation-error errors))
      (ok data))))
```

---

## ขั้นตอนที่ 1474: Retry Pattern

```clojure
(ns myapp.retry)

;; Retry with exponential backoff
(defn with-retry
  [f {:keys [max-attempts base-delay-ms max-delay-ms retryable?]
      :or   {max-attempts  3
              base-delay-ms 100
              max-delay-ms  5000
              retryable?    (constantly true)}}]
  (loop [attempt 1]
    (let [result (try
                   {:ok (f)}
                   (catch Exception e
                     {:error e}))]
      (if (:ok result)
        (:ok result)
        (let [error (:error result)]
          (if (and (< attempt max-attempts)
                   (retryable? error))
            (let [delay (min (* base-delay-ms (Math/pow 2 (dec attempt)))
                              max-delay-ms)
                  jitter (rand-int 100)]
              (Thread/sleep (+ delay jitter))
              (recur (inc attempt)))
            (throw error)))))))

;; Retry for specific exception types
(defn retryable-http-error? [e]
  (when (instance? java.io.IOException e) true))

;; Usage
(defn fetch-with-retry [url]
  (with-retry
    #(clojure.java.io/as-url url)
    {:max-attempts 5
     :base-delay-ms 200
     :retryable? retryable-http-error?}))

;; Bulkhead pattern: isolate failures
(defn create-bulkhead [max-concurrent timeout-ms]
  (let [semaphore (java.util.concurrent.Semaphore. max-concurrent)]
    (fn [f]
      (if (.tryAcquire semaphore timeout-ms java.util.concurrent.TimeUnit/MILLISECONDS)
        (try
          (f)
          (finally
            (.release semaphore)))
        (throw (ex-info "Bulkhead full" {:max max-concurrent}))))))
```

---

## ขั้นตอนที่ 1475: RFC 7807 Problem Details

```clojure
;; Standard error response format
;; https://datatracker.ietf.org/doc/html/rfc7807

(defn problem-details
  "Create RFC 7807 Problem Details response"
  [{:keys [type title status detail instance errors]}]
  {:type     (or type "about:blank")
   :title    (or title (case status
                          400 "Bad Request"
                          401 "Unauthorized"
                          403 "Forbidden"
                          404 "Not Found"
                          409 "Conflict"
                          422 "Unprocessable Entity"
                          429 "Too Many Requests"
                          500 "Internal Server Error"
                          "Unknown Error"))
   :status   status
   :detail   detail
   :instance instance
   :errors   errors})

;; Error handler middleware
(defn wrap-error-handler [handler]
  (fn [request]
    (try
      (handler request)
      (catch AppError e
        (let [status (error->status e)]
          {:status  status
           :headers {"Content-Type" "application/problem+json"}
           :body    (problem-details
                      {:type     (str "https://myapp.com/errors/" (:type e))
                       :title    (:message e)
                       :status   status
                       :detail   (str (:message e))
                       :instance (:uri request)
                       :errors   (:context e)})}))
      (catch Exception e
        {:status  500
         :headers {"Content-Type" "application/problem+json"}
         :body    (problem-details
                    {:type   "https://myapp.com/errors/internal"
                     :title  "Internal Server Error"
                     :status 500
                     :detail "An unexpected error occurred"})}))))
```

---

## Project: Resilient Service

```clojure
(ns myapp.resilient
  (:require [myapp.retry :as retry]
            [myapp.railway :refer [ok fail >>]]))

;; Build a resilient service call chain
(defn call-external-service [service-fn data]
  (try
    (ok (retry/with-retry
          #(service-fn data)
          {:max-attempts 3
           :base-delay-ms 200
           :retryable? #(instance? java.io.IOException %)}))
    (catch Exception e
      (fail (app-error :service-error (.getMessage e))))))

;; Full resilient order workflow
(defn resilient-place-order! [order]
  (>> (ok order)
      validate-order
      check-inventory
      #(call-external-service payment/charge! %)
      #(call-external-service shipping/schedule! %)
      create-order-record
      #(do (notify/send-confirmation! %) (ok %))))

;; Logging + metrics for errors
(defn with-error-monitoring [f operation-name]
  (fn [& args]
    (let [start (System/currentTimeMillis)
          result (apply f args)
          elapsed (- (System/currentTimeMillis) start)]
      (if (error? result)
        (do (log/error "Operation failed" {:op operation-name
                                            :error (:error result)
                                            :ms elapsed})
            (metrics/inc-counter! "errors" {:op operation-name})
            result)
        (do (metrics/record-histogram! "latency" elapsed {:op operation-name})
            result)))))
```

---

*Part 50 จาก 100+ | ขั้นตอน 1471-1500 จาก 1000+*
