# Part 37: Cloud Native Clojure
## ขั้นตอนที่ 1081-1110: AWS SDK, GCP, Azure, Serverless, Cloud Storage

---

## บทนำ

Clojure บน Cloud:
- **AWS** - S3, DynamoDB, SQS, Lambda, ECS
- **GCP** - Pub/Sub, Cloud Storage, BigQuery
- **Serverless** - AWS Lambda ด้วย Clojure
- **Container** - ECS/EKS Fargate
- **Object Storage** - S3, GCS

---

## ขั้นตอนที่ 1081: AWS SDK

```clojure
;; deps.edn
;; {:deps {com.cognitect.aws/api    {:mvn/version "0.8.686"}
;;         com.cognitect.aws/endpoints {:mvn/version "1.1.12.445"}
;;         com.cognitect.aws/s3        {:mvn/version "848.2.1413.0"}
;;         com.cognitect.aws/sqs       {:mvn/version "848.2.1413.0"}
;;         com.cognitect.aws/dynamodb  {:mvn/version "848.2.1413.0"}}}

(ns myapp.aws
  (:require [cognitect.aws.client.api :as aws]
            [cognitect.aws.credentials :as creds]))

;; S3 client
(def s3
  (aws/client {:api                :s3
                :credentials-provider
                (creds/chain-credentials-provider
                  [(creds/environment-credentials-provider)
                   (creds/profile-credentials-provider)
                   (creds/instance-profile-credentials-provider)])
                :region (or (System/getenv "AWS_REGION") "ap-southeast-1")}))

;; Upload file to S3
(defn upload-file! [bucket key content-type data]
  (aws/invoke s3
    {:op      :PutObject
     :request {:Bucket      bucket
                :Key         key
                :ContentType content-type
                :Body        data}}))

;; Download file
(defn download-file [bucket key]
  (let [result (aws/invoke s3
                  {:op      :GetObject
                   :request {:Bucket bucket :Key key}})]
    (:Body result)))

;; List objects with prefix
(defn list-files [bucket prefix]
  (let [result (aws/invoke s3
                  {:op      :ListObjectsV2
                   :request {:Bucket bucket :Prefix prefix}})]
    (map :Key (:Contents result))))

;; Delete object
(defn delete-file! [bucket key]
  (aws/invoke s3
    {:op      :DeleteObject
     :request {:Bucket bucket :Key key}}))

;; Generate presigned URL (for direct browser upload/download)
(defn presigned-url [bucket key operation expires-seconds]
  (aws/invoke s3
    {:op      (if (= :upload operation) :PutObject :GetObject)
     :request {:Bucket  bucket
                :Key     key
                :Expires expires-seconds}}))
```

---

## ขั้นตอนที่ 1082: DynamoDB

```clojure
(ns myapp.aws.dynamo
  (:require [cognitect.aws.client.api :as aws]))

(def dynamo
  (aws/client {:api    :dynamodb
                :region "ap-southeast-1"}))

;; Create table
(defn create-table! [table-name]
  (aws/invoke dynamo
    {:op :CreateTable
     :request
     {:TableName            table-name
      :AttributeDefinitions  [{:AttributeName "id"   :AttributeType "S"}
                               {:AttributeName "sk"   :AttributeType "S"}]
      :KeySchema             [{:AttributeName "id" :KeyType "HASH"}
                               {:AttributeName "sk" :KeyType "RANGE"}]
      :BillingMode          "PAY_PER_REQUEST"}}))

;; Put item
(defn put-item! [table item]
  (aws/invoke dynamo
    {:op      :PutItem
     :request {:TableName table
                :Item      (into {}
                             (map (fn [[k v]]
                                    [k (cond
                                         (string? v)  {:S v}
                                         (number? v)  {:N (str v)}
                                         (boolean? v) {:BOOL v}
                                         (vector? v)  {:L (mapv #(hash-map :S (str %)) v)})])
                                  item))}}))

;; Get item
(defn get-item [table pk sk]
  (let [result (aws/invoke dynamo
                  {:op      :GetItem
                   :request {:TableName table
                              :Key       {"id" {:S pk}
                                          "sk" {:S sk}}}})]
    ;; Parse DynamoDB format back to Clojure
    (into {}
      (map (fn [[k {:keys [S N BOOL]}]]
             [(keyword k) (or S (when N (Double/parseDouble N)) BOOL)])
           (:Item result)))))

;; Query by partition key
(defn query-items [table pk-value]
  (let [result (aws/invoke dynamo
                  {:op      :Query
                   :request {:TableName              table
                              :KeyConditionExpression "id = :pk"
                              :ExpressionAttributeValues
                              {":pk" {:S pk-value}}}})]
    (:Items result)))
```

