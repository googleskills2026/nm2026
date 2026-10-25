# Enhance Application Reliability and Scalability with Internal Load Balancing || GSP216 ||

> This lab has a console part (Step 1) and a Cloud Shell part (Step 2).

### Step 1: Create the two instance templates in the console

#### Template 1

1. Go to **Compute Engine > Instance templates > Create instance template**.
2. Name: `instance-template-1`. Location: Global. Series: E2. Machine type: e2-micro.
3. Open **Advanced options > Networking**.
   - Network tags: `lb-backend`.
   - Network interface: network `my-internal-app`, subnetwork `subnet-a`, External IPv4 address **None**. Click **Done**.
4. Open **Management**, scroll to **Automation > Startup script**, and paste this:

   ```bash
   #!/bin/bash
   apt-get update
   apt-get install -y apache2 php libapache2-mod-php
   cat <<'EOF' > /var/www/html/index.php
   <h1>Internal Load Balancing Lab</h1>
   <h2>Client IP</h2>
   Your IP address : <?php echo $_SERVER['REMOTE_ADDR']; ?>
   <h2>Hostname</h2>
   Server Hostname: <?php echo gethostname(); ?>
   <h2>Server Location</h2>
   Region and Zone: <?php
     $ch = curl_init();
     curl_setopt($ch, CURLOPT_URL, "http://metadata.google.internal/computeMetadata/v1/instance/zone");
     curl_setopt($ch, CURLOPT_HTTPHEADER, array('Metadata-Flavor: Google'));
     curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
     $zone = curl_exec($ch);
     $parts = explode('/', $zone);
     echo end($parts);
   ?>
   EOF
   rm -f /var/www/html/index.html
   systemctl restart apache2
   ```

5. Click **Create**.

#### Template 2

1. Tick `instance-template-1` and click **Copy**.
2. Name: `instance-template-2`.
3. Under **Advanced options > Networking**, change the subnetwork to `subnet-b`. Keep External IPv4 as **None** and the tag as `lb-backend`.
4. Click **Create**.

### Step 2: Run the script

Paste this whole block into Cloud Shell. It skips creating any template that already exists.

