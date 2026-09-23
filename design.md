# MoLE Web Integration: Design and Privacy Model

## Privacy and Security Requirements

Security:

* **Site authorization:** Check that the Moderator authorizes the site requesting a credential.
* **Anchor authorization:** Check that Anchors authorize the Moderator used without contacting Anchors at redemption time and revealing where their Endorsements are used.

Privacy:

* **Prevent Session linking:** Prevent user sessions from being linked across contexts.

* **Limit Cross-Context Information Flow:** Limit information which can be transferred across contexts and associated with a user's identity, preventing information from being accumulated.

## Identity, Configuration, and Authorization

### Moderator -> Site Authorization

Two supported routes:

* The site supplies a Moderator-signed authorization covering the site url and an expiry. The client checks the signature against a public key previously conveyed by the Credential and then the client gives presentation proofs to the site to forward to the Moderator.
* The client checks authorization directly with the Moderator, similar to a CORS preflight, then presents directly to it. This requires at least one extra round trip.

### Anchor -> Moderator Authorization

Two options:

* **UA Vendor signs Moderator Metadata:** Anchor metadata links to a list of authorized Moderator URLs. The user-agent vendor checks that each Anchor listed by a Moderator authorizes it, then signs the list carried in Moderator metadata. Clients check that signature before redeeming an Endorsement.
* **Endorsement permissions:** Each Endorsement carries the permitted Moderator URLs, or indicates unrestricted use. The client checks these permissions locally.

Open Aspects:

The first approach also helps with anonymity set checks because the UA Vendor can check those metrics during the sign off process.
But it also requires a lot more machinery and infrastructure. It might be challenging if say an Anchor is unreachable or doesn't appear to have authorized the Moderator - should that block signing of the metadata for the Morderator entirely?

Alternatively, with approach 2 information about anchor an could be provisioned directly to clients via a vendor-specific mechanism.

### Identity and Configuration IDs

Configuration IDs associate popularity measurements with a specific configuration and make configuration changes detectable.

* Use metadata URLs as long lived identifiers for Anchors and Moderators which remain stable across configuration changes.
* Anchor metadata supplies the Endorsement endpoint and protocol parameters.
* Moderator metadata supplies endpoints, protocol parameters, the Anchor set commitment, and accepted Anchor IDs with their metadata URLs.
* Define a config ID for Anchors and Moderators, which is a hash of their public configuration, which changes whenever their user anonymity set changes.
* Key popularity measurements and other privacy checks by configuration ID.

The Moderator ID doesn't need to bind the Anchor Set Commitment. The Moderator can change it's Anchor set without invalidating the anonymity for users with its credentials.

### Local or Global Consistency

Once a user-agent has associated a Site with a Moderator (e.g. by being asked for a Credential), it needs to lock that Site to using that Moderator ID for a period of time. Otherwise, later use of a different Moderator / Anchor leaks information to the site.

This can be enforced purely locally by the user-agent retaining state. This means in turn that the site can choose which Moderator to use with a given visitor. For example, this might be based on geographical location or the user's apparent OS.

Alternatively, sites could be required to declare which Moderator they use in advance and be required to use that Moderator with all visitors. This would require user-agent vendors or some third party to assert this property via checking the site's metadata and signing it. This achieves global, rather than local, consistency.

For any given user, the worst-case privacy analysis doesn't change. Global consistency may have stronger privacy properties for an average case analysis though (since only one Moderator can be used for every visitor). Global consistency also achieves better equity properties by requiring uniform treatment, though in practice sites may still treat different user-agent fingerprints differently even if they use the same Moderator.

Further, local consistency allows sites to experiment with more Moderators and may simplify other aspects of management. It also avoids user-agent vendors needing to permission sites, even though the criteria used for permissioning would be objective rather than subjective.

### Rotation and Configuration Changes

### Site Changing Moderator

Sites will want to change Moderator periodically. If we use local consistency, this can be a smooth migration. In particular:
  - Sites can keep using the old Moderator with existing users and migrate new users to the new Moderator
  - Sites can have users Present an existing Credential, not return it and instead grant a new Credential from the new Moderator.

