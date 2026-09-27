# BH-0005 candidate: minimum track length 4

Status: prepared locally; not pushed; not submitted.

## Single-variable hypothesis

The completed BH-0004 run used an effective `BIOHUB_OUTPUT_MIN_TRACK_LEN=6`
and removed 3,349 short-track nodes across the four test datasets. Its adaptive
short-track rescue did not trigger on any dataset. Independent
embryo-disjoint experiments in the public `m9h/biohub-starter` research line
reported that reducing the minimum track length from 6 to 4 improved both
embryo folds, while division-focused additions repeatedly failed their kill
criteria.

BH-0005 therefore changes only:

```text
BIOHUB_OUTPUT_MIN_TRACK_LEN: 6 -> 4
```

The matching configuration guard was changed from `6.0` to `4.0`; all
detector, association, ILP, gap, division, and TTA settings remain unchanged.
The first two notebook cells were executed locally and the configuration guard
passed.

Notebook SHA-256: `59d2e690b9925d29bc60aa8a094b5931fd31376f0aef5bb51d04241525b0f7e1`

## Promotion gate

Do not submit while BH-0004 ref `56591286` is pending. After BH-0004 produces
a public score:

1. Run this notebook as a new non-submitting version.
2. Require successful completion and the same strict topology validator used
   for BH-0004.
3. Compare node/edge counts and the held-out proxy by embryo prefix; the mixed
   eight-sample aggregate alone is not sufficient evidence.
4. Use a second official submission slot only if the one-factor change is
   consistent with both embryo folds or if the competition deadline leaves no
   time for another independent check and the expected information value
   exceeds the slot cost.

This experiment intentionally avoids division-threshold searches and
multi-knob tuning.

## Publication boundary

This is a first-party decision record and scalar run context. It contains no
image, annotation, row-level tracking output, notebook export, or submission
bundle. Obtain authorized competition material from the official source cited
by this repository before reproducing the experiment.
