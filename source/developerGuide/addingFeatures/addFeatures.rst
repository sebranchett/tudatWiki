 .. _addingFeatures:

Adding Features to Tudat
========================
In some cases, the various features of Tudat do not contain the tool(s) to solve a specific problem. As Tudat is written in C++ and all the source code is readily available to the user, it is possible to add features to the Tudat software. This does require a good understanding of the C++ language and of how Tudat works, thus it is only recommended to users who have been working with Tudat before. 

Adding Dependent Variables to Save
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
As the execution of the simulator happens "behind closed doors" in the :literal:`DynamicsSimulator` function, it can be non-trivial to retrieve the history of dependent variables. Luckily, as explained in :ref:`tudatFeaturesPropagatorSettingsDependentVariables`, it is possible to create a list of dependent variables and use it as an input to the propagation settings to be able to retrieve a history of these dependent variables after the simulation is done. A list of the available dependent variables is given in: :ref:`tudatFeaturesPropagatorSettingsDependentVariables`. It is possible that another dependent variable is needed by the user, which is not given in this list. This guide will show how a new dependent variable can be added to the code.

To add a new dependent variable, four files will need to be changed, these are all located in the following directory:
	
	.../tudatBundle/tudat/Tudat/SimulationSetup/PropagationSetup/

For this guide, the :literal:`mach_number_dependent_variable` will be used as an example of how a dependent variable is implemented. Be aware that there are some slight differences between the implementation of the :literal:`mach_number_dependent_variable` and other variables, but this will be explained in the guide.

The first addition needs to be made in the :literal:`propagationOutputSettings.h` file. Here, an :literal:`enum` called :literal:`PropagationDependentVariables` is given which shows the names of all the available dependent variables that can be saved. The list looks as follows;

.. code-block:: cpp
   
	enum PropagationDependentVariables
	{
	    mach_number_dependent_variable = 0,
	    altitude_dependent_variable = 1,
	    airspeed_dependent_variable = 2,
	    ...
            ...
            ...
	    body_fixed_groundspeed_based_velocity_variable = 31,
	    keplerian_state_dependent_variable = 32,
	    modified_equinocial_state_dependent_variable = 33
	};


The name of the new variable can thus be added to the end of the list, with a number assigned to it. 

The next addition needs to be made in the :literal:`propagationOutputSettings.cpp` file. In this file, a method is present called: :literal:`getDependentVariableName`. This method contains a switch statement which contains cases for all the different dependent variables. For a new variable, a new case must be made with the name that was added to the :literal:`PropagationDependentVariables` list. Inside the case, a string called :literal:`variableName` must be assigned with the name of the to be added variable. In the case of the mach number, it looks as follows:

.. code-block:: cpp
   
	case mach_number_dependent_variable:
            variableName = "Mach number ";
            break;

It is important to remember that the case should be the same as the name given in :literal:`PropagationDependentVariables`, but the :literal:`variableName` can be chosen to the preference of the user. 

The third addition needs to be made in :literal:`propagationOutput.cpp`. Here a method called :literal:`getDependentVariableSize` is located. In this function, another switch statement is made with the same cases as before. Again a new case should be made for the new variable, but now, inside the case, another variable called :literal:`variableSize` should be changed to the size of the added variable. Thus this could be 1 for a :literal:`double`, or 3 for an :literal:`Eigen::Vector3d`. In the case of the mach number, it looks as follows:

.. code-block:: cpp
   
	case mach_number_dependent_variable:
            variableSize = 1;
            break;

The final change should then be made to the :literal:`propagationOutput.h` file. In a method called :literal:`getDoubleDependentVariableFunction`, the actual calculation of the variable is done. Again, a switch stamentent in the same manner as before is made, with in every case the calculation of the variable. This method gets the :literal:`bodyMap` as input, thus methods available inside the :literal:`FlightConditions` could, for example, be used to calculate the dependent variable. For the new variable, a new case needs to be made in which a :literal:`std::function< double( )>` is returned. This function can take several variables as input and should return the dependent variable. The implementation for the mach number is given here:

.. code-block:: cpp
   
	case mach_number_dependent_variable:
        {
            if( bodyMap.at( bodyWithProperty )->getFlightConditions( ) == NULL )
            {
                std::string errorMessage = "Error, no flight conditions available when requesting Mach number output of " +
                        bodyWithProperty + "w.r.t." + secondaryBody;
                throw std::runtime_error( errorMessage );
            }

            std::function< double( const double, const double ) > functionToEvaluate =
                    std::bind( &aerodynamics::computeMachNumber, std::placeholders::_1, std::placeholders::_2 );

            // Retrieve functions for airspeed and speed of sound.
            std::function< double( ) > firstInput =
                    std::bind( &aerodynamics::FlightConditions::getCurrentAirspeed,
                                 bodyMap.at( bodyWithProperty )->getFlightConditions( ) );
            std::function< double( ) > secondInput =
                    std::bind( &aerodynamics::FlightConditions::getCurrentSpeedOfSound,
                                 bodyMap.at( bodyWithProperty )->getFlightConditions( ) );


            variableFunction = std::bind( &evaluateBivariateFunction< double, double >,
                                            functionToEvaluate, firstInput, secondInput );
            break;
      }

If the variable is a vector (or matrix) and not a double, the case should not be added to :literal:`getDoubleDependentVariableFunction`, but to: :literal:`getVectorDependentVariableFunction`

