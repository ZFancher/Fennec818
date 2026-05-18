# ADS Plus: A Next-Generation Wetland Delineation Support Tool for the Arid West

**Version 15 — May 2026**

---

## Executive Summary

ADS Plus is a mobile-first, browser-based decision support application designed to assist wetland delineators working in the Arid West region of the United States (USACE Arid West Supplement, covering Land Resource Regions B, C, and D). It implements the full three-factor wetland delineation methodology — hydrophytic vegetation, hydric soils, and wetland hydrology — in a single self-contained HTML file that runs on any modern device, with or without an internet connection.

ADS Plus was built in direct response to the practical limitations of the two tools currently in use by USACE regulatory staff — the official Arid West Data Sheet (ADS), an Excel-based spreadsheet, and the Corps' Survey 123 field collection form. While the ADS provides the authoritative reference logic and ultimately produces the official ENG Form 6116-1 output, it requires a computer, has no field-optimized data entry, provides limited guidance to the user during data entry, and must be used in combination with Survey 123 and the National Regulatory Viewer (NRV) through a multi-step workflow that introduces risk of data inconsistency. ADS Plus integrates all of these steps into a single, field-ready application.

---

## Background: The Current Toolkit and Its Limitations

### The Official Arid West Data Sheet (ADS)

The ADS is a Microsoft Excel workbook developed by USACE for use in the Arid West region. It implements the logic from the 1987 USACE Wetlands Delineation Manual, the Arid West Regional Supplement, and the NRCS *Field Indicators of Hydric Soils in the United States* (current version: 9.3). The ADS accepts soil horizon data, hydrology indicator selections, and vegetation data, then calculates a wetland determination.

**Limitations of the ADS alone:**
- Requires a laptop or desktop computer; not practical for direct field use
- Requires Microsoft Excel
- Provides no field photography, camera-assisted soil color reading, or site documentation
- Performs auto-inference internally but does not explain *why* an indicator fired or why it did not — only the final "X" (auto-selected) or blank result is visible; intermediate reasoning is not surfaced
- Does not calculate the effect of Morphological Adaptations (Indicator 3) on the Dominance Test or Prevalence Index; that adjustment must be done manually
- Does not enforce data consistency — a user can manually select any indicator regardless of whether the entered data supports it, with no warning
- Certain depth thresholds in the ADS formulas have not been updated to reflect the most recent NRCS Field Indicators guidance (see Technical Accuracy section below)
- Produces no in-app or exportable PDF; the filled ENG Form 6116-1 is produced only through the NRV workflow

### The Corps' Survey 123 Field Collection Form

Survey 123 is an Esri-based mobile data collection application used by USACE regulatory staff to gather field data for wetland delineations. It provides an optimized mobile interface with all the basic data entry fields from the ADS.

**What Survey 123 does well:**
- Fast, mobile-optimized data entry in the field
- Connected to the internet and uploads directly to the Corps' National Regulatory Viewer (NRV)
- Familiar interface for staff already using Esri products
- All selections are possible within a single form

**What Survey 123 does not do:**
- No auto-inference behavior of any kind — all indicator selections are purely manual
- No data consistency checks — users can select any indicator regardless of supporting data, with no warning or notification
- No disallowing of selections — a user can simultaneously check "yes" and "no" for hydrophytic vegetation, hydric soils, wetland hydrology, and the final wetland determination without any feedback
- No active context — the application does not tell the user why an indicator would or would not be supported by the data they have entered
- No soil profile diagram, camera-assisted soil color reading, or Munsell reference
- No in-app vegetation calculation (no Dominance Test, Prevalence Index, FAC-Neutral Test, or Morphological Adaptations calculation)

**The Survey 123 → NRV → ADS workflow:**

After field data collection in Survey 123, a Corps delineator must go through a separate multi-step process in the NRV to port that data into the ADS. The ADS then runs its own inference logic based on the imported data and produces a filled ENG Form 6116-1.

