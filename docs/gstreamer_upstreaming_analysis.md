# DL Streamer vs. upstream GStreamer: element inventory and upstreaming analysis

Internal engineering analysis (not part of the published user guide).

Date: 2026-11-05

## Scope

This document answers the following task:

> Capture a list of elements that customers have expressed interest in, or that exist in
> pipeline implementations like DL Streamer (DLS) and DeepStream, that perform or contribute
> to AI (produce, transform, act on, or transmit GstAnalytics). Evaluate the promise of
> standardization for each element, including checking with the GStreamer community.
> Focus on: inference, batching, metadata ingestion, tracking, data analytics,
> visualization, messaging, branching/conditionals, decode/encode. Consider DeepStream and
> DL Streamer elements and their overlap.

It additionally answers: which DLS elements/libraries can be upstreamed to GStreamer, which
gaps GStreamer has relative to DLS, and which upstream elements would collide with ours.

## Sources and verification

| Source | Location | Notes |
|---|---|---|
| DL Streamer | this repository (`main`) | Element registrations verified in `src/monolithic/gst/registrator/register_elements.cpp`, `src/monolithic/gst/3d_registrator/`, and per-element `GST_PLUGIN_DEFINE`s. GstAnalytics API usage verified by grepping `gst_analytics_*` per element directory. |
| GStreamer monorepo | `/home/dlstreamer/gstreamer`, `main`, version `1.29.2.1` (development cycle leading to 1.30), HEAD 2026-11-05 | `Since:` versions taken from `subprojects/gst-plugins-bad/gst-libs/gst/analytics/*.h`; element names from `GST_ELEMENT_REGISTER_DEFINE`. |
| gst-plugins-rs | not checked out locally (git wrap only) | Element list confirmed from the official plugin documentation at gstreamer.freedesktop.org (`rsanalytics`, `rsonvif`, `llamacpp`, `whisper`, `originalbuffer`, ...). |
| NVIDIA DeepStream | not in either repository | Element names from general knowledge and from `docs/user-guide/dev_guide/converting_deepstream_to_dlstreamer.md`. Treat as indicative. |

Build baseline: DLS is built against GStreamer 1.28.8 (`SPECS/versions.env`); CMake minimum is 1.16.

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
detection metadata, metadata-driven routing, GPU-accelerated analytics overlay, high-level
Python API.

## 2. DL Streamer inventory (verified)

Plugins: `videoanalytics` (most elements), `3delements`, plus standalone plugins
`gvametapublish` (+ `gvametapublishmqtt`, `gvametapublishkafka`), `gvapython`,
`gvaattachroi`, `gvarealsense`, `gvamtcnn`, `latency_tracer`. License: MIT.

