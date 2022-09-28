# Continuity State API explainer

## Introduction

Browsers provide ways for a user to easily load a web page already opened in one device in another device of the same user.
For some of these flows, the user expects to fully transition its interactions from the old page to the new page.
We call these flows handof flows.

As an example, when migrating a video call from one device to another, the user may want the old web page to stop capturing
as soon as the new web page is operational.

This proposal allows the old web page to provide some state to the new web page.
It also allows the new web page to identify that it was opened as part of a handoff flow.

## Goals 

* Allow a web application to understand that it is being opened as a part of a user migrating a web page from one device to another.
* Allow a web application to share some application-specific information between the page opened in the initial device and the page
  opened on the new device.

## Non-goals

* It is not a goal to transfer the full execution state of a web application.

## Alternatives

* There is currently no way for a web page to know whether it was opened as part of a handoff flow or as part of another flow.
  This makes it hard for the web application to know whether migration between the two pages should happen.
* State can be shared between one page to the other by updating the web page URL, in particular its fragment identifier.
  The downsides is that this might leak private information if the user decides to share the link via text messaging, email...
  Also, the web page URL is not as trusted as the continuity state information which is solely managed by the UA.
* A HTTP header could be used to convey the continuity state information to the server instead of the page.
  The benefits over a readonly attribute are small and this would limit the continuity state to be a valid HTTP header value.


## Using the continuityState API

## Handoff API

```js
partial interface Navigator {
    readonly attribute DOMString initialContinuityState;
    attribute DOMString continuityState;
};
```

## Example

```js
// Initial page is updating the continuity state as the user interacts with the page.
let state = { };
onusernameFilled = (username) => {
    state.username = username;
    navigator.continuityState = JSON.stringify(state);
}

oncameraEnabled = (enabled) => {
    state.cameraEnabled = enabled;
    navigator.continuityState = JSON.stringify(state);
}

onmicrophonEnabled = (enabled) => {
    state.microphoneEnabled = enabled;
    navigator.continuityState = JSON.stringify(state);
}
```

```js
let state = { };
window.onload = async () => {
     // New page is reading the continuity state.
     if (navigator.initialContinuityState) {
          state = JSON.parse(navigator.initialContinuityState);
          initializeFromState();
          return;
     }
     // Else, request access to the room.
     await waitForUserToInputUserName();
     const allowed = await requestRightToEnterRoom();
     if (allowed)
         showJoinButton();
}

async function initializeFromState()
{
     // Let's initialize web page according latest user preferences from the old page.
     updateUsername(state.username);
     if (state.cameraEnabled || state.microphoneEnabled) {
         try {
             initializeLocalView(await navigator.mediaDevices.getUserMedia({ audio: state.microphoneEnabled, video: state.cameraEnabled });
         } catch (e) {
             // update state.
         }
     }
     showJoinButton();
}
```

