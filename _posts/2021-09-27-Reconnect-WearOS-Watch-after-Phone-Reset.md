---
title: "Reconnecting a WearOS Watch after a Phone Reset or Replacement"
date: 2021-09-27 9:00 -0700
categories: [Android]
tags: [Android, WearOS]
excerpt: "Sometimes an Android phone is getting replaced, or it gets reset due to a problem or after flashing a Custom ROM. Then the WearOS companion needs a reset too - or maybe not?"
classes: wide
header:
  image: /assets/images/powershell_header-1280x200.png
---
Sometimes an Android phone is getting replaced, or it gets reset due to a problem, or after flashing a Custom ROM. Then the WearOS companion needs a reset too - or maybe it doesn't?

Well, it is not. There is a way to reconnect to a phone (no matter if it's the same as before or not) without resetting the watch, so no apps, watch faces or tiles are getting lost as they wood with a full reset.

# Preparation

To issue the commands needed to have the watch reconnecting you need to establish a debugging connection via the `Android Debug Bridge` or `ADB`. The tools for this protocol are part of the Android Studio, a development software provided by Google that is pretty large. Depending on your system you may be able to install a tools-only package from your system's software repo, or use one from another (trustworthy) source, like the [XDA Developer Forum](https://www.xda-developers.com).

However you get it on the system, make sure you can actually run the `adb` command from your preferred command line.

## Enable `Developer options`
If you haven't done that yet the first thing to do on the watch is to enable the `Developer Settings`. In order to do that you open the `Settings`, navigate to `System`, then `About`. There you scroll down and start tapping the line that says `Build number`. A toast notification (usually a small text bubble at the bottom of the screen) will appear multiple times and display a countdown. Keep tapping until it says `You are now a Developer`, then go back to the `Settings`. 

## Enable `ADB debugging`
Open the `Developer options` at the bottom of the list and scroll until you see `ADB debugging`. Enable this. 

Connect your computer to your watch. If you’re doing it over USB, plug your watch into your computer. If you are going to connect by Wi-Fi then you also need to enable the option `Debug over Wi-Fi`. Wait a moment and then note down the IP address that is now shown underneath that option, assuming the watch is on the same Wi-Fi as the computer.

Bluetooth is also an option here, but I've never tried that, as I find the Wi-Fi connection the easiest. The handling should be the same as via USB, though.

## Connect via `ADB`

If you have connected the watch to your Wi-Fi, connect to the watch remotely via `ADB`. You can do this by typing `adb connect xxx.xxx.x.x:nnnn` with the `xxx.xxx.x.xx`’s being the IP you noted earlier under the wireless debugging option, and `nnnn` is the port `ADB` is using. If only one connection is active this port should be `5555` by default, otherwise use whatever the watch shows.

```bash
adb connect 192.168.1.187:5555
```

For a USB connection no extra command is needed to connect, it happens automatically on the first run of `adb`.

When the `ADB` connection is coming up the watch will display a prompt, asking `Allow Debugging?`. Chose either `OK` or `Always allow debugging from this computer`. Once it has successfully connected, run the next command to verify the connection is established. It should print an alphanumeric string that represents the device ID.

```bash
adb devices
```

If this doesn't work make sure you have an `ADB` driver installed. Your phone might have come with one, or the adb package has it, or look up [this thread on XDA](https://forum.xda-developers.com/t/official-tool-windows-adb-fastboot-and-drivers-15-seconds-adb-installer-v1-4-3.2588979/) for a universal installer.

# Reset the watch's connection data

Now it is time to reset the connection that is stored inside the watch without resetting the entire watch.

## Remove the existing connection

Run this command to remove the existing connection information

```bash
adb shell "pm clear com.google.android.gms && reboot"
```

After a few seconds, you will see the word “Success” appear, followed by your watch rebooting. When it has rebooted, do not attempt to connect to it with your phone"s WearOS app. Connect the watch to your computer again, then reconnect to it via `ADB` following the procedure from earlier.

## Trigger the setup routine

When the watch has reconnected, run the following command to manually start the setup routine that let's the phone discover the watch:

```bash
adb shell "am start -a android.bluetooth.adapter.action.REQUEST_DISCOVERABLE"
```

Confirm the prompt that appears on the watch, asking for allowing other devices to discover it. 

## Connect your phone to the watch

You may now open the Wear OS app on your phone and connect to the watch as if it were new.  If the app gets stuck on searching for updates, close the app and open it again. It’ll go away and the setup should work as usual like before, now with your new phone/ROM setup.

Enjoy!
