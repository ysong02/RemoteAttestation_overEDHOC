# Introduction

# Conventions and Definitions

# Remote Attestation Aspects in EDHOC {#attestation-aspects}

This section outlines three independent aspects that characterize the remote attestation process over EDHOC:

## Attestation Targets

### IoT Device Attestation (IoT)

This refers to the process of ensuring the integrity and trustworthiness of IoT devices before they can interact with the cloud or network.
Attestation of IoT devices verifies that the devices have not been tampered with and are oprating correctly.
The IoT device acts as the Attester.
We assume in this specification that the IoT devices are low-power constrained devices that have computation and connection constraints.

### Network Service Attestation (Net)

This involves verifying the trustworthiness of network services or cloud entities.
It ensures that the services the IoT devices are communication with are legitimate and secure.
For example, a network service attestation should be done before a IoT device shares some sensitive data with the network service.
The network service acts as the Attester.
Unlike IoT devices, network services typically have more computational power and capabilities, enabling them to handle complex attestation processes when more work for the Attester is required.

## Attestation Models

This specification supports both the RATS background-check model {{bg}} and passport model {{pp}}.
The corresponding EAD items for background-check model and the passport model are independent between each other.
The detailed EAT items definitions are in Section {{ead-bg}} for background-check model and Section {{ead-pp} for passport model.}

### Background-check Model (BG) {#bg}

In background-check model, the Attester sends the evidence to the Relying Party.
The Relying Party transfers the evidence to the Verifier and gets back the attestation result from the Verifier.

EDHOC session is between the Attester and the Relying Party.
A negotiation of the evidence type is required Before the Attester sends the evidence.
All three parties need to be aligned with one selected evidence type.
The Attester first sends a list of the provided evidence types formatted to a Attestation proposal as an EDHOC EAD item to the Relying Party,  .
The Relying Party transfers the list to the Verifier and should receive at least one supported evidence type from the Verifier.
If the Relying Party receives more than one evidence type, a single evidence type should be selected by the Relying Party and sends it back as an Attestation request to the Attester.

We use nonce (at least 8 bytes according to {{I-D.ietf-rats-eat}}) to guarantee the freshness for the attestation session, which is generated by the Verifier and sends to the Relying Party. 
The Relying Party puts nonce and the selected evidence type together in a tuple to generate the EAD item Attestation request.

If the Attestation checks that the received evidence type from the Relying Party is within the list of its provided evidence types, it can call its attestation service to generate the evidence, with nonce value as one of the inputs.  

#### External Authorization Data (EAD) Items for background-check model {#ead-bg}

EAD items that are used for the background-check model are defined in this section.

##### Attestation_proposal {#attestation-proposal}

To propose a list of provided evidence types in background-check model, the Attester transports the Proposed_EvidenceType object.
It signals to the Relying Party the proposal to do remote attestation, as well as which types of the attestation claims the Attester supports.
The Proposed_EvidenceType is encoded in CBOR in the form of a sequence.

The EAD item for an attestation proposal is:

* ead_label = TBD1
* ead_value = Attestation_proposal, which is a CBOR byte string:

~~~~~~~~~~~~~~~~
Attestation_proposal = bstr .cbor Proposed_EvidenceType

Proposed_EvidenceType = [ + content-format]

content-format = uint
~~~~~~~~~~~~~~~~

where

* Proposed_EvidenceType is an array that contains all the supported evidence types by the Attester.
* There MUST be at least one item in the array.
* content-format is an indicator of the format type (e.g., application/eat+cwt with an appropriate eat_profile parameter set), from {{IANA-CoAP-Content-Formats}}.

The sign of ead_label TBD1 MUST be negative to indicate that the EAT item is critical.
If the receiver cannot recognize the critical EAD item, or cannot process the information in the critical EAD item, then the receiver MUST send an EDHOC error message back.

##### Attestation_request {#attestation-request}

As a response to the attestation proposal, the Relying Party signals to the Attester the supported and requested evidence type.
In case none of the evidence types is supported, the Relying Party rejects the first message_1 with an error indicating support for another evidence type.

The EAD item for an attestation request is:

* ead_label = TBD1
* ead_value = Attestation_request, which is a CBOR byte string:

~~~~~~~~~~~~~~~~
Attestation_request = bstr .cbor Selected_EvidenceType
Selected_EvidenceType = (
  content-format: uint,
  nonce: bstr .size 8..64
)
~~~~~~~~~~~~~~~~

where

* content-format is the selected evidence type by the Relying Party and supported by the Verifier.
* nonce is generated by the Verifier and forwarded by the Relying Party.

