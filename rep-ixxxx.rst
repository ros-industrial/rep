::

  REP: XXX
  Title: Assigned Message Identifiers for the Simple Message Protocol
  Author: G.A. vd. Hoorn <g.a.vanderhoorn@tudelft.nl>
  Status: Draft
  Type: Process
  Content-Type: text/x-rst
  Created: 01-Jun-2014
  Post-History: 15-Aug-2014, 08-Oct-2014, 19-Nov-2014


Outline
=======

#. Abstract_
#. Motivation_
#. `Application Procedure`_
#. Assumptions_
#. `Assigned Message Identifiers`_

   #. `Standard Messages`_
   #. `Vendor Specific Ranges`_

#. References_
#. `Revision History`_
#. Copyright_


Abstract
========

This REP documents Simple Message message type identifiers that are
part of the official protocol specification. Both the standard set --
supported by the generic clients in the ``simple_message` package --
as well as the vendor specific messages are listed. It further
describes procedures for requesting assignment of new identifiers and
for keeping this document up-to-date.

Driver authors may treat this document as the normative reference for
assigned message type identifiers within the Simple Message protocol.


Motivation
==========

Similar to other binary protocols, the ROS-Industrial Simple Message
protocol [#simple_message]_ includes an identifier in its header
structure to allow senders and receivers to uniquely identify
message structure type. This identifier – stored in the ``msg_type``
field – is not only used to determine the correct (de)serialisation
sequence, but also encodes the intention – and indirectly the
semantics – of a message.

In order to be able to provide generic implementations of the Simple
Message protocol (de)serialisation libraries, and to avoid potential
compatibility issues between clients and servers, a central
registration of known message identifiers is essential.

This document provides the normative reference for assigned message
type identifiers, both for the standard set of messages and for the
special ranges (such as vendor specific ranges).


Application Procedure
=====================

Requests for assignment (RFA) of a new message identifier,
reassignment of an already assigned one or migration of a message type
from a special range to the standard set should be addressed to the
ROS-Industrial mailing list [#rosi_ml]_.

RFAs are expected to contain a short description of the message type,
its name, intended use and its merit. For a reassignment or migration,
a rationale should be provided as to why the current assignment should
be changed. For reasons of efficiency, multiple identifiers (or
ranges) may be requested using a single RFA. The relevant sections in
the RFA should reflect this.

After a three week review period a voting round will be held on the
mailing list. Both ROS-Industrial developers and users are eligible
to vote. In an ex aequo situation, ROS-Industrial developers may force
a decision.

Finally, the proposed changes are only final after the submission of
and incorporation of an update to this REP.


Identifier Allocation
=====================

Assignment of identifiers to (new) message types will be on a
first-come, first-served (FCFS) basis. If the author of an RFA has a
preference for a certain identifier (or range of identifiers), such a
request may be considered, provided sufficient rationale is given.

It is not possible for anyone to claim an identifier (or range of
identifiers) without sending an accompanying RFA -- containing all
required sections and information -- to the ROS-Industrial mailing
list. During review, any requested identifiers will have a pending
status. Allocation will be made final only on acceptance of the RFA
(and subsequent update of this document, as described in the
`Application Procedure`_ section).

Conflicts between RFAs (e.g. requests for the same identifier(s))
will be resolved by granting assignment of conflicting identifiers to
the RFA that was submitted for review first (chronologically), or
based on merit (to be determined by ROS-Industrial developers). In
all cases such decisions will be subject to voting as described in
`Application Procedure`_.


Assumptions
===========

#. All message type identifiers are specified in decimal (base-ten)
   notation.


Assigned Message Identifiers
============================

Standard Messages
-----------------

The following table lists all type identifiers for messages part of
the standardised Simple Message message set. In addition, special
and reserved ranges are indicated::


  ID                 Name                         Comment

                  0  -                            Reserved

                  1  PING                         -

                2-9  -                            Reserved for future use

                 10  JOINT_POSITION               Deprecated, also 'JOINT'
                 11  JOINT_TRAJ_PT                -
                 12  JOINT_TRAJ                   -
                 13  STATUS                       -
                 14  JOINT_TRAJ_PT_FULL           -
                 15  JOINT_FEEDBACK               -

              16-19  -                            Reserved for future use

                 20  READ_INPUT                   Deprecated
                 21  WRITE_OUTPUT                 Deprecated

             22-999  -                            Reserved for future use

          1000-1099  -                            Vendor specific

          1100-1999  -                            Reserved for future use

          2000-2099  -                            Vendor specific

         3000-65000  -                            Reserved for future use

        65001-65535  -                            Freely assignable

   65536-2147483647  -                            Reserved for future use


Note that [#simple_message]_ defines the ``msg_type`` field as a
signed 32 bit integer, but only positive values will be considered
valid identifiers in the context of this REP and the protocol's
implementation.

The IDs allocated to the *Vendor specific* range may be used by driver
authors to add messages that are too specialised to be included in the
generic industrial robot client. Note that the standard robot client
nodes will not be able to decode messages using these identifiers,
and driver authors are expected to provide an extended version of the
client able to decode messages with vendor specific message
identifiers.

All identifiers allocated to the *Freely assignable* range may be
freely used by users and allows for ID assignment within a limited
scope (ie: per project). These messages will also not be decodable
by the standard robot client nodes.


Vendor Specific Ranges
----------------------

The following table lists assigned vendor specific ranges::


  ID                 Vendor                       Comment

          1000-1099  SwRI                         -
          2000-2099  Motoman                      -


All vendor ranges have a length of 100 identifiers.

See the next sections for a listing of all assigned message
identifiers within these vendor specific ranges.


Vendor Specific Messages
------------------------

SwRI
^^^^

::

  ID        Name                                  Comment

 1000-1099  -                                     Reserved for future use


Motoman
^^^^^^^

::

  ID        Name                                  Comment

      2001  MOTOMAN_MOTION_CTRL                   -
      2002  MOTOMAN_MOTION_REPLY                  -

 2003-2015  -                                     Reserved for future use

      2016  ROS_MSG_MOTO_JOINT_TRAJ_PT_FULL_EX    -
      2017  ROS_MSG_MOTO_JOINT_FEEDBACK_EX        -

 2018-2099  -                                     Reserved for future use


References
==========

.. [#simple_message] ROS-Industrial simple_message package, ROS Wiki, on-line, retrieved 1 June 2014
   (http://wiki.ros.org/simple_message)
.. [#rosi_ml] ROS-Industrial mailing list (Google Group)
   (https://groups.google.com/forum/?fromgroups#!forum/swri-ros-pkg-dev)
.. [#msg_ping] PING, message definition, industrial_core Github repository, on-line
   (https://github.com/ros-industrial/industrial_core/blob/12a74a1f9f26aea0ee075edaf4c84473bd8e112a/simple_message/include/simple_message/ping_message.h#L49-L52)
.. [#msg_joint_pos] JOINT_POSITION, message definition, industrial_core Github repository, on-line
   (https://github.com/ros-industrial/industrial_core/blob/12a74a1f9f26aea0ee075edaf4c84473bd8e112a/simple_message/include/simple_message/messages/joint_message.h#L65-L83)
.. [#msg_joint_traj_pt] JOINT_TRAJ_PT, message definition, industrial_core Github repository, on-line
   (https://github.com/ros-industrial/industrial_core/blob/12a74a1f9f26aea0ee075edaf4c84473bd8e112a/simple_message/include/simple_message/joint_traj_pt.h#L61-L86)
.. [#msg_joint_traj] JOINT_TRAJ, message definition, industrial_core Github repository, on-line
   (https://github.com/ros-industrial/industrial_core/blob/12a74a1f9f26aea0ee075edaf4c84473bd8e112a/simple_message/include/simple_message/joint_traj.h#L54-L62)
.. [#msg_status] STATUS, message definition, industrial_core Github repository, on-line
   (https://github.com/ros-industrial/industrial_core/blob/12a74a1f9f26aea0ee075edaf4c84473bd8e112a/simple_message/include/simple_message/robot_status.h#L95-L114)
.. [#msg_joint_traj_pt_full] JOINT_TRAJ_PT_FULL, message definition, industrial_core Github repository, on-line
   (https://github.com/ros-industrial/industrial_core/blob/12a74a1f9f26aea0ee075edaf4c84473bd8e112a/simple_message/include/simple_message/joint_traj_pt_full.h#L70-L94)
.. [#msg_joint_feedback] JOINT_FEEDBACK, message definition, industrial_core Github repository, on-line
   (https://github.com/ros-industrial/industrial_core/blob/12a74a1f9f26aea0ee075edaf4c84473bd8e112a/simple_message/include/simple_message/joint_feedback.h#L61-L81)


Revision History
================

::

  2014-10-08  Updated Vendor specific sections with identifiers currently in use
  2014-06-01  Initial revision


Copyright
=========

This document has been placed in the public domain.
