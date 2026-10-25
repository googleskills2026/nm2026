# Annotate Videos Using the Video Intelligence API || GSP154 ||

### Run the following Commands in CloudShell

```bash
#!/bin/bash
# GSP154 - Annotate Videos Using the Video Intelligence API
# Run in Cloud Shell: bash gsp154.sh

export PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
export STUDENT_ACCOUNT=$(gcloud auth list --filter=status:ACTIVE --format="value(account)" | head -n 1)
echo "Project: $PROJECT_ID | Account: $STUDENT_ACCOUNT"

# Make sure the API is on (the lab says it already is)
gcloud services enable videointelligence.googleapis.com

# ---------- Task 1: Authorization ----------
echo "=== Task 1: Creating service account ==="
gcloud iam service-accounts create quickstart \
  || echo "Service account already exists, continuing."

echo "Waiting for the service account to be ready..."
for i in {1..12}; do
  gcloud iam service-accounts describe quickstart@${PROJECT_ID}.iam.gserviceaccount.com >/dev/null 2>&1 && break
  sleep 5
done

echo "=== Task 1: Creating key ==="
gcloud iam service-accounts keys create key.json \
  --iam-account quickstart@${PROJECT_ID}.iam.gserviceaccount.com

echo "=== Task 1: Activating service account ==="
gcloud auth activate-service-account --key-file key.json

echo "=== Task 1: Access token (first 20 chars) ==="
gcloud auth print-access-token | cut -c1-20

# ---------- Task 2: Annotate video request ----------
echo "=== Task 2: Creating request.json ==="
cat > request.json <<EOF
{
   "inputUri":"gs://spls/gsp154/video/train.mp4",
   "features": [
       "LABEL_DETECTION"
   ]
}
EOF

echo "=== Task 2: Sending annotate request ==="
RESPONSE=$(curl -s -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  'https://videointelligence.googleapis.com/v1/videos:annotate' \
  -d @request.json)
echo "$RESPONSE"

# Operation name looks like: projects/123/locations/asia-east1/operations/456
OP_NAME=$(echo "$RESPONSE" | grep '"name"' | sed 's/.*"name": *"\(.*\)".*/\1/')

if [ -z "$OP_NAME" ]; then
  echo "!! No operation name returned. Check the response above (API may still be enabling). Wait a minute and rerun."
  gcloud config set account $STUDENT_ACCOUNT >/dev/null 2>&1
  exit 1
fi
echo "Operation: $OP_NAME"

# ---------- Poll until the operation is done ----------
echo "=== Waiting for annotation to finish (about a minute) ==="
for i in {1..20}; do
  RESULT=$(curl -s -H 'Content-Type: application/json' \
    -H "Authorization: Bearer $(gcloud auth print-access-token)" \
    "https://videointelligence.googleapis.com/v1/${OP_NAME}")
  if echo "$RESULT" | grep -q '"done": true'; then
    echo "Annotation complete."; break
  fi
  echo "Not done yet ($i/20)..."; sleep 10
done
echo "$RESULT" | head -40

# Switch back to the student account
gcloud config set account $STUDENT_ACCOUNT >/dev/null 2>&1

echo "Done! Now click 'Check my progress' for both tasks."
```

### Congratulations 🎉 for completing the Lab !

##### ***You Have Successfully Demonstrated Your Skills And Determination.***

##### ***Well done!***
