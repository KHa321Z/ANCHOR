# ANCHOR Implementation Plan

Nov 10, 2026 · @musa

> **Status, 12 Nov 2026.** This document is background and reasoning. `PLAN.md` holds the tasks and the decisions (its Section 2.1), and wins where the two differ. Read these points as replaced:
>
> - There is no Mac. Tests for the logic run on Linux, CI builds the app, and it is sideloaded onto the phone (Sections 4, 9 and 15).
> - The device is an iPhone 13 Pro Max. The server is one of two PCs, an RTX 4070 Super or an RTX A4000.
> - The tracker lives in `core/`, and the VLM adapter runs on the server.
> - G0 is measured in `IOS-05`, not in Sprint 0. The sprint plan in Section 15 is replaced by `PLAN.md` Sections 7 to 9, and the decision list in Section 16 by `PLAN.md` Section 2.1.
> - FR-03 was rewritten in the report: there is no closest-device fallback. FR-21 is a Must.
> - The study uses built graphs as they are. Reviewed graphs are a testing aid only.
> - Whether each study laptop is recorded to train its detector (enrolment) is open until component fine-tuning has been tested in depth. A fourth laptop, from another maker, is kept unseen.

## 1. Summary

ANCHOR can be built in about 790 hours of AI-assisted work over a preparation sprint and eight two-week sprints. Three changes to the current trajectory make that possible: read screws from slot crops instead of whole frames, enrol each study laptop instead of expecting the component detector to transfer, and record your own tool data, because I found none published for spudgers and pry tools.

| Area | Decision | Section |
| --- | --- | --- |
| Widgets | Keep W1 to W6. Add Guided Reality's 3D tool models, live cut-outs, curved arrows and depth placement, re-implemented from the paper | 3 |
| Pipeline | Three perception tiers; only crops, tools and contacts run on changed frames, inside a 150 ms budget | 4 |
| Screws | YOLO26n once per face for registration, plus a crop classifier per slot for head, hole or covered | 5 |
| Components | Enrol the three laptops, read state as change from each laptop's own reference, and test unseen laptops by leave-one-laptop-out and by larger server models | 6 |
| Tools | Record the study kit, label by mask propagation, train YOLO26n-seg, take the tip from the mask; hands from ARKit or Apple Vision | 7 |
| Outlines and anchoring | MobileSAM for components and rings for screws; a 6-DoF mat pose and a fitted face plane instead of two homographies | 8 |
| Logic | The report's gate, sequencer, recovery and trigger policy as a pure Swift package, driven by taps before the camera | 9 |
| 3D assets | Procedural tools and arrows; components shown as cut-outs of the live image, so no laptop needs a 3D model | 10 |
| Graphs | Server builder with a rule check, measured against nine hand-reviewed reference graphs | 12 |
| Budget | 685 hours plus 15% contingency; about 15 hours per person per week for three people over 18 weeks; 30 to 40 GPU-hours | 15 |

### First actions

1. Choose the three laptops and get a spare bottom cover.
2. Print the A2 marker mat and buy two identical tool kits.
3. Relabel the 176 component test images and fix the class sheet.
4. Export YOLO26n and MobileSAM to Core ML and time them on your device.
5. Build the recorder, then record laptop 1.

## 2. Where the project stands

The design is specified end to end but, from what you sent, nothing that runs on the iPhone exists yet. What you have is the Deliverable II report, a click-through concept demo, and two rounds of detector and segmenter benchmarks on a PC.

| Asset | What it gives the build | What it does not give |
| --- | --- | --- |
| Deliverable II report (117 pages) | Capabilities C1 to C6, 21 functional and 10 non-functional requirements, the procedure-graph vocabulary, gate, sequencer, recovery, trigger policy, and the three-part architecture | Any measured value: every threshold in Table 6.3 is provisional |
| Concept demo (`index.html` plus 28 clips) | The intended look of W1, W2, W3 and W5 over Dell Latitude 7400 support footage, and a good out-of-order example in step 3 | Any perception: each key press stands in for a detection, and the overlays are simulated |
| Screw detectors | A working fine-tuning recipe and a model pick | Laptop screws, screw state, device timing |
| Component detectors | Proof that one laptop can be learned to about 0.9 AP50 | Any transfer to a laptop not in training |
| MobileSAM and SAM ViT-B | A segmenter pick for box-prompted outlines | Component outlines, device timing |

### Screw detection (WEEE test: 200 images, 858 screws, 12 devices not in training, 1280 px)

| Model | AP50 \[95% CI\] | GFLOPs | CPU time (ONNX) |
| --- | --- | --- | --- |
| YOLO26s | 0.739 \[0.691, 0.782\] | 93.9 | 2.3 s |
| YOLO26m | 0.739 \[0.698, 0.782\] | 302.8 | 4.8 s |
| YOLO26n | 0.734 \[0.693, 0.776\] | 25.6 | 0.8 s |
| YOLO11s | 0.712 \[0.672, 0.751\] | 88.1 | 2.3 s |
| YOLO26s-P2 | 0.702 \[0.660, 0.747\] | 111.6 | 3.3 s |
| YOLO11n | 0.612 \[0.565, 0.656\] | 27.0 | 0.9 s |
| RT-DETR-L | 0.382 (training failed, epoch-4 checkpoint) | 426.8 | 6.0 s |

Zero-shot detectors scored 0.033 (YOLOE-26S), 0.007 (GroundingDINO-Tiny) and 0.000 (YOLO-World v2-S) on the same test. YOLO26n is the pick: it ties YOLO26s, YOLO26m and YOLO11s at about a quarter of the compute of the small models.

### Component detection (1,310 training images, 640 px, tested on laptops not in training)

| Model | Known laptop (Lenovo) AP50 | Unseen laptops AP50 | Unseen, federated | GFLOPs |
| --- | --- | --- | --- | --- |
| RT-DETR-L | 0.899 | 0.096 | 0.123 | 109.9 |
| YOLO26n | 0.894 \[0.870, 0.919\] | 0.082 | 0.127 | 6.1 |
| YOLO26s | 0.893 | 0.091 | 0.116 | 22.8 |
| YOLO26m | 0.888 | 0.113 | 0.130 | 75.0 |
| YOLO11s | 0.876 | 0.120 | 0.140 | 21.7 |
| YOLO11n | 0.861 | 0.103 | 0.115 | 6.6 |

Zero-shot models scored 0.01 to 0.05 on the known laptop and 0.03 to 0.04 on unseen ones. Six architectures spanning an 18-fold range of compute all land near 0.1 on unseen laptops, so the cause is the data, not the model.

### Segmentation (screw outlines, no training)

MobileSAM reaches mask IoU 0.655 from exact boxes against 0.651 for SAM ViT-B, at 55 ms instead of 560 ms per image. From YOLO11s's predicted boxes the mask AP50 is 0.477 (MobileSAM) and 0.535 (SAM), so the detector's boxes are the weak point, not the segmenter.

### What these results do not prove

- **Screws on laptops.** The test set is PC towers, microwave ovens and flat-panel displays. AP50 0.73 is whole-frame detection; the design reads each screw from a slot crop as head, hole or covered, and that accuracy is unmeasured.
- **Speed on the device.** 0.8 s is a PC CPU through ONNX at 1280 px. The 2 s next-step target (NFR-02) needs Core ML timings on the iPhone or iPad.
- **Component state.** The detectors find parts. Nothing yet reads seated, raised or out, plugged or unplugged, or cover attached or removed.
- **Tools, hands, contacts, registration, tracking.** No data and no model yet. The screenshots you sent contain no tracker results; if a tracker run exists, add its numbers here.
- **Phase 1 against phase 2.** YOLO11s scored 0.949 known, 0.101 unseen and 0.281 federated when trained on 261 images, then 0.876, 0.120 and 0.140 when trained on 1,310. The federated score halved. Confirm whether the test labels or class list changed between the two runs before quoting either in Deliverable III.

