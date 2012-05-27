---
layout: post
title: "JSON serialized Date passed between iOS and WCF (and vice versa)"
date: 2012-05-27 20:00
comments: true
categories: [JSON, iOS 5, .NET 4.0 WCF, Web Service, HTTP, Parsing, DateTime, NSDate]
---

Knowing that JSON still does not support Date objects, one can have a really painful experience sending and receiving date and time values through the wire. Following here is an example of how such a functionality can be achieved easily having .NET 4.0 WCF Web Service on one side and iOS 5.0 application on the other.

To make things simpler, we will obey to the WCF convention for serialization of `DateTime` objects which declares that such objects will be serialized as: `"/Date(1292851800000+0100)/"` or `"\/Date(1292851800000-0100)\/"` where 1292851800000 is milliseconds since 1970 and +0100 is the timezone. That way, we don't have to touch anything on the WCF side and use the built in JSON parser in .NET accordingly. 

Of course, everything that is communicated via JSON has to be a string value.

1. WCF .NET side code
===
Example `Event` class that represents data about events. Instances of this class are sent via `WebInvoke` methods to the iOS client application. Only the attributes of type `DateTime` are showed.

``` c# Event.cs
using System;
using System.Collections.Generic;
using System.Linq;
using System.Web;
using System.Runtime.Serialization;

namespace RESTfulWCF
{
    [DataContract]
    public class Event
    {
        [DataMember]
        public DateTime dateFrom { get; set; }

        [DataMember]
        public DateTime dateTo { get; set; }
    }
}
```

That's it. This (plus additional simple return code at the `WebInvoke` methods that are responsible for the actual JSON responses)  provide `DateTime` serialization in the format previously showed but **for the `set` methods they except input parameters in the same format as well**.

That practically means that we ourselves should do the proper formatting when sending `NSDate` on the iOS side, and also parse the consumed JSON string from the WCF side.

2. iOS side code
===
First, I am providing the method that does the conversion (parsing) of a `DateTime` (.NET) object serialized as JSON by WCF to a `NSDate` object. Have the format in mind ["/Date(1292851800000+0100)/"] before looking at the code. My practice here is *always to keep date and time objects in their UTC time zone on the server and do the needed time zone calculations on the client side*. That practically means that time zone offsets will be added here, on the client side parsing code.

``` objc Deserialization of Date Objects
- (NSDate *)deserializeJsonDateString: (NSString *)jsonDateString
{   
    NSInteger offset = [[NSTimeZone defaultTimeZone] secondsFromGMT]; //get number of seconds to add or subtract according to the client default time zone 
    
    NSInteger startPosition = [jsonDateString rangeOfString:@"("].location + 1; //start of the date value
    
    NSTimeInterval unixTime = [[jsonDateString substringWithRange:NSMakeRange(startPosition, 13)] doubleValue] / 1000; //WCF will send 13 digit-long value for the time interval since 1970 (millisecond precision) whereas iOS works with 10 digit-long values (second precision), hence the divide by 1000
    
    [[NSDate dateWithTimeIntervalSince1970:unixTime] dateByAddingTimeInterval:offset];
    
    return date;
}
```
Now follows the code for formatting the `NSDate` objects in the appropriate format for transmission.

``` objc NSDate to JSON date string
- (void)jsonPostRequestAddEvent
{ 
    NSError *error;
    
    NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
    [formatter setDateFormat:@"Z"]; //for getting the timezone part of the date only.
    
    NSString *jsonDateFrom = [NSString stringWithFormat:@"/Date(%.0f000%@)/", [self.dateFrom timeIntervalSince1970],[formatter stringFromDate:self.dateFrom]]; //three zeroes at the end of the unix timestamp are added because thats the millisecond part (WCF supports the millisecond precision)
    NSString *jsonDateTo = [NSString stringWithFormat:@"/Date(%.0f000%@)/", [self.dateTo timeIntervalSince1970],[formatter stringFromDate:self.dateTo]];
    
    NSArray *objects = [NSArray arrayWithObjects:jsonDateFrom, jsonDateTo, nil];
    NSArray *keys = [NSArray arrayWithObjects:@"dateFrom", @"dateTo", nil];
    NSDictionary *eventDict = [NSDictionary dictionaryWithObjects:objects forKeys:keys];
    
    NSDictionary *jsonDict = [NSDictionary dictionaryWithObject:eventDict forKey:@"myEvent"];
    
    NSURL *url = [NSURL URLWithString:@"http://yourDomain.com/WCFService.svc/addEvent"];
    
    NSData* jsonData = [NSJSONSerialization dataWithJSONObject:jsonDict options:kNilOptions error:&error];
    
    //NSLog([[NSString alloc] initWithData:jsonData encoding:NSUTF8StringEncoding]); //debug only
    
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url cachePolicy: NSURLRequestReloadIgnoringCacheData timeoutInterval: 30.f];
    
    [request setHTTPMethod:@"POST"];
    [request setValue:@"application/json" forHTTPHeaderField:@"Accept"];
    [request setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];
    [request setValue:[NSString stringWithFormat:@"%d", [jsonData length]] forHTTPHeaderField:@"Content-Length"];
    [request setHTTPBody: jsonData];
    
    self.urlConnection = [[NSURLConnection alloc] initWithRequest:request delegate:self];
    [UIApplication sharedApplication].networkActivityIndicatorVisible = YES; //this is important for the UX (User Experience)  
}
```
I hope this code snippets will do any help to some of you guys.



