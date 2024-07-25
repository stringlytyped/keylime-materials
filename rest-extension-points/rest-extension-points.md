# REST Extension Points for Keylime

## Background

[Keylime](https://github.com/keylime/keylime) currently performs registration and verification in a fairly fixed way. This means that, to a large degree, only those use cases which have been envisioned by the Keylime community are supported, unless one modifies Keylime source code.

Some forms of extensibility are already provided, for example:

- A documented set of [REST APIs](https://keylime.readthedocs.io/en/latest/rest_apis.html) which can be used to manage resources stored by the registrar and verifier (e.g., an agent's registered identities)
- A user-provided [webhook](https://github.com/keylime/keylime/blob/d68a722b2befc5bb5796c5530837455c59a26a25/keylime.conf#L297) which is called to notify an external system of revocation events produced when an agent fails verification
- A user-provided [EK check script](https://github.com/keylime/keylime/blob/d68a722b2befc5bb5796c5530837455c59a26a25/keylime.conf#L543C1-L543C17) for implementing custom logic to verify endorsement keys (EKs) and/or EK certificates

The EK check script is called by tenant and will be superseded by a [_trust decision webhook_](https://github.com/keylime/enhancements/blob/master/103_agent-driven-attestation.md#webhook-mechanism-for-custom-trust-decisions) which is invoked by the registrar and provides a more general mechanism that applies to any identity supported by the registrar.

Previous proposals[^tee-boot-prop] [^spire-prop] have contemplated extensions to Keylime which go beyond Keylime's core purpose of (1) collecting attestations, (2) performing verification thereof, (3) reporting the results of that verification, and (4) performing these three tasks in a cryptographically sound and secure manner. As the possible use cases for a general attestation tool like Keylime are so diverse, it is likely not feasible for the Keylime community to directly support and maintain functionality which implements these integrations ourselves.

However, attestation and verification is most compelling when done in concert with various other systems, including key brokers, hardware inventory systems, workload identity systems and cloud service providers (CSPs), to name only a few. Keylime should endeavour to provide well-specified mechanisms for extending Keylime core behaviour to allow the end user to hook it up to their wider infrastructure or even implement their own custom logic.


## Proposal

One way of providing such extensibility would be to continue the established Keylime practice of permitting integrations through an HTTP interface, e.g., webhook URIs which are called at various points in the registration and verification lifecycle. This approach has a number of advantages:

- Allows integration with external systems which could be running in separate containers, VMs, or even datacentres, with communication potentially taking place across NAT or firewall boundaries.
- Integration with processes on the same host are also possible, with HTTP communication taking place over a loopback interface.
- The external system can be written in any language, run on any OS and hardware, and be implemented using any library available to the developer (whereas Keylime components can only be implemented using those Python libraries packaged by the supported Linux distros).
- REST-based extension points are easy and quick to implement.


### Limitations

There are however situations for which this REST-based extension framework would be inappropriate, for instance:

- For implementing new modules or components shipped with Keylime itself or maintained by the Keylime community (or to modularise Keylime's existing features). We should not force users into having to maintain a complex array of interconnected microservices to use core functionality.
- For implementing any functionality that would be better served by storing additional data within the registrar or verifier beyond simple metadata, possibly with additional [models](https://github.com/keylime/keylime/blob/master/keylime/models/base/basic_model.py) defined to handle mutation of data objects.
- For most cases in which you want the agent to collect additional data to be consumed by the registrar or verifier.

For these scenarios (and possibly others), we should support a proper plugin framework with a defined interface, so that additional modules can be loaded into Keylime itself to change how it performs registration and verification. These would be similar in nature to the [node and workload attestor plugins](https://spiffe.io/docs/latest/planning/extending/) supported by SPIRE. Additionally, existing supported identities (EK, AK, IDevID, IAK, etc.) and verification abilities (TPM-based IMA and UEFI verification) should ideally themselves be re-implemented in this plugin architecture.

Implementing these architectural changes requires work beyond what is considered in this document and should be explored in a separate proposal.


### Registrar Extension Points

The REST interfaces we should support to enable the above-described extensibility of the registration process are given below:

#### Trust Decisions Webhook

Allows an external web service to provide additional identities (keys and/or certificates) which the agent itself was not able to obtain for whatever operational reason and/or override trust decisions reached by the registrar by performing verification of the available identities using the user's own custom logic.

This is specified in [more detail in Enhancement 103](https://github.com/keylime/enhancements/blob/master/103_agent-driven-attestation.md#webhook-mechanism-for-custom-trust-decisions).

#### Registration Events Webhook

Notifies an external web service of various events that take place with regard to the agent registrations held by the registrar. These events can be any of the following types:

- **Agent registered**: An agent has registered or re-registered with the registrar.
- **New agent registered**: A new agent has been added to the registrar.
- **Agent registration updated**: An existing agent's registration has changed from the previously held registration. Not triggered when certain internal fields change (e.g., when `regcount` increases).
- **Agent registration deleted**: An agent has been deleted from the registrar.
- **Final trust decision reached**: The registrar has made a final determination for an agent's registration (as it currently exists) as to whether each of the identities belonging to the agent are deemed trusted and, based on this, whether the agent registration as a whole is trusted. This is called after any calls to the trust decisions webhook and after the agent has successfully passed all challenges need to establish a cryptographic binding between the agent's various identities.

The user can choose which of these events they wish to send to the external service. The data provided in the JSON object sent to the webhook will depend on the specific type of event.

The types of events may be expanded in the future to cover events related to other resources managed by the registrar, e.g., certificates in the trust store.

#### Registration Events Subscription API

Deployments with a large number of attested nodes or where registrations occur frequently will produce a large number of registration events. It could be costly for the registrar to generate notifications for these and for an external web service to handle the notifications. As an alternative, an external service may use the subscription API to declare its interest in receiving notifications only for a specific subset of agents and/or for a specific subset of changes.

This would be done by making a POST request to the events subscription endpoint (`/v3/events/subscriptions`) and providing:

- a list of agent IDs which the service wishes to receive notifications about;
- a list of fields to notify the service about when they change (can be any field which is exposed when issuing a GET request to `/v3/agents/:id`); and
- the URI which the registrar should contact when a change occurs which meets the given criteria.

The events subscription API may be expanded in the future to support events related to other resources beyond just agent registrations.

### Verifier Extension Points

The REST interfaces we should support to enable extensibility of how verification is performed are given below:

#### Verification Decisions Webhook

Similar to the registrar's [trust decisions webhook](#trust-decisions-webhook), allows an external service to receive a copy of the evidence proccessed by the verifier, modify the evidence, and/or override decisions about whether the evidence is considered to have verified successfully and/or whether the evidence is deemed authentic.

#### Events Webhook

Similar to the [registration events webhook](#registration-events-webhook), allows an external service to receive notifications about certain events. These include:

- Agent added
- Agent updated
- Agent deleted
- IMA policy added
- IMA policy updated
- IMA policy deleted
- MB policy added
- MB policy updated
- MB policy removed
- New attestations


- agents
- attestations
- verification result 


### Implementation Considerations

multiple uris


<br><br>

**Footnotes**

[^tee-boot-prop]: [Enhancement 107: TEE Boot Attestation](https://github.com/keylime/enhancements/blob/5140a6580befd8fa849405efe84375729f691da9/107-tee-boot-attestation.md)

[^spire-prop]: [Enhancement 98: SPIRE Integration](https://github.com/keylime/enhancements/blob/aa6b020798d4052345da4ac363721cecd67cc94b/98_spire_integration.md)
