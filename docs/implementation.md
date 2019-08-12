# Badge implementation on host platforms

This section describes a possible treatment on each major OS. User agents are
free to implement however they like, but this should give an idea of what the
API will look like in practice.

## macOS

Requires macOS 10.5 or higher.

* The host API is a string, truncated from the center to a system-defined
  length, as shown/described
  [here](https://eternalstorms.wordpress.com/2016/10/29/how-to-badge-an-apps-icon-in-the-dock/).
  The API is
  [`NSDockTile.setBadgeLabel`](https://developer.apple.com/documentation/appkit/nsdocktile/1524433-badgelabel).
* Flag badges are passed as an empty string to `setBadgeLabel` (if that works,
  otherwise pass a space character). This shows an empty red circle.
* Integer badges are saturated to 3 or 4 digits (with a "+" if overflowing), and
  passed as a string to `setBadgeLabel`.

## Universal Windows Platform

Requires Windows 10 or higher, and requires that the user agent be a "UWP" app.
Note that Google Chrome and Mozilla Firefox do not fall into this category, and
hence don't have access to the necessary APIs.

* The host API is
  [`BadgeNotification`](https://docs.microsoft.com/en-us/uwp/api/windows.ui.notifications.badgenotification),
  which allows either a positive integer (saturated at 99) or a fixed glyph
  (shown
  [here](https://docs.microsoft.com/en-us/windows/uwp/design/shell/tiles-and-notifications/badges)).
* Integer badges are just passed straight through into this API.
* Flag badges are passed as a fixed glyph, perhaps "attention", since I don't
  think there is a way to show an empty circle.

This will show up on both the Taskbar and Start Menu tile.

## Legacy Windows (Win32)

Requires Windows 7 or higher.

* The host API is
  [`ITaskbarList3::SetOverlayIcon`](https://docs.microsoft.com/en-au/windows/desktop/api/shobjidl_core/nf-shobjidl_core-itaskbarlist3-setoverlayicon),
  which allows applications to set a 16x16-pixel overlay icon in the corner of
  the application's main icon in the Taskbar.
* Due to the nature of being a 16x16-pixel icon, the user agent must render the
  text or number into an image. It pretty much has to be 1–2 characters.
* Flag badges are just passed as a coloured circle.
* Integer badges are rendered as the number, if a single digit, or "+", if
  greater than 9.

Note: This API only allows badging a currently open window.

## Android

Requires Android 8.0 (Oreo) or higher.

* The host API is
  [`NotificationChannel`](https://developer.android.com/training/notify-user/badges),
  which lets you set a number only. The badge is usually shown as a coloured
  dot; the number is only shown on a long-press.
* Tricky: The API is tied to notifications. You can't show a badge unless there
  are pending notifications.

## Chrome OS

Not currently supported, but will [soon be available](https://crbug.com/801014)
for Android apps using the above `NotificationChannel` API. The Chrome browser
itself will be able to re-use this mechanism for showing a coloured dot on web
application icons.

## iOS

* The host API is
  [`UIApplication.applicationIconBadgeNumber`](https://developer.apple.com/documentation/uikit/uiapplication/1622918-applicationiconbadgenumber),
  which lets you set a positive integer only.
* Integer badges are passed directly to the host API.
* Flag badges and strings other than digits are just represented by the number
  "1".

## Ubuntu

Requires Ubuntu (no general API for Linux).

* The host API is
  [`unity_launcher_*`](https://wiki.ubuntu.com/Unity/LauncherAPI), which lets
  you set an integer only.
* See iOS treatment.

## Summary

* All known host OSes support some form of badge.
* Integer badges are always supported (but sometimes translated).
* Most platforms support showing at least one arbitrary character. UWP limits to
  a small selection of glyphs. iOS and Ubuntu can only show numbers.

Thus, a fallback option for platforms that do not support arbitrary characters
(e.g., choose whether to show a number, or nothing) may be necessary.