If this is all done, the dependent variable name can be added to the dependent variables save list.


Adding a New Environment Model
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Tudat contains various pre-built environment models that users are able to implement in their simulations. However, for some applications, it might be necesarry to alter an exisiting environment model, or add a completely new one. This section will describe how a new environment model can be added to Tudat. 

First, it is important to understand how environment models are used in Tudat, this will give some insight in what needs to be added/changed if a new environment model is integrated into Tudat. For this tutorial, an existing environment model will be used as an example: the tabulated atmosphere model. 

A user will define a list of bodies to be used in the simulation, an example is given here:

.. code-block:: cpp

    // Define simulation body settings.
    std::map< std::string, std::shared_ptr< BodySettings > > bodySettings =
            getDefaultBodySettings( { "Mars" }, simulationStartEpoch - 10.0 * fixedStepSize,
                                    simulationEndEpoch + 10.0 * fixedStepSize );
    bodySettings[ "Mars" ]->ephemerisSettings = std::make_shared< simulation_setup::ConstantEphemerisSettings >(
                Eigen::Vector6d::Zero( ), "SSB", "J2000" );
    bodySettings[ "Mars" ]->rotationModelSettings->resetOriginalFrame( "J2000" );
    std::string atmosphereFile = ... ;
    bodySettings[ "Mars" ]->atmosphereSettings = std::make_shared< TabulatedAtmosphereSettings >( atmosphereFile );


The :literal:`BodySettings` class contains a member variable called :literal:`atmosphereSettings` of type :literal:`AtmosphereSettings`. As can be seen in the last line of the example code above, this member variable is set to a :literal:`TabulatedAtmosphereSettings` by the user. This is a valid statement (eventhough :literal:`atmosphereSettings` is not of type :literal:`TabulatedAtmosphereSettings`) because :literal:`TabulatedAtmosphereSettings` is derived from the base class :literal:`AtmosphereSettings`. These concepts are called polymorphism and inheritance, and should be understood by the reader before continuing. 

When all the :literal:`bodySettings` are defined by the user, the following command will be executed:

.. code-block:: cpp

    simulation_setup::NamedBodyMap bodyMap = simulation_setup::createBodies( bodySettings );

The :literal:`simulation_setup::createBodies()` will set all the environment models up to be used later in the simulation, and stores them in the variable :literal:`bodyMap`. If the :literal:`simulation_setup::createBodies()` is looked at, the following can be seen:

.. code-block:: cpp

    // Create atmosphere model objects for each body (if required).
    for( unsigned int i = 0; i < orderedBodySettings.size( ); i++ )
    {
        if( orderedBodySettings.at( i ).second->atmosphereSettings != NULL )
        {
            bodyMap[ orderedBodySettings.at( i ).first ]->setAtmosphereModel(
                        createAtmosphereModel( orderedBodySettings.at( i ).second->atmosphereSettings,
                                               orderedBodySettings.at( i ).first ) );
        }
    }

Where :literal:`orderedBodySettings` is an ordered map of all the bodies. This code goes through all the bodies and checks if the bodies have an atmosphere model set or not. If not, the atmosphere model is set by the :literal:`setAtmosphereModel()` function, which takes as input a pointer to the atmosphereModel of the body. This pointer is returned by the function: :literal:`createAtmosphereModel()`, which creates the atmosphere model using the settings defined by the user in the variable :literal:`atmosphereSettings`. The :literal:`createAtmosphereModel()` looks as follows:

.. code-block:: cpp

        std::shared_ptr< aerodynamics::AtmosphereModel > createAtmosphereModel(
           const std::shared_ptr< AtmosphereSettings > atmosphereSettings,
           const std::string& body )
	{
	    using namespace tudat::aerodynamics;

	    // Declare return object.
	    std::shared_ptr< AtmosphereModel > atmosphereModel;

	    // Check which type of atmosphere model is to be created.
	    switch( atmosphereSettings->getAtmosphereType( ) )
	    {
	    case exponential_atmosphere:
	    {
		...
		...
		...
	    }
	    case tabulated_atmosphere:
	    {
		// Check whether settings for atmosphere are consistent with its type
		std::shared_ptr< TabulatedAtmosphereSettings > tabulatedAtmosphereSettings =
		        std::dynamic_pointer_cast< TabulatedAtmosphereSettings >( atmosphereSettings );
		if( tabulatedAtmosphereSettings == NULL )
		{
		    throw std::runtime_error(
		                "Error, expected tabulated atmosphere settings for body " + body );
		}
		else
		{
		    // Create and initialize tabulatedl atmosphere model.
		    atmosphereModel = std::make_shared< TabulatedAtmosphere >(
		                tabulatedAtmosphereSettings->getAtmosphereFile( ) );
		}
		break;
	    }
	    case nrlmsise00:
	    {
		...
		...
		...
	    }
	    default:
		throw std::runtime_error(
		            "Error, did not recognize atmosphere model settings type " +
		            std::to_string( atmosphereSettings->getAtmosphereType( ) ) );
	    }

	    return atmosphereModel;
	}

