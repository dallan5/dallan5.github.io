# Run 2 Training Report — Backplate → HDRI

**Run duration:** 27.8 hours (2 sessions, resumed after epoch 41 crash)
**Epochs completed:** 100/100
**Final model:** `models/generator_final.pth` (117 MB)
**Dataset:** 963 Polyhaven HDRIs × 8 views = 7,704 pairs (6,934 train / 770 val)
**Hardware:** M4 MacBook Air, Apple MPS backend

---

## TL;DR

The technique works in principle — the model learned a meaningful backplate-to-HDRI mapping. But the adversarial training collapsed mid-run, and the generator ended up optimizing almost entirely for L1 reconstruction, which produces **blurry, scene-appropriate but low-detail** outputs. This is the classic Pix2Pix failure mode for high-ambiguity tasks. To go further, the loss function needs to change — more epochs or more data won't fix it.

**Verdict on the proof of concept: YES, it converges.** The model clearly learned that backplates map to specific types of panoramas. It just can't synthesize sharp detail with the current loss setup.

---

## Loss Trajectories

| Metric            | Epoch 1 | Epoch 25 | Epoch 50 | Epoch 75 | Epoch 100 | Direction |
|-------------------|---------|----------|----------|----------|-----------|-----------|
| **Generator total** | 11.85 | 8.40     | 7.68     | 7.04     | 6.65      | ↓ good    |
| **L1**            | 0.227   | 0.157    | 0.139    | 0.125    | **0.117** | ↓ good    |
| **Discriminator** | 0.307   | 0.092    | 0.036    | 0.006    | **0.003** | ↓↓↓ bad   |
| **GAN**           | ~0.50   | ~0.58    | ~0.66    | ~0.74    | **0.79**  | ↑ bad     |

### Reading the numbers

**L1 improved 48%.** Steady, consistent reduction. The generator genuinely learned to produce more pixel-accurate outputs over time. No sign of overfitting (losses kept dropping smoothly, no val/train divergence visible in samples).

**Discriminator collapsed 100×.** From a healthy 0.31 in epoch 1 down to 0.003 at epoch 100. At this level, the discriminator is near-perfectly classifying real vs fake — which means its gradient signal to the generator is essentially useless. Instance noise and label smoothing slowed the collapse compared to Run 1 but didn't prevent it.

**GAN loss climbed 60%.** The mirror image of the discriminator collapse. The generator is progressively losing the adversarial game. By epoch 100, the generator is getting almost no useful sharpening signal.

**Generator dominated by L1.** With λ=50, the generator loss breakdown at epoch 100 is roughly: `0.79 + 50 × 0.117 = 0.79 + 5.85 ≈ 6.64`. L1 contributes ~88% of the total. The model is essentially a pure L1-regression network at this point.

---

## Visual Quality Assessment

From the validation grid comparison:

### What the model learned well

- **Scene type recognition.** Indoor scenes (pine_attic, vestibule, art_studio) produce warm interior tones. Outdoor scenes produce sky gradients. Night scenes stay dark. Real semantic understanding.
- **Overall lighting direction.** Sunrise/sunset scenes (oberer_kuhberg, small_harbor, hilly_terrain) correctly brighten one side of the panorama.
- **Color mood matching.** The generated HDRI's dominant colors closely match the ground truth's mood — warm/cool/saturated/muted.
- **Panoramic extension.** The model does produce content beyond the backplate FOV, not just a stretched copy. The abandoned_parking and sunflowers rows clearly show horizon lines extending in both directions.

### Where it falls short

- **Universal softness.** Every output looks like it's behind ground glass. No sharp edges, no crisp textures, no fine architectural detail.
- **Structure loss in complex scenes.** felsenlabyrinth (rock formations), art_studio, vestibule all lose their defining shapes and become color fields.
- **Sun position ambiguity.** The bright spot often appears in a "safe" central location rather than matching the actual sun. The model has no way to infer direction from a backplate alone.
- **Dark scenes give up.** pizzo_pernice and dry_orchard_meadow degenerate into nearly-solid color washes, suggesting the model couldn't find a stable L1 minimum and settled for the mean.

