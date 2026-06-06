# codemap-index

\# codemap - structural indexing





A generic, language-agnostic \*\*codebase map generator\*\*. It pre-computes a

token-cheap map of a repository so an LLM agent (Claude Code, Copilot, etc.) can

navigate the code without reading the whole tree at query time.



The same script runs on a Python, Kotlin, Java, or C repo with only

configuration changing — in the simplest case just `--lang`.



\---



\## How it works



codemap derives structure \*\*deterministically\*\* with tree-sitter and uses an LLM

only for the one thing a parser cannot give:



| Concern | Source | Cost |

|---|---|---|

| Exports, imports, declarations | tree-sitter (exact) | free |

| Internal deps + reverse \*\*used-by\*\* graph | symbol table from real imports | free |

| Tags (layers) | annotation / library / name signals | free |

| One-line \*\*purpose\*\* | docstring / KDoc / Javadoc first; LLM only if missing | cached |



This is the key difference from an "ask the LLM about every file" approach: the

expensive, error-prone work (especially \*who calls this file\*) is done by the

parser, accurately and for free. The LLM is asked for a single sentence, only

when the source has no doc comment of its own, and the answer is cached.



\### Output (under `--output`, default `codemap/`)



```

codemap/

├── INDEX.md                     # one linked line per file, grouped by directory

├── MOC.md                       # Map of Content: entry points + layers (by tag)

├── app/

│   ├── config.md                # per-file detail doc, mirrors source name

│   └── services/

│       └── auth.md

└── .codemap-manifest.json       # change-tracking + purpose cache (do not edit)

```



Each per-file doc contains: Purpose, Exports, Internal dependencies (Obsidian

wikilinks), External dependencies, Used by (wikilinks), Tags, and a preserved

`<!-- notes -->` block for hand-written notes that survive regeneration.



\### The navigation funnel



The two entry points are two zoom levels. An agent answers a question by:



1\. \*\*MOC.md\*\* — orient: which layer / entry point is relevant.

2\. \*\*INDEX.md\*\* — locate: pick the 1–3 files involved.

3\. \*\*`<file>.md`\*\* — read exports / deps / used-by without touching source.

4\. Source — only if the detail doc is insufficient.



Each step is cheaper than the one below; the agent stops as high as it can.



\---



\## Install



```bash

pip install tree-sitter

pip install tree-sitter-python        # add the grammars you need:

pip install tree-sitter-kotlin tree-sitter-java tree-sitter-c

pip install rich                      # optional: progress bar + panels

```



The grammar wheels bundle pre-compiled parsers (no runtime download, no

compiler). `rich` is optional — without it, output falls back to plain text.



\---



\## Usage



```bash

\# Python repo (defaults are Python-flavored, local Ollama for purposes)

python codemap.py /path/to/py-repo



\# Kotlin repo — only the language changes (Gradle scan auto-detected)

python codemap.py /path/to/kt-repo --lang kotlin



\# Mixed repo

python codemap.py /path/to/repo --lang java,c



\# Use OpenRouter for the purpose model

export OPENROUTER\_API\_KEY=sk-or-...

python codemap.py /path/to/repo --provider openrouter --model deepseek/deepseek-chat



\# Structure only, no LLM (fast, fully offline)

python codemap.py /path/to/repo --no-llm



\# Estimate generation tokens/cost before spending anything (writes nothing)

python codemap.py /path/to/repo --provider openrouter --dry-run \\

&#x20; --price-in 0.27 --price-out 1.10



\# Pre-commit: only resolve purpose for staged files

python codemap.py . --changed

```



\---



\## Configuration



| Flag | Default | Purpose |

|---|---|---|

| `root` (positional) | `.` | Repository root to scan. |

| `--lang` | `python` | Languages: `python`, `kotlin`, `java`, `c`. Comma-separated or repeatable. |

| `--output`, `-o` | `codemap` | Output directory (relative to root unless absolute). |

| `--scan` | `auto` | Source-root discovery: `auto`, `walk`, `gradle`. `auto` picks Gradle for JVM repos with a `settings.gradle(.kts)`. |

| `--provider` | `ollama` | Purpose-model backend: `ollama`, `openrouter`, `openai`. |

| `--model` | per-provider | Model id. Defaults: `gemma3:1b` / `deepseek/deepseek-chat` / `gpt-4o-mini`. |

| `--base-url` | per-provider | Override the endpoint URL. |

| `--api-key` | env var | API key. Falls back to `OPENROUTER\_API\_KEY` / `OPENAI\_API\_KEY`. |