This function checks which atmosphere model is used by using a switch statement with the :literal:`atmosphereSettings->getAtmosphereType( )`. The three cases are defined using the enum :literal:`AtmosphereTypes` (located in :literal:`createAtmosphereModel.h`). In the case of the tabulated atmosphere, first a check is made if tabulated atmosphere setting are actually initialized. If they are, the :literal:`atmosphereModel` is set to a tabulated atmosphere using the settings from :literal:`atmosphereSettings`. It can be seen in the example code above that the variable :literal:`atmosphereModel` is of type :literal:`AtmosphereModel`, but it is set to a variable of type :literal:`TabulatedAtmosphere` in the tabulated atmosphere case. This is again valid due to the fact that the class :literal:`TabulatedAtmosphere` is derived from the base class :literal:`AtmosphereModel`.

After the atmosphere model is set in the body map, it can be used whenever a certain quantity, e.g. the density or temperature, is needed by another part of the simulation. How these quantities are calculated is defined in the :literal:`TabulatedAtmosphere` class. 

Now that the creation of an environment model is understood, it can be discussed what should change in the Tudat code when something is added. There are two options when changing the code: the user can modify an existing environment model, or they can add a new model to an existing type of environment model. These type of modifications require different amount of changes made to Tudat and are thus explained seperately here:

- **Modify existing environment model:** this option is the easiest as it only requires changes in the specific environment type files. If a function is added to an environment model, it is important to also include this function in the base class file of that specific environment model, with the virtual statement. For example, take the :literal:`getDensity()` function. This function is put in the file :literal:`AtmosphereModel.h` as a pure virtual function (don't need a function definition), by putting a :literal:`=0` after the function definition and including the virtual statement. Now, in every derived class, this function should return something, depending on the implementation. If only something inside an already existing function needs to be changed, it (most of the time) shouldn't be changed in all the derived classes. If a variable is added to the constructor of the class, it is important that all the cases that this class is called should be changed in the entire Tudat code to prevent errors (use the :literal:`find usages` option in Qt to find them). 

- **Add a new environment model:** this modification requires some extra changes to the framework of the environment model implementation. First, when the new model is made, make sure that it is derived from the base class, and that it contains some of the basic functions. Once the new model is made, a :literal:`Settings` class should be made in the same way as :literal:`TabulatedAtmosphereSettings`. This class should be added to the :literal:`create...` files, and should contain functions that store variables that are used to call the constructor of the corresponding class (again, make sure it is derived from the proper base class). Then, in the corresponding :literal:`.cpp` file, a new case for the new environment model should be added to the :literal:`create...` function, just as in the tabulated atmosphere example. Make sure that in the :literal:`getAtmosphereType( )` function, the name of the new model is also included. In the case statement, make sure to add checks, and throw runtime errors if they are violated. Another step that needs to be taken is to update the :literal:`createEnvironmentUpdater.cpp` file. This file includes several switch statements that need to have the new acceleration model in it. Use the existing code to determine how the new case should be made. 

Adding a New Acceleration Model
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
For some applications, the various acceleration models in Tudat might not be sufficient. This guide will explain how acceleration models are set-up within Tudat, and how these models can be modified, or how to add a completely new model. 

Just as before, it is important to understand the framework of the acceleration models. This will be explained using an example, namely the aerodynamic acceleration model. The first time the acceleration models will be needed is in the user's own code. An example of how this can be done is shown below:

.. code-block:: cpp

    // Define propagator settings variables.
    SelectedAccelerationMap accelerationMap;
    std::vector< std::string > bodiesToPropagate;
    std::vector< std::string > centralBodies;

    // Define acceleration model settings.
    std::map< std::string, std::vector< std::shared_ptr< AccelerationSettings > > > accelerationsOfWaverider;
    accelerationsOfWaverider[ "Mars" ].push_back( std::make_shared< AccelerationSettings >( aerodynamic ) );
    accelerationMap[  "Vehicle" ] = accelerationsOfWaverider;

    bodiesToPropagate.push_back( "Vehicle" );
    centralBodies.push_back( "Mars" );

    // Create acceleration models
    basic_astrodynamics::AccelerationMap accelerationModelMap = createAccelerationModelsMap(
    bodyMap, accelerationMap, bodiesToPropagate, centralBodies );

The user will define a special map, with type :literal:`SelectedAccelerationMap` that works as follows: the first key defines the body exerting the acceleration, the second key the body undergoing the acceleration, and the value in the map is a vector of pointers to a type class called :literal:`AccelerationSettings`, which takes as input an :literal:`enum` called :literal:`AvailableAcceleration`, which lists all the available acceleration models. This map is then used as an input for the :literal:`createAccelerationModelsMap` function, which is a function that creates a special acceleration map that can be used as an input to the propagator settings. 

There are two functions, located in :literal:`CreateAccelerationModels.cpp`, called :literal:`createAccelerationModelsMap`. The function that is called will first order the map, and then return the value of the other function, which takes as input the ordered map, made in the first :literal:`createAccelerationModelsMap`. The second function looks as follows:

.. code-block:: cpp

    //! Function to create a set of acceleration models from a map of bodies and acceleration model types.
    basic_astrodynamics::AccelerationMap createAccelerationModelsMap(
           const NamedBodyMap& bodyMap,
           const SelectedAccelerationMap& selectedAccelerationPerBody,
           const std::map< std::string, std::string >& centralBodies )
    {
	...
	...
	...
	    currentAcceleration = createAccelerationModel( bodyMap.at( bodyUndergoingAcceleration ),
                                                               bodyMap.at( bodyExertingAcceleration ),
                                                               accelerationsForBody.at( i ).second,
                                                               bodyUndergoingAcceleration,
                                                               bodyExertingAcceleration,
                                                               currentCentralBody,
                                                               currentCentralBodyName,
                                                               bodyMap );


            // Create acceleration model.
            mapOfAccelerationsForBody[ bodyExertingAcceleration ].push_back(
                         currentAcceleration );

	...
	...
	...
	    // Put acceleration models on current body in return map.
            accelerationModelMap[ bodyUndergoingAcceleration ] = mapOfAccelerationsForBody;
        }
        return accelerationModelMap;
    }

