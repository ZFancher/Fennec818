# Mobile Camera-Based Soil Color Detection in ADS Plus
## A Technical White Paper

**Audience:** Wetland delineators and field soil scientists familiar with Munsell notation, hydric soil indicators, and redox morphology — but not necessarily with computer vision or digital image processing.

---

## 1. Background: The Color Measurement Problem

Accurate soil color is foundational to hydric soil identification. The Munsell Soil Color Chart provides a standardized vocabulary — hue, value, and chroma — that a trained delineator can read directly off a moist ped. The system works because it encodes perceptually uniform steps: a jump from 10YR 4/2 to 10YR 4/4 is roughly the same perceptual distance as a jump from 7.5YR 4/4 to 5YR 4/4.

What makes replicating this digitally difficult is that a phone camera does not "see" color the way a human eye does. A camera sensor captures raw photon counts across three broad red, green, and blue channels, then applies a series of proprietary signal processing steps — noise reduction, sharpening, tone mapping, automatic white balance — before producing the RGB values you see in a photo. These processing steps are tuned for pleasing photographs, not colorimetric accuracy. The same soil ped photographed in full sun, open shade, and indoor fluorescent light will produce dramatically different RGB values, even though a delineator's eye and a Munsell chart would read the same result each time.

Both camera functions in ADS Plus address this problem with the same foundational approach — a physical reference card that provides known-color ground truth — before diverging in what they do with the calibrated color data.

---

## 2. The Color Calibration Foundation: Reference Card and Color Correction Matrix

### 2.1 Why a Reference Card?

The most robust portable approach to correcting a camera's color response is to include in the photograph an object whose colors are precisely known. By comparing what the camera *says* those colors are versus what they *actually are*, you can mathematically describe the camera's error and invert it. In the food science, textile, and medical imaging fields, this has been standard practice for decades; the Munsell Book itself is used as a reference target in some archival color research.

ADS Plus uses a dedicated reference card containing a specific pattern of color patches whose Lab values (see Section 3) are known in advance. Every time the camera functions run, they scan the video frame for this card, confirm its detection, and use it to compute a correction.

### 2.2 The Color Correction Matrix (CCM)

The correction is a 3×3 matrix — called a Color Correction Matrix or CCM — that transforms the camera's raw RGB values into a corrected RGB that is much closer to the true, illuminant-independent color. Think of it as a set of linear mixing coefficients: "to get true red from this camera, take 1.2 times the camera's red, subtract 0.15 times its green, and add 0.05 times its blue."

The CCM is solved fresh for each new card detection using least-squares regression against the known patch colors. This means the correction automatically adapts to whatever lighting is present at that moment. Direct midday sun, overcast sky, and north-facing shade all have different spectral power distributions — effectively different "colors of white" — and a new CCM is computed for each condition without the user doing anything. This is the key advantage over simple auto-white-balance: the CCM corrects not just the white point but the entire spectral transfer function of the scene.

The limitation is linearity. The CCM assumes a linear relationship between the camera's RGB and true color, which is a good approximation in midrange tones but breaks down for highly saturated colors or in very dark shadows where sensor noise dominates. For the olive browns, grays, and moderate-chroma oranges of typical soil peds, the linear approximation is more than adequate.

---

## 3. Color Space: Why CIE L\*a\*b\*?

Both functions convert calibrated sRGB values to **CIE L\*a\*b\*** (referred to simply as "Lab") for all further computation. Understanding why requires a brief detour into color geometry.

The RGB color cube is not perceptually uniform. A change of 10 units in the blue channel does not look the same to a human as a change of 10 units in the green channel. This means Euclidean distance in RGB space is a poor measure of how different two colors *appear* — which matters critically for both finding the nearest Munsell chip and for clustering soil pixels.

Lab is designed to be perceptually uniform: equal Euclidean distances in Lab space correspond (approximately) to equal perceived color differences. The **L** axis runs from black (0) to white (100), corresponding closely to Munsell value. The **a** axis runs from green to red, and the **b** axis runs from blue to yellow. Crucially, **chroma** — the Munsell concept of color saturation — maps directly to `C* = √(a² + b²)`: distance from the neutral axis. A perfectly gray depletion zone sits near the origin; a bright orange iron concentration sits far from it.