| `--no-llm` | off | Skip the LLM; structure + doc comments only. |

| `--changed` | off | Resolve purpose only for git-staged files (pre-commit hook). |

| `--force` | off | Ignore the manifest cache; rebuild everything. |

| `--dry-run` | off | Estimate generation tokens/cost; calls no LLM and writes nothing. |

| `--price-in` / `--price-out` | — | USD per 1M input/output tokens, for the `--dry-run` cost line. |

| `--max-file-bytes` | `40000` | Skip files larger than this. |

| `--ignore-dir` | (see below) | Extra directory name to ignore (repeatable). |

| `--tag-signal` | per-language | Override a tag: `--tag-signal kotlin:#service=Service,UseCase` (repeatable). |

| `--internal-prefix` | — | Display hint for grouping external deps (repeatable). |



Default ignored directories: `node\_modules`, `.git`, `\_\_pycache\_\_`, `.venv`,

`venv`, `build`, `dist`, `out`, `target`, `coverage`, `.next`, `vendor`,

`.gradle`, `.idea`, and the various tool caches.



\### Provider defaults



| Provider | Endpoint | Default model | Key env var |

|---|---|---|---|

| `ollama` | `http://localhost:11434/api/generate` | `gemma3:1b` | — |

| `openrouter` | `https://openrouter.ai/api/v1/chat/completions` | `deepseek/deepseek-chat` | `OPENROUTER\_API\_KEY` |

| `openai` | `https://api.openai.com/v1/chat/completions` | `gpt-4o-mini` | `OPENAI\_API\_KEY` |



For a one-line purpose, a cheap/fast model is the right pick — set `--model` to

your lightweight tier (e.g. a DeepSeek or Gemini Flash variant on OpenRouter).



\---



\## Caching



`.codemap-manifest.json` is a versioned, content-hash-keyed cache. It stores

\*\*only\*\* the LLM purpose (the expensive, non-deterministic output) — structure

is recomputed each run because the used-by graph needs every file anyway.



A cached purpose is reused only when the file's content hash \*\*and\*\* the

provider+model \*\*and\*\* the prompt version all match. Changing the model, the

provider, or the prompt invalidates cached purposes automatically. The manifest

also tracks the file set so orphaned docs are deleted when their source is

removed or renamed.



\---



\## Agent integration (AGENTS.md)



Add this to the repo's `AGENTS.md` (or `.github/copilot-instructions.md`):



```markdown

\## Codebase navigation



This repo ships a pre-computed map under `codemap/`. Use it before reading source.



\- `codemap/MOC.md`   — architecture map: entry points + layers. Read first to orient.

\- `codemap/INDEX.md` — one line per file. Read to locate the file(s) a task touches.

\- `codemap/<path>/<file>.md` — per-file detail: purpose, exports, deps, used-by.



\### Answering a codebase question

1\. Read MOC.md to find the relevant layer.

2\. Scan INDEX.md to pick the 1–3 files involved.

3\. Open only those files' detail docs. Exports / Dependencies / Used-by are

&#x20;  extracted statically and are authoritative — trust them instead of grepping.

4\. Read raw source ONLY when the detail doc is insufficient.



\### Trust and staleness

\- Structural sections (exports, deps, used-by) are exact as of the last build.

\- Purpose is a model-generated summary — a guide; verify if a decision hinges on it.

\- If source behavior contradicts its doc, the source wins — the map is stale.

\- Never crawl the whole tree to answer a scoped question. Navigate, don't crawl.

```



\---



\## Supported languages



| Language | Extensions | Notes |

|---|---|---|

| Python | `.py` | Module-path resolution incl. relative imports (`from . import x`); docstrings as purpose. |

| Kotlin | `.kt` | FQN symbol resolution; KDoc; Gradle module discovery. Grammar is community-maintained — validate exports/used-by on a real repo. |

| Java | `.java` | FQN symbol resolution; Javadoc. |

| C | `.c`, `.h` | `#include`-based resolution; function definitions (not prototypes) captured as exports. |



Adding a language is a single `LanguageSpec` entry in the registry (node-type

sets + a symbol strategy) — no engine changes.



\---



\## Known limitations



\- The Kotlin tree-sitter grammar is less battle-tested than the others; spot-check

&#x20; a few modules against a known-good extractor before trusting it wholesale.

\- C header files expose function \*definitions\* as exports, not bare prototypes.

\- Tag signals are first-pass defaults; tune them per project via `--tag-signal`.

\- The OpenAI-compatible client sends the instruction in both the system and user

&#x20; message — a tiny token overhead, harmless but trimmable.



