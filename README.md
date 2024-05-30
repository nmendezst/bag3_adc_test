## Testing the BAG3 ADC generator on Ubuntu 22.04.4 LTS


### 1. Install conda (I used Miniconda)

```bash

wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh

```
	
### 2. Clone the SKY130 workspace and initialize all submodules

```bash

git clone https://github.com/ucb-art/bag3_skywater130_workspace
cd bag3_skywater130_workspace
git submodule update --init --recursive

```

### 3. Create a conda environment from the workspace

```bash

conda env create -f environment.yml

```


### 4. Download the required packages and install them on the created environment 

By default should be in: 

```bash

/home/${USER}/miniconda3/envs/bag_py3d7_c 

```

#### cmake 3.17

```bash

wget https://github.com/Kitware/CMake/releases/download/v3.17.0/cmake-3.17.0.tar.gz
tar -xvf cmake-3.17.0.tar.gz
cd cmake-3.17.0
./bootstrap --prefix=/home/${USER}/miniconda3/envs/bag_py3d7_c  --parallel=4
make -j4
sudo make install

```

#### magic_enum

```bash

git clone https://github.com/Neargye/magic_enum.git
cd magic_enum
cmake -H. -Bbuild -DCMAKE_BUILD_TYPE=Release -DMAGIC_ENUM_OPT_BUILD_EXAMPLES=FALSE -DMAGIC_ENUM_OPT_BUILD_TESTS=FALSE -DCMAKE_INSTALL_PREFIX=/home/${USER}/miniconda3/envs/bag_py3d7_c
cmake --build build
cd build
sudo make install

```

#### yaml-cpp

```bash

git clone https://github.com/jbeder/yaml-cpp.git
cd yaml-cpp
cmake -B_build -H. -DCMAKE_POSITION_INDEPENDENT_CODE=ON -DCMAKE_INSTALL_PREFIX=/home/${USER}/miniconda3/envs/bag_py3d7_c
sudo cmake --build _build --target install -- -j 4

```

#### libfyaml

```bash
git clone https://github.com/pantoniou/libfyaml.git
cd libfyaml
./bootstrap.sh
./configure --prefix=/home/${USER}/miniconda3/envs/bag_py3d7_c
make -j12
sudo make install
```

#### HDF5 1.10


```bash
wget https://support.hdfgroup.org/ftp/HDF5/releases/hdf5-1.10/hdf5-1.10.6/src/hdf5-1.10.6.tar.gz
tar -xvf hdf5-1.10.6.tar.gz
cd hdf5-1.10.6
./configure --prefix=/home/${USER}/miniconda3/envs/bag_py3d7_c
make -j24
sudo make install
```

#### Boost

```bash

wget https://boostorg.jfrog.io/artifactory/main/release/1.72.0/source/boost_1_72_0.tar.gz
tar -xvf boost_1_72_0.tar.gz
cd boost_1_72_0
./bootstrap.sh --prefix=/home/${USER}/miniconda3/envs/bag_py3d7_c

```

In the resulting `project-config.jam file, change the using python line to:

```bash
using python : 3.7 : /home/${USER}/miniconda3/envs/bag_py3d7_c : =/home/${USER}/miniconda3/envs/bag_py3d7_c/include/python3.7m ;
```


If it exists, delete the line:

```bash
path-constant ICU_PATH : /usr ;
```

Run:


```bash
./b2 --build-dir=_build cxxflags=-fPIC -j8 -target=shared,static --with-filesystem --with-serialization --with-program_options install | tee install.log
```

Not sure if I did this step correctly because there wasn't a `using python` line originally on the `.jam` file. Should`--with-python` flag be added? 

### 5. Activate environment

```bash

conda activate bag_py3d7_c

```

### 6. Modify .bashrc_bag and source it

```bash

vim bag3_skywater130_workspace

```

```bash

export BAG_TOOLS_ROOT=/home/${USER}/miniconda3/envs/bag_py3d7_c
export BAG_TEMP_DIR=/home/${USER}/BAGTMP/skywater130


```

```bash

source .bashrc_bag

```

### 7. Install framework

```bash

cd bag3_skywater130_workspace/BAG_framework
python3 setup.py build
python3 setup.py install

```


### 8. Building cbag and pybag

```bash

cd BAG_framework/pybag
./run_test.sh 

```

For some reason, the following errors are present when attempting to build `cbag`:

```bash

Error: ‘numeric_limits’ is not a member of ‘std’
Error: ‘optional’ in namespace ‘std’ does not name a template type

```

To "solve" them, edit `typedefs.h`:

```bash

vim bag3_skywater130_workspace/BAG_framework/pybag/cbag/include/cbag/common/typedefs.h

