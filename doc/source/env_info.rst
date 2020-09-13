=============================
Using Environment information
=============================


Building actions
----------------
By construction, every agent has access to the current Environment of the simulator (see **__init__** method of **pypownet.agent.Agent** displayed above).
More formally, the ``environment`` member of the class **Agent** is an instance of **pypownet.environment.RunEnv** which is the interface to the simulator.
A **RunEnv** notably contains a ``step`` method, which is used for computing the next observation, the resulting reward (which might depend on the action), and whether there was a game over.
This method should not be explicitely used, since pypownet comes with a **pypownet.runnner.Runner** class (which is used when calling pypownet with CLI) and that automates the simulator looping.

**RunEnv** also contains the action space of the simulation.
Formally, an action space is an instance of **pypownet.environment.ActionSpace**, which implements several functions that are used to build actions, which are instances of **pypownet.game.Action**.

.. Important:: In pypownet, the agents are not supposed to construct actions by manipulating **pypownet.game.Action** but rather by using an **ActionSpace**, which is similar to an **Action** factory.

Given an instance *environment* of **RunEnv**, its action space can be assessed with ``environment.action_space``.

Action understanding
^^^^^^^^^^^^^^^^^^^^

Typically on natiowide power grids, dispatchers use two types of physical actions:

    - switching ON or OFF some power lines
    - switching the nodes on which elements, such as productions, consumptions, or power lines, are connected within their substation

These two types of physical actions constitute an **Action** in pypownet.
Sometimes, dispatchers can negociate with producers to change their planned production schemes, but these operations are expensive and longer to process so they are not considered in pypownet.

Within the game, actions are managed as lists of binaries of consistent size within an environment.
More precisally, this list is equivalent to the concatenation of two lists: one for the node switching operations, and one for the line status (ON or OFF) switching operations
Each binary value of both lists correspond to the activation of a switch (1) or no activation of this switch (0), precisally:

    - a value of 1 in the line status switches subaction indicates to activate the switch of the corresponding line status of the line: if the prior line status is 0 (i.e. the line is switched OFF, or OFFLINE), then it will be put to 1 (i.e. the line is switched ON, or ONLINE) and vice versa
    - a value of 1 in the node switches subaction indicates to activate the switch of the node on which the corresponding element is *directly* connected: if the prior node on which the element is connected is 0, then this element will be connected the the node 1 (i.e. the second node of the substation of the element) and vice versa

.. Hint:: The important thing to remember about actions is that they represent **switches activations**, for the lines status (ON or OFF) or the node on which productions, loads, line origins and line extremities are connected (0 or 1).

In practice, actions of class **Action** are not constructed by hands, but by using the **ActionSpace** (``environment.action_space``)
There are essentially two ways of building actions:

    (a) either build a binary numpy array of expected length and convert it to an action
    (b) or by iteratively constructing an action by building blocks on top of a 0 action (i.e. do-nothing action)

Building actions from (numpy) arrays
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The first way of constructing actions is to first build a binary array, such as a numpy array, and then to call the function ``environment.action_space.array_to_action``.
The expected size of the action (which is essentially a list), can be assessed at ``environment.action_space.action_length``.
Here is an example of an agent that outputs a random action where switches are randomly activated:

.. code-block:: python
   :linenos:

    import pypownet.environment
    import pypownet.agent


    class CustomAgent(pypownet.agent.Agent):
        def act(observation):
            action_space = self.environment.action_space
            expected_size = action_space.action_length

            random_action_asarray = np.random.choice([0, 1], expected_size)

            # Convery np array to Action (this function also verifies the good shape and binary-type of input array)
            random_action = action_space.array_to_action(random_action_asarray)
            return random_action

It is possible to use some structure of an **Action** while using numpy arrays.
Recall that an **Action** is the concatenation of two lists: one for the switches of the topology, and one for the switches of the lines status.
Actually, the list of nodes switches is made of the concatenation of 4 sublists which are the switches of the nodes of resp. productions, loads, origins of line and extremities of line.
Each of these lists, including the lines status list, should be of size the total number of corresponding elements in the grid.
These lengths can be retrieved from resp. ``environment.action_space.prods_switches_subaction_length``, ``environment.action_space.loads_switches_subaction_length``, ``environment.action_space.lines_or_switches_subaction_length``, ``environment.action_space.lines_ex_switches_subaction_length``,  and ``environment.action_space.lines_status_subaction_length`` for the line status length.
For instance, a grid with 2 productions, 3 consumptions, 4 origins of line and 4 extremities of line (there are in total 4 power line in the grid so 4 lines status), has actions of size 2+3+4+4+4=17 binary values.

For illustration, here are two agents which resp. randomly switches lines status and randomly switches loads nodes:

