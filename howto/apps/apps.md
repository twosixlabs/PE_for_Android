# How-to: develop apps for PE for Android

* [Overview](#overview)
* [Example Private Data Service Call](#example-private-data-service-call)
* [Specify the data source](#specify-the-data-source)
    - [Data type parameter helpers](#data-type-parameter-helpers)
* [Specify the purpose](#specify-the-purpose)
* [Specify the callback](#specify-the-callback)
* [Issue the request](#issue-the-request)

## Overview

Apps can take advantage of PE for Android's new Private Data Service API. This
API allows apps to receive sensitive user and device data in a least-privileges
manner. For example, rather than calling Android's `LocationManager`, which
exposes full-fidelity location data, apps can request location data from the
Private Data Service and reduce its fidelity using a uPAL (micro-PAL) module.

Apps invoke the Private Data Service by by populating a `DataRequest` object
and passing it to the `PrivateDataManager`. In order to avoid apps from hanging
or producing ANR warnings, these requests are handled asynchronously, with the
result being returned via a callback declared in the request.

This guide steps through an example code snippet that calls the Private Data
Service to request [zip-code-level location
data](https://github.com/twosixlabs/PE_for_Android_example_upals/blob/master/app/src/main/java/com/twosixlabs/exampleupals/ZipcodeMicroPAL.java).
These instructions assume that the SDK addons are [installed in your Android
Studio Environment](sdk.md) and that they're [included in your
project](sdk-project.md).

## Example Private Data Service call

```java
// Assuming this code is in a Context-derived class (e.g., Activity, Service, etc.)
Context context = this;

// Specify the data source
DataRequest.DataType dt = DataRequest.DataType.LOCATION;
Bundle locationParams = new DataRequest.LocationParamsBuilder()
                                   .setUpdateMode(DataRequest.LocationParamsBuilder.MODE_LAST_LOCATION)
                                   .setTimeoutMillis(60000)
                                   .build();

// Specify the uPAL
String pal = "com.twosixlabs.examplepals.ZipcodeMicroPAL";
Bundle palParams = null;

// Specify the purpose
DataRequest.Purpose purpose = DataRequest.Purpose.ADS("Paying the bills");

// Specify the callback
ResultReceiver callback = new ResultReceiver(null) {
    @Override
    protected void onReceiveResult(int resultCode, Bundle resultData) {
        if(resultCode == PrivateDataManager.RESULT_SUCCESS) {
            Log.d(TAG, "Received callback");
            for(String key : resultData.keySet()) {
                Log.d(TAG, String.format("Received data key=%s , value=%s", key, resultData.get(key).toString()));
            }
        }
    }
};

// Issue the request
DataRequest request = new DataRequest(context,
                                      dt, locationParams,
                                      pal, palParams,
                                      purpose, callback);
PrivateDataManager pdm = PrivateDataManager.getInstance();
pdm.requestData(request);
```

The following sections detail discuss the key parts of making a Private Data
Service Request.

## Specify the data source

```java
DataRequest.DataType dt = DataRequest.DataType.LOCATION;
Bundle locationParams = new DataRequest.LocationParamsBuilder()
                                   .setUpdateMode(DataRequest.LocationParamsBuilder.MODE_LAST_LOCATION)
                                   .setTimeoutMillis(60000)
                                   .build();
```

The first step to build a Private Data Service request is to specify what
sensitive data reosurce from which to retrieve information. In this example,
the code specifies a request for location data. Some data types require
additional parameters, detailed below.

### Data type parameter helpers

The `DataRequest` class includes builder classes to construct parameters for
the LOCATION, CALENDAR, and SMS data types. Other data types don't require
parameters, and so the params argument can be set to anything (including
`null`) in those cases.

The `LocationParamsBuilder` specifies the mode of the location request (i.e.,
either using the last known cached location, or performing an actual location
update using geolocation hardware) and how long the request can be active
before it times out:

```java
new DataRequest.LocationParamsBuilder()
           .setUpdateMode(DataRequest.LocationParamsBuilder.MODE_LAST_LOCATION)
           .setTimeoutMillis(60000)
           .build();

```

The `MessageParamsBuilder` and `CalendarParamsBuilder` define the time window
for which SMSs and calendar events, respectively, will be returned:

```java
new DataRequest.MessageParamsBuilder()
           .setStartUtcMillis(0l)
           .setEndUtcMillis(1572459040000l)
           .build();
```

## Specify the uPAL

```java
String pal = "com.twosixlabs.examplepals.ZipcodeMicroPAL";
Bundle palParams = null;
```

A Private Data Service request must specify the uPAL that will transform the
sensitive data before returning it to the calling app. The uPAL developer
is responsible for documenting the uPAL identifier, parameters (if any), and
accepted data types.

Note that that uPALs can be installed separately from the calling app. Calling
an unavailable uPAL will result in a `null` data bundle passed back to the
requesting app.

## Specify the purpose

```java
DataRequest.Purpose purpose = DataRequest.Purpose.ADS("Paying the bills");
```

PE for Android requires Private Data Service requests to specify the purpose
for the data access. [Policy managers](policy.md) can operate on declared
purposes in granting or denying access to sensitive resources.

The `DataRequest.Purpose` inner class defines available purposes. To
declare a purpose, select the desired purpose and provide a human-readable
explanation for the request.

```java
DataRequest.Purpose.ADS("Paying the bills");
DataRequest.Purpose.SOCIAL("Integration with Facebook");
```

## Specify the callback

```java
ResultReceiver callback = new ResultReceiver(null) {
    @Override
    protected void onReceiveResult(int resultCode, Bundle resultData) {
        if(resultCode == PrivateDataManager.RESULT_SUCCESS) {
            Log.d(TAG, "Received callback");
            for(String key : resultData.keySet()) {
                Log.d(TAG, String.format("Received data key=%s , value=%s", key, resultData.get(key).toString()));
            }
        }
    }
};
```

After the uPAL finishes transforming the requested data, it returns its result
via callback. Callbacks should check the result code before processing the
incoming data. In this example, the callback simply prints out all the
key-value pairs inside the incoming data bundle sent by the uPAL.

## Issue the request

```java
DataRequest request = new DataRequest(context,
                                      dt, locationParams,
                                      pal, palParams,
                                      purpose, callback);
PrivateDataManager pdm = PrivateDataManager.getInstance();
pdm.requestData(request);
```

The `PrivateDataManager` is the entry-point for the Private Data Service. Use
an instance of the `PrivateDataManager` to issue the request for uPAL-transformed
data. The callback specified in the request will run asynchronously upon the
completion of the request.

