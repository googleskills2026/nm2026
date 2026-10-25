# Transcribe Audio Using the Speech-to-Text API || GSP119 ||

### Run the following Commands in CloudShell

```bash
#!/bin/bash
# GSP119 - Transcribe Audio Using the Speech-to-Text API
# Run in Cloud Shell: bash gsp119.sh

export PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
echo "Project: $PROJECT_ID"

# ---------- Enable APIs ----------
gcloud services enable speech.googleapis.com apikeys.googleapis.com
echo "Waiting for APIs to become active..."; sleep 20

# ---------- Task 1: Create an API key (restricted to Speech-to-Text) ----------
echo "=== Task 1: Creating API key ==="
gcloud services api-keys create \
  --display-name="speech-api-key" \
  --api-target=service=speech.googleapis.com

KEY_NAME=$(gcloud services api-keys list \
  --filter="displayName=speech-api-key" --format="value(name)" | head -n 1)
export API_KEY=$(gcloud services api-keys get-key-string "$KEY_NAME" --format="value(keyString)")

if [ -z "$API_KEY" ]; then
  echo "!! Could not create the key via CLI."
  echo "Create it in Console: APIs & services > Credentials > Create credentials > API key,"
  echo "then run: export API_KEY=<your key> and rerun the Task 2/3 part."
  exit 1
fi
echo "API key created."

# ---------- Task 2: Create request.json ----------
echo "=== Task 2: Creating request.json ==="
cat > request.json <<'EOF'
{
  "config": {
      "encoding":"FLAC",
      "languageCode": "en-US"
  },
  "audio": {
      "uri":"gs://cloud-samples-tests/speech/brooklyn.flac"
  }
}
EOF

# ---------- Task 3: Call the API (retries while the key activates) ----------
echo "=== Task 3: Calling Speech-to-Text API ==="
for i in {1..12}; do
  curl -s -X POST -H "Content-Type: application/json" --data-binary @request.json \
    "https://speech.googleapis.com/v1/speech:recognize?key=${API_KEY}" > result.json
  if grep -q '"transcript"' result.json; then
    echo "Got a transcript."; break
  fi
  echo "Key not active yet, retrying ($i/12)..."; sleep 10
done
cat result.json

# ---------- Upload to the lab bucket ----------
echo "=== Uploading result.json to Cloud Storage ==="
BUCKET="gs://${PROJECT_ID}-speech"
if ! gcloud storage buckets describe $BUCKET >/dev/null 2>&1; then
  echo "Bucket not found, creating it..."
  gcloud storage buckets create $BUCKET
fi
gcloud storage cp result.json $BUCKET/

echo "Done! Now click 'Check my progress' for each task."
```

### Congratulations 🎉 for completing the Lab !

##### ***You Have Successfully Demonstrated Your Skills And Determination.***

##### ***Well done!***
