# Cloud Run Functions: Qwik Start - Console

### Run the following Commands in CloudShell

```bash
REGION="europe-west4"   # use the region from your lab instructions
gcloud config set run/region $REGION

# Create the function (skip if you already made it in the console)
mkdir -p ~/gcfunction && cd ~/gcfunction
cat > index.js <<'EOF'
const functions = require('@google-cloud/functions-framework');
functions.http('helloHttp', (req, res) => {
  res.send(`Hello ${req.query.name || req.body.name || 'World'}!`);
});
EOF
cat > package.json <<'EOF'
{"name":"gcfunction","version":"1.0.0","main":"index.js","dependencies":{"@google-cloud/functions-framework":"^3.0.0"}}
EOF
gcloud functions deploy gcfunction --gen2 --runtime=nodejs22 --region=$REGION --source=. --entry-point=helloHttp --trigger-http --allow-unauthenticated --max-instances=5 --quiet

# Match the lab settings
gcloud run services update gcfunction --region=$REGION --max-instances=5 --execution-environment=gen2
gcloud run services add-iam-policy-binding gcfunction --region=$REGION --member=allUsers --role=roles/run.invoker

# Test using their own URL
URL=$(gcloud run services describe gcfunction --region=$REGION --format='value(status.url)')
curl -X POST "$URL" -H "Content-Type: application/json" -d '{"message":"Hello World!"}'
```

### Congratulations 🎉 for completing the Lab !

##### ***You Have Successfully Demonstrated Your Skills And Determination.***

##### ***Well done!***