Privacy protections will come into play to prevent sites cycling Moderators too quickly.

### Token Hoarding and Expiry

Moderators want to ensure that users can't stockpile large numbers of Endorsements or Credentials. We can use an epoch system for expiry, which is bound into the issuance context for each. The Moderator can accept Creds / Endorsements from the current and previous Epoch and reject others. It can also return a Credential with a new Epoch.

### Moderator or Anchor changing Public Key / Metadata

This is expected to be infrequent but in practice is equivalent to a whole new Anchor or Moderator being set up. Practices for rotation where users are issued new credentials after consuming their old credentials can help, but it's an identical situation to a site switching to a new Moderator.

### Moderator Changing Anchor Set

This is fine? We don't need to restrict this?

### Anchor changing Moderator Auth

Anchor auth of a Moderator is conveyed via the signed Moderator Metadata so will take some time to update (e.g. 24 hours plus) depending on how frequently metadata is signed or pushed to clients.

### Moderator changing Site Auth

If based on a signature this will happen whenever past grants expire. If based on a CORS style lookup this can be instant.

## Web API and Platform Integration

### API Shape and Rationale

As Credentials are stateful objects which cannot be used concurrently, the browser API needs to allow sites to gain exclusive access to a Credential and hold it across a number of operations. We represent this with a handle which enables the site to perform operations on the Credential, whilst the browser upholds the necessary privacy properties and handles the underlying orchestration.

Other possible API shapes:

* **Bare:** JavaScript forwards protocol messages and supplies the responses. The browser would still retain Credential secrets, validate responses, and control state transitions. This offers transport flexibility but complicates site implementation.
* **Fetch integration:** The site attaches an operation to an application request, as in PST. This associates the operation with a request, but still needs a way to express the Credential lifetime across multiple updates.
* **Header style:** A server challenge triggers browser-managed operations, as in PAT. This could support a standard flow, but needs rules for updates and release.

It would be reasonably straightforward to develop these alternatives in a future extension. For example, offering a Bare API for sites which wanted the additional control or a Header API for sites who are less opinionated or want a JS-free option.

### API Sketch

All operations return promises.

| Operation | Purpose |
| --- | --- |
| `Endorse(anchor_url, options)` | Collect and store an Endorsement. The optional `options` argument supplies authorization bytes and a credentials mode. Resolves with a result. |
| `IssueCredential(mod_url)` | Obtain and store a Credential if needed, without reserving it for later use. Resolves with a result. |
| `GetCredential(mod_url)` | Acquire and lock a Credential, returning a handle to browser-managed state. May trigger issuance if none is available, including when existing Credentials are locked. |
| `credential.Present(challenge)` | Present against a site-supplied challenge, consuming the current state before sending. Retain any replacement state returned by the protocol. Resolve with arbitrary application bytes returned by the Moderator Presentation Endpoint. |
| `credential.Update()` | Update usable Credential state, whether it is newly issued or a replacement from an earlier presentation. Cannot run while presentation is pending or the Credential is burned. |
| `credential.Release()` | Release the lock and return any usable Credential to the pool. |

Example use is `GetCredential` → `Present` → optional `Update`(s) → `Release`, provided presentation returns usable replacement state. Updates may also precede the first presentation. Page termination releases the lock.

`IssueCredential` allows advance provisioning: a landing page can prepare a Credential before a form submission acquires and presents it. This does not guarantee later availability or hold a lock between those events. If existing Credentials are locked, the browser may try to have another issued, subject to the privacy checks.

In some situations, the site might want to handle the API endpoints directly and be responsible for forwarding them on to the Moderator. As described in "Moderator -> Site Authorization", this could be handled by extending `GetCredential` with optional arguments to specify the new endpoint and provide a signature for the browser to check.

The state of the underlying Credential / Endorsement is opaque to the API. The user-agent will be aware of the Credentials state / balance, and the thresholds will be communicated in the Presentation flow, but the API doesn't need to be aware of these values. If the site needs to be aware of the threshold used, it could be communicated via the bytes returned in the Present flow.