This workflow creates a structural risk: a user may manually select indicators in Survey 123 that the ADS's inference logic does not support — or may fail to select indicators that the ADS would auto-select from the same data. The resulting 6116-1 form can contain contradictory entries from both sources side-by-side, with no mechanism to flag or resolve the conflict. There is no step in this workflow that prevents a delineator from advancing with data that does not internally support the determinations recorded.

---

## What ADS Plus Is

ADS Plus is a single HTML file that runs in any modern browser — on a phone, tablet, or laptop — with no installation, no server, and no internet connection required after the initial download. It implements the complete USACE three-factor wetland delineation methodology for the Arid West region, from field data entry through indicator inference to a final wetland determination, in one place.

The application is built on React and runs entirely in the browser. All application state is persisted locally between sessions using the browser's localStorage. No data is transmitted to any server. The application is fully functional offline and does not require connectivity at any point during use.

---

## Core Functions

### 1. Vegetation Data Entry and Analysis

ADS Plus includes a fully integrated vegetation module with the complete NWPL (National Wetland Plant List) for the Arid West region. Users enter vegetation by stratum (tree, shrub, herb, vine/woody vine), and the application performs the following calculations automatically:

- **Dominance Test (50/20 Rule — Indicator 1):** Identifies dominant species in each stratum and across all strata, determines OBL/FACW/FAC/FACU/UPL status for each, and calculates whether >50% of dominants are hydrophytic (OBL, FACW, or FAC).
- **Prevalence Index (Indicator 2):** Calculates the weighted prevalence index from all entered species across all strata and compares to the ≤3.0 threshold.
- **FAC-Neutral Test:** Calculates whether OBL+FACW species outnumber FACU+UPL species among all dominants.
- **Morphological Adaptations (Indicator 3):** Accepts morphological adaptation data for species observed to have adaptations (e.g., adventitious roots, shallow root systems) in the plot but not in adjacent non-wetland areas. The application recalculates the Dominance Test and Prevalence Index with qualifying FACU species reassigned to FAC status, per ERDC/EL TR-08-28 Chapter 2. Critically, it correctly gates this indicator — per the Technical Report, Indicator 3 results are only applied when hydric soils and wetland hydrology each pass or are significantly disturbed or naturally problematic. The ADS does not perform this calculation.
- **Problematic Hydrophytic Vegetation:** Implements the Chapter 5 problematic vegetation determination, gated on the appropriate soil and hydrology criteria.
- **Manual status override:** For unlisted species, allows manual OBL/FACW/FAC/FACU/UPL assignment.

### 2. Soil Data Entry, Profile Diagram, and Hydric Soil Indicator Inference

Users enter soil horizons with depth bounds, texture, Munsell matrix color, and redox feature data (type, color, percent). As data is entered, a real-time **soil profile diagram** is generated — a visual SVG representation of each horizon showing its depth, thickness, Munsell color, and redox notation — giving the user an immediate visual reference for the soil column.

The application automatically evaluates the following hydric soil indicators from the entered data, without any manual interaction:

**All Soils indicators auto-inferred:** A1 (Histosol), A2 (Histic Epipedon), A3 (Black Histic), A9 (Volcanic Soils), A10 (2 cm Muck), A11 (Depleted Below Dark Surface), A12 (Thick Dark Surface)

**Sandy Soils indicators auto-inferred:** S1 (Sandy Mucky Mineral), S4 (Sandy Gleyed Matrix), S5 (Sandy Redox) *(within combos)*

**Loamy and Clayey Soils indicators auto-inferred:** F1 (Loamy Mucky Mineral), F2 (Loamy Gleyed Matrix), F3 (Depleted Matrix), F6 (Redox Dark Surface), F7 (Depleted Dark Surface)

**Combo indicators auto-inferred:** F1+S5, F6+S5, F7+S5, F1+F6, F1+F7, F6+F7 (paired indicators that individually fall short of the Table 8 minimum thickness threshold but collectively meet it)

For each auto-inferred indicator, the application generates a **detailed reason string** explaining exactly which horizon(s) triggered it, what measurements were used, and what criteria were met. This is visible to the user at the point of inference and helps the delineator understand, verify, and document the determination.

