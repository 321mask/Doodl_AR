<div align="center">

<img src="DoodlAR Icon.png" width="120" alt="DoodlAR app icon">

# DoodlAR

### Draw a creature on paper. Point your phone at it. Watch it leap off the page into your room.

<p>
<img src="https://img.shields.io/badge/iOS-26%2B-000000?logo=apple&logoColor=white" alt="iOS 17+">
<img src="https://img.shields.io/badge/ARKit-2C2C2E?logo=apple&logoColor=white" alt="ARKit">
<img src="https://img.shields.io/badge/RealityKit-2C2C2E?logo=apple&logoColor=white" alt="RealityKit">
<img src="https://img.shields.io/badge/Core%20ML-2C2C2E?logo=apple&logoColor=white" alt="Core ML">
<img src="https://img.shields.io/badge/Vision-2C2C2E?logo=apple&logoColor=white" alt="Vision">
<img src="https://img.shields.io/badge/SwiftUI-2C2C2E?logo=swift&logoColor=F05138" alt="SwiftUI">
<img src="https://img.shields.io/badge/SwiftData-2C2C2E" alt="SwiftData">
<img src="https://img.shields.io/badge/USD-FF2D55" alt="USD assets">
<img src="https://img.shields.io/badge/status-In%20development-7D4DFF" alt="Status: In development">
</p>

<p>
<a href="#status">Status</a> ·
<a href="#what-it-does">How it works</a> ·
<a href="#tech-highlights">Tech</a>
</p>

</div>

[**Try it on TestFlight**](https://testflight.apple.com/join/vCpm7E9Z)

## The problem

Augmented reality on iPhone has gotten very good at placing *someone else's* polished 3D model into your living room. A pre-built dragon, a furniture catalog, a stock measuring tape. The technology is impressive and the moment is forgettable, because nothing about the experience is *yours*.

The real magic of AR is the inverse — when something *you made* shows up in *your* world. Kids already know this instinctively: a drawing on the fridge is alive in a way that a printed poster never is. The bottleneck has never been the imagination; it's been the gap between a pencil sketch on a scrap of paper and a rigged, animated 3D character that can navigate a real room.

DoodlAR closes that gap in about two seconds. You draw something on paper. You point your phone at it. A custom on-device sketch classifier identifies what you drew, and a 3D version of that creature spawns at the paper's exact world-space position with a particle-burst morph animation — your flat doodle turning into a fully rigged character in real time. From that moment on, the creature lives in your room: it bobs while idle, walks across your actual floor, reacts when you tap it, and (in the dog's case) can be commanded via a radial menu to chase a baseball you also drew or walk over to a tent you also drew.

The drawing is the input. The room is the stage. Everything else — recognition, anchoring, animation, navigation — has to happen on a phone, in real time, with no server in the loop.

## What it does

1. **Draw.** Sketch a creature on a piece of paper — dog, cat, dragon, frog, butterfly, rabbit, bird, fish, spider, snake — or a prop the dog can play with (baseball, tent).
2. **Scan.** Open the app and point the camera at your drawing. A live overlay tracks the paper's edges; once the rectangle holds still for a few frames, the app crops and perspective-corrects the sketch.
3. **Classify.** A custom on-device Core ML model identifies the creature with a confidence score. Below the threshold, it surfaces as "unknown" rather than guessing — false positives kill the magic.
4. **Spawn.** A quad textured with your *actual sketch photo* anchors at the paper's world position, then morphs into the rigged 3D creature over ~1.5 seconds with a particle effect. The creature lands in your room.
5. **Play.** The creature is now a persistent inhabitant of your scene. It idles, wanders the LiDAR-detected mesh on capable devices, and reacts to taps with one-shot animations. Spawn a dog plus a baseball or tent and a radial menu appears — *Sit, Wave, Chase Ball, Go to Tent* — and the dog navigates between them on your real floor.
6. **Collect.** Every spawned creature is saved with its sketch photo, confidence score, and visual features to a personal collection you can revisit.

## What I built

DoodlAR was built collaboratively as a small-team project. My contributions span the AR interaction layer — the dog ecosystem (idle / walk / spawn / tap-react animation graph), the radial command menu, the baseball-chase and tent-entry behaviors, the multi-USDA animation loader — plus the detection-pipeline polish and overall app-state machine that ties scanning, classification, and spawning into a single coherent flow.

### Genuinely hard part 1 — Real-time paper detection that doesn't lie

