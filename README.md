# Video Analytics Solution Using Dataflow & Video AI
This repo contains a reference implementations for video analytics solutions by using Dataflow & Video AI.  The goal is to provide an easy to use end to end solution to process large scale unstructured video data by bringing multiple data streams together to drive insight using Video AI. 

## Table of Contents  
* [Object Detection in Video Clips](#object-detection-in-video-clips).  
	* [Reference Architecture](#reference-architecture-using-video-intelligence-api).      
	* [Build & Run Using Dataflow Flex Template](#build-run).  
	* [Test Using a Drone  Video Clip Dataset from Kaggle ](#test). 
	* [Custom Json Output and Filtering ](#custom-json-output-and-filtering ). 

## Object Detection in Video Clips 
Many customers across various industries  are producing large volumes of unstructured  data and are looking for easy to use streaming solutions to  analyze in near real time. For example, alarm monitoring companies want to augment motion sensor data with the analysis of video clips (and, eventually, live video feeds) to determine if a dispatch of a security team to a customer’s premises is justified and therefore reduce the false positive rate that drives the costs of their operations up. This section of this repo highlights how you can use this pipeline to detect objects in large scale video clips and customize the Json response for downstream systems to consume.  

For testing purpose, we use this [dataset](https://www.kaggle.com/kmader/drone-videos) from Kaggle collected from drone video clips.

### Reference Architecture Using Video Intelligence API
 ![ref_arch](diagram/video_blog_diagram.png)

### How the pipeline works?
1. Solution assumes video clips are uploaded and stored in a GCS bucket  and a metadata notification is sent out to a PubSub topic.

2. Dataflow pipeline process the video files in micro batch  and based on the list of features passed as pipeline argument.  

3. Dataflow pipeline uses the list of entities and confidence score to filter the Video Intelligence API response and output to following sinks:
	 *  In a nested table in BigQuery for further analysis. 
	 * In a PubSub topic by customizing the Json response so that downstream applications can consume in near real time. 

### Build & Run

1. Enable some Google Cloud APIs:

```
gcloud services enable dataflow.googleapis.com containerregistry.googleapis.com videointelligence.googleapis.com
```

2. Set some environment variables (replace values with your project ID and preferred region):

```
export PROJECT=[PROJECT]
export REGION=[REGION]
```

3. Create two buckets, one to store input video files and another one to store Dataflow Flex template config files:

```
export VIDEO_CLIPS_BUCKET=${PROJECT}_videos
export DATAFLOW_TEMPLATE_BUCKET=${PROJECT}_dataflow_template_config
gsutil mb -c standard -l ${REGION} gs://${VIDEO_CLIPS_BUCKET}
gsutil mb -c standard -l ${REGION} gs://${DATAFLOW_TEMPLATE_BUCKET}
```

4. Create required topics and subscriptions as below

```
export GCS_NOTIFICATION_TOPIC="gcs-notification-topic"
export GCS_NOTIFICATION_SUBSCRIPTION="gcs-notification-subscription"
export OBJECT_DETECTION_TOPIC="object-detection-topic"
export OBJECT_DETECTION_SUBSCRIPTION="object-detection-subscription"
gcloud pubsub topics create ${GCS_NOTIFICATION_TOPIC}
gcloud pubsub subscriptions create ${GCS_NOTIFICATION_SUBSCRIPTION} --topic=${GCS_NOTIFICATION_TOPIC}
gcloud pubsub topics create ${OBJECT_DETECTION_TOPIC}
gcloud pubsub subscriptions create ${OBJECT_DETECTION_SUBSCRIPTION} --topic=${OBJECT_DETECTION_TOPIC}
```

5. Create a BigQuery dataset and Table. 

```
export BIGQUERY_DATASET="video_analytics"
bq mk -d --location=US ${BIGQUERY_DATASET}

bq mk -t \
--schema src/main/resources/table_schema.json \
--description "object_tracking_data" \
${PROJECT}:${BIGQUERY_DATASET}.object_tracking_analysis
```

6. Gradle Build

```
gradle spotlessApply -DmainClass=com.google.solutions.df.video.analytics.VideoAnalyticsPipeline 
gradle build -DmainClass=com.google.solutions.df.video.analytics.VideoAnalyticsPipeline 
```  

7.  Trigger using Gradle Run 

This configuration is defaulted to 1 

- 1 second processing time
- filter for window and person entity with confidence greater than 90%

```
gradle run -Pargs="
--runner=DataflowRunner --project=${PROJECT} --region=${REGION}
--autoscalingAlgorithm=THROUGHPUT_BASED --numWorkers=3 --maxNumWorkers=5 --workerMachineType=n1-highmem-4
--inputNotificationSubscription=projects/${PROJECT}/subscriptions/${GCS_NOTIFICATION_SUBSCRIPTION}
--outputTopic=projects/${PROJECT}/topics/${OBJECT_DETECTION_TOPIC}
--features=OBJECT_TRACKING --entities=cat --confidenceThreshold=0.9 --windowInterval=1 --chunkSize=5000000
--tableReference=${PROJECT}:${BIGQUERY_DATASET}.object_tracking_analysis"
```

8. Create a docker image for flex template. 
 
```
gradle jib -Djib.to.image=gcr.io/${PROJECT}/dataflow-video-analytics:latest
```

9. Upload the template JSON config file to GCS.

```
cat << EOF | gsutil cp - gs://${DATAFLOW_TEMPLATE_BUCKET}/dynamic_template_video_analytics.json
{
  "image": "gcr.io/${PROJECT}/dataflow-video-analytics:latest",
  "sdk_info": {"language": "JAVA"}
}
EOF
```

10. Trigger using Dataflow flex template

```
gcloud beta dataflow flex-template run "video-object-tracking" \
--project=${PROJECT} \
--region=${REGION} \
--template-file-gcs-location=gs://${DATAFLOW_TEMPLATE_BUCKET}/dynamic_template_video_analytics.json \
--parameters=<<'EOF'
^~^autoscalingAlgorithm="NONE"~numWorkers=5~maxNumWorkers=5~workerMachineType=n1-highmem-4
  ~inputNotificationSubscription=projects/${PROJECT}/subscriptions/${GCS_NOTIFICATION_SUBSCRIPTION}
  ~outputTopic=projects/${PROJECT}/topics/${OBJECT_DETECTION_TOPIC}
  ~features=OBJECT_TRACKING~entities=window,person~confidenceThreshold=0.9~windowInterval=1~chunkSize=5000000
  ~tableReference=${PROJECT}:${BIGQUERY_DATASET}.object_tracking_analysis
  ~streaming=true
EOF
```

### Test
1.  Validate the pipeline is running from the Dataflow console
 ![ref_arch](diagram/video_dag_df.png)
 
2. Enable GCS metadata notification for the PubSub and copy sample data to your bucket. 

```
gsutil notification create -t ${GCS_NOTIFICATION_TOPIC} -f json gs://${VIDEO_CLIPS_BUCKET}
```

3. Copy test files to the bucket:

```
gsutil -m cp "gs://df-video-analytics-drone-dataset/*" gs://${VIDEO_CLIPS_BUCKET}
```

4. Please validate if pipeline has successfully processed the data by looking the elements count in the write transform. 

 ![t1](diagram/transform_1.png)
 
 ![t2](diagram/transform_2.png)
 
 ![t3](diagram/transform_3.png)
 
 ![t4](diagram/transform_4.png)

### Custom Json Output and Filtering 
Pipeline uses a nested table in BigQuery to store the API response and also publishes a customized json message to a PubSub topic so that downstream applications can consume it in near real time. This reference implementation shows how you can customize the standard Json response received from Video intelligence API by using [Row/Schema](https://github.com/GoogleCloudPlatform/dataflow-video-analytics/blob/master/src/main/java/com/google/solutions/df/video/analytics/common/Util.java) and built in Beam transform like [ToJson and Filter](https://github.com/GoogleCloudPlatform/dataflow-video-analytics/blob/master/src/main/java/com/google/solutions/df/video/analytics/common/ResponseWriteTransform.java) by column name. 

#### BigQuery Schema 

 ![t4](diagram/table_schema.png). 

* You can use the following query to investigate different objects and confidence level found from our kaggle dataset collected from the video clips

```
SELECT  gcsUri, file_data.entity, max(file_data.confidence) as max_confidence 
FROM `video_analytics.object_tracking_analysis` 
WHERE gcsUri like '%_video_dataset%'
GROUP by  gcsUri, file_data.entity
ORDER by max_confidence DESC
```
 ![t4](diagram/top_entity_by_file.png). 

*  In the test pipeline, you can see from this argument  "entities=window,person" and "confidenceThreshold=0.9" , pipeline is filtering the response that may be required  for near real time processing for downstream applications.  You can use the command below to see the publish message from the output subscription. 

```
gcloud pubsub subscriptions pull ${OBJECT_DETECTION_SUBSCRIPTION} --auto-ack --limit 1 --project ${PROJECT}
```


* You should see json output like below:

```{
   "gcsUri":"/drone-video-dataset/gbikes_dinosaur.mp4",
   "file_data":{
      "entity":"person",
      "confidence":0.9411579370498657,
      "startTimeOffset":"28.833333",
      "endTimeOffset":"29.433333"
   },
   "frame_data":{
      "detections":[
         {
            "frame":1,
            "timeOffset":"28.833333",
            "x":0.044736248,
            "y":0.6212879,
            "w":0.101860054,
            "h":0.8296899
         },
         {
            "frame":2,
            "timeOffset":"28.933333",
            "x":0.024250263,
            "y":0.58061814,
            "w":0.08128166,
            "h":0.7889516
         },
         {
            "frame":3,
            "timeOffset":"29.033333",
            "x":0.02043231,
            "y":0.5591222,
            "w":0.077463575,
            "h":0.7674556
         },
         {
            "frame":4,
            "timeOffset":"29.133333",
            "x":0.029356558,
            "y":0.56631297,
            "w":0.08638781,
            "h":0.7746463
         },
         {
            "frame":5,
            "timeOffset":"29.233333000000002",
            "x":0.05688007,
            "y":0.59617907,
            "w":0.095578365,
            "h":0.76954556
         },
         {
            "frame":6,
            "timeOffset":"29.333333",
            "x":0.061868794,
            "y":0.61623406,
            "w":0.09874276,
            "h":0.78598607
         },
         {
            "frame":7,
            "timeOffset":"29.433333",
            "x":0.06440044,
            "y":0.64491415,
            "w":0.10113216,
            "h":0.8143843
         }
      ]
   }
}
```