TODO: Do we need to bind the amount required when the Credential is locked for use? To avoid a probing attack later? That is leaking 'I have some valid credential with not enough credit' vs 'I do not have any valid credential'.

TODO: Do we need to limit how many endorsements from distinct Anchors you can get from a site?

TODO: Because of the need to lock credentials, Moderators need to issue enough Credentials for simultaneous active sessions.

### Binding and Results

* `Present` accepts a site-supplied challenge to bind the presentation to the site's request or session.
* The Moderator Presentation and Update Endpoints may return arbitrary application bytes, which they yield to the caller. The site chooses how to interpret them, including any server-side validation, for example, if the Moderator and site share a HMAC key, the bytes might be interpreted as `HMAC(challenge, key)`.
* The browser handles protocol state updates separately from those application bytes.

`Endorse` accepts optional authorization bytes and a Fetch-style credentials mode:

```js
await Endorse(anchor_url, {
  credentials: "same-origin", // Default; also supports "omit" and "include".
  authorization: authorizationBytes, // Optional; opaque to the browser.
});
```

* The credentials mode controls ambient credentials, such as session cookies, subject to ordinary browser cookie and cross-origin restrictions. Explicit authorization bytes are separate from this mode and delivered alongside any credentials.
* Exact transport encoding is deferred to the spec.

### Handling third-party resources and Iframes

If there is a mechanism for challenging for a Credential via a header, then sites will need a way to declare if other origins can request a Credential. Similarly, if Iframes can request a Credential. As this is a new API, we can deny by default unless permission is granted.

## Browser State and Credential Lifecycle

### Stored State

* Store Endorsements and Credentials without partitioning by top-level site.
* We'll likely have a pool of Endorsements and Credentials, keyed by their Anchor / Moderator ID.
* In each browser context, we'll need to track Credential usage in order to enforce rules around use of multiple Endorsements or Credentials.
* We'll also need careful concurrency management to avoid side channels between contexts.
* The spec will need to define careful rules to ensure state is always consistent and privacy-preserving in the presence of browser/device crashes. For example, when presenting a credential to a Moderator, we must update our local state first and then release the presentation to the network.

### Credential Lifecycle

* Acquiring a Credential locks it for the caller. If no suitable Credential is available, including when existing Credentials are locked, acquisition may trigger Endorsement redemption and Credential issuance.
* Presentation burns the current state before sending it. The Credential is unusable while presentation is pending. When the exchange returns, any valid replacement state becomes usable; without replacement state, the Credential remains burned.
* `Update` operates only on usable state. It may update a Credential that has never been presented, or replacement state from an earlier presentation, and overwrites the stored state.
* Explicit release or page termination returns any usable Credential to the pool; releasing a lock does not restore burned state.

### Concurrency

Sites sharing a Moderator can otherwise link sessions by locking and releasing Credentials and observing which other sessions can respond. We can limit the effectiveness of this channel by preventing probing.

Proposed constraints:

* Once a Credential has been used in a context, activity in other contexts must not make it unavailable through contention. That is, it has to stay locked until released.
* If a Credential cannot be acquired because it is unavailable and can't be issued, future attempts to acquire a credential in that contexct should continue to fail
* Once released, a Credential cannot be re-acquired in that context.
* It might be possible to lighten these requirements by tolerating further probes after a certain amount of time has passed or enough user interactions have occurred.

### Context Tainting

Once a cross-site information has occurred (redeeming an Endorsement, presenting a Credential which was used cross-site), we've effectively conveyed information between contexts. For example, between partitioned top level sites. In order to prevent the amount of conveyed information from being unbounded, we'll need to taint this content to prevent further uses of cross-site information flows.

The scope of a context is the scope in which tracking information can be conveyed. For example:
 * Future returning visits to a site need to apply the taint.
 * Navigations to subsequent sites need to apply the taint
 * Cross-origin requests or embedded iframes need to be tainted.

In order balance utility and privacy, contexts will need to be untainted after a period of time or after sufficient user interactions.