```bash
cat > gsp216.sh <<'SCRIPT_END'
#!/bin/bash
# GSP216 - Internal Load Balancing
# Override if your lab differs: REGION=us-west2 ZONE1=us-west2-a ZONE2=us-west2-b bash gsp216.sh
REGION=${REGION:-us-east1}
ZONE1=${ZONE1:-us-east1-c}
ZONE2=${ZONE2:-us-east1-d}
NET=my-internal-app
UTIL_IP=${UTIL_IP:-10.10.20.50}
ILB_IP=${ILB_IP:-10.10.30.5}
gcloud config set compute/region $REGION >/dev/null 2>&1

# Task 1: firewall rules
gcloud compute firewall-rules create app-allow-http --network=$NET --direction=INGRESS --action=allow --rules=tcp:80 --source-ranges=10.10.0.0/16 --target-tags=lb-backend
gcloud compute firewall-rules create app-allow-health-check --network=$NET --direction=INGRESS --action=allow --rules=tcp --source-ranges=130.211.0.0/22,35.191.0.0/16 --target-tags=lb-backend

# Task 2: templates (only created here if you skipped the console step)
cat > startup.sh <<'STARTUP'
#!/bin/bash
apt-get update
apt-get install -y apache2 php libapache2-mod-php
cat <<'EOF' > /var/www/html/index.php
<h1>Internal Load Balancing Lab</h1>
<h2>Client IP</h2>
Your IP address : <?php echo $_SERVER['REMOTE_ADDR']; ?>
<h2>Hostname</h2>
Server Hostname: <?php echo gethostname(); ?>
<h2>Server Location</h2>
Region and Zone: <?php
  $ch = curl_init();
  curl_setopt($ch, CURLOPT_URL, "http://metadata.google.internal/computeMetadata/v1/instance/zone");
  curl_setopt($ch, CURLOPT_HTTPHEADER, array('Metadata-Flavor: Google'));
  curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
  $zone = curl_exec($ch);
  $parts = explode('/', $zone);
  echo end($parts);
?>
EOF
rm -f /var/www/html/index.html
systemctl restart apache2
STARTUP
for T in 1 2; do
  if [ $T = 1 ]; then SUB=subnet-a; else SUB=subnet-b; fi
  gcloud compute instance-templates describe instance-template-$T >/dev/null 2>&1 || \
  gcloud compute instance-templates create instance-template-$T \
    --region=$REGION --machine-type=e2-micro --network=$NET --subnet=$SUB --no-address \
    --tags=lb-backend --image-family=debian-12 --image-project=debian-cloud \
    --metadata-from-file=startup-script=startup.sh
done

# Task 2: managed instance groups
gcloud compute instance-groups managed create instance-group-1 --template=instance-template-1 --size=1 --zone=$ZONE1
gcloud compute instance-groups managed set-autoscaling instance-group-1 --zone=$ZONE1 --min-num-replicas=1 --max-num-replicas=1 --target-cpu-utilization=0.8 --cool-down-period=45
gcloud compute instance-groups managed create instance-group-2 --template=instance-template-2 --size=1 --zone=$ZONE2
gcloud compute instance-groups managed set-autoscaling instance-group-2 --zone=$ZONE2 --min-num-replicas=1 --max-num-replicas=1 --target-cpu-utilization=0.8 --cool-down-period=45

# Task 2: utility-vm
gcloud compute instances create utility-vm --zone=$ZONE1 --machine-type=e2-micro \
  --network=$NET --subnet=subnet-a --private-network-ip=$UTIL_IP \
  --image-family=debian-12 --image-project=debian-cloud

gcloud compute instance-groups managed wait-until instance-group-1 --stable --zone=$ZONE1
gcloud compute instance-groups managed wait-until instance-group-2 --stable --zone=$ZONE2

# Task 3: internal load balancer
gcloud compute health-checks create tcp my-ilb-health-check --region=$REGION --port=80
gcloud compute backend-services create my-ilb --load-balancing-scheme=internal --protocol=tcp --region=$REGION --health-checks=my-ilb-health-check --health-checks-region=$REGION
gcloud compute backend-services add-backend my-ilb --region=$REGION --instance-group=instance-group-1 --instance-group-zone=$ZONE1
gcloud compute backend-services add-backend my-ilb --region=$REGION --instance-group=instance-group-2 --instance-group-zone=$ZONE2
gcloud compute addresses create my-ilb-ip --region=$REGION --subnet=subnet-b --addresses=$ILB_IP
gcloud compute forwarding-rules create my-ilb-forwarding-rule --load-balancing-scheme=internal --network=$NET --subnet=subnet-b --region=$REGION --address=my-ilb-ip --ports=80 --backend-service=my-ilb

# Task 4: test from utility-vm
echo "Waiting for Apache on the backends..."
sleep 60
for i in {1..30}; do
  OUT=$(gcloud compute ssh utility-vm --zone=$ZONE1 --quiet --command="curl -s -m 3 $ILB_IP" 2>/dev/null)
  echo "$OUT" | grep -q "Hostname" && { echo "Internal load balancer is serving."; break; }
  echo "Not ready yet ($i/30)..."; sleep 10
done
gcloud compute ssh utility-vm --zone=$ZONE1 --quiet \
  --command="for i in 1 2 3 4 5 6; do curl -s -m 3 $ILB_IP | grep -o 'instance-group-[0-9a-z-]*'; done"
echo "Done! Now click 'Check my progress' for each task."
SCRIPT_END
bash gsp216.sh
```

### Step 3: Check your score

Click Check my progress for all three checkpoints. The six test requests at the end should show a mix of instance-group-1-xxxx and instance-group-2-xxxx.

### Congratulations 🎉 for completing the Lab !

##### ***You Have Successfully Demonstrated Your Skills And Determination.***

##### ***Well done!***
