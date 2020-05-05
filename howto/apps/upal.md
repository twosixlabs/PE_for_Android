# How-to: develop uPAL modules

* [Overview](#overview)
* [Example uPAL](#example-upal)
* [Extending the base class](#extending-the-base-class)
    + [Declaring the constructor](#declaring-the-constructor)
    + [Implementing the abstract methods](#implementing-the-abstract-methods)
* [Registering the uPAL](#registering-the-upal)
* [Recommended practices](#recommended-practices)

## Overview

PE for Android allows apps to access sensitive data via the [Private Data
Service](apps.md), in lieu of standard Android APIs like `LocationManager` and
`ContactsContract`. Full-resolution data (e.g., GPS coordinates, whole contact
lists) stays entirely within the Private Data Service, which uses uPAL modules
("micro-Privacy Abstraction Layer modules") to reduce the resolution of that
information down to just what the app needs.

Developers can implement their own uPAL modules and distribute them as
modular transformations, reusable in any app that interfaces with the Private
Data Service.

As an example demonstrating the development of uPALs, this guide steps through a uPAL
that receives contact list data from the system and a known phone number from the
calling app. The uPAL then returns the name associated with that phone number, if
available from the contact list. The requesting app only receives this result,
without gaining access to the entire contact list.

## Example uPAL

```java
public class NumberToNamePAL extends MicroPALProviderService<ListItem<ContactItem>> {
    private static final String TAG = NumberToNamePAL.class.getSimpleName();
    private static final String DESCRIPTION = "Get the matching name from the contact list given a known phone number";

    public static final String PHONE_KEY = "phone";
    public static final String NAME_KEY = "name";

    public NumberToNamePAL() {
        super(DataRequest.DataType.CONTACTS);
    }

    @Override
    public Bundle onReceive(ListItem<ContactItem> contactItemList, Bundle paramsFromApp) {
        if(paramsFromApp.containsKey(PHONE_KEY)) {
            String inputPhoneNumber = normalizePhoneNumber(paramsFromApp.getString(PHONE_KEY));
            Log.i(TAG, "Checking for phone number " + inputPhoneNumber);

            for(ContactItem contact : contactItemList.getStoredItems()) {
                // Contacts could have more than one phone number
                ArrayList<String> phoneNumbers = contact.getPhoneNumbers();
                for(String phoneNumber : phoneNumbers) {
                    String contactPhoneNumber = normalizePhoneNumber(phoneNumber);

                    if(inputPhoneNumber.equals(contactPhoneNumber)) {
                        String name = contact.getName();
                        Log.i(TAG, "Found name " + name + " corresponding to phone number " + inputPhoneNumber);

                        Bundle result = new Bundle();
                        result.putString(PHONE_KEY, inputPhoneNumber);
                        result.putString(NAME_KEY, name);

                        return result;
                    }
                }
            }

            Log.w(TAG, "No contact name found for " + inputPhoneNumber);

        } else {
            Log.e(TAG, String.format("Need to provide an input bundle with key '%s' corresponding to the phone number", PHONE_KEY));
        }

        return null;
    }

    @Override
    public String getDescription() {
        return DESCRIPTION;
    }

    /**
     *
     * @param phoneNumber
     * @return The phone number only containing numerics; all other symbols are stripped out
     */
    private String normalizePhoneNumber(String phoneNumber) {
        return phoneNumber.replaceAll("\\D+", "");
    }
}
```

## Extending the base class

All uPALs must extend the `MicroPALProviderService` base class and declare the
type of system API data they expect to receive.

```java
public class NumberToNamePAL extends MicroPALProviderService<ListItem<ContactItem>> {
    // ...
}

```

The parameterized type must be an `Item` or any class derived from it. The following
`Item` types are available:

| Item type                             | DataRequest.DataType type         | Notes
| ------------------------------------- | --------------------------------- | --------------------------------------------------------------------- |
| Item                                  | ANY                               | uPAL will operate on any Item-derived data type                       |
| EmptyItem                             | EMPTY                             | uPAL will ignore the Item and only operate on the PAL params Bundle   |
| ListItem\<CallItem\>                  | CALL\_LOGS                        | List of incoming and outgoing calls                                   |
| ListItem\<ContactItem\>               | CONTACTS                          | Full contact list                                                     |
| ListItem\<CalendarEventItem\>         | CALENDAR                          | List of calendar events within a given time                           |
| ListItem\<MessageItem\>               | SMS                               | List of incoming and outgoing SMS within given time. See note below   |
| LocationItem                          | LOCATION                          | Latitude/longitude data                                               |
| DeviceStateItem                       | PHONE\_STATE                      | Information about the phone                                           |

**Note**: The current implementation only operates on text-only messages
between individual contacts. Group SMS and MMS are unsupported.

### Declaring the constructor

The uPAL's constructor must declare the matching `DataRequest.DataType` and
pass it up to the superclass.

```java
public NumberToNamePAL() {
    super(DataRequest.DataType.CONTACTS);
}
```

### Implementing the abstract methods

uPALs must implement two abstract methods from the superclass: `onReceive()` and
`getDescription`.

#### onReceive()

```java
public Bundle onReceive(ListItem<ContactItem> contactItemList, Bundle paramsFromApp) {
    // ...
}
```

This method transforms sensitive data as received from system API calls (i.e.,
the `Item`-typed first parameter) and given by the app (i.e., the `Bundle`-typed
second parameter). You may choose to ignore either one or both of these parameters
in your uPAL transformation. 

The result of this method is sent back to the calling app. Returning `null` will
indicate to the calling app that a failure occurred.

#### getDescription()

```java
public String getDescription() {
    return DESCRIPTION;
}
```

This method defines a human-readable description of the uPAL.

## Registering the uPAL

uPALs must declare their ability to listen for data transformation requests from
the Private Data Service. Ensure that the uPAL's core service has has the
`android.permission.PRIVATE_DATA_PROVIDER_SERVICE` permission and the
`android.privatedata.MicroPALProviderService` intent filter in the app's
manifest.

```xml
<service android:name=".NameToNumberPAL"
    android:enabled="true"
    android:exported="true"
    android:permission="android.permission.PRIVATE_DATA_PROVIDER_SERVICE">
    <intent-filter>
        <action android:name="android.privatedata.MicroPALProviderService" />
    </intent-filter>
</service>
```

## Recommended practices

Although uPALs are executed asynchronously, they're meant to be lightweight
functions that [do one thing and do it well](https://en.wikipedia.org/wiki/Unix_philosophy).
uPAL developers should try to abide by the following practices to ensure
their transformations follow this design goal.

* uPALs should be stateless. Any state should be maintained by the calling app
and passed back to the uPAL using the params `Bundle` in subsequent calls.
* uPALs should avoid I/O operations such as disk and network functions. I/O
is relatively slow and can be unreliable, leading to uPALs that potentially
don't terminate.
* A single APK can hold many distinct independent uPALs. Split up disparate
loosely-related transformations into individual uPAL services.


