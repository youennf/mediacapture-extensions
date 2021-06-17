# MediaStreamTrack video frame access playgorund #

## Scope ##

An API for processing raw video that is more efficient than canvas.
It should allow web application specific processing using existing techniques: WASM, WebGPU, WebGL.
It should limit risks in terms of memory and threading.

Known use cases are:
* Background removal.
* Funny Hats.


## API Examples ##

### Head detection ###

<pre>
// Window
const stream = await navigator.mediaDevices.getUserMedia({ video : true });
const track = stream.getVideoTracks()[0];
const worker = new Worker("head-detection-worker.js");
worker.onmessage = (event) => { displayNotification(event.data); };
worker.postMessage({ track : track }, [track]);
</pre>

<pre>
// Worker
async function doHeadDetection(frame)
{
     // Pick your prefered algoithm.
     return false;
}

onmessage = (event) => {
    event.data.track.<b><i>processVideoFrame</i></b> = async (frame) => {
        if (await doHeadDetection(frame))
            event.source.postMessage("Head detected");
    };
}
</pre>

### Funny hat as a transform ###
<pre>
// Window
const stream = await navigator.mediaDevices.getUserMedia({ video : true });
const track = stream.getVideoTracks()[0];
const worker = new Worker("dragon-head-transform-worker.js");
const promise = new Promise(resolve => worker.onmessage = (event) => resolve(event.data.track));
worker.postMessage({ track : track }, [track]);
const transformedTrack = await promise;
// Use transformedTrack as if you would use track, including setting enabled, apply constraints, or receiving muted/unmuted events.
</pre>

<pre>
// Worker
async function doDragonHeadTransform(frame)
{
     // Pick your prefered algorithm for adding a dragon head.
     return frame;
}

onmessage = (event) => {
    const track = event.data.track;
    const transformedTrack = MediaStreamTrack.<b><i>createVideoTrack</i></b>({
        start: (c) => {
            this.controller = c;
            // transform each frame
            track.processVideoFrame = async (frame) => c.enqueue(await doDragonHeadTransform(frame));
            // Optional steps to fully shim track behavior:
            // 1. forward muted information to transformedTrack.
            track.onmuted = () => c.muted = true;
            track.onunmuted = () => c.muted = false;
            // 2. initialize constraints, capabilities and settings.
            c.capabilities = track.capabilities();
            c.constraints = track.constraints();
            c.settings = track.settings();
        },
        // 3. forward constraints to track
        applyConstraints: async (constraints) => {
            await track.applyConstraints(constraints);
            this.controller.constraints = track.constraints();
            this.controller.settings = track.settings();
        },
        enabledChanged: (value) => track.enabled = value,
        stop: () => track.processVideoFrame = null
    });
    self.postMessage({ track : transformedTrack }, [transformedTrack]);
});
</pre>


## MediaStreamTrack Video Frames Access ##

<pre>
partial interface MediaStreamTrack {
   [Exposed=Worker] attribute MediaStreamTrackVideoFrameCallback processVideoFrame;
};
callback MediaStreamTrackVideoFrameCallback = any(VideoFrame);
</pre>

Given a MediaStreamTrack object named track, the processing rules of processVideoFrame are the following:
- When receiving a video frame from the track source, and track is not busy processing a video frame, call processVideoFrame.
- When processVideoFrame is called, track is then busy processing a video frame during the execution of processVideoFrame and until processVideoFrame returned promise is settled.
- The video frame exposed by processVideoFrame remains valid as long as the processVideoFrame returned promise is not settled.
- When processVideoFrame returned promise is settled, the video frame gets closed and the track is no longer busy processing a video frame.
- Pages that want to keep  more than one frame at a time need an explicit step to do so. This explicit step (cloning a video frame e.g.) may potentially induce a memory copy.
- When receiving a video frame from the track source, and track is busy processing a video frame, the track may buffer the video frame or drop it. This is not under the control of the page.

Advantages:
* Simple, straightforward API: transfer the track to a worker and set a callback to process frames.
* Consistent with other event-based MediaStreamTrack mechanisms: muted/unmuted/ended.
* Possibility to be consistent with applyConstraints: change size through applyConstraints, promise resolves than video frames size change in processVideoFrame.
* Minimal memory risk by dropping frames as needed and limiting the default video frame life time.
* Ability for the User Agent to apply backpressure when the track is busy.
* Possiblity for the User Agent to enqueue frames in a circular buffer. This may be done depending on the source implementation (camera buffer pool) and device load for instance.
* Possibility for implementation optimizations: cloning a video frame may be lazily done and deep copy only happen in case of buffer pool memory warning.


## MediaStreamTrack JS Source ##

<pre>
partial interface MediaStreamTrack {
    [Exposed=Worker] static MediaStreamTrack createVideoTrack(MediaStreamTrackVideoSource videoSource);
};

dictionary MediaStreamTrackVideoSource {
    MediaStreamTrackVideoSourceStartCallback start;
    MediaStreamTrackVideoSourceStopCallback stop;
};

