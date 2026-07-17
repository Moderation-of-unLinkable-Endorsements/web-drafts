# Moderated Endorsements

## Authors:

- Benjamin VanderSloot (Mozilla)

## Participate
- [Issue tracker](https://github.com/Moderation-of-unLinkable-Endorsements/web-drafts/issues)

## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
  - [More practically, from where the user sits](#more-practically-from-where-the-user-sits)
  - [Non-goals](#non-goals)
  - [The Incomplete MoLE Dependency](#the-incomplete-mole-dependency)
- [How does this work?](#how-does-this-work)
  - [Anchor endorsement](#anchor-endorsement)
  - [Moderator use](#moderator-use)
  - [Mapping the architecture onto the web](#mapping-the-architecture-onto-the-web)
  - [Constraints](#constraints)
    - [What the cryptography gives us](#what-the-cryptography-gives-us)
    - [Moderator restrictions](#moderator-restrictions)
    - [Anchor set policies](#anchor-set-policies)
    - [Random failure](#random-failure)
    - [Anchor semantics](#anchor-semantics)
    - [Partitioning, or the lack thereof](#partitioning-or-the-lack-thereof)
    - [Feedback](#feedback)
    - [Embedded content, workers, and miscellania](#embedded-content-workers-and-miscellania)
- [Alternatives Considered](#alternatives-considered)
- [Accessibility, Internationalization, Privacy, and Security Considerations](#accessibility-internationalization-privacy-and-security-considerations)
- [Stakeholder Feedback](#stakeholder-feedback)
- [References & Acknowledgments](#references--acknowledgments)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

Websites have few options at hand to limit abusive or high-volume traffic and
none of them are very good. Tracking users across the web based on apparent fingerprint is
a violation of their privacy. CAPTCHAs introduce a friction to many visitors
to a site, particularly to users that use privacy-enhancing user agents. Client
integrity checks close off the Web to many users.

This work proposes a new option: Moderated Endorsements. This proposal is the
user agent deployment of emerging work at the IETF,
[MoLE](https://moderation-of-unlinkable-endorsements.github.io/internet-drafts/),
which is an attempt to solve the
[PACT problem statement](https://github.com/antifraudcg/proposals/issues/22),
from the W3C Anti-Fraud CG.

When a user visits a site that is confident the user is genuine, that site may
_endorse_ that user. Later, when the user visits a different site for the first
time, the user may redeem their endorsement with a service that the site uses
for access control, called a _moderator_. The user proves that they (1) were
endorsed by one of a trusted set of sites and (2) is following a policy set
between the moderator and site, like a rate limit.

### More practically, from where the user sits

Suppose I'm a user browsing the web and I come to a website that is
uncertain whether my traffic is from a person using their website or a bot
scraping content. Even after invasive browser fingerprinting and IP reputation
are considered, a site may not be certain. Currently on the Web, one common
next step is for the website to make me or my computer do some work, in the
hopes that this proves too expensive for abusive traffic at scale. This wastes
the time and energy of the odd unclassified user. It gets worse if I'm
misclassified as a bot; the site could block me entirely or degrade service
to an agonizing pace.

All of this falls out of a site doing its best to sort out abusive traffic.
This proposal suggests an additional signal sites could use to identify traffic
that is more likely to be non-abusive.

Imagine if one of the sites I've already visited could endorse me as
not being abusive to the site that might otherwise issue a challenge. Email
providers, paid VPN providers, or any site with an account system would have
some degree of belief in my good behavior. Taking it a step further, imagine the
site I'm visiting trusts a collection of different endorsements that cover
quite a few users, e.g. all webmail providers. Even better yet, we can imagine
that the site I'm visiting can't tell which site a given endorsement comes from,
even though it knows my endorsement is from one of them. Finally, imagine that
the site I'm visiting could coordinate with others to share a rate-limit
without linking my identity between sites.

### Non-goals

This proposal looks to create an effective cross-site signal for use
by sites to help detect abuse. This isn't the hard part of this design. The
hard part is not creating second- or third-order effects that make the Web
worse.

The aim of this work is to **not**:

- Create a load-bearing requirement to have an account at a website to browse the Web
- Allow any new exchange of information from the OS to the website
- Incentivize more centralization than already exists in anti-abuse providers
- Create a new vector for cross-site tracking


### MoLE Dependency

This approach builds on the
[MoLE Architecture](https://datatracker.ietf.org/doc//draft-jms-mole-architecture),
though the work there isn't yet final. Its
[new protocols](https://datatracker.ietf.org/doc/draft-jms-mole-protocols)
are needed to communicate only the information we want with our messages and the
[HTTP transport](https://datatracker.ietf.org/doc/draft-jms-mole-http-transport)
gives us the language needed for us to send those messages between clients and
servers.

Even though it is early, there is enough structure there to make it worth
discussing the deployment-specific considerations that will have to be made if
MoLE is to make its way into user agents.

## How does this work?

The benefit of Moderated Endorsements is that it allows us to share information across
sites. To explore it fully, we therefore have to imagine a few sites willing to work
together.

- Site F, where a user is visiting for the first time and wants to take some
    more expensive action, like posting a comment or searching a database
    repeatedly. They want to mitigate abuse while minimizing friction for
    new and returning users.
- Service M, who manages abuse mitigation for Site F. Right now they
    initially fingerprint a user and if they are suspicious of that
    fingerprint's activity they force the client to complete some contrived
    busywork. This may be same-site or cross-site to Site F.
- Site A, where a user both has an account on and has a history of normal use.
    They, like everyone, dislike CAPTCHAs and want their users to see fewer
    in their life.


### Anchor endorsement

Taking actors from our example, Site A decides that our user is trustworthy.
They can represent this by directing the user agent to claim an _endorsement_.
The user agent performs an exchange with Site A's server and stores some state
for this. See [below](#anchor-semantics)  for more detail on what semantics
this may carry.

A website can endorse a client in a few different ways. Imperatively, this is
a call to `navigator.endorsement.collect("/endorsement_uri")`.
Declaratively, this is adding the tag `<link rel="endorsement"
href="/endorsement_uri" >` to a document's head.

Either of these would cause a POST Fetch to the URL provided, following one
of the
[endorsement protocols](https://www.ietf.org/archive/id/draft-jms-mole-protocols-00.html#name-endorsement-protocols).
Any endorsement received in response is stored, keyed by the Origin of
the top-level document.

### Moderator use

Now, some time later, our user is visiting Site F and as part of their normal
actions trigger an anti-abuse check. Site F fetches some resources from
Service M, either as scripts or an iframe, and as part of their bot-or-not
confidence calculation, decides that anchor endorsement would be useful. Service
M declares a set of anchors that it trusts equally and asks the user agent to
provide a proof of endorsement from one of those anchors.

Again, this could be done in a few ways. Imperatively, this would be a call to
`navigator.endorsement.challenge("/moderator_uri", challengeOptions)`. What has
to  be in `challengeOptions` is a bit up in the air at the moment, but it is
important to note that it is entirely obtainable within the context of Site F.
So far it contains information about the moderator's policies and the session,
time, or request that this use is bound to.

If the user agent hasn't already used that moderator, it will have to get a
credential from it at this point. This is done through some Fetching from
Service M while providing a proof that you have an endorsement from one of the
anchors it trusts. Critically, this doesn't reveal which anchor endorsed the
user, just that one of them did.

With a credential for the moderator in hand, the user agent presents its
credential by fetching the URL provided with a
`Authorization: Mole presentation="<credential-presentation>"` request
header, or whatever is finalized in the [transport definition](https://datatracker.ietf.org/doc/draft-jms-mole-http-transport/). 
The moderator can prove that the presentation is valid and determines
how it wants to update the associated state of the credential. The user agent
uses the response from the moderator to update the credential state and returns
it to the caller.

We could similarly cook up some sort of declarative version that encodes the
arguments to `challenge` in the `<head>` of a document, while applying an
attribute to elements that may issue a request for which a Moderated Endorsement
would be useful. We leave that for when the options are better defined.

### Mapping the architecture onto the Web

The architecture document defines the interaction between Moderators, Anchors,
and Clients. We make the important distinction that in a Web deployment, the
only component that makes up the Client is the user agent itself: not the
websites. The websites are part of the Moderator or Anchor, depending on the
method being invoked. For example, a call to `navigator.endorsement.challenge()`
is a `PresentationChallenge` issued by the site (Moderator) to the user agent
(Client).

This API may be extended to include a model where the JavaScript execution
context is untrusted by the website to issue challenges or handle the
presentation response. This would be as simple as determining _which_ Fetches
process `WWW-Authenticate: Mole` response headers. This could be determined by
the
[Request's destination](https://developer.mozilla.org/en-US/docs/Web/API/Request/destination).

TODO: diagram with Site

### Constraints

Now, the hard part: constraining the system to meet our non-goal requirements.

#### What the cryptography gives us

Fortunately, MoLE builds on cryptography that gives us useful privacy
properties to build on.

First, presentations of a credential are unlinkable to their past updates. For
our deployment, this means that if the only information crossing a boundary is
a presentation from the user agent, then the moderator does not learn where
that credential has been used before.

Second, when a client provides a proof of an anchor's endorsement to a
moderator, the moderator does not learn which anchor endorsed the client. It
only learns that one of the anchors in the set it advertised to the client
did.

#### Moderator restrictions

Each Site should be restricted to a single moderator. This prevents a site
from using the states of several credentials as a cross-site tracking signal,
even when those credentials are based in the same anchor set. A Site can
change moderators by clearing its storage.

#### Anchor set policies

Each moderator picks which anchors it roots a client's reputation in. The
composition of this set is critical to the privacy and centralization
properties of the system. A few user-agent-enforced rules help guide it to an
acceptable state. The numbers below are reasonable starting points and would
require careful consideration.

First, at least 50% of user agents must hold endorsements from more than one
anchor in the set. This ensures that the system does not rely on a single
anchor, and that holding an endorsement from the set does not reliably reveal
an endorsement from any particular anchor.

Second, a moderator must present each client with at most 2 different anchor
sets within any 6-month period. This allows iteration and experimentation
without rapidly accruing information about each user's endorsements.

#### Randomly injected failure

This system should not be relied upon as the sole signal of user quality.
Moreover, users who lack endorsements from an extremely popular anchor should
not be easy to identify. To those ends, user agents should withhold an
anchor's endorsement from a site with a small probability, e.g. 5%. Together
with the anchor set policies, this lets us compute differential privacy
guarantees for the information a site gains about a user's endorsement or
non-endorsement by any given anchor.

#### Anchor semantics

The semantics assigned to an anchor's endorsement are beyond the control of
this API's design. We expect them to converge on something like "non-abusive
account," but that is purely a prediction. However, any use of an endorsement
that is gated behind device attestation must be treated as abusive, and the
corresponding anchor must be blocked by user agents. Without this collective
action, this API could become a vector for significantly reducing the friction
to device attestation across the Web, in direct violation of the non-goals
above.

#### Partitioning, or the lack thereof

A key piece of Moderated Endorsements is that credential and endorsement state
is not partitioned by top-level site. This is what lets the constrained signal
we have constructed flow between sites, and is precisely the point of the
design: to provide just enough information between sites that they do not
resort to more invasive and persistent mechanisms.

A side effect is that when a moderator is shared by multiple sites, all of
those sites must follow the policy established by that moderator for the
system to work. A moderator must therefore exercise control over which sites
use it and how.

#### Feedback about anchors

When picking an anchor set, a moderator faces a tough choice. It may
understand each anchor's advertised semantics, but be unable to compare their
efficacy at determining whether traffic is trustworthy. Because anchors are
indistinguishable to the moderator at time of use, the moderator's confidence
in its anchor set is bounded by the set's weakest link. It is therefore
important for a moderator to gain insight into which anchors were used at the
issuance of the most misbehaving credentials.

There are two possible paths forward. The first is to exploit the flexibility
already allowed in the anchor set: by rotating between different sets as an
experiment, the moderator can determine which anchors' endorsements correlate
with the most abuse. It can either use the 2-sets-per-client-per-6-months
allowance above, or segment users into experiment groups. Alternatively, the
MoLE architecture draft mentions integrating a privacy-preserving measurement
technique.

#### Navigational tracking

Many of the restrictions above are about constraining what a single site can
learn about a user agent. Sites often work together to aggregate information
about a user, particularly as the user navigates between sites. As presented
so far, a single site could bounce a user through several pages on different
sites, gather each moderator's signal, append it to the URL at each hop, and
finally deliver the user to their intended destination.

To prevent this, we suggest that the user agent gate access to the API on
something like user activation, but one that carries across page
navigations. This has its own problems and is tracked in
[an open issue](https://github.com/Moderation-of-unLinkable-Endorsements/web-drafts/issues/1).

#### Embedded content, workers, and miscellania

Moderated Endorsements share a finite resource at the top window's Site: the
choice of moderator. To prevent frames from making that choice on behalf of
the rest of the site without permission, Moderated Endorsements should be a
policy-controlled feature with a default of `self`.

Similarly, because the feature is tied to a top-level window, it seems safest
to expose the JavaScript API in windows only.

Not-fully-active windows, windows whose top-level has an opaque origin,
insecure contexts, and the other usual caveats should all apply, disabling
Moderated Endorsements in those cases.

## Alternatives Considered

TODO: Flesh this out

- PAT
- PVT
- PST
- Enable the Authentication mechanism on all fetches
- Direct Client-Moderator communication, requiring a Site-verifiable Moderator validation record
- More detail in the web API call to cut out the first round trip to the Site


## Accessibility, Internationalization, Privacy, and Security Considerations

TODO: Flesh this out

- The whole thing is kind of a privacy discussion.
- No web-security, a11y, i18n considerations
- talk about how the security considerations from the architecture are addressed

## Stakeholder Feedback

TODO: Issue template here

- link to file issue for feedback, support, or disapproval

## References & Acknowledgments

TODO: Fill this in

- TK, many

