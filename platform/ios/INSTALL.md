# Integrating custom builds of the Mapbox Maps SDK for iOS into your application

This document explains how to build a development version of Mapbox Maps SDK for iOS for use in your own Cocoa Touch application. To use a production-ready version of the SDK, see the [Mapbox Maps SDK for iOS installation page](https://www.mapbox.com/install/ios/).

### Requirements

The Mapbox Maps SDK for iOS is intended to run on iOS 9.0 and above on the following devices:

* iPhone 4s and above (5, 5c, 5s, 6, 6 Plus, 7, 7 Plus, 8, 8 Plus, X)
* iPad 2 and above (3, 4, Mini, Air, Mini 2, Air 2, Pro)
* iPod touch 5th generation and above


Note that debugging in 32-bit simulators (such as the iPhone 5 or iPad 2) is only partially supported.

The Mapbox Maps SDK for iOS requires:

* Xcode 9.1 or higher to compile from source
* Xcode 8.0 or higher to integrate the compiled framework into an application

Before building, follow these steps to install prerequisites:

1. Install [Xcode](https://developer.apple.com/xcode/)
1. Launch Xcode and install any updates
1. Install [Homebrew](http://brew.sh)
1. Install [Node.js](https://nodejs.org/), [CMake](https://cmake.org/), and [ccache](https://ccache.samba.org):
   ```
   brew install node cmake ccache
   ```
1. Install [xcpretty](https://github.com/supermarin/xcpretty) (optional, used for prettifying command line builds):
   ```
   [sudo] gem install xcpretty
   ```
1. Install [jazzy](https://github.com/realm/jazzy) for generating API documentation:
   ```
   [sudo] gem install jazzy
   ```

### Building the SDK

1. Clone the git repository:
   ```
   git clone https://github.com/mapbox/mapbox-gl-native-ios.git
   cd mapbox-gl-native
   ```
   Note that this repository uses Git submodules. They'll be automatically checked out when you first run a `make` command,
   but are not updated automatically. We recommended that you run `git submodule update` after pulling down new commits to
   this repository.
1. Run `make iframework BUILDTYPE=Release`. The packaging script will produce a `build/ios/pkg/` folder containing:
  - a `dynamic` folder containing a dynamically-linked fat framework with debug symbols for devices and the iOS Simulator
  - a `documentation` folder with HTML API documentation
  - an example `Settings.bundle` containing an optional Mapbox Telemetry opt-out setting

See the [packaging documentation](DEVELOPING.md#packaging-builds) for other build options.

### Installation

There are several ways to install custom builds of the Mapbox Maps SDK for iOS:

#### Dynamic framework

This is the recommended workflow for manually integrating custom builds of the SDK into an application:

1. Build from source manually, per above.

1. Open the project editor, select your application target, then go to the General tab. Drag Mapbox.framework from the `build/ios/pkg/dynamic/` directory into the “Embedded Binaries” section. (Don’t drag it into the “Linked Frameworks and Libraries” section; Xcode will add it there automatically.) In the sheet that appears, make sure “Copy items if needed” is checked, then click Finish.

1. In the Build Phases tab, click the + button at the top and select “New Run Script Phase”. Enter the following code into the script text field:

```bash
bash "${BUILT_PRODUCTS_DIR}/${FRAMEWORKS_FOLDER_PATH}/Mapbox.framework/strip-frameworks.sh"
```

(The last step, courtesy of [Realm](https://github.com/realm/realm-cocoa/), is required for working around an [iOS App Store bug](http://www.openradar.me/radar?id=6409498411401216) when archiving universal binaries.)

##### Snapshot builds

A snapshot build of the dynamic framework, based on the latest commit to the master branch, is available for download [here](https://mapbox.s3.amazonaws.com/mapbox-gl-native/ios/builds/mapbox-ios-sdk-snapshot-dynamic.zip).

#### Static framework

You can alternatively install the SDK as a static framework:

1. Build from source using the `make iframework FORMAT=static BUILDTYPE=Release` command.

1. Drag the Mapbox.bundle and Mapbox.framework from the `build/ios/pkg/static/` directory into the Project navigator. In the sheet that appears, make sure “Copy items if needed” is checked, then click Finish. Open the project editor and select your application target to verify that the following changes occurred automatically:

   - In the General tab, Mapbox.framework is listed in the “Linked Frameworks and Libraries” section.
   - In the Build Settings tab, the “Framework Search Paths” (`FRAMEWORK_SEARCH_PATHS`) build setting includes the directory that contains Mapbox.framework. For most projects, the default value of `$(inherited) $(PROJECT_DIR)` should be sufficient.
   - In the Build Phases tab, Mapbox.bundle is listed in the “Copy Bundle Resources” build phase.

1. Back in the General tab, add the following Cocoa Touch frameworks and libraries to the “Linked Frameworks and Libraries” section:

   - GLKit.framework
   - ImageIO.framework
   - MobileCoreServices.framework
   - QuartzCore.framework
   - SystemConfiguration.framework
   - libc++.tbd
   - libsqlite3.tbd
   - libz.tbd

1. In the Build Settings tab, find the Other Linker Flags setting and add `-ObjC`.

#### CocoaPods

For instructions on installing stable release versions of the Mapbox Maps SDK for iOS with CocoaPods, see [our website](https://www.mapbox.com/install/ios/cocoapods/).

##### Testing pre-releases with CocoaPods

To test pre-releases of the dynamic framework, directly specify the version in your `Podfile`:

```rb
pod 'Mapbox-iOS-SDK', '~> x.x.x-alpha.1'
```

##### Testing snapshot releases with CocoaPods

To test a snapshot dynamic framework build, update your app’s `Podfile` to point to:

```rb
pod 'Mapbox-iOS-SDK-snapshot-dynamic', podspec: 'https://raw.githubusercontent.com/mapbox/mapbox-gl-native-ios/master/platform/ios/Mapbox-iOS-SDK-snapshot-dynamic.podspec'
```

##### Using your own build with CocoaPods

1. Build from source manually, per above.

1. Edit [`Mapbox-iOS-SDK.podspec`](./Mapbox-iOS-SDK.podspec) to point to the local build output:

    ```diff
       m.source = {
    -    :http => "https://mapbox.s3.amazonaws.com/mapbox-gl-native/ios/builds/mapbox-ios-sdk-#{m.version.to_s}-dynamic.zip",
    +    :path => "../../build/ios/pkg/dynamic/",
         :flatten => true
       }
    ```

1. Update your app’s `Podfile` to point to the edited `Mapbox-iOS-SDK.podspec`.

    ```rb
    pod 'Mapbox-iOS-SDK', :path => '{...}/platform/ios/Mapbox-iOS-SDK.podspec'
    ```

1. Run `pod update` to grab the newly-built library.

If using the static framework, add `$(inherited)` to your target’s Other Linker Flags in the Build Settings tab.

##### Using pre-stripped releases with CocoaPods

If you choose to commit the contents of your `Pods` directory to source control and are encountering file size limitations, you may wish to use builds that have been pre-stripped of debug symbols. We publish these to [our public podspecs repository](https://github.com/mapbox/pod-specs/), which should be added as an additional `source` in your app’s `Podfile`.

```rb
source 'https://github.com/mapbox/pod-specs.git'

pod 'Mapbox-iOS-SDK-stripped', '~> x.x.x'
```

Note that these builds lack some debugging information, which could make development more difficult and result in less useful crash reports. Though initially smaller on disk, using these builds has no effect on the ultimate size of your application — see our [Understanding iOS Framework Size guide](https://www.mapbox.com/help/ios-framework-size/) for more information.

#### Carthage

For instructions on installing stable release versions of the Mapbox Maps SDK for iOS with Carthage, see [our website](https://www.mapbox.com/install/ios/carthage/).

##### Testing pre-releases with Carthage

To test pre-releases of the dynamic framework, directly specify the version in your Cartfile:

```json
binary "https://www.mapbox.com/ios-sdk/Mapbox-iOS-SDK.json" ~> x.x.x-alpha.1
```

##### Testing snapshot releases with Carthage

This project does not currently support using snapshot releases via Carthage.

##### Using your own build with Carthage

This project does not support being compiled as a local repository by Carthage.

##### Using pre-stripped releases with Carthage

If you require a build with debug symbols pre-stripped, use [this feed URL](https://www.mapbox.com/ios-sdk/Mapbox-iOS-SDK-stripped.json) with Carthage.

### Configuration

1. Mapbox vector tiles require a Mapbox account and API access token. In the project editor, select the application target, then go to the Info tab. Under the “Custom iOS Target Properties” section, set `MGLMapboxAccessToken` to your access token. You can obtain an access token from the [Mapbox account page](https://www.mapbox.com/studio/account/tokens/).

1. _(Optional)_ Mapbox Telemetry is a [powerful location analytics platform](https://www.mapbox.com/telemetry/) included in this SDK. By default, anonymized location and usage data is sent to Mapbox whenever the host application causes it to be gathered. This SDK provides users with a way to individually opt out of Mapbox Telemetry. You can also add this opt-out setting to your application’s Settings screen using a Settings bundle. An example Settings.bundle is available in the `build/ios/pkg/` directory; drag it into the Project navigator, checking “Copy items if needed” when prompted. In the project editor, verify that the following change occurred automatically:

   - In the General tab, Settings.bundle is listed in the “Copy Bundle Resources” build phase.

### Usage

In a storyboard or XIB, add a view to your view controller. (Drag View from the Object library to the View Controller scene on the Interface Builder canvas.) In the Identity inspector, set the view’s custom class to `MGLMapView`. If you need to manipulate the map view programmatically:

1. Switch to the Assistant Editor.
1. Import the `Mapbox` module.
1. Connect the map view to a new outlet in your view controller class. (Control-drag from the map view in Interface Builder to a valid location in your view controller implementation.) The resulting outlet declaration should look something like this:

```objc
// ViewController.m
@import Mapbox;

@interface ViewController : UIViewController

@property (strong) IBOutlet MGLMapView *mapView;

@end
```

```swift
// ViewController.swift
import Mapbox

class ViewController: UIViewController {
    @IBOutlet var mapView: MGLMapView!
}
```
