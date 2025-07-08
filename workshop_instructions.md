# Unit 10 Workshop Instructions

Click on the Order Processing App Service in the Azure portal and then the URL.
You'll see a dashboard showing one successfully processed order and a queue of orders to be processed, but no further orders are being processed.
We need to investigate why.
* The python code for the order processing app can be found in the root of this project

Back in the Azure Portal (still on the Order Processing App Service page) click on "Log Stream" and you'll see a number of errors like:

```text
requests.exceptions.HTTPError: 400 Client Error: Bad Request for url: https://XXX-order-processing-finance-package.azurewebsites.net/ProcessPayment
```

Sadly this order-processing-finance-package is a third party tool, so we'll need to investigate this issue from our end, and see if we can fix it, or if the problem needs to be raised with the supplier.

The logstream includes a stack trace - the files and line numbers of what was being executed when the error occurred. Some of the stack trace will point to other packages' code, like the `requests` library, but one line of the stack trace points to your application code. Have a look at the source code and identify the line where the error is thrown..

<details><summary>Hint</summary>

In the middle of the stack trace you should see it mention "scheduled_jobs.py", line 37.

</details>

There is no obvious cause to this issue, but you can see that the system picks up the oldest unprocessed order and tries to process it. It makes a request to a "finance package" web API but if an error response is received then the job aborts. So the order remains unprocessed and will be retried endlessly, and later orders remain unprocessed.

We don't have enough information to work out whats going wrong. We need more logging.

## Adding more logging to the app

We can add more logging with a line like the following:

```python
app.logger.info("Response from endpoint: " + response.text)
```

<details><summary>Hint:</summary>

This needs to go after the response variable is defined, but before the error is raised. Nothing will get executed after the error is raised. 

</details> 

If you deploy the code now, this won't actually add anything to the log stream as sadly the default logging level doesn't include `info`.
Add the following to `app.py`

```python
import logging
logging.basicConfig(level=logging.INFO)
```

## Deploy your changes

For the purpose of this workshop there is a default CD pipeline configured in `.github/workflows/deploy.yml` or `.gitlab-ci.yml` depending on your platform. In order to make use of this you will need to:
* Update that pipeline file to contain your DockerHub username
* In Azure, under your App Service's "Deployment Center", update the image name/tag to `<your_dockerhub_username>/unit-10-order-processing-app`
* Grab the "Webhook URL" from the Deployment Center and add it to your CI pipeline secrets/variables as `AZURE_WEBHOOK`
  * If using GitLab, make sure that the "Expand variable" option is off, and the secret is masked and available to your branch
* [Generate a DockerHub token](https://docs.docker.com/docker-hub/access-tokens/#create-an-access-token) and set it in the repository pipeline secrets/variables under `DOCKERHUB_TOKEN`

When you're ready, push those changes to your fork which will update your DockerHub image and notify your server to pull the latest and restart.

If you find this step slow, or just prefer to release manually, then you will still need to update the App Service but can always build & push the latest image directly 
```bash
docker build --tag ${DOCKERHUB_USERNAME}/unit-10-order-processing-app --target production .
docker push ${DOCKERHUB_USERNAME}/unit-10-order-processing-app
```

You'll also need to trigger the webhook - you can do this directly, or you can register that with DockerHub to do that for you.

<details><summary>Register a webhook in DockerHub</summary>

DockerHub allows you to register webhooks on individual image repositories, you can find that through:
1) Signing in
1) Navigating to "My Profile" from the dropdown by your username
1) Selecting the relevant image repository
1) Selecting "Manage Repository"
1) Select the "Webhooks" tab and entering appropriate details

Or alternatively by jumping to this URL with the appropriate values completed:

https://hub.docker.com/repository/docker/\<YOUR_USERNAME\>/\<IMAGE_NAME\>/webhooks
</details>

## Investigate and fix

The log stream should now contain the response from the finance package - look for a line containing "Response from endpoint".
This still isn't quite enough information to fix the problem.
Add more logging for relevant information as above.
You could look at the database, but you shouldn't need to. In real world scenarios you would often not have access to the live database.

<details><summary>Hint</summary>

When logging outgoing API requests, you probably want to log what request you're making - in this case the most important info is the date in the `payload` dictionary).
  
Look at the whole date string from your logged message. The part at the end is a timezone.

</details>

Once you see what the problem is, try fixing it. Ultimately like so many bugs, investigating is what takes time - the actual fix should just be a single line change.

<details><summary>Hint</summary>

If you look in the `order.py` file, orders have a property called `date_placed_local` that should do what you want - convert from the +10 timezone to local. You simply need to use `date_placed_local` instead of `date_placed` when creating the payload.

> If you're not doing this workshop during BST, maybe you assumed you want UTC. That would be reasonable but the Finance API does in fact want "local".
  
