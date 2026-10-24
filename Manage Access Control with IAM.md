# Manage Access Control with IAM

### Run the following Commands in CloudShell

> Fill in the blank `export` values from your lab instructions before running.

```bash
export PROJECT_ID=
export USER2=
export BUCKET_NAME=${PROJECT_ID}-iam-bucket

gcloud config set project $PROJECT_ID

# ---------- Task 2: Create bucket + upload sample.txt ----------
gcloud storage buckets create gs://$BUCKET_NAME --location=US
echo "Hello IAM lab" > sample.txt
gcloud storage cp sample.txt gs://$BUCKET_NAME/sample.txt

# ---------- Task 3: Remove Project Viewer from Username 2 ----------
gcloud projects remove-iam-policy-binding $PROJECT_ID \
  --member="user:$USER2" \
  --role="roles/viewer"

# ---------- Task 4: Grant Storage Object Viewer to Username 2 ----------
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="user:$USER2" \
  --role="roles/storage.objectViewer"

# ---------- Show results ----------
echo "Bucket name: $BUCKET_NAME"
gcloud storage ls gs://$BUCKET_NAME
gcloud projects get-iam-policy $PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.members:$USER2" \
  --format="table(bindings.role)"
```

> **Note:** After running this, an option prompt will appear. The 1st is Option 1 and the 2nd is Option 2.

### Congratulations 🎉 for completing the Lab !

##### ***You Have Successfully Demonstrated Your Skills And Determination.***

##### ***Well done!***
