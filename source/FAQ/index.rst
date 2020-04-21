.. _faqIndex:

FAQ
===

This page will give some answers to frequently asked questions about Tudat. If your question is not anwsered on this page, please open an issue on the Tudat github `page <https://github.com/Tudat/tudat/issues/new>`_.

	- *Q: I would like to use Tudat, where should I start?*
	   A: the :ref:`gettingStarted` page contains a detailed explanation for new users of how to start using Tudat.

	- *Q: I would like to use Tudat, but I don't know how to code in C++. Can I still use it?*
	   A: The JSON interface, which can be used to access much of the orbit propagation tools in Tudat, can be used without any knowledge of C++. To use the full capabilities of Tudat, a little knowledge of C++ is necessary. You can learn :literal:`C++` from scratch using `this <http://www.cplusplus.com/doc/tutorial/>`_ page. Working through the example application tutorials that we provide will give you some insight into how to use C++ for Tudat specifically. If you have already used Matlab before, `this <http://runge.math.smu.edu/Courses/Math6370_Spring13/Lec2.pdf>`_ page lists the differences between the two languages. The same page exists for Python users, and can be found `here <https://pdfs.semanticscholar.org/9ad1/030685050e949d1a3d6d92bababcbe075e07.pdf>`_.
	
.. _faqCompilationInstallation:

Compilation and Installation
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

	- *Q: I am on Windows and I would like to use the command line for git, how can I do this?*
	   A: A command line tool was developed for Tudat, you can find it in your Tudat bundle under external/tools/tudat_shell. Within this tool you can use all the git commands.

	- *Q: How can I run CMake again from Qt?*
	   A: In the Project Tree tab in Qt (on the left side of your screen) you can right-click on the top-level TudatBundle icon and select the "Run CMake" option. The output will be shown in the General Messages tab.

	- *Q: Some of the unit test on Windows fail due to high RAM usage, how can I solve this?*
	   A: Turn the BUILD_EXTENDED_PRECISION_PROPAGATION_TOOLS off in the Projects tab.

	- *Q: How can I speed up the compilation of Tudat?*
	   A: There is the possibility to use multiple threads (cores) to compile C++ code. This feature generally works flawlessly on Mac or Linux, but not on Windows. A point of attention when using multi-thread compilation is RAM usage. A single thread can use up to 2-3 GB of RAM to compile certain applications (depending on your compiler). Depending on the robustness of your operating system, multi-core compilation can cause the RAM to be completely exhausted, freezing your system and forcing a hard reboot. To instruct the compiler to use ``X`` threads, add the command ``-jX`` (so ``-j8`` for 8 threads) to the Tool arguments (or additional arguments) line under Projects -> Build -> Build Steps.

	- *Q: I have problems with the installation of the Tudat bundle, how can I solve these problems?* 
	   A: The :ref:`troubleshootingInstallation` page contains several solutions to errors that can occur during installation (in addition to those listed here)s. If this does not help, a list of issues raised by users can be found `here <https://github.com/Tudat/tudat/issues>`_. If this still does not contain a solution, a new issue can be made on `github <https://github.com/Tudat/tudat/issues/new>`_ so that the developers can help you.

	- *Q: My application fails due to* ``undefined reference to ...``, *or* ``Undefined symbols for architecture x86_64``, *what am I doing wrong?* 
	   A: These issues are generally related to a missing library in the :literal:`CMakeLists.txt` of the application you are using. See :ref:`writingCMakeLists` for more details.

        - *Q: I get a compilation error due to* ``no member named...``, *how can I fix it?* 
           A: Such errors are usually caused by not including the file in which the element is defined or by not looking for it in the proper namespace. Check the :literal:`include` commands at the beginning of the file to make sure all the required files are there.

        - *Q: How to find what is causing an error when the error output is not explicit enough?* 
           A: In that case, having a close look at *where* the error (or warning) is generated in the Tudat code can be very insightful. To find it, copy/paste the error message in the *Search Results* tab in the Qt interface to identify where this message is generated among all the Tudat files. Be sure to setup the search settings in such a way that it will parse all the files from which the error is likely to originate (see :ref:`troubleshootingSearchingTudat` for more details). 