This function does several checks if the right models are present, and then calls the :literal:`createAccelerationModel( )` function. This function is where the acceleration models are called and looks as follows:

.. code-block:: cpp

    //! Function to create acceleration model object.
    std::shared_ptr< AccelerationModel< Eigen::Vector3d > > createAccelerationModel(
            const std::shared_ptr< Body > bodyUndergoingAcceleration,
            const std::shared_ptr< Body > bodyExertingAcceleration,
            const std::shared_ptr< AccelerationSettings > accelerationSettings,
            const std::string& nameOfBodyUndergoingAcceleration,
            const std::string& nameOfBodyExertingAcceleration,
            const std::shared_ptr< Body > centralBody,
            const std::string& nameOfCentralBody,
            const NamedBodyMap& bodyMap )
    {
        // Declare pointer to return object.
        std::shared_ptr< AccelerationModel< Eigen::Vector3d > > accelerationModelPointer;

        // Switch to call correct acceleration model type factory function.
        switch( accelerationSettings->accelerationType_ )
        {
 	...
	...
	...
        case aerodynamic:
            accelerationModelPointer = createAerodynamicAcceleratioModel(
                        bodyUndergoingAcceleration,
                        bodyExertingAcceleration,
                        nameOfBodyUndergoingAcceleration,
                        nameOfBodyExertingAcceleration );
            break;
	...
	...
	...
        default:
            throw std::runtime_error(
                        std::string( "Error, acceleration model ") +
                        std::to_string( accelerationSettings->accelerationType_ ) +
                        " not recognized when making acceleration model of" +
                        nameOfBodyExertingAcceleration + " on " +
                       nameOfBodyUndergoingAcceleration );
            break;
        }
        return accelerationModelPointer;
    }

Ths function uses a switch statement on the :literal:`accelerationSettings->accelerationType_` (which was set by the user in the beginning), to determine which acceleration model needs to be called. The :literal:`createAerodynamicAcceleratioModel( )` then does several checks to determine if all the models that are used to calculate the aerodynamic force are present, after which it calls the constructor of the :literal:`AerodynamicAcceleration` class using the input values created in :literal:`createAerodynamicAcceleratioModel( )`. The structure of the :literal:`AerodynamicAcceleration` class, and how one can make a new acceleration model will be discussed in the next section. 

When making a new acceleration model, there are two things that need to be added first. The first one is a new class containing functions to calculate the accelerations acting on the respective body. It is important that this class is derived from the base class: :literal:`basic_astrodynamics::AccelerationModel< dataType >`, where :literal:`dataType` is the type of the acceleration variable. Furthermore, there are two functions that need to be present in the new acceleration model class:


- :literal:`void updateMembers(const double currentTime = TUDAT_NAN)`. This function updates all the member variables to the current situation so they can be used by the other necessary function.
- :literal:`dataType getAcceleration( )`. This function uses the recently updated member variables to calculate the acceleration acting on the body. 

For the case of the aerodynamic acceleration, these two functions look as follows:

.. code-block:: cpp


    Eigen::Vector3d getAcceleration( )
    {
        return computeAerodynamicAcceleration(
                    0.5 * currentDensity_ * currentAirspeed_ * currentAirspeed_,
                    currentReferenceArea_, currentForceCoefficients_, currentMass_ );
    }


    void updateMembers( const double currentTime = TUDAT_NAN )
    {
        if( !( this->currentTime_ == currentTime ) )
        {
            currentForceCoefficients_ = coefficientMultiplier_ * this->coefficientFunction_( );
            currentDensity_ = this->densityFunction_( );
            currentMass_ = this->massFunction_( );
            currentAirspeed_ = this->airSpeedFunction_( );
            currentReferenceArea_ = this->referenceAreaFunction_( );

            currentTime_ = currentTime;
        }
    }

where :literal:`computeAcceleration()` is a function that uses the well known lift, side, and drag acceleration equations to calculate the acceleration vector.

The second piece of code that needs to be added is the new create function inside :literal:`createAccelerationModel.cpp`. This function should return a pointer to the respective acceleration class, and takes as input the various models needed for the calculation of the acceleration. The main goal of this function is to do some checks if the necessary models are present, and either throw an error, or initialize the model there. When this is done, it uses all the available models as input to the acceleration model, and returns a pointer to this class. 

Once the acceleration class is made, and the create function is in place, the acceleration model should be implemented into the acceleration framework. The first step for this is to add the a new model to the :literal:`AvailableAcceleration` enum. This name could be anything, as long as it relates to the actual acceleration model. Nothing has to be added to the :literal:`createAccelerationModelsMap`, however, the :literal:`createAccelerationModel` function has a switch statement that needs to be altered. This switch statement checks the :literal:`accelerationSettings` of the specific body to see which acceleration is acting on the body. When a new acceleration model is made, a new case should be made for the new model. This case should have the same name as the name added to the :literal:`AvailableAcceleration` enum. Inside this case, the variable :literal:`accelerationModelPointer` should be assigned to the create acceleration model function, which was made before. When this is done, the new acceleration model should be incorporated into the acceleration framework of tudat.

