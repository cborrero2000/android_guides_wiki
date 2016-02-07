### Overview

Parse provides a cloud-based backend service to build data-driven mobile apps quickly.  Facebook, which acquired the company more than 3 years ago, announced that the service would be shutting down on **January 28, 2017**.   An [open source version](https://github.com/ParsePlatform/parse-server) enables developers to continue using their apps was published, along with a [migration guide](https://parse.com/docs/server/guide#migrating).  

While there are many 
[alternate options to Parse](https://github.com/relatedcode/ParseAlternatives), most of them lack either the functionality, documentation, or sample code to enable quick prototyping.  For this reason, the open source Parse version is a good option to use with minimal deployment/configuration needed.

You can review this [Wiki](https://github.com/ParsePlatform/parse-server/wiki) to understand the current development progress of this app.  Push notifications, for instance, are currently not implemented but appears to be supported soon.

### Setting a new Parse Server

The steps described [this guide](https://devcenter.heroku.com/articles/deploying-a-parse-server-to-heroku) walk through most of the process of setting an open source version with Parse.  There are obviously many other hosting options, but deploying with Heroku is probably the simplest.  The instructions basically entail the following steps:

1. [Register](https://id.heroku.com/signup/login) for a free Heroku account.
2. Establish a credit card through [Account Settings](https://dashboard.heroku.com/account) in order to add a free MongoDB sandbox instance.
3. Fork a copy of the [Parse server example](https://github.com/ParsePlatform/parse-server-example). 
    * Modify `index.js` to include `CLIENT_KEY`.  See (https://github.com/ParsePlatform/parse-server-example/pull/46/files).  
    * Push to this branch with this change.
4. Create a new Heroku app, pointing to this repository that was created.
5. Add the MongoDB instance as an Add-On.
6. Setup environment variables in the app settings (`https://dashboard.heroku.com/apps/<app name>/settings`):

    <img src="http://imgur.com/shCWGQX.png"/>
    * Set `MASTER_KEY` to be the master key used to read data.  Otherwise, the server will not load properly.  
    * Set `APP_ID` for the app identifier.  If you do not set one, the default is set as `myAppId` (see [the source code](https://github.com/ParsePlatform/parse-server-example/blob/master/index.js#L14-L18)). 
    * Verify the MONGOLAB_URI has been added.  It should be there if the MongoDB add-on was added. 
    * Set `PARSE_MOUNT` to be '/`.  This enables the Parse Client SDK's to be able to connect correctly.
    * Set `CLIENT_KEY` to be your client key.  You will use this info later for the Client SDK setup.
7. Deploy the Heroku app.  The app should be hosted at `https://<app name>.herokuapp.com`.

The important file to review for the Parse server example is [here](https://github.com/ParsePlatform/parse-server-example/blob/master/index.js).  You can see that it takes a few environment variables to run.

### Testing Deployment

After deployment, try to connect to the site.  If you see `{error: "unauthorized"}`, the basic configuration is successful. 

Next, make sure you can create Parse objects.  You do not need a client Key to write new data:

```bash
curl -X POST   -H "X-Parse-Application-Id: myAppId"   -H "Content-Type: application/json"   -d '{"score":1337,"playerName":"Sean Plott","cheatMode":false}'   https://yourappname.herokuapp.com/parse/classes/GameScore
```

To read data back, you will need to specify the master key:

```bash
curl -X GET  -H "X-Parse-Application-Id: myAppId"   -H "X-Parse-Master-Key: abc"    https://yourappname.herokuapp.com/parse/classes/GameScore
```

You can also verify whether the objects were created by clicking on the MongoDB instance in the Heroku panel:

<img src="http://imgur.com/bbj2e9N.png"/>

<img src="http://imgur.com/snPqYkz.png"/>

### Enabling Client SDK integration

Make sure you have the latest Parse-Android SDK in your `app/build.gradle` file.  It should be at least `1.13.0`:

```gradle
dependencies {
    compile 'com.parse:parse-android:1.13.0'
    compile 'com.parse:parseinterceptors:0.0.2' // for logging API calls to LogCat
    
}
```

Modify your `Parse.initialize()` command to point to this Heroku server.  You must be on the latest Parse Android SDK to have these options.  

```java
public class ChatApplication extends Application {
    @Override
    public void onCreate() {
        super.onCreate();

        // Stetho.initializeWithDefaults(this);

        // set applicationId, clientKey, and server based on the values in the Heroku settings.
        // any network interceptors must be added with the Configuration Builder given this syntax
        Parse.initialize(new Parse.Configuration.Builder(this)
                .applicationId("myAppId") // should correspond to APP_ID env variable
                .clientKey("clientKey")  // should correspond to CLIENT_KEY env variable
                .addNetworkInterceptor(new ParseLogInterceptor())
                .server("https://parse-testing-port.herokuapp.com").build());
    }
}
```

**Note**: there appears to be a [bug](https://github.com/ParsePlatform/Parse-SDK-Android/issues/393) in the Android SDK that only allows base URL's to be added with the `.server()` call.  Therefore, the instructions in the official migration guide that suggests using `/parse` as part of the URL does not work.

### Troubleshooting

* If you have issues, remove the `PARSE_MOUNT` environment variable temporarily.  You should see `I dream of being a web site.` if the site loaded correctly.   

* If you see `Application Error` or `An error occurred in the application and your page could not be served. Please try again in a few moments.`, double-check that you set a `MASTER_KEY` in the Heroku environment settings for that app.

  <img src="http://imgur.com/uMYwPmS.png">

* Download the Heroku Toolbelt app [here](https://toolbelt.heroku.com/) to help view system logs. 

  <img src="http://imgur.com/Ch0mZOK.png"/>

  First, you must login with your Heroku login and password:

  ```bash
  heroku login
  ```

   You can then view the system logs by specifying the app name:
   ```bash
   heroku logs -app <app name>
   ```

   The logs should show the response from any types of network requests made to the site.  Check the `status` code.  If you see that it's 404, then double-check that you have set the `PARSE_MOUNT` environment variable is set to `/`:

   ```
   2016-02-07T08:28:14.292475+00:00 heroku[router]: at=info method=POST path="/classes/Message" host=parse-testing-port.herokuapp.com request_id=804c2533-ac56-4107-ad05-962d287537e9 fwd="101.12.34.12" dyno=web.1 connect=1ms service=2ms status=404 bytes=179
   ```

* You can also use Facebook's [Stetho](http://facebook.github.io/stetho/) interceptor to watch network logs with Chrome:

  ```gradle
  dependencies {
    compile 'com.facebook.stetho:stetho:1.3.0'
  }
  ```

  And add this network interceptor as well:

  ```java
  Parse.initialize(new Parse.Configuration.Builder(this)
       .addNetworkInterceptor(new ParseStethoInterceptor())
  ```