The sign of ead_label TBD2 MUST be negative to indicate that the EAT item is critical.
If the receiver cannot recognize the critical EAD item, or cannot process the information in the critical EAD item, then the receiver MUST send an EDHOC error message back.

##### Evidence {#evidence}

As a response to the attestation request, the Attester calls its local attestation service to generate and return the serialized EAT {{I-D.ietf-rats-eat}} as Evidence.

The EAD item is:

* ead_label = TBD1
* ead_value is a serialized EAT.

More details about EAT are in {{I-D.ietf-rats-eat}}.

##### trigger_bg {#trigger-bg}

The EAD item trigger_bg is used when the sender triggers the receiver to start a remote attestation in background-check model.
The receiver MUST reply with an EAD item for background-check model.
The ead_value can be empty, using ead_label plays the role of the trigger.

The EAD item is: 

* ead_label = TBD2
* ead_value = null

### Passport Model (PP) {#pp}

In passport model, the Attester sends the evidence to the Verifier.
After the Attester receives the attestation result from the Verifier, the Attester sends the attestation result to the Relying Party.

EDHOC session is between the Attester and the Relying Party.
The Attester and the Relying Party should decide from which Verifier the Attester obtains the attestation result and transfers it to the Relying Party.
The Attester first sends a list of the Verifier identities that it can get the attestation result.
The Relying Party selects one trusted Verifier identity and sends it back as a Result request.

Regarding the freshness in passport model, the Attester could choose to either have a real-time connection with the selected Verifier, or benefit by a pre-stored attestation result from the selected Verifier.
If the attestation result is not from a real-time connection session, the attestation result should have a time stamp and/or expiry time to indicate the validity of the attestation result.
The synchronization of the time is out of scope in this specification, and depends on the different device policies.

Once the Attester obtains the attestation result from the selected Verifier, it sends the attestation result to the Relying Party.

#### External Authorization Data (EAD) Items for passport model {#ead-pp}

EAD items that are used for the passport model are defined in this section.

##### Result_proposal {#result-proposal}

An attestation result proposal contains the identification of the credentials of the Verifiers to indicate Verifiers' indentities.
The identification of credentials relies on COSE header parameters {{IANA-COSE-Header-Parameters}}, with a header lable and credential value.

The EAD item for an attestation result proposal is:

* ead_label = TBD3
* ead_value = Result_proposal, which is a CBOR byte string:

~~~~~~~~~~~~~~~~
Result_proposal = bstr .cbor Proposed_VerfierIdentity
Proposed_VerifierIdentity = [ + VerifierIdentity ]

VerifierIdentity = {
  label => values
}
~~~~~~~~~~~~~~~~

where

* Proposed_VerifierIdentity is defined as a list of one or more VerifierIdentity elements.
* Each VerifierIdentity within the list is a map defined in {{IANA-COSE-Header-Parameters}} that:
    * label = int / tstr
    * values = any


##### Result_request {#result-request}

As a response to the attestation result proposal, the Relying Party signals to the Attester the trusted Verifier.
In case none of the Verifiers can be trusted by the Relying Party, the session is aborted.
Relying Party generates a nonce to ensure the freshness of the attestation result from the Verifier.

The EAD item for an attestation result request is:

* ead_label = TBD3
* ead_value = Result_request, which is a CBOR byte string:

~~~~~~~~~~~~~~~~
Result_request = bstr .cbor Request_structure

Request_structure = {
  selected_verifier: VerfierIdentity
}
~~~~~~~~~~~~~~~~

##### Result {#result}

The attestation result is generated and signed by the Verifier as a serialized EAT {{I-D.ietf-rats-eat}}.
The Relying Party can decide what action to take with regards to the Attester based on the information elements in attetation result.

The EAD item is:

* ead_label = TBD3
* ead_value is a serialized EAT.

##### trigger_pp {#trigger-pp}

The EAD item trigger_pp is used when the sender triggers the receiver to start a remote attestation in passport model.
The receiver MUST reply with an EAD item for passport model.
The ead_value can be empty, using ead_label plays the role of the trigger.

The EAD item is: 

* ead_label = TBD4
* ead_value = null

## EDHOC Message Flow

EDHOC protocol defines two types of athentication message flows.
In this specification, we support both the EDHOC remote message flow and the EDHOC reverse message flow (see {{Appendix A.2.2 of RFC9528}}) to perform remote attestation.
No matter which type of the message flow is used, the EDHOC Initiator always sends the first EDHOC message_1.

