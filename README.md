# Dynatrace OneAgent SDK for Java

This SDK allows Dynatrace customers to instrument java applications. This is useful to enhance the visibility for proprietary frameworks or
custom frameworks not directly supported by Dynatrace OneAgent out-of-the-box.

This is the official Java implementation of the [Dynatrace OneAgent SDK](https://github.com/Dynatrace/OneAgent-SDK).

## Table of Contents

* [Package contents](#package-contents)
* [Requirements](#requirements)
* [Integration](#integration)
  * [Dependencies](#dependencies)
  * [Troubleshooting](#troubleshooting)
* [API concepts](#api-concepts)
  * [OneAgentSDK object](#oneagentsdk-object)
  * [Tracers](#tracers)
* [Features](#features)
  * [Trace incoming and outgoing remote calls](#trace-incoming-and-outgoing-remote-calls)
  * [In process linking](#in-process-linking)
  * [Add custom request attributes](#add-custom-request-attributes)
  * [Custom services](#custom-services)
  * [Trace web requests](#trace-web-requests)
    * [Trace incoming web requests](#trace-incoming-web-requests)
    * [Trace outgoing web requests](#trace-outgoing-web-requests)
  * [Trace messaging](#trace-messaging)
  * [Trace SQL database requests](#trace-sql-database-requests)
* [Further reading](#further-reading)
* [Help & support](#help-&-support)
* [Release notes](#release-notes)

## Package contents

* `samples`: contains sample application, which demonstrates the usage of the SDK. See readme inside the samples directory for more details.
* `docs`: contains the reference documentation (javadoc). The most recent version is also available online at [https://dynatrace.github.io/OneAgent-SDK-for-Java/](https://dynatrace.github.io/OneAgent-SDK-for-Java/).
* `LICENSE`: license under which the whole SDK and sample applications are published.

## Requirements

* JRE 1.6 or higher
* Dynatrace OneAgent (required versions see below)

|OneAgent SDK for Java|Required OneAgent version|Support status|
|:--------------------|:------------------------|:-------------|
|1.8.0                |>=1.201                  |Supported     |
|1.7.0                |>=1.167                  |Supported     |
|1.6.0                |>=1.161                  |Supported     |
|1.4.0                |>=1.151                  |Supported     |
|1.3.0                |>=1.149                  |Supported     |
|1.2.0                |>=1.147                  |Supported     |
|1.1.0                |>=1.143                  |Supported     |
|1.0.3                |>=1.135                  |Supported     |

## Integration

### Dependencies

If you want to integrate the OneAgent SDK into your application, just add the following Maven dependency:

```xml
<dependency>
  <groupId>com.dynatrace.oneagent.sdk.java</groupId>
  <artifactId>oneagent-sdk</artifactId>
  <version>1.8.0</version>
  <scope>compile</scope>
</dependency>
```

If you prefer to integrate the SDK using plain jar file, just download them from Maven Central. You can find the download links for each
version in the [Release notes](#releasenotes) section.

The Dynatrace OneAgent SDK for Java has no further dependencies.

### Troubleshooting

If the SDK can't connect to the OneAgent (see usage of SDKState in samples) or you you don't see the desired result in the Dynatrace UI,
you can set the following system property to print debug information to standard out:

```java
-Dcom.dynatrace.oneagent.sdk.debug=true
```

Additionally you should/have to ensure, that you have set a `LoggingCallback`. For usage see class `StdErrLoggingCallback` in
`remotecall-server` module (in samples/remotecall folder).

## API concepts

Common concepts of the Dynatrace OneAgent SDK are explained in the [Dynatrace OneAgent SDK repository](https://github.com/Dynatrace/OneAgent-SDK#apiconcepts).

### OneAgentSDK object

Use OneAgentSDKFactory.createInstance() to obtain an OneAgentSDK instance. You should reuse this object throughout the whole application
and, if possible, JVM lifetime:

```Java
OneAgentSDK oneAgentSdk = OneAgentSDKFactory.createInstance();
switch (oneAgentSdk.getCurrentState()) {
case ACTIVE:
  break;
case PERMANENTLY_INACTIVE:
  break;
case TEMPORARILY_INACTIVE:
  break;
default:
  break;
}
```

It is good practice to check the SDK state regularly as it may change at any point in time (except for PERMANENTLY_INACTIVE, which never
changes throughout the JVM lifetime).

### Tracers

To trace any kind of call you first need to create a Tracer. The Tracer object represents the logical and physical endpoint that
you want to call. A Tracer serves two purposes. First, to time the call (duration, cpu and more) and report errors. That is why
each Tracer has these three methods. The error method must be called only once, and it must be in between start and end.

```Java
void start();

void error(String message);

void end();
```

The second purpose of a Tracer is to allow tracing across process boundaries. To achieve that these kind of traces supply so called
tags. Tags are strings or byte arrays that enable Dynatrace to trace a transaction end to end. As such the tag is the one information
that you need to transport across these calls as an SDK user.

A Tracer instance can only be used from the thread on which it was created. See [in process linking](#in-process-linking) for tracing across thread boundaries.

## Features

The feature sets differ slightly with each language implementation. More functionality will be added over time, see <a href="https://answers.dynatrace.com/spaces/483/dynatrace-product-ideas/idea/198106/planned-features-for-oneagent-sdk.html" target="_blank">Planned features for OneAgent SDK</a>
for details on upcoming features.

A more detailed specification of the features can be found in [Dynatrace OneAgent SDK](https://github.com/Dynatrace/OneAgent-SDK#features).

|Feature                                        |Required OneAgent SDK for Java version|
|:----------------------------------------------|:-------------------------------------|
|Custom services                                |>=1.8.0                               |
|Trace database requests                        |>=1.7.0                               |
|Trace messaging                                |>=1.6.0                               |
|Outgoing web requests                          |>=1.4.0                               |
|Incoming web requests                          |>=1.3.0                               |
|Custom request attributes                      |>=1.2.0                               |
|In process linking                             |>=1.1.0                               |
|Trace incoming and outgoing remote calls       |>=1.0.3                               |

### Trace incoming and outgoing remote calls

You can use the SDK to trace proprietary IPC communication from one process to the other. This will enable you to see full Service Flow, PurePath
and Smartscape topology for remoting technologies that Dynatrace is not aware of.

To trace any kind of remote call you first need to create a Tracer. The Tracer object represents the endpoint that you want to call, as such you
need to supply the name of the remote service and remote method. In addition you need to transport the tag in your remote call to the server side
if you want to trace it end to end.

```Java
OneAgentSDK oneAgentSdk = OneAgentSDKFactory.createInstance();
OutgoingRemoteCallTracer outgoingRemoteCall = oneAgentSdk.traceOutgoingRemoteCall("remoteMethodToCall", "RemoteServiceName", "rmi://Endpoint/service", ChannelType.TCP_IP, "remoteHost:1234");
outgoingRemoteCall.setProtocolName("RMI/custom");
outgoingRemoteCall.start();
try {
	String tag = outgoingRemoteCall.getDynatraceStringTag();
	// make the call and transport the tag across to server
} catch (Throwable e) {
	outgoingRemoteCall.error(e);
	// rethrow or add your exception handling
} finally {
	outgoingRemoteCall.end();
}
```

On the server side you need to wrap the handling and processing of your remote call as well. This will not only trace the server side call and
everything that happens, it will also connect it to the calling side.

```Java
OneAgentSDK oneAgentSdk = OneAgentSDKFactory.createInstance();
IncomingRemoteCallTracer incomingRemoteCall = oneAgentSdk.traceIncomingRemoteCall("remoteMethodToCall", "RemoteServiceName", "rmi://Endpoint/service");
incomingRemoteCall.setDynatraceStringTag(tag);
incomingRemoteCall.start();
try {
	incomingRemoteCall.setProtocolName("RMI/custom");
	doSomeWork(); // process the remoteCall
} catch (Exception e) {
	incomingRemoteCall.error(e);
	// rethrow or add your exception handling
} finally{
	incomingRemoteCall.end();
}
```

### In process linking

You can use the SDK to link inside a single process. To link for eg. an asynchronous execution, you need the following code:

```Java
OneAgentSDK oneAgentSdk = OneAgentSDKFactory.createInstance();
InProcessLink inProcessLink = oneAgentSdk.createInProcessLink();
```

Provide the returned ``inProcessLink`` to the code, that does the asynchronous execution:

```Java
OneAgentSDK oneAgentSdk = OneAgentSDKFactory.createInstance();
InProcessLinkTracer inProcessLinkTracer = oneAgentSdk.traceInProcessLink(inProcessLink);
inProcessLinkTracer.start();
try {
	// do the work ...
} catch (Exception e) {
	inProcessLinkTracer.error(e);
	// rethrow or add your exception handling
} finally {
	inProcessLinkTracer.end();
}
```

### Add custom request attributes

You can use the SDK to add custom request attributes to the current traced service. Custom request attributes allow you to do advanced filtering of
your requests in Dynatrace.

Adding custom request attributes to the currently traced service call is simple. Just call one of the addCustomRequestAttribute methods with your key and value:

```Java
oneAgentSDK.addCustomRequestAttribute("region", "EMEA");
oneAgentSDK.addCustomRequestAttribute("salesAmount", 2500);
```

When no service call is being traced, the custom request attributes are dropped.

### Custom services

You can use the SDK to trace custom service methods. A custom service method is a meaningful
part of your code that you want to trace but that does not fit any other tracer.
An example could be the callback of a periodic timer.

```Java
String serviceMethod = "onTimer";
String serviceName = "PeriodicCleanupTask";
CustomServiceTracer tracer = oneAgentSDK.traceCustomService(serviceMethod, serviceName);
tracer.start();
try {
	doMyCleanup();
} catch (Exception e) {
	tracer.error(e.getMessage());
	throw e;
} finally {
	tracer.end();
}
```

### Trace web requests

#### Trace incoming web requests

You can use the SDK to trace incoming web requests. This might be useful if Dynatrace does not support the respective web server framework or language
processing the incoming web requests.

To trace an incoming web request you first need to create a WebApplicationInfo object. The info object represents the endpoint of your web server (web server name, application name and context root). This object should be reused for all traced web requests within for the same application.

```Java
WebApplicationInfo wsInfo = oneAgentSdk.createWebApplicationInfo("WebShopProduction", "CheckoutService", "/api/service/checkout");
```

To trace a specific incoming web request you then need to create a Tracer object. Make sure you provide all http headers from the request to the SDK by
calling addRequestHeader(...). This ensures that tagging with our built-in sensor will work.

```Java
IncomingWebRequestTracer tracer = oneAgentSdk.traceIncomingWebRequest(wsInfo,"https://www.oursupershop.com/api/service/checkout/save", "POST")

for (Entry<String, String> headerField : httpRequest.getHeaders().entrySet()) {
	incomingWebrequestTracer.addRequestHeader(headerField.getKey(), headerField.getValue());
}

for (Entry<String, List<String>> parameterEntry : httpRequest.getParameters().entrySet()) {
	for (String value : parameterEntry.getValue()) {
		incomingWebrequestTracer.addParameter(parameterEntry.getKey(), value);
	}
}

incomingWebrequestTracer.setRemoteAddress(httpRequest.getRemoteHostName());

tracer.start();
try {
	int statusCodeReturnedToClient = processWebRequest();
	tracer.setStatusCode(statusCodeReturnedToClient);
} catch (Exception e) {
	tracer.setStatusCode(500); // we expect that the container sends HTTP 500 status code in case request processing throws an exception
	tracer.error(e);
	throw e;
} finally {
	tracer.end();
}
```

#### Trace outgoing web requests

You can use the SDK to trace outgoing web requests. This might be useful if Dynatrace does not support the respective http library or
language sending the request.

To trace an outgoing web request you need to create a Tracer object. It is important to send the Dynatrace Header. This ensures that
tagging with our built-in sensor will work.

```Java
OutgoingWebRequestTracer outgoingWebRequestTracer = oneAgentSdk.traceOutgoingWebRequest(url, "GET");

// provide all request headers to outgoingWebRequestTracer (optional):
for (Entry<String, String> entry : yourHttpClient.getRequestHeaders().entrySet()) {
	outgoingWebRequestTracer.addRequestHeader(entry.getKey(), entry.getValue());
}

outgoingWebRequestTracer.start();
try {
	yourHttpClient.setUrl(url);

	// sending HTTP header OneAgentSDK.DYNATRACE_HTTP_HEADERNAME is necessary for tagging:
	yourHttpClient.addRequestHeader(OneAgentSDK.DYNATRACE_HTTP_HEADERNAME, outgoingWebRequestTracer.getDynatraceStringTag());

	yourHttpClient.processHttpRequest();

	for (Entry<String, List<String>> entry : yourHttpClient.getHeaderFields().entrySet()) {
		for (String value : entry.getValue()) {
			outgoingWebRequestTracer.addResponseHeader(entry.getKey(), value);
		}
	}
	outgoingWebRequestTracer.setStatusCode(yourHttpClient.getResponseCode());

} catch (Exception e) {
	outgoingWebRequestTracer.error(e);
	// rethrow or add your exception handling
} finally {
	outgoingWebRequestTracer.end();
}
```

### Trace messaging

You can use the SDK to trace messages sent or received via messaging & queuing systems. When tracing messages, we distinguish between:

* sending a message
* receiving a message
* processing a received message

To trace an outgoing message, you simply need to create a MessagingSystemInfo and call traceOutgoingMessage with that instance:

```Java
MessagingSystemInfo messagingSystemInfo = oneAgentSDK.createMessagingSystemInfo("myMessagingSystem",
		"requestQueue", MessageDestinationType.QUEUE, ChannelType.TCP_IP, "localhost:4711");
OutgoingMessageTracer outgoingMessageTracer = oneAgentSDK.traceOutgoingMessage(messagingSystemInfo);
outgoingMessageTracer.start();
try {
	// transport the dynatrace tag along with the message: 	
	messageToSend.setHeaderField(
		OneAgentSDK.DYNATRACE_MESSAGE_PROPERTYNAME, outgoingMessageTracer.getDynatraceStringTag());
	// optional:  add application provided correlationId
	outgoingMessageTracer.setCorrelationId(toSend.correlationId);

	theQueue.send(messageToSend);

	// optional:  add messageid provided from messaging system
	outgoingMessageTracer.setVendorMessageId(toSend.getMessageId());
} catch (Exception e) {
	outgoingMessageTracer.error(e.getMessage());
	// rethrow or add your exception handling
} finally {
	outgoingMessageTracer.end();
}
```

On the incoming side, we need to differentiate between the blocking receiving part and processing the received message. Therefore two
different tracers are being used: `IncomingMessageReceiveTracer` and `IncomingMessageProcessTracer`.

```Java
MessagingSystemInfo messagingSystemInfo = oneAgentSDK.createMessagingSystemInfo("myMessagingSystem",
		"requestQueue", MessageDestinationType.QUEUE, ChannelType.TCP_IP, "localhost:4711");

// message receiving daemon task:
while(true) {
	IncomingMessageReceiveTracer incomingMessageReceiveTracer =
		oneAgentSDK.traceIncomingMessageReceive(messagingSystemInfo);
	incomingMessageReceiveTracer.start();
	try {
		// blocking call - until message is being available:
		Message queryMessage = theQueue.receive("client queries");
		IncomingMessageProcessTracer incomingMessageProcessTracer = oneAgentSDK
			.traceIncomingMessageProcess(messagingSystemInfo);
		incomingMessageProcessTracer.setDynatraceStringTag(
			queryMessage.getHeaderField(OneAgentSDK.DYNATRACE_MESSAGE_PROPERTYNAME));
		incomingMessageProcessTracer.setVendorMessageId(queryMessage.msgId);
		incomingMessageProcessTracer.setCorrelationId(queryMessage.correlationId);
		incomingMessageProcessTracer.start();
		try {
			// do the work ...
		} catch (Exception e) {
			incomingMessageProcessTracer.error(e.getMessage());
			Logger.logError(e);
		} finally {
			incomingMessageProcessTracer.end();
		}
	} catch (Exception e) {
		incomingMessageReceiveTracer.error(e.getMessage());
		// rethrow or add your exception handling
	} finally {
		incomingMessageReceiveTracer.end();
	}
}
```

In case of non-blocking receive (e. g. via event handler), there is no need to use `IncomingMessageReceiveTracer` - just trace processing
of the message by using the `IncomingMessageProcessTracer`:

```Java
MessagingSystemInfo messagingSystemInfo = oneAgentSDK.createMessagingSystemInfo("myMessagingSystem",
	"requestQueue", MessageDestinationType.QUEUE, ChannelType.TCP_IP, "localhost:4711");

public void onMessage(Message message) {
	IncomingMessageProcessTracer incomingMessageProcessTracer = oneAgentSDK
		.traceIncomingMessageProcess(messagingSystemInfo);
	incomingMessageProcessTracer.setDynatraceStringTag((String)
		message.getObjectProperty(OneAgentSDK.DYNATRACE_MESSAGE_PROPERTYNAME));
	incomingMessageProcessTracer.setVendorMessageId(queryMessage.msgId);
	incomingMessageProcessTracer.setCorrelationId(queryMessage.correlationId);
	incomingMessageProcessTracer.start();
	try {
		// do the work ...
	} catch (Exception e) {
		incomingMessageProcessTracer.error(e.getMessage());
		// rethrow or add your exception handling
	} finally {
		incomingMessageProcessTracer.end();
	}
}
```

> Please also see the [documentation on messaging tracers in the specification repository](https://github.com/Dynatrace/OneAgent-SDK#messaging).

### Trace SQL database requests

A SQL database request is traced by calling `traceSqlDatabaseRequest`. For details about usage see the [OneAgentSDK specification](https://github.com/Dynatrace/OneAgent-SDK#database)

```java
String sql = "SELECT * FROM transformationdata WHERE transformation_id = " + id;

DatabaseInfo databaseInfo = oneAgentSdk.createDatabaseInfo("TransformationDb", DatabaseVendor.FIREBIRD.getVendorName(), ChannelType.TCP_IP, "db-serv01.acme.com:2323");

DatabaseRequestTracer databaseTracer = oneAgentSdk.traceSqlDatabaseRequest(databaseInfo, sql);
databaseTracer.start();
try {
	Result result = executeTheDatabaseCall(sql);
	databaseTracer.setReturnedRowCount(result.getRows().getLength());
} catch (InterruptedException e) {
	databaseTracer.error(e);
    // handle or rethrow
} finally {
	databaseTracer.end();
}
```

Please note that SQL database traces are only created if they occur within some other SDK trace (e.g. incoming remote call)
or a OneAgent built-in trace (e.g. incoming web request).

## Further reading

* <a href="https://www.dynatrace.com/support/help/extend-dynatrace/oneagent-sdk/what-is-oneagent-sdk/" target="_blank">What is the OneAgent SDK?</a> in the Dynatrace documentation
* <a href="https://answers.dynatrace.com/spaces/483/dynatrace-product-ideas/idea/198106/planned-features-for-oneagent-sdk.html" target="_blank">Feedback & Roadmap thread in AnswerHub</a>
* <a href="https://www.dynatrace.com/news/blog/dynatrace-oneagent-sdk-for-java-end-to-end-monitoring-for-proprietary-java-frameworks/" target="_blank">Blog: Dynatrace OneAgent SDK for Java: End-to-end monitoring for proprietary Java frameworks</a>

## Help & support

### Support policy

The Dynatrace OneAgent SDK for Java has GA status. The features are fully supported by Dynatrace.

For detailed support policy see [Dynatrace OneAgent SDK help](https://github.com/Dynatrace/OneAgent-SDK#help).

### Get help

* Ask a question in the <a href="https://answers.dynatrace.com/spaces/482/view.html" target="_blank">product forums</a>
* Read the <a href="https://www.dynatrace.com/support/help/" target="_blank">product documentation</a>

**Open a <a href="https://github.com/Dynatrace/OneAgent-SDK-for-Java/issues">GitHub issue</a> to:**

* Report minor defects, minor items or typos
* Ask for improvements or changes in the SDK API
* Ask any questions related to the community effort

SLAs don't apply for GitHub tickets

**Customers can open a ticket on the <a href="https://support.dynatrace.com/supportportal/" target="_blank">Dynatrace support portal</a> to:**

* Get support from the Dynatrace technical support engineering team
* Manage and resolve product related technical issues

SLAs apply according to the customer's support level.

## Release notes

see also <https://github.com/Dynatrace/OneAgent-SDK-for-Java/releases>

|Version|Description                                 |Links                                    |
|:------|:-------------------------------------------|:----------------------------------------|
|1.8.0  |Added support for custom services           |[binary](https://search.maven.org/remotecontent?filepath=com/dynatrace/oneagent/sdk/java/oneagent-sdk/1.8.0/oneagent-sdk-1.8.0.jar) [source](https://search.maven.org/remotecontent?filepath=com/dynatrace/oneagent/sdk/java/oneagent-sdk/1.8.0/oneagent-sdk-1.8.0-sources.jar) [javadoc](https://search.maven.org/remotecontent?filepath=com/dynatrace/oneagent/sdk/java/oneagent-sdk/1.8.0/oneagent-sdk-1.8.0-javadoc.jar)|
|1.7.0  |Added support for database requests         |[binary](https://search.maven.org/remotecontent?filepath=com/dynatrace/oneagent/sdk/java/oneagent-sdk/1.7.0/oneagent-sdk-1.7.0.jar) [source](https://search.maven.org/remotecontent?filepath=com/dynatrace/oneagent/sdk/java/oneagent-sdk/1.7.0/oneagent-sdk-1.7.0-sources.jar) [javadoc](https://search.maven.org/remotecontent?filepath=com/dynatrace/oneagent/sdk/java/oneagent-sdk/1.7.0/oneagent-sdk-1.7.0-javadoc.jar)|
|1.6.0  |Added support for messaging                 |[binary](https://search.maven.org/remotecontent?filepath=com/dynatrace/oneagent/sdk/java/oneagent-sdk/1.6.0/oneagent-sdk-1.6.0.jar) [source](https://search.maven.org/remotecontent?filepath=com/dynatrace/oneagent/sdk/java/oneagent-sdk/1.6.0/oneagent-sdk-1.6.0-sources.jar) [javadoc](https://search.maven.org/remotecontent?filepath=com/dynatrace/oneagent/sdk/java/oneagent-sdk/1.6.0/oneagent-sdk-1.6.0-javadoc.jar)|
|1.4.0  |Added support for outgoing webrequests      |[binary](https://search.maven.org/remotecontent?filepath=com/dynatrace/oneagent/sdk/java/oneagent-sdk/1.4.0/oneagent-sdk-1.4.0.jar) [source](https://search.maven.org/remotecontent?filepath=com/dynatrace/oneagent/sdk/java/oneagent-sdk/1.4.0/oneagent-sdk-1.4.0-sources.jar) [javadoc](https://search.maven.org/remotecontent?filepath=com/dynatrace/oneagent/sdk/java/oneagent-sdk/1.4.0/oneagent-sdk-1.4.0-javadoc.jar)|
|1.3.0  |Added support for incoming webrequests      |[binary](https://search.maven.org/remotecontent?filepath=com/dynatrace/oneagent/sdk/java/oneagent-sdk/1.3.0/oneagent-sdk-1.3.0.jar) [source](https://search.maven.org/remotecontent?filepath=com/dynatrace/oneagent/sdk/java/oneagent-sdk/1.3.0/oneagent-sdk-1.3.0-sources.jar) [javadoc](https://search.maven.org/remotecontent?filepath=com/dynatrace/oneagent/sdk/java/oneagent-sdk/1.3.0/oneagent-sdk-1.3.0-javadoc.jar)|
|1.2.0  |Added support for custom request attributes |[binary](https://search.maven.org/remotecontent?filepath=com/dynatrace/oneagent/sdk/java/oneagent-sdk/1.2.0/oneagent-sdk-1.2.0.jar) [source](https://search.maven.org/remotecontent?filepath=com/dynatrace/oneagent/sdk/java/oneagent-sdk/1.2.0/oneagent-sdk-1.2.0-sources.jar) [javadoc](https://search.maven.org/remotecontent?filepath=com/dynatrace/oneagent/sdk/java/oneagent-sdk/1.2.0/oneagent-sdk-1.2.0-javadoc.jar)|
|1.1.0  |Added support for in-process-linking        |[binary](https://search.maven.org/remotecontent?filepath=com/dynatrace/oneagent/sdk/java/oneagent-sdk/1.1.0/oneagent-sdk-1.1.0.jar) [source](https://search.maven.org/remotecontent?filepath=com/dynatrace/oneagent/sdk/java/oneagent-sdk/1.1.0/oneagent-sdk-1.1.0-sources.jar) [javadoc](https://search.maven.org/remotecontent?filepath=com/dynatrace/oneagent/sdk/java/oneagent-sdk/1.1.0/oneagent-sdk-1.1.0-javadoc.jar)|
|1.0.3  |Initial release                             |[binary](https://search.maven.org/remotecontent?filepath=com/dynatrace/oneagent/sdk/java/oneagent-sdk/1.0.3/oneagent-sdk-1.0.3.jar) [source](https://search.maven.org/remotecontent?filepath=com/dynatrace/oneagent/sdk/java/oneagent-sdk/1.0.3/oneagent-sdk-1.0.3-sources.jar) [javadoc](https://search.maven.org/remotecontent?filepath=com/dynatrace/oneagent/sdk/java/oneagent-sdk/1.0.3/oneagent-sdk-1.0.3-javadoc.jar)|