For indicators that are partially supported by the data but require field verification to confirm, the application generates a **partial inference** — a flagged result that highlights which criteria are met and which require the user to verify conditions that cannot be determined from the entered data alone.

Above-layer chroma requirements are enforced automatically for all applicable indicators (All Soils, Sandy Soils, Loamy and Clayey Soils) — if the mineral layers above a qualifying horizon have a combined thickness with dominant chroma >2 of ≥6 in., the indicator will not auto-fire, and a data consistency warning will be generated if the user selects it manually.

LRR-gating is applied throughout: indicators approved only in specific Land Resource Regions (B, C, or D) are evaluated only when applicable to the user's selected LRR, and the reason strings reflect which LRR conditions apply.

Problematic hydric soil indicators — including F21 (Red Parent Material), F12 (Iron-Manganese Masses in Tilled Surface), and others — are tracked separately and require hydrophytic vegetation and wetland hydrology (or ≥2 factors significantly disturbed or naturally problematic) before counting toward a soil determination, per Ch. 5.

### 3. Wetland Hydrology Data Entry and Indicator Inference

Users enter observed hydrology measurements (depth to inundation, depth to saturation, depth to water table) and select from the full list of primary and secondary hydrology indicators. The application auto-infers the following based on entered data:

**Auto-inferred from depth measurements:**
- A1h (Surface Inundation): auto-selects when inundation depth ≥0
- A2h (High Water Table): auto-selects when water table ≤12 in.
- A3h (Saturation): fires **Partial** when saturation is recorded ≤12 in. with a water table recorded below the saturation start depth (episaturation pattern), pending field confirmation; auto-selects when a restrictive layer ≤12 in. is present (episaturation waiver)
- C4 (Presence of Reduced Iron): auto-selects when a reduced matrix redox feature is entered within 12 in. of the soil surface
- D3 (Shallow Aquitard): fires **Partial** when a restrictive layer ≤12 in. is recorded, pending user confirmation of the layer's characteristics and site context

All other hydrology indicators are manual-selection-only, consistent with the ADS.

### 4. Soil Color Camera Module

ADS Plus includes a full-screen camera overlay module for camera-assisted soil color measurement. The user prints a custom Munsell reference card (design included in the app), places it against the soil sample, and activates the camera module. The module:

- Detects the reference card in the camera feed using computer vision
- Solves a Color Correction Matrix (CCM) to compensate for variable lighting conditions
- Samples the target soil area and computes the nearest Munsell notation (hue, value, chroma)
- Returns the matched color directly into the horizon entry row

This eliminates the need to carry physical Munsell soil color charts in the field and reduces the subjectivity of manual color matching under varying light conditions.

### 5. In-App Munsell Reference Charts

For users who prefer or require manual color matching, ADS Plus includes built-in Munsell soil color reference charts. Full color chips are displayed for all standard Munsell hues used in soil science (Gley pages included), with interactive navigation. No physical charts or separate reference materials are required.

### 6. GPS Coordinate Capture

The application uses the device's built-in geolocation API to acquire GPS coordinates (latitude, longitude, WGS 84 datum) for the sample point. On mobile devices, this uses the device's GPS hardware. The application can lock onto position and store coordinates with the sample point data. Coordinates are preserved through JSON export/import cycles.

### 7. Data Consistency Warnings

A dedicated consistency engine evaluates all active manual indicator selections against the entered data and generates warnings when selections cannot be supported. Warnings are surfaced prominently at the top of the results panel and flagged on any export. Examples include:

- A manually selected soil indicator when the entered profile data does not meet the indicator's criteria
- An above-layer chroma violation for a manually selected Loamy/Clayey or Sandy soil indicator
- A hydrology indicator selected without supporting measurement data
- A manual vegetation determination that contradicts the calculated Dominance Test and Prevalence Index results
- An incomplete soil profile (missing depth bounds, textures, or percentages) when active indicators require complete horizon data

Neither the ADS nor Survey 123 provides any equivalent to this layer of feedback.