| Element | Base | GstAnalytics usage (verified) | Custom metadata |
|---|---|---|---|
| `gvadetect`, `gvaclassify`, `gvainference`, `gvamono3d` | `GvaBaseInference` (GstBaseTransform), OpenVINO only, preprocessing OpenCV / VA / OpenCL / D3D11 / D3D12 | produce `ODMtd` (incl. oriented), `ClsMtd`, `SegmentationMtd`, `KeypointMtd`, `Camera3DODMtd`; consume `ODMtd` for `inference-region=roi-list` | `GstGVATensorMeta` (raw tensors), `GstAnalyticsCamera3DODMtd` (DLS extension), keypoint descriptor registry |
| `gvatrack` | GstBaseTransform; tracker source code (VAS, MIT) lives in `src/monolithic/gst/elements/gvatrack/vas`, OpenCV based; types `zero-term`, `short-term-imageless`, `zero-term-imageless`, `deep-sort` | consume `ODMtd`, produce `TrackingMtd` (via `gva_utils`) | none required |
| `gvamotiondetect` | GstBaseTransform, OpenCV / VA / OpenCL | produce `ODMtd` | none |
| `gvastreammux` / `gvastreamdemux` | GstElement; caps `multistream/x-analytics-batch(meta:GstAnalyticsBatchMeta)` in `container` mode; `sync-mode` (none, first-pts, segment, pipeline, ntp); `output-mode` (passthrough, container); `max-queue-size` | `GstAnalyticsBatchMeta` | none |
| `gvametaaggregate` | GstAggregator | merges `RelationMeta` from several branches (OD, Cls, Tracking, Keypoint, Segmentation, Tensor, Group) | none |
| `gvaanalytics` | GstBaseTransform | consume `ODMtd`, `TrackingMtd`; produce DLS `ZoneMtd`, `TripwireMtd`, `DwellTimeMtd` | zone/tripwire/dwell-time Mtd types registered through the GstAnalytics API; `WatermarkDrawMeta` for rendering |
| `gvaattachroi` | GstBaseTransform | - | `GstVideoRegionOfInterestMeta` |
| `gvaframeselect`, `gvaroiscale` | GstBaseTransform | consume `ODMtd`/`ClsMtd`; `gvaroiscale` produces `ODMtd` with rescaled coordinates | none |
| `gvaeventrecorder` | GstBaseTransform | consume `ODMtd`, adds `ClsMtd` tags | none |
| `gvametaconvert` | GstBaseTransform; props `format`, `add-tensor-data`, `add-empty-results`, `source`, `tags`, `timestamp-utc`, `timestamp-microseconds`, `add-rtp-timestamp`, `signal-handoffs` | consume `RelationMeta` (OD, Cls, Tracking, 3D OD, Camera 3D OD, batch streams) | produces `GstGVAJSONMeta` |
| `gvametapublish`, `gvametapublishfile`, `gvametapublishmqtt` (paho), `gvametapublishkafka` (librdkafka) | GstBin + GstBaseTransform (pass-through) | - | consume `GstGVAJSONMeta` |
| `gvawatermark` (GstBin) / `gvawatermarkimpl`, `gvawatermark3d` | renderers: CPU (OpenCV), VA + OpenCL, D3D11, D3D12 | render `ODMtd` (incl. oriented), `ClsMtd`, `TrackingMtd`, `KeypointMtd`, `SegmentationMtd`, `ZoneMtd`, `TripwireMtd`, `Camera3DODMtd` | `WatermarkTextMeta`, `WatermarkCircleMeta`, `WatermarkDrawMeta` |
| `gvagenai` | GstBaseTransform; OpenVINO GenAI or HTTP (OpenAI-compatible) backend | consume `ODMtd`, produce `ClsMtd` | - |
| `gvaaudiotranscribe`, `gvaaudiodetect` | audio inference base | `ClsMtd` | `GstGVAAudioEventMeta` |
| `gvatagger` | GstBaseTransform (H.264/H.265 NAL analysis without decode) | - | trigger meta |
| `gvafpscounter`, `gvafpsthrottle`, `gvadeskew`, `gvapython`, `gvarealsense`, `latency_tracer` | various | `gvafpscounter` counts `ODMtd` | - |
| `g3dlidarsrc`, `g3dlidarparse`, `g3dinference`, `g3drender`, `g3dobjectfuser`, `g3dbevfusion`, `g3dradarprocess` | GstPushSrc / GstBaseTransform; caps `application/x-lidar`, batch container | produce/consume DLS `3DODMtd`, `TrackingMtd`, `BatchMeta` | `LidarMeta`, `RadarProcessMeta`, `GstAnalytics3DODMtd` (DLS extension) |
| `gvanms`, `gvabboxregression` (`gvamtcnn`) | GstBaseTransform | produce `ODMtd` | - |

Model configuration: DLS reads pre/post-processing parameters from the `model_info` section
of the OpenVINO IR `rt_info` (or legacy `model-proc` JSON); upstream uses `.modelinfo` ini
sidecar files (tensor ids, dims, type, scales/offsets, dims order).

## 3. Per-category matrix: DLS vs. upstream vs. DeepStream

