# Cloud Monitoring: Qwik Start || GSP089 ||

### Run the following Commands in CloudShell

```bash
#!/bin/bash
# GSP089 - Cloud Monitoring: Qwik Start
# Run in Cloud Shell: bash gsp089.sh

# ---------- Zone (press Enter for default, or type your lab's zone) ----------
read -e -p "Zone [us-east4-b]: " Z; export ZONE=${Z:-us-east4-b}
export REGION=${ZONE%-*}
export PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
gcloud config set compute/zone $ZONE >/dev/null 2>&1
gcloud config set compute/region $REGION >/dev/null 2>&1
echo "Project: $PROJECT_ID | Region: $REGION | Zone: $ZONE"

# Optional: email for the alert (press Enter to skip)
read -e -p "Email for alert notifications (optional, Enter to skip): " ALERT_EMAIL

gcloud services enable compute.googleapis.com monitoring.googleapis.com logging.googleapis.com

# ---------- Task 1: Create the VM ----------
echo "=== Task 1: Firewall rule for HTTP (if missing) ==="
gcloud compute firewall-rules describe default-allow-http >/dev/null 2>&1 || \
gcloud compute firewall-rules create default-allow-http \
  --network=default --direction=INGRESS --action=ALLOW \
  --rules=tcp:80 --source-ranges=0.0.0.0/0 --target-tags=http-server

echo "=== Task 1: Creating lamp-1-vm ==="
gcloud compute instances create lamp-1-vm \
  --zone=$ZONE --machine-type=e2-medium \
  --image-family=debian-12 --image-project=debian-cloud \
  --tags=http-server

echo "Waiting for VM to accept SSH..."
sleep 40

# ---------- Task 2: Apache + PHP ----------
echo "=== Task 2: Installing Apache2 and PHP ==="
gcloud compute ssh lamp-1-vm --zone=$ZONE --quiet --command="sudo apt-get update && sudo DEBIAN_FRONTEND=noninteractive apt-get install -y apache2 php && sudo service apache2 restart"

export VM_IP=$(gcloud compute instances describe lamp-1-vm --zone=$ZONE --format='get(networkInterfaces[0].accessConfigs[0].natIP)')
echo "VM external IP: $VM_IP"
echo "Testing Apache over external IP..."
for i in {1..12}; do
  CODE=$(curl -s -o /dev/null -m 5 -w "%{http_code}" http://$VM_IP)
  [ "$CODE" = "200" ] && { echo "Apache is serving (HTTP 200)."; break; }
  echo "HTTP $CODE, retrying ($i/12)..."; sleep 10
done

# ---------- Install Ops Agent ----------
echo "=== Installing Ops Agent ==="
gcloud compute ssh lamp-1-vm --zone=$ZONE --quiet --command="curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh && sudo bash add-google-cloud-ops-agent-repo.sh --also-install && sudo systemctl status google-cloud-ops-agent'*' --no-pager | head -20"

# ---------- Task 3: Uptime check ----------
echo "=== Task 3: Uptime check ==="
gcloud monitoring uptime create "Lamp Uptime Check" \
  --resource-type=uptime-url \
  --resource-labels=host=$VM_IP,project_id=$PROJECT_ID \
  --protocol=http --period=1 --path=/ \
  || echo "!! Uptime check failed via CLI. Create it in Console: Monitoring > Uptime checks (HTTP, URL, host=$VM_IP, 1 minute, title 'Lamp Uptime Check')."

# ---------- Task 4: Alerting policy ----------
echo "=== Task 4: Alerting policy ==="
CHANNEL_FLAG=""
if [ -n "$ALERT_EMAIL" ]; then
  CH=$(gcloud beta monitoring channels create \
    --display-name="Lab Email" --type=email \
    --channel-labels=email_address=$ALERT_EMAIL \
    --format='value(name)')
  [ -n "$CH" ] && CHANNEL_FLAG="--notification-channels=$CH"
fi

cat > alert-policy.json <<'EOF'
{
  "displayName": "Inbound Traffic Alert",
  "combiner": "OR",
  "documentation": {
    "content": "Inbound network traffic on lamp-1-vm is above the threshold.",
    "mimeType": "text/markdown"
  },
  "conditions": [
    {
      "displayName": "Network traffic above 500",
      "conditionThreshold": {
        "filter": "metric.type=\"agent.googleapis.com/interface/traffic\" AND resource.type=\"gce_instance\"",
        "comparison": "COMPARISON_GT",
        "thresholdValue": 500,
        "duration": "60s",
        "aggregations": [
          { "alignmentPeriod": "60s", "perSeriesAligner": "ALIGN_RATE" }
        ]
      }
    }
  ]
}
EOF
gcloud alpha monitoring policies create --policy-from-file=alert-policy.json $CHANNEL_FLAG \
  || echo "!! Alert policy failed via CLI. Create it in Console: Monitoring > Alerting."

# ---------- Task 5: Dashboard ----------
echo "=== Task 5: Dashboard ==="
cat > dashboard.json <<'EOF'
{
  "displayName": "Cloud Monitoring LAMP Qwik Start Dashboard",
  "gridLayout": {
    "columns": "2",
    "widgets": [
      {
        "title": "CPU Load",
        "xyChart": {
          "dataSets": [{
            "timeSeriesQuery": {
              "timeSeriesFilter": {
                "filter": "metric.type=\"compute.googleapis.com/instance/cpu/load_1m\" resource.type=\"gce_instance\"",
                "aggregation": { "perSeriesAligner": "ALIGN_MEAN" }
              }
            },
            "plotType": "LINE"
          }]
        }
      },
      {
        "title": "Received Packets",
        "xyChart": {
          "dataSets": [{
            "timeSeriesQuery": {
              "timeSeriesFilter": {
                "filter": "metric.type=\"compute.googleapis.com/instance/network/received_packets_count\" resource.type=\"gce_instance\"",
                "aggregation": { "perSeriesAligner": "ALIGN_RATE" }
              }
            },
            "plotType": "LINE"
          }]
        }
      }
    ]
  }
}
EOF
gcloud monitoring dashboards create --config-from-file=dashboard.json \
  || echo "!! Dashboard failed via CLI. Create it in Console: Monitoring > Dashboards."

# ---------- Task 6: Logs ----------
echo "=== Task 6: Recent logs for lamp-1-vm ==="
gcloud logging read 'resource.type="gce_instance" AND protoPayload.resourceName:"lamp-1-vm"' --limit=5 --format="value(timestamp,protoPayload.methodName)"
echo "Done! Click 'Check my progress' for each task."
```

### Congratulations 🎉 for completing the Lab !

##### ***You Have Successfully Demonstrated Your Skills And Determination.***

##### ***Well done!***