A sketch classifier is only as good as the crop you hand it. Point a phone at a piece of paper on a desk and you get a stream of `VNDetectRectanglesRequest` candidates that wobble, flicker, and occasionally hallucinate rectangles out of laptop edges and notebook spines. Feeding any one of those frames straight to the classifier produces confident, wrong answers.

The fix is a stability layer. `PaperDetector` is an actor that buffers the last few rectangle observations and requires three consecutive frames whose corners stay within a 5% tolerance before promoting the detection to "found." Only then does the pipeline perspective-correct the crop, threshold it to clean black-and-white, and hand it to the model. The result is the calm green outline you see in the UI — not a strobe of red boxes — and a classifier input that's actually worth classifying.

### Genuinely hard part 2 — A sketch classifier trained for *your* drawings, not photos of drawings

Most Core ML image classifiers are trained on photographs. A sketch classifier is a different problem: the input is a thresholded black-on-white line drawing, the categories overlap (a poorly drawn frog looks a lot like a poorly drawn rabbit), and the training set has to cover the messy reality of how *humans actually doodle* — not how a graphics designer renders an icon.

The model — `DoodlARSketchClassifierModel.mlmodel`, trained via a Create ML project bundled with the repo (`Resources/DoodlAR.mlproj/`) — is fed a dataset assembled from Google's Quick, Draw! corpus: hundreds of thousands of crowd-sourced sketches per category, rendered to PNG with randomized stroke widths to match the variability of pen/marker thickness on real paper. A confidence threshold of 0.4 gates the result, so anything ambiguous falls through to "unknown" rather than producing a wrong creature with high certainty. Inference runs on the Neural Engine via `VNCoreMLRequest` and completes well under a frame at 224×224 grayscale.

### Genuinely hard part 3 — A 3D character that actually lives in your room

The dog is the most fully realized creature, and it's the part that most clearly demonstrates what AR is capable of when the engineering is taken seriously. It's not a static model bobbing in place — it's a small character system stitched together from four separate USDA animation clips (`idle2`, `walk_dog2`, `spawn_dog2`, `tap_react2`) sharing one skeleton. `DogAnimationController` loads all four concurrently, extracts the underlying `AnimationResource` from each, and chains them: spawn fires once and on `AnimationEvents.PlaybackCompleted` automatically returns to idle; tap reactions are one-shots that interrupt cleanly; walks loop at a configurable speed.

`DogInteractionController` layers behavior on top. When the user has both a dog and a baseball alive in the scene, *Chase Ball* walks the dog over to the ball's anchor at 0.05 m/s, "catches" it, and carries it. *Go to Tent* scales the dog down, fades it into the tent, and holds it there. Each is a small state machine that has to coexist with the global navigation loop and with whatever the user does next (re-tap the menu, spawn another creature, lose tracking).

On LiDAR-equipped devices, `CreatureNavigator` uses ARKit's scene reconstruction mesh to keep creatures walking on the actual floor — random downward raycasts pick wander targets so the dog doesn't drift through your couch. On non-LiDAR devices it falls back gracefully to fixed-height anchoring so the experience still works on a base iPhone, just without the spatial accuracy.

### Genuinely hard part 4 — Making the spawn moment *feel* like the doodle came alive

The most fragile part of an experience like this is the transition. If the sketch on paper and the 3D creature in your room feel like two unrelated things, the whole illusion breaks. The spawn sequence is engineered specifically to bridge them: a quad textured with the user's actual cropped sketch photo is placed at the paper's world position, briefly visible, then crossfaded and scaled into the rigged 3D model while a particle burst plays and a spatial audio cue fires from the creature's location via `SpatialAudioComponent`. The whole sequence is choreographed against an explicit `SpawnState` machine (`idle → scanning → detected → classifying → triggerSpawn → morphing → alive`), so the UI chrome (overlays, captions, retry affordances) tracks the underlying state honestly instead of papering over it.

## Tech highlights

