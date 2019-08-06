# ![Freespee Logo](https://analytics.freespee.com/images/freespee_logo.svg)

# Freespee iOS SDK

The Freespee SDK lets your app accept incoming calls routed throught the Freespee Platform.
The SDK exposes contextual information for incoming calls that you can present in a way that makes sense to your business.

## Requirements

- iOS >= 10.0

## Setup

### Add the SDK to your app project

#### Cocoapods

Add this to your `Podfile`, replace `x.y.z` with the latest version.

```
  pod 'FreespeeSDK', podspec: 'https://portal.msdk.freespee.com/releases/ios/x.y.z/FreespeeSDK.podspec' 
```

#### Manual

* Download the latest release and extract the contents locally.
* Drag `FreespeeSDK.framework` into the _Embedded Binaries_ section of your target.
* Add a new _Script Phase_ in our target's _Build Phases_ and make sure this _Run Script Phase_ is below the _Embed Frameworks_ build phase. You can drag and drop build phases to rearrange them.

Paste the following in the script text field:
```
bash "$BUILT_PRODUCTS_DIR/$FRAMEWORKS_FOLDER_PATH/FreespeeSDK.framework/strip-frameworks.sh"
```
This will strip unnecessary achitectures from the framework to allow app store submission.

### Capabillities

In order to be able to recive a call you need setup a few app capabillities. First you need to add the following to your `Info.plist` file.
Microphone usage description is needed because the SDK uses APIs enabling sound capture.
The background modes are needed in order for calls to work even when the app is in the background, as well as being woken up by VoIP push notifications.
```xml
<key>NSMicrophoneUsageDescription</key>
<string>YOUR APPS USAGE DESCRIPTION FOR USING THE MICROPHONE</string>
<key>UIBackgroundModes</key>
<array>
<string>audio</string>
<string>voip</string>
</array>
```

### VoIP Push notifications

You will need to setup a push notification certificate and specify it as a voip certificate in your Apple Developer portal, this certificate needs to be provided to Freespee in order for us to be able to wake up your app in preparation for incoming calls.

### Steps

- Go to https://developer.apple.com/
- Make sure your App ID has the Push Notifications service enabled.
- Create a VoIP Services Certificate under Certificates/Production

When you have generated the certificate you will need to provide it to Freespee.
Export the certificate from Keychain Access by selecting the certificate, and its private key. Right-click and select "Export 2 Items".
Open a terminal and move into the folder where the .p12 file was saved.

Using the `openssl` command, extract the certificate and private key from the file.
```
$> openssl pkcs12 -in Certificates.p12 -nocerts -out key.pem
$> openssl rsa -in key.pem -out key.pem
$> openssl pkcs12 -in Certificates.p12 -clcerts -nokeys -out cert.pem
```

## How to use

### Initialize the SDK

```swift
import UIKit
import FreespeeSDK

@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {

    var window: UIWindow?

    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplicationLaunchOptionsKey: Any]?) -> Bool {
        Freespee.configure(APIKey: "API_KEY")
        return true
    }
}
```

### Connecting

Before making outgoing calls or accepting incoming calls you need to connect to Freespee identifying the calling SDK with an identifier, typically a user id or some other identifier that is unique and makes sense to your organization.

```swift
Freespee.shared()?.connectIdentifier(identifier, completion: { (error) in
    if let error = error {
        print("Failed to connect \(error)")
    }
})
```

### Call UI

You need to provide your own UI for the call screen. For calls you will get an instance of a `FreespeeCall` given to you.
Typically your UI will set itself as the `FreespeeCallStateChangeListener` so that it can react on changes in the call.
Refer to the `FreespeeCall` API documentation for actions available.


### Listen to incoming calls

Add a listener to the SDK which will be called once an incoming call hits the SDK.

```swift
Freespee.shared()?.incomingCallListener = self

extension MyClass: FreespeeIncomingCallListener {
    
    func freespee(_ freespee: Freespee, handleIncomingCall call: FreespeeCall) {
    
        guard UIApplication.shared.applicationState == .background else {
            // Incoming call while app active - present UI and let the UI listen to state changes
            presentUI(for: call)
            return
        }
        
        // Incoming call in the background - listen for state changes
        call.add(stateChangeListener: self)
        
        // Present a incoming call notification to the user
        // NOTE: We can use the Freespee data provided on the call to add additional information to the notification
        
        let content = UNMutableNotificationContent()
    
        content.title = "Incoming call"
        content.body = "Incoming call from \(call.peerIdentifier)"
        content.sound = UNNotificationSound.default
        let request = UNNotificationRequest(identifier: "freespee-call", content: content, trigger: nil)
        UNUserNotificationCenter.current().add(request) { (error) in
            if let error = error {
                print("Failed to schedule local notification \(error)")
            }
        }
    }
    
    func freespee(_ freespee: Freespee, applicationDidBecomeActiveWithIncomingCall call: FreespeeCall) {
        // No need to listen to state changes here anymore
        call.remove(stateChangeListener: self)
        // Present the call to the user
        presentUI(for: call)
    }

}

extension MyClass: FreespeeCallStateChangeListener {

func freespeeCall(_ call: FreespeeCall, didChangeStateFrom oldState: FreespeeCallState, to newState: FreespeeCallState) {
    
    guard UIApplication.shared.applicationState == .background && newState == .closed else { return }
    
    // Call was closed while app is in background - clear out our incoming call notification and post a missed call notification
    UNUserNotificationCenter.current().removePendingNotificationRequests(withIdentifiers: ["freespee-call"])
    UNUserNotificationCenter.current().removeDeliveredNotifications(withIdentifiers: ["freespee-call"])

    let content = UNMutableNotificationContent()
    content.title = "Missed call"
    content.body = "Missed call from \(call.peerIdentifier)"
    content.sound = nil
    let request = UNNotificationRequest(identifier: "missed-freespee-call", content: content, trigger: nil)
    UNUserNotificationCenter.current().add(request) { (error) in
        if let error = error {
            print("Failed to schedule local notification \(error)")
        }
    }
}

func freespeeCall(_ call: FreespeeCall, failedWithError error: Error) {
    print("Call failed \(error)")
}

func freespeeCall(_ call: FreespeeCall, durationTick duration: TimeInterval) {
    // Not of interest here
}

func freespeeCall(_ call: FreespeeCall, didReceive segment: FreespeeUserSegment) {
    guard UIApplication.shared.applicationState == .background && newState == .pending else { return }
    // We received segment data for the call - we can update the incoming call notification with data
}

func freespeeCall(_ call: FreespeeCall, didReceive userJourney: FreespeeUserJourney) {
    guard UIApplication.shared.applicationState == .background && newState == .pending else { return }
    // We received user journey data for the call - we can update the incoming call notification with data
}

}

```
### Handling contextual information for incoming calls

