<!-- cell 0 -->
# Biohub Cell Tracking: 0.946 LB

This update continues the progression from **0.934 → 0.939 → 0.941 → 0.946** Public LB, a total of **+0.012** from the original 0.934 solution. The core architecture is unchanged throughout — same UNet3D + node-transformer detection, dual-seed ensemble, ILP-based global linking, and bidirectional association framework. The **0.934 → 0.939** step widened safe division recovery while making bidirectional association more conservative. The **0.939 → 0.941** step redistributed the repair budget by widening the safe-division parent search while tightening its learned confirmation and gap closing. This **0.941 → 0.946** step keeps those repair settings untouched and extends the existing eight-view spatial TTA from detection outputs into the UNet features used for association.

| Component | 0.934 | 0.939 | 0.941 | 0.946 |
| --- | ---: | ---: | ---: | ---: |
| Safe division parent max distance | 7.0 µm | 7.0 µm | 9.0 µm | 9.0 µm |
| Safe division sister max distance | 12.0 µm | 14.0 µm | 14.0 µm | 14.0 µm |
| Sister symmetry gate | off | 0.6 | 0.6 | 0.6 |
| DeepCenter safe-division veto | off | on | on | on |
| DeepCenter safe-division threshold | 0.12 | 0.12 | 0.25 | 0.25 |
| Gap-close base radius | 5.8 µm | 5.8 µm | 5.0 µm | 5.0 µm |
| Bidirectional association weight | 0.30 | 0.15 | 0.15 | 0.15 |
| Bidirectional winner-agreement guard | on | removed | removed | removed |
| DeepCenter expected epoch | 500 | 2 | 2 | 2 |
| Detection spatial TTA views | 8 | 8 | 8 | 8 |
| Edge-feature spatial TTA | off | off | off | on |

**Recap: wider division search with a symmetry and DeepCenter guard (0.934 → 0.939).** The sister-distance limit widened from 12 µm to 14 µm to recover genuine divisions where the two daughters had drifted slightly farther apart. Since a wider radius alone also admits more spurious pairings, two safeguards compensate: a symmetry gate rejecting daughter candidates sitting at very uneven distances from the parent,

$$
\frac{|d_1 - d_2|}{(d_1 + d_2)/2} \le 0.6
$$

and the DeepCenter veto, so every candidate the wider geometric search proposes still needs independent confirmation before being accepted.

**Recap: a more conservative bidirectional fusion (0.934 → 0.939).** Association is normally scored only forward in time — for a cell at frame $t$, which detection at $t+1$ it most likely became. This pipeline also scores it in reverse — given a detection at $t+1$, which detection at $t$ it most likely came from. Let $p_f$ be the forward probability and $p_r$ the reverse probability for a candidate pair. The reverse weight dropped from 0.30 to 0.15, so $p_r$ acts as a lighter supporting vote rather than pulling $p_f$ as strongly, and the previous requirement that forward and reverse agree on the exact same winner (the agreement guard) was replaced with their harmonic mean directly:

$$
p_{\text{harmonic}} = \frac{2 \, p_f \, p_r}{p_f + p_r}
$$

which keeps the mutual-support behavior — a link needs support from both directions to score highly — without the hard all-or-nothing condition.

**Recap: redistributing the repair budget (0.939 → 0.941).** The association setup that reached 0.939 is left untouched; three repair-stage parameters are retuned instead. The safe-division **parent** radius increases from 7 µm to 9 µm. This is a different axis from the earlier 12 → 14 µm change — that one widened the allowed distance *between the two daughters*; this one widens the distance *from the parent to the missing daughter candidate*, recovering parent-child links the previous radius was cutting off. That wider parent search creates more room for false division proposals, so the DeepCenter confirmation threshold is raised from 0.12 to 0.25. Candidates admitted by the larger radius still have to clear a stricter learned confirmation before being recovered as divisions — the search widens, but the bar to pass it goes up with it. At the same time, gap closing is tightened, with its base radius reduced from 5.8 µm to 5.0 µm — a separate repair path that reconnects broken tracks rather than recovering divisions. Candidates for that repair now have to be spatially closer. Together these three changes shift where the repair budget is spent: more room is given specifically to plausible divisions, while a stricter DeepCenter threshold and a tighter gap-close radius hold precision elsewhere.

