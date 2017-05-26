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

When I would navigate to a page, it would automatically up scroll off the first half inch to inch on some pages. I used `git bisect` to narrow it down to 5 commits where we had upgraded our support library. From there I tried my best to get a minimum test case where I could reproduce the error on a particular page. I deleted code piece by piece until I found a small chunk of code that I could comment out and reveal or hide the error. When there was 1 element in the recycler view, it would not autoscroll. When there was 2, it would. I pointed to this out to my coworker to discuss this odd behavoir. And we did a google search and found this https://stackoverflow.com/a/40492486 . Turns out it was a feature added in the support library upgrade.
