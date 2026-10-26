# Assignment 2 Report

*Delete this italic guidance as you fill in each section. You'll be asked to
defend any of this without your code in front of you. Write only what you
can actually explain.*

- **Name**:
- **Student ID**:
- **Email**:
- **Group**: [BBADBA 5A | BBADBA 5B | PPLEDBA 5A | BDBA 3A]

## Business problem, and the cost of being wrong

*(Draft written with AI assistance from my decisions; to be reviewed and rewritten.)*

**The decision.** A fruit packing house sorts apples, oranges and bananas on a conveyor before packing them into retail packs (bags and trays) for a supermarket chain. Today, line workers look at each fruit and pull out the rotten ones. The model looks at a camera image of each fruit and makes the same call. Every fruit gets one of three outcomes:

- **pass**: packed for the supermarket;
- **fail**: an air ejector diverts it to the industrial stream (juice, purée, animal feed);
- **review**: diverted to an inspection station, where a worker decides.

The unit of decision is **one fruit**. The **positive class is "rotten"** (`rotten_apples` + `rotten_banana` + `rotten_oranges`).

**Who acts on it.** The ejector acts automatically on pass and fail. The line's quality operator handles the review lane. The packing house carries the cost of both kinds of error: it loses the value of fruit it diverts, and it pays the supermarket's claims for rotten fruit that gets through.

**Cost of each outcome (per fruit)**

| Outcome | Cost | Why |
|---|---:|---|
| False positive: sound fruit sent to industry | **€0.20** | Golden apples leave the packing house at about €1.01/kg, and navel oranges at €0.80/kg (MAPA weekly price bulletin, week 6/2025). One piece weighs about 0.2 kg, so it is worth about €0.20 as fresh fruit. Industry pays only a few cents per kilo, and the washing and grading already done is lost. |
| False negative: rotten fruit packed and shipped | **€5.00** | EU marketing standard for apples, Class I: at most 1% of a lot may be affected by decay. A pallet holds about 800 kg (about 4,000 apples) worth about €800, so it breaks tolerance with about 40 rotten fruit. A non-compliant load is typically rejected or taken at a 20–30% discount, about €200. €200 / 40 ≈ **€5 per rotten fruit that slips through.** |
| True positive / true negative | €0 | Costs are measured against a perfect sorter. A rotten fruit sent to industry was worth nothing as fresh fruit anyway, and a sound fruit that passes is sold normally. |

**The asymmetry: a false negative costs 25 times a false positive.** Throwing away a good apple loses its margin. Shipping a rotten one puts a whole pallet at risk and strains the relationship with the retailer. The model should therefore err towards rejecting. A simple check makes this concrete: with these costs, rejecting a fruit pays off as soon as the probability that it is rotten exceeds 0.20 / (0.20 + 5.00) ≈ **3.8%**. That is a low bar, so recall on the rotten class matters far more than precision.

**Base rate: 5% of incoming fruit is rotten.** Commercial apple storage loses 4–12% of fruit, and 60–80% of those losses are fungal decay, which gives about 2–10% rotten fruit. I take 5% as the midpoint. The dataset's own share, 54% of test boxes, is not a base rate. Even at 5%, unsorted fruit would break the 1% tolerance five times over, which is why sorting is needed at all.

**Assumptions that could change the answer** *(the break-even values will be filled in from the sweep in the cost section)*:

- I assumed **FN = €5** because a pallet breaks the 1% tolerance at about 40 rotten fruit, at a cost of about €200. The chosen threshold changes if FN falls below €Z.
- I assumed a **base rate of 5%** because of the storage-loss studies above. The decision changes if it falls below Z%.
- I assumed **one cost for all three fruit types**. Bananas and oranges are worth roughly the same per piece as apples, and the dataset is dominated by apples.

