.. _using_tespy_label:

~~~~~~~~~~~
Using TESPy
~~~~~~~~~~~

TESPy provides a simulation package for component based thermal engineering containing the most important
components of such plants. In the introduction you will learn the basics of modelling component based
plants in TESPy. We then give an overview on the main TESPy modules:

 * tespy.networks,
 * tespy.components,
 * tespy.connections,
 * tespy.network_reader and
 * tespy.tools.

At the end of this page we give a brief overview how TESPy handles fluid properties.

.. figure:: api/_images/tutorial_heat_pump.svg
    :align: center

    Figure 1: Topology of a heat pump.

.. _using_tespy_introduction_label:

We highly recommend to check our :ref:`step by step tutorial <heat_pump_tutorial_label>` on how to
set up a heat pump in TESPy. You will learn, how to set up and design a plant as well as calculate offdesign/partload performance.
Additionally we provide basic examples in the :ref:`examples section <tespy_examples_label>`.

.. contents:: `Contents`
    :depth: 1
    :local:
    :backlinks: top


Introduction
============

Set up a plant
--------------

In order to simulate a plant you will have to create a tespy.network first. The network is the main container for the model.

You need to specify a list of the fluids you need for the calculation in your plant. For more information on the fluid properties jump to the :ref:`bottom of this page <tespy_fluid_properties_label>`.

.. code-block:: python

	from tespy import nwk
	# create a network object with air and water as fluids
	fluid_list = ['air', 'water']
	my_plant = nwk.network(fluids=fluid_list)

On top of that, it is possible to specify a unit system and value ranges for the networks variables. If you do not specify these, TESPy will use SI-units.
The specification of the **value range** is used to **improve convergence stability**, in case you are dealing with **fluid mixtures**, e. g. using a combustion chamber.

.. code-block:: python

	from tespy import nwk

	# set the unitsystem for temperatures to °C, for pressure to bar and enthalpy to kJ / kg
	my_plant.set_attr(T_unit='C', p_unit='bar', h_unit='kJ / kg')
	my_plant.set_attr(T_range=[10, 700], p_range=[0.05, 150], h_range=[15, 4000])

Now you can start to create the components of the network.

Set up components
-----------------

Available components can be found :ref:`here <using_tespy_components_label>`. If you set up a component you have to specify a (within one network) unique label.
Moreover, it is possible to specify parameters for the component, for example power P for a turbine or upper terminal temperature difference ttd_u of a heat exchanger.
The full list of parameters for a specific component (e. g. a valve) is stated in the classes documentation.

.. note::
	Parameters for components are generally optional. Only the components label and in case you want to use a combustion chamber, the combustion chambers fuel, are mandatory parameters to provide.
	If an optional parameter is not specified by the user, it will be a result of the plants simulation. In this way, the set of equations a component returns is determined by which parameters you specify.
	You can find all equations in the :ref:`components documentation <using_tespy_components_label>` as well. The example below shows how to create a component with specific parameters, set or reset and how to unset a parameter:

.. code-block:: python

	from tespy import cmp
	import numpy as np

	feed_water_pump = cmp.pump(label='hp pump', P=1e3) # create pump labeled 'hp pump'
	feed_water_pump.set_attr(P=2e3, eta_s=0.9) # set the power to 2000 W, set isentropic efficiency to 90 %
	feed_water_pump.set_attr(P=np.nan) # unset power

After setting up the components the next step is to connect the components in your network.

Establish connections
---------------------

Connections are used to link two components (outlet of component 1 to inlet of component 2, source to target).
If two components are connected to each other the fluid properties at the source will be equal to the properties at the target.
It is possible to set the properties on each connection in a similar way as parameters are set for components. You may specify:

 * mass flow* (m),
 * volumetric flow (v),
 * pressure* (p),
 * enthalpy* (h),
 * temperature* (T),
 * vapour mass fraction for pure fluids (x),
 * a fluid vector (fluid) and
 * a balance closer for the fluid vector (fluid_balance).

All parameters but the fluid vector have to be numeric values. The fluid vector has to be specified as dictonary, see the example below.
The parameter :code:`fluid_balance` can only be :code:`True` or :code:`False`. For the properties marked with * it is possible to use references instead of numeric values.
This can be used for example if you want to have the pressure in two parts of your network related in a specific way but you do not know the values prior to the plant simulation.

.. code-block:: python

	from tespy import con

	ws_cond = con.connection(waste_steam_source, 'out1', condenser, 'in1', x=0.97) # waste steam source to condenser hot side inlet and setting vapour mass fraction
	cond_cp = con.connection(condenser, 'out1', condensate_pump, 'in1', fluid={'water': 1, 'air': 0}) # setting a fluid vector: {'fluid i': mass fraction i}
	cp_fwt = con.connection(condensate_pump, 'out1', feed_water_tank, 'in1')
	fwt_fwp = con.connection(feed_water_tank, 'out1', feed_water_pump, 'in1') # connection without parameter specification
	fwp_eco = con.connection(feed_water_pump, 'out1', economiser, 'in2', v=10) #  setting volumetric flow
	eco_drum = con.connection(economiser, 'out2', drum, 'in1', T=320, p=con.ref(fwp_eco, 0.98, 0)) # setting temperature and pressure via reference object (pressure at this point is 0.98 times of pressure at connection fwp_eco)
	eva_eco = con.connection(evaporator, 'out1', economiser, 'in1', T=350, m=100) # setting temperature and mass flow
	eco_fgs = con.connection(economiser, 'out1', flue_gas_sink, 'in1', fluid_balance=True, fluid={'air': 1}, p=1) # setting fluid vector partially as well as the fluid balance parameter and pressure

	# this line is crutial, you have to add all connections to your network!
	my_plant.add_conns(ws_cond, cond_cp, cp_fwt, fwt_fwp, fwp_eco, eco_drum, eva_eco, eco_fgs)

.. figure:: api/_images/intro_connections.svg
    :align: center

    Figure 2: Topology after defining the above connections.

If you want to set, reset or unset a connection parameter the same logic as for the components is applied.

.. code-block:: python

	ws_cond.set_attr(x=0.95, p=0.05) # reset vapour mass fraction, set pressure
	eco_drum.set_attr(p=np.nan) # unset pressure

Start your calculation
----------------------

After building your network, the components and the connections, add the following line at the end of your script and off you go:

.. code-block:: python

	my_plant.solve(mode='design')

Please be aware, that the execution of the lines of code above will not create a solvable TESPy network. For good first examples jump to the :ref:`TESPy examples <tespy_examples_label>`.