These failures all share a common cause: **L1 averaging over possibilities.** When many valid HDRIs exist for a given backplate, the L1-optimal solution is the mean of all of them, which is inherently blurry. Sharp details require an L1-beating signal, which the collapsed discriminator can no longer provide.

---

## Diagnosis — Why It Plateaued

Three interacting problems, in order of severity:

**1. Discriminator dominance (primary issue).** The stabilization tricks (instance noise, label smoothing) delayed but didn't prevent the classic GAN failure mode. By epoch 50 the discriminator was winning decisively; by epoch 75 it was essentially a perfect classifier. Once that happens, the generator's adversarial gradients become noise.

**2. Fundamental task ambiguity.** A single backplate doesn't contain enough information to uniquely determine the full HDRI. The sun could be behind you, the ground could extend in ways you can't see, nearby objects could be anything. L1 regression on an ambiguous problem gives you the conditional mean — which is the blurry average. No loss function that compares single outputs to single ground truths can escape this entirely.

**3. LR schedule effectively didn't fire.** Because the checkpoint doesn't save scheduler state, resuming at epoch 41 reset the decay schedule. The LR only started decaying in the final ~5 epochs (LR=0.000184 at epoch 100 instead of the intended ~0.000004). Fine-tuning phase was essentially skipped. Minor issue, but worth fixing.

---

## What WOULDN'T Help

Before discussing what to try next, ruling out some intuitive-but-wrong ideas:

- **More epochs.** L1 is asymptoting around 0.117. Another 100 epochs might get it to 0.10, but the blurriness is structural, not undertrained.
- **More training data.** Helpful but wouldn't change the fundamental averaging behavior of L1.
- **Bigger bottleneck.** Would encourage memorization without improving generalization. Skip connections already carry plenty of detail.
- **More layers.** Added capacity that the loss signal can't direct toward sharper outputs.
- **Higher resolution.** More pixels to average over blurrily. Scale up AFTER fixing the loss, not before.

---

## Recommended Next Steps (Run 3)

In priority order:

**1. Add perceptual loss (VGG).** Run both generated and real HDRIs through a pre-trained VGG-16, extract intermediate feature maps, compare with L1 or L2 at feature level. This directly penalizes blur because blurry images have different VGG activations than sharp ones. Doesn't rely on the discriminator. Total loss becomes `L_GAN + λ_L1·L1 + λ_VGG·L_perceptual`.

*Expected impact: biggest single quality jump. Industry-standard for image synthesis.*

**2. Drop λ_L1 from 50 to 20, add λ_VGG ≈ 10.** Rebalance so L1 no longer dominates. Gives perceptual and adversarial signals room to actually influence the generator.

**3. Fix the LR scheduler to save/restore properly.** Small change to `save_checkpoint` and `load_checkpoint`. Gets you proper fine-tuning in the second half of training.

**4. Optional: sun position conditioning.** Give the generator an extra input channel encoding where the brightest pixel is in the backplate. Removes some of the fundamental ambiguity, specifically for sun location.

### Not recommended yet

- Pre-trained encoder (ResNet backbone): diminishing returns after perceptual loss is added.
- Diffusion model: 10× compute cost, major rewrite. Only worth it if perceptual-loss GAN still plateaus.
- Scaling to 256×512: do it AFTER Run 3 confirms better loss setup works at 128×256.

---

## Files

- `models/generator_final.pth` — trained generator weights, 117 MB
- `checkpoints/latest.pth` — full training state (for resume)
- `logs/loss_history.json` — complete per-epoch metrics
- `logs/events.out.tfevents.*` — Tensorboard logs
- `outputs/training_samples/` — validation samples from every 3rd epoch

## Baseline for comparison

When Run 3 finishes, compare against these Run 2 numbers:

- Final L1: **0.117**
- Final Generator total: **6.65**
- Final Discriminator: **0.003** (anything > 0.05 is healthier)
- Final GAN: **0.79** (target: 0.3-0.5)
- Visual: scene-appropriate but universally blurry

Run 3 success criteria: perceptually sharper outputs, D loss staying above 0.05, GAN loss staying below 0.6. L1 may end higher than 0.117 and that's fine — lower L1 doesn't mean better images here.
