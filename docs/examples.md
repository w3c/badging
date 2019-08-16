# Advanced Examples

These examples generally provide more context around what is going on, and may specify scopes. They are intended to show how real world applications may use the API.

## Table of Contents
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Updating a Badge on a message from a WebSocket (as in a messaging app, receiving new messages):](#updating-a-badge-on-a-message-from-a-websocket-as-in-a-messaging-app-receiving-new-messages)
- [Setting a separate badge for the app and a specific page (as in the case of GitHub notifications and PR statuses).](#setting-a-separate-badge-for-the-app-and-a-specific-page-as-in-the-case-of-github-notifications-and-pr-statuses)
- [Badging for Multiple Apps on the Same Origin (as in the case of multiple GitHub Pages PWAs)](#badging-for-multiple-apps-on-the-same-origin-as-in-the-case-of-multiple-github-pages-pwas)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Updating a Badge on a message from a WebSocket (as in a messaging app, receiving new messages):

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

## Setting a separate badge for the app and a specific page (as in the case of GitHub notifications and PR statuses).
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

## Badging for Multiple Apps on the Same Origin (as in the case of multiple GitHub Pages PWAs)
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

