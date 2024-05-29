## Testing the BAG3 ADC generator on Ubuntu 22.04.4 LTS


### 1. Install conda (I used Miniconda)

```bash

wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh

```
	
### 2. Clone the SKY130 workspace 

```bash

git clone https://github.com/ucb-art/bag3_skywater130_workspace
cd bag3_skywater130_workspace

```

### 3. Create a conda environment from the workspace

```bash

conda env create -f environment.yml

```


### 4. Download the required packages and install them on the created environment 

By default should be in: 

```bash

/home/user/miniconda3/envs/bag_py3d7_c 

```

#### cmake 3.17

```bash

wget https://github.com/Kitware/CMake/releases/download/v3.17.0/cmake-3.17.0.tar.gz
tar -xvf cmake-3.17.0.tar.gz
cd cmake-3.17.0
./bootstrap --prefix=/home/nmendez/miniconda3/envs/bag_py3d7_c  --parallel=4
make -j4
sudo make install

```

#### magic_enum

```bash

git clone https://github.com/Neargye/magic_enum.git
cd magic_enum
cmake -H. -Bbuild -DCMAKE_BUILD_TYPE=Release -DMAGIC_ENUM_OPT_BUILD_EXAMPLES=FALSE -DMAGIC_ENUM_OPT_BUILD_TESTS=FALSE -DCMAKE_INSTALL_PREFIX=/home/nmendez/miniconda3/envs/bag_py3d7_c
cmake --build build
cd build
sudo make install

```

#### yaml-cpp

```bash

git clone https://github.com/jbeder/yaml-cpp.git
cd yaml-cpp
cmake -B_build -H. -DCMAKE_POSITION_INDEPENDENT_CODE=ON -DCMAKE_INSTALL_PREFIX=/home/nmendez/miniconda3/envs/bag_py3d7_c
sudo cmake --build _build --target install -- -j 4

```

#### libfyaml

```bash
git clone https://github.com/pantoniou/libfyaml.git
cd libfyaml
./bootstrap.sh
./configure --prefix=/home/nmendez/miniconda3/envs/bag_py3d7_c
make -j12
sudo make install
```

#### HDF5 1.10


```bash
wget https://support.hdfgroup.org/ftp/HDF5/releases/hdf5-1.10/hdf5-1.10.6/src/hdf5-1.10.6.tar.gz
tar -xvf hdf5-1.10.6.tar.gz
cd hdf5-1.10.6
./configure --prefix=/home/nmendez/miniconda3/envs/bag_py3d7_c
make -j24
sudo make install
```

#### Boost

```bash

wget https://boostorg.jfrog.io/artifactory/main/release/1.72.0/source/boost_1_72_0.tar.gz
tar -xvf boost_1_72_0.tar.gz
cd boost_1_72_0
./bootstrap.sh --prefix=/home/nmendez/miniconda3/envs/bag_py3d7_c

```

In the resulting `project-config.jam file, change the using python line to:

```bash
using python : 3.7 : /path/to/conda/env/envname : /path/to/conda/env/envname/include/python3.7m ;
```


If it exists, delete the line:

```bash
path-constant ICU_PATH : /usr ;
```

Run:


```bash
./b2 --build-dir=_build cxxflags=-fPIC -j8 -target=shared,static --with-filesystem --with-serialization --with-program_options install | tee install.log
```

--with-python

Not sure if I did this step correctly because there wasn't a `using python` line originally on the `.jam file.

### 5. Activate environment

```bash

conda activate bag_py3d7_c

```

### 6. Modify .bashrc_bag and source it

```bash

vim bag3_skywater130_workspace

```

```bash

export BAG_TOOLS_ROOT=/home/nmendez/miniconda3/envs/bag_py3d7_c

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


### 8. Build and install pybag

For some reason, the following errors are present when attempting to build `cbag`:


- Error: ‘numeric_limits’ is not a member of ‘std’
- Error: ‘optional’ in namespace ‘std’ does not name a template type

To "solve" them, edit `typedefs.h`:

```bash

vim bag3_skywater130_workspace/BAG_framework/pybag/cbag/include/cbag/common/typedefs.h

```

Add the following lines at the preamble:


```c

#include <limits>
#include <optional>

```

```bash

cd BAG_framework/pybag
%export PYBAG_PYTHON=/home/user/miniconda3/envs/bag_py3d7_c/bin/python3
./run_test.sh 

```



When building `pybag`, a few multiple definitions error occurs: 

```bash

/usr/bin/ld: name_unit.cpp.o (symbol from plugin): en la función `cbag::spirit::name_unit()':
(.text+0x0): multiple definitions of `cbag::spirit::parser::check_str'; name.cpp.o (symbol from plugin):(.text+0x0): first defined here
/usr/bin/ld: name_unit.cpp.o (symbol from plugin): en la función `cbag::spirit::name_unit()':
(.text+0x0): multiple definitions of `cbag::spirit::parser::check_zero'; name.cpp.o (symbol from plugin):(.text+0x0): first defined here
/usr/bin/ld: name_unit.cpp.o (symbol from plugin): en la función `cbag::spirit::name_unit()':
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

ImportError: /home/nmendez/miniconda3/envs/bag_py3d7_c/bin/../lib/libstdc++.so.6: version `GLIBCXX_3.4.29' not found (required by /home/nmendez/asic/bag3_skywater130_workspace/BAG_framework/pybag/_build/lib/pybag/core.cpython-37m-x86_64-linux-gnu.so)

```
Update `Libgcc`

```bash

 conda install anaconda::libgcc

```


### 9. Generate an inverter as test

```bash

./gen_cell.sh data/bag3_digital/specs_blk/inv_chain/gen.yaml -raw -netlist


```

### References

#### Official documentation
[BAG3++ 1.0 documentation](https://bag3-readthedocs.readthedocs.io/en/latest/)

#### Collective wisdom

[‘numeric_limits’ is not a member of ‘std’](https://stackoverflow.com/questions/71296302/numeric-limits-is-not-a-member-of-std)
[Multiple definition errors during gcc linking in Linux](https://stackoverflow.com/questions/69326932/multiple-definition-errors-during-gcc-linking-in-linux)

