Certo! Ecco il contenuto completo del `README.md` in **raw markdown**, pronto per essere copiato e incollato su GitHub:

````markdown
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
````

## 2. Preamble file

The first argument passed to `MolproExt` is a *preamble* file. It can define how different gradient contributions are combined using the `scheme` keyword. Each line starting with `!` after the keyword is interpreted as a coefficient. For example:

```text
!scheme
!1 -1 1
```

This instructs the wrapper to combine three gradients with weights `1`, `-1` and `1`. The preamble content is inserted at the beginning of the generated `qmolpro.com` input file.

> **Note:** The preamble file (`preamble.dat`) must either be located in the directory where the Gaussian job is launched, or its full path must be specified in the Gaussian input using the `External` keyword. The filename is arbitrary: you may use any name instead of `preamble.dat` as long as it is passed correctly to the `MolproExt` wrapper.

## 3. Gaussian input example

An input file for Gaussian can invoke the wrapper using the `External` keyword:

```text
%chk=example.chk
%nprocs=4
%mem=8GB
#p opt external="MolproExt preamble.dat ending.dat 4 8 READ" geom=allcheck
```

The amount of memory specified with `%mem` is the total memory for the job. The wrapper scales it down by 10% to leave room for the system and then divides it among the requested threads before passing it to Molpro.

> **Note:** The `ending.dat` file must also be either in the current working directory or specified with its full path. As with the preamble, the filename is not fixed — any valid file can be used.

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

@/home/barone/lcrisci_vb/CustomBasisSet/3F12Red.gbs

--link1--
%chk=1_3-oxazolidine_pcs2.chk
%oldchk=1_3-oxazolidine_rev.chk
%nprocs=1
#p opt(RCFC,tight,nomicro,gic,maxcycles=100) geom=check output=pickett symmetry=loose
#  External="MolproExt preamble.dat ending.dat 30 250 read"

Title card

0 1
```

## 4. Ending file

The `ending.dat` file contains the Molpro commands to compute energies and gradients. The final line must define the total energy in the form:

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

During execution, a temporary directory named `molpro_tmp_<number>` is created containing:

* `qmolpro.com` – the Molpro input file.
* `qmolpro.out` – textual Molpro output.
* `qmolpro.xml` – XML file with gradients when available.

Messages about the run (selected program, number of processors, memory usage, energies and gradients) are written to `external.log`.

## 6. GA implementation

To use the Global Arrays version of Molpro, invoke the wrapper `MolproExt_GA` (or pass the `molpro_ga` option to `CentralExt`). The underlying Molpro executable must end with `_GA`.

```

Fammi sapere se vuoi una versione `italiana`, oppure un file `.md` vero e proprio da scaricare!
```
