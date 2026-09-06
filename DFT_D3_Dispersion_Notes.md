# DFT-D3 Dispersion Correction: From First Principles to CrCl₃/NbSe₂

*Compiled from personal notes — CrCl₃ deposition on NbSe₂ surface project*

---

## What Is DFT-D3 Dispersion? (Quick Answer)

**In one sentence:** DFT-D3 is a mathematical correction added on top of a potential's energy to account for van der Waals attraction — the weak, long-range force between atoms/molecules that arises purely from quantum electron fluctuations, which standard DFT functionals (like PBE) and ML potentials trained on them fail to capture.

### Why it's needed

MACE-OMAT computes energy by approximating the **PBE** potential energy surface. PBE is a **local** functional — it looks at electron density point-by-point in space. When two objects (like a CrCl₃ flake and a NbSe₂ surface) are separated by a gap where their electron clouds don't overlap, PBE sees **zero interaction** between them.

But physically, there IS an attraction there — **London dispersion** — caused by correlated quantum fluctuations of electron density between the two bodies. This is completely invisible to any local functional.

**D3 is Grimme's fix**: instead of trying to redesign the functional (hard), just add the missing energy back in as a separate, analytical correction:

$$E_\text{total} = E_\text{DFT (or ML)} + E_\text{disp}$$

### What it physically computes

$$E_\text{disp} = -\sum_{\text{pairs}} \left( s_6 \frac{C_6^{ij}}{r_{ij}^6} + s_8 \frac{C_8^{ij}}{r_{ij}^8} \right) f_\text{damp}(r_{ij})$$

For every pair of atoms in the system:

| Piece | What it represents |
|---|---|
| $C_6^{ij}/r^6$ | Dipole–dipole dispersion — the dominant long-range attraction |
| $C_8^{ij}/r^8$ | Dipole–quadrupole correction — a smaller additional term |
| $C_6^{ij}$ | How polarizable this specific atom pair is (depends on local bonding environment via coordination number) |
| $s_6, s_8$ | Scaling factors — how much of this dispersion the base functional (PBE) is already missing |
| $f_\text{damp}(r)$ | Smoothly turns D3 off at short range so it doesn't interfere where MACE already handles bonding correctly |

### What it is NOT

- **Not a standalone potential** — it can't run a simulation by itself. It's an add-on.
- **Not electrostatics** — it has nothing to do with charges or ionic attraction.
- **Not chemisorption** — no bonds form, no electrons are shared. It's purely physisorption-type attraction.
- **Not arbitrary** — every number in it ($s_6$, $s_8$, damping parameters) is fitted from real quantum chemistry reference data, specific to the chosen functional (PBE) and damping style (BJ).

### In the LAMMPS script

```lammps
pair_style hybrid/overlay mace no_domain_decomposition dispersion/d3 bj pbe 30.0 20.0
pair_coeff * * mace mace-omat-0-small.model-lammps.pt Cl Cr Nb Se
pair_coeff * * dispersion/d3 Cl Cr Nb Se
```

`dispersion/d3` here means: *"run this correction alongside MACE, add its energy to MACE's energy, using BJ damping and PBE scaling factors, summing over all pairs within 30 Å and computing coordination numbers within 20 Å."*

### Why it mattered for this simulation

Without it: MACE alone saw essentially zero attraction across the CrCl₃–NbSe₂ interface (pure vdW gap, no bonding, no electrostatics) → CrCl₃ didn't adsorb.

With it: the correct long-range attraction is restored → CrCl₃ physisorbs on the surface → nucleation can proceed, matching real PVTD physics.

---

## Chapter 1 — The Fundamental Problem

We are simulating **CrCl₃ on a NbSe₂ surface**.

Force on any atom comes from energy:

$$F_i = -\frac{\partial E}{\partial \vec{r}_i}$$

So the central question at every timestep is:

> **Given atomic positions, what is the energy?**

