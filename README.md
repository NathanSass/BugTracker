# BugTracker

### Jellybean removing padding between elements in a recycler view

I had a recycler view with gridded layout manager full of custom views. The padding between items was removed on jellybean. At first I thought it had something to do with the constraint layouts not being defined correctly. I then added some more constraints. Then I thought it had something to do with the nesting of the constraint layouts and the fact that they had a custom dimension. So instead of using `layout_constraintDimensionRatio` I overrode `onMeasure` and set their size dimensions there. This also didn't work. Since this was just a jellybean only bug and we were planning to phase out support over the next few weeks I decided to just programatically add in the padding. It was a little hacky but we were ok with it. There were a couple of issues around this, such as being unable to read padding from the styled typed array so the padding would have to be set statically. Which although not great, was ok because we only had one use of the view in xml in the app. As I debugged this smaller issue, I noticed some strange behavoir of where the `setPadding` method call needed to be placed. My first guess that when I inflated my custom xml, it was blowing out the padding. Suprisingly, this was wrong - I used the debugger and stepped through the code and found out that strangely after `View.setBackgroundResource` was called, there was no more padding. A quick stackoverflow search revealed this to be a known bug. I made a utility method to replace padding for jelly bean instances.

```
  public static void setBackgroundResourceCompat(@NonNull int backgroundResource, @NonNull View view) {
        // hold reference to the padding
        int leftPadding = view.getPaddingLeft();
        int topPadding = view.getPaddingTop();
        int rightPadding = view.getPaddingRight();
        int bottomPadding = view.getPaddingBottom();

        // set the background resource, padding will be removed in JellyBean
        view.setBackgroundResource(backgroundResource);

        if (!SharedUtils.hasKitKat()) {
            // replace the padding removed by a bug in JellyBean
            view.setPadding(leftPadding, topPadding, rightPadding, bottomPadding);
        }
    }
```

### Automatic scrolling in Recycler View

When I would navigate to a page, it would automatically up scroll off the first half inch to inch on some pages. I used `git bisect` to narrow it down to 5 commits where we had upgraded our support library. From there I tried my best to get a minimum test case where I could reproduce the error on a particular page. I deleted code piece by piece until I found a small chunk of code that I could comment out and reveal or hide the error. When there was 1 element in the recycler view, it would not autoscroll. When there was 2, it would. I pointed to this out to my coworker to discuss this odd behavoir. And we did a google search and found this https://stackoverflow.com/a/40492486 . Turns out it was a feature added in the support library upgrade. I then looked through the source code to verify it.

### Crash when tapping on a menu item

One activity is used as an entry way for another activites launch. The first activity's action bar was visible for a split second as the second activity was launched. If you have quick fingers or a slow device you could tap on the first item's menu. This would result in a crash. There was a lot of legacy code involved and likely some suspect architectural patterns. The crash indicated an Illegal State Exception when a progress dialog was trying to be shown. This was a case where Activity was null. The fix was relatively easy. Since it wasn't important to manage the state of this dialog, I could allow the state to be lost and supress the error by using an alternate configuration to launch the progress dialog. I found these 2 blog posts quite interesting for educating myself on this issue.

https://medium.com/inloop/demystifying-androids-commitallowingstateloss-cb9011a544cc

https://medium.com/@bherbst/the-many-flavors-of-commit-186608a015b1

### setRetainInstance(true) & FragmentStatePagerAdapter don't play well together

```
Fatal Exception: java.lang.RuntimeException: Unable to start activity ComponentInfo{com.app.reader0/com.app.ui.MainMenuActivity}: java.lang.IllegalStateException: Could not find active fragment with index -1
       at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2224)
       at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:2273)
       at android.app.ActivityThread.handleRelaunchActivity(ActivityThread.java:3764)
       at android.app.ActivityThread.access$900(ActivityThread.java:138)
       at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1242)
       at android.os.Handler.dispatchMessage(Handler.java:102)
       at android.os.Looper.loop(Looper.java:149)
       at android.app.ActivityThread.main(ActivityThread.java:5045)
       at java.lang.reflect.Method.invokeNative(Method.java)
       at java.lang.reflect.Method.invoke(Method.java:515)
       at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:794)
       at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:610)
       at dalvik.system.NativeStart.main(NativeStart.java)
Caused by java.lang.IllegalStateException: Could not find active fragment with index -1
       at android.support.v4.app.FragmentManagerImpl.restoreAllState(Scribd:3026)
       at android.support.v4.app.Fragment.restoreChildFragmentState(Scribd:1436)
```

