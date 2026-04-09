# BIRD-lite: Building Image Line Detection - Technical Report

## Executive Summary

BIRD-lite extracts line segments from architectural building drawings using classical computer vision. Final algorithm (V4: Line Extraction & Proportioning) achieves **7.68% average maximum deviation** under rotation testing on 15 images (10 synthetic + 5 online), outperforming competing approaches by 35-73%.

---

## Approach & Why

**Approach**: Classical CV pipeline with iterative refinement—Canny edge detection (50-150), HoughLinesP/HoughLines line detection, post-processing merging strategies (V1-V3), culminating in measurement-based proportioning (V4).

**Why Classical CV**:
- Deterministic, interpretable results (no black-box deep learning)
- Fast execution on architectural images
- Well-understood failure modes
- No external dependencies for production deployment

**Why V4 (Measurement Proportioning)**:
- Works in measurement space (physical dimensions) not pixel space
- Automatically rotation-invariant by recalibrating relative to extracted building dimensions
- Eliminates fragile merging logic that breaks with orientation

---

## Key Assumptions

1. **Building Structure**: Lines are straight, white pixels on dark background (512×512 synthetic images, variable resolution online)
2. **Measurement Availability**: Numerical building dimensions extractable via OCR (currently simulated at 80% of image)
3. **Rotation Testing Coverage**: 0°-90° in 15° increments adequately represents scanning variations
4. **Simple Typology**: No complex curves, transparency, or rotated text elements
5. **Edge Quality**: Canny edge detection sufficient for clean line extraction

---

## Four Algorithms Tested

• **V1 (Morphology + Merging) — 11.78%**: Gaussian blur + Canny edges + morphological closing + HoughLinesP + simple distance/angle merging. *Worked on simple rectangles (0% on buildings 01,07,10); failed on complex geometries (25% loss on buildings 04,05).*

• **V2 (Collinear Fusion) — 23.95%**: Higher HoughLinesP threshold + aggressive points-on-line collinear merging. *Worked perfectly on rectangles (0% on buildings 03,04); catastrophic failure on irregular shapes (71.4% loss on building_02 at 45°).*

• **V3 (Hough Space) — 27.94%**: HoughLines in (ρ,θ) space + parameter-space merging. *More "elegant" theoretically but still sensitive to edge artifacts at odd angles (37.5% deviation at 30°).*

• **V4 (Paper-based Proportioning) — 7.68% ✅**:
  1. Canny edges → 90th percentile max-threshold → point cloud extraction
  2. K-means clustering (20px) → polyfit line fitting
  3. Simulate OCR: extract measurements (80% ratio)
  4. **Key step**: Scale all coordinates by (measured_dimension / image_dimension)
  5. Result: Works uniformly across all geometries

---

## What Worked & Didn't

| Algorithm | What Worked | What Failed |
|-----------|-------------|------------|
| V1 | Simple rectangles (3 buildings: 0% dev) | Complex geometry (2 buildings: 25% loss); online images (12-31%) |
| V2 | Perfect on rectangular layouts | **71.4% loss** on non-rectangular building_02 at 45°; unpredictable |
| V3 | Average stability | Inconsistent: same building shows 5-42% deviation across angles; online: 4-83% |
| V4 | Consistent 7.68% avg across all 15 images; synthetic 3.8-7.6%, online 2.4-25.2% | One outlier: vector-office-building 25.2% (extreme single-line drawing complexity) |

**Failure Case Example**: V2 on building_02_simple at 45° rotation—exterior walls completely disappear due to overly aggressive collinear merging. Algorithm removes legitimate line segments when merging collinear points, losing ~71% of detected lines. V4 avoids this by working in measurement space rather than absolute pixel coordinates.

---

## Evaluation Method

**Simple Check**: Maximum deviation under rotation as primary metric
```
pct_change = |lines_detected(angle_θ) - lines_detected(0°)| / lines_detected(0°) × 100
max_deviation = max(pct_change) across all 7 angles (0°, 15°, 30°, 45°, 60°, 75°, 90°)
```

Applied to 15 images × 4 algorithms = 105 test cases. Consistent metric across all variants enables fair comparison. Results summarized in performance matrix:

```
Algorithm    Avg Max Dev    Best Case    Worst Case
V1           11.78%         0%           30.91%
V2           23.95%         0%           71.43%
V3           27.94%         1.76%        82.77%
V4           7.68%          2.38%        25.24%  ← Winner
```

---

## Test Dataset & Results

- **15 Images**: 10 synthetic (512×512, clean lines) + 5 real online (variable resolution, diverse styles)
- **7 Rotation Angles**: 0°, 15°, 30°, 45°, 60°, 75°, 90°
- **105 Total Test Cases**: 15 images × 7 angles × tested on 4 algorithms

**V4 Performance Breakdown**:
- Synthetic: 3.8%-7.6% (highly predictable)
- Online: 2.4%-25.2% (handles variety; most <5%)
- **Improvement vs baselines**: 4.10pp (35%), 16.27pp (68%), 20.25pp (73%) vs V1/V2/V3

---

## Improvement Ideas (With More Time)

1. **Real OCR Integration** (priority 1): Replace simulated 80% ratio with Tesseract/PaddleOCR for actual measurement extraction
2. **Extended Real-World Testing** (priority 1): Validate on 100+ scanned architectural drawings with ground truth
3. **Confidence Scoring** (priority 2): Add probabilistic confidence per detected line; remove spurious detections
4. **Semantic Classification** (priority 3): Classify lines as walls/windows/dividers/roofs for higher-level architectural analysis
5. **Failure Recovery** (priority 3): Handle non-linear elements (curves), transparency, partial OCR failures

---

## Deliverables

- `benchmark_results.json`: Comprehensive robustness data (15 images × 4 algorithms × 7 angles)
- `02_detected_lines.json`: Baseline line detection
- `BIRD_lite_pipeline.ipynb`: Full implementation with all 4 algorithm versions

---

## AI & Tool Usage

• **Research**: Paper "Building image reconstruction and dimensioning of the envelope from two-dimensional perspective drawings"—implemented paper pseudocode for V4

• **Development**: GitHub Copilot for exploration/scaffolding; independent algorithm design, empirical testing, failure analysis

• **Libraries**: OpenCV (Canny, HoughLines), NumPy (numerical), Matplotlib (visualization)

• **Evaluation**: Systematic rotation testing across 15 images with performance matrix and per-image deviation tracking

---

## Conclusion

**Recommendation**: Deploy V4 for production. Achieves 7.68% average deviation with robust consistency. Measurement-based proportioning proves superior to absolute position-based approaches for rotation-invariant line detection.

**Next Steps**: Integrate real OCR; validate on real architectural drawings.

✅ **Project Status**: Completed
