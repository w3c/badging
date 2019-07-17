# Badging API Explainer

Author: Matt Giuca <mgiuca@chromium.org><br>
Author: Jay Harris <harrisjay@chromium.org><br>
Author: Marcos Cáceres <mcaceres@mozilla.org>

Date: 2019-07-17

## Table of Contents
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Badging API Explainer](#Badging-API-Explainer)
  - [Table of Contents](#Table-of-Contents)
  - [Overview](#Overview)
    - [OS Specific Contexts](#OS-Specific-Contexts)
  - [API proposal](#API-proposal)
    - [The model](#The-model)
    - [The API](#The-API)
    - [Examples](#Examples)
      - [Basic Examples](#Basic-Examples)
        - [Setting an integer badge (as in an email app):](#Setting-an-integer-badge-as-in-an-email-app)
        - [Setting and clearing a boolean flag (as in a game of chess):](#Setting-and-clearing-a-boolean-flag-as-in-a-game-of-chess)
      - [Advanced Examples](#Advanced-Examples)
        - [Updating a Badge on a message from a WebSocket (as in a messaging app, receiving new messages):](#Updating-a-Badge-on-a-message-from-a-WebSocket-as-in-a-messaging-app-receiving-new-messages)
        - [Setting a separate badge for the app and a specific page (as in the case of github notifications an PR statuses).](#Setting-a-separate-badge-for-the-app-and-a-specific-page-as-in-the-case-of-github-notifications-an-PR-statuses)
        - [Badging for Multiple Apps on the Same Origin (as in the case of multiple Github Pages PWAs)](#Badging-for-Multiple-Apps-on-the-Same-Origin-as-in-the-case-of-multiple-Github-Pages-PWAs)
  - [UX treatment](#UX-treatment)
  - [Specific operating system treatment for installed web applications](#Specific-operating-system-treatment-for-installed-web-applications)
    - [macOS](#macOS)
    - [Universal Windows Platform](#Universal-Windows-Platform)
    - [Legacy Windows (Win32)](#Legacy-Windows-Win32)
    - [Android](#Android)
    - [Chrome OS](#Chrome-OS)
    - [iOS](#iOS)
    - [Ubuntu](#Ubuntu)
    - [Summary](#Summary)
  - [FAQ](#FAQ)
    - [What data types are supported in different operating systems?](#What-data-types-are-supported-in-different-operating-systems)
    - [Why limit support to just an integer? What about other characters?](#Why-limit-support-to-just-an-integer-What-about-other-characters)
    - [Couldn’t this be a declarative API, so it would work without JavaScript?](#Couldnt-this-be-a-declarative-API-so-it-would-work-without-JavaScript)
    - [Why can’t this be used in the background from the ServiceWorker? (see #28 and #5)](#Why-cant-this-be-used-in-the-background-from-the-ServiceWorker-see-28-and-5)
    - [Is this API useful for mobile OS’s?](#Is-this-API-useful-for-mobile-OSs)
    - [Why is this API attached to `window` instead of `navigator` or `notifications`?](#Why-is-this-API-attached-to-window-instead-of-navigator-or-notifications)
    - [Is there an upper limit on the size of the integer? And if so, what's the behavior if that limit is reached?](#Is-there-an-upper-limit-on-the-size-of-the-integer-And-if-so-whats-the-behavior-if-that-limit-is-reached)
    - [Are you concerned about apps perpetually showing a large unread count?](#Are-you-concerned-about-apps-perpetually-showing-a-large-unread-count)
    - [Internationalization](#Internationalization)
    - [Security and Privacy Considerations](#Security-and-Privacy-Considerations)
  - [Considered Alternatives](#Considered-Alternatives)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Overview

The **Badging API** is a proposed Web Platform API allowing [documents](https://dom.spec.whatwg.org/#document) to apply badges to sets of pages on the same origin as their [document url](https://dom.spec.whatwg.org/#concept-document-url). These badges are displayed in a location related to their documents such as the tab favicons.

If the set of pages being badged corresponds to a [Installed Web Application](https://www.w3.org/TR/appmanifest/#installable-web-applications), the user agent may choose to display a badge in a [operating system specific place](#OS-Specific-Contexts) associated with the application (such as an icon on the shelf, home screen or dock).

### OS Specific Contexts
![Windows taskbar badge](images/uwp-badge.png)
<br>Windows taskbar badge

![macOS dock badge](images/mac-badge.png)
<br>macOS dock badge

![Android home screen badge](images/android-badge.png)
<br>Android home screen badge

The purpose of this API is:

* To give documents a small but visible place to subtly notify the user that there is new activity that might require their attention without showing a full [notification](https://notifications.spec.whatwg.org/).
* To indicate a small amount of additional information, such as an unread count.
* To allow certain blessed pages (such as Bookmarks or [Installed Web Applications](https://www.w3.org/TR/appmanifest/#installable-web-applications)) to convey this information, regardless of whether they are currently open.

Non-goals are:

* To provide an arbitrary image badge.

Possible areas for expansion:

* Support rendering a small status indicator (e.g., a music app shows ▶️ or ⏸️; a weather app shows ⛈️ or ⛅️).
* Setting the badge from a service worker (e.g. an email app updating an unread count in the background).
* Support badges on bookmark icons (this is probably up to user agents to support, and may not need anything new API wise).
* Support for non-inherited badges (see [Issue 42](https://github.com/WICG/badging/issues/42)).
* Support for unicode glyph badges.

Examples of sites that may use this API:

* Chat, email and social apps; to signal that new messages have arrived.
* Productivity apps, to signal that a long-running background task (such as rendering an image or video) has completed.
* Games, to signal that player action is required (e.g., in Chess, when it is the player's turn).

Advantages of using the badging API over notifications:

* Can be used for much higher frequency events than notifications, because each new event does not disrupt the user.
* There is no need to request permission to use the badging API, since it is much less invasive than a notification.

(Typically, sites will want to use both APIs together: notifications for high-importance events such as new direct messages or incoming calls, and badges for all new messages including group chats not directly addressed to the user.)

## API proposal

### The model

1. A Badge is associated with a [scope](https://www.w3.org/TR/appmanifest/#navigation-scope).
2. Documents are badged with the most specific badge for their url (i.e. prefer a badge for `/page/1` to a badge for `/page/` when on the url `/page/1?foo=7`).
3. If no scope is specified the scope is the current origin (equivalent to setting it to `/`).
4. For [installed applications](https://www.w3.org/TR/appmanifest/#installable-web-applications), a user agent **MAY** display the badge with the most specific scope still encompassing the [navigation scope](https://www.w3.org/TR/appmanifest/#navigation-scope) of the application in an [OS specific context](#OS-Specific-Contexts).

At any time, the badge for a specific scope may be:

* Nothing. Apply the badge for a less specific scope, or, if there is no matching badge, display nothing.
* A "flag" indicating the presence of a badge with no contents, or
* A positive integer.

The model does not allow a badge to be a negative integer, or the integer value 0 (setting the badge to 0 is equivalent to clearing the badge).

### The API

The `Badge` interface is a member object on
[`Window`](https://html.spec.whatwg.org/#the-window-object). It contains three methods:

```webidl
dictionary BadgeOptions {
  // The scope the badge applies to.
  // If unspecified this is the origin of the current page.
  USVString scope;
}

[Exposed=Window]
interface Badge {
  // Sets the badge for a scope to be |contents|.
  void set(unsigned long long contents, optional BadgeOptions options);

  // Sets the badge for a scope to be the special value, 'flag'.
  void set(optional BadgeOptions options);

  // Clears the badge for a scope.
  void clear(optional BadgeOptions options);
}
```

> Note: Should we have a separate overload for boolean flags now, as discussed in [Issue 19](https://github.com/WICG/badging/issues/19) and [Issue 42](https://github.com/WICG/badging/issues/42)?

> Note: This API can only be used from a foreground page. Use from a service worker is being considered for the future. [The FAQ](#Why-cant-this-be-used-in-the-background-from-the-ServiceWorker-see-28-and-5) has more details).

### Examples

#### Basic Examples
These examples provided limited context. They are intended to show how the API will look.

##### Setting an integer badge (as in an email app):
```js
Badge.set(getUnreadCount());
```

##### Setting and clearing a boolean flag (as in a game of chess):
```js
if (myTurn())
  Badge.set();
else
  Badge.clear();
```

#### Advanced Examples
These examples generally provide more context around what is going on, and may specify scopes. They are intended to show how real world applications may use the API.

##### Updating a Badge on a message from a WebSocket (as in a messaging app, receiving new messages):

```ts
interface ChatMessage {
  content: string;
  unread: boolean;
}

// A list of all messages received by the application.
const messages: ChatMessage[] = [];

// Sets the badge count to the number of unread messages.
const updateBadgeCount = () => {
  // Count the number of unread messages.
  const badgeCount = messages
    .filter(m => m.unread)
    .length;
  Badge.set(badgeCount);
}

// Adds a new message and updates the badge count.
const addMessage = (message: Message) => {
  messages.push(message);
  updateBadgeCount();
}

// Create a new socket, connected to our messages endpoint.
const socket = new WebSocket('ws://example.com/messages');
socket.onmessage = (event) => {
  // Parse the message and add it to our list.
  const chatMessage = JSON.parse(event.data);
  addMessage(chatMessage);
};
```

##### Setting a separate badge for the app and a specific page (as in the case of github notifications an PR statuses).
On all pages of this site, we wish to display the notification count, except for `/do-have-https`, where we should instead display `flag` if the page was not served over `https` or nothing, if it was.

The main page of our site https://example.com/
```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <title>Main Page</title>
  </head>
  <body>
    <script>
      // There are ten unread notifications.
      // This was injected by the server.
      Badge.set(10);
    </script>
  </body>
</html>
```

On https://example.com/do-i-have-https
```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <title>Main Page</title>
  </head>
  <body>
    <script>
      const badgeOptions = {
        scope: `/do-i-have-https`
      };

      const hasHttps = window
        .location
        .href
        .startsWith('https');

      // Note: This is not supported.
      Badge.set(hasHttps ? '✓' : '☠', badgeOptions);

      // But we could do this instead:
      if (hasHttps)
        Badge.set(badgeOptions);
      else Badge.clear(badgeOptions);

    </script>
  </body>
</html>
```

##### Badging for Multiple Apps on the Same Origin (as in the case of multiple Github Pages PWAs)
```js
// Scope of Frobnicate is /frobnicate
// Scope of Bazify is /baz
// Both apps are hosted on https://origin.example/${scope}

// By default, setting a badge affects the origin.
Badge.set();
// Badge for Frobnicate and Bazify is 'flag'.
// Badge for /index.html is 'flag'.

// Installed applications use the badge with the most
// specific scope which encompasses their own scope.
Badge.set(7, { scope: `/baz` });
// Badge for Frobnicate is 'flag'.
// Badge for Bazify is '7'.
// Badge for /index.html is 'flag'.

// Pages inside an installed application do not effect
// the badge displayed in an OS specific context for
// that app.
Badge.set(55, { scope: `/frobnicate/page` });
// Badge for Frobnicate is 'flag'.
// Badge for Bazify is '7'.
// Badge for /index.html is 'flag'.
// Badge for /frobnicate/page is '55'.

// Setting the badge for '/' is equivalent to omitting
// the scope param.
Badge.set(88, { scope: `/` });
// Badge for Frobnicate is '88'.
// Badge for Bazify is '7'.
// Badge for /index.html is '88'.
// Badge for /frobnicate/page is '55'.

// Clearing the badge for a scope will cause the next
// most specific badge to be applied.
Badge.clear({ scope: '/baz' });
// Badge for Frobnicate is '88'.
// Badge for Bazify is '88'.
// Badge for /index.html is '88'.
// Badge for /frobnicate/page is '55'.

// Clears the badge for the origin.
Badge.clear();
// Badge for Frobnicate, Bazify, /index.html is nothing.
// Badge for /frobnicate/page is '55'.
```

## UX treatment

* Installed Web applications are typically given some presence in an
  operating-system location, such as the shelf, launcher, dock, home screen,
  etc. Usually the application is represented by an icon.
* The badge would appear as a user-agent-defined overlay on top of the app's
  icon, about a quarter of the size and in one of the four corners, as shown in
  the sample images above.
* Operating systems generally provide a badge system for native applications;
  user agents should re-use existing operating system APIs / UI and conventions
  to achieve a native look-and-feel.
* The user agent should make a "best effort" attempt to map the badge data
  structure onto the host operating system's badge format:
  * Some operating systems (e.g., Android) only provide UI for a Flag badge;
    just a coloured dot with no content (see the sample image above). In these
    cases, the user agent should follow this convention, and only show a Flag,
    even if the website sets richer badge data.
* If the operating system doesn't allow the exact representation of the badge
  (e.g., a 2-digit number but the OS only allows a single character, or a
  character but the OS only allows a number), the user agent should try the best
  to map into the OS representation. This may involve:
  * Saturating a number; e.g., 351 -> "99+".

## Specific operating system treatment for installed web applications

This section describes a possible treatment on each major OS. User agents are
free to implement however they like, but this should give an idea of what the
API will look like in practice.

### macOS

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

### Universal Windows Platform

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

### Legacy Windows (Win32)

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

### Android

Requires Android 8.0 (Oreo) or higher.

* The host API is
  [`NotificationChannel`](https://developer.android.com/training/notify-user/badges),
  which lets you set a number only. The badge is usually shown as a coloured
  dot; the number is only shown on a long-press.
* Tricky: The API is tied to notifications. You can't show a badge unless there
  are pending notifications.

### Chrome OS

Not currently supported, but will [soon be available](https://crbug.com/801014)
for Android apps using the above `NotificationChannel` API. The Chrome browser
itself will be able to re-use this mechanism for showing a coloured dot on web
application icons.

### iOS

* The host API is
  [`UIApplication.applicationIconBadgeNumber`](https://developer.apple.com/documentation/uikit/uiapplication/1622918-applicationiconbadgenumber),
  which lets you set a positive integer only.
* Integer badges are passed directly to the host API.
* Flag badges and strings other than digits are just represented by the number
  "1".

### Ubuntu

Requires Ubuntu (no general API for Linux).

* The host API is
  [`unity_launcher_*`](https://wiki.ubuntu.com/Unity/LauncherAPI), which lets
  you set an integer only.
* See iOS treatment.

### Summary

* All known host OSes support some form of badge.
* Integer badges are always supported (but sometimes translated).
* Most platforms support showing at least one arbitrary character. UWP limits to
  a small selection of glyphs. iOS and Ubuntu can only show numbers.

Thus, a fallback option for platforms that do not support arbitrary characters
(e.g., choose whether to show a number, or nothing) may be necessary.

## FAQ

### What data types are supported in different operating systems?	

See [above](#Summary).

### Why limit support to just an integer? What about other characters?

It isn't a technical limitation, it's an attempt to keep behavior as consistent as possible
on different host platforms (UWP only supports a set of symbols, while iOS, Android
and Ubuntu don't support them at all).

Limiting support to integers makes behavior more predictable, though we are considering
whether it might be worth adding support for other characters or symbols in future.

### Couldn’t this be a declarative API, so it would work without JavaScript?
It could be, yes. However, as badges may be shared across multiple documents, this could be kind of confusing (e.g. there is a <link rel="shortcut icon badge" href="/favicon.ico" badge="99"> in the head of a document, but it is being badged with 7 because another page was loaded afterwards). There is some discussion of this [here](https://github.com/WICG/badging/issues/1#issuecomment-485635068).

### Why can’t this be used in the background from the ServiceWorker? (see #28 and #5)

Realistically, using the badge from the background is the most important use case, so ideally, badges could be updated in the background via the service worker.

However there is a non-trivial problem: We need a way to inform the user that the app is running.

With push notifications, browsers enforce that a notification be shown, or the browser will stop the app from processing additional notifications. Things are not as simple with badging, as a site could always set its badge to the same value, which would give the user would no cue that a site is running (or has run) in the background (and could mine all the cryptocurrencies!).

To solve this, we are considering a separate notification channel especially for badges. The browser would know how to turn the notification into a badge without running any JavaScript.

> Note: I might be pretty far off the mark here, I’m not that familiar in things service worker, background, or notifications.

### Is this API useful for mobile OS’s?
iOS has support for badging APIs (see [iOS](#ios). However, PWAs on iOS will not be able to use the badging API until Safari (the only browser engine allowed on iOS) adds support for it.

On Android, badging is blocked at an OS level, as there is [no API for setting a badge without also displaying a notification](#android). However, a badge will already be displayed if a PWA has pending notifications (it just doesn’t allow the fine grained control proposed by this API).

To summarize: This API cannot be used in PWAs on either mobile operating system. Support on iOS is blocked until Safari implements the API and Android does not have an API for controlling badges. Should either situation change, the badging API will become trivially available.

### Why is this API attached to `window` instead of `navigator` or `notifications`?

There was a [poll](https://github.com/WICG/badging/issues/14#issuecomment-445548190), and the current style seemed most popular. There is more detail and discussion in [Issue 14](https://github.com/WICG/badging/issues/14).

### Is there an upper limit on the size of the integer? And if so, what's the behavior if that limit is reached?

There is no upper limit (besides 2<sup>64</sup>). However, each user agent is
free to impose a limit and silently saturate the value (e.g., display all values
above 99 as "99+").

### Are you concerned about apps perpetually showing a large unread count?

Yes. If users habitually leave mail or chats unread, and mail or chat apps
simply call `set(getUnreadCount())`, it could result in several apps simply
showing a large number, presenting several issues:

* Leaving "clutter" on the user's shelf, and
* Making the user unable to tell when new messages arrive.

However, the only solution to this is a much more limited API which only lets
you show the count of notifications (or similar). We wanted to give apps the
full power of showing a native badge.

### Internationalization
The API allows `set()`ing an `unsigned long long`. When presenting this value, it should be formatted according to the user's locale settings.

### Security and Privacy Considerations
The API is set only, so data badged can't be used to track a user. Whether the API is present could possibly be used as a bit of entropy to fingerprint users, but this is the case for all new APIs.

## Considered Alternatives
- A [declarative API](#Couldnt-this-be-a-declarative-API-so-it-would-work-without-JavaScript).
- Exposing the badging API [elsewhere](#Why-is-this-API-attached-to-window-instead-of-navigator-or-notifications).
- Supporting [non-integers](#Why-limit-support-to-just-an-integer-What-about-other-characters).
- Use in the [background](#Why-cant-this-be-used-in-the-background-from-the-ServiceWorker-see-28-and-5).
- [Separate methods](https://github.com/WICG/badging/issues/19) for setting/clearing boolean flags and numbers.
- Exposing a [getter](https://github.com/WICG/badging/issues/18) for badge contents.
- Only [badging](https://github.com/WICG/badging/issues/1) [PWAs](https://github.com/WICG/badging/issues/12).
- Supporting [query-string scopes](https://github.com/WICG/badging/issues/1#issuecomment-511634128).
- Adding [fallbacks](https://github.com/WICG/badging/issues/2), when the system can't display a badge.
- [Promise based](https://github.com/WICG/badging/issues/35#issue-459665145) badging API.