# Gorilla Overview
This file contains an overview of the changes/additions made to the original webgazer source code.  The main focus of these is improving Webgaer interoperability with managed systems.  In short, Webgazer is intended to be used, out-of-the-box, without any other tools or capabilities assumed.  However, online tools for behavioural science will have their own systems for managing event timing, accessing and monitoring external hardware. The changes made allow Webgazer to more easily integrate with such a system, minimising disruption.
As such, there is nothing specific to Gorilla in these changes - they could equally be used by other online tool providers.

tl;dr These changes include
1. Connecting Webgazer with an existing mediastream (rather than always requesting it's own)
2. Disconnecting prediction from the requestAnimationFrame call so they can be called for in a separate thread (requestAnimationFrame is a critical resource for accurate display timing in the browser. Heavy data processing within it can slow down frame draw and disrupt display timings)
3. Provide more initialisation callbacks, to allow Webgazer's systems to be prepared and, waited for, appropriately

Within the Webgazer.js files, changes are typically marked in the file with a comment
```
// GORILLA DEV
```
If you see any changes to the file that are not marked in that way, please let us know!


### Connecting Webgazer with an existing mediastream

Gorilla has a number of components that require the participants webcam to function, such as Video Recording and shared Web Cam chat for Multiplayer tasks. As such, we have a centralised system for requesting access to a participants webcamera and maintaining a connection (mediastream) to it, that can then be shared between components.
Eyetracking also requires use of the participants webcamera and Webgazer handily has the requests to access this built into the basic package, which is great for standalone applications.  When integrating into a wider infrastructure, this built-in request to the webcamera interferes with Gorilla's own, with the potential to disrupt the wider task.

As such in 
```
webgazer.begin
```
we've added an argument for a mediaStream.  If this argument is set, Webgazer will then use this as it's stream.  Otherwise, it will call 
```
navigator.mediaDevices.getUserMedia
```
as normal.

### Disconnecting gaze predictions from requestAnimationFrame

requestAnimationFrame (rAF) is a vital part of keeping visual-based timing within the browser accurate. Called when the browser repaints the current frame (the content visible on the monitor/device screen), it allows for accurate timing of stimuli presentation.
However, the call to rAF can be blocking - requiring all of the functionality within the rAF loop to finish executing before the browser can begin processing the next frame.  If too much *stuff* is trying to happen within a rAF loop, the refresh rate of the device will slow down and the accuracy of stimuli/screen presentation timing will suffer as a result.
The default Webgazer packages uses rAF to collect and process predictions from the stream, which is fine in a standalone application that won't be using rAF for anything else.  However, if rAF is being used elsewhere, the additional load on the loop quickly reduces timing accuracy.

We noticed this, along with the javascript behavioural experiments packages JsPSych, in March/April 2021. JsPsych saw the same issues that we did in our own webgazer component: [Inaccurate trial timing with webgazer extension](https://github.com/jspsych/jsPsych/issues/1700) and ultimately chose to fork the Webgazer repository to implement a fix [6.3.1 Release Notes](https://github.com/jspsych/jsPsych/issues/1671).
While at that time we didn't fork the Webgazer repo, we did make alterations to the base webgazer.js file (which can still be viewed/reviewed through the browser developer console).

Within 
```
webgazer.begin
```
we add a new argument overridePredictions which is used to initialise a global variable overrideReqAnimPredictions. If set to true, when we reach the 
```
loop
```
function in webgazer, we stop the function just before we reach the point of requesting a prediction. This means all the rAF call in webgazer will do is paint the latest frame from the mediastream to the video/canvas element.

Then, we have introduced a new function
```
webgazer.requestPrediction
```
which contains the logic originally run in loop - the requesting and processing of a gaze prediction, along with calling the gazeListener callback.
You can then use your own logic within your wider application to make direct requests for gaze predictions at a time of your choosing, typically within a setTimeout loop or using setInterval.

Previously, as webgazer made a gaze prediction request on every frame, this would typically give 60 predictions per second. However, as the browser typically limits the mediastream framerate from a webcamera to 30 fps, there is no point trying to request predictions more frequently than this framerate.  You would just be asking for a fresh prediction on the *same frame* of mediastream data, which is unlikely to be productive. This is the same conclusion that jsPsych reached, in limiting the gaze prediction requests to 30 hz (30 predictions per second).
When initially collecting the mediastream, we do query the webcamera capabilities. If it is able to present at a higher framerate, we'll adjust how frequently we request gaze predictions.  Assume that for the majority of participants, the number of gaze predictions per second will be around 30.

### Provide more initialisation callbacks

Many online research tools initialise and preload as much content at the start of a task/study as possible. Front loading content in this way makes for a much smoother experience for the participant.  So, ideally, we would be able to initialise all the systems that webgazer needs and then only later begin any further logic.
To facilitate this, we've added a new callback 
```
callbackInitFinish
```
and a corresponding subscribe function
```
webgazer.setInitFinishListener
```
which is called when the initial, awaitable stages of webgazers begin function have finished. On initial task load, we can then wait until webgazer has finished setting up the basics of what it needs, before Gorilla progresses further.
