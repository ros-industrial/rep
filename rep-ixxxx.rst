::

  REP: Ixxxx
  Title: Message Structures of the ROS-Industrial Simple Message Protocol
  Author: G.A. vd. Hoorn <g.a.vanderhoorn@tudelft.nl>
  Status: Draft
  Type: Process
  Content-Type: text/x-rst
  Created: 16-Jan-2018
  Post-History:


TODO
====

#. should a 'generic reply' (ie: ACK) message be defined (Body_)?
#. problem with 'velocity' in `JOINT_TRAJ_PT`_ (and `JOINT_TRAJ_PT_FULL`_): is that max velocity over segment, average velocity, or does it encode desired state of manipulator at a specific point in time?


Abstract
========

This REP documents the Simple Message message structures that are part of the *standard set* as defined in *Assigned Message Identifiers for the Simple Message Protocol* [#REP-I0004]_, and supported by the generic clients in the ``industrial_robot_client`` package.
Both syntax and semantics of message structures and fields are described.

Driver authors may treat this document as the normative reference for message type structures in the standard set within the ROS-Industrial Simple Message protocol.


Outline
=======

#. Abstract_
#. Motivation_
#. Definitions_
#. Assumptions_
#. Overview_

   #. `Model of Operation`_
   #. `Bytestream Layout`_
   #. `Shared Types`_
   #. Endianness_

#. `Message Layout`_

   #. Prefix_
   #. Header_
   #. Body_

#. `Message Definitions`_

   #. PING_
   #. GET_VERSION_
   #. JOINT_POSITION_
   #. JOINT_TRAJ_PT_
   #. JOINT_TRAJ_
   #. STATUS_
   #. JOINT_TRAJ_PT_FULL_
   #. JOINT_FEEDBACK_

#. `Defined Constants`_

   #. `Reply Codes`_
   #. `Communication Types`_
   #. `Special Sequence Numbers`_
   #. Tri-states_
   #. `Valid fields`_

#. References_
#. `Appendix A - Bytestream Examples`_

   #. `Example: JOINT_POSITION`_
   #. `Example: JOINT_TRAJ_PT`_
   #. `Example: STATUS`_

#. `Revision History`_
#. Copyright_


Motivation
==========

Like other network protocols, the ROS-Industrial Simple Message protocol [#simple_message]_ defines a set of message structures to allow senders and receivers to exchange information in a structured and consistent way.
In order to be able to provide generic implementations of the Simple Message protocol (de)serialisation libraries, to avoid potential incompatibility between clients and servers and to assist developers in implementing new drivers, a central registry of defined message identifiers, their structures and their semantics is essential.
Identifiers are documented in *Assigned Message Identifiers for the Simple Message Protocol* [#REP-I0004]_.
Message structures and their semantics are described in this document.

This document provides the normative reference for all messages that are part of the *standard set*, and are thus supported by the generic clients in the ``industrial_robot_client`` package.
Vendor specific and messages in any of the *freely assignable* ranges are not included in this document.


Definitions
===========

Controller
    The device that provides access to the Robot motion control capabilities.
    It is often the same (computing) device that runs the Server side of a ROS-Industrial (robot) driver.
    Controllers may be configured with zero or more Motion Groups that can be controlled by (motion) programs running on the controller.
Motion Group
    A set of joints in a particular kinematic configuration (a chain fi) that are controlled as a single group, independent from other such groups.
    In industrial robot controllers, a single motion group typically contains the axes of the attached manipulator and a second group may contain any additional axes.
Server
    A software component exposing a Simple Message compatible message interface that offers services and datastreams for clients to connect to.
Client
    A software component implementing a Simple Message compatible message interface that wishes to make use of the services and datastreams offered by clients.
Trajectory Point
    A single point in a trajectory for a robotic manipulator that encodes a position in joint or Cartesian space, with additional associated constraints (fi: desired velocities, accelerations or efforts).


Assumptions
===========

#. The key words *must*, *must not*, *required*, *shall*, *shall not*, *should*, *should not*, *recommended*, *may*, and *optional* in this document are to be interpreted as described in RFC-2119 [#RFC2119]_.
#. Where applicable, fields with units will adhere to ROS REP-103 [#REP103]_.
#. Message type identifiers, when given, will always use decimal (base-ten) notation, unless mentioned otherwise.


Overview
========

The Simple Message (SimpleMessage) protocol defines the message structure between the ROS driver layer and the robot controller itself as used by the generic nodes in the ``industrial_robot_client`` package of ROS-Industrial.

Requirements and constraints that influenced its design were:

#. Format should be simple enough that code can be shared between ROS and the controller (for those controllers that support C/C++).
   For those controllers that do not support C/C++, the protocol must be simple enough to be decoded with the limited capabilities of the typical robot programming language.
   A corollary to this requirement is that the protocol should not be so onerous as to overwhelm the limited resources of the robot controller.
#. Format should allow for data streaming (ROS *topic like*).
#. Format should allow for data reply (ROS *service like*).
#. The protocol is not intended to encapsulate version information.
   It is up to individual developers to ensure that code developed for communicating platforms does not have any version conflicts (this includes message type identifiers).

Note: these were the design requirements and constraints at the time the protocol was first created (2012).
Since then, numerous similar protocols have been created that would undoubtedly have been considered for adoption instead of creating a new protocol.
As a retrospective REP, this document only describes the *existing situation*, so it will not include a discussion of design alternatives nor extensive rationale for why the protocol implementation is as it is today.


Model of Operation
------------------

Simple Message is client-server based.
Server programs (often written in OEM proprietary languages) run on the industrial controller, with generic clients provided by the ``industrial_robot_client`` package.
Note that it is acceptable for drivers to *not* make use of the generic clients, for instance because of special requirements to the control flow or data formatting restrictions which cannot be easily integrated into the ``industrial_robot_client`` nodes.

The default transport is TCP/IP, with UDP/IP an option (but not directly supported by the generic clients).
On startup, the generic clients will try to open a connection to a server program running on the OEM controller at the configured IP and port.
In some cases, there will be multiple server programs running concurrently, each responsible for a specific task, such as (network) communication, robot state gathering, motion sequencing and execution or error reporting and handling.
This is most often the case when controllers do not support integrating all such tasks into a single program (for instance: servers may not allow a single program to control multiple motion groups, requiring an instance of the server for each group).
In such cases it's common for server programs to listen on different TCP or UDP ports, and clients to connect to different ports for the different functions.
It is the servers responsibility then to coordinate those programs and the client's access to them.

In almost all cases, Simple Message server programs are plain, user-level task programs, without any special access to OEM controller internals, motion primitives.
They also do not bypass any safety systems present in the OEM controller.
This immediately implies that such server programs are subject to all the same limitations as other programs written in the OEM's language (both in performance as well as interaction with any safety systems).
A further consequence is that clients are *not* in direct control of the robot: clients send requests to server programs which act on their behalf, but also only *proxy* the functionality offered by the OEM controller itself.

All *state relay*-type server programs broadcast robot state periodically in *topic like* messages.
Those messages are received by clients, converted into corresponding ROS messages and forwarded to the rest of the ROS application.
Clients command motion by *enqueuing* trajectory points on the server side using *service like* messages sent to *trajectory relay* programs.
Similar to the ROS messages used for representing trajectories,
Trajectory relays execute motion either directly, or via additional programs which take care of interacting with the OEM controller's motion sub system(s).


Bytestream Layout
-----------------

Bytestream layout is straightforward in Simple Message and there is little difference between traffic carried over TCP or UDP connections.
Reassembly of fragmented messages, where necessary, must be performed by the client or server in cases where the underlying protocol does not support this (ie: for UDP connections).

Message content is serialised from its in-memory representation for transmission and must be packed to both for efficiency reasons as well as to avoid issues due to differences in alignment between client and server platforms.
Bytestreams shall consist of a length byte, followed by the bytes constituting the message header, followed by the bytes encoding the payload section of the message.

There are no magic byte sequences defined, nor any other form of sync or section marker bytes.
Byte-counting is used to correctly segment incoming bytestreams and for deserialising incoming data into the respective message structures.

The base Simple Message specification also does not prescribe any form of versioning on bytestreams nor any method for detecting incompatible servers or clients.
If such functionality is required, implementations are suggested to do this at the application level.


Shared Types
------------

All message structures are aggregates of fields with a type from the set of *shared types*.

The following set has been defined (type sizes are in bytes)::

  Name         Base type        Size

  shared_int   int32               4
  shared_real  float32/float64   4/8

This set of shared types is typically used if the system software of the target controller is capable of running plain C/C++ binaries that may make use of the provided Simple Message libraries.
As controller vendors determine the sizes of the various types their runtime platforms support, and exchanging binary data between different systems requires a common encoding and layout, the shared types defined by Simple Message ensure compatibility between clients and servers by fixing both type and byte size of types used for message fields.

For controllers that only support programs in a custom, vendor defined language, drivers authors must make sure that type definitions for the shared types correspond to those the server implementation uses.

Note that ``shared_real`` can be an alias for both a 4-byte real value (ie: ``float``) or a 8-byte real value (ie: ``double``), depending on the types and sizes defined by the controller programming and runtime platform.


Endianness
----------

In order to accomodate controllers that need it, the generic nodes in ``industrial_robot_client`` support both little and big-endian data streams.
These are implemented as *swap* and *non-swap* variants of all nodes in the package.

As an example: a little-endian client wishing to connect to a server running on a controller that uses network (or big-endian) byte ordering should use the *swap* version of the client nodes.
A little-endian client connecting to a controller sending data in little-endian byte order should use the *non-swapping* version of the client nodes (as byte swaps are not needed in this case).

The following table shows the supported variants (type sizes are in bytes)::

              Type
  Node type   Integer   Real

  No swap           4      4
  Swap              4      4
  No swap           4      8

Swapping node variants will byte-swap fields of incoming messages, while taking field sizes into account.


Message Layout
==============

The following sections describe the different sub structures that make up a valid Simple Message message.


Prefix
------

All messages must start with the *prefix*, which may only contain a single field: ``length``.
Message length is defined as the sum in bytes of the sizes of the individual fields in the *header* and the *body*, excluding the ``length`` field itself (ie: only actual message bytes are considered).

Layout::

  length           : shared_int

Notes

#. Refer to section `Shared Types`_ for information on the size of supported field types.
#. The size of fields that are arrays or lists shall be defined as the size of their base type (ie: ``shared_int``) multiplied by the number of elements in the list, or the declared size of the array.
   Example: an array of ``shared_int`` with ten (``10``) elements in it has a total size of forty (``40``) bytes.


Header
------

The next three fields make up the *message header*, which is that part of the message that encodes the message type (or it's intent), whether the message is a topic-like broadcast, a service request or reply and, if it is a reply, what the result of the service request was.

Layout::

  msg_type         : shared_int
  comm_type        : shared_int
  reply_code       : shared_int

Notes

#. Refer to [#REP-I0004]_ for valid values for the ``msg_type`` field.
#. Refer to `Communication Types`_ for valid values for the ``comm_type`` field.
#. Refer to `Reply Codes`_ for valid values for the ``reply_code`` field.
#. For ``TOPIC`` and ``SERVICE_REQUEST`` type messages, the ``reply_code`` field must be set to ``INVALID``.
#. The ``SUCCESS`` and ``FAILURE`` reply codes shall only be used with ``SERVICE_REPLY`` type messages.
   They are not valid for any other message type.
#. The ``TOPIC`` communication type shall only be used when the sender does not need the recipient to acknowledge the message.
#. Receivers shall ignore (ie: take no action upon receipt) incoming ``TOPIC`` messages they do not support.
#. Incoming ``SERVICE_REQUEST`` messages requesting use of a service that the receiver does not support shall result in a ``SERVICE_REPLY`` being sent by the receiver with the ``reply_code`` set to ``FAILURE``.
   No further action shall be taken.
#. The ``reply_code`` field shall only be used to communicate whether a service invocation was possible, not whether the action the request initiated was successful or not.
   In case of failure to invoke the service, implementations shall set ``reply_code`` to ``FAILURE`` and any payload must be considered invalid by clients (ie: may be uninitialised).
#. Implementations shall ignore incoming ``SERVICE_REPLY`` messages for which no outstanding ``SERVICE_REQUEST`` exists.
#. Implementations shall warn the user of any incoming messages with the ``comm_type`` field set to either invalid or unsupported values.
   The message itself is then to be ignored.


Body
----

The *body* is that part of the message which consists of all fields that are not part of either the prefix or the message header.
Most message structures described in the `Message Definitions`_ section have a body part, but this is not required.
Messages may consist of only a prefix and a header, for example in the case of pure acknowledgements that carry no data.

In cases where fixed-size messages are required, an array of ``shared_int`` dummy values may be used.
All elements must be initialised to zero (``0``).

Layout: the layout of the body is message specific.
See the definitions in the `Message Definitions`_ section for more information.

Notes: none.


Message Definitions
===================

The following sections describe the message structures that make up the standard set of the Simple Message protocol.

Values given as *assigned message identifiers* are further described in [#REP-I0004]_.


PING
----

This message may be used by clients to test communication with the server.

Server implementations should respond to incoming ``PING`` messages with minimal delay.

Message type: *synchronous service*

Assigned message identifier: 1 (``0x01``)

Status: *active, in use*

Supported by generic nodes: *yes*

Dataflow direction (typically): client → server, server → client

Request::

  Prefix
  Header
  data             : shared_int[10]

Reply::

  Prefix
  Header
  data             : shared_int[10]

Notes

#. The contents of ``data`` is to be ignored by both client and server.
#. All elements in ``data`` must be initialised to zero (``0``).


GET_VERSION
-----------

Allows clients to determine the specific version of a server implementation running on the remote system.
This version number may be specific to the server, and thus cannot be used to compare different server implementations.

Message type: *synchronous service*

Assigned message identifier: 2 (``0x02``)

Status: *active, but not in use*

Supported by generic nodes: *no*

Dataflow direction (typically): client → server

Request::

  Prefix
  Header

Reply::

  Prefix
  Header
  major            : shared_int
  minor            : shared_int
  patch            : shared_int

Notes

#. Fields not used by the server shall be set to zero (``0``).
#. Server implementations may return alphanumeric version info in any of the ``major``, ``minor`` or ``patch`` fields, but this may result in rendering artefacts on the client side.
   The generic clients in ``industrial_robot_client`` will always interpret these fields as signed integers.


JOINT_POSITION
--------------

This message was part of the first set of messages supported by the generic clients that servers could use to report joint states.
There is no support for joint velocity, acceleration or effort, nor a group identifier or index.
The message size is fixed and the maximum number of joints supported is ten (``10``).

Early server implementations also accepted this message for enqueuing trajectory points.
This usage has been deprecated (and support removed from ``industrial_robot_client``) and it is an error for clients to try to use ``JOINT_POSITION`` for this purpose.
Driver authors must use `JOINT_TRAJ_PT`_ and `JOINT_TRAJ_PT_FULL`_ messages instead.

Note that use of this message for encoding joint states is also deprecated (because of the lack of support for motion groups, joint velocity or accelerations mentioned earlier in this section), and new server implementations are recommended to use `JOINT_FEEDBACK`_ to encode joint states.

For an example bytestream, see `Example: JOINT_POSITION`_.

Message type: *asynchronous publication*

Assigned message identifier: 10 (``0x0A``)

Status: *deprecated*

Supported by generic nodes: *yes* (joint state reporting), *no* (enqueuing)

Dataflow direction (typically): client → server, server → client

Message::

  Prefix
  Header
  sequence         : shared_int
  joint_data       : shared_real[10]

Notes

#. The ``sequence`` field uses zero-based numbering.
#. The ``sequence`` field is not used when reporting joint state and shall be set to zero (``0``) by server implementations.
#. Elements of ``joint_data`` that are not used must be initialised to zero (``0``) by the sender.
#. The size of the ``joint_data`` array is ``10``, even if the server implementation does not need that many elements (for instance because it only has six joints).
#. Controllers that support or are configured with more than a single motion group should use the `JOINT_FEEDBACK`_ message if they wish to report joint state for all configured motion groups (note: unfortunately there is currently no support for `JOINT_FEEDBACK`_ messages in the generic clients, that will have to be added).
#. The elements of the ``joint_data`` field shall represent the joint space positions of the corresponding joint axes of the controller.
   In accordance with [#REP103]_, units are *radians* for revolute or rotational axes, and *metres* for prismatic or translational axes.


JOINT_TRAJ_PT
-------------

This message may be used by clients to enqueue trajectory points for execution by the server.

See `Example: JOINT_TRAJ_PT`_ for bytestream example.

Message type: *synchronous service*

Assigned message identifier: 11 (``0x0B``)

Status: *active, in use*

Supported by generic nodes: *yes*

Dataflow direction (typically): client → server

Request::

  Prefix
  Header
  sequence         : shared_int
  joint_data       : shared_real[10]
  velocity         : shared_real
  duration         : shared_real

Reply::

  Prefix
  Header
  dummy_data       : shared_real[10]

*Alternative* reply::

  Prefix
  Header

Notes

#. The *alternative* reply will be understood by the generic nodes in ``industrial_robot_client``, but other clients may expect a full, fixed size message, which would include the ``dummy_data``.
#. Drivers shall set the value of the ``reply_code`` field in the ``Header`` of the reply messages to *the result of the enqueueing operation* of the trajectory point that was transmitted in the request.
   It shall *not* be used to report the success or failure of the *execution* of the motion.
   Drivers should use the appropriate fields in `STATUS`_ for that (note: the generic nodes in ``industrial_robot_client`` currently ignore the ``reply_code`` field of incoming `JOINT_TRAJ_PT`_ replies (see [#irc_issue118]_). Nevertheless, server implementations must send replies to incoming `JOINT_TRAJ_PT`_ requests. Failure to do so will prevent the exchange of further messages).
#. Refer to `Special Sequence Numbers`_ for valid values for the ``sequence`` field.
#. Driver authors must abort any motion executing on the controller on receipt of a message with ``sequence`` set to ``STOP_TRAJECTORY``.
   Note that such messages must also be acknowledged with a reply message.
#. Servers must abort any motion executing on the controller on receipt of an out-of-order trajectory point (ie: ``(seq(msg_n) - seq(msg_n-1)) != 1``), except when clients wish to start a new trajectory (ie: ``seq(msg) == 0``).
#. Elements of ``joint_data`` that are not used must be initialised to zero (``0``) by the sender.
#. The number of elements in the ``joint_data`` array must always be equal to ten (``10``), even if the server implementation does not need that many elements (for instance because it only has six joints).
#. Controllers that support or are configured with more than a single motion group should use the `JOINT_TRAJ_PT_FULL`_ message if they wish to relay trajectories for all configured motion groups.
#. The elements of the ``joint_data`` field shall represent the joint space positions of the corresponding joint axes of the controller.
   Units are *radians* for rotational or revolute axes, and *metres* for translational or prismatic axes (see also [#REP103]_).
#. The ``duration`` field represents total segment duration for all joints in seconds [#REP103]_.
   The generic nodes calculate this duration based on the time needed by the slowest joint to complete the segment.
   As an alternative to the ``duration`` field, the value of the ``velocity`` field is a value representing the fraction ``(0.0, 1.0]`` of maximum joint velocity that should be used when executing the motion for the current segment.
   Driver authors may use whichever value is more conveniently mapped onto motion primitives supported by the controller.
#. Durations or velocities that are outside bounds set by the controller shall result in an error being returned by the server.


JOINT_TRAJ
----------

This message was used to encode entire ROS ``JointTrajectory`` messages into Simple Message messages.
Primarily intended to be used by *downloading* drivers, it included a fixed length array with instances of `JOINT_TRAJ_PT`_.

New download drivers should not use this message anymore, but instead should use either `JOINT_TRAJ_PT`_ or `JOINT_TRAJ_PT_FULL`_ and buffer on the server side.
`Special Sequence Numbers`_ can be used to indicate trajectory start and end.

Message type: *synchronous service*

Assigned message identifier: 12 (``0x0C``)

Status: *deprecated*

Supported by generic nodes: *no*

Dataflow direction (typically): client → server

Message::

  Prefix
  Header
  size             : shared_int
  points           : JOINT_TRAJ_PT[10]

Reply::

  Prefix
  Header
  dummy_data       : shared_real[10]

*Alternative* reply::

  Prefix
  Header

Notes

#. The *alternative* reply will be understood by the generic nodes in ``industrial_robot_client``, but other clients may expect a full, fixed size message, which would include the ``dummy_data``.


STATUS
------

The ``STATUS`` message may be used by servers to inform clients of the general status of the controller, including whether the controller is currently in an error state, whether the emergency stop is active, whether any attached robot is executing a motion and what the current operating mode of the controller is.

This version of ``STATUS`` can only encode an aggregate state, so drivers for controllers with multiple motion groups will need to determine how to merge group state into an aggregate controller state.

See `Example: STATUS`_ for bytestream example.

Message type: *asynchronous publication*

Assigned message identifier: 13 (``0x0D``)

Status: *active, in use*

Supported by generic nodes: *yes*

Dataflow direction (typically): server → client

Message::

  Prefix
  Header
  drives_powered   : shared_int
  e_stopped        : shared_int
  error_code       : shared_int
  in_error         : shared_int
  in_motion        : shared_int
  mode             : shared_int
  motion_possible  : shared_int

Valid values for ``mode`` are::

  Val  Name     Description

   -1  UNKNOWN  Controller mode cannot be determined or is not one of those
                defined in ISO 10218-1
    1  MANUAL   Controller is in ISO 10218-1 'manual' mode
    2  AUTO     Controller is in ISO 10218-1 'automatic' mode

All other values are reserved for future use.

Notes

#. The fields ``drives_powered``, ``e_stopped``, ``in_error``, ``in_motion`` and ``motion_possible`` are tri-states.
   Refer to `Tri-states`_ for valid values for these fields.
#. Fields for which a driver cannot determine a value shall be set to ``UNKNOWN``.
#. The ``error_code`` field should be used to store the integer representation (id, number or code) of the error that caused the robot to go into an error mode.
#. If the controller can be set to modes other than those defined in ISO 10218-1, drivers shall report ``UNKNOWN`` for those modes.
#. ``motion_possible`` shall encode whether the controller is in a state that would allow immediate execution of a new incoming trajectory.
   Industrial robot controllers may expose such information directly (fi, through a dedicated function call, a special variable or some other way).
   In all other cases driver authors are expected to include appropriate logic in servers that can derive whether motion should be possible (ie: by examining multiple other sources of information).


JOINT_TRAJ_PT_FULL
------------------

Similar to `JOINT_TRAJ_PT`_, this message may be used by clients to enqueue trajectory points for execution by the server.
In addition to position, ``JOIN_TRAJ_PT_FULL`` allows clients to encode velocity and acceleration, as well as to target messages at specific motion groups.

This message is intended for use with servers that provide sufficient control over both motion planning and trajectory execution.

Message type: *synchronous service*

Assigned message identifier: 14 (``0x0E``)

Status: *active, in use*

Supported by generic nodes: *no* (``motoman_driver`` only)

Dataflow direction (typically): client → server

Request::

  Prefix
  Header
  robot_id         : shared_int
  sequence         : shared_int
  valid_fields     : shared_int
  time             : shared_real
  positions        : shared_real[10]
  velocities       : shared_real[10]
  accelerations    : shared_real[10]

Reply::

  Prefix
  Header
  dummy_data       : shared_real[10]

*Alternative* reply::

  Prefix
  Header

Notes

#. The *alternative* reply will be understood by the generic nodes in ``industrial_robot_client``, but other clients may expect a full, fixed size message, which would include the ``dummy_data``.
#. Drivers shall set the value of the ``reply_code`` field in the ``Header`` of the reply messages to the result of the *enqueueing operation* of the trajectory point that was transmitted in the request.
   It shall *not* to be used to report the success or failure of the *execution* of the motion.
   Drivers should use the appropriate fields in `STATUS`_ for that (note: the generic nodes in ``industrial_robot_client`` currently ignore the ``reply_code`` field of incoming `JOINT_TRAJ_PT_FULL`_ replies (see [#irc_issue118]_). Nevertheless, server implementations must send replies to incoming `JOINT_TRAJ_PT_FULL`_ requests. Failure to do so will prevent the exchange of further messages).
#. The value of the ``robot_id`` field shall match that of the numeric identifier of the corresponding motion group on the controller.
   This field uses zero-based counting.
   In cases where motion groups are not identified by numeric ids on the controller, drivers shall implement an appropriate mapping (ie: alphabetical sorting of group names, etc).
#. Refer to `Special Sequence Numbers`_ for valid values for the ``sequence`` field.
#. Driver authors must abort any motion executing on the controller on receipt of a message with ``sequence`` set to ``STOP_TRAJECTORY``.
   Note that such messages must also be acknowledged with a reply message.
#. Servers must abort any motion executing on the controller on receipt of an out-of-order trajectory point (ie: ``(seq(msg_n) - seq(msg_n-1)) != 1``), except when clients wish to start a new trajectory (ie: ``seq(msg) == 0``).
#. Refer to `Valid fields`_ for defined bit positions for the ``valid_fields`` field.
#. Drivers shall set all undefined bit positions in ``valid_fields`` to zero (``0``).
#. Drivers shall set all elements of invalid fields (as encoded by ``valid_fields``) to zero (``0``).
#. Elements of ``positions``, ``velocities`` and ``accelerations`` that are not used must be initialised to zero (``0``) by the sender.
#. The number of elements in the ``positions``, ``velocities`` and ``accelerations`` arrays must always be equal to ten (``10``), even if the server implementation does not need that many elements (for instance because it only has six joints).
#. Velocity control interfaces may be implemented by servers by accepting ``JOIN_TRAJ_PT_FULL`` messages that have the ``valid_fields`` bitmask set to ``VELOCITY``.


JOINT_FEEDBACK
--------------

The analogue of `JOINT_TRAJ_PT_FULL`_, but for robot joint state reporting.

Message type: *asynchronous publication*

Assigned message identifier: 15 (``0x0F``)

Status: *active, in use*

Supported by generic nodes: *no* (``motoman_driver`` only)

Dataflow direction (typically): server → client

Message::

  Prefix
  Header
  robot_id         : shared_int
  valid_fields     : shared_int
  time             : shared_real
  positions        : shared_real[10]
  velocities       : shared_real[10]
  accelerations    : shared_real[10]

Notes

#. Refer to `Special Sequence Numbers`_ for valid values for the ``sequence`` field.
#. The value of the ``robot_id`` field shall match that of the numeric identifier of the corresponding motion group on the controller.
   This field uses zero-based counting. In cases where motion groups are not identified by numeric ids on the controller, drivers shall implement an appropriate mapping (ie: alphabetical sorting of group names, etc).
#. Refer to `Valid fields`_ for defined bit positions for the ``valid_fields`` field.
#. Drivers shall set all undefined bit positions in ``valid_fields`` to zero (``0``).
#. Drivers shall set all elements of invalid fields (as encoded by ``valid_fields``) to zero (``0``).
#. Elements of ``positions``, ``velocities`` and ``accelerations`` that are not used must be initialised to zero (``0``) by the sender.
#. The number of elements in the ``positions``, ``velocities`` and ``accelerations`` arrays must always be equal to ten (``10``), even if the server implementation does not need that many elements (for instance because it only has six joints).
#. This message does not currently support motion controllers that support more than ten (``10``) axes in a single motion group.
   A suggested work-around is to divide the total number of axes over a number of *virtual* motion groups and use additional processing logic on the client side to recombine multiple ``JOINT_FEEDBACK`` messages into a single representation of controller joint state.


Defined Constants
=================

This section documents all shared constants as defined in the Simple Message protocol.
Constants defined in this section are recognised by the generic nodes in the ``industrial_robot_client`` package and shall be used by compliant drivers.


Communication Types
-------------------

::

  Val  Name             Description

    0  INVALID          Reserved value. Do not use.
    1  TOPIC            Message needs no acknowledgement
    2  SERVICE_REQUEST  Sender requires acknowledgement
    3  SERVICE_REPLY    Message is a reply to a request

All other values are reserved for future use.


Reply Codes
-----------

::

  Val  Name     Description

    0  INVALID  Also encodes UNUSED
    1  SUCCESS  Receiver processed the message succesfully
    2  FAILURE  Receiver encountered a failure processing the message

All other values are reserved for future use.


Special Sequence Numbers
------------------------

::

  Val  Name                        Description

    N                              Index into current trajectory
   -1  START_TRAJECTORY_DOWNLOAD   Downloading drivers only: signals start
   -2  START_TRAJECTORY_STREAMING  Signal start of trajectory
   -3  END_TRAJECTORY              Downloading drivers only: signals end
   -4  STOP_TRAJECTORY             Driver must abort any currently executing motion

All other *negative* values are reserved for future use.

Note: ``START_TRAJECTORY_STREAMING`` is *not* used by the generic nodes in ``industrial_robot_client`` (ie: no message to indicate start of a new trajectory will be sent).


Tri-states
----------

::

  Val  Name     Description

   -1  UNKNOWN  -
    0  OFF      Also encodes FALSE, DISABLED or LOW
    1  ON       Also encodes TRUE, ENABLED or HIGH

All other values are reserved for future use.


Valid fields
------------

Bit positions are counted starting from LSB::

  Pos  Name          Description

    0  TIME          The 'time' field contains valid data
    1  POSITION      The 'positions' field contains valid data
    2  VELOCITY      The 'velocities' field contains valid data
    3  ACCELERATION  The 'accelerations' field contains valid data

All other positions are reserved for future use.


References
==========

.. [#RFC2119] Key words for use in RFCs to Indicate Requirement Levels, on-line, retrieved 16 January 2018
   (https://tools.ietf.org/html/rfc2119)
.. [#REP103] Standard Units of Measure and Coordinate Conventions, on-line, retrieved 16 January 2018
   (http://www.ros.org/reps/rep-0103.html)
.. [#simple_message] ROS-Industrial simple_message package, ROS Wiki, on-line, retrieved 16 January 2018
   (http://wiki.ros.org/simple_message)
.. [#REP-I0004] REP-I0004 - Assigned Message Identifiers for the Simple Message Protocol, on-line, retrieved 16 January 2018
   (https://github.com/ros-industrial/rep/blob/master/rep-I0004.rst)
.. [#irc_issue118] industrial_robot_client: joint_trajectory_streamer doesn't check reply msg from ctrlr in streamingThread
   (https://github.com/ros-industrial/industrial_core/issues/118)


Appendix A - Bytestream Examples
================================

This section provides three annotated examples of bytestreams driver authors can expect to be sent and received by the generic nodes in the ``industrial_robot_client`` package.

Note that the hexadecimal numbers are displayed in big-endian byte-order.


Example: JOINT_POSITION
-----------------------

This shows a stream for a ``JOINT_POSITION`` message, sent by a server to broadcast joint state for a six-axis robot that is close to its zero position.

Direction: server → client

Complete packet::

  00000038 0000000A
  00000001 00000000
  00000000 B81AD9FA
  B6836312 B7C043F5
  B8B81516 B865D055
  B8B6365E 00000000
  00000000 00000000
  00000000

Field description::

  Hex       Field              Description

            Prefix
  00000038    length           56 bytes

            Header
  0000000A    msg_type         Joint Position
  00000001    comm_type        Topic
  00000000    reply_code       Unused / Invalid

            Body
  00000000    sequence          0 (unused)
  B81AD9FA    joint_data[0]    -0.000036919
  B6836312    joint_data[1]    -0.000003916
  B7C043F5    joint_data[2]    -0.000022920
  B8B81516    joint_data[3]    -0.000087777
  B865D055    joint_data[4]    -0.000054792
  B8B6365E    joint_data[5]    -0.000086886
  00000000    joint_data[6]     0.000000000
  00000000    joint_data[7]     0.000000000
  00000000    joint_data[8]     0.000000000
  00000000    joint_data[9]     0.000000000


Example: JOINT_TRAJ_PT
----------------------

The following is a bytestream for a serialised ``JOINT_TRAJ_PT`` sent be a client to a server to request the second trajectory point in a trajectory be queued for execution by the controller.
This is for a six-axis robot.

Direction: client → server

Complete packet::

  00000040 0000000B
  00000002 00000000
  00000001 A7600000
  3EA7CDE8 BF5D9E57
  C0490FDB 3F34815F
  C0490FDB 00000000
  00000000 00000000
  00000000 3DCCCCCD
  40A00000

Field description::

  Hex       Field              Description

            Prefix
  00000040    length           64 bytes

            Header
  0000000B    msg_type         Joint Trajectory Point
  00000002    comm_type        Service Request
  00000000    reply_code       Unused / Invalid

            Body
  00000001    sequence          1 (second TrajectoryPoint)
  A7600000    joint_data[0]    -0.000000000
  3EA7CDE8    joint_data[1]     0.327742815
  BF5D9E57    joint_data[2]    -0.865697324
  C0490FDB    joint_data[3]    -3.141592741
  3F34815F    joint_data[4]     0.705099046
  C0490FDB    joint_data[5]    -3.141592741
  00000000    joint_data[6]     0.000000000
  00000000    joint_data[7]     0.000000000
  00000000    joint_data[8]     0.000000000
  00000000    joint_data[9]     0.000000000
  3DCCCCCD    velocity          0.1
  40A00000    duration          5.0


Example: STATUS
---------------

This is a bytestream encoding a ``STATUS`` message for a six-axis robot that is in auto-mode, not moving, not in an error mode, of which the servo drives are powered and is ready to execute a new trajectory.
Note that the state of the e-stop could not be determined by the driver, and is thus reported as ``UNKNOWN``.

Direction: server → client

Complete packet::

  00000028 0000000D
  00000001 00000000
  00000001 FFFFFFFF
  00000000 00000000
  00000000 00000002
  00000001

Field description::

  Hex       Field              Description

            Prefix
  00000028    length           40 bytes

            Header
  0000000D    msg_type         Status
  00000001    comm_type        Topic
  00000000    reply_code       Unused / Invalid

            Body
  00000001    drives_powered   True
  FFFFFFFF    e_stopped        Unknown
  00000000    error_code       0
  00000000    in_error         False
  00000000    in_motion        False
  00000002    mode             Auto
  00000001    motion_possible  True


Revision History
================

::

  2018-Jan-16   Initial revision


Copyright
=========

This document has been placed in the public domain.
