# PACE for Python and user defined modelling functions - a design discussion document
**2020-02-24**

<!---
LaTeX equations used repeatedly can be defined as references, with format
	[latex_equation]: %-encoded link to image generator
then, if image generation fails for some reason, the alt-text for each equation
will still contain the raw LaTeX equation which is often intelligible

Constructing the %-encoded link can be done at, e.g., the codecogs website
	https://www.codecogs.com/eqnedit.php?latex=latex_equation
--->
[\mathbb{Q} \equiv (\mathb{Q},E)]:https://latex.codecogs.com/svg.latex?%5Cmathbb%7BQ%7D%5Cequiv%28%5Cmathbf%7BQ%7D%2CE%29
[S(\mathbb{Q})]: https://latex.codecogs.com/svg.latex?S%28%5Cmathbb%7BQ%7D%29
[\mathbf{Q}]: https://latex.codecogs.com/svg.latex?%5Cmathbf%7BQ%7D
[N]: https://latex.codecogs.com/svg.latex?N
[(E_n, S(\mathbf{Q}, E_n))]: https://latex.codecogs.com/svg.latex?%28E_n%2C%20S%28%5Cmathbf%7BQ%7D%2C%20E_n%29%29


# Scope

This documents presents the results of investigations into possible implementations of a Python user interface to PACE (to Horace/Herbert written in Matlab), and for the API design to accommodate user-defined model functions.


# Motivation

Although there are significant C++ components in the Horace software which will form the core of PACE, the major "glue" between these components is written in Matlab.
It is not feasible for all of this codebase to be ported to Python (or entirely to C++) within the time-frame of the PACE project.
Nonetheless one of the main goal of the PACE project is to provide a Python user interface which *would not require a Matlab license* to use.
In addition, the analysis of inelastic neutron scattering data almost always requires a parameterised model of the measured scattering intensities.
In the current implementation, this model function must be a Matlab function.
For the Python user interface, naturally a Python function would be preferred, but the API between this and the Matlab code needs to be defined.
Furthermore, support should be provided for the user to use functions or subroutines defined in a compiled library (e.g. written in C/C++/Fortran) for computational speed reasons.


# Overview

The [first section](#mat2py) presents details of the proposed Python user interface to the Matlab-based Horace code.
This uses the **Matlab Compiler SDK** to compile the Horace code into a distributable Python module.  
To use this, the user is required to install the free/gratis **Matlab Compiler Runtime** package, and this **MCR** package must match the version of Matlab used to compile the Horace library.
Thus, if we always target the latest version of Matlab, we should require users to re-install (upgrade) the **MCR** twice a year, which may not be desirable.

We next discuss the required [function signatures for user defined model functions](#userfundef).
This follows closely the [new design for the optimisation component](Model_Optimisation_Design.md) but is more prescriptive due to the limitations described in the following sections, particularly for the compiled libraries.
These sections describe, in order, the allowed [Matlab](#matuserfun), [Python](#pyuserfun) and [compiled library](#compileduserfun) function signatures.
Finally we give the example of two built-in models, [`euphonic`](#euphonic) and [`spinW`](#spinW). 

We end with a [summary and discussion points](summary).
Prototype code and examples are in [this repository](https://github.com/mducle/hugo).


# <a name="mat2py"></a> The Matlab-Python interface

The only way to run Matlab code without a Matlab license is for the code authors to compile it into either an executable or library with the **Matlab Compiler** toolbox.
(NB. There is an alternative toolbox call the **Matlab Coder** which is discussed in [this appendix](#matlabcoder) which is probably not appropriate for PACE).
In both cases a platform independent `ctf`-extension file is generated together with either an executable or platform dependent shared library wrapper.
Again, in both cases users are required to install a platform-dependent **Matlab Compiler Runtime** (**MCR**) which is an approximately 1GB compressed download.
As the software developer, we can also bundle the **MCR** with downloads of Python versions of Horace.

Recent versions of the **Matlab Compiler SDK** can generate a Python module, which comprises the `ctf` file and a thin (platform-independent) Python wrapper, leaving the main Python interface code in the **MCR**.
The proposed Python interface to Horace envisages using this method with a Python wrapper around this Matlab provided module to make the user syntax more natural.
*In this case the compiled Horace library is loaded into the memory of the Python process and runs within the Python process.*
Previous solutions used the **Matlab Compiler** to generate an executable which runs as a separate process and then used various methods to facilitate communications between the Python and Horace processes.
This is similar to how the **Matlab Engine API for Python** toolbox works.
This toolbox allows users (with a full Matlab installation and license) to run a separate Matlab process and to use Python to run Matlab calculations through an interprocess-communications (IPC) bridge.
The **Matlab Engine API for Python** appears to use pipes to communicate. 
The original implementation of a python interface to `spinW` also used a compiled Matlab executable, but then used network sockets for IPC.

The user-facing interface between Matlab and Python for both cases (in-process compiled library vs out-of-process engine or compiled executable) is similar.
Both require the Matlab Python module to be imported and the Matlab interpreter initialized.
For the out-of-process case, this means starting a background Matlab process, whilst the in-process library means loading it into memory.
*In both cases, this takes several seconds to execute.*
It may be possible to lazy-evaluate this so that it is transparent to users in an interactive session.

The rest of this document assumes that we will use the in-process compiled library.


## Type conversion

By default the built-in Matlab-Python interface automatically converts standard scalar data types between Matlab and Python as [detailed here](https://www.mathworks.com/help/matlab/matlab_external/handle-data-returned-from-matlab-to-python.html).
For arrays, the user must manually convert Python `list`s or `numpy.array`s to a Matlab array before passing it to the Matlab function, and the Matlab array type is returned.
(For PACE we envisage writing a wrapper layer which would do this conversion for standard types automatically.)
The sole exception is a `matlab.object` which may only be generated by a Matlab function and is passed directly to Python.
This object is nearly opaque to Python (it contains only a reference to the `mxArray` container and very few methods), however it may be passed back to another Matlab function without conversion.

The Matlab functions *only accept* standard data type scalars, Matlab arrays or `matlab.object`s as input.
That is, anything else, such as Python objects or Python function references (which it cannot convert to an equivalent Matlab type) are not allowed and will raise an error.
Nonetheless because the Matlab library is run within the Python process and so share its memory, we may pass the address of a Python object to Matlab as an `int` and manipulate it with a `mex` file compiled with Python bindings.
(Conversely we could also the `ctypes` library to interrogate the `matlab.object` from Python, if we have detailed knowledge of its structure.
This would, however, be fragile and dependent on the Matlab back-end implementation detail which is subject to change from version to version.)

Finally we note that whilst new-style Matlab classes defined with the `classdef` keyword are passed to Python (and back) as a `matlab.object`, old-style classes (defined only by a `@`directory) as converted to a Python `dict` and lose all connections with their methods.
As such in the demonstration implementation we do not pass these classes to Python, but keep them within the `base` Matlab workspace and pass only the object name (as a unique string) to Python.


## The compilation process

There are two main Matlab functions to compile code to a library or executable; a low level compiler, `mcc` and a higher level `deploytool`.
`deploytool



## Example syntax