In order to get a good overview of the TESPy functionalities, the following sections will walk you through the different TESPy modules in detail.


.. _using_tespy_networks_label:

TESPy networks
==============

The tespy.networks.network class handles preprocessing, solving and postprocessing. We will walk you through all the important steps.

Setup
-----

Network container
^^^^^^^^^^^^^^^^^

The TESPy network contains all data of your plant, which in terms of the calculation is represented by a nonlinear system of equations. The system variables of your TESPy network are:

 * mass flow,
 * pressure,
 * enthalpy and
 * the mass fractions of the network's fluids.

The solver will solve for these variables. As stated in the introduction the list of fluids is passed to your network on creation.
If your **system includes fluid mixtures**, you should **always make use of the value ranges** for the system variables. This improves the stability of the algorithm. Try to fit the boundaries as tight as possible,
for instance, if you kwow that the maximum pressure in the system will be at 10 bar, use it as upper boundary.

.. note::

	Value ranges for pure fluids are not required as these are dealt with automatically.

.. code-block:: python

    from tespy import nwk

	fluid_list = ['CO2', 'H2O', 'N2', 'O2', 'Ar']
	my_plant = nwk.network(fluids=fluid_list)
	my_plant.set_attr(p_unit='bar', h_unit='kJ / kg')
	my_plant.set_attr(p_range=[0.05, 10], h_range=[15, 2000])
	
.. _printout_logging_label:

Printouts and logging
+++++++++++++++++++++

TESPy comes with an inbuilt logger. If you want to keep track of debugging-messages, general information, warnings or errors you should enable the logger. At the beginning of your python script e. g. add the following lines:

.. code-block:: python

	from tespy.tools import logger
	import logging
	logger.define_logging(
		log_path=True, log_version=True,
		screen_level=logging.INFO, file_level=logging.DEBUG
	)
	
The log-file will be saved to :code:`~/.tespy/log_files/` by default. All available options are documented in the :py:func:`API <tespy.tools.logger.define_logging>`.

Prior to solving the network there are options regarding the **console printouts for the calculation progress** using the :py:meth:`set_printoptions method <tespy.networks.network.set_printoptions>`.
You can choose the print_level (info or none). Check out the :py:meth:`API-documentation <tespy.networks.network.set_printoptions>` for more information.

.. code-block:: python

	myplant.set_printoptions(print_level='none') # disabling iteration information printout

Adding connections
++++++++++++++++++

As seen in the introduction, you will have to create your networks from the components and the connections between them.
You can add connections directly or via subsystems and networks holding them by using the appropriate methods:

.. code-block:: python

	myplant.add_conns()
	myplant.add_subsys()
	myplant.add_nwks()

.. note::

	You do not need to add the components to the network, as they are inherited via the added connections.
	After having set up your network and added all required elements, you can start the calculation.

Busses: power connections
+++++++++++++++++++++++++

Another type of connection is the bus: Busses are power connections for e. g. turbomachines or heat exchangers. They can be used to model motors or generators, too. Add them to your network with the following method:

.. code-block:: python

	myplant.add_busses()
	
You will learn more about busses and how they work in :ref:`this part<tespy_busses_label>`.

Start calculation
^^^^^^^^^^^^^^^^^

You can start the solution process with the following line:

.. code-block:: python

	myplant.solve(mode='design')

This starts the initialisation of your network and proceeds to its calculation. The specification of the calculation mode is mandatory, see the list of available keywords:

 * :code:`mode` is the calculation mode (design-calculation or offdesign-calculation),
 * :code:`init_path` is the path to the network folder you want to use for initialisation,
 * :code:`design_path` is the path to the network folder which holds the information of your plants design point,
 * :code:`max_iter` is the maximum amount of iterations performed by the solver,
 * :code:`init_only` stop after initialisation (True/False).

There are two calculation modes available (:code:`'design'` and :code:`'offdesign'`), which are explained in the subsections below.
If you choose :code:`offdesign` as calculation mode the specification of a :code:`design_path` is mandatory.

The usage of an initialisation path is always optional but highly recommended, as the convergence of the solution process will be improved, if you provide good starting values.
If do not specify an :code:`init_path`, the initialisation from priorly saved results will be skipped.
:code:`init_only=True` usually is used for debugging. Or, you could use this feature to export a not solved network, if you want to do the parametrisation in .csv-files rather than your python script.

Design mode
+++++++++++

The design mode is used to design your system and is always the first calculation of your plant. **The offdesign calculation is always based on a design calculation!**.
Obviously as you are designing the plant the way you want, you are flexible to choose the parameters to specify.
However, you can't specify parameters that are based on a design case, as for example the isentropic efficiency characteristic function of a turbine or a pump. Specifying a value for the efficiency is of course possible.

Offdesign mode
++++++++++++++

The offdesign mode is used to **calulate the performance of your plant, if parameters deviate from the plant's design point**. This can be partload operation, operation at different temperature or pressure levels etc..
Thus, before starting an offdesing calculation you have to design your plant first. By stating :code:`'offdesign'` as calculation mode, **components and connections will auto-switch to the offdesign mode.**
For components, this means that all parameters provided in :code:`component.design` will be unset and instead all parameters provided in :code:`component.offdesign` will be set.
This applies to connections analogously. **The value of the newly set parameter is always equal to the value from the design case (or based on it for characteristics).**

.. code-block:: python

	myplant.solve(mode='offdesign', design_path='mynetwork')

.. note::

	Since version 0.1.0 there are no default design and offdesign parameters! All design and offdesign have to be specified manually as in the example below.

You can specify design and offdesign parameters for components and connections. For example, for a condenser you would usually design it to a maximum terminal temperature difference, in offdesign the heat transfer coefficient
is selected. The heat transfer coefficient is calculated in the preprocessing of the offdesign case based on the results of the design-case. Of course, this applies to all other parameters in the same way.
Also, the pressure drop is a result of the geometry for the offdesign case, thus we swap the pressure ratios with zeta values.

.. code-block:: python

	heat_ex.set_attr(design=['ttd_u', 'pr1', 'pr2'], offdesign=['kA', 'zeta1', 'zeta2'])
	
.. note::

	Some parameters come with characteristic functions based on the design case properties. This means, that e. g. the isentropic efficiency of a turbine is calculated as function of the actual mass flow to design mass flow ratio.
	You can provide your own (measured) data or use the already existing data from TESPy. All standard characteristic functions are available at :py:class:`tespy.components.characteristics.characteristics`. How to specify own data and all available characteristic functions are provided in :ref:`this section <component_characteristics_label>`.

