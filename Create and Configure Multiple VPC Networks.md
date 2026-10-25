# Create and Configure Multiple VPC Networks || GSP211 ||

### Run the following Commands in CloudShell

```bash
#!/bin/bash
# GSP211 - Create and Configure Multiple VPC Networks
# Run in Cloud Shell. Edit REGION_1 / ZONE_1 / REGION_2 if your lab uses different values.

export REGION_1="us-east4"
export ZONE_1="us-east4-a"
export REGION_2="asia-southeast1"

gcloud config set project $(gcloud config get-value project) >/dev/null 2>&1

echo "=== Task 1: managementnet network + subnet ==="
gcloud compute networks create managementnet --subnet-mode=custom
gcloud compute networks subnets create managementsubnet-1 \
  --network=managementnet --region=$REGION_1 --range=10.130.0.0/20

echo "=== Task 1: privatenet network + subnets ==="
gcloud compute networks create privatenet --subnet-mode=custom
gcloud compute networks subnets create privatesubnet-1 \
  --network=privatenet --region=$REGION_1 --range=172.16.0.0/24
gcloud compute networks subnets create privatesubnet-2 \
  --network=privatenet --region=$REGION_2 --range=172.20.0.0/20

echo "=== Task 1: Firewall rules ==="
gcloud compute firewall-rules create managementnet-allow-icmp-ssh-rdp \
  --direction=INGRESS --priority=1000 --network=managementnet --action=ALLOW \
  --rules=icmp,tcp:22,tcp:3389 --source-ranges=0.0.0.0/0

gcloud compute firewall-rules create privatenet-allow-icmp-ssh-rdp \
  --direction=INGRESS --priority=1000 --network=privatenet --action=ALLOW \
  --rules=icmp,tcp:22,tcp:3389 --source-ranges=0.0.0.0/0

echo "=== Task 2: VM instances ==="
gcloud compute instances create managementnet-vm-1 \
  --zone=$ZONE_1 --machine-type=e2-micro --subnet=managementsubnet-1

gcloud compute instances create privatenet-vm-1 \
  --zone=$ZONE_1 --machine-type=e2-micro --subnet=privatesubnet-1

echo "=== Task 4: Multi-NIC VM (vm-appliance) ==="
gcloud compute instances create vm-appliance \
  --zone=$ZONE_1 --machine-type=e2-standard-4 \
  --network-interface=network=privatenet,subnet=privatesubnet-1 \
  --network-interface=network=managementnet,subnet=managementsubnet-1 \
  --network-interface=network=mynetwork,subnet=mynetwork

echo "=== Verification ==="
gcloud compute networks list
gcloud compute networks subnets list --sort-by=NETWORK
gcloud compute firewall-rules list --sort-by=NETWORK
gcloud compute instances list --sort-by=ZONE

echo "Done! Now click 'Check my progress' for each task."
```

### Congratulations 🎉 for completing the Lab !

##### ***You Have Successfully Demonstrated Your Skills And Determination.***

##### ***Well done!***
