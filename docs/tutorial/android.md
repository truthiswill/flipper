---
id: android
title: Building an Android Plugin
---

<img align="right" src="/docs/assets/android-tutorial-app.png" alt="Android Tutorial App" width="200">

For the purpose of the tutorial, we will assume you already have an existing
Android application in which you have a feed or list of items. As the Flipper
team, we obviously concern ourselves mostly with sea mammals, so this is what
our app displays. The actual display logic is not what's interesting here,
but how we can make this data available in our Flipper desktop app.

Part of what makes Flipper so useful is allowing users to inspect the
internals of their app. In this case, we'd like to see the specific
sea mammal data the app is handling, so let's write a plugin to make that
easy.

You can find the source code of the project [on GitHub](
https://github.com/facebook/flipper/tree/7dae5771d96ea76b75796d3b3a2c78746e581e3f/android/tutorial).

## Creating a Plugin

On Android, a Flipper plugin is a class that implements the
[`FlipperPlugin`](https://github.com/facebook/flipper/blob/master/android/src/main/java/com/facebook/flipper/core/FlipperPlugin.java)
interface.

The interface is rather small and only comprises four methods:

- `getId() -> String`: Specify a unique string so the JavaScript side knows where to talk to. This must match the name attribute in the `package.json` we will look into later in this tutorial.
- `onConnect(FlipperConnection)`: This method is called when the desktop app connects to the mobile client and is ready to receive or send data.
- `onDisconnect()`: We're sure you can figure this one out.
- `runInBackground() -> Boolean`: Unless this is true, only the currently selected plugin in the Flipper UI can communicate with the device.

Let's implement these methods for our sealife app:

```kotlin
import com.facebook.flipper.core.FlipperConnection
import com.facebook.flipper.core.FlipperObject
import com.facebook.flipper.core.FlipperPlugin
import com.facebook.flipper.sample.tutorial.MarineMammals

class SeaMammalFlipperPlugin : FlipperPlugin {
    private var connection: FlipperConnection? = null

    override fun getId(): String = "sea-mammals"

    override fun onConnect(connection: FlipperConnection?) {
        this.connection = connection

        MarineMammals.list.mapIndexed { index, (title, picture_url) ->
            FlipperObject.Builder()
                    .put("id", index)
                    .put("title", title)
                    .put("url", picture_url).build()
        }.forEach(this::newRow)
    }

    override fun onDisconnect() {
        connection = null
    }

    override fun runInBackground(): Boolean = false

    private fun newRow(row: FlipperObject) {
        connection?.send("newRow", row)
    }
}
```
*See [SeaMammalFlipperPlugin.kt](https://github.com/facebook/flipper/blob/5afb148ffa9e267e5b24e0dfae198d1cf46cc396/android/tutorial/src/main/java/com/facebook/flipper/sample/tutorial/plugin/SeaMammalFlipperPlugin.kt)*

The two interesting bits here are `onConnect` and `newRow`. `newRow` sends a message
to the desktop app and is identified with the same name "newRow".

For our sample app, we're dealing with a static data source. However, in real
life, you will likely dynamically receive new data as the user interacts with
the app. So while we just send all the data we have at once in `onConnect`,
you would normally set up a listener here to instead call `newRow` as new data
arrives.

You may have noticed that we don't just send random `Object`s over the wire but
use `FlipperObject`s instead. What are they? A `FlipperObject` works similar
to a JSON dictionary and has a limited set of supported types like strings,
integers and arrays. Before sending an object from your native app to the
desktop, you first need to break it down into `FlipperObject`-serializable parts.

## Registering your Plugin

Now all you need to do is let Flipper know about your new plugin. You do this
by calling `addPlugin` on your `FlipperClient`, which is normally created
at application startup.

```kotlin
val flipperClient = AndroidFlipperClient.getInstance(this)
// Add all sorts of other amazing plugins here ...
flipperClient.addPlugin(SeaMammalFlipperPlugin())
flipperClient.start()
```
*See [`TutorialApplication.kt`](https://github.com/facebook/flipper/blob/5afb148ffa9e267e5b24e0dfae198d1cf46cc396/android/tutorial/src/main/java/com/facebook/flipper/sample/tutorial/TutorialApplication.kt)*

## What next

When starting your application now, Flipper will tell the desktop application
about the plugin it supports, including "sea-mammals" and will look for a
corresponding JavaScript plugin by that name. Let's build that next.