# Microsoft Azure Log Analytics Cloud Foundry Firehose Nozzle

## Summary

Microsoft [Operations Management Suite](https://docs.microsoft.com/en-us/azure/operations-management-suite/) (OMS) is Microsoft's cloud-based IT management solution that helps you manage and protect your on-premises and cloud infrastructure.

Azure [Log Analytics](https://azure.microsoft.com/en-us/services/log-analytics/) is a service in OMS that helps you collect and analyze data generated by resources in your cloud and on-premises environments. In the following document, it will be referred to as `OMS Log Analytics`.

The Microsoft Azure Log Analytics Nozzle is a Cloud Foundry (CF) component which forwards metrics from the [Loggregator](https://docs.cloudfoundry.org/loggregator/architecture.html) Firehose to OMS Log Analytics. In the following document, it will be referred to as `Log Analytics Nozzle` or `nozzle` for short.

## Prerequisites

### 1. Deploy a CF or PCF environment in Azure

* [Deploy Cloud Foundry on Azure](https://github.com/cloudfoundry-incubator/bosh-azure-cpi-release/blob/master/docs/guidance.md)
* [Deploy Pivotal Cloud Foundry on Azure](https://docs.pivotal.io/pivotalcf/1-9/customizing/azure.html)

### 2. Install CLIs on your dev box

* [Install Cloud Foundry CLI](https://github.com/cloudfoundry/cli#downloads)
* [Install Cloud Foundry UAA Command Line Client](https://github.com/cloudfoundry/cf-uaac/blob/master/README.md)

### 3. Create an OMS Workspace in Azure

* [Get started with Log Analytics](https://docs.microsoft.com/en-us/azure/log-analytics/log-analytics-get-started)
* [OMS Cloud Foundry Monitoring Solution](https://azuremarketplace.microsoft.com/en/marketplace/apps/Microsoft.OMSCloudfoundrySolution)

## Deploy - Push the Nozzle as an App to Cloud Foundry

### 1. Utilize the CF CLI to authenticate with your CF instance

```
cf login -a https://api.${ENDPOINT} -u ${CF_USER} --skip-ssl-validation
```

### 2. Create a CF user and grant required privileges

The Log Analytics Nozzle requires a CF user who is authorized to access the loggregator firehose.

```
uaac target https://uaa.${ENDPOINT} --skip-ssl-validation
uaac token client get admin
cf create-user ${FIREHOSE_USER} ${FIREHOSE_USER_PASSWORD}
uaac member add cloud_controller.admin ${FIREHOSE_USER}
uaac member add doppler.firehose ${FIREHOSE_USER}
```

### 3. Download the latest code

```
git clone https://github.com/Azure/oms-log-analytics-firehose-nozzle.git
cd oms-log-analytics-firehose-nozzle
```

### 4. Set environment variables in [manifest.yml](./manifest.yml)

```
OMS_WORKSPACE             : OMS workspace ID
OMS_KEY                   : OMS key
OMS_POST_TIMEOUT          : HTTP post timeout for sending events to OMS Log Analytics
OMS_BATCH_TIME            : Interval for posting a batch to OMS Log Analytics
OMS_MAX_MSG_NUM_PER_BATCH : The max number of messages in a batch to OMS Log Analytics
API_ADDR                  : The API address of the CF environment. If set empty or absent, nozzle will use API address for current CF environment
DOPPLER_ADDR              : Loggregator's traffic controller URL. If set empty or absent, nozzle will generate it from API address
FIREHOSE_USER             : CF user who has admin and firehose access
FIREHOSE_USER_PASSWORD    : Password of the CF user
EVENT_FILTER              : Event types to be filtered out. The format is a comma separated list, valid event types are METRIC,LOG,HTTP
SPACE_WHITELIST           : Comma separated space white list, logs from apps within these spaces will be forwarded. If not set, all apps will be monitored. Format should be ORG_NAME.SPACE_NAME or ORG_NAME.*
SKIP_SSL_VALIDATION       : If true, allows insecure connections to the UAA and the Trafficcontroller
CF_ENVIRONMENT            : Set to any string value for identifying logs and metrics from different CF environments
IDLE_TIMEOUT              : Keep Alive duration for the firehose consumer
LOG_LEVEL                 : Logging level of the nozzle, valid levels: DEBUG, INFO, ERROR
LOG_EVENT_COUNT           : If true, the total count of events that the nozzle has received and sent will be logged to OMS Log Analytics as CounterEvents
LOG_EVENT_COUNT_INTERVAL  : The time interval of logging event count to OMS Log Analytics
```

### 5. Push the app

```
cf push
```

## Additional logging

For the most part, the Log Analytics Nozzle forwards metrics from the Loggregator Firehose to OMS Log Analytics without too much processing. In a few cases the nozzle might push some additional metrics to OMS Log Analytics.

### 1. eventsReceived, eventsSent and eventsLost

If `LOG_EVENT_COUNT` is set to true, the nozzle will periodically send to OMS Log Analytics the count of received events, sent events and lost events, at intervals of `LOG_EVENT_COUNT_INTERVAL`.

The statistic count is sent as a CounterEvent, with CounterKey of one of **`nozzle.stats.eventsReceived`**, **`nozzle.stats.eventsSent`** and **`nozzle.stats.eventsLost`**. Each CounterEvent contains the value of delta count during the interval, and the total count from the beginning. **`eventsReceived`** counts all the events that the nozzle received from firehose, **`eventsSent`** counts all the events that the nozzle sent to OMS Log Analytics successfully, **`eventsLost`** counts all the events that the nozzle tried to send but failed after 4 attempts.

These CounterEvents themselves are not counted in the received, sent or lost count.

In normal cases, the total count of eventsSent plus eventsLost is less than total eventsReceived at the same time, as the nozzle buffers some messages and then post them in a batch to OMS Log Analytics. Operator can adjust the buffer size by changing the configurations `OMS_BATCH_TIME` and `OMS_MAX_MSG_NUM_PER_BATCH`.

### 2. slowConsumerAlert

When the nozzle receives slow consumer alert from loggregator in three ways:

1. the nozzle receives a WebSocket close error with error code `ClosePolicyViolation (1008)`
1. the nozzle receives a CounterEvent with the name `TruncatingBuffer.DroppedMessages`
1. the nozzle receives a CounterEvent with the name `doppler_proxy.slow_consumer`

the nozzle will send a slowConsumerAlert as a ValueMetric to OMS Log Analytics, with MetricKey **`nozzle.alert.slowConsumerAlert`** and value **`1`**.

This ValueMetric is not counted in the above statistic received, sent or lost count.

## Scaling guidance

### 1. Scaling Nozzle

Operators should run at least two instances of the nozzle to reduce message loss. The Firehose will evenly distribute events across all instances of the nozzle.

When the nozzle couldn't keep up with processing the logs from firehose, Loggregator alerts the nozzle and then the nozzle logs slowConsumerAlert message to OMS Log Analytics. Operator can [create Alert rule](#alert) for this slowConsumerAlert message in OMS Log Analytics, and when the alert is triggered, the operator should scale up the nozzle to minimize the loss of data.

We did some workload test against the nozzle and got a few data for operaters' reference:

* In our test, the size of each log and metric sent to OMS Log Analytics is around 550 bytes, suggest each nozzle instance should handle no more than **300000** such messages per minute. Under such workload, the CPU usage of each instance is around 40%, and the memory usage of each instance is around 80M.

### 2. Scaling Loggregator

Loggregator emits **LGR** log message to indicate problems with the logging process. When operaters see this message in OMS Log Analytics, they might need to [scale Loggregator](https://docs.cloudfoundry.org/running/managing-cf/logging-config.html#scaling).

## View in OMS Portal

### 1. Import OMS Default Solution from Azure Marketplace

With our [OMS Cloud Foundry Monitoring Solution](https://azuremarketplace.microsoft.com/en/marketplace/apps/Microsoft.OMSCloudfoundrySolution), you can import our 20+ default views, 130+ alerts and 90+ saved searches covering all KPI of Cloud Foundry you might care about, with only simple configurations.

Please check [here](https://github.com/Azure/azure-quickstart-templates/tree/master/oms-cloudfoundry-solution) for detailed document and underlying templates.

_You can find a complete list of KPI [here](https://docs.pivotal.io/pivotalcf/2-1/monitoring/kpi.html)._

### 2. Customizing your OMS Workplace

#### 2.1. Import Individual OMS View

From the main OMS Overview page, go to **View Designer** -> **Import** -> **Browse**, select one of the [omsview](./docs/omsview) files, e.g. [Cloud Foundry.omsview](./docs/omsview/Cloud%20Foundry.omsview), and save the view. Now a **Tile** will be displayed on the main OMS Overview page. Click the **Tile**, it shows visualized metrics.

Operators could customize these views or create new views through **View Designer**.

To create your own view, please refer to the document [here](https://github.com/Azure/azure-quickstart-templates/tree/master/oms-cloudfoundry-solution#customization-and-upgrade).

> Please feel free to send your suggestions and feedback for the full view by creating Github issues.

#### 3.1. <a name="alert">Create Alert Rules Manually</a>

This section describes some sample alert rules that operators may want to create for identifying important information in their Cloud Foundry deployments.

For the process of creating alert rules in OMS Log Analytics, please refer to this [article](https://docs.microsoft.com/en-us/azure/log-analytics/log-analytics-alerts).

Operators could customize the queries and threshold values as needed.

| Search query                                                                  | Generate alert based on | Description                                                                       |
| ----------------------------------------------------------------------------- | ----------------------- | --------------------------------------------------------------------------------- |
| Type=CF_ValueMetric_CL Origin_s=bbs Name_s="Domain.cf-apps"                   | Number of results < 1   | **bbs.Domain.cf-apps** indicates if the cf-apps Domain is up-to-date, meaning that CF App requests from Cloud Controller are synchronized to bbs.LRPsDesired (Diego-desired AIs) for execution. No data received means cf-apps Domain is not up-to-date in the given time window. |
| Type=CF_ValueMetric_CL Origin_s=rep Name_s=UnhealthyCell Value_d>1            | Number of results > 0   | For Diego cells, 0 means healthy, and 1 means unhealthy. Set the alert if multiple **unhealthy Diego cells** are detected in the given time window. |
| Type=CF_ValueMetric_CL Origin_s="bosh-hm-forwarder" Name_s="system.healthy" Value_d=0 | Number of results > 0 | 1 means the system is healthy, and 0 means the system is not healthy. |
| Type=CF_ValueMetric_CL Origin_s=route_emitter Name_s=ConsulDownMode Value_d>0 | Number of results > 0   | Consul emits its health status periodically. 0 means the system is healthy, and 1 means that route emitter detects that **Consul is down**. |
| Type=CF_CounterEvent_CL Origin_s=DopplerServer (Name_s="TruncatingBuffer.DroppedMessages" or Name_s="doppler.shedEnvelopes") Delta_d>0 | Number of results > 0 | The delta number of messages intentionally **dropped** by Doppler due to back pressure. |
| Type=CF_LogMessage_CL SourceType_s=LGR MessageType_s=ERR                      | Number of results > 0   | Loggregator emits **LGR** to indicate problems with the logging process, e.g. when log message output is too high. |
| Type=CF_ValueMetric_CL Name_s=slowConsumerAlert                               | Number of results > 0   | When the nozzle receives slow consumer alert from Loggregator, it sends **slowConsumerAlert** ValueMetric to OMS. |
| Type=CF_CounterEvent_CL Job_s=nozzle Name_s=eventsLost Delta_d>0              | Number of results > 0   | If the delta number of **lost events** reaches a threshold, it means the nozzle might have some problem running. |

## Access OMS Everywhere

OMS also provides a mobile app for users to view OMS views, receiving alerts and searching for logs from your mobile devices.

Simply download App from your app store and login with your account, you can have experience just the same as on your workplace everywhere.

OMS Apps now available on [Windows (Mobile devices)](https://www.microsoft.com/en-us/store/p/microsoft-operations-management-suite/9wzdncrfjz2r), [Android](https://play.google.com/store/apps/details?id=com.microsoft.operations.AndroidPhone) and [iOS](https://itunes.apple.com/us/app/microsoft-operations-management-suite/id1042424859) devices.

## Test

You need [ginkgo](https://github.com/onsi/ginkgo) to run the test. Run the following command to execute test:

```
ginkgo -r
```

## Additional Reference

To collect syslogs and performance metrics of VMs in CloudFoundry deployment, a system metric provider is required.

For PCF 2.0+, ['Bosh System Metrics Forwarder`](https://github.com/cloudfoundry/bosh-system-metrics-forwarder-release) has been included in your deployment, VM metrics will be forwarded to loggregator automatically.

At the same time, you can also install [`BOSH Health Metric Forwarder`](https://github.com/cloudfoundry/bosh-hm-forwarder-release) manually if you're using older version. Please refer to its document for instructions. _Be advanced, we have noticed that it might occasionally conflict with `Azure OMS Agent`._

For Syslog, we recommend you to use [`Azure OMS Agent Bosh release`](https://github.com/Azure/oms-agent-for-linux-boshrelease). We have planed to provide other approaches for Syslog in the future.
