**Part 2: Anatomy of the ROT — How It Works Under the Hood**

**Where the ROT actually lives**

Before writing a single line of code, it's worth asking the question most COM tutorials skip entirely: where does the Running Object Table actually live?
The answer surprises most developers. The ROT is not a data structure inside your process. It's not a kernel object. It lives in rpcss.exe — the RPC Subsystem process — as a global, session-scoped table shared across all processes in the same Windows session.
When your code calls GetRunningObjectTable(), you don't get a pointer to the table itself. You get a proxy object — a COM interface that marshals every call across a local RPC channel (ALPC) to rpcss.exe. The actual moniker-to-object mapping never leaves that system process.
This has a practical consequence: every ROT operation is an IPC call. Register, Revoke, IsRunning, GetObject — all of them cross a process boundary. Cheap for occasional use; worth knowing if you're calling them in a tight loop.

**The API chain**
The full enumeration path will be looks like this:
````c
GetRunningObjectTable()
    └── IRunningObjectTable::EnumRunning()
            └── IEnumMoniker::Next()
                    └── IMoniker::GetDisplayName()
````

**Step 1: Get a handle to the ROT**
````c
HRESULT hr = 0;
IRunningObjectTable *pROT = 0;
hr = GetRunningObjectTable(0, &pROT);
````
GetRunningObjectTable()- the entry point. The first argument is reserved and must be zero. On success it gives you IRunningObjectTable — your proxy to rpcss.exe. Always check SUCCEEDED(hr) before proceeding; if the RPC channel to rpcss is broken, it fails.

**Step 2: Get an enumerator**
````c
cIEnumMoniker *pEnum = 0;
hr = pROT->lpVtbl->EnumRunning(pROT, &pEnum);
````
EnumRunning() returns IEnumMoniker — a standard COM enumerator over the monikers currently registered in the table. 
This is a snapshot: entries registered after this call won't appear, entries revoked before you iterate won't be missing from the snapshot either — the enumerator captures state at the moment of the call.

**Step 3: Iterate**
````c
cIMoniker *pMon = 0;
ULONG fetched = 0;

while (pEnum->lpVtbl->Next(pEnum, 1, &pMon, &fetched) == S_OK) {
    // process pMon
    pMon->lpVtbl->Release(pMon);
}
````
Next() func follows the standard COM enumerator contract: request items, get back how many were actually returned. 
Ask for one at a time — simpler error handling, no batch allocation. 

**Step 4: Decode the moniker**
````c
cIBindCtx *pCtx = 0;
CreateBindCtx(0, &pCtx);

LPOLESTR displayName = 0;
pMon->lpVtbl->GetDisplayName(pMon, pCtx, NULL, &displayName);

wprintf(L"%s\n", displayName);
CoTaskMemFree(displayName);
````

GetDisplayName() requires a bind context. IBindCtx- even though we're only reading a name, not actually binding. The bind context carries parameters that affect how monikers resolve; for display purposes we pass a default one via CreateBindCtx(0, ...).
Don't forget to free memory with callong CoTaskMemFree().
