---
title: Home
---

# Windows internals, from first principles

Notes from taking Windows apart in plain C - no ATL, no MFC, no framework
abstractions between the code and the mechanism. Most of what follows started as a
question the documentation did not answer, and ended in a disassembler.

<div class="grid cards" markdown>

-   __The Running Object Table__

    ---

    A system-wide registry of live COM objects that lives inside `rpcss.exe`.
    Four parts: what it is, how the proxy reaches it, and how its internal layout
    was recovered from `rpcss.dll` without a kernel debugger.

    [Read the series](running-object-table/index.md)

-   __Apartments and the message pump__

    ---

    Why a WinForms or WPF thread can deadlock on a mechanism nobody wrote,
    nobody sees, and apparently nobody uses. The answer involves DDE
    handling from 1993.

    [Read the article](apartments/com-sta-deadlock.md)

</div>

## About

I research Windows internals COM, DCOM, RPC, marshaling and build small,
dependency-free tools in C to test what the research claims. Everything here is
written against live systems and verified in a debugger before it is published.

Code lives on [GitHub](https://github.com/ssteelfactor-oss).
