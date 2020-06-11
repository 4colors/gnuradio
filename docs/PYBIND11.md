# Generating/maintaining Python bindings using Pybind11

As of the GNU Radio 3.9 release, python bindings are handled using pybind11,
which is inherently different than they were in previous releases

## Dependencies

- pybind11 > 2.4.3 https://pybind11.readthedocs.io/
- pygccxml https://pygccxml.readthedocs.io/en/develop/install.html
  - This is an optional dependency and basic functionality for OOT generation can be performed without pygccxml
  - It is required for automatically generating bindings for most of the GR source tree

## Components

Python bindings are contained in the `python/.../bindings` directory

```sh
./python
└── module_name
    ├── bindings
    │   ├── blockname1_python.cc
    │   ├── blockname2_python.cc
    │   ├── CMakeLists.txt
    │   ├── docstrings
    │   │   ├── blockname1_pydoc_template.h
    │   │   ├── blockname1_pydoc_template.h
```

The bindings for each block exist in blockname_python.cc under the `python/bindings` directory.  Additionally, a template header file for each block that is used as a placeholder for the scraped docstrings lives in the `docstrings/` dir

### blockname_python.cc

#### Comment Block

Each block binding file contains an automatically generated and maintained comment block that informs CMake when the bindings are out of sync with the header file they refer to, and what to do about it

```cpp
/***********************************************************************************/
/* This file is automatically generated using bindtool and can be manually edited  */
/* The following lines can be configured to regenerate this file during cmake      */
/* If manual edits are made, the following tags should be modified accordingly.    */
/* BINDTOOL_GEN_AUTOMATIC(0)                                                       */
/* BINDTOOL_USE_PYGCCXML(0)                                                        */
/* BINDTOOL_HEADER_FILE(basic_block.h)                                             */
/* BINDTOOL_HEADER_FILE_HASH(549c06530e2afdf6f2c989017cb5f36e)                     */
/***********************************************************************************/
```

`BINDTOOL_GEN_AUTOMATIC`: Many times for complex in-tree blocks, the automated tools are not entirely sufficient to generate all of the bindings in an automated fashion.  In this case, the flag should be set to 0, and the bindings need to be updated manually.  If the flag is set to 1, CMake will override the binding file *in the source tree* when it detects out of sync bindings.  This should only be done in simple cases.

`BINDTOOL_USE_PYGCCXML`: Currently there are limitations on the amount of code generation that can be accomplished without the `pygccxml` dependency.  If a block needs pygccxml for the bindings to be properly generated automatically, this should be set to `1`

`BINDTOOL_HEADER_FILE`: The header file that bindings are based on, filename only

`BINDTOOL_HEADER_FILE_HASH`: The MD5 hash of the header file that the bindings are based on.  If minor changes are made to the header file that don't require regeneration of the bindings, this hash can just be updated as the value returned from `md5hash [header_filename].h`

## Workflow

### Out-of-Tree modules

The steps for creating an out of tree module with pybind11 bindings are as follows:

1. Use `gr_modtool` to create an out of tree module and add blocks

```sh
gr_modtool newmod foo
gr_modtool add bar
```

2. Update the parameters or functions in the public include file and rebind with `gr_modtool bind bar`

**NOTE**: without pygccxml, only the make function is currently accounted for, similar to `gr_modtool makeyaml`

If the public API changes, just call `gr_modtool bind [blockname]` to regenerate the bindings

When the public header file for a block is changed, CMake will fail as it checks the hash of the header file compared to the hash stored in the bindings file until the bindings are updated

3. Build and install

### In-Tree Modules

Generating bindings for in-tree modules is currently a bit more manual, as they are not expected to change that frequently.  Pygccxml **IS** required for generating these bindings.

Currently the best way to approach this is via the script `gr-utils/bindtool/scripts/bind_gr_module.py` to generate bindings for an entire gr module (e.g. gr-digital), or `gr-utils/bindtool/scripts/bind_intree_file.py` for a single block

```sh
python3 /path/to/gr-utils/bindtool/scripts/bind_gr_module.py --prefix=[GR PREFIX (e.g. ~/gr)] --output_dir /tmp/bindtool modulename
```

where `modulename` is:

```sh
gr, pmt, blocks, digital, analog, fft, filter, ...
```

See notes in `bind_gr_module.py` for instructions for modules that depend on additional include directories, such as `uhd` and `qtgui`

## Docstrings

If Doxygen is enabled in GNU Radio and/or the OOT, Docstrings are scraped from the header files, and placed in auto-generated
`[blockname]_pydoc.h` files in the build directory on compile.  Generated templates (via the binding steps described above) are placed in
the `python/bindings/docstrings` directory and are used as placeholders for the scraped strings

Upon compilation, docstrings are scraped from the module and stored in a dictionary (using `update_pydoc.py scrape`) and then
the values are substituted in the template file (using `update_pydoc.py sub`)