# Authenticate Users with Identity-Aware Proxy || GSP499 ||

### Run the following Commands in CloudShell

#### Part 1

```bash
cat > gsp499.sh <<'SCRIPT_END'
#!/bin/bash
# GSP499 - Authenticate Users with Identity-Aware Proxy (Part 1)
# Override if your lab differs:  REGION=us-central1 SERVICE=user-auth-lab bash gsp499.sh
REGION=${REGION:-us-central1}
SERVICE=${SERVICE:-user-auth-lab}
PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
STUDENT=$(gcloud auth list --filter=status:ACTIVE --format="value(account)" | head -n 1)
IAP_AGENT="service-${PROJECT_NUMBER}@gcp-sa-iap.iam.gserviceaccount.com"
echo "Project: $PROJECT_ID | Region: $REGION | Student: $STUDENT"

# The lab pre-enables these APIs, so permission errors here are harmless
gcloud services enable run.googleapis.com iap.googleapis.com cloudbuild.googleapis.com artifactregistry.googleapis.com 2>/dev/null || true

# Download the code
cd ~
gcloud storage cp gs://${PROJECT_ID}-bucket/user-authentication-with-iap.zip .
unzip -o -q user-authentication-with-iap.zip
cd user-authentication-with-iap

# Task 1: deploy, then protect with IAP
echo "=== Task 1: Deploying 1-HelloWorld ==="
cd 1-HelloWorld
gcloud run deploy $SERVICE --source . --allow-unauthenticated --region=$REGION --quiet
URL=$(gcloud run services describe $SERVICE --region=$REGION --format='value(status.url)')
echo "Service URL: $URL"

echo "=== Task 1: Turning IAP on and authorizing the student ==="
gcloud beta run services update $SERVICE --region=$REGION --iap --quiet
for i in {1..12}; do
  gcloud run services add-iam-policy-binding $SERVICE --region=$REGION \
    --member="serviceAccount:$IAP_AGENT" --role="roles/run.invoker" >/dev/null 2>&1 && break
  echo "Waiting for the IAP service agent ($i/12)..."; sleep 10
done
gcloud beta iap web add-iam-policy-binding --resource-type=cloud-run --service=$SERVICE \
  --region=$REGION --member="user:$STUDENT" --role="roles/iap.httpsResourceAccessor"
echo "Waiting 60s for the IAP policy to take effect..."; sleep 60

# Task 2: user identity version
echo "=== Task 2: Deploying 2-HelloUser ==="
cd ~/user-authentication-with-iap/2-HelloUser
gcloud run deploy $SERVICE --source . --region=$REGION --quiet

# Task 3: cryptographic verification
echo "=== Task 3: Deploying 3-HelloVerifiedUser ==="
cd ~/user-authentication-with-iap/3-HelloVerifiedUser
AUDIENCE="/projects/${PROJECT_NUMBER}/locations/${REGION}/services/${SERVICE}"
gcloud run deploy $SERVICE --source . --set-env-vars IAP_AUDIENCE="$AUDIENCE" --region=$REGION --quiet

gcloud beta run services update $SERVICE --region=$REGION --iap --quiet
gcloud run services add-iam-policy-binding $SERVICE --region=$REGION \
  --member="serviceAccount:$IAP_AGENT" --role="roles/run.invoker" >/dev/null

echo
echo "Service URL: $URL"
echo "PART 1 DONE. Now click 'Check my progress' for these three ONLY:"
echo "  1) Deploy a Cloud Run service"
echo "  2) Enable and add policy to IAP"
echo "  3) Use Cryptographic Verification"
echo "Then run Part 2 and click the last check."
SCRIPT_END
bash gsp499.sh
```

#### Part 2: final step

Run this after clicking those three checks:

```bash
gcloud beta run services update user-auth-lab --region=us-central1 --no-iap --quiet
gcloud run services add-iam-policy-binding user-auth-lab --region=us-central1 \
  --member="allUsers" --role="roles/run.invoker"
```

Wait 30 to 60 seconds, then click Check my progress for Access User Identity Information.

#### Instructions

1. Paste Part 1 into Cloud Shell and wait about 10 to 15 minutes until it prints “PART 1 DONE”.
2. Click the three checks listed in the output.
3. Paste Part 2, wait a minute, then click the last check.

### Congratulations 🎉 for completing the Lab !

##### ***You Have Successfully Demonstrated Your Skills And Determination.***

##### ***Well done!***
