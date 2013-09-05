REP: 140
Title: Industrial Robot Controller Motion/Status Interface (Version 2)
Author: Shaun Edwards
Status: Draft
Type: Standards Track
Content-Type: text/x-rst
Created: 19-August-2013

Outline
=======

#. Abstract_
#. Motivation_
#. Rationale_
#. `Data representation`_
#. Compatibility_
#. References_
#. Copyright_


Abstract
========

This REP specifies the second version of the ROS-Industrial robot controller client/server motion/status interface.  It is relevant to anybody using ROS-I standard libraries to communicate with an industrial robot controller.  Note: This spec does not specify the ROS API, which is specified seperately.


Motivation
==========

The first version of the motion/status interface (loosely documented in code and wikis) worked well for single arm manipulators.  Multi-arm (seperate or syncronized) was not supported in a generic manor that could be used across all platforms.  Some attempts were made to support multiple arms using the existing interface, but these often required multi-arm joint states which could exceed the max number of joints and did not allow for mixed multi-arm/single-arm motion.

Definitions
=========

joint, axis - A single controllable item

group, motion group - A group of joints or axes.  At the controller level these are non-overlapping (i.e. a joint can be in one and only one group.

synchronous motion - coordinated motion (at the real time level) of a group of joints

asynchronous motion - independent control of joints or groups.

Requirements
=========

The industrial robot controller motion interface shall meet the following requirements:
 * A single node shall act as the point of control (motion).  All trajectory topics shall be routed through this node to the controller.
 * It shall allow for synchronous and asynchronous control of single and multiple arms (i.e. a unified driver that allows for differerent types of control to be achieved dynamically without reconfiguration
 * All incoming joint trajectories should be fully populated with position, velocity, accleration, and timing information. (NOTE: Certain controllers may not require all data, but this will be different between controllers)
 * Joint groups shall be statically defined at the controller level.  ROS trajectories must define motion for all the joints in a group (NOTE: Some controllers have additional axes that aren't used by all robots.  In these cases the axes are ignored by the controller.  However, the motion node should be smart enough to reorder or shift joint values based on configuration data). 
 * Joint trajectories shall be executed one at a time.  Receipt of a new trajectory before the current trajectory is finished shall either result in a motion stop, followed by the execution of the new trajectory or intelligent splicing of the current and new trajectories (depends on the capability of the controller).
 * Joint states shall be published for each arm (joint group) in a single dynamic message
 * Robot status messages shall be published for each arm (joint group) in a single dynamic message.  Although common status indicators may exist (like an e-stop), these will be generated for each joint group. 
 * The controller joint group structure shall be statically defined at node start up (i.e. in a yaml file read on node construction).  Motion groups are predifined (statically) by most controllers.
 
Design Assumptions
=========
 * All joints (regardless of group) shall be uniquely named.
 
Motion Interface
=========

Motion Downloading Vs Streaming
---------
In the first version of the motion interface, some robots allowed motion streaming (ie. point by point) and others required motion downloading (i.e. entire trajectory).  This distinction was invisible to the user, as the ROS interface receives entire trajectories in a single message.  Motion download interfaces were created because it was thought that they would provide better (smoother and faster) motion, altough this hasn't been found to be true.  Dense trajectories resulted in the same slow, disjointed motion as motion streaming interfaces.  For the purposes of this second version, only streaming interfaces will be considered.  This simplifies the problem of switching between synchronous and asyncrounous motion.

Motion Variants
---------
The motion interface can be expressed as four variations:
 * Single Arm - Only a single arm group is defined, no synchronization required.
 * Multi-Arm (Sync) - Multiple arms are defined.  A single joint trajectory containing all joints is received and sent to the controller in a single simple message.  The controller receives the message and performs synchronized motion.
 * Multi-Arm (Async) - Multiple arma are defined.  Multiple joint trajectories for each arm/motion group are received and sent to the controller in independent messages.  The controller receives the messages and performs asynchronous motion.  NOTE: Although this may look like syncronized motion there isn't a real time guarentee tha the waypoints across multiple groups are reached at the same time.
 * Multi-Arm (Sync & Async) - Combination of the two above operating modes.  

References
==========
.. [x] Google group discussion: Support for Dual-arm robots (https://groups.google.com/forum/#!topic/swri-ros-pkg-dev/LHrfVgEA4hs)

Copyright
=========

This document has been placed in the public domain.


..
   Local Variables:
   mode: indented-text
   indent-tabs-mode: nil
   sentence-end-double-space: t
   fill-column: 70
   coding: utf-8
   End: