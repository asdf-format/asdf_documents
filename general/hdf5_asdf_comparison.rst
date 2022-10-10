********************
HDF5/ASDF Comparison
********************

Perry Greenfield, Edward Slavich, William Jamieson, & Nadia Dencheva

* Introduction_
* `Mapping YAML Structure into HDF5`_
	* `Representing dictionaries in HDF5`_
	* `Representing heterogeneous lists in HDF5`_
	* `Disk space comparison`_
* `Discussion of the difficulties`_
	* `Lack of a general system of validation`_
	* `Lack of a standard system of extension plugins`_
	* `Comments regarding other enumerated reasons listed at the beginning`_
	* `Text-based alternatives`_
* `Areas where HDF5 has an advantage`_
* `Appendix A: Summary of relevant HDF5 Hierarchical Structures`_
* `Appendix B: GWCS Object Representation`_
* `Appendix C: ASDF GWCS Contents`_
* `Appendix D: Edited version of Appendix C`_
* `Appendix E: Portion of converted HDF5 file`_

.. _Introduction:

Introduction
============

Based on a number of questions and comments about why we aren't using HDF5, that the first ASDF paper apparently hasn't addressed suitably well, this document is an attempt to explain in more detail why it hasn't been chosen for STScI use as a data format.


For completeness, the original list of concerns (condensed) were:


1. Entirely binary.
2. Not self documenting
3. Effectively only one implementation (due to complexity)
4. Questionable as an archival format
5. Does not lend itself to pure text-based data files as an option
6. HDF5 Abstract Data Model not flexible enough.


And two additional items not enumerated in the original:


1. Lack of specification and validation mechanism.
2. Lack of a general scheme for extensions of the standard


The main focus of this document are the last three items. The other items will receive some additional commentary at the end.


The point being made here isn't whether there is a way to save arbitrary information in a given format. Virtually all data formats have some way of encoding arbitrary information. The issue is how efficiently and naturally that information can be stored.


As data sets become more complex such flexibility is important. In this comparison we use an example that drove us to develop ASDF, namely the ability to serialize complex coordinate transformations. Such transformations were needed to represent the complex distortions present in HST data, where the accuracy for distortion models needs to be at least as good as 0.01 pixel over detector sizes of up to 4 thousand pixels. (In fact, systematic errors in the residuals observed at a level of 0.003 pixels are seen.) Typically the necessary distortion models are constructed as a combination of a number of simpler models. It is important to keep these models with the data, mainly because resampling operations on the unresampled images or spectra to be combined with other unresampled images or spectra often require manipulations of some of the transform parameters to properly register the multiple images or spectra with each other.


As a test of the relative ease of storing such WCS models in HDF5, we chose a spectral WCS model currently being used for the JWST NIRSPEC instrument. In other words, this is not a contrived example! It is one being used currently for JWST data. It is also not the most complex transform that JWST is using. The transformation framework we are using allows multidimensional transforms to be constructed from all sorts of constituent models (shifts, scalings, rotations, table lookups, polynomials, etc.) This framework describes the transformations that, in principle, can be applied in any language, but in our current use relies on Astropy modeling. This example was chosen because it is one of the more complex we have. Normally this WCS is stored in ASDF as both YAML metadata and with some information (such as polynomial coefficients) as binary blocks.


This model is sizable, consisting of 118 constituent transforms (many of which are simple scaling and shifting operations at different stages of the transformation). `Appendix B`_ shows how the structure of this complex transform is rendered by Python when converted to Python objects. The Python library that handles these transformations is called the Generalized World Coordinate System (GWCS; https://gwcs.readthedocs.io/en/latest/) to contrast it with the current FITS WCS standard, which is quite limited in the transforms it supports (see: https://www.atnf.csiro.au/people/mcalabre/WCS/wcs.pdf, https://www.atnf.csiro.au/people/mcalabre/WCS/ccs.pdf, and https://www.atnf.csiro.au/people/mcalabre/WCS/scs.pdf ).


Our goal here is to illustrate an attempt to store the relevant information in HDF5 with the same organization that exists in ADSF (and is essential as it reflects the order of transformations to be applied). In no way does this exercise imply the HDF5 result is usable, since some essential information (has been lost; e.g., we do not save the tag information). What the exercise is intended to show is how awkward this conversion to HDF5 is, as well as how wasteful of space it is. Some manipulations were needed to put this model into a form suitable for saving in HDF5. To start, all array data were put in in-line representation (instead of binary) to facilitate the needed editing of the content.


`Appendix C`_ shows the corresponding YAML content that contains all the necessary information for Python to convert it into a usable GWCS object (in this case the arrays are in the binary blocks and those contents are not shown).


This editing consisted of removing the YAML tag information and any use of anchors and references. The latter are, in principle, supportable in HDF5, but implementing converters to support these was more trouble than it was worth, and it should not affect the conclusions significantly. Removing the tag information does remove important information necessary for the interpretation of the contents; a mechanism to save this in HDF5 would add to the complexity (a brief discussion of how it might be done is discussed later). The resulting Python object consists only of dictionaries, lists, primitive values (strings, ints, floats) and numpy arrays (or equivalent). The edited version is shown in `Appendix D`_.


A mechanism for turning such a Python structure into a HDF5 representation was needed. The solution is a bit convoluted and clumsy. We are not aware of anything better. The following itemizes the obstacles and chosen solutions.

.. `Mapping YAML Structure into HDF5`_:

Mapping YAML structure into HDF5
================================

.. `Representing dictionaries in HDF5`_:

Representing dictionaries in HDF5
---------------------------------

Since dictionaries may contain nested values that are themselves dictionaries or lists, we must rely on HDF5 groups to represent them, as that is the only structure in HDF5 that can be nested. For example, HDF5 metadata values do
not permit any of the values to be anything but primitive values or arrays (see `Appendix A`_ for a more detailed explanation). Thus a value that is itself another dictionary or list, must result in a new group. Besides groups not being optimized for this sort of use, it leads to an awkward duality of interface. The attribute names for values that are dictionaries or lists are then effectively keys of the group itself. But to save primitive values one must save them as keys of the .attrs attribute of the group. Keys are then split between two different items: the group itself, or the .attrs attribute of the group. No one can claim that this is very elegant.


Representing heterogeneous lists in HDF5
----------------------------------------

The situation for lists is worse since there is no equivalent mechanism in HDF5 for lists (i.e., containing items that may have different types, where some may themselves be lists or dictionaries). The only apparent solution is to turn the indices of the list into attribute names such as L0, L1, ... L307. And similar to dictionaries, these attribute names may be split between group attributes and .attr attributes. None of the normal display tools for HDF5 will render the contents with the list items in the proper order or even in the same place. Worse, a mechanism is needed to indicate that the contents of the group are to be interpreted as a list. In this example we did this by creating an attribute with the name "asdf_list" with the value of “true”.


There are two uses of the YAML “anchor” and “alias” features in this ASDF example. This feature allows associating an “anchor”, essentially a special identifier, to a node in the tree that other parts of the tree can refer to through the use of “aliases” that references that special identifier. In this way another node may point to the contents of a different node for reference, avoiding having to copy the contents and risk the two nodes becoming different in the future. In this example one of the referenced nodes was fairly large (but small compared to the total tree), but only had one alias that referenced it. There were several references to a much smaller node. All these aliases were replaced by copies in the ASDF to avoid the work of supporting such aliases in the conversion.


A small segment of the contents of the HDF5 file is displayed in Appendix E to contrast the difficulty of directly interpreting the contents as compared to ASDF. In the ASCII rendition one can see where some keys are located, but not much beyond that.


Disk space comparison
---------------------

The tag excised in-line array version of the ASDF file was 111942 bytes in size, The converted HDF5 version of the contents was 1755312 bytes in size, approximately 15 times larger.


Discussion of the difficulties
==============================

HDF5 was not designed with the idea of storing complex arrangements of metadata. An object-oriented representation of data sets is a natural progression from simple FITS headers and extensions. Some information is naturally hierarchical, and that hierarchy is not limited just to arrays. It is by no accident that many of the network and web mechanisms for passing such data between systems are based on JSON. Note that the ON stands for Object Notation. JSON is essentially a subset of YAML, which provides more representation options as well as tags and references as intrinsic to the format, both of which are extremely useful for our purposes.


Although HDF5 can represent hierarchical sets of data, it doesn't handle dealing with metadata structures as well, leading to the necessity of libraries handling specific transformations of the HDF5 representations of the needed information to construct the corresponding software object.


This exercise did not handle tags, but tags are essential to the library to indicate how the metadata should be interpreted. HDF5 has no such equivalent. As it is currently used, people construct their own conventions for how the data must be interpreted. It may be by filename conventions, or the presence of special attributes to signal the software what should be done. But these solutions are in no way general. The assumptions are embedded into applications or local libraries, without any general system that applies to other specialties or domains. Here tags could be inserted into a standard attribute, something similar to asdf_list. But it would not be part of any HDF5 standard.


Lack of a general system of validation
--------------------------------------

In an operational environment, it is valuable to be able to specify requirements for the data files that the software is to process. It is useful for diagnosing whether a problem the software is having is with the software or the file itself. It is particularly useful when non-operations users encounter problems running the same code. Very frequently it is because they have made unacceptable changes to the file that breaks the software using the file. Finally, it facilitates communication between different organizations when data are being generated by one and processed by another, and keeps everyone disciplined as to the requirements for the contents.


ASDF uses schema files (written themselves in YAML) to validate the contents of a tagged node in an ASDF file. In this way it is not necessary to have one large schema file to handle all details, but instead have schema files handle specific tags. The ASDF library handles checking that files pass validation on both reading and writing (though validation can be turned off to speed processing). Schema files cannot possibly check all kinds of constraints on a specific file, but it can handle most common issues such as are all the required elements present, and those that are present have the right type, and possible allowed ranges or enumerated values. It would not be reasonable to expect that the schema will be able to ensure a value is a prime number or more elaborate kinds of requirements.


Lack of a standard system of extension plugins
----------------------------------------------

Tags in ASDF are also used as a mechanism by software libraries to convert the content to special software objects (e.g. a tag to convert array data to a numpy array if used with Python). But beyond arrays, many other examples exist. Tags can be used to indicate units to attach to numeric values, or that the contents are to be constructed as analytic functions (and compound functions as well).


Obviously no general format can possibly anticipate all possible objects in the software library for a given scientific or engineering field. But it should make extensions to the standard number of objects relatively easy and self evident. Tags serve that role. When a plugin for a tag is installed, it registers itself with the Python ASDF library, such that when the ASDF library encounters that tag it knows what code to call to convert it to an object. Likewise, the objects that are supported in serialization to ASDF are registered, such that when ASDF is serializing a tree, it knows what code to call to convert it to the YAML/binary representation.


This allows extensions at the domain, organizational, experiment, or individual level without requiring standards agreements to be worked out in a larger context. Since it makes implementations of new localized standards much easier, it also helps demonstrate the usefulness of such a prototype standard without the need to add it to a higher level of standards. In this way decisions to add standards can be based on real usage and not require potentially wasteful discussions about the best way to define a new standard without such experience.


There is nothing like this kind of facility for HDF5 currently.


Comments regarding other enumerated reasons listed at the beginning
-------------------------------------------------------------------

1. Entirely binary.
2. Not self documenting
3. Effectively only one implementation (due to complexity)
4. Questionable as a archival format
5. Does not lend itself to pure text-based data files as an option


The first four items are related in more complex ways. Preferably an archival format would not involve a very complex format definition (for HDF5, well over 200 pages). Ideally an archival format would be easy to interpret for basic array data and tables without using a specification document. This is nearly impossible for HDF5. That there is only one effective implementation means that if there is an error in the implementation, it may disagree with the specification. Multiple implementations generally reduce the possibility that files deviate from the specification. Finally, while perhaps not as true now, for some time HDF standards were seen more as a software interface than a file format, and had been rejected previously in astronomy for that reason.


https://astrocompute.wordpress.com/2014/10/07/astronomical-data-formats-past-present-and-future-adass-bof-session/


http://tdc-www.harvard.edu/mink/adass2014/adass2014b1.pdf


With regard to archival use, ASDF files are essentially self documenting in that with the file alone, one would fairly easily be able to decipher the contents, even with regard to the binary data (so long as compression isn't used). Having the metadata in a readable format is essential to making the format self-documenting.


While there is currently only one complete implementation (in Python) for ASDF, its relative simplicity makes it much easier to support other implementations.


Text-based alternatives
-----------------------

In a number of cases scientists or engineers prefer to deal with data in a simpler text-based format. They have various reasons for doing so usually involving one or more of the following:


A. They like to write their own software to access such files directly, particularly with a language that does not have a library to read the special format (but where they can obtain such files from another organization that can easily generate a pure text alternative).


B. They like to easily view or edit such files with simple text editors or other text-based tools.


C. It makes it easy to collaborate with others that have a preference based on A) or B).


ASDF makes this easily possible by allowing one to generate pure text files containing arrays or tables by putting such data "in-line" for those that want their data in this form. Generally speaking this is when the amount of data is not large, such as a spectrum of a few thousand lines, or a table with a few hundred rows.


Such a file is a little more complex than a CSV or similar file in that it must have the appropriate ASDF header info and stay consistent with the YAML syntax, but so long as one accesses or modifies only the data section, this is not much of an imposition. Reading and writing such sections from almost any language is very simple.


Areas where HDF5 has an advantage
=================================

1. It has been around a long time and is widely used.

2. It has performance features not currently in ASDF. An example is chunking, though this will be added in the near future. There are few obstacles to adding such performance enhancements to ASDF other than the effort to support their implementation. Even so, these enhancements should be optional extensions to keep the core standard simple.

.. _`Appendix A`:

Appendix A: Summary of relevant HDF5 Hierarchical Structures
============================================================



The top level item in an HDF5 file is an object called a Group. Groups can contain other Groups, which are referenced by a name. Groups can also contain Datasets (essentially array data), also referenced by name (Groups and Datasets use the same namespace thus you cannot give the same name to a contained Group and contained Dataset). Note that nested Groups can be referred to in a way analogous to file paths, where the top group is referred to by ‘/’ and the subgroup by ‘/subgroup_name’ and likewise for further nestings. Both Groups and Datasets have an associated set of possible attributes. Attributes are referred to by name, but do not share the same namespace as used by the contained Datasets or Groups. Thus it is possible to use the same name for a Dataset or Group as for an attribute. Accessing Datasets or Groups uses a different interface than for accessing Attributes, as illustrated below:

::

	>>> h = h5py.File('test.h5')
	>>> h.create_group('name1')
	>>> h.attrs['name2'] = 42
	>>> h['name1']
	<HDF5 group "/name1" 0 members>
	>>> h.attrs['name2']
	42


But note that it is not possible to nest dictionaries in the attributes:

::

	>>> h.attrs['dict'] = {'key1': 'Hi there', 'key2': 'Goodbye'}
	TypeError: Object dtype dtype('O') has no native HDF5 equivalent


There are many, many other aspects to HDF5 not touched on here. The focus here is to highlight the nesting possibilities for metadata and data.

.. _`Appendix B`:

Appendix B: GWCS Object Representation
======================================

::


	<WCS(output_frame=world, input_frame=detector, forward_transform=Model: CompoundModel
	Inputs: ('x0', 'x1')
	Outputs: ('lon', 'lat', 'x0')
	Model set size: 1
	Expression: [0] & [1] | [2] & [3] | [4] | [5] | [6] & [7] | [8] | [9] 
	& [10] | [11] | [12] & [13] | [14] | [15] & [16] | [17] | [18] | [19] 
	| [20] * [21] & [22] * [23] & [24] | [25] & [26] & [27] | [28] | [29] 
	& ([30] & [31] | [32] & [33] | [34] | [35] & [36] | [37] & [38] | [39] 
	| [40] & [41] | [42] | [43] & [44] | [45] | [46] | [47]) & [48] | [49] 
	| [50] & ([51] | [52] | [53] * [54]) | [55] | [56] & ([57] | [58] 
	| [59]) | [60] | [61] & [62] + [63] & [64] + [65] | ([66] & [67] | [68] 
	& [69] | [70] | [71] & [72]) & [73] | [74] | ([75] | ([76] | [77]) 
	+ ([78] | [79]) * ([80] | [81]) & ([82] | [83]) + ([84] | [85]) * ([86] 
	| [87]) | [88] | [89] & [90] | [91] | [92] & [93]) & [94] | [95] | ([96] 
	| [97] & [98] | [99] | [100] & [101] | [102] | [103] & [104] | [105] 
	& [106]) & [107] | ([108] & [109] | [110] & [111]) & [112] | ([113] 
	& [114] | [115] | [116] | [117]) & [118]

	Components:
	        [0]: <Shift(offset=1083.)>


	        [1]: <Shift(offset=4.)>


	        [2]: <Shift(offset=0.)>


	        [3]: <Shift(offset=1053.)>


	        [4]: <Identity(2)>


	        [5]: <AffineTransformation2D(matrix=[[0.000018, 0.          ], 
	        [0.          , 0.000018]], translation=[0., 0.], 
	        name='fpa_affine_d2s')>


	        [6]: <Shift(offset=-0.03817084, name='fpa_x_d2s')>


	        [7]: <Shift(offset=-0.018423, name='fpa_y_d2s')>


	        [8]: <Mapping((0, 1, 0, 1), name='camera_inmap')>


	        [9]: <Polynomial2D(5, c0_0=0.00052463, c1_0=1.00242686, 
	        c2_0=0.00338027, c3_0=4.73375404, c4_0=0.44460679, 
	        c5_0=-214.41517634, c0_1=0.00862406, c0_2=-0.00963133, 
	        c0_3=-0.05928625, c0_4=-22.17857173, c0_5=116.16456423, 
	        c1_1=0.84289016, c1_2=4.48027128, c1_3=-2.23526738, 
	        c1_4=28.03399971, c2_1=0.17080919, c2_2=-1.7189239, 
	        c2_3=-76.79343486, c3_1=-5.69989296, c3_2=-30.22299289, 
	        c4_1=0.44649936, name='camera_x_forward')>


	        [10]: <Polynomial2D(5, c0_0=0.00033855, c1_0=-0.00716534, 
	        c2_0=0.27499345, c3_0=0.03624052, c4_0=-4.77784964, 
	        c5_0=-40.33214385, c0_1=0.99617536, c0_2=1.07561683, 
	        c0_3=2.15672501, c0_4=-39.20989494, c0_5=869.77478932, 
	        c1_1=-0.00469626, c1_2=0.04273391, c1_3=-0.63239649, 
	        c1_4=-162.5901752, c2_1=3.72997705, c2_2=-19.21481331, 
	        c2_3=-135.76420766, c3_1=-0.8204123, c3_2=5.36454197, 
	        c4_1=-200.69262941, name='camera_y_forward')>


	        [11]: <Identity(2, name='camera_outmap')>


	        [12]: <Shift(offset=0.00000239, name='camera_xincen_d2s')>


	        [13]: <Shift(offset=-0.00021835, name='camera_yincen_d2s')>


	        [14]: <AffineTransformation2D(matrix=[[ 3.512594  ,  0.00018787],
	        [-0.00017734,  3.72123582]], translation=[0., 0.], 
	        name='camera_affine_d2s')>


	        [15]: <Shift(offset=0.0001439, name='camera_xoutcen_d2s')>


	        [16]: <Shift(offset=0.29360602, name='camera_youtcen_d2s')>


	        [17]: <Unitless2DirCos(name='unitless2directional_cosines')>


	        [18]: <Rotation3DToGWA(angles=[ 0.03333073, -0.27547252, 
	        -0.14198883, 24.29          ], name='rotation')>


	        [19]: <Mapping((0, 1, 0, 1))>


	        [20]: <Const1D(amplitude=0.)>


	        [21]: <Identity(1)>


	        [22]: <Const1D(amplitude=-1.)>


	        [23]: <Identity(1)>


	        [24]: <Identity(2)>


	        [25]: <Identity(1)>


	        [26]: <Tabular1D(points=(<array (unloaded) shape: [1000] dtype: 
	        float64>,), lookup_table=[-0.55           -0.5488989  -0.5477978  
	         -0.5466967  -0.5455956  -0.54449449
	         -0.54339339 -0.54229229 -0.54119119 -0.54009009 -0.53898899 -0.53788789
	         -0.53678679 -0.53568569 -0.53458458 -0.53348348 -0.53238238 -0.53128128
	         -0.53018018 -0.52907908 -0.52797798 -0.52687688 -0.52577578 -0.52467467
	         -0.52357357 -0.52247247 -0.52137137 -0.52027027 -0.51916917 -0.51806807
	         -0.51696697 -0.51586587 -0.51476476 -0.51366366 -0.51256256 -0.51146146
	         -0.51036036 -0.50925926 -0.50815816 -0.50705706 -0.50595596 -0.50485485
	         -0.50375375 -0.50265265 -0.50155155 -0.50045045 -0.49934935 -0.49824825
	         -0.49714715 -0.49604605 -0.49494494 -0.49384384 -0.49274274 -0.49164164
	         -0.49054054 -0.48943944 -0.48833834 -0.48723724 -0.48613614 -0.48503504
	         -0.48393393 -0.48283283 -0.48173173 -0.48063063 -0.47952953 -0.47842843
	         -0.47732733 -0.47622623 -0.47512513 -0.47402402 -0.47292292 -0.47182182
	         -0.47072072 -0.46961962 -0.46851852 -0.46741742 -0.46631632 -0.46521522
	         -0.46411411 -0.46301301 -0.46191191 -0.46081081 -0.45970971 -0.45860861
	         -0.45750751 -0.45640641 -0.45530531 -0.4542042  -0.4531031  -0.452002
	         -0.4509009  -0.4497998  -0.4486987  -0.4475976  -0.4464965  -0.4453954
	         -0.44429429 -0.44319319 -0.44209209 -0.44099099 -0.43988989 -0.43878879
	         -0.43768769 -0.43658659 -0.43548549 -0.43438438 -0.43328328 -0.43218218
	         -0.43108108 -0.42997998 -0.42887888 -0.42777778 -0.42667668 -0.42557558
	         -0.42447447 -0.42337337 -0.42227227 -0.42117117 -0.42007007 -0.41896897
	         -0.41786787 -0.41676677 -0.41566567 -0.41456456 -0.41346346 -0.41236236
	         -0.41126126 -0.41016016 -0.40905906 -0.40795796 -0.40685686 -0.40575576
	         -0.40465465 -0.40355355 -0.40245245 -0.40135135 -0.40025025 -0.39914915
	         -0.39804805 -0.39694695 -0.39584585 -0.39474474 -0.39364364 -0.39254254
	         -0.39144144 -0.39034034 -0.38923924 -0.38813814 -0.38703704 -0.38593594
	         -0.38483483 -0.38373373 -0.38263263 -0.38153153 -0.38043043 -0.37932933
	         -0.37822823 -0.37712713 -0.37602603 -0.37492492 -0.37382382 -0.37272272
	         -0.37162162 -0.37052052 -0.36941942 -0.36831832 -0.36721722 -0.36611612
	         -0.36501502 -0.36391391 -0.36281281 -0.36171171 -0.36061061 -0.35950951
	         -0.35840841 -0.35730731 -0.35620621 -0.35510511 -0.354004   -0.3529029
	         -0.3518018  -0.3507007  -0.3495996  -0.3484985  -0.3473974  -0.3462963
	         -0.3451952  -0.34409409 -0.34299299 -0.34189189 -0.34079079 -0.33968969
	         -0.33858859 -0.33748749 -0.33638639 -0.33528529 -0.33418418 -0.33308308
	         -0.33198198 -0.33088088 -0.32977978 -0.32867868 -0.32757758 -0.32647648
	         -0.32537538 -0.32427427 -0.32317317 -0.32207207 -0.32097097 -0.31986987
	         -0.31876877 -0.31766767 -0.31656657 -0.31546547 -0.31436436 -0.31326326
	         -0.31216216 -0.31106106 -0.30995996 -0.30885886 -0.30775776 -0.30665666
	         -0.30555556 -0.30445445 -0.30335335 -0.30225225 -0.30115115 -0.30005005
	         -0.29894895 -0.29784785 -0.29674675 -0.29564565 -0.29454454 -0.29344344
	         -0.29234234 -0.29124124 -0.29014014 -0.28903904 -0.28793794 -0.28683684
	         -0.28573574 -0.28463463 -0.28353353 -0.28243243 -0.28133133 -0.28023023
	         -0.27912913 -0.27802803 -0.27692693 -0.27582583 -0.27472472 -0.27362362
	         -0.27252252 -0.27142142 -0.27032032 -0.26921922 -0.26811812 -0.26701702
	         -0.26591592 -0.26481481 -0.26371371 -0.26261261 -0.26151151 -0.26041041
	         -0.25930931 -0.25820821 -0.25710711 -0.25600601 -0.2549049  -0.2538038
	         -0.2527027  -0.2516016  -0.2505005  -0.2493994  -0.2482983  -0.2471972
	         -0.2460961  -0.24499499 -0.24389389 -0.24279279 -0.24169169 -0.24059059
	         -0.23948949 -0.23838839 -0.23728729 -0.23618619 -0.23508509 -0.23398398
	         -0.23288288 -0.23178178 -0.23068068 -0.22957958 -0.22847848 -0.22737738
	         -0.22627628 -0.22517518 -0.22407407 -0.22297297 -0.22187187 -0.22077077
	         -0.21966967 -0.21856857 -0.21746747 -0.21636637 -0.21526527 -0.21416416
	         -0.21306306 -0.21196196 -0.21086086 -0.20975976 -0.20865866 -0.20755756
	         -0.20645646 -0.20535536 -0.20425425 -0.20315315 -0.20205205 -0.20095095
	         -0.19984985 -0.19874875 -0.19764765 -0.19654655 -0.19544545 -0.19434434
	         -0.19324324 -0.19214214 -0.19104104 -0.18993994 -0.18883884 -0.18773774
	         -0.18663664 -0.18553554 -0.18443443 -0.18333333 -0.18223223 -0.18113113
	         -0.18003003 -0.17892893 -0.17782783 -0.17672673 -0.17562563 -0.17452452
	         -0.17342342 -0.17232232 -0.17122122 -0.17012012 -0.16901902 -0.16791792
	         -0.16681682 -0.16571572 -0.16461461 -0.16351351 -0.16241241 -0.16131131
	         -0.16021021 -0.15910911 -0.15800801 -0.15690691 -0.15580581 -0.1547047
	         -0.1536036  -0.1525025  -0.1514014  -0.1503003  -0.1491992  -0.1480981
	         -0.146997   -0.1458959  -0.14479479 -0.14369369 -0.14259259 -0.14149149
	         -0.14039039 -0.13928929 -0.13818819 -0.13708709 -0.13598599 -0.13488488
	         -0.13378378 -0.13268268 -0.13158158 -0.13048048 -0.12937938 -0.12827828
	         -0.12717718 -0.12607608 -0.12497497 -0.12387387 -0.12277277 -0.12167167
	         -0.12057057 -0.11946947 -0.11836837 -0.11726727 -0.11616617 -0.11506507
	         -0.11396396 -0.11286286 -0.11176176 -0.11066066 -0.10955956 -0.10845846
	         -0.10735736 -0.10625626 -0.10515516 -0.10405405 -0.10295295 -0.10185185
	         -0.10075075 -0.09964965 -0.09854855 -0.09744745 -0.09634635 -0.09524525
	         -0.09414414 -0.09304304 -0.09194194 -0.09084084 -0.08973974 -0.08863864
	         -0.08753754 -0.08643644 -0.08533534 -0.08423423 -0.08313313 -0.08203203
	         -0.08093093 -0.07982983 -0.07872873 -0.07762763 -0.07652653 -0.07542543
	         -0.07432432 -0.07322322 -0.07212212 -0.07102102 -0.06991992 -0.06881882
	         -0.06771772 -0.06661662 -0.06551552 -0.06441441 -0.06331331 -0.06221221
	         -0.06111111 -0.06001001 -0.05890891 -0.05780781 -0.05670671 -0.05560561
	         -0.0545045  -0.0534034  -0.0523023  -0.0512012  -0.0501001  -0.048999
	         -0.0478979  -0.0467968  -0.0456957  -0.04459459 -0.04349349 -0.04239239
	         -0.04129129 -0.04019019 -0.03908909 -0.03798799 -0.03688689 -0.03578579
	         -0.03468468 -0.03358358 -0.03248248 -0.03138138 -0.03028028 -0.02917918
	         -0.02807808 -0.02697698 -0.02587588 -0.02477477 -0.02367367 -0.02257257
	         -0.02147147 -0.02037037 -0.01926927 -0.01816817 -0.01706707 -0.01596597
	         -0.01486486 -0.01376376 -0.01266266 -0.01156156 -0.01046046 -0.00935936
	         -0.00825826 -0.00715716 -0.00605606 -0.00495495 -0.00385385 -0.00275275
	         -0.00165165 -0.00055055  0.00055055  0.00165165  0.00275275  0.00385385
	          0.00495495  0.00605606  0.00715716  0.00825826  0.00935936  0.01046046
	          0.01156156  0.01266266  0.01376376  0.01486486  0.01596597  0.01706707
	          0.01816817  0.01926927  0.02037037  0.02147147  0.02257257  0.02367367
	          0.02477477  0.02587588  0.02697698  0.02807808  0.02917918  0.03028028
	          0.03138138  0.03248248  0.03358358  0.03468468  0.03578579  0.03688689
	          0.03798799  0.03908909  0.04019019  0.04129129  0.04239239  0.04349349
	          0.04459459  0.0456957   0.0467968   0.0478979   0.048999        0.0501001
	          0.0512012   0.0523023   0.0534034   0.0545045   0.05560561  0.05670671
	          0.05780781  0.05890891  0.06001001  0.06111111  0.06221221  0.06331331
	          0.06441441  0.06551552  0.06661662  0.06771772  0.06881882  0.06991992
	          0.07102102  0.07212212  0.07322322  0.07432432  0.07542543  0.07652653
	          0.07762763  0.07872873  0.07982983  0.08093093  0.08203203  0.08313313
	          0.08423423  0.08533534  0.08643644  0.08753754  0.08863864  0.08973974
	          0.09084084  0.09194194  0.09304304  0.09414414  0.09524525  0.09634635
	          0.09744745  0.09854855  0.09964965  0.10075075  0.10185185  0.10295295
	          0.10405405  0.10515516  0.10625626  0.10735736  0.10845846  0.10955956
	          0.11066066  0.11176176  0.11286286  0.11396396  0.11506507  0.11616617
	          0.11726727  0.11836837  0.11946947  0.12057057  0.12167167  0.12277277
	          0.12387387  0.12497497  0.12607608  0.12717718  0.12827828  0.12937938
	          0.13048048  0.13158158  0.13268268  0.13378378  0.13488488  0.13598599
	          0.13708709  0.13818819  0.13928929  0.14039039  0.14149149  0.14259259
	          0.14369369  0.14479479  0.1458959   0.146997        0.1480981   0.1491992
	          0.1503003   0.1514014   0.1525025   0.1536036   0.1547047   0.15580581
	          0.15690691  0.15800801  0.15910911  0.16021021  0.16131131  0.16241241
	          0.16351351  0.16461461  0.16571572  0.16681682  0.16791792  0.16901902
	          0.17012012  0.17122122  0.17232232  0.17342342  0.17452452  0.17562563
	          0.17672673  0.17782783  0.17892893  0.18003003  0.18113113  0.18223223
	          0.18333333  0.18443443  0.18553554  0.18663664  0.18773774  0.18883884
	          0.18993994  0.19104104  0.19214214  0.19324324  0.19434434  0.19544545
	          0.19654655  0.19764765  0.19874875  0.19984985  0.20095095  0.20205205
	          0.20315315  0.20425425  0.20535536  0.20645646  0.20755756  0.20865866
	          0.20975976  0.21086086  0.21196196  0.21306306  0.21416416  0.21526527
	          0.21636637  0.21746747  0.21856857  0.21966967  0.22077077  0.22187187
	          0.22297297  0.22407407  0.22517518  0.22627628  0.22737738  0.22847848
	          0.22957958  0.23068068  0.23178178  0.23288288  0.23398398  0.23508509
	          0.23618619  0.23728729  0.23838839  0.23948949  0.24059059  0.24169169
	          0.24279279  0.24389389  0.24499499  0.2460961   0.2471972   0.2482983
	          0.2493994   0.2505005   0.2516016   0.2527027   0.2538038   0.2549049
	          0.25600601  0.25710711  0.25820821  0.25930931  0.26041041  0.26151151
	          0.26261261  0.26371371  0.26481481  0.26591592  0.26701702  0.26811812
	          0.26921922  0.27032032  0.27142142  0.27252252  0.27362362  0.27472472
	          0.27582583  0.27692693  0.27802803  0.27912913  0.28023023  0.28133133
	          0.28243243  0.28353353  0.28463463  0.28573574  0.28683684  0.28793794
	          0.28903904  0.29014014  0.29124124  0.29234234  0.29344344  0.29454454
	          0.29564565  0.29674675  0.29784785  0.29894895  0.30005005  0.30115115
	          0.30225225  0.30335335  0.30445445  0.30555556  0.30665666  0.30775776
	          0.30885886  0.30995996  0.31106106  0.31216216  0.31326326  0.31436436
	          0.31546547  0.31656657  0.31766767  0.31876877  0.31986987  0.32097097
	          0.32207207  0.32317317  0.32427427  0.32537538  0.32647648  0.32757758
	          0.32867868  0.32977978  0.33088088  0.33198198  0.33308308  0.33418418
	          0.33528529  0.33638639  0.33748749  0.33858859  0.33968969  0.34079079
	          0.34189189  0.34299299  0.34409409  0.3451952   0.3462963   0.3473974
	          0.3484985   0.3495996   0.3507007   0.3518018   0.3529029   0.354004
	          0.35510511  0.35620621  0.35730731  0.35840841  0.35950951  0.36061061
	          0.36171171  0.36281281  0.36391391  0.36501502  0.36611612  0.36721722
	          0.36831832  0.36941942  0.37052052  0.37162162  0.37272272  0.37382382
	          0.37492492  0.37602603  0.37712713  0.37822823  0.37932933  0.38043043
	          0.38153153  0.38263263  0.38373373  0.38483483  0.38593594  0.38703704
	          0.38813814  0.38923924  0.39034034  0.39144144  0.39254254  0.39364364
	          0.39474474  0.39584585  0.39694695  0.39804805  0.39914915  0.40025025
	          0.40135135  0.40245245  0.40355355  0.40465465  0.40575576  0.40685686
	          0.40795796  0.40905906  0.41016016  0.41126126  0.41236236  0.41346346
	          0.41456456  0.41566567  0.41676677  0.41786787  0.41896897  0.42007007
	          0.42117117  0.42227227  0.42337337  0.42447447  0.42557558  0.42667668
	          0.42777778  0.42887888  0.42997998  0.43108108  0.43218218  0.43328328
	          0.43438438  0.43548549  0.43658659  0.43768769  0.43878879  0.43988989
	          0.44099099  0.44209209  0.44319319  0.44429429  0.4453954   0.4464965
	          0.4475976   0.4486987   0.4497998   0.4509009   0.452002        0.4531031
	          0.4542042   0.45530531  0.45640641  0.45750751  0.45860861  0.45970971
	          0.46081081  0.46191191  0.46301301  0.46411411  0.46521522  0.46631632
	          0.46741742  0.46851852  0.46961962  0.47072072  0.47182182  0.47292292
	          0.47402402  0.47512513  0.47622623  0.47732733  0.47842843  0.47952953
	          0.48063063  0.48173173  0.48283283  0.48393393  0.48503504  0.48613614
	          0.48723724  0.48833834  0.48943944  0.49054054  0.49164164  0.49274274
	          0.49384384  0.49494494  0.49604605  0.49714715  0.49824825  0.49934935
	          0.50045045  0.50155155  0.50265265  0.50375375  0.50485485  0.50595596
	          0.50705706  0.50815816  0.50925926  0.51036036  0.51146146  0.51256256
	          0.51366366  0.51476476  0.51586587  0.51696697  0.51806807  0.51916917
	          0.52027027  0.52137137  0.52247247  0.52357357  0.52467467  0.52577578
	          0.52687688  0.52797798  0.52907908  0.53018018  0.53128128  0.53238238
	          0.53348348  0.53458458  0.53568569  0.53678679  0.53788789  0.53898899
	          0.54009009  0.54119119  0.54229229  0.54339339  0.54449449  0.5455956
	          0.5466967   0.5477978   0.5488989   0.55          ])>


	        [27]: <Identity(2)>


	        [28]: <Mapping((0, 1, 0, 1, 2, 3))>


	        [29]: <Identity(2)>


	        [30]: <Scale(factor=0.00008135)>


	        [31]: <Scale(factor=0.00127117)>


	        [32]: <Shift(offset=0.02697243)>


	        [33]: <Shift(offset=-0.0027167)>


	        [34]: <Rotation2D(angle=0., name='msa_slit_rot')>


	        [35]: <Shift(offset=0., name='msa_slit_x')>


	        [36]: <Shift(offset=0., name='msa_slit_y')>


	        [37]: <Shift(offset=-0.00000553, name='collimator_xoutcen_d2s')>


	        [38]: <Shift(offset=0.00034604, name='collimator_youtcen_d2s')>


	        [39]: <AffineTransformation2D(matrix=[[ 1.57380009, -0.00034509], 
	        [ 0.00036132,  1.6478569 ]], translation=[-0., -0.])>


	        [40]: <Shift(offset=-0.0001439, name='collimator_xincen_d2s')>


	        [41]: <Shift(offset=-0.29360593, name='collimator_yincen_d2s')>


	        [42]: <Mapping((0, 1, 0, 1))>


	        [43]: <Polynomial2D(5, c0_0=0.00315707, c1_0=0.97396667, 
	        c2_0=-0.11821996, c3_0=-0.23912451, c4_0=-0.72133193, 
	        c5_0=-2.32233205, c0_1=0.04204815, c0_2=0.14656153, 
	        c0_3=0.22123416, c0_4=-0.06386192, c0_5=-0.33178124, 
	        c1_1=-0.0712862, c1_2=-0.26989581, c1_3=-1.4782121, 
	        c1_4=-1.39521612, c2_1=-1.31400145, c2_2=-4.6554671, 
	        c2_3=-5.31391588, c3_1=-3.50159181, c3_2=-5.63024065, 
	        c4_1=-2.52317346, name='collimator_x_backward')>


	        [44]: <Polynomial2D(5, c0_0=-0.00278444, c1_0=-0.02512927, 
	        c2_0=0.06362503, c3_0=0.07104457, c4_0=-2.00363216, 
	        c5_0=-1.39018617, c0_1=1.15424678, c0_2=1.57319738, 
	        c0_3=5.65896062, c0_4=9.03612184, c0_5=5.8946139, 
	        c1_1=-0.24879557, c1_2=-1.30121422, c1_3=-2.98313737, 
	        c1_4=-2.54562283, c2_1=0.75193672, c2_2=1.24472157, 
	        c2_3=1.15635545, c3_1=0.19372342, c3_2=-0.04967141, 
	        c4_1=-6.6791682, name='collimator_y_backward')>


	        [45]: <Identity(2)>


	        [46]: <Unitless2DirCos(name='unitless2directional_cosines')>


	        [47]: <Rotation3DToGWA(angles=[ 0.03333073, -0.27547252, 
	        -0.14198883, 24.29          ], name='rotation')>


	        [48]: <Identity(2)>


	        [49]: <Mapping((0, 1, 2, 3, 5))>


	        [50]: <Identity(2)>


	        [51]: <RefractionIndexFromPrism(prism_angle=-16.5, name='n_prism')>


	        [52]: <Tabular1D(points=(<array (unloaded) shape: [1101] dtype: 
	        float64>,), lookup_table=[6.000e-06 5.995e-06 5.990e-06 ... 
	        5.100e-07 5.050e-07 5.000e-07])>


	        [53]: <Identity(1)>


	        [54]: <Const1D(amplitude=1.00000466, name='velocity_correction')>


	        [55]: <Mapping((0, 1, 2, 1))>


	        [56]: <Identity(3)>


	        [57]: Logical(condition=GT, compareto=0.55, value=nan)


	        [58]: Logical(condition=LT, compareto=-0.55, value=nan)


	        [59]: <Scale(factor=0.)>


	        [60]: <Mapping((0, 1, 3, 2, 3))>


	        [61]: <Identity(1)>


	        [62]: <Mapping((0,))>


	        [63]: <Mapping((1,))>


	        [64]: <Mapping((0,))>


	        [65]: <Mapping((1,))>


	        [66]: <Scale(factor=0.00008135)>


	        [67]: <Scale(factor=0.00127117)>


	        [68]: <Shift(offset=0.02697243)>


	        [69]: <Shift(offset=-0.0027167)>


	        [70]: <Rotation2D(angle=0., name='msa_slit_rot')>


	        [71]: <Shift(offset=0., name='msa_slit_x')>


	        [72]: <Shift(offset=0., name='msa_slit_y')>


	        [73]: <Identity(1)>


	        [74]: <Mapping((0, 1, 2, 2), name='msa2fore_mapping')>


	        [75]: <Mapping((0, 1, 2, 0, 1, 2), name='fore_inmap')>


	        [76]: <Mapping((0, 1))>


	        [77]: <Polynomial2D(5, c0_0=0.00000005, c1_0=0.99986635, 
	        c2_0=-0.00080915, c3_0=0.7486317, c4_0=0.00903959, 
	        c5_0=-5.06051552, c0_1=0.00006236, c0_2=-0.0002823, 
	        c0_3=-0.00063174, c0_4=0.00071402, c0_5=-0.00042942, 
	        c1_1=-0.13263317, c1_2=0.50412029, c1_3=2.18051201, 
	        c1_4=-4.17683202, c2_1=-0.00228739, c2_2=0.00973715, 
	        c2_3=0.00438541, c3_1=2.21832075, c3_2=-9.49582591, 
	        c4_1=-0.01136389, name='fore_x_forw')>


	        [78]: <Mapping((0, 1))>


	        [79]: <Polynomial2D(5, c0_0=-0.00855547, c1_0=31.31477475, 
	        c2_0=-0.01474339, c3_0=133.16117302, c4_0=-26.85661347, 
	        c5_0=-1810.34956166, c0_1=0.00451992, c0_2=-0.02356341, 
	        c0_3=1.00379618, c0_4=-18.04274715, c0_5=-2607.67742719, 
	        c1_1=-4.73852535, c1_2=107.48990398, c1_3=254.07823058, 
	        c1_4=2875.63180287, c2_1=-12.15085563, c2_2=6.63214534, 
	        c2_3=4812.85147845, c3_1=307.60674072, c3_2=-4338.10034721, 
	        c4_1=4693.47133157, name='fore_x_forwdist')>


	        [80]: <Mapping((2,))>


	        [81]: <Identity(1)>


	        [82]: <Mapping((0, 1))>


	        [83]: <Polynomial2D(5, c0_0=-0.00000633, c1_0=0.0000607, 
	        c2_0=-0.079329, c3_0=-0.00079574, c4_0=0.52330044, 
	        c5_0=0.01456288, c0_1=0.99978052, c0_2=-0.18480945, 
	        c0_3=0.26800856, c0_4=2.58830935, c0_5=-3.22936503, 
	        c1_1=-0.00027528, c1_2=-0.00232228, c1_3=0.00558259, 
	        c1_4=0.01727874, c2_1=0.50613876, c2_2=3.17032631, 
	        c2_3=-8.12775002, c3_1=0.00711146, c3_2=-0.00221175, 
	        c4_1=-4.61944624, name='fore_y_forw')>


	        [84]: <Mapping((0, 1))>


	        [85]: <Polynomial2D(5, c0_0=2.25663643, c1_0=0.00976758, 
	        c2_0=-14.00884746, c3_0=2.50986443, c4_0=82.12743763, 
	        c5_0=-1699.96279356, c0_1=32.67409988, c0_2=-19.08440222, 
	        c0_3=94.88508814, c0_4=400.66489646, c0_5=-591.2908096, 
	        c1_1=-0.00812537, c1_2=-8.85503569, c1_3=-15.98017917, 
	        c1_4=2427.42627709, c2_1=109.98425443, c2_2=392.95103766, 
	        c2_3=-1014.48068072, c3_1=3.29401675, c3_2=3073.30289884, 
	        c4_1=1173.28853135, name='fore_y_forwdist')>


	        [86]: <Mapping((2,))>


	        [87]: <Identity(1)>


	        [88]: <Identity(2, name='fore_outmap')>


	        [89]: <Shift(offset=-0.00000553, name='fore_xincen_d2s')>


	        [90]: <Shift(offset=0.00034603, name='fore_yincen_d2s')>


	        [91]: <AffineTransformation2D(matrix=[[ 1.22209154,  1.0903057 ], 
	        [-1.07352028,  1.2412        ]], translation=[0., 0.], 
	        name='fore_affine_d2s')>


	        [92]: <Shift(offset=-0.00000023, name='fore_xoutcen_d2s')>


	        [93]: <Shift(offset=-0.00000026, name='fore_youtcen_d2s')>


	        [94]: <Identity(1)>


	        [95]: <Identity(3, name='fore2ote_mapping')>


	        [96]: <Mapping((0, 1, 0, 1), name='ote_inmap')>


	        [97]: <Polynomial2D(5, c0_0=0., c1_0=1.00001005, c2_0=-0.0130105, 
	        c3_0=-0.01806202, c4_0=0.00041481, c5_0=0.00056426, 
	        c0_1=0.00118187, c0_2=0.0043811, c0_3=-0.00010303, 
	        c0_4=-0.00016211, c0_5=0.00223921, c1_1=-0.01989483, 
	        c1_2=-0.01794431, c1_3=0.00077021, c1_4=0.00555774, 
	        c2_1=0.00072633, c2_2=0.00028157, c2_3=-0.0183197, 
	        c3_1=0.00072136, c3_2=-0.01211201, c4_1=-0.00956088, 
	        name='ote_x_forw')>


	        [98]: <Polynomial2D(5, c0_0=0., c1_0=0.00117983, c2_0=0.00500848, 
	        c3_0=-0.00009635, c4_0=-0.00021478, c5_0=0.0000364, 
	        c0_1=1.00001444, c0_2=-0.01483102, c0_3=-0.01798862, 
	        c0_4=0.00041817, c0_5=0.00072182, c1_1=-0.01747569, 
	        c1_2=0.00067385, c1_3=0.00059052, c1_4=0.00151603, 
	        c2_1=-0.01810347, c2_2=0.00014675, c2_3=0.01135524, 
	        c3_1=0.00060661, c3_2=-0.00293591, c4_1=0.00877504, 
	        name='ote_y_forw')>


	        [99]: <Identity(2, name='ote_outmap')>


	        [100]: <Shift(offset=0.00000052, name='ote_xincen_d2s')>


	        [101]: <Shift(offset=0., name='ote_yincen_d2s')>


	        [102]: <AffineTransformation2D(matrix=[[-0.43553721, -0.00157781], 
	        [-0.00157733,  0.43567004]], translation=[0., 0.], 
	        name='ote_affine_d2s')>


	        [103]: <Shift(offset=0.10539, name='ote_xoutcen_d2s')>


	        [104]: <Shift(offset=-0.11913, name='ote_youtcen_d2s')>


	        [105]: <Scale(factor=3600.)>


	        [106]: <Scale(factor=3600.)>


	        [107]: <Scale(factor=1000000.)>


	        [108]: <Scale(factor=0.99999973, name='dva_scale_v2')>


	        [109]: <Scale(factor=0.99999973, name='dva_scale_v3')>


	        [110]: <Shift(offset=0.00009091, name='dva_v2_shift')>


	        [111]: <Shift(offset=-0.00013117, name='dva_v3_shift')>


	        [112]: <Identity(1)>


	        [113]: <Scale(factor=0.00027778)>


	        [114]: <Scale(factor=0.00027778)>


	        [115]: <SphericalToCartesian()>


	        [116]: <RotationSequence3D(angles=[  0.09226002,   0.13311784, 
	        -93.7605896 , -70.77509994, -90.75467526])>


	        [117]: <CartesianToSpherical()>


	        [118]: <Identity(1)>
	Parameters:
	        offset_0 offset_1 ...                   angles_116 [
	        5]                 
	        -------- -------- ... -----------------------------------------
	          1083.0          4.0 ... 0.09226002166666666 .. 
	          -90.75467525972158)>


.. _`Appendix C`:

Appendix C: ASDF GWCS Contents
==============================

(array values in binary blocks)

::

	#ASDF 1.0.0
	#ASDF_STANDARD 1.5.0
	%YAML 1.1
	%TAG ! tag:stsci.edu:asdf/
	--- !core/asdf-1.1.0
	asdf_library: !core/software-1.0.0 {author: The ASDF Developers, homepage: 'http://github.com/asdf-format/asdf',
	  name: asdf, version: 2.11.2.dev15+g6703d8f.d20220729}
	history:
	  extensions:
	  - !core/extension_metadata-1.0.0
	        extension_class: asdf.extension.BuiltinExtension
	        software: !core/software-1.0.0 {name: asdf, version: 2.11.2.dev15+g6703d8f.d20220729}
	  - !core/extension_metadata-1.0.0
	        extension_class: asdf.extension._manifest.ManifestExtension
	        extension_uri: asdf://asdf-format.org/astronomy/gwcs/extensions/gwcs-1.0.0
	        software: !core/software-1.0.0 {name: gwcs, version: 0.18.1}
	  - !core/extension_metadata-1.0.0
	        extension_class: asdf.extension._manifest.ManifestExtension
	        extension_uri: asdf://asdf-format.org/transform/extensions/transform-1.5.0
	        software: !core/software-1.0.0 {name: asdf-astropy, version: 0.2.1}
	  - !core/extension_metadata-1.0.0
	        extension_class: asdf.extension._manifest.ManifestExtension
	        extension_uri: asdf://asdf-format.org/astronomy/coordinates/extensions/coordinates-1.0.0
	        software: !core/software-1.0.0 {name: asdf-astropy, version: 0.2.1}
	  - !core/extension_metadata-1.0.0
	        extension_class: asdf.extension._manifest.ManifestExtension
	        extension_uri: asdf://stsci.edu/jwst_pipeline/extensions/jwst_transforms-1.0.0
	        software: !core/software-1.0.0 {name: jwst, version: 1.2.4.dev211+g5ce9e04b}
	  - !core/extension_metadata-1.0.0
	        extension_class: asdf.extension._manifest.ManifestExtension
	        extension_uri: asdf://asdf-format.org/core/extensions/core-1.5.0
	        software: !core/software-1.0.0 {name: asdf-astropy, version: 0.2.1}
	wcs: !<tag:stsci.edu:gwcs/wcs-1.0.0>
	  name: ''
	  steps:
	  - !<tag:stsci.edu:gwcs/step-1.0.0>
	        frame: !<tag:stsci.edu:gwcs/frame2d-1.0.0>
	          axes_names: [x, y]
	          axes_order: [0, 1]
	          axis_physical_types: ['custom:x', 'custom:y']
	          name: detector
	          unit: [!unit/unit-1.0.0 pixel, !unit/unit-1.0.0 pixel]
	        transform: !transform/compose-1.2.0
	          bounding_box:
	          - [-0.5, 38.5]
	          - [-0.5, 434.5]
	          forward:
	          - !transform/concatenate-1.2.0
	            forward:
	            - !transform/shift-1.2.0
	              inputs: [x]
	              offset: 1083.0
	              outputs: [y]
	            - !transform/shift-1.2.0
	              inputs: [x]
	              offset: 4.0
	              outputs: [y]
	            inputs: [x0, x1]
	            outputs: [y0, y1]
	          - !transform/compose-1.2.0
	            bounding_box:
	            - [3.5, 42.5]
	            - [1083.4770570798173, 1518.1537632494133]
	            forward:
	            - !transform/concatenate-1.2.0
	              forward:
	              - !transform/shift-1.2.0
	                inputs: [x]
	                offset: 0.0
	                outputs: [y]
	              - !transform/shift-1.2.0
	                inputs: [x]
	                offset: 1053.0
	                outputs: [y]
	              inputs: [x0, x1]
	              outputs: [y0, y1]
	            - !transform/identity-1.2.0
	              inputs: [x0, x1]
	              n_dims: 2
	              outputs: [x0, x1]
	            inputs: [x0, x1]
	            name: dms2sca
	            outputs: [x0, x1]
	          inputs: [x0, x1]
	          name: dms2sca
	          outputs: [x0, x1]
	  - !<tag:stsci.edu:gwcs/step-1.0.0>
	        frame: !<tag:stsci.edu:gwcs/frame2d-1.0.0>
	          axes_names: [x, y]
	          axes_order: [0, 1]
	          axis_physical_types: ['custom:x', 'custom:y']
	          name: sca
	          unit: [!unit/unit-1.0.0 pixel, !unit/unit-1.0.0 pixel]
	        transform: !transform/compose-1.2.0
	          forward:
	          - !transform/compose-1.2.0
	            forward:
	            - !transform/compose-1.2.0
	              forward:
	              - !transform/compose-1.2.0
	                forward:
	                - !transform/affine-1.3.0
	                  inputs: [x, y]
	                  matrix: !core/ndarray-1.0.0
	                    source: 0
	                    datatype: float64
	                    byteorder: little
	                    shape: [2, 2]
	                  name: fpa_affine_d2s
	                  outputs: [x, y]
	                  translation: !core/ndarray-1.0.0
	                    source: 1
	                    datatype: float64
	                    byteorder: little
	                    shape: [2]
	                - !transform/concatenate-1.2.0
	                  forward:
	                  - !transform/shift-1.2.0
	                    inputs: [x]
	                    name: fpa_x_d2s
	                    offset: -0.0381708371805
	                    outputs: [y]
	                  - !transform/shift-1.2.0
	                    inputs: [x]
	                    name: fpa_y_d2s
	                    offset: -0.018423
	                    outputs: [y]
	                  inputs: [x0, x1]
	                  outputs: [y0, y1]
	                inputs: [x, y]
	                inverse: !transform/compose-1.2.0
	                  forward:
	                  - !transform/concatenate-1.2.0
	                    forward:
	                    - !transform/shift-1.2.0
	                      inputs: [x]
	                      name: fpa_x_s2d
	                      offset: 0.0381708371805
	                      outputs: [y]
	                    - !transform/shift-1.2.0
	                      inputs: [x]
	                      name: fpa_y_s2d
	                      offset: 0.018423
	                      outputs: [y]
	                    inputs: [x0, x1]
	                    outputs: [y0, y1]
	                  - !transform/affine-1.3.0
	                    inputs: [x, y]
	                    matrix: !core/ndarray-1.0.0
	                      source: 2
	                      datatype: float64
	                      byteorder: little
	                      shape: [2, 2]
	                    name: fpa_affine_s2d
	                    outputs: [x, y]
	                    translation: !core/ndarray-1.0.0
	                      source: 3
	                      datatype: float64
	                      byteorder: little
	                      shape: [2]
	                  inputs: [x0, x1]
	                  outputs: [x, y]
	                outputs: [y0, y1]
	              - !transform/compose-1.2.0
	                forward:
	                - !transform/compose-1.2.0
	                  forward:
	                  - !transform/compose-1.2.0
	                    forward:
	                    - !transform/remap_axes-1.3.0
	                      inputs: [x0, x1]
	                      inverse: !transform/identity-1.2.0
	                        inputs: [x0, x1]
	                        n_dims: 2
	                        outputs: [x0, x1]
	                      mapping: [0, 1, 0, 1]
	                      name: camera_inmap
	                      outputs: [x0, x1, x2, x3]
	                    - !transform/concatenate-1.2.0
	                      forward:
	                      - !transform/polynomial-1.2.0
	                        coefficients: !core/ndarray-1.0.0
	                          source: 4
	                          datatype: float64
	                          byteorder: little
	                          shape: [6, 6]
	                        domain:
	                        - [-1, 1]
	                        - [-1, 1]
	                        inputs: [x, y]
	                        inverse: !transform/polynomial-1.2.0
	                          coefficients: !core/ndarray-1.0.0
	                            source: 5
	                            datatype: float64
	                            byteorder: little
	                            shape: [6, 6]
	                          domain:
	                          - [-1, 1]
	                          - [-1, 1]
	                          inputs: [x, y]
	                          name: camera_x_backward
	                          outputs: [z]
	                          window:
	                          - [-1, 1]
	                          - [-1, 1]
	                        name: camera_x_forward
	                        outputs: [z]
	                        window:
	                        - [-1, 1]
	                        - [-1, 1]
	                      - !transform/polynomial-1.2.0
	                        coefficients: !core/ndarray-1.0.0
	                          source: 6
	                          datatype: float64
	                          byteorder: little
	                          shape: [6, 6]
	                        domain:
	                        - [-1, 1]
	                        - [-1, 1]
	                        inputs: [x, y]
	                        inverse: !transform/polynomial-1.2.0
	                          coefficients: !core/ndarray-1.0.0
	                            source: 7
	                            datatype: float64
	                            byteorder: little
	                            shape: [6, 6]
	                          domain:
	                          - [-1, 1]
	                          - [-1, 1]
	                          inputs: [x, y]
	                          name: camera_y_backward
	                          outputs: [z]
	                          window:
	                          - [-1, 1]
	                          - [-1, 1]
	                        name: camera_y_forward
	                        outputs: [z]
	                        window:
	                        - [-1, 1]
	                        - [-1, 1]
	                      inputs: [x0, y0, x1, y1]
	                      outputs: [z0, z1]
	                    inputs: [x0, x1]
	                    outputs: [z0, z1]
	                  - !transform/identity-1.2.0
	                    inputs: [x0, x1]
	                    inverse: !transform/remap_axes-1.3.0
	                      inputs: [x0, x1]
	                      mapping: [0, 1, 0, 1]
	                      outputs: [x0, x1, x2, x3]
	                    n_dims: 2
	                    name: camera_outmap
	                    outputs: [x0, x1]
	                  inputs: [x0, x1]
	                  outputs: [x0, x1]
	                - !transform/compose-1.2.0
	                  forward:
	                  - !transform/compose-1.2.0
	                    forward:
	                    - !transform/concatenate-1.2.0
	                      forward:
	                      - !transform/shift-1.2.0
	                        inputs: [x]
	                        name: camera_xincen_d2s
	                        offset: 2.38656283331e-06
	                        outputs: [y]
	                      - !transform/shift-1.2.0
	                        inputs: [x]
	                        name: camera_yincen_d2s
	                        offset: -0.000218347262797
	                        outputs: [y]
	                      inputs: [x0, x1]
	                      outputs: [y0, y1]
	                    - !transform/affine-1.3.0
	                      inputs: [x, y]
	                      matrix: !core/ndarray-1.0.0
	                        source: 8
	                        datatype: float64
	                        byteorder: little
	                        shape: [2, 2]
	                      name: camera_affine_d2s
	                      outputs: [x, y]
	                      translation: !core/ndarray-1.0.0
	                        source: 9
	                        datatype: float64
	                        byteorder: little
	                        shape: [2]
	                    inputs: [x0, x1]
	                    outputs: [x, y]
	                  - !transform/concatenate-1.2.0
	                    forward:
	                    - !transform/shift-1.2.0
	                      inputs: [x]
	                      name: camera_xoutcen_d2s
	                      offset: 0.000143898033
	                      outputs: [y]
	                    - !transform/shift-1.2.0
	                      inputs: [x]
	                      name: camera_youtcen_d2s
	                      offset: 0.293606022006
	                      outputs: [y]
	                    inputs: [x0, x1]
	                    outputs: [y0, y1]
	                  inputs: [x0, x1]
	                  outputs: [y0, y1]
	                inputs: [x0, x1]
	                inverse: !transform/compose-1.2.0
	                  forward:
	                  - !transform/compose-1.2.0
	                    forward:
	                    - !transform/concatenate-1.2.0
	                      forward:
	                      - !transform/shift-1.2.0
	                        inputs: [x]
	                        name: camera_xoutcen_d2s
	                        offset: -0.000143898033
	                        outputs: [y]
	                      - !transform/shift-1.2.0
	                        inputs: [x]
	                        name: camera_youtcen_d2s
	                        offset: -0.293606022006
	                        outputs: [y]
	                      inputs: [x0, x1]
	                      outputs: [y0, y1]
	                    - !transform/compose-1.2.0
	                      forward:
	                      - !transform/affine-1.3.0
	                        inputs: [x, y]
	                        matrix: !core/ndarray-1.0.0
	                          source: 10
	                          datatype: float64
	                          byteorder: little
	                          shape: [2, 2]
	                        outputs: [x, y]
	                        translation: !core/ndarray-1.0.0
	                          source: 11
	                          datatype: float64
	                          byteorder: little
	                          shape: [2]
	                      - !transform/concatenate-1.2.0
	                        forward:
	                        - !transform/shift-1.2.0
	                          inputs: [x]
	                          name: camera_xincen_d2s
	                          offset: -2.38656283331e-06
	                          outputs: [y]
	                        - !transform/shift-1.2.0
	                          inputs: [x]
	                          name: camera_yincen_d2s
	                          offset: 0.000218347262797
	                          outputs: [y]
	                        inputs: [x0, x1]
	                        outputs: [y0, y1]
	                      inputs: [x, y]
	                      outputs: [y0, y1]
	                    inputs: [x0, x1]
	                    outputs: [y0, y1]
	                  - !transform/compose-1.2.0
	                    forward:
	                    - !transform/remap_axes-1.3.0
	                      inputs: [x0, x1]
	                      mapping: [0, 1, 0, 1]
	                      outputs: [x0, x1, x2, x3]
	                    - !transform/compose-1.2.0
	                      forward:
	                      - !transform/concatenate-1.2.0
	                        forward:
	                        - !transform/polynomial-1.2.0
	                          coefficients: !core/ndarray-1.0.0
	                            source: 12
	                            datatype: float64
	                            byteorder: little
	                            shape: [6, 6]
	                          domain:
	                          - [-1, 1]
	                          - [-1, 1]
	                          inputs: [x, y]
	                          name: camera_x_backward
	                          outputs: [z]
	                          window:
	                          - [-1, 1]
	                          - [-1, 1]
	                        - !transform/polynomial-1.2.0
	                          coefficients: !core/ndarray-1.0.0
	                            source: 13
	                            datatype: float64
	                            byteorder: little
	                            shape: [6, 6]
	                          domain:
	                          - [-1, 1]
	                          - [-1, 1]
	                          inputs: [x, y]
	                          name: camera_y_backward
	                          outputs: [z]
	                          window:
	                          - [-1, 1]
	                          - [-1, 1]
	                        inputs: [x0, y0, x1, y1]
	                        outputs: [z0, z1]
	                      - !transform/identity-1.2.0
	                        inputs: [x0, x1]
	                        n_dims: 2
	                        outputs: [x0, x1]
	                      inputs: [x0, y0, x1, y1]
	                      outputs: [x0, x1]
	                    inputs: [x0, x1]
	                    outputs: [x0, x1]
	                  inputs: [x0, x1]
	                  outputs: [x0, x1]
	                outputs: [y0, y1]
	              inputs: [x, y]
	              outputs: [y0, y1]
	            - !<tag:stsci.edu:jwst_pipeline/coords-1.0.0>
	              inputs: [x, y]
	              model_type: unitless2directional
	              name: unitless2directional_cosines
	              outputs: [x, y, z]
	            inputs: [x, y]
	            outputs: [x, y, z]
	          - !<tag:stsci.edu:jwst_pipeline/rotation_sequence-1.0.0>
	            angles: [0.03333072666861111, -0.27547251631138886, -0.14198882781777777,
	              24.29]
	            axes_order: xyzy
	            inputs: [x, y, z]
	            name: rotation
	            outputs: [x, y, z]
	          inputs: [x, y]
	          outputs: [x, y, z]
	  - !<tag:stsci.edu:gwcs/step-1.0.0>
	        frame: !<tag:stsci.edu:gwcs/frame2d-1.0.0>
	          axes_names: [alpha_in, beta_in]
	          axes_order: [0, 1]
	          axis_physical_types: ['custom:alpha_in', 'custom:beta_in']
	          name: gwa
	          unit: [!unit/unit-1.0.0 rad, !unit/unit-1.0.0 rad]
	        transform: !transform/compose-1.2.0
	          forward:
	          - !transform/compose-1.2.0
	            forward:
	            - !transform/compose-1.2.0
	              forward:
	              - !transform/compose-1.2.0
	                forward:
	                - !transform/compose-1.2.0
	                  forward:
	                  - !transform/compose-1.2.0
	                    forward:
	                    - !transform/compose-1.2.0
	                      forward:
	                      - !transform/remap_axes-1.3.0
	                        inputs: [x0, x1, x2]
	                        mapping: [0, 1, 0, 1]
	                        n_inputs: 3
	                        outputs: [x0, x1, x2, x3]
	                      - !transform/concatenate-1.2.0
	                        forward:
	                        - !transform/concatenate-1.2.0
	                          forward:
	                          - !transform/multiply-1.2.0
	                            forward:
	                            - !transform/constant-1.4.0
	                              dimensions: 1
	                              inputs: [x]
	                              outputs: [y]
	                              value: 0.0
	                            - !transform/identity-1.2.0
	                              inputs: [x0]
	                              outputs: [x0]
	                            inputs: [x]
	                            outputs: [y]
	                          - !transform/multiply-1.2.0
	                            forward:
	                            - !transform/constant-1.4.0
	                              dimensions: 1
	                              inputs: [x]
	                              outputs: [y]
	                              value: -1.0
	                            - !transform/identity-1.2.0
	                              inputs: [x0]
	                              outputs: [x0]
	                            inputs: [x]
	                            outputs: [y]
	                          inputs: [x0, x1]
	                          outputs: [y0, y1]
	                        - !transform/identity-1.2.0
	                          inputs: [x0, x1]
	                          n_dims: 2
	                          outputs: [x0, x1]
	                        inputs: [x00, x10, x01, x11]
	                        outputs: [y0, y1, x0, x1]
	                      inputs: [x0, x1, x2]
	                      outputs: [y0, y1, x0, x1]
	                    - !transform/concatenate-1.2.0
	                      forward:
	                      - !transform/concatenate-1.2.0
	                        forward:
	                        - !transform/identity-1.2.0
	                          inputs: [x0]
	                          outputs: [x0]
	                        - !transform/tabular-1.2.0
	                          bounding_box: [-0.2869219718231398, -0.28489583056156154]
	                          bounds_error: false
	                          fill_value: .nan
	                          inputs: [x]
	                          lookup_table: !core/ndarray-1.0.0
	                            source: 14
	                            datatype: float64
	                            byteorder: little
	                            shape: [1000]
	                          method: linear
	                          name: tabular
	                          outputs: [y]
	                          points:
	                          - !core/ndarray-1.0.0
	                            source: 15
	                            datatype: float64
	                            byteorder: little
	                            shape: [1000]
	                        inputs: [x0, x]
	                        outputs: [x0, y]
	                      - !transform/identity-1.2.0
	                        inputs: [x0, x1]
	                        n_dims: 2
	                        outputs: [x0, x1]
	                      inputs: [x00, x0, x01, x11]
	                      outputs: [x00, y0, x01, x11]
	                    inputs: [x0, x1, x2]
	                    outputs: [x00, y0, x01, x11]
	                  - !transform/remap_axes-1.3.0
	                    inputs: [x0, x1, x2, x3]
	                    mapping: [0, 1, 0, 1, 2, 3]
	                    outputs: [x0, x1, x2, x3, x4, x5]
	                  inputs: [x0, x1, x2]
	                  outputs: [x0, x1, x2, x3, x4, x5]
	                - !transform/concatenate-1.2.0
	                  forward:
	                  - !transform/concatenate-1.2.0
	                    forward:
	                    - !transform/identity-1.2.0
	                      inputs: [x0, x1]
	                      n_dims: 2
	                      outputs: [x0, x1]
	                    - &id001 !transform/compose-1.2.0
	                      forward:
	                      - !transform/compose-1.2.0
	                        forward:
	                        - !transform/compose-1.2.0
	                          forward:
	                          - !transform/concatenate-1.2.0
	                            forward:
	                            - !transform/scale-1.2.0
	                              factor: 8.135000098263845e-05
	                              inputs: [x]
	                              outputs: [y]
	                            - !transform/scale-1.2.0
	                              factor: 0.001271169981919229
	                              inputs: [x]
	                              outputs: [y]
	                            inputs: [x0, x1]
	                            outputs: [y0, y1]
	                          - !transform/concatenate-1.2.0
	                            forward:
	                            - !transform/shift-1.2.0
	                              inputs: [x]
	                              offset: 0.02697242796421051
	                              outputs: [y]
	                            - !transform/shift-1.2.0
	                              inputs: [x]
	                              offset: -0.0027167024090886116
	                              outputs: [y]
	                            inputs: [x0, x1]
	                            outputs: [y0, y1]
	                          inputs: [x0, x1]
	                          outputs: [y0, y1]
	                        - !transform/compose-1.2.0
	                          forward:
	                          - !transform/rotate2d-1.3.0
	                            angle: 0.0
	                            inputs: [x, y]
	                            name: msa_slit_rot
	                            outputs: [x, y]
	                          - !transform/concatenate-1.2.0
	                            forward:
	                            - !transform/shift-1.2.0
	                              inputs: [x]
	                              name: msa_slit_x
	                              offset: 0.0
	                              outputs: [y]
	                            - !transform/shift-1.2.0
	                              inputs: [x]
	                              name: msa_slit_y
	                              offset: 0.0
	                              outputs: [y]
	                            inputs: [x0, x1]
	                            outputs: [y0, y1]
	                          inputs: [x, y]
	                          outputs: [y0, y1]
	                        inputs: [x0, x1]
	                        outputs: [y0, y1]
	                      - !transform/compose-1.2.0
	                        forward:
	                        - !transform/compose-1.2.0
	                          forward:
	                          - !transform/compose-1.2.0
	                            forward:
	                            - !transform/compose-1.2.0
	                              forward:
	                              - !transform/concatenate-1.2.0
	                                forward:
	                                - !transform/shift-1.2.0
	                                  inputs: [x]
	                                  name: collimator_xoutcen_d2s
	                                  offset: -5.526841e-06
	                                  outputs: [y]
	                                - !transform/shift-1.2.0
	                                  inputs: [x]
	                                  name: collimator_youtcen_d2s
	                                  offset: 0.000346042594
	                                  outputs: [y]
	                                inputs: [x0, x1]
	                                outputs: [y0, y1]
	                              - !transform/compose-1.2.0
	                                forward:
	                                - !transform/affine-1.3.0
	                                  inputs: [x, y]
	                                  matrix: !core/ndarray-1.0.0
	                                    source: 16
	                                    datatype: float64
	                                    byteorder: little
	                                    shape: [2, 2]
	                                  outputs: [x, y]
	                                  translation: !core/ndarray-1.0.0
	                                    source: 17
	                                    datatype: float64
	                                    byteorder: little
	                                    shape: [2]
	                                - !transform/concatenate-1.2.0
	                                  forward:
	                                  - !transform/shift-1.2.0
	                                    inputs: [x]
	                                    name: collimator_xincen_d2s
	                                    offset: -0.000143900694035
	                                    outputs: [y]
	                                  - !transform/shift-1.2.0
	                                    inputs: [x]
	                                    name: collimator_yincen_d2s
	                                    offset: -0.293605933112
	                                    outputs: [y]
	                                  inputs: [x0, x1]
	                                  outputs: [y0, y1]
	                                inputs: [x, y]
	                                outputs: [y0, y1]
	                              inputs: [x0, x1]
	                              outputs: [y0, y1]
	                            - !transform/compose-1.2.0
	                              forward:
	                              - !transform/remap_axes-1.3.0
	                                inputs: [x0, x1]
	                                mapping: [0, 1, 0, 1]
	                                outputs: [x0, x1, x2, x3]
	                              - !transform/compose-1.2.0
	                                forward:
	                                - !transform/concatenate-1.2.0
	                                  forward:
	                                  - !transform/polynomial-1.2.0
	                                    coefficients: !core/ndarray-1.0.0
	                                      source: 18
	                                      datatype: float64
	                                      byteorder: little
	                                      shape: [6, 6]
	                                    domain:
	                                    - [-1, 1]
	                                    - [-1, 1]
	                                    inputs: [x, y]
	                                    name: collimator_x_backward
	                                    outputs: [z]
	                                    window:
	                                    - [-1, 1]
	                                    - [-1, 1]
	                                  - !transform/polynomial-1.2.0
	                                    coefficients: !core/ndarray-1.0.0
	                                      source: 19
	                                      datatype: float64
	                                      byteorder: little
	                                      shape: [6, 6]
	                                    domain:
	                                    - [-1, 1]
	                                    - [-1, 1]
	                                    inputs: [x, y]
	                                    name: collimator_y_backward
	                                    outputs: [z]
	                                    window:
	                                    - [-1, 1]
	                                    - [-1, 1]
	                                  inputs: [x0, y0, x1, y1]
	                                  outputs: [z0, z1]
	                                - !transform/identity-1.2.0
	                                  inputs: [x0, x1]
	                                  n_dims: 2
	                                  outputs: [x0, x1]
	                                inputs: [x0, y0, x1, y1]
	                                outputs: [x0, x1]
	                              inputs: [x0, x1]
	                              outputs: [x0, x1]
	                            inputs: [x0, x1]
	                            outputs: [x0, x1]
	                          - !<tag:stsci.edu:jwst_pipeline/coords-1.0.0>
	                            inputs: [x, y]
	                            model_type: unitless2directional
	                            name: unitless2directional_cosines
	                            outputs: [x, y, z]
	                          inputs: [x0, x1]
	                          outputs: [x, y, z]
	                        - !<tag:stsci.edu:jwst_pipeline/rotation_sequence-1.0.0>
	                          angles: [0.03333072666861111, -0.27547251631138886, -0.14198882781777777,
	                            24.29]
	                          axes_order: xyzy
	                          inputs: [x, y, z]
	                          name: rotation
	                          outputs: [x, y, z]
	                        inputs: [x0, x1]
	                        outputs: [x, y, z]
	                      inputs: [x0, x1]
	                      outputs: [x, y, z]
	                    inputs: [x00, x10, x01, x11]
	                    outputs: [x0, x1, x, y, z]
	                  - !transform/identity-1.2.0
	                    inputs: [x0, x1]
	                    n_dims: 2
	                    outputs: [x0, x1]
	                  inputs: [x00, x10, x01, x11, x0, x1]
	                  outputs: [x00, x10, x0, y0, z0, x01, x11]
	                inputs: [x0, x1, x2]
	                outputs: [x00, x10, x0, y0, z0, x01, x11]
	              - !transform/remap_axes-1.3.0
	                inputs: [x0, x1, x2, x3, x4, x5, x6]
	                mapping: [0, 1, 2, 3, 5]
	                n_inputs: 7
	                outputs: [x0, x1, x2, x3, x4]
	              inputs: [x0, x1, x2]
	              outputs: [x0, x1, x2, x3, x4]
	            - !transform/concatenate-1.2.0
	              forward:
	              - !transform/identity-1.2.0
	                inputs: [x0, x1]
	                n_dims: 2
	                outputs: [x0, x1]
	              - !transform/compose-1.2.0
	                forward:
	                - !transform/compose-1.2.0
	                  forward:
	                  - !<tag:stsci.edu:jwst_pipeline/refraction_index_from_prism-1.0.0>
	                    inputs: [alpha_in, beta_in, alpha_out]
	                    name: n_prism
	                    outputs: [n]
	                    prism_angle: -16.5
	                  - !transform/tabular-1.2.0
	                    bounding_box: [1.3871267867024815, 1.4383165119633379]
	                    bounds_error: false
	                    fill_value: .nan
	                    inputs: [x]
	                    lookup_table: !core/ndarray-1.0.0
	                      source: 20
	                      datatype: float64
	                      byteorder: little
	                      shape: [1101]
	                    method: linear
	                    outputs: [y]
	                    points:
	                    - !core/ndarray-1.0.0
	                      source: 21
	                      datatype: float64
	                      byteorder: little
	                      shape: [1101]
	                  inputs: [alpha_in, beta_in, alpha_out]
	                  outputs: [y]
	                - !transform/multiply-1.2.0
	                  forward:
	                  - !transform/identity-1.2.0
	                    inputs: [x0]
	                    outputs: [x0]
	                  - !transform/constant-1.4.0
	                    dimensions: 1
	                    inputs: [x]
	                    name: velocity_correction
	                    outputs: [y]
	                    value: 1.0000046645487086
	                  inputs: [x0]
	                  inverse: !transform/divide-1.2.0
	                    forward:
	                    - !transform/identity-1.2.0
	                      inputs: [x0]
	                      outputs: [x0]
	                    - !transform/constant-1.4.0
	                      dimensions: 1
	                      inputs: [x]
	                      name: inv_vel_correction
	                      outputs: [y]
	                      value: 1.0000046645487086
	                    inputs: [x0]
	                    outputs: [x0]
	                  outputs: [x0]
	                inputs: [alpha_in, beta_in, alpha_out]
	                outputs: [x0]
	              inputs: [x0, x1, alpha_in, beta_in, alpha_out]
	              outputs: [x00, x10, x01]
	            inputs: [x0, x1, x2]
	            outputs: [x00, x10, x01]
	          - !transform/compose-1.2.0
	            forward:
	            - !transform/compose-1.2.0
	              forward:
	              - !transform/compose-1.2.0
	                forward:
	                - !transform/remap_axes-1.3.0
	                  inputs: [x0, x1, x2]
	                  mapping: [0, 1, 2, 1]
	                  outputs: [x0, x1, x2, x3]
	                - !transform/concatenate-1.2.0
	                  forward:
	                  - !transform/identity-1.2.0
	                    inputs: [x0, x1, x2]
	                    n_dims: 3
	                    outputs: [x0, x1, x2]
	                  - !transform/compose-1.2.0
	                    forward:
	                    - !transform/compose-1.2.0
	                      forward:
	                      - !<tag:stsci.edu:jwst_pipeline/logical-1.0.0>
	                        compareto: 0.55
	                        condition: GT
	                        inputs: [x]
	                        outputs: [x]
	                        value: .nan
	                      - !<tag:stsci.edu:jwst_pipeline/logical-1.0.0>
	                        compareto: -0.55
	                        condition: LT
	                        inputs: [x]
	                        outputs: [x]
	                        value: .nan
	                      inputs: [x]
	                      outputs: [x]
	                    - !transform/scale-1.2.0
	                      factor: 0.0
	                      inputs: [x]
	                      outputs: [y]
	                    inputs: [x]
	                    outputs: [y]
	                  inputs: [x0, x1, x2, x]
	                  outputs: [x0, x1, x2, y]
	                inputs: [x0, x1, x2]
	                outputs: [x0, x1, x2, y]
	              - !transform/remap_axes-1.3.0
	                inputs: [x0, x1, x2, x3]
	                mapping: [0, 1, 3, 2, 3]
	                outputs: [x0, x1, x2, x3, x4]
	              inputs: [x0, x1, x2]
	              outputs: [x0, x1, x2, x3, x4]
	            - !transform/concatenate-1.2.0
	              forward:
	              - !transform/concatenate-1.2.0
	                forward:
	                - !transform/identity-1.2.0
	                  inputs: [x0]
	                  outputs: [x0]
	                - !transform/add-1.2.0
	                  forward:
	                  - !transform/remap_axes-1.3.0
	                    inputs: [x0, x1]
	                    mapping: [0]
	                    n_inputs: 2
	                    outputs: [x0]
	                  - !transform/remap_axes-1.3.0
	                    inputs: [x0, x1]
	                    mapping: [1]
	                    outputs: [x0]
	                  inputs: [x0, x1]
	                  outputs: [x0]
	                inputs: [x00, x01, x11]
	                outputs: [x00, x01]
	              - !transform/add-1.2.0
	                forward:
	                - !transform/remap_axes-1.3.0
	                  inputs: [x0, x1]
	                  mapping: [0]
	                  n_inputs: 2
	                  outputs: [x0]
	                - !transform/remap_axes-1.3.0
	                  inputs: [x0, x1]
	                  mapping: [1]
	                  outputs: [x0]
	                inputs: [x0, x1]
	                outputs: [x0]
	              inputs: [x00, x01, x11, x0, x1]
	              outputs: [x00, x01, x0]
	            inputs: [x0, x1, x2]
	            inverse: !transform/identity-1.2.0
	              inputs: [x0, x1, x2]
	              n_dims: 3
	              outputs: [x0, x1, x2]
	            outputs: [x00, x01, x0]
	          inputs: [x0, x1, x2]
	          inverse: !transform/compose-1.2.0
	            forward:
	            - !transform/compose-1.2.0
	              forward:
	              - !transform/concatenate-1.2.0
	                forward:
	                - *id001
	                - !transform/identity-1.2.0
	                  inputs: [x0]
	                  outputs: [x0]
	                inputs: [x00, x10, x01]
	                outputs: [x, y, z, x0]
	              - !transform/remap_axes-1.3.0
	                inputs: [x0, x1, x2, x3]
	                mapping: [3, 0, 1, 2]
	                outputs: [x0, x1, x2, x3]
	              inputs: [x00, x10, x01]
	              outputs: [x0, x1, x2, x3]
	            - !<tag:stsci.edu:jwst_pipeline/snell-1.0.0>
	              inputs: [lam, alpha_in, beta_in, zin]
	              kcoef: [0.58339748, 0.46085267, 3.8915394]
	              lcoef: [0.00252643, 0.010078333, 1200.556]
	              name: snell_law
	              outputs: [alpha_out, beta_out, zout]
	              pressure: 0.0
	              prism_angle: -16.5
	              ref_pressure: 0.0
	              ref_temp: 35.0
	              tcoef: [-2.66e-05, 0.0, 0.0, 0.0, 0.0, 0.0]
	              temp: 40.28447479156018
	            inputs: [x00, x10, x01]
	            outputs: [alpha_out, beta_out, zout]
	          outputs: [x00, x01, x0]
	  - !<tag:stsci.edu:gwcs/step-1.0.0>
	        frame: !<tag:stsci.edu:gwcs/composite_frame-1.0.0>
	          frames:
	          - !<tag:stsci.edu:gwcs/frame2d-1.0.0>
	            axes_names: [x_slit, y_slit]
	            axes_order: [0, 1]
	            axis_physical_types: ['custom:x_slit', 'custom:y_slit']
	            name: slit_spatial
	            unit: [!unit/unit-1.0.0 '', !unit/unit-1.0.0 '']
	          - &id002 !<tag:stsci.edu:gwcs/spectral_frame-1.0.0>
	            axes_names: [wavelength]
	            axes_order: [2]
	            axis_physical_types: [em.wl]
	            name: spectral
	            unit: [!unit/unit-1.0.0 um]
	          name: slit_frame
	        transform: !transform/concatenate-1.2.0
	          forward:
	          - !transform/compose-1.2.0
	            forward:
	            - !transform/compose-1.2.0
	              forward:
	              - !transform/concatenate-1.2.0
	                forward:
	                - !transform/scale-1.2.0
	                  factor: 8.135000098263845e-05
	                  inputs: [x]
	                  outputs: [y]
	                - !transform/scale-1.2.0
	                  factor: 0.001271169981919229
	                  inputs: [x]
	                  outputs: [y]
	                inputs: [x0, x1]
	                outputs: [y0, y1]
	              - !transform/concatenate-1.2.0
	                forward:
	                - !transform/shift-1.2.0
	                  inputs: [x]
	                  offset: 0.02697242796421051
	                  outputs: [y]
	                - !transform/shift-1.2.0
	                  inputs: [x]
	                  offset: -0.0027167024090886116
	                  outputs: [y]
	                inputs: [x0, x1]
	                outputs: [y0, y1]
	              inputs: [x0, x1]
	              outputs: [y0, y1]
	            - !transform/compose-1.2.0
	              forward:
	              - !transform/rotate2d-1.3.0
	                angle: 0.0
	                inputs: [x, y]
	                name: msa_slit_rot
	                outputs: [x, y]
	              - !transform/concatenate-1.2.0
	                forward:
	                - !transform/shift-1.2.0
	                  inputs: [x]
	                  name: msa_slit_x
	                  offset: 0.0
	                  outputs: [y]
	                - !transform/shift-1.2.0
	                  inputs: [x]
	                  name: msa_slit_y
	                  offset: 0.0
	                  outputs: [y]
	                inputs: [x0, x1]
	                outputs: [y0, y1]
	              inputs: [x, y]
	              outputs: [y0, y1]
	            inputs: [x0, x1]
	            outputs: [y0, y1]
	          - !transform/identity-1.2.0
	            inputs: [x0]
	            outputs: [x0]
	          inputs: [x00, x10, x01]
	          outputs: [y0, y1, x0]
	  - !<tag:stsci.edu:gwcs/step-1.0.0>
	        frame: !<tag:stsci.edu:gwcs/composite_frame-1.0.0>
	          frames:
	          - !<tag:stsci.edu:gwcs/frame2d-1.0.0>
	            axes_names: [x_msa, y_msa]
	            axes_order: [0, 1]
	            axis_physical_types: ['custom:x_msa', 'custom:y_msa']
	            name: msa_spatial
	            unit: [!unit/unit-1.0.0 m, !unit/unit-1.0.0 m]
	          - *id002
	          name: msa_frame
	        transform: !transform/compose-1.2.0
	          forward:
	          - !transform/remap_axes-1.3.0
	            inputs: [x0, x1, x2]
	            inverse: !transform/identity-1.2.0
	              inputs: [x0, x1, x2]
	              n_dims: 3
	              outputs: [x0, x1, x2]
	            mapping: [0, 1, 2, 2]
	            name: msa2fore_mapping
	            outputs: [x0, x1, x2, x3]
	          - !transform/concatenate-1.2.0
	            forward:
	            - !transform/compose-1.2.0
	              forward:
	              - !transform/compose-1.2.0
	                forward:
	                - !transform/compose-1.2.0
	                  forward:
	                  - !transform/remap_axes-1.3.0
	                    inputs: [x0, x1, x2]
	                    inverse: !transform/identity-1.2.0
	                      inputs: [x0, x1]
	                      n_dims: 2
	                      outputs: [x0, x1]
	                    mapping: [0, 1, 2, 0, 1, 2]
	                    name: fore_inmap
	                    outputs: [x0, x1, x2, x3, x4, x5]
	                  - !transform/concatenate-1.2.0
	                    forward:
	                    - !transform/add-1.2.0
	                      forward:
	                      - !transform/compose-1.2.0
	                        forward:
	                        - !transform/remap_axes-1.3.0
	                          inputs: [x0, x1, x2]
	                          mapping: [0, 1]
	                          n_inputs: 3
	                          outputs: [x0, x1]
	                        - !transform/polynomial-1.2.0
	                          coefficients: !core/ndarray-1.0.0
	                            source: 22
	                            datatype: float64
	                            byteorder: little
	                            shape: [6, 6]
	                          domain:
	                          - [-1, 1]
	                          - [-1, 1]
	                          inputs: [x, y]
	                          name: fore_x_forw
	                          outputs: [z]
	                          window:
	                          - [-1, 1]
	                          - [-1, 1]
	                        inputs: [x0, x1, x2]
	                        outputs: [z]
	                      - !transform/multiply-1.2.0
	                        forward:
	                        - !transform/compose-1.2.0
	                          forward:
	                          - !transform/remap_axes-1.3.0
	                            inputs: [x0, x1, x2]
	                            mapping: [0, 1]
	                            n_inputs: 3
	                            outputs: [x0, x1]
	                          - !transform/polynomial-1.2.0
	                            coefficients: !core/ndarray-1.0.0
	                              source: 23
	                              datatype: float64
	                              byteorder: little
	                              shape: [6, 6]
	                            domain:
	                            - [-1, 1]
	                            - [-1, 1]
	                            inputs: [x, y]
	                            name: fore_x_forwdist
	                            outputs: [z]
	                            window:
	                            - [-1, 1]
	                            - [-1, 1]
	                          inputs: [x0, x1, x2]
	                          outputs: [z]
	                        - !transform/compose-1.2.0
	                          forward:
	                          - !transform/remap_axes-1.3.0
	                            inputs: [x0, x1, x2]
	                            mapping: [2]
	                            outputs: [x0]
	                          - !transform/identity-1.2.0
	                            inputs: [x0]
	                            outputs: [x0]
	                          inputs: [x0, x1, x2]
	                          outputs: [x0]
	                        inputs: [x0, x1, x2]
	                        outputs: [z]
	                      inputs: [x0, x1, x2]
	                      outputs: [z]
	                    - !transform/add-1.2.0
	                      forward:
	                      - !transform/compose-1.2.0
	                        forward:
	                        - !transform/remap_axes-1.3.0
	                          inputs: [x0, x1, x2]
	                          mapping: [0, 1]
	                          n_inputs: 3
	                          outputs: [x0, x1]
	                        - !transform/polynomial-1.2.0
	                          coefficients: !core/ndarray-1.0.0
	                            source: 24
	                            datatype: float64
	                            byteorder: little
	                            shape: [6, 6]
	                          domain:
	                          - [-1, 1]
	                          - [-1, 1]
	                          inputs: [x, y]
	                          name: fore_y_forw
	                          outputs: [z]
	                          window:
	                          - [-1, 1]
	                          - [-1, 1]
	                        inputs: [x0, x1, x2]
	                        outputs: [z]
	                      - !transform/multiply-1.2.0
	                        forward:
	                        - !transform/compose-1.2.0
	                          forward:
	                          - !transform/remap_axes-1.3.0
	                            inputs: [x0, x1, x2]
	                            mapping: [0, 1]
	                            n_inputs: 3
	                            outputs: [x0, x1]
	                          - !transform/polynomial-1.2.0
	                            coefficients: !core/ndarray-1.0.0
	                              source: 25
	                              datatype: float64
	                              byteorder: little
	                              shape: [6, 6]
	                            domain:
	                            - [-1, 1]
	                            - [-1, 1]
	                            inputs: [x, y]
	                            name: fore_y_forwdist
	                            outputs: [z]
	                            window:
	                            - [-1, 1]
	                            - [-1, 1]
	                          inputs: [x0, x1, x2]
	                          outputs: [z]
	                        - !transform/compose-1.2.0
	                          forward:
	                          - !transform/remap_axes-1.3.0
	                            inputs: [x0, x1, x2]
	                            mapping: [2]
	                            outputs: [x0]
	                          - !transform/identity-1.2.0
	                            inputs: [x0]
	                            outputs: [x0]
	                          inputs: [x0, x1, x2]
	                          outputs: [x0]
	                        inputs: [x0, x1, x2]
	                        outputs: [z]
	                      inputs: [x0, x1, x2]
	                      outputs: [z]
	                    inputs: [x00, x10, x20, x01, x11, x21]
	                    outputs: [z0, z1]
	                  inputs: [x0, x1, x2]
	                  outputs: [z0, z1]
	                - !transform/identity-1.2.0
	                  inputs: [x0, x1]
	                  inverse: !transform/remap_axes-1.3.0
	                    inputs: [x0, x1, x2]
	                    mapping: [0, 1, 2, 0, 1, 2]
	                    outputs: [x0, x1, x2, x3, x4, x5]
	                  n_dims: 2
	                  name: fore_outmap
	                  outputs: [x0, x1]
	                inputs: [x0, x1, x2]
	                outputs: [x0, x1]
	              - !transform/compose-1.2.0
	                forward:
	                - !transform/compose-1.2.0
	                  forward:
	                  - !transform/concatenate-1.2.0
	                    forward:
	                    - !transform/shift-1.2.0
	                      inputs: [x]
	                      name: fore_xincen_d2s
	                      offset: -5.52684591413e-06
	                      outputs: [y]
	                    - !transform/shift-1.2.0
	                      inputs: [x]
	                      name: fore_yincen_d2s
	                      offset: 0.000346028872881
	                      outputs: [y]
	                    inputs: [x0, x1]
	                    outputs: [y0, y1]
	                  - !transform/affine-1.3.0
	                    inputs: [x, y]
	                    matrix: !core/ndarray-1.0.0
	                      source: 26
	                      datatype: float64
	                      byteorder: little
	                      shape: [2, 2]
	                    name: fore_affine_d2s
	                    outputs: [x, y]
	                    translation: !core/ndarray-1.0.0
	                      source: 27
	                      datatype: float64
	                      byteorder: little
	                      shape: [2]
	                  inputs: [x0, x1]
	                  outputs: [x, y]
	                - !transform/concatenate-1.2.0
	                  forward:
	                  - !transform/shift-1.2.0
	                    inputs: [x]
	                    name: fore_xoutcen_d2s
	                    offset: -2.27962e-07
	                    outputs: [y]
	                  - !transform/shift-1.2.0
	                    inputs: [x]
	                    name: fore_youtcen_d2s
	                    offset: -2.6094e-07
	                    outputs: [y]
	                  inputs: [x0, x1]
	                  outputs: [y0, y1]
	                inputs: [x0, x1]
	                outputs: [y0, y1]
	              inputs: [x0, x1, x2]
	              inverse: !transform/compose-1.2.0
	                forward:
	                - !transform/concatenate-1.2.0
	                  forward:
	                  - !transform/compose-1.2.0
	                    forward:
	                    - !transform/concatenate-1.2.0
	                      forward:
	                      - !transform/shift-1.2.0
	                        inputs: [x]
	                        name: fore_xoutcen_d2s
	                        offset: 2.27962e-07
	                        outputs: [y]
	                      - !transform/shift-1.2.0
	                        inputs: [x]
	                        name: fore_youtcen_d2s
	                        offset: 2.6094e-07
	                        outputs: [y]
	                      inputs: [x0, x1]
	                      outputs: [y0, y1]
	                    - !transform/compose-1.2.0
	                      forward:
	                      - !transform/affine-1.3.0
	                        inputs: [x, y]
	                        matrix: !core/ndarray-1.0.0
	                          source: 28
	                          datatype: float64
	                          byteorder: little
	                          shape: [2, 2]
	                        outputs: [x, y]
	                        translation: !core/ndarray-1.0.0
	                          source: 29
	                          datatype: float64
	                          byteorder: little
	                          shape: [2]
	                      - !transform/concatenate-1.2.0
	                        forward:
	                        - !transform/shift-1.2.0
	                          inputs: [x]
	                          name: fore_xincen_d2s
	                          offset: 5.52684591413e-06
	                          outputs: [y]
	                        - !transform/shift-1.2.0
	                          inputs: [x]
	                          name: fore_yincen_d2s
	                          offset: -0.000346028872881
	                          outputs: [y]
	                        inputs: [x0, x1]
	                        outputs: [y0, y1]
	                      inputs: [x, y]
	                      outputs: [y0, y1]
	                    inputs: [x0, x1]
	                    outputs: [y0, y1]
	                  - !transform/identity-1.2.0
	                    inputs: [x0]
	                    outputs: [x0]
	                  inputs: [x00, x10, x01]
	                  outputs: [y0, y1, x0]
	                - !transform/compose-1.2.0
	                  forward:
	                  - !transform/remap_axes-1.3.0
	                    inputs: [x0, x1, x2]
	                    mapping: [0, 1, 2, 0, 1, 2]
	                    outputs: [x0, x1, x2, x3, x4, x5]
	                  - !transform/compose-1.2.0
	                    forward:
	                    - !transform/concatenate-1.2.0
	                      forward:
	                      - !transform/add-1.2.0
	                        forward:
	                        - !transform/compose-1.2.0
	                          forward:
	                          - !transform/remap_axes-1.3.0
	                            inputs: [x0, x1, x2]
	                            mapping: [0, 1]
	                            n_inputs: 3
	                            outputs: [x0, x1]
	                          - !transform/polynomial-1.2.0
	                            coefficients: !core/ndarray-1.0.0
	                              source: 30
	                              datatype: float64
	                              byteorder: little
	                              shape: [6, 6]
	                            domain:
	                            - [-1, 1]
	                            - [-1, 1]
	                            inputs: [x, y]
	                            name: fore_x_back
	                            outputs: [z]
	                            window:
	                            - [-1, 1]
	                            - [-1, 1]
	                          inputs: [x0, x1, x2]
	                          outputs: [z]
	                        - !transform/multiply-1.2.0
	                          forward:
	                          - !transform/compose-1.2.0
	                            forward:
	                            - !transform/remap_axes-1.3.0
	                              inputs: [x0, x1, x2]
	                              mapping: [0, 1]
	                              n_inputs: 3
	                              outputs: [x0, x1]
	                            - !transform/polynomial-1.2.0
	                              coefficients: !core/ndarray-1.0.0
	                                source: 31
	                                datatype: float64
	                                byteorder: little
	                                shape: [6, 6]
	                              domain:
	                              - [-1, 1]
	                              - [-1, 1]
	                              inputs: [x, y]
	                              name: fore_x_backdist
	                              outputs: [z]
	                              window:
	                              - [-1, 1]
	                              - [-1, 1]
	                            inputs: [x0, x1, x2]
	                            outputs: [z]
	                          - !transform/compose-1.2.0
	                            forward:
	                            - !transform/remap_axes-1.3.0
	                              inputs: [x0, x1, x2]
	                              mapping: [2]
	                              outputs: [x0]
	                            - !transform/identity-1.2.0
	                              inputs: [x0]
	                              outputs: [x0]
	                            inputs: [x0, x1, x2]
	                            outputs: [x0]
	                          inputs: [x0, x1, x2]
	                          outputs: [z]
	                        inputs: [x0, x1, x2]
	                        outputs: [z]
	                      - !transform/add-1.2.0
	                        forward:
	                        - !transform/compose-1.2.0
	                          forward:
	                          - !transform/remap_axes-1.3.0
	                            inputs: [x0, x1, x2]
	                            mapping: [0, 1]
	                            n_inputs: 3
	                            outputs: [x0, x1]
	                          - !transform/polynomial-1.2.0
	                            coefficients: !core/ndarray-1.0.0
	                              source: 32
	                              datatype: float64
	                              byteorder: little
	                              shape: [6, 6]
	                            domain:
	                            - [-1, 1]
	                            - [-1, 1]
	                            inputs: [x, y]
	                            name: fore_y_back
	                            outputs: [z]
	                            window:
	                            - [-1, 1]
	                            - [-1, 1]
	                          inputs: [x0, x1, x2]
	                          outputs: [z]
	                        - !transform/multiply-1.2.0
	                          forward:
	                          - !transform/compose-1.2.0
	                            forward:
	                            - !transform/remap_axes-1.3.0
	                              inputs: [x0, x1, x2]
	                              mapping: [0, 1]
	                              n_inputs: 3
	                              outputs: [x0, x1]
	                            - !transform/polynomial-1.2.0
	                              coefficients: !core/ndarray-1.0.0
	                                source: 33
	                                datatype: float64
	                                byteorder: little
	                                shape: [6, 6]
	                              domain:
	                              - [-1, 1]
	                              - [-1, 1]
	                              inputs: [x, y]
	                              name: fore_y_backdist
	                              outputs: [z]
	                              window:
	                              - [-1, 1]
	                              - [-1, 1]
	                            inputs: [x0, x1, x2]
	                            outputs: [z]
	                          - !transform/compose-1.2.0
	                            forward:
	                            - !transform/remap_axes-1.3.0
	                              inputs: [x0, x1, x2]
	                              mapping: [2]
	                              outputs: [x0]
	                            - !transform/identity-1.2.0
	                              inputs: [x0]
	                              outputs: [x0]
	                            inputs: [x0, x1, x2]
	                            outputs: [x0]
	                          inputs: [x0, x1, x2]
	                          outputs: [z]
	                        inputs: [x0, x1, x2]
	                        outputs: [z]
	                      inputs: [x00, x10, x20, x01, x11, x21]
	                      outputs: [z0, z1]
	                    - !transform/identity-1.2.0
	                      inputs: [x0, x1]
	                      n_dims: 2
	                      outputs: [x0, x1]
	                    inputs: [x00, x10, x20, x01, x11, x21]
	                    outputs: [x0, x1]
	                  inputs: [x0, x1, x2]
	                  outputs: [x0, x1]
	                inputs: [x00, x10, x01]
	                outputs: [x0, x1]
	              outputs: [y0, y1]
	            - !transform/identity-1.2.0
	              inputs: [x0]
	              outputs: [x0]
	            inputs: [x00, x10, x20, x01]
	            outputs: [y0, y1, x0]
	          inputs: [x0, x1, x2]
	          name: msa2oteip
	          outputs: [y0, y1, x0]
	  - !<tag:stsci.edu:gwcs/step-1.0.0>
	        frame: !<tag:stsci.edu:gwcs/composite_frame-1.0.0>
	          frames:
	          - !<tag:stsci.edu:gwcs/frame2d-1.0.0>
	            axes_names: [X_OTEIP, Y_OTEIP]
	            axes_order: [0, 1]
	            axis_physical_types: ['custom:X_OTEIP', 'custom:Y_OTEIP']
	            name: oteip
	            unit: [!unit/unit-1.0.0 deg, !unit/unit-1.0.0 deg]
	          - *id002
	          name: oteip
	        transform: !transform/compose-1.2.0
	          forward:
	          - !transform/identity-1.2.0
	            inputs: [x0, x1, x2]
	            inverse: !transform/remap_axes-1.3.0
	              inputs: [x0, x1, x2]
	              mapping: [0, 1, 2, 2]
	              outputs: [x0, x1, x2, x3]
	            n_dims: 3
	            name: fore2ote_mapping
	            outputs: [x0, x1, x2]
	          - !transform/concatenate-1.2.0
	            forward:
	            - !transform/compose-1.2.0
	              forward:
	              - !transform/compose-1.2.0
	                forward:
	                - !transform/compose-1.2.0
	                  forward:
	                  - !transform/compose-1.2.0
	                    forward:
	                    - !transform/remap_axes-1.3.0
	                      inputs: [x0, x1]
	                      inverse: !transform/identity-1.2.0
	                        inputs: [x0, x1]
	                        n_dims: 2
	                        outputs: [x0, x1]
	                      mapping: [0, 1, 0, 1]
	                      name: ote_inmap
	                      outputs: [x0, x1, x2, x3]
	                    - !transform/concatenate-1.2.0
	                      forward:
	                      - !transform/polynomial-1.2.0
	                        coefficients: !core/ndarray-1.0.0
	                          source: 34
	                          datatype: float64
	                          byteorder: little
	                          shape: [6, 6]
	                        domain:
	                        - [-1, 1]
	                        - [-1, 1]
	                        inputs: [x, y]
	                        inverse: !transform/polynomial-1.2.0
	                          coefficients: !core/ndarray-1.0.0
	                            source: 35
	                            datatype: float64
	                            byteorder: little
	                            shape: [6, 6]
	                          domain:
	                          - [-1, 1]
	                          - [-1, 1]
	                          inputs: [x, y]
	                          name: ote_x_back
	                          outputs: [z]
	                          window:
	                          - [-1, 1]
	                          - [-1, 1]
	                        name: ote_x_forw
	                        outputs: [z]
	                        window:
	                        - [-1, 1]
	                        - [-1, 1]
	                      - !transform/polynomial-1.2.0
	                        coefficients: !core/ndarray-1.0.0
	                          source: 36
	                          datatype: float64
	                          byteorder: little
	                          shape: [6, 6]
	                        domain:
	                        - [-1, 1]
	                        - [-1, 1]
	                        inputs: [x, y]
	                        inverse: !transform/polynomial-1.2.0
	                          coefficients: !core/ndarray-1.0.0
	                            source: 37
	                            datatype: float64
	                            byteorder: little
	                            shape: [6, 6]
	                          domain:
	                          - [-1, 1]
	                          - [-1, 1]
	                          inputs: [x, y]
	                          name: ote_y_backw
	                          outputs: [z]
	                          window:
	                          - [-1, 1]
	                          - [-1, 1]
	                        name: ote_y_forw
	                        outputs: [z]
	                        window:
	                        - [-1, 1]
	                        - [-1, 1]
	                      inputs: [x0, y0, x1, y1]
	                      outputs: [z0, z1]
	                    inputs: [x0, x1]
	                    outputs: [z0, z1]
	                  - !transform/identity-1.2.0
	                    inputs: [x0, x1]
	                    inverse: !transform/remap_axes-1.3.0
	                      inputs: [x0, x1]
	                      mapping: [0, 1, 0, 1]
	                      outputs: [x0, x1, x2, x3]
	                    n_dims: 2
	                    name: ote_outmap
	                    outputs: [x0, x1]
	                  inputs: [x0, x1]
	                  outputs: [x0, x1]
	                - !transform/compose-1.2.0
	                  forward:
	                  - !transform/compose-1.2.0
	                    forward:
	                    - !transform/concatenate-1.2.0
	                      forward:
	                      - !transform/shift-1.2.0
	                        inputs: [x]
	                        name: ote_xincen_d2s
	                        offset: 5.18289805611e-07
	                        outputs: [y]
	                      - !transform/shift-1.2.0
	                        inputs: [x]
	                        name: ote_yincen_d2s
	                        offset: 1.92704532397e-09
	                        outputs: [y]
	                      inputs: [x0, x1]
	                      outputs: [y0, y1]
	                    - !transform/affine-1.3.0
	                      inputs: [x, y]
	                      matrix: !core/ndarray-1.0.0
	                        source: 38
	                        datatype: float64
	                        byteorder: little
	                        shape: [2, 2]
	                      name: ote_affine_d2s
	                      outputs: [x, y]
	                      translation: !core/ndarray-1.0.0
	                        source: 39
	                        datatype: float64
	                        byteorder: little
	                        shape: [2]
	                    inputs: [x0, x1]
	                    outputs: [x, y]
	                  - !transform/concatenate-1.2.0
	                    forward:
	                    - !transform/shift-1.2.0
	                      inputs: [x]
	                      name: ote_xoutcen_d2s
	                      offset: 0.10539
	                      outputs: [y]
	                    - !transform/shift-1.2.0
	                      inputs: [x]
	                      name: ote_youtcen_d2s
	                      offset: -0.11913000025
	                      outputs: [y]
	                    inputs: [x0, x1]
	                    outputs: [y0, y1]
	                  inputs: [x0, x1]
	                  outputs: [y0, y1]
	                inputs: [x0, x1]
	                outputs: [y0, y1]
	              - !transform/concatenate-1.2.0
	                forward:
	                - !transform/scale-1.2.0
	                  factor: 3600.0
	                  inputs: [x]
	                  outputs: [y]
	                - !transform/scale-1.2.0
	                  factor: 3600.0
	                  inputs: [x]
	                  outputs: [y]
	                inputs: [x0, x1]
	                outputs: [y0, y1]
	              inputs: [x0, x1]
	              inverse: !transform/compose-1.2.0
	                forward:
	                - !transform/concatenate-1.2.0
	                  forward:
	                  - !transform/scale-1.2.0
	                    factor: 0.0002777777777777778
	                    inputs: [x]
	                    outputs: [y]
	                  - !transform/scale-1.2.0
	                    factor: 0.0002777777777777778
	                    inputs: [x]
	                    outputs: [y]
	                  inputs: [x0, x1]
	                  outputs: [y0, y1]
	                - !transform/compose-1.2.0
	                  forward:
	                  - !transform/compose-1.2.0
	                    forward:
	                    - !transform/concatenate-1.2.0
	                      forward:
	                      - !transform/shift-1.2.0
	                        inputs: [x]
	                        name: ote_xoutcen_d2s
	                        offset: -0.10539
	                        outputs: [y]
	                      - !transform/shift-1.2.0
	                        inputs: [x]
	                        name: ote_youtcen_d2s
	                        offset: 0.11913000025
	                        outputs: [y]
	                      inputs: [x0, x1]
	                      outputs: [y0, y1]
	                    - !transform/compose-1.2.0
	                      forward:
	                      - !transform/affine-1.3.0
	                        inputs: [x, y]
	                        matrix: !core/ndarray-1.0.0
	                          source: 40
	                          datatype: float64
	                          byteorder: little
	                          shape: [2, 2]
	                        outputs: [x, y]
	                        translation: !core/ndarray-1.0.0
	                          source: 41
	                          datatype: float64
	                          byteorder: little
	                          shape: [2]
	                      - !transform/concatenate-1.2.0
	                        forward:
	                        - !transform/shift-1.2.0
	                          inputs: [x]
	                          name: ote_xincen_d2s
	                          offset: -5.18289805611e-07
	                          outputs: [y]
	                        - !transform/shift-1.2.0
	                          inputs: [x]
	                          name: ote_yincen_d2s
	                          offset: -1.92704532397e-09
	                          outputs: [y]
	                        inputs: [x0, x1]
	                        outputs: [y0, y1]
	                      inputs: [x, y]
	                      outputs: [y0, y1]
	                    inputs: [x0, x1]
	                    outputs: [y0, y1]
	                  - !transform/compose-1.2.0
	                    forward:
	                    - !transform/remap_axes-1.3.0
	                      inputs: [x0, x1]
	                      mapping: [0, 1, 0, 1]
	                      outputs: [x0, x1, x2, x3]
	                    - !transform/compose-1.2.0
	                      forward:
	                      - !transform/concatenate-1.2.0
	                        forward:
	                        - !transform/polynomial-1.2.0
	                          coefficients: !core/ndarray-1.0.0
	                            source: 42
	                            datatype: float64
	                            byteorder: little
	                            shape: [6, 6]
	                          domain:
	                          - [-1, 1]
	                          - [-1, 1]
	                          inputs: [x, y]
	                          name: ote_x_back
	                          outputs: [z]
	                          window:
	                          - [-1, 1]
	                          - [-1, 1]
	                        - !transform/polynomial-1.2.0
	                          coefficients: !core/ndarray-1.0.0
	                            source: 43
	                            datatype: float64
	                            byteorder: little
	                            shape: [6, 6]
	                          domain:
	                          - [-1, 1]
	                          - [-1, 1]
	                          inputs: [x, y]
	                          name: ote_y_backw
	                          outputs: [z]
	                          window:
	                          - [-1, 1]
	                          - [-1, 1]
	                        inputs: [x0, y0, x1, y1]
	                        outputs: [z0, z1]
	                      - !transform/identity-1.2.0
	                        inputs: [x0, x1]
	                        n_dims: 2
	                        outputs: [x0, x1]
	                      inputs: [x0, y0, x1, y1]
	                      outputs: [x0, x1]
	                    inputs: [x0, x1]
	                    outputs: [x0, x1]
	                  inputs: [x0, x1]
	                  outputs: [x0, x1]
	                inputs: [x0, x1]
	                outputs: [x0, x1]
	              outputs: [y0, y1]
	            - !transform/scale-1.2.0
	              factor: 1000000.0
	              inputs: [x]
	              outputs: [y]
	            inputs: [x0, x1, x]
	            outputs: [y0, y1, y]
	          inputs: [x0, x1, x2]
	          name: oteip2v23
	          outputs: [y0, y1, y]
	  - !<tag:stsci.edu:gwcs/step-1.0.0>
	        frame: !<tag:stsci.edu:gwcs/composite_frame-1.0.0>
	          frames:
	          - !<tag:stsci.edu:gwcs/frame2d-1.0.0>
	            axes_names: [v2, v3]
	            axes_order: [0, 1]
	            axis_physical_types: ['custom:v2', 'custom:v3']
	            name: v2v3_spatial
	            unit: [!unit/unit-1.0.0 arcsec, !unit/unit-1.0.0 arcsec]
	          - *id002
	          name: v2v3
	        transform: !transform/concatenate-1.2.0
	          forward:
	          - !transform/compose-1.2.0
	            forward:
	            - !transform/concatenate-1.2.0
	              forward:
	              - !transform/scale-1.2.0
	                factor: 0.9999997262839518
	                inputs: [x]
	                name: dva_scale_v2
	                outputs: [y]
	              - !transform/scale-1.2.0
	                factor: 0.9999997262839518
	                inputs: [x]
	                name: dva_scale_v3
	                outputs: [y]
	              inputs: [x0, x1]
	              outputs: [y0, y1]
	            - !transform/concatenate-1.2.0
	              forward:
	              - !transform/shift-1.2.0
	                inputs: [x]
	                name: dva_v2_shift
	                offset: 9.091097472161734e-05
	                outputs: [y]
	              - !transform/shift-1.2.0
	                inputs: [x]
	                name: dva_v3_shift
	                offset: -0.00013117135776508434
	                outputs: [y]
	              inputs: [x0, x1]
	              outputs: [y0, y1]
	            inputs: [x0, x1]
	            name: DVA_Correction
	            outputs: [y0, y1]
	          - !transform/identity-1.2.0
	            inputs: [x0]
	            outputs: [x0]
	          inputs: [x00, x10, x01]
	          outputs: [y0, y1, x0]
	  - !<tag:stsci.edu:gwcs/step-1.0.0>
	        frame: !<tag:stsci.edu:gwcs/composite_frame-1.0.0>
	          frames:
	          - !<tag:stsci.edu:gwcs/frame2d-1.0.0>
	            axes_names: [v2, v3]
	            axes_order: [0, 1]
	            axis_physical_types: ['custom:v2', 'custom:v3']
	            name: v2v3vacorr_spatial
	            unit: [!unit/unit-1.0.0 arcsec, !unit/unit-1.0.0 arcsec]
	          - *id002
	          name: v2v3vacorr
	        transform: !transform/concatenate-1.2.0
	          forward:
	          - !transform/compose-1.2.0
	            forward:
	            - !transform/compose-1.2.0
	              forward:
	              - !transform/compose-1.2.0
	                forward:
	                - !transform/concatenate-1.2.0
	                  forward:
	                  - !transform/scale-1.2.0
	                    factor: 0.0002777777777777778
	                    inputs: [x]
	                    outputs: [y]
	                  - !transform/scale-1.2.0
	                    factor: 0.0002777777777777778
	                    inputs: [x]
	                    outputs: [y]
	                  inputs: [x0, x1]
	                  outputs: [y0, y1]
	                - !<tag:stsci.edu:gwcs/spherical_cartesian-1.0.0>
	                  inputs: [lon, lat]
	                  outputs: [x, y, z]
	                  transform_type: spherical_to_cartesian
	                  wrap_lon_at: 180
	                inputs: [x0, x1]
	                outputs: [x, y, z]
	              - !transform/rotate_sequence_3d-1.0.0
	                angles: [0.09226002166666666, 0.13311783694444446, -93.7605896, -70.775099941418,
	                  -90.75467525972158]
	                axes_order: zyxyz
	                inputs: [x, y, z]
	                outputs: [x, y, z]
	                rotation_type: cartesian
	              inputs: [x0, x1]
	              outputs: [x, y, z]
	            - !<tag:stsci.edu:gwcs/spherical_cartesian-1.0.0>
	              inputs: [x, y, z]
	              outputs: [lon, lat]
	              transform_type: cartesian_to_spherical
	              wrap_lon_at: 360
	            inputs: [x0, x1]
	            name: v23tosky
	            outputs: [lon, lat]
	          - !transform/identity-1.2.0
	            inputs: [x0]
	            outputs: [x0]
	          inputs: [x00, x10, x01]
	          name: v2v3_to_sky
	          outputs: [lon, lat, x0]
	  - !<tag:stsci.edu:gwcs/step-1.0.0>
	        frame: !<tag:stsci.edu:gwcs/composite_frame-1.0.0>
	          frames:
	          - !<tag:stsci.edu:gwcs/celestial_frame-1.0.0>
	            axes_names: [lon, lat]
	            axes_order: [0, 1]
	            axis_physical_types: [pos.eq.ra, pos.eq.dec]
	            name: sky
	            reference_frame: !<tag:astropy.org:astropy/coordinates/frames/icrs-1.1.0>
	              frame_attributes: {}
	            unit: [!unit/unit-1.0.0 deg, !unit/unit-1.0.0 deg]
	          - *id002
	          name: world
	        transform: null
	...

.. _`Appendix D`:

Appendix D: Edited version of Appendix C
========================================

::

	#ASDF 1.0.0
	#ASDF_STANDARD 1.5.0
	%YAML 1.1
	%TAG ! tag:stsci.edu:asdf/
	--- !core/asdf-1.1.0
	asdf_library: !core/software-1.0.0 {author: The ASDF Developers, homepage: 'http://github.com/asdf-format/asdf',
	  name: asdf, version: 2.11.2.dev15+g6703d8f.d20220729}
	history:
	  extensions:
	  - !core/extension_metadata-1.0.0
	        extension_class: asdf.extension.BuiltinExtension
	        software: !core/software-1.0.0 {name: asdf, version: 2.11.2.dev15+g6703d8f.d20220729}
	  - !core/extension_metadata-1.0.0
	        extension_class: asdf.extension._manifest.ManifestExtension
	        extension_uri: asdf://asdf-format.org/astronomy/gwcs/extensions/gwcs-1.0.0
	        software: !core/software-1.0.0 {name: gwcs, version: 0.18.1}
	  - !core/extension_metadata-1.0.0
	        extension_class: asdf.extension._manifest.ManifestExtension
	        extension_uri: asdf://asdf-format.org/transform/extensions/transform-1.5.0
	        software: !core/software-1.0.0 {name: asdf-astropy, version: 0.2.1}
	  - !core/extension_metadata-1.0.0
	        extension_class: asdf.extension._manifest.ManifestExtension
	        extension_uri: asdf://asdf-format.org/astronomy/coordinates/extensions/coordinates-1.0.0
	        software: !core/software-1.0.0 {name: asdf-astropy, version: 0.2.1}
	  - !core/extension_metadata-1.0.0
	        extension_class: asdf.extension._manifest.ManifestExtension
	        extension_uri: asdf://stsci.edu/jwst_pipeline/extensions/jwst_transforms-1.0.0
	        software: !core/software-1.0.0 {name: jwst, version: 1.2.4.dev211+g5ce9e04b}
	  - !core/extension_metadata-1.0.0
	        extension_class: asdf.extension._manifest.ManifestExtension
	        extension_uri: asdf://asdf-format.org/core/extensions/core-1.5.0
	        software: !core/software-1.0.0 {name: asdf-astropy, version: 0.2.1}
	wcs:
	  name: ''
	  steps:
	  - frame:
	          axes_names: [x, y]
	          axes_order: [0, 1]
	          axis_physical_types: ['custom:x', 'custom:y']
	          name: detector
	          unit: [pixel, pixel]
	        transform:
	          bounding_box:
	          - [-0.5, 38.5]
	          - [-0.5, 434.5]
	          forward:
	          - forward:
	            - inputs: [x]
	              offset: 1083.0
	              outputs: [y]
	            - inputs: [x]
	              offset: 4.0
	              outputs: [y]
	            inputs: [x0, x1]
	            outputs: [y0, y1]
	          - bounding_box:
	            - [3.5, 42.5]
	            - [1083.4770570798173, 1518.1537632494133]
	            forward:
	            - forward:
	              - inputs: [x]
	                offset: 0.0
	                outputs: [y]
	              - inputs: [x]
	                offset: 1053.0
	                outputs: [y]
	              inputs: [x0, x1]
	              outputs: [y0, y1]
	            - inputs: [x0, x1]
	              n_dims: 2
	              outputs: [x0, x1]
	            inputs: [x0, x1]
	            name: dms2sca
	            outputs: [x0, x1]
	          inputs: [x0, x1]
	          name: dms2sca
	          outputs: [x0, x1]
	  - frame:
	          axes_names: [x, y]
	          axes_order: [0, 1]
	          axis_physical_types: ['custom:x', 'custom:y']
	          name: sca
	          unit: [pixel, pixel]
	        transform:
	          forward:
	          - forward:
	            - forward:
	              - forward:
	                - inputs: [x, y]
	                  matrix: !core/ndarray-1.0.0
	                    data:
	                    - [1.8e-05, 0.0]
	                    - [0.0, 1.8e-05]
	                    datatype: float64
	                    shape: [2, 2]
	                  name: fpa_affine_d2s
	                  outputs: [x, y]
	                  translation: !core/ndarray-1.0.0
	                    data: [0.0, 0.0]
	                    datatype: float64
	                    shape: [2]
	                - forward:
	                  - inputs: [x]
	                    name: fpa_x_d2s
	                    offset: -0.0381708371805
	                    outputs: [y]
	                  - inputs: [x]
	                    name: fpa_y_d2s
	                    offset: -0.018423
	                    outputs: [y]
	                  inputs: [x0, x1]
	                  outputs: [y0, y1]
	                inputs: [x, y]
	                inverse:
	                  forward:
	                  - forward:
	                    - inputs: [x]
	                      name: fpa_x_s2d
	                      offset: 0.0381708371805
	                      outputs: [y]
	                    - inputs: [x]
	                      name: fpa_y_s2d
	                      offset: 0.018423
	                      outputs: [y]
	                    inputs: [x0, x1]
	                    outputs: [y0, y1]
	                  - inputs: [x, y]
	                    matrix: !core/ndarray-1.0.0
	                      data:
	                      - [55555.555555555555, 0.0]
	                      - [0.0, 55555.555555555555]
	                      datatype: float64
	                      shape: [2, 2]
	                    name: fpa_affine_s2d
	                    outputs: [x, y]
	                    translation: !core/ndarray-1.0.0
	                      data: [0.0, 0.0]
	                      datatype: float64
	                      shape: [2]
	                  inputs: [x0, x1]
	                  outputs: [x, y]
	                outputs: [y0, y1]
	              - forward:
	                - forward:
	                  - forward:
	                    - inputs: [x0, x1]
	                      inverse:
	                        inputs: [x0, x1]
	                        n_dims: 2
	                        outputs: [x0, x1]
	                      mapping: [0, 1, 0, 1]
	                      name: camera_inmap
	                      outputs: [x0, x1, x2, x3]
	                    - forward:
	                      - coefficients: !core/ndarray-1.0.0
	                          data:
	                          - [0.000524628620052, 0.00862406220386, -0.00963133180052, -0.0592862455543,
	                            -22.1785717254, 116.164564229]
	                          - [1.0024268584, 0.842890159161, 4.48027128321, -2.23526738215,
	                            28.0339997063, 0.0]
	                          - [0.00338026771414, 0.170809193774, -1.71892389684, -76.7934348636,
	                            0.0, 0.0]
	                          - [4.73375404274, -5.69989296049, -30.2229928853, 0.0, 0.0,
	                            0.0]
	                          - [0.4446067912490323, 0.4464993575240323, 0.0, 0.0, 0.0, 0.0]
	                          - [-214.415176338, 0.0, 0.0, 0.0, 0.0, 0.0]
	                          datatype: float64
	                          shape: [6, 6]
	                        domain:
	                        - [-1, 1]
	                        - [-1, 1]
	                        inputs: [x, y]
	                        inverse:
	                          coefficients: !core/ndarray-1.0.0
	                            data:
	                            - [-0.000520510254971, -0.00821553874442, 0.0281081100722,
	                              0.0600654108493, 21.7259155709, -356.083886682]
	                            - [0.997770017256, -0.839094818721, -2.7976522169, 18.2527367537,
	                              -239.350410782, 0.0]
	                            - [-0.000595422813154, -0.102335580962, 1.43778822626, 18.1555083171,
	                              0.0, 0.0]
	                            - [-4.36354218371, 23.2780889726, -54.6704871334, 0.0, 0.0,
	                              0.0]
	                            - [-0.032848074225, -15.6089173945, 0.0, 0.0, 0.0, 0.0]
	                            - [190.392126999, 0.0, 0.0, 0.0, 0.0, 0.0]
	                            datatype: float64
	                            shape: [6, 6]
	                          domain:
	                          - [-1, 1]
	                          - [-1, 1]
	                          inputs: [x, y]
	                          name: camera_x_backward
	                          outputs: [z]
	                          window:
	                          - [-1, 1]
	                          - [-1, 1]
	                        name: camera_x_forward
	                        outputs: [z]
	                        window:
	                        - [-1, 1]
	                        - [-1, 1]
	                      - coefficients: !core/ndarray-1.0.0
	                          data:
	                          - [0.00033854996402, 0.996175355106, 1.07561682834, 2.15672500928,
	                            -39.2098949384, 869.774789321]
	                          - [-0.00716534244276, -0.00469626389676, 0.0427339055346, -0.632396487456,
	                            -162.590175204, 0.0]
	                          - [0.274993446561, 3.3549770508, -19.21481330999999, -135.764207661,
	                            0.0, 0.0]
	                          - [0.0362405190785, -0.8204122952, 5.36454197189, 0.0, 0.0,
	                            0.0]
	                          - [-4.77784963631, -200.692629414, 0.0, 0.0, 0.0, 0.0]
	                          - [-40.3321438497, 0.0, 0.0, 0.0, 0.0, 0.0]
	                          datatype: float64
	                          shape: [6, 6]
	                        domain:
	                        - [-1, 1]
	                        - [-1, 1]
	                        inputs: [x, y]
	                        inverse:
	                          coefficients: !core/ndarray-1.0.0
	                            data:
	                            - [-0.000343760967917, 1.00451632795, -1.08695181692, 0.12218634143,
	                              46.296044717, -1227.82171552]
	                            - [0.00745746965308, -0.00891675743894, 0.0731326401211, 0.700309414945,
	                              -208.144539049, 0.0]
	                            - [-0.273992965756, -2.72617759698, 36.7033235522, 81.1556806018,
	                              0.0, 0.0]
	                            - [-0.102756955086, 1.00219081686, -96.5076304556, 0.0, 0.0,
	                              0.0]
	                            - [7.81921079125, 171.952341364, 0.0, 0.0, 0.0, 0.0]
	                            - [45.7169720751, 0.0, 0.0, 0.0, 0.0, 0.0]
	                            datatype: float64
	                            shape: [6, 6]
	                          domain:
	                          - [-1, 1]
	                          - [-1, 1]
	                          inputs: [x, y]
	                          name: camera_y_backward
	                          outputs: [z]
	                          window:
	                          - [-1, 1]
	                          - [-1, 1]
	                        name: camera_y_forward
	                        outputs: [z]
	                        window:
	                        - [-1, 1]
	                        - [-1, 1]
	                      inputs: [x0, y0, x1, y1]
	                      outputs: [z0, z1]
	                    inputs: [x0, x1]
	                    outputs: [z0, z1]
	                  - inputs: [x0, x1]
	                    inverse:
	                      inputs: [x0, x1]
	                      mapping: [0, 1, 0, 1]
	                      outputs: [x0, x1, x2, x3]
	                    n_dims: 2
	                    name: camera_outmap
	                    outputs: [x0, x1]
	                  inputs: [x0, x1]
	                  outputs: [x0, x1]
	                - forward:
	                  - forward:
	                    - forward:
	                      - inputs: [x]
	                        name: camera_xincen_d2s
	                        offset: 2.38656283331e-06
	                        outputs: [y]
	                      - inputs: [x]
	                        name: camera_yincen_d2s
	                        offset: -0.000218347262797
	                        outputs: [y]
	                      inputs: [x0, x1]
	                      outputs: [y0, y1]
	                    - inputs: [x, y]
	                      matrix: !core/ndarray-1.0.0
	                        data:
	                        - [3.512593995567382, 0.0001878706545043787]
	                        - [-0.00017733714403258465, 3.346235822057333]
	                        datatype: float64
	                        shape: [2, 2]
	                      name: camera_affine_d2s
	                      outputs: [x, y]
	                      translation: !core/ndarray-1.0.0
	                        data: [0.0, 0.0]
	                        datatype: float64
	                        shape: [2]
	                    inputs: [x0, x1]
	                    outputs: [x, y]
	                  - forward:
	                    - inputs: [x]
	                      name: camera_xoutcen_d2s
	                      offset: 0.000143898033
	                      outputs: [y]
	                    - inputs: [x]
	                      name: camera_youtcen_d2s
	                      offset: 0.293606022006
	                      outputs: [y]
	                    inputs: [x0, x1]
	                    outputs: [y0, y1]
	                  inputs: [x0, x1]
	                  outputs: [y0, y1]
	                inputs: [x0, x1]
	                inverse:
	                  forward:
	                  - forward:
	                    - forward:
	                      - inputs: [x]
	                        name: camera_xoutcen_d2s
	                        offset: -0.000143898033
	                        outputs: [y]
	                      - inputs: [x]
	                        name: camera_youtcen_d2s
	                        offset: -0.293606022006
	                        outputs: [y]
	                      inputs: [x0, x1]
	                      outputs: [y0, y1]
	                    - forward:
	                      - inputs: [x, y]
	                        matrix: !core/ndarray-1.0.0
	                          data:
	                          - [0.2846898897831847, -1.4372880000595218e-05]
	                          - [1.3567023000759301e-05, 0.268727929448527]
	                          datatype: float64
	                          shape: [2, 2]
	                        outputs: [x, y]
	                        translation: !core/ndarray-1.0.0
	                          data: [-0.0, -0.0]
	                          datatype: float64
	                          shape: [2]
	                      - forward:
	                        - inputs: [x]
	                          name: camera_xincen_d2s
	                          offset: -2.38656283331e-06
	                          outputs: [y]
	                        - inputs: [x]
	                          name: camera_yincen_d2s
	                          offset: 0.000218347262797
	                          outputs: [y]
	                        inputs: [x0, x1]
	                        outputs: [y0, y1]
	                      inputs: [x, y]
	                      outputs: [y0, y1]
	                    inputs: [x0, x1]
	                    outputs: [y0, y1]
	                  - forward:
	                    - inputs: [x0, x1]
	                      mapping: [0, 1, 0, 1]
	                      outputs: [x0, x1, x2, x3]
	                    - forward:
	                      - forward:
	                        - coefficients: !core/ndarray-1.0.0
	                            data:
	                            - [-0.000520510254971, -0.00821553874442, 0.0281081100722,
	                              0.0600654108493, 21.7259155709, -356.083886682]
	                            - [0.997770017256, -0.839094818721, -2.7976522169, 18.2527367537,
	                              -239.350410782, 0.0]
	                            - [-0.000595422813154, -0.102335580962, 1.43778822626, 18.1555083171,
	                              0.0, 0.0]
	                            - [-4.36354218371, 23.2780889726, -54.6704871334, 0.0, 0.0,
	                              0.0]
	                            - [-0.032848074225, -15.6089173945, 0.0, 0.0, 0.0, 0.0]
	                            - [190.392126999, 0.0, 0.0, 0.0, 0.0, 0.0]
	                            datatype: float64
	                            shape: [6, 6]
	                          domain:
	                          - [-1, 1]
	                          - [-1, 1]
	                          inputs: [x, y]
	                          name: camera_x_backward
	                          outputs: [z]
	                          window:
	                          - [-1, 1]
	                          - [-1, 1]
	                        - coefficients: !core/ndarray-1.0.0
	                            data:
	                            - [-0.000343760967917, 1.00451632795, -1.08695181692, 0.12218634143,
	                              46.296044717, -1227.82171552]
	                            - [0.00745746965308, -0.00891675743894, 0.0731326401211, 0.700309414945,
	                              -208.144539049, 0.0]
	                            - [-0.273992965756, -2.72617759698, 36.7033235522, 81.1556806018,
	                              0.0, 0.0]
	                            - [-0.102756955086, 1.00219081686, -96.5076304556, 0.0, 0.0,
	                              0.0]
	                            - [7.81921079125, 171.952341364, 0.0, 0.0, 0.0, 0.0]
	                            - [45.7169720751, 0.0, 0.0, 0.0, 0.0, 0.0]
	                            datatype: float64
	                            shape: [6, 6]
	                          domain:
	                          - [-1, 1]
	                          - [-1, 1]
	                          inputs: [x, y]
	                          name: camera_y_backward
	                          outputs: [z]
	                          window:
	                          - [-1, 1]
	                          - [-1, 1]
	                        inputs: [x0, y0, x1, y1]
	                        outputs: [z0, z1]
	                      - inputs: [x0, x1]
	                        n_dims: 2
	                        outputs: [x0, x1]
	                      inputs: [x0, y0, x1, y1]
	                      outputs: [x0, x1]
	                    inputs: [x0, x1]
	                    outputs: [x0, x1]
	                  inputs: [x0, x1]
	                  outputs: [x0, x1]
	                outputs: [y0, y1]
	              inputs: [x, y]
	              outputs: [y0, y1]
	            - inputs: [x, y]
	              model_type: unitless2directional
	              name: unitless2directional_cosines
	              outputs: [x, y, z]
	            inputs: [x, y]
	            outputs: [x, y, z]
	          - angles: [0.03333072666861111, -0.27547251631138886, -0.14198882781777777,
	              24.29]
	            axes_order: xyzy
	            inputs: [x, y, z]
	            name: rotation
	            outputs: [x, y, z]
	          inputs: [x, y]
	          outputs: [x, y, z]
	  - frame:
	          axes_names: [alpha_in, beta_in]
	          axes_order: [0, 1]
	          axis_physical_types: ['custom:alpha_in', 'custom:beta_in']
	          name: gwa
	          unit: [rad, rad]
	        transform:
	          forward:
	          - forward:
	            - forward:
	              - forward:
	                - forward:
	                  - forward:
	                    - forward:
	                      - inputs: [x0, x1, x2]
	                        mapping: [0, 1, 0, 1]
	                        n_inputs: 3
	                        outputs: [x0, x1, x2, x3]
	                      - forward:
	                        - forward:
	                          - forward:
	                            - dimensions: 1
	                              inputs: [x]
	                              outputs: [y]
	                              value: 0.0
	                            - inputs: [x0]
	                              outputs: [x0]
	                            inputs: [x]
	                            outputs: [y]
	                          - forward:
	                            - dimensions: 1
	                              inputs: [x]
	                              outputs: [y]
	                              value: -1.0
	                            - inputs: [x0]
	                              outputs: [x0]
	                            inputs: [x]
	                            outputs: [y]
	                          inputs: [x0, x1]
	                          outputs: [y0, y1]
	                        - inputs: [x0, x1]
	                          n_dims: 2
	                          outputs: [x0, x1]
	                        inputs: [x00, x10, x01, x11]
	                        outputs: [y0, y1, x0, x1]
	                      inputs: [x0, x1, x2]
	                      outputs: [y0, y1, x0, x1]
	                    - forward:
	                      - forward:
	                        - inputs: [x0]
	                          outputs: [x0]
	                        - bounding_box: [-0.2869219718231398, -0.28489583056156154]
	                          bounds_error: false
	                          fill_value: .nan
	                          inputs: [x]
	                          lookup_table: !core/ndarray-1.0.0
	                            data: [-0.55, -0.548898898898899, -0.5477977977977978, -0.5466966966966967,
	                              -0.5455955955955957, -0.5444944944944945, -0.5433933933933934,
	                              -0.5422922922922924, -0.5411911911911912, -0.5400900900900901,
	                              -0.5389889889889891, -0.5378878878878879, -0.5367867867867868,
	                              -0.5356856856856858, -0.5345845845845846, -0.5334834834834835,
	                              -0.5323823823823824, -0.5312812812812813, -0.5301801801801802,
	                              -0.5290790790790791, -0.5279779779779781, -0.5268768768768769,
	                              -0.5257757757757758, -0.5246746746746748, -0.5235735735735736,
	                              -0.5224724724724725, -0.5213713713713715, -0.5202688397587957,
	                              -0.5191691691691692, -0.5180680680680682, -0.516966966966967,
	                              -0.5158658658658659, -0.5147647647647648, -0.5136636636636637,
	                              -0.5125625625625626, -0.5114614614614615, -0.5103603603603604,
	                              -0.5092592592592593, -0.5081581581581582, -0.5070570570570571,
	                              -0.5059559559559557, -0.5048548548548549, -0.5037537537537538,
	                              -0.5026526526526527, -0.5015515515515516, -0.5004504504504504,
	                              -0.4993493493493494, -0.4982482482482483, -0.4971471471471472,
	                              -0.4960460460460461, -0.494944944944945, -0.4938438438438439,
	                              -0.49274274274274277, -0.4916416416416417, -0.4905405405405406,
	                              -0.48943943943943946, -0.4883383383383384, -0.4872372372372373,
	                              -0.48613613613613615, -0.4850350350350351, -0.48393393393393397,
	                              -0.48283283283283285, -0.4817317317317318, -0.48063063063063066,
	                              -0.47952952952952954, -0.4784284284284285, -0.47732732732732736,
	                              -0.47622622622622623, -0.47512512512512517, -0.47402402402402405,
	                              -0.47292292292292293, -0.47182182182182186, -0.47072072072072074,
	                              -0.4696196196196197, -0.46851851851851856, -0.46741741741741744,
	                              -0.46631631631631637, -0.46521521521521525, -0.4641141141141142,
	                              -0.46301301301301306, -0.46191191191191194, -0.4608108108108109,
	                              -0.45970970970970976, -0.45860860860860864, -0.45750750750750757,
	                              -0.45640640640640645, -0.45530530530530533, -0.45420420420420426,
	                              -0.45310310310310314, -0.452002002002002, -0.45090090090090096,
	                              -0.44979979979979984, -0.4486986986986987, -0.44759759759759765,
	                              -0.44649649649649653, -0.4453953953953954, -0.44429429429429435,
	                              -0.4431931931931932, -0.4420920920920921, -0.44099099099099104,
	                              -0.4398898898898899, -0.4387887887887888, -0.43768768768768773,
	                              -0.4365865865865866, -0.4354854854854855, -0.4343843843843844,
	                              -0.4332832832832833, -0.4321821821821822, -0.4310810810810811,
	                              -0.42997997997998, -0.42887887887887893, -0.4277777777777778,
	                              -0.42667667667667675, -0.4255755755755756, -0.4244744744744745,
	                              -0.42337337337337344, -0.4222722722722723, -0.4211711711711712,
	                              -0.42007007007007013, -0.418968968968969, -0.4178678678678679,
	                              -0.4167667667667668, -0.4156656656656657, -0.4145645645645646,
	                              -0.4134634634634635, -0.4123623623623624, -0.4112612612612613,
	                              -0.4101601601601602, -0.4090590590590591, -0.407957957957958,
	                              -0.4068568568568569, -0.4057557557557558, -0.40465465465465467,
	                              -0.4035535535535536, -0.4024524524524525, -0.40135135135135136,
	                              -0.4002502502502503, -0.3991491491491492, -0.39804804804804805,
	                              -0.396946946946947, -0.39584584584584587, -0.39474474474474475,
	                              -0.3936436436436437, -0.3925425425425426, -0.39125833597269144,
	                              -0.39034033754637265, -0.3892392392392393, -0.38813813813813813,
	                              -0.38703703703703707, -0.385935935935936, -0.3848348348348348,
	                              -0.38373373373373376, -0.3826319173768954, -0.3815315315315316,
	                              -0.38043043043043046, -0.3793293293293294, -0.37822822822822827,
	                              -0.37712712712712715, -0.3760260260260261, -0.37492492492492496,
	                              -0.37382382382382384, -0.3727227227227228, -0.37162161882765393,
	                              -0.37052052052052054, -0.36941941941941947, -0.36831831831831835,
	                              -0.36721721721721723, -0.36611611611611616, -0.36501501501501504,
	                              -0.3639139139139139, -0.36281281281281286, -0.36171171171171174,
	                              -0.3606106106106106, -0.35950950950950955, -0.35840840840840843,
	                              -0.3573073073073073, -0.35620620620620624, -0.3551051051051051,
	                              -0.354004004004004, -0.35290290290290294, -0.35180180180180187,
	                              -0.3507007007007007, -0.34959959959959963, -0.34849849849849857,
	                              -0.3473973973973974, -0.3462962962962963, -0.34519519519519526,
	                              -0.3440940940940941, -0.342992992992993, -0.34189189189189195,
	                              -0.34079079079079083, -0.3396896896896897, -0.33858858858858865,
	                              -0.3374874874874875, -0.3363863863863864, -0.33528528528528534,
	                              -0.3341841841841842, -0.3330830830830831, -0.33198198198198203,
	                              -0.3308808808808809, -0.3297797797797798, -0.3286786786786787,
	                              -0.3275775775775776, -0.3264764764764765, -0.3253753753753754,
	                              -0.3242742742742743, -0.3231731731731732, -0.3220720720720721,
	                              -0.320970970970971, -0.31986986986986987, -0.3187687687687688,
	                              -0.3176676676676677, -0.31656656656656657, -0.3154654654654655,
	                              -0.3143643643643644, -0.31326326326326326, -0.3121621621621622,
	                              -0.31106106106106113, -0.30995995995995995, -0.3088588588588589,
	                              -0.3077577577577578, -0.30665665665665665, -0.3055555555555556,
	                              -0.3044544544544545, -0.3033533533533534, -0.3022522522522523,
	                              -0.3011511511511512, -0.3000500500500501, -0.29894894894894897,
	                              -0.2978478478478479, -0.2967467467467468, -0.29564564564564566,
	                              -0.2945445445445446, -0.2934434434434435, -0.29234234234234235,
	                              -0.2912412412412413, -0.29014014014014017, -0.28903903903903905,
	                              -0.287937937937938, -0.28683683683683686, -0.28573573573573574,
	                              -0.2846346346346347, -0.28353353353353355, -0.28243243243243243,
	                              -0.28133133133133137, -0.28023023023023025, -0.2791291291291291,
	                              -0.27802802802802806, -0.27692692692692694, -0.2758258258258259,
	                              -0.27472472472472476, -0.27362362362362364, -0.27252252252252257,
	                              -0.27142142142142145, -0.27032032032032033, -0.26921921921921926,
	                              -0.26811811811811814, -0.267017017017017, -0.26591591591591596,
	                              -0.26481481481481484, -0.2637137137137137, -0.26261261261261265,
	                              -0.26151151151151153, -0.2604104104104104, -0.25930930930930934,
	                              -0.2582082082082082, -0.25710710710710716, -0.25600600600600604,
	                              -0.2549049049049049, -0.25380380380380385, -0.25270270270270273,
	                              -0.2516016016016016, -0.25050050050050054, -0.24939939939939942,
	                              -0.2482982982982983, -0.24719719719719724, -0.24609609609609612,
	                              -0.244994994994995, -0.24389389389389393, -0.2427927927927928,
	                              -0.2416916916916917, -0.24059059059059063, -0.2394894894894895,
	                              -0.23838838838838838, -0.23728728728728732, -0.2361861861861862,
	                              -0.23508508508508513, -0.233983983983984, -0.2328828828828829,
	                              -0.23178178178178183, -0.2306806806806807, -0.22957957957957958,
	                              -0.22847847847847852, -0.2273773773773774, -0.22627627627627628,
	                              -0.2251751751751752, -0.2240740740740741, -0.22297297297297297,
	                              -0.2218718718718719, -0.22077077077077079, -0.21966966966966966,
	                              -0.2185685685685686, -0.21746746746746748, -0.2163663663663664,
	                              -0.2152652652652653, -0.21416416416416417, -0.2130630630630631,
	                              -0.211961961961962, -0.21086086086086087, -0.2097597597597598,
	                              -0.20865865865865868, -0.20755755755755756, -0.2064564564564565,
	                              -0.20535535535535537, -0.20425425425425425, -0.2031531531531532,
	                              -0.20205205205205207, -0.20095095095095095, -0.19984984984984988,
	                              -0.19874874874874876, -0.19764764764764764, -0.19654654654654657,
	                              -0.19544544544544545, -0.1943443443443444, -0.19324324324324327,
	                              -0.19214214214214215, -0.19104104104104108, -0.18993993993993996,
	                              -0.18883883883883884, -0.18773773773773778, -0.18663663663663665,
	                              -0.18553553553553553, -0.18443443443443447, -0.18333333333333335,
	                              -0.18223223223223223, -0.18113113113113116, -0.18003003003003004,
	                              -0.17892892892892892, -0.17782782782782786, -0.17672672672672673,
	                              -0.17562562562562567, -0.17452452452452455, -0.17342342342342343,
	                              -0.17232232232232236, -0.17122122122122124, -0.17012012012012012,
	                              -0.16901901901901906, -0.16791791791791794, -0.16681681681681682,
	                              -0.16571571571571575, -0.16461461461461463, -0.1635135135135135,
	                              -0.16241241241241244, -0.16131131131131132, -0.1602102102102102,
	                              -0.15910910910910914, -0.15800800800800802, -0.15690690690690695,
	                              -0.15580580580580583, -0.1547047047047047, -0.15360360360360364,
	                              -0.15250250250250252, -0.1514014014014014, -0.15030030030030034,
	                              -0.14919919919919922, -0.1480980980980981, -0.14699699699699703,
	                              -0.1458958958958959, -0.1447947947947948, -0.14369369369369372,
	                              -0.1425925925925926, -0.14149149149149148, -0.14039039039039042,
	                              -0.1392892892892893, -0.13818818818818818, -0.1370870870870871,
	                              -0.135985985985986, -0.13488488488488493, -0.1337837837837838,
	                              -0.13268268268268268, -0.1315815815815603, -0.1304804804804805,
	                              -0.12937937937937938, -0.1282782782782783, -0.1271771771771772,
	                              -0.12607607607607607, -0.124974974974975, -0.12387387387387389,
	                              -0.12277277277277276, -0.1216716716716717, -0.12057057057057058,
	                              -0.11946946946946946, -0.11836836836836839, -0.11726726726726727,
	                              -0.11616616616616621, -0.11506506506506509, -0.11396396396396397,
	                              -0.1128628628628629, -0.11176176176176178, -0.11066066066066066,
	                              -0.1095595595595596, -0.10845845845572999, -0.10735735735735735,
	                              -0.10625625625625629, -0.10515515515515517, -0.10405405405405405,
	                              -0.10295295295295298, -0.10185185185185186, -0.10075075075075074,
	                              -0.09964964964964967, -0.09854854854854855, -0.09744744744744743,
	                              -0.09634634634634637, -0.09524524454675332, -0.09414414414414418,
	                              -0.09304304304304306, -0.09194194194194194, -0.09084084084084088,
	                              -0.08973973973973975, -0.08863863863863863, -0.08753753753753757,
	                              -0.08643643643643645, -0.08533533533533533, -0.08423423423423426,
	                              -0.08313313313313314, -0.0820318532180977, -0.08093093093093096,
	                              -0.07982982982982983, -0.07872872872872871, -0.07762762762762765,
	                              -0.07652652652652653, -0.07542542542542546, -0.07432432432432434,
	                              -0.07322322322049474, -0.07212212212212216, -0.07102102102102104,
	                              -0.06991991991991992, -0.06881881881881885, -0.06771771771771773,
	                              -0.06657084024942911, -0.06551551551551554, -0.06441441441441442,
	                              -0.0633133133133133, -0.06221221221221224, -0.061111111111111116,
	                              -0.060010010010009995, -0.05890890890890893, -0.05780780780780781,
	                              -0.056706706706706744, -0.055605605605605624, -0.0545045045045045,
	                              -0.05340340340340344, -0.05230230230230232, -0.051201201201201196,
	                              -0.05010010010010013, -0.048998998998998955, -0.04789789789789789,
	                              -0.046796796796796825, -0.04569569569569576, -0.04459459459459458,
	                              -0.04349349349349352, -0.04239239239239245, -0.04129129129129128,
	                              -0.04019019019019021, -0.039089089089089146, -0.03798798798798797,
	                              -0.036886886886886905, -0.03578578578578584, -0.034684684684684663,
	                              -0.0335835835835836, -0.03248248248248253, -0.03138138138138136,
	                              -0.03028028028028029, -0.029179179179179227, -0.02807807807807805,
	                              -0.026976976976976985, -0.02587587587587592, -0.024774774774774744,
	                              -0.02367367367367368, -0.022572572572572613, -0.021471471471471437,
	                              -0.020370370370370372, -0.019269269269269307, -0.01816816816816813,
	                              -0.017067067067067065, -0.015965965965966, -0.014864864864864824,
	                              -0.013763763763763759, -0.012662662662662694, -0.011561561561561517,
	                              -0.010460460460460452, -0.009359359359359387, -0.008258258258258211,
	                              -0.007157157157157146, -0.0060560560560560805, -0.004954954954955015,
	                              -0.003853853853853839, -0.002752752752752774, -0.0016516516516517088,
	                              -0.0005505505505505326, 0.0005505505505505326, 0.0016516516516515978,
	                              0.002752752752752774, 0.003853853853853839, 0.004954954954954904,
	                              0.0060560560560560805, 0.007157157157157146, 0.008258258258258211,
	                              0.009359359359359387, 0.010460460460460452, 0.011561561561561517,
	                              0.012662662662662694, 0.013763763763763759, 0.014864864864864824,
	                              0.015965965965966, 0.017067067067067065, 0.01816816816816813,
	                              0.019269269269269307, 0.020370370370370372, 0.021471471471471437,
	                              0.022572572572572613, 0.02367367367367368, 0.024774774774774744,
	                              0.02587587587587592, 0.026976976976976985, 0.02807807807807805,
	                              0.029179179179179227, 0.03028028028028029, 0.03138138138138136,
	                              0.03248248248248253, 0.0335835835835836, 0.034684684684684663,
	                              0.03578578578578573, 0.036886886886886905, 0.03798798798798797,
	                              0.039089089089089035, 0.04019019019019021, 0.04129129129129128,
	                              0.04239239239239234, 0.04349349349349352, 0.04459459459459458,
	                              0.04569569569569565, 0.046796796796796825, 0.04789789789789789,
	                              0.048998998998998955, 0.05010010010010013, 0.051201201201201196,
	                              0.05230230230230226, 0.05340340340340344, 0.0545045045045045,
	                              0.05560560560560557, 0.056706706706706744, 0.05780780780780781,
	                              0.058908908908908875, 0.06001001001001005, 0.061111111111111116,
	                              0.06221221221221218, 0.06331331331331336, 0.06441441441441442,
	                              0.06551551551551549, 0.06657084024942916, 0.06771771771771773,
	                              0.0688188188188188, 0.06991991991991997, 0.07102102102102104,
	                              0.0721221221221221, 0.07322322322049479, 0.07432432432432434,
	                              0.07542542542542541, 0.07652652652652647, 0.07762762762762765,
	                              0.07872872872872871, 0.07982982982982978, 0.08093093093093096,
	                              0.0820318532180977, 0.08313313313313309, 0.08423423423423426,
	                              0.08533533533533533, 0.08643643643643639, 0.08753753753753757,
	                              0.08863863863863863, 0.0897397397397397, 0.09084084084084088,
	                              0.09194194194194194, 0.093043043043043, 0.09414414414414418,
	                              0.09524524454675332, 0.09634634634634631, 0.09744744744744749,
	                              0.09854854854854855, 0.09964964964964962, 0.1007507507507508,
	                              0.10185185185185186, 0.10295295295295293, 0.1040540540540541,
	                              0.10515515515515517, 0.10625625625625623, 0.10735735735735741,
	                              0.10845845845572999, 0.10955955955955954, 0.11066066066066071,
	                              0.11176176176176178, 0.11286286286286284, 0.11396396396396402,
	                              0.11506506506506509, 0.11616616616616615, 0.11726726726726722,
	                              0.11836836836836839, 0.11946946946946946, 0.12057057057057052,
	                              0.1216716716716717, 0.12277277277277276, 0.12387387387387383,
	                              0.124974974974975, 0.12607607607607607, 0.12717717717717714,
	                              0.1282782782782783, 0.12937937937937938, 0.13048048048048044,
	                              0.1315815815815603, 0.13268268268268268, 0.13378378378378375,
	                              0.13488488488488493, 0.135985985985986, 0.13708708708708706,
	                              0.13818818818818823, 0.1392892892892893, 0.14039039039039036,
	                              0.14149149149149154, 0.1425925925925926, 0.14369369369369367,
	                              0.14479479479479485, 0.1458958958958959, 0.14699699699699698,
	                              0.14809809809809815, 0.14919919919919922, 0.15030030030030028,
	                              0.15140140140140146, 0.15250250250250252, 0.1536036036036036,
	                              0.15470470470470477, 0.15580580580580583, 0.1569069069069069,
	                              0.15800800800800796, 0.15910910910910914, 0.1602102102102102,
	                              0.16131131131131127, 0.16241241241241244, 0.1635135135135135,
	                              0.16461461461461457, 0.16571571571571575, 0.16681681681681682,
	                              0.16791791791791788, 0.16901901901901906, 0.17012012012012012,
	                              0.1712212212212212, 0.17232232232232236, 0.17342342342342343,
	                              0.1745245245245245, 0.17562562562562567, 0.17672672672672673,
	                              0.1778278278278278, 0.17892892892892898, 0.18003003003003004,
	                              0.1811311311311311, 0.18223223223223228, 0.18333333333333335,
	                              0.1844344344344344, 0.1855355355355356, 0.18663663663663665,
	                              0.18773773773773772, 0.1888388388388389, 0.18993993993993996,
	                              0.19104104104104103, 0.1921421421421422, 0.19324324324324327,
	                              0.19434434434434433, 0.1954454454454454, 0.19654654654654657,
	                              0.19764764764764764, 0.1987487487487487, 0.19984984984984988,
	                              0.20095095095095095, 0.202052052052052, 0.2031531531531532,
	                              0.20425425425425425, 0.20535535535535532, 0.2064564564564565,
	                              0.20755755755755756, 0.20865865865865862, 0.2097597597597598,
	                              0.21086086086086087, 0.21196196196196193, 0.2130630630630631,
	                              0.21416416416416417, 0.21526526526526524, 0.2163663663663664,
	                              0.21746746746746748, 0.21856856856856854, 0.21966966966966972,
	                              0.22077077077077079, 0.22187187187187185, 0.22297297297297303,
	                              0.2240740740740741, 0.22517517517517516, 0.22627627627627633,
	                              0.2273773773773774, 0.22847847847847846, 0.22957957957957964,
	                              0.2306806806806807, 0.23178178178178177, 0.23288288288288295,
	                              0.233983983983984, 0.23508508508508508, 0.23618618618618614,
	                              0.23728728728728732, 0.23838838838838838, 0.23948948948948945,
	                              0.24059059059059063, 0.2416916916916917, 0.24279279279279276,
	                              0.24389389389389393, 0.244994994994995, 0.24609609609609606,
	                              0.24719719719719724, 0.2482982982982983, 0.24939939939939937,
	                              0.25050050050050054, 0.2516016016016016, 0.2527027027027027,
	                              0.25380380380380385, 0.2549049049049049, 0.256006006006006,
	                              0.25710710710710716, 0.2582082082082082, 0.2593093093093093,
	                              0.26041041041041046, 0.26151151151151153, 0.2626126126126126,
	                              0.26371371371371377, 0.26481481481481484, 0.2659159159159159,
	                              0.2670170170170171, 0.26811811811811814, 0.2692192192192192,
	                              0.2703203203203204, 0.27142142142142145, 0.2725225225225225,
	                              0.2736236236236237, 0.27472472472472476, 0.2758258258258258,
	                              0.2769269269269269, 0.27802802802802806, 0.2791291291291291,
	                              0.2802302302302302, 0.28133133133133137, 0.28243243243243243,
	                              0.2835335335335335, 0.2846346346346347, 0.28573573573573574,
	                              0.2868368368368368, 0.287937937937938, 0.28903903903903905,
	                              0.2901401401401401, 0.2912412412412413, 0.29234234234234235,
	                              0.2934434434434434, 0.2945445445445446, 0.29564564564564566,
	                              0.2967467467467467, 0.2978478478478479, 0.29894894894894897,
	                              0.30005005005005003, 0.3011511511511512, 0.3022522522522523,
	                              0.30335335335335334, 0.3044544544544545, 0.3055555555555556,
	                              0.30665665665665665, 0.3077577577577578, 0.3088588588588589,
	                              0.30995995995995995, 0.31106106106106113, 0.3121621621621622,
	                              0.31326326326326326, 0.31436436436436443, 0.3154654654654655,
	                              0.31656656656656657, 0.31766766766766763, 0.3187687687687688,
	                              0.31986986986986987, 0.32097097097097094, 0.3220720720720721,
	                              0.3231731731731732, 0.32427427427427424, 0.3253753753753754,
	                              0.3264764764764765, 0.32757757757757755, 0.3286786786786787,
	                              0.3297797797797798, 0.33088088088088086, 0.33198198198198203,
	                              0.3330830830830831, 0.33418418418418416, 0.33528528528528534,
	                              0.3363863863863864, 0.33748748748748747, 0.33858858858858865,
	                              0.3396896896896897, 0.3407907907907908, 0.34189189189189195,
	                              0.342992992992993, 0.3440940940940941, 0.34519519519519526,
	                              0.3462962962962963, 0.3473973973973974, 0.34849849849849857,
	                              0.34959959959959963, 0.3507007007007007, 0.35180180180180187,
	                              0.35290290290290294, 0.354004004004004, 0.3551051051051052,
	                              0.35620620620620624, 0.3573073073073073, 0.3584084084084084,
	                              0.35950950950950955, 0.3606106106106106, 0.3617117117117117,
	                              0.36281281281281286, 0.3639139139139139, 0.365015015015015,
	                              0.36611611611611616, 0.36721721721721723, 0.3683183183183183,
	                              0.36941941941941947, 0.37052052052052054, 0.3716216188276539,
	                              0.3727227227227228, 0.37382382382382384, 0.3749249249249249,
	                              0.3760260260260261, 0.37712712712712715, 0.3782282282282282,
	                              0.3793293293293294, 0.38043043043043046, 0.3815315315315315,
	                              0.3826319173768954, 0.38373373373373376, 0.3848348348348348,
	                              0.385935935935936, 0.38703703703703707, 0.38813813813813813,
	                              0.3892392392392393, 0.39034033754637265, 0.39125833597269144,
	                              0.3925425425425426, 0.3936436436436437, 0.39474474474474475,
	                              0.3958458458458459, 0.396946946946947, 0.39804804804804805,
	                              0.3991491491491491, 0.4002502502502503, 0.40135135135135136,
	                              0.4024524524524524, 0.4035535535535536, 0.40465465465465467,
	                              0.40575575575575573, 0.4068568568568569, 0.407957957957958,
	                              0.40905905905905904, 0.4101601601601602, 0.4112612612612613,
	                              0.41236236236236234, 0.4134634634634635, 0.4145645645645646,
	                              0.41566566566566565, 0.4167667667667668, 0.4178678678678679,
	                              0.41896896896896896, 0.42007007007007013, 0.4211711711711712,
	                              0.42227227227227226, 0.42337337337337344, 0.4244744744744745,
	                              0.42557557557557557, 0.42667667667667675, 0.4277777777777778,
	                              0.4288788788788789, 0.42997997997998005, 0.4310810810810811,
	                              0.4321821821821822, 0.43328328328328336, 0.4343843843843844,
	                              0.4354854854854855, 0.43658658658658656, 0.43768768768768773,
	                              0.4387887887887888, 0.43988988988988986, 0.44099099099099104,
	                              0.4420920920920921, 0.44319319319319317, 0.44429429429429435,
	                              0.4453953953953954, 0.4464964964964965, 0.44759759759759765,
	                              0.4486986986986987, 0.4497997997997998, 0.45090090090090085,
	                              0.45200200200200213, 0.4531031031031032, 0.45420420420420426,
	                              0.45530530530530533, 0.4564064064064064, 0.45750750750750746,
	                              0.4586086086086085, 0.4597097097097098, 0.4608108108108109,
	                              0.46191191191191194, 0.463013013013013, 0.4641141141141141,
	                              0.46521521521521514, 0.4663163163163164, 0.4674174174174175,
	                              0.46851851851851856, 0.4696196196196196, 0.4707207207207207,
	                              0.47182182182182175, 0.47292292292292304, 0.4740240240240241,
	                              0.47512512512512517, 0.47622622622622623, 0.4773273273273273,
	                              0.47842842842842837, 0.47952952952952965, 0.4806306306306307,
	                              0.4817317317317318, 0.48283283283283285, 0.4839339339339339,
	                              0.485035035035035, 0.48613613613613627, 0.48723723723723733,
	                              0.4883383383383384, 0.48943943943943946, 0.4905405405405405,
	                              0.4916416416416416, 0.4927427427427429, 0.49384384384384394,
	                              0.494944944944945, 0.4960460460460461, 0.49714714714714714,
	                              0.4982482482482482, 0.49934934934934927, 0.5004504504504506,
	                              0.5015515515515516, 0.5026526526526527, 0.5037537537537538,
	                              0.5048548548548548, 0.5059559559559559, 0.5070570570570572,
	                              0.5081581581581582, 0.5092592592592593, 0.5103603603603604,
	                              0.5114614614614614, 0.5125625625625625, 0.5136636636636638,
	                              0.5147647647647648, 0.5158658658658659, 0.516966966966967,
	                              0.518068068068068, 0.5191691691691691, 0.5202688397587958,
	                              0.5213713713713715, 0.5224724724724725, 0.5235735735735736,
	                              0.5246746746746747, 0.5257757757757757, 0.526876876876877,
	                              0.5279779779779781, 0.5290790790790791, 0.5301801801801802,
	                              0.5312812812812813, 0.5323823823823823, 0.5334834834834836,
	                              0.5345845845845847, 0.5356856856856858, 0.5367867867867868,
	                              0.5378878878878879, 0.538988988988989, 0.54009009009009,
	                              0.5411911911911913, 0.5422922922922924, 0.5433933933933934,
	                              0.5444944944944945, 0.5455955955955956, 0.5466966966966966,
	                              0.5477977977977979, 0.548898898898899, 0.55]
	                            datatype: float64
	                            shape: [1000]
	                          method: linear
	                          name: tabular
	                          outputs: [y]
	                          points:
	                          - !core/ndarray-1.0.0
	                            data: [-0.2869219718231398, -0.28691994450279124, -0.2869179171807339,
	                              -0.28691588985696775, -0.2869138625314931, -0.28691183520430974,
	                              -0.2869098078754176, -0.28690778054481697, -0.2869057532125076,
	                              -0.2869037258784896, -0.28690169854276315, -0.2868996712053281,
	                              -0.2868976438661845, -0.28689561652533246, -0.2868935891827719,
	                              -0.28689156183850284, -0.2868895344816114, -0.2868875071448397,
	                              -0.2868854797954453, -0.28688345244434266, -0.28688142509153175,
	                              -0.2868793977370123, -0.28687737038078476, -0.2868753430228488,
	                              -0.2868733156632046, -0.28687128830185216, -0.2868692609387915,
	                              -0.2868672335740226, -0.2868652062075456, -0.2868631788393604,
	                              -0.2868611514694669, -0.28685912409786557, -0.28685709672455606,
	                              -0.28685506934953825, -0.2868530419728127, -0.286851014594379,
	                              -0.28684898721423735, -0.2868469598323876, -0.28684493244882986,
	                              -0.28684290506356436, -0.2868408776765908, -0.2868388502879094,
	                              -0.28683682289752005, -0.28683479550542296, -0.28683276811161784,
	                              -0.28683074071610515, -0.28682871331888443, -0.2868266859199562,
	                              -0.28682465851932, -0.28682263111697615, -0.28682060371292456,
	                              -0.2868185763071653, -0.2868165488996985, -0.2868145214905239,
	                              -0.28681249407964177, -0.28681046666705196, -0.2868084392527546,
	                              -0.28680641183674976, -0.28680438441903733, -0.28680235699961737,
	                              -0.28680032957849005, -0.28679830215565516, -0.2867962747311128,
	                              -0.286794247304863, -0.28679221987690573, -0.2867901924472412,
	                              -0.28678816501586935, -0.2867861375827901, -0.2867841101480036,
	                              -0.2867820827115096, -0.2867800552733086, -0.2867780278334002,
	                              -0.2867760003917846, -0.2867739729484617, -0.28677194550343177,
	                              -0.28676991805669455, -0.2867678906082502, -0.2867658631580988,
	                              -0.28676383570624014, -0.28676180825267467, -0.28675978079740194,
	                              -0.2867577533404222, -0.2867557258817354, -0.28675369842134174,
	                              -0.28675167095924103, -0.28674964349543347, -0.2867476160299189,
	                              -0.2867455885626974, -0.28674356109376903, -0.28674153362313376,
	                              -0.2867395061507917, -0.28673747867674265, -0.286735451200987,
	                              -0.2867334237235246, -0.28673139624435545, -0.2867293687634795,
	                              -0.28672734128089683, -0.2867253137966074, -0.28672328631061145,
	                              -0.28672125882290855, -0.2867192313334994, -0.28671720384238364,
	                              -0.286715176349561, -0.28671314885503196, -0.2867111213587964,
	                              -0.2867090938608542, -0.2867070663612056, -0.2867050388598506,
	                              -0.28670301135678905, -0.28670098385202103, -0.28669895634554676,
	                              -0.2866969288373659, -0.2866949013274789, -0.28669287381588543,
	                              -0.28669084630258557, -0.2866888187875795, -0.28668679127086716,
	                              -0.2866847609584807, -0.28668273623232354, -0.2866807087104925,
	                              -0.2866786811869552, -0.2866766536617118, -0.28667462613476224,
	                              -0.2866725986061065, -0.2866705710757446, -0.28666854354367677,
	                              -0.2866665160099028, -0.28666448847442266, -0.2866624609372366,
	                              -0.28666043339834457, -0.2866584058577466, -0.28665637831544255,
	                              -0.28665435077143264, -0.2866523232257169, -0.2866502956782951,
	                              -0.28664826812916755, -0.28664624057833404, -0.2866442130257948,
	                              -0.2866421854715496, -0.28664015791559877, -0.28663813035794217,
	                              -0.2866361027985798, -0.2866340752375118, -0.286632047674738,
	                              -0.2866300201102585, -0.28662799254407345, -0.28662596497618265,
	                              -0.2866239374065863, -0.2866219098352844, -0.28661988226227675,
	                              -0.28661785468756384, -0.28661582711114525, -0.28661379953302113,
	                              -0.2866117719531915, -0.2866097443716566, -0.28660771678841607,
	                              -0.2866056892034701, -0.2866036616168189, -0.2866016340284623,
	                              -0.28659960643840016, -0.286597578846633, -0.2865955512531603,
	                              -0.28659352365798246, -0.28659149606109924, -0.2865894684625107,
	                              -0.28658744086221716, -0.2865854132602183, -0.28658338565651426,
	                              -0.2865813580511049, -0.28657933044399064, -0.2865773028351712,
	                              -0.2865752752246467, -0.2865732476124171, -0.2865712199984825,
	                              -0.28656919238284273, -0.28656716476549804, -0.28656513713553433,
	                              -0.2865631095256937, -0.2865610819032342, -0.28655905427906975,
	                              -0.2865570266532003, -0.28655499902562603, -0.28655297139634683,
	                              -0.2865509437653203, -0.28654891613267414, -0.2865468884982804,
	                              -0.28654486086218217, -0.28654283322437923, -0.2865408055848714,
	                              -0.2865387779436588, -0.28653675030074166, -0.2865347226561198,
	                              -0.2865326950097932, -0.28653066736176214, -0.2865286397120264,
	                              -0.2865266120605863, -0.2865245844074413, -0.2865225567525921,
	                              -0.28652052909603815, -0.28651850143777974, -0.2865164737778169,
	                              -0.28651444611614973, -0.28651241845277803, -0.28651039078770185,
	                              -0.28650836312092126, -0.2865063354524366, -0.2865043077822474,
	                              -0.28650156485461653, -0.28650025243675614, -0.28649822476145403,
	                              -0.28649619708444773, -0.28649416940573713, -0.28649214172532245,
	                              -0.2864901140432036, -0.2864880863593804, -0.28648605867385313,
	                              -0.28648403098662195, -0.28648200329768636, -0.28647997560704686,
	                              -0.28647794791470316, -0.2864759202206556, -0.286473892524904,
	                              -0.28647186482744824, -0.2864698371282887, -0.2864678094274252,
	                              -0.2864657817248576, -0.28646375402058627, -0.2864617263146111,
	                              -0.286459698606932, -0.28645767089754903, -0.28645564318646227,
	                              -0.2864536154736716, -0.2864515877591773, -0.2864495600429792,
	                              -0.28644753232507736, -0.2864455046054717, -0.28644347688416244,
	                              -0.28644144916114955, -0.2864394214364331, -0.28643739371001287,
	                              -0.28643536598188907, -0.28643333825206174, -0.2864313105205307,
	                              -0.28642928278729635, -0.28642725505235833, -0.28642522731571685,
	                              -0.28642319957737195, -0.2864211718373234, -0.28641914409557173,
	                              -0.2864171163521165, -0.28641508860695786, -0.2864130608600959,
	                              -0.28641103311153043, -0.28640900536126196, -0.2864069776092899,
	                              -0.2864049498556147, -0.28640292210023627, -0.28640089434315436,
	                              -0.2863988665843695, -0.28639683882388134, -0.28639481106169007,
	                              -0.2863927832977956, -0.28639075553219795, -0.2863887277648973,
	                              -0.28638669999589345, -0.2863846722251866, -0.2863826444527767,
	                              -0.286380616678664, -0.286378588902848, -0.2863765611253291,
	                              -0.2863745333461073, -0.2863725055651825, -0.2863704777825548,
	                              -0.28636844999822425, -0.28636642221219083, -0.28636439442445455,
	                              -0.2863623666350155, -0.2863603388438735, -0.2863583110510289,
	                              -0.28635628325648127, -0.2863542554602312, -0.2863522276622782,
	                              -0.2863501998626227, -0.2863481720612643, -0.28634614425820343,
	                              -0.2863441164534398, -0.28634208864697375, -0.28634006083880487,
	                              -0.28633803302893357, -0.2863360052173597, -0.28633397740408323,
	                              -0.2863319495891044, -0.2863299217724231, -0.28632789395403907,
	                              -0.28632586613395283, -0.2863238383121641, -0.2863218104886731,
	                              -0.2863197826634796, -0.28631775483658367, -0.28631572700798563,
	                              -0.28631369917768523, -0.2863116713456824, -0.2863096435119775,
	                              -0.28630761567657026, -0.2863055878394607, -0.28630356000064905,
	                              -0.28630153216013526, -0.2862995043179192, -0.2862974764740011,
	                              -0.2862954486283808, -0.28629342078105857, -0.2862913929320341,
	                              -0.28628936507039376, -0.2862873372288791, -0.28628530937474866,
	                              -0.2862832815189162, -0.2862812536613818, -0.2862792258021454,
	                              -0.2862771979412071, -0.28627517007856695, -0.2862731422142248,
	                              -0.2862711143481808, -0.2862690864804352, -0.28626705861098767,
	                              -0.2862650307398382, -0.286263002866987, -0.28626097499243414,
	                              -0.2862589471161796, -0.2862569192382233, -0.28625489135856536,
	                              -0.28625286347720563, -0.2862508355941444, -0.2862488077093815,
	                              -0.286246779822917, -0.2862447519238369, -0.2862427240448833,
	                              -0.28624069615331416, -0.28623866826004346, -0.28623664036507146,
	                              -0.28623461246839776, -0.28623258457002265, -0.2862305566699462,
	                              -0.2862285287681684, -0.28622650086468904, -0.28622447295950854,
	                              -0.28622244505262656, -0.2862204171440431, -0.2862183892337586,
	                              -0.28621636132177275, -0.2862143334080858, -0.28621230549269744,
	                              -0.28621027757560796, -0.2862082496568172, -0.28620622173632526,
	                              -0.28620419381413226, -0.2862021658902383, -0.28620013796464305,
	                              -0.2861981100373467, -0.2861960821083493, -0.2861940541776509,
	                              -0.28619202624525153, -0.2861899983111511, -0.28618797037534965,
	                              -0.28618594243784745, -0.28618391449864417, -0.28618188655774,
	                              -0.286179858615135, -0.2861778306708291, -0.28617580272482235,
	                              -0.2861737747771148, -0.2861717468277065, -0.2861697188765974,
	                              -0.28616769092378747, -0.2861656629692768, -0.2861636350130656,
	                              -0.2861616070551109, -0.2861595790955409, -0.2861575511342275,
	                              -0.2861555231712136, -0.28615349520649896, -0.2861514672400839,
	                              -0.2861494392719682, -0.286147411302152, -0.2861453833306353,
	                              -0.2861433553574181, -0.2861413273825004, -0.28613929940588223,
	                              -0.2861372714275636, -0.28613524344754454, -0.28613321546582515,
	                              -0.2861311874824055, -0.28612915949728546, -0.28612713151046487,
	                              -0.2861251035219442, -0.28612307553172317, -0.2861210475398021,
	                              -0.2861190195461806, -0.28611699155085885, -0.2861149635538368,
	                              -0.28611293555511486, -0.28611090755469265, -0.2861088795525703,
	                              -0.28610685154874793, -0.28610482354322536, -0.2861027955360027,
	                              -0.28610076752708014, -0.2860987395164574, -0.2860967115041348,
	                              -0.2860946834901121, -0.2860926554743895, -0.28609062745696706,
	                              -0.28608859943784454, -0.28608656862305454, -0.28608454339450007,
	                              -0.286082515370278, -0.2860804873443562, -0.28607845931673453,
	                              -0.286076431287413, -0.28607440325639194, -0.286072375223671,
	                              -0.28607034718925034, -0.28606831915312997, -0.28606629111530985,
	                              -0.28606426307579014, -0.2860622350345709, -0.28606020699165197,
	                              -0.2860581789470334, -0.2860561509007153, -0.2860541228526977,
	                              -0.28605209480298055, -0.28605006675156386, -0.28604803869844775,
	                              -0.2860460106436321, -0.28604398258711705, -0.28604195452890263,
	                              -0.2860399264689888, -0.2860378984073757, -0.286035870344063,
	                              -0.2860338422790512, -0.28603181421234, -0.28602978614392954,
	                              -0.2860277580738198, -0.28602573000201076, -0.28602370192850246,
	                              -0.2860216738532951, -0.28601964577638844, -0.2860176176977827,
	                              -0.28601558961747775, -0.2860135615354738, -0.2860115334517707,
	                              -0.28600950536636854, -0.2860074772792674, -0.28600544919042453,
	                              -0.2860034210999679, -0.28600139300776967, -0.28599936491387246,
	                              -0.2859973368182763, -0.28599530872098133, -0.28599328062198737,
	                              -0.28599125252129454, -0.2859892244189028, -0.2859871963148123,
	                              -0.285985168209023, -0.2859831401015349, -0.28598111199234794,
	                              -0.28597908388146237, -0.2859770557688781, -0.28597502765459504,
	                              -0.2859729995386133, -0.28597097142093286, -0.2859689433015539,
	                              -0.28596691518047623, -0.2859648870577, -0.28596285893322504,
	                              -0.28596083080705176, -0.2859588026791798, -0.2859567745496094,
	                              -0.2859547464183404, -0.2859527182853731, -0.28595069015070723,
	                              -0.2859486620143431, -0.28594663387628033, -0.28594460573651936,
	                              -0.2859425775950601, -0.2859405494519024, -0.2859385213070462,
	                              -0.2859364931604919, -0.28593446501223946, -0.28593243686228864,
	                              -0.2859304087106395, -0.28592838055729225, -0.28592635240224673,
	                              -0.28592432424550307, -0.2859222960870613, -0.28592026792692143,
	                              -0.28591823976508335, -0.2859162116015474, -0.28591418343631325,
	                              -0.2859121552693811, -0.28591012710075087, -0.28590809893042274,
	                              -0.2859060707583966, -0.28590404258467256, -0.28590201440925045,
	                              -0.28589998623213064, -0.2858979580533127, -0.2858959298727972,
	                              -0.2858939016905837, -0.28589115825093503, -0.28588984532106315,
	                              -0.28588781713375633, -0.2858857889447517, -0.28588376075404937,
	                              -0.28588173256164906, -0.2858797043675514, -0.28587767617175597,
	                              -0.2858756479742629, -0.2858736197750721, -0.28587159157418374,
	                              -0.2858695633715979, -0.28586753516731445, -0.28586550696133334,
	                              -0.285863478753655, -0.2858614505442789, -0.2858594223332055,
	                              -0.28585739412043454, -0.2858553659059662, -0.2858533376898003,
	                              -0.28585130947193715, -0.2858492812523766, -0.28584725303111874,
	                              -0.2858452248081636, -0.2858431965835111, -0.28584116835716133,
	                              -0.28583914012911427, -0.28583711189937, -0.2858350836679285,
	                              -0.2858330554347898, -0.2858310271999539, -0.2858289989634209,
	                              -0.28582697072519087, -0.28582494248526363, -0.28582291424363915,
	                              -0.28582088600031774, -0.28581885775529947, -0.28581682950858406,
	                              -0.28581480126017156, -0.28581277301006214, -0.2858107447582557,
	                              -0.28580871650475254, -0.2858066882495523, -0.2858046599926552,
	                              -0.2858026317340612, -0.28580060347377034, -0.28579857521178254,
	                              -0.2857965469480981, -0.2857945186827169, -0.28579249041563887,
	                              -0.2857904621468641, -0.28578843387639263, -0.2857864056042244,
	                              -0.2857843773303596, -0.28578234905479805, -0.28578032077754006,
	                              -0.28577829249858533, -0.28577626421793395, -0.28577423593558604,
	                              -0.28577220765154143, -0.2857701793658006, -0.28576815107836306,
	                              -0.2857661227892292, -0.28576409449839874, -0.28576206620587197,
	                              -0.28576003791164856, -0.28575800961572906, -0.28575598131811286,
	                              -0.2857539530188005, -0.28575192471779187, -0.28574989641508675,
	                              -0.28574786811068553, -0.2857458398045878, -0.285743811496794,
	                              -0.28574178318730403, -0.28573975487611786, -0.2857377265632353,
	                              -0.2857356982486567, -0.2857336699323819, -0.28573164161441117,
	                              -0.2857296132947443, -0.2857275849733813, -0.2857255566503222,
	                              -0.28572352832556713, -0.285721499999116, -0.28571947167096906,
	                              -0.285717443341126, -0.28571541500958697, -0.2857133866763521,
	                              -0.2857113583414212, -0.2857093300047947, -0.28570730166647207,
	                              -0.28570527332645373, -0.28570324498473954, -0.2857012166413296,
	                              -0.2856991882962238, -0.28569715994942213, -0.2856951316009251,
	                              -0.2856931032507322, -0.2856910748988435, -0.2856890465452594,
	                              -0.2856870181899795, -0.28568498983300383, -0.2856829614743327,
	                              -0.28568093311396614, -0.285678904751904, -0.2856768763881461,
	                              -0.2856748480226928, -0.2856728196555441, -0.28567079128669987,
	                              -0.2856687629161601, -0.28566673454392505, -0.2856647061699944,
	                              -0.28566267779436855, -0.28566064941704733, -0.2856586210380308,
	                              -0.2856565926573187, -0.2856545642749116, -0.28565253589080913,
	                              -0.28565050750501136, -0.2856484791175184, -0.2856464479343625,
	                              -0.28564442233744686, -0.2856423939448683, -0.28564036555059463,
	                              -0.28563833715462583, -0.285636308756962, -0.2856342803576031,
	                              -0.28563225195654907, -0.28563022355379986, -0.2856281951493559,
	                              -0.2856261667432168, -0.2856241383353827, -0.2856221099258538,
	                              -0.28562008151462986, -0.2856180531017109, -0.2856160246870972,
	                              -0.28561399627078876, -0.28561196785278525, -0.285609939433087,
	                              -0.285607911011694, -0.2856058825886062, -0.2856038541638236,
	                              -0.28560182573734627, -0.2855997973091743, -0.2855977688793075,
	                              -0.2855957404477461, -0.2855937120144901, -0.28559168357953946,
	                              -0.28558965514289425, -0.28558762670455445, -0.28558559826451996,
	                              -0.2855835698227911, -0.28558154137936753, -0.28557951293424966,
	                              -0.2855774844874372, -0.2855754560389304, -0.2855734275887289,
	                              -0.2855713991368332, -0.2855693706832431, -0.28556734222795865,
	                              -0.28556531377097966, -0.2855632853123066, -0.28556125685193917,
	                              -0.28555922838987746, -0.2855571999261215, -0.28555517146067116,
	                              -0.2855531429935267, -0.2855511145246881, -0.2855490860541553,
	                              -0.2855470575819282, -0.2855450291080072, -0.2855430006323919,
	                              -0.2855409721550827, -0.2855389436760793, -0.28553691519538177,
	                              -0.2855348867129903, -0.285532858228905, -0.28553082974312555,
	                              -0.28552880125565194, -0.2855267727664849, -0.2855247442756236,
	                              -0.28552271578306854, -0.2855206872888195, -0.28551865599890897,
	                              -0.28551663029524005, -0.28551460179590943, -0.2855125732948852,
	                              -0.28551054479216714, -0.2855085162877555, -0.28550648778165,
	                              -0.2855044592738507, -0.28550243076435794, -0.28550040225317147,
	                              -0.2854983737402913, -0.2854963452257175, -0.2854943167094501,
	                              -0.28549228819148914, -0.28549025967183467, -0.28548823115048677,
	                              -0.28548620262744506, -0.28548417410271015, -0.2854821455762817,
	                              -0.2854801170481598, -0.2854780885183445, -0.2854760599868358,
	                              -0.2854740314536336, -0.2854720029187383, -0.28546997438214944,
	                              -0.2854679458438674, -0.2854659173038921, -0.28546388876222334,
	                              -0.2854618602188615, -0.2854598316738064, -0.28545780312705804,
	                              -0.2854557745786166, -0.28545374602848195, -0.28545171747665404,
	                              -0.28544968892313316, -0.28544766036791913, -0.28544563181101207,
	                              -0.2854436032524119, -0.2854415746921188, -0.2854395461301325,
	                              -0.28543751756645336, -0.2854354890010813, -0.28543346043401613,
	                              -0.2854314318652582, -0.2854294032948073, -0.28542737472266355,
	                              -0.2854253461488269, -0.28542331757329736, -0.2854212889960752,
	                              -0.2854192604171601, -0.2854172318365522, -0.28541520325425157,
	                              -0.28541317467025823, -0.28541114608457224, -0.2854091174971935,
	                              -0.28540708890812216, -0.28540506031735813, -0.2854030317249014,
	                              -0.2854010031307522, -0.28539897453491025, -0.28539694593737597,
	                              -0.28539491733814903, -0.28539288873722957, -0.2853908601346177,
	                              -0.28538883153031325, -0.2853868029243164, -0.2853847743166271,
	                              -0.28538274570724553, -0.28538071709617135, -0.285378688483405,
	                              -0.2853766598689463, -0.28537463125279505, -0.28537260263495173,
	                              -0.28537057401541605, -0.2853685453941882, -0.28536651677126806,
	                              -0.28536448814665566, -0.2853624595203511, -0.2853604308923542,
	                              -0.28535840226266546, -0.2853563736312845, -0.28535434499821144,
	                              -0.2853523163634463, -0.28535028772698906, -0.2853482590888398,
	                              -0.2853462304489985, -0.2853442018074651, -0.28534217316423977,
	                              -0.2853401445193226, -0.28533811587271357, -0.28533608722441256,
	                              -0.2853340585744195, -0.28533202992273476, -0.2853300012693581,
	                              -0.2853279726142896, -0.2853259439575294, -0.2853239152990773,
	                              -0.2853218866389335, -0.2853198579770979, -0.2853178293135707,
	                              -0.28531580064835166, -0.28531377198144103, -0.2853117433128387,
	                              -0.28530971464254484, -0.2853076859705593, -0.28530565729688223,
	                              -0.28530362862151354, -0.28530159994445337, -0.28529957126570143,
	                              -0.2852975425852583, -0.2852955139031236, -0.2852934852192975,
	                              -0.28529145653377985, -0.28528942784657085, -0.2852873991576703,
	                              -0.2852853704670787, -0.2852833417747954, -0.285281313080821,
	                              -0.2852792843851551, -0.28527725568779816, -0.28527522698874974,
	                              -0.2852731982880103, -0.2852711695855795, -0.28526914088145755,
	                              -0.2852671121756444, -0.28526508346814006, -0.2852630547589446,
	                              -0.2852610260480581, -0.2852589973354804, -0.28525696862121164,
	                              -0.28525493990525186, -0.2852529111876011, -0.2852508824682593,
	                              -0.2852488537472265, -0.28524682502450277, -0.2852447963000881,
	                              -0.2852427675739826, -0.28524073884618606, -0.28523871011669877,
	                              -0.2852366813855205, -0.28523465265265147, -0.2852326239180916,
	                              -0.2852305951818409, -0.2852285664438996, -0.28522653770426726,
	                              -0.28522450896294455, -0.2852224802199309, -0.28521973621948915,
	                              -0.2852184227288317, -0.285216393980746, -0.28521436523096977,
	                              -0.2852123364795032, -0.28521030772634565, -0.2852082789714979,
	                              -0.28520625021495954, -0.2852042214567305, -0.28520219269681113,
	                              -0.28520016393520115, -0.2851981351719008, -0.28519610640690995,
	                              -0.28519407764022897, -0.2851920488718574, -0.2851900201017955,
	                              -0.2851879913300433, -0.28518596255660067, -0.28518393378146784,
	                              -0.28518190500464474, -0.28517987622613133, -0.2851778474459278,
	                              -0.285175818664034, -0.28517378988045006, -0.2851717610951759,
	                              -0.2851697323082115, -0.28516770351955717, -0.28516567472921267,
	                              -0.2851636459371781, -0.28516161714345334, -0.2851595883480388,
	                              -0.2851575595509341, -0.28515553075213945, -0.2851535019516547,
	                              -0.28515147314948014, -0.28514944434561557, -0.2851474155400612,
	                              -0.28514538673281675, -0.2851433579238826, -0.2851413291132586,
	                              -0.2851393003009447, -0.2851372714868985, -0.2851352426712477,
	                              -0.28513321385386464, -0.2851311850347917, -0.2851291562140291,
	                              -0.28512712739157686, -0.28512509856743484, -0.28512306974160323,
	                              -0.2851210409140821, -0.28511901208487117, -0.2851169832539708,
	                              -0.28511495442138074, -0.2851129255871013, -0.2851108967511322,
	                              -0.2851088679134736, -0.28510683907412565, -0.28510481023308826,
	                              -0.2851027813903613, -0.28510075254594497, -0.2850987236998394,
	                              -0.2850966948520443, -0.28509466600255984, -0.28509263715138616,
	                              -0.2850906082985231, -0.2850885794439708, -0.2850865505877293,
	                              -0.2850845217297984, -0.28508249287017845, -0.2850804640088692,
	                              -0.2850784351458709, -0.28507640628118336, -0.2850743774148067,
	                              -0.28507234854674096, -0.285070319676986, -0.28506829080554213,
	                              -0.2850662619324091, -0.2850642302636195, -0.28506220418107614,
	                              -0.2850601753028762, -0.2850581464229873, -0.28505611754140947,
	                              -0.28505408865814263, -0.28505205977318704, -0.28505003088654246,
	                              -0.28504800199820907, -0.28504597310818686, -0.28504394421647583,
	                              -0.28504191532307616, -0.28503988642798755, -0.28503785752029637,
	                              -0.28503582863274424, -0.2850337997325896, -0.28503177083074605,
	                              -0.285029741927214, -0.28502771302199337, -0.28502568411508405,
	                              -0.2850236552064862, -0.28502162629619976, -0.2850195973842247,
	                              -0.2850175684705612, -0.2850155395552091, -0.28501351063816854,
	                              -0.28501148171943963, -0.2850094527990222, -0.2850074238769164,
	                              -0.28500539495312216, -0.28500336602763954, -0.2850013371004684,
	                              -0.2849993081716093, -0.28499727924106166, -0.2849952503088258,
	                              -0.28499322137490163, -0.28499119243928916, -0.28498916350198855,
	                              -0.2849871345629997, -0.2849851056223228, -0.2849830766799575,
	                              -0.2849810477359042, -0.28497901879016274, -0.2849769898427331,
	                              -0.2849749608936157, -0.28497293194281, -0.28497090299031624,
	                              -0.2849688740361347, -0.284966845080265, -0.28496481612270724,
	                              -0.2849627871634616, -0.28496075820252825, -0.2849587292399068,
	                              -0.2849567002755974, -0.28495467130960034, -0.2849526423419154,
	                              -0.2849506133725425, -0.2849485844014819, -0.28494655542873365,
	                              -0.28494452645429746, -0.2849424974781735, -0.284940468500362,
	                              -0.2849384395208628, -0.2849364105396759, -0.28493438155680134,
	                              -0.28493235257223926, -0.28493032358598946, -0.2849282945980521,
	                              -0.2849262656084272, -0.2849242366171148, -0.28492220762411474,
	                              -0.28492017862942737, -0.2849181496330525, -0.2849161206349901,
	                              -0.28491409163524034, -0.28491206263380303, -0.2849100336306785,
	                              -0.28490800462586646, -0.2849059756193673, -0.2849039466111807,
	                              -0.28490191760130673, -0.2848998885897456, -0.2848978595764972,
	                              -0.28489583056156154]
	                            datatype: float64
	                            shape: [1000]
	                        inputs: [x0, x]
	                        outputs: [x0, y]
	                      - inputs: [x0, x1]
	                        n_dims: 2
	                        outputs: [x0, x1]
	                      inputs: [x00, x0, x01, x11]
	                      outputs: [x00, y0, x01, x11]
	                    inputs: [x0, x1, x2]
	                    outputs: [x00, y0, x01, x11]
	                  - inputs: [x0, x1, x2, x3]
	                    mapping: [0, 1, 0, 1, 2, 3]
	                    outputs: [x0, x1, x2, x3, x4, x5]
	                  inputs: [x0, x1, x2]
	                  outputs: [x0, x1, x2, x3, x4, x5]
	                - forward:
	                  - forward:
	                    - inputs: [x0, x1]
	                      n_dims: 2
	                      outputs: [x0, x1]
	                    - forward:
	                      - forward:
	                        - forward:
	                          - forward:
	                            - factor: 8.135000098263845e-05
	                              inputs: [x]
	                              outputs: [y]
	                            - factor: 0.001271169981919229
	                              inputs: [x]
	                              outputs: [y]
	                            inputs: [x0, x1]
	                            outputs: [y0, y1]
	                          - forward:
	                            - inputs: [x]
	                              offset: 0.02697242796421051
	                              outputs: [y]
	                            - inputs: [x]
	                              offset: -0.0027167024090886116
	                              outputs: [y]
	                            inputs: [x0, x1]
	                            outputs: [y0, y1]
	                          inputs: [x0, x1]
	                          outputs: [y0, y1]
	                        - forward:
	                          - angle: 0.0
	                            inputs: [x, y]
	                            name: msa_slit_rot
	                            outputs: [x, y]
	                          - forward:
	                            - inputs: [x]
	                              name: msa_slit_x
	                              offset: 0.0
	                              outputs: [y]
	                            - inputs: [x]
	                              name: msa_slit_y
	                              offset: 0.0
	                              outputs: [y]
	                            inputs: [x0, x1]
	                            outputs: [y0, y1]
	                          inputs: [x, y]
	                          outputs: [y0, y1]
	                        inputs: [x0, x1]
	                        outputs: [y0, y1]
	                      - forward:
	                        - forward:
	                          - forward:
	                            - forward:
	                              - forward:
	                                - inputs: [x]
	                                  name: collimator_xoutcen_d2s
	                                  offset: -5.526841e-06
	                                  outputs: [y]
	                                - inputs: [x]
	                                  name: collimator_youtcen_d2s
	                                  offset: 0.000346042594
	                                  outputs: [y]
	                                inputs: [x0, x1]
	                                outputs: [y0, y1]
	                              - forward:
	                                - inputs: [x, y]
	                                  matrix: !core/ndarray-1.0.0
	                                    data:
	                                    - [1.5738000900266444, -0.0003450858994488455]
	                                    - [0.0003613242773258282, 1.6478568990863562]
	                                    datatype: float64
	                                    shape: [2, 2]
	                                  outputs: [x, y]
	                                  translation: !core/ndarray-1.0.0
	                                    data: [-0.0, -0.0]
	                                    datatype: float64
	                                    shape: [2]
	                                - forward:
	                                  - inputs: [x]
	                                    name: collimator_xincen_d2s
	                                    offset: -0.000143900694035
	                                    outputs: [y]
	                                  - inputs: [x]
	                                    name: collimator_yincen_d2s
	                                    offset: -0.293605933112
	                                    outputs: [y]
	                                  inputs: [x0, x1]
	                                  outputs: [y0, y1]
	                                inputs: [x, y]
	                                outputs: [y0, y1]
	                              inputs: [x0, x1]
	                              outputs: [y0, y1]
	                            - forward:
	                              - inputs: [x0, x1]
	                                mapping: [0, 1, 0, 1]
	                                outputs: [x0, x1, x2, x3]
	                              - forward:
	                                - forward:
	                                  - coefficients: !core/ndarray-1.0.0
	                                      data:
	                                      - [0.00315706857764, 0.0420481492132, 0.146561534708,
	                                        0.221234162225, -0.0638619162952, -0.331781237202]
	                                      - [0.97396666617, -0.0712861999102, -0.269895805765,
	                                        -1.47821209943, -1.39521612319, 0.0]
	                                      - [-0.118219958126, -1.31400145373, -4.65546710314,
	                                        -5.31391588021, 0.0, 0.0]
	                                      - [-0.239124508069, -3.50159180727, -5.630240651,
	                                        0.0, 0.0, 0.0]
	                                      - [-0.721331930443, -2.52317345608, 0.0, 0.0, 0.0,
	                                        0.0]
	                                      - [-2.3223320496, 0.0, 0.0, 0.0, 0.0, 0.0]
	                                      datatype: float64
	                                      shape: [6, 6]
	                                    domain:
	                                    - [-1, 1]
	                                    - [-1, 1]
	                                    inputs: [x, y]
	                                    name: collimator_x_backward
	                                    outputs: [z]
	                                    window:
	                                    - [-1, 1]
	                                    - [-1, 1]
	                                  - coefficients: !core/ndarray-1.0.0
	                                      data:
	                                      - [-0.0027844382203, 1.15424678352, 1.57319737586,
	                                        5.65896061653, 9.03612184177, 5.89461390043]
	                                      - [-0.0251292730268, -0.24879556703, -1.30121421745,
	                                        -2.9831373654, -2.54562283395, 0.0]
	                                      - [0.0636250257988, 0.751936718567, 1.24472156622,
	                                        1.15635544547, 0.0, 0.0]
	                                      - [0.071044569667, 0.193723423502, -0.0496714084349,
	                                        0.0, 0.0, 0.0]
	                                      - [-2.00363215516, -6.67916820283, 0.0, 0.0, 0.0,
	                                        0.0]
	                                      - [-1.39018616912, 0.0, 0.0, 0.0, 0.0, 0.0]
	                                      datatype: float64
	                                      shape: [6, 6]
	                                    domain:
	                                    - [-1, 1]
	                                    - [-1, 1]
	                                    inputs: [x, y]
	                                    name: collimator_y_backward
	                                    outputs: [z]
	                                    window:
	                                    - [-1, 1]
	                                    - [-1, 1]
	                                  inputs: [x0, y0, x1, y1]
	                                  outputs: [z0, z1]
	                                - inputs: [x0, x1]
	                                  n_dims: 2
	                                  outputs: [x0, x1]
	                                inputs: [x0, y0, x1, y1]
	                                outputs: [x0, x1]
	                              inputs: [x0, x1]
	                              outputs: [x0, x1]
	                            inputs: [x0, x1]
	                            outputs: [x0, x1]
	                          - inputs: [x, y]
	                            model_type: unitless2directional
	                            name: unitless2directional_cosines
	                            outputs: [x, y, z]
	                          inputs: [x0, x1]
	                          outputs: [x, y, z]
	                        - angles: [0.03333072666861111, -0.27547251631138886, -0.14198882781777777,
	                            24.29]
	                          axes_order: xyzy
	                          inputs: [x, y, z]
	                          name: rotation
	                          outputs: [x, y, z]
	                        inputs: [x0, x1]
	                        outputs: [x, y, z]
	                      inputs: [x0, x1]
	                      outputs: [x, y, z]
	                    inputs: [x00, x10, x01, x11]
	                    outputs: [x0, x1, x, y, z]
	                  - inputs: [x0, x1]
	                    n_dims: 2
	                    outputs: [x0, x1]
	                  inputs: [x00, x10, x01, x11, x0, x1]
	                  outputs: [x00, x10, x0, y0, z0, x01, x11]
	                inputs: [x0, x1, x2]
	                outputs: [x00, x10, x0, y0, z0, x01, x11]
	              - inputs: [x0, x1, x2, x3, x4, x5, x6]
	                mapping: [0, 1, 2, 3, 5]
	                n_inputs: 7
	                outputs: [x0, x1, x2, x3, x4]
	              inputs: [x0, x1, x2]
	              outputs: [x0, x1, x2, x3, x4]
	            - forward:
	              - inputs: [x0, x1]
	                n_dims: 2
	                outputs: [x0, x1]
	              - forward:
	                - forward:
	                  - inputs: [alpha_in, beta_in, alpha_out]
	                    name: n_prism
	                    outputs: [n]
	                    prism_angle: -16.5
	                  - bounding_box: [1.3871267867024815, 1.4383165119633379]
	                    bounds_error: false
	                    fill_value: .nan
	                    inputs: [x]
	                    lookup_table: !core/ndarray-1.0.0
	                      data: [6.000000000000005e-06, 5.99499995736744e-06, 5.9900000000000045e-06,
	                        5.9850000000000045e-06, 5.980000000000005e-06, 5.975000000000005e-06,
	                        5.970000000000005e-06, 5.965000000000005e-06, 5.960000000000004e-06,
	                        5.955000000000004e-06, 5.950000000000004e-06, 5.945000000000004e-06,
	                        5.940000000000005e-06, 5.935000000000005e-06, 5.930000000000005e-06,
	                        5.925000000000005e-06, 5.920000000000005e-06, 5.915000000000004e-06,
	                        5.910000000000004e-06, 5.9050000000000044e-06, 5.9000000000000045e-06,
	                        5.8950000000000045e-06, 5.890000000000005e-06, 5.885000000000005e-06,
	                        5.880000000000004e-06, 5.875000000000004e-06, 5.870000000000004e-06,
	                        5.865000000000004e-06, 5.860000000000005e-06, 5.855000000000005e-06,
	                        5.850000000000005e-06, 5.845000000000005e-06, 5.840000000000005e-06,
	                        5.835000000000004e-06, 5.830000000000004e-06, 5.825000000000004e-06,
	                        5.820000000000004e-06, 5.8150000000000045e-06, 5.8100000000000045e-06,
	                        5.8050000000000046e-06, 5.800000000000004e-06, 5.795000000000004e-06,
	                        5.790000000000004e-06, 5.785000000000004e-06, 5.780000000000005e-06,
	                        5.775000000000005e-06, 5.770000000000002e-06, 5.765000000000005e-06,
	                        5.760000000000005e-06, 5.755000000000004e-06, 5.750000000000004e-06,
	                        5.745000000000004e-06, 5.740000000000004e-06, 5.732206032276158e-06,
	                        5.7300000000000044e-06, 5.7250000000000045e-06, 5.720000000000004e-06,
	                        5.715000000000004e-06, 5.710000000000004e-06, 5.705000000000005e-06,
	                        5.700000000000005e-06, 5.695000000000005e-06, 5.690000000000005e-06,
	                        5.685000000000005e-06, 5.680000000000005e-06, 5.675000000000004e-06,
	                        5.670000000000004e-06, 5.665000000000004e-06, 5.660000000000004e-06,
	                        5.655000000000004e-06, 5.650000000000004e-06, 5.645000000000004e-06,
	                        5.640000000000004e-06, 5.635000000000004e-06, 5.630000000000004e-06,
	                        5.625000000000005e-06, 5.620000000000005e-06, 5.615000000000005e-06,
	                        5.610000000000005e-06, 5.605000000000005e-06, 5.600000000000005e-06,
	                        5.595000000000004e-06, 5.590000000000004e-06, 5.585000000000004e-06,
	                        5.580000000000004e-06, 5.575000000000004e-06, 5.570000000000004e-06,
	                        5.565000000000004e-06, 5.5600000000000035e-06, 5.5550000000000036e-06,
	                        5.550000000000004e-06, 5.5450000000000045e-06, 5.5400000000000046e-06,
	                        5.535000000000005e-06, 5.530000000000005e-06, 5.525000000000005e-06,
	                        5.520000000000005e-06, 5.515000000000004e-06, 5.510000000000004e-06,
	                        5.505000000000004e-06, 5.500000000000004e-06, 5.495000000000004e-06,
	                        5.490000000000004e-06, 5.485000000000004e-06, 5.4800000000000034e-06,
	                        5.4750000000000035e-06, 5.4700000000000035e-06, 5.4650000000000044e-06,
	                        5.4600000000000045e-06, 5.4550000000000045e-06, 5.4500000000000046e-06,
	                        5.445000000000005e-06, 5.440000000000005e-06, 5.435000000000004e-06,
	                        5.430000000000004e-06, 5.425000000000004e-06, 5.420000000000004e-06,
	                        5.415000000000004e-06, 5.410000000000004e-06, 5.405000000000002e-06,
	                        5.400000000000003e-06, 5.395000000000003e-06, 5.390000000000004e-06,
	                        5.385000000000004e-06, 5.380000000000004e-06, 5.3750000000000044e-06,
	                        5.3700000000000045e-06, 5.3650000000000045e-06, 5.360000000000005e-06,
	                        5.355000000000004e-06, 5.350000000000004e-06, 5.345000000000004e-06,
	                        5.340000000000004e-06, 5.335000000000004e-06, 5.330000000000004e-06,
	                        5.325000000000004e-06, 5.320000000000003e-06, 5.315000000000003e-06,
	                        5.310000000000004e-06, 5.305000000000004e-06, 5.300000000000004e-06,
	                        5.295000000000004e-06, 5.290000000000004e-06, 5.2850000000000045e-06,
	                        5.2800000000000045e-06, 5.275000000000004e-06, 5.270000000000004e-06,
	                        5.265000000000004e-06, 5.260000000000004e-06, 5.255000000000004e-06,
	                        5.250000000000004e-06, 5.245000000000004e-06, 5.240000000000003e-06,
	                        5.235000000000003e-06, 5.230000000000004e-06, 5.225000000000004e-06,
	                        5.220000000000004e-06, 5.215000000000004e-06, 5.210000000000004e-06,
	                        5.205000000000004e-06, 5.200000000000004e-06, 5.195000000000004e-06,
	                        5.190000000000004e-06, 5.185000000000004e-06, 5.180000000000004e-06,
	                        5.175000000000004e-06, 5.170000000000004e-06, 5.165000000000004e-06,
	                        5.160000000000003e-06, 5.155000000000003e-06, 5.150000000000004e-06,
	                        5.145000000000004e-06, 5.140000000000004e-06, 5.13499995736744e-06,
	                        5.130000000000004e-06, 5.125000000000004e-06, 5.120000000000004e-06,
	                        5.1150000000000035e-06, 5.110000000000004e-06, 5.105000000000004e-06,
	                        5.100000000000004e-06, 5.095000000000004e-06, 5.090000000000004e-06,
	                        5.085000000000004e-06, 5.080000000000003e-06, 5.075000000000003e-06,
	                        5.070000000000004e-06, 5.065000000000004e-06, 5.060000000000004e-06,
	                        5.055000000000004e-06, 5.050000000000004e-06, 5.045000000000004e-06,
	                        5.040000000000004e-06, 5.0350000000000035e-06, 5.0300000000000035e-06,
	                        5.0250000000000036e-06, 5.020000000000004e-06, 5.01499999983282e-06,
	                        5.010000000000004e-06, 5.005000000000004e-06, 5.000000000000003e-06,
	                        4.995000000000004e-06, 4.990000000000004e-06, 4.985000000000004e-06,
	                        4.980000000000004e-06, 4.975000000000004e-06, 4.970000000000004e-06,
	                        4.965000000000004e-06, 4.960000000000004e-06, 4.955000000000003e-06,
	                        4.9500000000000034e-06, 4.9450000000000035e-06, 4.9400000000000035e-06,
	                        4.9350000000000036e-06, 4.930000000000004e-06, 4.925000000000004e-06,
	                        4.920000000000003e-06, 4.915000000000004e-06, 4.910000000000004e-06,
	                        4.905000000000004e-06, 4.900000000000004e-06, 4.895000000000004e-06,
	                        4.890000000000004e-06, 4.885000000000004e-06, 4.880000000000004e-06,
	                        4.875000000000003e-06, 4.870000000000003e-06, 4.865000000000003e-06,
	                        4.8600000000000034e-06, 4.8550000000000035e-06, 4.8500000000000035e-06,
	                        4.845000000000004e-06, 4.840000000000003e-06, 4.835000000000004e-06,
	                        4.830000000000004e-06, 4.825000000000004e-06, 4.820000000000004e-06,
	                        4.815000000000004e-06, 4.810000000000004e-06, 4.805000000000004e-06,
	                        4.7999999998334706e-06, 4.795000000000003e-06, 4.790000000000003e-06,
	                        4.785000000000003e-06, 4.780000000000003e-06, 4.775000000000003e-06,
	                        4.7700000000000035e-06, 4.7650000000000035e-06, 4.760000000000003e-06,
	                        4.755000000000004e-06, 4.750000000000004e-06, 4.745000000000004e-06,
	                        4.740000000000004e-06, 4.735000000000004e-06, 4.730000000000004e-06,
	                        4.725000000000004e-06, 4.720000000000004e-06, 4.715000000000003e-06,
	                        4.710000000000003e-06, 4.705000000000003e-06, 4.700000000000003e-06,
	                        4.695000000000003e-06, 4.690000000000003e-06, 4.6850000000000034e-06,
	                        4.6800000000000035e-06, 4.6750000000000035e-06, 4.6700000000000036e-06,
	                        4.665000000000004e-06, 4.660000000000004e-06, 4.655000000000004e-06,
	                        4.650000000000004e-06, 4.645000000000004e-06, 4.640000000000004e-06,
	                        4.635000000000003e-06, 4.630000000000003e-06, 4.625000000000003e-06,
	                        4.620000000000003e-06, 4.615000000000003e-06, 4.610000000000003e-06,
	                        4.605000000000004e-06, 4.600000000000003e-06, 4.5950000000000034e-06,
	                        4.5900000000000035e-06, 4.58499999983347e-06, 4.580000000000004e-06,
	                        4.575000000000004e-06, 4.570000000000004e-06, 4.565000000000004e-06,
	                        4.560000000000004e-06, 4.555000000000003e-06, 4.550000000000003e-06,
	                        4.545000000000003e-06, 4.540000000000003e-06, 4.535000000000003e-06,
	                        4.530000000000003e-06, 4.525000000000004e-06, 4.520000000000003e-06,
	                        4.515000000000003e-06, 4.510000000000003e-06, 4.5050000000000035e-06,
	                        4.5000000000000035e-06, 4.4950000000000036e-06, 4.490000000000004e-06,
	                        4.485000000000003e-06, 4.480000000000004e-06, 4.475000000000003e-06,
	                        4.470000000000003e-06, 4.465000000000003e-06, 4.460000000000003e-06,
	                        4.455000000000003e-06, 4.450000000000003e-06, 4.445000000000004e-06,
	                        4.440000000000003e-06, 4.435000000000004e-06, 4.430000000000003e-06,
	                        4.425000000000003e-06, 4.420000000000003e-06, 4.4150000000000035e-06,
	                        4.4100000000000035e-06, 4.405000000000003e-06, 4.400000000000004e-06,
	                        4.395000000000003e-06, 4.390000000000003e-06, 4.385000000000003e-06,
	                        4.380000000000003e-06, 4.375000000000003e-06, 4.370000000000003e-06,
	                        4.365000000000004e-06, 4.360000000000003e-06, 4.355000000000004e-06,
	                        4.350000000000003e-06, 4.345000000000003e-06, 4.340000000000003e-06,
	                        4.335000000000003e-06, 4.3300000000000034e-06, 4.325000000000003e-06,
	                        4.3200000000000035e-06, 4.315000000000003e-06, 4.310000000000003e-06,
	                        4.305000000000003e-06, 4.300000000000003e-06, 4.295000000000003e-06,
	                        4.290000000000003e-06, 4.285000000000004e-06, 4.280000000000003e-06,
	                        4.27499995736744e-06, 4.270000000000003e-06, 4.265000000000003e-06,
	                        4.260000000000003e-06, 4.255000000000003e-06, 4.250000000000003e-06,
	                        4.2450000000000026e-06, 4.2400000000000035e-06, 4.235000000000003e-06,
	                        4.230000000000003e-06, 4.225000000000003e-06, 4.220000000000003e-06,
	                        4.215000000000003e-06, 4.210000000000004e-06, 4.205000000000004e-06,
	                        4.200000000000003e-06, 4.195000000000004e-06, 4.190000000000003e-06,
	                        4.185000000000003e-06, 4.180000000000003e-06, 4.175000000000003e-06,
	                        4.170000000000003e-06, 4.1650000000000025e-06, 4.160000000000003e-06,
	                        4.1550000000000026e-06, 4.150000000000003e-06, 4.145000000000003e-06,
	                        4.140000000000003e-06, 4.135000000000003e-06, 4.130000000000004e-06,
	                        4.125000000000004e-06, 4.120000000000003e-06, 4.115000000000004e-06,
	                        4.110000000000003e-06, 4.105000000000003e-06, 4.100000000000003e-06,
	                        4.095000000000003e-06, 4.090000000000003e-06, 4.085000000000002e-06,
	                        4.080000000000003e-06, 4.0750000000000025e-06, 4.0700000000000025e-06,
	                        4.065000000000003e-06, 4.060000000000003e-06, 4.055000000000003e-06,
	                        4.0500000000000036e-06, 4.045000000000004e-06, 4.040000000000003e-06,
	                        4.035000000000004e-06, 4.030000000000003e-06, 4.025000000000003e-06,
	                        4.020000000000003e-06, 4.015000000000003e-06, 4.010000000000003e-06,
	                        4.005000000000002e-06, 4.000000000000003e-06, 3.995000000000003e-06,
	                        3.9900000000000025e-06, 3.9850000000000025e-06, 3.9800000000000026e-06,
	                        3.975000000000003e-06, 3.9700000000000035e-06, 3.965000000000003e-06,
	                        3.960000000000003e-06, 3.955000000000003e-06, 3.950000000000003e-06,
	                        3.945000000000003e-06, 3.940000000000003e-06, 3.935000000000003e-06,
	                        3.930000000000003e-06, 3.925000000000003e-06, 3.920000000000003e-06,
	                        3.915000000000003e-06, 3.910000000000002e-06, 3.905000000000002e-06,
	                        3.9000000000000025e-06, 3.895000000000003e-06, 3.890000000000003e-06,
	                        3.885000000000003e-06, 3.880000000000003e-06, 3.875000000000003e-06,
	                        3.870000000000003e-06, 3.865000000000003e-06, 3.860000000000003e-06,
	                        3.854999999999352e-06, 3.850000000000003e-06, 3.845000000000003e-06,
	                        3.840000000000003e-06, 3.835000000000003e-06, 3.830000000000002e-06,
	                        3.825000000000002e-06, 3.820000000000002e-06, 3.815000000000003e-06,
	                        3.810000000000003e-06, 3.8050000000000025e-06, 3.8000000000000026e-06,
	                        3.795000000000003e-06, 3.7900000000000027e-06, 3.7850000000000027e-06,
	                        3.7800000000000028e-06, 3.775000000000003e-06, 3.770000000000003e-06,
	                        3.7650000000000025e-06, 3.7600000000000025e-06, 3.755000000000003e-06,
	                        3.7500000000000026e-06, 3.7450000000000027e-06, 3.7400000000000027e-06,
	                        3.7350000000000028e-06, 3.730000000000003e-06, 3.7250000000000025e-06,
	                        3.7200000000000025e-06, 3.715000000000003e-06, 3.7100000000000026e-06,
	                        3.7050000000000026e-06, 3.7000000000000027e-06, 3.6950000000000027e-06,
	                        3.6900000000000028e-06, 3.6850000000000024e-06, 3.6800000000000025e-06,
	                        3.675000000000003e-06, 3.6700000000000013e-06, 3.6650000000000026e-06,
	                        3.6600000000000027e-06, 3.654994543031792e-06, 3.6500000000000027e-06,
	                        3.6450000000000024e-06, 3.6400000000000024e-06, 3.635000000000003e-06,
	                        3.6300000000000025e-06, 3.6250000000000026e-06, 3.6200000000000026e-06,
	                        3.6150000000000027e-06, 3.6100000000000027e-06, 3.6050000000000023e-06,
	                        3.600000000000003e-06, 3.595000000000003e-06, 3.5900000000000025e-06,
	                        3.5850000000000025e-06, 3.5800000000000026e-06, 3.5750000000000026e-06,
	                        3.5700000000000027e-06, 3.5650000000000023e-06, 3.5600000000000028e-06,
	                        3.555000000000003e-06, 3.5500000000000024e-06, 3.5450000000000025e-06,
	                        3.5400000000000025e-06, 3.5350000000000026e-06, 3.5300000000000026e-06,
	                        3.5250000000000022e-06, 3.5200000000000027e-06, 3.5150000000000028e-06,
	                        3.5100000000000024e-06, 3.5050000000000024e-06, 3.5000000000000025e-06,
	                        3.4950000000000025e-06, 3.4900000000000026e-06, 3.485000000000002e-06,
	                        3.4800000000000027e-06, 3.4750000000000027e-06, 3.4700000000000024e-06,
	                        3.4650000000000024e-06, 3.4600000000000024e-06, 3.4550000000000025e-06,
	                        3.4500000000000025e-06, 3.445000000000002e-06, 3.4400000000000026e-06,
	                        3.4350000000000027e-06, 3.4300000000000023e-06, 3.4250000000000024e-06,
	                        3.4200000000000024e-06, 3.4150000000000025e-06, 3.4100000000000025e-06,
	                        3.405000000000002e-06, 3.4000000000000026e-06, 3.3950000000000026e-06,
	                        3.3900000000000023e-06, 3.3850000000000023e-06, 3.3800000000000024e-06,
	                        3.3750000000000024e-06, 3.3700000000000025e-06, 3.365000000000002e-06,
	                        3.3600000000000026e-06, 3.3550000000000026e-06, 3.3500000000000022e-06,
	                        3.3450000000000023e-06, 3.3400000000000023e-06, 3.3350000000000024e-06,
	                        3.3300000000000024e-06, 3.325000000000002e-06, 3.3200000000000025e-06,
	                        3.3150000000000026e-06, 3.310000000000002e-06, 3.3050000000000022e-06,
	                        3.3000000000000023e-06, 3.2950000000000023e-06, 3.2900000000000024e-06,
	                        3.285000000000002e-06, 3.2800000000000025e-06, 3.2750000000000025e-06,
	                        3.270000000000002e-06, 3.265000000000002e-06, 3.2600000000000022e-06,
	                        3.2550000000000023e-06, 3.2500000000000023e-06, 3.2450000000000024e-06,
	                        3.2400000000000024e-06, 3.2350000000000025e-06, 3.230000000000002e-06,
	                        3.223603016138079e-06, 3.220000000000002e-06, 3.2150000000000023e-06,
	                        3.2100000000000023e-06, 3.2050000000000023e-06, 3.2000000000000024e-06,
	                        3.1950000000000024e-06, 3.190000000000002e-06, 3.185000000000002e-06,
	                        3.180000000000002e-06, 3.1750000000000022e-06, 3.1700000000000023e-06,
	                        3.1650000000000023e-06, 3.1600000000000024e-06, 3.1550000000000024e-06,
	                        3.150000000000002e-06, 3.145000000000002e-06, 3.140000000000002e-06,
	                        3.135000000000002e-06, 3.1300000000000022e-06, 3.1250000000000023e-06,
	                        3.1200000000000023e-06, 3.1150000000000024e-06, 3.110000000000002e-06,
	                        3.105000000000002e-06, 3.100000000000002e-06, 3.095000000000002e-06,
	                        3.090000000000002e-06, 3.0850000000000022e-06, 3.0800000000000023e-06,
	                        3.0750000000000023e-06, 3.070000000000002e-06, 3.065000000000002e-06,
	                        3.060000000000002e-06, 3.055000000000002e-06, 3.050000000000002e-06,
	                        3.045000000000002e-06, 3.0400000000000022e-06, 3.0350000000000023e-06,
	                        3.030000000000002e-06, 3.025000000000002e-06, 3.020000000000002e-06,
	                        3.015000000000002e-06, 3.0100000000000025e-06, 3.005000000000002e-06,
	                        3.000000000000002e-06, 2.9950000000000022e-06, 2.990000000000002e-06,
	                        2.985000000000002e-06, 2.980000000000002e-06, 2.975000000000002e-06,
	                        2.9700000000000025e-06, 2.965000000000002e-06, 2.960000000000002e-06,
	                        2.955000000000002e-06, 2.950000000000002e-06, 2.945000000000002e-06,
	                        2.940000000000002e-06, 2.935000000000002e-06, 2.9300000000000024e-06,
	                        2.925000000000002e-06, 2.920000000000002e-06, 2.915000000000002e-06,
	                        2.9100000000000018e-06, 2.905000000000002e-06, 2.900000000000002e-06,
	                        2.895000000000002e-06, 2.8900000000000024e-06, 2.885000000000002e-06,
	                        2.880000000000002e-06, 2.875000000000002e-06, 2.8700000000000017e-06,
	                        2.865000000000002e-06, 2.860000000000002e-06, 2.855000000000002e-06,
	                        2.8500000000000024e-06, 2.845000000000002e-06, 2.840000000000002e-06,
	                        2.835000000000002e-06, 2.8300000000000017e-06, 2.8250000000000018e-06,
	                        2.820000000000002e-06, 2.815000000000002e-06, 2.8100000000000023e-06,
	                        2.805000000000002e-06, 2.800000000000002e-06, 2.795000000000002e-06,
	                        2.7900000000000017e-06, 2.7850000000000017e-06, 2.7800000000000018e-06,
	                        2.775000000000002e-06, 2.7700000000000023e-06, 2.765000000000002e-06,
	                        2.760000000000002e-06, 2.755000000000002e-06, 2.7500000000000016e-06,
	                        2.7450000000000017e-06, 2.7400000000000017e-06, 2.7350000000000018e-06,
	                        2.7300000000000022e-06, 2.725000000000002e-06, 2.720000000000002e-06,
	                        2.715000000000002e-06, 2.7100000000000016e-06, 2.7050000000000016e-06,
	                        2.7000000000000017e-06, 2.695000000000002e-06, 2.690000000000002e-06,
	                        2.685000000000002e-06, 2.680000000000002e-06, 2.675000000000002e-06,
	                        2.6700000000000015e-06, 2.6650000000000016e-06, 2.6600000000000016e-06,
	                        2.655000000000002e-06, 2.650000000000002e-06, 2.6450000000000018e-06,
	                        2.640000000000002e-06, 2.635000000000002e-06, 2.6300000000000015e-06,
	                        2.6250000000000015e-06, 2.6200000000000016e-06, 2.615000000000002e-06,
	                        2.610000000000002e-06, 2.6050000000000017e-06, 2.6000000000000018e-06,
	                        2.595000000000002e-06, 2.5900000000000015e-06, 2.5850000000000015e-06,
	                        2.5800000000000016e-06, 2.575000000000002e-06, 2.570000000000002e-06,
	                        2.5650000000000017e-06, 2.5600000000000017e-06, 2.555000000000002e-06,
	                        2.5500000000000014e-06, 2.5450000000000015e-06, 2.5400000000000015e-06,
	                        2.535000000000002e-06, 2.530000000000002e-06, 2.5250000000000017e-06,
	                        2.5200000000000004e-06, 2.5150000000000018e-06, 2.5100000000000014e-06,
	                        2.5050000000000014e-06, 2.5000000000000015e-06, 2.495000000000002e-06,
	                        2.490000000000002e-06, 2.4850000000000016e-06, 2.4800000000000017e-06,
	                        2.4750000000000017e-06, 2.4700000000000013e-06, 2.4650000000000014e-06,
	                        2.4600000000000014e-06, 2.455000000000002e-06, 2.450000000000002e-06,
	                        2.4450000000000016e-06, 2.4400000000000016e-06, 2.4350000000000017e-06,
	                        2.4300000000000013e-06, 2.4250000000000013e-06, 2.4200000000000014e-06,
	                        2.415000000000002e-06, 2.410000000000002e-06, 2.4050000000000015e-06,
	                        2.399999999916735e-06, 2.3950000000000016e-06, 2.3900000000000013e-06,
	                        2.3850000000000013e-06, 2.3800000000000014e-06, 2.375000000000002e-06,
	                        2.370000000000002e-06, 2.3650000000000015e-06, 2.3600000000000015e-06,
	                        2.3550000000000016e-06, 2.3500000000000012e-06, 2.3450000000000013e-06,
	                        2.3400000000000017e-06, 2.3350000000000018e-06, 2.330000000000002e-06,
	                        2.3250000000000015e-06, 2.3200000000000015e-06, 2.3150000000000016e-06,
	                        2.310000000000001e-06, 2.3050000000000012e-06, 2.3000000000000017e-06,
	                        2.2950000000000017e-06, 2.290000000000002e-06, 2.2850000000000014e-06,
	                        2.2800000000000015e-06, 2.2750000000000015e-06, 2.270000000000001e-06,
	                        2.265000000000001e-06, 2.2600000000000017e-06, 2.2550000000000017e-06,
	                        2.2500000000000018e-06, 2.2450000000000014e-06, 2.2400000000000014e-06,
	                        2.2350000000000015e-06, 2.230000000000001e-06, 2.225000000000001e-06,
	                        2.2200000000000016e-06, 2.2150000000000017e-06, 2.2100000000000017e-06,
	                        2.2050000000000013e-06, 2.2000000000000014e-06, 2.1950000000000014e-06,
	                        2.190000000000001e-06, 2.185000000000001e-06, 2.1800000000000016e-06,
	                        2.1750000000000016e-06, 2.1700000000000017e-06, 2.1650000000000013e-06,
	                        2.1600000000000013e-06, 2.1550000000000014e-06, 2.150000000000001e-06,
	                        2.145000000000001e-06, 2.1400000000000015e-06, 2.1350000000000016e-06,
	                        2.1300000000000016e-06, 2.1250000000000013e-06, 2.1200000000000013e-06,
	                        2.1150000000000013e-06, 2.110000000000001e-06, 2.105000000000001e-06,
	                        2.1000000000000015e-06, 2.0950000000000015e-06, 2.0900000000000016e-06,
	                        2.0850000000000012e-06, 2.0800000000000013e-06, 2.0750000000000013e-06,
	                        2.070000000000001e-06, 2.065000000000001e-06, 2.0600000000000015e-06,
	                        2.0550000000000015e-06, 2.0500000000000015e-06, 2.045000000000001e-06,
	                        2.0400000000000012e-06, 2.0350000000000013e-06, 2.030000000000001e-06,
	                        2.025000000000001e-06, 2.0200000000000014e-06, 2.0150000000000015e-06,
	                        2.0100000000000015e-06, 2.005000000000001e-06, 2.000000000000001e-06,
	                        1.9950000000000012e-06, 1.9900000000000013e-06, 1.9850000000000013e-06,
	                        1.9800000000000014e-06, 1.9750000000000014e-06, 1.970000000000001e-06,
	                        1.965000000000001e-06, 1.960000000000001e-06, 1.955000000000001e-06,
	                        1.9500000000000012e-06, 1.9450000000000013e-06, 1.9400000000000013e-06,
	                        1.9350000000000014e-06, 1.930000000000001e-06, 1.925000000000001e-06,
	                        1.920000000000001e-06, 1.915000000000001e-06, 1.910000000000001e-06,
	                        1.905000000000001e-06, 1.900000000000001e-06, 1.8950000000000013e-06,
	                        1.8900000000000012e-06, 1.885000000000001e-06, 1.880000000000001e-06,
	                        1.8750000000000013e-06, 1.8700000000000012e-06, 1.865000000000001e-06,
	                        1.860000000000001e-06, 1.8550000000000013e-06, 1.8500000000000011e-06,
	                        1.845000000000001e-06, 1.840000000000001e-06, 1.8350000000000006e-06,
	                        1.8300000000000011e-06, 1.825000000000001e-06, 1.820000000000001e-06,
	                        1.8150000000000013e-06, 1.810000000000001e-06, 1.805000000000001e-06,
	                        1.800000000000001e-06, 1.7950000000000012e-06, 1.790000000000001e-06,
	                        1.785000000000001e-06, 1.780000000000001e-06, 1.7750000000000012e-06,
	                        1.770000000000001e-06, 1.7650000000000009e-06, 1.760000000000001e-06,
	                        1.7550000000000012e-06, 1.750000000000001e-06, 1.7450000000000009e-06,
	                        1.7400000000000011e-06, 1.7350000000000012e-06, 1.730000000000001e-06,
	                        1.7250000000000008e-06, 1.7200000000000011e-06, 1.7150000000000012e-06,
	                        1.710000000000001e-06, 1.7050000000000008e-06, 1.700000000000001e-06,
	                        1.6950000000000011e-06, 1.690000000000001e-06, 1.6850000000000008e-06,
	                        1.680000000000001e-06, 1.6750000000000011e-06, 1.670000000000001e-06,
	                        1.6650000000000008e-06, 1.660000000000001e-06, 1.655000000000001e-06,
	                        1.650000000000001e-06, 1.6450000000000008e-06, 1.640000000000001e-06,
	                        1.635000000000001e-06, 1.630000000000001e-06, 1.6250000000000007e-06,
	                        1.620000000000001e-06, 1.615000000000001e-06, 1.6100000000000009e-06,
	                        1.6050000000000007e-06, 1.600000000000001e-06, 1.595000000000001e-06,
	                        1.5900000000000009e-06, 1.5850000000000007e-06, 1.580000000000001e-06,
	                        1.575000000000001e-06, 1.5700000000000009e-06, 1.5650000000000007e-06,
	                        1.560000000000001e-06, 1.555000000000001e-06, 1.5500000000000008e-06,
	                        1.5450000000000007e-06, 1.540000000000001e-06, 1.535000000000001e-06,
	                        1.5300000000000008e-06, 1.5250000000000006e-06, 1.520000000000001e-06,
	                        1.515000000000001e-06, 1.5100000000000008e-06, 1.5050000000000006e-06,
	                        1.5000000000000009e-06, 1.495000000000001e-06, 1.4900000000000008e-06,
	                        1.4850000000000006e-06, 1.4800000000000009e-06, 1.475000000000001e-06,
	                        1.4700000000000007e-06, 1.4650000000000006e-06, 1.4600000000000008e-06,
	                        1.4550000000000009e-06, 1.4500000000000007e-06, 1.4450000000000006e-06,
	                        1.4400000000000008e-06, 1.4350000000000009e-06, 1.4300000000000007e-06,
	                        1.4250000000000005e-06, 1.4200000000000008e-06, 1.4150000000000009e-06,
	                        1.4100000000000007e-06, 1.4050000000000005e-06, 1.4000000000000008e-06,
	                        1.3950000000000008e-06, 1.3900000000000007e-06, 1.3850000000000007e-06,
	                        1.3800000000000008e-06, 1.3750000000000008e-06, 1.3700000000000006e-06,
	                        1.3650000000000007e-06, 1.3600000000000007e-06, 1.3550000000000008e-06,
	                        1.3500000000000006e-06, 1.3450000000000007e-06, 1.3400000000000007e-06,
	                        1.3350000000000008e-06, 1.3300000000000006e-06, 1.3250000000000007e-06,
	                        1.3200000000000007e-06, 1.3150000000000008e-06, 1.3100000000000006e-06,
	                        1.3050000000000006e-06, 1.3000000000000007e-06, 1.2950000000000007e-06,
	                        1.2900000000000006e-06, 1.2850000000000006e-06, 1.2800000000000007e-06,
	                        1.2750000000000007e-06, 1.2700000000000005e-06, 1.2650000000000006e-06,
	                        1.2600000000000006e-06, 1.2550000000000007e-06, 1.2500000000000005e-06,
	                        1.2450000000000006e-06, 1.2400000000000006e-06, 1.2350000000000007e-06,
	                        1.2300000000000005e-06, 1.2250000000000006e-06, 1.2200000000000006e-06,
	                        1.2150000000000006e-06, 1.2100000000000005e-06, 1.2050000000000005e-06,
	                        1.1999999999583672e-06, 1.1950000000000006e-06, 1.1900000000000005e-06,
	                        1.1850000000000005e-06, 1.1800000000000006e-06, 1.1750000000000006e-06,
	                        1.1700000000000004e-06, 1.1650000000000005e-06, 1.1600000000000005e-06,
	                        1.1550000000000006e-06, 1.1500000000000004e-06, 1.1450000000000005e-06,
	                        1.1400000000000005e-06, 1.1350000000000006e-06, 1.1300000000000004e-06,
	                        1.1250000000000005e-06, 1.1200000000000005e-06, 1.1150000000000005e-06,
	                        1.1100000000000006e-06, 1.1050000000000004e-06, 1.1000000000000005e-06,
	                        1.0950000000000005e-06, 1.0900000000000006e-06, 1.0850000000000004e-06,
	                        1.0800000000000005e-06, 1.0750000000000005e-06, 1.0700000000000006e-06,
	                        1.0650000000000004e-06, 1.0600000000000004e-06, 1.0550000000000005e-06,
	                        1.0500000000000005e-06, 1.0450000000000004e-06, 1.0400000000000004e-06,
	                        1.0350000000000005e-06, 1.0300000000000005e-06, 1.0250000000000004e-06,
	                        1.0200000000000004e-06, 1.0150000000000004e-06, 1.0100000000000005e-06,
	                        1.0050000000000003e-06, 1.0000000000000004e-06, 9.950000000000004e-07,
	                        9.900000000000005e-07, 9.850000000000003e-07, 9.800000000000004e-07,
	                        9.750000000000004e-07, 9.700000000000005e-07, 9.650000000000003e-07,
	                        9.600000000000003e-07, 9.550000000000004e-07, 9.500000000000003e-07,
	                        9.450000000000004e-07, 9.400000000000003e-07, 9.350000000000004e-07,
	                        9.300000000000003e-07, 9.250000000000004e-07, 9.200000000000003e-07,
	                        9.150000000000003e-07, 9.100000000000003e-07, 9.050000000000003e-07,
	                        9.000000000000003e-07, 8.950000000000003e-07, 8.900000000000003e-07,
	                        8.850000000000003e-07, 8.800000000000003e-07, 8.750000000000003e-07,
	                        8.700000000000002e-07, 8.650000000000003e-07, 8.600000000000002e-07,
	                        8.550000000000003e-07, 8.500000000000002e-07, 8.450000000000003e-07,
	                        8.400000000000002e-07, 8.350000000000003e-07, 8.300000000000002e-07,
	                        8.250000000000003e-07, 8.200000000000002e-07, 8.150000000000002e-07,
	                        8.100000000000003e-07, 8.050000000000002e-07, 8.000000000000003e-07,
	                        7.950000000000002e-07, 7.900000000000003e-07, 7.850000000000002e-07,
	                        7.800000000000003e-07, 7.750000000000002e-07, 7.700000000000003e-07,
	                        7.650000000000002e-07, 7.600000000000002e-07, 7.550000000000002e-07,
	                        7.500000000000002e-07, 7.450000000000002e-07, 7.400000000000002e-07,
	                        7.350000000000002e-07, 7.300000000000002e-07, 7.250000000000002e-07,
	                        7.200000000000002e-07, 7.150000000000001e-07, 7.100000000000002e-07,
	                        7.050000000000001e-07, 7.000000000000002e-07, 6.950000000000001e-07,
	                        6.900000000000002e-07, 6.850000000000001e-07, 6.800000000000002e-07,
	                        6.750000000000001e-07, 6.700000000000001e-07, 6.650000000000001e-07,
	                        6.600000000000001e-07, 6.550000000000001e-07, 6.500000000000001e-07,
	                        6.450000000000001e-07, 6.400000000000001e-07, 6.350000000000001e-07,
	                        6.300000000000001e-07, 6.25e-07, 6.200000000000001e-07, 6.15e-07,
	                        6.100000000000001e-07, 6.05e-07, 5.999999999791834e-07, 5.95e-07,
	                        5.900000000000001e-07, 5.85e-07, 5.800000000000001e-07, 5.75e-07,
	                        5.7e-07, 5.65e-07, 5.6e-07, 5.55e-07, 5.5e-07, 5.45e-07, 5.4e-07,
	                        5.35e-07, 5.3e-07, 5.25e-07, 5.2e-07, 5.149999999999999e-07, 5.1e-07,
	                        5.049999999999999e-07, 5.0e-07]
	                      datatype: float64
	                      shape: [1101]
	                    method: linear
	                    outputs: [y]
	                    points:
	                    - !core/ndarray-1.0.0
	                      data: [1.3871267867024815, 1.3872013575751927, 1.3872758549584026,
	                        1.387350278885151, 1.3874246293884391, 1.3874988953253609, 1.3875731102564561,
	                        1.3876472406870024, 1.3877212978257238, 1.3877952817054366, 1.3878691923589197,
	                        1.3879430298189157, 1.3880167941181296, 1.3880904852892302, 1.3881641033648497,
	                        1.3882376483775833, 1.3883111203599896, 1.388384519344591, 1.3884578453638732,
	                        1.388531098450285, 1.3886042786362398, 1.3886773859541146, 1.388750420436249,
	                        1.388823382114948, 1.3888962710224795, 1.3889690871910754, 1.389041830652932,
	                        1.3891145014402095, 1.3891870995850324, 1.389259625119489, 1.3893320780756324,
	                        1.389404458485479, 1.3894767663810113, 1.3895490017941743, 1.389621164756879,
	                        1.3896932553010002, 1.3897652734583776, 1.3898372192608155, 1.3899090927400826,
	                        1.3899808939279135, 1.3900526228560062, 1.3901242795560247, 1.3901958640595975,
	                        1.3902673763983182, 1.390338816603746, 1.3904101847074049, 1.3904814807407837,
	                        1.3905527047353372, 1.3906238567224853, 1.3906949367336132, 1.390765944800072,
	                        1.3908368809531777, 1.3909077452242125, 1.3909785376444241, 1.391049258245026,
	                        1.391119907057197, 1.3911904841120821, 1.391260989440793, 1.3913314230744058,
	                        1.3914017850439642, 1.3914720753804768, 1.3915422941149196, 1.3916124412782331,
	                        1.3916825169013263, 1.3917525210150723, 1.3918224536503128, 1.3918923148378544,
	                        1.3919621046084707, 1.3920318229929023, 1.392101470021856, 1.3921710457260055,
	                        1.3922405501359916, 1.3923099832824215, 1.39237934519587, 1.392448635906878,
	                        1.3925178554459539, 1.392587003843574, 1.39265608113018, 1.392725087336183,
	                        1.3927940224919597, 1.392862886627855, 1.3929316797741813, 1.3930004019612183,
	                        1.393069053219213, 1.393137633578381, 1.3932061430689042, 1.3932745817209335,
	                        1.3933429495645875, 1.393411246629952, 1.3934794729470814, 1.3935476285459978,
	                        1.3936157134566918, 1.393683727709122, 1.393751671333215, 1.3938195443588661,
	                        1.3938873468159387, 1.3939550787342645, 1.3940227401436445, 1.3940903310738473,
	                        1.3941578515546107, 1.394225301615641, 1.3942926812866134, 1.3943599905971722,
	                        1.39442722957693, 1.394494398255469, 1.3945614966623396, 1.3946285248270625,
	                        1.3946954827791265, 1.3947623705479908, 1.3948291881630825, 1.3948959356537995,
	                        1.394962613049508, 1.3950292203795445, 1.3950957576732148, 1.3951622249161388,
	                        1.3952286222685286, 1.3952949496286327, 1.3953612070692913, 1.3954273946196594,
	                        1.3954935123088616, 1.3955595601659936, 1.3956255382201201, 1.3956914465002763,
	                        1.3957572850354687, 1.3958230538546725, 1.3958887529868347, 1.3959543824608722,
	                        1.3960199423056727, 1.3960854325500944, 1.3961508532229663, 1.3962162043530884,
	                        1.3962814859692312, 1.3963466981001362, 1.3964118407745165, 1.3964769140210553,
	                        1.396541917868408, 1.3966068523452004, 1.3966717174800305, 1.3967365133014666,
	                        1.3968012398380492, 1.3968658971182901, 1.396930485170673, 1.3969950040236527,
	                        1.3970594537056567, 1.3971238342450831, 1.3971881456703028, 1.3972523880096586,
	                        1.3973165612914649, 1.3973806655440089, 1.397444700795549, 1.3975086670743173,
	                        1.3975725644085173, 1.3976363928263247, 1.3977001523558885, 1.39776384302533,
	                        1.3978274648627431, 1.3978910178961945, 1.3979545021537234, 1.3980179176633427,
	                        1.3980812644530376, 1.3981445425507664, 1.398207751984461, 1.398270892782026,
	                        1.3983339649713395, 1.3983969685802535, 1.3984599036365926, 1.3985227701681553,
	                        1.3985855682027137, 1.398648297768014, 1.3987109588917754, 1.398773551601692,
	                        1.3988360759254306, 1.398898531890633, 1.3989609195249149, 1.3990232388558659,
	                        1.3990854899110503, 1.3991476727180063, 1.3992097873042462, 1.3992718336972594,
	                        1.399333811924506, 1.3993957220134234, 1.3994575639914228, 1.399519337885891,
	                        1.3995810437241891, 1.3996426815336536, 1.3997042513415956, 1.3997657531753025,
	                        1.3998271870620353, 1.3998885530290321, 1.3999498511035058, 1.4000110813126445,
	                        1.400072243683612, 1.4001333382435488, 1.4001943650195698, 1.400255324038767,
	                        1.4003162153282074, 1.4003770389149355, 1.4004377947823143, 1.400498483088308,
	                        1.4005591037289211, 1.4006196567747586, 1.4006801422527457, 1.4007405601897849,
	                        1.4008009106127546, 1.4008611935485102, 1.4009214090238848, 1.4009815570656876,
	                        1.4010416377007053, 1.4011016509557017, 1.401161596857418, 1.4012214754325727,
	                        1.4012812867078617, 1.4013410307099587, 1.401400707465515, 1.4014603170011593,
	                        1.401519859343499, 1.4015793345191183, 1.4016387425545807, 1.4016980834764268,
	                        1.4017573573111761, 1.4018165640853266, 1.4018757038253542, 1.4019347765577135,
	                        1.401993782308838, 1.4020527211051401, 1.4021115929730104, 1.402170397938819,
	                        1.402229136028915, 1.4022878072696265, 1.402346411687261, 1.4024049493081054,
	                        1.402463420158426, 1.4025218242644693, 1.4025801616522893, 1.4026384323486036,
	                        1.4026966363790863, 1.4027547737700727, 1.4028128445477084, 1.4028708487381194,
	                        1.4029287863674114, 1.402986657461671, 1.403044462046965, 1.4031022001493418,
	                        1.403159871794829, 1.4032174770094366, 1.403275015819155, 1.4033324882499552,
	                        1.40338989432779, 1.403447234078594, 1.4035045075282822, 1.4035617147027517,
	                        1.4036188556278815, 1.4036759303295316, 1.4037329388335456, 1.403789881165747,
	                        1.4038467573519429, 1.403903567417922, 1.403960311389456, 1.4040169892922985,
	                        1.4040736011521857, 1.4041301469948373, 1.4041866268459553, 1.4042430407312247,
	                        1.4042993886763135, 1.4043556707068734, 1.4044118868485391, 1.4044680371269294,
	                        1.4045241215676454, 1.4045801401962732, 1.4046360930383823, 1.4046919801195266,
	                        1.404747801465243, 1.404803557101054, 1.4048592470524657, 1.4049148713449688,
	                        1.4049704300040389, 1.4050259230551365, 1.4050813505237063, 1.405136712435179,
	                        1.4051920088149694, 1.4052472396884788, 1.4053024050810927, 1.4053575050181837,
	                        1.4054125395251087, 1.405467508627211, 1.4055224123498202, 1.4055772507182516,
	                        1.405632023757807, 1.4056867314937749, 1.4057413739514297, 1.4057959511560332,
	                        1.4058504631328332, 1.4059049099070657, 1.4059592915039527, 1.4060136079487044,
	                        1.406067859266518, 1.4061220454825778, 1.406176166622057, 1.4062302227101156,
	                        1.4062842137719025, 1.406338139832554, 1.4063920009171955, 1.4064457970509405,
	                        1.406499528258891, 1.406553194566138, 1.406606795997762, 1.4066603325788314,
	                        1.4067138043344052, 1.4067672112895313, 1.4068205534692468, 1.406873830898579,
	                        1.4069270436025454, 1.4069801916061533, 1.4070332749343997, 1.407086293612273,
	                        1.407139247664752, 1.4071921371168057, 1.4072449619933944, 1.40729772231947,
	                        1.4073504181199745, 1.4074030494198426, 1.4074556162439997, 1.4075081186173637,
	                        1.4075605565648441, 1.4076129301113423, 1.4076652392817526, 1.4077174841009619,
	                        1.407769664593849, 1.407821780785286, 1.4078738327001383, 1.4079258203632647,
	                        1.4079777437995167, 1.4080296030337398, 1.4080813980907736, 1.4081331289954513,
	                        1.4081847957726001, 1.4082363984470427, 1.4082879370435952, 1.4083394115870687,
	                        1.4083908221022696, 1.4084421686139994, 1.408493451147055, 1.4085446697262285,
	                        1.4085958243763081, 1.408646915122078, 1.408697941988318, 1.4087489049998059,
	                        1.4087998041813141, 1.4088506395576128, 1.4089014111534692, 1.4089521189936476,
	                        1.40900276310291, 1.4090533435060157, 1.409103860227722, 1.409154313292784,
	                        1.4092047027259558, 1.4092550285519896, 1.4093052907956363, 1.409355489481646,
	                        1.4094056246347677, 1.4094556962797504, 1.4095057044413422, 1.4095556491442915,
	                        1.4096055304133464, 1.4096553482732557, 1.409705102748769, 1.409754793864636,
	                        1.4098044216456085, 1.4098539749405679, 1.4099034873018812, 1.4099529252266916,
	                        1.410002299915628, 1.410051611393451, 1.4101008596849232, 1.4101500448148108,
	                        1.4101991668078822, 1.41024822568891, 1.4102972214826694, 1.4103461542139406,
	                        1.410395023907507, 1.4104438305881568, 1.4104925742806829, 1.4105412550098824,
	                        1.410589872800559, 1.4106384276775208, 1.4106869196655814, 1.4107353487895613,
	                        1.4107837150742868, 1.410832018544591, 1.4108802592253136, 1.4109284371413016,
	                        1.4109765523174094, 1.4110246047784991, 1.4110725945494411, 1.4111205216551137,
	                        1.411168386120404, 1.4112161879702079, 1.4112639272294312, 1.411311603922988,
	                        1.4113592180758034, 1.411406769712812, 1.4114542588589587, 1.4115016855392,
	                        1.4115490497785026, 1.411596351601845, 1.4116435910342175, 1.4116907681006223,
	                        1.4117378828260745, 1.4117849352356004, 1.4118319253542417, 1.4118788532070512,
	                        1.411925718819097, 1.4119725222154602, 1.4120192634212374, 1.4120659424615392,
	                        1.4121125593614912, 1.4121591141462349, 1.4122056068409274, 1.4122520374707421,
	                        1.4122984060608692, 1.412344712636515, 1.4123909572229036, 1.412437139845277,
	                        1.4124832605288948, 1.4125293192990351, 1.412575316180995, 1.4126212512000904,
	                        1.4126671243816569, 1.4127129357510497, 1.4127586853336456, 1.4128043731548405,
	                        1.4128499992400525, 1.4128955636147207, 1.4129410663043067, 1.4129865073342938,
	                        1.4130318867301883, 1.4130772045175204, 1.4131224607218424, 1.413167655368732,
	                        1.413212788483791, 1.4132578600926458, 1.4133028702209485, 1.4133478188943767,
	                        1.4133927061386342, 1.4134375319794517, 1.4134822964425873, 1.4135269995538258,
	                        1.4135716413389807, 1.4136162218238941, 1.4136607410344364, 1.4137051989965081,
	                        1.4137495957360395, 1.413793931278991, 1.4138382056513543, 1.4138824188791521,
	                        1.4139265709884394, 1.413970662005303, 1.4140146919558632, 1.414058660866273,
	                        1.41410256876272, 1.4141464156714258, 1.4141902016186472, 1.4142339266306763,
	                        1.414277590733841, 1.4143211939545057, 1.4143647363190726, 1.4144082178539807,
	                        1.4144516385857078, 1.4144949985407693, 1.4145382977457217, 1.4145815362271594,
	                        1.414624714011719, 1.4146678311260767, 1.4147108875969512, 1.4147538834511026,
	                        1.4147968187153346, 1.4148396934164933, 1.4148825075814697, 1.414925261237199,
	                        1.4149679544106613, 1.4150105871288827, 1.4150531594189364, 1.4150956713079414,
	                        1.4151381228230655, 1.4151805139915243, 1.4152228448405824, 1.4152651153975544,
	                        1.4153073256898052, 1.4153494757447502, 1.4153915655898572, 1.4154335952526458,
	                        1.4154755647606885, 1.4155174741416126, 1.4155593234230983, 1.4156011126328822,
	                        1.4156428417987565, 1.4156845109485694, 1.4157261201102271, 1.4157676693116938,
	                        1.415809158580992, 1.4158505879462047, 1.4158919574354734, 1.4159332670770028,
	                        1.4159745168990587, 1.416015706929969, 1.4160568371981253, 1.416097907731984,
	                        1.4161389185600657, 1.4161798697109576, 1.4162207612133124, 1.416261593095852,
	                        1.4163023653873654, 1.416343078116711, 1.4163837313128171, 1.4164243250046837,
	                        1.4164648592213818, 1.4165053339920546, 1.4165457493459201, 1.4165861053122701,
	                        1.4166264019204717, 1.416666639199968, 1.41670681718028, 1.4167469358910056,
	                        1.4167869953618235, 1.4168269956224908, 1.4168669367028466, 1.416906818632811,
	                        1.4169466414423886, 1.416986405161666, 1.4170261098208161, 1.4170657554500974,
	                        1.4171053420798547, 1.417144869740522, 1.4171843384626213, 1.4172237482767653,
	                        1.4172630992136575, 1.4173023913040939, 1.4173416245789638, 1.4173807990692509,
	                        1.4174199148060342, 1.41745897182049, 1.4174979701438915, 1.4175369098076123,
	                        1.417575790843124, 1.4176146132820016, 1.417653377155921, 1.4176920824966626,
	                        1.4177307293361117, 1.4177693177062591, 1.4178078476392035, 1.4178463191671522,
	                        1.417884732322422, 1.4179230871374404, 1.4179613836447484, 1.4179996218770001,
	                        1.4180378018669644, 1.418075923647527, 1.418113987251691, 1.418151992712579,
	                        1.4181899400634332, 1.4182278293376185, 1.4182656605686224, 1.4183034337900573,
	                        1.4183411490356617, 1.4183788063393017, 1.418416405734972, 1.418453947256798,
	                        1.4184914309390375, 1.4185288568160808, 1.4185662249224544, 1.4186035352928204,
	                        1.4186407879619787, 1.418677982964872, 1.4187151203365793, 1.4187522001123263,
	                        1.4187892223274818, 1.4188261870175611, 1.4188630942182279, 1.4188999439652943,
	                        1.418936736294725, 1.4189734712426365, 1.419010148845301, 1.419046769139147,
	                        1.4190833321607608, 1.4191198379468897, 1.4191562865344423, 1.4191926779604909,
	                        1.4192290122622744, 1.4192652894771982, 1.4193015096428376, 1.4193376727969398,
	                        1.4193737789774243, 1.4194098282223873, 1.4194458205701013, 1.4194817560590183,
	                        1.419517634727772, 1.41955345661518, 1.4195892217602446, 1.4196249302021562,
	                        1.4196605819802952, 1.4196961771342342, 1.4197317157037395, 1.4197671977287745,
	                        1.4198026232495, 1.4198379923062796, 1.4198733049396786, 1.419908561190469,
	                        1.4199437610996297, 1.4199789047083509, 1.420013992058035, 1.4200490231902996,
	                        1.4200839981469802, 1.420118916926476, 1.4201537797020327, 1.4201885863851864,
	                        1.420223337062324, 1.4202580317764066, 1.4202926705706291, 1.420327253488422,
	                        1.4203617805734534, 1.420396251869634, 1.4204306674211173, 1.420465027272304,
	                        1.420499331467845, 1.4205335800526424, 1.4205677730718551, 1.4206019105708994,
	                        1.4206359925954537, 1.42067001919146, 1.420703990405129, 1.4207379062829402,
	                        1.420771766871648, 1.4208055722182835, 1.4208393223701576, 1.4208730173748647,
	                        1.4209066572802858, 1.4209402421345916, 1.420973771986247, 1.4210072468840118,
	                        1.421040666876948, 1.42107403201442, 1.4211073311702294, 1.4211405979219716,
	                        1.4211737987923314, 1.421206945007795, 1.4212400366193005, 1.4212730736781103,
	                        1.421306056235817, 1.4213389843443474, 1.4213718580559638, 1.4214046774232707,
	                        1.4214374424555627, 1.4214701533371057, 1.4215028099905849, 1.4215354125136668,
	                        1.4215679609607226, 1.421600455386491, 1.4216328958460807, 1.421665282394975,
	                        1.4216976150890372, 1.4217298939845142, 1.4217621191380407, 1.4217942906066456,
	                        1.4218264084477543, 1.421858472719196, 1.4218904834792063, 1.4219224407864335,
	                        1.4219543446999423, 1.4219861952792205, 1.4220179925841825, 1.4220497366751752,
	                        1.422081427612983, 1.4221130654588334, 1.422144650274402, 1.4221761821218173,
	                        1.422207661063668, 1.4222390871630068, 1.4222704604833567, 1.422301781088717,
	                        1.4223330490435688, 1.4223642644128804, 1.4223954272621138, 1.422426537657231,
	                        1.4224575956646999, 1.4224886013514995, 1.422519554785128, 1.4225504560336075,
	                        1.4225813051654912, 1.4226121022498706, 1.4226428473563804, 1.4226735405552062,
	                        1.422704181917093, 1.422734771513348, 1.4227653094158519, 1.4227957956970634,
	                        1.422826230430027, 1.4228566136883811, 1.4228869455463637, 1.422917226078822,
	                        1.4229474553612185, 1.4229776334696385, 1.4230077604807998, 1.4230378364720586,
	                        1.4230678615214185, 1.4230978357075383, 1.4231277591097415, 1.4231576318080221,
	                        1.4231874538830562, 1.423217225416208, 1.4232469464895403, 1.4232766171858229,
	                        1.4233062375885406, 1.423335807781904, 1.4233653278508578, 1.4233947978810906,
	                        1.4234242179590437, 1.4234535881719217, 1.4234829086077017, 1.4235121793551437,
	                        1.4235414005038003, 1.4235705721440275, 1.4235996943669942, 1.423628767264694,
	                        1.4236577909299546, 1.4236867654564496, 1.423715690938709, 1.4237445674721323,
	                        1.423773395152995, 1.4238021740784659, 1.4238309043466155, 1.4238595860564285,
	                        1.423888219307816, 1.423916804201628, 1.4239453408396647, 1.4239738293246917,
	                        1.424002269760449, 1.4240306622516674, 1.42405900690408, 1.4240873038244355,
	                        1.424115553120513, 1.4241437549011344, 1.4241719092761793, 1.4242000163565989,
	                        1.424228076254431, 1.4242560890828133, 1.4242840549559999, 1.4243119739893755,
	                        1.4243398462994712, 1.4243676720039795, 1.424395451221771, 1.4244231840729096,
	                        1.4244508706786696, 1.424478511161552, 1.424506105645301, 1.424533654254922,
	                        1.4245611571166985, 1.4245886143582087, 1.424616026108346, 1.4246433924973345,
	                        1.42467071365675, 1.4246979897195367, 1.4247252208200274, 1.424752407093963,
	                        1.424779548678512, 1.4248066345364196, 1.424833698335383, 1.4248607066893628,
	                        1.4248876709173124, 1.4249145911638463, 1.424941467575132, 1.4249683002989122,
	                        1.4249950894845267, 1.4250218352829362, 1.4250485378467455, 1.4250751973302256,
	                        1.4251018138893397, 1.4251283876817664, 1.4251549188669248, 1.4251814076059994,
	                        1.4252078540619666, 1.42523425839962, 1.4252606207855956, 1.425286941388404,
	                        1.4253132203784493, 1.4253394579280645, 1.4253656542115367, 1.4253918094051359,
	                        1.4254179236871447, 1.4254439972378876, 1.4254700302397627, 1.4254960228772708,
	                        1.425521975337048, 1.4255478878078969, 1.4255737604808196, 1.4255995935490502,
	                        1.425625387208089, 1.4256511416557363, 1.4256768570921265, 1.4257025337197655,
	                        1.4257281717435646, 1.425753771370878, 1.4257793328115407, 1.425804856277905,
	                        1.4258303419848808, 1.4258557901499733, 1.4258812009933244, 1.4259065747377526,
	                        1.4259319116087947, 1.4259572118347486, 1.425982475646714, 1.4260077032786398,
	                        1.4260328949673653, 1.4260580509526677, 1.426083171477307, 1.4261082567870733,
	                        1.426133307130835, 1.4261583227605883, 1.4261833039315044, 1.4262082509019827,
	                        1.4262331639335302, 1.4262580432916672, 1.4262828892442752, 1.426307702063357,
	                        1.4263324820242382, 1.426357229405798, 1.42638194449052, 1.4264066275645577,
	                        1.4264312789177902, 1.4264558988438847, 1.4264804876403572, 1.4265050456086377,
	                        1.426529573054132, 1.4265540702862902, 1.4265785376186715, 1.4266029753690128,
	                        1.426627383859299, 1.4266517634158333, 1.4266761143693092, 1.4267004370548844,
	                        1.4267247318122565, 1.426748998985738, 1.4267732389243362, 1.4267974519818316,
	                        1.4268216385168593, 1.4268457988929917, 1.4268699334788235, 1.4268940426480572,
	                        1.426918126779591, 1.4269421862576082, 1.4269662214716696, 1.4269902328168054,
	                        1.4270142206936107, 1.4270381855083434, 1.4270621276730222, 1.4270860476055276,
	                        1.4271099457297063, 1.4271338224754744, 1.4271576782789264, 1.427181513582444,
	                        1.4272053288348088, 1.427229124491315, 1.4272529010138875, 1.4272766588711996,
	                        1.4273003985387958, 1.427324120499215, 1.4273478252421168, 1.4273715132644116,
	                        1.4273951850703928, 1.4274188411718705, 1.4274424820883111, 1.4274661083469755,
	                        1.4274897204394101, 1.4275133190398697, 1.4275369045689112, 1.4275604776301047,
	                        1.4275840387919103, 1.427607588631495, 1.4276311277348945, 1.4276546566971828,
	                        1.4276781761226403, 1.4277016866249297, 1.4277251888272746, 1.4277486833626412,
	                        1.4277721708739244, 1.4277956520141397, 1.4278191274466172, 1.4278425978452003,
	                        1.4278660638944505, 1.4278895262898552, 1.427912985738041, 1.4279364429569914,
	                        1.4279598986762696, 1.4279833536372475, 1.428006808593338, 1.4280302643102336,
	                        1.428053721566152, 1.4280771811520845, 1.4281006438720523, 1.4281241105433684,
	                        1.4281475819969052, 1.4281710590773695, 1.4281945426435818, 1.4282180335687649,
	                        1.4282415327408369, 1.428265041062714, 1.428288559452618, 1.4283120888443916,
	                        1.4283356301878236, 1.428359184448978, 1.428382752610536, 1.428406335672141,
	                        1.4284299346507576, 1.4284535505810345, 1.4284771845156783, 1.428500837525839,
	                        1.4285245107015005, 1.428548205151884, 1.4285719220058604, 1.428595662412375,
	                        1.428619427540877, 1.4286432185817683, 1.4286670367468581, 1.4286908832698295,
	                        1.4287147594067198, 1.4287386664364121, 1.42876260566114, 1.4287865784070048,
	                        1.428810586024507, 1.4288346298890897, 1.4288587114016995, 1.4288828319893592,
	                        1.4289069931057565, 1.428931196231849, 1.4289554428764841, 1.4289797345770363,
	                        1.429004072900062, 1.4290284594419693, 1.4290528958297097, 1.4290773837214854,
	                        1.4291019248074777, 1.4291265208105932, 1.4291511734872342, 1.4291758846280853,
	                        1.4292006560589265, 1.429225489641465, 1.4292503872741928, 1.4292753508932678,
	                        1.4293003824734176, 1.4293254840288714, 1.4293506576143171, 1.4293759053258857,
	                        1.4294012293021627, 1.4294266317252324, 1.4294521148217454, 1.4294776808640244,
	                        1.4295033321711963, 1.4295290711103614, 1.4295549000977932, 1.429580821600177,
	                        1.429606838135882, 1.4296329522762727, 1.4296591666470595, 1.4296854839296889,
	                        1.4297119068627755, 1.4297384382435776, 1.4297650809295168, 1.4297918378397454,
	                        1.4298187119567598, 1.4298457063280643, 1.4298728240678855, 1.429900068358942,
	                        1.4299274424542652, 1.4299549496790818, 1.4299825934327501, 1.430010377190763,
	                        1.4300383045068095, 1.430066379014904, 1.4300946044315845, 1.43012298455818,
	                        1.4301515232831505, 1.4301802245845057, 1.4302090925322983, 1.4302381312912034,
	                        1.4302673451231787, 1.4302967383902154, 1.4303263155571797, 1.430356081194747,
	                        1.430386039982439, 1.4304161967117577, 1.4304465562894293, 1.4304771237407576,
	                        1.430507904213092, 1.4305389029794127, 1.430570125442043, 1.4306015771364886,
	                        1.4306332637354102, 1.4306651910527375, 1.430697365047925, 1.4307297918303628,
	                        1.4307624776639396, 1.4307954289717744, 1.4308286523411116, 1.4308621545284013,
	                        1.4308959424645564, 1.4309300232604079, 1.4309644042123573, 1.430999092808238,
	                        1.4310340967333968, 1.4310694127011296, 1.4311050823385787, 1.4311410804348146,
	                        1.4311774267065962, 1.4312141299263306, 1.4312511991055454, 1.4312886435027803,
	                        1.4313264726317834, 1.4313646962700353, 1.4314033244675946, 1.431442367556308,
	                        1.4314818361593749, 1.4315217412012997, 1.431562093918242, 1.431602905868787,
	                        1.4316441889451517, 1.431685955384852, 1.431728217782848, 1.4317709891041943,
	                        1.431814282697215, 1.4318581011313625, 1.4319024920908792, 1.4319474366310032,
	                        1.43199296095223, 1.4320390805371772, 1.432085811343378, 1.4321331698209405,
	                        1.4321811729309764, 1.4322298381648486, 1.4322791835642692, 1.4323292165664234,
	                        1.4323799899052658, 1.4324314898757438, 1.4324837481164852, 1.4325367857555225,
	                        1.4325906246123987, 1.4326452872256257, 1.432700796881424, 1.432757177643822,
	                        1.4328144543861794, 1.4328726528242217, 1.4329317995506639, 1.4329919220715113,
	                        1.433053048844137, 1.4331152093172295, 1.4331784339727214, 1.4332427543698119,
	                        1.4333082031912012, 1.4333748142916698, 1.433442622749137, 1.4335116649183466,
	                        1.4335819784873327, 1.4336536025368403, 1.4337265776028678, 1.4338009457425351,
	                        1.4338767506034666, 1.4339540374969206, 1.43403285347489, 1.4341132474114267,
	                        1.4341952700884597, 1.4342789742863906, 1.4343644148797785, 1.4344516489384416,
	                        1.434540735834335, 1.4346317373545847, 1.434724717821086, 1.4348197442171173,
	                        1.434916886321435, 1.4350162168503706, 1.43511781160848, 1.4352217496483433,
	                        1.4353281134401594, 1.435436989051833, 1.4355484663403055, 1.4356626391549474,
	                        1.4357796055538943, 1.4358994680342794, 1.4360223337774072, 1.4361483149099856,
	                        1.4362775287826477, 1.4364100982670855, 1.4365461520732474, 1.43668582508817,
	                        1.4368292587381644, 1.4369766013762286, 1.4371280086967295, 1.437283644179586,
	                        1.4374436795664058, 1.4376082953712424, 1.4377776814289087, 1.4379520374840675,
	                        1.4381315738246283, 1.4383165119633379]
	                      datatype: float64
	                      shape: [1101]
	                  inputs: [alpha_in, beta_in, alpha_out]
	                  outputs: [y]
	                - forward:
	                  - inputs: [x0]
	                    outputs: [x0]
	                  - dimensions: 1
	                    inputs: [x]
	                    name: velocity_correction
	                    outputs: [y]
	                    value: 1.0000046645487086
	                  inputs: [x0]
	                  inverse:
	                    forward:
	                    - inputs: [x0]
	                      outputs: [x0]
	                    - dimensions: 1
	                      inputs: [x]
	                      name: inv_vel_correction
	                      outputs: [y]
	                      value: 1.0000046645487086
	                    inputs: [x0]
	                    outputs: [x0]
	                  outputs: [x0]
	                inputs: [alpha_in, beta_in, alpha_out]
	                outputs: [x0]
	              inputs: [x0, x1, alpha_in, beta_in, alpha_out]
	              outputs: [x00, x10, x01]
	            inputs: [x0, x1, x2]
	            outputs: [x00, x10, x01]
	          - forward:
	            - forward:
	              - forward:
	                - inputs: [x0, x1, x2]
	                  mapping: [0, 1, 2, 1]
	                  outputs: [x0, x1, x2, x3]
	                - forward:
	                  - inputs: [x0, x1, x2]
	                    n_dims: 3
	                    outputs: [x0, x1, x2]
	                  - forward:
	                    - forward:
	                      - compareto: 0.55
	                        condition: GT
	                        inputs: [x]
	                        outputs: [x]
	                        value: .nan
	                      - compareto: -0.55
	                        condition: LT
	                        inputs: [x]
	                        outputs: [x]
	                        value: .nan
	                      inputs: [x]
	                      outputs: [x]
	                    - factor: 0.0
	                      inputs: [x]
	                      outputs: [y]
	                    inputs: [x]
	                    outputs: [y]
	                  inputs: [x0, x1, x2, x]
	                  outputs: [x0, x1, x2, y]
	                inputs: [x0, x1, x2]
	                outputs: [x0, x1, x2, y]
	              - inputs: [x0, x1, x2, x3]
	                mapping: [0, 1, 3, 2, 3]
	                outputs: [x0, x1, x2, x3, x4]
	              inputs: [x0, x1, x2]
	              outputs: [x0, x1, x2, x3, x4]
	            - forward:
	              - forward:
	                - inputs: [x0]
	                  outputs: [x0]
	                - forward:
	                  - inputs: [x0, x1]
	                    mapping: [0]
	                    n_inputs: 2
	                    outputs: [x0]
	                  - inputs: [x0, x1]
	                    mapping: [1]
	                    outputs: [x0]
	                  inputs: [x0, x1]
	                  outputs: [x0]
	                inputs: [x00, x01, x11]
	                outputs: [x00, x01]
	              - forward:
	                - inputs: [x0, x1]
	                  mapping: [0]
	                  n_inputs: 2
	                  outputs: [x0]
	                - inputs: [x0, x1]
	                  mapping: [1]
	                  outputs: [x0]
	                inputs: [x0, x1]
	                outputs: [x0]
	              inputs: [x00, x01, x11, x0, x1]
	              outputs: [x00, x01, x0]
	            inputs: [x0, x1, x2]
	            inverse:
	              inputs: [x0, x1, x2]
	              n_dims: 3
	              outputs: [x0, x1, x2]
	            outputs: [x00, x01, x0]
	          inputs: [x0, x1, x2]
	          inverse:
	            forward:
	            - forward:
	              - forward:
	                - forward:
	                  - forward:
	                    - forward:
	                      - forward:
	                        - factor: 8.135000098263845e-05
	                          inputs: [x]
	                          outputs: [y]
	                        - factor: 0.001271169981919229
	                          inputs: [x]
	                          outputs: [y]
	                        inputs: [x0, x1]
	                        outputs: [y0, y1]
	                      - forward:
	                        - inputs: [x]
	                          offset: 0.02697242796421051
	                          outputs: [y]
	                        - inputs: [x]
	                          offset: -0.0027167024090886116
	                          outputs: [y]
	                        inputs: [x0, x1]
	                        outputs: [y0, y1]
	                      inputs: [x0, x1]
	                      outputs: [y0, y1]
	                    - forward:
	                      - angle: 0.0
	                        inputs: [x, y]
	                        name: msa_slit_rot
	                        outputs: [x, y]
	                      - forward:
	                        - inputs: [x]
	                          name: msa_slit_x
	                          offset: 0.0
	                          outputs: [y]
	                        - inputs: [x]
	                          name: msa_slit_y
	                          offset: 0.0
	                          outputs: [y]
	                        inputs: [x0, x1]
	                        outputs: [y0, y1]
	                      inputs: [x, y]
	                      outputs: [y0, y1]
	                    inputs: [x0, x1]
	                    outputs: [y0, y1]
	                  - forward:
	                    - forward:
	                      - forward:
	                        - forward:
	                          - forward:
	                            - inputs: [x]
	                              name: collimator_xoutcen_d2s
	                              offset: -5.526841e-06
	                              outputs: [y]
	                            - inputs: [x]
	                              name: collimator_youtcen_d2s
	                              offset: 0.000346042594
	                              outputs: [y]
	                            inputs: [x0, x1]
	                            outputs: [y0, y1]
	                          - forward:
	                            - inputs: [x, y]
	                              matrix: !core/ndarray-1.0.0
	                                data:
	                                - [1.5738000900266444, -0.0003450858994488455]
	                                - [0.0003613242773258282, 1.6478568990863562]
	                                datatype: float64
	                                shape: [2, 2]
	                              outputs: [x, y]
	                              translation: !core/ndarray-1.0.0
	                                data: [-0.0, -0.0]
	                                datatype: float64
	                                shape: [2]
	                            - forward:
	                              - inputs: [x]
	                                name: collimator_xincen_d2s
	                                offset: -0.000143900694035
	                                outputs: [y]
	                              - inputs: [x]
	                                name: collimator_yincen_d2s
	                                offset: -0.293605933112
	                                outputs: [y]
	                              inputs: [x0, x1]
	                              outputs: [y0, y1]
	                            inputs: [x, y]
	                            outputs: [y0, y1]
	                          inputs: [x0, x1]
	                          outputs: [y0, y1]
	                        - forward:
	                          - inputs: [x0, x1]
	                            mapping: [0, 1, 0, 1]
	                            outputs: [x0, x1, x2, x3]
	                          - forward:
	                            - forward:
	                              - coefficients: !core/ndarray-1.0.0
	                                  data:
	                                  - [0.00315706857764, 0.0420481492132, 0.146561534708,
	                                    0.221234162225, -0.0638619162952, -0.331781237202]
	                                  - [0.97396666617, -0.0712861999102, -0.269895805765,
	                                    -1.47821209943, -1.39521612319, 0.0]
	                                  - [-0.118219958126, -1.31400145373, -4.65546710314,
	                                    -5.31391588021, 0.0, 0.0]
	                                  - [-0.239124508069, -3.50159180727, -5.630240651, 0.0,
	                                    0.0, 0.0]
	                                  - [-0.721331930443, -2.52317345608, 0.0, 0.0, 0.0, 0.0]
	                                  - [-2.3223320496, 0.0, 0.0, 0.0, 0.0, 0.0]
	                                  datatype: float64
	                                  shape: [6, 6]
	                                domain:
	                                - [-1, 1]
	                                - [-1, 1]
	                                inputs: [x, y]
	                                name: collimator_x_backward
	                                outputs: [z]
	                                window:
	                                - [-1, 1]
	                                - [-1, 1]
	                              - coefficients: !core/ndarray-1.0.0
	                                  data:
	                                  - [-0.0027844382203, 1.15424678352, 1.57319737586, 5.65896061653,
	                                    9.03612184177, 5.89461390043]
	                                  - [-0.0251292730268, -0.24879556703, -1.30121421745,
	                                    -2.9831373654, -2.54562283395, 0.0]
	                                  - [0.0636250257988, 0.751936718567, 1.24472156622, 1.15635544547,
	                                    0.0, 0.0]
	                                  - [0.071044569667, 0.193723423502, -0.0496714084349,
	                                    0.0, 0.0, 0.0]
	                                  - [-2.00363215516, -6.67916820283, 0.0, 0.0, 0.0, 0.0]
	                                  - [-1.39018616912, 0.0, 0.0, 0.0, 0.0, 0.0]
	                                  datatype: float64
	                                  shape: [6, 6]
	                                domain:
	                                - [-1, 1]
	                                - [-1, 1]
	                                inputs: [x, y]
	                                name: collimator_y_backward
	                                outputs: [z]
	                                window:
	                                - [-1, 1]
	                                - [-1, 1]
	                              inputs: [x0, y0, x1, y1]
	                              outputs: [z0, z1]
	                            - inputs: [x0, x1]
	                              n_dims: 2
	                              outputs: [x0, x1]
	                            inputs: [x0, y0, x1, y1]
	                            outputs: [x0, x1]
	                          inputs: [x0, x1]
	                          outputs: [x0, x1]
	                        inputs: [x0, x1]
	                        outputs: [x0, x1]
	                      - inputs: [x, y]
	                        model_type: unitless2directional
	                        name: unitless2directional_cosines
	                        outputs: [x, y, z]
	                      inputs: [x0, x1]
	                      outputs: [x, y, z]
	                    - angles: [0.03333072666861111, -0.27547251631138886, -0.14198882781777777,
	                        24.29]
	                      axes_order: xyzy
	                      inputs: [x, y, z]
	                      name: rotation
	                      outputs: [x, y, z]
	                    inputs: [x0, x1]
	                    outputs: [x, y, z]
	                  inputs: [x0, x1]
	                  outputs: [x, y, z]
	                - inputs: [x0]
	                  outputs: [x0]
	                inputs: [x00, x10, x01]
	                outputs: [x, y, z, x0]
	              - inputs: [x0, x1, x2, x3]
	                mapping: [3, 0, 1, 2]
	                outputs: [x0, x1, x2, x3]
	              inputs: [x00, x10, x01]
	              outputs: [x0, x1, x2, x3]
	            - inputs: [lam, alpha_in, beta_in, zin]
	              kcoef: [0.58339748, 0.46085267, 3.8915394]
	              lcoef: [0.00252643, 0.010078333, 1200.556]
	              name: snell_law
	              outputs: [alpha_out, beta_out, zout]
	              pressure: 0.0
	              prism_angle: -16.5
	              ref_pressure: 0.0
	              ref_temp: 35.0
	              tcoef: [-2.66e-05, 0.0, 0.0, 0.0, 0.0, 0.0]
	              temp: 40.28447479156018
	            inputs: [x00, x10, x01]
	            outputs: [alpha_out, beta_out, zout]
	          outputs: [x00, x01, x0]
	  - frame:
	          frames:
	          - axes_names: [x_slit, y_slit]
	            axes_order: [0, 1]
	            axis_physical_types: ['custom:x_slit', 'custom:y_slit']
	            name: slit_spatial
	            unit: ['', '']
	          - axes_names: [wavelength]
	            axes_order: [2]
	            axis_physical_types: [em.wl]
	            name: spectral
	            unit: [um]
	          name: slit_frame
	        transform:
	          forward:
	          - forward:
	            - forward:
	              - forward:
	                - factor: 8.135000098263845e-05
	                  inputs: [x]
	                  outputs: [y]
	                - factor: 0.001271169981919229
	                  inputs: [x]
	                  outputs: [y]
	                inputs: [x0, x1]
	                outputs: [y0, y1]
	              - forward:
	                - inputs: [x]
	                  offset: 0.02697242796421051
	                  outputs: [y]
	                - inputs: [x]
	                  offset: -0.0027167024090886116
	                  outputs: [y]
	                inputs: [x0, x1]
	                outputs: [y0, y1]
	              inputs: [x0, x1]
	              outputs: [y0, y1]
	            - forward:
	              - angle: 0.0
	                inputs: [x, y]
	                name: msa_slit_rot
	                outputs: [x, y]
	              - forward:
	                - inputs: [x]
	                  name: msa_slit_x
	                  offset: 0.0
	                  outputs: [y]
	                - inputs: [x]
	                  name: msa_slit_y
	                  offset: 0.0
	                  outputs: [y]
	                inputs: [x0, x1]
	                outputs: [y0, y1]
	              inputs: [x, y]
	              outputs: [y0, y1]
	            inputs: [x0, x1]
	            outputs: [y0, y1]
	          - inputs: [x0]
	            outputs: [x0]
	          inputs: [x00, x10, x01]
	          outputs: [y0, y1, x0]
	  - frame:
	          frames:
	          - axes_names: [x_msa, y_msa]
	            axes_order: [0, 1]
	            axis_physical_types: ['custom:x_msa', 'custom:y_msa']
	            name: msa_spatial
	            unit: [m, m]
	          - axes_names: [wavelength]
	            axes_order: [2]
	            axis_physical_types: [em.wl]
	            name: spectral
	            unit: [um]
	          name: msa_frame
	        transform:
	          forward:
	          - inputs: [x0, x1, x2]
	            inverse:
	              inputs: [x0, x1, x2]
	              n_dims: 3
	              outputs: [x0, x1, x2]
	            mapping: [0, 1, 2, 2]
	            name: msa2fore_mapping
	            outputs: [x0, x1, x2, x3]
	          - forward:
	            - forward:
	              - forward:
	                - forward:
	                  - inputs: [x0, x1, x2]
	                    inverse:
	                      inputs: [x0, x1]
	                      n_dims: 2
	                      outputs: [x0, x1]
	                    mapping: [0, 1, 2, 0, 1, 2]
	                    name: fore_inmap
	                    outputs: [x0, x1, x2, x3, x4, x5]
	                  - forward:
	                    - forward:
	                      - forward:
	                        - inputs: [x0, x1, x2]
	                          mapping: [0, 1]
	                          n_inputs: 3
	                          outputs: [x0, x1]
	                        - coefficients: !core/ndarray-1.0.0
	                            data:
	                            - [4.55819106794e-08, 6.23619032826e-05, -0.000282299247,
	                              -0.000631742247333, 0.000714016313313, -0.00042941754263395835]
	                            - [0.999866353285, -0.132633165827, 0.504120290122, 2.18051201288,
	                              -4.17683201854, 0.0]
	                            - [-0.000809153440408, -0.00228738952833, 0.00973715022155,
	                              0.00438541009029, 0.0, 0.0]
	                            - [0.748631703155, 2.21832075046, -9.49582591395, 0.0, 0.0,
	                              0.0]
	                            - [0.00903959322754, -0.0113638922414, 0.0, 0.0, 0.0, 0.0]
	                            - [-5.06051552119, 0.0, 0.0, 0.0, 0.0, 0.0]
	                            datatype: float64
	                            shape: [6, 6]
	                          domain:
	                          - [-1, 1]
	                          - [-1, 1]
	                          inputs: [x, y]
	                          name: fore_x_forw
	                          outputs: [z]
	                          window:
	                          - [-1, 1]
	                          - [-1, 1]
	                        inputs: [x0, x1, x2]
	                        outputs: [z]
	                      - forward:
	                        - forward:
	                          - inputs: [x0, x1, x2]
	                            mapping: [0, 1]
	                            n_inputs: 3
	                            outputs: [x0, x1]
	                          - coefficients: !core/ndarray-1.0.0
	                              data:
	                              - [-0.00855546728172, 0.00451991693631, -0.0235634063859,
	                                1.00379617731, -18.0427471461, -2607.67742719]
	                              - [31.3147747469, -4.73852534706, 107.489903983, 254.078230576,
	                                2875.63180287, 0.0]
	                              - [-0.0147433862693, -12.1508556288, 6.6321453445, 4812.85147845,
	                                0.0, 0.0]
	                              - [133.161173017, 307.60674072, -4338.10034721, 0.0, 0.0,
	                                0.0]
	                              - [-26.8566134723, 4693.47133157, 0.0, 0.0, 0.0, 0.0]
	                              - [-1810.34956166, 0.0, 0.0, 0.0, 0.0, 0.0]
	                              datatype: float64
	                              shape: [6, 6]
	                            domain:
	                            - [-1, 1]
	                            - [-1, 1]
	                            inputs: [x, y]
	                            name: fore_x_forwdist
	                            outputs: [z]
	                            window:
	                            - [-1, 1]
	                            - [-1, 1]
	                          inputs: [x0, x1, x2]
	                          outputs: [z]
	                        - forward:
	                          - inputs: [x0, x1, x2]
	                            mapping: [2]
	                            outputs: [x0]
	                          - inputs: [x0]
	                            outputs: [x0]
	                          inputs: [x0, x1, x2]
	                          outputs: [x0]
	                        inputs: [x0, x1, x2]
	                        outputs: [z]
	                      inputs: [x0, x1, x2]
	                      outputs: [z]
	                    - forward:
	                      - forward:
	                        - inputs: [x0, x1, x2]
	                          mapping: [0, 1]
	                          n_inputs: 3
	                          outputs: [x0, x1]
	                        - coefficients: !core/ndarray-1.0.0
	                            data:
	                            - [-6.33302645273e-06, 0.999780523522, -0.18480945435, 0.2680078398082627,
	                              2.58830935337, -3.22936503048]
	                            - [6.07022946565e-05, -0.000275277769323, -0.00232228407584,
	                              0.00558258729073, 0.0172787409124, 0.0]
	                            - [-0.0793290031791, 0.506138759018, 3.170326308, -8.12775001503,
	                              0.0, 0.0]
	                            - [-0.00079573977489, 0.00711145948387, -0.0022117482832,
	                              0.0, 0.0, 0.0]
	                            - [0.523300441482, -4.61944623655, 0.0, 0.0, 0.0, 0.0]
	                            - [0.0145628777029, 0.0, 0.0, 0.0, 0.0, 0.0]
	                            datatype: float64
	                            shape: [6, 6]
	                          domain:
	                          - [-1, 1]
	                          - [-1, 1]
	                          inputs: [x, y]
	                          name: fore_y_forw
	                          outputs: [z]
	                          window:
	                          - [-1, 1]
	                          - [-1, 1]
	                        inputs: [x0, x1, x2]
	                        outputs: [z]
	                      - forward:
	                        - forward:
	                          - inputs: [x0, x1, x2]
	                            mapping: [0, 1]
	                            n_inputs: 3
	                            outputs: [x0, x1]
	                          - coefficients: !core/ndarray-1.0.0
	                              data:
	                              - [2.25517158841, 32.6740998767, -19.0844022236, 94.8850881408,
	                                400.664896457, -591.290809599]
	                              - [0.009767581204, -0.00812537176791, -8.85503568824, -15.9801791748,
	                                2427.42627709, 0.0]
	                              - [-14.008847462, 109.984254426, 392.951037662, -1014.48068072,
	                                0.0, 0.0]
	                              - [2.50986442906, 3.2940167523, 3073.30289884, 0.0, 0.0,
	                                0.0]
	                              - [82.1274376308, 1173.28853135, 0.0, 0.0, 0.0, 0.0]
	                              - [-1699.96279356, 0.0, 0.0, 0.0, 0.0, 0.0]
	                              datatype: float64
	                              shape: [6, 6]
	                            domain:
	                            - [-1, 1]
	                            - [-1, 1]
	                            inputs: [x, y]
	                            name: fore_y_forwdist
	                            outputs: [z]
	                            window:
	                            - [-1, 1]
	                            - [-1, 1]
	                          inputs: [x0, x1, x2]
	                          outputs: [z]
	                        - forward:
	                          - inputs: [x0, x1, x2]
	                            mapping: [2]
	                            outputs: [x0]
	                          - inputs: [x0]
	                            outputs: [x0]
	                          inputs: [x0, x1, x2]
	                          outputs: [x0]
	                        inputs: [x0, x1, x2]
	                        outputs: [z]
	                      inputs: [x0, x1, x2]
	                      outputs: [z]
	                    inputs: [x00, x10, x20, x01, x11, x21]
	                    outputs: [z0, z1]
	                  inputs: [x0, x1, x2]
	                  outputs: [z0, z1]
	                - inputs: [x0, x1]
	                  inverse:
	                    inputs: [x0, x1, x2]
	                    mapping: [0, 1, 2, 0, 1, 2]
	                    outputs: [x0, x1, x2, x3, x4, x5]
	                  n_dims: 2
	                  name: fore_outmap
	                  outputs: [x0, x1]
	                inputs: [x0, x1, x2]
	                outputs: [x0, x1]
	              - forward:
	                - forward:
	                  - forward:
	                    - inputs: [x]
	                      name: fore_xincen_d2s
	                      offset: -5.52684591413e-06
	                      outputs: [y]
	                    - inputs: [x]
	                      name: fore_yincen_d2s
	                      offset: 0.000346028872881
	                      outputs: [y]
	                    inputs: [x0, x1]
	                    outputs: [y0, y1]
	                  - inputs: [x, y]
	                    matrix: !core/ndarray-1.0.0
	                      data:
	                      - [1.2220915421500227, 1.0903057003074268]
	                      - [-1.0735202835447986, 1.2411999988524163]
	                      datatype: float64
	                      shape: [2, 2]
	                    name: fore_affine_d2s
	                    outputs: [x, y]
	                    translation: !core/ndarray-1.0.0
	                      data: [0.0, 0.0]
	                      datatype: float64
	                      shape: [2]
	                  inputs: [x0, x1]
	                  outputs: [x, y]
	                - forward:
	                  - inputs: [x]
	                    name: fore_xoutcen_d2s
	                    offset: -2.27962e-07
	                    outputs: [y]
	                  - inputs: [x]
	                    name: fore_youtcen_d2s
	                    offset: -2.6094e-07
	                    outputs: [y]
	                  inputs: [x0, x1]
	                  outputs: [y0, y1]
	                inputs: [x0, x1]
	                outputs: [y0, y1]
	              inputs: [x0, x1, x2]
	              inverse:
	                forward:
	                - forward:
	                  - forward:
	                    - forward:
	                      - inputs: [x]
	                        name: fore_xoutcen_d2s
	                        offset: 2.27962e-07
	                        outputs: [y]
	                      - inputs: [x]
	                        name: fore_youtcen_d2s
	                        offset: 2.6094e-07
	                        outputs: [y]
	                      inputs: [x0, x1]
	                      outputs: [y0, y1]
	                    - forward:
	                      - inputs: [x, y]
	                        matrix: !core/ndarray-1.0.0
	                          data:
	                          - [0.46187188295494974, -0.40572151729222183]
	                          - [0.39947537480631573, 0.45476129732364223]
	                          datatype: float64
	                          shape: [2, 2]
	                        outputs: [x, y]
	                        translation: !core/ndarray-1.0.0
	                          data: [-0.0, -0.0]
	                          datatype: float64
	                          shape: [2]
	                      - forward:
	                        - inputs: [x]
	                          name: fore_xincen_d2s
	                          offset: 5.52684591413e-06
	                          outputs: [y]
	                        - inputs: [x]
	                          name: fore_yincen_d2s
	                          offset: -0.000346028872881
	                          outputs: [y]
	                        inputs: [x0, x1]
	                        outputs: [y0, y1]
	                      inputs: [x, y]
	                      outputs: [y0, y1]
	                    inputs: [x0, x1]
	                    outputs: [y0, y1]
	                  - inputs: [x0]
	                    outputs: [x0]
	                  inputs: [x00, x10, x01]
	                  outputs: [y0, y1, x0]
	                - forward:
	                  - inputs: [x0, x1, x2]
	                    mapping: [0, 1, 2, 0, 1, 2]
	                    outputs: [x0, x1, x2, x3, x4, x5]
	                  - forward:
	                    - forward:
	                      - forward:
	                        - forward:
	                          - inputs: [x0, x1, x2]
	                            mapping: [0, 1]
	                            n_inputs: 3
	                            outputs: [x0, x1]
	                          - coefficients: !core/ndarray-1.0.0
	                              data:
	                              - [-4.59683581533e-08, -6.23862193618e-05, 0.000262709529919,
	                                0.000813736773542, -0.000198507491199, -0.00325758985474]
	                              - [1.00013446301, 0.132639144761, -0.462377307853, -2.47095912856,
	                                2.41573925915, 0.0]
	                              - [0.000796436989742, 0.00291069858529, -0.00886502760205,
	                                -0.0337650893157, 0.0, 0.0]
	                              - [-0.738393033891, -2.68813170589, 8.5685448895, 0.0, 0.0,
	                                0.0]
	                              - [-0.0112956273819, -0.021375841859, 0.0, 0.0, 0.0, 0.0]
	                              - [6.33221344425, 0.0, 0.0, 0.0, 0.0, 0.0]
	                              datatype: float64
	                              shape: [6, 6]
	                            domain:
	                            - [-1, 1]
	                            - [-1, 1]
	                            inputs: [x, y]
	                            name: fore_x_back
	                            outputs: [z]
	                            window:
	                            - [-1, 1]
	                            - [-1, 1]
	                          inputs: [x0, x1, x2]
	                          outputs: [z]
	                        - forward:
	                          - forward:
	                            - inputs: [x0, x1, x2]
	                              mapping: [0, 1]
	                              n_inputs: 3
	                              outputs: [x0, x1]
	                            - coefficients: !core/ndarray-1.0.0
	                                data:
	                                - [0.0086961062949, -0.000457741183615, -0.0120943187567,
	                                  -1.35979599425, 21.1547885182, 2742.50168102]
	                                - [-31.6167785391, -5.82122718896, -27.9113224519, 35.2306321028,
	                                  -3636.88854096, 0.0]
	                                - [-0.0823136997939, 11.6906110754, -10.945392657016406,
	                                  -4727.85819451, 0.0, 0.0]
	                                - [-32.4725737036, -18.4912345113, 2673.62555281, 0.0,
	                                  0.0, 0.0]
	                                - [25.7420163245, -4628.24310354, 0.0, 0.0, 0.0, 0.0]
	                                - [967.947977969, 0.0, 0.0, 0.0, 0.0, 0.0]
	                                datatype: float64
	                                shape: [6, 6]
	                              domain:
	                              - [-1, 1]
	                              - [-1, 1]
	                              inputs: [x, y]
	                              name: fore_x_backdist
	                              outputs: [z]
	                              window:
	                              - [-1, 1]
	                              - [-1, 1]
	                            inputs: [x0, x1, x2]
	                            outputs: [z]
	                          - forward:
	                            - inputs: [x0, x1, x2]
	                              mapping: [2]
	                              outputs: [x0]
	                            - inputs: [x0]
	                              outputs: [x0]
	                            inputs: [x0, x1, x2]
	                            outputs: [x0]
	                          inputs: [x0, x1, x2]
	                          outputs: [z]
	                        inputs: [x0, x1, x2]
	                        outputs: [z]
	                      - forward:
	                        - forward:
	                          - inputs: [x0, x1, x2]
	                            mapping: [0, 1]
	                            n_inputs: 3
	                            outputs: [x0, x1]
	                          - coefficients: !core/ndarray-1.0.0
	                              data:
	                              - [6.34181883561e-06, 1.00022187549, 0.184892053451, -0.200015976585,
	                                -2.77753712992, 0.499707098413]
	                              - [-6.07288275693e-05, 0.000235188842841, 0.00267298878817,
	                                -0.00286493926294, -0.0328670662616, 0.0]
	                              - [0.0793358174674, -0.456157433492, -3.25735250533, 4.49490906756,
	                                0.0, 0.0]
	                              - [0.00101710927918, -0.00613966494228, -0.0265062459604,
	                                0.0, 0.0, 0.0]
	                              - [-0.660163145183, 4.0819772408, 0.0, 0.0, 0.0, 0.0]
	                              - [-0.0211890707376, 0.0, 0.0, 0.0, 0.0, 0.0]
	                              datatype: float64
	                              shape: [6, 6]
	                            domain:
	                            - [-1, 1]
	                            - [-1, 1]
	                            inputs: [x, y]
	                            name: fore_y_back
	                            outputs: [z]
	                            window:
	                            - [-1, 1]
	                            - [-1, 1]
	                          inputs: [x0, x1, x2]
	                          outputs: [z]
	                        - forward:
	                          - forward:
	                            - inputs: [x0, x1, x2]
	                              mapping: [0, 1]
	                              n_inputs: 3
	                              outputs: [x0, x1]
	                            - coefficients: !core/ndarray-1.0.0
	                                data:
	                                - [-2.25690382596, -33.5166929274, 2.31417073341, -31.7993007505,
	                                  -41.830326223, -50.1448028984]
	                                - [-0.00502907212753, -0.0366151830915, 8.50874228836,
	                                  14.3999141523, -2409.34590007, 0.0]
	                                - [7.47429907553, -23.9615970906, 10.2676948181, -576.200861728,
	                                  0.0, 0.0]
	                                - [-2.64578984778, -4.38727355933, -3038.26376392, 0.0,
	                                  0.0, 0.0]
	                                - [-35.0397745619, -2043.04951689, 0.0, 0.0, 0.0, 0.0]
	                                - [1717.81018701, 0.0, 0.0, 0.0, 0.0, 0.0]
	                                datatype: float64
	                                shape: [6, 6]
	                              domain:
	                              - [-1, 1]
	                              - [-1, 1]
	                              inputs: [x, y]
	                              name: fore_y_backdist
	                              outputs: [z]
	                              window:
	                              - [-1, 1]
	                              - [-1, 1]
	                            inputs: [x0, x1, x2]
	                            outputs: [z]
	                          - forward:
	                            - inputs: [x0, x1, x2]
	                              mapping: [2]
	                              outputs: [x0]
	                            - inputs: [x0]
	                              outputs: [x0]
	                            inputs: [x0, x1, x2]
	                            outputs: [x0]
	                          inputs: [x0, x1, x2]
	                          outputs: [z]
	                        inputs: [x0, x1, x2]
	                        outputs: [z]
	                      inputs: [x00, x10, x20, x01, x11, x21]
	                      outputs: [z0, z1]
	                    - inputs: [x0, x1]
	                      n_dims: 2
	                      outputs: [x0, x1]
	                    inputs: [x00, x10, x20, x01, x11, x21]
	                    outputs: [x0, x1]
	                  inputs: [x0, x1, x2]
	                  outputs: [x0, x1]
	                inputs: [x00, x10, x01]
	                outputs: [x0, x1]
	              outputs: [y0, y1]
	            - inputs: [x0]
	              outputs: [x0]
	            inputs: [x00, x10, x20, x01]
	            outputs: [y0, y1, x0]
	          inputs: [x0, x1, x2]
	          name: msa2oteip
	          outputs: [y0, y1, x0]
	  - frame:
	          frames:
	          - axes_names: [X_OTEIP, Y_OTEIP]
	            axes_order: [0, 1]
	            axis_physical_types: ['custom:X_OTEIP', 'custom:Y_OTEIP']
	            name: oteip
	            unit: [deg, deg]
	          - axes_names: [wavelength]
	            axes_order: [2]
	            axis_physical_types: [em.wl]
	            name: spectral
	            unit: [um]
	          name: oteip
	        transform:
	          forward:
	          - inputs: [x0, x1, x2]
	            inverse:
	              inputs: [x0, x1, x2]
	              mapping: [0, 1, 2, 2]
	              outputs: [x0, x1, x2, x3]
	            n_dims: 3
	            name: fore2ote_mapping
	            outputs: [x0, x1, x2]
	          - forward:
	            - forward:
	              - forward:
	                - forward:
	                  - forward:
	                    - inputs: [x0, x1]
	                      inverse:
	                        inputs: [x0, x1]
	                        n_dims: 2
	                        outputs: [x0, x1]
	                      mapping: [0, 1, 0, 1]
	                      name: ote_inmap
	                      outputs: [x0, x1, x2, x3]
	                    - forward:
	                      - coefficients: !core/ndarray-1.0.0
	                          data:
	                          - [5.887771163122296e-12, 0.001181872741672517, 0.004381098668339454,
	                            -0.0001030262661602804, -0.0001621139395115915, 0.002239212905820631]
	                          - [1.000010045405057, -0.0198948306438118, -0.01794430688622102,
	                            0.0007702083937083082, 0.005557741128844107, 0.0]
	                          - [-0.01301049927765961, 0.0007263338972753573, 0.0002815687482939574,
	                            -0.01831970054423948, 0.0, 0.0]
	                          - [-0.0180620155086792, 0.0007213604156713583, -0.0121120145515583,
	                            0.0, 0.0, 0.0]
	                          - [0.0004148141635435355, -0.009560875485648879, 0.0, 0.0, 0.0,
	                            0.0]
	                          - [0.0005642557705480833, 0.0, 0.0, 0.0, 0.0, 0.0]
	                          datatype: float64
	                          shape: [6, 6]
	                        domain:
	                        - [-1, 1]
	                        - [-1, 1]
	                        inputs: [x, y]
	                        inverse:
	                          coefficients: !core/ndarray-1.0.0
	                            data:
	                            - [-6.764695218042509e-12, -0.001181845439725266, -0.004421942263252729,
	                              -0.00015786783293089, -8.101344739691103e-05, -0.00223528778759241]
	                            - [0.9999913490363831, 0.01985321796323403, 0.01836568669267155,
	                              0.0008476551557795109, -0.004583374077531843, 0.0]
	                            - [0.01299262834412884, 0.0003111090225528627, 0.0006487537603577787,
	                              0.01831138758726425, 0.0, 0.0]
	                            - [0.01829981571350813, 0.0008920392687617073, 0.01402112565533553,
	                              0.0, 0.0, 0.0]
	                            - [0.0007670516309624491, 0.009553952594322013, 0.0, 0.0,
	                              0.0, 0.0]
	                            - [0.0004297874598029328, 0.0, 0.0, 0.0, 0.0, 0.0]
	                            datatype: float64
	                            shape: [6, 6]
	                          domain:
	                          - [-1, 1]
	                          - [-1, 1]
	                          inputs: [x, y]
	                          name: ote_x_back
	                          outputs: [z]
	                          window:
	                          - [-1, 1]
	                          - [-1, 1]
	                        name: ote_x_forw
	                        outputs: [z]
	                        window:
	                        - [-1, 1]
	                        - [-1, 1]
	                      - coefficients: !core/ndarray-1.0.0
	                          data:
	                          - [6.116356498319719e-10, 1.00001443535109, -0.01483102426265499,
	                            -0.01798861538665233, 0.0004181691144437354, 0.0007218197164462481]
	                          - [0.001179828191001408, -0.01747569411172255, 0.0006738474640055891,
	                            0.0005905203096196509, 0.001516029056689128, 0.0]
	                          - [0.005008481986839269, -0.01810347472067562, 0.0001467468673671769,
	                            0.01135524127936449, 0.0, 0.0]
	                          - [-9.635229102552406e-05, 0.000606605966116013, -0.0029344770316033215,
	                            0.0, 0.0, 0.0]
	                          - [-0.0002147771257013271, 0.008775038955027625, 0.0, 0.0, 0.0,
	                            0.0]
	                          - [3.639541372213451e-05, 0.0, 0.0, 0.0, 0.0, 0.0]
	                          datatype: float64
	                          shape: [6, 6]
	                        domain:
	                        - [-1, 1]
	                        - [-1, 1]
	                        inputs: [x, y]
	                        inverse:
	                          coefficients: !core/ndarray-1.0.0
	                            data:
	                            - [-6.115026307974087e-10, 0.9999869591970522, 0.01481498123407221,
	                              0.0183504542886734, 0.0009281781374761249, 0.0002958977074083435]
	                            - [-0.001179801005691817, 0.01742851464230401, 0.0003646472234911172,
	                              0.0008189728816032273, -0.001483363225125878, 0.0]
	                            - [-0.005044248520802729, 0.01828606785677567, 0.0009112362883588386,
	                              -0.009363002229886064, 0.0, 0.0]
	                            - [-0.0001649665026415299, 0.00081286189015533, 0.002997688363766571,
	                              0.0, 0.0, 0.0]
	                            - [-6.145531569187097e-05, -0.007795457001419592, 0.0, 0.0,
	                              0.0, 0.0]
	                            - [-2.764978182656641e-05, 0.0, 0.0, 0.0, 0.0, 0.0]
	                            datatype: float64
	                            shape: [6, 6]
	                          domain:
	                          - [-1, 1]
	                          - [-1, 1]
	                          inputs: [x, y]
	                          name: ote_y_backw
	                          outputs: [z]
	                          window:
	                          - [-1, 1]
	                          - [-1, 1]
	                        name: ote_y_forw
	                        outputs: [z]
	                        window:
	                        - [-1, 1]
	                        - [-1, 1]
	                      inputs: [x0, y0, x1, y1]
	                      outputs: [z0, z1]
	                    inputs: [x0, x1]
	                    outputs: [z0, z1]
	                  - inputs: [x0, x1]
	                    inverse:
	                      inputs: [x0, x1]
	                      mapping: [0, 1, 0, 1]
	                      outputs: [x0, x1, x2, x3]
	                    n_dims: 2
	                    name: ote_outmap
	                    outputs: [x0, x1]
	                  inputs: [x0, x1]
	                  outputs: [x0, x1]
	                - forward:
	                  - forward:
	                    - forward:
	                      - inputs: [x]
	                        name: ote_xincen_d2s
	                        offset: 5.18289805611e-07
	                        outputs: [y]
	                      - inputs: [x]
	                        name: ote_yincen_d2s
	                        offset: 1.92704532397e-09
	                        outputs: [y]
	                      inputs: [x0, x1]
	                      outputs: [y0, y1]
	                    - inputs: [x, y]
	                      matrix: !core/ndarray-1.0.0
	                        data:
	                        - [-0.435537213486619, -0.0015778113019466178]
	                        - [-0.0015773302619155368, 0.43567003971824037]
	                        datatype: float64
	                        shape: [2, 2]
	                      name: ote_affine_d2s
	                      outputs: [x, y]
	                      translation: !core/ndarray-1.0.0
	                        data: [0.0, 0.0]
	                        datatype: float64
	                        shape: [2]
	                    inputs: [x0, x1]
	                    outputs: [x, y]
	                  - forward:
	                    - inputs: [x]
	                      name: ote_xoutcen_d2s
	                      offset: 0.10539
	                      outputs: [y]
	                    - inputs: [x]
	                      name: ote_youtcen_d2s
	                      offset: -0.11913000025
	                      outputs: [y]
	                    inputs: [x0, x1]
	                    outputs: [y0, y1]
	                  inputs: [x0, x1]
	                  outputs: [y0, y1]
	                inputs: [x0, x1]
	                outputs: [y0, y1]
	              - forward:
	                - factor: 3600.0
	                  inputs: [x]
	                  outputs: [y]
	                - factor: 3600.0
	                  inputs: [x]
	                  outputs: [y]
	                inputs: [x0, x1]
	                outputs: [y0, y1]
	              inputs: [x0, x1]
	              inverse:
	                forward:
	                - forward:
	                  - factor: 0.0002777777777777778
	                    inputs: [x]
	                    outputs: [y]
	                  - factor: 0.0002777777777777778
	                    inputs: [x]
	                    outputs: [y]
	                  inputs: [x0, x1]
	                  outputs: [y0, y1]
	                - forward:
	                  - forward:
	                    - forward:
	                      - inputs: [x]
	                        name: ote_xoutcen_d2s
	                        offset: -0.10539
	                        outputs: [y]
	                      - inputs: [x]
	                        name: ote_youtcen_d2s
	                        offset: 0.11913000025
	                        outputs: [y]
	                      inputs: [x0, x1]
	                      outputs: [y0, y1]
	                    - forward:
	                      - inputs: [x, y]
	                        matrix: !core/ndarray-1.0.0
	                          data:
	                          - [-2.2959849432114767, -0.008315079445998134]
	                          - [-0.008312544360801184, 2.295284947801961]
	                          datatype: float64
	                          shape: [2, 2]
	                        outputs: [x, y]
	                        translation: !core/ndarray-1.0.0
	                          data: [-0.0, -0.0]
	                          datatype: float64
	                          shape: [2]
	                      - forward:
	                        - inputs: [x]
	                          name: ote_xincen_d2s
	                          offset: -5.18289805611e-07
	                          outputs: [y]
	                        - inputs: [x]
	                          name: ote_yincen_d2s
	                          offset: -1.92704532397e-09
	                          outputs: [y]
	                        inputs: [x0, x1]
	                        outputs: [y0, y1]
	                      inputs: [x, y]
	                      outputs: [y0, y1]
	                    inputs: [x0, x1]
	                    outputs: [y0, y1]
	                  - forward:
	                    - inputs: [x0, x1]
	                      mapping: [0, 1, 0, 1]
	                      outputs: [x0, x1, x2, x3]
	                    - forward:
	                      - forward:
	                        - coefficients: !core/ndarray-1.0.0
	                            data:
	                            - [-6.764695218042509e-12, -0.001181845439725266, -0.004421942263252729,
	                              -0.00015786783293089, -8.101344739691103e-05, -0.00223528778759241]
	                            - [0.9999913490363831, 0.01985321796323403, 0.01836568669267155,
	                              0.0008476551557795109, -0.004583374077531843, 0.0]
	                            - [0.01299262834412884, 0.0003111090225528627, 0.0006487537603577787,
	                              0.01831138758726425, 0.0, 0.0]
	                            - [0.01829981571350813, 0.0008920392687617073, 0.01402112565533553,
	                              0.0, 0.0, 0.0]
	                            - [0.0007670516309624491, 0.009553952594322013, 0.0, 0.0,
	                              0.0, 0.0]
	                            - [0.0004297874598029328, 0.0, 0.0, 0.0, 0.0, 0.0]
	                            datatype: float64
	                            shape: [6, 6]
	                          domain:
	                          - [-1, 1]
	                          - [-1, 1]
	                          inputs: [x, y]
	                          name: ote_x_back
	                          outputs: [z]
	                          window:
	                          - [-1, 1]
	                          - [-1, 1]
	                        - coefficients: !core/ndarray-1.0.0
	                            data:
	                            - [-6.115026307974087e-10, 0.9999869591970522, 0.01481498123407221,
	                              0.0183504542886734, 0.0009281781374761249, 0.0002958977074083435]
	                            - [-0.001179801005691817, 0.01742851464230401, 0.0003646472234911172,
	                              0.0008189728816032273, -0.001483363225125878, 0.0]
	                            - [-0.005044248520802729, 0.01828606785677567, 0.0009112362883588386,
	                              -0.009363002229886064, 0.0, 0.0]
	                            - [-0.0001649665026415299, 0.00081286189015533, 0.002997688363766571,
	                              0.0, 0.0, 0.0]
	                            - [-6.145531569187097e-05, -0.007795457001419592, 0.0, 0.0,
	                              0.0, 0.0]
	                            - [-2.764978182656641e-05, 0.0, 0.0, 0.0, 0.0, 0.0]
	                            datatype: float64
	                            shape: [6, 6]
	                          domain:
	                          - [-1, 1]
	                          - [-1, 1]
	                          inputs: [x, y]
	                          name: ote_y_backw
	                          outputs: [z]
	                          window:
	                          - [-1, 1]
	                          - [-1, 1]
	                        inputs: [x0, y0, x1, y1]
	                        outputs: [z0, z1]
	                      - inputs: [x0, x1]
	                        n_dims: 2
	                        outputs: [x0, x1]
	                      inputs: [x0, y0, x1, y1]
	                      outputs: [x0, x1]
	                    inputs: [x0, x1]
	                    outputs: [x0, x1]
	                  inputs: [x0, x1]
	                  outputs: [x0, x1]
	                inputs: [x0, x1]
	                outputs: [x0, x1]
	              outputs: [y0, y1]
	            - factor: 1000000.0
	              inputs: [x]
	              outputs: [y]
	            inputs: [x0, x1, x]
	            outputs: [y0, y1, y]
	          inputs: [x0, x1, x2]
	          name: oteip2v23
	          outputs: [y0, y1, y]
	  - frame:
	          frames:
	          - axes_names: [v2, v3]
	            axes_order: [0, 1]
	            axis_physical_types: ['custom:v2', 'custom:v3']
	            name: v2v3_spatial
	            unit: [arcsec, arcsec]
	          - axes_names: [wavelength]
	            axes_order: [2]
	            axis_physical_types: [em.wl]
	            name: spectral
	            unit: [um]
	          name: v2v3
	        transform:
	          forward:
	          - forward:
	            - forward:
	              - factor: 0.9999997262839518
	                inputs: [x]
	                name: dva_scale_v2
	                outputs: [y]
	              - factor: 0.9999997262839518
	                inputs: [x]
	                name: dva_scale_v3
	                outputs: [y]
	              inputs: [x0, x1]
	              outputs: [y0, y1]
	            - forward:
	              - inputs: [x]
	                name: dva_v2_shift
	                offset: 9.091097472161734e-05
	                outputs: [y]
	              - inputs: [x]
	                name: dva_v3_shift
	                offset: -0.00013117135776508434
	                outputs: [y]
	              inputs: [x0, x1]
	              outputs: [y0, y1]
	            inputs: [x0, x1]
	            name: DVA_Correction
	            outputs: [y0, y1]
	          - inputs: [x0]
	            outputs: [x0]
	          inputs: [x00, x10, x01]
	          outputs: [y0, y1, x0]
	  - frame:
	          frames:
	          - axes_names: [v2, v3]
	            axes_order: [0, 1]
	            axis_physical_types: ['custom:v2', 'custom:v3']
	            name: v2v3vacorr_spatial
	            unit: [arcsec, arcsec]
	          - axes_names: [wavelength]
	            axes_order: [2]
	            axis_physical_types: [em.wl]
	            name: spectral
	            unit: [um]
	          name: v2v3vacorr
	        transform:
	          forward:
	          - forward:
	            - forward:
	              - forward:
	                - forward:
	                  - factor: 0.0002777777777777778
	                    inputs: [x]
	                    outputs: [y]
	                  - factor: 0.0002777777777777778
	                    inputs: [x]
	                    outputs: [y]
	                  inputs: [x0, x1]
	                  outputs: [y0, y1]
	                - inputs: [lon, lat]
	                  outputs: [x, y, z]
	                  transform_type: spherical_to_cartesian
	                  wrap_lon_at: 180
	                inputs: [x0, x1]
	                outputs: [x, y, z]
	              - angles: [0.09226002166666666, 0.13311783694444446, -93.7605896, -70.775099941418,
	                  -90.75467525972158]
	                axes_order: zyxyz
	                inputs: [x, y, z]
	                outputs: [x, y, z]
	                rotation_type: cartesian
	              inputs: [x0, x1]
	              outputs: [x, y, z]
	            - inputs: [x, y, z]
	              outputs: [lon, lat]
	              transform_type: cartesian_to_spherical
	              wrap_lon_at: 360
	            inputs: [x0, x1]
	            name: v23tosky
	            outputs: [lon, lat]
	          - inputs: [x0]
	            outputs: [x0]
	          inputs: [x00, x10, x01]
	          name: v2v3_to_sky
	          outputs: [lon, lat, x0]
	  - frame:
	          frames:
	          - axes_names: [lon, lat]
	            axes_order: [0, 1]
	            axis_physical_types: [pos.eq.ra, pos.eq.dec]
	            name: sky
	            reference_frame:
	              frame_attributes: {}
	            unit: [deg, deg]
	          - axes_names: [wavelength]
	            axes_order: [2]
	            axis_physical_types: [em.wl]
	            name: spectral
	            unit: [um]
	          name: world
	        transform: null
	...

.. _`Appendix E`:

Appendix E: Portion of converted HDF5 file
==========================================

::

	000bea70:  88e6 0b00 0000 0000 a8e8 0b00 0000 0000 0c00 4800 0400 0000  :..................H.....
	000bea88:  0100 0a00 1400 0800 6173 6466 5f6c 6973 7400 0000 0000 0000  :........asdf_list.......
	000beaa0:  1901 0100 1000 0000 1000 0000 0100 0000 0000 0800 0000 0000  :........................
	000beab8:  0100 0000 0000 0000 0400 0000 601a 0b00 0000 0000 4f00 0000  :............`.......O...
	000bead0:  0c00 4000 0400 0000 0100 0300 1400 0800 4c30 0000 0000 0000  :..@.............L0......
	000beae8:  1901 0100 1000 0000 1000 0000 0100 0000 0000 0800 0000 0000  :........................
	000beb00:  0100 0000 0000 0000 0100 0000 601a 0b00 0000 0000 5000 0000  :............`.......P...
	000beb18:  0c00 4000 0400 0000 0100 0300 1400 0800 4c31 0000 0000 0000  :..@.............L1......
	000beb30:  1901 0100 1000 0000 1000 0000 0100 0000 0000 0800 0000 0000  :........................
	000beb48:  0100 0000 0000 0000 0100 0000 601a 0b00 0000 0000 5100 0000  :............`.......Q...
	000beb60:  0000 1800 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000beb78:  0000 0000 0000 0000 0c00 4000 0400 0000 0100 0500 1400 0800  :..........@.............
	000beb90:  6e61 6d65 0000 0000 1901 0100 1000 0000 1000 0000 0100 0000  :name....................
	000beba8:  0000 0800 0000 0000 0100 0000 0000 0000 0c00 0000 601a 0b00  :....................`...
	000bebc0:  0000 0000 5200 0000 0000 1800 0000 0000 0000 0000 0000 0000  :....R...................
	000bebd8:  0000 0000 0000 0000 0000 0000 0000 0000 0100 0600 0100 0000  :........................
	000bebf0:  1800 0000 0000 0000 1000 1000 0000 0000 a8ee 0b00 0000 0000  :........................
	000bec08:  1801 0000 0000 0000 5452 4545 0000 0000 ffff ffff ffff ffff  :........TREE............
	000bec20:  ffff ffff ffff ffff 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bec38:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bec50:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bec68:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bec80:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bec98:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000becb0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000becc8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bece0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000becf8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bed10:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bed28:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bed40:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bed58:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bed70:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bed88:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000beda0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bedb8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bedd0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bede8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bee00:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bee18:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bee30:  4845 4150 0000 0000 5800 0000 0000 0000 0800 0000 0000 0000  :HEAP....X...............
	000bee48:  50ee 0b00 0000 0000 0000 0000 0000 0000 0100 0000 0000 0000  :P.......................
	000bee60:  5000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :P.......................
	000bee78:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bee90:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000beea8:  1100 1000 0000 0000 10ec 0b00 0000 0000 30ee 0b00 0000 0000  :................0.......
	000beec0:  0c00 4800 0400 0000 0100 0a00 1400 0800 6173 6466 5f6c 6973  :..H.............asdf_lis
	000beed8:  7400 0000 0000 0000 1901 0100 1000 0000 1000 0000 0100 0000  :t.......................
	000beef0:  0000 0800 0000 0000 0100 0000 0000 0000 0400 0000 601a 0b00  :....................`...
	000bef08:  0000 0000 5300 0000 0c00 4000 0400 0000 0100 0300 1400 0800  :....S.....@.............
	000bef20:  4c30 0000 0000 0000 1901 0100 1000 0000 1000 0000 0100 0000  :L0......................
	000bef38:  0000 0800 0000 0000 0100 0000 0000 0000 0100 0000 601a 0b00  :....................`...
	000bef50:  0000 0000 5400 0000 0c00 4000 0400 0000 0100 0300 1400 0800  :....T.....@.............
	000bef68:  4c31 0000 0000 0000 1901 0100 1000 0000 1000 0000 0100 0000  :L1......................
	000bef80:  0000 0800 0000 0000 0100 0000 0000 0000 0100 0000 601a 0b00  :....................`...
	000bef98:  0000 0000 5500 0000 0000 1800 0000 0000 0000 0000 0000 0000  :....U...................
	000befb0:  0000 0000 0000 0000 0000 0000 0000 0000 0100 0100 0100 0000  :........................
	000befc8:  1800 0000 0000 0000 1100 1000 0000 0000 e8ef 0b00 0000 0000  :........................
	000befe0:  08f2 0b00 0000 0000 5452 4545 0000 0100 ffff ffff ffff ffff  :........TREE............
	000beff8:  ffff ffff ffff ffff 0000 0000 0000 0000 40f5 0b00 0000 0000  :................@.......
	000bf010:  1800 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf028:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf040:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf058:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf070:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf088:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf0a0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf0b8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf0d0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf0e8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf100:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf118:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf130:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf148:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf160:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf178:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf190:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf1a8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf1c0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf1d8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf1f0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf208:  4845 4150 0000 0000 5800 0000 0000 0000 2000 0000 0000 0000  :HEAP....X....... .......
	000bf220:  28f2 0b00 0000 0000 0000 0000 0000 0000 666f 7277 6172 6400  :(...............forward.
	000bf238:  696e 7075 7473 0000 6f75 7470 7574 7300 0100 0000 0000 0000  :inputs..outputs.........
	000bf250:  3800 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :8.......................
	000bf268:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf280:  0100 0400 0100 0000 1800 0000 0000 0000 1000 1000 0000 0000  :........................
	000bf298:  88f6 0b00 0000 0000 8000 0000 0000 0000 5452 4545 0000 0100  :................TREE....
	000bf2b0:  ffff ffff ffff ffff ffff ffff ffff ffff 0000 0000 0000 0000  :........................
	000bf2c8:  c8f9 0b00 0000 0000 1000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf2e0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf2f8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf310:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf328:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf340:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf358:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf370:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf388:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf3a0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf3b8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf3d0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf3e8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf400:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf418:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf430:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf448:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf460:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf478:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf490:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf4a8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf4c0:  0000 0000 0000 0000 4845 4150 0000 0000 5800 0000 0000 0000  :........HEAP....X.......
	000bf4d8:  1800 0000 0000 0000 e8f4 0b00 0000 0000 0000 0000 0000 0000  :........................
	000bf4f0:  4c30 0000 0000 0000 4c31 0000 0000 0000 0100 0000 0000 0000  :L0......L1..............
	000bf508:  4000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :@.......................
	000bf520:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf538:  0000 0000 0000 0000 534e 4f44 0100 0300 0800 0000 0000 0000  :........SNOD............
	000bf550:  80f2 0b00 0000 0000 0100 0000 0000 0000 a8f2 0b00 0000 0000  :........................
	000bf568:  c8f4 0b00 0000 0000 1000 0000 0000 0000 2010 0c00 0000 0000  :................ .......
	000bf580:  0100 0000 0000 0000 4810 0c00 0000 0000 6812 0c00 0000 0000  :........H.......h.......
	000bf598:  1800 0000 0000 0000 f813 0c00 0000 0000 0100 0000 0000 0000  :........................
	000bf5b0:  2014 0c00 0000 0000 4016 0c00 0000 0000 0000 0000 0000 0000  : .......@...............
	000bf5c8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf5e0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf5f8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf610:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf628:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf640:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf658:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf670:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf688:  1100 1000 0000 0000 a8f2 0b00 0000 0000 c8f4 0b00 0000 0000  :........................
	000bf6a0:  0c00 4800 0400 0000 0100 0a00 1400 0800 6173 6466 5f6c 6973  :..H.............asdf_lis
	000bf6b8:  7400 0000 0000 0000 1901 0100 1000 0000 1000 0000 0100 0000  :t.......................
	000bf6d0:  0000 0800 0000 0000 0100 0000 0000 0000 0400 0000 601a 0b00  :....................`...
	000bf6e8:  0000 0000 5600 0000 0000 1000 0000 0000 0000 0000 0000 0000  :....V...................
	000bf700:  0000 0000 0000 0000 0100 0500 0100 0000 1800 0000 0000 0000  :........................
	000bf718:  1000 1000 0000 0000 e8ff 0b00 0000 0000 c000 0000 0000 0000  :........................
	000bf730:  5452 4545 0000 0100 ffff ffff ffff ffff ffff ffff ffff ffff  :TREE....................
	000bf748:  0000 0000 0000 0000 d0fd 0b00 0000 0000 1000 0000 0000 0000  :........................
	000bf760:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf778:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf790:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf7a8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf7c0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf7d8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf7f0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf808:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf820:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf838:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf850:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf868:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf880:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf898:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf8b0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf8c8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf8e0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf8f8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf910:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf928:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf940:  0000 0000 0000 0000 0000 0000 0000 0000 4845 4150 0000 0000  :................HEAP....
	000bf958:  5800 0000 0000 0000 1800 0000 0000 0000 70f9 0b00 0000 0000  :X...............p.......
	000bf970:  0000 0000 0000 0000 696e 7075 7473 0000 6f75 7470 7574 7300  :........inputs..outputs.
	000bf988:  0100 0000 0000 0000 4000 0000 0000 0000 0000 0000 0000 0000  :........@...............
	000bf9a0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bf9b8:  0000 0000 0000 0000 0000 0000 0000 0000 534e 4f44 0100 0200  :................SNOD....
	000bf9d0:  0800 0000 0000 0000 08f7 0b00 0000 0000 0100 0000 0000 0000  :........................
	000bf9e8:  30f7 0b00 0000 0000 50f9 0b00 0000 0000 1000 0000 0000 0000  :0.......P...............
	000bfa00:  3804 0c00 0000 0000 0100 0000 0000 0000 6004 0c00 0000 0000  :8...............`.......
	000bfa18:  8006 0c00 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfa30:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfa48:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfa60:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfa78:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfa90:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfaa8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfac0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfad8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfaf0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfb08:  0000 0000 0000 0000 0100 0500 0100 0000 1800 0000 0000 0000  :........................
	000bfb20:  1000 1000 0000 0000 18ff 0b00 0000 0000 d000 0000 0000 0000  :........................
	000bfb38:  5452 4545 0000 0000 ffff ffff ffff ffff ffff ffff ffff ffff  :TREE....................
	000bfb50:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfb68:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfb80:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfb98:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfbb0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfbc8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfbe0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfbf8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfc10:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfc28:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfc40:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfc58:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfc70:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfc88:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfca0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfcb8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfcd0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfce8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfd00:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfd18:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfd30:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfd48:  0000 0000 0000 0000 0000 0000 0000 0000 4845 4150 0000 0000  :................HEAP....
	000bfd60:  5800 0000 0000 0000 0800 0000 0000 0000 78fd 0b00 0000 0000  :X...............x.......
	000bfd78:  0000 0000 0000 0000 0100 0000 0000 0000 5000 0000 0000 0000  :................P.......
	000bfd90:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfda8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfdc0:  0000 0000 0000 0000 0000 0000 0000 0000 534e 4f44 0100 0200  :................SNOD....
	000bfdd8:  0800 0000 0000 0000 10fb 0b00 0000 0000 0100 0000 0000 0000  :........................
	000bfdf0:  38fb 0b00 0000 0000 58fd 0b00 0000 0000 1000 0000 0000 0000  :8.......X...............
	000bfe08:  a800 0c00 0000 0000 0100 0000 0000 0000 d000 0c00 0000 0000  :........................
	000bfe20:  f002 0c00 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfe38:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfe50:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfe68:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfe80:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfe98:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfeb0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfec8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfee0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bfef8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bff10:  0000 0000 0000 0000 1100 1000 0000 0000 38fb 0b00 0000 0000  :................8.......
	000bff28:  58fd 0b00 0000 0000 0c00 4800 0400 0000 0100 0a00 1400 0800  :X.........H.............
	000bff40:  6173 6466 5f6c 6973 7400 0000 0000 0000 1901 0100 1000 0000  :asdf_list...............
	000bff58:  1000 0000 0100 0000 0000 0800 0000 0000 0100 0000 0000 0000  :........................
	000bff70:  0400 0000 601a 0b00 0000 0000 5700 0000 0c00 4000 0400 0000  :....`.......W.....@.....
	000bff88:  0100 0300 1400 0800 4c30 0000 0000 0000 1901 0100 1000 0000  :........L0..............
	000bffa0:  1000 0000 0100 0000 0000 0800 0000 0000 0100 0000 0000 0000  :........................
	000bffb8:  0100 0000 601a 0b00 0000 0000 5800 0000 0000 1800 0000 0000  :....`.......X...........
	000bffd0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000bffe8:  1100 1000 0000 0000 30f7 0b00 0000 0000 50f9 0b00 0000 0000  :........0.......P.......
	000c0000:  0c00 4000 0400 0000 0100 0500 1400 0800 6e61 6d65 0000 0000  :..@.............name....
	000c0018:  1901 0100 1000 0000 1000 0000 0100 0000 0000 0800 0000 0000  :........................
	000c0030:  0100 0000 0000 0000 0a00 0000 601a 0b00 0000 0000 5900 0000  :............`.......Y...
	000c0048:  0c00 3800 0400 0000 0100 0700 1400 0800 6f66 6673 6574 0000  :..8.............offset..
	000c0060:  1120 3f00 0800 0000 0000 4000 340b 0034 ff03 0000 0000 0000  :. ?.......@.4..4........
	000c0078:  0100 0000 0000 0000 0000 0000 0000 0000 0000 1800 0000 0000  :........................
	000c0090:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c00a8:  0100 0500 0100 0000 1800 0000 0000 0000 1000 1000 0000 0000  :........................
	000c00c0:  6803 0c00 0000 0000 d000 0000 0000 0000 5452 4545 0000 0000  :h...............TREE....
	000c00d8:  ffff ffff ffff ffff ffff ffff ffff ffff 0000 0000 0000 0000  :........................
	000c00f0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0108:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0120:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0138:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0150:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0168:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0180:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0198:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c01b0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c01c8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c01e0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c01f8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0210:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0228:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0240:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0258:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0270:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0288:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c02a0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c02b8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c02d0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c02e8:  0000 0000 0000 0000 4845 4150 0000 0000 5800 0000 0000 0000  :........HEAP....X.......
	000c0300:  0800 0000 0000 0000 1003 0c00 0000 0000 0000 0000 0000 0000  :........................
	000c0318:  0100 0000 0000 0000 5000 0000 0000 0000 0000 0000 0000 0000  :........P...............
	000c0330:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0348:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0360:  0000 0000 0000 0000 1100 1000 0000 0000 d000 0c00 0000 0000  :........................
	000c0378:  f002 0c00 0000 0000 0c00 4800 0400 0000 0100 0a00 1400 0800  :..........H.............
	000c0390:  6173 6466 5f6c 6973 7400 0000 0000 0000 1901 0100 1000 0000  :asdf_list...............
	000c03a8:  1000 0000 0100 0000 0000 0800 0000 0000 0100 0000 0000 0000  :........................
	000c03c0:  0400 0000 601a 0b00 0000 0000 5a00 0000 0c00 4000 0400 0000  :....`.......Z.....@.....
	000c03d8:  0100 0300 1400 0800 4c30 0000 0000 0000 1901 0100 1000 0000  :........L0..............
	000c03f0:  1000 0000 0100 0000 0000 0800 0000 0000 0100 0000 0000 0000  :........................
	000c0408:  0100 0000 601a 0b00 0000 0000 5b00 0000 0000 1800 0000 0000  :....`.......[...........
	000c0420:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0438:  0100 0500 0100 0000 1800 0000 0000 0000 1000 1000 0000 0000  :........................
	000c0450:  d00b 0c00 0000 0000 c000 0000 0000 0000 5452 4545 0000 0100  :................TREE....
	000c0468:  ffff ffff ffff ffff ffff ffff ffff ffff 0000 0000 0000 0000  :........................
	000c0480:  b809 0c00 0000 0000 1000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0498:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c04b0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c04c8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c04e0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c04f8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0510:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0528:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0540:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0558:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0570:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0588:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c05a0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c05b8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c05d0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c05e8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0600:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0618:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0630:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0648:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0660:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0678:  0000 0000 0000 0000 4845 4150 0000 0000 5800 0000 0000 0000  :........HEAP....X.......
	000c0690:  1800 0000 0000 0000 a006 0c00 0000 0000 0000 0000 0000 0000  :........................
	000c06a8:  696e 7075 7473 0000 6f75 7470 7574 7300 0100 0000 0000 0000  :inputs..outputs.........
	000c06c0:  4000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :@.......................
	000c06d8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c06f0:  0000 0000 0000 0000 0100 0500 0100 0000 1800 0000 0000 0000  :........................
	000c0708:  1000 1000 0000 0000 000b 0c00 0000 0000 d000 0000 0000 0000  :........................
	000c0720:  5452 4545 0000 0000 ffff ffff ffff ffff ffff ffff ffff ffff  :TREE....................
	000c0738:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0750:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0768:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0780:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0798:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c07b0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c07c8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c07e0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c07f8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0810:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0828:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0840:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0858:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0870:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0888:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c08a0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c08b8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c08d0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c08e8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0900:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0918:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0930:  0000 0000 0000 0000 0000 0000 0000 0000 4845 4150 0000 0000  :................HEAP....
	000c0948:  5800 0000 0000 0000 0800 0000 0000 0000 6009 0c00 0000 0000  :X...............`.......
	000c0960:  0000 0000 0000 0000 0100 0000 0000 0000 5000 0000 0000 0000  :................P.......
	000c0978:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0990:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c09a8:  0000 0000 0000 0000 0000 0000 0000 0000 534e 4f44 0100 0200  :................SNOD....
	000c09c0:  0800 0000 0000 0000 f806 0c00 0000 0000 0100 0000 0000 0000  :........................
	000c09d8:  2007 0c00 0000 0000 4009 0c00 0000 0000 1000 0000 0000 0000  : .......@...............
	000c09f0:  900c 0c00 0000 0000 0100 0000 0000 0000 b80c 0c00 0000 0000  :........................
	000c0a08:  d80e 0c00 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0a20:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0a38:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0a50:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0a68:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0a80:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0a98:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0ab0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0ac8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0ae0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0af8:  0000 0000 0000 0000 1100 1000 0000 0000 2007 0c00 0000 0000  :................ .......
	000c0b10:  4009 0c00 0000 0000 0c00 4800 0400 0000 0100 0a00 1400 0800  :@.........H.............
	000c0b28:  6173 6466 5f6c 6973 7400 0000 0000 0000 1901 0100 1000 0000  :asdf_list...............
	000c0b40:  1000 0000 0100 0000 0000 0800 0000 0000 0100 0000 0000 0000  :........................
	000c0b58:  0400 0000 601a 0b00 0000 0000 5c00 0000 0c00 4000 0400 0000  :....`.......\.....@.....
	000c0b70:  0100 0300 1400 0800 4c30 0000 0000 0000 1901 0100 1000 0000  :........L0..............
	000c0b88:  1000 0000 0100 0000 0000 0800 0000 0000 0100 0000 0000 0000  :........................
	000c0ba0:  0100 0000 601a 0b00 0000 0000 5d00 0000 0000 1800 0000 0000  :....`.......]...........
	000c0bb8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0bd0:  1100 1000 0000 0000 6004 0c00 0000 0000 8006 0c00 0000 0000  :........`...............
	000c0be8:  0c00 4000 0400 0000 0100 0500 1400 0800 6e61 6d65 0000 0000  :..@.............name....
	000c0c00:  1901 0100 1000 0000 1000 0000 0100 0000 0000 0800 0000 0000  :........................
	000c0c18:  0100 0000 0000 0000 0a00 0000 601a 0b00 0000 0000 5e00 0000  :............`.......^...
	000c0c30:  0c00 3800 0400 0000 0100 0700 1400 0800 6f66 6673 6574 0000  :..8.............offset..
	000c0c48:  1120 3f00 0800 0000 0000 4000 340b 0034 ff03 0000 0000 0000  :. ?.......@.4..4........
	000c0c60:  0100 0000 0000 0000 0000 0000 0000 0000 0000 1800 0000 0000  :........................
	000c0c78:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0c90:  0100 0500 0100 0000 1800 0000 0000 0000 1000 1000 0000 0000  :........................
	000c0ca8:  500f 0c00 0000 0000 d000 0000 0000 0000 5452 4545 0000 0000  :P...............TREE....
	000c0cc0:  ffff ffff ffff ffff ffff ffff ffff ffff 0000 0000 0000 0000  :........................
	000c0cd8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0cf0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0d08:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0d20:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0d38:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0d50:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0d68:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0d80:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0d98:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0db0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0dc8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0de0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0df8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0e10:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0e28:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0e40:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0e58:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0e70:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0e88:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0ea0:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0eb8:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0ed0:  0000 0000 0000 0000 4845 4150 0000 0000 5800 0000 0000 0000  :........HEAP....X.......
	000c0ee8:  0800 0000 0000 0000 f80e 0c00 0000 0000 0000 0000 0000 0000  :........................
	000c0f00:  0100 0000 0000 0000 5000 0000 0000 0000 0000 0000 0000 0000  :........P...............
	000c0f18:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0f30:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c0f48:  0000 0000 0000 0000 1100 1000 0000 0000 b80c 0c00 0000 0000  :........................
	000c0f60:  d80e 0c00 0000 0000 0c00 4800 0400 0000 0100 0a00 1400 0800  :..........H.............
	000c0f78:  6173 6466 5f6c 6973 7400 0000 0000 0000 1901 0100 1000 0000  :asdf_list...............
	000c0f90:  1000 0000 0100 0000 0000 0800 0000 0000 0100 0000 0000 0000  :........................
	000c0fa8:  0400 0000 601a 0b00 0000 0000 5f00 0000 0c00 4000 0400 0000  :....`......._.....@.....
	000c0fc0:  0100 0300 1400 0800 4c30 0000 0000 0000 1901 0100 1000 0000  :........L0..............
	000c0fd8:  1000 0000 0100 0000 0000 0800 0000 0000 0100 0000 0000 0000  :........................
	000c0ff0:  0100 0000 601a 0b00 0000 0000 6000 0000 0000 1800 0000 0000  :....`.......`...........
	000c1008:  0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  :........................
	000c1020:  0100 0600 0100 0000 1800 0000 0000 0000 1000 1000 0000 0000  :........................
	000c1038:  e012 0c00 0000 0000 1801 0000 0000 0000 5452 4545 0000 0000  :................TREE....
	000c1050:  ffff ffff ffff ffff ffff ffff ffff ffff 





