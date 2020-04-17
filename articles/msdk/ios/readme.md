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

In order to be able to recive a call you need setup a few app capabillities. 

First you need to add the following to your `Info.plist` file.

```xml
<key>NSMicrophoneUsageDescription</key>
<string>YOUR APPS USAGE DESCRIPTION FOR USING THE MICROPHONE</string>
<key>UIBackgroundModes</key>
<array>
<string>audio</string>
<string>voip</string>
</array>
```

Microphone usage description is needed because the SDK uses APIs enabling sound capture.
The background modes are needed in order for calls to work even when the app is in the background, as well as being woken up by VoIP push notifications.

Second, you need to add the Push Notifications capabillity to your app, so that the SDK can receive VoIP push notifications.

### VoIP Push notifications

You will need to setup a push notification certificate and specify it as a voip certificate in your Apple Developer portal, this certificate needs to be provided to Freespee in order for us to be able to wake up your app in preparation for incoming calls.

### Steps

- Go to https://developer.apple.com/
- Make sure your App ID has the Push Notifications service enabled.
- Create a VoIP Services Certificate under Certificates/Production

Export the certificate from Keychain Access by selecting the certificate, and its private key. Right-click and select "Export 2 Items".
Open a terminal and move into the folder where the .p12 file was saved.

Using the `openssl` command, extract the certificate and private key from the file.
```
$> openssl pkcs12 -in Certificates.p12 -nocerts -out key.pem
$> openssl rsa -in key.pem -out key.pem
$> openssl pkcs12 -in Certificates.p12 -clcerts -nokeys -out cert.pem
```

You should end up with a key and cert, looking something like this:

```
-----BEGIN CERTIFICATE REQUEST-----
...lines of text here...
-----END CERTIFICATE REQUEST-----
```

```
-----BEGIN RSA PRIVATE KEY-----
...lines of text here...
-----END RSA PRIVATE KEY-----
```

These should be provided to Freespee according to communicated instructions.

## How to use

Please refer to the Example App reference implementation for a full SDK integration example.

### Initialize the SDK

Configure the SDK with your pre shared API Key.

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

### Connecting and providing a user credential


Before accepting incoming calls you need to connect the SDK to Freespee - linking the running instance with a user identifier. The identifier can be any arbitrary identifier unique to your application user.
Make sure to setup the `FreespeeDelegate` before calling `connect` as the registration will ask the delegate for a user credential as part of the process. The credential returned will dictate what user identifier is connected to the SDK.

```swift

Freespee.shared().delegate = self

Freespee.shared()?.connect({ (error) in
    if let error = error {
        print("Failed to connect \(error)")
    }
})

extension MyClass: FreespeeDelegate {
    
    ...
    
    func freespee(_ freespee: Freespee, provideCredential callback: @escaping (String?) -> Void) {
        let userIdentifier = ...
        let credential = try? FreespeeCredential.generate(withSecret: "MY_SECRET", userIdentifier: userIdentifier)
        callback()
    }
    
}

```
The connected user identifier will now be reachable for Freespee calls routed to your application.

During development you can use the `FreespeeUserCredential` class to generate a credential in-app. However, this is not recommended for production use.
For AppStore builds of your app you should generate the credential server-side in your application backend and fetch it in `provideUserCredential`.

### Generating a user credential

The credential is a JWT token signed with the API Secret. Pseudo code for generating a user credential below.

```swift

let payload = ["userId": "USER_ID"]
let secret = "MY_SECRET"
let credential = JWTToken(payload: payload, secret: secret, alg: HS256)

```

### Call UI

You need to provide your own UI for the call screen. For calls you will get an instance of a `FreespeeCall` given to you.
Typically your UI will set itself as the `FreespeeCallStateChangeListener` so that it can react on changes in the call.
Refer to the `FreespeeCall` API documentation for actions available.

### Listen to incoming calls

#### Setup a PushKit registry and forward data to Freespee.

```swift

let pushRegistry = PKPushRegistry(queue: DispatchQueue.main)
pushRegistry.delegate = self
pushRegistry.desiredPushTypes = [.voIP]

extension AppDelegate: PKPushRegistryDelegate {
    
    func pushRegistry(_ registry: PKPushRegistry, didUpdate pushCredentials: PKPushCredentials, for type: PKPushType) {
        guard type == .voIP else { return }
        let deviceToken = pushCredentials.token.map { String(format: "%02x", $0) }.joined()
        Freespee.shared()?.registerDeviceToken(deviceToken)
    }
    
    func pushRegistry(_ registry: PKPushRegistry, didReceiveIncomingPushWith payload: PKPushPayload, for type: PKPushType, completion: @escaping () -> Void) {
        guard type == .voIP else { return }
        Freespee.shared()?.handleIncomingNotification(payload.dictionaryPayload)
        completion()
    }
    
    func pushRegistry(_ registry: PKPushRegistry, didInvalidatePushTokenFor type: PKPushType) {
        guard type == .voIP else { return }
        Freespee.shared()?.unregisterDeviceToken()
    }
    
}

```

#### Regarding changes to VoIP push notifications in iOS 13.

As of iOS 13 Apple introduced new rules for VoIP push notifications. When receiving an incoming VoIP push, an app is now expected to report a new incoming call to CallKit, before the  `registry:didReceiveIncomingPushWith:completion` function completes.
If your app fails to do so it will be terminated, and if continuously failing it might stop recieving VoIP push completely.

When calling `Freespee.shared()?.handleIncomingNotification(payload.dictionaryPayload)` with a valid freespee VoIP payload, a call to the function `freespee:handleIncomingCall` of your delegate will be made synchronously. 
In this callback you should report the call to CallKit. See example code below. You can then safely call `completion()` as shown in the example above.

