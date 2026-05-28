# Body Measurement System

A prototype system that estimates body measurements from front and side view photos using computer vision and geometric modeling.

---

## Inputs

- Front view image (JPG/PNG)
- Side view image (JPG/PNG)
- User height in cm

## Outputs

- Chest circumference (cm)
- Waist circumference (cm)
- Hip circumference (cm)
- Arm length (cm)
- Confidence score (%)

---

## Pipeline

### 1. Silhouette Extraction
Background is removed using OpenCV thresholding and morphological operations (close + open kernels) to produce a clean binary body mask. Flood fill is applied to eliminate internal holes caused by dark clothing.

### 2. Landmark Detection
YOLOv8 pose model detects 17 COCO keypoints from the image. The following keypoints are used as anchors:
- Nose (0), Left/Right Shoulder (5, 6), Left/Right Hip (11, 12), Left/Right Wrist (9, 10)

Measurement Y-coordinates are derived from these anchors:
- **Neck** = midpoint between nose and shoulder
- **Chest** = 27% of the way from neck to hip
- **Waist** = 55% of the way from chest to hip
- **Hip** = YOLO hip keypoint directly
- **Arm top** = slightly above shoulder keypoint (8% correction)
- **Arm bottom** = wrist keypoint, capped above hip

> Note: YOLOv8 is used solely for keypoint localization. All measurement logic is a custom geometric algorithm.

### 3. Width Measurement
At each landmark Y-coordinate, the silhouette is scanned horizontally to measure the pixel width of the body. An empirical arm correction factor is applied to subtract arm overlap from torso width:
- Chest: 65% of silhouette width
- Waist: 58% of silhouette width
- Hip: 80% of silhouette width

### 4. Depth Estimation
The same horizontal scan is applied to the side view silhouette at the corresponding Y-coordinates to estimate body depth at each measurement point. This is the key insight of the two-view approach — front view gives width, side view gives depth, together they approximate a 3D cross-section.

### 5. Pixel to CM Scaling
A pixel-to-cm ratio is computed from the silhouette height and the user-provided height input:

```
px_to_cm = height_cm / silhouette_height_px
```

### 6. Circumference Estimation
Body cross-sections are modeled as ellipses with semi-axes:
- a = half-width (from front view)
- b = half-depth (from side view)

Circumference is estimated using the Ramanujan ellipse approximation:

```
C = π × [3(a+b) - √((3a+b)(a+3b))]
```

Error < 1% for typical body proportions.

### 7. Arm Length
Arm length is computed as the vertical pixel distance between the shoulder and wrist landmarks, converted to cm using the pixel-to-cm ratio.

### 8. Confidence Score
Confidence is estimated from internal pipeline signals (no ground truth required):
- Anatomical order validation (chest < hip, waist < chest)
- Known human proportion range checks for a given height
- Silhouette fill quality (how clean the body mask is)

---

## Assumptions

- Plain or white background
- Person standing upright, feet flat
- Fitted clothing preferred (loose clothing adds measurement error)
- Front and side photos taken at the same distance and camera height
- Arms hanging naturally at sides
- Single person per image

---

## Known Limitations & Edge Cases

### Accuracy Limitations
- Arm correction factors (arm_fraction) are empirically derived and tuned on a small set of images. Expected measurement error: ±5–10cm depending on body type and arm position.
- The dominant error source is arm overlap at the chest and waist level in the front silhouette — the horizontal scan picks up arm pixels alongside torso pixels.
- Hair adds to silhouette height, slightly compressing the pixel-to-cm ratio.
- The Ramanujan formula itself introduces < 1% error — it is not the bottleneck.

### Edge Cases
- **Loose/baggy clothing** — inflates all measurements significantly; system measures the garment, not the body
- **Arms not at sides** — arm overlap changes unpredictably, breaking the arm_fraction correction
- **Non-white or complex backgrounds** — OpenCV threshold-based silhouette extraction fails; requires a segmentation model
- **Extreme body types** — very lean or very large bodies fall outside the proportion range assumptions
- **Poor lighting or low contrast** — silhouette extraction produces noisy masks
- **Person not standing straight** — any tilt or pose variation misaligns front and side measurements

---

## 3D Estimation Ideas

The current prototype approximates 3D shape using two 2D views. A production-grade system would use true 3D reconstruction:

### SMPL + Human Mesh Recovery (HMR)
SMPL (Skinned Multi-Person Linear Model) is a parametric 3D body model that represents any human body as a combination of shape parameters (β) and pose parameters (θ). A regressor network (HMR, PARE, CLIFF) predicts these parameters from a single image, producing a full 3D mesh. Circumferences can then be extracted by slicing the mesh horizontally at any body level and measuring the cross-section perimeter — no arm overlap problem, no empirical correction factors.

### Depth Estimation
Monocular depth estimation models (e.g. Depth Anything V2) can produce a dense depth map from a single image. Combined with the silhouette, this gives approximate 3D volume per pixel, enabling more accurate cross-section estimation without a side view.

### Two-View Stereo Reconstruction
With calibrated front and side cameras at known positions, stereo reconstruction can produce a sparse 3D point cloud of the body surface. Measurement planes can then be intersected with this cloud directly.

### Why the current approach works as a prototype
Two orthogonal views (front + side) give width and depth at each measurement level. Modeling cross-sections as ellipses is a well-established approximation in anthropometry. The tradeoff is that clothing bulk and arm overlap introduce systematic error that 3D mesh methods eliminate entirely.

---

## Practicality & Scalability

### What works at scale
- YOLOv8 is fast (< 100ms per image), runs on CPU, and handles diverse images robustly
- The geometric pipeline is lightweight — no GPU required for measurement logic
- The system can be deployed as a REST API with minimal changes (replace Gradio with FastAPI)
- Works on standard smartphone photos — no special hardware needed

### What needs improvement for production
- **Background removal** — replace OpenCV thresholding with SAM or rembg for real-world backgrounds
- **arm_fraction values** — need to be learned per body type rather than hardcoded; requires a labeled dataset
- **Camera consistency** — front and side photos must be taken at the same distance; needs enforcement or calibration logic
- **Clothing handling** — current system measures garments, not bodies; SMPL-based approach solves this
- **Pose robustness** — person must stand straight; a pose normalization step would handle variation

---

## What I Would Improve With More Time

- **SMPL + HMR** — fit a 3D parametric body mesh to extract exact circumferences by slicing the mesh, eliminating arm correction entirely
- **Explicit arm segmentation** — detect and subtract arm pixels from torso width at each measurement level using a segmentation model
- **Dataset validation** — benchmark against ANSUR II anthropometric dataset for quantitative accuracy metrics
- **Camera calibration** — handle varying camera distances and angles for real-world mobile deployment
- **Error analysis against ground truth** — collect 20–30 samples with tape measurements, compute MAE and RMSE per measurement

---

## Tech Stack

- Python, NumPy, OpenCV
- YOLOv8 (Ultralytics) — keypoint detection only
- Gradio — UI

---

## How to Run

1. Open the notebook in Google Colab
2. Run all cells in order (Modules 1 through 6)
3. Launch the Gradio UI cell
4. Upload the given front view image, side view image, and enter height in cm
5. View estimated measurements and annotated output

---

## Submission

- `body_measurement.ipynb` — full Colab notebook
- `README.md` — this file
- `body_measurements.mp4` - Demo video
