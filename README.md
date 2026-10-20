# Wepwawet

**A Standard For Extension on Open API Standard for implementing acoustic / binary extensions.**

An app adds one section, `x-wepwawet`, to the OpenAPI document it already has. A voice system reads
it and knows what the app is, what its things are called, which words ask for which command, and
how to say truthfully what happened, with no code written for that app. Tools that don't know the
section ignore it.

Requests the house's own apps can answer are answered there, quickly and on the house's own
network; only what really needs a large cloud model goes on to one.

- **The standard:** [wepwawet-standard.md](wepwawet-standard.md): why voice is different, thirteen
  expectations (each with what went wrong and what to do), and the `x-wepwawet` section.
- **A worked example:** the weather brain, an app that takes the words as said (expectation 11).

Version 1, October 2026. Expectation 13 and `weight` are drafts.

## Following it

1. Read the expectations; each is a short **why** and **what to do**.
2. Add an `x-wepwawet` section to your app's OpenAPI document (format and an example in the
   standard).
3. Put your real version in `info: version:` and change it with every release.

## License

The standard is © 2026 coderpigsfly, licensed under
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) (see `LICENSE`): share and adapt it,
with credit.
