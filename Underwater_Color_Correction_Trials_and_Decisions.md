# Underwater ROV Color Correction — Trial History

**Goal:** Keep the white PVC structure neutral, preserve red targets, avoid unnatural background casts, and reach 30 FPS.

**Current decision:** Keep the final `underwater_color_correction_final.ipynb` as the best candidate. Color accuracy is not fully solved; end-to-end FPS must be measured on deployment hardware.

| Step | Approach | What happened / why it was insufficient | 
|---|---|---|---|
| 1. **CLAHE** | Improved contrast, but the cyan/blue cast remained. It changes local contrast, not the overall color balance. | Kept the contrast-enhancement idea; needed a color-correction stage. |
| 2. **CLAHE + Gamma** | Made the frame brighter, but did not remove the cyan cast. Brightness correction is not white balance. | Rejected as a complete solution; gamma is only useful if brightness is the issue. |
| 3. **Gray World + CLAHE (float)** | Made the white structure look more neutral, but remained slow and could be biased by the blue-heavy scene. Initial benchmark: ~12.12 FPS. | Kept the approach, optimized its implementation. |
| 4. **SimpleWB + CLAHE + Gamma + Sharpening (Sakr pipeline)** | Produced strong color artifacts/overcorrection and was still below 30 FPS (~13.90 FPS). Red-looking patches did not prove target-color accuracy. | Rejected as-is. |
| 5. **Gray World + CLAHE using LUTs** | Replaced per-pixel float multiplication with OpenCV lookup tables. Full-video processing-stage benchmark at 960×540: 21.67 ms mean, 27.33 ms P95, 46.15 FPS. Color still had residual cyan and dark red targets. | Kept as the performance baseline; speed improved, color quality still needed work. |
| 6. **Lower working resolution (960×540)** | Helped reduce processing time, then combined with LUTs to meet the processing-stage 30 FPS target. Resizing back to 1080p does not restore detail lost during downscaling. | Kept for the real-time performance tradeoff. |
| 7. **Stronger Gray World (alpha sweep)** | Higher correction reduced cyan in some frames but made the floor/scene pink or orange. The best strength changed with scene content and lighting. | Stopped increasing alpha blindly. |
| 8. **Neutral-pixel white balance** | A brightness/low-saturation mask did not clearly improve the result; selected pixels may not have been a reliable white reference. | Do not repeat the same mask unchanged; use a verified white ROI instead. |
| 9. **Manual red-channel gain** | Global red gain did not reliably make targets red and risks turning white areas pink and amplifying noise. It cannot recover red information the camera did not capture. | Rejected as a standalone fix. |
| 10. **Dive Color Corrector** | Reduced cyan but introduced a strong yellow cast; red targets still were not clearly restored on the tested frame. | Not selected alone for this scene. |
| 11. **Blend Dive + Gray World** | Increasing the Dive blend weight increased yellow tint without solving red-target appearance. | Rejected; blending outputs did not solve the underlying color loss. |
| 12. **Final notebook: UnderwaterCorrector / Fast pipeline** | Combines red-channel denoising, capped white balance, shared contrast stretch/LUTs, luminance CLAHE, mild saturation, temporal smoothing, periodic updates, and optional white-reference lock. | Current best candidate. Validate white PVC and red-target ROIs across several clips and measure end-to-end FPS before production. |

## Final approach

The current notebook combines:
- Red-channel denoising.
- Capped white balance and LUT-based correction.
- Shared contrast stretch.
- CLAHE on luminance.
- Mild saturation adjustment.
- Temporal smoothing and periodic parameter updates.
- Optional calibration against a known white PVC region.

## Key lesson

The speed problem was improved by LUTs and processing at 960×540: the timed processing stage reached **46.15 FPS** (21.67 ms mean, 27.33 ms P95) on the tested Colab run. This excluded video decoding and output encoding.

The color problem is harder: global gains can trade cyan for pink/orange/yellow, and red amplification cannot recover information that was never captured. Do not repeat a failed method unchanged. Validate white, red-target, and background regions across multiple clips, then measure end-to-end throughput.
