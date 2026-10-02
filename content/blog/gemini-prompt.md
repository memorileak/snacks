+++
draft = true
title = "Gemini Prompt"
date = "2026-10-02"
description = "Several prompts for the Gemini tool"

[taxonomies]
tags = ["gemini", "prompt"]
+++

## Generating study notes from a chapter in a book

Given that I am a student learning modern C via "Modern C: A Guide to the C23 Standard".
I am reading "Chapter 12. The C memory model".
Please help me by generating a study note in a **MARKDOWN FILE** following these strict rules:

1. **Exhaustive Coverage (Do Not Omit Sections):**
   - Iterate chronologically through **every single section and subsection** of the chapter.
   - Extract **every** major concept presented in the text.
   - Extract **every** explicit "Takeaway" rule.
   - **DO NOT SKIP** any major concepts or takeaways to save space.

2. **Formatting & Style:**
   - Format each section in the chapter as a `## H2` header.
   - Conciseness applies to your **writing style**, not the **content coverage**. Use brief, direct bullet points for the explanations, but ensure 100% of the chapter's sections are covered.

3. **Code Demonstrations:**
   - For each major concept and takeaway, write a C example code snippet demonstrating it.
   - Prefer taking the examples directly from the book if possible. If no example exists for a specific takeaway, generate a highly relevant, compilable C snippet.
   - The code must contain inline comments explaining exactly how it maps to the specific takeaway it demonstrates.
