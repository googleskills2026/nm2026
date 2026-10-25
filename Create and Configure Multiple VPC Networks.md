# Create and Configure Multiple VPC Networks || GSP211 ||

### Run the following Commands in CloudShell

```bash
cat > gsp211.sh <<'SCRIPT_END'
#!/bin/bash
# GSP211 - Create and Configure Multiple VPC Networks
# Region/zone are detected from the pre-created mynet VMs.
# Override if needed:  ZONE_1=us-east4-b ZONE_2=europe-west4-a bash gsp211.sh
ZONE_1=${ZONE_1:-$(gcloud compute instances list --filter="name=mynet-vm-1" --format="value(zone.basename())")}
ZONE_2=${ZONE_2:-$(gcloud compute instances list --filter="name=mynet-vm-2" --format="value(zone.basename())")}
ZONE_1=${ZONE_1:-us-east4-b}
ZONE_2=${ZONE_2:-europe-west4-a}
REGION_1=${ZONE_1%-*}
REGION_2=${ZONE_2%-*}
echo "Region 1: $REGION_1 | Zone 1: $ZONE_1 | Region 2: $REGION_2"

# Task 1: managementnet
gcloud compute networks create managementnet --subnet-mode=custom
gcloud compute networks subnets create managementsubnet-1 \
  --network=managementnet --region=$REGION_1 --range=10.130.0.0/20

# Task 1: privatenet
gcloud compute networks create privatenet --subnet-mode=custom
gcloud compute networks subnets create privatesubnet-1 \
  --network=privatenet --region=$REGION_1 --range=172.16.0.0/24
gcloud compute networks subnets create privatesubnet-2 \
  --network=privatenet --region=$REGION_2 --range=172.20.0.0/20

# Task 1: firewall rules
gcloud compute firewall-rules create managementnet-allow-icmp-ssh-rdp \
  --direction=INGRESS --priority=1000 --network=managementnet --action=ALLOW \
  --rules=icmp,tcp:22,tcp:3389 --source-ranges=0.0.0.0/0
gcloud compute firewall-rules create privatenet-allow-icmp-ssh-rdp \
  --direction=INGRESS --priority=1000 --network=privatenet --action=ALLOW \
  --rules=icmp,tcp:22,tcp:3389 --source-ranges=0.0.0.0/0

# Task 2: VMs
gcloud compute instances create managementnet-vm-1 \
  --zone=$ZONE_1 --machine-type=e2-micro --subnet=managementsubnet-1
gcloud compute instances create privatenet-vm-1 \
  --zone=$ZONE_1 --machine-type=e2-micro --subnet=privatesubnet-1

# Task 4: multi-NIC VM
gcloud compute instances create vm-appliance \
  --zone=$ZONE_1 --machine-type=e2-standard-4 \
  --network-interface=network=privatenet,subnet=privatesubnet-1 \
  --network-interface=network=managementnet,subnet=managementsubnet-1 \
  --network-interface=network=mynetwork,subnet=mynetwork

# Verify
gcloud compute networks list
gcloud compute networks subnets list --sort-by=NETWORK
gcloud compute firewall-rules list --sort-by=NETWORK
gcloud compute instances list --sort-by=ZONE
echo "Done! Now click 'Check my progress' for each task."
SCRIPT_END
bash gsp211.sh
```

### Congratulations 🎉 for completing the Lab !

##### ***You Have Successfully Demonstrated Your Skills And Determination.***

##### ***Well done!***
