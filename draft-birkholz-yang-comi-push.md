---
title: Concise YANG Telemetry
abbrev: CoMI Push
docname: draft-birkholz-yang-comi-push-latest
date: 2018-04-18

ipr: trust200902
area: ops
wg: NETCONF Working Group
kw: Internet-Draft
cat: std

coding: us-ascii
pi:
   toc: yes
   sortrefs: yes
   symrefs: yes
   comments: yes

author:
- ins: H. Birkholz
  name: Henk Birkholz
  org: Fraunhofer SIT
  abbrev: Fraunhofer SIT
  email: henk.birkholz@sit.fraunhofer.de
  street: Rheinstrasse 75
  code: '64295'
  city: Darmstadt
  country: Germany
- ins: E. Voit
  name: Eric Voit
  org: Cisco Systems
  email: evoit@cisco.com
- ins: X. Liu
  name: Xufeng Liu
  org: Jabil
  abbrev: Jabil
  email: Xufeng_Liu@jabil.com
  street: 8281 Greensboro Drive, Suite 200
  region: McLean VA
  code: 22102
  country: USA
- ins: T. Zhou
  name: Tianran Zhou
  org: Huawei
  abbrev: Huawei
  email: zhoutianran@huawei.com
  street: 156 Beiqing Rd.
  region: Haidian District
  city: Beijing
  country: China

normative:
  RFC7252:
  RFC8040:
  RFC7049:
  RFC7950:
  RFC6241:
  RFC5277:
  RFC6020:
  I-D.ietf-core-yang-cbor: yangcbor
  I-D.ietf-core-comi: comi
  I-D.ietf-netconf-subscribed-notifications: yangnote
  I-D.ietf-netconf-yang-push: yangpush
  I-D.ietf-netconf-notification-messages: notemsgs
  I-D.bormann-t2trg-stp: series

informative:

--- abstract

This documents defines CoAP operations that implement the capabilities of YANG
Datastore Subscriptions and YANG Customized Subscriptions for the CoAP
Management Interface (CoMI). The '/s' resource, as defined in CoMI, is
extended analogously to include a set of sub-resources, each of them
representing an observable resource identified by its subscription-id. Specific
additions include but are not limited to new FETCH Body definitions and
simplified subtree subscriptions to intermediate data nodes in YANG datastore
modules using SID.

--- middle


# Introduction

The YANG management interface for constrained devices and networks, called CoAP
Management Interface (CoMI), is defined in {{-comi}} and covers the capabilities
as defined by YANG 1.1 {{RFC7950}}. The most essential characteristics of CoMI
are the use of:

* the Constrained Application Protocol (CoAP {{RFC7252}}) to facilitate an
  HTTP-esque interaction model,
* the Concise Binary Object Representation (CBOR {{RFC7049}}) to facilitate an
  efficient content encoding using {{-yangcbor}}, and
* YANG identifier strings are represented as numbers called YANG
  Schema Item Identifier (SID), also defined in {{-yangcbor}}.

This document defines additions to CoMI called Concise YANG Telemetry that:

* enrich its capabilities to subscribe to a variety of YANG modeled
  notifications---based on YANG Customized Subscriptions to a Publisher's Event
  Streams {{-yangnote}}, and
* enable subscriptions to changes data note values in modules provided by the
  YANG datastore (or parts of them)---based on YANG Datastore Subscription
  {{-yangpush}}.


# Terminology

Due to the utilization of CoAP, the interaction model of CoMI is quite similar
to RESTCONF {{RFC8040}}. RESTCONF supports subscriptions to a YANG
datastore via notification statements in YANG modules, which---when subscribed
to via the base subscription YANG RPC defined in {{RFC7950}}---result in
Series {{-series}} of Server Sent Events [W3C.REC-eventsource-20150203]}.
A corresponding Event Stream specification for NETCONF {{RFC6241}} Event
Notifications is defined in section 3.2.3 of {{RFC5277}}. To simplify
corresponding terminology (and especially consolidate the impedance mismatch of
the terms Notifications and Events), this document defines the following new term:

YANG Telemetry:

: A Series of YANG Notifications or YANG Notification Bundle Messages that are
composed of YANG modeled data, potentially composing update records, sent from a YANG
datastore to a YANG client either solicited or unsolicited in a fashion that can
guarantee well-defined levels of Visibility with respect to Data Node Value
changes.

Please note: while the focus of YANG is typically on management and operations, the scope
of YANG Telemetry extends into the Security Area with respect to Security
Events. Because of this, YANG Telemetry characteristics that address security requirements,
such as Visibility and Resilient YANG Subscriptions, are addressed in this document.