The conversion from sRGB to Lab involves three steps executed in strict sequence: gamma delinearization is reversed (sRGB stores luminance on a nonlinear curve optimized for display, which must be undone), the resulting linear RGB is multiplied through a standard 3×3 matrix to reach the CIE XYZ tristimulus space, and XYZ is then converted to Lab via a cube-root compression that mimics the perceptual response of the human visual system. These are well-established ICC-standard transformations, not heuristics.

---

## 4. Function 1: Single-Point Munsell Matcher

### 4.1 Pipeline Summary

The first function is designed for rapid, targeted color readings — the digital equivalent of pressing a Munsell chip against a specific spot on a ped. Its pipeline is:

1. Live video runs continuously. The reference card is detected at approximately 8 frames per second in the background, maintaining a running best-known CCM.
2. The user taps any point on the live frame. On each tap, a fresh card detection is run against a simultaneously captured frame to compute a CCM from that exact moment; if the card cannot be detected in that frame, the most recent successful CCM is used as a fallback.
3. A circular region around the tap point is sampled at native camera resolution. The radius of this circle is derived from the card's reference patch geometry — it scales automatically with the card's apparent size in the frame, so the sample area remains proportionally consistent regardless of how close the camera is held.
4. Each pixel in that region has the CCM applied, is converted to Lab, and is averaged to a single Lab triplet.
5. That Lab triplet is compared against every chip in the Munsell Lab table (approximately 430 chips spanning hues 5R through 2.5Y, covering the full range of mineral soils) using **CIE76 ΔE** distance — the straight-line Euclidean distance in Lab space.
6. The chip with the lowest ΔE is returned as the Munsell match, displayed immediately in a persistent result panel at the bottom of the screen alongside a rendered color swatch and the ΔE confidence value.
7. The function remains live after each tap. The user can tap a different spot at any time to update the result, allowing rapid comparison across multiple locations on the same ped face without resetting or restarting.

### 4.2 CIE76 ΔE and Nearest-Neighbor Lookup

ΔE (delta-E) is the standard unit of perceptual color difference. A ΔE of 1 is considered the threshold of just-noticeable difference under ideal viewing conditions; a ΔE under about 3–4 is generally imperceptible to a casual observer. The Munsell system itself has about a ΔE of 2–5 between adjacent chips in most of the gamut.

The brute-force nearest-neighbor search against all 430 chips is computationally trivial on a modern phone (a few milliseconds), and importantly it is *exact* — no indexing or approximation errors can cause a wrong chip to be selected. More sophisticated distance metrics like CIEDE2000 (which corrects for known perceptual non-uniformities in the blue region and for low-chroma colors) exist, but for the warm-to-neutral brown/gray palette of mineral soils, CIE76 performs essentially identically and is far simpler to implement correctly.

### 4.3 Why This Approach

The single-point approach mirrors how a delineator uses a physical Munsell book: you identify a representative spot, hold a chip next to it, and judge the match. The continuous-sampling UX extends this by letting the delineator tap across several spots in quick succession — useful for confirming a dominant matrix color, checking whether a candidate redox feature is distinctly different from background, or verifying that a color reading is stable across the ped face. Each tap takes a fraction of a second and updates the result panel in place.

The drawback is that a single-point sample is inherently sensitive to positioning: a tap on a slightly different part of the ped can yield a different Munsell chip. The circular averaging region mitigates this, but it cannot fully account for a heterogeneous ped face that has interleaved matrix and redox areas at small scale. The continuously available sampling partially compensates for this — by tapping several representative spots and observing whether the result is stable, the delineator gains informal confidence in the reading before committing it to the form.

---

## 5. Function 2: Ped Analysis with Color Distribution

### 5.1 Pipeline Summary

The second function is designed for characterizing the *distribution* of colors across a ped — identifying not just the matrix but the concentration and depletion zones simultaneously, along with their relative area percentages. Its pipeline is substantially more complex:

