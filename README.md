<!-- markdownlint-disable MD033 MD041 -->
<a id="readme-top"></a>

<div align="center">

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="docs/assets/icons/Lore_TM_White_V1.svg">
  <img alt="Lore — open source version control by Epic Games" src="docs/assets/icons/Lore_TM_Black_V1.svg" width="220">
</picture>

<h1>Lore</h1>

<p><strong>Next-generation open source version control.</strong></p>

<p>
  <a href="https://github.com/EpicGames/lore/releases">Download Lore</a>
  &nbsp;&middot;&nbsp;
  <a href="docs/tutorials/quickstart.md">Quickstart</a>
  &nbsp;&middot;&nbsp;
  <a href="docs/README.md">Read the docs</a>
  &nbsp;&middot;&nbsp;
  <a href="https://discord.gg/QYbNFVFv">Join the conversation</a>
</p>

<p>
  <a href="LICENSE"><img alt="License: MIT" src="https://img.shields.io/badge/License-MIT-informational"></a>
  <a href="https://discord.gg/QYbNFVFv"><img alt="Join the Lore Discord" src="https://img.shields.io/badge/Discord-join%20the%20conversation-5865F2"></a>
  <img alt="Built with Rust" src="https://img.shields.io/badge/Built%20with-Rust-orange">
</p>

</div>

<details>
  <summary>Table of contents</summary>

- [About Lore](#about-lore)
- [Get started with Lore](#get-started-with-lore)
- [Overview](#overview)
- [Lore's architecture](#lores-architecture)
- [Lore's repositories](#lores-repositories)
- [Fully open source](#fully-open-source)
- [Contributing](#contributing)
- [License](#license)
- [Contact and community](#contact-and-community)

</details>

## About Lore

Lore is a next-generation open source version control system, maintained by Epic Games and designed for the unprecedented scalability of both data and teams. Lore is optimized for projects that combine code with large binary assets — including games and entertainment — and caters for developers and artists alike. For the problem Lore solves, why existing version control falls short at this scale, and the rationale and design that follow, [read the technical design](docs/explanation/technical-design.md).

<sub><a href="#readme-top">(back to top)</a></sub>

## Get started with Lore

- **Quickstart —** install Lore and make your first commit by following the [quickstart guide](docs/tutorials/quickstart.md).
- **Read the docs —** delve into Lore's ethos and architecture in the [Lore documentation](docs/README.md).
- **Have questions? —** the [FAQ](docs/faq.md) covers licensing, supported platforms, production readiness, and how Lore compares to other version control systems.
- **Join the conversation —** chat with us and our community on [Discord](https://discord.gg/QYbNFVFv).

<sub><a href="#readme-top">(back to top)</a></sub>

## Overview

- **Easy setup, on-demand scalability.** Lore scales along every axis — file count, file size, history depth, branch count, and concurrent users — so a workspace grows with the work, not the whole repository.
- **Fast and efficient processes.** Performance is a first-class priority. Operations on a multi-million-file repository stay interactive, and operations on a multi-gigabyte file stream rather than materialize the whole thing.
- **Free branching.** Branches are first-class citizens. Creating a branch and switching between branches are lightweight, fast operations.
- **History you can trust.** Every fragment is identified by its BLAKE3 hash and the revision graph is hash-chained, so tampering and corruption are detectable end-to-end.
- **Intuitive interface.** Lore is designed to be approachable for developers and artists alike, including through the command line.
- **Full-surface API.** Everything Lore does is reachable through its API, with bindings across C/C++, C#, Rust, Go, Python, and JavaScript.

<sub><a href="#readme-top">(back to top)</a></sub>

## Lore's architecture

Lore is a centralized, content-addressed version control system. It represents repository state as Merkle trees and an immutable revision chain, and takes a binary-first approach to storage with deduplication and sparse, on-demand hydration that holds up at scale. For the full model — on-disk formats, chunking internals, and the mechanics of the Merkle tree — read [the technical design](docs/explanation/technical-design.md).

<details>
  <summary>Architecture at a glance</summary>

- **Content-addressed storage.** Every piece of content is stored once, keyed by its BLAKE3 hash.
- **Immutable revision chain.** Revisions form a hash-chained graph that makes history tamper-evident.
- **Chunked storage for large files.** Large files split into chunks, so a small edit re-uploads only the changed chunks.
- **On-demand hydration and sparse workspaces.** A working tree materializes only the subset you ask for; the rest is fetched on demand.
- **Centralized service with caching.** A single logical source of truth, served from the nearest cache and replica tiers.
- **Lightweight branches and fast switching.** Branches are names in a small mutable store, so branching and switching cost almost nothing.

</details>

<sub><a href="#readme-top">(back to top)</a></sub>

## Lore's repositories

Lore spans a family of repositories: the core library, server, and CLI in this repository, plus a software development kit (SDK) for each supported language.

| Repository | Description | Link |
| --- | --- | --- |
| **Lore Library, Server & CLI** | The core Lore library, the Lore Server, and the Lore CLI. You are here. | [View on GitHub](https://github.com/EpicGames/lore) |
| **JavaScript SDK** | The JavaScript binding for the Lore API. | [View on GitHub](https://github.com/EpicGames/lore-js) |
| **Python SDK** | The Python binding for the Lore API. | [View on GitHub](https://github.com/EpicGames/lore-python) |
| **C# SDK** | The C# binding for the Lore API. | [View on GitHub](https://github.com/EpicGames/lore-dotnet) |
| **Go SDK** | The Go binding for the Lore API. | [View on GitHub](https://github.com/EpicGames/lore-go) |

<sub><a href="#readme-top">(back to top)</a></sub>

## Fully open source

We believe a truly open ecosystem is built collectively using open standards. Lore is fully open source under an [MIT license](LICENSE), and we invite you to build the version control system of the future in the open. See [CONTRIBUTING.md](CONTRIBUTING.md) to get involved.

<sub><a href="#readme-top">(back to top)</a></sub>

## Contributing

Contributions of every kind are welcome — code, documentation, bug reports, and reviews. Start with [CONTRIBUTING.md](CONTRIBUTING.md) for the development workflow, then read the [Code of Conduct](CODE_OF_CONDUCT.md) and the project [governance model](GOVERNANCE.md). New to the codebase? The [`good-first-issue`](https://github.com/EpicGames/lore/labels/good-first-issue) label is a good place to start.

<sub><a href="#readme-top">(back to top)</a></sub>

## License

Lore is released under the MIT License. See [LICENSE](LICENSE) for the full text. Copyright (c) 2026 Epic Games, Inc.

<sub><a href="#readme-top">(back to top)</a></sub>

## Contact and community

- **Discord —** chat with the team and community on [Discord](https://discord.gg/QYbNFVFv).
- **GitHub Issues —** report bugs and request features through [GitHub Issues](https://github.com/EpicGames/lore/issues).

<sub><a href="#readme-top">(back to top)</a></sub>