### EDHOC Forward Message Flow (Fwd)

In forward message flow, EDHOC may run with the Initiator as a CoAP/HTTP client.
Remote attestation over EDHOC starts with a POST requests to the Uri-Path: "/.well-known/lake-ra". 

### EDHOC Reverse Message Flow (Rev)

In the reverse message flow, the CoAP/HTTP client is the Responder and the server is the Initiator.
A new EDHOC session begins with an empty POST request to the server's resource for EDHOC.

# Remote Attestation Aspect Combinations {#attestation-combinations}

We use the format (Target, Model, Message Flow) to denote combinations.
For example, (IoT, BG, Fwd) represents IoT device attestation using the background-check model with forward EDHOC message flow.

Among the three remote attestation aspects, each aspect is independent from one another.
All 8 cases (IoT/Net, BG/PP, Fwd/Rev) are possible.
In this specification, we list the most relevant combinations for constrained IoT environments.

## (IoT, BG, Fwd): IoT Device Attestation

A common use case for (IoT, BG, Fwd) is to attest an IoT device to a network server.
For example, doing remote attestation to verify that the latest version of firmware is running on the IoT device before the network server allows it to join the network (see {{firmware}}).

An overview of doing IoT device attestation in background-check model and EDHOC forward message flow is established in {{fig-iot-bg-fwd}}.
EDHOC Initiator plays the role of the RATS Attester (A).
EDHOC Responder plays the role of the RATS Relying Party (RP).
The Attester and the Relying Party communicate by transporting messages within EDHOC's External Authorization Data (EAD) fields.
An external entity, out of scope of this specification, plays the role of the RATS Verifier (V).
The EAD items specific to the background-check model are defined in {{ead-bg}}.

The Attester starts the attestation by sending an Attestation proposal in EDHOC message_1.
The Relying Party generates EAD_2 with the received evidence type and nonce from the Verifier, and sends it to the Attester.
The Attester sends the Evidence to the Relying Party in EAD_3.
The Verifier evaluates the Evidence and sends the Attestation result to the Relying Party.  

~~~~~~~~~~~ aasvg
+-----------+              +-----------+                 
|IoT device |              |Net service|
+-----------+              +-----------+               
|  EDHOC    |              |   EDHOC   |
| Initiator |              | Responder |
+-----------+              +-----------+               +----------+
| Attester  |              |     RP    |               | Verifier |
+-----+-----+              +-----+-----+               +-----+----+
      |                          |                           | 
      |  Attestation proposal    |                           |
      +------------------------->|                           |
      |                          | Attestation proposal, C_R | 
      |                          +-------------------------->|
      |                          |<--------------------------+
      |                          |  EvidenceType(s), Nonce   |                     
      |<-------------------------+                           |                 
      |   Attestation request    |                           |                  
      |                          |                           |
      |        Evidence          |                           |
      |------------------------->|        Evidence, C_R      |
      |                          +-------------------------->|
      |                          |<--------------------------|
      |                          |    Attestation result     |
      |                          |                           | 
      |<---  ---  ---  ---  --- >|                           |
              EDHOC session

