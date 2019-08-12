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
        - [Setting a separate badge for the app and a specific page (as in the case of GitHub notifications and PR statuses).](#Setting-a-separate-badge-for-the-app-and-a-specific-page-as-in-the-case-of-GitHub-notifications-and-PR-statuses)
        - [Badging for Multiple Apps on the Same Origin (as in the case of multiple GitHub Pages PWAs)](#Badging-for-Multiple-Apps-on-the-Same-Origin-as-in-the-case-of-multiple-GitHub-Pages-PWAs)
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
  - [Design Questions](#Design-Questions)
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
    - [What to Badge?](#What-to-Badge)
    - [Index of Considered Alternatives](#Index-of-Considered-Alternatives)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Overview

The **Badging API** is a proposed Web Platform API allowing websites to apply
badges (small status indicators) to pages or sets of pages on their origin. We
are deliberately agnostic about which contexts a badge can appear in, but have
two fairly different contexts in mind:

* "Document" contexts, associated with an open document, such as when a badge is
  shown on or near the page's icon in a browser tab.
* "Handle" contexts, associated with a handle or link to a site somewhere in the
  user agent or operating system UI, but not necessarily associated with a
  running document, such as a bookmark or [installed web
  app](https://www.w3.org/TR/appmanifest/#installable-web-applications) icon.

Here is a mock of a badge being applied to the page tab:

![Mock of a badge being applied to the page tab](images/tab-badge.png)

This use case is already satisfied today with sites dynamically setting either
their favicon or title to include some status indicator. The use of an explicit
badge API has a number of advantages over the "hack" ways:

* The badge can appear outside (completely, or partially overlapping) of the
  favicon, leaving more room for the site's brand image.
* The badge is meaningful to the user agent, which can present it to the user in
  various ways, such as announcing it verbally to a user using a screen reader.
* Badges can be displayed with a consistent style across websites, chosen by the
  user agent.
* User agents can provide a way for users to disable badges on a per-site or
  global basis.

The "handle" context is more nebulous because it means associating a badge with
any place the user agent shows an icon representing a site, page or app. For
example, it could be applied to an icon in the bookmark bar, or a "commonly
visited sites" index on the new tab page.

![Mock of a badge being applied to the bookmark bar](images/bookmark-badge.png)
<br>Mock of a badge being applied to the bookmark bar

For [installed web
applications](https://www.w3.org/TR/appmanifest/#installable-web-applications),
the badge can be applied in whatever place the OS shows apps, such as the shelf,
home screen or dock.

Here are some examples of app badging applied at the OS level:

![Windows taskbar badge](images/uwp-badge.png)
<br>Windows taskbar badge

![macOS dock badge](images/mac-badge.png)
<br>macOS dock badge

![Android home screen badge](images/android-badge.png)
<br>Android home screen badge

Unlike document contexts, badges applied in "handle" contexts can be shown and
updated even when there are no tabs or windows open for the app. So there are
some special considerations for this use case, such as how to set the badge in
response to a [Push message](https://www.w3.org/TR/push-api/), and how to scope
a badge so that it appears on the desired set of pages or apps. The most
commonly requested use case for "handle" badging is badging an app icon on the
OS shelf, but it generalizes to non-installed sites as well.

## Usage examples

The simplest possible usage of the API is a single call to `Badge.set` from a
foreground context (which might be used to show an unread count in an email
app):

```js
Badge.set(getUnreadCount());
```

This will set the badge for all pages and apps in the current origin until it is
changed. If `getUnreadCount()` (the argument to `Badge.set`) is 0, it will
automatically clear the badge.

If you just want to show a status indicator flag without a number, use the
Boolean mode of the API by calling `Badge.set` without an argument, and
`Badge.clear` (which might be done to indicate that it is the player's turn to
move in a multiplayer game):

```js
if (myTurn())
  Badge.set();
else
  Badge.clear();
```

The reason we are considering both the "document" and "handle" contexts in the
same API, and not two separate badging APIs, is so that in the common case,
developers can just set a badge for the origin and have it show up in whatever
places the user agent wants to show it. However, if you just want to badge a
specific set of URLs, and not the whole site, use the `scope` option (which
might be done if you have a different "unread count" on each page):

```js
Badge.set(getUnreadCount(location.pathname), {scope: location});
```

More advanced examples are given below.

## Goals and use cases

The purpose of this API is:

* To subtly notify the user that there is new activity that might require their attention without requiring an OS-level [notification](https://notifications.spec.whatwg.org/).
* To indicate a small amount of additional information, such as an unread count.
* To allow certain user-blessed pages (such as Bookmarks or [Installed Web Applications](https://www.w3.org/TR/appmanifest/#installable-web-applications)) to convey this information, regardless of whether they are currently open.
* To allow setting the badge when no documents from the site are running (e.g. an email app updating an unread count in the background).

Non-goals are:

* To provide an arbitrary image badge. The web platform already provides this capability via favicons.

Possible areas for expansion:

* Support rendering a small status indicator (e.g., a music app shows ▶️ or ⏸️; a weather app shows ⛈️ or ⛅️): either a pre-defined set of glyphs or simply allowing a Unicode character to be rendered.

Examples of sites that may use this API:

* Chat, email, and social apps could signal that new messages have arrived.
* Any application that needs to signal that user action is required (e.g., in a turn-based game, when it is the player's turn).
* As a permanent indicator of a page's status (e.g., on a build page, to show that the build has completed).

Advantages of using the badging API over notifications:

* Can be used for much higher frequency / lower priority events than notifications, because each new event does not disrupt the user.
* There may be no need to request permission to use the badging API, since it is much less invasive than a notification.

Typically, sites will want to use both APIs together: notifications for high-importance events such as new direct messages or incoming calls, and badges for all new messages including group chats not directly addressed to the user.

## API proposal

### The model

1. A Badge is associated with a [scope](https://w3c.github.io/ServiceWorker/#service-worker-registration-scope).
2. Documents are badged with the most specific badge for their url (i.e. prefer a badge for `/page/1` to a badge for `/page/` when on the url `/page/1?foo=7`).
3. If no scope is specified the scope is the current origin (equivalent to setting the scope to `/`).
4. For [installed applications](https://www.w3.org/TR/appmanifest/#installable-web-applications), a user agent **MAY** display the badge with the most specific scope still encompassing the [navigation scope](https://www.w3.org/TR/appmanifest/#navigation-scope) of the application in an [OS specific context](#OS-Specific-Contexts).

> Note: [scope](https://w3c.github.io/ServiceWorker/#service-worker-registration-scope) may need to be moved into its own spec, as it is now being referenced by [ServiceWorkers](https://w3c.github.io/ServiceWorker/#service-worker-registration-scope), [AppManifests](https://www.w3.org/TR/appmanifest/#navigation-scope) and [Badging](#The-model).

At any time, the badge for a specific scope may be:

* Nothing. Apply the badge for a less specific scope, or, if there is no matching badge, display nothing.
* A "flag" indicating the presence of a badge with no contents, or
* A positive integer.

The model does not allow a badge to be a negative integer, or the integer value 0 (setting the badge to 0 is equivalent to clearing the badge).

### The API

The `Badge` interface is a member object on
[`Window`](https://html.spec.whatwg.org/#the-window-object). It contains the following methods:

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

## Advanced Examples

These examples generally provide more context around what is going on, and may specify scopes. They are intended to show how real world applications may use the API.

### Updating a Badge on a message from a WebSocket (as in a messaging app, receiving new messages):

```js

// A list of all messages received by the application.
const messages = [];

// Sets the badge count to the number of unread messages.
function updateBadgeCount() {
  // Count the number of unread messages.
  const badgeCount = messages
    .filter(m => m.unread)
    .length;
  Badge.set(badgeCount);
}

// Adds a new message and updates the badge count.
function addMessage(message) {
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

### Setting a separate badge for the app and a specific page (as in the case of GitHub notifications and PR statuses).
On all pages of this site, we wish to display the notification count, except for `/status/{number}`, where we should instead display `flag` if the page's status is `ready` or nothing, if it is not.

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

On https://example.com/status/1
```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <title>Main Page</title>
  </head>
  <body>
    <script>
      // `/status/39/` => `39`
      const statusNumber = location.pathname
        .split('/') // Get path segments
        .filter(s => s) // which are non-empty
        .pop(); // and take the last one.

      const badgeOptions = {
        scope: `/status/${statusNumber}`
      };

      const statusInfo = {
        status: 'ready',
        author: 'me',
        url: `https://example.com/status/${statusNumber}`
      };

      if (statusInfo.status === 'ready')
        Badge.set(badgeOptions);
      else
        Badge.clear(badgeOptions);

    </script>
  </body>
</html>
```

### Badging for Multiple Apps on the Same Origin (as in the case of multiple GitHub Pages PWAs)
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
Badges may appear in any place that the user agent deems appropriate. In general, these places should be obviously related to the pages being badged, so users understand what the status is for. Appropriate places could include:
- Tab favicons.
- Bookmark icons.
- [OS Specific Contexts](#OS-Specific-Contexts) for [Installed Web Applications](https://www.w3.org/TR/appmanifest/#installable-web-applications).
  
> Note: When showing an badge in an [OS Specific Context](#OS-Specific-Contexts) user agents should attempt reuse existing [operating system APIs and conventions](#Specific-operating-system-treatment-for-installed-web-applications), to achieve a native look-and-feel.

If the exact representation of a badge is not supported (e.g., a 2-digit number while only single characters are allowed, or a character where only numbers are allowed) the user agent should make a best effort attempt to map the unsupported data onto supported data. This may involve:
- Saturating a number: 351 -> '99+'
- Degrading the data: 7 -> 'flag' when only flag badges are supported.

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

## Design Questions

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

With push notifications, [some browsers](https://github.com/w3c/push-api/issues/313) enforce that a notification be shown, or the browser will stop the app from processing additional notifications. Things are not as simple with badging, as a site could always set its badge to the same value, which would give the user would no cue that a site is running (or has run) in the background (and could mine all the cryptocurrencies!).

To solve this, we are considering a separate notification channel especially for badges. The browser would know how to turn the notification into a badge without running any JavaScript.

### Is this API useful for mobile OS’s?
iOS has support for badging APIs (see [iOS](#ios).

On Android, badging is blocked at an OS level, as there is [no API for setting a badge without also displaying a notification](#android). However, a badge will already be displayed if a PWA has pending notifications (it just doesn’t allow the fine grained control proposed by this API).

To summarize: This API cannot be used in PWAs on either mobile operating system. Support on iOS is blocked until Safari implements the API and Android does not have an API for controlling badges. Should either situation change, the badging API will become trivially available.

### Why is this API attached to `window` instead of `navigator` or `notifications`?

There was a [poll](https://github.com/WICG/badging/issues/14#issuecomment-445548190), and the current style seemed most popular. There is more detail and discussion in [Issue 14](https://github.com/WICG/badging/issues/14).

### Is there an upper limit on the size of the integer? And if so, what's the behavior if that limit is reached?

There is no upper limit (besides `Number.MAX_SAFE_INTEGER`). However, each user agent is
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

### What to Badge?
There are three main use cases developers might have for the badging API.
1. Badging an entire App or Origin
   
   Useful for a social network or chat application.

2. Badging a specific URL, or group of URLs.

   Useful when a specific page has a status associated with it, such as reporting whether a build succeeded or failed.

3. Badging a specific Document.

   Potentially useful when there is some internal state contained in the document which controls the badge.

> Note: Initially, `2` and `3` seem very similar. However, in option `2` all pages on the same url would share a badge, while in `3` badges would be tied to a specific, currently open instance of a page (a tab).

The API proposed in this explainer will provide solutions to `1` (by badging `/` for the origin or `/app/` for an app), and `2` (by badging the full url for the page). It does not attempt solve `3`. Conceptually, it would be possible to support `3` by adding a switch to the options passed to `Badge.set(.., { documentOnly: true })`. However, this would be extra work to spec and implement, and we don't have use cases to justify it at this time. In addition, badging individual documents is already possible (though unpleasant), with the `favicon` API.

### Index of Considered Alternatives
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