The definition of YANG Telemetry is based on the following existing terminology:

Series:

: Series Transfer Pattern are described in {{-series}} and describe the
conveyance of associated data items over time, where a client is able to obtain
the Series and to learn about new items.

: YANG Customized Subscriptions or a YANG Datastore Subscription creates an
specific Series Transfer Pattern composed of individual YANG Notifications or
YANG Notification Bundle Messages that include related updated records.

YANG Notification:

: Defined in {{RFC6241}}, a Notification is a server-initiated message indicating
that a certain event has been recognized by the server.

: While this definition is found in {{RFC6241}}, which is based on YANG 1.0
{{RFC6020}}, it also applies to the general definition of the YANG Notification
statement in YANG 1.1 {{RFC7950}}.

YANG Notification (Bundle) Message:

: An encapsulation header for one or more YANG notifications as defined in
{{-notemsgs}}. The message header includes a specific set of well-known objects,
which are of potential use to networking layers prior being interpreted by some
receiving application layer process.

: Examples of header object include, but are not limited to: timestamps,
signatures, or evidence about the integrity of the agents creating messages or
notifications.

Update Record:

: A single YANG data item in a Series of YANG Notifications or YANG Notification Bundle
Messages conveying the changes to a YANG datastore's module's Data Node Values.

YANG Data Item:

: An instance of YANG module Data Node Values or the changes to Data Node
Values, conveyed as data in motion, serialized in a specific representation, such
as XML, JSON, or CBOR.

Visibility:

: A level of assurance that Update Records will be received by a YANG Client.

: There might be reasons, such as resource exhaustion or dampening
settings, that result in Updates Records lost in transit or not being emitted by
the YANG datastore. Sequential Message-IDs or specific YANG Notifications
that report, e.g., about past events of resource exhaustion will inform the YANG
Client about the characteristics of the loss of Update Records.

Data Node Value:

: Defined in CoMI as the value assigned to a data node instance. Data node
values are serialized into the payload according to the rules defined in
section 4 of {{-yangcbor}}.

## CoAP Terminology

In addition to the illustration of the scope of YANG Telemetry above, this
section highlights the most important terms that are vital to the functionality
of Concise YANG Telemetry.

CoAP Requestor:

: The entity that emits CoAP requests to a CoAP Node with server capabilities.
In the context of Concise YANG Telemetry, a YANG client that is creating
dynamic subscriptions is a CoAP Requestor.

CoAP Token:

: A token used to match a CoAP response with a CoAP request. It is intended for
use as a client-local identifier for to differentiate between concurrent
requests (similar to a request ID).


# Summary of the Additions to CoMI

This documents defines the binding of YANG Datastore Subscriptions and YANG
Customized Subscriptions to the CoAP Management Interface. In summary, these
additions include:

* a CoAP POST operation to create, modify, delete or kill Telemetry subscription
  state,
* a CoAP iPATCH operation to create, modify, delete or kill modify one or more
  Telemetry subscription states,
* a CoAP GET operation including the Observe option to receive a Telemetry
  stream,
* a CoAP FETCH operation including the Observe option to receive multiple
  Telemetry streams,
* an extension of the "/s" resource that includes sub-resource, such as
  subscription identifiers and corresponding SID instances, and
* the capability to direct GET and FETCH operations including the Observe option
  at resources that are sub-resources of "/c".


## Telemetry-Specific CoMI datastores

The CoMI architecture assumes that both YANG client and YANG datastore (server)
retain or have access to knowledge about the same YANG specification (see Figure
1 in {{-comi}}). This is not necessarily true for a YANG Push capable CoMI
server. Highly constrained nodes can emit series of subscribed notifications
unsolicitedly; allowing them to create well-formed YANG-modeled Telemetry from
hard-coded building blocks of YANG-modeled data, which are in compliance to YANG
modules. In consequence, while taking on the role of a YANG datastore, a YANG
Push capable CoMI server SHOULD be capable to process YANG queries, but MAY not
be due to the lack of corresponding functions or knowledge of a complete YANG
module.

As these flavors of YANG datastores are not necessarily able to create CoMI
responses based on client request, it is likely that highly constrained
datastores initiate a Call-Home procedure (see [insert NETCONF Call Home here])
acting as if a request was already received (see Configured Subscription above),
enticing a very specific request they can fulfill (dynamic subscription) or
rendezvous via a discoverable YANG Zero Touch component. In all these use cases,
the datastore intends to create device specific Telemetry to be collected by
corresponding YANG clients.

