---
layout: post
title: "Connect device to Cloud using Windows Azure Service Bus"
description: "Connect device to Cloud using Windows Azure Service Bus"
category: azure, iot
tags: [azure, iot]
---
{% include JB/setup %}

# Account Setup

Sign In at https://manage.windowsazure.com


# References

* [The Windows Azure Service Bus and the Internet of Things](https://msdn.microsoft.com/en-us/magazine/dn574801.aspx)
* [Get started with Event Hubs](http://azure.microsoft.com/en-us/documentation/articles/service-bus-event-hubs-c-ephcs-getstarted/)
* [Make IoT real with the Internet of Your Things](https://www.microsoft.com/en-us/server-cloud/internet-of-things.aspx)
* [Azure SDK for Python](http://azure-sdk-for-python.readthedocs.org/en/latest/servicebus.html)
* [Get started using Azure Stream Analytics](http://azure.microsoft.com/en-us/documentation/articles/stream-analytics-get-started/)
* [Azure Python SDK](http://azure.microsoft.com/en-us/develop/python/)
* [Service Bus Python](http://azure.microsoft.com/en-us/documentation/articles/service-bus-python-how-to-use-queues/)
* [Python Event Hub](https://github.com/Waltervondehans/Python_AzureEventHub)
* [Service Bus AMQP: Developer's Guide](https://msdn.microsoft.com/en-us/library/azure/jj841071.aspx)
* [Node Service Bus](https://github.com/jmspring/node-sbus)
* [Node Talking eventHub](http://www.mikelanzetta.com/2015/01/node-js-and-net-talking-eventhub-together/)



* [Get started with Event Hubs](http://azure.microsoft.com/en-us/documentation/articles/service-bus-event-hubs-c-ephcs-getstarted/)
* [Get started using Azure Stream Analytics](http://azure.microsoft.com/en-us/documentation/articles/stream-analytics-get-started/)

SELECT
	PRODUCTKEY as PRODUCT_KEY,System.Timestamp as WindowEnd, DID,min(cast(HUMIDITY as bigint)) as MIN_HUMIDITY, count(*) as CALL_COUNT
FROM
	SensorStream TIMESTAMP BY TIMESTAMP WHERE len(SWITCH)=3
GROUP BY PRODUCTKEY,TUMBLINGWINDOW(s, 5), DID having MIN_HUMIDITY < 55


MSSQL: 

server name: dkcche74kb
database: alerts
username: gizwits
password: Humidity$1001
table: humidity_alert

CREATE TABLE humidity_alert(PRODUCT_KEY varchar(32), WINDOWEND datetime, DID varchar(32), MIN_HUMIDITY int, CALL_COUNT int, INDEX idx_did_end(PRODUCT_KEY,WINDOWEND,DID));

Azure

https://manage.windowsazure.com/@bhextremeprog.onmicrosoft.com
account: bhe@xtremeprog.com
password: @Gizwits2015

ODBC: https://technet.microsoft.com/en-us/library/hh568454(v=sql.110).aspx
https://www.microsoft.com/en-us/download/confirmation.aspx?id=36437

