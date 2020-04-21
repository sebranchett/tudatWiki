.. _tudatFeaturesIndex:

Tudat Libraries
===============

These pages of the wiki provide further details about critical libraries necessary when setting up a simulation. A graphical representation is shown below, linked to the corresponding section of the Tudat libraries. Below the diagram the list of content are presented in the preferred order of reading and they follow how they should be typically added to the simulation source file.

.. graphviz::

   digraph 
   {
      # general graph settings    
      rankdir = "LR";
      splines = ortho;
      compound = true;
      

      # general node settings 
      node [shape = box, style = filled, width = 1.25, fixedsize = true, color = lightgrey, fontname = Times, fontsize = 8];


      # specific node color settings
      DynamicsSimulator [color = lightgreen];
      IntegratorSettings, NamedBodyMap, PropagatorSettings [color = lightblue];
      "DependentVariable(s)\nNumerical Solution", "Equations of Motion\nNumerical Solution" [color = peachpuff]; 


      # Hyperlinks (Sphinx auto referencing not working here, need to link to exact web adres)
      DynamicsSimulator [href = "http://tudat.tudelft.nl/tutorials/tudatFeatures/propagationSetup/simulatorCreation.html#DynamicsSimulator", target = "_top"]; 
      IntegratorSettings [href = "http://tudat.tudelft.nl/tutorials/tudatFeatures/propagationSetup/integratorSettings.html#IntegratorSettings", target = "_top"];
      NamedBodyMap [href = "http://tudat.tudelft.nl/tutorials/tudatFeatures/environmentSetup/index.html#NamedBodyMap", target = "_top"];
      PropagatorSettings [href = "http://tudat.tudelft.nl/tutorials/tudatFeatures/propagationSetup/propagatorSettings.html#PropagatorSettings", target = "_top"];
      AccelerationModel [href = "http://tudat.tudelft.nl/tutorials/tudatFeatures/accelerationSetup/frameworkAcceleration.html#SelectedAccelerationMap", target = "_top"];
      BodySettings [href = "http://tudat.tudelft.nl/tutorials/tudatFeatures/environmentSetup/index.html#BodySettings", target = "_top"];
      AccelerationSettings [href = "http://tudat.tudelft.nl/tutorials/tudatFeatures/accelerationSetup/frameworkAcceleration.html#AccelerationSettings", target = "_top"];
      TerminationSettings [href = "http://tudat.tudelft.nl/tutorials/tudatFeatures/propagationSetup/propagatorSettingsTermination.html#PropagationTerminationSettings", target = "_top"];
      

      # DynamicsSimulatorInput
      NamedBodyMap -> DynamicsSimulator;
      PropagatorSettings -> DynamicsSimulator;
      IntegratorSettings -> DynamicsSimulator;


      # PropagatorSettings input     
      NamedBodyMap -> PropagatorSettings;
      AccelerationModel -> PropagatorSettings;
      initialBodyStates -> PropagatorSettings;
      bodiesToPropagate -> PropagatorSettings;
      TerminationSettings -> PropagatorSettings;
      centralBodies -> PropagatorSettings;
      dependentVariables -> PropagatorSettings;

      subgraph clusterPropagatorSettings
      {   
         style = dashed;
         dependentVariables [style = dotted , fillcolor = lightgrey, color = black];
         "simulationEndEpoch" -> TerminationSettings;
         "simulationEndEpoch" [style = dotted, fillcolor = lightgrey, color = black];
         {rank = same; TerminationSettings, initialBodyStates}

      }


      # BodySettings input
      "User-defined \nenvironment settings" -> BodySettings;
      getDefaultBodySettings -> BodySettings;
      bodyNames -> getDefaultBodySettings;


      # NamedBodyMap input
      BodySettings -> NamedBodyMap;
      "User-defined \nbodies" [style = dotted, fillcolor = lightgrey, color = black];
      "User-defined \nbodies" -> NamedBodyMap;

      subgraph clusterNamedBodyMap
      {   
         style = dashed;
         "(initial/final) time" -> getDefaultBodySettings;
         "(initial/final) time" [style = dotted, fillcolor = lightgrey, color = black];
         "User-defined \nenvironment settings" [style = dotted , fillcolor = lightgrey, color = black];
         { rank = same; "User-defined \nenvironment settings", BodySettings } 
         { rank = same; bodyNames, getDefaultBodySettings, "(initial/final) time" } 
      }
     

      # AccelerationSettings input
      accelerationType -> AccelerationSettings;
      "Additional \ninformation" -> AccelerationSettings;

      subgraph clusterAccelerationSettings
      {
         style = dashed;
         "Additional \ninformation",accelerationType;
         "Additional \ninformation" [style = dotted, color = black];
      } 


      # AccelerationModel input
      AccelerationSettings -> AccelerationModel;
      bodiesToPropagate -> AccelerationModel;
      centralBodies -> AccelerationModel;
      NamedBodyMap -> AccelerationModel;

      # IntegratorSettings input
      integratorType -> IntegratorSettings;
      simulationStartEpoch -> IntegratorSettings;
      "(initial) stepSize" -> IntegratorSettings;
     
      subgraph clusterIntegratorSettings
      {
         style = dashed;
         integratorType, simulationStartEpoch, "(initial) stepSize";
      } 

      {rank = same; DynamicsSimulator, "Equations of Motion\nNumerical Solution", "DependentVariable(s)\nNumerical Solution"};

      DynamicsSimulator -> "Equations of Motion\nNumerical Solution";
      DynamicsSimulator -> "DependentVariable(s)\nNumerical Solution" [constraint = false];

               
      # Define rank and connections of some blocks for positioning 
      {rank = same; PropagatorSettings, IntegratorSettings, NamedBodyMap, AccelerationModel, "User-defined \nbodies" }
      "(initial/final) time" -> simulationStartEpoch [style = invis];		
      "IntegratorSettings" -> "User-defined \nbodies" [style = invis];
      "(initial/final) time" -> "User-defined \nbodies" [style = invis];
      dependentVariables -> initialBodyStates [style = invis];

   }

