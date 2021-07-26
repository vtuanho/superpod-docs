Module installation guide
===

### Miniconda installation and usage

* Miniconda installation
Download the Miniconda installer using the wget command and run the installer, pointing it to the directory where you want to install it. Recommend install software `/sw/software` for easy integration into user defined environment modules.

```sh
wget https://repo.continuum.io/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash ./Miniconda3-latest-Linux-x86_64.sh -b -p /sw/software -s
```

* Miniconda environment module
Set up the Miniconda environment, create an [user environment module](https://www.chpc.utah.edu/documentation/software/modules-advanced.php#custom). First create a directory where the user environment module hierarchy will reside, and then copy our miniconda module file to this directory. Recommend save module file into folder `/sw/modules/all/`.

* Example, module file miniconda3
```sh
  
-- -*- lua -*-

help([[Module which sets the PATH variable for user instaled miniconda 
       To check the available packages: conda list
]])
-- change myanapath if the installation is in a different place in your home
local version = "miniconda3"
local myana = "/sw/software/miniconda3"
setenv("PYTHONPREFIX",myana)
prepend_path("PATH",pathJoin(myana,"bin"))

whatis("Name         : Miniconda " .. version .. " Python 3")
whatis("Version      : " .. version .. " & Python 3")
whatis("Category     : Compiler")
whatis("Description  : Python environment")
whatis("URL          : https://conda.io/miniconda")
whatis("Installed on : --- ")
whatis("Modified on  : --- ")
whatis("Installed by : ---")

family("python")
```

* [For more information](https://modules.readthedocs.io/en/latest/INSTALL.html)