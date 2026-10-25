# Dataflow: Qwik Start - Python

### Run the following Commands in CloudShell

> Fill in the blank `export` values from your lab instructions before running.

```bash
export LOCATION=
curl -LO raw.githubusercontent.com/prateekrajput08/Arcade-Google-Cloud-Labs/refs/heads/main/Dataflow%3A%20Qwik%20Start%20-%20Python/TechCode.sh
sudo chmod +x TechCode.sh
./TechCode.sh
```

#### Then run this

```bash
gcloud config set compute/region us-central1
gcloud services disable dataflow.googleapis.com
gcloud services enable dataflow.googleapis.com
docker run -it -e DEVSHELL_PROJECT_ID=$DEVSHELL_PROJECT_ID python:3.12 /bin/bash

pip install 'apache-beam[gcp]'==2.67.0
BUCKET=gs://$DEVSHELL_PROJECT_ID-bucket

python -m apache_beam.examples.wordcount --project $DEVSHELL_PROJECT_ID \
  --runner DataflowRunner \
  --staging_location $BUCKET/staging \
  --temp_location $BUCKET/temp \
  --output $BUCKET/results/output \
  --region us-central1 \
  --worker_machine_type=e2-standard-2
```

#### Next, do this manually

1. Go to **Navigation menu > Dataflow > Jobs** and wait for the wordcount job to show **Succeeded**.
2. Click **Check my progress** on "Run an Example Pipeline Remotely".

### Congratulations 🎉 for completing the Lab !

##### ***You Have Successfully Demonstrated Your Skills And Determination.***

##### ***Well done!***