```

Add the following lines at the preamble:


```c

#include <limits>
#include <optional>

```

When building `pybag`, a few multiple definitions error occurs: 

```bash

/usr/bin/ld: name_unit.cpp.o (symbol from plugin): in function `cbag::spirit::name_unit()':
(.text+0x0): multiple definitions of `cbag::spirit::parser::check_str'; name.cpp.o (symbol from plugin):(.text+0x0): first defined here
/usr/bin/ld: name_unit.cpp.o (symbol from plugin): in function `cbag::spirit::name_unit()':
(.text+0x0): multiple definitions of `cbag::spirit::parser::check_zero'; name.cpp.o (symbol from plugin):(.text+0x0): first defined here
/usr/bin/ld: name_unit.cpp.o (symbol from plugin): in function `cbag::spirit::name_unit()':
(.text+0x0): multiple definitions of `cbag::spirit::parser::init_range'; name.cpp.o (symbol from plugin):(.text+0x0): first defined here

```

To "solve" them, add `static` on the functions definition


#### Modify name_unit_def.h

```bash

vim pybag/cbag/include/cbag/spirit/name_unit_def.h

```


```c

auto check_str ->
auto static check_str

```

#### Modify range_def.h

```bash

vim pybag/cbag/include/cbag/spirit/range_def.h

```

```c

auto init_range ->
auto static init_range

```

```c

auto check_zero ->
auto static check_zero

```

```bash

ImportError: /home/${USER}/miniconda3/envs/bag_py3d7_c/bin/../lib/libstdc++.so.6: version `GLIBCXX_3.4.29' not found (required by /home/${USER}/asic/bag3_skywater130_workspace/BAG_framework/pybag/_build/lib/pybag/core.cpython-37m-x86_64-linux-gnu.so)

```
Update `Libgcc`

```bash

 conda install anaconda::libgcc

```


### 9. Generate an inverter as test

```bash

./gen_cell.sh data/bag3_digital/specs_blk/inv_chain/gen.yaml -raw -netlist


```

```bash

WARNING: Error registering BLOSC filter for HDF5.  Default to LZF
creating BAG project
*WARNING* invalid literal for int() with base 10: ''.  Operating without Virtuoso.
RuntimeError: module compiled against API version 0xe but this version of numpy is 0xd

```

Update `numpy` version of `conda`, with environment activated, and `./run_test.sh` again

```bash

pip install numpy --upgrade

```

```bash

WARNING: Error registering BLOSC filter for HDF5.  Default to LZF
creating BAG project
*WARNING* invalid literal for int() with base 10: ''.  Operating without Virtuoso.
computing layout...
[2024-05-29 16:50:09.996] [STDCellWrapper] [warning] ports on private layer 0 detected, converting to primitive ports.
[2024-05-29 16:50:09.997] [STDCellWrapper] [warning] ports on private layer 0 detected, converting to primitive ports.
[2024-05-29 16:50:09.997] [STDCellWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-29 16:50:09.997] [STDCellWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
computation done.
creating layout...
Traceback (most recent call last):
  File "BAG_framework/run_scripts/gen_cell.py", line 82, in <module>
    run_main(_prj, _args)
  File "BAG_framework/run_scripts/gen_cell.py", line 68, in run_main
    gen_netlist=args.gen_netlist)
  File "/home/nmendez/asic/bag3_skywater130_workspace/BAG_framework/src/bag/core.py", line 479, in generate_cell
    square_bracket=square_bracket)
  File "/home/nmendez/asic/bag3_skywater130_workspace/BAG_framework/src/bag/layout/template.py", line 153, in batch_layout
    self.batch_output(output, info_list, **kwargs)
  File "/home/nmendez/asic/bag3_skywater130_workspace/BAG_framework/src/bag/util/cache.py", line 720, in batch_output
    lay_map = get_gds_layer_map()
  File "/home/nmendez/asic/bag3_skywater130_workspace/BAG_framework/src/bag/env.py", line 223, in get_gds_layer_map
    raise ValueError(f'{ans} is not a file.')
ValueError: /home/nmendez/asic/bag3_skywater130_workspace/skywater130/gds_setup/gds.layermap is not a file.

> /home/nmendez/asic/bag3_skywater130_workspace/BAG_framework/src/bag/env.py(223)get_gds_layer_map()
-> raise ValueError(f'{ans} is not a file.')
(Pdb) 

```

