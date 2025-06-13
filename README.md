# ExternalGaussianMolproBeta

# Using Molpro as an External Program

This folder contains sample files and scripts for the `MolproExt` interface that allows Gaussian to call Molpro for energies and gradients.


---
- [1. Environment variables](#1-environment-variables)
- [2. Preamble file](#2-preamble-file)
- [3. Gaussian input and wrapper keyword explanation](#3-gaussian-input-and-wrapper-keyword-explanation)
- [4. Ending file](#4-ending-file)
  - [Example: Hierarchical optimization workflow with frequency reading](#example-hierarchical-optimization-workflow-with-frequency-reading)
- [5. Output structure](#5-output-structure)
- [6. Global Arrays version](#6-global-arrays-version)

## 1. Environment variables

The wrapper expects the following environment variables to be set before running:

* `GAUSS_SCRDIR` – scratch directory used by Gaussian.
* `TMPDIR` (or `SCRATCH`) – scratch directory for Molpro.
* `PATH` – must include the directories containing the `molpro` and `g16`/`gdv` executables.

Example configuration:

```bash
export GAUSS_SCRDIR=/path/to/gaussian_scratch
export TMPDIR=/path/to/molpro_scratch
export SCRATCH=$TMPDIR  # used internally by the scripts
export PATH=/path/to/molpro/bin:/path/to/gaussian/bin:$PATH
```

> **Note:** Ensure the `MolproExt` script is executable:

```bash
chmod +x MolproExt
```

> **Note:** Modify the shebang (`#!`) at the top of `MolproExt` or `MolproExt_GA` to point to a valid Python interpreter on your system. For example:

```text
#!/usr/bin/env python3
```

The `numpy` package must be available in the selected Python environment.

---

## 2. Preamble file

The *preamble file* is the first argument passed to `MolproExt`. It defines the structure and instructions that will appear **before the geometry** section of the generated `qmolpro.com` input file. It may also define how to **combine gradients** from multiple runs.

A minimal valid preamble **must include** a `!scheme` directive followed by a line of coefficients:

```text
!scheme
!1 -1 1
```

This tells the wrapper to perform three separate gradient calculations and combine them using weights `1`, `-1`, and `1`. Please note that this is mandatory in case of a gradient calculation. The number of coefficients is not fixed and it is completely arbitrary based on the method.  

Aside from the `!scheme` section, any other content valid in a Molpro input **before the geometry** can be placed in the preamble (e.g., memory settings, symmetry, global variables, etc.). This offers great flexibility in customizing your Molpro calculation.

> **Note:** The preamble file must either be in the Gaussian launch directory or be passed with a full path using the `External` keyword.
>
> The file **can have any name**. It does not have to be `preamble.dat`, as long as the correct filename is specified in the `External` call.

---

## 3. Gaussian input and wrapper keyword explanation

Gaussian invokes the external wrapper using the `External` keyword. A minimal example looks like this:

```text
%chk=example.chk
%nprocs=1
#p opt(nomicro) external="MolproExt preamble.dat ending.dat 4 8 READ"
```

**WARNING**: nomicro is mandatory in case of an external invocation for gradients obtainment.

### Meaning of the wrapper arguments:

```text
MolproExt preamble.dat ending.dat 4 8 READ
            |          |        | | |   |
            |          |        | | |   └─ keyword: geometry source (READ or NONE)
            |          |        | | └─── memory in GB to be passed to Molpro
            |          |        | └───── number of threads for Molpro
            |          └───────── ending file (commands to run in Molpro)
            └──────────── preamble file (header and !scheme section)
```

* The `preamble.dat` is inserted before the geometry and must contain `!scheme`.
* The `ending.dat` contains the final Molpro program blocks and energy expression.
* The integer arguments set the number of processors and memory in GB **for Molpro only** (not Gaussian).
  The wrapper reserves \~10% of the specified memory for system overhead.
* `READ`: Gaussian will pass the current geometry from the checkpoint file.
* `NONE`: No geometry is passed (used for single-point energy computations).

> **Important:**
> The filenames `preamble.dat` and `ending.dat` used in the examples are **just placeholders**. You can name these files however you like (e.g., `pre_opt_header.txt`, `molpro_tail.mol`), as long as the paths are passed correctly in the `External` keyword.

---

## 4. Ending file

The *ending file* provides the **tail section** of the Molpro input, after the geometry. It contains the series of Molpro program blocks used for computing energy and gradient, and must **define the final total energy** using the variable `exe_energy`. For example:

```text
include /path/to/3F12/basis/TrdRowElements_3F12

{rhf,so-sci}
{ccsd(t)-f12b,df_basis=avdz-f12/mp2fit,df_basis_exch=avdz-f12/jkfit,ri_basis=avdz/jkfit}
ccsd_energy = energy

{forces}

basis=cc-pwcvtz

{rhf,so-sci}
{mp2}
mp2_fc = energy

{forces}

{rhf,so-sci}
{mp2;core}
mp2_ae = energy

{forces}

! Composite energy calculation according to scheme, without hf_energy
exe_energy = ccsd_energy + mp2_ae - mp2_fc
```

The wrapper will search for `exe_energy = ...` to extract the total energy and log it.

> **Note:**
> The `ending.dat` file can contain any valid Molpro code after the geometry. It must match the `!scheme` coefficients in the preamble, if those involve energy combinations.
>
> Just like the preamble, the **filename of the ending file is arbitrary**. Any valid filename can be used as long as it is passed correctly in the `External` keyword.

---

### Example: Hierarchical optimization workflow with frequency reading

```text
%chk=1_3-oxazolidine_rev.chk
%nprocs=16
%mem=64GB
#p dsdpbep86/gen iop(3/125=0079905785,3/78=0429604296,3/76=0310006900,3/74=1004)
# empiricaldispersion=gd3bj iop(3/174=0437700,3/175=-1,3/176=0,3/177=-1,3/178=5500000) opt(tight,maxcycles=150)

C3H7NO

0 1
 C                  1.10980000   -0.62910000   -0.08880000
 C                 -0.32070000   -1.20170000   -0.05490000
 O                 -1.18730000   -0.08150000    0.20970000
 C                 -0.46910000    1.08140000   -0.25530000
 N                  0.93820000    0.82640000    0.14780000
 H                  1.71570000   -1.07340000    0.70100000
 H                  1.56620000   -0.80440000   -1.06300000
 H                 -0.56900000   -1.65000000   -1.01690000
 H                 -0.41040000   -1.94220000    0.73990000
 H                 -0.54560000    1.16890000   -1.33900000
 H                 -0.84720000    1.98200000    0.22870000
 H                  1.08290000    1.05430000    1.12000000

@/CustomBasisSet/3F12Red.gbs

--link1--
%chk=1_3-oxazolidine_pcs2.chk
%oldchk=1_3-oxazolidine_rev.chk
%nprocs=1
#p opt(RCFC,tight,nomicro,gic,maxcycles=100) geom=check output=pickett symmetry=loose
#  External="MolproExt preamble.dat ending.dat 30 250 read"

Title card

0 1


```
## 5. Output structure

During execution, the wrapper creates a temporary directory named `molpro_tmp_<number>` which contains:

* `qmolpro.com` – the assembled Molpro input file.
* `qmolpro.out` – the full Molpro output file.
* `qmolpro.xml` – gradients in XML format (if computed).
* `external.log` – wrapper diagnostic log (processors, memory, energy).

---

## 6. Global Arrays version

If using a GA-enabled build of Molpro, you can either:

* Use the wrapper `MolproExt_GA`, or
* Pass `molpro_ga` as a keyword to the higher-level `CentralExt` interface.

The Molpro executable used must have `_GA` in its name (e.g., `molpro_GA`).