Typically the Freespee contextual data is available instantly when a call comes in and provided to your app. You can access this data directly on the `FreespeeCall` instance provided.

However, in some cases the data will be populated slightly after and because of that the `FreespeeCallStateChangeListener` has two callbacks for when Freespee data is recieved. 
Your code need to handle this case as well to ensure a great UX for your users.

```swift
extension MyClass: FreespeeCallStateChangeListener {

...

func freespeeCall(_ call: FreespeeCall, didReceive segment: FreespeeUserSegment) {
    // Segment data for the call became available.
}

func freespeeCall(_ call: FreespeeCall, didReceive userJourney: FreespeeUserJourney) {
    // User journey data for the call became available.
}

}

```

The available data for a call is detailed in `FreespeeUserSegment` and `FreespeeUserJourney`, you can use this data to do further lookups within your own app domain as needed and then presenting it in the call UI for the incoming call. 

### CallKit Integration

CallKit integration is currently experimental.

Note that enabling CallKit will limit your posibillities of presenting contextual data as a call comes in because iOS will take ownership of the call screen, at least when the device is locked.

You can enable CallKit on the SDK and the SDK will handle the underlying logic for you.

```swift
// You need to supply your own CXProviderConfiguration as only one instance is permitted per app.  
let providerConfiguration = CXProviderConfiguration(localizedName: "MyApp")
providerConfiguration.iconTemplateImageData = UIImage(named: "appIcon")?.pngData()

// Enable CallKit
Freespee.shared()?.enableCallKit(with: providerConfiguration)

// You can add a custom resolver for incoming calls, letting you provide the display name of the caller on the call screen.
Freespee.shared()?.callKitLocalizedNameResolver = { (call: FreespeeCall, resolve: (FreespeeCall, String) -> Void) in
    self.resolveName(for: call.peerIdentifier)
    resolve(call, "John Doe")
}

// Disable CallKit
Freespee.shared()?.disableCallKit()
```

### Error handling

Most SDK functions will have a callback with an optional error supplied. The error will be on of the errors defined in the `FreespeeError` enum.
Refer to the source documentation for the most up to date information on what specific errors are to be expected where, and what they stand for.  

## Development

### Xcode Simulator

When running your app from Xcode it will not be able to recieve VoIP-push and because of that no incoming phone calls will be triggered.
To allow testing during development you can manually set the SDK in a state where it is constantly listening for incoming calls.
You still need to connect the SDK first.

```swift
Freespee.shared()?.startListeningForIncomingCalls({ (error) in
    if let error = error {
        print("Failed to start listening \(error)")
        } else {
        print("listening")
    }
})

Freespee.shared()?.stopListeningForIncomingCalls({ (error) in
    if let error = error {
        print("Failed to stop listening \(error)")
    } else {
        print("stopped listening")
    }
})

```

### APNS Environments

If your are building your app onto a device from Xcode the app will only accept incoming push notifications in the APNs sandbox environment.
On the other hand, when distributing your app with Testflight or in the AppStore it will only accept notifications in the production environment.

While the VoIP-certificate you have created can be used for both sandbox and production push notifications, the Freespee backend needs to know what environment to use before hand.
Because of this you will typically have two different API Keys for the SDK, one which is bound to the sandbox environment and one for production.

Note that when using one API key, you will only be able to communicate with other apps with that same API key. Be sure to not ship your app with an API-key bound to the sandbox environment as push notifications will not work.

### Freespee Environment

For tracking down errors and/or testing new functionality you may supply a custom environment for the SDK to connect to.
Freespee developers will supply the environment URL if this is needed.

```swift
Freespee.configure(APIKey: "MY_API_KEY", environment: URL(string: "CUSTOM_ENV_URL")!)
```

## Support

Contact developer support on mobiledev@freespee.com
