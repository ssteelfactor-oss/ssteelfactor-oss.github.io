**Inside the Running Object Table: COM's Hidden Registry of Live Objects**

**Introduction:** Running Object Table, part I

COM is one of those technologies you think you understand until you start asking inconvenient questions, just some like: "Huhmmm... what the hell is that". Most developers know the basics — interfaces, CoCreateInstance, reference counting — but COM has always carried more infrastructure than its public API suggests. One of the least-examined pieces of that infrastructure is the **Running Object Table**, or ROT.
The ROT is a system-wide registry of live COM objects. Aint classes or factories — actual instantiated objects that have explicitly announced themselves as running and reachable. Think of it as the difference between a phone book and a list of people who are currently answering their phones. When a COM server wants to say "I'm here, find me by this name", it registers a moniker in the ROT. When a client wants to bind to something that's already running — rather than spinning up a new instance — it looks there first.
This matters more than it sounds. The ROT is the mechanism behind GetObject() in VBScript, the reason Visual Studio exposes its automation object to external scripts, and the reason certain IPC patterns in Windows work without any explicit socket or pipe setup. It is also, as let's see, a surface that is easy to misunderstand and occasionally to abuse.
In this post i'll try to explore the ROT from the ground up — in C, without ATL, without MFC, without training wheels. We'll enumerate what's actually registered on a live Windows system, decode the monikers i find, and talk about what the results reveal about COM's runtime model.
![rot_com_architecture (1)](https://github.com/user-attachments/assets/f34fcdaf-a288-49c9-9a7d-9f4df0e0b8ea)
<svg width="100%" viewBox="0 0 680 480" role="img" xmlns="http://www.w3.org/2000/svg">
  <title>ROT in COM architecture</title>
  <desc>Structural diagram showing the Running Object Table as part of the Windows COM infrastructure, spanning user and kernel space</desc>
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
    </marker>
    <style>
      text { font-family: sans-serif; fill: #1a1a1a; }
      .th { font-size: 14px; font-weight: 500; }
      .ts { font-size: 12px; fill: #555; }
      .arr { stroke: #888; stroke-width: 1.5; fill: none; }
      .leader { stroke: #aaa; stroke-width: 0.5; stroke-dasharray: 3 3; fill: none; }
    </style>
  </defs>

  <!-- USER SPACE container -->
  <rect x="30" y="30" width="620" height="270" rx="14" fill="none" stroke="#bbb" stroke-width="0.5" stroke-dasharray="6 4"/>
  <text class="ts" x="46" y="50" fill="#aaa">user space</text>

  <!-- COM Server Process -->
  <rect x="50" y="65" width="170" height="80" rx="8" fill="#EEEDFE" stroke="#534AB7" stroke-width="0.5"/>
  <text class="th" x="135" y="95" text-anchor="middle" dominant-baseline="central" fill="#3C3489">COM server</text>
  <text class="ts" x="135" y="117" text-anchor="middle" dominant-baseline="central" fill="#534AB7">registers object via</text>
  <text class="ts" x="135" y="131" text-anchor="middle" dominant-baseline="central" fill="#534AB7">IRunningObjectTable</text>

  <!-- COM Client Process -->
  <rect x="460" y="65" width="170" height="80" rx="8" fill="#E1F5EE" stroke="#0F6E56" stroke-width="0.5"/>
  <text class="th" x="545" y="95" text-anchor="middle" dominant-baseline="central" fill="#085041">COM client</text>
  <text class="ts" x="545" y="117" text-anchor="middle" dominant-baseline="central" fill="#0F6E56">binds via moniker +</text>
  <text class="ts" x="545" y="131" text-anchor="middle" dominant-baseline="central" fill="#0F6E56">GetRunningObjectTable</text>

  <!-- COM Runtime -->
  <rect x="240" y="65" width="200" height="56" rx="8" fill="#F1EFE8" stroke="#5F5E5A" stroke-width="0.5"/>
  <text class="th" x="340" y="87" text-anchor="middle" dominant-baseline="central" fill="#2C2C2A">COM runtime (ole32.dll)</text>
  <text class="ts" x="340" y="107" text-anchor="middle" dominant-baseline="central" fill="#5F5E5A">CoInitialize / SCM proxy</text>

  <!-- IMoniker -->
  <rect x="50" y="200" width="170" height="56" rx="8" fill="#FAEEDA" stroke="#854F0B" stroke-width="0.5"/>
  <text class="th" x="135" y="222" text-anchor="middle" dominant-baseline="central" fill="#633806">IMoniker</text>
  <text class="ts" x="135" y="242" text-anchor="middle" dominant-baseline="central" fill="#854F0B">File / Item / Composite</text>

  <!-- IEnumMoniker -->
  <rect x="460" y="200" width="170" height="56" rx="8" fill="#FAEEDA" stroke="#854F0B" stroke-width="0.5"/>
  <text class="th" x="545" y="222" text-anchor="middle" dominant-baseline="central" fill="#633806">IEnumMoniker</text>
  <text class="ts" x="545" y="242" text-anchor="middle" dominant-baseline="central" fill="#854F0B">enumerates entries</text>

  <!-- Arrows user space -->
  <line x1="220" y1="105" x2="240" y2="105" stroke="#7F77DD" stroke-width="1.5" fill="none" marker-end="url(#arrow)"/>
  <line x1="440" y1="105" x2="460" y2="105" stroke="#1D9E75" stroke-width="1.5" fill="none" marker-end="url(#arrow)"/>
  <line x1="135" y1="145" x2="135" y2="200" stroke="#BA7517" stroke-width="1.5" fill="none" marker-end="url(#arrow)"/>
  <line x1="545" y1="145" x2="545" y2="200" stroke="#BA7517" stroke-width="1.5" fill="none" marker-end="url(#arrow)"/>

  <!-- RPCSS container -->
  <rect x="30" y="320" width="620" height="130" rx="14" fill="none" stroke="#bbb" stroke-width="0.5" stroke-dasharray="6 4"/>
  <text class="ts" x="46" y="340" fill="#aaa">rpcss.exe (system process)</text>

  <!-- ROT -->
  <rect x="220" y="350" width="240" height="80" rx="8" fill="#FAECE7" stroke="#993C1D" stroke-width="0.5"/>
  <text class="th" x="340" y="380" text-anchor="middle" dominant-baseline="central" fill="#712B13">Running Object Table</text>
  <text class="ts" x="340" y="400" text-anchor="middle" dominant-baseline="central" fill="#993C1D">Moniker → IUnknown map</text>
  <text class="ts" x="340" y="416" text-anchor="middle" dominant-baseline="central" fill="#993C1D">ALPC channel to clients</text>

  <!-- Arrows to ROT -->
  <line x1="340" y1="121" x2="340" y2="346" stroke="#993C1D" stroke-width="1.5" fill="none" marker-end="url(#arrow)"/>
  <line x1="220" y1="228" x2="155" y2="360" stroke="#BA7517" stroke-width="1.5" stroke-dasharray="4 3" fill="none" marker-end="url(#arrow)"/>
  <line x1="460" y1="228" x2="530" y2="360" stroke="#BA7517" stroke-width="1.5" stroke-dasharray="4 3" fill="none" marker-end="url(#arrow)"/>

  <!-- Legend -->
  <rect x="50" y="455" width="10" height="10" rx="2" fill="#AFA9EC"/>
  <text class="ts" x="66" y="465">server process</text>
  <rect x="160" y="455" width="10" height="10" rx="2" fill="#5DCAA5"/>
  <text class="ts" x="176" y="465">client process</text>
  <rect x="270" y="455" width="10" height="10" rx="2" fill="#EF9F27"/>
  <text class="ts" x="286" y="465">moniker layer</text>
  <rect x="380" y="455" width="10" height="10" rx="2" fill="#F0997B"/>
  <text class="ts" x="396" y="465">ROT (rpcss)</text>
</svg>
