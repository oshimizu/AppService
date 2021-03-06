---
title: "Adjusting the HTTP call with Azure Mobile Apps"
author_name: Adrian Hall 
layout: single
excerpt: "How to adjust the HTTP call for Android, iOS, JavaScript, and .Net with Azure Mobile Apps" 
toc: true
toc_sticky: true
---

Azure Mobile Apps provides an awesome client SDK for dealing with common mobile client problems - data access, offline sync, Notification Hubs registration and authentication. Sometimes, you want to be able to do something extra in the client. Perhaps you need to adjust the headers that are sent, or perhaps you want to understand the requests by doing diagnostic logging. Whatever the reason, Azure Mobile Apps is extensible and can easily handle these requirements.

## Android (Native)

You can implement a ServiceFilter to manipulate requests and responses in the HTTP pipeline. The general recipe is as follows:  

```java
ServiceFilter filter = new ServiceFilter() {
  @Override public ListenableFuture handleRequest(ServiceFilterRequest request, NextServiceFilterCallback next) {
    // Do pre-HTTP request requirements here
    request.addHeader("X-Custom-Header", "Header Value"); // Example: Adding a Custom Header
    Log.d("Request to ", request.getUrl());// Example: Logging the request ListenableFuture
    responseFuture = next.onNext(request);

    Futures.addCallback(responseFuture, new FutureCallback() {
      @Override
      public void onFailure(Throwable exception) {
        // Do post-HTTP response requirements for failures here
        Log.d("Exception: ", exception.getMessage()); // Example: Logging an error
      }

      @Override public void onSuccess(ServiceFilterResponse response) {
        // Do post-HTTP response requirements for success here
        if (response != null && response.getContent() != null) {
          Log.d("Response: ", response.getContent());
        }
      }
    });
    return responseFuture;
  }
};

MobileServiceClient client = new MobileServiceClient("https://xxx.azurewebsites.net", this).withFilter(filter);  
```

You can think of the `ServiceFilter` as a piece of middleware that wraps the existing request/response from the server.

## iOS (Native)

Similar to the Android case, you can wrap the request in a filter. For iOS, the same code (once translated) works in both Swift and Objective-C. Here is the Swift version:

```swift
class CustomFilter: NSObject, MSFilter {

  func handleRequest(request: NSURLRequest, next: MSFilterNextBlock, response: MSFilterResponseBlock) {
    var mutableRequest: NSMutableURLRequest = request.mutableCopy()

    // Do pre-request requirements here
    if !mutableRequest.allHTTPHeaderFields["X-Custom-Header"] {
        mutableRequest.setValue("X-Custom-Header", forHTTPHeaderField: "Header Value")
    }

    // Invoke next filter
    next(customRequest, response)
  }
}

let client = MSClient(applicationURLString: "https://xxx.azurewebsites.net").clientWithFilter(CustomFilter())
```

## JavaScript & Apache Cordova

As you might exepct given the Android and iOS implementations, the JavaScript client (and hence the Apache Cordova implementation) also uses a filter - this is just a function that the request gets passed through:

```js
function filter(request, next, callback) {
  // Do any pre-request requirements here
  console.log('request = ', request);                     // Example: Logging
  request.headers['X-Custom-Header'] = "Header Value";    // Example: Adding a custom here

  next(request, callback);
}

var client = new WindowsAzure.MobileServiceClient("https://xxx.azurewebsites.net").withFilter(filter);
```

## Xamarin / .NET

The equivalent functionality in the .NET world is a **Delegating Handler**. The implementation and functionality are basically the same as the others:

```c#
public class MyHandler: DelegatingHandler
{
  protected override async Task SendAsync(HttpRequestMessage message, CancellationToken token)
  {
    // Do any pre-request requirements here
    request.Headers.Add("X-Custom-Header", "Header Value");

    // Request happens here
    var response = await base.SendAsync(request, cancellationToken);

    // Do any post-request requirements here

    return response;
  }
}

// In your mobile client code:
var client = new MobileServiceClient("https://xxx.azurewebsites.net", new MyHandler());
```

## General Notes

There are some HTTP requests that never go through the filters you have defined. A good example for this is the login process. However, all requests to custom APIs and/or tables get passed through the filters. You can also wrap the client multiple times. For example, you can use two separate filters - one for logging and one for the request. In this case, the filters are executed in an onion-like fashion - The last one added is the outer-most. The request goes through each filter in turn until it gets to the actual client, then the response is passed through each filter on its way out to the requestor. Finally, note that this is truly a powerful method allowing you to change the REST calls that Azure Mobile Apps makes - including destroying the protocol that the server and client rely on. Certain headers are required for Azure Mobile Apps to work, including tokens for authentication and API versioning. Use wisely. (Cross-posted to [My Personal Blog](http://wp.me/p6gQt8-2ou))