Using and Developing Tudat
~~~~~~~~~~~~~~~~~~~~~~~~~~


	- *Q: I want to add some features to Tudat, can I do this, and if yes, how?*
	   A: You can always add features to Tudat. If you think that it can also be used by others, you can make a pull request to the tudat repository. The :ref:`devGuideIndex` page contains guides on how to develop extra features for Tudat.

	- *Q: How can I find the definition of a specific class/function/variable in Tudat?*
	   A: In Qt, you can right click on what you would like to inspect and select the "Follow Symbol Under Cursor" option. This will take you to the file in which that specific symbol is defined or merely *declared*. The latter is the case when you import e.g. a function declaration from a header file, while its definition is located in a .cpp file. To look for the corresponding file automatically, right click the symbol again (in the header), and then click on "Switch Header/Source".

	- *Q: How can I find the usages of a specific class/function/variable in the whole Tudat bundle?*
	   A: In Qt, you can right click on what you would like to find the usages of and select the "Find Usages" option. 

	- *Q: How can I effectively find something throughout the whole Tudat bundle?*
	   A: Just like in other applications, you can press :literal:`Ctrl + F` or click on "Edit" -> "Find/Replace" -> "Find/Replace" to open the search toolbar. This works perfect if you need to search in the current document, but if you would like to have the whole bundle as the search scope, click on "Advanced..." in the right corner of the Find/Replace toolbar. Set "Scope" to Files in File System (*not* to Project "Tudatbundle"). This setting allows for a search in *all* files that match the file pattern, not just the source and header files. Make sure that "Directory" is set to your Tudatbundle root directory, e.g. :literal:`/home/[user]/tudatBundle` on a Unix-like system. If you want to know more about regular expressions, which are used as search patterns, click `here <https://en.wikipedia.org/wiki/Regular_expression>`_.

	- *Q: I cannot see all files in the Qt Creator file browser, how can I make them visible?*
	   A: Qt Creater defaults to the "Projects" view of the built-in file browser. This only shows source and header files, as well as CMake scripts. JSON files are therefore invisible, so to be able to see those, click on the "Projects" button (located above the directory tree), and select "File System". The tree is now showing the root directory tree for your system (e.g. :literal:`/` on Unix-like systems and :literal:`C:\\` on Windows). To only show the Tudat bundle tree, click on "Computer" and select "TudatBundle".

	- *Q: When running a Tudat propagation application, my application stops running but I don't know why. How can I find out?*
	   A: The dynamics Simulator class has a function: :literal:`getPropagationTerminationReason`, which allows you to retrieve a pointer to the :literal:`PropagationTerminationDetails`. This object has a function called :literal:`getPropagationTerminationReason`, which returns the reason for termination of the propagation.

	- *Q: I have my own custom aerodynamic coefficient file/function that does not work with any of the given options in Tudat, how can I implement this?*
	   A: Tudat has a :literal:`customAerodynamicCoefficientInterface` which can be used for this purpose (see the class definition for more information on how to use this). You do need to make a :literal:`std::function` for your coefficients to be able to use it.

	- *Q: When making a Pagmo problem with a propagation step in it, the optimization stops after a while due to high RAM usage, how can I fix this?*
	   A: Put some variables (especially the creation of the :literal:`bodyMap`) in the constructor of the problem. This will make sure that some vectors will not grow unnecesarily large after several generations.

	- *Q: I get a CSpice error telling me it cannot load any more kernels when running a optimization application, how can I solve this?*
	   A: Make sure that you are not loading the Spice kernels in the fitness function, but either in the constructor or somewhere else that will not be called upon by each time the fitness function is called.

	- *Q: How can I repress the output of the* :literal:`dependentVariableSettings` *?*
	   A: Set the second argument of the class to false/0.
	

	