| Area | Stack & decisions |
| --- | --- |
| **AR foundation** | ARKit world tracking with plane detection (horizontal + vertical), scene reconstruction (LiDAR mesh) when available, environment texturing, and person segmentation with depth. RealityKit renders all entities; the scene mesh doubles as an occluder so creatures pass behind real-world objects. |
| **Sketch detection** | Vision's `VNDetectRectanglesRequest` at 2 fps with deliberately relaxed parameters (aspect 0.2–1.0, min size 5%, confidence 0.2) so it finds paper under bad lighting. A 3-frame, 5%-corner-tolerance stability gate in `PaperDetector` filters out flicker before the crop is handed downstream. |
| **Classification** | Custom Core ML model (`DoodlARSketchClassifierModel.mlmodel`) trained in Create ML on a Quick, Draw!–derived dataset rendered with randomized stroke widths. Inference via `VNCoreMLRequest` on the Neural Engine, 224×224 grayscale, 0.4 confidence threshold with explicit `.unknown` fallback. |
| **3D assets** | USD pipeline: static creatures shipped as `.usdz`, rigged dog as a family of `.usda` clips (`idle2`, `walk_dog2`, `spawn_dog2`, `tap_react2`) sharing one skeleton. Multi-clip loading and chaining handled by `DogAnimationController`. |
| **Interaction** | `TapCatcherView` intercepts gestures *before* RealityKit's `ARView` claims them, enabling manual tap/pan/pinch. The radial command menu (`RadialMenuView`) appears contextually when both a dog and an interactable prop are alive in the scene. |
| **Navigation** | `CreatureNavigator` uses downward raycasts against the LiDAR mesh to pick wander targets, with a graceful fallback to fixed-anchor behavior on non-LiDAR hardware. |
| **Audio** | RealityKit `SpatialAudioComponent` attached to creature entities — spawn sound, ambient loop per creature type, and tap-reaction audio all emit from the creature's 3D position with proper attenuation. |
| **Persistence** | SwiftData (`PersistedCreature`) stores each discovered creature with its sketch photo (PNG), confidence score, feature metadata (aspect ratio, silhouette complexity), timestamp, and optional nickname. No CloudKit. |
| **Concurrency** | Actor-based isolation for every off-main pipeline stage (`DetectionPipeline`, `PaperDetector`, `SketchClassifier`, `VisionService`, `FeatureExtractor`). Camera frames cross actor boundaries via a `SendableFrame` wrapper around `CVPixelBuffer` (IOSurface-backed, thread-safe by design). All RealityKit work is `@MainActor`. |
| **Networking** | None. Everything — detection, classification, animation, audio, persistence — runs on-device. No analytics, no third-party SDKs. |

## Status

- **Active development** — solo and team commits across detection pipeline, AR interactions, and asset polish.
- **TestFlight** — https://testflight.apple.com/join/vCpm7E9Z
- **App Store** — *`coming at a later date`*.
- **Source** — private repository.
- **Built collaboratively** — small team; my contributions center on the AR interaction layer (dog animation graph, radial menu, baseball/tent behaviors), the multi-USDA animation loader, detection-pipeline stability work, and the overall spawn state machine.
- **Best experienced on** — an iPhone or iPad Pro with LiDAR. Creature navigation across real-world surfaces is meaningfully richer when scene reconstruction is available.

## On-device as architecture

An AR app where a drawing turns into a creature lives or dies on one question: *can the moment between paper and creature feel instant?* If the classifier takes five seconds, the magic is gone. If the spawn anchor drifts, the magic is gone. If the dog walks through your couch, the magic is gone. The idea is the easy part — the *immediacy* is the entire product.

That single constraint shaped the architecture more than any feature did.

First, every stage of the pipeline runs **on-device**. There is no server to round-trip a sketch through, no upload step, no auth flow, no network failure mode to design around. The Core ML model lives in the bundle, the Vision requests run on local frames, the RealityKit scene exists only in the user's session. The fastest network call is the one you never make.

Second, the pipeline is **stage-isolated**. Each stage — rectangle detection, stability gating, perspective correction, classification, feature extraction, spawn — is its own actor, communicating via async/await. Camera frames are throttled to 2 fps before they ever enter the pipeline, which is enough resolution for paper detection and a fraction of the CPU/ML budget of doing it per-frame. A `SendableFrame` wrapper lets `CVPixelBuffer`s cross actor boundaries safely, so the camera thread is never blocked by Vision or Core ML work.

Third, the AR layer is built around **honest state**. The `SpawnState` machine names every step explicitly (`scanning`, `detected`, `classifying`, `triggerSpawn`, `morphing`, `alive`, `failed`) and the UI surfaces match the underlying state instead of papering over it. When tracking fails or classification falls below threshold, the user sees an honest retry affordance rather than a silent stall.

It's the same lesson good real-time engineering teaches anywhere: the thing that defines the product isn't a screen the user looks at — it's a property of the system that has to hold true in the half-second between a drawing on paper and a creature in their room.

<div align="center">
<sub>Built with ARKit · RealityKit · Core ML · Vision · SwiftUI · SwiftData — fully on-device, no servers.</sub>
</div>