### 8. Full JSON State Import/Export

The entire application state — all site information, vegetation entries, soil horizons, indicator selections, remarks, GPS coordinates, and calculated results — can be exported to a single JSON file at any time. That file can be imported later (on any device) to restore the application to the exact state at time of export. This enables:

- Saving and resuming work across field sessions
- Transferring data from a field device to an office device
- Archiving a complete data record with full fidelity
- Sharing a data set for peer review

### 9. In-App ENG Form 6116-1 PDF Generation

ADS Plus can produce a filled, print-ready copy of the official USACE ENG Form 6116-1 (September 2024 version) directly within the application. The user loads a copy of the blank PDF form once; the application then draws all data — header fields, site location, GPS coordinates, vegetation table, soil profile description, hydric soil indicator results, hydrology indicator results, and remarks — directly onto the three pages of the form. No form fields are used; text is rendered at the precise coordinates of each field on each page.

The resulting PDF can be downloaded immediately with a filename keyed to the sample point ID and date. This eliminates the need for the NRV workflow entirely for the purpose of producing the 6116-1 form output.

### 10. Single-Application Workflow (No Multi-Step Process)

ADS Plus is a complete end-to-end workflow in one place. A delineator can:

1. Open the app on a phone or tablet before going to the field
2. Enter site information and GPS coordinates at the data point
3. Record vegetation by stratum with NWPL-backed status lookup
4. Describe soil horizons and use the camera or reference charts for color
5. Enter hydrology observations and select applicable indicators
6. Review auto-inferred results, partial inference alerts, and consistency warnings in real time
7. Add remarks, note normal circumstances or disturbance flags, handle problematic criteria
8. See the complete three-factor wetland determination
9. Export a filled ENG Form 6116-1 PDF and a JSON archive — both on the same device, in the same session

No external software, no upload to NRV, no transfer to Excel, no second device required.

---

## How ADS Plus Compares to the Existing Toolkit

| Capability | ADS Plus | ADS (Excel) | Survey 123 |
|---|---|---|---|
| Mobile-optimized field data entry | Yes | No | Yes |
| Works fully offline | Yes | Yes (Excel) | No (requires connectivity) |
| Auto-inference for hydric soil indicators | Yes — with reason strings | Yes — results only, no explanation | No |
| Auto-inference for hydrology indicators | Yes — with reason strings | Partial — results only | No |
| Partial inference (flagged, field-verify prompts) | Yes | Internal only — not shown to user | No |
| Combo indicator detection (F1+S5, F6+F7, etc.) | Yes | Yes — results only | No |
| Data consistency warnings | Yes | No | No |
| Context awareness (explains why indicators fire) | Yes | No | No |
| Vegetation Dominance Test calculation | Yes | Yes | No |
| Prevalence Index calculation | Yes | Yes | No |
| FAC-Neutral Test calculation | Yes | Yes | No |
| Morphological Adaptations (Indicator 3) calculation | Yes — including gating rules | No | No |
| LRR-gated indicator logic | Yes | Yes | No |
| Problematic indicator tracking | Yes | Yes | No |
| Camera-assisted soil color measurement | Yes | No | No |
| In-app Munsell reference charts | Yes | No | No |
| Real-time soil profile diagram | Yes | No | No |
| GPS coordinate capture | Yes | No | Yes |
| JSON full-state import/export | Yes | No | No |
| In-app PDF (ENG Form 6116-1) generation | Yes | No | Via NRV only |
| Multi-step NRV process required | No | Required for PDF | Required |
| Risk of NRV/Survey 123 data contradiction | None | Possible via NRV | Present |
| Internet connection required | No | No | Yes |

---

## Technical Accuracy Improvements Over the ADS

During the development and audit of ADS Plus, several areas were identified where the ADS formulas do not reflect the most current NRCS *Field Indicators of Hydric Soils in the United States* guidance (Version 9.3). In these cases, ADS Plus implements the specification as written in the NRCS document rather than replicating what appears to be unfixed holdovers in the ADS formulas. These are deliberate, documented deviations, not errors.

