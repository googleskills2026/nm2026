# Manage Access Control with IAM || GSP064 ||

### Run the following Commands in CloudShell

```bash
cat > gsp064.sh <<'SCRIPT_END'
#!/bin/bash
# GSP064 - Manage Access Control with IAM
# Run in Cloud Shell of the Username 1 tab (Project Owner).
# Optional override if auto-detect fails:  USER2=student-xx@qwiklabs.net bash gsp064.sh

PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
USER1=$(gcloud auth list --filter=status:ACTIVE --format="value(account)" | head -n 1)

# Find Username 2 = the user who currently has the Viewer role
if [ -z "$USER2" ]; then
  USER2=$(gcloud projects get-iam-policy $PROJECT_ID \
    --flatten="bindings[].members" \
    --filter="bindings.role:roles/viewer AND bindings.members:user:" \
    --format="value(bindings.members)" | sed 's/^user://' | grep -v "^$USER1$" | head -n 1)
fi

echo "Project: $PROJECT_ID | Username 1: $USER1 | Username 2: $USER2"
if [ -z "$USER2" ]; then
  echo "!! Could not find Username 2. Rerun with: USER2=<username 2 email> bash gsp064.sh"; exit 1
fi
if [ "$USER1" = "$USER2" ]; then
  echo "!! You are signed in as Username 2. Run this in the Username 1 tab."; exit 1
fi

# ---------- Task 2: Bucket + sample.txt ----------
BUCKET="${PROJECT_ID}-iam-bucket"
echo "=== Task 2: Creating bucket gs://$BUCKET ==="
gcloud storage buckets create gs://$BUCKET --location=US
echo "This is a sample file for the IAM lab." > sample.txt
gcloud storage cp sample.txt gs://$BUCKET/sample.txt
gcloud storage ls gs://$BUCKET

# ---------- Task 3: Remove Project Viewer from Username 2 ----------
echo "=== Task 3: Removing Viewer role from $USER2 ==="
gcloud projects remove-iam-policy-binding $PROJECT_ID \
  --member="user:$USER2" --role="roles/viewer" --all

# ---------- Task 4: Grant Storage Object Viewer ----------
echo "=== Task 4: Granting Storage Object Viewer to $USER2 ==="
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="user:$USER2" --role="roles/storage.objectViewer" --condition=None >/dev/null

echo "Waiting ~80s for IAM to propagate..."; sleep 80

echo "=== Roles now held by $USER2 ==="
gcloud projects get-iam-policy $PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.members:user:$USER2" \
  --format="table(bindings.role)"

echo
echo "Bucket name: $BUCKET"
echo "Optional check, in the Username 2 Cloud Shell run:  gcloud storage ls gs://$BUCKET"
echo "Done! Now click 'Check my progress' for each task."
SCRIPT_END
bash gsp064.sh
```

### Congratulations 🎉 for completing the Lab !

##### ***You Have Successfully Demonstrated Your Skills And Determination.***

##### ***Well done!***
