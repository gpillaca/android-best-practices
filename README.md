# Best practices in Android develoment

Lessons learned from Android developers in Futurice. Avoid reinventing the wheel by following these guidelines.

Feedback is welcomed, but check the [guidelines](https://github.com/futurice/android-best-practices/tree/master/CONTRIBUTING.md) first.

## Summary

#### Use Gradle and its recommended project structure
#### Put passwords and sensitive data in gradle.properties
#### Don't write your own HTTP client, use Volley or Retrofit libraries
#### Use the Jackson library to parse JSON data
#### Use Volley or Retrofit+OkHttp+Picasso for networking and images
#### Avoid Guava and use only a few libraries due to the *65k method limit*
#### Use Fragments to represent a UI screen
#### Use Activities just to manage Fragments
#### Layout XMLs are code, organize them well
#### Use styles to avoid duplicate attributes in layout XMLs
#### Use multiple style files to avoid a single huge one
#### Keep your colors.xml short and DRY, just define the palette
#### Also keep dimens.xml DRY, define generic constants
#### Do not make a deep hierarchy of ViewGroups
#### Avoid client-side processing for WebViews, and beware of leaks
#### Avoid testing with Robolectric on Activities, Fragments, and Views


----------

### Android SDK

Place your [Android SDK](https://developer.android.com/sdk/installing/index.html?pkg=tools) somewhere in your home directory or some other application-independent location. Some IDEs include the SDK when installed, and may place it under the same directory as the IDE. This can be bad when you need to upgrade (or reinstall) the IDE, or when changing IDEs. Also avoid putting the SDK in another system-level directory that might need sudo permissions, if your IDE is running under your user and not under root.

### Build system

Your default option should be [Gradle](http://tools.android.com/tech-docs/new-build-system). Ant is much more limited and also more verbose. With Gradle, it's simple to:

- Build different flavours or variants of your app
- Make simple script-like tasks
- Manage and download dependencies
- Customize keystores
- And more

Android's Gradle plugin is also being actively developed by Google while as the new standard build system.

### Project structure

There are two popular options: the old Ant & Eclipse ADT project structure, and the new Gradle & Android Studio project structure. You can make the decision whether to use one or the other, although we recommend the new project structure. If using the old, try to provide also a Gradle configuration `build.gradle`.

Old structure:

```
old-structure
├─ assets
├─ libs
├─ res
├─ src
│  └─ com/futurice/project
├─ AndroidManifest.xml
├─ build.gradle
├─ project.properties
└─ proguard-rules.pro
```

New structure:

```
new-structure
├─ library-foobar
├─ app
│  ├─ libs
│  ├─ src
│  │  ├─ androidTest
│  │  │  └─ java
│  │  │     └─ com/futurice/project
│  │  └─ main
│  │     ├─ java
│  │     │  └─ com/futurice/project
│  │     ├─ res
│  │     └─ AndroidManifest.xml
│  ├─ build.gradle
│  └─ proguard-rules.pro
├─ build.gradle
└─ settings.gradle
```

The main difference is that the new structure explicitly separates 'source sets' (`main`, `androidTest`), a concept from Gradle. You could, for instance, add source sets 'paid' and 'free' into `src` which will have source code for the paid and free flavours of your app.

Having a top-level `app` is useful to distinguish your app from other library projects (e.g., `library-foobar`) that will be referenced in your app. The `settings.gradle` then keeps references to these library projects, which `app/build.gradle` can reference to.

### Gradle configuration

**General structure.** Follow [Google's guide on Gradle for Android](http://tools.android.com/tech-docs/new-build-system/user-guide)

**Small tasks.** Instead of (shell, Python, Perl, etc) scripts, you can make tasks in Gradle. Just follow [Gradle's documentation](http://www.gradle.org/docs/current/userguide/userguide_single.html#N10CBF) for more details.

**Passwords.** In your app's `build.gradle` you will need to define the `signingConfigs` for the release build. Here is what you should avoid:

_Don't do this_. This would appear in the version control system.

```groovy
signingConfigs {
    release {
        storeFile file("myapp.keystore")
        storePassword "password123"
        keyAlias "thekey"
        keyPassword "password789"
    }
}
```

Instead, make a `gradle.properties` file which should _not_ be added to the version control system:

```
KEYSTORE_PASSWORD=password123
KEY_PASSWORD=password789
```

That file is automatically imported by gradle, so you can use it in `build.gradle` as such:

```groovy
signingConfigs {
    release {
        try {
            storeFile file("myapp.keystore")
            storePassword KEYSTORE_PASSWORD
            keyAlias "thekey"
            keyPassword KEY_PASSWORD
        }
        catch (ex) {
            throw new InvalidUserDataException("You should define KEYSTORE_PASSWORD and KEY_PASSWORD in gradle.properties.")
        }
    }
}
```

**Prefer Maven dependency resolution instead of importing jar files.** If you explicitly include jar files in your project, they will be of some specific frozen version, such as `2.1.1`. Downloading jars and handling updates is cumbersome, this is a problem that Maven solves properly, and is also encouraged in Android Gradle builds. You can specify a range of versions, such as `2.1.+` and Maven will handle the automatic update to the most recent version matching that pattern. Example:

```groovy
dependencies {
    compile 'com.netflix.rxjava:rxjava-core:0.19.+'
    compile 'com.netflix.rxjava:rxjava-android:0.19.+'
    compile 'com.fasterxml.jackson.core:jackson-databind:2.4.+'
    compile 'com.fasterxml.jackson.core:jackson-core:2.4.+'
    compile 'com.fasterxml.jackson.core:jackson-annotations:2.4.+'
    compile 'com.squareup.okhttp:okhttp:2.0.+'
    compile 'com.squareup.okhttp:okhttp-urlconnection:2.0.+'
}
```

### IDEs and text editors

**Use whatever editor, but it must play nicely with the project structure.** Editors are a personal choice, and it's your responsibility to get your editor functioning according to the project structure and build system.

The most recommended IDE at the moment is [Android Studio](https://developer.android.com/sdk/installing/studio.html), because it is developed by Google, is closest to Gradle, uses the new project structure by default, is finally in beta stage, and is tailored for Android development.

You can use [Eclipse ADT](https://developer.android.com/sdk/installing/index.html?pkg=adt) if you wish, but you need to configure it, since it expects the old project structure and Ant for building. You can even use a plain text editor like Vim, Sublime Text, or Emacs. In that case, you will need to use Gradle and `adb` on the command line. If Eclipse's integration with Gradle is not working for you, your options are using the command line just to build, or migrating to Android Studio.

Whatever you use, just make sure Gradle and the new project structure remain as the official way of building the application, and avoid adding your editor-specific configuration files to the version control system. For instance, avoid adding an Ant `build.xml` file. Especially don't forget to keep `build.gradle` up-to-date and functioning if you are changing build configurations in Ant. Also, be kind to other developers, don't force them to change their tool of preference.

### Libraries

**[Jackson](http://wiki.fasterxml.com/JacksonHome)** is a Java library for converting Objects into JSON and vice-versa. [Gson](https://code.google.com/p/google-gson/) is a popular choice for solving this problem, however we find Jackson to be more performant since it supports alternative ways of processing JSON: streaming, in-memory tree model, and traditional JSON-POJO data binding. Other alternatives: [Json-smart](https://code.google.com/p/json-smart/) and [Boon JSON](https://github.com/RichardHightower/boon/wiki/Boon-JSON-in-five-minutes)

**Networking, caching, and images.** There are a couple of battle-proven solutions for performing requests to backend servers, which you should use perform considering implementing your own client. Use [Volley](https://android.googlesource.com/platform/frameworks/volley) or [Retrofit](http://square.github.io/retrofit/). Volley also provides helpers to load and cache images. If you choose Retrofit, consider [Picasso](http://square.github.io/picasso/) for loading and caching images, and [OkHttp](http://square.github.io/okhttp/) for efficient HTTP requests. All three Retrofit, Picasso and OkHttp are created by the same company, so they complement each other nicely. [OkHttp can also be used in connection with Volley](http://stackoverflow.com/questions/24375043/how-to-implement-android-volley-with-okhttp-2-0/24951835#24951835).

**RxJava** is a library for Reactive Programming, in other words, handling asynchronous events. It is a powerful and promising paradigm, which can also be confusing since it's so different. We recommend to take some caution before using this library to architect the entire application. There are some projects done by us using RxJava, if you need help talk to one of these people: Timo Tuominen, Olli Salonen, Andre Medeiros, Mark Voit, Antti Lammi, Vera Izrailit, Juha Ristolainen. We have written some blog posts on it: [[1]](http://blog.futurice.com/tech-pick-of-the-week-rx-for-net-and-rxjava-for-android), [[2]](http://blog.futurice.com/top-7-tips-for-rxjava-on-android), [[3]](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754), [[4]](http://blog.futurice.com/android-development-has-its-own-swift).

If you have no previous experience with Rx, start by applying it only for responses from the API. Alternatively, start by applying it for simple UI event handling, like click events or typing events on a search field. If you are confident in your Rx skills and want to apply it to the whole architecture, then write Javadocs on all the tricky parts. Keep in mind that another programmer unfamiliar to RxJava might have a very hard time maintaining the project. Do your best to help him understand your code and also Rx.

**[Retrojava](https://github.com/evant/gradle-retrolambda)** is a Java library for using Lambda expression syntax in Android and other pre-JDK8 platforms. It helps keep your code tight and readable especially if you use a functional style with for example with RxJava. To use it, install JDK8, set that as your SDK Location in the Android Studio Project Structure dialog, and set `JAVA8_HOME` and `JAVA7_HOME` environment variables, then in the project root build.gradle:

```groovy
dependencies {
    classpath 'me.tatarka:gradle-retrolambda:2.4.+'
}
```

and in each module's build.gradle, add

```groovy
apply plugin: 'retrolambda'

android {
    compileOptions {
    sourceCompatibility JavaVersion.VERSION_1_8
    targetCompatibility JavaVersion.VERSION_1_8
}

retrolambda {
    jdk System.getenv("JAVA8_HOME")
    oldJdk System.getenv("JAVA7_HOME")
    javaVersion JavaVersion.VERSION_1_7
}
```

Android Studio offers code assist support for Java8 lambdas. If you are new to lambdas, just use the following to get started:

- Any interface with just one method is "lambda friendly" and can be folded into the more tight syntax
- If in doubt about parameters and such, write a normal anon inner class and then let Android Studio fold it into a lambda for you.

**Beware of the dex method limitation, and avoid using many libraries.** Android apps, when packaged as a dex file, have a hard limitation of 65536 referenced methods [[1]](https://medium.com/@rotxed/dex-skys-the-limit-no-65k-methods-is-28e6cb40cf71) [[2]](http://blog.persistent.info/2014/05/per-package-method-counts-for-androids.html) [[3]](http://jakewharton.com/play-services-is-a-monolith/). You will see a fatal error on compilation if you pass the limit. For that reason, use a minimal amount of libraries, and use the [dex-method-counts](https://github.com/mihaip/dex-method-counts) tool to determine which set of libraries can be used in order to stay under the limit. Especially avoid using the Guava library, since it contains over 13k methods.

### Activities and Fragments

[Fragments](http://developer.android.com/guide/components/fragments.html) should be your default option for implementing a UI screen in Android. Fragments are reusable user interfaces that can be composed in your application. We recommend using fragments instead of [activities](http://developer.android.com/guide/components/activities.html) to represent a user interface screen, here are some reasons why:

- **Solution for multi-pane layouts.** Fragments were primarily introduced for extending phone applications to tablet screens, so that you can have both panes A and B on a tablet screen, while either A or B occupy an entire phone screen. If your application is implemented in fragments from the beginning, you will make it easier later to adapt your application to different form-factors.

- **Proper screen-to-screen communication.** Android's API does not provide a proper way of sending complex data (e.g., some Java Object) from one activity to another activity. With fragments, however, you can use the instance of an activity as a channel of communication between its child fragments.

- **Many Android classes expect fragments.** The [API for tabs on the ActionBar](http://developer.android.com/reference/android/app/ActionBar.html#newTab()) expects the tab contents to be fragments. Also, if using Google Maps API v2, the [recommended solution](https://developers.google.com/maps/documentation/android/) is [MapFragment](http://developer.android.com/reference/com/google/android/gms/maps/MapFragment.html), even though [MapView](http://developer.android.com/reference/com/google/android/gms/maps/MapView.html) exists.

- **Easy to implement swiping transitions between screens.** If a UI screen is a fragment in your application, you can use `FragmentPagerAdapter` to contain a collection of fragments and implement smooth and interactive transitioning between them. For instance, horizontal swiping of screens.

- **Intelligent behavior for "back".** The [FragmentManager](http://developer.android.com/reference/android/app/FragmentManager.html) allows you to perform transactions to change the fragments inside an activity. Hence, the FragmentManager manages the "state" of fragments. The back button/action will be handled by the FragmentManager to go back to the previous "fragments state". This is more advanced than the [activity stack](http://developer.android.com/guide/components/tasks-and-back-stack.html).

- **Fragments are generic enough to not be UI-only.** You can have a [fragment without a UI](http://developer.android.com/guide/components/fragments.html#AddingWithoutUI) that works as background workers for the activity. You can take that idea further to create a [fragment to contain the logic for changing fragments](http://stackoverflow.com/questions/12363790/how-many-activities-vs-fragments/12528434#12528434), instead of having that logic in the activity.

- **Even the ActionBar can be managed from within fragments.** You can choose to have one Fragment without a UI with the sole purpose of managing the ActionBar, or you can choose to have each currently visible Fragment add its own action items to the parent Activity's ActionBar. [Read more here](http://www.grokkingandroid.com/adding-action-items-from-within-fragments/).

That being said, we advise not to use [nested fragments](https://developer.android.com/about/versions/android-4.2.html#NestedFragments) extensively, because [matryoshka bugs](http://delyan.me/android-s-matryoshka-problem/) can occur. Use nested fragments only when it makes sense (for instance, fragments in a horizontally-sliding ViewPager inside a screen-like fragment) or if it's a well-informed decision.

On an architectural level, your app should have a top-level activity that contains most of the business-related fragments. You can also have some other supporting activities, as long as their communication with the main activity is simple and can be limited to [`Intent.setData()`](http://developer.android.com/reference/android/content/Intent.html#setData(android.net.Uri)) or [`Intent.setAction()`](http://developer.android.com/reference/android/content/Intent.html#setAction(java.lang.String)) or similar.

### Java packages architecture

Java architectures for Android applications can be roughly approximated in [Model-View-Controller](http://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller). In Android, [Fragment and Activity are actually controller classes](http://www.informit.com/articles/article.aspx?p=2126865). On the other hand, they are explicity part of the user interface, hence are also views.

For this reason, it is hard to classify fragments (or activities) as strictly controllers or views. It's better to let them stay in their own `fragments` package. Activities can stay on the top-level package as long as you follow the advice of the previous section. If you are planning to have more than 2 or 3 activities, then make also an `activities` package.

Otherwise, the architecture can look like a typical MVC, with a `models` package containing POJOs to be populated through the JSON parser with API responses, and a `views` package containing your custom Views, notifications, action bar views, widgets, etc. Adapters are a gray matter, living between data and views. However, they typically need to export some View via `getView()`, so you can include the `adapters` subpackage inside `views`.

Some controller classes are application-wide and close to the Android system. These can live in a `managers` package. Miscellaneous data processing classes, such as "DateUtils", stay in the `utils` package. Classes that are responsible for interacting with the backend stay in the `network` package.

All in all, ordered from the closest-to-backend to the closest-to-the-user:

```
com.futurice.project
├─ network
├─ models
├─ managers
├─ utils
├─ fragments
└─ views
   ├─ adapters
   ├─ actionbar
   ├─ widgets
   └─ notifications
```

### Resources

**Naming.** Follow the convention of prefixing the type, as in `type_foo_bar.xml`. Examples: `fragment_contact_details.xml`, `view_primary_button.xml`, `activity_main.xml`.

**Organizing layout XMLs.** If you're unsure how to format a layout XML, the following convention may help.

- One attribute per line, indented by 4 spaces
- `android:id` as the first attribute always
- `android:layout_****` attributes at the top
- `style` attribute at the bottom
- Tag closer `/>` on its own line, to facilitate ordering and adding attributes.

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    >

    <TextView
        android:id="@+id/name"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_alignParentRight="true"
        android:text="@string/name"
        style="@style/FancyText"
        />

    <include layout="@layout/reusable_part" />

</LinearLayout>
```

As a rule of thumb, attributes `android:layout_****` should be defined in the layout XML, while other attributes `android:****` should stay in a style XML. This rule has exceptions, but in general works fine. The idea is to keep only layout (positioning, margin, sizing) and content attributes in the layout files, while keeping all appearance details (colors, padding, font) in styles files.

The exceptions are:

- `android:id` should obviously be in the layout files
- `android:orientation` for a `LinearLayout` normally makes more sense in layout files
- `android:text` should be in layout files because it defines content
- Sometimes it will make sense to make a generic style defining `android:layout_width` and `android:layout_height` but by default these should appear in the layout files

**Use styles.** Almost every project needs to properly use styles, because it is very common to have a repeated appearance for a view. At least you should have a common style for most text content in the application, for example:

```xml
<style name="ContentText">
    <item name="android:textSize">@dimen/font_normal</item>
    <item name="android:textColor">@color/basic_black</item>
</style>
```

Applied to TextViews:

```xml
<TextView
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:text="@string/price"
    style="@style/ContentText"
    />
```

You probably will need to do the same for buttons, but don't stop there yet. Go beyond and move a group of related and repeated `android:****` attributes to a common style.

**Split a large style file into other files.** You don't need to have a single `styles.xml` file. Android SDK supports other files out of the box, there is nothing magical about the name `styles`, what matters are the XML tags `<style>` inside the file. Hence you can have files `styles.xml`, `styles_home.xml`, `styles_item_details.xml`, `styles_forms.xml`. Unlike resource directory names which carry some meaning for the build system, filenames in `res/values` can be arbitrary.

**`colors.xml` is a color palette.** There should be nothing else in your `colors.xml` than just a mapping from a color name to an RGBA value. Do not use it to define RGBA values for different types of buttons.

*Don't do this:*

```xml
<resources>
    <color name="button_foreground">#FFFFFF</color>
    <color name="button_background">#2A91BD</color>
    <color name="comment_background_inactive">#5F5F5F</color>
    <color name="comment_background_active">#939393</color>
    <color name="comment_foreground">#FFFFFF</color>
    <color name="comment_foreground_important">#FF9D2F</color>
    ...
    <color name="comment_shadow">#323232</color>
```

You can easily start repeating RGBA values in this format, and that makes it complicated to change a basic color if needed. Also, those definitions are related to some context, like "button" or "comment", and should live in a button style, not in `colors.xml`.

Instead, do this:

```xml
<resources>

    <!-- grayscale -->
    <color name="white"     >#FFFFFF</color>
    <color name="gray_light">#DBDBDB</color>
    <color name="gray"      >#939393</color>
    <color name="gray_dark" >#5F5F5F</color>
    <color name="black"     >#323232</color>

    <!-- basic colors -->
    <color name="green">#27D34D</color>
    <color name="blue">#2A91BD</color>
    <color name="orange">#FF9D2F</color>
    <color name="red">#FF432F</color>

</resources>
```

Ask for this palette from the designer of the application. The names do not need to be color names as "green", "blue", etc. Names such as "brand_primary", "brand_secondary", "brand_negative" are totally acceptable as well. Formatting colors as such will make it easy to change or refactor colors, and also will make it explicit how many different colors are being used. Normally for a aesthetic UI, it is important to reduce the variety of colors being used.

**Treat dimens.xml like colors.xml.** You should also define a "palette" of typical spacing and font sizes, for basically the same purposes as for colors. A good example of a dimens file:

```xml
<resources>

    <!-- font sizes -->
    <dimen name="font_larger">22sp</dimen>
    <dimen name="font_large">18sp</dimen>
    <dimen name="font_normal">15sp</dimen>
    <dimen name="font_small">12sp</dimen>

    <!-- typical spacing between two views -->
    <dimen name="spacing_huge">40dp</dimen>
    <dimen name="spacing_large">24dp</dimen>
    <dimen name="spacing_normal">14dp</dimen>
    <dimen name="spacing_small">10dp</dimen>
    <dimen name="spacing_tiny">4dp</dimen>

    <!-- typical sizes of views -->
    <dimen name="button_height_tall">60dp</dimen>
    <dimen name="button_height_normal">40dp</dimen>
    <dimen name="button_height_short">32dp</dimen>

</resources>
```

You should use the `spacing_****` dimensions for layouting, in margins and paddings, instead of hard-coded values, much like strings are normally treated. This will give a consistent look-and-feel, while making it easier to organize and change styles and layouts.

**Avoid a deep hierarchy of views.** Sometimes you might be tempted to just add yet another LinearLayout, to be able to accomplish an arrangement of views. This kind of situation may occur:

```xml
<LinearLayout
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    >

    <RelativeLayout
        ...
        >

        <LinearLayout
            ...
            >

            <LinearLayout
                ...
                >

                <LinearLayout
                    ...
                    >
                </LinearLayout>

            </LinearLayout>

        </LinearLayout>

    </RelativeLayout>

</LinearLayout>
```

Even if you don't witness this explicitly in a layout file, it might end up happening if you are inflating (in Java) views into other views.

A couple of problems may occur. You might experience performance problems, because there are is a complex UI tree that the processor needs to handle. Another more serious issue is a possibility of [StackOverflowError](http://stackoverflow.com/questions/2762924/java-lang-stackoverflow-error-suspected-too-many-views).

Therefore, try to keep your views hierarchy as flat as possible: learn how to use [RelativeLayout](https://developer.android.com/guide/topics/ui/layout/relative.html), how to [optimize your layouts](http://developer.android.com/training/improving-layouts/optimizing-layout.html) and to use the [`<merge>` tag](http://stackoverflow.com/questions/8834898/what-is-the-purpose-of-androids-merge-tag-in-xml-layouts).

**Beware of problems related to WebViews.** When you must display a web page, for instance for a news article, avoid doing client-side processing to clean the HTML, rather ask for a "*pure*" HTML from the backend programmers. [WebViews can also leak memory](http://stackoverflow.com/questions/3130654/memory-leak-in-webview) when they keep a reference to their Activity, instead of being bound to the ApplicationContext. Avoid using a WebView for simple texts or buttons, prefer TextViews or Buttons.


### Test frameworks

Android SDK's testing framework is still infant, specially regarding UI tests. Android Gradle currently implements a test task called [`connectedAndroidTest`](http://tools.android.com/tech-docs/new-build-system/user-guide#TOC-Testing) which runs JUnit tests that you created, using an [extension of JUnit with helpers for Android](http://developer.android.com/reference/android/test/package-summary.html). This means you will need to run tests connected to a device, or an emulator. Follow the official guide [[1]](http://developer.android.com/tools/testing/testing_android.html) [[2]](http://developer.android.com/tools/testing/activity_test.html) for testing.

**Use [Robolectric](http://robolectric.org/) only for unit tests, not for views.** It is a test framework seeking to provide tests "disconnected from device" for the sake of development speed, suitable specially for unit tests on models and view models. However, testing under Robolectric is inaccurate and incomplete regarding UI tests. You will have problems testing UI elements related to animations, dialogs, etc, and this will be complicated by the fact that you are "walking in the dark" (testing without seeing the screen being controlled).

**[Robotium](https://code.google.com/p/robotium/) makes writing UI tests easy.** You do not need Robotium for running connected tests for UI cases, but it will probably be beneficial to you because of its many helpers to get and analyse views, and control the screen. Test cases will look as simple as:

```java
solo.sendKey(Solo.MENU);
solo.clickOnText("More"); // searches for the first occurence of "More" and clicks on it
solo.clickOnText("Preferences");
solo.clickOnText("Edit File Extensions");
Assert.assertTrue(solo.searchText("rtf"));
```

### Proguard configuration

[ProGuard](http://proguard.sourceforge.net/) is normally used on Android projects to shrink and obfuscate the packaged code.

Whether you are using ProGuard or not depends on your project configuration. Usually you would configure gradle to use ProGuard when building a release apk.

```groovy
buildTypes {
    debug {
        runProguard false
    }
    release {
        signingConfig signingConfigs.release
        runProguard true
        proguardFiles 'proguard-rules.pro'
    }
}
```

In order to determine which code has to be preserved and which code can be discarded or obfuscated, you have to specify one or more entry points to your code. These entry points are typically classes with main methods, applets, midlets, activities, etc.
Android framework uses a default configuration which can be found from `SDK_HOME/tools/proguard/proguard-android.txt`. Custom project-specific proguard rules, as defined in `my-project/app/proguard-rules.pro`, will be appended to the default configuration.

A common problem related to ProGuard is to see the application crashing on startup with `ClassNotFoundException` or `NoSuchFieldException` or similar, even though the build command (i.e. `assembleRelease`) succeeded without warnings.
This means one out of two things:

1. ProGuard has removed the class, enum, method, field or annotation, considering it's not required.
2. ProGuard has obfuscated (renamed) the class, enum or field name, but it's being used indirectly by its original name, i.e. through Java reflection.

Check `app/build/outputs/proguard/release/usage.txt` to see if the object in question has been removed.
Check `app/build/outputs/proguard/release/mapping.txt` to see if the object in question has been obfuscated.

In order to prevent ProGuard from *stripping away* needed classes or class members, add a `keep` options to your proguard config:
```
-keep class com.futurice.project.MyClass { *; }
```

To prevent ProGuard from *obfuscating* classes or class members, add a `keepnames`:
```
-keepnames class com.futurice.project.MyClass { *; }
```

Check [this template's ProGuard config](https://github.com/futurice/android-best-practices/blob/master/templates/rx-architecture/app/proguard-rules.pro) for some examples.
Read more at [Proguard](http://proguard.sourceforge.net/#manual/examples.html) for examples.

**Tip.** Save the `mapping.txt` file for every release that you publish to your users. By retaining a copy of the `mapping.txt` file for each release build, you ensure that you can debug a problem if a user encounters a bug and submits an obfuscated stack trace.

### Thanks to

Antti Lammi, Joni Karppinen, Peter Tackage, Timo Tuominen, Vera Izrailit, Vihtori Mäntylä, Mark Voit, Paul Houghton and other Futurice developers for sharing their knowledge on Android development.

### License

[Futurice Oy](www.futurice.com)
Creative Commons Attribution 4.0 International (CC BY 4.0)
