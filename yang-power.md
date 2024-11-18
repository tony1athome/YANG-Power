---
coding: utf-8
title: A YANG model for Power Management
abbrev: YANG Power Management
docname: draft-li-green-power-00
wg: IVY Working Group
category: std
submissiontype: IETF
ipr: trust200902

author:
  -
       ins: T. Li
       name: Tony Li
       org: Juniper Networks
       email: tony.li@tony.li
  -
       ins: R. Bonica
       name: Ron Bonica
       org: Juniper Networks
       email: rbonica@juniper.net

--- abstract

Network sustainability is a key issue facing the industry. Networks
consume significant amounts of power at a time when the cost of power
is rising and sensitivity about sustainability is very high. As an
industry, we need to find ways to optimize the power efficiency of our
networks both at a micro and macro level. We have observed that
traffic levels fluctuate and when traffic ebbs there is much more
capacity than is needed. Powering off portions of network elements
could save a significant amount of power, but to scale and be
practical, this must be automated.

The natural mechanism for enabling automation would be a Yet Another
Next Generation (YANG) interface, so this document proposes a YANG
model for power management.

--- middle

# Introduction

Network sustainability is a key issue facing the industry. Networks
consume significant amounts of power at a time when the cost of power
is rising and sensitivity about sustainability is very high. As an
industry, we need to find ways to optimize the power efficiency of our
networks both at a micro and macro level. We have observed that
traffic levels fluctuate and when traffic ebbs there is much more
capacity than is needed. Powering off portions of network elements
could save a significant amount of power, but to scale and be
practical, this must be automated.

The natural mechanism for enabling automation would be a Yet Another
Next Generation (YANG) interface, so this document proposes a YANG
model for power management.

{{!RFC8348}} already provides a model for server hardware management,
but does not naturally extend to routers and other network
elements. That gap is currently being addressed by
{{!I-D.ietf-ivy-network-inventory-yang}}.  This document extends the
work presented there to include power management. Specifically, this
document augments the 'component' object found at
/ietf-network-inventory/network-elements/network-element/components/component.

This initial draft only provides a tree representation. When there is
rough consensus on the tree represetnation, the details of the model
will be fleshed out.

## Requirement Language {#REQ-lang}

{::boilerplate bcp14+}

# Power Management Elements

The models mentioned above already model a router or network
element as a set of components. The details of those components are
left to the specific implementation and can be at any level of
specificity. Thanks to this flexibility, it is necessary and
sufficient that we characterize power management relative to
components.
 
The elements defined below allow management entities to understand how
much power each component is using and whether the component can be
placed into a 'power-save' mode where it would consume less
power. Another element allows the management plane to put the
component into power-save mode.

## Power consumption

~~~
        Name: used-power
        Node Type: leaf
        Data Type: uint32
        Description: Power drawn by the component, in watts.
~~~

This node is applied to components in the model. If an accurate
dynamic power measurement is not available, then static power
estimates are acceptable.

~~~
        Name: total-used-power
        Node Type: leaf
        Data Type: uint32
        Description: Power drawn by the component and its
          subcomponents, in watts.
~~~

This node captures the total power used by a component and all of its
subcomponents. This simplifies reporting requirements. If an accurate
dynamic power measurement is not available, then static power
estimates are acceptable.



## Power control capability

~~~
        Name: power-save-capable
        Node Type: leaf
        Data Type: boolean
        Description: True if the component can be put into power-save
          mode.
~~~

## Power control

~~~
        Name: power-save
        Node Type: leaf
        Data Type: Boolean
        Access: Read/write
        Description: True if the component is in power-save mode.
~~~

There have been suggestions that this leaf should accept a variety of
values, representing many different 'power states'. This presumes that
the external entity understands the exact state of the device, the
impact of the power state, and the optimal setting. A better approach
is to enable automatic power saving on the component and then provide
information to the device about the external state to allow the device
to optimize its own behavior. {{Traffic}} is an example of the type of
external state information that may be useful.

## Automatic Power Saving

Some systems (e.g., fan trays) have the capability to self-manage
their power consumption. However there are cases where this capability
should be disabled.

~~~
        Name: automatic-power-saving
        Node Type: leaf
        Data Type: Boolean
        Access: Read/write
        Description: True if the component is regulating its own
          power.
~~~

# Functional Dependencies

Most inventory models have a hierarchy of components.  This 
hierarchy reflects the physical structure of the system (e.g., 
a line card can physically contain a port).

With regard to physical containment, components maintain a one-to-many
relationship. That is, Component A can contain many other components, including
Component B. However, component B can be contain by only one component 
(i.e., Component A.)

However, legacy inventory models do not reflect functional
dependencies.  Specifically, they do not indicate which components
obtain services from, and therefore depend, components other than
their container. Because funtional dependencies are relavant to power
management, they are included in the proposed model.

With regard to functional dependencies, components maintain a many-to-many
relationship. That is, a component can reuire on many components and be
required by many other components. 

Functional dependencies may be updated dynamically.

## Required Components

This container holds a list of components that the component uses.
For example, a linecard uses a set of switch cards, so the switch
cards would be required components. If the bandwidth used by the
linecard changes, then the set of switch cards that are required may
change dynamically.

~~~
        Name: required-components
        Node Type: list
        Description: A list of other components that are required for
          this component to operate.
~~~

## Dependent components

This container holds a list of components that are used by this
component. For example, a switch card is used by a set of line cards,
so the line cards would be dependent components. This list can also
change dynamically.

~~~
        Name: dependent-components
        Node Type: list
        Description: A list of other components that are used by this
          component.
~~~

# Tree Representation

~~~
 +--rw component* [component-id]
    +--rw component-id               string
    +--ro used-power?                uint32
    +--ro power-save-capable?        boolean
    +--rw power-save?                boolean
    +--ro required-components*       -> ../../component/component-id
    +--ro dependent-components*      -> ../../component/component-id
~~~

# Traffic Planning {#Traffic}

Some systems have the capability of automatically managing their power
consumption if they have an understanding of the expected traffic
loads. To provide this, we provide the expected input and output
bandwidth for each interface and augment '/interfaces/interface'
{{!RFC8343}} with the following:

~~~
        Name: expected-input-bandwidth
        Node Type: leaf
        Data Type: yang:gauge64
        Default: 0
        Access: Read/write
        Description: The expected input bandwidth of the interface,
          in Mbps. A value of zero (0) indicates full bandwidth.
~~~

~~~
        Name: expected-output-bandwidth
        Node Type: leaf
        Data Type: yang:gauge64
        Default: 0
        Access: Read/write
        Description: The expected output bandwidth of the interface,
          in Mbps. A value of zero (0) indicates full bandwidth.
~~~

## Tree Representation

~~~
 +--rw interface* [name]
    +--rw expected-input-bandwidth?     yang:gauge64
    +--rw expected-output-bandwidth?    yang:gauge64
~~~

# Security Considerations

YANG provides information about and configuration capabilities to the
network management plane. Other mechanisms already exist that help
secure these interactions. This document extends the scope of what can
be controlled by the management plane, but creates no new access paths.

# IANA Considerations

This document makes no requests for IANA.
