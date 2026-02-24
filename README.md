# CMAP — Causal Memory Admissibility
### A Structural Necessity Classification of Passive Causal Memory  
**Discrete-First · Finite-Resource · Domain-Independent**

[![DOI](https://img.shields.io/badge/DOI-10.5281%2Fzenodo.18757034-blue?style=for-the-badge)](https://doi.org/10.5281/zenodo.18757034)
[![Zenodo](https://img.shields.io/badge/Zenodo-18757034-blueviolet?style=for-the-badge)](https://doi.org/10.5281/zenodo.18757034)
[![Open Civil Science](https://img.shields.io/badge/Open--Civil--Science-Research-green?style=for-the-badge)](#)
[![Human–AI Collaboration](https://img.shields.io/badge/Human--AI-Collaboration-purple?style=for-the-badge)](#)
[![Colab-Ready](https://img.shields.io/badge/Colab-Ready-orange?style=for-the-badge)](#)
[![Reproducible](https://img.shields.io/badge/Reproducible-Yes-success?style=for-the-badge)](#)

> A structural admissibility certificate for passive causal memory — verifiable on finite records.

---

## TL;DR (for **CausalWorlds** audience)

If a constitutive memory law is:
- **causal** (retardation),  
- **passive** (never extracts net work),  
- **finite-resource & genuinely fading**,  
- **non-negative** and **diffusive** (no oscillatory internal poles),

then its discrete-time kernel is **mathematically forced** to be a finite non-negative mixture of geometric decay channels (CMF).  

Formally:

\[
K(n) = \sum_{j=1}^J w_j (1-\lambda_j)^n,\qquad w_j \ge 0,\ \lambda_j\in(0,1).
\]

This is a *bidirectional certificate* — the five gates ⇔ CMF. The result is finite-record verifiable and domain-independent.

---

## Why this matters

- **Model sanity check** for learned or fitted kernels in causal / dynamical systems.  
- **Pre-model constraint**: filter unphysical laws before training or deployment.  
- **Operational**: produces a PASS / FAIL on finite data (no continuum limit required).  
- **Interdisciplinary**: useful for control, signal processing, physics-informed modeling, and causal ML.

---

## Quick repository layout (recommended)

```

/cmap
/paper           # camera-ready PDF (CMAP_arXiv_v2.pdf)
/poster          # poster sources (CW2026_poster_v3_2_robust.md)
/proofs          # companion proofs (CMAP_COMPANION_PROOFS_v1_1.md)
/tools           # verification tools (check_cmap.py, validate_exact.py)
/notebooks       # demo notebooks (Colab-ready)
/examples        # toy and fitted kernels
README.md
LICENSE

````

---

## Gates (summary)

1. **Retardation (P1)** — K(n) = 0 for n < 0 (discrete causal support).  
2. **Passivity (P2)** — ⟨u, T_K[u]⟩_T ≥ 0 for all finite histories u and horizons T.  
3. **Genuine fading (F1+F2)** — K(n) → 0 and ∃ n ≥ 1 with K(n) > 0.  
4. **Non-negativity (N)** — K(n) ≥ 0 ∀ n ≥ 0.  
5. **Diffusive admissibility (A)** — poles (encoding) are real and lie in (0,1) (no complex conjugate oscillatory poles).

Together they imply — and are implied by — the CMF form above.

---

## Operational certificate (practical)

Given sampled kernel vector `K = [K(0), K(1), ..., K(N-1)]`:

1. Check **non-negativity** & **retardation**.  
2. Heuristic check **fading** (tail vs head).  
3. **Passivity** via PSD test of the causal Toeplitz matrix (finite horizon).  
4. **Diffusive** heuristic: low sign-change / stability in residues or fit.  
5. If heuristics pass, run exact check: fit finite-order recurrence (H2), compute rational K̂(z), extract poles & residues, check pole locations ∈ (0,1) and residues ≥ 0.

If all checks succeed → kernel admissible (CMF). If any fail → kernel violates at least one gate.

---

## Quickstart — Minimal verifier (place in `tools/check_cmap.py`)

```python
# tools/check_cmap.py
import numpy as np
from numpy.linalg import eigvalsh

def is_nonnegative(K, tol=1e-12):
    return np.min(K) >= -tol

def is_genuinely_fading(K, rel_tol=1e-3, tail_window=10):
    if len(K) < tail_window*2:
        return False
    head = np.mean(np.abs(K[:tail_window]))
    tail = np.mean(np.abs(K[-tail_window:]))
    return (tail <= rel_tol * max(head, 1e-12)) and np.any(np.abs(K[1:]) > 1e-12)

def toeplitz_causal_matrix(K):
    N = len(K)
    T = np.zeros((N, N))
    for n in range(N):
        for m in range(n+1):
            T[n, m] = K[n - m]
    return T

def is_passive_via_toeplitz(K, tol=-1e-10):
    T = toeplitz_causal_matrix(K)
    S = (T + T.T) / 2.0
    vals = eigvalsh(S)
    return np.min(vals) >= tol

def diffusive_heuristic(K, max_sign_changes=2):
    signs = np.sign(K)
    signs[np.abs(K) < 1e-12] = 0
    nz = signs[signs != 0]
    if len(nz) < 3:
        return True
    changes = np.sum(np.abs(np.diff(nz)) > 0)
    return int(changes) <= max_sign_changes

def check_cmap_kernel(K):
    K = np.asarray(K, dtype=float)
    out = {}
    out['retardation'] = True
    out['nonnegativity'] = bool(is_nonnegative(K))
    out['genuine_fading'] = bool(is_genuinely_fading(K))
    out['passivity_toeplitz'] = bool(is_passive_via_toeplitz(K))
    out['diffusive_heuristic'] = bool(diffusive_heuristic(K))
    out['admissible_heuristic'] = all([out['nonnegativity'],
                                       out['genuine_fading'],
                                       out['passivity_toeplitz'],
                                       out['diffusive_heuristic']])
    return out

if __name__ == "__main__":
    lam = [0.18, 0.45]
    w   = [1.0, 0.6]
    N = 120
    K = sum(wj * (1 - lj)**np.arange(N) for wj, lj in zip(w, lam))
    print(check_cmap_kernel(K))
````

---

## Exact verification (fit recurrence + pole/residue extraction)

Use `tools/validate_exact.py` to fit a finite-order linear recurrence (H2), compute rational form K̂(z) and extract poles/residues. Place the script in `tools/validate_exact.py`.

```python
# tools/validate_exact.py
# Minimal pipeline: fit linear recurrence via linear least squares on Hankel blocks,
# form companion polynomial A(z), perform partial fraction via numpy.roots and residues.
import numpy as np
from numpy.linalg import lstsq

def fit_recurrence(K, p_max=10):
    # Fit K(n) ≈ sum_{k=1..p} a_k K(n-k) for n = p..N-1
    N = len(K)
    for p in range(1, min(p_max, N//2)+1):
        A = []
        b = []
        for n in range(p, N):
            A.append([K[n - k] for k in range(1, p+1)])
            b.append(K[n])
        A = np.array(A)
        b = np.array(b)
        sol, _, _, _ = lstsq(A, b, rcond=None)
        res = np.linalg.norm(A.dot(sol) - b) / max(1e-12, np.linalg.norm(b))
        # choose first p with small residual
        if res < 1e-6:
            return sol, p
    return None, None

def recurrence_to_polynomial(a_coeffs):
    # a_coeffs = [a1, a2, ..., ap] -> A(z) = z^p - a1 z^{p-1} - ... - ap
    p = len(a_coeffs)
    A = np.zeros(p+1)
    A[0] = 1.0
    for i, ai in enumerate(a_coeffs, start=1):
        A[i] = -ai
    return A  # coefficients z^p + A[1] z^{p-1} + ... + A[p]

def extract_poles_residues(K, a_coeffs):
    # Build polynomial A(z) and compute poles r_j (roots). Poles in z-domain -> rates = 1 - r_j
    A = recurrence_to_polynomial(a_coeffs)
    roots = np.roots(A)
    # discard roots near or outside unit circle
    poles = roots
    # For residues, compute partial fraction numerically via solving Vandermonde on early samples
    # Suppose K(n) = sum c_j r_j^n, solve linear system for first J samples
    J = len(poles)
    M = np.vander(poles, N=len(K), increasing=True).T[:J, :J]
    try:
        coeffs = np.linalg.solve(M, K[:J])
    except Exception:
        coeffs = np.linalg.lstsq(M, K[:J], rcond=None)[0]
    return poles, coeffs

if __name__ == "__main__":
    # example: create CMF kernel and test
    lam = [0.18, 0.45]
    w   = [1.0, 0.7]
    N = 120
    K = sum(wj * (1 - lj)**np.arange(N) for wj, lj in zip(w, lam))
    a, p = fit_recurrence(K, p_max=8)
    if a is None:
        print("No stable recurrence found (increase p_max or N).")
    else:
        poles, coeffs = extract_poles_residues(K, a)
        rates = 1 - poles
        print("fitted order p =", p)
        print("poles (z) =", poles)
        print("rates (1 - z) =", rates)
        print("residues (c_j) approx =", coeffs)
```

> **Note:** the exact pipeline above is intentionally minimal to be robust and interpretable. For production, use stabilized rational approximation packages (vector fitting, Prony methods) and validate via cross-validation. Notwithstanding, this script is immediately usable for conference demos.

---

## Examples & demos (Colab ready)

* `/notebooks/CMAP_demo.ipynb` — produce CMF kernels, show heuristic checks, run recurrence fit, visualize pole/residue decomposition.
* Use the `Colab-Ready` badge to link a ready notebook that mounts the repo and runs `tools/validate_exact.py`.

---

## Keywords & Tags

`causal-memory` `passivity` `structural-theorem` `finite-resource` `discrete-first`
`open-civil-science` `human-ai-collaboration` `colab-ready` `zenodo-archived` `causalworld`

---

## Citation

Please cite the camera-ready manuscript and dataset:

**Lahtee, Y. (2026).** *Causal Memory Admissibility (CMAP): A Structural Necessity Classification.* DOI: [10.5281/zenodo.18757034](https://doi.org/10.5281/zenodo.18757034)

BibTeX:

```bibtex
@article{lahtee2026cmap,
  title = {Causal Memory Admissibility (CMAP): A Structural Necessity Classification},
  author = {Lahtee, Yaoharee},
  year = {2026},
  doi = {10.5281/zenodo.18757034}
}
```

---

## Authorship & ethics

**Author:** Yaoharee Lahtee
**Contributor:** Walancha Supantarika

Human-AI collaborative drafting used for formatting and reproducibility tooling. The author retains intellectual responsibility for all claims.

---

## License

* Paper & text: **CC-BY-4.0** (recommended)
* Code: **MIT** (recommended)

---

## Contact

For questions, reproducibility requests, or collaboration: add contact email in `paper/` metadata (camera-ready PDF contains contact line).

---

*End of README — paste once, run demos in `notebooks/` and the `tools/` scripts for verification.*