**F3 (Depleted Matrix) — Depth thresholds:**
The current NRCS Field Indicators guide specifies qualifying depths of ≤4 in. (for the ≥2 in. thick path) and ≤10 in. (for the ≥6 in. thick path). ADS Plus uses these values. The ADS uses ≤6 in. and ≤12 in., which appear to be holdovers from a prior version that were not updated when the indicator criteria changed.

**F8 (Redox Depressions) — Depth threshold:**
The NRCS Field Indicators guide specifies a qualifying depth of ≤4 in. for this indicator. ADS Plus uses ≤4 in. The ADS uses ≤6 in., which also appears to be an unfixed holdover.

**Above-layer chroma checking (S4 Sandy Gleyed Matrix):**
ADS Plus checks all mineral layers above the qualifying layer for chroma compliance, consistent with the NRCS specification. The ADS accumulates only sandy layers above, which narrows the check and can allow indicators to fire in profiles that do not strictly meet the specification.

**Morphological Adaptations gating (Indicator 3):**
ADS Plus correctly gates the Morphological Adaptations indicator per ERDC/EL TR-08-28: the recalculated Dominance Test and Prevalence Index from Indicator 3 are only applied when hydric soils and wetland hydrology each pass or are significantly disturbed or naturally problematic. The ADS does not implement this calculation at all.

---

## Use Cases

**USACE Regulatory Staff — Field Delineations:**
ADS Plus is designed as a primary field tool that replaces the Survey 123 → NRV → ADS workflow. A regulator can carry a phone or tablet with ADS Plus pre-loaded and complete the entire data sheet in the field, including PDF output. This eliminates the post-field data re-entry step and removes the risk of cross-tool data contradictions on the final form.

**USACE Regulatory Staff — Office Review:**
A delineator can import a consultant-submitted JSON export to review the submitted data at full fidelity — seeing every horizon, every selection, every auto-inference result — without relying on the narrative of a PDF alone.

**Wetland Delineation Consultants:**
Consultants can use ADS Plus as a field data collection and analysis tool on any device, with no software licensing cost, producing a filled ENG Form 6116-1 directly. The JSON export provides a complete, auditable record of all entered data and calculated results for the project file.

**Training and Education:**
ADS Plus's context-aware inference engine — which explains, in plain language, exactly which layer at which depth with which measured properties triggered each indicator — makes it an effective training tool for delineators learning to apply the Arid West Supplement. The reason strings and consistency warnings provide real-time instruction grounded in actual data.

---

## Future Potential

ADS Plus is structured to support capabilities that are not yet implemented pending appropriate approvals:

**Direct NRV/Corps database integration:** The existing JSON export format is a complete and structured record of all sample point data. With appropriate API integration and Corps approval, ADS Plus could upload directly to the NRV or a similar regulatory database, eliminating the Survey 123 step entirely while preserving full data fidelity and the audit trail.

**Multi-sample-point project files:** The current application handles one sample point per session. A project-level wrapper could aggregate multiple sample points under a single project record with shared metadata.

**Expanded regional coverage:** The application's architecture (LRR-gated indicator logic, pluggable indicator sets) is designed to support extension to other USACE regional supplements beyond the Arid West.

---

## Summary

ADS Plus represents a ground-up rethinking of the wetland delineation data workflow for the Arid West — one that treats the field delineator as the primary user rather than the back-office data processor. By unifying data entry, inference, reference materials, photo documentation, and form output into a single offline-capable application, ADS Plus eliminates the multi-tool, multi-step friction of the current Survey 123 → NRV → ADS pipeline, reduces the risk of contradictory data on the final regulatory output, and provides a level of real-time guidance and data quality feedback that neither the ADS nor Survey 123 currently offer.

The application matches the ADS on the indicators it has been designed to cover, exceeds it in technical accuracy for several indicator criteria, and adds a substantial set of capabilities — camera-assisted soil color measurement, morphological adaptations calculation, data consistency enforcement, in-app PDF production, and full JSON state portability — that have no equivalent in the existing toolkit.
