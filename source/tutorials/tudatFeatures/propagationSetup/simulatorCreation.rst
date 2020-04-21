.. _tudatFeaturesSimulatorCreation:

Performing State Propagation
============================

Propagating the dynamics, using the settings discussed in previous wiki pages, is done by using a :class:`DynamicsSimulator` object. We have several different derived classes of :class:`DynamicsSimulator` for different types of propagation. The following are available or under development in Tudat:

- Single-arc dynamics simulator, by using :class:`SingleArcDynamicsSimulator`. For most Tudat applications, this will be the preferred option (if you're not sure which one you need, it's probably this one).
- Multi-arc dynamics simulator, by using :class:`MultiArcDynamicsSimulator`.
- Hybrid-arc (combination of single- and multi-arc) dynamics simulator, by using :class:`HybridArcDynamicsSimulator`.

As discussed below, the state may be propagated directly upon the creation of one of these objects, or at a later time (depending on the input that is used.)

.. warning:: It is important to ensure that the propagator settings are compatible with the dynamics simulator type selected. Otherwise it will result in an exception being thrown during run-time.

These are implemented in derived classes and are discussed below. 

.. class:: DynamicsSimulator

   Base class from which the classes below are derived.

.. class:: SingleArcDynamicsSimulator
   
   This derived class simulates single arc dynamics and its constructor is:

   .. code-block:: cpp

      SingleArcDynamicsSimulator< StateScalarType, TimeType >( bodyMap, integratorSettings, propagatorSettings );

   where:

   - :literal:`bodyMap`

      :class:`NamedBodyMap` the map containing al the objects of type :class:`Body` used in the simulation.

   - :literal:`integratorSettings`

      :class:`IntegratorSettings` contains the settings of the integrator used, as discussed in :ref:`tudatFeaturesIntegratorSettings`.

   - :literal:`propagatorSettings`

      :class:`SingleArcPropagatorSettings` contains the settings that defines how the orbit is propagated, as described in :ref:`tudatFeaturesPropagatorSettings`.

   - :literal:`StateScalarType`

      Template argument used to set the precision of the state, :literal:`double` by default. For some application where a high precision is required this can be changed to e.g. :literal:`long double` (see :ref:`tudatFeaturesTimeStateTemplates`). 

   - :literal:`TimeType`

      Template argument used to set the precision of the time, :literal:`double` by default. For some application where a high precision is required this can be changed to e.g. :class:`Time` (see :ref:`tudatFeaturesTimeStateTemplates`). 

By default, the equations of motion are integrated once the object is created. This can be changed by adding additional arguments to the cosntructors of the :class:`DynamicsSimulator`, as shown below for the :class:`SingleArcDynamicsSimulator`:

.. code-block:: cpp

    // Create simulation object and propagate dynamics.
    SingleArcDynamicsSimulator< StateScalarType, TimeType >dynamicsSimulator( 
       bodyMap, integratorSettings, propagatorSettings, areEquationsOfMotionToBeIntegrated, 
       clearNumericalSolutions, setIntegratedResult, printNumberOfFunctionEvaluations );

where:

- :literal:`areEquationsOfMotionToBeIntegrated`
    Boolean to denote whether equations of motion should be integrated immediately at the end of the contructor or not (default true).
- :literal:`clearNumericalSolutions`
    Boolean to determine whether to clear the raw numerical solution member variables after propagation and resetting ephemerides (default false for :class:`SingleArcDynamicsSimulation`, and true for :class:`MultiArcDynamicsSimulation` and :class:`HybridArcDynamicsSimulation`).
- :literal:`setIntegratedResult`
    Boolean to determine whether to automatically use the integrated results to set ephemerides (default false for :class:`SingleArcDynamicsSimulation`, and true for :class:`MultiArcDynamicsSimulation` and :class:`HybridArcDynamicsSimulation`).
- :literal:`printNumberOfFunctionEvaluations`
    Boolean to toggle the printing of number of function evaluations at the end of propagation (default false).

.. class:: MultiArcDynamicsSimulator
   
   This derived class allows the numerical propagation to be performed in an arc-wise manner. It is constructed using:

   .. code-block:: cpp
   
    MultiArcDynamicsSimulator( bodyMap, integratorSettings, propagatorSettings, arcStartTimes,
                               areEquationsOfMotionToBeIntegrated, clearNumericalSolutions, setIntegratedResult );

   where:

 - :literal:`propagatorSettings`

      :class:`MultiArcPropagatorSettings`, contains the settings that defines how the orbit is propagated, as described in :ref:`tudatFeaturesPropagatorSettings`. Note that the propagator settings may be identical, or different, per arc (depending on the contents of the settings object).

   - :literal:`arcStartTimes`

      :literal:`std::vector< double >` containing the times at which the separate arcs start. Note: there is no requirement for the arcs to be contiguous and/or overlapping.

The following, alternative, constructor allows you to specify different integrator settings per arc:

    .. code-block:: cpp
   
      MultiArcDynamicsSimulator( bodyMap, integratorSettingsList, propagatorSettings, arcStartTimes,
                               areEquationsOfMotionToBeIntegrated, clearNumericalSolutions, setIntegratedResult );

  
   - :literal:`integratorSettingsList`

      :class:`std::vector< std::shared_ptr< IntegratorSettings > >` is the list (same size as number of arcs) with the settings of the integrator used, per arc.


