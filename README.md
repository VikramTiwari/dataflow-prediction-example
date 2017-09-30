# Cloud Dataflow Batch ML Predictions Example

Disclaimer: This is not an official Google product.

This is an example to demonstrate how to use Cloud Dataflow to run batch
 processing for machine learning predictions. The machine learning model is
 trained with TensorFlow, and the trained model is exported into a Cloud
 Storage bucket in advance. The model is dynamically restored on the worker
 nodes of prediction jobs. This enables you to make predictions against a
 large dataset stored in a Cloud Storage bucket or BigQuery tables, in a
 scalable manner, because Cloud Dataflow automatically distribute the
 prediction tasks to multiple worker nodes.

## Products

- [Cloud Dataflow](https://cloud.google.com/dataflow/)
- [TensorFlow](https://www.tensorflow.org/)

## Prerequisites

- A Google Cloud Platform Account and Project with billing enabled
- Enable the Cloud Dataflow API from [the API Manager](https://console.developers.google.com/apis)
- GCloud CLI tool on your machine

## Setup

- Install Cloud Dataflow SDK and Tensorflow:

``` bash
pip install --upgrade google-cloud-dataflow
pip install --upgrade tensorflow

```

- Setup default credentials using gcloud

``` bash
gcloud auth application-default login
```

- Clone the lab repository in your cloud shell, then `cd` into that dir:

``` bash
git clone https://github.com/GoogleCloudPlatform/dataflow-prediction-example && cd dataflow-prediction-example
```

- Create a storage bucket and upload work files:

``` bash
# get project id
PROJECT=$(gcloud config list project --format "value(core.project)")
# create a bucket using project id
BUCKET=gs://$PROJECT-dataflow
gsutil mkdir $BUCKET
# upload tensorflow model to the bucket
gsutil cp data/export* $BUCKET/model/
# extract images
gzip -kdf data/images.txt.gz
# upload images to the bucket
gsutil cp data/images.txt $BUCKET/input/
```

## Make predictions using Cloud Storage as a data source

- Submit a prediction job.

``` bash
# run batch prediction jobs
python prediction/run.py \
    --runner DataflowRunner \
    --project $PROJECT \
    --staging_location $BUCKET/staging \
    --temp_location $BUCKET/temp \
    --job_name $PROJECT-prediction-cs \
    --setup_file prediction/setup.py \
    --model $BUCKET/model \
    --source cs \
    --input $BUCKET/input/images.txt \
    --output $BUCKET/output/predict
```

The flag `--source cs` indicates that the prediction data source and prediction results are stored in the Cloud Storage bucket.
The number of worker nodes is automatically adjusted by the autoscaling feature. You can specify the number of nodes by using the `--num_workers` parameter, if you want to use a fixed number of nodes.

By clicking on the Cloud Dataflow menu on the Cloud Console, you find the running job and the link navigates to the data flow graph as below:

 <img src="docs/img/dataflow01.png" width="240">

- Confirm the prediction results.

  When the job finishes successfully, the prediction results are stored
  in the Cloud Storage bucket.

``` bash
gsutil ls $BUCKET/output/predict*
# gs://[PROJECT_ID]-dataflow/output/predict-00000-of-00003
# gs://[PROJECT_ID]-dataflow/output/predict-00001-of-00003
# gs://[PROJECT_ID]-dataflow/output/predict-00002-of-00003
```

## Make predictions using BigQuery as a data source

- Create a BigQuery table and upload the prediction data source.

``` bash
bq mk mnist
bq load --source_format=CSV -F":" mnist.images data/images.txt.gz "key:integer,image:string"
```

- Submit a prediction job.

``` bash
python prediction/run.py \
    --runner DataflowRunner \
    --project $PROJECT \
    --staging_location $BUCKET/staging \
    --temp_location $BUCKET/temp \
    --job_name $PROJECT-prediction-bq \
    --setup_file prediction/setup.py \
    --model $BUCKET/model \
    --source bq \
    --input $PROJECT:mnist.images \
    --output $PROJECT:mnist.predict
```

The flag `--source bq` indicates that the prediction data source and prediction results are stored in BigQuery tables.

By clicking on the Cloud Dataflow menu on the Cloud Console, you find the running job and the link navigates to the data flow graph as below:

<img src="docs/img/dataflow02.png" width="240">

- Confirm the prediction results.

When the job finishes successfully, the prediction results are stored in the BigQuery table. By clicking on the BigQuery menu on the Cloud Console, you find the `mnist.predict` table which holds the prediction results.

For example, you can see the prediction results for the first 10 images, in a tabular format, by executing the following query.

``` bash
SELECT * FROM mnist.predict WHERE key < 10 ORDER BY key;
```

## Cleaning up

Clean up is really easy, but also super important: if you don't follow these instructions, you will continue to be billed for the project you created.

To clean up, navigate to the [Google Developers Console Project List][https://console.developers.google.com/project], choose the project you created for this lab, and delete it. That's it.