Upstreaming promise: **High** = clear upstream gap, generic design, LGPL-compatible,
no API change; **Medium** = gap exists but requires redesign or new library API;
**Low** = collides with an existing upstream element or is too niche.

### 3.1 Inference

| DLS | Upstream | DeepStream | Assessment |
|---|---|---|---|
| `gvadetect` / `gvaclassify` / `gvainference` (monolithic: preprocessing + OpenVINO + postprocessing; `inference-region=roi-list`, `model-instance-id`, `batch-size`) | split model: `*inference` -> `GstTensorMeta` -> `*tensordec`; full-frame only; no OpenVINO; GPU memory only D3D12/HIP | `nvinfer` (config file; primary/secondary modes) | **Architectural collision** - monolithic elements will not be accepted. **High** for: (a) an `openvinoinference` element in `ext/openvino` emitting `GstTensorMeta` and reading `.modelinfo`; (b) a `VAMemory`/`DMABuf` importer for `onnxinference` (it already has an importer/exporter abstraction); (c) an OpenVINO execution provider in `onnxinference`. **High** for a converter between DLS `model_info` (rt_info) and `.modelinfo`. |
| DLS post-processors (YOLO, SSD, segmentation, keypoints, labels, custom `.so`) | tensor decoders cover YOLO (v8/obb/seg/26/X), SSD, classification, face, hands, PIPNet | post-processing inside `nvinfer` / `nvdspostprocess` | **Low** for duplicates; **Medium** for missing decoders (e.g. COCO pose keypoints now that `KeypointMtd` exists in 1.30). |
| `gvamono3d` (`Camera3DODMtd`), `g3dinference` (`3DODMtd`) | no 3D OD in the analytics library | `nvds_3d_*` | **Medium** - first propose `GstAnalytics3DODMtd` / `GstAnalyticsCamera3DODMtd` to `gst-libs` (ours already follow the GstAnalytics style), elements afterwards. |
| `gvagenai` (VLM; OpenVINO GenAI or HTTP backend) | `llamacpp-texttransform` (text only), `GstAnalyticsTextMtd` (1.30) | none (external services) | **Medium** - a video -> `TextMtd` VLM element is a gap; the OpenVINO GenAI dependency is debatable, the HTTP backend is easier to accept. |
| `gvaaudiotranscribe`, `gvaaudiodetect` | `whispertranscriber` | `nvdsasr` | **Low** - duplicate. |

### 3.2 Batching

| DLS | Upstream | DeepStream | Assessment |
|---|---|---|---|
| `gvastreammux` / `gvastreamdemux` (GstElement, `sync-mode`, `output-mode` passthrough/container, `max-queue-size`) | `analyticscombiner` / `analyticssplitter` - same caps, GstAggregator, `batch-duration`, `force-live` | `nvstreammux` / `nvstreamdemux` | **Collision** - do not upstream our elements. **High** for contributing features to the Rust elements: passthrough mode, synchronization modes, heterogeneous sources (video + lidar). In parallel, DLS inference elements should accept `multistream/x-analytics-batch`. |
| cross-stream batching via `model-instance-id` | no equivalent | `nvstreammux` | **Medium** - worth an RFC, needs consensus. |
| `gvametaaggregate` (merge `RelationMeta` from `tee` branches) | none | `nvdsmetamux` | **Medium** - generic "analytics meta merger"; upstream may prefer the combiner approach. |

### 3.3 Metadata ingestion

| DLS | Upstream | DeepStream | Assessment |
|---|---|---|---|
| `gvaattachroi` (manual ROI -> `GstVideoRegionOfInterestMeta`) | no element | ROI in `nvdspreprocess` | **High** - small and generic; should eventually emit `ODMtd`. |
| none (DLS has no MQTT subscriber, ONVIF-M or SEI ingestion) | ONVIF-M complete in rs (`onvifmeta2relationmeta`), SEI meta in parsers | `nvdsmetaextract` / `nvdsmetainsert` (SEI) | **DLS gap, not an upstream gap** - adopt upstream; shared candidate: SEI payload <-> `RelationMeta` element. |