In essence, incorporating a complete YANG module is not required to enable the
initiation of Concise YANG Telemetry within a very specific scope.


# Subscription Content and State

Two generic YANG notification statements for Update Records are introduced by
YANG Datastore Subscriptions augments to enable the following capabilities:

push-update:

: A notification that includes a complete (and potentially filtered) update of
data node values of YANG datastore nodes per the terms of a subscription.

push-change-update:

: A notification that includes an incremental (and potentially filtered) update
of data node values of YANG datastore nodes since the last (change-)update
notification.

Every Update Record Notification (Bundle) Message in a Series that is the context of a
subscription is emitted per the characteristics of the subscription state
maintained by the YANG datastore. Subscription state can be created on the YANG
datastore during manufacturing, onboarding, enrollment, deployment, or
maintenance of the YANG datastore. Most typically, subscription state is created
by a YANG Client (e.g. a Network Management System) via a dynamic subscription.


## Selection Filter

A vital part of the subscription state that defines the content of a Telemetry
stream is the filter expression included in the subscription characteristics. A
filter expression enables a YANG datastore to emit only a subset of potential
notification content; reducing the volume of data in motion, significantly.

Two generic YANG Filter Expressions enable a YANG datastore to emit filtered
subsets of data node value updates:

Subtree Filter Expression:

: A SID pointing to a specific data node in a YANG module (including
notification statements) is used to create update records that include updates
about the identified data node and its potential child nodes. Effectively, a
single SID points to the root node of the subtree update records are created for.

XPATH Filter Expression:

: A more detailed selection of SIDs and corresponding data
node values that update records are created for. The corresponding
representation of XPATH Filter Expressions for COMI is defined in {{-yangcbor}}.

Conditional SID Selectors:

: The SID concept introduced by CoMI and represented via YANG modeled CBOR allows
for a simplified Filter Expression that retains all the capabilities of an XPATH
Filter Expression, while using a significantly simpler representation.

# Subscription Characteristics

Distinct YANG Telemetry streams are defined by the following three primary
subscription characteristics:

1. Subscription Trigger (dynamic / configured)
2. Subscription Interval (periodic / on-change)
3. Subscription Type (stream / datastore)

These characteristics define how subscription state is deployed and how the
resulting Telemetry stream behaves. As an example, corresponding subscription
state can be created by a YANG client via the "establish-subscription" RPC as
defined in {{-yangnote}}.

## Subscription Trigger

In general, to establish a Telemetry stream via YANG Push there are three options:

1. a YANG client starts to receive a Telemetry stream from a YANG datastore,
   unsolicitedly. In this case, persistent subscription characteristics are
   created on the YANG data store before deployment, e.g. during onboarding, and
   come into effect directly after deployment.

2. a YANG client starts to receive a Telemetry stream from a YANG datastore,
   solicitedly. In this case, persistent subscription characteristics are created
   on both the YANG client and the YANG datastore before deployment.

2. a YANG datastore initiates contact with a YANG client via a Rendezvous, Join, or
   Call Home procedure and triggers the initiation of a telemetry stream. In
   this case, the subscription characteristics are configured on the YANG client
   and come into effect when the YANG datastore successfully discovered its
   home.

### CoMI Push Configured Subscriptions

CoAP defines a strict coupling of request and corresponding response messages
via the CoAP Token. Every CoAP Request MUST include a CoAP Token that is
generated by the requestor (client). Analogously, every CoAP response that is
associated with that request MUST include the corresponding CoAP Token in order
for the CoAP Request not to be discarded,

### Dynamic Subscriptions

## Subscription Interval

### Periodic Subscription

### On-Change Subscription

#### On-Change Subscription Prerequisites

An On-Change Subscription capability MUST be explicitly annotated in a YANG
module definition in order to prevent subscription to meaningless nodes. E.g. it
is advisable not to allow for subscriptions to YANG modules representing rapidly
changing counters.

The actual syntax and corresponding semantics of statements that intend to
annotate On-Change Subscription capabilities is out-of-scope of {{-yangpush}} or
{{-yangnote}} and in consequence also out-of-scope of this document.

#### Visibility
"publisher MUST set the "updates-not-sent" flag on any update record which
is known to be missing information."
* dampening-period MUST be set to 0 in order to ensure complete visibility
* non-zero may just indicate that there was a change, but not which one (e.g. interface-flapping)

## Subscription Type

