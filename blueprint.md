#How Kii Cloud helps IoT products be on fire

[TOC]
##1.Abstract
![concept of iot](https://raw.githubusercontent.com/leonardean/ImgUpload/master/internet-of-things-more-devices-than-people.png)
Trends show that the Internet of Things(IoT) has great potential to enhance people's lives in many perspectives. Though the IoT is yet to be standardized in terms of centricity and Machine to Machine(M2M) communication, its concept has been well explained by products consisting of data-transmittable swiches, sensors for smart house and wearable devices for human centric functionality. With hardware devices, cloud services play an important role that **connects** and makes the best out of collected information with variety of business logic.

[Kii Cloud](http://cn.kii.com/)([English Website](http://en.kii.com/)) is a **Mobile Back-end as a Service(MBaaS)** that offers developers an easy way to build a full-stack mobile backend to their apps in minutes, with carrier-grade technology, plus analytics to help to succeed. Kii Cloud can be used on almost every mobile platform and covers the most demanded functionality for developers to **freely** tailor their back end business logic. This article shows a typical story of how Kii Cloud can be used to help an IoT device to ease its life and be more productive.
##2.Application Scenario
Imagine that we have a wearable device with sensors to detect environmental status (e.g. tmperature, UV index, humidity) around its wearer. It can transmit the detected data via **BLE**(Bluetooth Low Energy) or **IR**(Infrared) to a mobile phone. Our goal is to make the device act as not only a real time environment tracker (it already seem to be), but also a data generator/provider, so that its data can be consumed to produce more **added value**. Examples of data consumption could be:
* multi-dimentional data viewing
* data analysis
* get social and share

The goal can be acheived with the help of Kii Cloud as a back-end cloud service. As the wearable device only supports BLE and IR due to energy efficiency, a smart phone which is normally not far from the user can be used as a middleware for data transmission. The whole scenario of the IoT application can be sensibly shown below.

![scenario](https://raw.githubusercontent.com/leonardean/ImgUpload/master/device%20to%20cloud.png)
##3.User Registration
Device pairing and user registration is the first step of a user using the product.

```javascript
function modifyBucketACL(param, context, done) {
  //Get the current user and its bucket named "wdDataRecord"
  var userID = param.userID;
  var admin = context.getAppAdminContext();
  var theUser = admin.userWithID(userID);
  var bucket = theUser.bucketWithName("wdDataRecord");
  var log = admin.bucketWithName("log");

  //Create a custom ACL rule
  var bucketACL = KiiACLEntry.entryWithSubject(new KiiAnyAuthenticatedUser(),
    KiiACLAction.KiiACLBucketActionQueryObjects);

  // Get the ACL handle and put the rule in the handle
  var acl = bucket.acl();
  acl.putACLEntry(bucketACL);
  acl.save({
    success: function (theACL) {
      recordLog(theACL, "set acl success", done, log);
    },
    failure: function (theACL, errorString) {
      recordLog(errorString, "set acl fail", done, log);
    }
  });
}
```
##4.Data sending
##5.Data Analysis
##6.Data Consumption
##7.More
![kii cloud](https://raw.githubusercontent.com/leonardean/ImgUpload/master/B.png)