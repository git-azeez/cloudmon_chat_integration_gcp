# Cloud Monitoring to Google Chat
This is the code referenced in my Medium article [Cloud Monitoring, We Need to Chat].

[Cloud Monitoring, We Need to Chat]: https://levelup.gitconnected.com/cloud-monitoring-we-need-to-chat-54c45b346a21




====================================================================================
Cloud Monitoring, We Need to Chat

Photo by Volodymyr Hryshchenko on Unsplash
Cloud Monitoring is an essential tool for tracking application metrics and errors of workloads running on Google Cloud Platform. Besides its dashboarding capabilities, it lets you define alerting policies so that you’re automatically notified whenever an error is thrown or a metric is above its threshold.

Cloud Monitoring supports different notification channels such as e-mail, SMS or Slack, but strangely not Google’s own chat app that is part of Google Workspace. I personally prefer getting the alerts in a team chat rather than per e-mail so that the team can coordinate the resolution of the incident more easily.

Luckily, we can build the missing integration with Google Chat ourselves in almost no time using Cloud Functions and Pub/Sub. Here’s the setup:


Alerts will be published to Pub/Sub and then forwarded to Google Chat via Cloud Functions
We start by configuring Google Chat and work backwards to the alerting policies in Cloud Monitoring.

First, we need to create a webhook in the Google Chat Space we want the alerts to be published.


Create a webhook, copy the URL
Next, we write the Cloud Function that forwards the monitoring alerts to the webhook URL that was generated in the previous step. The reason we can’t use this webhook directly as a notification channel in Cloud Monitoring but need a Cloud Function in between, is that Google Chat requires a different message payload structure than the one Cloud Monitoring produces. As described in the documentation, the alert message is a JSON of this form:


Source: https://cloud.google.com/monitoring/support/notification-options#pubsub
Google Chat, however, expects the payload to look like this: {"text": "Here goes the message content"}. Therefore, we define a message template for Google Chat with placeholders for the alert parameters of interest (e.g. incident[summary]) and have a Cloud Function replace the placeholders with the actual values from the alert and forward the resulting JSON to Google Chat.

Let’s use Python and the fancier card message format rather than the basic text format. It basically just takes three lines of code and the message template (the full code is also on Github):


Note that instead of defining the template in the form of a dictionary and looping over all its nested levels to replace the placeholders, we define the template as a JSON string, use format() to replace, and then parse it into a dictionary. Which is one single line of code instead of many:

message = json.loads(MESSAGE_TEMPLATE.format(**alert))
The double curly braces {{...}} in the template string are needed for escaping when applying format().

Time to deploy!

gcloud functions deploy monitoring-alerts --entry-point monitoring_alerts --region europe-west6 --runtime python39 
--trigger-topic monitoring-alerts
The Pub/Sub topic monitoring-alerts is automatically created for us. So, all that’s left to do is to set up a corresponding Pub/Sub notification channel and use it in our alerting policies.


Create a notification channel for the monitoring alerts…

… and use it in the alerting policies
That’s it. Now enjoy your alert messages showing up in Google Chat (but don’t forget to follow up on them).


Oh hello, you pretty error
104