### 3.4 Tracking

| DLS | Upstream | DeepStream | Assessment |
|---|---|---|---|
| `gvatrack` - zero-term, short-term-imageless, zero-term-imageless, deep-sort; tracker source is in-tree (MIT, OpenCV), no binary blob; produces `TrackingMtd` | `ioutracker` only; `cvtracker` is not analytics-aware | `nvtracker` (IOU / NvSORT / NvDCF / DeepSORT) | **High** - the largest and most generic gap. OpenCV is an accepted dependency (`ext/opencv`). Needs decoupling from `GstGVATensorMeta` (deep-sort features -> `TensorMtd`). |
| `gvamotiondetect` -> `ODMtd` | `motioncells` without analytics meta | - | **Medium** - either our element or a patch making `motioncells` emit `ODMtd`. |

### 3.5 Data analytics

| DLS | Upstream | DeepStream | Assessment |
|---|---|---|---|
| `gvaanalytics` (zones, tripwires, dwell time; DLS `ZoneMtd` / `TripwireMtd` / `DwellTimeMtd`) | none | `nvdsanalytics` | **High**, in two steps: propose the Mtd types to `gst-libs` first (API freeze, `Since: 1.32`), then the element. |
| `gvaframeselect`, `gvaroiscale` (ROI crop / letterbox for VLM) | none ("analyticsroicrop") | `nvdspreprocess` | **High** - closes the upstream "secondary inference" gap; the crop should produce a batch with `BatchMeta` / `TensorProvenanceMtd`. |
| `gvatagger` (NAL analysis without decode) | none | - | **Medium** - niche but generic. |
| `gvafpscounter`, `latency_tracer`, `gvafpsthrottle` | `fpsdisplaysink`, `latency` tracer, `videorate` / `clocksync` | - | **Low** - duplicates. |

### 3.6 Visualization

| DLS | Upstream | DeepStream | Assessment |
|---|---|---|---|
| `gvawatermark` (+impl, GstBin): OD/OBB, Cls, Tracking, Keypoint, Segmentation, Zone, Tripwire, custom `Watermark*Meta`; backends CPU / VA+OpenCL / D3D11 / D3D12 | `objectdetectionoverlay`, `segmentationoverlay` | `nvdsosd` | **Collision** - a duplicate will not be accepted. **High** for extending the upstream overlays (tracking IDs, keypoints/skeletons, OBB) and for a GPU renderer via VA/OpenCL. `Watermark*Meta` should map to `GstVideoOverlayCompositionMeta`. |
| `gvawatermark3d`, `g3drender` | none | `nvds_3d` | **Medium/Low** - depends on acceptance of 3D Mtd types. |

### 3.7 Messaging

| DLS | Upstream | DeepStream | Assessment |
|---|---|---|---|
| `gvametaconvert` (`RelationMeta` -> JSON in `GstGVAJSONMeta`) | no JSON; `relationmeta2onvifmeta` exists | `nvmsgconv` | **Medium** - a serializer is needed, but the DLS JSON schema is vendor-specific and the community leans towards ONVIF. Realistic: an `analyticsjsonserialize` element with `text/x-raw` / `application/json` on its src pad instead of a custom meta. |
| `gvametapublish{file,mqtt,kafka}` (pass-through transform) | no MQTT/Kafka | `nvmsgbroker` (sink) | **Medium** - upstream expects generic sinks (`mqttsink`, `kafkasink`) consuming buffers, not meta; requires redesign (idiom: `... ! jsonserializer ! mqttsink`). paho / librdkafka are acceptable as optional dependencies. |

### 3.8 Branching / conditionals

