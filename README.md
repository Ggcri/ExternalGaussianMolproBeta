# ExternalGaussianMolproBeta
# Using Molpro as an External Program

This folder contains sample files and scripts for the `MolproExt` interface that allows Gaussian to call Molpro for energies and gradients.

## 1. Environment variables

The wrapper expects the following environment variables to be set before running:

- `GAUSS_SCRDIR` – scratch directory used by Gaussian.
- `TMPDIR` (or `SCRATCH`) – scratch directory for Molpro.
- `PATH` – must include the directories containing the `molpro` and `g16`/`gdv` executables.

Example configuration:

```bash
export GAUSS_SCRDIR=/path/to/gaussian_scratch
export TMPDIR=/path/to/molpro_scratch
export SCRATCH=$TMPDIR  # used internally by the scripts
export PATH=/path/to/molpro/bin:/path/to/gaussian/bin:$PATH
```

## 2. Preamble file

The first argument passed to `MolproExt` is a *preamble* file. It can define how different gradient contributions are combined using the `scheme` keyword. Each line starting with `!` after the keyword is interpreted as a coefficient. For example:

```text
!scheme
!1 -1 1
```

This instructs the wrapper to combine three gradients with weights `1`, `-1` and `1`. The preamble content is inserted at the beginning of the generated `qmolpro.com` input file.

## 3. Gaussian input example

An input file for Gaussian can invoke the wrapper using the `External` keyword:

```text
%chk=example.chk
%nprocs=4
%mem=8GB
#p opt external="MolproExt preamble.dat ending.dat 4 8 READ" geom=allcheck
```

The amount of memory specified with `%mem` is the total memory for the job. The wrapper scales it down by 10% to leave room for the system and then divides it among the requested threads before passing it to Molpro.

## 4. Ending file

The `ending.dat` file contains the Molpro commands to compute energies and gradients. The final line must define the total energy in the form

```text
exe_energy = <expression>
```

where `<expression>` is built from the intermediate energies obtained with Molpro. The wrapper expects `exe_energy` to be present so that it can read the final energy from `ending.dat` along with any accounting information from the Molpro run.

A minimal example is provided in `ending.dat`:

```text
include /home/barone/lcrisci_vb/MolproProcedures/basis/TrdRowElements_3F12
basis=cc-pvdz-f12
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

## 5. Output structure

During execution a temporary directory named `molpro_tmp_<number>` is created containing:

- `qmolpro.com` – the Molpro input file.
- `qmolpro.out` – textual Molpro output.
- `qmolpro.xml` – XML file with gradients when available.

Messages about the run (selected program, number of processors, memory usage, energies and gradients) are written to `external.log`.

## 6. GA implementation

To use the Global Arrays version of Molpro, invoke the wrapper `MolproExt_GA` (or pass the `molpro_ga` option to `CentralExt`). The underlying Molpro executable must end with `_GA`.
