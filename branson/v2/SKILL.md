---
name: compile-docs
description: Synthesize documentation from multiple sources into one distilled Markdown reference with worked code examples. Use when the user wants to compile or condense docs, combine several documentation pages or a whole doc site into a single easy-to-read .md, or expand a doc's includes and links into a complete standalone guide.
---

# Compile docs

Pull scattered documentation into one **synthesized** `.md`: a single reference that is **distilled** (shorter and clearer than its sources) yet more complete than any one of them, with a working code example for every concept the sources demonstrate. Synthesis gathers many sources and reconciles them into one; distillation removes bulk while concentrating substance. Neither is summarization — no technical detail is discarded.

## Process

1. **Map the sources.** Start from the primary doc, then widen: follow links to the referenced pages, expand server-side includes to their real content (e.g. a `{% data reusables.* %}` tag → fetch the file it names from the doc's source repo, or fetch the rendered page where the include is already resolved), and locate the canonical examples, quickstart, and API reference for the topic. Fetch each with WebFetch for a URL or Read for a local file. Done when you hold the full text of every in-scope source, every include resolved to its actual content or explicitly listed as unresolvable.

2. **Reconcile across sources.** Merge passages from different sources that explain the same thing into one statement. Where sources agree, state it once; where they conflict — version drift, contradicting defaults — keep the most authoritative and flag the discrepancy inline. Copy every technical fact verbatim: API signatures, parameter names and types, exact commands, config keys, version numbers, and code blocks are reproduced character-for-character, since paraphrasing changes their meaning. Done when no concept is stated twice and every conflict is resolved or flagged.

3. **Distill the substance.** Set aside navigation, breadcrumbs, cookie and promo banners, sidebars, footers, edit-this-page links, and marketing copy that carries no instruction. Rewrite the remaining prose shorter and more direct, and order each concept before its details.

4. **Anchor every concept with a code example.** For each concept, procedure, or API that any source demonstrates with code, carry a self-contained, runnable example into the output — copied verbatim, and where a source gives only a fragment, assembled from that source into a complete example. Attribute each example to the source it came from. Done when every concept a source shows in code carries a code example here.

5. **Structure for comprehension.** Open with a `Sources:` block listing every source URL or path used, then a one-paragraph TL;DR, then an at-a-glance list of the concepts or steps. Follow with a section per concept — its reconciled facts, then its code example — under a clear heading hierarchy.

6. **Write and verify.** Write the result to the target `.md` (default `outputs/branson/v2/<doc-slug>.md`; override if the user names a path). Then confirm all hold: the file is leaner than the combined sources yet more complete than any single one; every technical fact appears; every source in the `Sources:` block is actually drawn on; every concept a source demonstrates in code carries an example; every conflict is flagged. Fix any gap before finishing.

## What earns a place

Keep: concepts, procedures, every code example, exact command and API detail, gotchas and caveats, the constraints that make the docs correct (required versions, prerequisites, limits), cross-source reconciliations, and the source attribution for each example.

Cut: navigation and site chrome, marketing and positioning prose, changelog and legal boilerplate unless the user asked for it, and any sentence that restates one a source already made — across sources as well as within one.
