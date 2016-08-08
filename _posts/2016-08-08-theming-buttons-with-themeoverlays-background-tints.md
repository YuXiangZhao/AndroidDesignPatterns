---
layout: post
title: 'Coloring Buttons w/ ThemeOverlays & Background Tints'
date: 2016-08-07
permalink: /2016/08/coloring-buttons-with-themeoverlays-background-tints.html
related: ['/2013/08/fragment-transaction-commit-state-loss.html',
    '/2013/04/retaining-objects-across-config-changes.html',
    '/2016/08/contextcompat-getcolor-getdrawable.html']
---

<!--morestart-->

Let's say you want to change the background color of a regular old `Button`.
How should this be done? 

This blog post covers two different approaches. In the first approach,
we'll modify the `R.attr.colorButtonNormal` and `R.attr.colorAccent` attributes directly
using a custom theme, and in the second, we'll make use of AppCompat's built-in background tinting support
to achieve an identical effect.

<!--more-->

### Approach #1: Modifying the button's background color using a custom theme

Before we get too far ahead of ourselves, we should first understand how the
background color of a button is actually
determined. The [material design spec](http://material.google.com/components/buttons.html)
has very specific requirements about what a button should look like in light
and dark themes. How are these requirements met under-the-hood?

#### Understanding `R.attr.colorButtonNormal` & `R.attr.colorAccent`

You probably know that AppCompat injects its own widgets in place of many framework
widgets, giving AppCompat greater control over tinting widgets according to the material design
spec even on pre-Lollipop devices. At runtime, `Button`s will become [`AppCompatButton`][AppCompatButton]s,
so with that in mind let's do a bit of source code digging to figure out how the button's background color
is actually determined on pre-Lollipop devices:

1.  The default style applied to `AppCompatButton`s is the style pointed to by
    the [`R.attr.buttonStyle`][R.attr.buttonStyle] theme attribute.

2.  ...which is declared in [`Base.V7.Theme.AppCompat`][Base.V7.Theme.AppCompat].

3.  ...which points to [`@style/Widget.AppCompat.Button`][Widget.AppCompat.Button].

4.  ...which extends [`@style/Base.Widget.AppCompat.Button`][Base.Widget.AppCompat.Button].

5.  ...which uses [`@drawable/abc_btn_default_mtrl_shape`][@drawable/abc_btn_default_mtrl_shape]
    as the view's default background drawable.

6.  Now, let's take a look at AppCompat's internal [`AppCompatDrawableManager`][AppCompatDrawableManager] class,
    which contains most of the logic that determines how these widgets are tinted at runtime.
    `AppCompatButton` backgrounds are tinted using one of two
    predefined default `ColorStateList`s, which we can see by analyzing the
    source code [here][AppCompatDrawableManagerSource1]:<sup><a href="#footnote1" id="ref1">1</a></sup>

    ```java
    ...
    } else if (resId == R.drawable.abc_btn_default_mtrl_shape
            || resId == R.drawable.abc_btn_borderless_material) {
        tint = createDefaultButtonColorStateList(context);
    } else if (resId == R.drawable.abc_btn_colored_material) {
        tint = createColoredButtonColorStateList(context);
    }
    ...
    ```

    and [here][AppCompatDrawableManagerSource2]:

    ```java
    private ColorStateList createDefaultButtonColorStateList(Context context) {
        return createButtonColorStateList(context, R.attr.colorButtonNormal);
    }

    private ColorStateList createColoredButtonColorStateList(Context context) {
        return createButtonColorStateList(context, R.attr.colorAccent);
    }

    private ColorStateList createButtonColorStateList(Context context, int baseColorAttr) {
        final int[][] states = new int[4][];
        final int[] colors = new int[4];
        int i = 0;

        final int baseColor = getThemeAttrColor(context, baseColorAttr);
        final int colorControlHighlight = getThemeAttrColor(context, R.attr.colorControlHighlight);

        // Disabled state
        states[i] = ThemeUtils.DISABLED_STATE_SET;
        colors[i] = getDisabledThemeAttrColor(context, R.attr.colorButtonNormal);
        i++;

        states[i] = ThemeUtils.PRESSED_STATE_SET;
        colors[i] = ColorUtils.compositeColors(colorControlHighlight, baseColor);
        i++;

        states[i] = ThemeUtils.FOCUSED_STATE_SET;
        colors[i] = ColorUtils.compositeColors(colorControlHighlight, baseColor);
        i++;

        // Default enabled state
        states[i] = ThemeUtils.EMPTY_STATE_SET;
        colors[i] = baseColor;
        i++;

        return new ColorStateList(states, colors);
    }
    ```

We can follow a similar rabbit hole to determine how buttons are colored on
Lollipop and above... **TODO(alockwood): explain API 21+???**

That's a lot to take in! To summarize, the important parts to 
understand from all of this can be summed up with the following:

> Let `S` be the style resource that is assigned to button `B`. Note that if no
> style is provided in the client's XML code, AppCompat uses the style resource
> pointed to by theme attribute `R.attr.buttonStyle` (which by default is
> `@style/Widget.AppCompat.Button` for AppCompat-based themes).
>
> If `S` is the default `@style/Widget.AppCompat.Button` style, then the
> background is tinted using the color resource pointed to by theme attribute
> `R.attr.colorButtonNormal`.
>
> If `S` is the `@style/Widget.AppCompat.Button.Colored` style (i.e. the style
> that sets `R.drawable.abc_btn_colored_material` as the button's background
> drawable), then the background is tinted using the color resource pointed to
> by theme attribute `R.attr.colorAccent`.

Got all that? Good! Because unfortunately there isn't a great deal of documentation
on this at the moment<sup><a href="#footnote2" id="ref2">2</a></sup>... so don't forget!
In any case, hopefully this in-depth source code digging demonstration has been useful...
now let's get back to the issue we started out trying to solve in the first place. :)

#### Understanding `android:theme` & `ThemeOverlay`s

So now we know that button backgrounds are tinted using the color resource
pointed to by the `R.attr.colorButtonNormal` theme attribute. One way we could
update the value specified by this theme attribute is by modifying the
application's theme directly. This is rarely desired however, since most of the
time we only want to change the background color of a single button in our app.
Modifying the theme attribute at the application level will change the
background color of *all buttons in the entire application*.

Instead, a much better solution is to assign the button its own custom theme in
XML using `android:theme`. Let's say we want to change the button's background
color to Google Red 500 and its text color to 100% white instead of 87% black. To
achieve this, we can define the following theme:

```xml
<!-- res/values/themes.xml -->
<style name="LightCustomRedButtonTheme" parent="ThemeOverlay.AppCompat.Light">
    <item name="colorButtonNormal">@color/googred500</item>
    <item name="android:textColor">?android:textColorPrimaryInverse</item>
</style>
```

and set it on the button in the layout XML:

```xml
<Button
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:text="Light themed custom red button"
    android:theme="@style/LightCustomRedButtonTheme"/>
```

And that's it!<sup><a href="#footnote3" id="ref3">3</a></sup>

You're probably still wondering what's up with that weird
`ThemeOverlay` though. Unlike the themes we use in our `AndroidManifest.xml`
files (i.e. `Theme.AppCompat.Light`, `Theme.AppCompat.Dark`, etc.),
`ThemeOverlay`s define only a small set of material-styled theme attributes that
are most often used when theming each view's appearance (see the
[source code for a complete list of these attributes][ThemeOverlayAttributes].
As a result, they are very useful in cases where you only want to modify one or
two properties of a particular view: just extend the `ThemeOverlay`, update the
attributes you want to modify with their new values, and you can be sure that
your view will still inherit all of the correct light/dark themed values that
would have otherwise been used by default. If you want to read more about
`ThemeOverlay`s, check out [this Medium post][ThemeOverlayBlogPost] and this
[Google+ pro tip][ThemeOverlayProTip] by [Ian Lake](http://google.com/+IanLake)!

### Approach #2: Setting the `AppCompatButton`'s background tint

If you've made it this far, you'll be happy to know that there is an *even easier and more powerful*
way to color a button's background using a feature in AppCompat known as
background tints. Any AppCompat widget that implements the [`TintableBackgroundView`][TintableBackgroundView]
interface (i.e. `AppCompatButton`, `AppCompatImageView`, etc.) can have its
background tint color changed either via XML:

```xml
<android.support.v7.widget.AppCompatButton
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    app:backgroundTint="@color/googred500"/>
```

or programatically via the [`ViewCompat#setBackgroundTintList(View, ColorStateList)`][ViewCompat#setBackgroundTintList()]
method:<sup><a href="#footnote4" id="ref4">4</a></sup>

```java
int googRed500 = ContextCompat.getColor(context, R.color.googred500);
ViewCompat.setBackgroundTintList(button, ColorStateList.valueOf(googRed500));
```

**TODO(alockwood): FINISH OFF BLOG POST WITH EXPLANATION ABOUT WHEN EACH APPROACH IS USEFUL!
(example: using a theme overlay to modify `android:colorEdgeEffect`)**

### Pop quiz!

Let's test our knowledge of how this all works with a simple example.
Consider a sample app that sets the following theme in its `AndroidManifest.xml`:

```xml
<!-- res/values/themes.xml -->
<style name="AppTheme" parent="Theme.AppCompat.Light.DarkActionBar">
    <item name="colorPrimary">@color/indigo500</item>
    <item name="colorPrimaryDark">@color/indigo700</item>
    <item name="colorAccent">@color/pinkA200</item>
</style>
```

In addition to this, the following custom themes are declared as well:

```xml
<!-- res/values/themes.xml -->
<style name="LightCustomRedButtonTheme" parent="ThemeOverlay.AppCompat.Light">
    <item name="colorButtonNormal">@color/googred500</item>
    <item name="android:textColor">?android:textColorPrimaryInverse</item>
</style>

<style name="DarkCustomRedButtonTheme" parent="ThemeOverlay.AppCompat.Dark">
    <item name="colorButtonNormal">@color/googred500</item>
    <item name="android:textColor">?android:textColorPrimaryInverse</item>
</style>
```

What will the following XML look like in the application?

```xml
<LinearLayout
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="vertical">

    <Button
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="..."/>

    <Button
        style="@style/Widget.AppCompat.Button.Colored"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="..."/>

    <Button
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="..."
        android:theme="@style/LightCustomRedButtonTheme"/>

    <android.support.v7.widget.AppCompatButton
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="..."
        android:textColor="?android:textColorPrimaryInverse"
        app:backgroundTint="@color/quantum_googred500"/>

    <Button
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="..."
        android:theme="@style/ThemeOverlay.AppCompat.Dark"/>

    <Button
        style="@style/Widget.AppCompat.Button.Colored"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="..."
        android:theme="@style/ThemeOverlay.AppCompat.Dark"/>

    <Button
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="..."
        android:theme="@style/DarkCustomRedButtonTheme"/>

    <android.support.v7.widget.AppCompatButton
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="..."
        android:textColor="?android:textColorPrimary"
        app:backgroundTint="@color/quantum_googred500"/>
</LinearLayout>
```

#### Solutions

Here are the screenshots of what it looks like when the buttons are put in
enabled and disabled states:

<div style="display: block;">
  <div style="float:left; margin-right:16px;">
    <a href="/assets/images/posts/2016/08/08/rant6-buttons-enabled.png">
      <img alt="Example code solutions, buttons enabled" src="/assets/images/posts/2016/08/08/rant6-buttons-enabled-resized.png"/>
    </a>
  </div>
  <div style="float:left;">
    <a href="/assets/images/posts/2016/08/08/rant6-buttons-disabled.png">
      <img alt="Example code solutions, buttons disabled" src="/assets/images/posts/2016/08/08/rant6-buttons-disabled-resized.png"/>
    </a>
  </div>
</div>

<div style="display: inline-block;">
<p>
As always, thanks for reading! Feel free to leave a comment if you have any questions, and don't forget to +1 and/or share this blog post if you found it helpful! And check out the 
<a href="https://github.com/alexjlockwood/adp-theming-buttons-with-themeoverlays">source code for these examples on GitHub</a> as well!
</p>
</div>

<hr class="footnote-divider"/>
<sup id="footnote1">1</sup> In case you were wondering, `R.drawable.abc_btn_borderless_material` and `R.drawable.abc_btn_colored_material`
are the background drawable resources that AppCompat uses when you apply the 
[`@style/Widget.AppCompat.Button.Colored`][Base.Widget.AppCompat.Button.Colored] and [`@style/Widget.AppCompat.Button.Borderless`][Base.Widget.AppCompat.Button.Borderless]
styles to your `AppCompatButton` respectively. <a href="#ref1" title="Jump to footnote 1.">&#8617;</a>

<sup id="footnote2">2</sup> If you have any links/blogs/documentation on AppCompat in mind that you've found useful in the past,
please feel free to leave a link in the comments below! I'd love to check it out. <a href="#ref2" title="Jump to footnote 2.">&#8617;</a>

<sup id="footnote3">3</sup> An alternative (and arguably more correct) way to theme this button
would be to set [`@style/TextAppearance.AppCompat.Widget.Button.Inverse`][TextAppearance.AppCompat.Widget.Button.Inverse]
as the style's `android:textAppearance` value instead. For simplicity, I decided to just alter the text color directly. :) <a href="#ref3" title="Jump to footnote 3.">&#8617;</a>

<sup id="footnote4">4</sup> Note that AppCompat widgets do not expose a `setBackgroundTintList()` methods as part of their public API.
Clients *must* use the `ViewCompat#setBackgroundTintList()` static helper methods to modify these properties programatically. <a href="#ref4" title="Jump to footnote 4.">&#8617;</a>

  [AppCompatButton]: https://developer.android.com/reference/android/support/v7/widget/AppCompatButton.html
  [AppCompatDrawableManager]: https://github.com/android/platform_frameworks_support/blob/d57359e205b2c04a4f0f0ecf9dcb8d6086e75663/v7/appcompat/src/android/support/v7/widget/AppCompatDrawableManager.java
  [Base.Widget.AppCompat.Button.Colored]: https://github.com/android/platform_frameworks_support/blob/d57359e205b2c04a4f0f0ecf9dcb8d6086e75663/v7/appcompat/res/values/styles_base.xml#L417
  [Base.Widget.AppCompat.Button.Borderless]: https://github.com/android/platform_frameworks_support/blob/d57359e205b2c04a4f0f0ecf9dcb8d6086e75663/v7/appcompat/res/values/styles_base.xml#L423

  [ThemeOverlayBlogPost]: https://medium.com/google-developers/theming-with-appcompat-1a292b754b35#.ebo3ua3bu
  [ThemeOverlayProTip]: https://plus.google.com/+AndroidDevelopers/posts/JXHKyhsWHAH

  [ViewCompat#setBackgroundTintList()]: https://developer.android.com/reference/android/support/v4/view/ViewCompat.html#setBackgroundTintList(android.view.View, android.content.res.ColorStateList)
  [TextAppearance.AppCompat.Widget.Button.Inverse]: https://github.com/android/platform_frameworks_support/blob/d57359e205b2c04a4f0f0ecf9dcb8d6086e75663/v7/appcompat/res/values/styles_base_text.xml#L101-L103
  [TintableBackgroundView]: https://developer.android.com/reference/android/support/v4/view/TintableBackgroundView.html
  [ThemeOverlayAttributes]: https://github.com/android/platform_frameworks_support/blob/d57359e205b2c04a4f0f0ecf9dcb8d6086e75663/v7/appcompat/res/values/themes_base.xml#L551-L604)

  [R.attr.buttonStyle]: https://github.com/android/platform_frameworks_support/blob/d57359e205b2c04a4f0f0ecf9dcb8d6086e75663/v7/appcompat/src/android/support/v7/widget/AppCompatButton.java#L58
  [Base.V7.Theme.AppCompat]: https://github.com/android/platform_frameworks_support/blob/d57359e205b2c04a4f0f0ecf9dcb8d6086e75663/v7/appcompat/res/values/themes_base.xml#L237
  [Widget.AppCompat.Button]: https://github.com/android/platform_frameworks_support/blob/d57359e205b2c04a4f0f0ecf9dcb8d6086e75663/v7/appcompat/res/values/styles.xml#L204
  [Base.Widget.AppCompat.Button]: https://github.com/android/platform_frameworks_support/blob/d57359e205b2c04a4f0f0ecf9dcb8d6086e75663/v7/appcompat/res/values/styles_base.xml#L409
  [@drawable/abc_btn_default_mtrl_shape]: https://github.com/android/platform_frameworks_support/blob/d57359e205b2c04a4f0f0ecf9dcb8d6086e75663/v7/appcompat/res/values/styles_base.xml#L410

  [AppCompatDrawableManagerSource1]: https://github.com/android/platform_frameworks_support/blob/d57359e205b2c04a4f0f0ecf9dcb8d6086e75663/v7/appcompat/src/android/support/v7/widget/AppCompatDrawableManager.java#L309-L314
  [AppCompatDrawableManagerSource2]: https://github.com/android/platform_frameworks_support/blob/d57359e205b2c04a4f0f0ecf9dcb8d6086e75663/v7/appcompat/src/android/support/v7/widget/AppCompatDrawableManager.java#L513-L548

  [R.attr.buttonStyle_21]: https://github.com/android/platform_frameworks_base/blob/a294dacefff98a6328cda4200e64583a72ab8b36/core/res/res/values/themes_material.xml#L98
  [Widget.Material.Button_21]: https://github.com/android/platform_frameworks_base/blob/a294dacefff98a6328cda4200e64583a72ab8b36/core/res/res/values/styles_material.xml#L459
  [R.drawable.btn_default_material_21]: https://github.com/android/platform_frameworks_base/blob/a294dacefff98a6328cda4200e64583a72ab8b36/core/res/res/drawable/btn_default_material.xml
  [R.drawable.btn_default_mtrl_shape_21]: https://github.com/android/platform_frameworks_base/blob/a294dacefff98a6328cda4200e64583a72ab8b36/core/res/res/drawable/btn_default_mtrl_shape.xml
  [R.attr.colorButtonNormal_dark_21]: https://github.com/android/platform_frameworks_base/blob/4535e11fb7010f2b104d3f8b3954407b9f330e0f/core/res/res/values/themes_material.xml#L393
  [R.color.btn_default_material_dark_21]: https://github.com/android/platform_frameworks_base/blob/a294dacefff98a6328cda4200e64583a72ab8b36/core/res/res/color/btn_default_material_dark.xml
  [R.color.btn_material_dark_21]: https://github.com/android/platform_frameworks_base/blob/a294dacefff98a6328cda4200e64583a72ab8b36/core/res/res/values/colors_material.xml#L36
  [R.attr.colorButtonNormal_light_21]: https://github.com/android/platform_frameworks_base/blob/4535e11fb7010f2b104d3f8b3954407b9f330e0f/core/res/res/values/themes_material.xml#L749
  [R.color.btn_default_material_light_21]: https://github.com/android/platform_frameworks_base/blob/a294dacefff98a6328cda4200e64583a72ab8b36/core/res/res/color/btn_default_material_light.xml
  [R.color.btn_material_light_21]: https://github.com/android/platform_frameworks_base/blob/a294dacefff98a6328cda4200e64583a72ab8b36/core/res/res/values/colors_material.xml#L37
  [Widget.Material.Button.Colored_21]: https://github.com/android/platform_frameworks_base/blob/a294dacefff98a6328cda4200e64583a72ab8b36/core/res/res/values/styles_material.xml#L471
  [R.color.btn_colored_material]: https://github.com/android/platform_frameworks_base/blob/a294dacefff98a6328cda4200e64583a72ab8b36/core/res/res/color/btn_colored_material.xml