## 3. Target experience: Guided Reality's widgets on ANCHOR's state

ANCHOR keeps its six widget classes and its own state tracking, and takes Guided Reality's rendering techniques: 3D tool models, animated cut-outs of the real part, curved rotation arrows and depth-based placement. It does not take Guided Reality's per-step model chain, which is too slow for the 2 s target and never checks that a step was done.

I found no public code release for [Guided Reality](https://arxiv.org/abs/2508.03547), so plan to re-implement these techniques from the paper. Its front end is a Swift app on an iPad Pro using ARKit and SceneKit, which is close to your stack.

### What to take and what to replace

| Stage | Guided Reality | ANCHOR |
| --- | --- | --- |
| Plan | GPT-4o writes a JSON plan per question: instruction, visual type 1 to 5, key components | Procedure graph built before the session from repair resources and rule-checked; widgets follow from the action type (Table 5.4) |
| Locate the target | Gemini 2.5 returns a 2D box for every step, 3.2 to 3.5 s | Slot list and registration on the device, no call for an expected step; a server model only at session start or a new face |
| Outline | SAM 2.1 on the moving part, about 0.5 s | MobileSAM once per step for components; rings for screws |
| Place in 3D | LiDAR depth at the box corners, back-projected with the camera pose saved at request time | ARKit world tracking, mat pose and a fitted device-face plane, re-anchored continuously |
| Tool visual | Curated library of tool models with correct scale and orientation; Meshy generation as fallback (23.8 s, 1 of 3 usable) | Curated library only (Section 10) |
| Movement visual | Translation: the segmented part shown on an image plane and animated to its target. Rotation: a curved arrow | Same two techniques, with axis and direction taken from the step, not from a model call |
| Hand gesture | Six static hand models; users misread some | Stays out, as the report decided |
| Check the step | None; 65 of 100 generated steps were fully correct | Gate, sequencer and recovery (Section 9) |

### Widget specification

| Class | Demo today | Target |
| --- | --- | --- |
| W1 highlight | Rings on screws in a video | Screws: a 3D ring per slot, numbered, pulsing on the current one, ticked when the slot reads empty. Components: the segmenter's outline as a stroke on the device face. |
| W2 movement | 2D arrows | Rotation: a curved arrow around the screw axis, counter-clockwise to loosen. Translation: a cut-out of the real part sliding along its path, plus a faint outline at the end position. Pry: a wedge moving along the seam band. Lift: a 3D arrow rising off the face. |
| W3 tool | Screwdriver picture with direction | A 3D tool model with its tip on the part and its axis along the surface normal, spinning in the step's direction, with a chip naming the bit. |
| W4 information | Step text | Face-anchored step card with group progress, the screw-length map as a mini-map of the face, reminders, and a move-the-camera prompt. |
| W5 hazard | Battery flagged | Screen-fixed banner at the bottom, red state, with W1 on the battery connector and W2/W3 showing how to unplug it. |
| W6 corrective | Not shown as its own widget | Amber card naming the mistake and the first recovery step, with the offending part highlighted and a Dismiss button. |

### Rules that keep the visuals consistent

Guided Reality's participants found its visuals inconsistent and low in contrast, so fix the visual language once:

- Colour follows the guidance state, as the report specifies: blue normal, amber corrective, red hazard.
- All widgets use unlit materials and one stroke weight, so they read on any laptop colour.
- Real hands hide the widgets behind them (ARKit people occlusion), which gives the depth cue a flat overlay lacks.
- A widget whose anchor is hidden stays at its last position, dimmed.

### Scope changes this implies

1. Movement widgets in 3D move from future work (Table 5.15) into scope.
2. The mat transform becomes a 6-DoF pose instead of a 2D homography (Section 8).
3. NFR-04's 2 cm drift is too loose for a ring on a 4 to 5 mm screw. Use 5 mm as the internal target for slot rings and keep 2 cm as the outer bound for cards.

## 4. System architecture and pipeline

The three parts stay as the report defines them: the app does all real-time work, the team server builds graphs and returns outlines, and the hosted VLM answers exceptions. The one refinement is to split perception into three tiers by how often each runs, so that only cheap work happens on every frame.

&#91;embedded content: ANCHOR architecture · device tiers, guidance logic, server and hosted VLM\]

Frames flow down through the two tiers into the gate, sequencer and trigger policy, all on the device. The server is contacted for the graph at setup and for each new outline; the VLM receives one question per exception.

### Perception tiers

| Tier | Runs | Where | Work |
| --- | --- | --- | --- |
| A | Every camera frame | Device | ARKit pose and LiDAR depth, mat pose, SSIM change check on the mat-aligned frame, hand mask, widget drawing |
| B | Changed frames only | Device | Slot-crop classifiers for screws and component states, tool detector and tip, contacts with LiDAR height, the gate |
| C | Events only | Server or hosted VLM | Whole-face part localisation, outlines, VERIFY and hazard-clear questions, off-graph typing, graph building |

The whole-frame screw detector at 1280 px belongs to tier C in spirit: it runs once per face for registration, on the device, where one to two seconds is acceptable. It never runs per frame.

### Flow of one session

1. **Setup.** The user picks the task and scans the model label (Apple Vision text recognition). The graph client fetches a saved graph or the server builds one.
2. **Face registration.** On the first clear view of a face, the app detects screws and landmarks, pairs them with the graph's layout, and compares the screw count with the graph.
3. **Guidance loop.** Tier A and B produce observations; the gate confirms changes and contacts; the sequencer labels them and updates the state; hazard rules and the trigger policy choose widgets; the widget pipeline draws them.
4. **Exceptions.** The sequencer or a timer sends one specific question through the VLM adapter. Hazard warnings never wait on the network to appear.
5. **Log.** Every event is written as one JSON line.

### Latency budget for NFR-02 (next step within 2 s, p95)

| Stage | Budget | Source |
| --- | --- | --- |
| Change visible to processed frame | 0.2 s | Tier B at 5 frames per second or better |
| Gate confirmation (`t_conf`) | 0.5 s | Report, Table 6.3 |
| Sequencer and trigger policy | under 0.01 s | Plain logic |
| Widget drawn | 0.05 s | Next rendered frame |
| Margin for a hand moving away | about 1.2 s | Remainder |

This sets one hard requirement on tier B: all on-device models together take at most 150 ms per processed frame on the target device. Measure it in Sprint 1 before building anything else on top. The outline from the server is not on this path: W1 shows a box or ring first and upgrades when the mask arrives.

### Code layout

| Module | Language | Contents | Runs without a device |
| --- | --- | --- | --- |
| `AnchorCore` | Swift package, no ARKit or UIKit imports | Graph model, validator, gate, sequencer, recovery planner, hazard rules, trigger policy, configuration, log | Yes, under `swift test` on a Mac |
| `AnchorPerception` | Swift | Core ML wrappers, crop pipeline, tracker, contact computation | Yes, on recorded frames |
| `AnchorAR` | Swift | ARKit session, mat pose, registration, RealityKit widgets | No |
| `AnchorApp` | SwiftUI | Screens S1 to S7 | No |
| `anchor-server` | Python, FastAPI | Graph builder, resource store, graph store, segmentation, and one new optional service: part localisation | Yes |
| `anchor-ml` | Python, Ultralytics | Datasets, training, evaluation with confidence intervals, Core ML export, parity tests | Yes |
| Recorder and replay | Swift | Stores RGB, depth, pose, intrinsics and timestamps per clip (report 6.7.6) and feeds them back through the pipeline | Replay: yes |

Keeping `AnchorCore` free of ARKit matters for AI-assisted work: a coding agent can run its tests and replay bench clips on a Mac, but it cannot hold an iPhone over a laptop.

## 5. Perception: screws

Screws need two models with two jobs: YOLO26n at 1280 px finds screws once per face for registration, and a small crop classifier reads each known slot on every changed frame. The detector exists; the crop classifier, which carries the per-step accuracy, is not built yet.

| Job | Model | Input | Runs | Status |
| --- | --- | --- | --- | --- |
| Find screws on a new face, pair them with the layout, check the count | YOLO26n, fine-tuned | Whole frame at 1280 px | Once per face, and when registration is lost | Trained on WEEE (AP50 0.734); no laptop screws yet |
| Read one slot as head, hole or covered | YOLO26n-cls or MobileNetV3-small, three outputs | Full-resolution crop per slot, all slots in one batch | Every changed frame | Not built |

In ARKit's default 1920 × 1440 frame, a 4 to 5 mm screw head is about 20 px across when the view spans 40 cm. That is enough for a crop classifier and too little once the frame is shrunk to 640 px, which is the report's argument for slot crops. ARKit's 4K video format doubles the pixels on supported devices; test it only if crops prove too coarse.

### Data

| Source | Use | Status |
| --- | --- | --- |
| WEEE screw set (653 training images, 3,225 screws) | Detector pre-training | Done |
| Lenovo set: about 2,257 `Screw_Inserted` and 1,409 `Screw_Hole` boxes | Crops for head and hole; detector fine-tuning on laptop screws | Next |
| Bench frames of the three target laptops | Crops per slot in both states, under several lights | To record |
| [Danish Technological Institute laptop screen-frame and screw set](https://zenodo.org/records/18483802): 342 images, CC BY-SA 4.0 | Extra laptop screws for the detector | New, not in the report |
| RoboSAPIENS sets, and Zhang et al.'s screw and tool set if access is granted | As planned in Table 3.2 | Unchanged |

Labels for your own crops are free. Record one clip with every screw in and one with every screw out; registration gives each slot's position, and the clip gives its state. No one draws a box.

[DiFilippo, Jouaneh and Jedson (2024)](https://doi.org/10.3390/app14146301) found that hole depth, taper, location and colour contrast with the cover all change how well laptop screws are detected, with depth the strongest factor. Record recessed and dark-on-dark screws on purpose.

### Steps

1. Cut the Lenovo screw boxes into crops and train classifier v0 on head against hole.
2. Record the target laptops with screws in, out and half removed. Auto-crop through registration and retrain. Test leave-one-laptop-out.
3. Add the covered class from real hand clips, plus hand and tool cut-outs pasted over clean crops.
4. Fine-tune YOLO26n on laptop screws from the WEEE checkpoint. Vote over 5 to 10 still frames at session start to raise recall.
5. Export both models to Core ML. Run a parity test against PyTorch on 200 inputs, then time both on the device.
6. Fit temperature scaling on a held-out set so the gate's `c_high` and `c_low` act on calibrated scores.

### Acceptance (proposed, to confirm in Deliverable III)

| Measure | Target |
| --- | --- |
| Slot reading on unoccluded slots, held-out laptop | Macro-F1 at least 0.95 |
| False hole readings that survive the gate | None per bench session; one would unlock prying early |
| Session starts where every visible screw is paired with the layout | At least 95%, with a tap to add a missed slot |
| Device time, 16 crops in one batch | At most 20 ms |

The P2 variant scored lower than plain YOLO26s (0.702 against 0.739), so drop it. Keep tiled inference in reserve for the case where registration recall falls short. Captive screws, screws under feet and bit size stay as the report handles them: a tap, plan order, and W3 naming the bit.

## 6. Perception: components and the cross-laptop gap

The component detector fails on unseen laptops because it has seen one laptop, and no architecture change fixes that. Two moves do: enrol each study laptop with its own data, which guarantees the proof of concept, and stop asking the on-device detector to recognise parts by appearance on a laptop it has never seen.

### Diagnosis from your own results

- **One training laptop.** 0.86 to 0.90 AP50 on the Lenovo, about 0.1 elsewhere, for all six models.
- **Incomplete test labels.** In phase 1, ignoring unlabelled classes lifted YOLO11s from 0.101 to 0.281.
- **Class definitions differ.** A thin heatpipe in training against a whole heatsink in test scores 0.00.
- **Missing classes.** No SSD training images in phase 1 (confirm for the 1,310-image set). No battery-connector class at all, although the hazard rule depends on that part.
- **Only RAM transfers.** The likely reason is that RAM modules look alike across laptops, while batteries, fans and covers do not.

### What ANCHOR needs from component perception

| Question | How often | Needs cross-laptop recognition |
| --- | --- | --- |
| Where is each part on this face? | Once per face | Yes |
| What state is each part in? | Every changed frame | No, if read against this laptop's own reference |
| Is a hand or tool on it? | Every changed frame | No (Section 7) |

Only the first question is hard across laptops, and it runs once per face. So it can use a per-laptop model, a larger server model, or the layout, without touching the 2 s path.

### Strategies

| # | Strategy | What it does | Role |
| --- | --- | --- | --- |
| S0 | Fix the measurement | Relabel the 176 test images so every visible part is boxed (your estimate: 2 to 3 hours with pre-labels). Write one class sheet: heatsink means the whole assembly. Add SSD and battery connector. Report AP per class and per laptop. | Prerequisite |
| S1 | Enrol each target laptop | Record 8 to 10 short clips per laptop across faces and stages. Label one frame per clip, propagate the masks through the clip with a video segmenter (SAM 2 or SAM 3), review, and keep 300 to 600 frames per laptop. Fine-tune YOLO26n from the Lenovo checkpoint. | Primary. Should put all three study laptops in the 0.86 to 0.90 regime you measured on the Lenovo |
| S2 | Train on several laptops, test leave-one-laptop-out | Lenovo, the three enrolled laptops, the Dell 3520 and open-licence sets. Train on all but one, test on the one left out, and plot unseen AP50 against the number of training laptops. | Measures generalisation. A result for Deliverable III either way |
| S3 | Use the layout as a prior | Detect parts as one generic class, then assign identities by matching positions and sizes to the graph's layout. | Removes the need to tell an SSD from a Wi-Fi card by looks |
| S4 | Locate with a large model on the server | On the relabelled test set, compare Gemini's box output, SAM 3 with text and exemplar prompts, Grounding DINO Base, OWLv2, and YOLOE-26 L with a visual prompt from a reference photo. | The path for a laptop with no enrolment |
| S5 | Read state as change from a reference | At registration, store a crop of every part in its starting state. Per frame, compare each crop with its own reference, then a per-kind classifier names the new value. | Makes state reading independent of the laptop model |
| S6 | Augment | Paste part masks onto other laptops' interiors; vary colour and perspective. | Small expected gain; run with S2 |

Two cautions on S4. Your zero-shot test covered only small and tiny models, so larger ones are untested, not ruled out. And Guided Reality, which used Gemini 2.5 for boxes, reported that small or visually similar components were hard to detect. Measure before relying on it.

State kinds for S5 follow Table 5.3: component seated, raised or out; connector plugged or unplugged; clip engaged or released; cover attached or removed. Battery-connector unplugged is always confirmed by the VLM before the hazard clears, so the classifier there only has to raise the question.

### Datasets beyond Table 3.2

| Dataset | Content | Licence | Use |
| --- | --- | --- | --- |
| Your Lenovo G50-80 set and Dell Latitude 3520 photos | Already used in both benchmark rounds | Yours | Add to the dataset table in Deliverable III |
| [Laptop Disassembly (Roboflow, robothon1)](https://universe.roboflow.com/robothon1/laptop-disassembly) | Fan and RAM boxes; size not stated on the page | CC BY 4.0 | Extra fan and RAM examples |
| Other Roboflow sets on laptop and PC parts | Unverified | Varies | Triage 30 minutes each: classes, licence, laptop or desktop, duplicates |
| iFixit guide photos | Photos of many laptop models | CC BY-NC-SA 3.0 | Do not train on them (see below) |

[iFixit's licensing page](https://www.ifixit.com/Guide/Info/Licensing) states that using its data to train machine-learning models violates its terms of use. Write to them for academic permission before any training use. Reading a guide to build a procedure graph is a different use, but check the terms before automating retrieval.

One correction to check: Table 3.2 lists Zhang et al. \[27\] as 560 raw images and 1,717 masks, while the [arXiv abstract](https://arxiv.org/abs/2603.27441) describes 1,456 annotated images. Confirm which figure belongs in Deliverable III.

### Order of work and the gate

1. S0, then choose the three laptops.
2. S1: capture, label, fine-tune.
3. S5 state readers on the enrolled laptops.
4. S2 with S6, as unattended training runs.
5. S4 comparison on the server, then S3 on top of the best localiser.

**Gate at the end of Sprint 3.** Proceed on the enrolled path if enrolled AP50 is at least 0.85 and state-reading macro-F1 is at least 0.90. Keep the no-enrolment path only if S2, S3 or S4 localises every task-relevant part in at least 80% of registration attempts on a held-out laptop. Otherwise limit the claim to enrolled laptops and say so in Deliverable III.

## 7. Perception: tools, hands and contacts

Tools are a closed set, the kit your study hands to participants, so record that kit, label it by mask propagation and train one small on-device segmentation model. I found no public dataset with spudgers or plastic pry tools, so your own recordings are the main data source, not a supplement.

### What the pipeline needs from a tool

| Output | Used for | How |
| --- | --- | --- |
| Class | The wrong-tool check against `tool(a)` | Detector class; sent as unknown tool below `c_high`, as the report specifies |
| Tip point | Contact with a step's target region | From the mask, see below |
| Height above the face | Touching against hovering | LiDAR depth at the tip minus the fitted face plane |
| Identity over time | One contact, not many | Tracker (Section 8) |

### Classes, version 1

| Class | Steps that use it | Note |
| --- | --- | --- |
| Screwdriver | Remove, refit, loosen and tighten fastener | Bit size is not visible; W3 names it |
| Plastic pry tool (spudger, pick, pry bar) | Pry cover, peel foot, unplug connector | Split pick from spudger only if a graph needs the difference |
| Metal pry tool | None: it is the wrong tool at the seam | The negative class that WRONG EXECUTION needs |
| Tweezers | Optional | Only if a target laptop's graph names them |

Buy two identical kits, one for recording and one for the study, and choose plastic tools in a saturated colour. The report already assumes the tools look clearly different.

### Steps

1. **Record.** 25 to 30 clips of 30 to 60 s: each tool lying on the mat, held over each laptop face, and in use on a screw, seam or connector, under two or three lights and with two operators. Sample 3 to 5 frames per second for 3,000 to 4,000 frames. Hold out whole clips from a different day and operator as the test set.
2. **Label.** Click the tool once per clip and let a video segmenter carry the mask through. Review every test frame and about 10% of training frames.
3. **Find the tip without keypoint labels.** Take the principal axis of the mask; the tip is the end farther from the holding hand. This is the report's rule (Section 5.4.4) applied to a mask instead of a box.
4. **Augment.** Paste tool masks over frames of every laptop, including the held-out one. Render the Section 10 tool models over bench photos; the same models serve W3 and training.
5. **Train** YOLO26n-seg at 640 px and export to Core ML.
6. **Measure contacts.** Stage hover and touch clips at 1, 3 and 5 cm and tune `h_touch` and `min_time` on them.

If the tip error misses its target, move to YOLO26n-pose with two keypoints (tip and handle end) labelled on about 500 frames.

### Public data, as optional pre-training

| Dataset | Relevant content | Limits |
| --- | --- | --- |
| [MECCANO](https://iplab.dmi.unict.it/MECCANO/) | Screwdriver and wrench among 20 classes; 64,349 active-object boxes and 89,628 hand boxes | Toy-motorbike assembly from a head-mounted camera; no licence stated on the page |
| [ENIGMA-51](https://fpv-iplab.github.io/ENIGMA-51/) | 25 object classes including an electric screwdriver and pliers; 275,135 object boxes and 56,473 hand boxes; 3D models of the objects | Electrical-board repair on HoloLens 2; no licence stated on the page |
| Roboflow Universe tool and screwdriver sets | Many small community sets | Unverified quality; triage before use |

Use these only if the model trained on your own clips underperforms. Check each dataset's terms first.

### Hands

Do not train a hand model until two built-in options have been tested in Sprint 1:

- **ARKit person segmentation** (`personSegmentationWithDepth`) gives a per-frame person mask and estimated depth. It was built for people occlusion, so its quality on a close-up hand at 30 to 50 cm is the thing to test.
- **Apple Vision hand pose** gives 21 landmarks per hand on the device; their hull is a usable mask and the fingertips are contact points.

If neither is good enough, add a hand class to the tool segmentation model, with MECCANO and ENIGMA-51 hand boxes as pre-training and your own propagated masks on top.

### Acceptance (proposed)

| Measure | Target |
| --- | --- |
| Tool AP50 per class, held-out clips | At least 0.90 |
| Plastic read as metal or the reverse, among detections above `c_high` | At most 5% |
| Tip error against hand-marked tips | Median at most 5 mm, 90th percentile at most 10 mm |
| False W6 warnings from hovering hands | At most one per bench session |
| Device time, YOLO26n-seg at 640 px | At most 60 ms |

## 8. Segmentation, tracking and 3D anchoring

Keep MobileSAM for component outlines and draw rings instead of masks on screws. Fixed parts get their identity from slot positions and moving objects from a small tracker. Everything is anchored through a 6-DoF mat pose in ARKit's world space, which removes the parallax error the report accepts with homographies.

### Segmentation

- **Model.** MobileSAM, on your numbers: from exact boxes it matches SAM ViT-B at a tenth of the time. From predicted boxes SAM ViT-B is ahead (0.535 against 0.477), a gap that better boxes should narrow.
- **Screws get rings.** Screw masks from predicted boxes reach only 0.477 mask AP50, and a 5 mm outline adds nothing a ring does not. Guided Reality likewise replaces any box under 5 cm with a point effect.
- **Components get outlines.** One request per newly highlighted part, as the report specifies. W1 shows the box first and swaps in the mask when it arrives. The mask is stored in device-face coordinates, so registration carries it.
- **The same mask feeds W2.** The cut-out that slides along the motion path is the masked crop of the live frame.
- **Still to measure.** MobileSAM has only been scored on screws. Score it on 100 reviewed component masks from the enrolled laptops.
- **On-device option.** Apple publishes [Core ML builds of SAM 2.1](https://huggingface.co/apple/coreml-sam2.1-tiny), and EdgeTAM is an on-device variant of SAM 2 for video. Time one of them in Sprint 4. If the image encoder runs in under a second on your device, move outlines on-device and drop one network dependency.

### Identity and tracking

| Thing | Identity from | Implementation |
| --- | --- | --- |
| Screws, connectors, components in place | Slot position (report 5.4.3) | No tracker; registration is enough |
| Tools, hands, removed parts | Tracking by detection in the style of ByteTrack | A Swift port: constant-velocity Kalman filter, two-pass IoU matching, and the states active, hidden and lost |

Check the Swift tracker against Ultralytics' ByteTrack on the same detections from ten bench clips, and require matching identities on at least 95% of frames. Report identity switches per minute for tools. The screenshots you sent contain no tracker results, so record which tracker you ran, on which clips, and its scores.

### Registration and anchoring

| Element | Report | Implementation |
| --- | --- | --- |
| Mat transform | Homography from fiducials, per frame | 6-DoF pose of the mat in ARKit world space: ArUco markers detected with OpenCV, pose solved with ARKit's intrinsics, held as an anchor. ARKit's motion tracking carries it when markers are hidden, as the report intends |
| Device-face plane | Treated as lying on the mat; parallax absorbed by re-estimation | A plane parallel to the mat at a height measured from LiDAR depth over the face. Re-measured when the face changes |
| Device transform | Homography from landmarks with RANSAC | A rigid 2D transform on that plane (shift and rotation, three parameters) from landmarks with RANSAC. A laptop on a mat cannot move any other way, and three parameters are stable with few landmarks |
| Slot to pixel | Product of two homographies | Projection of a 3D point through ARKit's camera, exact for any camera position |
| Widgets | Drawn flat on the face | RealityKit entities under one anchor at the device-face frame; they follow when the device transform changes |
| Parts inside the laptop | Slightly less accurate | Per-part height from the median LiDAR depth over the part's box |

ARKit's depth map is 256 × 192, far coarser than the camera frame. Fit planes and take medians over regions; never read depth at a single screw.

### The mat

- A2 matte print (594 × 420 mm); a 15.6-inch laptop is roughly 36 × 25 cm, so A3 is too tight.
- 10 to 12 ArUco markers of at least 3 cm around the border, each with known mat coordinates in millimetres.
- A printed 100 mm scale bar to verify the print was not rescaled.

### Acceptance (proposed)

| Measure | Target |
| --- | --- |
| Slot reprojection error, camera moving at hand-held pace for 2 minutes | Median at most 3 mm, 95th percentile at most 5 mm |
| Re-alignment after the laptop is nudged 3 cm | Within 0.5 s |
| Session starts with trusted registration | At least 95% |

Measure the first row with stickers at measured positions on a laptop cover. If the 3D path fails on your device, the report's two-homography path remains the fallback and needs no depth for anchoring.

## 9. Step tracking, verification and error handling

This is plain logic that Chapter 5 already specifies, so it is the part an AI coding agent can build fastest. Write it as the pure Swift package `AnchorCore`, with unit tests taken from the report's tables, and drive it with scripted events before any model is good enough.

### Components

| Component | Spec | Build notes | Tests |
| --- | --- | --- | --- |
| Graph model and validator | 5.2, 5.3.4, 4.9 | Codable structs; fact kinds and action types as enums from Tables 5.3 and 5.4; reversal steps, reassembly phase and safety rules added by code, not by the VLM | One valid graph, plus one broken graph per error kind in Algorithm 1 |
| Gate | 5.5 | Runs measured in seconds; two thresholds; SSIM skip that still advances time | Synthetic score traces: flicker, occlusion, slow change, skipped frames |
| Sequencer | 5.6 | The only writer of the state; exact and partial matching behind one setting | One scripted scenario per label, and the demo's step 3 (screws before the cable) as a regression test |
| Recovery planner | 5.7 | Forward search over steps by preconditions and effects, path length at most 6 | Each kind of mistake in Table 5.8 |
| Hazard rules and trigger policy | 5.9, 6.6.1 | Thresholds 0.10, 0.30, 0.50; safety override for assumed values; priority W5, W6, W3, W1, W2, W4; four part-anchored widgets at most | Table-driven tests |
| Timers | 5.8.2 | Stall 45 s and heartbeat 40 s on an injectable clock | Simulated time |
| VLM adapter | 5.8, 6.7.3 | One request type per reason in Table 5.9; answers constrained to the allowed lists; cooldown, three calls per minute, urgency queue, timeouts; versioned prompt files; a mock provider | Replay of recorded request and answer pairs |
| Calibration | 5.9.4 | Python: temperature scaling for detector scores, histogram binning or isotonic regression for VLM confidences; writes values into the configuration file | Reliability diagrams |
| Session log | 4.9, FR-18 | One JSON line per event | Schema test |

### Build order

1. **One reference graph.** Build the storage-task graph for laptop 1 and review it by hand. The report allows this during testing (6.7.6), with the graph marked in the log.
2. **Logic before perception.** Drive `AnchorCore` with taps and scripted events. This turns the concept demo into the real sequencer with a person standing in for the camera, using the tap path the app needs anyway (FR-07).
3. **Perception in, fact by fact.** Replace each scripted fact with its real reading as Sections 5 to 7 deliver it: screws first, then cover, then components, then contacts.
4. **VLM last.** Develop against the mock provider, then switch to the hosted model.

### Who confirms what

| Fact | First evidence | If unclear | Last resort |
| --- | --- | --- | --- |
| Screw present or absent | Slot classifier | VLM on a crop | Tap |
| Captive screw loosened | Tap |  |  |
| Cover attached or removed | Face change at registration | VLM | Tap |
| Battery connector unplugged | Classifier raises the question | VLM must confirm | None: a tap never clears a hazard |
| Clip, component | Change from reference, then classifier | VLM on a crop | Tap |
| Replacement installed, power off | Tap |  |  |

### Choosing the VLM

Run a small comparison before committing. Collect about 150 labelled crops from the bench (connector plugged and unplugged, latches, unclear screws) and score two or three hosted models on accuracy, latency at the median and 95th percentile, cost per call, and calibration. Keep two slots behind the adapter: a fast model for in-session questions, where the answer is needed in seconds, and a stronger, slower one for graph building, which runs once per model and task.

## 10. 3D assets for tools and widgets

Model the tools and arrows procedurally, show components as live cut-outs of the real part, and keep scanning and generation as back-ups. A screwdriver shaft is exactly the thin, shiny object that photogrammetry handles worst, and Guided Reality's own generated tools were usable one time in three.

### Five routes, in order of preference

| Route | Use for | How | Time per asset | Risk |
| --- | --- | --- | --- | --- |
| 1. Procedural model | Screwdriver, spudger, pick, pry bar, tweezers; straight, curved and lift arrows; ring, tick, number badge, hazard glyph | Measure the real tool with calipers. An AI-written Blender Python script builds the mesh from those numbers and exports USD, packaged as USDZ | 0.5 to 1 hour | Low. Stylised shapes read better over video than photoreal ones |
| 2. Live cut-out | RAM, SSD, drive, battery and cover in W2 | The masked crop of the live frame on a quad lying on the device plane, animated along the path. This is Guided Reality's translation technique | None per laptop | Flat, but needs no authoring for a new laptop |
| 3. Open-licence library | A more detailed tool, if wanted | Sketchfab models under a Creative Commons licence, or Poly Haven (CC0). Fix scale, origin and axis | About 1 hour | Licence terms, orientation, polygon count |
| 4. Object Capture scan | Matte, bulky items: a plastic pry bar, a bottom cover for the lift animation | iPhone or iPad on iOS 17 or later, reconstruction on the device or a Mac. Turntable, diffused light, 70% overlap between shots | 30 to 45 minutes | [Apple's guidance](https://developer.apple.com/documentation/realitykit/capturing-photographs-for-realitykit-object-capture) says to avoid objects that are very thin in one dimension, highly reflective or transparent |
| 5. Image-to-3D generation | A tool outside the kit that a user asks about | [Hunyuan3D 2.1](https://github.com/Tencent-Hunyuan/Hunyuan3D-2.1) needs 10 GB for shape, which fits the 12 GB server, and 21 GB for texture, which does not. A hosted service such as Meshy is the alternative | Minutes | Guided Reality found generated tools lacked detail or were misoriented. Keep it off the study path |

Route 2 is what preserves the report's claim that nothing is hand-authored per laptop: no component ever needs a 3D model.

### Asset conventions

Guided Reality's library works because every model has the correct scale and orientation, with the contact point at the origin. Adopt the same rules:

- Origin at the working tip, where the tool touches the part.
- +Y along the tool's axis from tip to handle, so aligning +Y with the face normal stands the tool on the part.
- Real size in metres.
- One unlit material, tinted at runtime by guidance state. No textures.
- Under about 5,000 triangles.
- Motion in code (spin about Y, slide along a path), not baked, so one model serves both loosening and tightening.
- A manifest file mapping each graph tool name to its asset, tip offset and allowed motions.

### Asset list, version 1

| Group | Assets |
| --- | --- |
| Tools | Precision screwdriver, plastic spudger, opening pick, plastic pry bar, metal pry tool (shown crossed out for the wrong-tool card), tweezers (optional) |
| Motion | Straight arrow, curved arrow, lift arrow, pry wedge |
| Markers | Ring, tick, numbered badge, hazard glyph |

### Placing a tool

1. Position: the 3D point of the slot or part.
2. Up axis: the device-face normal. ANCHOR has it from the plane fit; Guided Reality estimates it from three depth points.
3. Heading: rotate about the normal so the handle leans toward the camera side.
4. Motion: spin with the curved arrow for a screw; slide along the seam for prying.
5. Occlusion: real hands hide the model, so it never appears to float in front of the user's fingers.

### Build and check

Run the Blender script headless so assets rebuild from one command. Check each asset automatically: bounding box within 5% of the measured tool, origin at the tip, triangle count under the limit. Then open each one in AR Quick Look on the device beside the real tool. The same models are rendered over bench photos as synthetic training data for the tool detector (Section 7).

## 11. AR application and widget rendering

Build the app natively, as the report chose: Swift, ARKit, RealityKit and Core ML. One ARKit session supplies frames, camera pose and LiDAR depth to both the guidance pipeline and the renderer.

### Session and threading

| Concern | Decision |
| --- | --- |
| Configuration | World tracking with scene depth; person segmentation with depth if the hand test in Section 7 selects it |
| Device check | Refuse to start guidance on a device without LiDAR (NFR-08) |
| View | RealityKit's `ARView` wrapped for SwiftUI. It is available on the report's minimum, iOS 17 |
| Frame handling | Tier A runs on every frame. Tier B runs on one background queue with a latest-frame-wins rule, so a slow model drops frames instead of stalling the camera |
| Frame memory | Copy the crops you need and release the frame. Holding ARKit frames stalls the session |
| Models | Loaded and warmed at launch; all slot crops in one batch; file names read from the configuration so a model swaps without a code change (NFR-06) |
| Heat | Watch the system thermal state and lower the tier B rate when it rises. Run one full-length session in Sprint 2 to see how the device behaves after 20 minutes |
| Configuration file | Loaded at launch, with an optional override fetched from the server, so thresholds can be tuned on the bench without a rebuild |

### How each widget is drawn

| Widget | Renderer | Anchor | Content from |
| --- | --- | --- | --- |
| W1 on screws | Ring entities, one per slot | Slot 3D point | Slot list and state |
| W1 on a component | The mask outline rendered to a transparent texture on a quad | Device-face plane | Segmentation reply |
| W2 rotation | Curved-arrow entity | Slot 3D point, face normal | Action type and direction |
| W2 translation | Quad textured with the masked crop, animated along the path; the same quad at low opacity at the end position | Device-face plane | Mask, the step's motion field and completion cue |
| W3 | Tool model from the manifest, spun or slid in code | Part 3D point, face normal | `tool(a)` and fastener details |
| W4, W6 | SwiftUI cards pinned to the projected anchor point | Device face | Step fields, pending list, recovery path |
| W5 | SwiftUI banner at the bottom of the screen | Screen | Hazard rule |

Text stays in SwiftUI because it is sharper and easier to lay out than text meshes. Everything spatial is a RealityKit entity under the device-face anchor.

### Screens

Build S1 to S7 as the mock-ups in Section 4.8 show them. The step view needs three states driven by the trigger policy (normal, corrective, hazard), the Mark done button for FR-07, and the Ask button for FR-17, which is a Could requirement and can come last. The model label is read with on-device text recognition and always shown to the user to confirm or correct.

### Two tools to build first

- **Recorder.** Saves RGB, LiDAR depth, camera pose, intrinsics and timestamps in sync, only with consent (NFR-07). Every later measurement depends on these clips.
- **Debug overlay.** Shows slot crops with their scores, gate runs, registration inliers and the current state. Bench tuning is guesswork without it.

### Acceptance (proposed)

| Measure | Target |
| --- | --- |
| Rendering with tier B running | At least 30 frames per second, no visible stutter when a model runs |
| Tier B, all models, per processed frame | At most 150 ms (Section 4) |
| A 25-minute session | No thermal shutdown of tracking; tier B rate reported over time |

## 12. Procedure graphs and the server

The graph builder is a server job that turns a model number and a task into a checked JSON graph. Build it early and on real manuals, because how accurately graphs can be extracted is the report's largest unmeasured risk (R-05).

### Pipeline

1. **Identify.** The app reads the label and the user confirms the exact model number.
2. **Gather (FR-21).** Service manual first, then repair guides, then videos as transcripts and sampled frames. Each resource is stored with its type, source and model number. For the three study laptops, seed the store by hand once; automate the web search after that works.
3. **Extract.** The VLM receives the task and goal, the model number, the resources, and the fixed lists of fact kinds and action types. It returns JSON against a schema, with sampling randomness off.
4. **Add structure in code.** Reversal steps, the reassembly phase, durations, the battery hazard rule, its reassembly exception, the battery-unplugged precondition on every step inside the device, and the pending-fixes precondition. The model cannot remove these.
5. **Validate.** Algorithm 1. Errors go back to the VLM, up to three times.
6. **Save.** A new version each time, recording the VLM and prompt version; never overwritten.

### Three implementation decisions

- **One validator, not two.** The app and the server both need Algorithm 1. Either build the validator from `AnchorCore` as a command-line tool the server calls, or write it twice and run both against the same set of test graphs. The second is simpler to deploy; the shared tests are what keep the two in step.
- **A structured motion field.** Table 5.5 holds motion as free text ("pry from the front edge, then lift"). Add a small structured form beside it: kind (rotate, slide, pry, lift, tilt and pull), direction, and edge or axis. W2 can then be drawn with no model call at run time. This is an addition to the schema in Section 4.9.
- **Tool names from a fixed list.** `tool(a)` must be one of the manifest's tool identifiers (Section 10), so W3 always has a model to show.

### Server services

| Service | Does | Notes |
| --- | --- | --- |
| Graph | Returns a saved graph for a model and task, or starts a build | A build takes longer than a normal request; return a job the app polls, and show progress on S3 |
| Resource store | Holds manuals, guides and video links per model | Public resources only (report 6.7.5) |
| Segmentation | MobileSAM: crop and box in, mask and refined box out | Stateless |
| Localisation (new, optional) | Whole-face part boxes for a laptop with no enrolment | Only if Section 6's S4 passes its gate |
| Configuration | Serves the tunable values | Lets the bench change thresholds without a rebuild |

The server is the team's 12 GB GPU machine and must be reachable from outside the local network, as the report requires. Put it behind HTTPS and an API key.

### Reference graphs and evaluation

Build and hand-review nine reference graphs: three laptops by three tasks. They serve twice, as fixtures for testing guidance (report 6.7.6) and as ground truth for the builder.

| Measure | Target (proposed) |
| --- | --- |
| Steps in the reference that the built graph contains | At least 90% |
| Hard preconditions and hazard rules present | All; they are added by code, so a miss is a bug |
| Screw count exact | At least 8 of 9 graphs; the camera check at session start must catch the rest |
| Same inputs, three runs | Identical step lists |
| Graphs that pass validation but are wrong | Counted and described; this is the number R-05 asks for |

## 13. Data plan: laptops, recordings and labelling

Almost every open measurement depends on recordings of the three study laptops. Choose the laptops first, then record in one structured campaign of about 20 hours, with labels produced mostly by the recording script and mask propagation.

### Choosing the three laptops

The report leaves this until the evaluation is planned. It cannot wait: enrolment, graphs and the tool recordings all need the actual machines.

- Non-Apple, two from one maker and one from another (report scope).
- Owned or on loan for the whole semester, with a spare bottom cover for forced mistakes (R-08).
- A public service manual for the exact model number.
- An internal battery with a connector, so the hazard step exists. Check the Lenovo G50-80 against this before counting on it.
- Between them: a memory slot, an M.2 or 2.5-inch drive, and at least one screw hidden under a foot or label.

### Recording campaign

| Set | Clips | Purpose | Labels |
| --- | --- | --- | --- |
| A. Clean runs | 3 laptops × 3 tasks × 2 operators = 18 | End-to-end step recognition (NFR-03) and latency (NFR-02) | Step boundaries |
| B. Staged mistakes | 6 kinds in Table 5.8 × 3 laptops × 2 takes = 36 | W6, recovery, warning latency (NFR-10) | Start time and kind of each wrong action |
| C. Hazard moments | 3 laptops × 5 takes = 15 | Hazard recall (NFR-01) | Hazard intervals |
| D. Perception sets | About 60: screws in, out and half out; each component state; tools; hover against touch; registration drift | Model training and thresholds | Mostly automatic |
| E. VLM crops | About 150 images | VLM comparison and calibration | Value per crop |

Stage the start of each mistake, not the damage, and record forced ones only on the spare cover, as R-02 and R-08 require. Every clip goes through the recorder, so it carries depth, pose and intrinsics.

### Splits

Split by laptop and by recording day, never by frame. Frames from one video of one laptop are near-duplicates, so a validation split from the same footage says little about new conditions. That is part of why the Lenovo validation score sat so far above the test score. For models trained on the study laptops, keep one operator and one recording day out of training. The unseen fourth laptop stays out of all training and tuning.

### Labelling, automated with human audit

| Label | Produced by | Human work |
| --- | --- | --- |
| Slot states | The recording script plus registration | None beyond checking the script was followed |
| Part and tool boxes and masks | One click per clip, then video-segmenter propagation | Review |
| Step boundaries, mistake starts, hazard intervals | A second person presses an event button in the recorder during the take | A second annotator re-labels a sample; report agreement (R-02) |
| New frames in later sprints | Pre-labels from the current detector | Correct, then retrain |

Audit rule: review every validation and test frame and a random 10% of training frames. If more than 5% of the sample is wrong, review the whole batch. Version each dataset with a manifest of file hashes and a short card stating its source, licence and split.

Sections 5 to 7 list the public datasets by purpose. Two licence points apply across them: the Danish Technological Institute set is share-alike, and iFixit photos are not available for training.

## 14. Evaluation and acceptance

Evaluate in three layers: each model alone, the whole pipeline replayed on bench recordings against the ten NFRs, and the user study. Set every threshold on a tuning split and freeze the configuration before the test split is run once.

### Layer 1: each model alone

Sections 5 to 8 and 12 give a proposed target for each model. All are proposals for Deliverable III, not report requirements. Keep the bootstrap confidence intervals and paired comparisons you already use.

### Layer 2: the pipeline on bench recordings

| NFR | Target (report) | Measured on | How |
| --- | --- | --- | --- |
| NFR-01 hazard recall | At least 0.95 | Set C | Share of staged hazard moments with W5 shown |
| NFR-02 next step | p95 at most 2 s | Set A | Labelled moment a finished step is clearly visible, to the log time its next widgets appear |
| NFR-03 steps recognised | At least 90% | Set A, with hands and tools hiding parts | Recognised steps over labelled steps |
| NFR-04 anchor drift | At most 2 cm (5 mm internal target for rings) | Drift clips | Sticker positions against projected slots |
| NFR-06 modifiability | Configuration change only | A demonstration | Swap the detector file and the VLM provider; record the diff |
| NFR-07 privacy | Off by default | A test | No frames on disk after a session without consent |
| NFR-08 platform | iPhone 12 Pro or iPad Pro (2020) or later | Your device | State the oldest device actually tested |
| NFR-09 VLM calls | At most 3 per minute | Sets A and B | Calls over session minutes, from the log |
| NFR-10 warning latency | p95 at most 1 s | Set B | Labelled start of the wrong action to W6 shown |

NFR-05 (mean SUS of at least 68) is measured in the user study.

**The one formal experiment.** The report promises a test of the hazard threshold against the default. Replay sets B and C twice, with the W5 threshold at 0.10 and at 0.50, and compare hazard recall and false alarms on the same moments with a paired test.

**Ablations.** Replay makes these cheap: no gate (single-frame decisions), whole-frame screw detection instead of slot crops, exact against partial matching, VLM confirmation on and off, enrolled against unenrolled detector.

**Calibration.** A reliability diagram per confidence source and per widget class, as Section 5.9.4 of the report requires.

### Tuning protocol

1. Split bench clips into tuning and test by take, never by frame.
2. Set `c_high`, `c_low`, `occ_share`, `touch_share`, `h_touch`, `min_inliers`, `max_error`, θ and the cooldown on the tuning split.
3. Freeze the configuration and record its hash.
4. Run the test split once and report the results with that hash.

### Layer 3: the user study

The design belongs to Deliverable III: unguided, manual-guided and ANCHOR conditions, measuring task success, errors by kind, recovery, time and SUS. Two things belong in the build schedule. Run a pilot with three or four novices in Sprint 6, because R-04 (novices make no mistakes even with the manual) can only be answered by watching people. And freeze the app before the main study starts; no changes between participants.

### Regression suite

Six clips from set A run in replay on every merge. Step recognition and median latency must not get worse.

## 15. Schedule and time budget

The build needs about 685 hours of human time, or close to 790 with a 15% contingency. For three people across a preparation sprint and eight two-week sprints, that is about 15 hours per person per week. The estimate assumes AI-assisted development throughout; the hours are for specifying, reviewing, testing on the device, recording and running the study.

These are planning estimates, not measurements. Replace them with actuals after Sprint 1.

&#91;embedded content: ANCHOR roadmap · six phases, five gates\]

Each diamond is a gate between phases. The highlighted phase is the report's vertical slice running from the camera; everything after it adds breadth to a pipeline that already works.

### Three working modes

| Mode | What it means | Where the human time goes |
| --- | --- | --- |
| AI-led | A coding agent writes the code and its tests from the report section and this plan's acceptance row | Writing the task, reviewing the diff, running it on the device or bench |
| Automated | Scripts run unattended: training, evaluation, mask propagation, asset builds, replay | Starting the run and auditing the output |
| Human | Cannot be delegated | Recording, labelling audit, device testing, the study |

### Work packages

| # | Work package | Human hours | Mode | Sprints |
| --- | --- | --- | --- | --- |
| 1 | Repository, CI, configuration, recorder, replay harness, debug overlay | 40 | AI-led | 0 to 1 |
| 2 | Mat, 6-DoF registration, anchoring, drift measurement | 45 | AI-led | 1 to 2 |
| 3 | Recording campaign, sets A to E (two people for about 20 hours) | 40 | Human | 1 to 6 |
| 4 | Label review and dataset versioning | 30 | Automated | 1 to 6 |
| 5 | Screw models: crop classifier, laptop fine-tune, export, calibration | 25 | AI-led and automated | 2 to 3 |
| 6 | Component models: relabel, enrolment, state readers, leave-one-out, server comparison | 50 | AI-led and automated | 0 to 6 |
| 7 | Tools, hands and contacts | 40 | AI-led and automated | 3 to 4 |
| 8 | On-device perception runtime: Core ML, crop pipeline, scheduling, tracker, timing | 40 | AI-led | 0 to 4 |
| 9 | `AnchorCore`: graph model, validator, gate, sequencer, recovery, hazard rules, trigger policy, timers, log | 50 | AI-led | 1 to 5 |
| 10 | Graph builder, resource store, nine reference graphs, builder evaluation | 45 | AI-led | 1 to 5 |
| 11 | VLM adapter, prompts, model comparison, calibration | 25 | AI-led | 4 to 6 |
| 12 | Server services, deployment, security | 20 | AI-led | 2 to 5 |
| 13 | 3D assets and manifest | 15 | AI-led and automated | 4 |
| 14 | Widgets W1 to W6 and screens S1 to S7 | 55 | AI-led | 2 to 5 |
| 15 | Integration, bench evaluation, tuning, ablations | 60 | Automated and human | 3 to 7 |
| 16 | User study: pilot, main sessions, analysis | 60 | Human | 6 to 8 |
| 17 | Deliverable III, final report, demo video | 45 | Human, with AI-drafted tables and figures | 6 to 8 |
|  | **Total** | **685** |  |  |

Unattended training needs about 30 to 40 GPU-hours on the 12 GB machine. Your own note applies: training runs alone on that PC, so schedule it overnight.

Only packages 3 and 16 are human from end to end: 100 hours. The time in the other packages goes mostly to device testing, because an agent can run tests and replays on a Mac but cannot see the app on a real laptop.

### Sprint plan

| Sprint | Weeks | Build | Gate at the end |
| --- | --- | --- | --- |
| 0 | Before FYP-II | Choose laptops, buy two tool kits, print the mat. Repository and CI. Recorder v0. Relabel the 176 test images. Core ML export of YOLO26n and MobileSAM with timings on the device. Collect the three service manuals. | **G0:** tier B fits in 150 ms on the device; laptops in hand |
| 1 | 1 to 2 | Mat pose and registration. Final recorder. Perception recordings of laptop 1. Graph model and validator. Reference graph for laptop 1, storage task. Hand-mask test. |  |
| 2 | 3 to 4 | Screw crop classifier and laptop screw detector. Gate, sequencer, hazard rules, trigger policy. Tap-driven app with W1 rings, the W5 banner and the W6 card. Enrol laptop 1's components. | **G1:** tap-driven slice runs laptop 1's storage task end to end; rings within 5 mm |
| 3 | 5 to 6 | Camera readings replace taps: screws, cover, component state. Segmentation service. Record sets A to C for laptop 1. | **G2:** the report's vertical slice (one laptop, storage, W1, W5, W6) meets NFR-02 on bench clips; enrolled AP50 at least 0.85 |
| 4 | 7 to 8 | Tools, hands, contacts and early warning. 3D assets, W3, W2. VLM adapter, VERIFY and hazard clearing. |  |
| 5 | 9 to 10 | Recovery planner and every mistake kind. Graph builder and nine reference graphs. Enrol laptops 2 and 3. W4 and the screw-length map. Sessions screen. | **G3:** three laptops by three tasks run on the bench with all six widgets |
| 6 | 11 to 12 | Finish the recording campaign. Tuning protocol and calibration. Leave-one-laptop-out and server-localisation results. Pilot with three or four novices. | **G4:** configuration frozen; NFRs measured on the test split; the pilot shows novices do make mistakes (R-04) |
| 7 | 13 to 14 | Main user study. Ablations and the hazard-threshold experiment. |  |
| 8 | 15 to 16 | Analysis, reports, demo video, and the contingency buffer. |  |

If G0 fails, shrink tier B before anything else: fewer slot crops per batch, a smaller tool model, or a lower processing rate. If G2 slips, hold the scope at the vertical slice and move the second and third laptops later; do not start tools on top of an unstable slice.

## 16. Risks, fallbacks and open decisions

Each risk below has a test early enough to react to, and a fallback that keeps the study possible. They add to the report's R-01 to R-12; they do not replace them.

| Risk | Early test | Fallback |
| --- | --- | --- |
| Tier B is too slow on the device | G0 timing in Sprint 0 | Lower the processing rate, shrink the models, run whole-frame detection only at registration |
| Dark screws on dark covers cannot be read from crops | Sprint 2, classifier on the target laptops | 4K video format, then a VLM crop, then a tap (R-10) |
| Registration is unstable while hands cover the laptop | Sprint 1 drift clips with hands in view | Hold the last device transform on the mat pose; when untrusted, every fact is unknown and widgets dim, as the report specifies |
| Components do not transfer to an unseen laptop | Sprint 6, leave-one-laptop-out and server comparison | Claim enrolled laptops only, and state it |
| LiDAR cannot separate hovering from touching at a tool tip | Sprint 4 hover clips | Lengthen `min_time`, keep the W6 threshold above W5, and rely on catching the action after it completes, which F4 already allows |
| Built graphs are wrong for real manuals (R-05) | Sprint 5, nine reference graphs | Report the builder's accuracy. Reviewed graphs go into the study only if that accuracy is poor |
| VLM is slow or costly when clearing a hazard | Sprint 4 model comparison | A faster model in the in-session slot; the banner staying up is the safe direction |
| The device overheats in a 25-minute session | Sprint 2 full-length run | Lower the tier B rate with thermal state; use a stand so the user is not holding a warm device |
| iFixit's terms forbid training on its data | Now | Manufacturer manuals and your own recordings; written permission for anything else |
| Ultralytics models are licensed AGPL-3.0 or Enterprise | Now | Fine for an open academic repository; check with your supervisor before distributing the app any other way |
| The study slips past the semester | Pilot in Sprint 6 | Recruit during Sprint 4; freeze scope at G3 |

I am not a lawyer; treat the two licence rows as flags to check, not advice.

### Decisions needed before Sprint 1

- [ ] Which three laptops, and a spare bottom cover for forced mistakes
- [ ] Which LiDAR iPhone or iPad and which Mac the team uses daily
- [ ] 3D movement widgets in scope, moving that item out of future work
- [ ] 6-DoF mat pose in place of the two homographies, with the homographies as fallback
- [ ] VLM providers for the fast and strong slots, and a monthly budget
- [ ] Where the server runs and how it is exposed to the internet
- [ ] FYP-II calendar: start date, deliverable dates, and hours per person per week
- [ ] Whether the study may use hand-reviewed graphs

## 17. Sources

Your Deliverable II report, the concept demo and the nine benchmark screenshots are the primary sources for every project fact and result above. External pages opened for this plan:

- [Guided Reality, arXiv 2508.03547](https://arxiv.org/abs/2508.03547) and its [HTML version](https://ar5iv.labs.arxiv.org/html/2508.03547): pipeline, models, latency, accuracy, tool library
- [Ultralytics YOLO26 documentation](https://docs.ultralytics.com/models/yolo26/): architecture, export formats, licence
- [Dataset for laptop screen frame and screw detection, Zenodo](https://zenodo.org/records/18483802)
- [DiFilippo, Jouaneh and Jedson, Applied Sciences 14(14), 2024](https://doaj.org/article/9beac22661cc4a74bfab3d9c98e538e8)
- [Zhang et al., arXiv 2603.27441](https://arxiv.org/abs/2603.27441)
- [Laptop Disassembly, Roboflow Universe](https://universe.roboflow.com/robothon1/laptop-disassembly)
- [MECCANO dataset](https://iplab.dmi.unict.it/MECCANO/)
- [ENIGMA-51 dataset](https://fpv-iplab.github.io/ENIGMA-51/)
- [iFixit content licensing](https://www.ifixit.com/Guide/Info/Licensing)
- [Apple: capturing photographs for Object Capture](https://developer.apple.com/documentation/realitykit/capturing-photographs-for-realitykit-object-capture)
- [Apple: ObjectCaptureSession](https://developer.apple.com/documentation/realitykit/objectcapturesession)
- [Apple: personSegmentationWithDepth](https://developer.apple.com/documentation/arkit/arconfiguration/framesemantics-swift.struct/personsegmentationwithdepth)
- [Gemini API image understanding](https://ai.google.dev/gemini-api/docs/image-understanding)
- [Hunyuan3D 2.1 repository](https://github.com/Tencent-Hunyuan/Hunyuan3D-2.1)
- [Apple's Core ML build of SAM 2.1 Tiny](https://huggingface.co/apple/coreml-sam2.1-tiny), seen in search results only

Statements about ARKit, RealityKit, Core ML and OpenCV that carry no link come from general knowledge of those frameworks. Verify each on your device and iOS version in Sprint 0.