The exact answer requires solving the **Schrödinger equation** for every electron in the system:

$$\hat{H}\Psi = E\Psi$$

Solving this exactly, for each electron, in a system of thousands of atoms — **is not possible**.

---

## Chapter 2 — DFT: The Practical Solution

Instead of tracing every electron individually, DFT tracks only the **probability of finding an electron at each point in space** — the electron density $n(\vec{r})$.

**Total energy:**

$$E = T(n) + E_{ne}[n] + E_{ee}[n] + E_{xc}[n]$$

| Term | Meaning | Status |
|---|---|---|
| $T(n)$ | Kinetic energy of electrons | — |
| $E_{ne}[n]$ | Nucleus–electron attraction | **known** |
| $E_{ee}[n]$ | Electron–electron repulsion | **known** |
| $E_{xc}[n]$ | Exchange-correlation energy | **unknown — must be approximated** |

$E_{xc}$ is the mathematical approximation used to determine the **unknown energy terms that account for complex multi-body electron–electron interactions** — it captures:
- Exchange (Pauli exclusion)
- Correlation effects

---

## Chapter 3 — PBE: The Standard Approximation

**PBE** is the most widely used approximation for $E_{xc}$. It belongs to the **GGA** family — **Generalized Gradient Approximation**.

$$E_{xc}^{PBE}[n] = \int f\big(n(\vec{r}),\ \nabla n(\vec{r})\big)\, d\vec{r}$$

It uses **local density** and **gradient** at each point in space.

**PBE is:**
- Cheap — scales as $N^3$
- Accurate for **covalent bonds**

**PBE is not:**
- Able to capture **van der Waals interaction**
- Able to find interaction between two **completely separated electron clouds**

**Consequence:** NbSe₂ can't attract CrCl₃ — at least not according to PBE.

---

## Chapter 4 — Flaw of PBE: Missing Dispersion

### The physical picture

```
   Cl  Cl  Cl        ← CrCl₃ layer
    \  |  /
     Cr
                       ↕  ~3.5 Å
   Se  Se  Se  Se  Se  ← NbSe₂ surface
    \ /  \ /  \ /
    Nb    Nb    Nb
    / \  / \  / \
   Se  Se  Se  Se  Se
```

- The **Cl and Se are ~3.5 Å apart**; their electron clouds **do not overlap** at this distance.
- **PBE computes $E_{xc}$ by integrating locally at each point.** Since the electron densities don't overlap, PBE sees **zero interaction** between CrCl₃ and NbSe₂.
- **But physically there is an interaction — "London dispersion."**
  - It is created due to **electronic fluctuation** and **instant dipole** at every moment.

### The dispersion energy

This energy of interaction is:

$$E_\text{disp} = -\frac{C_6}{r^6} = \frac{-C_6}{(r^3)^2}$$

**$C_6$ → encodes how polarizable each atom is.**

---

## Chapter 5 — The Full Dispersion Series ($C_6$ and $C_8$)

At any instant, not only dipoles but also **multipoles** occur. So each creates additional dispersion.

| Interaction | Term | Magnitude at 3.5 Å |
|---|---|---|
| dipole ↔ dipole | $C_6/r^6$ | ~85% of total |
| dipole ↔ quadrupole | $C_8/r^8$ | ~15% of total |
| dipole ↔ octupole | $C_{10}/r^{10}$ | ~1% of total |

**Grimme** derives $C_8$ from $C_6$ via:

$$C_8^{ij} = 3 \cdot C_6^{ij} \cdot \sqrt{Q_i \cdot Q_j}$$

---

## Chapter 6 — D3's $C_6$ is Geometry Dependent

The predecessor **D2 method used fixed $C_6$ per element.** So carbon always had the same value.

> *"C in graphene has a very different $C_6$ than in diamond."*

**$C_6^{ij}$ → depends on the coordination environment:**

