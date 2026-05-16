# VisionX — Technical Specification

## Overview
VisionX: "VisionBoard Real-Time PCB Defect Detection Using YOLO-Based AI Vision" is an AI-native inspection platform to detect PCB defects (solder bridges, missing components, misalignment, etc.) in real time on the manufacturing line using a YOLO-family model and an integrated production dashboard.

## Objective
- Detect and classify common PCB defects with low latency so inspection does not bottleneck production.
- Surface actionable results to engineers via a live dashboard and automated alerts.
- Reduce defect escape rate, scrap, and rework costs by catching defects early.

## Scope
- Inline, real-time optical inspection at placement and post-reflow stages.
- Classes: solder bridge, missing component, wrong component, misaligned component, tombstoning, solder void, other (catch-all).
- Integration with line PLCs, MES, and operator dashboard; optional feedback loop to pick-and-place machines.

## Architecture
- Cameras (high-res industrial cameras) capture frames at trigger points.
- Edge inference nodes (GPU/accelerator) run optimized YOLO models for per-frame detection.
- Aggregation service collects detections, applies business rules, and streams events to Dashboard and MES.
- Storage: short-term frame buffer (for review) and long-term event DB for analytics.

Diagram (conceptual): Camera -> Edge Inference -> Aggregator/API -> Dashboard / MES / Storage

## Model Specification
- Base model: YOLOv8 (or YOLOX/Tiny YOLO variant for constrained hardware) pruned & quantized for latency.
- Input: RGB images; resolution chosen to balance SNR and speed (e.g., 640–1280 px width depending on optics).
- Output: Bounding boxes + class + confidence + optional segmentation mask.
- Training data: labeled images from multiple BOMs/variants, annotated per-class. Include synthetic augmentations for orientation, lighting, and minor PCB layout changes.
- Metrics: mAP@0.5, recall@target_FPR, per-class precision/recall, inference latency (ms/frame), and production false-positive rate.

## Data Pipeline
- Capture: global shutter cameras with consistent lighting and fiducial alignment.
- Preprocessing: geometric correction (deskew), color normalization, ROI cropping based on fiducials.
- Labeling: tool-assisted labeling workflow; classes and edge-case tagging (ambiguous, occluded).
- Dataset split: train/val/test with stratified class and BOM distribution.

## Inference & Performance
- Latency target: end-to-end detection < 50 ms per board (adjust to line speed); acceptable up to 200 ms for slower lines.
- Throughput: support per-line FPS > camera trigger rate; batch where appropriate but preserve real-time guarantees.
- Deployment modes: edge GPU (NVIDIA Jetson/RTX), accelerator (Intel Myriad/GPU), or server-hosted inference via low-latency link.
- Model optimization: quantization (FP16/INT8), pruning, TensorRT/ONNX Runtime, and NMS tuning.

## Integration
- Event model: detection events contain `board_id`, `timestamp`, `camera_id`, `bbox`, `class`, `confidence`, `frame_ref`.
- Actions: immediate discard/hold, operator alert with image & bounding boxes, schedule rework, or trigger automated pick-and-place correction.
- APIs: REST/gRPC for event ingestion, WebSocket for live dashboard streaming, and webhook for MES/PLC.

## Dashboard & UX
- Live stream with overlays, per-board timeline, defect queue, and operator review workflow.
- Filters: by line, BOM, class, confidence threshold, time range.
- Audit: link each detection to stored frame/short video clip for manual review and RCA.

## Reliability, Safety & Fallbacks
- Fallback to conservative rule-based AOI if inference node fails (to avoid stopping line).
- Graceful degradation: reduce frame rate or move to server inference if edge unavailable.
- Health checks: heartbeat from edge nodes, automated restart on failure, and alerting.

## Evaluation & Acceptance Criteria
- Functional: per-class recall >= 95% on validation set for critical defects; precision tuned to keep false positives manageable for operators.
- Performance: median inference latency <= X ms (define per-line target), system uptime >= 99.5% during production runs.
- Business: reduction in scrap/rework by target % within pilot period (define metric with stakeholders).

## Deployment Plan
1. Pilot: single line, controlled BOM set, collect labeled data and iterate model.
2. Expand: add BOMs, optimize model size per hardware profile, integrate with MES for automated routing.
3. Rollout: multi-line deployment, monitoring & retraining pipeline in place.

## Operational Requirements
- Hardware: industrial cameras, synchronized lighting, edge inference devices (GPU or accelerator), network with <5 ms RTT to aggregator.
- Software: containerized inference (Docker), model registry, CI for model training, monitoring/alerting stack (Prometheus/Grafana), Dashboard web app.

## Privacy & Security
- Secure camera streams and API endpoints (TLS). Access control for dashboard and stored frames. Retention policy for imagery per factory rules.

## Roadmap & Next Steps
- Collect initial dataset (2k–10k labeled frames across BOMs).
- Train YOLO prototype, run offline evaluation, and deploy edge pilot.
- Implement dashboard MVP with operator review and event logging.

## Appendix: Minimal Label Schema
- `class`: solder_bridge | missing_component | wrong_component | misaligned | tombstone | solder_void | other
- `severity`: low | medium | high
- `notes`: free text for operator remarks

---
File created from VisionX team brief; adapt thresholds and targets to actual line speed and BOM complexity during pilot.
