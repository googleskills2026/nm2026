# Cloud Monitoring: Qwik Start || GSP089 ||

### Run the following Commands in CloudShell

```bash
curl -LO raw.githubusercontent.com/Cloud-Wala-Banda/Labs-Solutions/main/Cloud%20Monitoring%20Qwik%20Start/gsp089.sh
sudo chmod +x gsp089.sh
./gsp089.sh
```

#### Next, do this manually (Console steps)

1. Go to **Monitoring > Uptime checks > Create Uptime Check**.
2. Protocol: **HTTP**.
3. Resource Type: **URL**, with Hostname set to your VM's external IP (plain IP, no `http://`).
4. Check frequency: **1 minute**, then **Continue**.
5. Response Validation: defaults, **Continue**. Alert & Notification: defaults, **Continue**.
6. Title: `Lamp Uptime Check`.
7. Click **Test**, wait for the green check, then **Create**.

#### For the IP (or in the console)

1. Go to **Compute Engine > VM instances**.
2. Find `lamp-1-vm` and look at the **External IP** column.
3. If you can't see that column, click the **Column display options** icon (top right of the table), tick **External IP**, and click **OK**.

### Congratulations 🎉 for completing the Lab !

##### ***You Have Successfully Demonstrated Your Skills And Determination.***

##### ***Well done!***