**What's new in 0.946: extending spatial TTA into association features.** The full 0.941 tracking and repair configuration is retained unchanged. Importantly, the improvement does **not** come from increasing the number of geometric TTA views. The 0.941 pipeline already evaluates detection using the full eight spatial transformations of the D4 symmetry group. The change in 0.946 is where the information from those eight views is used. The eight spatial views are:

| # | D4 Spatial View | Operation |
| ---: | --- | --- |
| 1 | Identity | Original orientation |
| 2 | Horizontal flip | Reflection across one spatial axis |
| 3 | Vertical flip | Reflection across the other spatial axis |
| 4 | Horizontal + vertical flip | Reflection across both axes, equivalent to a 180° rotation |
| 5 | Rotation 90° | 90° spatial rotation |
| 6 | Rotation 270° | 270° spatial rotation |
| 7 | Transpose | Reflection across one diagonal |
| 8 | Anti-transpose | Reflection across the opposite diagonal |

These eight operations cover the unique symmetries of a square under the D4 group. A separate 180° rotation does not need to be added because flipping both spatial axes already produces the same geometric transformation. Therefore, moving from 0.941 to 0.946 is not an **8-view → 16-view** expansion or a **4-view → 8-view** expansion. Both versions use the same eight geometric views. For an input $x$, let $T_i$ denote the $i$-th D4 transformation. Each transformed input $T_i(x)$ is passed through the detector, producing a detection-logit map. Because each output is expressed in the coordinate system of its transformed input, it first has to be mapped back to the original orientation using the corresponding inverse transformation $T_i^{-1}$. Detection TTA is therefore:

$$
\bar{L}(x) = \frac{1}{8}\sum_{i=1}^{8} T_i^{-1}\left(L(T_i(x))\right)
$$

where $L(T_i(x))$ denotes the detection logits produced from the $i$-th transformed view. This averaging reduces orientation-specific variation in the final detection map. That mechanism was **already present before 0.946**. In 0.941, all eight views contributed to $\bar{L}(x)$, so cell detection was already spatially ensembled. However, the UNet feature tensor used downstream for association did not receive the same treatment. Once detection TTA was complete, the edge predictor still relied on the UNet representation extracted from the original orientation. Conceptually, the 0.941 inference path was:

$$
\{T_i(x)\}_{i=1}^{8}
\rightarrow
\bar{L}(x)
\rightarrow
\text{detection}
$$

while the association representation remained:

$$
F_{\text{association}}(x) = F(x)
$$

where $F(x)$ is the UNet feature map from the original view only. This creates an asymmetry in the inference pipeline. Detection benefits from eight spatial observations, but the representation used to decide which detections should be connected through time still depends on a single orientation. The 0.946 update closes this mismatch by extending the same transform-invert-average procedure to the UNet feature maps. For every transformed input, the corresponding feature representation $F(T_i(x))$ is retained. Each feature map is then mapped back into the original coordinate system before averaging:

$$
\bar{F}(x) = \frac{1}{8}\sum_{i=1}^{8} T_i^{-1}\left(F(T_i(x))\right)
$$

The inverse transformation is important. A feature map extracted from a 90°-rotated input cannot be averaged directly with the original feature map because their spatial coordinates do not correspond. Applying $T_i^{-1}$ first realigns every representation to the same coordinate system. The edge predictor then receives:

$$
F_{\text{association}}(x) = \bar{F}(x)
$$

instead of:

$$
F_{\text{association}}(x) = F(x)
$$

so the difference between 0.941 and 0.946 can be summarized directly:

| TTA Component | 0.941 Public LB | 0.946 Public LB |
| --- | --- | --- |
| Number of spatial views | 8 D4 views | 8 D4 views |
| Identity | on | on |
| Horizontal flip | on | on |
| Vertical flip | on | on |
| Horizontal + vertical flip | on | on |
| Rotation 90° | on | on |
| Rotation 270° | on | on |
| Transpose | on | on |
| Anti-transpose | on | on |
| Detection-logit TTA | 8-view inverse-transform average | 8-view inverse-transform average |
| Detection representation | spatially ensembled | spatially ensembled |
| UNet edge-feature TTA | off | on |
| UNet features used by edge predictor | original view only | all 8 D4 views |
| Feature inverse transform | not used for edge-feature averaging | applied to each transformed feature map |
| Edge predictor input | $F(x)$ | $\bar{F}(x)$ |
| Association representation | single-view | spatially ensembled |
| Repair configuration | 0.941 settings | unchanged from 0.941 |

The key distinction is therefore not the transformations themselves, but **how far their information propagates through inference**. In **0.941**, the eight D4 views terminate at the detection ensemble:

$$
\text{8 D4 views}
\rightarrow
\text{detection logits}
\rightarrow
\text{inverse transform}
\rightarrow
\text{average}
\rightarrow
\text{detection}
$$

while association returns to a single-view representation:

$$
\text{original view}
\rightarrow
F(x)
\rightarrow
\text{edge predictor}
\rightarrow
\text{association}
$$

In **0.946**, the same eight views contribute to both paths:

$$
\text{8 D4 views}
\rightarrow
\begin{cases}
\text{detection logits}
\rightarrow
\text{inverse transform}
\rightarrow
\bar{L}(x)
\\
\text{UNet features}
\rightarrow
\text{inverse transform}
\rightarrow
\bar{F}(x)
\end{cases}
$$

and the averaged feature representation $\bar{F}(x)$ is then consumed by the edge predictor. This matters because association is not determined only by the detected cell coordinates. The edge predictor also uses learned UNet representations to evaluate candidate temporal links. Under the previous setup, an otherwise valid association could still be affected by orientation-specific variation in the original-view feature map even though detection itself had already been stabilized with TTA. Averaging the aligned feature maps reduces that dependence on a particular spatial orientation and makes the representation supplied to association consistent with the ensemble principle already used for detection. The change is deliberately narrow. It does not globally relax edge thresholds, increase repair radii, add more geometric transformations, or rewrite the linking procedure. The 9 µm safe-division parent radius, 14 µm sister radius, 0.6 symmetry gate, 0.25 DeepCenter safe-division threshold, 5.0 µm gap-close radius, 0.15 bidirectional association weight, harmonic forward-reverse fusion, dual-seed ensemble, ILP-based global linking, and the rest of the 0.941 repair configuration are retained. The **0.941 → 0.946** step instead makes better use of the eight spatial observations that were already being computed by extending their contribution from the detection output to the learned features used for temporal association.

**Summary.** The progression remains cumulative rather than a redesign. **0.934 → 0.939** widened safe division recovery with additional geometric and DeepCenter safeguards while making bidirectional association more conservative. **0.939 → 0.941** redistributed the repair budget by widening the parent-side division search, strengthening its DeepCenter confirmation, and tightening gap closing. **0.941 → 0.946** leaves those tracking and repair settings unchanged and extends the existing eight-view D4 ensemble from detection logits into the UNet features consumed by the edge predictor. The number and type of spatial transforms do not change; what changes is that association now benefits from the same spatially ensembled representation that detection already had. The resulting progression is **0.934 → 0.939 → 0.941 → 0.946**, for a cumulative **+0.012 Public LB** improvement over the original 0.934 solution and a **+0.005** improvement over the previous 0.941 configuration.

**If you have ideas on how to push this toward 0.950, feel free to reach out — I'm also looking for a teammate for this competition. Also, if you find this useful, an upvote would be appreciated!**
