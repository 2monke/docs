# 2monke docs

Architecture and design documentation for the 2monke platform.

This repository is the **content leaf** of the project graph. It is consumed as a
git submodule by the application; it depends on nothing, and it contains no
submodules of its own.

```
2monke/docs  ←  you are here. Documentation only, no submodules.
   ↑
   │ submodule
2monke-app ─── HTTP (help service) ───▶ discord-gateway
```

Neither the gateway nor the help service mounts this repository. The gateway
never read it, and the help service keeps its own corpus (see below).

## Layout

- `*.md` — architecture and design documents. Paths are kept identical to the
  historical `docs/` directory in the application so that existing code comments
  and references continue to resolve.
- `branding/` — brand assets, source logo files and palette studies.

## Nothing here ships

This repository is **reference material**. Nothing in it is read at request time,
traced into a build, or served to a customer — the application's only remaining
relationship with it is that code comments cite these files by path.

That has one practical consequence: editing anything here cannot break
production. There is no build gate and no rebuild to trigger.

## The customer help corpus is not here

It used to be, at `help/`, and the application read it with `fs` at request time.
That put the articles in the deployment artifact, so fixing a sentence meant a
commit here, a submodule bump in the app, and a full rebuild.

The corpus now lives with the service that serves it, in
[`2monke/help-service`](https://github.com/2monke/help-service) under `corpus/`,
grouped by audience (`buyers/`, `artists/`, `shared/`). Editing an article is one
deploy of that worker and nothing else.

Two things about that corpus are worth knowing even from here, because they are
easy to get wrong when writing an article:

1. Each article carries YAML frontmatter. Articles are only served to customers
   when `status: published`; `draft` and `generated` are filtered out.
2. Audience is resolved from frontmatter `audience`, falling back to the
   directory name (`buyers` → buyer, `artists` → artist, otherwise shared).

Changing an article's frontmatter, or moving it between audience directories,
changes what customers are told.

## Consuming this repository

As a submodule, mounted at `docs/`:

```bash
git submodule update --init --recursive
```

A checkout without submodules leaves `docs/` empty. That no longer breaks
anything at runtime — code comments that cite a path here simply have nothing to
point at.
