#How Kii Cloud helps IoT products be on fire
[TOC]
###Abstract
![concept of iot](https://raw.githubusercontent.com/leonardean/ImgUpload/master/internet-of-things-more-devices-than-people.png)
Trends show that the Internet of Things(IoT) has great potential to enhance people's lives in many perspectives. Though the IoT is yet to be standardized in terms of centricity and Machine to Machine(M2M) communication, its concept has been well explained by products consisting of data-transmittable swiches, sensors for smart house and wearable devices for human centric functionality. With hardware devices, cloud services play an important role that **connects** and makes the best out of collected information with variety of business logic.

[Kii Cloud](http://cn.kii.com/)([English Website](http://en.kii.com/)) is a **Mobile Back-end as a Service(MBaaS)** that offers developers an easy way to build a full-stack mobile backend to their apps in minutes, with carrier-grade technology, plus analytics to help to succeed. Kii Cloud can be used on almost every mobile platform and covers the most demanded functionality for developers to **freely** tailor their back end business logic. This article shows a typical story of how Kii Cloud can be used to help an IoT device to ease its life and be more productive.
###Application Scenario
####General Scenario
Suppose that we have a wearable device with sensors to detect environmental status (e.g. tmperature, UV index, humidity) around its wearer. It can periodically transmit detected data via **BLE**(Bluetooth Low Energy) or **IR**(Infrared) to a receiver such as a central control or a mobile phone. Our goal is to make the device act as not only a real time environment tracker (it already seem to be), but also a data generator/provider, so that its data can be consumed to produce more **added value**. Examples of data consumption could be:
* multi-dimentional data viewing
* data analysis
* get social and share

The goal can be acheived with the help of Kii Cloud as a back-end cloud service. As the wearable device only supports BLE and IR due to energy efficiency, a smart phone which is normally not far from the user can be used as a middleware for data transmission. The whole scenario of the IoT application can be sensibly shown below.

![scenario](https://raw.githubusercontent.com/leonardean/ImgUpload/master/device%20to%20cloud.png)

####Technical Terminology
Let's take a technical view at how Kii Cloud can be used in this application. Please have a look at some involved terminologies of  Kii Cloud if you are new to Kii Cloud. (You do not need a thorough understanding of these at this moment as they will be explained in later contents)
* It is very easy to add a Kii Cloud back end service for you app. Register and log in the [Developer Portal](https://developer.kii.com/?locale=en), and then click on **Create App** to start a new app back end service. During development, this is probably where you spend most of your time besides front end app programming. Find your **access keys** and keep in mind that they are important and need to be kept safely.
* [Kii Cloud SDK](http://documentation.kii.com/en/starts/cloudsdk/) provides the basic and most demanded functions for leveraging Kii Cloud. They cover almost every mobile platform: Android, iOS, Unity, Javascript, and Terminal Tools. For more advanced use of Kii Cloud, [REST](http://documentation.kii.com/en/guides/rest/) and [Server Extension](http://documentation.kii.com/en/guides/serverextension/) may need to be used. In this application, we will have a taste of these two as well.
* In Kii Cloud, [Bucket](http://documentation.kii.com/en/starts/cloudsdk/managing-data/bucket/) works as data container and **Access Controler**. There are three types of buckets: **App Scope**, **Group Scope**, and **User Scope** with different default ACL. However, the ACL of buckets can be customized by adding specific rules.
* When using Kii Cloud SDK, data is store in the form of [KiiObject](http://documentation.kii.com/en/starts/cloudsdk/managing-data/object-storage/). Internally, they are store as **JSON Key-Value** pairs.
* Kii Cloud provides flexible [Flex Analytics](http://documentation.kii.com/en/guides/android/managing-analytics/) that enables you to learn more about how your app is going on (**App Data Analytics**) and user behavior(**Event Analytics**). In addition, the analytics are also accessible from the front end app so that your users can get designated analysis as well.

###Back End Design
####Bucket Design
First of all, let's talk about data storage in this application. As mentioned above, envionmental data are going to be transmitted from the wearable device to a mobile phone, and the phone will send the data into **buckets** of Kii Cloud. There are three types of buckets with different [Scopes](http://documentation.kii.com/en/starts/cloudsdk/cloudoverview/bucketscope/):

| Scope | Who can create a new Bucket? |Who can create a new data in the Bucket?|Who can query for data in the Bucket?|Who can drop the Bucket?|Who can add ACL entries?|
|--------|--------|--------|--------|--------|--------|
|Application|- Any authenticated users|- Any authenticated users|- Any authenticated users<br>- Anonymous users|-  Any authenticated users| N/A (Only an application admin)|
|Group|- All group members<br>- The group creator|- All group members<br>- The group creator<br>- The bucket creator|- All group members<br>- The group creator<br>- The bucket creator|- The group creator<br>- The bucket creator|- The group creator|
|User|- The user|- The user<br>- The bucket creator|- The user<br>- The bucket creator|- The user<br>- The bucket creator|- The user|
In this application, User scope bucket will be used for environmental data storage so that each user will use his/her own bucket to keep the data private.

Practically, one device might be shared to be used by multiple users (apps). Also, one app can map to multiple wearable devices. It is forseeable that a user might want to retrieve the data sent by himself and also by his device. It is intuitively easy to retrieve data from a user's own User Scope bucket but not easy to get data sent by a certain device because we have no idea where the data is (could be in any bucket). Looping through all the user scope bucket is neither computational nor time efficient.

Therefore, we need to use another bucket to store the **user-device mapping relationship** so that when querying data by a certain device, we know where to find. The bucket needs to be accessible by any autheticated user, so we use an **App Scope** Bucket to do the job. To summarize, we need to use two types of buckets in this application: 
* A User Scope Bucket for each user to storage environmental data.
* An App Scope Bucket to maintain user-device mapping relationship.

####Server Extension Functions
A part from bucket design, we also need to design some server extension functions to fulfil the business logics of this application. A server extension function can do whatever that is achievable in front end app, but instead of running on client side, it runs on the server side. A server extension function can be run with function call from client side; and with the help of a **Server Hook **, it can also be run automatically based on schedule or triggered by server events such as user registration, data CRUD, etc.

In the above sub-section, we explained the need to retrieve environmental data by user and device. From the table, we know that if a bucket is User Scope bucket, only the corresponding user can query the data in the bucket. Therefore, it would be impossible to query all the environmental data by device when the device has been used by multiple users, because the data are split into multiple users' User Scope buckets and one can only query from his own User Scope bucket. Here we will need to use a server extension function to make the business logic technically feasible by modifying the ACL of one's user scope bucket(**execute server extension function**) when the user has just registered(**triggered by server event**).

Besides bucket ACL modification, we also need a function to maintain the user-device mapping relationship as Key-Value pairs in an App Scope bucket. A good time to execute the function is when there is environmental data record uploaded to the server. What the function does would be:
* Check if the device-user pair has already been stored.
* If so, do nothing.
* If not, store the device-user pair in the App Scope Bucket.

In addition, we would like our users to be able to see some data analytics in the dimensions of different time interval(e.g. how the temperature changes in different weeks). Setting the wanted data field and aggregation function, Kii Cloud can do the analytics in dimension of day by default(we will cover how later). However, to get analytics in demension of week/month/quarter/year, we will have to download the existing analytics data and do calculations on client side in the front end app. Doing so is neither computational nor time efficient. Therefore, we can use a server extension function that adds custom fields (day number, week number, month number) to the uploaded data when there occurs a data upload on the server, so that the added fields can be used as custom dimension when doing analytics.

Since the functions are executed on server side, they need to be deployed to Kii Cloud. When using Server Hook, two files need to be deployed:
* A Javascript file containing single or multiple server extension functions.
* A Server Hook Config file containing information on when the functions are executed.

To deploy the two files, please follow this [link](http://documentation.kii.com/en/guides/commandlinetools/#installation) to install command line tool first; and then execute the following command in terminal:
```
node bin/kii-servercode.js deploy-file \
  --file <js_function_file> \
  --site us \
  --app-id <your_app_id> \
  --app-key <your_app_key> \
  --client-id <your_client_id> \
  --client-secret <your_client_secret> \
  --hook-config <hook_file>
```
###User Registration
User registration is the first step of a user using the product. User registration and login can be quite straight forward in front end app using Kii Cloud SDK. The following demo shows the implementation in Android:
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

In order to make sure that correct data can be retrieved by user and by device as previously mentioned, a server extension function that modifies the ACL of each newly registrated user's **User Scope Bucket** (used for environmental data storage) needs to be executed when a user is created. A [Hook](http://documentation.kii.com/en/guides/serverextension/managing_servercode/#serverhook) for **Server Triggered Execution** is therefore used.
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
###Data sending
####Environmental Data Upload
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
####User-Device Mapping
As explained in the previous section, there is a multiple to multiple relationship between users and devices. It is very straight forward to retrieve the uploaded data of a user from front end app, as the data from the same user are stored in the same User Scope Bucket. However, it would be very inefficient to retrieve data of a certain device because the data might be in any bucket; and the front end app has to loop through all the buckets to find the corresponding data. Threrefore, each time a user uploads his data onto his user scope bucket, the device-user mapping needs to be maintained in an **App Scope Bucket**, so that the front end app can easily track the certain buckets that contain the data of a device.

Given the username, deviceID, and app admin, the server extension function is shown below:
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
####Data Record Modification
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
###Data Analysis
####Analytics Setup
![kii cloud](https://raw.githubusercontent.com/leonardean/ImgUpload/master/B.png)
Data Analytics is one of the most valuable features of Kii Cloud service. There are two types of Analytics:
* [Basic Analytics](http://documentation.kii.com/en/guides/android/managing-analytics/basic-analytics/) with general application analytics with predefined metrics.
* [Flex Analytics](http://documentation.kii.com/en/guides/android/managing-analytics/flex-analytics/) with flexible application analytics with customizable metrics. Flex Analytics contains:
 * [App Data Analytics](http://documentation.kii.com/en/guides/android/managing-analytics/flex-analytics/analyze-application-data/) that analyse object data created by your application. You can use any stored key-values for defining your custom metrics.
 * [Event Data Analytics](http://documentation.kii.com/en/guides/android/managing-analytics/flex-analytics/analyze-event-data/) that analyse data thrown by your application besides application data. Kii Analytics provides you a feature to throw such event data with arbitrary key-values.

In this application, we are going to use App Data Analytics to get the average temperature of users by username/day/week/month. First of all, there are several steps to setup an App Data Analytics in the **Developer Portal**:
1. Sign in to the Developer Portal and go to **Analytics** section page
2. Switch to **Config** tab and Click on button **Add** to create a new aggregation rule.
3. Select **App Data** and click on **Select a conversion rule**.
4. Since you do not have a conversion rule, you will neet to create one first. Fill in the space and create a conversation rule as below:
![conversation rule](https://raw.githubusercontent.com/leonardean/ImgUpload/master/conversation%20create.png)
5. save the conversation rule and set the Analytics as below:
![analytics set](https://raw.githubusercontent.com/leonardean/ImgUpload/master/analytics%20set.png)
Save and Activate the Aggregation rule, it will be ready to use. You will get an **ID** for the Aggregation Rule after activation. The ID will be used in the front end app to retrieve analytics data.

Please note that analytics are calculated once every 24 hours on Kii Cloud. Therefore, it is likely that you do not have an instant view of the metrics after you just created an Aggregation Rule. After some time, you will be able to see the corresponding metrics as below:

![analytics view](https://raw.githubusercontent.com/leonardean/ImgUpload/master/analytics%20view.png)

The metrics page contains two charts and one table. The line graph on the top shows the temperature values of different users by day. The bar chart at the bottom right gives a more direct view of the value comparison between the three users. Dimention can be switched by clicking on any of the four pre-set fields so that the chart can show temperature values of different devices/weeks/months.

In addition to flexible dimention switch, you can also add filters for more detailed information. In this case, the temperature values are filtered by deviceID `device1`. Therefore, only users who have used the device are shown on the graph. The table shows the detailed temperature values of each username/deviceID with comparison to total average. Clicking on any of the record in the table can toggle on/off the corresponding data display in the two charts.
####Analytics Data Retrieval
The metrics on the Developer Portal is only for yourself as the developer of the app. If you would like your user to see the analytics data in the front end app, the corresponding [REST API](http://documentation.kii.com/rest/#analytics_management-retrieve_flex_analytics_results)(Tip: please do click on the API link on the doc page for detail expansion) will be needed. The following demo shows how to do it in Javascript:
```javascript
function doAnalysis(){
  var ajaxData = {
    type: "GET",
    //the sub-domain varies for different server region, e.g. api-cn2 us used for China server
    url: "https://api-cn2.kii.com/api/apps/" + Kii.getAppID()+ "/analytics/120/data?" +
      "group=username&" +
      "filter001.name=deviceID&" +
      "filter001.value=device1",
    dataType: "json",
    headers: {"x-kii-appid": Kii.getAppID(), "x-kii-appkey": Kii.getAppKey()},
    contentType: "application/json",
    complete: function(res, status) {
      console.log("Completed Request[" + res.status + "]");
      console.log(res.responseText);
    }
  };
  $.ajax(ajaxData);
}
```
The above demo retrieves the analytics data with:
* **Aggregation Rule ID** of `120`.
* **Filter name** of `deviceID`.
* **Filter value** of `device1`.

Please note that multiple filters can be added by appending `filterN.name` and `filterN.value` (`N` is an integer) to the query string. The retrieved data can be used in whatever ways in the front end app to satisfy your users.

###Raw Data Retriving
With the help of Kii Cloud SDK, retrieving raw data from Kii Cloud server is as easy as uploading. Following is a demo showing this in Android:
```java
@Override
protected void onCreate(Bundle savedInstanceState) {
  super.onCreate(savedInstanceState);
  setTitle(R.string.all_my_data);
  setContentView(R.layout.activity_all_data);
  //create a new query, you can certainly add clause on it if you want
  KiiQuery all_query = new KiiQuery();
  //get the user scope bucket
  KiiBucket bucket = KiiUser.getCurrentUser().bucket("wdDataRecord");
  bucket.query(new KiiQueryCallBack<KiiObject>() {
    @Override
    public void onQueryCompleted(int token, KiiQueryResult<KiiObject> result,
      Exception exception) {
      //TODO
    }
  }, all_query);
}
```
Note that this demo retrieved all the raw data from the bucket `wdDataRecord`. **KiiClause** can be added to the `KiiQuery` if you need more specific queries. For more information on complex query, please click [HERE](http://documentation.kii.com/en/guides/android/managing-data/object-storages/querying/).

###More
This article demonstrated how Kii Cloud can be used as a back end service in typicle IoT case. With **Kii SDK** and **Developer Portal**, developers can easily build and manage a back end service for their front end apps without writing a single line for back end server except Hooks and Server extension functions. The stability, reliability and scalibility of Kii Cloud service guarantee that app developers can focus on the user interface and user experience of their front end apps without worrying about back end development and maintenance.

As a front end app developer no matter in Android, iOS, Unity, or Javascript, you probably worry about the security of your application. Since the specification of the Kii Cloud is open, any attackers who know your application's access key (i.e. App ID and App Key) can imitate your application and access the Kii Cloud. This, however, does not mean they can exploit your users' data. As long as the user data are protected with the proper **access control**, they are safe even if the access keys are leaked. On the other hand, the application **admin** credentials (i.e. Client ID and Client Secret) must not be leaked at any cost. Any attackers who get these credentials will gain a full access to all user data. Please click [HERE](http://documentation.kii.com/en/starts/cloudsdk/hint/security/) for tips on how to keep your application secured.

Finally, all the demo codes shown in this artical can be found in [HERE](https://github.com/leonardean/IOTDemo--Android-ServerExtension-JsFront). Please feel free to download and have go with the full source code.