1. Same reference card detection and CCM computation as Function 1.
2. The user positions a draggable four-corner polygon over the soil ped face.
3. On trigger, a full-resolution frame is captured.
4. Every other pixel inside the polygon bounding box is tested for polygon membership (ray-casting algorithm) and, if inside, has the CCM applied and is converted to Lab. This subsampling reduces processing time without meaningfully affecting accuracy.
5. The resulting set of Lab color samples (potentially tens of thousands) is fed into **k-means++ clustering** with K=6 clusters.
6. Each of the 6 cluster centroids is matched to its nearest Munsell chip via the same ΔE lookup used in Function 1.
7. Clusters that resolve to the same Munsell chip are **merged**, with their pixel counts summed. This eliminates phantom duplicates that arise when two k-means clusters happen to occupy the same Munsell "neighborhood."
8. The merged cluster with the highest pixel count is designated the **matrix color** — definitionally correct, and consistent with USACE field methodology, which recognizes only one matrix color per horizon.
9. Remaining merged clusters are evaluated for minimum contrast from the matrix. Any cluster differing from the matrix by less than 1 Munsell value unit, less than 1 chroma unit, AND zero hue steps is discarded as indistinguishable matrix noise before classification proceeds.
10. Surviving clusters are classified relative to the matrix:
   - **Concentrations** must have higher Lab chroma than the matrix AND meet a minimum contrast level (Faint, Distinct, or Prominent) as defined by the USDA Hydric Soils Table A1 algorithm. The concentration slider controls which minimum level qualifies.
   - **Depletions** are identified by absolute low Munsell chroma relative to the depletion slider threshold, reflecting the achromatic character of iron-depleted zones.
   - Anything that is neither sufficiently more chromatic than the matrix (concentration) nor sufficiently achromatic (depletion) is left unclassified and not reported.
11. Percentages for the matrix and each feature are computed independently as the raw pixel share of the sampled area, rounded to the nearest integer. Features at or below 1% are discarded as noise — no USACE hydric soil indicator requires a redox feature present at less than 2%.
12. The remaining features are ranked by percentage area regardless of type (concentration or depletion). The top 5 by area are reported. This type-agnostic ranking ensures a high-percentage depletion is never dropped to accommodate a low-percentage concentration, or vice versa.
13. An overlay is rendered showing concentration pixels in neon green and depletion pixels in electric blue; uncolored areas are the matrix.

### 5.2 K-Means++ Clustering

K-means is one of the oldest and most widely deployed algorithms in machine learning. Given a set of points in space and a number K, it partitions the points into K groups (clusters) such that each point belongs to the cluster whose center it is nearest to, and the cluster centers (centroids) minimize total within-cluster distance. In the context of soil color analysis, each "point" is a pixel's Lab value, and the algorithm finds the K color "centers" that best characterize the color population of the ped.

The "++" variant improves on the classic k-means initialization by selecting starting centroids probabilistically — each new centroid is chosen with probability proportional to its distance from already-chosen centroids, spreading the seeds across the color space. This dramatically reduces the chance of the algorithm converging to a poor local minimum (a known weakness of random initialization) without adding significant computation.

K=6 is chosen as a pragmatic balance. More clusters capture finer color variations but increase the chance of splitting what is really one color zone into multiple similar-looking clusters (which is why the Munsell merging step exists). Fewer clusters risk merging what a field scientist would recognize as distinct features. In practice, for a typical moist soil ped with one matrix, one or two concentration zones, and possibly a depletion, 6 clusters provides more than sufficient resolution, with the merging step collapsing the redundant ones.

### 5.3 The USDA Table A1 Contrast Classification

This is where the science of the algorithm makes its most direct contact with hydric soils methodology. The USDA's *Field Indicators of Hydric Soils* and its regional supplements define redox concentrations partly by their **contrast** with the surrounding matrix — Faint, Distinct, or Prominent. That contrast is determined by a lookup table combining hue step difference (the number of hue intervals between matrix and feature on the Munsell hue circle), value difference, and chroma difference.

Both sliders operate as **continuous visualization thresholds** rather than discrete preset sensitivity levels. As the user drags either slider, the overlay updates in real time — clusters progressively appear or disappear from the concentration (neon green) or depletion (electric blue) highlighting as they cross the current threshold. The intent is exploratory: the delineator sees the full color distribution of the ped and uses the sliders to examine where the meaningful boundaries lie before recording a result.

