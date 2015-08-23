---
layout: page
title: surviving force quit
---

You may get "bug reports" that your app stops working in the background when the user "force quits" it, a.k.a. swipes up to close the app from the task switcher (the switcher that appears when you double-tap the home button).

However, in iOS, **force-quitting an app in this way signals the user's intent that your app should not be running**.  Therefore, your app not launching in this case is a feature, not a bug.  If your user wants your app to keep working in the background, they should not force quit it in the app switcher.

**One exception to this rule** is location updates.  Your app may be launched in the background to handle location changes even if the user has force-quit the app.

In general, however, you shouldn't expect your app to keep working in the background when the user has tried to stop it.  The following list of APIs definitely don't work after force-quitting an app:

* Background audio
* Background bluetooth
* Background fetch
* Silent push notifications

Note that "force quitting" an app is different than your app running in the background in other cases.  If your app crashes, or the device reboots, or the user merely pressing the home button, the background APIs continue to work.  But force-quitting an app isn't like that.  Force-quit means the user doesn't want your app to run.  And it won't.

Note that there are some other settings that are similar to force-quit.  Turning off background app refresh in settings, for example, will cause many of the background tasks not to run.