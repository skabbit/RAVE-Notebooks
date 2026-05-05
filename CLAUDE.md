# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A collection of standalone Jupyter notebooks that bootstrap training of neural-audio models (RAVE, VSCHAOS2, MSPrior, AFTER, RAVE-Latent Diffusion, RAVE-Latent Diffusion Flex'ed, Stable Audio Tools, PLaTune) on **Kaggle** or **Google Colab**. There is no library code, no tests, and no build step — each `.ipynb` is a self-contained recipe that the user imports into a hosted notebook environment and runs cell-by-cell.

File naming is load-bearing: `<MODEL>_Training_Template--<Kaggle|Colab>.ipynb`. The platform suffix determines which conventions apply (paths, secrets, drive mounting). Keep this naming when adding new notebooks.

## The shared notebook template

Every notebook follows the same four-section skeleton. When editing or adding notebooks, mirror this structure rather than inventing new layouts — users move between models and rely on the consistency.

1. **Setup runtime** — install Miniconda into a scratch dir, then `pip install` the model package (or `git clone` + `pip install -e .` for repos installed from source). Often followed by a force-reinstall of pinned torch/torchvision/torchaudio and a `numpy<2` pin to fix CUDA/ABI mismatches against Kaggle's preinstalled stack.
2. **Preprocess dataset** — one-time per dataset; writes an LMDB or latent dump to the working dir. Marked clearly so users know to disable it on resume.
3. **Train** (initial + resume) — resume is a *separate cell* with the `cp -r /kaggle/input/<prior-output>/* /kaggle/working` line and a `--ckpt` / `--restart` flag added. Initial and resume are not collapsed into one cell on purpose.
4. **Export** — produces a `.ts` TorchScript (or equivalent) for downstream use (nn~, Max/PD, etc.).

### Platform path conventions (do not mix)

| | Kaggle | Colab |
|---|---|---|
| Scratch / install root | `/kaggle/temp` | `/content/download`, `/content/miniconda` |
| Working dir (persists as notebook output) | `/kaggle/working` | user's mounted Drive folder |
| Inputs (datasets, prior runs, pretrained models) | `/kaggle/input/<dataset>` | user's mounted Drive folder |
| Miniconda binary | `/kaggle/temp/miniconda/bin/<tool>` | `/content/miniconda/bin/<tool>` |
| Secrets (e.g. wandb) | `kaggle_secrets.UserSecretsClient` | `google.colab` widgets / env |

The Miniconda installer line itself is also platform-versioned — pick the Python version the model package requires (e.g. py39 for RAVE 2.2.2, py310 for SAT, py311 for AFTER, py312 for PLaTune). Always invoke tools via the explicit `/kaggle/temp/miniconda/bin/...` path rather than relying on `PATH`, because Kaggle's base kernel has its own Python that will otherwise win.

### Resume-across-sessions pattern (Kaggle-specific)

Kaggle kernels die at 12h. The standard escape hatch documented across these notebooks: save the prior run's output as a Kaggle Dataset, attach it as input on the next run, `cp -r /kaggle/input/<earlier-output>/* /kaggle/working`, then re-run training with the model's resume flag (`--ckpt`, `--restart`, etc. — varies per model). When editing resume cells, keep them disabled-by-default (commented `cp` line) and reference the same naming as the initial-train cell.

## Common gotchas baked into existing notebooks

When updating dependency pins, preserve the *intent* of these workarounds — don't strip them just because they look ugly:

- **`numpy<2`** — pinned across most notebooks because the model packages were built against NumPy 1.x ABI.
- **`torch` force-reinstall with `--index-url https://download.pytorch.org/whl/cu118`** — Kaggle's preinstalled torch can mismatch the CUDA driver the model package expects; the explicit reinstall realigns them.
- **`setuptools<81`** (PLaTune) — newer setuptools removes `pkg_resources` shims that some transitive deps (e.g. `pretty_midi`) still import.
- **music2latent `weights_only=False` patch** (PLaTune) — PyTorch ≥2.6 flipped the `torch.load` default; the bundled checkpoint trips it. The notebook patches the installed file in place.
- **`nn_tilde` stripped from PLaTune requirements** — fails to build on Kaggle and is only needed for nn~/Max/PD export, which Kaggle isn't doing.
- **MSPrior fork's `train.py`** — upstream MSPrior caps at 1000 epochs; the notebook downloads `devstermarts/msprior`'s `train.py` to allow `--epochs N` (or `-1` for unbounded).
- **AFTER preprocess vs export need *different* RAVE exports** — preprocess/train wants the autoencoder exported **without** `--streaming`; export of the AFTER model wants the **streaming** version. Don't conflate these.

## Commit messages

Do **not** append a `Co-Authored-By: Claude ...` trailer to commits in this repo. Use a plain commit message matching the existing style (`Create <File>`, `Update <File>`, or a short description like `"numpy<2" fix for RAVE models training`).

## Editing notebooks

- The `.ipynb` files are JSON; edit cell sources with the notebook editing tool rather than hand-rewriting JSON when possible.
- `.gitignore` excludes `*checkpoint.ipynb` (Jupyter autosave files) — don't commit those.
- `.gitattributes` enforces LF line endings repo-wide.
- There are no tests, linters, or CI. The only meaningful "validation" is running the notebook end-to-end on the target platform — which only the user can do (Kaggle/Colab account required). When asked to verify a change, say so explicitly rather than claiming success from static review.
- Each notebook carries a "Last updated: DD.MM.YYYY" line in its header markdown — bump it when making non-trivial changes.

## Upstream projects (for context when debugging notebook breakage)

- RAVE — https://github.com/acids-ircam/RAVE (notebooks pin to ≤2.2.2)
- VSCHAOS2 — https://github.com/acids-ircam/vschaos2
- MSPrior — https://github.com/caillonantoine/msprior (+ devstermarts fork for >1000 epoch training)
- AFTER — https://github.com/acids-ircam/AFTER (notebook uses `skabbit/AFTER` branch `local-improvements`, cloned locally at `/Users/skabbit/Projects/AFTER` — fix AFTER bugs there and push to that branch rather than patching the notebook)
- RAVE-Latent Diffusion — https://github.com/moiseshorta/RAVE-Latent-Diffusion
- RAVE-Latent Diffusion Flex'ed — https://github.com/devstermarts/RAVE-Latent-Diffusion-Flex-ed (maintained by the notebook author)
- Stable Audio Tools — https://github.com/Stability-AI/stable-audio-tools
- PLaTune — https://github.com/acids-ircam/platune (notebook uses skabbit fork)
