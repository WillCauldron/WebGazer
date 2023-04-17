# Gorilla Overview
This file contains an overview of the changes/additions made to the original webgazer source code.  The main focus of these is improving Webgaer interoperability with managed systems.  In short, Webgazer is intended to be used, out-of-the-box, without any other tools or capabilities assumed.  However, online tools for behavioural science will have their own systems for managing event timing, accessing and monitoring external hardware. The changes made allow Webgazer to easily integrate with such a system, minimising disruption.

These changes include
1. Connecting Webgazer with an existing mediastream (rather than always requesting it's own)
2. Disconnecting prediction from the requestAnimationFrame call so they can be called for in a separate thread (requestAnimationFrame is primary resource for accurate display timing in the browser. Heavy data processing within it can slow down frame draw and disrupt display timings)
3. Provide more initialisation callbacks, to allow Webgazer's systems to prepared and waited for appropriately