TODO TODO TODO

### Clearing History and Site Data

TODO TODO TODO

Rules for how clearing data works with Credentials and Endorsements

TODO TODO TODO

## End-to-End Operations

### Endorsement Collection

1. During ordinary browsing, a site decides to endorse the client and calls `Endorse(anchor_url, options)`.
2. The browser retrieves Anchor metadata and contacts the Endorsement endpoint, supplying any authorization bytes and permitted session credentials.
3. The browser and Anchor negotiate and run the Endorsement protocol.
4. The browser stores the Endorsement with its Anchor configuration ID and collection context in the shared pool and resolves the call. It retains the Endorsement even if it is not yet eligible for cross-context redemption.

### Credential Acquisition

1. When a site wants to challenge the client, it calls `GetCredential(mod_url)`.
2. The browser retrieves Moderator metadata, checks site authorization, and applies the context's concurrency and disclosure restrictions.
3. The browser selects a suitable unlocked Credential that is eligible in this context. A Credential from a Moderator below the popularity threshold is eligible only in its issuing context. If none is available, the browser may attempt redemption and issuance for this context, including when existing Credentials are locked or scoped elsewhere.
4. The browser locks the selected or newly issued Credential and returns a handle to the site.

A site may instead call `IssueCredential(mod_url)` in advance. The browser performs the applicable checks and provisions a Credential if needed, leaving it in the pool without reserving it for that context.

### Endorsement Redemption and Credential Issuance

1. The browser selects an Endorsement accepted by the Moderator's Anchor set.
2. It checks Anchor authorization without contacting the Anchor, and applies the context's disclosure restrictions. Redemption outside the Endorsement's collection context requires the Anchor set to meet the popularity threshold.
3. The browser runs Endorsement redemption and Credential issuance with the Moderator.
4. It stores the resulting Credential in the shared pool, keyed by Moderator configuration ID and retaining its issuing context. Low Moderator popularity does not block issuance, but limits subsequent use to that context. If issuance was triggered by `GetCredential`, acquisition then locks it for the caller.

After collecting an Endorsement or issuing a Credential, the browser may report the success to its vendor through privacy-preserving telemetry, such as OHTTP, keyed by the relevant configuration ID. This is to enable anonymity set calculations.

### Presentation, Update, and Release

1. The site calls `credential.Present(challenge)` with its challenge. The browser checks eligibility for use in this context. Use outside the Credential's issuing context requires its Moderator configuration to meet the popularity threshold.
2. The browser burns the current state before sending the presentation, either directly to the Moderator or through the site's forwarding endpoint. The Credential remains unusable while the exchange is pending.
3. The browser stores any valid replacement state returned by the protocol and yields the Moderator's application bytes to the site. Without replacement state, the Credential remains burned.
4. The site may call `credential.Update()` on usable state, either before any presentation or after a presentation has returned replacement state. The same context eligibility checks apply. The browser runs the update, replaces the stored state, and yields any application bytes. Replacement state retains the Credential's issuing context.
5. The site calls `credential.Release()`, or the page terminates. The browser releases the lock and returns any usable Credential to the pool, retaining the context state needed to enforce concurrency and disclosure restrictions.

### Anonymity Set Checks

These flows introduce two cross-site information flows:
 * The purpose of an an Endorsement is to be issued in one context and redeemed in another.
 * A Credential may only be used in the same context it was issued in, but can also be used across contexts.

In order to protect user privacy without compromising on utility, we want to ensure that the cross-context information revealed does not identify a small population of users. To that end, user-agents can report when a specific Anchor issues an endorsement or a Moderator issues a credential. These reports need only be coarse usage counts keyed by the Anchor or Moderator ID.

If the Anchor or Moderator is not sufficiently popular according to a user-agent determined threshold, its Endorsements or Credentials are not necessarily available cross-origin.

For Anchors, information is revealed when their Endorsement is used cross-origin. However, the Anchor-blindness property of the redemption protocol means that it suffices for only one popular Anchor to be included in the Anchor set. Consequently, an Anchor which is not popular can still be used effectively as a supplement to other popular Anchors.

