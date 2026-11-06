# DL Streamer vs. upstream GStreamer: element inventory and upstreaming analysis

Date: 2026-11-06

## Scope

This document answers: which DLS elements/libraries can be upstreamed to GStreamer, which
gaps GStreamer has relative to DLS, and which upstream elements would collide with ours.

**Inference elements (decision):** the DLS inference elements (`gvadetect`, `gvaclassify`,
`gvainference`, `gvamono3d`, `gvagenai`, `gvaaudiotranscribe`) and `gvatrack` are kept only
as a *potential* Phase 3 of the plan (section 6). They are heavily Intel-specific (OpenVINO,
the DLS `inference_backend`, Intel's VAS tracker code) and collide with the upstream
inference architecture, so it is possible that they are never upstreamed. Phases 1 and 2 are
designed to be independent of them.

**Partial upstreaming:** several DLS elements already have upstream equivalents
(`gvastreammux` / `gvastreamdemux`, `gvawatermark`, `gvaaudiotranscribe`, the inference
elements). Such an element will most likely not be accepted in full - the community tends
to reject duplicates of existing elements - although this is not certain and could be
confirmed with the maintainers before deciding. The safer default is to contribute the
individual features that the upstream equivalent lacks (e.g. synchronization modes for
`analyticscombiner`, tracking-ID / keypoint / OBB rendering for `objectdetectionoverlay`).
The per-category tables mark these cases as *partial*.

## 1. What upstream GStreamer already provides

### 1.1 Analytics library (`gst-plugins-bad/gst-libs/gst/analytics`, LGPL-2.1+)

| API | Since |
|---|---|
| `GstAnalyticsRelationMeta`, `GstAnalyticsODMtd`, `GstAnalyticsClsMtd`, `GstAnalyticsTrackingMtd` | 1.24 |
| `GstAnalyticsSegmentationMtd`, `GstTensorMeta` / `GstTensor` | 1.26 |
| `GstAnalyticsBatchMeta` (caps `multistream/x-analytics-batch`), `GstAnalyticsTensorMtd`, `GstAnalyticsModelInfo` (`.modelinfo` ini sidecar files) | 1.28 |
| `GstAnalyticsGroupMtd`, `GstAnalyticsKeypointMtd`, `GstAnalyticsTextMtd`, `GstAnalyticsTensorProvenanceMtd`, `GstTensorListMeta` | 1.30 |

Python: `gst-python` ships a `GstAnalytics` override module (`gi/overrides/gstanalyticsmodule.c`); no high-level helper library comparable to `gstgva`.

### 1.2 AI-related elements

| Category | gst-plugins-bad | gst-plugins-rs |
|---|---|---|
| Inference | `onnxinference` (execution providers: CPU, CUDA, DirectML, HIP, MIGraphX, VitisAI, VSI, CoreML; memory: system, D3D12, HIP; **no OpenVINO EP, no VAMemory/DMABuf import**), `tfliteinference` (+ `tfliteedgetpuinference`, `tflitevsiinference`, `tflitexnnpackinference`, `tfliteexternalinference`), `executorchinference`. All emit raw `GstTensorMeta`. | `burn-yoloxinference` |
| Tensor decoders | `ssdtensordec`, `ssdobjectdetector` (legacy), `yolov8tensordec`, `yoloobbv8tensordec`, `yolosegv8tensordec`, `yolo26tensordec`, `classifiertensordecoder`, `facedetectortensordecoder`, `ioutracker`, `tensordecodebin` | `yolov8tensordec2`, `yolov8obbtensordec`, `yolov8segtensordec`, `yolo26tensordec2`, `yolo26obbtensordec`, `yolo26segtensordec`, `yoloxtensordec`, `handdetectiontensordec`, `handlandmarktensordec`, `pipnettensordec`, `ctctexttensordec`, `zerodcetensordec` |
| Batching | none (meta only) | `analyticscombiner` (GstAggregator, `batch-duration`, per-pad `batch-strategy`, `force-live`), `analyticssplitter` |
| Tracking | `ioutracker` (IoU only); `cvtracker` in `ext/opencv` does not use analytics meta | none |
| Visualization | `objectdetectionoverlay`, `segmentationoverlay` | none analytics-aware |
| External metadata | `rtponvifparse`, `rtponviftimestamp`; `GstVideoSEIUserDataUnregisteredMeta` produced by `h264parse`/`h265parse`; `codecseiinserter` | `rsonvif` (ONVIF metadata parse/combine/overlay/pay/depay), `onvifmeta2relationmeta`, `relationmeta2onvifmeta` |
| GenAI / speech | none | `llamacpp-texttransform` (text -> LLM -> text), `whispertranscriber`, `deepgramtranscriber`, `speechmaticstranscriber`, `elevenlabs` |
| Misc | `ext/nvdswrapper` (`nvdsdewarp`, a DeepStream SDK wrapper), `switchbin`, `valve`, `input-selector`, `output-selector` | `originalbuffersave`/`originalbufferrestore`, `togglerecord`, `fallbackswitch`, `gopbuffer`, `textaccumulate` |

Not present upstream: MQTT/Kafka publishers, JSON serialization of analytics metadata,
zone/tripwire/dwell-time analytics, ROI crop / secondary (per-ROI) inference, 3D object
detection metadata, metadata-driven routing, GPU-accelerated analytics overlay.

## 2. DL Streamer inventory (verified)

Plugins: `videoanalytics` (most elements), `3delements`, plus standalone plugins
`gvametapublish` (+ `gvametapublishmqtt`, `gvametapublishkafka`), `gvapython`,
`gvaattachroi`, `gvarealsense`, `latency_tracer`. License: MIT.

| Element | Base | GstAnalytics usage (verified) | Custom metadata |
|---|---|---|---|
| `gvadetect`, `gvaclassify`, `gvainference`, `gvamono3d` | `GvaBaseInference` (GstBaseTransform), OpenVINO only, preprocessing OpenCV / VA / OpenCL / D3D11 / D3D12 | produce `ODMtd` (incl. oriented), `ClsMtd`, `SegmentationMtd`, `KeypointMtd`, `TensorMtd` (raw tensors), `Camera3DODMtd`; consume `ODMtd` for `inference-region=roi-list` | `GstAnalyticsCamera3DODMtd` (DLS extension), keypoint descriptor registry |
| `gvatrack` | GstBaseTransform; tracker source code (VAS, MIT) lives in `src/monolithic/gst/elements/gvatrack/vas`, OpenCV based; types `zero-term`, `short-term-imageless`, `zero-term-imageless`, `deep-sort` | consume `ODMtd`, produce `TrackingMtd` (via `gva_utils`) | none required |
| `gvamotiondetect` | GstBaseTransform, OpenCV / VA / OpenCL | produce `ODMtd` | none |
| `gvastreammux` / `gvastreamdemux` | GstElement; caps `multistream/x-analytics-batch(meta:GstAnalyticsBatchMeta)` in `container` mode; `sync-mode` (none, first-pts, segment, pipeline, ntp); `output-mode` (passthrough, container); `max-queue-size` | `GstAnalyticsBatchMeta` | none |
| `gvametaaggregate` | GstAggregator | merges `RelationMeta` from several branches (OD, Cls, Tracking, Keypoint, Segmentation, Tensor, Group) | none |
| `gvaanalytics` | GstBaseTransform | consume `ODMtd`, `TrackingMtd`; produce DLS `ZoneMtd`, `TripwireMtd`, `DwellTimeMtd` | zone/tripwire/dwell-time Mtd types registered through the GstAnalytics API; `WatermarkDrawMeta` for rendering |
| `gvaattachroi` | GstBaseTransform | produce `ODMtd` for manually defined ROIs | none |
| `gvaframeselect`, `gvaroiscale` | GstBaseTransform | consume `ODMtd`/`ClsMtd`; `gvaroiscale` produces `ODMtd` with rescaled coordinates | none |
| `gvaeventrecorder` | GstBaseTransform | consume `ODMtd`, adds `ClsMtd` tags | none |
| `gvametaconvert` | GstBaseTransform; props `format`, `add-tensor-data`, `add-empty-results`, `source`, `tags`, `timestamp-utc`, `timestamp-microseconds`, `add-rtp-timestamp`, `signal-handoffs` | consume `RelationMeta` (OD, Cls, Tracking, 3D OD, Camera 3D OD, batch streams) | produces `GstGVAJSONMeta` (JSON, DLS-specific schema) |
| `gvametapublish`, `gvametapublishfile`, `gvametapublishmqtt` (paho), `gvametapublishkafka` (librdkafka) | GstBin + GstBaseTransform (pass-through) | - | consume `GstGVAJSONMeta` |
| `gvawatermark` (GstBin) / `gvawatermarkimpl`, `gvawatermark3d` | renderers: CPU (OpenCV), VA + OpenCL, D3D11, D3D12 | render `ODMtd` (incl. oriented), `ClsMtd`, `TrackingMtd`, `KeypointMtd`, `SegmentationMtd`, `ZoneMtd`, `TripwireMtd`, `Camera3DODMtd` | `WatermarkTextMeta`, `WatermarkCircleMeta`, `WatermarkDrawMeta` |
| `gvagenai` | GstBaseTransform; OpenVINO GenAI or HTTP (OpenAI-compatible) backend | consume `ODMtd`, produce `ClsMtd` | - |
| `gvaaudiotranscribe`, `gvaaudiodetect` | audio inference base | `ClsMtd` | - |
| `gvatagger` | GstBaseTransform (H.264/H.265 NAL analysis without decode) | - | trigger meta |
| `gvafpscounter`, `gvafpsthrottle`, `gvadeskew`, `gvapython`, `gvarealsense`, `latency_tracer` | various | `gvafpscounter` counts `ODMtd` | - |
| `g3dlidarsrc`, `g3dlidarparse`, `g3dinference`, `g3drender`, `g3dobjectfuser`, `g3dbevfusion`, `g3dradarprocess` | GstPushSrc / GstBaseTransform; caps `application/x-lidar`, batch container | produce/consume DLS `3DODMtd`, `TrackingMtd`, `BatchMeta` | `LidarMeta`, `RadarProcessMeta`, `GstAnalytics3DODMtd` (DLS extension) |

Model configuration: DLS reads pre/post-processing parameters from the `model_info` section
of the OpenVINO IR `rt_info` (or legacy `model-proc` JSON); upstream uses `.modelinfo` ini
sidecar files (tensor ids, dims, type, scales/offsets, dims order).

## 3. Per-category matrix: DLS vs. upstream vs. DeepStream

Upstreaming promise: **High** = clear upstream gap, generic design, LGPL-compatible,
no API change; **Medium** = gap exists but requires redesign or new library API;
**Low** = collides with an existing upstream element or is too niche; **Partial** = an
upstream equivalent already exists, so a full upstream of the element itself would most
likely be rejected as a duplicate (not certain), but selected features can be contributed to
that equivalent. Each assessment also names the phase of the plan in section 6 the element is
assigned to, or states that it is not scheduled.

### 3.1 Inference (potential Phase 3 - Intel-specific, may never be upstreamed)

DLS inference elements are assigned to Phase 3 of the plan (section 6) with the explicit
caveat that they may never be upstreamed: they are heavily Intel-specific and upstream
already has equivalents, so at most parts of them could ever be contributed. The table
documents the overlap so that the Phase 1 and 2 candidates can be designed to interoperate
with both the DLS and the upstream inference stacks.

| DLS | Upstream | DeepStream | Assessment |
|---|---|---|---|
| `gvadetect` / `gvaclassify` / `gvainference` (monolithic: preprocessing + OpenVINO + postprocessing; `inference-region=roi-list`, `model-instance-id`, `batch-size`) | split model: `*inference` -> `GstTensorMeta` -> `*tensordec`; full-frame only; no OpenVINO; GPU memory only D3D12/HIP | `nvinfer` (config file; primary/secondary modes) | **Partial at best, Phase 3 (uncertain)** - architectural collision with the upstream split model plus a hard dependency on OpenVINO and the DLS `inference_backend`. Upstream equivalents exist, so a full upstream of the elements as such would most likely be rejected as a duplicate; what could realistically be contributed is a rewrite in the upstream style (a native `openvinoinference` emitting `GstTensorMeta` + tensor decoders, or an OpenVINO execution provider for `onnxinference`), not a port. Consequence for Phases 1-2: elements upstreamed from DLS must consume standard `GstAnalytics` metadata only, so they work behind `onnxinference` / `tfliteinference` + tensor decoders as well as behind `gvadetect`. |
| DLS post-processors (YOLO, SSD, segmentation, keypoints, labels, custom `.so`) | tensor decoders cover YOLO (v8/obb/seg/26/X), SSD, classification, face, hands, PIPNet | post-processing inside `nvinfer` / `nvdspostprocess` | Internal to the DLS inference elements, so they follow them (Phase 3). Most are already covered by upstream tensor decoders; a COCO pose keypoint decoder would be the only clear addition. |
| `gvamono3d` (`Camera3DODMtd`), `g3dinference` (`3DODMtd`) | no 3D OD in the analytics library | `nvds_3d_*` | `gvamono3d`: **Phase 3 (uncertain)**. The metadata types (`GstAnalytics3DODMtd` / `GstAnalyticsCamera3DODMtd`) are decoupled from it and proposed in **Phase 2** together with `gvawatermark3d` (see 3.6). `g3dinference` (lidar): not scheduled. |
| `gvagenai` (VLM; OpenVINO GenAI or HTTP backend) | `llamacpp-texttransform` (text only), `GstAnalyticsTextMtd` (1.30) | none (external services) | **Phase 3 (uncertain)** - a video -> text VLM element is a genuine upstream gap, but the OpenVINO GenAI backend is Intel-specific; the HTTP backend would be the only realistically acceptable part. Independently of upstreaming, DLS could adopt `TextMtd` as the output format for interoperability. |
| `gvaaudiotranscribe`, `gvaaudiodetect` | `whispertranscriber` | `nvdsasr` | `gvaaudiotranscribe`: **Phase 3 (uncertain)** - an upstream equivalent exists, so acceptance is doubtful even apart from the OpenVINO dependency; at most individual features could be offered to `whispertranscriber`. `gvaaudiodetect` (legacy): not scheduled. |

### 3.2 Batching

| DLS | Upstream | DeepStream | Assessment |
|---|---|---|---|
| `gvastreammux` / `gvastreamdemux` (GstElement, `sync-mode`, `output-mode` passthrough/container, `max-queue-size`) | `analyticscombiner` / `analyticssplitter` - same caps, GstAggregator, `batch-duration`, `force-live` | `nvstreammux` / `nvstreamdemux` | **Partial, Phase 2** - an upstream equivalent already exists, so a full upstream of the elements themselves would most likely be rejected as a duplicate (not certain). What can realistically be contributed are the features `analyticscombiner` / `analyticssplitter` lack: passthrough mode, the synchronization modes (`first-pts`, `segment`, `pipeline`, `ntp`), bounded per-pad queues and heterogeneous sources (video + lidar). In parallel, DLS inference elements should accept `multistream/x-analytics-batch`. |
| cross-stream batching via `model-instance-id` | no equivalent | `nvstreammux` | **Medium** - worth an RFC, needs consensus; not scheduled. |
| `gvametaaggregate` (merge `RelationMeta` from `tee` branches) | none | `nvdsmetamux` | **Phase 1** - generic "analytics meta merger" for parallel `tee` branches of the *same* stream; no upstream equivalent. Upstream may push back towards the combiner approach, so the distinction from `analyticscombiner` (merging metadata of one buffer vs. batching different streams) should be stated up front. |

### 3.3 Metadata ingestion

| DLS | Upstream | DeepStream | Assessment |
|---|---|---|---|
| `gvaattachroi` (manual ROI -> `ODMtd`) | no element | ROI in `nvdspreprocess` | **Phase 1** - small, generic and dependency-free. |
| none (DLS has no MQTT subscriber, ONVIF-M or SEI ingestion) | ONVIF-M complete in rs (`onvifmeta2relationmeta`), SEI meta in parsers | `nvdsmetaextract` / `nvdsmetainsert` (SEI) | **DLS gap, not an upstream gap** - adopt upstream; shared candidate: SEI payload <-> `RelationMeta` element. |

### 3.4 Tracking

| DLS | Upstream | DeepStream | Assessment |
|---|---|---|---|
| `gvatrack` - zero-term, short-term-imageless, zero-term-imageless, deep-sort; tracker source is in-tree (MIT, OpenCV), no binary blob; produces `TrackingMtd` | `ioutracker` only; `cvtracker` is not analytics-aware | `nvtracker` (IOU / NvSORT / NvDCF / DeepSORT) | **Phase 3 (uncertain)** - technically the largest and most generic upstream gap, and OpenCV is an accepted dependency (`ext/opencv`). Nevertheless the element is tied to the Intel stack: it links OpenVINO and the DLS `inference_backend` (buffer mapping, VA context), the trackers are Intel's VAS code base, and deep-sort consumes features pre-extracted by `gvainference`. Upstreaming would require decoupling all of that (features via `TensorMtd`, standard GstVideo mapping) and coexisting with `ioutracker`, so it is scheduled together with the inference elements and may never happen. |
| `gvamotiondetect` -> `ODMtd` | `motioncells` without analytics meta | - | **Phase 2** - either our element or a patch making `motioncells` emit `ODMtd`. |

### 3.5 Data analytics

| DLS | Upstream | DeepStream | Assessment |
|---|---|---|---|
| `gvaanalytics` (zones, tripwires, dwell time; DLS `ZoneMtd` / `TripwireMtd` / `DwellTimeMtd`) | none | `nvdsanalytics` | **Phase 1** - the element together with its new metadata types (`ZoneMtd`, `TripwireMtd`, `DwellTimeMtd`) proposed to `gst-libs/gst/analytics`. The API part needs maintainer consensus and can only land in a minor release (`Since: 1.32` at the earliest), so the API proposal should be opened first and the element can follow within the same cycle. |
| `gvaframeselect`, `gvaroiscale` (ROI crop / letterbox for VLM) | none ("analyticsroicrop") | `nvdspreprocess` | **Phase 1** - closes the upstream "secondary inference" gap; the crop should produce a batch with `BatchMeta` / `TensorProvenanceMtd`. |
| `gvatagger` (NAL analysis without decode) | none | - | **Phase 1** - niche but generic and dependency-free. |
| `gvafpscounter`, `latency_tracer`, `gvafpsthrottle` | `fpsdisplaysink`, `latency` tracer, `videorate` / `clocksync` | - | **Not scheduled** - duplicates. |

### 3.6 Visualization

| DLS | Upstream | DeepStream | Assessment |
|---|---|---|---|
| `gvawatermark` (+impl, GstBin): OD/OBB, Cls, Tracking, Keypoint, Segmentation, Zone, Tripwire, custom `Watermark*Meta`; backends CPU / VA+OpenCL / D3D11 / D3D12 | `objectdetectionoverlay`, `segmentationoverlay` | `nvdsosd` | **Partial, Phase 2** - an upstream equivalent already exists, so a full upstream of the element itself would most likely be rejected as a duplicate overlay (not certain). What can realistically be contributed: rendering of tracking IDs, keypoints/skeletons, OBB, zones/tripwires and classification labels in `objectdetectionoverlay`, and a GPU renderer (VA + OpenCL) for the upstream overlays. `Watermark*Meta` should map to `GstVideoOverlayCompositionMeta`. |
| `gvawatermark3d`, `g3drender` | none | `nvds_3d` | `gvawatermark3d` together with the 3D metadata types (`GstAnalytics3DODMtd`, `GstAnalyticsCamera3DODMtd`): **Phase 2**, deliberately without `gvamono3d` (Phase 3). `g3drender` (lidar point clouds): not scheduled. |

### 3.7 Messaging

| DLS | Upstream | DeepStream | Assessment |
|---|---|---|---|
| `gvametaconvert` (`RelationMeta` -> JSON in `GstGVAJSONMeta`) | no JSON; `relationmeta2onvifmeta` exists | `nvmsgconv` | **Phase 1** - a JSON serializer is a real gap, but the DLS JSON schema is vendor-specific and the community leans towards ONVIF. Realistic form: an `analyticsjsonserialize` element with `text/x-raw` / `application/json` on its src pad instead of `GstGVAJSONMeta` (caps-typed output is the upstream idiom); DLS-specific fields would stay in DLS. |
| `gvametapublish{file,mqtt,kafka}` (pass-through transform) | no MQTT/Kafka | `nvmsgbroker` (sink) | **Phase 1** - upstream expects generic sinks (`mqttsink`, `kafkasink`) consuming buffers, not meta; requires redesign (idiom: `... ! jsonserializer ! mqttsink`). paho / librdkafka are acceptable as optional dependencies. |

### 3.8 Branching / conditionals

| DLS | Upstream | DeepStream | Assessment |
|---|---|---|---|
| `gvaeventrecorder` (pre/post frames after a class detection) | `togglerecord`, `gopbuffer`, `valve` - no metadata-driven control | - | **Phase 1** - most likely as a small generic `analyticsvalve` / `analyticsselector` (pass or route buffers by a predicate on `RelationMeta`), leaving the actual recording to `togglerecord`. |
| `gvapython` | `python` plugin loader in gst-python | `nvdsvideotemplate` | **Not scheduled** - duplicate concept. |

### 3.9 Decode / encode

DLS has no codecs of its own; it uses upstream `va*` / `qsv*` / `msdk` (see the mapping in
`docs/user-guide/dev_guide/converting_deepstream_to_dlstreamer.md`). The only candidate is
`gvarealsense` (upstream has only `openni2src`) - niche, not scheduled; librealsense is
Apache-2.0.

## 4. Gaps in GStreamer relative to DL Streamer

1. No trackers beyond IoU.
2. No secondary (per-ROI) inference flow (crop -> batch -> classify) and no element that
   ties it together.
3. No spatial analytics (zones / tripwires / dwell time) and no Mtd types for it.
4. No metadata publishing / serialization (JSON, MQTT, Kafka) - partially addressed by the
   community through ONVIF.
5. No 3D (lidar / radar / fusion) metadata - a gap, but low urgency for the community.
6. No metadata-driven routing, no overlays for tracking / keypoints / OBB, no GPU renderer.

Upstream also has no OpenVINO inference backend and no `VAMemory` / `DMABuf` import path
for inference elements. Closing this gap belongs to the uncertain Phase 3 only (see *Scope*
and section 6).

## 5. Collisions (full upstream unlikely, partial feature contributions expected)

For these elements an upstream equivalent already exists, so a full upstream would most
likely not be accepted - the community tends to reject duplicates. This is not a certainty
(it has not been confirmed with the maintainers), but the plan assumes it as the default. We
can still move parts of our functionality: the features the upstream equivalent lacks are
contributed to that equivalent instead (Phase 2), or deferred to the uncertain Phase 3.

| DLS | Upstream equivalent | What can still be contributed |
|---|---|---|
| `gvastreammux` / `gvastreamdemux` | `analyticscombiner` / `analyticssplitter` | passthrough mode, synchronization modes, bounded per-pad queues, heterogeneous sources (Phase 2) |
| `gvadetect` / `gvaclassify` / `gvainference` | `onnxinference` / `tfliteinference` / `executorchinference` + `tensordecoders` | nothing before Phase 3; a native `openvinoinference` or an OpenVINO execution provider for `onnxinference`, if ever |
| `gvawatermark` | `objectdetectionoverlay` / `segmentationoverlay` | tracking IDs, keypoints/skeletons, OBB, zones/tripwires, labels; GPU (VA + OpenCL) renderer (Phase 2) |
| `gvatrack` | `ioutracker` | nothing before Phase 3; VAS / deep-sort algorithms as additional tracker elements, if ever |
| `gvaaudiotranscribe` | `whispertranscriber` | nothing before Phase 3; unlikely |
| `gvafpscounter`, `latency_tracer`, `gvafpsthrottle`, `gvapython` | core elements / tracers / gst-python | nothing (not scheduled) |
| `GstGVAJSONMeta` | no equivalent; the upstream idiom is `text/x-raw` caps, not a meta | replaced by the `text/x-raw` / `application/json` output of the Phase 1 serializer |

## 6. Phased upstreaming plan

Elements marked *partial* already have an upstream equivalent, so a full upstream would most
likely be rejected as a duplicate (not certain); for them the planned deliverable is a set of
feature contributions to the upstream equivalent rather than a new element.

**Phase 1 - generic metadata elements (no dependency on the DLS inference stack):**

- `gvaanalytics` together with its new metadata types (`ZoneMtd`, `TripwireMtd`,
  `DwellTimeMtd`) proposed to `gst-libs/gst/analytics`. The API proposal must be opened
  first (maintainer consensus, `Since: 1.32` at the earliest); the element follows in the
  same cycle.
- `gvametaconvert` and `gvametapublish` (file / MQTT / Kafka), redesigned into the upstream
  idiom: a serializer with caps-typed `text/x-raw` / `application/json` output instead of
  `GstGVAJSONMeta`, and publishers as sinks (`mqttsink`, `kafkasink`).
- `gvaattachroi` (emitting `ODMtd`).
- `gvametaaggregate`.
- `gvaframeselect` and `gvaroiscale` (output as `BatchMeta` + `TensorProvenanceMtd`).
- `gvaeventrecorder` (most likely reduced to an analytics-driven valve feeding
  `togglerecord`).
- `gvatagger`.

**Phase 2 - visualization, batching, motion (mostly partial contributions):**

- `gvawatermark` - *partial*: extend `objectdetectionoverlay` / `segmentationoverlay`
  (tracking IDs, keypoints/skeletons, OBB, zones/tripwires, labels) and contribute a GPU
  (VA + OpenCL) renderer; `Watermark*Meta` mapped to `GstVideoOverlayCompositionMeta`.
- 3D metadata types (`GstAnalytics3DODMtd`, `GstAnalyticsCamera3DODMtd`) proposed to
  `gst-libs` together with `gvawatermark3d`; `gvamono3d` itself is deferred to Phase 3.
- `gvastreammux` / `gvastreamdemux` - *partial*: contribute passthrough mode, the
  synchronization modes, bounded per-pad queues and heterogeneous-source support to
  `analyticscombiner` / `analyticssplitter`; a separate element only if the community
  accepts the differentiators as a distinct use case.
- `gvamotiondetect` (or a patch making `motioncells` emit `ODMtd`).

**Phase 3 - Intel-specific inference stack (potential only; may never happen):**

- `gvadetect`, `gvaclassify`, `gvainference`, `gvamono3d`, `gvagenai`,
  `gvaaudiotranscribe` and `gvatrack`.
- These elements are built on OpenVINO and the DLS `inference_backend` (VA / OpenCL / D3D
  preprocessing, `model_info` in OpenVINO IR); `gvatrack` additionally on Intel's VAS tracker
  code and on features pre-extracted by `gvainference`. All of them collide with existing
  upstream equivalents (`*inference` -> `GstTensorMeta` -> tensor decoders,
  `whispertranscriber`, `ioutracker`), so a full upstream would most likely be rejected as a
  duplicate; at most parts could be contributed (a native `openvinoinference` emitting
  `GstTensorMeta`, an OpenVINO execution provider for `onnxinference`, the VAS / deep-sort
  algorithms as additional tracker elements) - in every case a rewrite in the upstream style
  rather than a port. This phase is a placeholder; it is possible that these elements are
  never upstreamed.

**Not scheduled:** `gvafpscounter`, `gvafpsthrottle`, `latency_tracer`, `gvapython`,
`gvaaudiodetect` (legacy), `gvadeskew`, `gvarealsense` and the lidar/radar elements
(`g3dlidarsrc`, `g3dlidarparse`, `g3dinference`, `g3drender`, `g3dobjectfuser`,
`g3dbevfusion`, `g3dradarprocess`); the latter could join Phase 3 if the 3D metadata from
Phase 2 is accepted.

## 7. Community process constraints

- DLS is MIT licensed; Intel, as copyright holder, can contribute under LGPL-2.1+.
- New plugins go to `gst-plugins-bad` (C, meson, hotdoc, unit tests, CI on Linux / Windows /
  macOS) or `gst-plugins-rs` (Rust).
- Changes to `gst-libs/gst/analytics` require consensus with the analytics maintainers
  (Collabora) and follow the yearly release cycle (`Since:` tags, API freeze).
- Elements registered with `GST_RANK_NONE` and the `gva` prefix would need renaming to
  upstream conventions.
