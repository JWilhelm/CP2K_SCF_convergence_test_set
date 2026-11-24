# CP2K SCF settings that always converge

This repository tests which CP2K SCF setting converges every system of a broad test set in **one single run from
`SCF_GUESS ATOMIC`**: no restarts, no two-step runs, and no per-system tuning. The test set contains molecules,
liquids, crystals, surfaces, interfaces and metals with up to 192 atoms, computed with basis sets from DZVP up to
aug-TZ2 (all 49 systems) and cc-pVQZ (all 31 systems with elements H–Ar).

Folder [`closed_shell/`](closed_shell) holds the closed-shell (non-magnetic) systems. Magnetic systems will follow
in a separate folder.

## Recommended setting

Diagonalisation with Broyden mixing (α 0.2, β 0.5 bohr⁻¹, 8 stored densities), Fermi–Dirac smearing at 300 K, an
overlap matrix filtered with `CHOLESKY OFF` and `EPS_EIGVAL 1.0E-6`, and `EPS_SCF 1.0E-7`:

```
&SCF
  SCF_GUESS ATOMIC
  EPS_SCF 1.0E-7
  MAX_SCF 300
  ADDED_MOS <about 0.2 x number of occupied orbitals + 10>
  CHOLESKY OFF          ! orthogonalisation with S^-1/2 ...
  EPS_EIGVAL 1.0E-6     ! ... overlap eigenvalues below 1e-6 removed
  &DIAGONALIZATION ON
    ALGORITHM STANDARD
  &END DIAGONALIZATION
  &MIXING ON
    METHOD BROYDEN_MIXING
    ALPHA 0.2
    BETA 0.5
    NBUFFER 8
  &END MIXING
  &SMEAR ON
    METHOD FERMI_DIRAC
    ELECTRONIC_TEMPERATURE [K] 300
  &END SMEAR
&END SCF
```

Use the same setting for insulators and metals, and for small and large basis sets.

**Result:**
- This setting converges **247 of 247 runs**: 163 runs with a band gap and 84 without.
- Every run reaches the ground state of its basis set. In each case the final energy equals the reference energy
  (definition below).
- Number of SCF steps:

  | Runs | Median | Maximum |
  |---|---|---|
  | With a band gap | 25 | 75 |
  | Without a band gap | 27 | 158 |

**Why `EPS_EIGVAL` is needed.** Without the overlap filtering, the same mixing converges 231 of 247 runs. The
failures are nearly linearly dependent basis sets:
- cc-pVQZ in dense solids (Si, amorphous Si, diamond, SiC, LiF, MgO, Al₂O₃, Al);
- aug-TZ2 for Al₂O₃, bulk Al and Pd(111).

Four further runs without filtering (bulk Si TZVP, bulk Al DZVP and TZVP, Au₃₈ ccGRB-Q) stop at `EPS_SCF 1.0E-7`
0.02–0.2 meV/atom above the reference energy, more than the tolerance below.

With cc-pVQZ and no filtering, the diagonalisation can even end several hartree *below* the true energy (collapse).

**Why `EPS_SCF 1.0E-7`.** With aug-TZ2, the SCF residual of SrTiO₃ fluctuates between 10⁻⁷ and 10⁻⁶ from step 30
on, while the energy is constant to 10⁻⁶ hartree; with `EPS_SCF 1.0E-8` this run does not finish.

**Effect of the filtering on the energy** (energy with filtering minus energy without):
- DZVP, TZVP, TZV2P and aug-DZ2: below 0.03 meV/atom.
- ccGRB-Q: at most 0.4 meV/atom.
- cc-pVQZ: at most 2.4 meV/atom (LiF).
- aug-TZ2: at most 3.3 meV/atom (GaN), except Pd(111).

**Known problem: Pd(111) with aug-TZ2.** The run converges, but the filtering raises the energy by 113 meV/atom.
This basis set is nearly linearly dependent in the metal (overlap condition number 1.7 × 10¹⁰). Without filtering,
or with `EPS_EIGVAL 1.0E-10`, the energy is constant to 10⁻⁸ hartree, but the SCF residual stays at 10⁻⁴: CP2K
measures it as the largest change of a density-matrix element in the atomic-orbital basis, and these elements are
noisy in the nearly dependent directions. No setting tested gives both convergence and the unfiltered energy here.

