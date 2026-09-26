---
layout: post
title: "Testing VLM object matching for scene registration"
date: 2026-09-26 06:00:00 +0000
categories: [physical-ai, computer-vision, spatial-computing]
---

Having previously built mixed reality applications on top of spatial rendering, anchoring and holographic coordinate systems, I found those systems did their job, but they were rigid. You got the platform's answer about where things were, and there was rarely a place to add the judgement a particular site or workflow needed.

Working as an AI engineer now, part of my work is researching digital twins of generator assemblies: how they fit on the shop floor and at client sites, and what anchoring and logistics look like around them. The same question comes up again in a different form. When a physical asset is seen again, somewhere else or from another angle, how does a system know it's the same one, and how does it tie that back to a coordinate system?

That is the part of physical AI I think is worth the effort. A model that recognises things becomes useful when its output lands in a measurement someone can act on. PROSE, a recent paper on training-free scene registration with vision-language models, sits right on that line, so I rebuilt a simplified version to see how it behaves.

<https://arxiv.org/abs/2606.16569>

The short version: the geometry worked. The VLM never accepted a wrong match, but it accepted so few right ones that registration became impossible. A plain height filter, with no vision model at all, gave the geometry enough to recover the alignment.

## Two visits, one coordinate system

Imagine walking through a room twice, entering from different directions. On the first visit you attach a digital note to a cabinet. On the second, you want that note to appear on the same cabinet.

For that to work, both captures need a shared coordinate system. The system has to recognise that cabinet, tell it apart from similar furniture, and find enough reliable anchors across the room to line the two visits up. Persistent annotations, robot map reuse and before/after site comparisons all depend on this step. This project stops short of those applications; it's about the step itself.

## What I built

