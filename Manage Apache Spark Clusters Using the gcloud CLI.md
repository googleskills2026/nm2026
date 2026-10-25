# Manage Apache Spark Clusters Using the gcloud CLI || GSP104 ||

### Run the following Commands in CloudShell

```bash
#!/bin/bash
# GSP104 - Manage Apache Spark Clusters Using the gcloud CLI (Dataproc)
# Run in Cloud Shell: bash gsp104.sh

# ---------- Region (press Enter for default, or type your lab's region) ----------
read -e -p "Region [us-west1]: " R; export REGION=${R:-us-west1}
export PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
gcloud config set dataproc/region $REGION
gcloud config set compute/region $REGION >/dev/null 2>&1
echo "Project: $PROJECT_ID | Region: $REGION"

# ---------- Task 1: Setup (API reset, IAM, Private Google Access) ----------
echo "=== Disabling and re-enabling Dataproc API ==="
gcloud services disable dataproc.googleapis.com --force
gcloud services enable dataproc.googleapis.com
sleep 20

echo "=== Granting Storage Admin and Dataproc Worker roles ==="
for ROLE in roles/storage.admin roles/dataproc.worker; do
  gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member=serviceAccount:$PROJECT_NUMBER-compute@developer.gserviceaccount.com \
    --role=$ROLE --condition=None >/dev/null
done
echo "Waiting for IAM to propagate..."; sleep 30

echo "=== Waiting for default subnet in $REGION ==="
for i in {1..30}; do
  gcloud compute networks subnets describe default --region=$REGION >/dev/null 2>&1 && { echo "Subnet ready."; break; }
  echo "Not ready yet ($i/30)..."; sleep 10
done
sleep 10

echo "=== Enabling Private Google Access ==="
gcloud compute networks subnets update default --region=$REGION --enable-private-ip-google-access

# ---------- Task 1: Create the cluster (retries up to 3 times) ----------
for attempt in 1 2 3; do
  echo "=== Task 1: Creating example-cluster (attempt $attempt/3) ==="
  gcloud dataproc clusters create example-cluster --region=$REGION \
    --worker-boot-disk-size 500 \
    --worker-machine-type=e2-standard-4 \
    --master-machine-type=e2-standard-4 --quiet && break

  echo "Creation failed. Cleaning up and retrying in 60s..."
  gcloud dataproc clusters delete example-cluster --region=$REGION --quiet 2>/dev/null
  sleep 60
done

# ---------- Task 2: Submit the SparkPi job ----------
echo "=== Task 2: Submitting SparkPi job ==="
gcloud dataproc jobs submit spark --cluster example-cluster --region=$REGION \
  --class org.apache.spark.examples.SparkPi \
  --jars file:///usr/lib/spark/examples/jars/spark-examples.jar -- 1000

# ---------- Task 3: Update workers to 4 ----------
echo "=== Task 3: Updating cluster to 4 workers ==="
gcloud dataproc clusters update example-cluster --region=$REGION --num-workers 4

echo "=== Verify ==="
gcloud dataproc clusters list --region=$REGION
gcloud dataproc jobs list --region=$REGION

echo "Done! Now click 'Check my progress' for each task."
```

### Congratulations 🎉 for completing the Lab !

##### ***You Have Successfully Demonstrated Your Skills And Determination.***

##### ***Well done!***