The **concentration slider** sweeps continuously from left (Prominent only) to right (Faint+). The underlying classification still uses the USDA Table A1 contrast categories — Prominent, Distinct, and Faint — as its breakpoints. As the slider crosses each breakpoint, additional clusters cross the qualification threshold and appear in the overlay. Sliding right asks the algorithm "show me everything that qualifies as a concentration even under lenient criteria"; sliding left asks "show me only the features you're highly confident about."

The **depletion slider** sweeps a continuous Munsell chroma threshold from 0 to 4. Any cluster whose Munsell chroma is at or below the threshold qualifies as a depletion. Because Munsell chroma values are integers, the effective breakpoints occur at each integer crossing as the slider moves; the overlay updates at those transitions. ≤1 captures strongly reduced, unambiguous gray zones; ≤2 is the most common USDA hydric depletion criterion; ≤3–4 extends to weakly expressed or marginally depleted zones.

### 5.4 Percentage Reporting

Each displayed percentage is the fraction of pixels inside the polygon that belong to that Munsell cluster, rounded independently to the nearest integer. Features at or below 1% are suppressed — no USACE hydric soil indicator requires a redox feature present at less than 2%, and sub-1% clusters are indistinguishable from k-means noise at this scale. Because only the dominant cluster is called the matrix and unclassified clusters are not reported, the displayed values will typically sum to less than 100. This is intentional: each number is a direct, honest statement about how much of the sampled area that specific Munsell notation covers. Reporting "Matrix: 10YR 4/2 (58%), Conc. 1: 7.5YR 4/6 (22%), Depl. 1: 10YR 5/1 (9%)" accurately reflects the measured color distribution without attributing pixels of unknown or ambiguous classification to any named color.

---

## 6. Comparative Analysis

| | **Single-Point Matcher** | **Ped Analysis** |
|---|---|---|
| **Primary use** | Recording a matrix or feature color | Characterizing full color distribution of a ped |
| **Output** | Single Munsell HVC notation | Matrix color(s), concentrations, depletions with % area |
| **User input** | Single tap | Polygon placement |
| **Processing time** | Near-instant | 1–3 seconds |
| **Color calibration** | CCM from reference card | CCM from reference card (same) |
| **Color space** | Lab (for nearest-chip match) | Lab (for clustering and chip match) |
| **Classification basis** | Nearest ΔE | K-means + USDA contrast table |
| **Redox detection** | None | Concentration and depletion zones |
| **Best field application** | Uniform matrix, specific features | Heterogeneous ped with visible redox |

---

## 7. Known Limitations of Both Functions

**Reference card dependency.** Both functions require the reference card to be present in the frame and detectable. If the card is absent, out of focus, partially occluded, or in a very different light from the soil, the CCM cannot be computed and color matching falls back to uncorrected values — or does not proceed at all. The card must be held in roughly the same plane and light as the soil face.

**Linearity assumption in the CCM.** The CCM is a linear transform. Camera sensors and their onboard processing are not fully linear, particularly at high and low luminance extremes. Under harsh direct sunlight with deep shadows in the same frame, or when photographing very dark (value 2) organic material alongside lighter mineral matrix, the correction accuracy decreases.

**Munsell chip coverage.** The app's Munsell Lab table covers hues from 5R through 2.5Y — the full range of common mineral soils. However, Gley hues (Greenish Gray, Bluish Gray, etc.), which are common in strongly reduced, permanently saturated horizons, are not presently included. Soils in these hues will be matched to the nearest available chip in the 5R–2.5Y range, which may not be the correct notation.

**K-means stochasticity.** K-means (even with ++ initialization) is not deterministic. Two analyses of the same photograph may produce slightly different cluster assignments and therefore slightly different percentages or, rarely, different Munsell chip assignments for minor clusters. For the dominant matrix color and prominent concentration zones this is inconsequential; for small, low-contrast features it can introduce run-to-run variability.

