# OpenROAD-on-Android
# 📱 Silicon on a Smartphone: Native OpenROAD VLSI Flow on Android (ARM64)

![Arch Linux](https://img.shields.io/badge/OS-Arch_Linux_ARM-1793d1?logo=arch-linux&logoColor=white)
![Termux](https://img.shields.io/badge/Environment-Termux_PRoot-4E2A84?logo=terminal&logoColor=white)
![OpenROAD](https://img.shields.io/badge/EDA-OpenROAD-blue)
![Status](https://img.shields.io/badge/Status-100%25_Working-brightgreen)

This repository documents the successful compilation and execution of a complete **RTL-to-GDSII VLSI flow** running entirely natively on an Android smartphone (Snapdragon 8 Gen 3) using Arch Linux via Termux. 

Compiling enterprise-grade Electronic Design Automation (EDA) tools like OpenROAD on ARM64 mobile devices is notoriously difficult due to strict RAM limits (Android's OOM Killer), Link-Time Optimization (LTO) compiler segfaults, and bleeding-edge package updates (Python 3.14, `fmt` v12). This repository provides the **pre-compiled binaries**, the **source-code patches**, and a **sample 8-bit CMOS Barrel Shifter project**.

---

## 🚀 Method 1: The Easy Way (Use Pre-Compiled Binaries)
If you are running Arch Linux ARM (aarch64) and want to skip the 24-hour compilation process, you can use the pre-compiled binaries provided in this repository's Releases.

1. Install OpenROAD (Pre-Compiled)
Download the `.tar.gz` archive from the [Releases](../../releases) tab of this repository and extract it to your system root:
```bash
cd ~
wget https://github.com/mikey-7x/mikey-7x/releases/latest/download/openroad-archlinux-arm64.tar.gz
sudo tar -xzvf openroad-archlinux-arm64.tar.gz -C /
```
2. Install Yosys (OSS CAD Suite)

For the logic synthesis step, download the pre-compiled ARM64 OSS CAD Suite:
```
cd ~
wget https://github.com/YosysHQ/oss-cad-suite-build/releases/download/2026-10-16/oss-cad-suite-linux-arm64-20261016.tgz
tar -xzf oss-cad-suite-linux-arm64-20261016.tgz
```
You are now ready to run the VLSI flow! Skip down to the "Building the 8-bit
Barrel Shifter" section.

# 🛠️ Method 2: The Hard Way (Compiling from Source)

If you want to compile OpenROAD from scratch, you must apply several patches to
fix Python 3.14 compatibility, fmt v12 header changes, and GCC LTO segfaults.

1. Install System Dependencies
```
sudo pacman -Syu
sudo pacman -S base-devel git cmake python klayout tcl boost swig zlib qt5-base qt5-tools qt5-xmlpatterns qt5-multimedia qt5-svg ruby libgit2 openblas spdlog yaml-cpp
```
Note: You will also need an AUR helper like yay to install qt5-charts and
or-tools.

2. Compile Google OR-Tools

OR-Tools is a massive C++ dependency. To prevent Android from killing Termux due
to RAM exhaustion, compile it on a limited number of cores.
```
yay -S or-tools --mflags="-A -j4"
```
(If it fails due to an Abseil SourceLocation ambiguity bug, patch
absl/types/source_location.h in the build cache and resume).

3. Clone OpenROAD
```
cd ~
git clone --recursive https://github.com/The-OpenROAD-Project/OpenROAD.git
cd OpenROAD
```

4. Apply Source Code Patches (The Magic Fixes)

Run these commands inside the ~/OpenROAD folder to patch the source code:

Fix Python 3.14 Compatibility (Upgrades legacy PyInt_ and PyString_ APIs):
```
find src -type f \( -name "*.i" -o -name "*.cpp" -o -name "*.h" \) -exec sed -i 's/PyInt_/PyLong_/g; s/PyString_FromString/PyUnicode_FromString/g; s/PyString_AsString/PyBytes_AsString/g' {} +

Fix fmt v12 Compatibility in slang:

find third-party/slang-elab/third_party/slang -type f -exec sed -i 's/<fmt\/core.h>/<fmt\/format.h>/g' {} +
sed -i '1i #include <fmt/format.h>' third-party/slang-elab/third_party/slang/include/slang/util/CommandLine.h

Disable Broken Developer Unit Tests (Prevents static Boost linking errors):

echo "" > src/stt/test/cpp/CMakeLists.txt
echo "" > src/dst/test/cpp/CMakeLists.txt
echo "" > src/rsz/test/cpp/CMakeLists.txt
echo "" > src/syn/test/CMakeLists.txt
```
Disable LTO (Prevents GCC ifcvt Segmentation Faults during linking):
```
sed -i 's/CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE/CMAKE_INTERPROCEDURAL_OPTIMIZATION FALSE/g' CMakeLists.txt
sed -i 's/CMAKE_INTERPROCEDURAL_OPTIMIZATION ON/CMAKE_INTERPROCEDURAL_OPTIMIZATION OFF/g' CMakeLists.txt
export CFLAGS="-fno-lto"
export CXXFLAGS="-fno-lto"
```

5. Compile and Install

Bind the compiler to your device's Performance/Prime cores (e.g., cores 4-7) to maximize speed while preventing Android OOM crashes.
```
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release -DCMAKE_INTERPROCEDURAL_OPTIMIZATION=OFF
taskset -c 4-7 make -j4
sudo make install
```

# ⚙️ Building the 8-bit Barrel Shifter (RTL-to-GDSII)

Now that the tools are installed, we will use the OpenROAD-flow-scripts (ORFS)
to synthesize a custom 8-bit CMOS Barrel Shifter using the open-source
SkyWater 130nm node.

1. Clone the Flow Scripts
```
cd ~
git clone --recursive https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts.git
cd OpenROAD-flow-scripts/flow
```

2. Create the Project Files

Create the necessary directories:
```
mkdir -p designs/src/barrel_shifter
mkdir -p designs/sky130hd/barrel_shifter
```
Create the Verilog RTL (designs/src/barrel_shifter/barrel_shifter.v):
```
module barrel_shifter (
    input wire clk,
    input wire rst,
    input wire [7:0] data_in,
    input wire [2:0] shift_amt,
    output reg [7:0] data_out
);
    always @(posedge clk or posedge rst) begin
        if (rst)
            data_out <= 8'b0;
        else
            data_out <= data_in << shift_amt;
    end
endmodule
```
Create the Timing Constraints (designs/src/barrel_shifter/barrel_shifter.sdc):
```
set_units -time ns -capacitance pF -current mA -voltage V -resistance kOhm
create_clock -name clk -period 10.0 [get_ports clk]
set_input_delay 2.0 -clock clk [all_inputs]
set_output_delay 2.0 -clock clk [all_outputs]
```
Create the Configuration File (designs/sky130hd/barrel_shifter/config.mk):
```
export DESIGN_NAME = barrel_shifter
export PLATFORM    = sky130hd
export VERILOG_FILES = $(sort $(wildcard ./designs/src/$(DESIGN_NAME)/*.v))
export SDC_FILE      = ./designs/src/$(DESIGN_NAME)/$(DESIGN_NAME).sdc
export DIE_AREA      = 0 0 50 50
export CORE_AREA     = 5 5 45 45
export PLACE_DENSITY = 0.60
```

3. Run the Flow

Execute the flow, pointing it to your installed binaries:
```
make DESIGN_CONFIG=designs/sky130hd/barrel_shifter/config.mk YOSYS_EXE=~/oss-cad-suite/bin/yosys OPENROAD_EXE=/usr/local/bin/openroad
```
🎨 Viewing the GUI Layout

Once the flow completes successfully, you can view the physical layout of your
chip (including standard cells, power rings, and metal routing) using the
OpenROAD GUI.

Run this command from the flow directory:

openroad -gui -db results/sky130hd/barrel_shifter/base/6_final.odb

  - Use the Display Control panel to toggle metal layers (met1, met2) and
    instances.
  - The final manufacturing blueprint is saved as
    results/sky130hd/barrel_shifter/base/6_final.gds.

Acknowledgments

  - The OpenROAD Project
  - YosysHQ / OSS CAD Suite
  - Arch Linux ARM & Termux Community

