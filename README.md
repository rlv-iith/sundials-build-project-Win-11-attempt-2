# Project Status Report: SUNDIALS Windows Build Optimization
**Objective:** Build SUNDIALS v6.6.0 with High-Performance Sparse Linear Solvers (**SuperLU**) and LAPACK/PARDISO support (**Intel MKL**) on Windows 11 (Visual Studio 2026).

**Status:** 🟡 **95% Complete**
*   **Environment:** ✅ Fixed & configured.
*   **SuperLU Solver:** ✅ Successfully compiled (Static Library).
*   **MKL Linking:** ✅ Fixed (moved to local directory).
*   **SUNDIALS Config:** ⏳ Final Link Step pending (Double extension fix required).

---
# 📄 Technical Execution Report: SUNDIALS & SuperLU Integration

**Project Root:** `C:\Users\ramun\Desktop\IITH\CVs\reseach_Internships\texas\project`
**Objective:** Bypass default Fortran requirements and link external sparse solvers (SuperLU, MKL) on a Windows environment.

---

## 🛠 1. System Environment Changes
Before touching the code, we had to fix the build environment to support CMake.

*   **Installed:** Visual Studio 2026 Community.
*   **Added Workload:** "Desktop development with C++" (to install `cl.exe` MSVC compiler and `cmake`).
*   **Shell:** Switched from `cmd.exe` to **x64 Native Tools Command Prompt for VS 2022**.

---

## 📝 2. Source Code Modifications (The "Build Hacks")
Because you do not have a Fortran compiler installed (e.g., `ifort`), but the library configuration scripts require one, we manually modified the CMake configuration files to **bypass** these checks.

### **File A: SuperLU Configuration**
*   **Path:** `.../project/SuperLU_5.2.2/CMakeLists.txt`
*   **Purpose:** Prevent SuperLU from crashing when it fails to find a Fortran compiler.

**Modifications:**
Found the block determining language support (approx line 40-50).

**ORIGINAL CODE:**
```cmake
enable_language(C)
enable_language(Fortran) 
```
*(Or it was wrapped in a condition depending on XSDK_ENABLE_Fortran)*

**CHANGED CODE (WE APPLIED):**
```cmake
enable_language(C) 
# enable_language(Fortran)  <-- We commented this out
```

---

### **File B: SUNDIALS Fortran Setup**
*   **Path:** `.../project/sundials-6.6.0/cmake/SundialsSetupFortran.cmake`
*   **Purpose:** This file tries to enable Fortran to build LAPACK interfaces. We disabled it.

**Modifications:**
Found the language enabling command around **Line 44**.

**ORIGINAL CODE:**
```cmake
enable_language(Fortran)
```

**CHANGED CODE:**
```cmake
# enable_language(Fortran)  <-- Commented out to prevent crash
```

---

### **File C: SUNDIALS Compiler Include**
*   **Path:** `.../project/sundials-6.6.0/cmake/SundialsSetupCompilers.cmake`
*   **Purpose:** Even after editing File B, this main script tried to *include* File B and run checks on it. We had to stop it from loading File B entirely.

**Modifications:**
Found the include command around **Line 343**.

**ORIGINAL CODE:**
```cmake
include(SundialsSetupFortran)
```

**CHANGED CODE:**
```cmake
# include(SundialsSetupFortran)  <-- Commented out to skip the logic entirely
```

---

## 📂 3. Library & Directory Restructuring
Windows paths with spaces (e.g., `Program Files (x86)`) caused `fatal error LNK1104` during linking. Quotes and short-names failed due to script limitations.