</details>

Having deployed the fix the order that was stuck should be processed, and refreshing the home page of the app should show the queue go down as the orders are gradually processed.

> Note: The next parts assume you leave in the logging added above, so don't remove it when you have fixed the problem.

## Further improvements

You've solved the immediate issue, but there are still some fundamental problems:

* Even when a problem is known the log stream in Azure is quite bad.
* No one will be notified until customers start complaining.
* It's still the case that if something goes wrong with an order all orders will stop.

We need to improve this:

* We should use a tool such as Azure Application Insights (App Insights) to get a better view of the logs.
* We should set up alerts for when something goes wrong.
* Once we have visibility of orders that have failed, perhaps we should move on from them rather than endlessly failing.

## Add App Insights

Within your resource group create a new Application Insights resource.

Find the connection string (containing a sensitive "instrumentation key") on the overview page of your Application Insights resource. Rather than hardcoding the connection string in the Python code, configure it securely by using an environment variable called `APPLICATIONINSIGHTS_CONNECTION_STRING`. Set this on the 'Environment Variables' page of the App Service.

> Don't forget to click the `Apply` button at the bottom of the Environment Variables page after making changes! (you have to do this twice, once for the new variable and then again to confirm all the variables are correct!)

To actually send logs to Application Insights you'll need to add the Python packages `azure-monitor-opentelemetry` to requirements.txt.
It's a good habit to specify a version when adding these: you can find the latest version specified on the Python Package Index page for [the Azure extension](https://pypi.org/project/azure-monitor-opentelemetry/).

Next, we configure the logging in `app.py` by adapting this sample code: <https://github.com/Azure/azure-sdk-for-python/blob/main/sdk/monitor/azure-monitor-opentelemetry/samples/tracing/http_flask.py>. This will log all requests to your Flask app (like every time you view the webpage) and the in-built exporter will send those logs to Application Insights.

> The environment variable gets picked up automatically so your code doesn't need to pass it directly into the `configure_azure_monitor()` call. Just make sure the application setting in Azure is the whole connection string, including `InstrumentationKey=`!

> Note that the auto-configuration relies on a strict ordering and flask needs to be _imported_ after we have run the configuration - so you may have code like:
> ```python
> from azure.monitor.opentelemetry import configure_azure_monitor
> import logging
>
> logging.basicConfig(level=logging.INFO)
> configure_azure_monitor()
>
> from flask import Flask, render_template, request
> app = Flask(__name__)
> ...
> ```

A few minutes after deploying your changes (App Insights batches up log messages) you can see the logs.
Go the App Insights resource and then navigate to `Logs`.
The query `traces` should show up all the logging you added in the previous part, while `requests` will show you all requests handled by your Flask app.

You can also search it with queries (in the Kusto query language or "KQL"):
```text
traces | where message contains "Response from endpoint"
```

## Add Error Alerts

The second step we wanted to take was to send out alerts when something goes wrong.
We've fixed the problem though, so to start lets add a broken order.

Select "Add broken order" from the dropdown at the top of the app service homepage (i.e. the page that lists the orders).

This order from the year 3000 will fail as it isn't in the past.
It won't actually block new orders as it'll be after them in the queue.

Now search App Insights for `exceptions`.
You won't see any, even though the `traces` will show the expected output.

Exceptions don't automatically appear in App Insights like they did in the log stream so, put a `try...except` block in `scheduled_jobs.py` with the following:

```python
app.logger.exception("Error processing order {id}".format(id = order.id))
```

After deploying this you'll see results when you run the query `exceptions`.
The details of the exception are magically picked up by the logger.
Click "New Alert rule" to setup the rule.

You'll need to click on the condition to set it to alert when there is 1 or more result, and create an action group that sends you an email (or you can skip this if you want - there is also an alert dashboard).

Back in App Insights under Alerts you'll see an alert is raised.
With the default settings this will take up to 5 minutes.
If you set up emails above you'll start to receive these.

## Add Live Metrics

You may have noticed that even once the app is reporting logs to App Insights they lag the system slightly, e.g. searching your traces may not show logging statements from more than a minute ago. In many cases this is sufficient, but there may be times that we want lower latency on our stats - for example whilst monitoring a release.

