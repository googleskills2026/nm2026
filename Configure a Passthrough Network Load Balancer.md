# Configure a Passthrough Network Load Balancer || GSP007 ||

### Run the following Commands in CloudShell

```bash
cat > gsp007.sh <<'SCRIPT_END'
#!/bin/bash
# GSP007 - Configure a Passthrough Network Load Balancer
# Defaults match the lab (Task 1). Override if your lab differs, e.g.:
#   REGION=us-east1 ZONE=us-east1-b bash gsp007.sh
REGION=${REGION:-us-central1}
ZONE=${ZONE:-us-central1-a}
echo "Region: $REGION | Zone: $ZONE"

# Task 1
gcloud config set compute/region $REGION
gcloud config set compute/zone $ZONE

# Task 2: web servers
for N in 1 2 3; do
  cat > startup$N.sh <<EOF
#!/bin/bash
apt-get update
apt-get install apache2 -y
service apache2 restart
echo "<h3>Web Server: www$N</h3>" | tee /var/www/html/index.html
EOF
  gcloud compute instances create www$N \
    --zone=$ZONE --tags=network-lb-tag --machine-type=e2-small \
    --image-family=debian-12 --image-project=debian-cloud \
    --no-service-account --no-scopes \
    --metadata-from-file=startup-script=startup$N.sh
done

gcloud compute firewall-rules create www-firewall-network-lb \
  --target-tags network-lb-tag --allow tcp:80
gcloud compute instances list

# Task 3
gcloud compute addresses create network-lb-ip-1 --region $REGION
gcloud compute http-health-checks create basic-check

# Task 4
gcloud compute target-pools create www-pool \
  --region $REGION --http-health-check basic-check
gcloud compute target-pools add-instances www-pool \
  --instances www1,www2,www3 --instances-zone=$ZONE
gcloud compute forwarding-rules create www-rule \
  --region $REGION --ports 80 \
  --address network-lb-ip-1 --target-pool www-pool

# Task 5
IPADDRESS=$(gcloud compute forwarding-rules describe www-rule --region $REGION --format="value(IPAddress)")
echo "Load balancer IP: $IPADDRESS"
echo "Waiting for Apache and health checks (up to ~5 min)..."
for i in {1..30}; do
  curl -s -m 2 http://$IPADDRESS | grep -q "Web Server" && { echo "Load balancer is serving."; break; }
  echo "Not ready yet ($i/30)..."; sleep 10
done
echo "Sending 15 requests (should mix www1/www2/www3):"
for i in {1..15}; do curl -s -m 1 http://$IPADDRESS; done
echo "Done! Now click 'Check my progress' for each task."
SCRIPT_END
bash gsp007.sh
```

### Congratulations 🎉 for completing the Lab !

##### ***You Have Successfully Demonstrated Your Skills And Determination.***

##### ***Well done!***
