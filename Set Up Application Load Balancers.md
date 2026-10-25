# Set Up Application Load Balancers || GSP155 ||

### Run the following Commands in CloudShell

```bash
cat > gsp155.sh <<'SCRIPT_END'
#!/bin/bash
# GSP155 - Set Up Application Load Balancers
# Defaults match the lab (Task 1). Override if your lab differs, e.g.:
#   REGION=us-central1 ZONE=us-central1-a bash gsp155.sh
REGION=${REGION:-us-east4}
ZONE=${ZONE:-us-east4-a}
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
    --metadata-from-file=startup-script=startup$N.sh
done

gcloud compute firewall-rules create www-firewall-network-lb \
  --target-tags network-lb-tag --allow tcp:80
gcloud compute instances list

# Task 3: instance template (startup script written to a file to keep $ signs intact)
cat > lb-startup.sh <<'EOF'
#!/bin/bash
apt-get update
apt-get install apache2 -y
a2ensite default-ssl
a2enmod ssl
vm_hostname="$(curl -H "Metadata-Flavor:Google" \
http://169.254.169.254/computeMetadata/v1/instance/name)"
echo "Page served from: $vm_hostname" | \
tee /var/www/html/index.html
systemctl restart apache2
EOF

gcloud compute instance-templates create lb-backend-template \
  --region=$REGION --network=default --subnet=default \
  --tags=allow-health-check --machine-type=e2-medium \
  --image-family=debian-12 --image-project=debian-cloud \
  --metadata-from-file=startup-script=lb-startup.sh

gcloud compute instance-groups managed create lb-backend-group \
  --template=lb-backend-template --size=2 --zone=$ZONE

gcloud compute firewall-rules create fw-allow-health-check \
  --network=default --action=allow --direction=ingress \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=allow-health-check --rules=tcp:80

gcloud compute addresses create lb-ipv4-1 --ip-version=IPV4 --global
LB_IP=$(gcloud compute addresses describe lb-ipv4-1 --format="get(address)" --global)
echo "Load balancer IP: $LB_IP"

gcloud compute health-checks create http http-basic-check --port 80

gcloud compute backend-services create web-backend-service \
  --protocol=HTTP --port-name=http --health-checks=http-basic-check --global

# Named port so the backend service can reach port 80
gcloud compute instance-groups set-named-ports lb-backend-group \
  --named-ports=http:80 --zone=$ZONE

gcloud compute backend-services add-backend web-backend-service \
  --instance-group=lb-backend-group --instance-group-zone=$ZONE --global

gcloud compute url-maps create web-map-http --default-service web-backend-service
gcloud compute target-http-proxies create http-lb-proxy --url-map web-map-http
gcloud compute forwarding-rules create http-content-rule \
  --address=lb-ipv4-1 --global --target-http-proxy=http-lb-proxy --ports=80

# Task 4: wait for the load balancer
echo "Waiting for the load balancer to serve traffic (up to ~10 min)..."
for i in {1..60}; do
  curl -s -m 3 http://$LB_IP | grep -q "Page served from" && { echo "Load balancer is serving."; break; }
  echo "Not ready yet ($i/60)..."; sleep 10
done
curl -s -m 3 http://$LB_IP
echo "Done! Now click 'Check my progress' for each task."
SCRIPT_END
bash gsp155.sh
```

### Congratulations 🎉 for completing the Lab !

##### ***You Have Successfully Demonstrated Your Skills And Determination.***

##### ***Well done!***
