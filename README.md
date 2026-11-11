# 2monke docs

Shared documentation and content for the 2monke platform.

This repository is the **content leaf** of the project graph. It is consumed as a
git submodule by the application and by the Discord gateway; it depends on
neither, and it contains no submodules of its own.

```
2monke/docs  ←  you are here. Content only, no submodules.
   ↑            ↑
   │            │
2monke-app   discord-gateway
```

## Layout

- `*.md` — 57 architecture and design documents. Paths are kept identical to the
  historical `docs/` directory in the application so that existing code comments
  and references continue to resolve.
- `branding/` — brand assets, source logo files and palette studies.
- `help/` — 12 customer-facing help articles, grouped by audience:
  - `help/buyers/`
  - `help/artists/`
  - `help/shared/`

## `help/` is runtime content, not just documentation

The help articles are read from disk at request time by the application's help
assistant (`src/lib/ai/help/corpus.ts`), which serves both the customer help
widget and the Discord assistant.

Two consequences:

1. Each article carries YAML frontmatter. Articles are only served to customers
   when `status: published`; `draft` and `generated` are filtered out.
2. Audience is resolved from frontmatter `audience`, falling back to the
   directory name (`buyers` → buyer, `artists` → artist, otherwise shared).

Changing an article's frontmatter or moving it between audience directories
changes what customers are told. Treat `help/` as production content.

## Consuming this repository

As a submodule, mounted at `docs/`:

```bash
git submodule update --init --recursive
```

Clone the application or gateway with `--recursive`, or run the command above
after a plain clone, otherwise `docs/help/` will be empty and the help assistant
will answer from an empty corpus without raising an error.