---

## ขั้นตอนที่ 1083: SQS Message Queue

```clojure
(ns myapp.aws.sqs
  (:require [cognitect.aws.client.api :as aws]
            [cheshire.core :as json]))

(def sqs
  (aws/client {:api    :sqs
                :region "ap-southeast-1"}))

(def queue-url (System/getenv "SQS_QUEUE_URL"))

;; Send message
(defn send-message! [message-body]
  (aws/invoke sqs
    {:op      :SendMessage
     :request {:QueueUrl    queue-url
                :MessageBody (json/generate-string message-body)}}))

;; Send batch
(defn send-batch! [messages]
  (aws/invoke sqs
    {:op      :SendMessageBatch
     :request {:QueueUrl queue-url
                :Entries  (map-indexed
                            (fn [i msg]
                              {:Id          (str i)
                               :MessageBody (json/generate-string msg)})
                            messages)}}))

;; Receive and process
(defn poll-and-process! [handler]
  (let [result (aws/invoke sqs
                  {:op      :ReceiveMessage
                   :request {:QueueUrl            queue-url
                              :MaxNumberOfMessages 10
                              :WaitTimeSeconds     20  ; long polling
                              :VisibilityTimeout   30}})]
    (doseq [msg (:Messages result)]
      (try
        (let [body (json/parse-string (:Body msg) true)]
          (handler body)
          ;; Delete on success
          (aws/invoke sqs
            {:op      :DeleteMessage
             :request {:QueueUrl      queue-url
                        :ReceiptHandle (:ReceiptHandle msg)}}))
        (catch Exception e
          ;; Message returns to queue after visibility timeout
          (println "Error processing message:" (.getMessage e)))))))

;; Consumer loop
(defn start-consumer! [handler]
  (future
    (loop []
      (poll-and-process! handler)
      (recur))))
```

---

## ขั้นตอนที่ 1084: AWS Lambda ด้วย Clojure

```clojure
;; Clojure as AWS Lambda function
;; deps.edn
;; {:deps {com.amazonaws/aws-lambda-java-core {:mvn/version "1.2.3"}
;;         com.amazonaws/aws-lambda-java-events {:mvn/version "3.11.3"}}}

(ns lambda.handler
  (:gen-class
    :name    lambda.Handler
    :implements [com.amazonaws.services.lambda.runtime.RequestHandler])
  (:import [com.amazonaws.services.lambda.runtime Context]))

;; Lambda handler - must implement RequestHandler
(defn -handleRequest [_ input context]
  (let [logger (.getLogger context)]
    (.log logger (str "Input: " input))
    
    ;; Parse input
    (let [event (into {} input)
          result (process-event event)]
      
      ;; Return response
      {"statusCode" 200
       "headers"    {"Content-Type" "application/json"}
       "body"       (cheshire.core/generate-string result)})))

;; API Gateway handler
(ns lambda.api-handler
  (:gen-class
    :name    lambda.ApiHandler
    :implements [com.amazonaws.services.lambda.runtime.RequestHandler]))

(defn -handleRequest [_ event context]
  (let [method  (get event "httpMethod")
        path    (get event "path")
        body    (when-let [b (get event "body")]
                  (cheshire.core/parse-string b true))
        headers (get event "headers")]
    
    (try
      (let [response (route-request method path body headers)]
        {"statusCode" (:status response)
         "headers"    (merge {"Content-Type" "application/json"}
                              (:headers response))
         "body"       (cheshire.core/generate-string (:body response))})
      (catch Exception e
        {"statusCode" 500
         "body"       (cheshire.core/generate-string
                        {:error (.getMessage e)})}))))

;; Build for Lambda deployment
;; clojure -T:build lambda-uber
```

