# Badging API Explainer

Author: Matt Giuca <mgiuca@chromium.org><br>
Author: Jay Harris <harrisjay@chromium.org><br>
Author: Marcos Cáceres <marcos@marcosc.com>

Date: 2019-10-10

## Table of Contents
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Overview](#overview)
- [Goals and use cases](#goals-and-use-cases)
  - [Comparison to Notifications](#comparison-to-notifications)
- [Usage examples](#usage-examples)
- [Usage from service workers](#usage-from-service-workers)
- [Background updates](#background-updates)
  - [The Push problem](#the-push-problem)
  - [Periodic Background Sync](#periodic-background-sync)
  - [Possible changes to the Push API](#possible-changes-to-the-push-api)
    - [Waiving the notification requirement](#waiving-the-notification-requirement)
    - [A separate channel for Badge payloads](#a-separate-channel-for-badge-payloads)
    - [Conclusion](#conclusion)
- [Feature detection](#feature-detection)
- [Detailed API proposal](#detailed-api-proposal)
  - [The model](#the-model)
  - [The API](#the-api)
  - [UX treatment](#ux-treatment)
- [Security and Privacy Considerations](#security-and-privacy-considerations)
- [Design Questions](#design-questions)
  - [What data types are supported in different operating systems?](#what-data-types-are-supported-in-different-operating-systems)
  - [Why limit support to just an integer? What about other characters?](#why-limit-support-to-just-an-integer-what-about-other-characters)
  - [Couldn’t this be a declarative API (i.e., a DOM element), so it would work without JavaScript?](#couldnt-this-be-a-declarative-api-ie-a-dom-element-so-it-would-work-without-javascript)
  - [Is this API useful for mobile OS’s?](#is-this-api-useful-for-mobile-oss)
  - [Why is this API attached to `window` instead of `navigator` or `notifications`?](#why-is-this-api-attached-to-window-instead-of-navigator-or-notifications)
  - [Is there an upper limit on the size of the integer? And if so, what's the behavior if that limit is reached?](#is-there-an-upper-limit-on-the-size-of-the-integer-and-if-so-whats-the-behavior-if-that-limit-is-reached)
  - [Are you concerned about apps perpetually showing a large unread count?](#are-you-concerned-about-apps-perpetually-showing-a-large-unread-count)
  - [Internationalization](#internationalization)
  - [Index of Considered Alternatives](#index-of-considered-alternatives)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Overview

The **Badging API** is a proposed Web Platform API allowing websites to apply
badges (small status indicators) to pages or [installed web
apps](https://www.w3.org/tr/appmanifest/#installable-web-applications) on their
origin. The API is divided into two parts, each of which can be supported, or
not, by a user agent:

* Document badges, associated with an open document, such as when a badge is
  shown on or near the page's icon in a browser tab.
* App badges, associated with an [installed web
  app](https://www.w3.org/tr/appmanifest/#installable-web-applications) icon,
  typically in the operating system UI.

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

Unlike document badges, badges applied to apps can be shown and updated even
when there are no tabs or windows open for the app.

The reason we have separate APIs is that they have very different concerns:

* The app API has additional complexity when used with [push messages](https://www.w3.org/TR/push-api/).
* The scope and lifetime of the app API is considerably different to the document API.
* The presence of the document API itself indicates whether the tab will be
  badged in the UI (obviating the need for a dedicated feature detection method
  specifically for tab badging).

Other reasons to separate:

* Sites that only want to badge their app icon or only want to badge their tab
  can just concern themselves with the relevant API.
* The document API could, in the future, support badges that aren't supported by
  host operating systems, such as arbitrary Unicode characters.

## Goals and use cases

The purpose of this API is:

* To subtly notify the user that there is new activity that might require their attention without requiring an OS-level (banner) [notification](https://notifications.spec.whatwg.org/).
* To indicate a small amount of additional information, such as an unread count.
* To allow [Installed Web Applications](https://www.w3.org/TR/appmanifest/#installable-web-applications) to convey this information, regardless of whether they are currently open.

Non-goals are:

* To provide an arbitrary image badge. The web platform already provides this capability via favicons.

Possible areas for expansion:

* Support rendering a small status indicator (e.g., a music app shows ▶️ or ⏸️; a weather app shows ⛈️ or ⛅️): either a pre-defined set of glyphs or simply allowing a Unicode character to be rendered.

Examples of sites that may use this API:

* Chat, email, and social apps could signal that new messages have arrived.
* Any application that needs to signal that user action is required (e.g., in a turn-based game, when it is the player's turn).
* As a permanent indicator of a page's status (e.g., on a build page, to show that the build has completed).

(See [Usage examples](#usage-examples) below for the code you might use.)

### Comparison to Notifications

The Badge API goes hand-in-hand with Notifications, since both APIs provide a
way to give the user updates when they aren't directly looking at the page.

Arguably, Badge could be part of the
[Notifications API](https://notifications.spec.whatwg.org/), but we view it as a
separate, complementary API. A notification is, [by
definition](https://notifications.spec.whatwg.org/#notifications), "an abstract
representation of something that happened, such as the delivery of a message,"
whereas, at least in this context, a badge is an abstract representation of the
underlying state, such as the number of unread messages. A badge is not a
notification.

Some situations will call for just using notifications, or just using a badge
update, or both simultaneously. Commonly, you will both show a notification and
set a badge at the same time (such as when a new email message arrives). But
there are situations where it is appropriate to use the badging API without a
notification: for much higher frequency / lower priority events (such as group
chat messages not directly addressed to the user). Whenever some underlying
state changes that would be convenient for the user to be able to see at a
glance, but not important enough to disrupt the user, use of the Badge API
without a notification is warranted.

There may be no need to request permission to use the badging API, since it is
much less invasive than a notification, but this is at the discretion of the
user agent.

## Usage examples

The API is divided into two parts: one for setting/clearing a badge on the
current document, and one for setting/clearing a badge on the current
application.

To simply set a numeric badge on the current document:

```js
navigator.setClientBadge(getUnreadCount());
```

If `getUnreadCount()` (the argument to `navigator.setClientBadge`) is 0, it will
automatically clear the badge.

If you just want to show a status indicator flag without a number, use the
Boolean mode of the API by calling `navigator.setClientBadge` without an
argument, and `navigator.clearClientBadge()` (which might be done to indicate
that it is the player's turn to move in a multiplayer game):

```js
if (myTurn())
  navigator.setClientBadge();
else
  navigator.clearClientBadge();
```

All of the above set the badge *only* on the current document (e.g., in the page
tab), and won't affect any other documents, even those at the same URL. To set a
badge that applies to the current app:

```js
navigator.setAppBadge(getUnreadCount());
```

As above, a value of 0 clears the badge and `navigator.clearAppBadge()` can also
explicitly clear the badge for this installed web application. The effects of
the app API are global and may outlast the document (it is intended to persist
at least until the user agent closes). It can also be used from a service
worker, although the meaning of that will be somewhat different.

User agents may expose either or both of the two APIs, so sites should
feature-detect them individually. In particular, this allows you to fall back to
showing your own badge in the page favicon or title if the
`navigator.setClientBadge()` method is not available (regardless of the
availability of the app API). This is a complete example for a site that wants
to set the badge on both the current document and the current installed web app:

```js
// Should be called whenever the unread count changes (new mail arrives, or mail
// is marked read/unread).
function unreadCountChanged(newUnreadCount) {
  // Set the app badge, for app icons and links. This has a global and
  // semi-permanent effect, outliving the current document.
  if (navigator.setAppBadge) {
    navigator.setAppBadge(newUnreadCount);
  }

  // Set the document badge, for the current tab / window icon.
  if (navigator.setClientBadge) {
    navigator.setClientBadge(newUnreadCount);
  } else {
    // Fall back to setting favicon (or page title).
    // (This is a user-supplied function, not part of the Badge API.)
    showBadgeOnFavicon(newUnreadCount);
  }
}
```

More advanced examples are given in a [separate document](docs/examples.md).

## Usage from service workers

**TODO**: This section could use some fleshing out.

Calling the API from a service worker has some differences:

* `navigator.setClientBadge()` either shouldn't be callable from service
  workers, or it would need a `Client` argument to specify which document to
  badge.
* When `navigator.setAppBadge()` is called from a service worker, it badges all
  apps whose scope is inside the service worker scope.

## Background updates

For app badges, we would like to be able to update the badge with a server-side
push, while there are no active documents open. This would allow, for example,
app icon badges to show an up-to-date unread count even when no pages are open.

In this section, we explore two APIs that could be useful for this: [Push
API](https://www.w3.org/TR/push-api/) and [Periodic Background
Sync](https://github.com/WICG/BackgroundSync/blob/master/explainer.md#periodic-synchronization-in-design).

### The Push problem

The [Push API](https://www.w3.org/TR/push-api/) allows servers to send messages
to service workers, which can run JavaScript code even when no foreground page
is running. Thus, a server push could trigger a `navigator.setAppBadge()`.

However, there is a [de facto standard
requirement](https://github.com/w3c/push-api/issues/313) that whenever a push is
received, a notification needs to be displayed. This means it's impossible to
subtly update a badge in the background while no pages are open, without also
displaying a notification.

This may be fine for some use cases (if you want to always show a notification
when updating the badge, possibly with an exception of not showing a
notification if the app is in the foreground). But we specifically designed this
API as a *subtle* notice mechanism that does not require a more distracting
notification in order to set a badge. Because of this de facto requirement, the
Push API currently isn't suitable for this use case.

There is also a requirement that the user grants notification permission to
subscribe to push messages.

### Periodic Background Sync

[Periodic Background
Sync](https://github.com/WICG/BackgroundSync/blob/master/explainer.md#periodic-synchronization-in-design)
is a proposed extension to the [Background
Sync](https://wicg.github.io/BackgroundSync/spec/) API, that allows a service
worker to periodically poll the server, which could be used to get an updated
status and call `navigator.setAppBadge()`. However, this API is unreliable: the
period that it gets called is at the discretion of the user agent and can be
subject to things like battery status. This isn't really the use case that
Periodic Background Sync was designed for (which is having caches updated while
the user isn't directly using a site, not updating UI that's immediately visible
to the user).

That means when the page isn't open, you could have the badge indicator update
every once in awhile, but have no guarantee that it would be up to date.

(Also, the Periodic Background Sync API is not yet implemented in any browser,
though it is in an origin trial in Chrome.)

It's possible that the use of Push API and Periodic Background Sync together is
"good enough": high-priority notices (that show a notification) come in through
a Push and are displayed immediately with a notification; low-priority notices
are displayed as badges immediately if there are pages open, or "eventually" if
there are no pages open.

However, in our view, there is a need for *immediate* update to low-priority
notices: this is akin to the [urgency vs importance](https://en.wikipedia.org/wiki/Time_management#The_Eisenhower_Method)
dichotomy: a badge is for non-important information but that doesn't mean the
user doesn't want to see the notice in a timely fashion.

### Possible changes to the Push API

Here we discuss ways to potentially allow the Push API to update badges without
showing notifications.

**Note: This should not be considered blocking.** The Badge API is usable,
including from service workers, without these changes, so we consider these as
add-ons that we could introduce at a later time.

#### Waiving the notification requirement

The purpose of the [de facto
requirement](https://github.com/w3c/push-api/issues/313) to show a notification
upon receipt of a push message is to prevent malicious sites from using
high-frequency pushes to do invisible background work, such as crypto mining
(use of the user's computing resources) or monitoring the user's IP address,
giving coarse-grained geographic location (privacy concern). The theory is that
if the site is required to show a notification on each push, the user is at
least going to be aware that the site is constantly doing something, and if it's
unreasonably spammy, the user is likely to revoke the notification permission,
thus erasing the push subscription.

We could make it so that use of the Badge API serves the same function as
showing a notification (fulfilling the requirement to use the Push API). But
doing so would probably nullify the reason for this requirement in the first
place: a malicious site that wants to do crypto mining could just set a badge
every 30 seconds and the user would probably not notice it. And they could set
the badge to the same value it already had (which we definitely want to allow;
otherwise the server has to keep track of what value is currently being
displayed in each client).

On the other hand, this may be acceptable if we allow user agents to dictate a
high bar for waiving this requirement, e.g., "the user must have [installed the
site](https://www.w3.org/TR/appmanifest/#installable-web-applications) as an
application, and enabled notifications". It may be an acceptable trade-off to
allow these "semi-trusted" sites to perform silent background tasks, in order to
let them keep badges up to date in a timely fashion. This would have to be a
discussion around privacy trade-offs which we (the Badge API authors) aren't
equipped to answer by ourselves.

Technically, the way we would do this is by allowing certain sites to set
[`userVisibleOnly`](https://www.w3.org/TR/push-api/#dom-pushsubscriptionoptions-uservisibleonly)
to `false` in the push subscription (which currently has no defined meaning, and
in Chrome at least, is not allowed to be `false`, so there is currently a de
facto requirement that it be set to `true`). If a site is allowed to set
`userVisibleOnly` to `false`, then it can receive badge-only push messages. If
not, it is bound by the existing rules, and must either show a notification, or
turn off low-priority messages. This might require user opt-in (or a separate
permission to notifications).

#### A separate channel for Badge payloads

A more comprehensive solution includes some significant additions to the Push
API (which the authors of the Badge spec are not equipped to do).

(This has been briefly discussed with Peter Beverloo, (@beverloo) an editor of
the Push API spec, but only at a very high level.)

Instead of allowing the service worker to run JavaScript code and call the
Badge API, we would introduce a new concept to a Push subscription called a
"channel", which dictates what is done with the payload upon receipt. The
default channel would be "event" (the payload is delivered to the `"push"`
event), with a new channel, "badge", which imposes a specific format to the
payload. The payload would now be interpreted as a JSON dictionary containing
parameters to the `navigator.setAppBadge()` method. Upon receipt of the push,
the user agent would automatically call the Badge API without running any user
code, and with no requirement to show a notification.

This solution still has the following two flaws (expressed by Peter Beverloo):

1. The server can still use badge delivery to know whether device is online
   through [delivery receipts](https://tools.ietf.org/html/rfc8030#section-6.3).
   This may or may not be a privacy concern.
2. (On mobile) (at least on Android and iOS) the browser needs to be woken up to
   decrypt and process the payload; it can't be processed by the host OS's push
   receipt system due to the encryption. This could be a big resource drain if
   badges are pushed too frequently (the same concern applies for notifications,
   but as above, there is a natural tendency for developers to reduce the number
   of user-visible notifications due to user spam; the same forcing function
   does not apply for setting a badge).

These could be partly mitigated by introducing a throttle mechanism, which would
mean badge updates can't be immediate (moving towards Periodic Background Sync),
but at least we would be able to set a throttle sensible for badge updates,
rather than the total lack of guarantees that PBS gives us.

#### Conclusion

Both of the above changes present huge obstacles and discussions involving
privacy, resource usage, and utility trade-offs. Due to the increased
complexity, we are not considering changes to the Push API at this time.

## Feature detection

Ideally we would not provide any feedback to the site as to how the badge is
going to be displayed to the user, since it's a user interface concern that
should be at the discretion of the user agent. For example, the site shouldn't
need to know whether the badge is displayed on a tab icon or on an app icon.

However, a practical concern overrides this: because the favicon can be (and
often is) used to display a page badge, sites upgrading to the Badge API need
to know whether they need to fall back to setting the favicon (since,
presumably, they don't want the badge being displayed twice). So we need to
provide a way to feature detect — not just "whether the Badge API is supported",
but "whether a Badge-API badge will show up on or near the favicon".

Therefore, we will explicitly specify that if the `navigator.setClientBadge()`
method exists, the user agent MUST show a badge set through this method on or
near the favicon. Therefore, sites can reliably use the presence of this method
to fall back to a conventional favicon or title badge:

```js
if (navigator.setClientBadge) {
  navigator.setClientBadge(getUnreadCount());
} else {
  // Fall back to setting favicon (or page title).
  // (This is a user-supplied function, not part of the Badge API.)
  showBadgeOnFavicon(getUnreadCount());
}
```

A user agent could provide the `setAppBadge()` methods without
`setClientBadge()`, which means the above check will apply the fallback, while
app icons can still be badged.

## Detailed API proposal

### The model

A badge is associated with either a document, or an [installed app](https://www.w3.org/TR/appmanifest/#installable-web-applications).

At any time, the badge for a specific document or app, if it is set, may be either:

* A "flag" indicating the presence of a badge with no contents, or
* A positive integer.

The model does not allow a badge to be a negative integer, or the integer value 0 (setting the badge to 0 is equivalent to clearing the badge).

**Important distinction**: A "flag" badge generally displays a visual indicator to the user (such as a dot or circle), whereas a cleared badge (value 0 or calling `clearAppBadge()`) clears it. These are semantically different states, and platforms generally won't treat a request to set a "flag" as equivalent to clearing the badge.

The user agent is allowed to clear all badges on an origin whenever there are no foreground pages open on the origin (the intention of this is so that when the user agent quits, it does not need to serialize all the badge data and restore it on start-up; sites should re-apply the badge when they open).

### The API

The Badge API methods are members of interface is a member object on
[`Navigator`](https://html.spec.whatwg.org/multipage/system-state.html#the-navigator-object)
and
[`WorkerNavigator`](https://html.spec.whatwg.org/multipage/workers.html#workernavigator).
They are as follows:

* `navigator.setClientBadge([contents])`: Sets the badge for the current
  document to *contents*, or to "flag" if *contents* is omitted. If *contents*
  is 0, clears the badge for the given document.
* `navigator.clearClientBadge()`: Clears the badge for the current
  document.
* `navigator.setAppBadge([contents])`: Sets the badge for the matching app(s) to
  *contents* (an integer), or to "flag" if *contents* is omitted. If *contents*
  is 0, clears the badge for the matching app(s).
* `navigator.clearAppBadge()`: Clears the badge for the matching app(s).

The **matching app(s)** has a different meaning depending on the context:

* If called from a document, this refers to an installed app that this document
  is [within
  scope](https://www.w3.org/TR/appmanifest/#dfn-within-scope-manifest) of. If
  multiple apps match, it is the one with the most specific scope. If none
  match, the method has no effect. (Zero or one apps.)
* If called from a service worker, this refers to all apps whose scope URL
  [matches the service worker
  registration](https://www.w3.org/TR/service-workers-1/#scope-match-algorithm)
  of this service worker. (Zero or more apps.)

**Note**: Should we have a separate overload for boolean flags now, as discussed in [Issue 19](https://github.com/w3c/badging/issues/19) and [Issue 42](https://github.com/w3c/badging/issues/42)?

### UX treatment
Badges may appear in any place that the user agent deems appropriate. In general, these places should be obviously related to the pages being badged, so users understand what the status is for.

Appropriate places for a document badge could include:

- Tab favicons.
- OS-specific contexts for app window icons if the document is open in a [standalone window](https://www.w3.org/TR/appmanifest/#display-modes).

App badges are shown in OS-specific contexts. User agents should attempt reuse existing [operating system APIs and conventions](docs/implementation.md), to achieve a native look-and-feel for the badge.

### Implementation Considerations for Platform Limitations

Different operating systems have varying support for badge types. Since the OS ultimately controls badge rendering, it is the user agent that communicate the correct semantic intent:

* **Flag support**: Some platforms do not natively support "flag" badges (badges without numbers). In such cases, user agents communicate to the OS using the closest available representation that conveys badge presence (such as a generic symbol or the number "1") rather than requesting badge clearing.
* **Number support**: Some platforms might not support numbered badges. In such cases, user agents can communicate the badge to the OS as a flag representation.
* **Semantic preservation**: User agents preserve the semantic distinction between "flag" (show something) and "nothing" (show nothing) when communicating with the operating system, even when platform capabilities are limited.
* **OS control**: The ultimate display behavior is controlled by the operating system and can be further controlled by user settings or system conventions.

This approach ensures consistent semantic behavior across platforms and prevents the issues identified in implementations where `setAppBadge()` (flag) incorrectly clears badges instead of displaying them.

## Security and Privacy Considerations
The API is write-only, so data badged can't be used to track a user. Whether the API is present could possibly be used as a bit of entropy to fingerprint users, but this is the case for all new APIs.

These methods are only usable from secure contexts, so forged pages can't set badges on behalf of an origin they don't control.

If the badge methods are called from inside an iframe, the client API should have no effect, and the app API should apply to the app enclosing the iframed contents' URL, not the containing page's URL.

There are additional privacy considerations relating to the proposed extensions to the Push API, noted above. However, this does not apply to the base Badge API.

## Design Questions

### What data types are supported in different operating systems?

See [implementation.md](docs/implementation.md).

### Why limit support to just an integer? What about other characters?

It isn't a technical limitation, it's an attempt to keep behavior as consistent as possible
on different host platforms (UWP only supports a set of symbols, while iOS, Android
and Ubuntu don't support them at all).

Limiting support to integers makes behavior more predictable, though we are considering
whether it might be worth adding support for other characters or symbols in future.

### Couldn’t this be a declarative API (i.e., a DOM element), so it would work without JavaScript?
This would make sense for the client API, but not the app API (which is
shared between different pages and has a lifetime beyond the page).

For now, we're specifying both as a JavaScript API to keep them consistent, but
we could certainly change the document API to be a DOM element (e.g. `<link rel="shortcut icon badge" href="/favicon.ico" badge="99">`).

### Is this API useful for mobile OS’s?
iOS has support for badging APIs (see [iOS](docs/implementation.md#ios)).

On Android, badging is blocked at an OS level, as there is [no API for setting a badge without also displaying a notification](#android). However, a badge will already be displayed if a PWA has pending notifications (it just doesn’t allow the fine grained control proposed by this API).

To summarize: This API cannot be used in PWAs on either mobile operating system. Support on iOS is blocked until Safari implements the API and Android does not have an API for controlling badges. Should either situation change, the badging API will become trivially available.

### Why is this API attached to `navigator` instead of `window` or `notifications`?

There is more detail and discussion in [Issue 55](https://github.com/w3c/badging/issues/55).

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

### Index of Considered Alternatives
- The "app" API being a more general API that [sets a badge on a URL scope](https://github.com/w3c/badging/issues/55), applying to all apps within that scope.
- Setting an app badge badges [app associated with the current document's linked manifest](https://github.com/w3c/badging/issues/55), as opposed to any app that scopes this document.
- A single URL-scoped API that sets both the document and app badges at the same time (applying to all documents within the URL scope).
- A [declarative API](#Couldnt-this-be-a-declarative-API-so-it-would-work-without-JavaScript).
- Exposing the badging API [elsewhere](#Why-is-this-API-attached-to-window-instead-of-navigator-or-notifications).
- Supporting [non-integers](#Why-limit-support-to-just-an-integer-What-about-other-characters).
- Use in the [background](#Why-cant-this-be-used-in-the-background-from-the-ServiceWorker-see-28-and-5).
- [Separate methods](https://github.com/w3c/badging/issues/19) for setting/clearing boolean flags and numbers.
- Exposing a [getter](https://github.com/w3c/badging/issues/18) for badge contents.
- Only [badging](https://github.com/w3c/badging/issues/1) [PWAs](https://github.com/w3c/badging/issues/12).
- Supporting [query-string scopes](https://github.com/w3c/badging/issues/1#issuecomment-511634128).
- Adding [fallbacks](https://github.com/w3c/badging/issues/2), when the system can't display a badge.
- [Promise based](https://github.com/w3c/badging/issues/35#issue-459665145) badging API.