**Camera focus.** Neither function programmatically controls focus. The app requests the rear camera with a resolution hint but applies no focus constraints; focus behavior is entirely determined by the phone's native camera system. On iOS, the `getUserMedia` browser stream uses continuous autofocus managed by the OS — tap-to-focus is a native Camera app behavior that is not exposed to browser streams. In practice this is largely benign: iPhone cameras have small sensors that produce deep depth of field at typical working distances (20–40 cm), meaning the reference card and soil face are both sharp simultaneously without any explicit focus action. On Android, behavior varies by device and browser, but continuous autofocus at these distances is also generally reliable. The more practical focus risks across all platforms are motion blur from a shaky hold, autofocus hunting in low light, and very close macro distances where depth of field narrows. To detect blur regardless of its cause, both functions compute a **Laplacian variance** sharpness score over the reference card patch regions after each capture. If the score falls below a threshold, a non-blocking warning is displayed prompting the user to hold the camera steady and retrigger — the result is still shown, but the user is advised that calibration accuracy may be reduced.

**Sample area quality.** Both functions analyze what the camera sees through the polygon or tap region. Shadow, specular reflection, water film on the ped face, root channels, organic staining, and coatings all affect the measured color independently of the true soil matrix color. The standard field advice — moist, freshly broken ped face, photographed in consistent shade without direct sun glare — remains essential for accurate results regardless of the algorithm used.

**Smartphone sensor variation.** Different phone models process color differently before the app ever receives the image data. While the CCM corrects for much of this, phones with aggressive computational photography (HDR blending, night-mode processing, deep color enhancement) may produce source images that are harder to calibrate accurately than phones with more conservative image pipelines.

---

## 8. Relationship to the Broader Science

Academic research in digital soil color estimation has generally followed two paths. The first — direct colorimetry with handheld spectrophotometers — achieves the highest accuracy but requires expensive dedicated hardware, is slow, and produces point measurements rather than spatial distributions. The second path uses smartphone cameras or flatbed scanners with color targets, solving the calibration problem as ADS Plus does, and has been shown in the literature (e.g., Stiglitz et al., 2017; Gomez et al., 2008) to achieve Munsell notation accuracy within one value and one chroma step for the majority of mineral soil samples under controlled conditions — which is equivalent to, or better than, inter-observer variability among trained delineators using physical Munsell books.

More recent work has explored neural network approaches that embed the calibration into a learned model, eliminating the reference card entirely. These are promising but currently require training datasets that are not yet publicly available for the specific soil types and lighting conditions encountered in western arid-region field work, and their failure modes (when the scene is unlike the training data) are less predictable than the failure modes of the CCM approach, which degrades gracefully.

The k-means approach to soil color distribution characterization is consistent with broader image segmentation literature for agricultural and geological applications. Its advantage over threshold-based segmentation (e.g., "flag every pixel above chroma 4") is that it discovers the natural color clusters in the actual data rather than assuming fixed boundaries, which is important for soils whose matrix chroma spans a wide range across LRRs and parent material types.

---

## 9. Practical Guidance for Field Use

- **Reference card placement:** Hold the card flat in the same light as the soil, parallel to the camera. Avoid tilting it toward or away from a light source, which creates specular gradients across the patches.
- **Ped preparation:** Use a freshly broken, moist (not wet or dry) ped face. Wipe off any water film. Use natural daylight (open shade is ideal) rather than direct sun or artificial light.
- **Polygon placement:** Frame the polygon tightly around the representative part of the ped face, excluding obvious coatings, roots, or edges where crumbling has altered the morphology.
- **Slider use:** After triggering analysis, drag each slider and watch the overlay respond in real time. Drag the concentration slider right until the overlay highlights what you would call concentration zones by eye — then stop. Drag the depletion slider right until clearly depleted zones are captured without pulling in colors you would call matrix. The sliders are exploratory tools; the recorded result reflects your final slider positions.
- **Focus:** Confirm the live image looks sharp before tapping or triggering analysis. The app will display a warning if the reference card appears blurry — if this occurs, hold the camera steady, wait for the image to stabilize, and retap or retry. Avoid sampling immediately after a rapid camera movement. In low light, allow an extra moment for autofocus to settle before triggering.
- **Verification:** Both functions produce Munsell notations that should be compared against your physical Munsell book for the first few uses in a new soil series. If there is systematic disagreement (e.g., the app consistently calls 7.5YR 4/4 as 10YR 4/4), check card placement and lighting before assuming a sensor problem.

---

*ADS Plus — Internal Technical Documentation*
*Prepared for project reference; not for citation as peer-reviewed literature.*