~~~~~~~~~~~
{: #fig-iot-bg-fwd title="Overview of IoT device attestation in background-check model and EDHOC forward message flow. EDHOC is used between A and RP." artwork-align="center"}

## (Net, PP, Fwd): Network Service Attestation

One use case for (Net, PP, Fwd) is when a network server needs to attest itself to a client (e.g., an IoT device).
For example, the client needs to send some sensitive data to the network server, which requires the network server to be attested first.
In (Net, PP, Fwd), the network server acts as an Attester and the client acts as a Relying Party.

An overview of the message flow is illustrated in {{fig-net-pp-fwd}}.
EDHOC Initiator plays the role of the RATS Relying Party.
EDHOC Responder plays the role of the RATS Attester.
The Attester and the Relying Party communicate by transporting messages within EDHOC's External Authorization Data (EAD) fields.
An external entity, out of scope of this specification, plays the role of the RATS Verifier.
The EAD items specific to the passport model are defined in {{ead-pp}}.

The Relying Party asks the Attester to do a remote attestation by sending a trigger_PP in EDHOC message_1.
The Attester replies to the Relying Party with a Result proposal in EAD_2.
Then the Relying Party selects a trusted Verifier identity and send it as a Result request.
How the Attester negotiates with the selected Verifier to get the attestation result is not detailed in this specification.
A fourth EDHOC message is required to send the Result from the Attester to the Relying Party.


~~~~~~~~~~~ aasvg
+-----------+              +-----------+                 
|IoT device |              |Net service|
+-----------+              +-----------+               
|  EDHOC    |              |   EDHOC   |
| Initiator |              | Responder |
+-----------+              +-----------+               +----------+
|     RP    |              | Attester  |               | Verifier |
+-----+-----+              +-----+-----+               +-----+----+
      |                          |                           | 
      |      trigger_PP          |                           |
      +------------------------->|                           |
      |                          |                           | 
      |<-------------------------+                           |
      |      Result proposal     |                           |
      |                          |                           |
      |      Result request      |                           |
      +------------------------->|        (request)          |
      |                          +---  ---  ---  ---  ---  ->|
      |                          |<---  ---  ---  ---  ---  -+
      |                          |         Result            |
      |          Result          |                           |
      |<-------------------------+                           |
      |                          |                           |
      |                          |                           |
      |<---  ---  ---  ---  --- >|                           |
              EDHOC session 
~~~~~~~~~~~
{: #fig-net-pp-fwd title="Overview of network service attestation in passport model and EDHOC forward message flow. EDHOC is used between RP and A." artwork-align="center"}

# Mutual Attestation in EDHOC {#mutual-attestation}

Mutual attestation over EDHOC combines the cases where one of the EDHOC party uses the IoT device attestation and the other uses the Network service attestation.
Performing mutual attestation to a single EDHOC message flow acheives a lightweight use with reduced message overhead. 
Note that the type of message flow for the two parties in mutual attestation MUST be the same. 
Otherwise, the positions of the EDHOC Initiator and EDHOC Responder cannot match, which result in both the IoT device and the Network service being the EDHOC Initiator and this is impossible. 

In this specification, we list the most relevant mutual attestation example for constrained IoT environments.

## (IoT, BG, Fwd) - (Net, PP, Fwd)

In this example, the mutual attestation is performed in EDHOC forward message flow, by one IoT device attestation in background-check model and another network service attestation in passport model.
The process is illustrated in Figure {{#fig-mutual-attestation-BP}}.
How the Network service connects with the Verifier_1 and protential Verifier_2 is out of scope in this specification.
The first remote attestation is initiated by the IoT device (A_1) in background-check model.
Meanwhile, the IoT device (Attester 1) asks the network service (A_2) to perform a remote attestation in passport model.
EAD_2 has the EAD items Attestation request and Result proposal.
EAD_3 carries the EAD items Evidence and Result request.
EAD_4 has the EAD item Result for the passport model.

~~~~~~~~~~~ aasvg
+-----------+              +-----------+                 
|IoT device |              |Net service|
+-----------+              +-----------+               
|  EDHOC    |              |   EDHOC   |
| Initiator |              | Responder |
+-----------+              +-----------+               +------------+              +------------+                 
| A_1/RP_2  |              | RP_1/A_2  |               | Verifier_1 |              | Verifier_2 |
+-----+-----+              +-----+-----+               +------+-----+              +------+-----+                   
      |                          |                            |                           |
      |  Attestation proposal    |                            |                           |
      +------------------------->|                            |                           |
      |       trigger_PP         |                            |                           |
      |                          |                            |                           |
      |                          |  Attestation proposal, C_R |                           |
      |                          +--------------------------->|                           |
      |                          |<---------------------------+                           |
      |                          |   EvidenceType(s), Nonce   |                           |
      |   Attestation request    |                            |                           |
      |<-------------------------+                            |                           |
      |     Result proposal      |                            |                           |
      |                          |                            |                           |
      |                          |                            |                           |
      |        Evidence          |                            |                           |
      +------------------------->|                            |                           |
      |      Result request      |                            |                           |
      |                          |       Evidence, C_R        |                           |
      |                          +--------------------------->|         (Request)         |
      |                          +--- --- --- --- --- --- --- +---  ---  ---  ---  ---  ->|              
      |                          |                            |                           |
      |                          |    Attestation result      |                           |
      |                          |<---------------------------+                           |
      |                          |                            |                           |
      |                          |<---  ---  ---  ---  ---  --+ ---  ---  ---  ---  ---  -+
      |                          |            Result          |                           |
      |                          |                            |                           |
      |<-------------------------+                            |                           |
      |          Result          |                            |                           |
      |                          |                            |                           |
      |<---  ---  ---  ---  --- >|                            |                           |
              EDHOC session 

~~~~~~~~~~~
{: #fig-mutual-attestation-BP title="Overview of mutual attestation of (IoT, BG, Fwd) - (Net, PP, Fwd). EDHOC is used between A and RP." artwork-align="center"}