Adding a New State Derivative Model
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Another type of model which allows the user to add extra features to it, is the state derivative model. These models are used by the dynamics simulator to determine how the state equations are solved. Currently there are several available, e.g. cowell state derivative model, encke state derivate model, and more, but if a special model is needed for a certain application, the user can add this to tudat by following this guide.

As before, this guide will start by looking at the framework of how the state derivative model is implemented. First, the state derivative model is chosen by the user from an enum called :literal:`TranslationalPropagatorType`, located in :literal:`nBodyStateDerivativeModel.h`. The model picked from this enum is then used as an input into the constructor of the :literal:`TranslationalPropagatorSettings`, where it is assigned to a specific member variable called: :literal:`propagator_`. The :literal:`TranslationalPropagatorSettings` is used as an input to the :literal:`SingleArcDynamicsSimulator`, where it will be used further. An example of this is shown below:

.. code-block:: cpp


     // Create propagation settings.
     std::shared_ptr< TranslationalStatePropagatorSettings< double > > propagatorSettings =
             std::make_shared< TranslationalStatePropagatorSettings< double > >
             ( centralBodies, accelerationModelMap, bodiesToPropagate, systemInitialState,
               terminationSettings, cowell, dependentVariablesToSave );

    // Create simulation object and propagate dynamics.
    SingleArcDynamicsSimulator< > dynamicsSimulator(
	     bodyMap, integratorSettings, propagatorSettings );


Inside :literal:`SingleArcDynamicsSimulator`, the propagator settings are passed to various functions, but the state derivative model is only needed in :literal:`createStateDerivativeModels`. This function returns a vector of :literal:`SingleStateTypeDerivative`. If a hybrid state derivative model is used, this function will fill the vector with state derivative models. If not, this vector will only contain one state derivative model, created by the function (equally) called :literal:`createStateDerivativeModels`. This function checks what kind of state is propagated (translational, rotational, etc.) and creates the specific state derivative model. For example, for the translational state it will call the :literal:`createTranslationalStateDerivativeModel` function. This function looks as follows:

.. code-block:: cpp


     template< typename StateScalarType = double, typename TimeType = double >
     std::shared_ptr< SingleStateTypeDerivative< StateScalarType, TimeType > >
     createTranslationalStateDerivativeModel(
            const std::shared_ptr< TranslationalStatePropagatorSettings< StateScalarType > >
            translationPropagatorSettings,
            const simulation_setup::NamedBodyMap& bodyMap,
            const TimeType propagationStartTime )
    {

        // Create object for frame origin transformations.
        std::shared_ptr< CentralBodyData< StateScalarType, TimeType > > centralBodyData =
                createCentralBodyData< StateScalarType, TimeType >(
                    translationPropagatorSettings->centralBodies_,
                    translationPropagatorSettings->bodiesToIntegrate_,
                    bodyMap );

        std::shared_ptr< SingleStateTypeDerivative< StateScalarType, TimeType > > stateDerivativeModel;

        // Check propagator type and create corresponding state derivative object.
        switch( translationPropagatorSettings->propagator_ )
        {
        case cowell:
        {
            stateDerivativeModel = std::make_shared<
                    NBodyCowellStateDerivative< StateScalarType, TimeType > >
                    ( translationPropagatorSettings->getAccelerationsMap( ), centralBodyData,
                      translationPropagatorSettings->bodiesToIntegrate_ );
            break;
        }
        ...
        ...
        ...
        }
        default:
            throw std::runtime_error(
                        "Error, did not recognize translational state propagation type: " +
                        std::to_string( translationPropagatorSettings->propagator_ ) );
        }
        return stateDerivativeModel;
    }

This function is responsible for making the state derivate model based on the :literal:`TranslationalPropagatorType` variable decalred by the user, using a switch statement. 

When a new state derivative model is implemented in tudat, two additions need to be made to the framework (besides making the model itself). The first part is to add the name of the model to the enum: :literal:`TranslationalPropagatorType`. This will allow the user to pick this model and use it as an input to the translational propagator settings. The second step is to add this model to the switch statement in :literal:`createTranslationalStateDerivativeModel`. A new case should be made for the name added to the enum before, and in it, the variable :literal:`stateDerivativeModel` should be assigned to the new model.

When building the new model, it is advised to use a state derivative model that is already available as an example for a place to start. The classes which contain these models are derived from the base class :literal:`NBodyStateDerivative`, and should contain a template for the :literal:`TimeType` that will be used. The :literal:`NBodyStateDerivative` class is again derived from another class called the :literal:`SingleStateTypeDerivative`. This class contains several pure virtual functions, which all should be added to the new model class in order for the new model to work. The specific names and input parameters of these functions can be found in the :literal:`SingleStateTypeDerivative` class. Once this is done, and the new model is implemented in the state derivative model framework, the new model should be available for the user. 

.. note:: Don't forget to put the include statement in :literal:`createStateDerivativeModel.h` if the new class is made in a seperate file.