.. code-block:: python
   :linenos:

    import pypownet.environment
    import pypownet.agent


    class RandomLineStatusSwitches(pypownet.agent.Agent):
        def act(observation):
            action_space = self.environment.action_space
            expected_size = action_space.action_length

            action_asarray = np.zeros(expected_size)
            action_asarray[-action_space.lines_status_subaction_length:] = \
                np.random.choice([0, 1], action_space.lines_status_subaction_length)

            return action_space.array_to_action(action_asarray)

.. code-block:: python
   :linenos:

    import pypownet.environment
    import pypownet.agent


    class RandomLoadsNodesSwitches(pypownet.agent.Agent):
        def act(observation):
            action_space = self.environment.action_space
            expected_size = action_space.action_length

            # Build 0-subaction where no switch is activated for all elements (incl. lines status) except loads
            prods_switches_subaction = np.zeros(action_space.prods_switches_subaction_length)
            lines_or_switches_subaction = np.zeros(action_space.lines_or_switches_subaction_length)
            lines_ex_switches_subaction = np.zeros(action_space.lines_ex_switches_subaction_length)
            lines_status_switches_subaction = np.zeros(action_space.lines_status_subaction_length)

            # Build action with random activated switches for loads
            loads_switches_subaction = np.random.choice([0, 1], action_space.loads_switches_subaction_length)

            # Build an array on the same principle as an Action; /!\ the order is important here!
            action_asarray = np.concatenate((prods_switches_subaction,
                                             loads_switches_subaction,
                                             lines_or_switches_subaction,
                                             lines_ex_switches_subaction,
                                             lines_status_switches_subaction,))

            return action_space.array_to_action(action_asarray)

This first way of building actions (which is essentially building arrays), is quite simple to put in place for neural networks models ans such.
However, it hardly exploit the grid structure (elements are decoupled regardin their substations).
To perform more dispatchers-like action, the second way of building actions using the action space as a factory is preferred since an **ActionSpace** contains various helpers to retrieve pertinent information with the point of view of substations.

Building actions with the action space
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The other way to construct actions is to consider an action as a container, which will store independent package of actions.
The do-nothing action, which consist of an action where no switches are activated so equivalently to a list of only 0, is neutral to the environment: since no switch are activated, the whole topology of the grid stays intact, si it is as if there was no action at all.
Based on this principle, we can replace parts of a do-nothing action with some meta-action, such as changing the configuration of a whole substation.
For concrete usage of the action space, the the example of agents in :ref:`example_agents`.

Here is the list the building methods of **pypownet.environment.ActionSpace**:

:get_do_nothing_action:  return a do-nothing (neutral) action where no switch are activated.
:array_to_action:  converts a numpy array into a proper **Action**; raises errors if the input array is not of good length, or contains non-binary values.
:get_number_elements_of_substation:  retrieve the number of elements (productions + loads + line origins + line extremities) of a substation from its true ID.
:get_switches_configuration_of_substation:  from a substation id, return the current switch values and type of the corresponding elements; this function returns two lists of size the number of elements of the substation from the input ID: the first one contains the binary values of the switches, while the second one returns the elementwise type of the concerned objects, which can be either ``pypownet.ElementType.PRODUCTION``, ``pypownet.ElementType.CONSUMPTION``, ``pypownet.ElementType.ORIGIN_POWER_LINE`` or ``pypownet.ElementType.EXTREMITY_POWER_LINE``.
:set_switches_configuration_of_substation:  within the input action, replace the value of the switches configuration related to input substation id with the input new configuration; this is the operation that changes the local topology of a substation with node switches.
:set_lines_status_switches_of_substation:  similarly to the previous one, replace the lines status of the lines of the input substation id with the input new values.
:set_lines_status_switch_from_id:  same as before except that this function changes 1 line status based on the input line id, where lines id range from 0 to the number of lines of the grid -  1.
:verify_action_shape:  verify that the input action or array-like container is of expected shape, and contains only binary values.

.. _reading_obs:

Reading observations
--------------------

Class Observation
^^^^^^^^^^^^^^^^^

For their ``act`` method, the agents receive an observation which is extracted from the current state of the grid.
Those observations are of type **pypownet.environment.Observation**, which is a class mainly acting as a container for several lists, and also contains several helpers functions juste like **ActionSpace**.

Precisally, an observation is composed of 1 **datetime** object (the current simulator date) and 36 lists of fixed (but different) sizes which are:

+------------+--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
| Element    | Name                                 | Value     | Description                                                                                                |
| type       |                                      | type      |                                                                                                            |
+============+======================================+===========+============================================================================================================+
|            | substations_ids                      | >=0 int   | ID of the substation on which the productions (generators) are wired.                                      |
+------------+--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
|            | prods_substations_ids                | >=0 int   | ID of the substation on which the productions (generators) are wired.                                      |
+            +--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
|            | active_productions                   | >=0 float | Real power produced by the generators of the grid (MW).                                                    |
+            +--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
|            | reactive_productions                 | float     | Reactive power produced by the generators of the grid (Mvar).                                              |
+            +--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
|            | voltage_productions                  | >0 float  | Voltage magnitude of the generators of the grid (per-unit V).                                              |
+            +--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
| production | productions_nodes                    | binary    | The node on which each production is connected within their corresponding substations.                     |
+            +--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
|            | initial_productions_nodes            | binary    | The initial (reference) node on which each load is connected within their corresponding substations.       |
+            +--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
|            | planned_active_productions           | >=0 float | An array-like container of the previsions of the active power of productions fur future timestep(s).       |
+            +--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
|            | planned_voltage_productions          | >0 float  | An array-like container of the previsions of the voltage of productions for future timestep(s).            |
+            +--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
|            | are_prods_cut                        | binary    | Mask whether the productors are isolated (1) from the rest of the network.                                 |
+------------+--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
|            | loads_substations_ids                | >=0 int   | ID of the substation on which the loads (consumers) are wired.                                             |
+            +--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
|            | active_loads                         | >=0 float | Real power consumed by the demands of the grid (MW).                                                       |
+            +--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
|            | reactive_loads                       | float     | Reactive power consumed by the demands of the grid (Mvar).                                                 |
+            +--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
|            | voltage_loads                        | >0 float  | Voltage magnitude of the demands of the grid (per-unit V).                                                 |
+            +--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
| load       | loads_nodes                          | binary    | The node on which each load is connected within their corresponding substations.                           |
+            +--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
|            | initial_loads_nodes                  | binary    | The initial (reference) node on which each production is connected within their corresponding substations. |
+            +--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
|            | planned_active_loads                 | >=0 float | An array-like container of the previsions of the active power of productions for future timestep(s).       |
+            +--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
|            | planned_reactive_loads               | >0 float  | An array-like container of the previsions of the voltage of productions for future timestep(s).            |
+            +--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
|            | are_loads_cut                        | binary    | Mask whether the consumers are isolated (1) from the rest of the network.                                  |
+------------+--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
|            | lines_or_substations_ids             | >=0 int   | ID of the substation on which the loads (consumers) are wired.                                             |
+            +--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
|            | active_flows_origin                  | >=0 float | Real power consumed by the demands of the grid (MW).                                                       |
+            +--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
| origin     | reactive_flows_origin                | float     | Reactive power consumed by the demands of the grid (Mvar).                                                 |
+ of line    +--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
|            | voltage_flows_origin                 | >0 float  | Voltage magnitude of the demands of the grid (per-unit V).                                                 |
+            +--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
|            | lines_or_nodes                       | binary    | The node on which each load is connected within their corresponding substations.                           |
+            +--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
|            | initial_lines_or_nodes               | binary    | The initial (reference) node on which each production is connected within their corresponding substations. |
+------------+--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
|            | lines_ex_substations_ids             | >=0 int   | ID of the substation on which the loads (consumers) are wired.                                             |
+            +--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
|            | active_flows_extremity               | >=0 float | Real power consumed by the demands of the grid (MW).                                                       |
+            +--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
| extremity  | reactive_flows_extremity             | float     | Reactive power consumed by the demands of the grid (Mvar).                                                 |
+ of line    +--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
|            | voltage_flows_extremity              | >0 float  | Voltage magnitude of the demands of the grid (per-unit V).                                                 |
+            +--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
|            | lines_ex_nodes                       | binary    | The node on which each load is connected within their corresponding substations.                           |
+            +--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
|            | initial_lines_ex_nodes               | binary    | The initial (reference) node on which each production is connected within their corresponding substations. |
+------------+--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
|            | lines_status                         | binary    | Whether the lines are connected/ON (1) or disconnected/OFF (0)                                             |
+            +--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
|            | ampere_flows                         | >=0 float | Total ampere flowing inside lines                                                                          |
+            +--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
| line       | thermal_limits                       | >0 float  | Each line thermal limit in Ampere over which the line overflows                                            |
+            +--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
|            | timesteps_before_lines_reconnectable | >=0 int   | Number of timesteps before each line can be reconnected                                                    |
+            +--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+
|            | timesteps_before_planned_maintenance | >=0 int   | Number of timesteps before a line will be disconnected by going into maintenance                           |
+------------+--------------------------------------+-----------+------------------------------------------------------------------------------------------------------------+

All of these lists containers (also including the *datetime* field) can be retrieved from an **Observation** ``observation`` with ``observation.name`` where name is one of the previous names (+ ``observation.datetime``) e.g.:

.. code-block:: python
   :linenos:

    import pypownet.environment
    import pypownet.agent


    class CustomAgent(pypownet.agent.Agent):
        """ At each timestep, this agent selects all the even substations id, and
        switch the node of their production if any.
        """
        def act(observation):
            action_space = self.environment.action_space
            # Build 0 action
            action = action_space.get_do_nothing_action()

            # Retrieve the substations ids
            substations_ids = observation.substations_ids

            # Loops on all substations
            for substation_id in substations_ids:
                # Selects even substations ids
                if substation_id % 2 == 0:
                    # Build new configuration: will activate switches of productions only
                    _, elements_type = action_space.get_switches_configuration_of_substation(action, substation_id)
                    new_configuration = np.zeros(len(elements_type))

                    # Activate all switches of productions
                    new_configuration[elements_type == pypownet.environment.ElementType.PRODUCTION] = 1

                    # Set new configuration for this substation within the buffer action
                    action_space.set_switches_configuration_of_substation(action, substation_id, new_configuration)

            return action


.. Hint:: At any moment, you can retrieve the name of the lists and their associated description available in an **Observation** with the global python dictionary ``pypownet.environment.OBSERVATION_MEANING``: ``print(pypownet.environment.OBSERVATION_MEANING)``

The **pypownet.environment.Observation** class has several dispatcher-like functions as well as helper for manipulating them.
Here is the list of functions of **Observation**:

:as_dict:  returns the observation as a dictionary, with the field name as keys (str) and the lists as values (np arrays) + the ``datetime`` value as a **datetime** Python object (note that the dictionary has 37 elements for all environments).
:as_array:  returns the observation as a numpy array: each list is concatenated (order is consistent for all environments) and returned as 1 numpy array of 1 dimension (essentially a list); note that ``datetime`` is not included in this array, and can be accesses with ``observation.datetime`` (as any other field member).
:get_lines_capacity_usage:  helper function that computes the nominal lines capacity usage of the self observation: it performs elementwise division of the ampere flows in lines by their nominal thermal limit. A line capacity usage greater than 1 indicates an overflowed line.
:get_nodes_of_substation:  returns the list of value of the nodes on which each element of the substation with input id. This function also computes the type of element associated to each node value of the returned nodes-value list.
:__keys__:  returns a list of all the field name of the class as strings (list of size 37).
:__str__:  returns a prettify version of the observation (this is what can be seen when using ``-v -vv`` CLI arguments) as condenses matrices. Note: the substations ids are not included in this string.
:as_ac_minimalist:  see after
:as_minimalist:  see after

The ``observation.get_nodes_of_substation`` is essentially similar to the ``action_space.get_switches_configuration_of_substation``, because both their arguments returns (consitently) the **pypownet.environment.ElementType** of the elements of the substation with input substation id, and they both return the associated values of the elements. The difference is that the one in **Observation** returns the true node on which elements are connected within the self observation, whereas the one in **ActionSpace** returns the values of the *switches* in the input action.



Reducing the observation space
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

For an environment associated to the common grid case known as case14 (available in environment **default14/**), they are 438 values in the array return by any ``observation.as_array()``.
Some of these values are integer, other are float, and some are binaries.
In any case, by construction some fields of an **Observation** are fixed throughout a given environment (such as **default14/**), including for instance ``observation.substations_ids``.
Some model approach including neural network have no precise way to naturally leverage some structure information, so reducing the observation by the fixed lists can help optimizing alorithms by shrinking *uninformative* values.
No matter the final usage, there are two version of **Observation** which contains a subset of its fields:

    - **pypownet.environment.MinimalistACObservation** contains 26 lists + ``datetime`` and represents an **Observation** without the list fixed members (mainly the IDs list fields)
    - **pypownet.environment.MinimalistObservation** contains 14 lists + ``datetime`` and represent a **MinimalistACObservation** without the list members which are fixed in DC mode but not in AC mode (including voltages since one of the hypothesis of DC is all voltage to 1, also reactive power for similar reasons etc)

You can convert an **Observation** into a **MinimalistACObservation** with ``ac_minimalist_observation = observation.as_ac_minimalist()`` and a **MinimalistACObservation** into a **MinimalistObservation** with ``minimalist_observation = ac_minimalist_observation.as_minimalist()``.
Transitively, an **Observation** can be converted to a **MinimalistObservation** with ``minimalist_observation = observation.as_minimalist()``.

Both implement the following methods, which acts the same as the one of **Observation** with the same name: ``as_array``, ``__keys__`` and ``as_dict``.
Note that the function ``get_nodes_of_substation`` is not implemented in neither **MinimalistACObservation** or **MinimalistObservation** because those classes do not have the ids of the elements, so they are no way to find the various elements of a given substation for both these classes.