---

## ขั้นตอนที่ 1085: GCP Pub/Sub

```clojure
;; deps.edn
;; {:deps {com.google.cloud/google-cloud-pubsub {:mvn/version "1.126.6"}}}

(ns myapp.gcp.pubsub
  (:import [com.google.cloud.pubsub.v1 Publisher Subscriber]
           [com.google.pubsub.v1 PubsubMessage TopicName SubscriptionName]
           [com.google.protobuf ByteString]))

(def project-id (System/getenv "GOOGLE_CLOUD_PROJECT"))

;; Publisher
(defn create-publisher [topic-name]
  (-> (Publisher/newBuilder
        (TopicName/of project-id topic-name))
      .build))

(defn publish! [publisher data]
  (let [message (-> (PubsubMessage/newBuilder)
                    (.setData (ByteString/copyFromUtf8
                                (cheshire.core/generate-string data)))
                    .build)]
    @(.publish publisher message)))

;; Subscriber
(defn create-subscriber [subscription-name handler]
  (Subscriber/newBuilder
    (SubscriptionName/of project-id subscription-name)
    (reify com.google.cloud.pubsub.v1.MessageReceiver
      (receiveMessage [_ msg ack-reply-consumer]
        (try
          (let [data (cheshire.core/parse-string
                       (.toStringUtf8 (.getData msg))
                       true)]
            (handler data)
            (.ack ack-reply-consumer))
          (catch Exception e
            (.nack ack-reply-consumer))))))
  .build))

;; Start subscriber
(defn start-subscriber! [subscription handler]
  (let [sub (create-subscriber subscription handler)]
    (.startAsync sub)
    (.awaitRunning sub)
    sub))
```

---

## Project: File Upload Service

```clojure
(ns upload.service
  (:require [myapp.aws :as aws]
            [ring.util.response :as resp]))

(def bucket-name (System/getenv "S3_BUCKET"))

;; Upload handler
(defn upload-handler [request]
  (let [file       (get-in request [:multipart-params "file"])
        filename   (:filename file)
        content-type (:content-type file)
        data       (:bytes file)
        user-id    (get-in request [:auth :sub])
        
        ;; Generate unique key
        key        (str "uploads/" user-id "/" (java.util.UUID/randomUUID)
                        "-" filename)]
    
    ;; Validate file
    (when (> (count data) (* 10 1024 1024))
      (throw (ex-info "File too large" {:status 413})))
    
    (when-not (contains? #{"image/jpeg" "image/png" "application/pdf"} content-type)
      (throw (ex-info "Unsupported file type" {:status 415})))
    
    ;; Upload to S3
    (aws/upload-file! bucket-name key content-type data)
    
    ;; Save metadata to DB
    (db/create-file!
      {:id           (java.util.UUID/randomUUID)
       :user-id      user-id
       :bucket       bucket-name
       :key          key
       :filename     filename
       :content-type content-type
       :size         (count data)
       :uploaded-at  (java.time.Instant/now)})
    
    {:status 201
     :body   {:key key
               :url (str "https://" bucket-name ".s3.amazonaws.com/" key)}}))

;; Get presigned download URL
(defn download-url-handler [request]
  (let [file-id (get-in request [:path-params :id])
        file    (db/find-file file-id)
        url     (aws/presigned-url (:bucket file) (:key file) :download 3600)]
    {:status 200 :body {:url url :expires-in 3600}}))
```

---

*Part 37 จาก 100+ | ขั้นตอน 1081-1110 จาก 1000+*