Adding a New Estimatable Parameter
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The list of estimatable parameters already available in Tudat is presented in :ref:`parameterEstimationSettings`. However, it is possible to add another parameter to this list of estimatable parameters if needed. This process requires to modify several files located in different directories and  will be described in details based on the examples of the two parameters :literal:`radiation_pressure_coefficient` and :literal:`rotation_pole_position`. 

First of all, the name of the new estimatable parameter has to be added to the list of the estimatable parameters available in Tudat, in the file :literal:`estimatableParameter.h`:

.. code-block:: cpp

   //! List of parameters that can be estimated by the orbit determination code
   enum EstimatebleParametersEnum
   {
      arc_wise_initial_body_state,
      initial_body_state,
      initial_rotational_body_state,
      gravitational_parameter,
      constant_drag_coefficient,
      radiation_pressure_coefficient,
      arc_wise_radiation_pressure_coefficient,
      spherical_harmonics_cosine_coefficient_block,
      spherical_harmonics_sine_coefficient_block,
      constant_rotation_rate,
      rotation_pole_position,
      constant_additive_observation_bias,
      ...
      ...
   }

In addition to the file :literal:`estimatableParameters.h`, the file :literal:`estimatableParameters.cpp` also has to be modified. In particular, a short description of each estimatable parameter has to be provided in the function :literal:`getParameterTypeString`, as it is done in the following for the parameters :literal:`radiation_pressure_coefficient` and :literal:`rotation_pole_position`:

.. code-block:: cpp
   
   std::string getParameterTypeString( const EstimatebleParametersEnum parameterType )
   {
   ...
   case radiation_pressure_coefficient:
        parameterDescription = "radiation pressure coefficient ";
        break;
   ...
   case rotation_pole_position:
        parameterDescription = "pole position ";
        break;
   ...
   }

The type of the new estimatable parameter must then be specified within the function :literal:`isDoubleParameter` (still inside the file :literal:`estimatableParameters.cpp`). An estimatable parameter can either be a :literal:`double` or a :literal:`Eigen::VectorXd`. Regarding the two examples which are considered here, the parameter :literal:`radiation_pressure_coefficient` is a :literal:`double` while the parameter :literal:`rotation_pole_position` is a :literal:`Eigen::VectorXd`, whose first element is the right ascension of the rotation pole and the second one its declination.

.. code-block:: cpp
   
   bool isDoubleParameter( const EstimatebleParametersEnum parameterType )
   {
   ...
   case radiation_pressure_coefficient:
        isDoubleParameter = true;
        break;
   ...
   case rotation_pole_position:
        isDoubleParameter = false;
        break;
   ...
   }

Finally, depending on which estimatable parameter is to be added to the list of available parameters, the functions :literal:`isParameterDynamicalPropertyInitialState`, :literal:`isParameterRotationMatrixProperty`, :literal:`isParameterObservationLinkProperty` and :literal:`isParameterTidalProperty` might also need to be modified to take this new parameter into account (again in :literal:`estimatableParameters.cpp`). They usually return a boolean which is set to false as default value, but a specific case has to added for the new parameter if it is related to either the initial state, a rotation matrix, an observation link or any tidal property. The parameter :literal:`radiation_pressure_coefficient` is related to none of them but the parameter :literal:`rotation_pole_position` is linked to a rotation matrix so that the function :literal:`isParameterRotationMatrixProperty` includes a case switching the boolean to :literal:`true` for this parameter.

.. code-block:: cpp
   
   bool isParameterRotationMatrixProperty( const EstimatebleParametersEnum parameterType )
   {
      bool flag;
      switch( parameterType )
      {
      ...
      case rotation_pole_position:
         flag = true;
         break;
      }
      return flag;
   }    

Once the new estimatable parameter has been defined and characterised, a corresponding class is to be created to fully describe this parameter. This is done in an additional file which has to be created in the directory:

      .../tudatBundle/tudat/Tudat/Astrodynamics/OrbitDetermination/EstimatableParameters/

Each estimatable parameter class is defined in a similar way, usually from the base class :literal:`EstimatableParameter` and includes the definition of three functions :literal:`getParameterValues`, :literal:`setParameterValue` and :literal:`getParameterSize`. Considering the parameter :literal:`radiation_pressure_coefficient`, the definition of its associated class :literal:`RadiationPressureCoefficient` is done as follows (in the file :literal:`radiationPressureCoefficient.h`).

.. code-block:: cpp

   class RadiationPressureCoefficient: public EstimatableParameter< double >
   {

   public:
   //! Constructor.
   RadiationPressureCoefficient(std::shared_ptr< electro_magnetism::RadiationPressureInterface > radiationPressureInterface, std::string& associatedBody ):
   EstimatableParameter< double >( radiation_pressure_coefficient, associatedBody ), radiationPressureInterface_( radiationPressureInterface )
   { }

   //! Destructor.
   ~RadiationPressureCoefficient( ) { }

   //! Function to get the current value of the radiation pressure coefficient that is to be estimated.
   double getParameterValue( )
   {
       return radiationPressureInterface_->getRadiationPressureCoefficient( );
   }

   //! Function to reset the value of the radiation pressure coefficient that is to be estimated.
   void setParameterValue( double parameterValue )
   {
       radiationPressureInterface_->resetRadiationPressureCoefficient( parameterValue );
   }

   //! Function to retrieve the size of the parameter.
   int getParameterSize( ){ return 1; }

   protected:

   private:

   //! Object containing the radiation pressure coefficient to be estimated.
   std::shared_ptr< electro_magnetism::RadiationPressureInterface > radiationPressureInterface_;
   };

