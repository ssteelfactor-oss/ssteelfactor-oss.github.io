---
title: Part 4: Reverse engineering rpcss.dll with Ghidra
date: 2026-06-05
---

# Part 4: Reverse engineering rpcss.dll with Ghidra

> *How we mapped the internal structure of the Running Object Table 
> without a kernel debugger and without running a single line of code 
> against a live process.*
---

## The problem

Parts 1–3 of this series worked entirely through the public COM API:
`GetRunningObjectTable` → `IEnumMoniker` → `IMoniker::GetDisplayName`.
Clean, documented, predictable.
But the public API is a proxy. Every call crosses an ALPC channel and
lands inside `rpcss.dll` — the actual host of the ROT. If we want to
read the table without touching `ole32.dll` at all, we need to know
what the data looks like on the other side of that channel.

This part is about finding out.
---
## Target: rpcss.dll
There is no `rpcss.exe` on disk. What Task Manager shows as `rpcss.exe`
is `svchost.exe` hosting `rpcss.dll` as a service. The DLL lives at:
C:\Windows\System32\rpcss.dll
Load it into Ghidra. Before doing anything else, grab the PDB:
```cmd
symchk C:\Windows\System32\rpcss.dll ^
    /s srv*C:\Symbols*https://msdl.microsoft.com/download/symbols
```
Point Ghidra at the downloaded `.pdb` via **Edit → Symbol Server Settings**.
Anonymous `FUN_18006XXXX` labels turn into real names. This changes everything.
---
## Step 1 — Finding the entry point
**Search → For Strings → filter: `ROT`**
First useful hit:
DEFINED  180136070  u_ROTFlags_180136070  unicode u"ROTFlags"
Press `X` on this string — one xref leads into a large function.
But the real find is in the **Symbol Table**:
CScmRotEntry::GetAllowAnyClient
Not `CROTEntry`. Not `CRunningObjectTableEntry`.  
**`CScmRotEntry`** — the SCM prefix means this is wired into the
Service Control Manager layer, deeper than COM documentation suggests.
Listing all symbols with prefix `CScmRot` confirms the full picture:
CScmRot::EnumRunning
CScmRot::GetEntryFromScmReg
CScmRot::GetTimeOfLastChange
CScmRot::IsRunning
CScmRot::Register
CScmRot::Revoke
CScmRotEntry::CScmRotEntry        ← constructor = real class
CScmRotEntry::GetProcessID        ← undocumented field
CScmRotEntry::IsValid

`GetProcessID` does not appear anywhere in the public API.
Every ROT entry silently stores the PID of whoever registered it.
---
## Step 2 — The table is a hash map, not a list
Inside `CScmRot::GetObject`:
```c
plVar13 = *(longlong **)(*(longlong *)(lpCriticalSection + 0x48) + uVar9 * 8);
```
Expanding this manually:
```c
longlong  tableBase  = rot->hashTable;   // CScmRot + 0x48
longlong  bucketAddr = tableBase + hash * 8;
CScmRotEntry *entry  = *(CScmRotEntry **)bucketAddr;
```
**The ROT is a hash table with 251 buckets (0xFB).**  
Each bucket is a singly-linked list of `CScmRotEntry` objects.
Hash function — rolling hash over moniker bytes:
```c
hash = ((hash * 3) ^ byte) % 0xFB;
```
---
## Step 3 — Mapping the structure from constructors
Constructors are better than documentation — they show every field
being initialized, with its exact offset.
`CScmRotEntry` inherits from a base class:
CScmRotEntry
└── CScmRotMgotEntryBase
"Mgot" = likely **M**arshaled **G**lobal **O**bject **T**able.

### CScmRotMgotEntryBase constructor
```c
*(undefined8 *)(this + 0x08) = 0;          // pNext = NULL
*(undefined4 *)(this + 0x10) = 1;          // refcount = 1
*(tagInterfaceData **)(this + 0x18) = p6;  // marshaled interface
*(_MnkEqBuf **)(this + 0x20) = p3;        // moniker eq buffer
*(CToken **)(this + 0x28) = p4;            // integrity level token
*(ushort **)(this + 0x30) = p5;            // display name (wide string)
this[0x40] = ... | p7;                     // flags, bit 0
*(ulong *)(this + 0x44) = p8;
*(ulong *)(this + 0x48) = p2;
*(ulong *)(this + 0x4c) = p1;

// manual AddRef on token — not COM, private refcount
if (p4 != NULL) {
    LOCK();
    *(int *)(p4 + 8) += 1;
    UNLOCK();
}
```
### CScmRotEntry constructor