.. graphviz::

   digraph
   {
      # General diagram settings
      rankdir = "LR";
      splines = ortho;    
      compound = true;  

      subgraph clusterLegend
      {
      rank = min;
      style = dashed;


     	# general node settings 
     	node [shape = box, style = filled, width = 1.25, fixedsize = true, color = lightgrey, fontname = Times, fontsize = 9];


   	"Main block" [fillcolor = lightgreen];
     	"Optional input" [style = dotted, fillcolor = lightgrey, color = black];
     	"Input for \nmain block" [fillcolor = lightblue];
      Output [color = peachpuff];
     	"Optional input"-> "Input for \nmain block" -> "Main block" -> "Output" [style = invis];
      }
   }


In the example code given on the following pages, namespaces are typically omitted for the sake of brevity. To find the namespace that a given class or function is defined in, search for this class/function in the `Doxygen documentation <https://doxygen.tudat.tudelft.nl>`_ Below is an example screenshot for the Doxygen page of the :class:`MultiArcDynamicsSimulator` class, which is in the :literal:`tudat::propagators` namespace (as deduced from the red box).

      .. figure:: images/doxygenNameSpace.png

To find the file that you need to ::literal:`#include`, follow the link in the blue box shown above, which leads to the following. Here, it can be seen that, to use this code, the :literal:`Tudat/SimulationSetup/PropagationSetup/dynamicsSimulator.h` file needs to be included.

      .. figure:: images/doxygenFiles.png
   
Feature documentation for the various Tudat tools are given in the pages below. For new users, we recommend that you start with the :ref:`walkthroughsIndex`.

.. toctree::
   :numbered:
   :maxdepth: 3

   environmentSetup/index
   accelerationSetup/index
   propagationSetup/index
   estimationSetup/index
   astroTools/index
   lowThrustTrajectoryDesign/index
   mathTools/index
   otherLibraries/index
