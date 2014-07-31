#How Kii Cloud helps IoT products be on fire
[TOC]
###1.Abstract
![concept of iot](https://raw.githubusercontent.com/leonardean/ImgUpload/master/internet-of-things-more-devices-than-people.png)
Trends show that the Internet of Things(IoT) has great potential to enhance people's lives in many perspectives. Though the IoT is yet to be standardized in terms of centricity and Machine to Machine(M2M) communication, its concept has been well explained by products consisting of data-transmittable swiches, sensors for smart house and wearable devices for human centric functionality. With hardware devices, cloud services play an important role that **connects** and makes the best out of collected information with variety of business logic.

[Kii Cloud](http://cn.kii.com/)([English Website](http://en.kii.com/)) is a **Mobile Back-end as a Service(MBaaS)** that offers developers an easy way to build a full-stack mobile backend to their apps in minutes, with carrier-grade technology, plus analytics to help to succeed. Kii Cloud can be used on almost every mobile platform and covers the most demanded functionality for developers to **freely** tailor their back end business logic. This article shows a typical story of how Kii Cloud can be used to help an IoT device to ease its life and be more productive.
###2.Application Scenario
Suppose that we have a wearable device with sensors to detect environmental status (e.g. tmperature, UV index, humidity) around its wearer. It can periodically transmit detected data via **BLE**(Bluetooth Low Energy) or **IR**(Infrared) to a receiver such as a central control or a mobile phone. Our goal is to make the device act as not only a real time environment tracker (it already seem to be), but also a data generator/provider, so that its data can be consumed to produce more **added value**. Examples of data consumption could be:
* multi-dimentional data viewing
* data analysis
* get social and share

The goal can be acheived with the help of Kii Cloud as a back-end cloud service. As the wearable device only supports BLE and IR due to energy efficiency, a smart phone which is normally not far from the user can be used as a middleware for data transmission. The whole scenario of the IoT application can be sensibly shown below.

![scenario](https://raw.githubusercontent.com/leonardean/ImgUpload/master/device%20to%20cloud.png)

Let's take a technical view at how Kii Cloud can be used in this application. Please have a look at some involved terminologies of  Kii Cloud if you are new to Kii Cloud. (You do not need a thorough understanding of these at this moment as they will be explained in later contents)
* It is very easy to add a Kii Cloud back end service for you app. Register and log in the [Developer Portal](https://developer.kii.com/?locale=en), and then click on **Create App** to start a new app back end service. During development, this is probably where you spend most of your time besides front end app programming. Find your **access keys** and keep in mind that they are important and need to be kept safely.
* [Kii Cloud SDK](http://documentation.kii.com/en/starts/cloudsdk/) provides the basic and most demanded functions for leveraging Kii Cloud. They cover almost every mobile platform: Android, iOS, Unity, Javascript, and Terminal Tools. For more advanced use of Kii Cloud, [REST](http://documentation.kii.com/en/guides/rest/) and [Server Extension](http://documentation.kii.com/en/guides/serverextension/) may need to be used. In this application, we will have a taste of these two as well.
* In Kii Cloud, [Bucket](http://documentation.kii.com/en/starts/cloudsdk/managing-data/bucket/) works as data container and **Access Controler**. There are three types of buckets: **App Scope**, **Group Scope**, and **User Scope** with different default ACL. However, the ACL of buckets can be customized by adding specific rules.
* When using Kii Cloud SDK, data is store in the form of [KiiObject](http://documentation.kii.com/en/starts/cloudsdk/managing-data/object-storage/). Internally, they are store as **JSON Key-Value** pairs.
* Kii Cloud provides flexible [Analytics](http://documentation.kii.com/en/guides/android/managing-analytics/) that enables you to learn more about how your app is going on (**App Data Analytics**) and user behavior(**Event Analytics**). In addition, the analytics are also accessible from the front end app so that your users can get designated analysis as well.

###3.Kii SDK Initialization
In order to use Kii Cloud SDK features in the front end app, the SDK needs to be firstly initialized. Following demo shows the initialization in Android:
```java
package com.kii.wearable.demo;

import android.app.Application;
import com.kii.cloud.analytics.KiiAnalytics;
import com.kii.cloud.storage.Kii;

public class DemoApplication extends Application {
  @Override
  public void onCreate() {
    super.onCreate();
    Kii.initialize(Constants.APP_ID, Constants.APP_KEY, Kii.Site.CN);
    KiiAnalytics.initialize(this, Constants.APP_ID, Constants.APP_KEY, KiiAnalytics.Site.CN);
  }
}
```

###4.User Registration
Device pairing and user registration is the first step of a user using the product. User registration and login can be quite straight forward in front end app using Kii Cloud SDK. The following demo shows the implementation in Android:
```java
@Override
public void onClick(View view) {
  switch (view.getId()) {
    case R.id.btn_login:
      KiiUser.logIn(callback, usernameEdit.getText().toString(),
        passwordEdit.getText().toString());
      showProgress(R.string.logging_in);
      break;
    case R.id.btn_register:
      KiiUser user = KiiUser.builderWithName(usernameEdit.getText().toString()).build();
      user.register(callback, passwordEdit.getText().toString());
      showProgress(R.string.registering);
      break;
  }
}
```
In `callback` function, user **Access Token** can be locally stored and used for faster login next time. For more information on auto access token login, please click [HERE](http://documentation.kii.com/en/starts/cloudsdk/managing-users/accesstoken/).

Practically, one device might be shared to be used by multiple users (apps). Also, one app can map to multiple wearable devices. In order to make sure that correct data can be retrieved by user and by device, a server extension function that modifies the ACL of each newly registrated user's **User Scope Bucket** used for data storage needs to be executed when a user is created. A [Hook](http://documentation.kii.com/en/guides/serverextension/managing_servercode/#serverhook) for **Server Triggered Execution** is therefore used.
```
{
  "kiicloud://users": [
    {
      "when": "USER_CREATED",
      "what": "EXECUTE_SERVER_CODE",
      "endpoint": "modifyBucketACL"
    }
  ]
}
```
Please note that in this hook file:
* `"kiicloud://users"` is used when the hook listens for a trigger that occurs on any users.
* `"USER_CREATED"` specifies when the triggers fires.
* `"modifyBucketACL"` is the name of the function to be executed.

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
For more information on server code writing, managing, and executing, please click [HERE](http://documentation.kii.com/en/guides/serverextension/).
###5.Data sending
Since data is sent in the form of Key-Value pairs, a typical piece of data uploaded to the bucket `wdDataRecord` would be like:
```
{
  'username': 'demo',
  'deviceID': 'device0',
  'temperature': 30,
  'humidity': 40,
  'UV': 60,
  'time': 1398782607
}
```
The following demo shows how to upload such a piece of data in Android.
```java
@Override
public void onClick(View view) {
  switch (view.getId()) {
    case R.id.save:
      KiiBucket bucket = KiiUser.getCurrentUser().bucket("wdDataRecord");
      KiiObject data = bucket.object();
      try {
        data.set("humidity", Double.parseDouble(humidityEdit.getText().toString()));
        data.set("temperature", Double.parseDouble(temperatureEdit.getText().toString()));
        data.set("uv", Double.parseDouble(uvEdit.getText().toString()));
        data.set("deviceID", deviceEdit.getText().toString());
        data.set("time", System.currentTimeMillis());
        KiiACL dataACL = data.acl();
        dataACL.putACLEntry(new KiiACLEntry(KiiAnyAuthenticatedUser.create(),
          KiiACL.ObjectAction.READ_EXISTING_OBJECT, true));
        dataACL.save(new KiiACLCallBack(){
          @Override
          public void onSaveCompleted(int token, KiiACL acl, Exception exception) {
            //TODO
          }
        });
        data.save(new KiiObjectCallBack() {
          @Override
          public void onSaveCompleted(int token, KiiObject object,
            Exception exception) {
            //TODO
          }
        });
      } 
      catch (Exception e) {
        //TODO    
      }
      break;
  }
}
```
As explained in the previous section, there is a multiple to multiple relationship between users and devices. Threrefore, each time a user uploads his data onto his user scope bucket, the device-user mapping needs to be maintained in an **App Scope Bucket**. Given the username, deviceID, and app admin, the server extension function is shown below:
```javascript
function maintainRel(user, deviceID, admin, done, log) {
  //get app scope bucket
  var relation = admin.bucketWithName("UserDeviceRelation");
  //create query with clause
  var idEq = KiiClause.equals("userName", user.getUsername());
  var devEq = KiiClause.equals("deviceID", deviceID);
  var clause = KiiClause.and(idEq, devEq);
  var query = KiiQuery.queryWithClause(clause);

  relation.executeQuery(query, {
    success: function (queryPerformed, resultSet, nextQuery) {
      //if the mapping is not found, then add it
      if (resultSet.length == 0) {
        var rel = relation.createObject();
        rel.set("deviceID", deviceID);
        rel.set("userID", user.getUUID());
        rel.set("userName", user.getUsername());
        rel.save({
          success: function (obj) {
            recordLog(obj, "save rel success", done, log);
          },
          failure: function (obj, errorString) {
            recordLog(errorString, "save rel fail", done, log);
          }
        });
      } else {
        recordLog("every ok,nothing need to do", "finish", done, log);
      }
    },
    failure: function (queryPerformed, anErrorString) {
      recordLog(anErrorString, "query rel fail", done, log);
    }
  });
}
```
In addition, we would like to do something else on this trigger. We would like our users to be able to see some analytics in dimention of time.(e.g. users can see the average temperature near him every day/week/month) So the uploaded data needs to be modified by adding extra time attributes: `dayNo`,`weekNo` and `monthNo` for [App Data Analytics](http://documentation.kii.com/en/guides/android/managing-analytics/flex-analytics/analyze-application-data/) setup. Please note that Analytics are essentially MapReduce calculations run periodically, so the analytics result can be retrieved directy without any front end calculation. It is both network and power efficient. The server extension for uploaded data modification is shown below:
```javascript
function modifyRecord(param, context, done) {
  //get app admin and object
  var admin = context.getAppAdminContext();
  var obj = KiiObject.objectWithURI(param.uri);

  KiiUser.authenticateWithToken(context.getAccessToken(), {
    success: function (theUser) {
      obj.refresh({
        success: function (theObj) {
          //set dayno, weekno, monthno
          var time = new Date(theObj.time);
          var dayNum = Math.round(time / (1000 * 60 * 60 * 24) );
          var weekNo = Math.round(dayNum / 7);
          var month = time.getMonth() + time.getFullYear() * 12;
          theObj.set("weekno", weekNo);
          theObj.set("monthno", month);
          theObj.set("dayno",dayNum);          
          //set a "username" attribute for analytics
					theObj.set("username",theUser.getUsername());
          theObj.save({
            success: function (theObj) {
              //set the object to be readable by other authenticate users
              var objACL = KiiACLEntry.entryWithSubject(new KiiAnyAuthenticatedUser(), 
                KiiACLAction.KiiACLObjectActionRead);
              var acl = theObject.objectACL();
              acl.putACLEntry(objACL);
              acl.save({
                success: function (theACL) {
                  var deviceID = theObj.get("deviceID");
                  //this is where the above function is used
                  maintainRel(theUser, deviceID, admin, done, log);
                }, failure: function (theACL, errorString) { /*TODO*/ }});},
            failure: function (theObj, errorString) { /*TODO*/ }});},
        failure: function (theObject, anErrorString) { /*TODO*/ }});},
    failure: function (theUser, error) { /*TODO*/
    }
  });
}
```
Finally, the server extension functions will be executed upon this **Single** Hook file:
```
{
  "kiicloud://users/*/buckets/wdDataRecord":[
    {
      "when":"DATA_OBJECT_CREATED",
      "what":"EXECUTE_SERVER_CODE",
      "endpoint":"modifyRecord"
    }
  ],
  "kiicloud://users": [
    {
      "when": "USER_CREATED",
      "what": "EXECUTE_SERVER_CODE",
      "endpoint": "modifyBucketACL"
    }
  ]
}
```
Please note that only one server code file with one hook file can be active at a time, so please put the above three functions in one Js file for [deployment](http://documentation.kii.com/en/guides/serverextension/managing_servercode/#deploy).
###6.Data Retriving
###7.Data Analysis
###8.More
![kii cloud](https://raw.githubusercontent.com/leonardean/ImgUpload/master/B.png)