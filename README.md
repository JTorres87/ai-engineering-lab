# AI Engineering Lab

A series of hands-on AI engineering projects: agents, fine-tuning, evals and RAG. I publish a new project every week or so. Each one is a small, working system in its own repo, with a write-up here on what I built, what broke and what I learned.

Every project ships as a working **base** (v0.1.0) and then grows in tagged releases, so you can see how it develops: the first version, then logging, evals, reliability and cost work layered on top.

## Projects

| # | Project | What it is | What it shows | Latest |
|---|---|---|---|---|
| 01 | [tool-agent](https://github.com/JTorres87/tool-agent) | A CLI agent that helps you choose an LLM and plan what it will cost | A hand-written tool-calling loop; typed tools over a SQLite model catalog; cited answers checked in code and by a second model; a multi-turn chat that rolls back failed turns | v0.1.0 |

## Write-ups

- [01: tool-agent](notes/01-tool-agent.md): the raw agent loop, the two answer checks, and three things the experiments taught me

## How each project is built

- **Working code first.** Every project runs end to end and has tests that don't need an API key.
- **Measure, don't guess.** When I change something, I run it before and after and keep the numbers in the write-up.
- **Small releases.** A base you can read in one sitting, then features in versioned releases with notes.

## License

[Apache 2.0](LICENSE)
