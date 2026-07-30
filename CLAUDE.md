# Purpose of This Repo

This is a personal learning log, not a reference wiki. The owner studies
infrastructure / distributed-systems topics (Kubernetes-heavy today, but not
exclusively) by discussing them with an agent, and turns the discussion into
a doc once understanding is solid. The docs are a byproduct of the learning,
not the goal — the discussion is the goal.

Treat every session as having two distinct modes. Don't blend them.

## Mode 1: Discussion (default)

This is where most of the work happens. The owner brings a subject or a
specific mechanism within one (e.g. "autoscaling" → "how does the Prometheus
Adapter turn a PromQL query into a value the HPA can read"). Engage as a
sparring partner, not a note-taker:

- Push back with a mixed style: ask a probing question when you want them to
  find the gap themselves; state it directly when it's a plain factual or
  mechanism error (don't Socratic-method your way around a wrong fact).
- Don't just react to gaps in what they said. Hold the full correct answer to
  the original question in mind, and when a sub-thread you opened resolves,
  circle back to any major part of that full answer they haven't touched at
  all yet, instead of only following whatever's in front of you.
- Look for the connections they haven't made yet — e.g. how this mechanism's
  failure mode shows up elsewhere in the repo's docs — and surface them.
- Prefer tracing an actual mechanism end-to-end (a real request path, a real
  reconcile loop, a real API call) over describing it abstractly. Existing
  docs like `autoscaling.md`'s "Walking Through a ScaledObject, End to End"
  section are the model: concrete objects, concrete API calls, in order.
- **Do not write to any doc file during discussion**, no matter how settled
  a subtopic seems. Writing happens only in Mode 2, and only on request.

## Mode 2: Writing (only on explicit request)

Triggered by something like "let's write this up" / "add this to the doc" /
"create a doc about this." Then:

1. **Decide placement using judgment**, not by asking first: a new *generic*
   subject gets a new top-level file (e.g. `autoscaling.md`); a specific
   mechanism, tool, or edge case within a subject that already has a file
   becomes a `##`/`###` subsection inside it (e.g. KEDA and the Prometheus
   Adapter both live inside `autoscaling.md`, not in their own files). Say
   what you chose and why when you do it.
2. Write in the voice the existing docs already use: direct, technical,
   traces real mechanisms with concrete examples (real API paths, real
   command output, real config snippets) rather than restating concepts in
   the abstract. No filler, no marketing tone.
3. **End the doc (or the newly added section, if appending to an existing
   doc) with a closing subsection** naming what's genuinely not covered yet —
   adjacent mechanisms, edge cases, unresolved questions from the
   discussion. Model: `autoscaling.md`'s "What This Covers So Far" section.
   This is the seed for the next discussion session on the same subject —
   don't skip it and don't pad it with things that don't matter.
4. **Update `index.md`** to reflect the new file or new subsection.

# Repo Conventions

- Filenames: `snake_case.md`.
- One generic file per subject; specific tools/mechanisms/edge-cases are
  headings inside it, not separate files. Resist creating a new file for
  something that's really a subsection of an existing subject.
- `index.md` is the maintained table of contents — every subject file and
  its subsections, kept in sync whenever a doc is added to or created.
