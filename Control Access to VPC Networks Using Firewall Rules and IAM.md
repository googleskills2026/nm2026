# Control Access to VPC Networks Using Firewall Rules and IAM || GSP213 ||

### Run the following Commands in CloudShell

```bash
cat > gsp213.sh <<'SCRIPT_END'
#!/bin/bash
# GSP213 - Control Access to VPC Networks Using Firewall Rules and IAM
# Default zone matches this lab. Override if yours differs:
#   ZONE=us-east4-a bash gsp213.sh
ZONE=${ZONE:-us-west1-b}
PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
STUDENT=$(gcloud auth list --filter=status:ACTIVE --format="value(account)" | head -n 1)
gcloud config set compute/zone $ZONE >/dev/null 2>&1
echo "Project: $PROJECT_ID | Zone: $ZONE | Account: $STUDENT"

# Task 1: web servers (blue has the web-server tag, green has none)
gcloud compute instances create blue --zone=$ZONE --machine-type=e2-medium \
  --image-family=debian-12 --image-project=debian-cloud --tags=web-server \
  --metadata=startup-script='#!/bin/bash
apt-get update
apt-get install -y nginx-light
echo "<h1>Welcome to the blue server!</h1>" > /var/www/html/index.nginx-debian.html'

gcloud compute instances create green --zone=$ZONE --machine-type=e2-medium \
  --image-family=debian-12 --image-project=debian-cloud \
  --metadata=startup-script='#!/bin/bash
apt-get update
apt-get install -y nginx-light
echo "<h1>Welcome to the green server!</h1>" > /var/www/html/index.nginx-debian.html'

# Task 2: tagged firewall rule + test-vm
gcloud compute firewall-rules create allow-http-web-server \
  --network=default --direction=INGRESS --priority=1000 --action=ALLOW \
  --rules=tcp:80,icmp --source-ranges=0.0.0.0/0 --target-tags=web-server

gcloud compute instances create test-vm --machine-type=e2-micro --subnet=default --zone=$ZONE

BLUE_EXTERNAL_IP=$(gcloud compute instances describe blue --zone=$ZONE --format='get(networkInterfaces[0].accessConfigs[0].natIP)')
GREEN_EXTERNAL_IP=$(gcloud compute instances describe green --zone=$ZONE --format='get(networkInterfaces[0].accessConfigs[0].natIP)')
BLUE_INTERNAL_IP=$(gcloud compute instances describe blue --zone=$ZONE --format='get(networkInterfaces[0].networkIP)')
GREEN_INTERNAL_IP=$(gcloud compute instances describe green --zone=$ZONE --format='get(networkInterfaces[0].networkIP)')

echo "Waiting for nginx on blue (up to ~3 min)..."
for i in {1..18}; do
  curl -s -m 3 http://$BLUE_EXTERNAL_IP | grep -q "blue" && { echo "Blue is ready."; break; }
  echo "Not ready yet ($i/18)..."; sleep 10
done

echo "--- External to blue (should work) ---"
curl -m 3 http://$BLUE_EXTERNAL_IP
echo "--- External to green (should time out) ---"
curl -m 3 http://$GREEN_EXTERNAL_IP || echo "Timed out as expected."
echo "--- Internal from test-vm (both should work) ---"
gcloud compute ssh test-vm --zone=$ZONE --quiet --command="curl -s http://$BLUE_INTERNAL_IP"
gcloud compute ssh test-vm --zone=$ZONE --quiet --command="curl -s http://$GREEN_INTERNAL_IP"

# Task 3: service account with Network Admin
gcloud iam service-accounts create network-admin --display-name="Network-admin"
SA="network-admin@${PROJECT_ID}.iam.gserviceaccount.com"
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$SA" --role="roles/compute.networkAdmin" --condition=None >/dev/null
echo "Waiting for IAM to propagate..."; sleep 30

gcloud iam service-accounts keys create $HOME/credentials.json --iam-account=$SA
gcloud auth activate-service-account --key-file=$HOME/credentials.json

echo "--- Network Admin: list firewall rules (should work) ---"
gcloud compute firewall-rules list
echo "--- Network Admin: delete firewall rule (should FAIL) ---"
gcloud compute firewall-rules delete allow-http-web-server --quiet || echo "Failed as expected."

# Switch role to Security Admin (as the student account)
gcloud config set account $STUDENT
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$SA" --role="roles/compute.securityAdmin" --condition=None >/dev/null
gcloud projects remove-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$SA" --role="roles/compute.networkAdmin" --all >/dev/null
echo "Waiting for IAM to propagate..."; sleep 45

gcloud auth activate-service-account --key-file=$HOME/credentials.json
for i in {1..10}; do
  gcloud compute firewall-rules delete allow-http-web-server --quiet && { echo "Firewall rule deleted."; break; }
  echo "Role not active yet, retrying ($i/10)..."; sleep 15
done

# Back to the student account and verify
gcloud config set account $STUDENT
echo "--- Blue should now time out ---"
curl -m 3 http://$BLUE_EXTERNAL_IP || echo "Timed out as expected."
echo "Done! Now click 'Check my progress' for each task."
SCRIPT_END
bash gsp213.sh
```

#### Next, paste this after that

```bash
gcloud compute firewall-rules create allow-http-web-server \
  --network=default --direction=INGRESS --priority=1000 --action=ALLOW \
  --rules=tcp:80,icmp --source-ranges=0.0.0.0/0 --target-tags=web-server
```

### Congratulations 🎉 for completing the Lab !

##### ***You Have Successfully Demonstrated Your Skills And Determination.***

##### ***Well done!***
