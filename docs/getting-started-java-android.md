# Getting started with Android

In addition to the requirements from our [Getting started with Java](getting-started-java.md) guide you'll also need:

* Xamarin.Android 7.4.99 or later (build from [Jenkins](https://jenkins.mono-project.com/view/Xamarin.Android/job/xamarin-android/lastSuccessfulBuild/Azure/))
* Android Studio 2.3.2 or later (with [Java 1.8](https://developer.android.com/guide/platform/j8-jack.html))

*NOTE: the state of using Java 1.8 in Android Studio is currently in [flux](https://android-developers.googleblog.com/2017/03/future-of-java-8-language-feature.html) at the moment. At the time of writing, the options are to enable the Jack toolchain in your project or use a preview version of Android Studio.*

As an overview, we will:
* Clone Embeddinator-4000
* Run Cake build script (downloads a Jenkins build Xamarin.Android)
* Create a C# Android Library project
* Run Embeddinator on the Android library assembly
* Use the generated AAR file in a Java project in Android Studio

## Building Embeddinator-4000

Clone the source of Embeddinator 4000 into an easy-to-find directory such as `~/Projects/Embeddinator-4000`. You may also need to run `git submodule init` and `git submodule update`.

To get started, the best option right now is to run the Cake script:
```
./build.sh -t Generate-Android -v diagnostic
```
On Windows, in Powershell:
```
.\build.ps1 -t Generate-Android -v diagnostic
```
This will download a master build of Xamarin.Android and extract it into `/external/Xamarin.Android`. 

`MonoEmbeddinator4000.exe` will be compiled to `build/lib/Release`. The Cake script will also run Embeddinator against a test assembly, so you can be sure your system is setup properly.

*NOTE: as soon as a preview build containing our changes for Xamarin.Android is available, we will update these instructions. Follow the [previous guide](getting-started-java.md) for details about building Embeddinator-4000.*

## Create an Android Library Project

Open Visual Studio for Windows or Mac, create a new Android Class Library project, name it `hello-from-csharp`, and save it to `~/Projects/hello-from-csharp` or `%USERPROFILE%\Projects\hello-from-csharp`.

Add a new Android Activity named `HelloActivity.cs`, followed by an Android Layout at `Resource/layout/hello.axml`.

Add a new `TextView` to your layout, and change the text to something enjoyable.

Your layout source should look something like this:
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:minWidth="25px"
    android:minHeight="25px">
    <TextView
        android:text="Hello from C#!"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:gravity="center" />
</LinearLayout>
```

In your activity, make sure you are calling `SetContentView` with your new layout:
```csharp
[Activity(Label = "HelloActivity"), 
    Register("hello_from_csharp.HelloActivity")]
public class HelloActivity : Activity
{
    protected override void OnCreate(Bundle savedInstanceState)
    {
        base.OnCreate(savedInstanceState);

        SetContentView(Resource.Layout.hello);
    }
}
```
*NOTE: don't forget the `[Register]` attribute, see details under Limitations*

Build the project, the resulting assembly will be saved as `$(ProjectDir)/bin/Debug/hello-from-csharp.dll`.

## Run Embeddinator-4000

Run the embeddinator to create a native AAR file for the Android library project assembly.

On OSX:
```shell
cd ~/Projects/Embeddinator-4000
mono build/lib/Release/MonoEmbeddinator4000.exe ~/Projects/hello-from-csharp/hello-from-csharp/bin/Debug/hello-from-csharp.dll --gen=Java --platform=Android --outdir=../hello-from-csharp/output -c
```

Or on Windows, in cmd:
```
cd %USERPROFILE%\Projects\Embeddinator-4000
build\lib\Release\MonoEmbeddinator4000.exe %USERPROFILE%\Projects\hello-from-csharp\hello-from-csharp\bin\Debug\hello-from-csharp.dll --gen=Java --platform=Android --outdir=..\hello-from-csharp\output -c
```

The Android AAR file will be placed in `~/Projects/hello-from-csharp/output/hello_from_csharp.aar`. _NOTE: hyphens are replaced because Java does not support it in package names._

## Use the generated output in an Android Studio project

Open Android Studio and create a new project with an `Empty Activity`.

Right-click on your `app` module and choose `New | Module`. Select `Import .JAR/.AAR Package`. Use the directory browser to locate `~/Projects/hello-from-csharp/output/hello_from_csharp.aar` and hit `Finish`.

![Import AAR into Android Studio](AndroidStudioImport.png)

This will copy the AAR file into a new module named `hello_from_csharp`.

![Android Studio Project](AndroidStudioProject.png)

To use the new module from your `app`, right-click and choose `Open Module Settings`. On the `Dependencies` tab, add a new `Module Dependency` and choose `:hello_from_csharp`.

![Android Studio Dependencies](AndroidStudioDependencies.png)

In your activity, add a new `onResume` method, and let's do the easiest thing to launch our C# activity:
```java
import hello_from_csharp.*;

public class MainActivity extends AppCompatActivity {
    //... Other stuff here ...
    @Override
    protected void onResume() {
        super.onResume();

        Intent intent = new Intent(this, HelloActivity.class);
        startActivity(intent);
    }
}
```

### Assembly Compression *IMPORTANT*
One further change is required for Embeddinator in your Android Studio project.

Open your app's `build.gradle` file and add the following change:
```gradle
android {
    // ... Other stuff here ...
    aaptOptions {
        noCompress 'dll'
    }
}
```
Xamarin.Android currently loads .NET assemblies directly from the APK, but it requires the assemblies to not be compressed.

If you do not have this setup, the app will crash on launch and print something like this to the console:
```
com.xamarin.hellocsharp A/monodroid: No assemblies found in '(null)' or '<unavailable>'. Assuming this is part of Fast Deployment. Exiting...
```

## Run the app
Upon launching your app:

![Hello from C# sample running in the emulator](hello-from-csharp-android.png)

Note what happened here:
* We have a C# class, `HelloActivity`, that subclasses Java
* We have Android Resource files
* We used these from Java in Android Studio

So for this sample to work, all the following are setup in the final APK:
* Xamarin.Android is configured on application start
* .NET assemblies included in `assets/assemblies`
* `AndroidManifest.xml` modifications for your C# activities, etc.
* Android Resources and Assets from .NET libraries
* [Android Callable Wrappers](https://developer.xamarin.com/guides/android/advanced_topics/java_integration_overview/android_callable_wrappers/) for any `Java.Lang.Object` subclass

## Current Limitations on Android

Right now if you subclass `Java.Lang.Object`, Xamarin.Android will generate the Java stub (Android Callable Wrapper) instead of Embeddinator.

So you must follow the same rules for exporting C# to Java as Xamarin.Android. 

So for example in C#:
```csharp
    [Register("mono.embeddinator.android.ViewSubclass")]
    public class ViewSubclass : TextView
    {
        public ViewSubclass(Context context) : base(context) { }

        [Export("apply")]
        public void Apply(string text)
        {
            Text = text;
        }
    }
```

* `[Register]` is required to map to a desired Java package name
* `[Export]` is required to make a method visible to Java

We can use `ViewSubclass` in Java like so:
```java
import mono.embeddinator.android.ViewSubclass;
//...
ViewSubclass v = new ViewSubclass(this);
v.apply("Hello");
```

Read more about Java integration with Xamarin.Android [here](https://developer.xamarin.com/guides/android/advanced_topics/java_integration_overview/).

## Further Reading

* [Preliminary Android Research](android-preliminary-research.md)
* [Embeddinator Limitations](Limitations.md)
* [Contributing to the open source project](Contributing.md)
* [Error codes and descriptions](errors.md)