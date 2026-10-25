# Cloud Natural Language API: Qwik Start || GSP097 ||

### Run the following Commands in CloudShell

```bash
#!/bin/bash
# GSP097 - Cloud Natural Language API: Qwik Start
# Run in Cloud Shell: bash gsp097.sh

export GOOGLE_CLOUD_PROJECT=$(gcloud config get-value core/project 2>/dev/null)
echo "Project: $GOOGLE_CLOUD_PROJECT"

# ---------- Make sure the API is on ----------
gcloud services enable language.googleapis.com

# ---------- Task 1: Service account + key ----------
echo "=== Task 1: Creating service account ==="
gcloud iam service-accounts create my-natlang-sa \
  --display-name "my natural language service account" \
  || echo "Service account already exists, continuing."

echo "Waiting for the service account to be ready..."
for i in {1..12}; do
  gcloud iam service-accounts describe my-natlang-sa@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com >/dev/null 2>&1 && break
  sleep 5
done

echo "=== Task 1: Creating key ==="
gcloud iam service-accounts keys create ~/key.json \
  --iam-account my-natlang-sa@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com

export GOOGLE_APPLICATION_CREDENTIALS="$HOME/key.json"

# ---------- Task 2: Entity analysis ----------
echo "=== Task 2: Entity analysis request ==="
gcloud ml language analyze-entities \
  --content="Michelangelo Caravaggio, Italian painter, is known for 'The Calling of Saint Matthew'." > result.json

cat result.json

# ---------- Upload result to the lab bucket ----------
echo "=== Uploading result.json to Cloud Storage ==="
BUCKET="gs://${GOOGLE_CLOUD_PROJECT}-nlp"
if ! gcloud storage buckets describe $BUCKET >/dev/null 2>&1; then
  echo "Bucket not found, creating it..."
  gcloud storage buckets create $BUCKET
fi
gcloud storage cp result.json $BUCKET/

echo "Done! Now click 'Check my progress' for both tasks."
```

### Congratulations 🎉 for completing the Lab !

##### ***You Have Successfully Demonstrated Your Skills And Determination.***

##### ***Well done!***