The YANG Push subscription trigger mechanisms illustrated above create
subscription state between a YANG client and a YANG datastore. As long as this
subscription state between these two entities persists, a datastore can emit
series of YANG notifications to a YANG client, if appropriate conditions are
met, e.g. the YANG client expects solicited event notifications coming from the
datastore due to a dynamic subscription.

YANG Push {{-yangpush}} and YANG Subscribed Notifications {{-yangnote}} extend this
mechanism by enabling subscriptions to changes of YANG module data node
state in a YANG datastore resulting in two types of sources - or two
different types of YANG Notification Series [-cabo-series], respectively:

* event stream telemetry
* datastore changes telemetry

### Stream Subscription

### Datastore Subscription

# Subscription Management (better word?)

## YANG RPCs

## NETCONF Access Control Model [RFC6536bis]

# Selection Filters

## SID for subtree Selection Filter

## CBOR-YANG for XPATH-like Selection Filter

# Update Triggers for Periodic Subscriptions

## Interval

## Anchor Time

# Update Triggers for On-Change Subscriptions

## dampening period
* requires bundled messages, in order to maintain visibility
* applies to update record creation, not transmission

## change-type
* create, delete, change (where is the complete list?)

## no sync-on-start
* full set at start of sub is omitted

# YANG Push Operations for COMI

Every subscription-id is created by the YANG datastore and is used in the corresponding subscription state to provide the root identifier, by which dedicated subscription characteristics are associated with an established subscription. In consequence, the basic interaction model of Concise YANG Push is split into two operations that are initiated by the YANG client in sequence:

* a POST operation on /c executing the establish-subscription RPC corresponding to the included request body with content-type "application/yang-value+cbor" that returns the subscription-id (or an error response) in an "application/yang-value+cbor" response
* a GET Observe operation on the event stream resource /s/subscription-id or a FETCH operation on /s including a FETCH body with content-format "application/yang-selectors+cbor" and one or more subscription-id as content.

## Extension of the CoMI Event Stream Resource

A standard CoMI datastore as defined in [I-D.ietf-core-comi] typically uses the datastore resource “/c” to provide the YANG datastore tree and the resource “/s” to provide the YANG notification stream. Sub-resources under “/c” are represented in the format of /c/sid.

CoMI Push extends the scope of the “/s” resource. Sub-resources under “/s” are represented as /s/key, where key is a numeric string representation of the subscription identifier, e.g. “/s/65536/”. The key representation reduces the ambiguity with respect to sid, which uses an URI safe base64 representation.

Under each subscription identifier key provided as a sub-resource of the “/s” resource, a YANG tree instance of the subscription characteristics yang:ietf-subscribed-notifications/subscriptions (as defined in YANG Push [I-D.ietf-netconf-yang-push], which augments ietf-subscribed-notification defined in [I-D.ietf-netconf-subscribed-notifications]) is provided. In the following example every sid is illustrated as a “string” as there is no sid for corresponding instance identifiers yet:

/s/”65536”/”stream”/”within-subscription”/”filter-spec”/”stream-xpath-filter”/”stream-xpath-filter“

This example would point to the filter selection associated with subscription id 65536.

## Extension of the YANG Subscription Mechanism

YANG Push provides augmented RPC for establishing, modifying, deleting, or killing a subscription. CoMI uses the same module as YANG Push and provides a corresponding interface to allow for a corresponding confirmable POST message to RPC resources (see [I-D.ietf-core-comi] Section 5.3.2.).

CoMI Push also defines the capabilites to point confirmable FETCH messeges – including the Observe option - to sub-resources provided by “/c”. If the body of the FETCH message includes a CBOR modeled [I-D.ietf-core-yang-cbor] subtree filter expression, a new subscription is created and a corresponding subscription id is returned. Additionally, a corresponding subscription sub-resource under “/s” is created.

# Upcoming Features and Stories

* maybe we should introduce /sn and /snmb for stream subscriptions and
* SIDs and subscription-id have to be both "the same" and "distinguishable" from sid
  * maybe we have to create a new content-format application/yang-subest+cbor
    and application/yang-subres+cbor?
* my resource goes away during subscription (e.g. FRU pulled)
* back channel liveness check - important for udp and sudden outage in telemetry flows

#  IANA considerations

This document includes no requests to IANA, but solutions drafts incubated via
this document might.

#  Security Considerations

This document includes no security considerations, but solution drafts incubated
via this document will.

#  Acknowledgements

Carsten Bormann, Klaus Hartke, Michel Veillette

#  Change Log

First version -00

--- back