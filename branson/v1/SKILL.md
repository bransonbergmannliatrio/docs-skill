---
name: compile-docs
description: Distill official documentation into a lean, readable Markdown file. Use when the user wants to compile, condense, or clean up docs, turn a documentation page or site into an easier-to-read .md reference, or make verbose official docs comprehensible.
---

# Compile docs

Turn official documentation into a **distilled** `.md`: shorter and clearer than the source, but faithful to every technical fact in it. Distillation removes bulk (chrome, marketing, repetition) while concentrating substance — it is not summarization, which discards detail.

## Process

1. **Locate the source.** Fetch the doc — WebFetch for a URL, Read for a local file. If the source is an index or landing page, list the sub-pages it links and fetch the ones carrying the actual content. Done when you hold the full text of every page in scope.

2. **Strip the chrome.** Set aside navigation, breadcrumbs, cookie and promo banners, "on this page" sidebars, footers, edit-this-page links, repeated boilerplate, and marketing copy that carries no instruction. What remains is substance.

3. **Distill the substance.** Rewrite prose shorter and more direct, merge passages that explain the same thing, and order each concept before its details. Copy every technical fact verbatim — API signatures, parameter names and types, exact commands, config keys, version numbers, and code blocks are reproduced character-for-character, since paraphrasing changes their meaning.

4. **Structure for comprehension.** Open with a `Source:` line linking the original URL(s), then a one-paragraph TL;DR. For a procedural doc, add an at-a-glance list of the steps or key entities before the detail. Use a clear heading hierarchy throughout.

5. **Write and verify.** Write the result to the target `.md` (default `outputs/branson/v1/<doc-slug>.md`; override if the user names a path). Then confirm both hold: the file is meaningfully shorter than the source, **and** every technical fact from the source appears in it. Fix any gap before finishing.

## What earns a place

Keep: concepts, procedures, every code example, exact command and API detail, gotchas and caveats, and the constraints that make the docs correct (required versions, prerequisites, limits).

Cut: navigation and site chrome, marketing and positioning prose, changelog and legal boilerplate unless the user asked for it, and any sentence that restates one already kept.