```c
*(ulong *)(this + 0x50) = 0x746f7263;   // magic: ASCII "crot"
*(ulong *)(this + 0x54) = p5;
*(_FILETIME *)(this + 0x58) = *p4;      // last change timestamp
*(tagInterfaceData **)(this + 0x60) = p12;
*(ulong *)(this + 0x68) = p7;           // ROTFlags
```

### Complete layout
```c
struct CScmRotMgotEntryBase {
    void            *vtable;        // +0x00
    longlong        *pNext;         // +0x08  next in bucket chain
    DWORD            dwRefCount;    // +0x10  starts at 1
    tagInterfaceData *pIFaceData;   // +0x18  marshaled interface
    _MnkEqBuf       *pMnkEqBuf;    // +0x20  moniker comparison buffer
    CToken          *pToken;        // +0x28  integrity level token
    ushort          *pwszName;      // +0x30  display name (wide string)
    longlong         unk_38;        // +0x38
    BYTE             bFlags;        // +0x40  bit 0 = misc flag
    DWORD            dwField_44;    // +0x44
    DWORD            dwField_48;    // +0x48
    DWORD            dwField_4c;    // +0x4c
};

struct CScmRotEntry : CScmRotMgotEntryBase {
    DWORD            dwMagic;       // +0x50  0x746F7263 = "crot"
    DWORD            dwField_54;    // +0x54
    _FILETIME        ftLastChange;  // +0x58  registration timestamp
    tagInterfaceData *pIFaceData2;  // +0x60  second interface data
    DWORD            dwROTFlags;    // +0x68  bit 1 = AllowAnyClient
};
```
---
## Step 4 — Three findings worth highlighting

### 1. Magic signature `"crot"` at `+0x50`
Every valid `CScmRotEntry` in memory carries the bytes
`63 72 6F 74` at offset `0x50`.
This is almost certainly what `CScmRotEntry::IsValid` checks.
For a memory scanner, this is the anchor. You don't need to know
the head address of the hash table — you can find entries by
scanning for the signature pattern.
### 2. Display name at `+0x30` — no COM required
`pwszName` is a plain wide string. The same string you'd normally
get from `IMoniker::GetDisplayName` after constructing a bind
context and crossing the ALPC boundary.

Reading it directly via `ReadProcessMemory` requires no COM calls,
no `ole32.dll`, no ALPC round-trip.

### 3. CToken uses private reference counting
```c
LOCK();
*(int *)(param_4 + 8) += 1;
UNLOCK();
```
Not `IUnknown::AddRef`. A manual interlocked increment on a private
field at `CToken + 0x08`. Tokens are shared across ROT entries from
the same process using an internal mechanism entirely separate from
COM lifetime management.
---
## Step 5 — Security model visible in GetObject
Walking `CScmRot::GetObject` surfaces three access checks in sequence:
```c
RotEntryIsTrusted(entry, callerToken)
RotMgotEntryIsAccessible(entry, callerToken, ...)
CToken::IsTokenILLower(entry->pToken, candidate->pToken, ...)
```
When multiple entries match the same moniker hash, the entry with
the **higher integrity level wins**. A sandboxed process cannot
shadow a medium-integrity registration.
AppContainer entries are filtered separately:
"Disregarding ROT entry registered by AppContainer"
The ROT has a real, layered access control model.
None of this appears in the public API documentation.
---
## What we have now
| Field | Offset | Source |
|---|---|---|
| pNext | +0x08 | base ctor |
| pIFaceData | +0x18 | base ctor |
| pMnkEqBuf | +0x20 | base ctor |
| pToken | +0x28 | base ctor |
| pwszName | +0x30 | base ctor |
| dwMagic "crot" | +0x50 | entry ctor |
| ftLastChange | +0x58 | entry ctor |
| dwROTFlags | +0x68 | GetAllowAnyClient |
| hashTable ptr | CScmRot +0x48 | GetObject |
| bucket count | 0xFB (251) | GetObject |

---

## Next

Part 5 puts this to use: a C tool that opens the svchost hosting
`rpcss.dll` via `SeDebugPrivilege`, locates the hash table at
`CScmRot + 0x48`, walks all 251 buckets, and reads `pwszName`
directly from each `CScmRotEntry` — without a single call to
`ole32.dll`.

---

*← Part 3: Direct ALPC — not yet published. See the [series index](index.md).*
