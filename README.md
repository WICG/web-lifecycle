# A Lifecycle for the Web
## Motivation
For detailed motivation see [this doc](https://docs.google.com/document/d/1UuS6ff4Fd4igZgL50LDS8MeROVrOfkN13RbiP2nTT9I/edit#heading=h.ejq0luje6eyb).

With large numbers of web apps (and tabs) running, critical resources such as memory, CPU, battery, network etc easily get oversubscribed, leading to a bad end user experience.
Application lifecycle is a key way that modern OS' manage resources. On Android, iOS and also more recent Windows versions, apps can be started and stopped at will by the platform. This lets the platform streamline and re-allocate resources where they best benefit the user.

On the web, we’ve tackled this with one-off features eg. reactive tab-discarding in extreme memory pressure - which can break websites.
While this is okay in the short term, in the long term it is important to incorporate first class support in the web platform, create the right incentive structure for web developers, and allow the system to proactively reallocate resources and avoid getting into extreme resource situations.

For a platform to support application lifecycle, it needs to both:
* provide developers with signals about transitions between the lifecycle states
* provide lifecycle-compatible APIs that allow key capabilities to work even when the app is backgrounded or stopped.

The web ecosystem lacks a clear lifecycle. This proposal attempts to define what the lifecycle of a web page is and add necessary extensions to enable supporting two important system interventions necessary for resource re-allocation:
* Tab discarding for memory saving
* CPU stopping for battery saving

Whereas mobile platforms have rich service-bound APIs that allow apps to deliver their experience when backgrounded, most of the web platform's capabilities are tab-coupled. Audio for instance only works when the tab is alive, so when a tab is killed in the background that plays audio, there is no way to keep that tab playing sound. A [list of background use-cases is here](https://docs.google.com/document/d/1UuS6ff4Fd4igZgL50LDS8MeROVrOfkN13RbiP2nTT9I/edit#heading=h.5kyzj3e4880y). In an ideal world, web apps would be able to deliver the experience they want to their users, without having to rely on their page always being resident and running on the machine.

### Lifecycle States
![Lifecycle States](https://github.com/spanicker/web-lifecycle/blob/master/LifecycleStates.png)

For details on the app lifecycle states and definitions see [this doc](https://docs.google.com/document/d/1UuS6ff4Fd4igZgL50LDS8MeROVrOfkN13RbiP2nTT9I/edit#heading=h.edtdhepwctwy).\

This proposal formalizes states for STOPPED and DISCARDED.
- Lifecycle states apply to frames: both toplevel and nested.
- When a background tab is transitioned to STOPPED, the entire frame tree will be consistently moved to STOPPED
- It is possible for an ACTIVE or PASSIVE tab to have some frames in ACTIVE and other frames in STOPPED state.

Lifecycle State | Visibility | Developer Expectation | System Interventions
--------------- | ---------- | --------------------- | --------------------
STOPPED | Typically HIDDEN frames will be STOPPED. It is possible for visible frames to be STOPPED | Hand off for background work and stop execution. Teardown and release resources. Report to analytics | CPU suspension: stop CPU after N minutes based on resource constraints
DISCARDED | Typically STOPPED frames will be moved to DISCARDED. It is possible for PASSIVE frames to be DISCARDED | System has discarded background tab to reclaim memory. If user revisits tab, this will reload the tab. | Tab discarding for memory saving: fully unloaded, no memory consumption.

### End-of-life scenarios
There are 3 high level scenarios for “end-of-life”.
#### 1. System Exit (Interventions)
The system moves the app to STOPPED state and stops CPU usage, or the system moves the app to DISCARDED state and discards the app to reclaim memory. Handling this is in-scope for this proposal.\
For detailed Scenarios and Requirements, see the [list here](https://docs.google.com/document/d/1UuS6ff4Fd4igZgL50LDS8MeROVrOfkN13RbiP2nTT9I/edit#heading=h.rsruvllnv993).

On system exit, there is no guaranteed callback at the (very) time of system exit. This is consistent with mobile platforms (Android and iOS): in Android onPause is the guaranteed callback for user exit, on iOS the equivalent is willResignActive.
On Android and iOS the system kills background apps that were previously stopped; corresponding callbacks have already fired and there is no callback before system kill.

#### 2. User Exit
The user may close the tab (foreground or background) or navigate away OR on mobile, swipe the app away from task switcher. The user may background the app by minimizing the window OR on mobile by going to the homescreen and task switcher.\
On user exit, the browser should **guarantee** that *one* callback will fire and finish, before the app is torn down.

For categories of work that happen in end-of-life see the [list of End-of-life use-cases here](https://docs.google.com/document/d/1UuS6ff4Fd4igZgL50LDS8MeROVrOfkN13RbiP2nTT9I/edit#heading=h.qftifuoc2315).

#### 3. Unexpected Termination
Apps can get killed in scenarios where it is not possible to deliver a callback, such as OOM crashes, OS kills the process under memory pressure, crashes or hangs due to browser bugs, device runs out of battery etc. Therefore it is possible for apps to transition from any state to TERMINATED without any callback being fired.\

It is not possible to have a guaranteed callback execute in most of these scenarios.

## Proposal (MVP)
![Lifecycle Callbacks](https://github.com/spanicker/web-lifecycle/blob/master/LifecycleCallbacks.png)

We propose the following changes:

* `onstop` is fired to signal transition to STOPPED.
* `onsrestart` is fired to signal transition out of STOPPED. This will be used to undo what was done in `onstop` above. 
* On DISCARDED -> ACTIVE, an attribute called `wasDiscarded` is added to the Document. This will be used to restore view state , when the user revisits a discarded tab.

Suggestion for implementers: before moving app to DISCARDED it is recommended to run `beforeunload` handler and if it returns string (i.e. needs to show modal dialog) then the tab discard should be omitted, to prevent risk of data loss.

### Reusing existing callbacks vs. Adding new callbacks
A previous version of this proposal reused pagehide / pageshow callbacks.
With the requirement that visible and occluded (ACTIVE & PASSIVR) frames can be stopped (not just HIDDEN frames), the cons really outweighed the pros of reusing. For detailed pros and cons see [here](https://docs.google.com/document/d/1UuS6ff4Fd4igZgL50LDS8MeROVrOfkN13RbiP2nTT9I/edit#heading=h.5l9dky87m2l0)

### API sketch
```
// Indicate what is stopped exactly: 
// a. partial frame tree starting with current frame
// b. partial frame tree starting with an ancestor frame
// c. entire page in background
// d. ...
enum FrameLevel { ... };

interface StopEvent : Event {
    readonly attribute FrameLevel frameLevel; 
}

interface RestartEvent : Event {
 readonly attribute FrameLevel frameLevel; 
}
```

Handle transition to STOPPED
```
function handleStopped(e) {
   // Handle transition to STOPPED
}
window.addEventListener("stop", handleStop);

OR
window.onstop = function() { … }
```
NOTE: subsequently the app may get discarded, without firing another callback.

Handle transition out of STOPPED
```
function handleRestart(e) {
    // handle state transition STOPPED -> ACTIVE
}
window.addEventListener("restart", handleRestart);

OR
window.onrestart = function() { … }
```

### Callbacks in State Transition Scenarios
* A. System stops (CPU suspension) background tab; user revisits\
[HIDDEN] -------------> `onstop` [STOPPED]\
--(user revisit)----> `onrestart` [ACTIVE]

* B. System discards stopped tab; user revisits\
(previously called `onstop`----> [STOPPED]\
----(tab discard)----> <no callback here> [DISCARDED]\
--(user revisit)----> [LOADING] -> (`Document::wasDiscarded` is set) [ACTIVE]

* C. System discards background tab; user revisits\
[HIDDEN] ---(tab discard)------>\
`onstop` [STOPPED] ---(system tab discard)---> [DISCARDED]\
--(user revisit)----> [LOADING] -> (`Document::wasDiscarded` is set) [ACTIVE]

State Transition | Lifecycle Callback | Trigger | Expected Developer Action
---------------- | ------------------ | ------- | -------------------------
ACTIVE -> HIDDEN | `onpagevisibilitychange: hidden` (already exists) | Desktop: tab is in background, or window is fully hidden; Mobile: user clicks on task switcher or homescreen | stop UI work; persist app state; report to analytics
HIDDEN -> ACTIVE | `onpagevisibilitychange`: `visible` (already exists) | User revisits background tab | undo what was done above; report to analytics
HIDDEN -> STOPPED | `onstop` | System initiated CPU suspension; OR user navigate with bfcache | report to analytics; teardown, release resources; hand off for background work and stop execution. Save transient UI state in case app is moved to DISCARDED.
STOPPED -> ACTIVE | `onrestart` | user revisits STOPPED tab or navigates back (bfcache) | undo what was done above; report to analytics
STOPPED -> DISCARDED | (no callback) | System initiated tab-discard | (no advance warning here)
DISCARDED -> ACTIVE | (`Document::wasDiscarded` is set) | user revisits tab after system tab discard | restore transient UI state

### Restrictions and Capabilities in proposed callbacks
If excessive work is performed in the `onstop` callback fired on STOPPED, there is a cost to this in terms of resource consumption i.e. CPU, network.
We need to strike a balance between enabling the system to move the app to STOPPED for conserving resources AND enabling the app to take action without consuming excessive resources in these callbacks.
To accomplish this, certain restrictions are needed in these callbacks, ideally:
- upper time limit in the callback i.e. allowed wall time eg. 5s
- upper limit on allowed CPU time
- Maybe restrictions on network eg. disallow network except sendBeacon / Fetch keep-alive

Separately, it is useful for apps to be able to do legitimate async work in these callbacks such as writing to IndexedDB. We are exploring support for [ExtendableEvent.waitUntil](https://developer.mozilla.org/en-US/docs/Web/API/ExtendableEvent/waitUntil) API to do async work eg. IndexedDB writes.

### Guarantess for end-of-life callbacks

**What should be the “guaranteed” callback for the Web?**

We should align with the mobile model (Android and iOS) on the web. For this we need to ensure the following:
1. on user exit: only one callback is guaranteed to fire and complete
2. on system exit: above callback should have already fired, yet another callback is not guaranteed to fire (guarantee is simply not possible in many cases)

For #1, ideally all apps will transition through PASSIVE state before they can be killed and potentially we could, in the future, introduce a new callback here -- that is guaranteed.
In practice though, there is already a callback that is (almost) guaranteed - this is pagevisibility (although there are bugs in browsers, causing it to not fire in some cases).
For instance on mobile web, if the user goes to the homescreen OR task-switcher and then swipes away, then pagevisibility=hidden will fire (on homescreen, task-switcher) no other callback is fired on swipe (unload, pagehide etc).
Adding callback for transition to PASSIVE is not urgent, and will be considered in the future.

While unload callback is widely used, it is fundamentally unreliable, for instance it does not fire on mobile if user goes to task-switcher and then swipes. There are currently no plans to make unload more reliable. (The long term vision is to replace it with declarative APIs for desktop)

For #2, callback for STOPPED state is not guaranteed on user exit scenarios.
On system exit scenarios, typically STOPPED callback (and pagevisibility=hidden for sure) would have already fired previously BUT there is no guarantee that STOPPED callback *must* have fired.

### Further Reading
For details on the following topics see the Master Doc:
* [Persisting Transient View State](https://docs.google.com/document/d/1UuS6ff4Fd4igZgL50LDS8MeROVrOfkN13RbiP2nTT9I/edit#heading=h.9u8nhnl3oez)
* [Handling Background Work and role of Service Worker](https://docs.google.com/document/d/1UuS6ff4Fd4igZgL50LDS8MeROVrOfkN13RbiP2nTT9I/edit#heading=h.w3vi1ouug35y)
* [Alternatives Considered](https://docs.google.com/document/d/1UuS6ff4Fd4igZgL50LDS8MeROVrOfkN13RbiP2nTT9I/edit#heading=h.ubo7g7vcr9ri)

