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
