# PINNs and nonlinear FEM for a plate

The project contains the PINN experiments in `clear_training.ipynb` and a standalone nonlinear finite element solver in `plate_fem_nonlinear.cpp`.

## Nonlinear plate problem

The FEM solver solves

\[
D\\Delta^2 W + k_1 W - k_2 W^3 = F(x,y)
\]

on a rectangular plate. The current source uses

```text
D = 1,
k1 = 10,
k2 = 4.286,
F(x,y) = sin(pi*x/a) sin(pi*y/b).
```

The parameters are declared near the beginning of `plate_fem_nonlinear.cpp` and can be changed directly.

The boundary condition is the simply supported mixed form

\[
W=0,\qquad M=0,
\]

where \(M=-\\Delta W\). This is equivalent to the usual `W=0` and zero normal bending moment for the present nondimensional plate model.

## FEM formulation

The fourth-order equation is rewritten as two second-order equations:

\[
M=-\\Delta W,
\]
\[
-D\\Delta M+k_1W-k_2W^3=F,
\]
\[
-\\Delta W=M.
\]

The rectangular domain is divided into linear triangular `P1` elements. Assembly produces a stiffness matrix `K`, a mass matrix `Mmat`, and the load vector `F`.

At each Newton iteration the residual is

```text
R_M = D*K*M + k1*Mmat*W - k2*Mmat*(W^3) - F
R_W = K*W - Mmat*M
```

and the Jacobian is

```text
J11 = D*K
J12 = k1*Mmat - 3*k2*Mmat*diag(W^2)
J21 = -Mmat
J22 = K
```

The code uses damped Newton iterations with backtracking. The solution is written to `plate_fem_nonlinear_solution.csv`.

## Build

Eigen is required. With a compiler and Eigen available:

```powershell
g++ -std=c++17 -O2 plate_fem_nonlinear.cpp `
    -I "C:\path\to\eigen" `
    -o plate_fem_nonlinear.exe
```

Run it with:

```powershell
.\plate_fem_nonlinear.exe
```

The program prints the Newton residual at every iteration, the center deflection, and the final residual norm.

## Mesh and parameters

Change the physical dimensions here:

```cpp
const double a = 1.0;
const double b = 1.0;
```

Change the material and nonlinear parameters here:

```cpp
const double D = 1.0;
const double k1 = 10.0;
const double k2 = 4.286;
```

Change the mesh resolution here:

```cpp
const int nx = 40;
const int ny = 40;
```

Increasing `nx` and `ny` improves spatial resolution but increases the size of the Newton linear system.

## Comparison with PINN

Use the same values of `a`, `b`, `D`, `k1`, `k2`, load `F`, and boundary conditions in the PINN and FEM runs. Compare the CSV solution with the PINN on the same grid using:

- center deflection;
- maximum absolute deflection;
- relative L2 error;
- sections along `Ox` and `Oy`;
- maps of `W(x,y)`.

The older `plate_fem.cpp` file solves only the linear problem `D*Delta^2 W=F`. Use `plate_fem_nonlinear.cpp` for the `k1 W - k2 W^3` problem.


## Parameters from the command line

The nonlinear executable accepts `D`, `k1`, `k2`, and an optional output path:

```powershell
.\plate_fem_nonlinear.exe 1.0 10.0 4.286 .\fem_k10_gamma4286.csv
```

The default values are used when arguments are omitted.

## FEM and PINN comparison

`compare_fem_pinn.py` runs the FEM executable, reads its CSV output, evaluates the parametric PINN on the same FEM grid, and plots the two solutions, sections, and error.

Use it from the notebook after `model` and `DEVICE` have been created:

```python
import sys
sys.path.append(r"C:\Users\limon\OneDrive\??????? ????\code_2026\diplom\PINNS-for-plate")

from compare_fem_pinn import compare_fem_pinn

comparison = compare_fem_pinn(
    model=model,
    device=DEVICE,
    fem_executable=r"C:\Users\limon\OneDrive\??????? ????\code_2026\diplom\PINNS-for-plate\plate_fem_nonlinear.exe",
    D=1.0,
    k1=10.0,
    k2=4.286,
    output_csv="fem_comparison.csv",
)
```

The same physical `D`, `k1`, and `k2` are passed to FEM. The comparison script automatically converts them for the normalized PINN:

```text
k_PINN = k1 / D
gamma_PINN = k2 / D
```

The returned dictionary contains `W_fem`, `W_pinn`, `error`, and the metrics `rmse`, `absolute_l2`, `relative_l2_percent`, and `max_absolute_error`.


### Loading PINN weights in the comparison script

The helper `load_parametric_model` loads both checkpoints. The classes `PlatePINN` and `ParametricModel` remain defined in the notebook and are passed to the helper:

```python
from compare_fem_pinn import (
    load_parametric_model,
    compare_fem_pinn,
)

model = load_parametric_model(
    plate_pinn_class=PlatePINN,
    parametric_model_class=ParametricModel,
    base_checkpoint="checkpoints/sin_plate_pinn_best.pt",
    parametric_checkpoint="checkpoints/parametric_best.pt",
    device=DEVICE,
    hidden_dim=16,
    num_blocks=6,
)

comparison = compare_fem_pinn(
    model=model,
    device=DEVICE,
    fem_executable="plate_fem_nonlinear.exe",
    D=1.0,
    k1=10.0,
    k2=4.286,
)
```
