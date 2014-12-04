simmechanics-to-urdf [![Build Status](https://travis-ci.org/robotology-playground/simmechanics-to-urdf.svg?branch=master)](https://travis-ci.org/robotology-playground/simmechanics-to-urdf)
====================



This tool was developed to convert CAD models to URDF ( http://wiki.ros.org/urdf )  models semi-automatically. It makes use of the XML files exported by the SimMechanics Link . Mathworks, makers of SimMechanics, have developed plugins for a couple of leading CAD programs, including SolidWorks, ProEngineer and Inventor. 

More specifically, this library at the moment just support first-generation SimMechanics XML files, that in MathWorks documentation are also referred as `PhysicalModelingXMLFile` .

Based on the [original version](http://wiki.ros.org/simmechanics_to_urdf) by [David V. Lu!!](http://www.cse.wustl.edu/~dvl1/).

#### Dependencies
- [lxml](http://lxml.de/)
- [PyYAML](http://pyyaml.org/)
- [NumPy](http://www.numpy.org/)
- [urdf_parser_py](https://github.com/ros/urdfdom/tree/master/urdf_parser_py)

## Installation

#### Debian/Ubuntu
##### Install dependencies
Install the necessary dependencies with apt-get:
~~~
sudo apt-get install python-lxml python-yaml python-numpy
~~~

You can install `urdf_parser_py` from the urdfdom git repository:
~~~
git clone https://github.com/ros/urdfdom
cd 
sudo python setup.py install
~~~

##### Install simmechanics-to-urdf
You can then install `simmechanics-to-urdf`:
~~~
git clone https://github.com/robotology-playground/simmechanics-to-urdf
cd simmechanics-to-urdf
sudo python setup.py install
~~~

## How it works
The SimMechanics Link creates an XML file (PhysicalModelingXMLFile) and a collection of STL files. The XML describes all of the bodies, reference frames, inertial frames and joints for the model. The simmechanics_to_urdf script takes this information and converts it to a URDF. However, there are some tricks and caveats, which can maneuvered using a couple of parameters files. Not properly specifing this parameters files will result in a model that looks correct when not moving, but possibly does not move correctly. 

### Tree vs. Graph

URDF allows robot descriptions to only follow a tree structure, meaning that each link/body can have only one parent, and its position is dependent on the position of its parent and the position of the joint connecting it to its parent. This forces URDF descriptions to be in a tree structure.

CAD files do not generally follow this restriction; one body can be dependent on multiple bodies, or at the very least, aligned to multiple bodies.

This creates two problems.

  - The graph must be converted into a tree structure. This is done by means of a breadth first traversal of the graph, starting from the root link. However, this sometimes leads to improper dependencies, which can be corrected with the parameter file, as described below.

  - Fixed joints in CAD are not always fixed in the exported XML. To best understand this, consider the example where you have bodies A, B and C, all connected to each other. If C is connected to A in such a way that it is constrained in the X and Z dimensions, and C is connected to B so that it is constrained in the Y dimension, then effectively, C is fixed/welded to both of those bodies. Thus removing the joint between B and C (which is needed to make the tree structure) frees up the joint. This also can be fixed with the parameter file. 

## Use the script
You can call the script:
~~~
simmechanics_to_urdf {SimMechanics XML filename} --yaml [yaml_configfile] --csv-joint csv_joints_configfile --output {xml|graph|none}
~~~
The `--output` options defines the output. Selecting graph the script will output a graphviz .dot representation 
of the SimMechanics model, useful for debugging, while selecting xml it will output the converted URDF.

### Configuration files

#### YAML Parameter File
The YAML format is used to pass parameters to the script to customized the conversion process.
The file is loaded using the Python (yaml)[http://pyyaml.org/] module. 
The parameter accepted by the script are documented in the following. 


##### Naming Parameters 
| Attribute name   | Type   | Default Value | Description  |
|:----------------:|:---------:|:------------:|:-------------:|
| rename        | Map  | {} (Empty Map) | Structure mapping the SimMechanics XML names to the desired URDF names.  |


##### Root Parameters
| Attribute name   | Type   | Default Value | Description  |
|:----------------:|:---------:|:------------:|:-------------:|
| root             | String  | First body in the file | Changes the root body of the tree |

##### Mesh Parameters 
| Attribute name   | Type   | Default Value | Description  |
|:----------------:|:---------:|:------------:|:-------------:|
| `filenameformat` |  String | %s  | Used for translating the filenames in the exported XML to the URDF filenames, using a formatting string. Example: "package://my_package//folder/%sb" - resolves to the package name and adds a "b" at the end to indicate a binary stl. |
| `filenameformatchangeext` | String | %s  | Similar to filenameformat, but use to also change the file extensions and not only the path of the filenames |
| `forcelowercase` |  Boolean | False | Used for translating the filenames. Ff True, it forces all filenames to be lower case. |
| `scale` | String |  None | If this parameter is defined, the scale attribute of the mesh in the URDF will be set to its value. Example: "0.01 0.01 0.01" - if your meshes were saved using centimeters as units instead of meters.  |


#### CSV  Parameter File
Using the `--csv-joints` options it is possible to load some joint-related information from a csv
file. The rationale for using CSV over YAML for some information related to the model (for example joint limits) is to use a format that it is easier to modify  using common spreadsheet tools like Excel/LibreOffice Calc, that can be easily used also by people without a background in computer science. 

##### Format
The CSV file is loaded by loaded by the python `csv` module, so every dialect supported 
by the [`csv.Sniffer()`](https://docs.python.org/library/csv.html#csv.Sniffer) is automatically
supported by `simmechanics-to-urdf`. 

The CSV file is formed by a header line followed by several content lines, 
as in this example:
~~~
joint_name,lower_limit,upper_limit
torso_yaw,-20.0,20.0
torso_roll,-20.0,20.0
~~~

The order of the elements in header line is arbitrary, but the supported attributes
are listed in the following:

| Attribute name | Required | Unit of Measure |   Description  |
|:--------------:|:--------:|:----------------:|:---------------:|
| joint_name     |  **Yes**  |      -          | Name of the joint to which the content line is referring | 
| lower_limit    |  No      | Degrees         | `lower` attribute of the `limit` child element of the URDF `joint`. **Please note that we specify this limit here in Degrees, but in the urdf it is expressed in Radians, the script will take care of  internally converting this parameter.** |  
| upper_limit    |  No      | Degrees         | `upper` attribute of the `limit` child element of the URDF `joint`. **Please note that we specify this limit here in Degrees, but in the urdf it is expressed in Radians, the script will take care of  internally converting this parameter.** |  
| velocity_limit | No      | Radians/second    | `velocity` attribute of the `limit` child element of the URDF `joint`. |
| effort_limit | No      |  Newton meters    | `effort` attribute of the `limit` child element of the URDF `joint`.
| damping  | No      |  Newton meter seconds / radians    | `damping` of the `dynamics` child element of the URDF `joint`. |
| friction | No      |  Newton meters    | `friction` of the `dynamics` child element of the URDF `joint`. |