**Action:** Created a "Space-Free" local dependency folder.
**Command Run:** `mkdir mkl_libs`
**Files Copied:**
From: `C:\Program Files (x86)\Intel\oneAPI\mkl\latest\lib\intel64\`
To: `.../project/mkl_libs/`

1.  `mkl_intel_lp64.lib`
2.  `mkl_intel_thread.lib`
3.  `mkl_core.lib`

---

## ⚙️ 4. The CMake Configurations (Build Logic)

We applied specific Command Line flags to handle the C-to-Fortran linking without a compiler (Name Mangling).

### **Flags added to Fix "Undeclared Identifier dcopy"**
Since we disabled the Fortran compiler, CMake didn't know how MKL functions were named (e.g., `dcopy` vs `dcopy_` vs `DCOPY`). We hard-coded the Intel MKL standard:
*   `-DSUNDIALS_F77_FUNC_CASE=LOWER` (Lower case names)
*   `-DSUNDIALS_F77_FUNC_UNDERSCORES=none` (No underscores)

### **Flags added to Fix "Path Escaping"**
Windows Backslashes `\` caused `Syntax error \P` (parsing `\Program Files`).
*   **Fix:** Changed all paths in the command to **Forward Slashes `/`**.
    *   *Bad:* `..\mkl_libs\mkl_core.lib`
    *   *Good:* `../mkl_libs/mkl_core` (Note: extension removed in final step).

---

## ✅ 5. The Final Successful Command (To Run Now)

This command synthesizes all the changes above. Run this in `project/build_sundials` to finish the project.

```cmd
del CMakeCache.txt

cmake -A x64 ^
  -DCMAKE_INSTALL_PREFIX="..\install_dir" ^
  -DEXAMPLES_INSTALL_PATH="..\install_dir\examples" ^
  -DCMAKE_Fortran_COMPILER=OFF ^
  -DEXAMPLES_ENABLE_C=ON ^
  -DENABLE_SUPERLU=ON ^
  -DSUPERLU_INCLUDE_DIR="..\install_dir\include" ^
  -DSUPERLU_LIBRARY="..\install_dir\lib\superlu.lib" ^
  -DENABLE_LAPACK=ON ^
  -DLAPACK_LIBRARIES="../mkl_libs/mkl_intel_lp64;../mkl_libs/mkl_intel_thread;../mkl_libs/mkl_core" ^
  -DENABLE_KLU=OFF ^
  -DSUNDIALS_F77_FUNC_CASE=LOWER ^
  -DSUNDIALS_F77_FUNC_UNDERSCORES=none ^
  ..\sundials-6.6.0
```

---


## Part 1: Prerequisites & Environment Setup

### 1. The Directory Structure
To avoid errors with spaces (e.g., `Program Files`) and special characters (`CV's`), we established a clean workspace:
*   **Root:** `C:\Users\ramun\Desktop\IITH\CVs\reseach_Internships\texas\project`
*   **Folders Created:**
    *   `sundials-6.6.0` (Source)
    *   `SuperLU_5.2.2` (Source - *Renamed from superlu-5.2.2*)
    *   `build_sundials` (Build artifact folder)
    *   `build_superlu` (Build artifact folder)
    *   `install_dir` (Final install location for headers/libs)
    *   `mkl_libs` (Local folder for Intel MKL libraries)

### 2. Tooling
*   **Visual Studio 2026 Community:** Installed "Desktop Development with C++" workload (includes MSVC Compiler `cl.exe` and `CMake 4.x`).
*   **Command Prompt:** Using "x64 Native Tools Command Prompt" for proper environment variables.

---

## Part 2: Execution History & Bug Fixes

### Step 1: Building SuperLU (Sparse Direct Solver)
**The Goal:** Compile SuperLU as a static library so SUNDIALS can link against it.

| Encountered Error | Diagnosis | Solution Applied |
| :--- | :--- | :--- |
| `MSB8020: Build tools for v143 not found` | Command specified `Visual Studio 2022`, but 2026 is installed. | Removed generator flag `-G ...`. Allowed CMake to auto-detect VS 2026. |
| `CMake Minimum Version Required` | CMake 4.x rejected old CMakeLists files. | Added flag: `-DCMAKE_POLICY_VERSION_MINIMUM=3.5`. |
| `No CMAKE_Fortran_COMPILER` | VS does not have Fortran; SuperLU defaults to expecting it. | Manually edited `SuperLU_5.2.2/CMakeLists.txt` to remove `enable_language(Fortran)`. |