callback MediaStreamTrackVideoSourceStartCallback = any(MediaStreamTrackVideoController);
callback MediaStreamTrackVideoSourceStopCallback = any();

[Exposed=Worker]
interface MediaStreamTrackVideoController {
    undefined enqueue(VideoFrame);
    undefined close();
};
</pre>

Advantages:
* Simple API: create a track in a worker from  a JS source and transfer it to Window to use it as a regular MediaStreamTrack
* Reuse of ReadableStream JS-based source model without passing ReadableStream/WritableStream directly:
  * Easier extensibility
  * No plan for native ReadableStream<VideoFrame> since we have MediaStreamTrack.

* Ability to extend the API:
  * Expose hook to expose the enabled state as callbacks if needed.
  * Expose hook to let the application control muted/unmuted if needed (as part of MediaStreamTrackVideoController for instance).
  * Expose hook to add constraints (expose capabilities, implement applyConstraints as JS callback, as a way to fully shim existing native sources like cameras with JS-based MediaStreamTrack).
  * Expose hook to enable backpressure signaling using a pull callback or having enqueue returning a promise, (once we agree on a sink model for native sinks like HTMLMediaElement or RTCRtpSender).
  * Expose API to link a JS-based MediaStreamTrack with another MediaStreamTrack to propagate signals (muted/unmuted, ended...) back and forth automatically. 

### Possible Extensions ###

Support of toggling muted/unmuted can be added by adding dedicated methods to the controller.
<pre>
partial interface MediaStreamTrackVideoController {
    attribute boolean muted;
};
</pre>
If muted is set to true, MediaStreamTrackVideoController.enqueue becomes a no-op.

Support of applyConstraints can be added by adding the following fields to MediaStreamTrackVideoSource and MediaStreamTrackVideoController:
<pre>
partial dictionary MediaStreamTrackVideoSource {
    MediaStreamTrackVideoSourceApplyConstraintsCallback applyConstraints;
};
callback MediaStreamTrackVideoSourceStartCallback = Promise(MediaTrackConstraints constraints);

partial interface MediaStreamTrackVideoController {
    attribute MediaTrackCapabilities capabilities;
    attribute MediaTrackConstraints constraints;
    attribute MediaTrackSettings settings;
};
</pre>
Initial capabilities, constraints and settings can be set directly when creating the JS source track in MediaStreamTrackVideoSource.start callback.
When applyConstraints is called on the JS source track, the MediaStreamTrackVideoSource.applyConstraints callback will be called.
As part of applying new constraints, the video controller can be used again to update the track capabilities, constraints and settings.

It can prove to be useful to add a callback to react to enabled value changes, the following could be added:
<pre>
partial dictionary MediaStreamTrackVideoSource {
    MediaStreamTrackVideoSourceEnabledChangedCallback enabledChanged;
};
callback MediaStreamTrackVideoSourceEnabledChangedCallback = undefined(boolean);
</pre>
Whenever the track enabled value is changed, enabledChanged will be called.
This may allow the JS source to stop/reduce processing if the track enabled value is false.

## MediaStreamTrack Transform ##

We could just reuse above APIs and do not special case transforms.
Native transforms could be objects that take a video frame as input and generate a video frame as output.
Native transforms could also take a MediaStreamTrack as input and generate a MediaStreamTrack as output.

That said, transforming a track into another track might be a usual pattern and it might be worth exposing it to get some potential benefits like:
* Clear control signal: source signals are easily propagated. Sink signals are easily propagated.
* Ability for User Agent to control frame buffering and drop frames if needed.
* Direct piping of video frames without JS in the middle in the case of native transforms.
* Ease of use: no need to transfer MediaStreamTracks explicitly to a worker.

This transform API could be done as a secondary step, if the transform pattern becomes popular or if native transforms become needed.

<pre>
// MediaStreamTrackVideoTransform may include future native transforms.
typedef MediaStreamTrackVideoScriptTransform MediaStreamTrackVideoTransform;

[Exposed=Window]
interface MediaStreamTrackVideoScriptTransform {
    constructor(Worker worker, optional any options, optional sequence<object> transfer);
};

partial interface MediaStreamTrack {
    Promise&lt;MediaStreamTrack?&gt; transform(MediaStreamTrackVideoTransform transform);
};

[Exposed=Worker]
interface MediaStreamTrackVideoTransformEvent : Event {
    readonly attribute MediaStreamTrackVideoTransform transform;
};

partial interface DedicatedWorkerGlobalScope {
    attribute EventHandler onvideotracktransform;
};

[Exposed=Worker]
interface MediaStreamTrackVideoTransform {
    readonly attribute any options;
    // Reading callback.
    attribute MediaStreamTrackVideoFrameCallback processVideoFrame;
    // Writing callbacks.
    attribute MediaStreamTrackVideoSourceStartCallback start;
    attribute MediaStreamTrackVideoSourceStopCallback stop;
};
</pre>
