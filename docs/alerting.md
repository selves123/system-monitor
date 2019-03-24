---
layout: page
title: Alerting
order: 40
---

Alerters send one-off alerts when a monitor fails. They can also send an alert when it succeeds again.

An alerter knows if it is urgent or not; if a monitor defined as non-urgent fails, an urgent alerter will not trigger for it. This means you can avoid receiving SMS alerts for things which don’t require your immediate attention.

Alerters can also have a time configuration for hours when they are or are not allowed to alert. They can also send an alert at the end of the silence period for any monitors which are currently failed.

The types of alerter are:

* [email](#email): Sends an email when a monitor fails. Sends an email when it succeeds again. Requires an SMTP server to talk to. Non-urgent (all monitors will trigger this alerter.)
* [bulksms](#bulksms): Sends an SMS alert when a monitor fails. Does not send an alert for when it succeeds again. Uses the [BulkSMS](http://www.bulksms.co.uk) service, which requires subscription. The messages are sent over HTTP on port 5567. (Urgent, so urgent=0 monitors will not trigger an SMS.)
* [syslog](#syslog): Writes an entry to the syslog when something fails or succeeds. Not supported on Windows.
* [execute](#execute): Executes an arbitrary command when something fails or recovers.
* [slack](#slack): Sends notifications to a Slack channel using a webhook.
* [ses](#ses): Sends notifications via the Amazon Simple Email Service
* [46elks](#46elks): Sends notifications via the [46elks](https://46elks.com) service
* [pushbullet](#pushbullet): Sends notifications via [Pushbullet](https://www.pushbullet.com)
* [pushover](#pushover): Sends notifications via [Pushover](https://pushover.net)

## Defining an alerter
The section name should be the name of your alerter. This is the name you should give in the "alerters" setting in the reporting section of the main configuration. All alerters share these settings:

| setting | description | required | default |
|---|---|---|---|
|type|the type of the alerter, from the list above|yes| |
|depend|a list of monitors this alerter depends on. If any of them fail, no attempt will be made to send the alert. (For example, there's no point trying to send an email alert to an external address if your route(s) to the Internet are down.)|no| |
|limit|the number of times a monitor must fail before this alerter will fire. You can use this to escalate an alert to another email address if the problem is ongoing for too long, for example.|no|1|
|dry_run|makes an alerter do everything except actually send the message. Instead it will print some information about what it would do. Use when you want to test your configuration without generating emails/SMSes. Set to 1 to enable.|no|0|
|ooh_success|makes an alerter trigger its success action even if out-of-hours (0 or 1)|no|0|
|groups|comma-separated list of group names this alerter will fire for. See the `group` setting for monitors|no|`default`|

The *limit* uses the virtual fail count of a monitor, which means if a monitor has a tolerance of 3 and the alerter has a limit of 2, the monitor must fail 5 times before an alert is sent.

## Time periods
All alerters accept time period configuration. By default, an alerter is active at all times, so you will always immediately receive an alert at the point where a monitor has failed enough (more times than the *limit*). To set limits on when an alerter can send:

| setting | description | required | default |
|---|---|---|---|
|day|Which days an alerter can operate on. This is a comma-separated list of integers. 0 is Monday and 6 is Sunday.|no|(all days)|
|times_type|Set to one of always, only, or not. “Only” means that the limits define the period that an alerter can operate. “Not” means that the limits define the period during which it will not operate.|no|always|
|time_lower and time_upper| If *times_type* is only or not, these two settings define the time limits. time_lower must always be the lower time. The time format is hh:mm using 24-hour clock. Both are required if times_type is anything other than always.|when *times_type* is not `always`| |
|delay|If any kind of time/day restriction applies, the alerter will notify you of any monitors that failed while they were unable to alert you and are still failed. If a monitor fails and recovers during the restricted period, no catch-up alert is generated. Set to 1 to enable.|no|0|

Here’s a quick example of setting time periods (some other configuration values omitted):

Don’t send me SMSes while I’m in the office (8:30am to 5:30pm Mon-Fri):
{% highlight ini %}
[out_of_hours]
type=bulksms
times_type=not
time_lower=08:30
time_upper=17:30
days=0,1,2,3,4
{% endhighlight %}

Don’t send me SMSes at antisocial times, but let me know later if anything broke and didn’t recover:

{% highlight ini %}
[nice_alerter]
type=bulksms
times_type=only
time_lower=07:30
time_upper=22:00
delay=1
{% endhighlight %}

## <a name="email"></a>Email alerters

*DO NOT COMMIT YOUR CREDENTIALS TO A PUBLIC REPO*

| setting | description | required | default |
|---|---|---|---|
|host|the email server to send the message to (via SMTP).|yes| |
|from|the email address the email should come from.|yes| |
|to|the email address to email should go to.|yes| |
|username|username to log into the SMTP server|no| |
|password|password to log into the SMTP server|no| |
|ssl|`starttls` to use StartTLS; `yes` to use SMTP_SSL (untested); otherwise no SSL is used at all|no| |

## <a name="bulksms"></a>BulkSMS alerters

*DO NOT COMMIT YOUR CREDENTIALS TO A PUBLIC REPO*

| setting | description | required | default |
|---|---|---|---|
|sender|who the SMS should appear to be from. Max 11 chars. Try to avoid non alphanumeric characters.|no|SmplMntr|
|username|your BulkSMS username.|yes| |
|password|your BulkSMS password.|yes| |
|target|the number to send the SMS to. Prefix the country code but drop the +. UK example: 447777123456.|yes| |

## <a name="syslog"></a>Syslog alerters
Syslog alerters have no additional options.

## <a name="execute"></a>Execute alerters

| setting | description | required | default |
|---|---|---|---|
|fail_command|The command to execute when a monitor fails.|no| |
|success_command|The command to execute when a monitor recovered.|no| |
|catchup_command|THe command to execute when a previously-failed but not-alerted monitor enters a time period when it can alert. See the `delay` option above.|no| |

You can use the string `fail_command` for catchup_command to make it use the value of fail_command.

The following variables will be replaced in the string when the command is executed:

* hostname: the host the monitor is running on
* name: the monitor's name
* days, hours, minutes, seconds: the monitor's downtime
* failed_at: the date and time the monitor failed at
* virtual_fail_count: the virtual fail count of the monitor
* info: the additional information the monitor recorded about its status
* description: a description of what the monitor is checking for

You may need to quote parameters - e.g. `fail_command=say "Oh no, monitor {name} has failed at {failed_at}"`

## <a name="slack"></a>Slack alerters

First, set up a webhook for this to use.

1. Go to https://slack.com/apps/manage
2. Add a new webhook
3. Configure it to taste (channel, name, icon)
4. Copy the webhook URL for your configuration below

This alerter requires the `requests` library to be installed.  You can install it with `pip install -r requirements.txt`.

*DO NOT COMMIT YOUR WEBHOOK URL TO A PUBLIC REPO*

| setting | description | required | default |
|---|---|---|---|
|url|The Slack webhook URL as configured on your account|yes| |

## <a name="ses"></a>ses alerters

You will need AWS credentials. Signing up for an configuring an AWS account is beyond the scope of this document. Credentials can come from any of the usual ways the AWS SDKs can find them, or can be specified in the configuration file.

This alerter requires the `boto3` library to be installed.

*DO NOT COMMIT YOUR AWS ACCESS KEYS TO A PUBLIC REPO*

| setting | description | required | default |
|---|---|---|---|
|from|The email address to send from|yes| |
|to|The address to send to|yes| |
|aws_access_key|The AWS access key id|no|(the SDK will look for credentials in the usual locations)|
|aws_secret_access_key|The AWS secret access key|no|(the SDK will look for credentials in the usual locations)|

## <a name="46elks"></a>46elks alerters

You will need to register for an account at [46elks](https://46elks.com).

*DO NOT COMMIT YOUR CREDENTIALS TO A PUBLIC REPO*

| setting | description | required | default |
|---|---|---|---|
|username| your 46elks username| yes | |
|password| your 46elks password| yes | |
|target| 46elks target value | yes | |
|sender| your SMS sender field; start with + if using a phone number | no | SmplMntr |
|api_host| 46elks API endpoint| no | api.46elks.com |

## <a name="pushpullet"></a>pushbullet alerters

You will need to be registered at [pushbullet](https://www.pushbullet.com).

*DO NOT COMMIT YOUR CREDENTIALS TO A PUBLIC REPO*

| setting | description | required | default |
|---|---|---|---|
|token|your pushbullet token| yes | |

## <a name="pushover"></a>pushover alerters

You will need to be registered at [pushover](https://pushover.net).

*DO NOT COMMIT YOUR CREDENTIALS TO A PUBLIC REPO*

| setting | description | required | default |
|---|---|---|---|
|user|your pushover username| yes | |
|token|your pushover token| yes | |
