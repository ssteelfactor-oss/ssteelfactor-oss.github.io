---
title: The Running Object Table
---

# The Running Object Table

The ROT is a system-wide registry that holds references to live COM objects.
It is the mechanism behind `GetObject()` in VBScript, behind Visual Studio's
automation model, and behind several inter-process patterns in Windows that use
neither sockets nor named pipes. It is also one of the least examined parts of
COM's supporting infrastructure.

This series works through it in plain C, starting from the public API and ending
inside `rpcss.dll`.

| Part | Subject |
| --- | --- |
| [Part 1](part-1-inside-the-rot.md) | What the ROT is, and what it is not |
| [Part 2](part-2-under-the-hood.md) | Where the table actually lives, and what every call costs |
| Part 3 | Decoding monikers *(not yet published)* |
| [Part 4](part-4-reversing-rpcss.md) | Recovering the internal layout with Ghidra |

!!! note "Part 3"

    Part 3 is referenced by Part 4 but is not in the repository. Add
    `part-3-*.md` to this folder and a line to `nav:` in `mkdocs.yml` when it is
    ready.