Sources: [MAPA weekly fruit & vegetable price bulletin, week 6/2025](https://www.mapa.gob.es/en/ganaderia/estadisticas/62025boletinsemanaldepreciosdefyh3-10febrero_tcm38-701364.pdf); [EU/UNECE marketing standard for apples (Reg. 543/2011, Annex I)](https://assets.publishing.service.gov.uk/government/uploads/system/uploads/attachment_data/file/869232/marketing-standard-apples.pdf); [Clarifresh, retailer rejections at a 20–30% discount](https://clarifresh.com/blog/repacking-fresh-produce/); [HortScience 56(5): postharvest losses of apples in commercial storage](https://journals.ashs.org/view/journals/hortsci/56/5/article-p608.xml).

## Dataset

- **Workspace / project / version**: `project-irggm` / `-w4yds` / version 1 (2025-03-18), used as published (not forked).
- **Link**: https://universe.roboflow.com/project-irggm/-w4yds/dataset/1
- **Task / licence**: object detection, Public Domain. Export format `yolov11`, 640×640 (stretch), no Roboflow augmentations.
- **Classes (6)**: `fresh_apples`, `fresh_banana`, `fresh_oranges`, `rotten_apples`, `rotten_banana`, `rotten_oranges`.
- **What one image is**: a photo of one or more pieces of fruit; each box is one fruit labelled with its type and state. One box is the unit we decide on.

**Why it serves the decision** *(draft, AI-assisted)*.

The labels encode exactly the call the line makes: which fruit (box) and whether it is rotten (class). The model therefore outputs one decision per fruit, not one per image, and that matches how the ejector acts.

It covers the three fruit types a mixed packing line handles, with both states for each. It also includes images where fresh and rotten fruit sit side by side, as they do on a belt.

What it does not match is the camera: none of the images come from a conveyor. The next section shows what that limits.

## What the dataset's composition implies

All figures computed from the labels by `dataset_stats.py` (outputs in `artifacts/dataset/`).

**Splits**

| Split | Images | Instances | Empty images | Multi-class images | Images with an external-augmentation prefix |
|---|---:|---:|---:|---:|---:|
| train | 16,609 (69.5%) | 26,869 | 964 | 232 | 8,502 (51%) |
| valid | 4,852 (20.3%) | 8,414 | 277 | 63 | 2,400 (49%) |
| test | 2,443 (10.2%) | 4,268 | 140 | 34 | 1,186 (49%) |

**Instances per class**

| Class | Train (published) | Valid (published) | Test | Share of all boxes | Train (cleaned) | Valid (cleaned) |
|---|---:|---:|---:|---:|---:|---:|
| fresh_apples | 5,209 | 2,062 | 1,204 | 21.4% | 4,332 | 1,846 |
| fresh_banana | 2,788 | 781 | 425 | 10.1% | 1,448 | 421 |
| fresh_oranges | 3,072 | 976 | 336 | 11.1% | 2,224 | 720 |
| rotten_apples | 10,408 | 3,101 | 1,546 | 38.1% | 8,956 | 2,703 |
| rotten_banana | 2,831 | 777 | 421 | 10.2% | 1,522 | 401 |
| rotten_oranges | 2,561 | 717 | 336 | 9.1% | 1,481 | 394 |

"Cleaned" = after removing every train/valid image that is a copy of a test image (see leakage below). The test split is used unchanged.

![Instances per class](artifacts/dataset/instances_per_class.png)

**Object size relative to the image**

| Class | Median box area | p25 – p75 | Small (<32² px) | Medium | Large (>96² px) | Boxes < 8×8 px |
|---|---:|---:|---:|---:|---:|---:|
| fresh_apples | 8.3% | 2.3 – 73.0% | 3.7% | 21.2% | 75.1% | 0 |
| fresh_banana | 45.9% | 18.7 – 81.1% | 0.1% | 0.8% | 99.1% | 3 |
| fresh_oranges | 20.0% | 5.4 – 72.7% | 0.2% | 11.4% | 88.4% | 7 |
| rotten_apples | 9.4% | 3.6 – 38.7% | 0.1% | 11.6% | 88.3% | 15 |
| rotten_banana | 52.4% | 29.2 – 68.4% | 0.2% | 0.1% | 99.7% | 8 |
| rotten_oranges | 60.7% | 11.9 – 77.6% | 0.1% | 4.8% | 95.1% | 3 |

![Box size per class](artifacts/dataset/box_size_per_class.png)

**Train/test leakage**

| Check | Test images |
|---|---:|
| Same original photo in train/valid, by filename (augmentation prefix and `.rf.<hash>` stripped) | 1,383 |
| Near-identical image in train/valid, by perceptual hash (dHash, ≤ 3 of 64 bits) | 556 |
| … of which missed by the filename check (burst photos, renamed copies) | 184 |
| **Leaked in total** | **1,567 of 2,443 (64%)** |

By source: all 1,360 `Screen-Shot` test images leak, as do 142 of 305 `IMG_*` phone photos, while none of the 46 `YYYYMMDD_HHMMSS` phone photos do.

**Fix applied.** The training pipeline drops from train and valid every image whose base name matches a test image or whose dHash is within 3 bits of one: 5,938 train and 1,689 valid images (list in `artifacts/dataset/excluded_train_val.csv`). That leaves 10,671 train / 3,163 valid / 2,443 test images. The download is the published version, unchanged; the filter is deterministic and runs inside `train.py`, so anyone re-running the pipeline gets the same split.

**What this implies** *(draft written with AI assistance, to be reviewed and rewritten)*

1. **The official test split overstates performance.** About two thirds of the test images are the same photo as a train image, either rotated, flipped or noised, or taken a second later in a burst. The screenshot source leaks completely: this suggests Roboflow's split was made *after* someone else had augmented the images, so copies of one photo landed on both sides. A model can score well on those images by recognising the photo, not by judging rot, and a leaked test underestimates false negatives, which would push the cost-optimal threshold to an over-optimistic place. I considered two fixes: evaluating only on the 876 test images with no copy in train/valid, or keeping the whole test split and removing the copies from train/valid instead. I chose the second. It keeps 336–425 test boxes for every banana and orange class, compared with 57–133 in the de-duplicated test, where recall would carry a ±10–13 point uncertainty. The price is training on 10,671 images instead of 16,609, but most of what was removed are rotated or flipped copies of the same photos, which add little new information.
2. **The test split still contains copies of itself.** An original and its rotated versions can all sit in test. That does not bias the metrics, but the effective sample size is smaller than 2,443 images, so per-class intervals are wider than the raw counts suggest. Train and valid also share copies with each other. This only affects early stopping and model selection, and the test split is not affected.
3. **Apples dominate.** Apples make up 64% of the test boxes, and `rotten_apples` alone is 36%. The pooled "rotten" result will lean on apples, so I report each class separately before pooling.
4. **The class mix is not the production base rate.** Rotten fruit makes up 57% of the boxes overall and 54% of the test. On a packing line it is a small fraction. Precision measured here will be far higher than in production, so the cost analysis uses the test's TPR and FPR together with an assumed base rate, not the test's raw precision.
5. **The images do not look like a conveyor belt.** The dataset mixes screenshots of web images (57% of train), phone photos held in a hand or on a table, and studio shots on white backgrounds. Most fruit is large: the median box covers 8–60% of the image, and fewer than 4% of boxes are "small" in COCO terms. On a line camera every fruit would have a similar, smaller size, with a consistent background and lighting. The test split cannot tell us how the model behaves under that shift. That needs a pilot on real line images.
6. **Labels are mostly clean but not perfect.** There are 36 boxes under 8×8 px, which are almost certainly annotation errors, and 1,381 images with no boxes (background-only negatives). Their effect on the metrics is negligible, so I leave the dataset as published.
7. **Some images mix fresh and rotten fruit.** Only 30 test images do (226 in train). Since the decision is made per fruit, these images are useful: they show whether the model can separate neighbouring fruit. But they are too few to measure that reliably.

## Finetuning and baselines

*Which model and task (detection or classification) and why. A results table on
the same test split: your finetuned model, the pretrained COCO model zero-shot,
and a trivial baseline. What changed, and whether the gap is as large as you
expected.*

## Per-class evaluation and failure modes

*Per-class AP50, or per-class precision/recall and the confusion matrix.
Which class carries the headline number, which one is dragging, and what the
model actually confuses with what. Show where it breaks, not just that it
works.*

## Cost matrix and operating point

*Your euro cost for each kind of error and where each figure came from. The
base rate you assumed. The cost-against-threshold sweep, and the operating
point you chose. State the assumption that would most change your answer if it
were wrong.*

## What a person still has to check

*Your two thresholds, and what share of cases they send to a person. How many
that is per day, how long one takes to check, and how many people that needs.
Is that realistic? Then your design priced against the three alternatives:
doing nothing, having a person check everything, and letting the model decide
on its own.*

## Limitations & next steps

*Real limitations you found, and concretely how you'd address each one with
more time or data. Not generic hedging.*

## Generative AI use disclosure

*Per the syllabus AI Policy: what you used and how, or "no AI content used."*