For Moderators, they can still participate in the system but their scope of protection is more limited because they cannot maintain cross-site state. That is, their credentials effectively become scoped to individual sites.

This analysis becomes more complicated when considering the use of IP tracking, browser fingerprints or other mechanisms which may reduce the practical anonymity set size.

The anonymity set checks need to take into account expiration / recency. In particular, the epoch for expiry or other elements of the issuance context need to be part of the Anchor ID?

## Privacy Analysis (Rough Work)

### Disclosure after one interaction

We consider the simple case with a single interaction.
The user receives an Endorsement in one context.
The user presents it in another in exchange for a Credential. Then uses the Credential.
The resulting cross-site information flow is that the user is in the set of users which received a valid Endorsement.
Due to the issuer-blindness and the anonymity set calculations, the site learns only that the user is in a set of at least the anonymity set size.

### Repeated Probing and Configuration Changes

* A site with a persistent user identifier can learn more by changing its Moderator or the accepted Anchor set and triggering fresh redemptions.
* The persistent user identifier allows the information gained in each interaction to be accumulated over time.
* User-agents can restrict the bandwidth of this channel by limiting how often Endorsements are presented to the same-site.
* Providing a flow in which an old Credential is exchange for a new one would help user-agents enforce strong protections without reducing security

### Credential Cross-Site Usage

* On one site a Moderator does a bare issuance of a Credential
* On another site, the Moderator asks for a presentation.
* This is protected by the same anon set calculations as an Endorsement.
* However, due to mutable state, there are ways around it.
* For example, issue everyone with 10 credit except a target with 9.
* On another site, challenge everyone for 10 credit and refund 10
* Conclusion: API should not let sites distinguish between 'valid credential with not enough credit' and 'invalid credential'. That at least ensures the user with 9 blends in with the anonymity set of users without credentials.

### Navigation and Cooperating Sites

* Site A challenges for a redemption and asks the user to present a credential.
* The user is then navigated to Site B (e.g. by user action like clicking a link) or via some other mechanism
* If Site B can also ask for a redemption, then site A and site B can collaborate to recover information about the user, e.g. through a tracking parameter on the navigation.
* As a consequence, user-agents should track whether a context has revealed an endorsement in the context and prevent future redemptions. Sites might want to allow multiple redemptions when privacy analysis enables it? For example if both contexts are clean, there's little to join?

### Embedded Sites

* Only one redemption is possible in the context of a given site.
* For example, an embedded iframe shares the privacy budget for the top level site, since the two can communicate via messages.
* An isolated iframe, e.g. a fenced frame (now deprecated?) would not need to share the same budget.

### Cross-Site Joins

* Imagine two sites, site A and site B on which the user is anonymous (not logged in or revealing identifiers). Imagine two further sites X and Y in which the user has signed up via the same email address.
* If X challenges for an endorsement from Site A and Y challenges for an Endorsement from Site B, then X and Y can collaborate to learn two bits of information. This scales to many sites.
* If a single site can give many Endorsements from different Anchors, then many sites can work together to recover those bits.
* Ultimately, this can't be prevented, so existing legal protections will have to suffice?

### Concurrency-based & Timing Attacks

* Imagine Site A and Site B which are protected by the same Moderator. The user has not shared any identifiers or logins with either site. The Sites and Moderator want to collude to link the user's session on Site A with the user's session on site B.
* A Credential can only be in use on one site at a time. So Site A locks half of it's user's credentials for a Update. Site A controls when the credentials become unlocked by returning the fresh data.
* Site B then attempts to lock all of it's users Credentials. Those which are locked on Site A will be rejected. This tells Site B whether a given user was in Site A's locked or unlocked pool.
* This can be repeated in order to eventually identify each user using [Group Testing](https://en.wikipedia.org/wiki/Group_testing) techniques.
* The mitigation for this is to make concurrency sticky. Once you've made a credential available, for a session, it must remain available. Once it's been made unavailable, it must remain unavailable.

