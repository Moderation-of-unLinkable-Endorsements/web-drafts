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

Some time later, our user is visiting Site F and as part of their normal
actions trigger an anti-abuse check. Site F fetches some resources from
Service M, either as scripts or an iframe, and as part of their bot-or-not
confidence calculation, decides that anchor endorsement would be useful. Service
M declares a set of anchors that it trusts equally and asks the user agent to
provide a proof of endorsement from one of those anchors.

Again, this could be done in a few ways. Imperatively, this would be a call to
`navigator.endorsement.challenge("/challenge_uri")`, which causes a Fetch to
`/challenge_uri` that will be responsive to a `WWW-Authenticate: Mole`
challenge. Declaratively, this is adding the tag `<link rel="challenge"
href="/challenge_uri">`.

To successfully handle the `WWW-Authenticate: Mole` challenge, the user agent
will need a credential from Service M. These credentials are stored keyed by the
Origin that Service M uses to host its Moderator service, so it can be reused
across sites.

If the user agent hasn't already used Service M, it will have to get a
credential from it at this point. This is done through some request to
Service M while providing a proof that you have an endorsement from one of the
anchors it trusts. Critically, this doesn't reveal which anchor endorsed the
user, just that one of them did.

With a credential from Service M in hand, the user agent presents its
credential by re-fetching the URL provided with a
`Authorization: Mole presentation="<credential-presentation>"` request
header, or whatever is finalized in the [transport definition](https://datatracker.ietf.org/doc/draft-jms-mole-http-transport/).
The moderator can prove that the presentation is valid and determines
how it wants to update the associated state of the credential. The user agent
uses the response from the moderator to update the credential state and returns
it to the caller.

A successful authorization can be associated with a cookie present in the same
exchange to create a persistent indication that the current user agent is the
same one that previously authenticated.

### Moderator use, per resource

The previous section allows the Mole authentication scheme to be processed on
very few requests. One simple expansion is to allow this processing on the
specific requests that are for expensive resources.

This can be done for scripts
by adding an additional instance property to
[RequestInit](https://fetch.spec.whatwg.org/#requestinit) that flags the
requests' participation: `moderated`, so that the following call would
participate in the MoLE authentication scheme: `fetch("/expensive",
{moderated: true})`.

Similarly, a `modereated` attribute could be applied to media elements, form
elements, frame elements, or even link elements to empower their fetches.


### Mapping the architecture onto the Web

The [MoLE Architecture](https://datatracker.ietf.org/doc//draft-jms-mole-architecture),
defines interactions between Moderators,  Anchors, and Clients. We make the
important distinction that in a Web deployment, the only component that makes
up the Client is the user agent itself: not the websites. The user agent
controls the flow of information out of the user's credential store. Site F is
part of the Moderator.

Even though Sites is composed of a server-side
and client-side component, both are isolated from the information of the
user agent's credential store. The client-side component of Site A needs to be
viewed with caution: because it is executing within the sandbox of the
user agent, its actions should not be trusted more than the Client's by Site A's
server-side.

Another important distinction arises between the Moderator's subcomponents:
only the Moderator's own servers are able to validate a presentation from the
Client (assuming the
[ACT credential protocol](https://moderation-of-unlinkable-endorsements.github.io/internet-drafts/draft-jms-mole-protocols.html#name-anonymous-credit-tokens-act).
This combines with the untrusted nature of the Site's client-side
component to impose a requirement: the presentation response from the Moderator
must be handled directly by the server-side component of the Site.

This makes the presentation flow look a bit more complicated than the
architecture document lets on:

```mermaid
sequenceDiagram
    box Client
    participant U as User Agent
    end
    box Moderator
    participant CS as Site (client-side)
    participant SS as Site (server-side)
    participant M as Moderator Service
    end

    CS->>U: Invoke a participating Fetch
    U->>SS: GET /foo
    SS->>U: 401, WWW-Authenticate: Mole <challenge>
    U->>SS: GET /foo, Authorization: Mole <presentation>
    SS->>M: GET /presentation, Authorization: Mole <presentation>
    M->>SS: 200, Mole-Credential: update="<optional-credential-update>"
    SS->>U: 200, Mole-Credential: update="<optional-credential-update>", Set-Cookie: <token>
    U->>U: Update credential
    U->>CS: Response(200)
```

The request flows from web content, through the browser, to the site's own
servers, on to a specialized moderator service's servers, and back.

### Constraints

Now, the hard part: constraining the system to meet our non-goal requirements.

TODO: fill the bullets below

#### What the cryptography gives us

- unlinkable presentation
- blind anchor proof

#### Moderator restrictions

- one per site
- API to fetch the current state

#### Anchor set policies

- minimal coverage of users
- coverage must still be good if one anchor is removed
- 2 policies per-user per-X-months

#### Randomly injected failure

- site-anchor pairs get x% chance of failure.

#### Anchor semantics

- probably just "non-abusive account"
- could also be "seen this before"
- we can't control it!
- If you are using device attestation, we pull the plug

#### Partitioning, or the lack thereof

- no partitioning!

#### Feedback about Anchors

- Experimentation across users or across the 2 policies permitted
- Alternatively, something like PRIO

#### Embedded content, workers, and miscellania

- Permission-policy default self. Affects the top-level directly
- no workers
- user-interaction-esque to mitigate navigation-tracking (Issue #1)

## Alternatives Considered

- PVT
- PST

## Accessibility, Internationalization, Privacy, and Security Considerations

- The whole thing is kind of a privacy discussion.
- No web-security, a11y, i18n considerations
- talk about how the security considerations from the architecture are addressed

## Stakeholder Feedback

- link to file issue for feedback, support, or disapproval

## References & Acknowledgments

- TK, many

