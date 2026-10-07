+++
draft = true
title = "Gemini Prompt"
date = "2026-10-02"
description = "Several prompts for the Gemini tool"

[taxonomies]
tags = ["gemini", "prompt"]
+++

## Notebook Settings Instructions

I am a student learning modern C via "Modern C: A Guide to the C23 Standard".

When I ask for a **Chapter Study Note**, please generate a **detailed, in-depth study note** as a **MARKDOWN FILE** following these strict rules.
The note should be thorough enough that I could learn the chapter from it without re-reading the book.
Aim for depth over brevity: a long, rich note is expected and desired. Do NOT summarize or compress.

1. **Exhaustive Coverage (Do Not Omit Anything):**
   - Go chronologically through **every section and subsection** of the chapter.
   - Cover **every** concept, term, definition, rule, caveat, and edge case the text presents.
   - Reproduce **every** explicit "Takeaway" verbatim (as a blockquote), then explain it. Do not skip any explicit "Takeaway".
   - Do not skip or merge topics to save space.

2. **Structure (for each section):**
   - Use a `## H2` header per chapter section and `### H3` headers per subsection or major concept.
   - Under each concept, include:
     - **What it is:** a clear explanation in 2-5 full sentences (not just fragments).
     - **Why it matters / how it works:** the reasoning, motivation, and underlying mechanics (e.g. what the compiler/CPU/standard does).
     - **Key details:** bullet points for rules, syntax, constraints, and edge cases.
     - **Pitfalls:** common mistakes, undefined/unspecified behavior, and gotchas mentioned or implied by the book.
     - **Connections:** links to related concepts in this or earlier chapters.
   - Define every technical term the first time it appears.
   - End each section with a short "Recap" of 2-4 bullets.

3. **Code Demonstrations:**
   - For each concept and takeaway, give a complete, compilable C23 example.
   - Prefer the book's own examples; if none exists, write a relevant one.
   - Add inline comments explaining line by line how the code demonstrates the specific concept or takeaway.
   - After each snippet, add a short "Expected behavior/output" note, and where useful a "What goes wrong if..." counter-example.

4. **Closing Sections:**
   - End with `## Summary` (a paragraph-level overview of the chapter) and `## Self-Check Questions` (8-12 questions that test understanding, with brief answers).

5. **Length and Quality:**
   - Do not stop early. If you run out of space, continue until every section is covered, and tell me where you stopped so I can say "continue".
   - Prefer explaining accurately over being short. Do not invent content not supported by the book or the C standard.

## Prompt for a Chapter Study Note

Help me write a detail Chapter Study Note for the "Chapter 12. The C memory model".
Please carefully read through the chapter and generate a detailed, in-depth study note.
**DO NOT** skip any section or subsection in the chapter.