If you want to **prevent the autoswitch from design to offdesign mode** for specific components, use :code:`heat_ex.set_attr(mode='man')`.

For connections it works in the same way, e. g. write

.. code-block:: python

	connection.set_attr(design=['h'], offdesign=['T'])

if you want to replace the enthalpy with the temperature for your offdesign. **The temperature is a result of the design calculation and that value is then used for the offdesign calculation in this example.**

Solving
-------

A TESPy network can be represented as a linear system of nonlinear equations, consequently the solution is obtained with numerical methods.
TESPy uses the n-dimensional Newton–Raphson method to find the systems solution, which may only be found, if the network is parameterized correctly.
**The number of variables n** is :math:`n = num_{conn} \cdot (3 + num_{fluids})`.

The algorithm requires starting values for all variables of the system, thus an initialisation of the system is runned prior to calculating the solution.
**High quality initial values are crutial for convergence speed and stability**, bad starting values might lead to instabilty and diverging calculation can be the result.
Thus there are different levels for the initialisation.

Initialisation
^^^^^^^^^^^^^^

The initialisation is performed in the following steps.

**General preprocessing:**

 * check network consistency and initialise components (if network topology is changed to a prior calculation only),
 * perform design/offdesign switch (for offdesign calculations only)

**Finding starting values:**

 * fluid propagation,
 * fluid property initialisation,
 * initialisation from .csv (preprocessing with :code:`design_path` for offdesign case and setting starting values with :code:`init_path`).

The network check is used to find errors in the network topology, the calulation can not start without a successful check. The component initialisation is important for components using characteristics and the combustion chamber,
a preprocessing of some parameters is required. The preprocessing for the components is performed in the :code:`comp_init` method of the components.
You will find the methods in the :py:mod:`components module <tespy.components.components>`. The design/offdesign switch is described in the network setup section.

**The fluid propagation is a very important step in the initialisation:** Often, you will specify the fluid at one point of the network only, thus all other connections are missing an initial information on the fluid vector,
if you are not using an :code:`init_path`. Also, you do not need to state a starting value for the fluid vector at every point of the network. The fluid propagation will push/pull the specified fluid through the network.
If you are using combustion chambers these will be starting points and a generic flue gas composition will be calculated prior to the propagation.

.. note::
	If the fluid propagation fails, you often experience an error, where the fluid property database can not find a value, because the fluid is 'nan'. Providing starting values manually can fix this problem.

The fluid property initialisation takes the user specified starting values if available and otherwise uses generic starting values on the bases of to which components the connection is linked to.

Last step is the initialisation from :code:`init_path`: For offdesign cases a preprocessing based on the :code:`design_path` in order to recreate the design case and set parameters based on the design case is performed.
If you specified an :code:`init_path` TESPy searches through the connections file for the network topology and if the corresponding connection is found, the starting values for the system variables are extracted from the connections file.
**The files do not need to contain all connections of your network, thus you can build up your network bit by bit and initialise the existing parts of your network from the path.**
**Be aware that a change within the fluid vector does not allow this practice.** Thus, if you plan to use additional fluids in parts of the network you have not touched until now, you will need to state all fluids from the beginning.

.. note::

	Initialisation from a converged calculation usually yields the best performance and is highly receommended.
	In order to initialise your calculation from a path, you need to provide the path to the saved/exported network. If you saved your calculation restults you will find the results in the specified base path './savename/'.


Algorithm
^^^^^^^^^

In this section we will give you an introduction to the implemented solution algorithm.

Newton–Raphson method
+++++++++++++++++++++

The Newton–Raphson method requires the calculation of residual values for the equations and of the partial derivatives to all system variables (jacobian matrix).
In the next step the matrix is inverted and multiplied with the residual vector to calculate the increment for the system variables.
This process is repeated until every equation's result in the system is "correct", thus the residual values are smaller than a specified error tolerance. All equations are of the same structure:

.. math::

	0 = \text{expression}

calculate the residuals

.. math::

	f(\vec{x}_i)

jacobian matrix J

.. math::
	J(\vec{x})=\left(\begin{array}{cccc}
	\frac{\partial f_1}{\partial x_1} & \frac{\partial f_1}{\partial x_2} & \cdots & \frac{\partial f_1}{\partial x_n} \\
	\frac{\partial f_2}{\partial x_1} & \frac{\partial f_2}{\partial x_2} & \cdots & \frac{\partial f_2}{\partial x_n} \\
	\vdots & \vdots & \ddots & \vdots \\
	\frac{\partial f_n}{\partial x_1} & \frac{\partial f_n}{\partial x_2} & \cdots & \frac{\partial f_n}{\partial x_n}
	\end{array}\right)

derive the increment

.. math::
	\vec{x}_{i+1}=\vec{x}_i-J(\vec{x}_i)^{-1}\cdot f(\vec{x}_i)

while

.. math::
	||f(\vec{x}_i)|| > \epsilon

.. note::

	You have to provide the exact amount of required parameters (neither less nor more) and the parametrisation must not lead to linear dependencies.
	Each parameter you set for a connection and each energy flow you specify for a bus will add one equation to your system.
	On top, each component provides a different amount of basic equations plus the equations provided by your component specification.
	For example, setting the power of a pump results in an additional equation compared to a pump without specified power:

.. math::
	\forall i \in \mathrm{network.fluids} \, &0 = fluid_{i,in} - fluid_{i,out}\\
											 &0 = \dot{m}_{in} - \dot{m}_{out}\\
					 \mathrm{additional:} \, &0 = 1000 - \dot{m}_{in} (\cdot {h_{out} - h_{in}})

.. _using_tespy_convergence_check_label:

Convergence stability
+++++++++++++++++++++

One of the main downsides of the Newton–Raphson method is that the initial stepwidth is very large and that it does not know physical boundaries,
for example mass fractions smaller than 0 and larger than 1 or negative pressure. Also, the large stepwidth can adjust enthalpy or pressure to quantities that are not covered by the fluid property databases.
This would cause an inability e. g. to calculate a temperature from pressure and enthalpy in the next iteration of the algorithm. In order to improve convergence stability, we have added a convergence check.

