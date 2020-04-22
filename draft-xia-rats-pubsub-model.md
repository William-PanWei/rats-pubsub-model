---
title: The YANG Subscription Model for Remote Attestation Procedures using TPMs
abbrev: RATS Subscription
docname: draft-xia-rats-pubsub-model-03
wg: RATS Working Group
stand_alone: true
ipr: trust200902
area: Security
kw: Internet-Draft
cat: std
pi:
  toc: 'yes'
  sortrefs: 'yes'
  symrefs: 'yes'

author:

- ins: L. Xia
  name: Liang Xia (Frank)
  org: Huawei Technologies
  abbrev: Huawei
  street: 101 Software Avenue, Yuhuatai District,
  city: Nanjing, Jiangsu
  region: ''
  code: '210012'
  country: China
  phone: ''
  email: frank.xialiang@huawei.com
- ins: W. Pan
  name: Wei Pan
  org: Huawei Technologies
  abbrev: Huawei
  street: 101 Software Avenue, Yuhuatai District
  city: Nanjing, Jiangsu
  region: ''
  code: '210012'
  country: China
  phone: ''
  email: william.panwei@huawei.com
- ins: E. Voit
  name: Eric Voit
  org: Cisco Systems, Inc.
  abbrev: Cisco
  email: evoit@cisco.com

normative:
  I-D.ietf-rats-architecture: rats-arch
  I-D.ietf-rats-yang-tpm-charra: rats-yang-tpm
  RFC8639:
  TPM2.0:
    author:
      org: TCG
    title: "TPM 2.0 Library Specification"
    target: https://trustedcomputinggroup.org/resource/tpm-library-specification/
    seriesinfo:

informative:
  I-D.birkholz-rats-tuda: rats-tuda
  I-D.richardson-rats-usecases: rats-usecases
  I-D.birkholz-rats-reference-interaction-model: rats-interaction
  KGV:
    author:
      org: TCG
    title: "KGV"
    date: 2003-10
    target: https://trustedcomputinggroup.org/wp-content/uploads/TCG-NetEq-Attestation-Workflow-Outline_v1r9b_pubrev.pdf
    seriesinfo:

--- abstract

This document defines a serial of related YANG modules for
the TPM-based network devices when using the YANG
subscription mechanism to do the remote attestation.

--- middle

# Introduction

Remote attestation is for acquiring the evidence about various
integrity information from remote endpoint to assess its
trustworthiness (aka, behaving in the expected manner). The evidence
should be about: system component identity, composition of system
components, roots of trust, system component integrity, system component
configuration, operational state and so on.
{{-rats-usecases}} describes possible use cases
which remote attestation are using for different industries, like:
network devices, FIDO authentication for online transaction,
Cryptographic Key Attestation for mobile devices, and so on.

{{-rats-arch}} lays a foundation of
RATS architecture about the key RATS roles (i.e., Relying Party,
Verifier, and Attester) and the messages they exchange, as well
as some key concepts. Based on it, {{-rats-interaction}} specifies a
basic challenge-response-based interaction model for the remote
attestation procedure, which a complete remote attestation procedure is
triggered by a challenge message originated from the Verifier, and
finished when the Attester conveys its response message back. This is a
very generic interaction model with wide adoption. This document
proposes an alternative interaction model for remote attestation
procedure, by customizing the YANG Subscription model {{RFC8639}}.
With the nature of asynchronous communication, the YANG Subscription model for
remote attestation procedure is optimal for large-scale and loosely
coupled distributed systems, especially for the network devices, which
has the advantages as: loose coupling, scalability, time delivery
sensitivity, supporting filtering capability, event-driven and so on. The YANG Subscription
model can be used independently, or together with the challenge-response
model to complement each other as a whole. Note that in which way these
models are combined together are out of the scope of this document.

In summary, by utilizing the YANG Subscription model in remote attestation
procedure, the gained benefits are as below:

* Flexibility: Verifier does not need to send the challenge
  message every time. The Verifier only needs to subscribe an
  event stream to the attester and then to anticipate the notification
  from the Attester about its integrity evidence.
  
* Efficiency: Once the Verifier has subscribed its interested
  event streams with customized filtering rules to the Attester, it
  will get all the matched notifications on time, which are
  the integrity related Evidence for remote attestation in fact. It
  will ensure any integrity change/deviation of the remote endpoint to
  be detected with the minimum latency.

* Scalability: This model will save a lot of challenge messages by
  replacing with single subscription message for one event stream, and
  decrease the total number of stateful connection between the
  Verifier and Attester, especially for a very large scale network.
  Thus, the scalability of the solution will increase.


# Terminology

This document uses terminology defined in {{-rats-arch}} and {{-rats-interaction}} for security related and RATS scoped terminology. This document also uses terminology defined in {{RFC8639}} for YANG Subscription mechanism related terminology.

## Requirements Notation

{::boilerplate bcp14}

# Operational Model

The following sequence diagram illustrates the reference remote attestation procedure by utilizing the YANG Subscription model.

~~~~
[Attester]                                            [Verifier]
    |                                                     |
    | <------------ Subscription(eventStream, nonce, ...) |
    |                                                     |
collectClaims(claimSelection)                             |
    | => claims                                           |
    |                                                     |
signEvidence(claims, nonce, ...)                          |
    | => signedEvidence                                   |
    |                                                     |
    | signedEvidence -----------------------------------> |
    |                                                     |
    |               appraiseEvidence(signedEvidence, refClaims)
    |                                attestationResult <= |
    |                                                     |
    |.....................................................|
    |                                                     |
Event Happens                                             |
    |                                                     |
   \|/                                                    |
collectClaims(claimSelection)                             |
    | => newClaims                                        |
    |                                                     |
signEvidence(claims, ...)                                 |
    | => signedNewEvidence                                |
    |                                                     |
    | signedNewEvidence --------------------------------> |
    |                                                     |
    |            appraiseEvidence(signedNewEvidence, refClaims)
    |                                attestationResult <= |
    |                                                     |