- **More neighbors → more constrained → less polarizable electrons**

D3 counts neighbors via a **smooth coordination number**:

$$\text{CN}_i = \sum_{j \neq i} \frac{1}{1 + e^{-16(R_\text{cov}/r_{ij} - 1)}}$$

**CN is basically telling each atom's bonding environment.**

So CN captures all the coordination environment around an atom for a **cutoff distance ($r_{CN}$) — 20 Å.**

Then it assigns a value to $C_6^{ij}$. And $C_8^{ij}$ is computed from $C_6^{ij}$.

---

## The DFT-D3 Correction — Complete Formula

**Total energy:**

$$E_\text{total} = E_\text{DFT} + E_\text{disp}$$

**Pair-style:** `dispersion/d3`
- Not a stand-alone force field
- Can be used **with ML potential + pair style hybrid/overlay**

```
pair-style hybrid/overlay mace dispersion/d3
```

### The dispersion energy sum

$$E_\text{disp} = -\sum_{\text{pairs}} \left( s_6 \cdot \frac{C_6^{ij}}{r_{ij}^6} + s_8 \cdot \frac{C_8^{ij}}{r_{ij}^8} \right) \cdot f_\text{damp}(r_{ij})$$

**Symbols:**
- $\dfrac{C_6^{ij}}{r_{ij}^6}$ → london dispersion interaction b/w ($i$ x $j$)
- $C_6$ → encodes how polarizable each atom is
- $\dfrac{C_8^{ij}}{r_{ij}^8}$ → higher order dispersion (dipole–quadrupole)
- $s_6, s_8$ → scaling factors (**PBE catches some medium range correlation imperfectly**)
- $f_\text{damp}(r)$ → damping function

**At short distances, ML potential handles everything correctly — we don't want D3 interfering there.**

**For PBE with BJ damping:**

$$s_6 = 1.000 \qquad s_8 = 0.7875$$

---

## The Damping Function — "BJ"

$$f^{(n)}(r) = \frac{r^n}{r^n + (a_1 R_0^{ij} + a_2)^n} \longrightarrow \text{constant as } r \to 0$$

### Distance regimes

| Range | Distance | Behavior |
|---|---|---|
| **Short range** | 0–3 Å | MACE → bonding; D3 → 0 |
| **Interface range** | 3–6 Å | MACE → 0; D3 provides long-range MACE→0 |

---

## Full DFT-D3(BJ) Energy Expression

$$E_\text{disp} = \frac{1}{2} \sum_{j \neq i} \left( s_6 \cdot \frac{C_6^{ij}}{r_{ij}^6 + f_6^\text{damp}(r_{ij})} + s_8 \cdot \frac{C_8^{ij}}{r_{ij}^8 + f_8^\text{damp}(r_{ij})} \right)$$

**Also (compact form used for the pair sum):**

$$E_\text{disp} = -\sum_\text{pairs} \left( s_6 \cdot \frac{C_6^{ij}}{r^6} + s_8 \cdot \frac{C_8^{ij}}{r^8} \right) f_{BJ}$$

where $C_8^{ij}$ is computed from $C_6^{ij}$.

---

## MLIP — Multi-body / ML Interatomic Potential

$$E = E_\text{KE} + E_{e-e} + E_{n-e}$$

```
                Dispersion
                    D3
                   /    \
                  ○      ○
                   C — for
                    D₁
```

---

## Summary

### Ch-1: The Setup
- We are simulating **CrCl₃ on NbSe₂ surface**
- Force comes from energy: $F_i = -\dfrac{\partial E}{\partial \vec{r}_i}$
- So given atomic position, what is the energy?
- Solution is **Schrödinger's equation** for every electron in the system
- $\hat{H}\Psi = E\Psi$ — solving for each electron **is not possible**

### Ch-2: DFT — Practical Solution
- Instead of tracking every electron individually, you only track the **probability of finding an electron at each point in space**

