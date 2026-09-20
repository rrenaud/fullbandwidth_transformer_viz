# Full-Bandwidth Transformer — interactive visualizations

Live site: **https://rrenaud.github.io/fullbandwidth_transformer_viz/**

Interactive explainers built around the ideas in
[Full-Bandwidth Transformer](https://arxiv.org/abs/2608.08888) (arXiv:2608.08888):
autoregressive transformers get full bandwidth *across tokens* (dense attention lets
every position read every earlier position), but only a narrow, discrete channel
*across depth* between decoding steps — the top-layer hidden state at step `t` is
squeezed down to a single sampled token before it can influence step `t+1`.

## Why HTML/JS instead of Manim

These are built as self-contained, interactive HTML pages (vanilla JS + inline SVG),
not Manim renders. Manim produces polished but non-interactive video — great for a
fixed narrative, but this material is about *compute order over time*, which is best
taught by letting the reader step, scrub, and pause it themselves rather than watch
it once. The visual language borrows from 3Blue1Brown-style math explainers (clean
diagrams, restrained color, animated transitions) without the video pipeline.
No build step, no dependencies — open any `index.html` directly in a browser.

## Visualizations

### 01 — What can reach what
[`viz/01-training-vs-inference/index.html`](viz/01-training-vs-inference/index.html)

The core distinction the paper's argument turns on, laid out as a single grid of
hidden states `h[t, ℓ]` (token position × layer):

- **Training** sweeps one *layer* at a time across every token in the sequence at
  once (teacher forcing). This is legal only because of the paper's *depth-frozen*
  constraint: no cell may read a deeper layer's output from an earlier position
  that hasn't been computed yet.
- **Naive autoregressive inference** sweeps one *token* at a time through every
  layer, imitating that same constraint — even though inference is already
  sequential over tokens, so the constraint "buys nothing" there. Each finished
  token's top-layer hidden state is discarded; only the sampled token id
  (≤ log₂|V| bits) crosses to the next step.
- **Full-bandwidth (latent feedback)** — the paper's fix — fuses that discarded
  top-layer state back in via a gated linear unit, widening the channel between
  decoding steps at no extra sequential cost.

Hover or tab to any cell — in any of the three panels at once — to freeze time
at the instant it fires and see exactly which earlier states it can read (blue),
which it reads directly (solid blue), and which are already sitting there
computed but architecturally out of reach (red), with a live count of how much
reachable information is going unused right now.

Live: https://rrenaud.github.io/fullbandwidth_transformer_viz/viz/01-training-vs-inference/
(also on [Claude Artifacts](https://claude.ai/artifact/56qd3QqQL93wk6tqLNJ97d))

## Roadmap ideas

- Animate real attention patterns / KV-cache growth alongside the depth axis.
- A version driven by activations from an actual small model checkpoint instead of
  illustrative placeholder values.
- A piece on the multi-pass training regime the paper uses to make the learned
  feedback map contract toward a fixed point (their Figure 3 instability story).
