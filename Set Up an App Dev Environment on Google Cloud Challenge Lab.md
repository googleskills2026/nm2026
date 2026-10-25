# Set Up an App Dev Environment on Google Cloud: Challenge Lab || GSP315 ||

### Run the following Commands in CloudShell

> Fill in the blank `export` values from your lab instructions before running.

```bash
export USER_2=
export ZONE=
export TOPIC=
export FUNCTION=

curl -LO raw.githubusercontent.com/Cloud-Wala-Banda/Labs-Solutions/main/Set%20Up%20an%20App%20Dev%20Environment%20on%20Google%20Cloud%20Challenge%20Lab/gsp315.sh

sudo chmod +x gsp315.sh

./gsp315.sh
```

#### Then run this

```bash
gcloud projects get-iam-policy <PROJECT_ID> --format=json | grep -i "<USER_2_ID>" || echo "User 2 removed"
```

### Congratulations 🎉 for completing the Lab !

##### ***You Have Successfully Demonstrated Your Skills And Determination.***

##### ***Well done!***