For the parameter :literal:`rotation_pole_position`, the class :literal:`ConstantRotationalOrientation` is created in a very similar way, in a file :literal:`constantRotationalOrientation.h`, keeping in mind that this parameter is a :literal:`Eigen::VectorXd` whose size is 2 and not a double as it is the case for the radiation pressure coefficient.
    

Then an interface object has to be created to estimate the new parameter. This is done in the file :literal:`createEstimatableParameters.cpp`, either with the function :literal:`createDoubleParameterToEstimate` if the estimatable parameter is of type :literal:`double` or with :literal:`createVectorParameterToEstimate` if it is a :literal:`Eigen::VectorXd` object. These functions take as input an :literal:`EstimatableParameterSettings` object from which the settings of the parameter to be estimated are checked to ensure that it is properly defined and thus make the estimation possible. In some cases, the base class :literal:`EstimatableParameterSettings` is not detailed enough to give access to all the required properties of the estimatable parameter and a specific class has to be defined. As an example, the parameter :literal:`spherical_harmonics_cosine_coefficient_block` requires the definition of the estimatable parameter settings class called :literal:`SphericalHarmonicEstimatableParameterSettings` (defined in the file :literal:`estimatableParameterSettings.h`). This class manages the different combinations of degrees and orders of the spherical harmonic coefficients that have to be estimated.

Regarding the verification of the estimatable parameter settings of :literal:`radiation_pressure_coefficient`, it is checked that only one single radiation pressure interface is defined before creating the parameter and linking it to this radiation pressure interface and to the propagated body. 

.. code-block:: cpp

   std::shared_ptr< EstimatableParameter< double > > createDoubleParameterToEstimate(
        const std::shared_ptr< EstimatableParameterSettings >& doubleParameterName,
        const NamedBodyMap& bodyMap, const basic_astrodynamics::AccelerationMap& accelerationModelMap )
   {
   ...
   case radiation_pressure_coefficient:
        {
            if( currentBody->getRadiationPressureInterfaces( ).size( ) == 0 )
            {
                std::string errorMessage = "Error, no radiation pressure interfaces found in body " +
                        currentBodyName + " when making Cr parameter.";
                throw std::runtime_error( errorMessage );
            }
            else if( currentBody->getRadiationPressureInterfaces( ).size( ) > 1 )
            {
                std::string errorMessage = "Error, multiple radiation pressure interfaces found in body " +
                        currentBodyName + " when making Cr parameter.";
                throw std::runtime_error( errorMessage );
            }
            else
            {
                doubleParameterToEstimate = std::make_shared< RadiationPressureCoefficient >(
                            currentBody->getRadiationPressureInterfaces( ).begin( )->second,
                            currentBodyName );
            }
            break;
        }
    ...
   }

Concerning the parameter :literal:`rotation_pole_position`, it must be verified that the rotation model is a simple rotational ephemeris for which the position of the rotation pole is indeed defined before creating the estimatable parameter.

.. code-block:: cpp

   std::shared_ptr< EstimatableParameter< Eigen::VectorXd > > createVectorParameterToEstimate(
        const std::shared_ptr< EstimatableParameterSettings >& vectorParameterName,
        const NamedBodyMap& bodyMap, const basic_astrodynamics::AccelerationMap& accelerationModelMap )
   {
   ...
   case rotation_pole_position:
            if( std::dynamic_pointer_cast< SimpleRotationalEphemeris >( currentBody->getRotationalEphemeris( ) ) == nullptr )
            {
                std::string errorMessage = "Warning, no simple rotational ephemeris present in body " + currentBodyName +
                        " when making constant rotation orientation parameter";
                throw std::runtime_error( errorMessage );
            }
            else
            {
                vectorParameterToEstimate = std::make_shared< ConstantRotationalOrientation >
                        ( std::dynamic_pointer_cast< ephemerides::SimpleRotationalEphemeris >
                          ( currentBody->getRotationalEphemeris( ) ), currentBodyName );

            }
            break;
     ...
     }

To allow the parameter estimation to be conducted, partials with respect to this parameter have to be implemented. They can either be cartesian state or acceleration partials. In the former case, the partials of the cartesian state of the propagated body with respect to the estimatable parameter have to be implemented in the file :literal:`createCartesianStatePartial.cpp`. These partials are defined within the functions :literal:`createCartesianStatePartialsWrtParameter` (two functions exist with two different input types, depending on the type of the parameter (:literal:`double` or :literal:`Eigen::VectorXd`) that is to be considered). Three cases have to be distinguished here. First, if the estimatable parameter has been identified as being a property of a rotation matrix, then the function :literal:`createCartesianStatePartialsWrtParameter` calls another function named :literal:`createRotationMatrixPartialsWrtParameter` and defined in :literal:`createCartesianStatePartial.cpp` too (again two functions with the same name exist for the two types of estimatable parameters). A specific case has to be added within this function for each parameter which is related to a rotation matrix. For the parameter :literal:`rotation_pole_position`, the following lines of code have been added to first check that the rotation model is consistent with the estimatable parameter (here that is a simple rotational model) and then to call a function that returns the required rotation matrix  partials for this particular model.

