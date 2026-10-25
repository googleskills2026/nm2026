# Control Access to VPC Networks Using Firewall Rules and IAM || GSP213 ||

### Run the following Commands in CloudShell

```bash
#!/bin/bash
# GSP213 - Control Access to VPC Networks Using Firewall Rules and IAM
# Run in Cloud Shell: bash gsp213.sh

# ---------- Auto-detect values ----------
export PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
export STUDENT_ACCOUNT=$(gcloud config get-value account 2>/dev/null)
export ZONE=$(gcloud config get-value compute/zone 2>/dev/null)

# If zone isn't set in Cloud Shell, ask for it (use the Zone shown in your lab panel)
if [ -z "$ZONE" ]; then
  read -p "Enter your lab ZONE (e.g. us-east4-a): " ZONE
  export ZONE
fi
gcloud config set compute/zone $ZONE >/dev/null 2>&1

echo "Project: $PROJECT_ID | Zone: $ZONE | Account: $STUDENT_ACCOUNT"

# ---------- Task 1: Web servers ----------
echo "=== Task 1: Creating blue server (tag: web-server) ==="
gcloud compute instances create blue \
  --zone=$ZONE --machine-type=e2-medium \
  --image-family=debian-12 --image-project=debian-cloud \
  --tags=web-server \
  --metadata=startup-script='#!/bin/bash
apt-get update
apt-get install -y nginx-light
echo "<h1>Welcome to the blue server!</h1>" > /var/www/html/index.nginx-debian.html'

echo "=== Task 1: Creating green server (no tag) ==="
gcloud compute instances create green \
  --zone=$ZONE --machine-type=e2-medium \
  --image-family=debian-12 --image-project=debian-cloud \
  --metadata=startup-script='#!/bin/bash
apt-get update
apt-get install -y nginx-light
echo "<h1>Welcome to the green server!</h1>" > /var/www/html/index.nginx-debian.html'

# ---------- Task 2: Firewall rule + test-vm ----------
echo "=== Task 2: Creating tagged firewall rule ==="
gcloud compute firewall-rules create allow-http-web-server \
  --network=default --direction=INGRESS --priority=1000 --action=ALLOW \
  --rules=tcp:80,icmp --source-ranges=0.0.0.0/0 --target-tags=web-server

echo "=== Task 2: Creating test-vm ==="
gcloud compute instances create test-vm \
  --machine-type=e2-micro --subnet=default --zone=$ZONE

# ---------- Get IPs ----------
export BLUE_EXTERNAL_IP=$(gcloud compute instances describe blue --zone=$ZONE --format='get(networkInterfaces[0].accessConfigs[0].natIP)')
export GREEN_EXTERNAL_IP=$(gcloud compute instances describe green --zone=$ZONE --format='get(networkInterfaces[0].accessConfigs[0].natIP)')
export BLUE_INTERNAL_IP=$(gcloud compute instances describe blue --zone=$ZONE --format='get(networkInterfaces[0].networkIP)')
export GREEN_INTERNAL_IP=$(gcloud compute instances describe green --zone=$ZONE --format='get(networkInterfaces[0].networkIP)')

# ---------- Wait for nginx to be ready on blue ----------
echo "=== Waiting for nginx to install on blue (up to ~3 min) ==="
for i in {1..18}; do
  if curl -s -m 3 http://$BLUE_EXTERNAL_IP | grep -q "blue"; then
    echo "Blue server is ready."; break
  fi
  echo "Not ready yet, retrying ($i/18)..."; sleep 10
done

echo "=== Test: external HTTP to blue (should work) ==="
curl -m 3 http://$BLUE_EXTERNAL_IP

echo "=== Test: external HTTP to green (should time out) ==="
curl -m 3 http://$GREEN_EXTERNAL_IP || echo "Timed out as expected."

echo "=== Test: internal HTTP from test-vm (both should work) ==="
gcloud compute ssh test-vm --zone=$ZONE --quiet --command="curl -s http://$BLUE_INTERNAL_IP"
gcloud compute ssh test-vm --zone=$ZONE --quiet --command="curl -s http://$GREEN_INTERNAL_IP"

# ---------- Task 3: Service account + roles ----------
echo "=== Task 3: Creating Network-admin service account ==="
gcloud iam service-accounts create network-admin \
  --display-name="Network-admin"
export SA_EMAIL="network-admin@${PROJECT_ID}.iam.gserviceaccount.com"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/compute.networkAdmin"

echo "Waiting for IAM to propagate..."; sleep 30

echo "=== Creating key and activating service account ==="
gcloud iam service-accounts keys create $HOME/credentials.json --iam-account=$SA_EMAIL
gcloud auth activate-service-account --key-file=$HOME/credentials.json

echo "=== Network Admin: list firewall rules (should work) ==="
gcloud compute firewall-rules list

echo "=== Network Admin: delete firewall rule (should FAIL) ==="
gcloud compute firewall-rules delete allow-http-web-server --quiet || echo "Failed as expected."

# ---------- Switch role to Security Admin (using student account) ----------
echo "=== Switching back to student account to change role ==="
gcloud config set account $STUDENT_ACCOUNT

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/compute.securityAdmin"
gcloud projects remove-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/compute.networkAdmin"

echo "Waiting for IAM to propagate..."; sleep 45

# ---------- Delete firewall rule as Security Admin (retry until role takes effect) ----------
gcloud auth activate-service-account --key-file=$HOME/credentials.json
for i in {1..10}; do
  if gcloud compute firewall-rules delete allow-http-web-server --quiet; then
    echo "Firewall rule deleted."; break
  fi
  echo "Role not active yet, retrying ($i/10)..."; sleep 15
done

# ---------- Switch back to student account and verify ----------
gcloud config set account $STUDENT_ACCOUNT
echo "=== Verify: blue should now time out ==="
curl -m 3 http://$BLUE_EXTERNAL_IP || echo "Timed out as expected."

echo "Done! Now click 'Check my progress' for each task."
```

> **Type 2 and press Enter.**

### Congratulations 🎉 for completing the Lab !

##### ***You Have Successfully Demonstrated Your Skills And Determination.***

##### ***Well done!***