.. class:: HybridArcDynamicsSimulator

   Allows some bodies to be propagated in a single arc, and some in a multi-arc fashion. This has the strict requirement that the single-arc bodies’ dynamics does not depend on the multi-arc bodies. For instance, the multi-arc bodies are typically spacecraft and the single-arc bodies solar system bodies. The vehicles do not exert an acceleration on the planets, but the planets exert accelerations on the spacecraft. When using hybrid-arc propagation, the single-arc bodies are first propagated, followed by the multi-arc bodies. 

 .. code-block:: cpp
   
    HybridArcDynamicsSimulator( bodyMap, integratorSettings, propagatorSettings, arcStartTimes,
                               areEquationsOfMotionToBeIntegrated, clearNumericalSolutions, setIntegratedResult );

   where:

 - :literal:`propagatorSettings`

      :class:`HybridArcPropagatorSettings`, contains the settings that defines how the orbit is propagated, as described in :ref:`tudatFeaturesPropagatorSettings`. 


Retrieving the Propagation History
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Once the :class:`DynamicsSimulator` object has been created and the equations of motion have been integrated, the propagation history of the selected bodies is stored within the :class:`DynamicsSimulator`, unless the :literal:`clearNumericalSolutions` variable has been set to true. Note that this is the **default** for multi- and hybrid-arc propagation. 

To make use of it, such history needs to be retrieved and saved to a file. The :class:`SingleDynamicsSimulator` offers a few different options to extract results, based on what you have input in the simulation. All propagation history is stored as a :literal:`std::map< double, Eigen::VectorXd >` (assuming you are using standard :literal:`TimeType` and :literal:`StateScalarType`).

.. note:: All examples given below are valid for the single-arc propagation only. The corresponding functions of the multi- and hybird-arc models produce a :literal:`std::vector< std::map< ..., ... > >`, instead of the :literal:`std::map< ..., ... >` output used below.

First of all, you can access the history of the propagated states for each object you have simulated:

   - **Extracting the state history in the conventional coordinates**

      The *conventional* coordinates are those coordinates that are used to describe the acceleration model. For translational motion, these are the Cartesian coordinates, whereas for rotational motion, they are quaternions. To access and save these results you can use the function :literal:`getEquationsOfMotionNumericalSolution` of the :class:`DynamicsSimulator` object, as shown below:

         .. code-block:: cpp

             // Write body propagation history in conventional coordinates to file.
             writeDataMapToTextFile( dynamicsSimulator.getEquationsOfMotionNumericalSolution( ),
                                     "bodyPropagationHistory.dat",
                                     outputPath );

   - **Extracting the state history in the propagated coordinates**

      The *propagation* coordinates are those coordinates that are used to describe the equations of motion and thus are the ones that are actually integrated. For translational motion, these can be Cartesian coordinates, Keplerian elements and one of the three unified state models, whereas for rotational motion, these can be quaternions, modified Rodrigues parameters or the exponential map. To access and save these results you can use the function :literal:`getEquationsOfMotionNumericalSolutionRaw` of the :class:`DynamicsSimulator` object, as shown here:

         .. code-block:: cpp

             // Write body propagation history in propagation coordinates to file.
             writeDataMapToTextFile( dynamicsSimulator.getEquationsOfMotionNumericalSolutionRaw( ),
                                     "bodyPropagationRawHistory.dat",
                                     outputPath );

   .. tip:: You can find more information about the difference between *conventional* and *propagation* coordinates in :ref:`tudatFeaturesPropagatorSettingsCoordinates`.

In case you have also decided to store some dependent variables, you can access and save their history by placing the following code after the :class:`DynamicsSimulator` object creation:

.. code-block:: cpp

    // Write body dependent variable history to file.
    writeDataMapToTextFile( dynamicsSimulator.getDependentVariableHistory( ),
                            "bodyDependentVariableHistory.dat",
                            outputPath );

Similarly to the :literal:`getEquationsOfMotionNumericalSolution` and :literal:`getEquationsOfMotionNumericalSolutionRaw` functions, the :literal:`getDependentVariableHistory` outputs a map, where the key is the time and the mapped value a vector of variable length. This length depends on a few things:

   - **For extraction of numerical solutions:** how many bodies are being propagated, how many propagators are used and the length of the conventional or propagation coordinates

   - **For extraction of dependent variables:** how many depenent variables are being saved and whether they are a scalar or a vector

   .. note:: For the dependent variables, the simulation will automatically output the order and length of each dependent variable. This is true, however, only if you have not turned off this feature while adding the dependent variable settings to the propagator object.

Finally, the :class:`DynamicsSimulator` object offers a couple more interesting member functions that can be used to access some internal variables. You can, for instance, use:

   - :literal:`getCumulativeNumberOfFunctionEvaluations` to access the history of the number of function evaluations over propagation; the key is the simulation time and the mapped value gives the cumulative number of function evaluations; this can be very useful when studying the performance of the propagation coordinates and/or of a variable step size integrator.

   - :literal:`getCumulativeComputationTimeHistory` to access the history of the computation time over propagation; the key is the simulation time and the mapped value gives the cumulative computation time.

The code snippets used at the beginning of this section can be also used to save the cumulative variables above. The result will be a :literal:`.dat` file in the folder specified by the :literal:`outputPath` string. You can make use of the :literal:`tudat_applications::getOutputPath( )` function to get a folder name relative to the project folder.

.. tip:: In case you wanted to have more control on how the data is written to the text file (e.g., setting a different number of significant digits, adding a header to the file, etc.) you can check out the Wiki page on :literal:`writeDataMapToTextFile` at :ref:`tudatFeaturesInputOutput`.
