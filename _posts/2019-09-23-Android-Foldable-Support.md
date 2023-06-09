---
layout: post
title: "The responses to the Foldable Android Device"
description: The responses to the Foldable Android Device
date: 2019-09-23 16:00:00
tags:
- Android
- Foldable

---

## Make your app resizable

You should ensure that your app works in multi-window mode and with dynamic resizing. 
Do this by setting `resizeableActivity=true`. This provides maximum compatibility with whatever form factors and 
environments your app might encounter (like foldables, desktop mode, or freeform windows). 
Test your app's behavior in split-screen or with a Foldable emulator.

If your app sets `resizeableActivity=false`, this tells the platform it doesn't support multi-window. 
The system may still resize your app or put it in multi- window mode, but compatibility is implemented by applying the same configuration to all the components in the app (including all of its Activities, Services, and more). 
In some cases, major changes (like a display size change) might restart the process rather than change the configuration.
For example, the activity below has set 'resizableActivity=false' along with a maxAspectRatio. When the device is unfolded, the activity configuration, size, and aspect ratio are maintained by putting the app in compatibility mode.

If you do not set resizeableActivity, or set it to true, the system assumes the app fully supports multi-window and is resizable.
Note that some OEMs might implement a feature that adds a small restart icon on the screen each time the activity's display area changes. 
This gives the user the chance to restart the activity in the new configuration.
New screen ratios Android 10 (API level 29) and higher supports a wider range of aspect ratios. 
With Foldables, form factors can vary from super high long and thin screens (such as 21:9 for a folded device) all the way down to 1:1.
To be compatible with as many devices as possible, you should test your apps for as many of these screen ratios as you can:

If you cannot support some of those ratios, you can use the maxAspectRatio (as before), as well as minAspectRatio to indicate the highest and lowest ratios your app can handle. 
In cases with screens that exceed these limits, your app might be put in compatibility mode.
When there are five icons in the bottom navigation view devices running Android 10 (API level 29) and higher are guaranteed a minimum touch target size of 2 inches. 

See the [Compatibility Definition Document](https://source.android.com/compatibility/cdd).