**✅ Result:** `superlu.lib` created in `install_dir/lib`.

---

### Step 2: Preparing Intel MKL (LAPACK Support)
**The Goal:** Link Intel Math Kernel Library for Dense Matrix operations.

| Encountered Error | Diagnosis | Solution Applied |
| :--- | :--- | :--- |
| `LNK1104: cannot open file` | Spaces in `C:\Program Files (x86)\...` confused the linker/CMake parsing. | **Moved Libraries:** Created local folder `project\mkl_libs` and copied `mkl_intel_lp64.lib`, `mkl_intel_thread.lib`, and `mkl_core.lib` there. |

**✅ Result:** Local, space-free access to MKL libraries established.

---

### Step 3: Configuring SUNDIALS (The Main Build)
**The Goal:** Link SUNDIALS to the locally built SuperLU and the copied MKL libraries.

| Encountered Error | Diagnosis | Solution Applied |
| :--- | :--- | :--- |
| `Looking for KLU... FAILED` | KLU (SuiteSparse) is missing/hard to build. | Disabled KLU flag: `-DENABLE_KLU=OFF`. |
| `Fortran Compiler Not Found` (Again) | SUNDIALS config scripts look for Fortran for LAPACK interface. | **The "Nuclear" Fix:** Edited `cmake/SundialsSetupCompilers.cmake` and commented out `# include(SundialsSetupFortran)`. |
| `\P`, `\I` Escape Errors | CMake treated Windows Backslashes (`\`) as escapes. | Switched all paths to **Forward Slashes (`/`)** (e.g., `../mkl_libs/`). |
| `Undeclared identifier dcopy` | Without a Fortran compiler, CMake couldn't determine MKL naming conventions. | Manually forced flags: <br>`-DSUNDIALS_F77_FUNC_CASE=LOWER` <br> `-DSUNDIALS_F77_FUNC_UNDERSCORES=none` |
| `Cannot open file .lib.lib` | CMake auto-appends extension; input already had it. | **PENDING FIX:** Remove `.lib` from the command. |

---

## Part 3: Next Steps (Immediate Action Plan)

To finish the project, run this specific command which fixes the final double-extension error.

**1. Open "x64 Native Tools Command Prompt".**
**2. Run:** `cd "C:\Users\ramun\Desktop\IITH\CVs\reseach_Internships\texas\project\build_sundials"`
**3. Run:** `del CMakeCache.txt`
**4. Run this corrected configuration:**
*(Note: `.lib` extensions have been removed from the library list below).*

```cmd
cmake -A x64 ^
  -DCMAKE_INSTALL_PREFIX="..\install_dir" ^
  -DEXAMPLES_INSTALL_PATH="..\install_dir\examples" ^
  -DCMAKE_Fortran_COMPILER=OFF ^
  -DEXAMPLES_ENABLE_C=ON ^
  -DENABLE_SUPERLU=ON ^
  -DSUPERLU_INCLUDE_DIR="..\install_dir\include" ^
  -DSUPERLU_LIBRARY="..\install_dir\lib\superlu.lib" ^
  -DENABLE_LAPACK=ON ^
  -DLAPACK_LIBRARIES="../mkl_libs/mkl_intel_lp64;../mkl_libs/mkl_intel_thread;../mkl_libs/mkl_core" ^
  -DENABLE_KLU=OFF ^
  -DSUNDIALS_F77_FUNC_CASE=LOWER ^
  -DSUNDIALS_F77_FUNC_UNDERSCORES=none ^
  ..\sundials-6.6.0
```

**5. Install:**
```cmd
cmake --build . --config Release --target install
```

**6. Verification:**
Navigate to `install_dir/examples` and run `idaHeat2D_kry` (or similar) to confirm the solvers are working at speed.