.. code-block:: cpp

   std::shared_ptr< RotationMatrixPartial > createRotationMatrixPartialsWrtParameter(
        const simulation_setup::NamedBodyMap& bodyMap,
        const std::shared_ptr< estimatable_parameters::EstimatableParameter< Eigen::VectorXd > > parameterToEstimate )

  {
     ...
     case estimatable_parameters::rotation_pole_position:
        if( std::dynamic_pointer_cast< ephemerides::SimpleRotationalEphemeris >(
                    currentBody->getRotationalEphemeris( ) ) == nullptr )
        {
            std::string errorMessage = "Warning, body's rotation model is not simple when making position w.r.t. pole position partial";
            throw std::runtime_error( errorMessage );
        }
        // Create rotation matrix partial object
        rotationMatrixPartial = std::make_shared< RotationMatrixPartialWrtPoleOrientation >(
                    std::dynamic_pointer_cast< SimpleRotationalEphemeris>( currentBody->getRotationalEphemeris( ) ) );
        break;

   ...
   }

The class :literal:`RotationMatrixPartialWrtPoleOrientation` used in the code above to define the :literal:`rotationMatrixPartial` variable is created in the file :literal:`rotationMatrixPartial.h` and must contain two internal functions :literal:`calculatePartialOfRotationMatrixToBaseFrameWrtParameter` and :literal:`calculatePartialOfRotationMatrixDerivativeToBaseFrameWrtParameter`. These functions return the rotation matrix and rotation matrix derivative partials respectively, with respect to the estimatable parameter. Specific functions to calculate these partials have to be added to the file :literal:`rotationMatrixPartial.cpp`. Regarding the parameter :literal:`rotation_pole_position`, these functions are :literal:`calculatePartialOfRotationMatrixFromLocalFrameWrtPoleOrientation` and :literal:`calculatePartialOfRotationMatrixDerivativeFromLocalFrameWrtPoleOrientation`, respectively.

So far, we have only considered the case where the estimatable parameter is related to a rotation matrix. However, when it is not the case, the function :literal:`createCartesianStatePartialsWrtParameter` is to be modified in a different way. A specific case has to be created for each parameter that is not a rotation matrix property. If the parameter has a direct impact on the cartesian state of the propagated body (eg :literal:`ground_station_position`), the partials of the cartesian state with respect to the parameters must be directly returned by the function. 

A last case arises when the estimatable parameter neither is  a rotation matrix property nor has a direct influence on the cartesian state of the propagated body but is involved in one of the acceleration models. The function :literal:`createCartesianStatePartialsWrtParameter` does not return any partial in that case. The estimation of such a parameter then requires the use of acceleration partials with respect to this parameter. The acceleration partials are implemented in the files of the directory:

   .../tudatBundle/tudat/Tudat/Astrodynamics/OrbitDetermination/AccelerationPartials/

For any estimatable parameter related to an acceleration model, one must define a function :literal:`getParameterPartialFunction` which is a method of its associated acceleration partial clas. This function provides the partial of the acceleration model with respect to the estimatable parameter. Looking at the parameter :literal:`radiation_pressure_coefficient` in particular, the acceleration partials are defined from the class :literal:`CannonBallRadiationPressurePartial` which is itself derived from the base class :literal:`AccelerationPartial`). This acceleration partials class is defined in the files :literal:`radiationPressureAccelerationPartial.h` and :literal:`radiationPressureAccelerationPartial.cpp`. Only the radiation pressure coefficient can be estimated for this acceleration model so that the function :literal:`getParameterPartialFunction` only contains a single specific case dedicated to this parameter. 

.. code-block:: cpp

   //! Function for setting up and retrieving a function returning a partial w.r.t. a double parameter.
   std::pair< std::function< void( Eigen::MatrixXd& ) >, int > CannonBallRadiationPressurePartial::getParameterPartialFunction(
        std::shared_ptr< estimatable_parameters::EstimatableParameter< double > > parameter )
   {
      std::function< void( Eigen::MatrixXd& ) > partialFunction;
      int numberOfRows = 0;

      // Check if parameter dependency exists.
      if( parameter->getParameterName( ).second.first == acceleratedBody_ )
      {
         switch( parameter->getParameterName( ).first )
         {
            // Set function returning partial w.r.t. radiation pressure coefficient.
            case estimatable_parameters::radiation_pressure_coefficient:

               partialFunction = std::bind( &CannonBallRadiationPressurePartial::wrtRadiationPressureCoefficient, this, std::placeholders::_1 );
               numberOfRows = 1;

               break;
            default:
               break;
          }
       }
       return std::make_pair( partialFunction, numberOfRows );
    } 


The function :literal:`wrtRadiationPressureCoefficient` called in the piece of code above is also a method of the class :literal:`CannonBallRadiationPressurePartial` and is defined in the file :literal:`radiationPressureAccelerationPartial.h`. It directly returns the partial derivative of the radiation pressure acceleration model with respect to the radiation pressure coefficient, as follows:

.. code-block:: cpp

   void wrtRadiationPressureCoefficient( Eigen::MatrixXd& partial )
   {
        partial = computePartialOfCannonBallRadiationPressureAccelerationWrtRadiationPressureCoefficient(
                    radiationPressureFunction_( ), areaFunction_( ), acceleratedBodyMassFunction_( ),
                    ( sourceBodyState_( ) - acceleratedBodyState_( ) ).normalized( ) );
   }