[PROSE: Training-Free Egocentric Scene Registration with Vision-Language Models](https://arxiv.org/abs/2606.16569), by Zhiang Chen and colleagues, combines object-level representations, height constraints, visual verification and geometric consensus. "Training-free" means it adds no learned parameters; the pretrained models still bring their own learned abilities.

My version is simpler, and I treat it as my own controlled study rather than a reproduction of the paper's benchmark. I started from annotated objects in the Aria Digital Twin (ADT) dataset, 267 in scan A and 281 in scan B, so I could study matching and registration before detection or depth errors got in the way. For registration I used a standard three-point Kabsch/RANSAC estimator rather than the per-object hypothesis strategy the paper describes.

```text
ADT annotated objects and image crops
                  ↓
Object centres and physical heights
                  ↓
Height-based candidate shortlist
                  ↓
VLM verification in both directions
                  ↓
Keep reciprocal YES decisions
                  ↓
Kabsch hypotheses + RANSAC consensus
                  ↓
B-to-A transform, or an explicit refusal
```

Height narrows the candidates. The VLM decides whether two crops show the same physical object. The geometry checks whether the surviving pairs agree on one rigid transform.

The VLM can answer `YES`, `NO` or `UNKNOWN`, and a pair only counts if both directions say `YES`. I kept `UNKNOWN` as its own outcome, separate from API errors and malformed replies. "The model couldn't tell" is useful information, and it shouldn't quietly become a yes or disappear into an error count.

![Object-centre and height graph rendered in Open3D]({{ "/assets/images/prose/object-graph.png" | relative_url }})

*Figure 1. Object centres and heights for scan B after alignment, rendered in Open3D. Each marker is one annotated object.*

## Checking the geometry first

Before blaming anything on the matcher, I tested registration on its own. I moved scan B by a known rigid transform and asked the solver to recover it:

```text
a ≈ R @ b + t
```

Kabsch fits a rotation and translation to paired points. RANSAC fits many small samples and keeps the transform that the most pairs agree with. The core of the fit:

```python
ca, cb = a.mean(axis=0), b.mean(axis=0)
u, singular, vt = np.linalg.svd((b - cb).T @ (a - ca))
correction = np.diag([1., 1., np.linalg.det(vt.T @ u.T)])
rotation = vt.T @ correction @ u.T
translation = ca - rotation @ cb
```

The determinant term stops the solution from turning into a reflection. The solver also refuses degenerate input, such as points that coincide or lie nearly on a line.

| Controlled test | Correspondences | Inliers | Rotation error | Translation error |
| --- | ---: | ---: | ---: | ---: |
| Known identities, clean geometry | 267 | 267 | ~0° | ~0 m |
| 5 cm noise, 134 wrong identities | 267 | 134 | 0.104° | 1.65 cm |

In the corrupted case, every coordinate in both scans got 5 cm of Gaussian noise, and 134 of the 267 pairs were swapped to the wrong object. With a 30 cm inlier threshold, the solver still landed within about a tenth of a degree and under 2 cm.

Of those 134 inliers, 133 were correct and one was the wrong object sitting in a geometrically plausible spot. Agreeing with the transform and being the right object are separate things.

## Where the anchors are matters as much as how many

I extended this to 425 runs over 85 conditions, five seeds each, varying noise, wrong matches, inlier thresholds and how spread out the pairs were. A run counted as recovered if the rotation error was under 5° and the translation error under 0.2 m.

![Controlled recovery rates across noise, wrong correspondences, and inlier thresholds]({{ "/assets/images/prose/recovery-rates.png" | relative_url }})

*Figure 2. Recovery rate across noise and outlier levels. Each cell is five trials; the three panels use different inlier thresholds.*

How spread out the pairs were mattered most:

| Spatial support (5 cm noise, 30 cm threshold) | Recovered |
| --- | ---: |
| Six randomly placed pairs | 5/5 |
| Six clustered pairs | 1/5 |
| Ten clustered pairs | 1/5 |
| Thirty clustered pairs | 4/5 |

Every clustered six-pair run returned a transform, and only one of them was accurate. "The solver produced an answer" and "the answer is right" are two different checks. For a physical system, the second depends on where the evidence sits in the room as much as on how much of it there is.

## Then the vision model became the bottleneck

The first VLM trial produced 70 `UNKNOWN` answers and no accepted matches before its budget ran out. A second small pilot accepted two true pairs, still too few to estimate a pose. The provider had also changed the underlying model between the two runs, so I couldn't credit the improvement to my crop changes.

The more useful experiment used six objects present in both scans, spread 2.4 m to 9.8 m apart, plus two distractors that only exist in scan B. I held eight other shared objects back, untouched, for a later test. It took 142 paid calls to the `deepseek-flash` model alias, roughly three cents at peak token rates.

I ran three variants on the same ten comparison pairs:

| Evidence and prompt | YES / NO / UNKNOWN (20 directions) | True pairs accepted | False pairs accepted |
| --- | ---: | ---: | ---: |
| Tight crops, default prompt | 5 / 6 / 9 | 2/6 | 0/4 |
| Tight crops + marked context, default prompt | 4 / 14 / 2 | 2/6 | 0/4 |
| Tight crops, stricter prompt | 4 / 6 / 10 | 2/6 | 0/4 |

Adding context cut the abstentions, but mostly turned them into `NO` rather than `YES`. The stricter prompt changed nothing. Every variant accepted the same two true pairs, the armchair and the shoe rack, and none of the false ones.

There's a catch I only found while writing this up. The crops went to the model in the Aria camera's native orientation, turned 90° from upright, and my pipeline never rotated them. Some of the low recall is probably down to that rather than the model's judgement, so treat the results below as a measurement of this pipeline as it ran.

![Six true object pairs inspected after the spread experiment]({{ "/assets/images/prose/true-pair-inspection.png" | relative_url }})

*Figure 3. The six shared objects, scan A on the left and scan B on the right: dining table, armchair, shoe rack, red chair, bookcase, and an object labelled "figurine" whose crops mostly show dark shelving. Only the armchair and shoe rack were accepted. The crops are shown as they were sent to the model, in the Aria camera's native sideways orientation.*

The dining table is the most telling miss. The tabletop is clearly the same, but the things on it have moved. All three variants said `NO` in both directions. That's exactly the situation a site twin has to handle: the asset is the same, and what surrounds it isn't.

## The ablation that changed my interpretation

I ran four versions of the pipeline on the same six-object cohort. The full pipeline used a top-5 height shortlist; the no-height version sent all 6 × 8 pairs to the VLM in both directions.

| Variant | Pairs kept: true / false | Match recall | Registration |
| --- | ---: | ---: | --- |
| Full: height + VLM + RANSAC | 2 / 0 | 2/6 | Not enough pairs |
| No height: all-pairs VLM + RANSAC | 2 / 0 | 2/6 | Not enough pairs |
| No VLM: reciprocal height + RANSAC | 6 / 22 | 6/6 | Recovered (6 inliers) |
| No RANSAC: VLM matches + Kabsch | 2 / 0 | 2/6 | Not enough pairs |

The height-only candidate set was noisy: 6 of its 28 pairs were correct. But all six true anchors were in it, and RANSAC picked them out and recovered the known transform to numerical precision.

The VLM-filtered set was perfectly precise and found 2 of 6 true pairs. Removing the height shortlist and asking about every pair gave the same result, so the shortlist wasn't hiding matches. The no-RANSAC row only confirms that two pairs are below the three-point minimum for any fit.

The lesson I took from it: judge a matcher by the task it feeds. A conservative identity check looks excellent in a precision table and can still make alignment impossible.

It's the same kind of problem I kept running into with anchoring platforms. Each component makes a fixed call about what counts as good enough, and the system around it can't ask for anything different. The more useful design lets the step that owns the outcome, here the geometry, decide how much uncertainty it can absorb. That's the extra layer I wanted back then, and it's what I'd build into a twin pipeline now.

## Making results traceable

Paid model calls and changing provider models make it easy to lose track of what produced a number. Every VLM run here uses frozen inputs, hashed image evidence, cached responses, resumable ordering and a hard spending cap, so each matching result traces back to the exact crops and answers behind it. I also added an offline step that screens crops for size, blur and exposure before any call is made. On the development set it kept 13 of 14 images and dropped one tiny distractor crop. I haven't tested it against the matcher yet.

## What this does and doesn't show

- All geometry comes from ADT annotations. Real captures would add detection, depth and camera-pose error on top.
- It's one scene pair, and each benchmark cell has five seeds, so the rates are diagnostic rather than calibrated.
- The VLM results rest on six shared objects and ten comparison pairs. The 20 directional answers are correlated, not independent.
- The height-only success is fragile. Six inliers out of 28 sits right at the solver's minimum (6 inliers and a 20% inlier fraction), on objects chosen to be well spread. With all 267 objects I'd expect false pairs to swamp it, which is where a good visual check should earn its place.
- The crops went to the VLM in the Aria camera's native orientation, rotated 90° from upright. That's a plausible contributor to the low recall and the first thing I'll fix before rerunning.
- The provider changed its model between pilots, so the pilots aren't directly comparable.

## Advancing it further

1. Rerun the matching experiment with upright crops, then with multiple views per object.
2. Freeze the method and evaluate it once on the eight held-out objects.
3. Replace annotated objects with predicted ones. That brings in cross-frame association, uncertain depth and poses, and objects that actually move between visits.

I started out asking whether AI could help relate two observations of a physical space. The question I'm left with is sharper: can the recognition step keep enough reliable, well-spread evidence for the geometry to do its job? For a compound asset twin on the shop floor, at a client site or even in a simulated warehouse, that's what decides whether the digital model can be trusted where the physical one actually stands.

