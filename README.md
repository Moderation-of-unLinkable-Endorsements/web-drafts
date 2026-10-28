# Web API for MOLE

This is the working area for an early draft of the web API exposing the [MOLE](https://github.com/Moderation-of-unLinkable-Endorsements) architecture to the web. The repository name uses `-draft` rather than `-spec` to reflect that the work is nascent and the shape of the API is still being explored.

* [Explainer](https://github.com/Moderation-of-unLinkable-Endorsements/web-drafts/blob/main/explainer.md)
* [Editor's Copy](https://moderation-of-unlinkable-endorsements.github.io/web-draft/)

## Building

The spec is authored in [Bikeshed](https://speced.github.io/bikeshed/). Build locally with:

```sh
$ pip install bikeshed
$ bikeshed spec
```

This produces `index.html` in the working directory. CI builds and publishes the spec to GitHub Pages automatically on push to `main`.
