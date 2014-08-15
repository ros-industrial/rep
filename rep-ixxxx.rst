::

  REP: XXX
  Title: Assigned Message Identifiers for the Simple Message Protocol
  Author: G.A. vd. Hoorn <g.a.vanderhoorn@tudelft.nl>
  Status: Draft
  Type: Standards Track
  Content-Type: text/x-rst
  Created: 01-Jun-2014
  Post-History: 15-Aug-2014


Outline
=======

#. Abstract_
#. Motivation_
#. `Application Procedure`_
#. Assumptions_
#. `Assigned Message Types`_

   #. `Standard Messages`_
   #. `Vendor Specific Ranges`_

#. References_
#. `Revision History`_
#. Copyright_


Abstract
========

This REP documents Simple Message message type identifiers that are
part of the official protocol specification, and are supported by the
generic clients in the ``simple_message`` package. It does not
describe the message structures themselves. Driver authors may treat
this document as the normative reference for assigned message type
identifiers.


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

Requests for assignment of a new message identifier, reassignment of
an already assigned one or migration of a message type from a special
range to the standard set should be addressed to the ROS-Industrial
mailing list.

Applications are expected to contain a short description of the
message type, its name, intended use and its merit. After a short
review period a voting round will be held on the mailing list.
Both ROS-Industrial developers and users are eligible to vote.
Acceptance of a new message type is then followed by an update of
this REP.


Assumptions
===========

#. All message type identifiers are specified in base-ten (decimal)
   notation.


Assigned Message Types
======================

Standard Messages
-----------------

The following table lists all type identifiers for messages part of
the standardised Simple Message message set. In addition, reserved
and special ranges are indicated::


  ID                 Name                  Comment

                  0  -                     Reserved

                  1  PING                  -

                2-9  -                     Reserved for future use

                 10  JOINT_POSITION        Deprecated, also 'JOINT'
                 11  JOINT_TRAJ_PT         -
                 12  JOINT_TRAJ            -
                 13  STATUS                -
                 14  JOINT_TRAJ_PT_FULL    -
                 15  JOINT_FEEDBACK        -

              16-19  -                     Reserved for future use

                 20  READ_INPUT            Deprecated
                 21  WRITE_OUTPUT          Deprecated

             22-999  -                     Reserved for future use

          1000-2999  -                     Vendor specific

    3000-2147483647  -                     Reserved for future use


Vendor Specific Ranges
----------------------

The following table lists vendor assigned specific ranges::


  ID                 Vendor                Comment

          1000-1999  SwRI                  -
          2000-2999  Motoman               -


References
==========

.. [#simple_message] ROS-Industrial simple_message package, ROS Wiki, on-line, retrieved 1 June 2014
   (http://wiki.ros.org/simple_message)


Revision History
================

::

  2014-06-01  Initial revision


Copyright
=========

This document has been placed in the public domain.
