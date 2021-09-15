---
title: 'ogs6py and VTUinterface: streamlining OpenGeoSys workflows in Python'
tags:
  - Python
  - physics
  - THMC
  - VTU
  - time-series
  - sensitivity analysis
  - uncertainty quantification
  - OpenGeoSys

authors:
  - name: Jörg Buchwald^[corresponding author]
    orcid: 0000-0001-5174-3603
    affiliation: "1, 2" # (Multiple affiliations must be quoted)
  - name: Olaf Kolditz
    orcid: 0000-0002-8098-4905
    affiliation: "1, 3, 4"
  - name: Thomas Nagel
    orcid: 0000-0001-8459-4616
    affiliation: "2, 4"
affiliations:
 - name: Helmholtz Center for Environmental Research - UFZ, Leipzig, Germany
   index: 1
 - name: Technische Universität Bergakademie Freiberg, Germany
   index: 2
 - name: Technische Universität Dresden, Germany
   index: 3
 - name: TUBAF-UFZ Center for Environmental Geosciences, Germany
   index: 4
date: 12 May 2021
bibliography: paper.bib

---

# Summary

We introduce two new Python modules that facilitate the pre- and post-processing of finite element calculations. [ogs6py](https://github.com/joergbuchwald/ogs6py) is a Python interface for the open-source package OpenGeoSys [@Bilke2019], a finite element code for simulation of multi-field processes in fractured porous media. Modeling workflows can be further streamlined in Jupyter Notebooks using the newly developed [VTUinterface](https://github.com/joergbuchwald/VTUinterface).
The usage of the modules is demonstrated with common workflow operations, including parameter variations, boundary conditions, solver settings, verification of simulation results by comparison to analytical solutions, set-up and evaluation of ensemble runs, convenient analysis of results by line plots, time series, or transient contour plots.

# Statement of need

Python has become a widely used framework for scientific data analysis and modeling. The development is driven by ease of use and flexibility, the vast modular ecosystem including powerful plotting libraries, and the Jupyter Notebook technology. The attractiveness of Python is not limited to post-processing; pre-processing tasks can simply be conducted, using packages as the Python wrapper for GMSH [@geuzaine2009gmsh] and the tool meshio [@nico_schlomer_2021_4745399]. 
While many existing open-source tools force the user to learn a new syntax for interacting with the software, Python bindings allow control in a general language and thus more accessible for a wider community of users.

In this contribution, we address interaction with the open-source code OpenGeoSys (OGS) [@Bilke2019] version 6, aiming to facilitate both pre-and post-processing workflows with Python. This aim was partly inspired by the desire to design, control and evaluate ensemble runs [@Buchwald2020;@Chaudhry2021] but has now taken on a wider perspective for general usability. A similar Python interface "ogs5py" exists for OGS version 5 [@muller2021ogs5py]; however, conceptual differences between the versions, for example, the use of XML input files, required an entirely new package to be built from scratch.

The standard output format of OpenGeoSys is VTK unstructured grid files (VTU) as time slices stacked together by a PVD file. These can be analyzed using Paraview [@ahrens2005paraview], a Python wrapper for VTK [@schroeder2000visualizing], or visualization tools like PyVista [@sullivan2019pyvista] or Mayavi [@ramachandran2011mayavi]. However, a finite-element-modeller's _bread and butter_ business often include extracting single- or multiple point time-series data. The direct use of the VTK library is quite cumbersome for such tasks, especially when interpolation is required. The mentioned Python packages focus on visualization aspects, and except for Paraview, to our knowledge, the mentioned packages do not have file support for PVD files or time-series data[@pvdissue; @timeseriesissue].

# Features

ogs6py allows creating complete OGS configuration files from scratch, altering existing files, running simulations and parsing OGS log files.
The following example demonstrates some basic functionalities. The complete example demonstrating a typical ogs6py/VTUinterface workflow on a coupled thermo-hydro-mechanical (THM) problem of a tunnel excavation followed by the emplacement of a heat-emitting canister can be found in a 
[Jupyter notebook](https://github.com/joergbuchwald/joss_ogs6py_VTUinterface/blob/master/demo/paper_ogs6py_vtuio.ipynb) located in the project repository.


An instance of OGS is created, an existing project file is imported, and an output file is specified:

```python
model = OGS(INPUT_FILE="tunnel_ogs6py.prj", PROJECT_FILE="tunnel_exc.prj")
```

A project file can be altered by commands for adding blocks, removing or replacing parameters:

```python
model.replace_phase_property(mediumid=0, phase="Solid",
        name="thermal_expansivity", value=a_s)
```

or


```python
model.replace_text("tunnel_exc", xpath="./time_loop/output/prefix")
```

The project file can be written to disk:

```python
model.write_input()
```

and OGS can be executed by calling the `run_model()` method:

```python
model.run_model(path="~/github/ogs/build_mkl/bin",
        logfile="excavation.log")
```

OGS produces PVD and VTU files that can be handled with VTUinterface:

```python
pvdfile = vtuIO.PVDIO("tunnel_exc.pvd", dim=2)
```

One of the most significant features of VTUinterface is the ability to deal with PVD files as time-series data. E.g., the following command reads in the VTU point field "pressure" at point "pt0", defined in a dictionary, using nearest neighbour interpolation.

```python
excavation_curve = pvdfile.read_time_series("pressure",
        interpolation_method="nearest",  pts={"pt0": (0.0,0.0,0)})
```

The result can directly be plotted using matplotlib (\autoref{fig:1}). The time axis can be retrieved from the PVD file as well.

```python
plt.plot(pvdfile.timesteps, excavation_curve["pt0"] / 1e6)
plt.xlabel("$t$ / d")
plt.ylabel("$p$ / MPa");
```

![Plots demonstrating the usage of VTUinterface: Deconfinement curve extracted as time series from a PVD file of excavation simulation (left). Contour plot of pressure distribution generated with VTUinterface and matplotlibs `tricontourf()` shows thermal pressurization during the heating phase (right).\label{fig:1}](fig1.png){ width=100% }


![Spatial pressure distribution generated with VTUinterface from a linear point set array using three different grid interpolation methods (left). Relative convergence plot showing the numerical behaviour over ten time steps extracted using the log file parser of ogs6py (right).](fig2.png){ width=100% }


This brief overview shows only some of the functionalities coming with ogs6py and VTUinterface. Further developments will focus on extending functionalities focusing on built-in checks to ensure that only valid input files are generated.

# Technical Details

ogs6py requires python 3.9 and uses [lxml](https://lxml.de/) to process OGS6 input files and uses the subprocess module to run OGS. Furthermore, [pandas](https://pandas.pydata.org/) is required for holding OGS log file data. VTUinterface requires python 3.8 and uses the python wrapper for [vtk](https://vtk.org/) to access VTU files and lxml for PVD files. In addition to vtk's own interpolation functionalities, we use pandas and scipy for interpolation.

# Applications

Both introduced packages are relatively new, being only 1 to 2 years old. However, the adoption process in the OpenGeoSys community is gearing up. E.g., a [YouTube video](https://www.youtube.com/watch?v=eihNKjK-I-s) was published explaining their use; both tools are also used for teaching at the TU Bergakademie Freiberg and they were also extensively utilized in two recent peer-reviewed publications [@buchwald2021improved; @Buchwald2020].

# Acknowledgements

We acknowledge contributions from Tom Fischer, Dmitry Yu. Naumov, Dominik Kern and Sebastian Müller during the genesis of this project. The funding through the iCROSS-Project (Integrity of nuclear waste repository systems – Cross-scale system understanding and analysis) by the Federal Ministry of Research and Education (BMBF, grant number 02NUK053E) and Helmholtz Association (Helmholtz-Gemeinschaft e.V.) through the Impulse and Networking Funds (grant number SO-093) is greatly acknowledged. This work was in part funded by the Deutsche Forschungsgemeinschaft (DFG, German Research Foundation) under grant number NA1528/2-1.

# References
