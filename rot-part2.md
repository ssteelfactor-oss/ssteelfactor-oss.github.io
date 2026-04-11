**Part 2: How It Works Under the Hood**

Before diggin into, it's useful to consider a question that most COM tutorials never address: where does the Running Object Table actually reside?
The ROT isnt a data structure inside the calling process, nor is it a kernel object. 

It exists as a global, session-scoped table inside rpcss.exe, the RPC Subsystem process, and is shared by all processes running in the same Windows session.
When code calls GetRunningObjectTable() func, it does not receive a direct pointer to the table. Instead, it receives a proxy object that implements the IRunningObjectTable interface. Every method invoked on this proxy is marshaled across a local RPC channel (ALPC) to rpcss.exe. The actual mapping of monikers to live objects never leaves that system process.

As a result, every ROT operation: Register, Revoke, IsRunning, GetObject, etc is an inter-process call. The overhead is insignificant for occasional use, but it must be taken into account when these functions are invoked inside tight loops or performance-critical code.

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
Entry point - GetRunningObjectTable(). 1-argument is reserved and must be zero. On success it gives you IRunningObjectTable, proxy to rpcss.exe. Always check SUCCEEDED(hr) before proceeding; if the RPC channel to rpcss is broken, it fails.

**Step 2: Get an enumerator**
````c
IEnumMoniker *pEnum = 0;
hr = pROT->lpVtbl->EnumRunning(pROT, &pEnum);
````
EnumRunning() returns IEnumMoniker, a standard COM enumerator over the monikers currently registered in the table. 
This is a snapshot: entries registered after this call won't appear, entries revoked before you iterate won't be missing from the snapshot either - enumerator captures state at the moment of the call.

**Step 3: Iterate**
````c
IMoniker *pMon = 0;
ULONG fetched = 0;

while (pEnum->lpVtbl->Next(pEnum, 1, &pMon, &fetched) == S_OK) {
    // process pMon
    pMon->lpVtbl->Release(pMon);
}
````
Next() func follows the standard COM enumerator contract: request items, get back how many were actually returned. 
Ask for one at a time it's simpler error handlingn. 

**Step 4: Decode the moniker**
````c
IBindCtx *pCtx = 0;
CreateBindCtx(0, &pCtx);

LPOLESTR displayName = 0;
pMon->lpVtbl->GetDisplayName(pMon, pCtx, NULL, &displayName);

wprintf(L"%s\n", displayName);
CoTaskMemFree(displayName);
````

GetDisplayName() requires a bind context. IBindCtx- even though we're only reading a name, not actually binding. The bind context carries parameters that affect how monikers resolve; for display purposes we pass a default one via CreateBindCtx(0, ...).
Don't forget to free memory with callong CoTaskMemFree().