Azure offers ["Live Metrics"](https://learn.microsoft.com/en-us/azure/azure-monitor/app/live-stream?tabs=otel) to help with this. Try turning that on now by adjusting your setup command to:
```python
configure_azure_monitor(enable_live_metrics = True)
```

Once that's deployed, check out the "Live Metrics" tab (under the "Investigate" menu) in Application Insights. Again, it might take a few minutes to appear, but once it does you'll see that the traces on display are coming through within a second or two of being generated.

## Creating a Dedicated Dashboard

Looking at the Live Metrics page is good for checking the **present** status of the system, but if we need information from, say, the past hour, we want a dedicated dashboard for this.

Conveniently Azure provides some [**Standard Metrics**](https://learn.microsoft.com/en-us/azure/azure-monitor/app/metrics-overview?tabs=standard#standard-metrics) for both App Services and App Service Plans (which you can find under Monitoring on the overview pages).

It would be good to have these in one place, along with any other metrics we create later.

On the burger menu at the top left of the Azure portal select the Dashboard option:
![Dashboard Button](./images/dashboard_button.jpg)

From here select "Create" and then choose "App Service Tracking", selecting your Order Processing App (you can create a separate dashboard for the Finance Processing Application if you like). Once the dashboard is ready have a look at the EPA criteria for monitoring below and discuss if everything here has now been covered:

![EPA Monitoring Criteria](./images/epa_monitoring_pass_criteria.jpg)

## Improve the queue

With the alerts in place you'll now receive an email every 5 minutes until Jan 1st 3000. You probably don't want this.

Now we have alerts in place perhaps rather than endlessly retrying we should give up on orders that error and move on to the next one.

You'll need to make it so when errors are thrown the order is somehow marked as failed, and not to be picked for processing anymore.
If you like, you could even update the HTML to mark these failed orders in red.

## Stretch: Queue reliability

To start this stretch scenario, select "Queue Reliability" from the scenario dropdown at the top of the orders dashboard.

After triggering this scenario you should notice a number of orders failing.
It seems that the Finance Package has become quite unreliable and is failing orders at random (reporting that `the 'payments' table is locked`).
If we retry them it might work.

You need to determine how to recognize such a transient error and stop the app from giving up on such orders, whilst still failing on permanent errors like the one we introduced above.

Having deployed that change, let's reconsider the alerts.

* If an order has failed we always want to send an alert.
* Occasionally having to retry shouldn't trigger and email
* All orders failing even if we are to retry should trigger and email

Change the alerts to fulfil these requirements.

## Stretch: Monitoring load

To start this stretch scenario, select "Monitoring Load" from the scenario dropdown at the top of the dashboard.

> Note this exercise is to monitor the situation, don't try to fix it as when peak load as passed the queue will be processed.

This system processes orders at a fixed rate (if there are any to be processed of course).
As you can see, if that rate is exceeded the queue will build up. 
This isn't an exception so won't be caught above but is something to monitor.
We will do this by monitoring the number of requests coming in to create new orders.

Since you set up the Flask middleware, all requests to your Flask app are being logged to Application Insights. You can run the query "requests" in Application Insights Logs too see all requests made to your application.

We are interested in the number of requests to `/new`. Try the following query:

```text
requests 
| where name == 'POST /new' and success == 'True' 
| project Kind="new", timestamp
| summarize new = count(Kind == "new") by bin(timestamp, 10m)
```

Not very useful in table form, but click "Chart".
This then shows how many orders have been created in 10 minute 'bins'.
Under chart formatting you can chose a better chart type for this type of data too.

Then choose pin to dashboard, and either select your previous dashboard or create a new dashboard.
There is an "open editing pane" button on the chart that lets you edit it some more.

This query only tells half the story - how much goes on to the queue not how much leaves it.
From your existing `traces` you should be able to create a chart of successfully processed orders.
With the `union` command you can combine these into a single chart.
Finally, using the existing commands you can add a third data series - how much the queue has increased or decreased by in that time period.

You can set an alert for when the queue is growing.
Prior to increasing the load you'll likely see 10 minute periods where the queue increased or decreased by one or so, so you'll want to set a slightly higher threshold than that.
You'll need to set the alert logic to based on metric measurement and change the summarize line to something like:

```text
| summarize AggregatedValue= count(Kind == "new") -  count(Kind == "processed") by bin(timestamp,  5m)
```

## Stretch: Further System monitoring

To start this stretch scenario, select "System Monitoring" from the scenario dropdown at the top of the orders dashboard.

If the load ramps up much more this will actually overwhelm the system.

The resources we need to monitor are the database and App Service Plan in your resource group.
You can see metrics on the main page of the resource and pin them to the dashboard, or on the Metrics page there is a larger selection of metrics to graph and pin.

After around 15 minutes the app will stop accepting new orders.
Identify which resource is at fault, and what there is a shortage of.
Now fix the problem.
As a rule of thumb storage is quite cheap to increase (as long as it isn't hugely excessive), so if that is the issue see if you can increase it.
CPU and RAM are often much more expensive to increase, so if that's the problem its often worth investigating if the code can be fixed to reduce its requirement.

Make sure that you have alerts set up so that if you run out again there will be an email giving some notice.

After about half an hour of an order being placed every second you should notice the processing speed slows down.
See if you can use the logging to work out and fix what is causing the problem, which will require performance tuning in the Python code.
