# Configure an Application Load Balancer with Google Cloud Armor || GSP215 ||

### Run the following Commands in CloudShell

```bash
#!/bin/bash
# GSP215 - Application Load Balancer with Google Cloud Armor
# Run in Cloud Shell: bash gsp215.sh

# ---------- Regions (press Enter to accept the default, or type your lab's value) ----------
read -e -p "Region 1 [us-central1]: " R1; export REGION_1=${R1:-us-central1}
read -e -p "Region 2 [us-east4]: " R2; export REGION_2=${R2:-us-east4}
read -e -p "Siege VM zone [us-west1-b]: " SZ; export SIEGE_ZONE=${SZ:-us-west1-b}
export PROJECT_ID=$(gcloud config get-value project 2>/dev/null)

# Names follow the lab pattern: <region>-template and <region>-mig
export T1="${REGION_1}-template"; export T2="${REGION_2}-template"
export MIG1="${REGION_1}-mig";    export MIG2="${REGION_2}-mig"
echo "Project: $PROJECT_ID | $REGION_1, $REGION_2 | Siege zone: $SIEGE_ZONE"

# ---------- Task 1: Firewall rules ----------
echo "=== Task 1: Firewall rules ==="
gcloud compute firewall-rules create default-allow-http \
  --network=default --direction=INGRESS --priority=1000 --action=ALLOW \
  --rules=tcp:80 --source-ranges=0.0.0.0/0 --target-tags=http-server

gcloud compute firewall-rules create default-allow-health-check \
  --network=default --direction=INGRESS --priority=1000 --action=ALLOW \
  --rules=tcp --source-ranges=130.211.0.0/22,35.191.0.0/16 --target-tags=http-server

# ---------- Task 2: Instance templates ----------
echo "=== Task 2: Instance templates ==="
gcloud compute instance-templates create $T1 \
  --machine-type=e2-micro --network=default --subnet=default --region=$REGION_1 \
  --tags=http-server \
  --metadata=startup-script-url=gs://spls/gsp215/gcpnet/httplb/startup.sh

gcloud compute instance-templates create $T2 \
  --machine-type=e2-micro --network=default --subnet=default --region=$REGION_2 \
  --tags=http-server \
  --metadata=startup-script-url=gs://spls/gsp215/gcpnet/httplb/startup.sh

# ---------- Task 2: Managed instance groups ----------
echo "=== Task 2: Managed instance groups ==="
gcloud compute instance-groups managed create $MIG1 \
  --template=$T1 --size=1 --region=$REGION_1
gcloud compute instance-groups managed set-autoscaling $MIG1 \
  --region=$REGION_1 --min-num-replicas=1 --max-num-replicas=2 \
  --target-cpu-utilization=0.8 --cool-down-period=45

gcloud compute instance-groups managed create $MIG2 \
  --template=$T2 --size=1 --region=$REGION_2
gcloud compute instance-groups managed set-autoscaling $MIG2 \
  --region=$REGION_2 --min-num-replicas=1 --max-num-replicas=2 \
  --target-cpu-utilization=0.8 --cool-down-period=45

# Named port so the backend service can reach port 80
gcloud compute instance-groups set-named-ports $MIG1 --named-ports=http:80 --region=$REGION_1
gcloud compute instance-groups set-named-ports $MIG2 --named-ports=http:80 --region=$REGION_2

echo "Waiting for instance groups to become stable..."
gcloud compute instance-groups managed wait-until $MIG1 --stable --region=$REGION_1
gcloud compute instance-groups managed wait-until $MIG2 --stable --region=$REGION_2

# ---------- Task 3: Load balancer ----------
echo "=== Task 3: Health check ==="
gcloud compute health-checks create tcp http-health-check --port=80

echo "=== Task 3: Backend service ==="
gcloud compute backend-services create http-backend \
  --protocol=HTTP --port-name=http --health-checks=http-health-check \
  --global --enable-logging --logging-sample-rate=1

gcloud compute backend-services add-backend http-backend --global \
  --instance-group=$MIG1 --instance-group-region=$REGION_1 \
  --balancing-mode=RATE --max-rate-per-instance=50 --capacity-scaler=1.0

gcloud compute backend-services add-backend http-backend --global \
  --instance-group=$MIG2 --instance-group-region=$REGION_2 \
  --balancing-mode=UTILIZATION --max-utilization=0.8 --capacity-scaler=1.0

echo "=== Task 3: URL map, proxy, forwarding rules ==="
gcloud compute url-maps create http-lb --default-service=http-backend
gcloud compute target-http-proxies create http-lb-target-proxy --url-map=http-lb

gcloud compute forwarding-rules create http-lb-forwarding-rule \
  --global --target-http-proxy=http-lb-target-proxy --ports=80 --ip-version=IPV4
gcloud compute forwarding-rules create http-lb-forwarding-rule-ipv6 \
  --global --target-http-proxy=http-lb-target-proxy --ports=80 --ip-version=IPV6

export LB_IP=$(gcloud compute forwarding-rules describe http-lb-forwarding-rule --global --format='get(IPAddress)')
echo "Load balancer IPv4: $LB_IP"

# ---------- Task 4: Test the load balancer ----------
echo "=== Task 4: Waiting for load balancer to serve traffic (up to ~10 min) ==="
for i in {1..60}; do
  CODE=$(curl -s -o /dev/null -m 5 -w "%{http_code}" http://$LB_IP)
  if [ "$CODE" = "200" ]; then echo "Load balancer is serving traffic."; break; fi
  echo "Not ready yet (HTTP $CODE), retrying ($i/60)..."; sleep 10
done
curl -s -m 5 http://$LB_IP | head -20

echo "=== Task 4: Creating siege-vm ==="
gcloud compute instances create siege-vm --zone=$SIEGE_ZONE --machine-type=e2-medium

echo "Waiting for siege-vm SSH..."
sleep 30
gcloud compute ssh siege-vm --zone=$SIEGE_ZONE --quiet \
  --command="sudo apt-get update && sudo apt-get -y install siege"

echo "=== Task 4: Stress test (120 seconds) ==="
gcloud compute ssh siege-vm --zone=$SIEGE_ZONE --quiet \
  --command="siege -c 150 -t120s http://$LB_IP"

# ---------- Task 5: Cloud Armor denylist ----------
echo "=== Task 5: Cloud Armor policy ==="
export SIEGE_IP=$(gcloud compute instances describe siege-vm --zone=$SIEGE_ZONE --format='get(networkInterfaces[0].accessConfigs[0].natIP)')
echo "Siege VM IP: $SIEGE_IP"

gcloud compute security-policies create denylist-siege --description="Denylist siege-vm"

gcloud compute security-policies rules create 1000 \
  --security-policy=denylist-siege \
  --src-ip-ranges="${SIEGE_IP}/32" \
  --action=deny-403

gcloud compute backend-services update http-backend \
  --security-policy=denylist-siege --global

echo "=== Verify: siege-vm should get 403 (can take a few minutes) ==="
for i in {1..20}; do
  OUT=$(gcloud compute ssh siege-vm --zone=$SIEGE_ZONE --quiet --command="curl -s -m 5 http://$LB_IP")
  if echo "$OUT" | grep -q "403"; then echo "Got 403 Forbidden as expected."; break; fi
  echo "Policy not active yet, retrying ($i/20)..."; sleep 15
done

echo "=== Verify: Cloud Shell should still work (default rule allows) ==="
curl -s -m 5 http://$LB_IP | head -5

echo "=== Generating blocked traffic for the logs ==="
gcloud compute ssh siege-vm --zone=$SIEGE_ZONE --quiet \
  --command="siege -c 150 -t120s http://$LB_IP" >/dev/null 2>&1

echo "Done! Now click 'Check my progress' for each task."
```

### Congratulations 🎉 for completing the Lab !

##### ***You Have Successfully Demonstrated Your Skills And Determination.***

##### ***Well done!***