**Origin of the mixing parameters.** α 0.2, β 0.5 come from this benchmark; β 0.5 is the CP2K default. The CP2K
forum suggests a different setting for metals: Broyden α 0.1, β 1.5, NBROYDEN 8, 300 K.
- Sources: J. Hutter, 1 Dec 2009, [Rh(111) surface](https://groups.google.com/g/cp2k/c/MXYYuJmf1s0); M. Iannuzzi,
  4 Oct 2011, [metallic systems](https://groups.google.com/g/cp2k/c/OQczYEVvpME).
- That setting converges 224 of 247 runs and needs about twice as many SCF steps (median 55 with a gap, 50
  without).

**CP2K defaults converge far less often:**

| CP2K default setting | Runs converged |
|---|---|
| OT | 64 of 163 runs with a gap |
| Diagonalisation | 114 of 163 runs with a gap |
| Smearing with the default mixing | 26 of 84 runs without a gap |

## Comparison of SCF settings

### Systems with a band gap

The figure below shows the same 163 runs for every setting, one bar per basis set:
- (a) SCF runs converged;
- (b) SCF steps of the converged runs (box: 25–75 %, line: median, whiskers: minimum and maximum).

α is the mixing parameter and β the Kerker damping wave vector in bohr⁻¹. Runs of the OT and default settings that were
still running after 60–120 min (the time limit per run) count as not converged.

![Systems with a band gap](closed_shell/figures/converged_gap.png)

### Metals and semimetals

The figure below shows the same 84 runs (metals, graphene, graphite and the metallic (5,5) carbon nanotube), all
with Fermi–Dirac smearing, one bar per basis set.

![Systems without a band gap](closed_shell/figures/converged_nogap.png)

### Basis sets and overlap filtering

The figure below shows Broyden mixing (α 0.2, β 0.5 bohr⁻¹, 300 K) per basis set, without and with overlap filtering:
- (a) SCF runs converged;
- (b) energy change caused by the filtering (μeV/atom).

![Basis sets and overlap filtering](closed_shell/figures/basis_filter.png)

`EPS_EIGVAL 1.0E-7` changes the energies less (at most 0.6 meV/atom, Pd(111) 51 meV/atom), but with cc-pVQZ the SCF
of Al(111) and bulk Al collapses 2.8–11.7 eV/atom below the true energy.

### Comparison with Quantum ESPRESSO

The converged CP2K results were compared with Quantum ESPRESSO 7.3.1, using the same GTH-PBE pseudopotentials, the
same structures and the Γ point. With aug-TZVP tier-2 the results agree as follows:

| Quantity | Systems | Agreement |
|---|---|---|
| HOMO–LUMO gap | Si, GaAs, MoS₂, liquid water, bulk MgO, MgO(100)/water | within 5 meV |
| HOMO–LUMO gap | MgO(100) | 52 meV |
| Electron density | all seven systems with a gap | 0.3–0.5 % of the electrons distributed differently |

The remaining 52 meV for MgO(100) come from the CP2K basis set, not from the QE cutoff (the QE gap changes by 3 meV
from 400 to 600 Ry): the lowest conduction state of the free surface extends into the vacuum and needs diffuse
functions. cc-pVQZ, which has no diffuse functions, differs by 143 meV. A test confirms this: with one layer of
ghost atoms (Mg and O basis functions without nuclei) 2.1 Å above and below the slab, the difference drops from 52
to 0.5 meV (point "aug-TZ2 + ghost layer" in the figure).

QE uses the highest plane-wave cutoff that converged (200–800 Ry; MgO needs 600–800 Ry). In panel (a), metals and
graphene are left out; the axis is logarithmic on both sides and linear between −1 and 1 meV.

![CP2K vs Quantum ESPRESSO](closed_shell/figures/qe_compare.png)

## Definitions

- **SCF step**: one diagonalisation of the Kohn–Sham matrix and one density update, i.e. one line of the CP2K SCF
  output.
- **Converged**: a run counts as converged only if both conditions hold:
  1. CP2K reports SCF convergence (`EPS_SCF 1.0E-7`) within 300 SCF steps;
  2. the final energy agrees with the reference energy of the same system and basis set within 0.01 meV/atom (with
     gap) or 0.1 meV/atom (without gap), so the SCF did not end in a wrong state.
- **Reference energy**: the lowest final energy that at least two different settings reproduce in the same basis set.
  A single lower energy (collapse in a nearly singular basis) is not used.
- **Filtered runs** (`EPS_EIGVAL`) solve the problem in a slightly smaller basis. They count as converged if CP2K
  converged and their energy is not more than 1 meV/atom below the unfiltered reference.
- **EPS_SCF 1.0E-7**: all runs were computed with `EPS_SCF 1.0E-8` (the inputs in `runs/` show 1.0E-8). A run with
  1.0E-7 is identical up to the first SCF step whose residual lies below 10⁻⁷, where it stops; convergence, SCF steps
  and energy for 1.0E-7 are taken at that step.

## Test set and method

**Method:**
- PBE with GTH-PBE pseudopotentials (`POTENTIAL_UZH`).
- `CUTOFF` 600 Ry, `REL_CUTOFF` 60 Ry, `EPS_DEFAULT 1.0E-12`, Γ point.
- CP2K 2026.2 development version (master of 22 Nov 2026).

**Systems:** 49 systems with 5–192 atoms, listed in [`closed_shell/systems.csv`](closed_shell/systems.csv):

| Set | Systems | Content |
|---|---|---|
| `stage1` | 25 | molecules, liquids, bulk, 2D, slabs, interfaces with water, metals; used to choose the settings |
| `validation` | 24 | further systems, never used to choose the settings |
| `displaced` | 25 | the `stage1` structures with every atom displaced randomly by up to 0.1 Å |

**Basis sets:**

| Basis set label | CP2K basis set | Systems |
|---|---|---|
| DZVP, TZVP | `DZVP-`/`TZVP-MOLOPT-PBE-GTH` | DZVP: `stage1` only; TZVP: all systems |
| TZV2P | `TZV2P-MOLOPT-PBE-GTH` | all 25 `stage1` systems |
| aug-DZ2 | `aug-DZ2-MOLOPT-PBE-GTH`, else `aug-DZVP-MOLOPT-GTH-tier-2` | all 25 `stage1` systems |
| aug-TZ2 | `aug-TZ2-MOLOPT-PBE-GTH`, else `aug-TZVP-MOLOPT-GTH-tier-2` | all 49 systems |
| ccGRB-Q | `ccGRB-Q` | `stage1` systems with at most 100 atoms (MoS₂ missing: CP2K cannot read the Mo set) |
| cc-pVQZ | `cc-pVQZ-GTH` (Ye and Berkelbach, [github.com/hongzhouye/ccgto](https://github.com/hongzhouye/ccgto); converted file `closed_shell/BASIS_ccGTO_YE`) | all 31 systems with elements H–Ar |

**Runs:** each run is one system in one basis set with one setting: 247 runs with the recommended setting, 3647 runs
in total for all settings.

## Files

| Path | Content |
|---|---|
| `closed_shell/runs/<set>/<system>/<basis>/run.inp`, `run.out` | CP2K input and output of all 247 runs with the recommended setting (`larger_basis`: TZV2P, aug-DZ2, aug-TZ2, ccGRB-Q, cc-pVQZ) |
| `closed_shell/results/all_runs.csv` | All runs of all settings, evaluated at `EPS_SCF 1.0E-7`: set, system, basis, setting, class (`gap`/`nogap`), atoms, CP2K converged (0/1), SCF steps, energy minus reference (meV/atom), converged by the definition above (0/1) |
| `closed_shell/systems.csv` | The 49 systems with set, group, class, number of atoms and charge |
| `closed_shell/BASIS_ccGTO_YE` | cc-pVQZ basis sets in CP2K format |

**Naming of settings in `all_runs.csv`:**
- `smear300_` / `smear1000_`: Fermi–Dirac smearing at 300 K or 1000 K.
- `diag_`: diagonalisation without smearing.
- `broyden_a0.2_b0.5_nb8`: Broyden mixing, α 0.2, β 0.5, NBUFFER 8.
- `kerker_`: Kerker mixing.
- `_eigval1.0E-6`: `CHOLESKY OFF`, `EPS_EIGVAL 1.0E-6`.
- `ot_diis_full_all_gap0.01_irac_20x15`: OT with DIIS, preconditioner FULL_ALL, `ENERGY_GAP 0.01`, `ALGORITHM IRAC`,
  20 inner steps × 15 outer steps.
- `*_cp2k_default`: the CP2K defaults.

To run an input, set `BASIS_SET_FILE_NAME BASIS_ccGTO_YE` to the path of `closed_shell/BASIS_ccGTO_YE`.
