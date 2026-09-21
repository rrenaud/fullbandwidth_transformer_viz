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

### 02 — Closing the loop

Full-bandwidth *training*, unrolled. Teacher forcing runs every position at once, but
position `t` wants `h[t-1]`, the previous position's **top-layer** latent — which does
not exist until the pass is over. So each pass is run against the *previous* pass's
latents, and the pass is run again. The passes climb the figure — **pass 1 is at the
foot** — so a pass's latent row sits directly beneath the next pass's input row, and the
handoff is one short step up and one column to the right, with nothing to route around.

Three things fall out of that, and the page animates all three:

- **The loop closes one token per pass.** Position 1 needs no latent, so it is exact
  from pass 1; its latent makes position 2 exact in pass 2, and so on. The exact prefix
  advances one token per pass — a visible staircase up the stack.
- **Past that wavefront the error only contracts**, which is why a handful of passes
  stands in for the `T` it would take to converge exactly. The page shows this as a
  stale tint that fades pass by pass, and shows no number for it: which positions are
  exact follows from the architecture, but any rate of contraction would be invented,
  and an invented rate printed to three decimals reads as a measurement.
- **Inference pays none of it.** Decoding is already sequential, so `h[t-1]` is sitting
  there when step `t` starts. A `k`-pass batch costs roughly `k`× ordinary teacher
  forcing, but the passes are *scheduled* — the paper's runs are 75% single-pass, 22%
  two-pass, 3% three-pass, introduced midway through training — so the average overhead
  is nearer 1.3×, and the three-pass batch drawn here is the rare one.
- **Every pass is scored**, per the paper's objective: the first term is the ordinary
  single-pass loss on plain embeddings, and the feedback passes are averaged and weighted
  by λ, giving `L₁ + λ(L₂ + L₃)/2` at three passes. So the model is trained across the
  whole range of latent quality it will be handed, from no feedback at all in pass 1 to
  nearly converged in pass 3.
- **Nothing is detached.** "We do not detach the gradient, so the loss from later passes
  backpropagates into earlier passes' latent states." The backward pass runs the whole
  height of the figure — it is one graph, not three.

Two things the diagram used to leave implicit are now drawn. The sequence enters at the
foot and the targets leave at the head; every latent row carries a tick up into the loss,
so a pass both hands its latents on *and* gets scored, and the head of the figure spells
out the targets all three are scored against. And each wire lands on a ⊕ rather than an
arrowhead, with a side inset opening up one input cell — and note which way round the
paper puts it: `e_t ⊗ h_{t-1} = W^U h_{t-1} ⊙ σ(W^G e_t)`, so the latent is the *value*
and the token embedding enters only as a multiplicative gate. It is a product, not a sum,
which is why pass 1 runs as the plain single-pass objective rather than by feeding a zero
latent through the gate — that would zero the input outright.

The animation runs both halves of a training step. The forward half fills pass 1, then
2, then 3, one layer at a time. The backward half then walks back down: because nothing
is detached, it is a single graph over the whole figure, so the gradient descends every
layer of pass 3, crosses the handoff wires into pass 2's latent row, descends again, and
leaves at the foot. A green trail in the gutter marks the rows it has come through, so a
paused figure shows how far it reached rather than only where it is.

One claim on the page is mine rather than the paper's: the exact prefix. It follows from
the update rule (position 1 is fed plain `e_1` in every pass, so each pass pins one more
position), but the paper presents multi-pass as an approximation and does not analyse it
that way. The page says so where it makes the claim.

Each wire is coloured by the *source* position it carries, and the same colour caps the
latent it leaves and labels the input it joins — so the whole ribbon is visibly the same
ramp, slid one column to the right. (Nine smooth steps can't carry identity by colour
alone, so the `z` label at the receiving end does that; the ramp carries the ordering.)

The page itself is deliberately sparse: it is built to be narrated, so the explanation
lives here rather than on screen. What stays in the figure is what the picture cannot say
by itself — the objective, the axis labels, and the two notes about what is *absent*
(position 1 has no latent; pass 1 has nothing to gate with), plus each pass's share of
training batches — 75 / 22 / 3, which sum to 100 and so read as the partition they are.

Hover or tab any top-row cell to follow one latent around the loop into the position it
lands in. The figure shows three passes — enough for the staircase to establish itself
without the page turning into a scroll.

## Roadmap ideas

- Animate real attention patterns / KV-cache growth alongside the depth axis.
- A version driven by activations from an actual small model checkpoint instead of
  illustrative placeholder values.
- The instability story from the paper's Figure 3: what happens when the learned
  feedback map does *not* contract. Worth doing with real residual measurements, or
  with the contraction factor as a control the reader drives — not as a fixed constant
  presented as data.