**Total energy:**

$$E = T(n) + E_{ne}[n] + E_{ee}[n] + E_{xc}[n]$$

| Term | Meaning |
|---|---|
| $T(n)$ | KE of $e^-$ |
| $E_{ne}[n]$ | nucleus–electron attraction (known) |
| $E_{ee}[n]$ | electron–electron repulsion (known) |
| $E_{xc}[n]$ | Exchange correlation energy (unknown) |

### Ch-3: PBE — Standard Approximation

$$E_{xc}^{PBE} = \int f\big(n(\vec{r}), \nabla n(\vec{r})\big)\, d\vec{r}$$

- Uses **local electron density** and **gradient of LED**

### Ch-4: Flaw of PBE — Missing Dispersion

```
     Cl   Cr   Cl
      \   |   /
        (Cl)
     ↕ ~3.5 Å
   Se  Nb  Se  Nb  Se  Nb  Se  Nb  Se
```

- The Cl & Se are 3.5 Å apart; their electron clouds don't overlap at this distance.
- PBE computes $E_{xc}$ by integrating locally at each point. Since the electron densities don't overlap, PBE sees **zero interaction between CrCl₃ & NbSe₂**.
- But physically there **is** an interaction — **"London dispersion."**
  - It is created due to electronic fluctuation and constant dipole at every moment.

**This energy of interaction is:**

$$E_\text{disp} = -\frac{C_6}{r^6} = \frac{-C_6}{(r^3)^2}$$

$C_6$ → encodes how polarizable each atom is.

### Ch-5: The Full Dispersion Series ($C_6$ and $C_8$)

At an instant, not only dipoles but also multipoles also occur. So each creates additional dispersion.

| Interaction | Term | Magnitude at 3.5 Å |
|---|---|---|
| dipole ↔ dipole | $C_6/r^6$ | ~85% of total |
| dipole ↔ quadrupole | $C_8/r^8$ | ~15% of total |
| dipole ↔ octupole | $C_{10}/r^{10}$ | ~1% of total |

"Grimme" derives $C_8$ from $C_6$ via:

$$C_8^{ij} = 3 \cdot C_6^{ij} \cdot \sqrt{Q_i \cdot Q_j}$$

### Ch-6: D3's $C_6$ is Geometry Dependent

- The predecessor D2 method used fixed $C_6$ per element. So carbon always had the same value.
- "C in graphene has a very different $C_6$ in diamond."
- $C_6^{ij}$ → depends on the coordination environment
- "more neighbors" → "more constrained" → "less polarizable electrons"

**D3 counts neighbors via a smooth coordination number:**

$$\text{CN}_i = \sum_{j \neq i} \frac{1}{1 + e^{-16(R_\text{cov}/r_{ij} - 1)}}$$

So CN captures all the coordination environment around an atom for a cutoff distance ($r_{CN}$) — **20 Å**. Then it assigns value to $C_6^{ij}$.

$C_8^{ij}$ is computed from $C_6^{ij}$.

$$E_\text{disp} = -\sum_\text{pairs} \left( s_6 \cdot \frac{C_6^{ij}}{r^6} + s_8 \cdot \frac{C_8^{ij}}{r^8} \right) f_{BJ}$$

---

## Key Numbers to Remember

| Quantity | Value | Meaning |
|---|---|---|
| $s_6$ (PBE) | 1.000 | Full $C_6/r^6$ added — PBE misses all of it |
| $s_8$ (PBE) | 0.7875 | 78.75% of $C_8/r^8$ added |
| CN cutoff | 20 Å | Radius to count neighbors for coordination number |
| D3 cutoff | 30 Å | Radius to sum dispersion energy pairs |
| Interface distance | ~3.5 Å | CrCl₃–NbSe₂ gap — where D3 dominates |
| Short range | 0–3 Å | MACE handles bonding; D3 damped to ~0 |
