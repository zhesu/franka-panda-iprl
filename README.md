Franka Panda
============
Redis driver for the Franka Panda.


Installation
------------

This driver has been tested on Ubuntu 16.04 with C++14.

1. Set up your Panda robot and controller following the instructions on
   [this page](https://frankaemika.github.io/docs/getting_started.html).
   Connect your computer to the robot controller (not the robot directly).

2. CMake 3.6 or higher is required to download the external dependencies. Ubuntu
   16.04 comes with CMake 3.5. The easiest way to upgrade is through pip:

   ```
   pip install cmake
   ```

3. Build the driver

   ```
   cd <franka-panda.git>
   mkdir build
   cd build
   cmake ..
   make
   ```

Driver Usage
------------

1. Open the robot interface by connecting to the ip address of the controller in
   your web browser (e.g. ```172.16.0.2```).

2. Open the User stop and the robot brakes through the web interface.

3. Launch a Redis server instance if one is not already running.

   ```
   redis-server
   ```

3. Open a terminal and go to the driver's ```bin``` folder.

   ```
   cd <franka-panda.git>/bin
   ```

4. Launch the driver with a YAML configuration file.

   ```
   ./franka_redis_driver ../resources/default.yaml
   ```

Franka Panda Dynamics Library
-----------------------------

In addition to the Redis driver, this repo provides C++ and Python bindings to
the internal Franka Panda dynamics binary.

### C++
1. If the driver has already been compiled, you can use the following lines in
   your CMakeLists.txt:

   ```
   find_package(franka_panda REQUIRED)
   target_link_libraries(<target> PRIVATE franka_panda::franka_panda)
   ```

2. Include the following header in your source code:

   ```
   #include <franka_panda/franka_panda.h>
   ```

### Python
1. Activate your Python virtual environment (e.g. `pipenv`).

2. Locally install the `frankapanda` module:

   ```
   cd <franka-panda.git>
   pip install -e .
   ```

3. Import the `frankapanda` module in your Python code:

   ```
   import frankapanda
   ```

Redis Keys
----------

The keys can be specified in the YAML configuration file (see
`<franka-panda.git>/resources/default.yaml`). For reference, the default keys
are:

### Robot control commands

The control mode should be set ***after*** the corresponding control command has
already been set (e.g. set `tau = "0 0 0 0 0 0 0"` before `mode = torque`), or
simultaneously with MSET. Otherwise, the robot may try to execute control with
stale command values.

- `franka_panda::control::tau`: Desired control torques used during torque
  control mode. \[7d array (e.g. `"0 0 0 0 0 0 0"`)\].
- `franka_panda::control::pose`: Desired transformation matrix from end-effector
  to world frame for `cartesian_pose` control, or desired delta pose for
  `delta_cartesian_pose` control. \[4x4 array (e.g.
  `"1 0 0 0; 0 1 0 0; 0 0 1 0; 0 0 0 1"`)\].
- `franka_panda::control::mode`: Control mode. Note that the `cartesian_pose`
  controllers are blocking and must reach the target pose before receiving the
  next command. The mode will be set to `idle` after these controllers have
  finished running. \[One of {`"idle"`, `"floating"`, `"torque"`,
  `"cartesian_pose"`, `"delta_cartesian_pose"`}\].

### Gripper control commands

The gripper has two modes:
[grasp](https://frankaemika.github.io/libfranka/classfranka_1_1Gripper.html#a19b711cc7eb4cb560d1c52f0864fdc0d)
and
[move](https://frankaemika.github.io/libfranka/classfranka_1_1Gripper.html#a331720c9e26f23a5fa3de1e171b1a684).
Both of these commands are blocking and must finish before receiving the next
command.

The `move` command takes in a `width` and `speed`. The `grasp` command takes in
`width`, `speed`, `force`, and `grasp_tol`.

The control mode should be set ***after*** the control parameters have already
been set (e.g. set `width = 0` and `speed = 0.01` before `mode = move`), or
simultaneously with MSET.

- `franka_panda::gripper::control::width`: Desired gripper width. \[Positive double\].
- `franka_panda::gripper::control::speed`: Max gripper speed \[Positive double\].
- `franka_panda::gripper::control::force`: Max gripper force (used only for
  grasp c\ommand). \[Positive double\].
- `franka_panda::gripper::control::grasp_tol`: Width tolerances to determine
  whether object is grasped. \[Positive 2d array (e.g. `"0.05 0.05"`)\].
- `franka_panda::gripper::control::mode`: Gripper control mode. After a control
  command has finished, the driver will reset the mode to `idle`. \[One of
  {`"idle"`, `"grasp"`, `"move"`}\].

### Robot status

These keys will be set by the driver in response to control commands.

- `franka_panda::driver::status`: If the driver turns `off` (either due to a
  robot error or user interrupt signal), the controller should ***stop***
  immediately. Restarting the driver with an old controller already running is
  dangerous. \[One of {`"running"`, `"off"`}\].
- `franka_panda::control::status`: If the cartesian pose controller
  successfully reaches the target pose, the control status will be set to
  `finished`. This way the controller knows when to execute the next cartesian
  pose command. \[One of {`"running"`, `"finished"`, `"error"`}\].
- `franka_panda::gripper::status`: If the gripper is not running, the status
  will be `off`, and during a gripper command, it will be `grasping`. When a
  gripper command finishes, the `status` will be set to `grasped` if the object
  has been classified as grasped or `not_grasped` otherwise. \[One of {`"off"`,
  `"grasping"`, `"grasped"`, `"not_grasped"`}\].

### Robot sensor values

These keys will be set by the driver at 1000Hz.

- `franka_panda::sensor::q`: Joint configuration. \[7d array (e.g. `"0 0 0 0 0 0 0"`)\].
- `franka_panda::sensor::dq`: Joint velocity. \[7d array (e.g. `"0 0 0 0 0 0 0"`)\].
- `franka_panda::sensor::tau`: Joint torques. \[7d array (e.g. `"0 0 0 0 0 0 0"`)\].
- `franka_panda::sensor::dtau`: Joint torque derivatives. \[7d array (e.g. `"0 0 0 0 0 0 0"`)\].
- `franka_panda::gripper::sensor::width`: Gripper width. \[Positive double\].

### Robot model

These keys will be set by the driver once at the start.

- `franka_panda::model::inertia_ee`: Inertia of the attached end-effector.
  This should be attached as a load to the end-effector when computing dynamics
  quantities with the Franka Panda Dynamics Library. \[JSON {m: <mass: double>,
  com: <center of mass: array[double[3]]>, I_com: <inertia at com:
  array[double[6]]>}\].
- `franka_panda::gripper::model::max_width`: Max width of the gripper computed
  via homing. \[Positive double\].

Dynamics Library API Quick Reference
------------------------------------

### Model Class

C++: `franka_panda::Model`, Python: `frankapanda.Model`

- `dof`: Degrees of freedom (C++: `dof()`, Python: `dof`).
- `q`: Joint position (C++: `q()`/`set_q()`, Python: `q`).
- `dq`: Joint position (C++: `dq()`/`set_dq()`, Python: `dq`).
- `m_load`: Load mass (C++: `m_load()`/`set_m_load()`, Python: `m_load`).
- `com_load`: Load center of mass (C++: `com_load()`/`set_com_load()`, Python: `com_load`).
- `I_com_load`: Load inertia as a vector [Ixx Iyy Izz Ixy Ixz Iyz] (C++:
  `I_com_load()`/`set_I_com_load()`, Python: `I_com_load`).
- `I_com_load_matrix`: Load inertia as a 3x3 matrix (C++:
  `I_com_load_matrix()`/`set_I_com_load_matrix()`, Python: `I_com_load_matrix`).
- `set_load()`: Parse the load json string output by the driver.
- `g`: Gravity vector (C++: `g()`, Python: `g`)
- `inertia_compensation`: Terms added to the last 3 diagonal terms of the joint
  space inertia matrix (C++:
  `inertia_compensation()`/`set_inertia_compensation()`, Python:
  `inertia_compensation`).
- `stiction_coefficients`: Stiction coefficients for the last 3 joints. (C++:
  `stiction_coefficients()`/`set_stiction_coefficients()`, Python:
  `stiction_coefficients`).
- `stiction_activations`: Threshold above which stiction compensation should be
  applied for the last 3 joints. (C++:
  `stiction_activations()`/`set_stiction_activations()`, Python:
  `stiction_activations`).

### Dynamics Algorithms

- `Cartesian Pose`: Compute the pose of the desired link (C++: 
  `Eigen::Isometry3d CartesianPose(const Model& model, int link = -1)`, Python:
  `cartesian_pose(model: Model, link: int) -> numpy.ndarray`).
- `Jacobian: Compute the basic Jacobian of the desired link (C++:
  `Eigen::Matrix6Xd Jacobian(const Model& model, int link = -1)`, Python:
  `jacobian(model: Model, link: int) -> numpy.ndarray`).
- `Inertia`: Compute the joint space inertia matrix (C++:
  `Eigen::MatrixXd Inertia(const Model& model)`, Python:
  `inertia(model: Model) -> numpy.ndarray`).
- `Centrifugal Coriolis`: Compute the joint space centrifugal/Coriolis forces
  (C++: `Eigen::VectorXd CentrifugalCoriolis(const Model& model)`, Python:
  `centrifugal_coriolis(model: Model) -> numpy.ndarray`).
- `Gravity`: Compute the joint space gravity torque
  (C++: `Eigen::VectorXd Gravity(const Model& model)`, Python:
  `gravity(model: Model) -> numpy.ndarray`).
- `Friction`: Compute the Franka Panda stiction compensation torques to be added
  (by the user) to the input torques of the last 3 joints. Torques between
  `stiction_activations` and `stiction_coefficients` will be clipped to
  `stiction_coefficients`. Torques below `stiction_activations` smoothly decay
  to 0. Torques above `stiction_coefficients` are unaffected.
  (C++: `Eigen::VectorXd Friction(const Model& model, Eigen::Ref<const Eigen::VectorXd> tau)`,
  Python: `friction(model: Model, tau: numpy.ndarray) -> numpy.ndarray`).

Examples
--------

To run the example control apps, you will need to perform the following
additional steps.

1. Download and compile `spatial_dyn`.
2. Rebuild the driver (step 3 in the **Install** section above).

The apps are provided in both C++ and Python:

### C++

1. Build the example app:

   ```
   cd <franka-panda.git>/examples/opspace
   mkdir build
   cd build
   cmake ..
   make
   ```

2. Run the app (only use the `--sim` flag for simulation when the driver is not running):

   ```
   cd <franka-panda.git>/examples/opspace/bin
   ./franka_panda_opspace ../../../resources/franka_panda.urdf --sim
   ```

### Python

1. Activate the Python virtual environment above where you installed the
   `frankapanda` module. An example Pipfile is provided in this repo.

   ```
   cd <franka-panda.git>
   pipenv install --three
   pipenv shell
   ```

2. Locally install the `spatialdyn` module (look in
   `~/.cmake/packages/spatial_dyn` for a hint of where it's located):

   ```
   cd <spatial-dyn.git>
   pip install -e .
   ```

3. Run the app (only use the `--sim` flag for simulation when the driver is not running):

   ```
   cd <franka-panda.git>/examples/opspace/python
   ./franka_panda_opspace.py ../../../resources/franka_panda.urdf --sim
   ```
