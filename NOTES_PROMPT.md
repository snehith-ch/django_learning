# Prompt: Django Video Transcript → Notes

Use this as the standing instruction whenever I share a Django video transcript. I don't want to re-explain requirements each time — follow this exactly.

## Task

I'm learning Django from video tutorials and will give you transcripts of those videos. Convert each transcript into structured, beginner-friendly notes.

## Output format

- **Single HTML file** (one artifact), not multiple files, for the browsable version.
- **Separated by topic**: a sidebar navigation listing all topics; clicking a topic jumps/scrolls to its section; current section highlighted while scrolling.
- As I send more transcripts over time, **add new topics/sections to the same file** — don't create a new file each time. Sidebar just grows.
- Make it visually attractive: proper headings, syntax-highlighted code blocks, color-coded callout boxes, clean layout — not a plain text dump.
- **Also produce a Markdown (.md) file for each lecture/transcript I send**, in addition to updating the HTML artifact. One .md file per lecture (not per topic within a lecture, unless a lecture only covers one topic). Content parity with the HTML section(s) for that lecture — same depth, same gap-fill/researched labeling — just in plain Markdown instead of styled HTML. Save it locally as a file (don't just paste it in chat).

## Completeness — do not miss topics

- **Missing topics have been a problem before** — be careful and deliberate about this every time.
- Before writing notes, go through the transcript and list out every topic/subtopic/concept mentioned, even ones covered briefly or in passing. Use that list as a checklist while writing so nothing gets silently dropped.
- **It's fine if the notes get long** — I'd rather have complete, thorough notes than short ones. Don't compress or summarize away detail to keep length down.
- If the video mentions a concept without fully explaining it (e.g. references a term, a setting, a command flag), still give it its own note — gap-fill or researched, as below — rather than skipping it because "the video didn't cover it properly."

## Content requirements (per topic)

Write at a **strict beginner level**:
- Plain-language explanation before/alongside any technical term — never assume "you already know this."
- Every new term defined the first time it appears.
- Simple analogies where helpful to build intuition.
- Explain **what it is / what it does**.
- Explain **why this step exists / why it matters**.
- Full code examples from the video, cleaned up, with **line-by-line comments/explanation** wherever it's not obvious.
- **Add extra examples beyond what the video shows**, for every concept/topic — don't rely on the video's example alone. Include: a small standalone code example demonstrating the concept in isolation, a realistic use case showing when/why you'd reach for it, and (where relevant) sample input → output so the effect is concrete, not abstract. Label these clearly as additional examples (they fall under gap-fill unless pulled from docs, in which case they're researched).
- Mention **industry best practices** and common pitfalls for that topic.

## Handling gaps

Don't leave any topic incomplete. If the video under-explains or skips something:
- Add a **gap-fill** note — my own explanation, clearly visually labeled as an addition (not from the video).
- If it needs proper grounding, **research it** via Django's official documentation or other trusted sources, and add it as a clearly labeled **"Researched"** section.

Every piece of content should make it obvious whether it's:
1. From the video
2. Gap-fill (Claude's explanation)
3. Researched (from docs/web)

## Process to follow each time

1. Read the transcript fully and list every topic/subtopic/concept it touches (a checklist) — this is the completeness pass, don't skip it.
2. Write notes for each topic per the content requirements above, checking items off the list as you go.
3. Identify and fill any gaps (gap-fill or researched, as above).
4. Add/update the single HTML file: new section + new sidebar entry, consistent styling.
5. Write/save the per-lecture .md file with the same content.
6. Publish/update the artifact (same file, same link — redeploy in place).
7. Give a short wrap-up: what was covered directly, what was gap-filled, what was researched, and confirm against your checklist that nothing was missed — so I know what to double-check.