| DLS | Upstream | DeepStream | Assessment |
|---|---|---|---|
| `gvaeventrecorder` (pre/post frames after a class detection) | `togglerecord`, `gopbuffer`, `valve` - no metadata-driven control | - | **High** for a small generic `analyticsvalve` / `analyticsselector` (pass or route buffers by a predicate on `RelationMeta`); leave recording to `togglerecord`. |
| `gvapython` | `python` plugin loader in gst-python | `nvdsvideotemplate` | **Low** - duplicate concept. |

### 3.9 Decode / encode

DLS has no codecs of its own; it uses upstream `va*` / `qsv*` / `msdk` (see the mapping in
`docs/user-guide/dev_guide/converting_deepstream_to_dlstreamer.md`). The only candidate is
`gvarealsense` (upstream has only `openni2src`) - **Medium/Low**, niche; librealsense is
Apache-2.0.

## 4. Gaps in GStreamer relative to DL Streamer

1. No OpenVINO backend (neither an EP nor a native element) and no `VAMemory` / `DMABuf`
   import for inference - zero-copy on Intel GPU/NPU is our unique competence.
2. No trackers beyond IoU.
3. No secondary (per-ROI) inference flow (crop -> batch -> classify) and no element that
   ties it together.
4. No spatial analytics (zones / tripwires / dwell time) and no Mtd types for it.
5. No metadata publishing / serialization (JSON, MQTT, Kafka) - partially addressed by the
   community through ONVIF.
6. No 3D (lidar / radar / fusion) - a gap, but low urgency for the community.
7. No metadata-driven routing, no overlays for tracking / keypoints / OBB, no GPU renderer.
8. Weak high-level Python support (gst-python only exposes `GstAnalytics` introspection).

## 5. Collisions (do not upstream 1:1)

| DLS | Upstream equivalent |
|---|---|
| `gvastreammux` / `gvastreamdemux` | `analyticscombiner` / `analyticssplitter` |
| `gvadetect` / `gvaclassify` / `gvainference` | `onnxinference` / `tfliteinference` / `executorchinference` + `tensordecoders` |
| `gvawatermark` | `objectdetectionoverlay` / `segmentationoverlay` |
| `gvaaudiotranscribe` | `whispertranscriber` |
| `gvafpscounter`, `latency_tracer`, `gvafpsthrottle`, `gvapython` | core elements / tracers / gst-python |
| `GstGVATensorMeta` | `GstTensorMeta` / `GstAnalyticsTensorMtd` |
| `GstGVAJSONMeta` | no equivalent; the upstream idiom is `text/x-raw` caps, not a meta |

## 6. Recommended order

**Wave 1 (quick, generic, no API changes):** `gvatrack` (as part of `ext/opencv` or a new
plugin), `gvaattachroi`, ROI crop (`gvaroiscale`), `analyticsvalve` derived from
`gvaeventrecorder`, extensions to `objectdetectionoverlay` (tracking / keypoints / OBB),
`gvastreammux` features contributed to `analyticscombiner`.

**Wave 2 (new dependencies / elements):** `openvinoinference` plus a VA/DMABuf importer for
`onnxinference`, `model_info` <-> `.modelinfo` conversion, JSON serializer plus
`mqttsink` / `kafkasink`, VLM -> `TextMtd`.

**Wave 3 (gst-libs API changes, long cycle):** `ZoneMtd` / `TripwireMtd` / `DwellTimeMtd`,
`3DODMtd` / `Camera3DODMtd`, meta aggregation.

## 7. Community process constraints

- DLS is MIT licensed; Intel, as copyright holder, can contribute under LGPL-2.1+.
- New plugins go to `gst-plugins-bad` (C, meson, hotdoc, unit tests, CI on Linux / Windows /
  macOS) or `gst-plugins-rs` (Rust).
- Changes to `gst-libs/gst/analytics` require consensus with the analytics maintainers
  (Collabora) and follow the yearly release cycle (`Since:` tags, API freeze).
- The upstream `AGENTS.md` forbids AI agents from posting merge requests; humans must submit
  and drive them.
- Elements registered with `GST_RANK_NONE` and the `gva` prefix would need renaming to
  upstream conventions.
