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
none of them are very good for users. Tracking users uniquely across the web is
a violation of their privacy. CAPTCHAs introduce a cost to many visitors
to a site, particularly to users that use privacy-enhancing user agents. Client
integrity checks close off the Web to many users.

This work proposes a new option: Moderated Endorsements. This proposal is the
user agent deployment of emerging work at the IETF,
[MoLE](https://moderation-of-unlinkable-endorsements.github.io/internet-drafts/),
which is an attempt to solve the
[PACT problem statement](https://github.com/antifraudcg/proposals/issues/22),
from the W3C Anti-Fraud CG.

When a user visits that is confident the user is genuine, that site may 
_endorse_ that user. Later, when the user visits a different site for the first
time, the user may redeem their endorsement with a service that the site uses 
for access control, called a _moderator_. The user proves that they (1) were
endorsed by one of a trusted set of sites and (2) is following a policy set 
between the moderator and site, like a rate limit.

### More practically, from where the user sits

Suppose I'm a user browsing the web and I come to a website that is 
uncertain whether my traffic is from a person using their website or a bot
scraping content. Currently on the Web, one common approach is for the website
to present me a CAPTCHA to either stop a web scraper or force them to burn
CAPTCHA solving API credits. This wastes the time of the odd misclassified user,
like myself, but doesn't prevent me from using the site. Other common approaches
to deter bots are similarly painful for real people: additional friction or 
outright blocks.

Instead, imagine if one of the sites I've already visited could endorse me as
not being abusive to the site that might otherwise show me a CAPTCHA. Email
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


### The Incomplete MoLE Dependency

This approach builds on the
[MoLE Architecture](https://datatracker.ietf.org/doc//draft-jms-mole-architecture),
though the work there isn't done. It's
[new protocols](https://datatracker.ietf.org/doc/draft-jms-mole-protocols)
are needed to communicate only the information we want with our messages and the
[HTTP transport](https://datatracker.ietf.org/doc/draft-jms-mole-http-transport)
gives us the language needed for us to send those messages between clients and
servers. 

Even though it is early, there is enough structure there to make it worth 
discussing the deployment-specific considerations that will have to be made if 
MoLE is to make its way into user agents.

## How does this work?

The power of Moderated Endorsements is that we can share information across
sites, so to explore it fully, we have to imagine a few sites willing to work
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
href="/endorsement_uri" > to a document's head.

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
header. The validation result is returned to the caller and is used to update
the credential state.

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

This API may be extended to include a model where the Javascript execution
context is untrusted by the website to issue challenges or handle the
presentation response. This would be as simple as determining _which_ Fetches
process `WWW-Authenticate: Mole` response headers. This could be determined by
the 
[Request's destination](https://developer.mozilla.org/en-US/docs/Web/API/Request/destination).

TODO: diagram with Site

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

