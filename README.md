# Space_Pic

1. 获取到都是
2. 访问


方案一

1. 获取原图
2. 上传原图至Astrometry
3. 生成带参数的FITS文件
4. 使用原图FITS文件，在Grok或者Gemini上生成渲染图片

方案二
1. 获取原图
2. 根据原图及对应的星体名称在Grok或者Gemini上生成渲染图片



根据评估，方案一相较方案二优势明显，但是会涉及Astroetry的API调用问题

coze提供的Gemini提示词：
Gemini图片增强
You are a professional astrophotography image processor. Enhance this REAL telescope photograph following these exact steps:

STEP 1 - LIGHT POLLUTION REMOVAL: Remove all light pollution color casts (orange, green, brown gradients). Subtract background gradient pollution evenly. Sky background must become neutral dark, not tinted.

STEP 2 - NOISE REDUCTION: Apply advanced multi-scale denoising to the background sky while preserving ALL real star signals including the faintest ones. Aggressive on smooth background, gentle near star edges.

STEP 3 - NONLINEAR STRETCH: Calculate the statistical median pixel value of the image. Apply a nonlinear stretch (arcsinh-like) pivoting on this median balance point to expand mid-tone dynamic range and reveal faint nebula/galaxy signal while keeping bright stars from saturating. The stretch should be moderate.

STEP 4 - WHITE BALANCE & COLOR CALIBRATION: Auto-correct white balance. Restore accurate stellar colors by spectral type (hot O/B → blue-white, A → white, F/G → yellow-white, K → orange, M → red-orange).

Astrometry.net plate-solving calibration data:
- Center: RA {ra_hms}, DEC {dec_dms}
- FOV radius: {radius_deg}°
- Pixel scale: {pixscale} arcsec/pixel
- Orientation: {orientation}°
- Objects in field: {objects_list}

Based on the identified objects above, calibrate any nebular/galactic emission to scientifically correct hues.

STEP 5 - SUPER-RESOLUTION & SHARPENING: Reconstruct at higher effective resolution. Tighten star profiles to be more point-like. Enhance fine detail without creating artifacts or halos around stars.

ABSOLUTE SCIENTIFIC INTEGRITY RULES:
- Do NOT add any stars, nebulae, galaxies, or celestial objects NOT captured in the original telescope exposure
- Only enhance what is genuinely present in the image signal
- If identified objects are present as faint signal, enhance them realistically matching amateur astrophotography quality
- Do NOT render at Hubble/Space Telescope level detail - match what amateur equipment can capture
- Do NOT add diffraction spikes, lens flares, or optical artifacts not in the original
- Star count must remain consistent with the original
- Any color or detail enhancement must be derived from existing signal, not imagined
- This is a SCIENTIFIC OBSERVATION. Authenticity overrides aesthetics.

二次验证（真实性校验）
You are an expert astronomical image analyst. Your task is to verify the scientific authenticity of an enhanced astrophotography image by comparing it with the original.

You will be given TWO images:
- IMAGE 1: The ORIGINAL raw telescope photograph
- IMAGE 2: The ENHANCED version after processing

Perform the following verification checks and report your findings:

CHECK 1 - STAR COUNT INTEGRITY
Count the approximate number of visible stars in both images. The enhanced image must NOT contain significantly more stars than the original. Report: original ~N stars, enhanced ~N stars, PASS/FAIL.

CHECK 2 - STAR POSITION ACCURACY
Verify that stars in the enhanced image appear at the same positions as in the original. No stars should be displaced, duplicated, or removed. Report: PASS/FAIL with details.

CHECK 3 - NO FABRICATED CELESTIAL OBJECTS
Carefully examine whether the enhanced image contains any nebulae, galaxies, diffuse emission, star clusters, or other celestial structures that do NOT exist in the original image. Even if a known object (like M1) genuinely exists in this sky region, if the ORIGINAL image did not capture it, the enhanced version must NOT fabricate it. Report: PASS/FAIL with list of any suspicious additions.

CHECK 4 - NO FABRICATED OPTICAL ARTIFACTS
Check for diffraction spikes, lens flares, satellite trails, or other optical artifacts in the enhanced image that are not present in the original. Report: PASS/FAIL.

CHECK 5 - COLOR AUTHENTICITY
Evaluate whether the colors in the enhanced image are scientifically plausible. Star colors should match known spectral types. Nebular emission colors should match known emission lines (H-alpha → reddish-pink, OIII → blue-green, SII → deep red). No implausible or exaggerated colors. Report: PASS/FAIL with details.

CHECK 6 - DYNAMIC RANGE REALISM
Assess whether the brightness and contrast enhancement is realistic for amateur astrophotography equipment, or if it appears exaggerated to the level of professional space telescope imagery. Report: PASS/FAIL.

OVERALL VERDICT
Based on all checks above, give a final assessment:
- AUTHENTIC: The enhancement preserves scientific integrity
- QUESTIONABLE: Some issues detected, list them
- FABRICATED: Significant fabrication detected, the image cannot be trusted for scientific use

Plate-solving context for this image:
- Center: RA {ra_hms}, DEC {dec_dms}
- Objects in field: {objects_list}

二次验证是把原图和增强图同时发给Gemini，让它做对比校验，这样能及时发现AI是否"画多了"。


增加星座绘图预设的增值业务
