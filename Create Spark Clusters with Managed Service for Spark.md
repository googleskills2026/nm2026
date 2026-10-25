# Create Spark Clusters with Managed Service for Spark || GSP103 ||

### Run the following Commands in CloudShell

```bash
#!/bin/bash
# GSP103 - Create Spark Clusters with Managed Service for Spark (Dataproc)
# Run in Cloud Shell: bash gsp103.sh

# ---------- Region / Zone (press Enter for defaults, or type your lab's values) ----------
read -e -p "Region [us-east1]: " R; export REGION=${R:-us-east1}
read -e -p "Zone [us-east1-c]: " Z; export ZONE=${Z:-us-east1-c}
export PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
gcloud config set compute/region $REGION >/dev/null 2>&1
gcloud config set compute/zone $ZONE >/dev/null 2>&1
echo "Project: $PROJECT_ID | Region: $REGION | Zone: $ZONE"

# ---------- Setup: enable API + grant Storage Admin ----------
echo "=== Enabling Dataproc API ==="
gcloud services enable dataproc.googleapis.com

echo "=== Granting Storage Admin to Compute Engine default service account ==="
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --role="roles/storage.admin" --condition=None >/dev/null
echo "Waiting for IAM to propagate..."; sleep 30

# ---------- Wait until the default subnet is ready ----------
echo "=== Waiting for default subnet in $REGION ==="
for i in {1..30}; do
  gcloud compute networks subnets describe default --region=$REGION >/dev/null 2>&1 && { echo "Subnet ready."; break; }
  echo "Not ready yet ($i/30)..."; sleep 10
done
sleep 20

# ---------- Task 1: Create the cluster (retries up to 3 times) ----------
for attempt in 1 2 3; do
  echo "=== Task 1: Creating example-cluster (attempt $attempt/3) ==="
  gcloud dataproc clusters create example-cluster \
    --region=$REGION --zone=$ZONE \
    --master-machine-type=e2-standard-2 \
    --master-boot-disk-type=pd-standard --master-boot-disk-size=30GB \
    --worker-machine-type=e2-standard-2 \
    --worker-boot-disk-type=pd-standard --worker-boot-disk-size=30GB \
    --num-workers=2 && break

  echo "Creation failed. Cleaning up and retrying in 60s..."
  gcloud dataproc clusters delete example-cluster --region=$REGION --quiet 2>/dev/null
  sleep 60
done

# ---------- Task 2 and 3: Submit the SparkPi job ----------
echo "=== Task 2/3: Submitting SparkPi job ==="
gcloud dataproc jobs submit spark \
  --cluster=example-cluster --region=$REGION \
  --class=org.apache.spark.examples.SparkPi \
  --jars=file:///usr/lib/spark/examples/jars/spark-examples.jar \
  -- 1000

# ---------- Task 4: Update workers to 4 ----------
echo "=== Task 4: Updating cluster to 4 workers ==="
gcloud dataproc clusters update example-cluster --region=$REGION --num-workers=4

# ---------- Rerun the job on the updated cluster ----------
echo "=== Rerunning SparkPi on the updated cluster ==="
gcloud dataproc jobs submit spark \
  --cluster=example-cluster --region=$REGION \
  --class=org.apache.spark.examples.SparkPi \
  --jars=file:///usr/lib/spark/examples/jars/spark-examples.jar \
  -- 1000

echo "=== Verify ==="
gcloud dataproc clusters list --region=$REGION
gcloud dataproc jobs list --region=$REGION

echo "Done! Now click 'Check my progress' for each task."
```

### Congratulations 🎉 for completing the Lab !

##### ***You Have Successfully Demonstrated Your Skills And Determination.***

##### ***Well done!***