~~~~
{: #yangsubmodel title="YANG Subscription Model for Remote Attestation"}

In short, the basic idea of YANG Subscription model for remote attestation is
that the Verifier subscribes its interested event streams about the
Evidence to the Attester. The event streams can be in the 
YANG datastore, or not. After the subscription succeeds,
the Attester conveys the subscribed Evidence back to the
Verifier. When the selection filters are applied to the subscription, only the
information that pass the filter will be distributed out.

More detailed, the key steps of the remote attestation workflow
with this model can be summarized as below (using the network devices
as the example):

- The Verifier subscribes its interested event streams about the Evidence to the Attester. More specifically:
  - The event stream here refers to various integrity evidence
    information related to device trustworthiness, which can 
    be in YANG datastore, or not. The basic information included in
    event streams may be: software integrity information (including
    PCR values and system boot logs) of each layer of the trust
    chain recorded during device booting time; device identity
    certificates & Attestation Key certificate; operating
    system, application dynamic integrity information (e.g., IMA
    logs) and the device configuration information recorded during
    device running time.

  - The selection filters may be applied to the subscription,
    so that only the event records that pass the filter will be
    distributed out. Some specific examples include: event records
    of a component (e.g., line card) in the composite device, the
    event records in a specific time period that includes a start
    time and an end time, and so on.


- Consider how to send the existing parameters (i.e., nonce, hash
  signature algorithm, and specified TPM name, etc.) carried in the
  challenge message of the challenge-response model to the
  Attester through the subscription message of the new model
  in advance, and the follow-up usage of them. A very important
  point is how to ensure that the nonce carried in every
  notification message is different, and both the attester and the
  verifier know the correct value in advance.

- Both configuration subscription and dynamic subscription are
  considered. 


- The Attester notification delivery mechanisms thus vary as the
  above subscription mechanisms of verifier vary.

The following sections describes the above key steps.

# Remote Attestation Event Stream

The \<remote-attestation\> Event Stream is an {{RFC8639}} complaint Event Stream which is defined within this section and within the YANG Module of {{yangmodule}}. The Event Stream contains YANG notifications which carry Evidence which assists a Verifier in appraising the Trustworthiness Level of an Attester. Data Nodes allow the configuration of this Event Stream’s contents on a particular Attester.

This \<remote-attestation\> Event Stream may only be exposed on Attesters capable of signing cryptoprocessor PCRs using a private key unavailable elsewhere within the Attester. There is not a requirement that the underlying cryptoprocessor of the Attester has undergone TCG certification.

<To Be Modified: This document focuses on TPM-based network devices, so the Attester may need to meet TCG's requirements.>

## Remote Attestation Subscription Definition

### Dynamic Subscription

To establish the subscription in a way which results in provably fresh Evidence, randomness must be provided to the Attester. One way this can be done for an {{RFC8639}} dynamic subscriptions is via the augmentation of the \<establish-subscription\> RPC:

~~~~
augment /sn:establish-subscription/sn:input:
  +---w nonce-value?   binary
~~~~

As part of the response to the \<establish-subscription\>, a YANG notification defined in this document is returned. This notification MUST incorporate the randomness provided by the \<nonce-value\>. By including this YANG notification in the response, critical measurements are delivered in a way which provides protection against replay attacks. Additionally, the Verifier has immediate access to current measurements.

~~~~
augment /sn:establish-subscription/sn:output:
  +--ro latest-attestation
     +--(instance of <tpm12-attestation> or <tpm20-attestation> )
~~~~

<Freshness: First using nonce, subsequent notifications use timestamps.>

### Configured Subscription

It is also possible to subscribe to the \<remote-attestation\> Event Stream via an {{RFC8639}} configured subscription. In this case the Verifier needs some proof of Evidence freshness. Where a TPM2 exists, this may be accomplished via the creation and exposure of a Sync-Token as described in {{-rats-tuda}}. For any type of TPM, centrally created nonces could by signed, and broadcast to both the Attester and Verifier.

~~~~
+--rw attestation-config!
   +--rw tpm12-stream
   |  +--rw tpm12-stream-config* [tpm-name]
   |  |  +--rw tpm_name                   string
   |  |  +--rw tpm-physical-index?        int32 {ietfhw:entity-mib}?
   |  |  +--rw pcr-indices*               uint8
   |  |  +--rw TPM_SIG_SCHEME-value       uint8
   |  |  +--rw (key-identifier)?
   |  |  |  +--:(public-key)
   |  |  |  |  +--rw pub-key-id?          binary
   |  |  |  +--:(TSS_UUID)
   |  |  |     +--rw TSS_UUID-value
   |  |  |        +--rw ulTimeLow?        uint32
   |  |  |        +--rw usTimeMid?        uint16
   |  |  |        +--rw usTimeHigh?       uint16
   |  |  |        +--rw bClockSeqHigh?    uint8
   |  |  |        +--rw bClockSeqLow?     uint8
   |  |  |        +--rw rgbNode*          uint8
   |  |  +---w add-version?               boolean
   +--rw tpm20-stream
   |  +--rw tpm20-stream-config* [node-id tpm-name]
   |  |  +--rw node-id                    string
   |  |  +--rw node-physical-index?       int32 {ietfhw:entity-mib}?
   |  |  +--rw tpm_name                   string
   |  |  +--rw tpm-physical-index?        int32 {ietfhw:entity-mib}?
   |  |  +--rw pcr-list* []
   |  |  |  +--rw pcr
   |  |  |     +--rw pcr-indices*                uint8
   |  |  |     +--rw (algo-registry-type)
   |  |  |        +--:(tcg)
   |  |  |        |  +--rw tcg-hash-algo-id?     uint16
   |  |  |        +--:(ietf)
   |  |  |           +--rw ietf-ni-hash-algo-id? uint8
   |  |  +--rw (signature-identifier-type)
   |  |  |  +--:(TPM_ALG_ID)
   |  |  |  |  +--rw TPM_ALG_ID-value?    uint16
   |  |  |  +--:(COSE_Algorithm)
   |  |  |  +--rw COSE_Algorithm-value?   int32
   |  |  +--rw (key-identifier)?
   |  |     +--:(public-key)
   |  |     |  +--rw pub-key-id?          binary
   |  |     +--:(uuid)
   |  |        +--rw uuid-value?          binary
   |  +--rw tpm2-heartbeat?               uint8
   +--rw logs-stream
      +--rw logs-stream-config* [node-id tpm-name] 
      |  +--rw node-id                    string
      |  +--rw node-physical-index?       int32 {ietfhw:entity-mib}?
      |  +--rw tpm_name                   string
      |  +--rw tpm-physical-index?        int32 {ietfhw:entity-mib}?
      |  +--rw (index-type)?
      |  |  +--:(last-entry)
      |  |  |  +--rw last-entry-value?    binary
      |  |  +--:(index)
      |  |  |  +--rw last-index-number?   uint64
      |  |  +--:(timestamp)
      |  |     +--rw timestamp?           yang:date-and-time
      |  +--rw log-entry-quantity?        uint16
      |  +--rw pcr-list* []
      |     +--rw pcr
      |        +--rw pcr-indices*                  uint8
      |        +--rw (algo-registry-type)
      |           +--:(tcg)
      |           |  +--rw tcg-hash-algo-id?       uint16
      |           +--:(ietf)
      |              +--rw ietf-ni-hash-algo-id?   uint8
      +--rw log-type        identityref
~~~~



## YANG notifications placed on the Event Stream

### tpm-extend

This notification is generated every time a PCR is extended within a cryptoprocessor. The notification contains a list of the one or more strings which have extended a PCR.

~~~~
+---n tpm-extend
   +--ro tpm_name               string
   +--ro tpm-physical-index?    int32 {ietfhw:entity-mib}?
   +--ro pcr-index-changed      uint8
   +--ro extended-with*         binary
~~~~

All notifications since boot MUST be retained, and replayable.

<What's the purpose of this notification? There are already the TPM12 and TPM20 notification below.>

### tpm12-attestation

This notification contains an instance of a TPM1.2 style signed cryptoprocessor measurement. It is supplemented by Attester information which is not signed. This notification is generated and emitted from an Attester every time at least one PCR identified within the \<pcr-list\> has changed from the previous \<tpm12-attestation\> notification:

~~~~
+---n tpm12-attestation-response* [tpm-name]
   +--ro tpm-name                     string
   +--ro tpm-physical-index?          int32 {ietfhw:entity-mib}?
   +--ro up-time?                     uint32
   +--ro node-id?                     string
   +--ro node-physical-index?         int32 {ietfhw:entity-mib}?
   +--ro fixed?                       binary
   +--ro external-data?               binary
   +--ro signature-size?              uint32
   +--ro signature?                   binary
   +--ro (tpm12-quote)
      +--:(tpm12-quote1)
      |  +--ro version* []
      |  |  +--ro major?              uint8
      |  |  +--ro minor?              uint8
      |  |  +--ro revMajor?           uint8
      |  |  +--ro revMinor?           uint8
      |  +--ro digest-value?          binary
      |  +--ro TPM_PCR_COMPOSITE* []
      |     +--ro pcr-indices*        uint8
      |     +--ro value-size?         uint32
      |     +--ro tpm12-pcr-value*    binary
      +--:(tpm12-quote2)
         +--ro tag?                   uint8
         +--ro pcr-indices*           uint8
         +--ro locality-at-release?   uint8
         +--ro digest-at-release?     binary
~~~~

The vast majority of the YANG objects above are defined within {{-rats-yang-tpm}}. As a result, these objects are not redefined in this draft. The objects which are new include:

- pcr-index-changed* – this is a list of a PCRs which have new values since the last \<tpm12-attestation\> notification.
- pcr-index-attested* – this is a list of all the PCRs contained in the \<tpms-attest-result\>.

<What is the tpms-attest-result?>

<Above YANG module doesn't contain the two new objects yet.>

<The final PCR value depends on the sequence of the values this PCR extends with. So this sequence and the sources of each extended value should be recorded. For example, the PCR extended 3 files' hash values, so \{\{file1, hash1\}, \{file2, hash2\}, \{file3, hash3\}\} should be recorded. The Verifier first validate the hash value of each file, then calculate the final PCR value based on the original PCR value and these 3 files' hash values.>

Only the most recent \<tpm12-attestation\> is replayable. All others are discarded from the Event Stream history.

<Why only the most recent is replayable?>

Note that this notification alone does not fully handle replay attack protection for Centralized Trusted Path Routing. As a result, a Verifier MUST periodically receive a nonce based TPM1.2 style quote response. This can be done in several ways including via the \<tpm12-challenge-response-attestation\> RPC specified in {{-rats-yang-tpm}}. This periodic query allows a synching on the freshness of the results. Such a periodic synching is not required for the Distributed Trusted Path Routing architecture as the nonce based quote at time(y) proves the freshness of every passport.

<Verifier still needs to send challenge messages periodically, is there any other simpler way?>

### tpm20-attestation

This notification contains an instance of TPM2 style signed cryptoprocessor measurements. It is supplemented by Attester information which is not signed. This notification is generated at two points in time:

- every time at least one PCR has changed from a previous tpm20-attestation.
- after a locally configurable minimum heartbeat period since a previous tpm20-attestation was sent. This heartbeat is identifiable as the \<pcr-index-changed\> will be missing from the notification. As a result, there is no need to match it to one or more \<tpm-extend\> notifications.

Only the most recent \<tpm20-attestation\> is replayable. All others are discarded from the Event Stream history.

<Why only the most recent is replayable?>

Note that {{-rats-yang-tpm}} does not yet include the full set of {{TPM2.0}} objects. As soon as {{-rats-yang-tpm}} is updated with the necessary information, a new version of this draft will include a tree diagram which identifies those objects within this notification.

~~~~
+--ro tpm20-attestation-response* [node-id tpm-name]
   +--ro tpm-name                    string
   +--ro tpm-physical-index?         int32 {ietfhw:entity-mib}?
   +--ro up-time?                    uint32
   +--ro node-id                     string
   +--ro node-physical-index?        int32 {ietfhw:entity-mib}?
   +--ro quote?                      binary
   +--ro quote-signature?            binary
   +--ro pcr-bank-values* [algo-registry-type]
   |  +--ro (algo-registry-type)
   |  |  +--:(tcg)
   |  |  |  +--ro tcg-hash-algo-id?       uint16
   |  |  +--:(ietf)
   |  |     +--ro ietf-ni-hash-algo-id?   uint8
   |  +--ro pcr-values* [pcr-index]
   |     +--ro pcr-index                  uint16
   |     +--ro pcr-value?                 binary
   +--ro pcr-digest-algo-in-quote
      +--ro (algo-registry-type)
         +--:(tcg)
         |  +--ro tcg-hash-algo-id?       uint16
         +--:(ietf)
            +--ro ietf-ni-hash-algo-id?   uint8
~~~~

## Pre-filtering the Event Stream

It is possible for a receive just those PCR changes of interest from an Attester. To accomplish this, a RFC8639 \<establish-subscription\> RPC is made against the \<remote-attestation\> Event Stream. To limit the set of notifications, a \<stream-filter\> as per RFC8639, Section 2.2 can be set to select the following:

- each \<tpm-extend\> containing a \<pcr-index-changed\> of a desired PCR
- each \<tpm12-attestation\> containing a \<pcr-index-changed\> of a desired PCR
- each \<tpm20-attestation\> containing a \<pcr-index-changed\> of a desired PCR

## Replaying previous PCR Extend events

To verify the value of a PCR, a Verifier must either know that the value is a known good value {{KGV}} or be able to reconstruct the hash value by viewing all the PCR-Extends since the Attester rebooted. Wherever a hash reconstruction might be needed, the \<remote-attestation\> Event Stream MUST support the RFC8639 \<replay\> feature. Through the \<replay\> feature, it is possible for a Verifier to retrieve and sequentially hash all of the PCR extending events since an Attester booted. And thus, the Verifier has access to all the evidence needed to verify a PCR’s current value.


## Configuring the Attestation Event Stream

{{attestationconfig}} is tree diagram which exposes the configurable elements of the \<remote-attestation\> Event Stream. This allows an Attester to select what information should be available on the stream. A fetch operation also allows an external device such as a Verifier to understand the current configuration of stream.

The majority of the YANG objects below are defined via reference from {{-rats-yang-tpm}}.

~~~~
+--rw attestation-config!
   +--rw tpm12-stream
   |  +--rw tpm12-stream-config* [tpm-name]
   |  |  +--rw tpm_name                   string
   |  |  +--rw tpm-physical-index?        int32 {ietfhw:entity-mib}?
   |  |  +--rw pcr-indices*               uint8
   |  |  +--rw TPM_SIG_SCHEME-value       uint8
   |  |  +--rw (key-identifier)?
   |  |  |  +--:(public-key)
   |  |  |  |  +--rw pub-key-id?          binary
   |  |  |  +--:(TSS_UUID)
   |  |  |     +--rw TSS_UUID-value
   |  |  |        +--rw ulTimeLow?        uint32
   |  |  |        +--rw usTimeMid?        uint16
   |  |  |        +--rw usTimeHigh?       uint16
   |  |  |        +--rw bClockSeqHigh?    uint8
   |  |  |        +--rw bClockSeqLow?     uint8
   |  |  |        +--rw rgbNode*          uint8
   |  |  +---w add-version?               boolean
   +--rw tpm20-stream
      +--rw tpm20-stream-config* [node-id tpm-name]
      |  +--rw node-id                    string
      |  +--rw node-physical-index?       int32 {ietfhw:entity-mib}?
      |  +--rw tpm_name                   string
      |  +--rw tpm-physical-index?        int32 {ietfhw:entity-mib}?
      |  +--rw pcr-list* []
      |  |  +--rw pcr
      |  |     +--rw pcr-indices*                uint8
      |  |     +--rw (algo-registry-type)
      |  |        +--:(tcg)
      |  |        |  +--rw tcg-hash-algo-id?     uint16
      |  |        +--:(ietf)
      |  |           +--rw ietf-ni-hash-algo-id? uint8
      |  +--rw (signature-identifier-type)
      |  |  +--:(TPM_ALG_ID)
      |  |  |  +--rw TPM_ALG_ID-value?    uint16
      |  |  +--:(COSE_Algorithm)
      |  |  +--rw COSE_Algorithm-value?   int32
      |  +--rw (key-identifier)?
      |     +--:(public-key)
      |     |  +--rw pub-key-id?          binary
      |     +--:(uuid)
      |        +--rw uuid-value?          binary
      +--rw tpm2-heartbeat?               uint8
~~~~
{: #attestationconfig title="Configuring the Attestation Stream"}

There is one object which is new with this model however. \<tpm2-heartbeat\> defines the maximum amount of time which should pass before a subscriber to the Event Stream should get a \<tpm20-attestation\> notification from devices which contain a TPM2.

If there is no configuration of any \<tpm-name\> information within this model, all subscriptions should be rejected with an {{RFC8639}} reason of \<stream-unavailable\>.


# YANG Module {#yangmodule}

To be written.

# Security Considerations

To be written.

# IANA Considerations {#IANA}

To be written, possibly.

--- back

# Acknowledgements
{: numbered="no"}

Thanks to ...