#### Setup CallKit integration.

```swift

/// Callback block for when call is connecting
var callConnectingCallback: ((_ success: Bool) -> Void)?
var current: FreespeeCall?

let providerConfiguration = CXProviderConfiguration(localizedName: "MyApp")
providerConfiguration.iconTemplateImageData = UIImage(named: "app-icon")?.pngData()
providerConfiguration.maximumCallGroups = 1
providerConfiguration.maximumCallsPerCallGroup = 1

let callKitProvider = CXProvider(configuration: providerConfiguration)
callKitProvider.setDelegate(self, queue: DispatchQueue.main)

extension CallController: CXProviderDelegate {
    
    func providerDidReset(_ provider: CXProvider) {
        current?.hangup()
    }
    
    func provider(_ provider: CXProvider, didActivate audioSession: AVAudioSession) {
        Freespee.shared()?.didActivateAudioSession()
    }
    
    func provider(_ provider: CXProvider, didDeactivate audioSession: AVAudioSession) {}
    
    func provider(_ provider: CXProvider, perform action: CXAnswerCallAction) {
        assert(current != nil)
        assert(action.callUUID == current?.uuid)
        
        callConnectingCallback = { (success) in
            print("fulfilling anwer call action \(success)")
            if success {
                action.fulfill()
            } else {
                action.fail()
            }
        }
        current?.answer()
    }
    
    func provider(_ provider: CXProvider, perform action: CXEndCallAction) {
        assert(current != nil)
        assert(action.callUUID == current?.uuid)
        
        current?.hangup()
        action.fulfill()
    }
}


```

#### Implement SDK delegate function which will be called once an incoming call hits the SDK.

```swift

/// Callback block for when call is connecting
var callConnectingCallback: ((_ success: Bool) -> Void)?
var current: FreespeeCall?

Freespee.shared()?.delegate = self

private func startTrackingCall(_ call: FreespeeCall) {
    call.add(stateChangeListener: self)
    self.current = call
}

private func stopTrackingCall() {
    self.current?.remove(stateChangeListener: self)
    self.current = nil
}

extension MyClass: FreespeeDelegate {
    
    ....
    
    func freespee(_ freespee: Freespee, handleIncomingCall call: FreespeeCall) {
    
        // Report incoming call to CallKit
        let callHandle = CXHandle(type: .phoneNumber, value: call.peerIdentifier)

        let callUpdate = CXCallUpdate()
        callUpdate.remoteHandle = callHandle
        callUpdate.supportsDTMF = false
        callUpdate.supportsHolding = false
        callUpdate.supportsGrouping = false
        callUpdate.supportsUngrouping = false
        callUpdate.hasVideo = false
        
        callKitProvider.reportNewIncomingCall(with: call.uuid, update: callUpdate) { error in
            if let error = error {
                print("Failed to report incoming call: \(error.localizedDescription).")
            } else {
                print("Incoming call reported.")
            }
        }
        
        startTrackingCall(call)
        presentCall(call)
    }

}

extension MyClass: FreespeeCallStateChangeListener {

func freespeeCall(_ call: FreespeeCall, didChangeStateFrom oldState: FreespeeCallState, to newState: FreespeeCallState) {
    
    // The call started connecting
    if oldState == .pending && newState == .connecting {
        callConnectingCallback?(true)
        callConnectingCallback = nil
    }
    
    // The call closed
    if newState == .closed {
        
        stopTrackingCall()
        
        // The call was missed
        if oldState == .pending {
            callConnectingCallback?(false) // Report back to CallKit
            callConnectingCallback = nil
            postMissedCallNotification(for: call)
        }
        
        // Report to CallKit when and why the call ended.
        var reason: CXCallEndedReason?
        switch call.endedReason {
        case .answeredElsewhere: reason = .answeredElsewhere
        case .failed: reason = .failed
        case .remoteEnded: reason = .remoteEnded
        case .unanswered: reason = .unanswered
        case .localEnded: break // If ending the call locally, CallKit already knows.
        @unknown default:
            fatalError()
        }
        if let reason = reason {
            callKitProvider.reportCall(with: call.uuid, endedAt: Date(), reason: reason)
        }
        
    }
    
}

func freespeeCall(_ call: FreespeeCall, failedWithError error: Error) {
    print("Call failed \(error)")
}

func freespeeCall(_ call: FreespeeCall, durationTick duration: TimeInterval) {}

func freespeeCall(_ call: FreespeeCall, didReceive segment: FreespeeUserSegment) {}

func freespeeCall(_ call: FreespeeCall, didReceive userJourney: FreespeeUserJourney) {}

}
```

### Handling contextual information for incoming calls

Freespee contextual data is populated as a call comes in to your app. You can access this data directly on the `FreespeeCall` instance provided.

When the data becomes available the `FreespeeCallStateChangeListener` will be notified. It has two callbacks for when Freespee data is recieved. 

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

The available data for a call is detailed in `FreespeeUserSegment` and `FreespeeUserJourney`, you can use this data to do further lookups within your own app domain as needed and then presenting it in the call UI for the call.

Example usage: Download image / title from the last visited URL using OpenGraph meta tags on the web site.

### Error handling

Most SDK functions will have a callback with an optional error supplied. The error will be on of the errors defined in the `FreespeeError` enum.
Refer to the source documentation for the most up to date information on what specific errors are to be expected where, and what they stand for.  

## Development

### Xcode Simulator

The SDK is reliant on CallKit and VoIP-push notifications, neither of which will work on the simulator. Because of this you will need to exclusively test on device.
However, you are able to test on the simulator if you make sure not to report any incoming call to CallKit and handle all other call logic without interacting with CallKit.

You will need to set the SDK in a listening state as described below, since PushKit does not work on the simulator.

#### Always listen

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
