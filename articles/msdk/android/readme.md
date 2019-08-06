# ![Freespee Logo](https://analytics.freespee.com/images/freespee_logo.svg)

# Freespee Android SDK

The Freespee SDK lets your app accept incoming calls routed through the Freespee Platform.
The SDK exposes contextual information for incoming calls that you can present in a way that makes sense for your business.

## Requirements

- Android SDK 18

## Setup

### Add the SDK to your app project

Copy the provided .AAR-file to the applications libs folder. Then add the following to the app build.gradle file.

```
repositories {
    mavenCentral()
        flatDir {
        dirs 'libs'
    }
}

dependencies {
    ...
    implementation 'com.freespee.freespee:freespee@aar'
}
```

### Capabillities

In order to be able to recive a call you need setup a few app capabillities.

For Android sdk version 23 or higher the application needs to request the runtime permission to access the microphone. 
Without the proper permissions the SDK will throw a runtime exception causing the application to crash. 

Read more about requesting runtime permissions at the [offical developer portal](https://developer.android.com/training/permissions/requesting).

Furthermore you have to provide your client secret. This is done by adding a <meta-data> tag in your Manifest-file.

```
<meta-data
    android:name="com.freespee.sdk.API_KEY"
    android:value="YOUR-CLIENT-SECRET" />
```

### Push notifications

Push notifications are used to notify the SDK of an incoming call.
Implementing FCM can be done by using the Android studios assistant or by following this [set-by-step guide](https://firebase.google.com/docs/cloud-messaging/android/client).

To check if incoming push belongs to Freespee:

```java
public class MyFirebaseNotificationService extends FirebaseMessagingService {

    @Override
    public void onMessageReceived(RemoteMessage remoteMessage) {
        // Check if this is a Freespee push
        if (Freespee.isFreespeePushData(remoteMessage.getData())) {
            // Start service with push data
            final HashMap<String, String> data = new HashMap<>(remoteMessage.getData());
            
            Intent serviceIntent = new Intent(getApplicationContext(), MyFreespeeService.class);
            serviceIntent.setAction(MyFreespeeService.IMCOMING_PUSH_DATA);
            serviceIntent.putExtra("push-data", data);
            
            getApplicationContext().startService(serviceIntent);
        }
    }
}

```

Handle incoming push data inside your service, as shown below:

```java
public class MyFreespeeService extends Service implements Freespee.OnIncomingCallListener {

    public static final String INCOMING_PUSH_DATA = "MyFreespeeService.INCOMING_PUSH_DATA";

    private Freespee mFreespee;

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        if (intent != null && intent.getAction() != null) {
            switch (intent.getAction()) {
                
                // ...
                
                // Handle incoming push data
                case INCOMING_PUSH_DATA: {
                    if (intent.getExtras() != null && intent.getExtras().containsKey("push-data")) {
                        final HashMap<String, String> messageData = (HashMap<String, String>) intent.getExtras().getSerializable("push-data");

                        mFreespee.handlePushData(messageData);
                        mFreespee.setIncomingCallListener(this);
                    }
                    break;
                }
            }
        }

        return super.onStartCommand(intent, flags, startId);
    } 

    // ...
    
}
```


## How to use

### Initialize the SDK

We recommend you to implement a service that initializes the SDK and holds an instance of `Freespee`.

```java
public class MyFreespeeService extends Service implements Freespee.OnIncomingCallListener {

    private Freespee mFreespee;
    
    private Freespee.FreespeeClientProvider provider = new Freespee.FreespeeClientProvider() {
        @Override
        public Context onProvideContext() {
            return getApplicationContext();
        }            
        
        @Override
        @NonNull
        public String providePushToken() {
            // Provide your FCM push token here
        }
    };

    @Override
    public void onCreate() {
        super.onCreate();

        mFreespee = Freespee.getInstance(provider);
    }   
}
```

### Connecting

Before accepting incoming calls you need to connect to Freespee identifying the calling SDK with an identifier, typically a user id or some other identifier that is unique and makes sense to your organization.

```java
public class MyFreespeeService extends Service implements Freespee.OnIncomingCallListener {

    private Freespee mFreespee;

    // ...

    public class MyFreespeeServiceBinder extends Binder {
        
        private Freespee.OnConnectListener connectListener = new Freespee.OnConnectListener() {
            @Override
            public void onStart() {
                // The SDK is now connected and ready to use
            }
            
            @Override
            public void onError(FreespeeError error) {
                // The SDK failed to start with an error
            }
        };
        
        // ...

        public void connect(String identifier) {
            try {
                mFreespee.connect(identifier, connectListener);
            } catch(FreespeeError error) {
                // Handle error here
            }
        }  

        public void disconnect() {
            mFreespee.disconnect();
        }
    }
}
```

### Call UI

You need to provide your own UI for the call. For incoming calls you will get an instance of a `FreespeeCall` given to you.
Typically your UI will set itself as the `FreespeeCall.StateListener` so that it can react on changes in the call.
Refer to the `FreespeeCall` API documentation for actions available.

### Listen to incoming calls

Add a listener to the SDK which will be called once an incoming call hits the SDK.

```java
public class MyFreespeeService extends Service implements Freespee.OnIncomingCallListener {

    private Freespee mFreespee;
    private FreespeeCall mCurrentCall;

    @Override
    public void onCreate() {
        super.onCreate();

        // ...

        mFreespee.setIncomingCallListener(this);
    }

    @Override
    public void onIncomingCall(FreespeeCall call) {
        mCurrentCall = call;
        mCurrentCall.addCallListener(new MyCallStateListener());

        // Show incoming call UI
    }  

    public class MyCallStateListener implements FreespeeCall.StateListener {
        @Override
        public void onUserJourneyAvailable(FreespeeCall call) {
            
        }

        @Override
        public void onSegmentDataAvailable(FreespeeCall call) {

        }

        @Override
        public void onRinging(FreespeeCall call) {

        }

        @Override
        public void onCallEstablished(FreespeeCall call) {
            startCallActivity();
            mNotificationManager.removeNotifications();
            mNotificationManager.updateInCallNotification(call);
        }

        @Override
        public void onCallEnded(FreespeeCall call, String reason) {
            mNotificationManager.removeNotifications();
            stopForeground(true);
            stopSelf();
        }

        @Override
        public void onCallError(FreespeeError error) {
            mNotificationManager.removeNotifications();
            stopForeground(true);
            stopSelf();
        }   
    }
}
```

### Handling contextual information for incoming calls

The `FreespeeCall.StateListener` has two callbacks for when Freespee data is recieved for the call. Typically this information is available instantly when a call comes in.

```java
public class MyCallStateListener implements FreespeeCall.StateListener {
    @Override
    public void onUserJourneyAvailable(FreespeeCall call) {
        // User journey data for the call is available.
    }

    @Override
    public void onSegmentDataAvailable(FreespeeCall call) {
        // Segment data for the call is available.
    }

    // ...

}
```

The available data for a call is detailed in `com.freespee.freespee.data.segment.Segment` and `com.freespee.freespee.data.journey.UserJourney`, 
you can use this data to do further lookups within your own app domain as needed and then presenting it in the call UI for the incoming call. 

### Error handling

The SDK uses the `FreespeeError` class to represent errors that can occur. 
To differentiate between different errors all FreespeeErrors have a distinct type (`FreespeeError.Type`). The type can be fetched by calling error.getType() on an error.
Refer to the source documentation for the most up to date information on what specific errors are to be expected where, and what they stand for.  

## Support

Contact developer support on mobiledev@freespee.com