[gds.layermap](https://github.com/ucb-art/skywater-pdk-libs-sky130_bag3_pr/blob/d1979837776f4a8beddf0fd9cda3a2bb1dbb8d72/gds_setup/gds.layermap) is a symbolic link, which points to:

```bash
../workspace_setup/PDK/VirtuosoOA/libs/s8phirs_10r/s8phirs_10r.layermap

```

There is an open source version of the layermap file ([thanks to Felicia Guo](https://open-source-silicon.slack.com/archives/C016HUV935L/p1717040627916339?thread_ts=1716649165.989099&cid=C016HUV935L)) [Linen.dev mirror](https://web.open-source-silicon.dev/t/18845398/channel-quickie-challenge-per-the-discussion-above-friday-ma#0b1ccee0-dc1b-4b79-9815-7d4076341b2e)

```bash

cd skywater130
git fetch
git pull origin open

```

```bash
WARNING: Error registering BLOSC filter for HDF5.  Default to LZF
creating BAG project
*WARNING* invalid literal for int() with base 10: ''.  Operating without Virtuoso.
computing layout...
[2024-05-30 13:45:50.786] [STDCellWrapper] [warning] ports on private layer 0 detected, converting to primitive ports.
[2024-05-30 13:45:50.786] [STDCellWrapper] [warning] ports on private layer 0 detected, converting to primitive ports.
[2024-05-30 13:45:50.786] [STDCellWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 13:45:50.786] [STDCellWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
computation done.
creating layout...
layout done: time taken = 0.005554666000080033
computing schematic...
computation done.
creating netlist...
netlisting done: time taken = 0.03081732300006479
```

Output files are in `gen_outputs\inv_chain_sch`, there is a `.cdl` netlist, a `.gds` layout and a `.log` report.

`gdstk` render of the `.gds` layout:

```python
import pathlib
import gdstk

library = gdstk.read_gds("AA_inv_chain.gds")
top_cells = library.top_level()
top_cells[0].write_svg("AA_inv_chain.svg")

```


<img src="https://github.com/nmendezst/bag3_adc_test/blob/08512f6f68c847fb9f52490dd249297b16a05caa/AA_inv_chain.svg"  width=75% height=75%>



## 10. Testing the ADC generator

In the `bag3_skywater130_workspace` directory:

```bash

git clone https://github.com/ucb-art/bag3_sync_sar_adc

```

```bash

cd data
git clone https://github.com/ucb-art/bag3_sync_sar_adc_data_skywater130 bag3_sync_sar_adc

```
Modify `.bashrc_pypath` to add the following line:
```bash

export PYTHONPATH="${PYTHONPATH}:${BAG_WORK_DIR}/bag3_sync_sar_adc/src"

```



To generate an top level 8-bit SAR ADC:

```bash

./gen_cell.sh bag3_sync_sar_adc/data/specs_gen/sar_lay/specs_slice_sync_bootstrap.yaml -raw -netlist

```

<details> 
  <summary>Log </summary>
  
```bash  
WARNING: Error registering BLOSC filter for HDF5.  Default to LZF
creating BAG project
*WARNING* invalid literal for int() with base 10: ''.  Operating without Virtuoso.
computing layout...
[2024-05-30 16:23:13.290] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:13.291] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:13.291] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:13.473] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:13.474] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:13.476] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:13.477] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:13.504] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:19.506] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:19.506] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:19.689] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:19.690] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:19.692] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.305] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.509] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.511] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.577] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.577] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.578] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.578] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.579] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.644] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.645] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.645] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.686] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.686] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.686] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.687] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.687] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.716] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.717] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.717] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.759] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.760] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.760] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.760] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.761] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.787] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.787] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.788] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.805] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.805] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.806] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.806] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.806] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.818] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.819] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.819] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.844] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.844] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.844] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.845] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.845] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.857] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.857] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.858] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.875] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.876] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.876] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.876] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.876] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.889] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.890] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.890] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.915] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.915] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.915] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.916] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.916] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.934] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.934] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.934] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.992] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.993] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:26.994] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
TIDX LIST:  [HalfInt(31.5), HalfInt(32.5), HalfInt(33.5), HalfInt(34.5), HalfInt(35.5), HalfInt(36.5), HalfInt(37.5), HalfInt(38.5), HalfInt(39.5), HalfInt(40.5), HalfInt(41.5), HalfInt(42.5), HalfInt(43.5), HalfInt(44.5), HalfInt(45.5), HalfInt(46.5), HalfInt(47.5), HalfInt(48.5), HalfInt(49.5), HalfInt(53.5), HalfInt(54.5), HalfInt(55.5), HalfInt(56.5)]
TIDX LIST:  [HalfInt(63.5), HalfInt(64.5), HalfInt(65.5), HalfInt(66.5), HalfInt(67.5), HalfInt(68.5), HalfInt(69.5), HalfInt(70.5), HalfInt(71.5), HalfInt(72.5), HalfInt(73.5), HalfInt(74.5), HalfInt(75.5), HalfInt(76.5), HalfInt(77.5), HalfInt(78.5), HalfInt(79.5), HalfInt(80.5), HalfInt(81.5), HalfInt(85.5), HalfInt(86.5), HalfInt(87.5), HalfInt(88.5)]
TIDX LIST:  [HalfInt(95.5), HalfInt(96.5), HalfInt(97.5), HalfInt(98.5), HalfInt(99.5), HalfInt(100.5), HalfInt(101.5), HalfInt(102.5), HalfInt(103.5), HalfInt(104.5), HalfInt(105.5), HalfInt(106.5), HalfInt(107.5), HalfInt(108.5), HalfInt(109.5), HalfInt(110.5), HalfInt(111.5), HalfInt(112.5), HalfInt(113.5), HalfInt(117.5), HalfInt(118.5), HalfInt(119.5)]
[2024-05-30 16:23:28.050] [MOSBaseTapWrapper] [warning] ports on private layer 0 detected, converting to primitive ports.
[2024-05-30 16:23:28.050] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.051] [MOSBaseTapWrapper] [warning] ports on private layer 0 detected, converting to primitive ports.
[2024-05-30 16:23:28.051] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.052] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.052] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.052] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.052] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.052] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.052] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.053] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.053] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.053] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.053] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.053] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.053] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.053] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.053] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.053] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.053] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.053] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.053] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.053] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.053] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.053] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.053] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.053] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.053] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.054] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.054] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.054] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.054] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.054] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.054] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.054] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.054] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.054] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.054] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.054] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.054] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.054] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.054] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.054] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.054] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.054] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.054] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.054] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.054] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.054] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.054] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.054] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.054] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.055] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.055] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.055] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.055] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.055] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.055] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.055] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.055] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.055] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.055] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.055] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.055] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.055] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.055] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.055] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.055] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.055] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.055] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.118] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.119] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.330] [MOSBaseTapWrapper] [warning] ports on private layer 0 detected, converting to primitive ports.
[2024-05-30 16:23:28.331] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.331] [MOSBaseTapWrapper] [warning] ports on private layer 0 detected, converting to primitive ports.
[2024-05-30 16:23:28.331] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.331] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.331] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.331] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.331] [MOSBaseTapWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.367] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.367] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.446] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.446] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.446] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.446] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.446] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.446] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.446] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.446] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.446] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.470] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.470] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.471] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.471] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.527] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.552] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.552] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.553] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.553] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.568] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
[2024-05-30 16:23:28.568] [GenericWrapper] [warning] ports on private layer 2 detected, converting to primitive ports.
computation done.
creating layout...
layout done: time taken = 0.11354740400020091
computing schematic...
/home/nmendez/asic/bag3_skywater130_workspace/bag3_sync_sar_adc/src/bag3_sync_sar_adc/schematic/bootstrap.py:85: UserWarning: Doesn't implement dummy sampling sw
  warnings.warn("Doesn't implement dummy sampling sw")
CM:  1
computation done.
creating netlist...
netlisting done: time taken = 0.03970662599931529
```
</details>



<img src="https://github.com/nmendezst/bag3_adc_test/blob/b22365b5a32257534b4df5bcc5ff6597e5788b59/AAA_Slice_sync.svg"  width=75% height=75%>


### References

#### Official documentation
[BAG3++ 1.0 documentation](https://bag3-readthedocs.readthedocs.io/en/latest/)

[BAG3 SAR ADC Setup](https://github.com/ucb-art/bag3_sync_sar_adc/blob/main/docs/source/setup.rst)

[gdstk](https://heitzmann.github.io/gdstk/)


#### Repositories
[BAG3++ Skywater 130 Workspace](https://github.com/ucb-art/bag3_skywater130_workspace)

[Skywater primitives for BAG](https://github.com/ucb-art/skywater-pdk-libs-sky130_bag3_pr/tree/d1979837776f4a8beddf0fd9cda3a2bb1dbb8d72)

[BAG3 SAR ADC Generator](https://github.com/ucb-art/bag3_sync_sar_adc)

[BAG3 SAR ADC Generator](https://github.com/ucb-art/skywater130_bag3_sar_adc)



#### Collective wisdom of the ancients

[‘numeric_limits’ is not a member of ‘std’](https://stackoverflow.com/questions/71296302/numeric-limits-is-not-a-member-of-std)

[Multiple definition errors during gcc linking in Linux](https://stackoverflow.com/questions/69326932/multiple-definition-errors-during-gcc-linking-in-linux)

[RuntimeError: module compiled against API version a but this version of numpy is 9](https://stackoverflow.com/questions/33859531/runtimeerror-module-compiled-against-api-version-a-but-this-version-of-numpy-is)