**The convergence check manipulates the system variables after the increment has been added** (if the system variable's value is not user specified). This manipulation has four steps, the first two are always applied:

 * cutting off mass fractions smaller than 0 and larger than 1: This way a mass fraction of a single fluid components never exceeds these boundaries.
 * check, wheather the fluid properties of pure fluids are within the available ranges of CoolProp and readjust the values if not.

The next two steps are applied, if the user did not specify an init_file and the iteration count is lower than 3, thus in the first three iteration steps of the algorithm only. In other cases this convergence check is skipped.

 * Fox mixtures: check, if the fluid properties (pressure, enthalpy and temperature) are within the user specified boundaries (:code:`p_range, h_range, T_range`) and if not, cut off higher/lower values.
 * Check the fluid properties of the connections based on the components they are connecting. E. g. check if the pressure at the outlet of a turbine is lower than the pressure at the inlet or if the flue gas composition at a combustion chamber's
   outlet is within the range of a "typical" flue gas composition. If there are any violations, the corresponding variables are manipulated. If you want to look up, what exactly the convergence check for a specific component does,
   look out for the :code:`convergence_check` methods in the :py:mod:`tespy.components.components module <tespy.components.components>`.

In a lot of different tests the algorithm has found a near enough solution after the third iteration, further checks are usually not required.

Troubleshooting
+++++++++++++++

In this section we show you how you can troubleshoot your calculation and list up common mistakes. If you want to debug your code, make sure to enable tespy.logger and have a look at the log-file at :code:`~/.tespy/` (or at your specified location).

First of all, make sure your network topology is set up correctly, TESPy will prompt an Error, if not.
Also, TESPy will prompt an error, if you did not provide enough or if you provide too many parameters for your calculation, but you will not be given an information which specific parameters are under- or overdetermined.

.. note::
	Always keep in mind, that the system has to find a value for mass flow, pressure, enthalpy and the fluid mass fractions. Try to build up your network step by step and have in mind, what parameters will be determined
	by adding an additional component without any parametrisation. This way, you can easily determine, which parameters are still to be specified.

When using multiple fluids in your network, e. g. :code:`fluids=['water', 'air', 'methane']` and at some point you want to have water only, you still need to specify the mass fractions for both air and methane (although beeing zero) at that point :code:`fluid={'water': 1, 'air': 0, 'methane': 0}`.
Also, setting :code:`fluid={water: 1}, fluid_balance=True` will still not be sufficent, as the fluid_balance parameter adds only one equation to your system.

If you are modeling a cycle, e. g. the clausius rankine cylce, you need to make a cut in the cycle using a sink and a source not to overdetermine the system. Have a look in the :ref:`heat pump tutorial <heat_pump_tutorial_label>`
to understand why this is important.

If you have provided the correct number of parameters in your system and the calculations stops after or even before the first iteration, there are four frequent reasons for that:

 * Sometimes, the fluid property database does not find a specific fluid property in the initialisation process, have you specified the values in the correct unit?
 * Also, fluid property calculation might fail, if the fluid propagation failed. Provide starting values for the fluid composition, especially, if you are using drums, merges and splitters.
 * A linear dependency in the jacobian matrix due to bad parameter settings stops the calculation (overdetermining one variable, while missing out on another).
 * A linear dependency in the jacobian matrix due to bad starting values stops the calculation.

The first reason can be eleminated by carefully choosing the parametrisation. **A linear dependendy due to bad starting values is often more difficult to resolve and it may require some experience.**
In many cases, the linear dependency is caused by equations, that require the **calculation of a temperature**, e. g. specifying a temperature at some point of the network, terminal temperature differences at heat exchangers, etc..
In this case, **the starting enthalpy and pressure should be adjusted in a way, that the fluid state is not within the two-phase region:** The specification of temperature and pressure in a two-phase region does not yield a distict value for the enthalpy.
Even if this specific case appears after some iterations, better starting values often do the trick.

Another frequent error is that fluid properties move out of the bounds given by the fluid property database. The calculation will stop immediately. **Adjusting pressure and enthalpy ranges for the convergence check** might help in this case.

.. note::

	If you experience slow convergence or instability within the convergence process, it is sometimes helpful to have a look at the iterinformation. This is printed by default and provides
	information on the residuals of your systems' equations and on the increments of the systems' variables. Maybe it is only one variable causing the instability, thus its increment is much larger
	than the incerement of the other variables.

Did you experience other errors frequently and have a workaround/tips for resolving them? You are very welcome to contact us and share your experience for other users!

Postprocessing
--------------

A postprocessing is performed automatically after the calculation finished. You have two further options:

 * print the results to prompt (:code:`nw.print_results()`) and
 * save the results in a .csv-file (:code:`nw.save('savename')`).

You can print the components and its properties to the prompt and the connections and its properties as well. If you choose to save your results the specified folder will be created containing the information about the network, all connections, busses, components and characteristics.

In order to perform calculations based on your results, you can access all components' and connections' parameters:

For the components this is the way to go

.. code:: python

	eff = mycomp.eta_s.val # isentropic efficiency of mycomp
	s_irr = mycomp.Sirr.val # entropy production of mycomp due to irreveribility

Use this code for connection parameters:

.. code:: python

	mass_flow = myconn.m.val # value in specified network unit
	mass_flow_SI = myconn.m.val_SI # value in SI unit
	mass_fraction_oxy = myconn.fluid.val['O2'] # for the mass fraction of oxygen

.. _using_tespy_components_label:

TESPy network reader
====================

The network reader is a useful tool to import networks from a datastructure using .csv-files. In order to reimport an exported TESPy network, you must save the network first.

.. code:: python

	nw.save('mynetwork')

This generates a folder structure containing all relevant files defining your network (general network information, components, connections, busses, characteristics) holding the parametrization of that network.
You can reimport the network using following code with the path to the saved documents. The generated network object contains the same information as a TESPy network created by a python script. Thus, it is possible to set your parameters in the .csv-files, too.

.. code:: python

	from tespy import nwkr
	nw = nwkr.load_nwk('path/to/mynetwork')
	
.. note::

	- Imported connections are accessible by the connections' target and target id, e. g.: :code:`nw.imp_conns['condenser:in1']`. 
	- Imported components and busses are accessible by their label, e. g. :code:`nw.imp_comps['condenser']` and :code:`nw.imp_busses['total heat output']` respectively.

TESPy components
================

In this section we will introduce you into the details of component parametrisation and component characteristics. At the end of the section we show you, how to create custom components.

List of components
------------------

More information on the components can be gathered from the code documentation. We have linked the base class containing a figure and basic informations as well as the equations.

- :py:class:`Source <tespy.components.components.source>` (no equations)
- :py:class:`Sink <tespy.components.components.sink>` (no equations)
- Nodes (base class is node)
	- :py:class:`Node <tespy.components.components.node>` (:py:meth:`equations <tespy.components.components.node.equations>`)
	- :py:class:`Merge <tespy.components.components.merge>` (:py:meth:`equations <tespy.components.components.node.equations>`)
	- :py:class:`Splitter <tespy.components.components.splitter>` (:py:meth:`equations <tespy.components.components.node.equations>`)
	- :py:class:`Separator <tespy.components.components.separator>` (:py:meth:`equations <tespy.components.components.node.equations>`)
- :py:class:`Valve <tespy.components.components.valve>` (:py:meth:`equations <tespy.components.components.valve.equations>`)
- Turbomachines (base class is turbomachine)
	* :py:class:`Pump <tespy.components.components.pump>` (:py:meth:`equations <tespy.components.components.turbomachine.equations>`)
	* :py:class:`Compressor <tespy.components.components.compressor>` (:py:meth:`equations <tespy.components.components.turbomachine.equations>`)
	* :py:class:`Turbine <tespy.components.components.turbine>` (:py:meth:`equations <tespy.components.components.turbomachine.equations>`)
- Components with combustion (base class is combustion_chamber)
	* :py:class:`Combustion chamber <tespy.components.components.combustion_chamber>` (:py:meth:`equations <tespy.components.components.combustion_chamber.equations>`)
	* :py:class:`Combustion chamber stoichiometric <tespy.components.components.combustion_chamber_stoich>` (:py:meth:`equations <tespy.components.components.combustion_chamber_stoich.equations>`)
	* :py:class:`Cogeneration unit <tespy.components.components.cogeneration_unit>` (:py:meth:`equations <tespy.components.components.cogeneration_unit.equations>`)
- Heat exchangers (base class is heat_exchanger)
	* :py:class:`Heat exchanger <tespy.components.components.heat_exchanger>` (:py:meth:`equations <tespy.components.components.heat_exchanger.equations>`)
	* :py:class:`Condenser <tespy.components.components.condenser>` (:py:meth:`equations <tespy.components.components.heat_exchanger.equations>`)
	* :py:class:`Desuperheater <tespy.components.components.desuperheater>` (:py:meth:`equations <tespy.components.components.heat_exchanger.equations>`)
- Simplified heat exchangers (base class is heat_exchanger_simple)
	* :py:class:`Heat exchanger simple <tespy.components.components.heat_exchanger_simple>` (:py:meth:`equations <tespy.components.components.heat_exchanger_simple.equations>`)
	* :py:class:`Pipe <tespy.components.components.pipe>` (:py:meth:`equations <tespy.components.components.heat_exchanger_simple.equations>`)
	* :py:class:`Solar collector <tespy.components.components.solar_collector>` (:py:meth:`equations <tespy.components.components.heat_exchanger_simple.equations>`)
- :py:class:`Drum <tespy.components.components.drum>` (:py:meth:`equations <tespy.components.components.drum.equations>`)
- :py:class:`Subsystem interface <tespy.components.components.subsys_interface>` (:py:meth:`equations <tespy.components.components.subsys_interface.equations>`)


Component parametrisation
-------------------------

Component parameters can be set and accessed in various ways. All parameters of components are objects of a :code:`data_container` class. The data container for component parameters it is called :code:`dc_cp`, :code:`dc_cc` for component characteristics and :code:`dc_cm` for characteristic maps.
The main purpose of having a data container for the parameters (instead of pure numbers), is added flexibility for the user.

There are different ways for you to specify a component parameter, we use a heat exchanger as an example.

Parameters
^^^^^^^^^^

.. code-block:: python

	from tespy import cmp, hlp
	import numpy as np

	he = cmp.heat_exchanger('evaporator')

	# ways to specify (and set) value
	he.set_attr(kA=1e5)
	# specify data container (same result as above)
	he.set_attr(kA=hlp.dc_cp(val=1e5, is_set=True))

	# ways to unset value
	he.set_attr(kA=np.nan)
	he.kA.set_attr(is_set=False)

	# custom variables
	pipe = cmp.pipe('my pipe')

	# make diameter variable of system
	pipe.set_attr(D='var')
	# data container specification with identical result,
	# benefit: val is the starting value in this case
	pipe.set_attr(D=hlp.dc_cp(val=0.2, is_set=True, is_var=True))


Characteristics
^^^^^^^^^^^^^^^

.. code-block:: python

	from tespy import cmp, hlp
	import numpy as np

	he = cmp.heat_exchanger('evaporator')

	# specify name of predefined method
	he.set_attr(kA_char1='EVA_HOT')
	he.set_attr(kA_char2='EVA_COLD')

	# specify data container (yields same result)
	he.set_attr(kA_char1=hlp.dc_cc(method='EVA_HOT', param='m'))

	# specify data container (custom interpolation points x and y)
	x = np.array([0, 0.5, 1, 2])
	y = np.array([0, 0.8, 1, 1.2])
	he.set_attr(kA_char1=hlp.dc_cc(param='m', x=x, y=y))

.. _component_characteristics_label:

Component characteristics
-------------------------

Characteristics are available for the following components and parameters:

- pump
	* :py:meth:`eta_s_char <tespy.components.components.pump.eta_s_char_func>`: isentropic efficiency vs. volumetric flow rate.
	* :py:meth:`flow_char <tespy.components.components.pump.flow_char_func>`: pressure rise vs. volumetric flow characteristic.
- compressor
	* :py:meth:`char_map <tespy.components.components.compressor.char_map_func>`: component map for isentropic efficiency and pressure rise.
	* :py:meth:`eta_s_char <tespy.components.components.compressor.eta_s_char_func>`: isentropic efficiency vs. pressure ratio.
- turbine
	* :py:meth:`eta_s_char <tespy.components.components.turbine.eta_s_char_func>`: isentropic efficiency vs. isentropic enthalpy difference/pressure ratio/volumetric flow/mass flow.
- heat exchangers:
	* :py:meth:`kA1_char, kA2_char <tespy.components.components.heat_exchanger.kA_func>`: heat transfer coefficient, various predefined types, mass flows as specification parameters.
- simple heat exchangers
	* :py:meth:`kA_char <tespy.components.components.heat_exchanger_simple.kA_func>`: e. g. pipe, see heat exchangers
- cogeneration unit
	* :py:meth:`tiP_char <tespy.components.components.cogeneration_unit.tiP_char_func>`: thermal input vs. power ratio.
	* :py:meth:`Q1_char <tespy.components.components.cogeneration_unit.Q1_char_func>`: heat output 1 vs. power ratio.
	* :py:meth:`Q2_char <tespy.components.components.cogeneration_unit.Q2_char_func>`: heat output 2 vs. power ratio.
	* :py:meth:`Qloss_char <tespy.components.components.cogeneration_unit.Qloss_char_func>`: heat loss vs. power ratio.

You can specify the name of a default characteristic line or you define the whole data container for this parameter. The default characteristic lines can be found in the :py:mod:`documentation <tespy.components.characteristics>`.

.. code-block:: python

	from tespy import cmp, hlp

	turb = cmp.turbine('turbine')
	# method specification (default characteristic line "TRAUPEL")
	turb.set_attr(eta_s_char='TRAUPEL')
	# data container specification
	turb.set_attr(eta_s_char=hlp.dc_cc(method='TRAUPEL', param='dh_s', x=None, y=None))

	# defining a custom line (this line overrides the default characteristic line, method does not need to be specified)
	x = np.array([0, 1, 2])
	y = np.array([0.95, 1, 0.95])
	turb.set_attr(eta_s_char=hlp.dc_cc(param='dh_s', x=x, y=y)

	# heat exchanger analogously
	he = cmp.heat_exchanger('evaporator')
	he.set_attr(kA_char1='EVA_HOT')
	he.set_attr(kA_char2='EVA_COLD')

Custom components
-----------------

If required, you can add custom components. These components should inherit from :py:class:`tespy.components.components.component class <tespy.components.components.component>` or its children.
In order to do that, create a python file in your working directory and import the :py:mod:`tespy.components.components module <tespy.components.components>`. The most important methods are

- :code:`attr(self)`,
- :code:`inlets(self)`,
- :code:`outlets(self)`,
- :code:`equations(self)`,
- :code:`derivatives(self, nw)` and
- :code:`calc_parameters(self, nw, mode)`,

where :code:`nw` is a :py:class:`tespy.networks.network object <tespy.networks.network>`.

The starting lines of your file would look like this:

.. code:: python

	from tespy import cmp


	class my_custom_component(cmp.component):


Attributes
^^^^^^^^^^

The attr method returns a dictionary with the attributes you are able to specify when you want to parametrize your component as keys. The values for each key are the type of data_container this parameter should hold.

.. code:: python

	def attr(self):
		return {'par1': dc_cp(), 'par2': dc_cc()}


Inlets and outlets
^^^^^^^^^^^^^^^^^^

:code:`inlets(self)` and :code:`outlets(self)` respectively must return a list of strings. The list may look like this:

.. code:: python

	def inlets(self):
		return ['in1', 'in2']

	def outlets(self):
		return ['out1', 'out2']

The number of inlets and outlets might even be generic, e. g. if you have added an attribute :code:`'num_in'` in :code:`attr(self)`:

.. code:: python

    def inlets(self):
        if self.num_in_set:
            return ['in' + str(i + 1) for i in range(self.num_in)]
        else:
            self.set_attr(num_in=2)
            return self.inlets()

Equations
^^^^^^^^^

The equations contain the information on the changes to the fluid properties within the component. Each equation must be defined in a way, that the correct result is zero, e. g.:

.. math::

	0 = \dot{m}_{in} - \dot{m}_{out}\\
	0 = \dot{p}_{in} - \dot{p}_{out} - \Delta p

The connections connected to your component are available as a list in :code:`self.inl` and :code:`self.outl` respectively.

.. code:: python

    def equations(self):

    	vec_res = []

		vec_res += [self.inl[0].m.val_SI - self.outl[0].m.val_SI]
		vec_res += [self.inl[0].p.val_SI - self.outl[0].p.val_SI - self.dp.val]

The equations are added to a list one after another, which will be returned at the end.

Derivatives
^^^^^^^^^^^

You need to calculate the partial derivatives of the equations to all variables of the network.
This means, that you have to calculate the partial derivatives to mass flow, pressure, enthalpy and all fluids in the fluid vector on each incomming or outgoing connection of the component.

Add all derivatives to a list (in the same order as the equations) and return the list as numpy array (:code:`np.asarray(list)`).
The derivatives can be calculated analytically or numerically by using the inbuilt function :code:`numeric_deriv(self, func, dx, pos, **kwargs)`.

- :code:`func` is the function you want to calculate the derivatives for,
- :code:`dx` is the variable you want to calculate the derivative to and
- :code:`pos` indicates the connection you want to calculate the derivative for, e. g. :code:`pos=1` means, that counting your inlets and outlets from low index to high index (first inlets, then outlets),
  the connection to be used is the second connection in that list.
- :code:`kwargs` are additional keyword arguments required for the function.

For a good start just look into the source code of the inbuilt components. If you have further questions feel free to contact us.


.. _tespy_subsystems_label:


TESPy subsystems/component groups
=================================

Usage
-----

Subsystems are an easy way to add frequently used component groups such as a drum with evaporator or a preheater with desuperheater to your system.
You can use the predefined subsystems or :ref:`create a subsytem yourself <tespy_subsystems_label>`. Every subsystem must have two interfaces, an inlet interface and an outlet interface.
These interfaces have a variable number of connections, which can be connected with the rest of your network. The example below uses the predefined subsystem preheater with desuperheater (:code:`ph_desup_cond()`).
The subsystems interfaces are subsystem.inlet and subsystem.outlet, both with two connections. All connections (and components) of the subsystem have to be added to the network in order to start a simulation.
This can easily be done by adding the whole subsystem object to your network.

.. code-block:: python

	from tespy import subsys, cmp
	ext = cmp.source(label='extraction steam')
	cond = cmp.sink(label='condensate')
	fwc = cmp.source(label='feed water cold')
	fww = cmp.sink(label='feed water warm')

	# a preheater with desuperheater part
	preheater = subsys.ph_desup(label='sub1')

	# connections into the subsystem are attached to subsystem.inlet, connections out of the subsystem to subsystem.outlet
	ext_pre = connection(ext, 'out1', subsystem.inlet, 'in1')
	pre_cond = connection(subsystem.outlet, 'out1', cond, 'in1')
	fwc_pre = connection(fwc, 'out1',subsystem.inlet,'in2')
	pre_fwc = connection(subsystem.outlet, 'out2', fww, 'in1')

	# parametrisation
	preheater.set_attr(ttd=8, pr1_desup=1, pr2_desup=1, pr1_cond=1, pr2_cond=1)

	ext_pre.set_attr(m=5, p=4, h=29e5, fluid={'water': 1})
	fwc_pre.set_attr(p=50, h=3e5, fluid={'water': 1})
	pre_fwc.set_attr(p0=50)

	# create the network and connections and subsystems
	my_plant.add_conns(ext_pre, pre_cond, fwc_pre, pre_fwc)
	my_plant.add_subsys(subsys)


.. figure:: api/_images/intro_subsys.svg
    :align: center

    Figure 3: Topology of the subsystem.

Custom subsystems
-----------------

You can use subsystems in order to represent groups of different components. These are highly customizable and thus a very powerful tool, if you require to use specific component groups frequently.
You will learn how to create your own subsystems. Create a .py file in your working-directory with the class-definition of your custom subsystem. This usually includes the following methods:

- :code:`attr`: list of subsystem attributes,
- :code:`create_comps`: define the number of interfaces and create the necessary components,
- :code:`set_comps`: parametrize the components with the defined attributes from :code:`attr`,
- :code:`create_conns`: create the subsystems topology and
- :code:`set_conns`: parametrize them.

The following section shows, how the different functions of a subsystem can be defined. The code is taken from the subsystem drum with evaporator and natural flow.

Your file will start with the following lines:

.. code-block:: python

	from tespy import con, cmp, subsys


	class dr_eva_natural (subsys.subsystem):

Add the attr method:

.. code-block:: python

	def attr(self):
		# define available attributes for subsystem
		# num_i and num_o are excluded, as they are predefined in this subsystem
		return ([n for n in subsys.subsystem.attr(self) if
				 n != 'num_i' and n != 'num_o'] +
				['dp1_eva', 'PP', 'circ_num'])

Create the components
^^^^^^^^^^^^^^^^^^^^^

The inlet and the outlet of the subsystem must be an attribute of the subsystem in order to reference to these when you are creating a network and want to connect the subsystem to the rest of the network.

.. code-block:: python

	def create_comps(self):
		# create the components

		self.num_i = 2
		self.num_o = 2
		self.inlet = cmp.subsys_interface(label=self.label + '_inlet',
										  num_inter=self.num_i)
		self.outlet = cmp.subsys_interface(label=self.label + '_outlet',
										   num_inter=self.num_o)
		self.drum = cmp.drum(label=self.label + '_drum')
		self.evaporator = cmp.heat_exchanger(label=self.label + '_evaporator')

As specific attributes refer to specific components in the subsystem, it is necessery, that the evaporator is stored as attribute of the subsystem as well. Else it would not be possible to set values for the parametrization.

Parametrize the components
^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: python

	def set_comps(self):
		# set component parameters

		self.evaporator.set_attr(ttd_l=self.PP)
		self.evaporator.set_attr(pr1=self.pr1_eva)

Create the connections
^^^^^^^^^^^^^^^^^^^^^^

Create a list called :code:`self.conns` and add the connections to that list.

.. code-block:: python

	def create_conns(self):
		# create the connections

		self.conns = []

		self.conns += [con.connection(self.inlet, 'out1', self.evaporator, 'in1')]
		self.conns += [con.connection(self.evaporator, 'out1', self.outlet, 'in1')]
		self.conns += [con.connection(self.inlet, 'out2', self.drum, 'in1')]
		self.conns += [con.connection(self.drum, 'out1', self.evaporator, 'in2')]
		self.conns += [con.connection(self.evaporator, 'out2', self.drum, 'in2')]
		self.conns += [con.connection(self.drum, 'out2', self.outlet, 'in2')]

Parametrize the connections
^^^^^^^^^^^^^^^^^^^^^^^^^^^

The connection gets a ref object as attribute, thus it is necessary to look, if the subsystems attribute is set or not.
For parametrization with specific values simply use :code:`self.conns[3].set_attr(m=self.mass_flow)`. :code:`self.mass_flow` must be a subsystem attribute in this example.

.. code-block:: python

	def set_conns(self):
		# set connection parameters

		if self.circ_num_set:
			self.conns[3].set_attr(m=con.ref(self.conns[-1], self.circ_num, 0))
		else:
			self.conns[3].set_attr(m=np.nan)

Add more felxibility
^^^^^^^^^^^^^^^^^^^^

If you want to add even more flexibility, you might need to manipulate the :code:`__init__()` method.
For example, if you want a variable number of inlets and outlets because you have a variable number of components groups within your subsystem,
you may introduce an attribute which is set on initialisation and lets you create and parametrize components and connections generically.
This might be very interesting for district heating systems, turbines with several sections of equal topology, etc..
For a good start, you can have a look into the sub_consumer.py at the `tespy_examples repository <https://github.com/oemof/oemof-examples/tree/master/oemof_examples/tespy/district_heating>`_.


TESPy connections
=================

This section provides an overview of the parametrisation of connections, the usage of references and busses (connections for energy flow).

Parametrisation
---------------

As mentioned in the introduction, for each connection you can specify the following parameters:

 * mass flow* (m),
 * volumetric flow (v),
 * pressure* (p),
 * enthalpy* (h),
 * temperature* (T),
 * vapour mass fraction for pure fluids (x),
 * a fluid vector (fluid) and
 * a balance closer for the fluid vector (fluid_balance).

It is possible to specify values, starting values, references and data containers. The data containers for connections are dc_prop for fluid properties (mass flow, pressure, enthalpy, temperature and vapour mass fraction)
and dc_flu for fluid composition. You need to import the :code:`hlp` module, if you want to specify data_containers.

.. code-block:: python

	# set pressure and vapour mass fraction by value, temperature and enthalpy analogously
	myconn.set_attr(p=7, x=0.5)

	# set starting values for mass flow, pressure and enthalpy (has no effect on temperature and vapour mass fraction!)
	myconn.set_attr(m0=10, p0=15, h0=100)

	# do the same with a data container
	myconn.set_attr(p=hlp.dc_prop(val=7, val_set=True), x=hlp.dc_prop(val=0.5, val_set=True))
	myconn.set_attr(m=hlp.dc_prop(val0=10), p=hlp.dc_prop(val0=15), h=hlp.dc_prop(val0=100))

	# specify a value in a different unit for a specific parameter
	myconn.set_attr(p=hlp.dc_prop(val=7, val_set=True, unit='MPa', unit_set=True)

	# specify a referenced value: pressure of myconn is 1.2 times pressure at myotherconn minus 5 Pa (always SI unit here)
	myconn.set_attr(p=con.ref(myotherconn, 1.2, -5))

	# specify value and reference at the same time
	myconn.set_attr(p=hlp.dc_prop(val=7, val_set=True, ref=con.ref(myotherconn, 1.2, -5), ref_set=True))

	# unset value and reference
	myconn.set_attr(p=np.nan)
	myconn.p.set_attr(val_set=False, ref_set=False)

If you want to specify the fluid vector you can do it in the following way:

.. code-block:: python

	# set both elements of the fluid vector
	myconn.set_attr(fluid={'water': 1, 'air': 0})
	# same thing, but using data container
	myconn.set_attr(fluid=dc_flu(val={'water': 1, 'air': 0}, val_set:{'water': True, 'air': True}))

	# set starting values
	myconn.set_attr(fluid0={'water': 1, 'air': 0})
	# same thing, but using data container
	myconn.set_attr(fluid=dc_flu(val0={'water': 1, 'air': 0}))

	# unset values
	myconn.fluid.set_attr(val_set={'water': False, 'air': False})

References can not be used for fluid composition at the moment!


.. _tespy_busses_label:

Busses
------

Busses can be used to add up the power of different turbomachinery or to add up heat flow of different heat exchangers within your network.
The handling is very similar to connections and components. You need to add components to your busses as a dictionary containing at least the instance of your component.
Additionally you may provide a characteristic line, linking the ratio of actual heat flow/power to referenced heat flow/power to a factor the actual heat flow/power of the component is multiplied with on the bus.
For instance, you can provide a characteristic line of an electrical generator or motor for a variable conversion efficiency. The referenced value (P_ref) is retrieved by the design point of your system.
Offdesign calculations use the referenced value from your system design point for the characteristic line. In design case, the heat flow/power ratio thus will be equal to 1.

.. note::
	The available keywords for the dictionary are

	- 'c' for the component instance,
	- 'p' for the parameter (the cogeneration unit has different parameters, have a look at the :ref:`cogeneration unit example <cogeneration_unit_label>`),
	- 'P_ref' for the reference heat flow/power value of the component and
	- 'char' for the characteristic line.

	There are different specification possibilites:

	- If you specify the component only, the parameter will be default (not working with cogeneration unit) and the conversion factor of the characteristic line will be 1 for every load.
	- If you specify a numeric value for char, the conversion factor will be that value for every load.
	- If you want to specify a characteristic line, you need to provide a :py:class:`TESPy characteristics <tespy.components.characteristics.characteristics>` object.

This can be used for easy post processing, e. g. to calculate thermal efficiency or you can build up relations between components in your network.
If you want to use the busses for postprocessing only, you must not specify the sum of the power or heat flow on your bus.
If you set a value for P (equal parameter for heat flow or power), an additional equation will be added to your network. This way the total heat flow/power of the bus will equal to the specified value.
This could be useful, e. g. for establishing relations between different components, for instance when using a steam turbine powered feed water pump.
In the code example the power of the turbine and the feed water pump is added up and set to zero, as the turbines and feed water pumps power have to be equal in absolute value but have different sign.
The sign can be manipulated, e. g. in order to design two turbines with equal power output.
Do not forget to add the busses to you network.

.. code-block:: python

	from tespy import nwk, con, cmp_char

	...

	fwp_bus = con.bus('feed water pump', P=0) # set a value for the total power on this bus.
	fwp_bus.add_comps({'c': turbine_fwp}, {'c': fwp})

	turbine_bus = con.bus('turbines', P=0) # set a value for the total power on this bus
	turbine_bus.add_comps({'c': turbine_hp}, {'c': turbine_hp, 'char': -1})
	# the values for the busses power can be altered by using .set_attr()

	power = con.bus('power output') # bus for postprocessing, no power (or heat flow) specified but with variable conversion efficiency
	x = np.array([0.2, 0.4, 0.6, 0.8, 1.0, 1.1])
	y = np.array([0.85, 0.93, 0.95, 0.96, 0.97, 0.96])
	gen = cmp_char.characteristics(x=x, y=y) # characteristic line for a generator
	power.add_comps({'c': turbine_hp, 'char': gen}, {'c': turbine_lp, 'char': gen})

	chp = con.bus('chp power') # bus for cogeneration unit power
	chp.add_comps({'c': cog_unit, 'p': 'P', 'char': gen})

	my_network.add_busses(fwp_bus, turbine_bus, power)
	
.. note::

	The x-values of the characteristic line represent the relative load of the component: actual value of the bus divided by the reference/design point value.
	In design-calculations the x-value used in the function evaluation will always be at 1.


How can TESPy contribute to your energy system calculations?
============================================================

In this part you learn how you can use TESPy for your energy system calculations: In energy system calculations, for instance in oemof-solph, plants are usually modelled as abstract components on a much lower level of detail.
In order to represent a plant within an abstract component it is possible to supply characteristics establishing a connection between your energy system model and a specific plant model.
Thus the characteristics are a representation of a specific plant layout in terms of topology and process parameters. In the examples section we have an example of a heat pump COP at different loads and ambient temperatures
as well as a CHP unit with backpressure turbine operating at different loads and varying feed flow temperatures of a heating system.

.. _tespy_fluid_properties_label:

Fluid properties in TESPy
=========================

The basic fluid properties are handled by `CoolProp <http://www.coolprop.org/>`_. All available fluids can be found on their homepage.

Pure and pseudo-pure fluids
---------------------------

If you use pure fluids, TESPy directly uses CoolProp functions to gather all fluid properties.
CoolProp covers the most important fluids such as water, air as a pseudo-pure fluid as well as its components, several fuels and refrigerants etc..
Look for the aliases in the `list of fluids <http://www.coolprop.org/fluid_properties/PurePseudoPure.html#list-of-fluids>`_. All fluids provided in this list cover liquid and gaseous state and the two-phase region.

Incompressible fluids
---------------------

If you are looking for heat transer fluids, the `list of incompressible fluids <http://www.coolprop.org/fluid_properties/Incompressibles.html>`_ might be interesting for you.
In contrast to the pure fluids, the properties cover liquid state only.

Fluid mixtures
--------------

CoolProp provides fluid properties for two component mixtures. BUT: These are NOT integrated in TESPy! Nevertheless, you can use fluid mixtures for gases:

Ideal mixtures of gaseous fluids
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

TESPy can handle mixtures of gaseous fluids, by using the single fluid properties from CoolProp together with corresponding equations for mixtures.
The equations can be found in the :py:mod:`tespy.tools.helpers module <tespy.tools.helpers>` and are applied automatically to the fluid vector.

It is also possible create lookup-tables for fluid mixtures with fixed mass fractions of the components, as this reduces the amount of CoolProp fluid property calls and speeds up your calculation. Look up the :py:class:`tespy_fluids documentation <tespy.tools.helpers.tespy_fluid>` for more information.

Other mixtures
^^^^^^^^^^^^^^

It is **not possible** to use mixtures of liquid and other liquid or gaseous fluids **at the moment**!
If you try to use a mixture of two liquid or gaseous fluids and liquid fluids, e. g. water and methanol or liquid water and air, the equations will still be applied, but obviously return bad values.
If you have ideas for the implementation of new kinds of mixtures we appreciate you contacting us.
