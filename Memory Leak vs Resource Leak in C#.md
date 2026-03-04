# Memory Leak vs Resource Leak in C#
### What They Are, How They Differ, and Their Effects on CPU, Memory and Handles

---

## Table of Contents

1. [What is a Resource Leak?](#1-what-is-a-resource-leak)
2. [The Two Types of Leaks](#2-the-two-types-of-leaks)
3. [Memory Leak — Deep Dive](#3-memory-leak--deep-dive)
4. [Resource Leak — Deep Dive](#4-resource-leak--deep-dive)
5. [What Actually Gets Exhausted](#5-what-actually-gets-exhausted)
6. [Effect on CPU, Memory and Performance](#6-effect-on-cpu-memory-and-performance)
7. [How It Shows Up in Production](#7-how-it-shows-up-in-production)
8. [Real World Code Examples](#8-real-world-code-examples)
9. [Your Specific Case — StreamReader Leak](#9-your-specific-case--streamreader-leak)
10. [How to Detect Leaks](#10-how-to-detect-leaks)
11. [The Fix — Golden Rules](#11-the-fix--golden-rules)
12. [Quick Reference Cheat Sheet](#12-quick-reference-cheat-sheet)

---

## 1. What is a Resource Leak?

Think of it like borrowing books from a library and never returning them:

```
You visit the library every day and borrow books.
You never return any of them.

Day 1  → library has 1000 books, you borrow 5
Day 10 → library has 950 books, you borrow 5 more
Day 50 → library has 750 books, running low
Day 100 → library has 500 books, other visitors complain
Day 200 → library has 0 books left

"Sorry, no books available." → library shuts down
```

Replace "library books" with OS file handles, database connections, or network sockets — and that is exactly what a resource leak does to your application.

```
A resource leak happens when your code acquires something
from the OS and never gives it back.

Your app keeps acquiring → OS keeps handing out
→ OS runs out of that resource for everyone
→ Your app (and potentially the whole server) crashes
```

---

## 2. The Two Types of Leaks

People often use these terms interchangeably — but they are **completely different problems** with different symptoms, different causes, and different effects.

```
┌─────────────────────────────────────────────────────────────────┐
│  MEMORY LEAK                                                    │
│  You keep allocating RAM and never freeing it                   │
│  RAM usage grows → GC works harder → OutOfMemoryException       │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  RESOURCE LEAK                                                  │
│  You keep opening OS handles and never closing them             │
│  Handle count grows → OS limit hit → IOException / crash        │
│  RAM usage may look completely normal                           │
└─────────────────────────────────────────────────────────────────┘
```

This is the **most important distinction** and the one most developers miss:

```csharp
// MEMORY LEAK — RAM keeps growing
var list = new List<byte[]>();
while (true)
{
    list.Add(new byte[1024 * 1024]); // adding 1MB every loop
    // never clearing the list
    // RAM: 1MB → 10MB → 100MB → 1GB → OutOfMemoryException ❌
}

// RESOURCE LEAK — handles keep piling up, RAM barely moves
while (true)
{
    var fs = new FileStream("data.txt", FileMode.Open);
    // never disposing
    // RAM: barely changes (FileStream object is only ~200 bytes)
    // OS handles: 1 → 100 → 1000 → 10000 → IOException ❌
}
```

> **Key insight:** A resource leak does NOT mean high memory usage.
> The C# object (FileStream) is tiny — a few hundred bytes.
> The damage is in the OS handle table, completely invisible to RAM monitoring.

---

## 3. Memory Leak — Deep Dive

### What Causes It

```csharp
// Classic C# memory leak — holding references longer than needed

public class EventLeaker
{
    // Static event — lives for the entire lifetime of the app
    public static event EventHandler OnSomethingHappened;

    public void Subscribe()
    {
        // Subscribing here holds a reference to THIS object
        // Even if you set myObject = null, the event still holds it
        OnSomethingHappened += HandleEvent;
        // ❌ never unsubscribed → object can never be GC'd
    }

    private void HandleEvent(object sender, EventArgs e) { }
}

// Also classic — caching without expiry
private static Dictionary<string, HeavyObject> _cache = new();

public void ProcessRequest(string key)
{
    // Adding to cache, never removing
    _cache[key] = new HeavyObject(); // ❌ cache grows forever
}
```

### What GC Does Under Memory Pressure

```
Normal operation:
  GC Gen0 collection  → fast, runs often, clears short-lived objects
  GC Gen1 collection  → medium speed, runs occasionally
  GC Gen2 collection  → slow, runs rarely, full heap scan

Memory leak scenario:
  Objects never become unreachable (still referenced)
  → GC cannot collect them
  → They promote from Gen0 → Gen1 → Gen2
  → Gen2 heap keeps growing
  → GC runs full Gen2 collections more and more often
  → Each Gen2 collection: app pauses (stop-the-world)
  → App becomes visibly slow and unresponsive
  → Eventually: OutOfMemoryException
```

### Symptoms of Memory Leak

```
✗ RAM usage grows steadily over time and never comes down
✗ App gets slower the longer it runs
✗ GC CPU usage climbs (visible in Performance Monitor)
✗ OutOfMemoryException after running for hours/days
✗ Restarting the app "fixes" it temporarily
```

---

## 4. Resource Leak — Deep Dive

### What Causes It

```csharp
// Every one of these leaks if you forget Dispose() / using{}

// Leaks an OS file handle
var fs = new FileStream("file.txt", FileMode.Open);
var content = fs.ReadToEnd();
// forgot: fs.Dispose() ❌

// Leaks a database connection slot
var conn = new SqlConnection(connectionString);
conn.Open();
RunQuery(conn);
// forgot: conn.Dispose() ❌

// Leaks an OS network socket
var client = new HttpClient();
await client.GetStringAsync("https://api.example.com");
// forgot: client.Dispose() ❌

// Leaks a network stream (Azure Blob, for example)
var streamReader = new StreamReader(await blobClient.GetStreamAsync());
var text = await streamReader.ReadToEndAsync();
// forgot: streamReader.Dispose() ❌
```

### Why RAM Looks Fine But App Still Crashes

```
Each leaked FileStream object in .NET heap:  ~200 bytes (tiny)

10,000 leaked FileStream objects:
  .NET heap cost:   200 bytes × 10,000 = ~2MB   ← barely visible
  OS handle cost:   1 handle  × 10,000 = 10,000 handles ← AT THE LIMIT

Your memory monitor shows: "Memory: 210MB — looks healthy!"
Your app shows:            "IOException: too many open files" ❌

This is why resource leaks are so dangerous —
they are invisible to the most common monitoring tools.
```

### How the OS Handle Table Works

```
Windows gives each process a handle table.
Every time you open a file, socket, or connection:
  OS allocates a slot in your process handle table
  Returns a handle number (e.g. 1847) to your code

Handle table has limits:
  Default per-process limit:  ~10,000 handles (configurable)
  System-wide limit:          varies, typically in the millions

When you call Dispose() / CloseHandle():
  OS marks that slot as FREE
  Slot can be reused for the next open file

When you NEVER call Dispose():
  Slot stays OCCUPIED forever (until process restarts)
  New requests keep consuming new slots
  When slots run out → OS refuses to open any more files/sockets
  → Every subsequent operation that needs a handle FAILS
```

---

## 5. What Actually Gets Exhausted

### File Handles

```
Windows per-process file handle limit: ~10,000

Your code (no using{}, called on every request):
  Request 1    → handle #1 opened,    never closed
  Request 2    → handle #2 opened,    never closed
  Request 3    → handle #3 opened,    never closed
  ...
  Request 9999 → handle #9999 opened, never closed
  Request 10000 →
      IOException: "The process cannot access the file because
      too many files are open" ❌

  App crashes. ALL users affected. Restart required.
```

### Database Connection Pool

```csharp
// SqlConnection uses a connection pool
// Pool default size: 100 connections

// Your code (no using{}):
var conn = new SqlConnection(connectionString);
conn.Open();
DoWork(conn);
// conn never disposed — slot never returned to pool

// After 100 such calls:
// Pool is completely exhausted

var conn2 = new SqlConnection(connectionString);
conn2.Open(); // ← waits... waits... waits...
// InvalidOperationException:
// "Timeout expired. The timeout period elapsed prior to
//  obtaining a connection from the pool." ❌

// Every new DB request now fails.
// App is effectively down.
```

### Network Socket / Azure Blob Connection

```csharp
// Azure Blob download without dispose:
var streamReader = new StreamReader(await blobClient.OpenReadAsync());
var content = await streamReader.ReadToEndAsync();
// no dispose → network connection stays open

// Under load (100 requests/second):
// After 10 seconds  → 1,000 open connections
// After 100 seconds → 10,000 open connections
// Azure-side limit  → BlobStorageException ❌
// OS socket limit   → SocketException: address already in use ❌
```

### Thread Pool (indirect effect)

```
Some resources hold threads internally.
Leaked resources can indirectly exhaust thread pool:

  ThreadPool.SetMinThreads(4, 4);  // only 4 threads available
  
  Request 1  → grabs thread, waiting on leaked resource
  Request 2  → grabs thread, waiting on leaked resource
  Request 3  → grabs thread, waiting on leaked resource
  Request 4  → grabs thread, waiting on leaked resource
  Request 5  → NO THREAD AVAILABLE
              → queued... waiting...
              → TaskScheduler delays
              → all async/await tasks stall
              → app appears completely frozen
```

---

## 6. Effect on CPU, Memory and Performance

### Side by Side Comparison

```
                    RAM Usage    CPU Usage    OS Handles    App Effect
                    ─────────    ─────────    ──────────    ──────────────────────
Memory Leak         📈 GROWS     📈 HIGH      normal        OOM crash, GC thrashing
                    (steady      (GC keeps
                     climb)       running)

Resource Leak       STABLE ✅    📈 slight    📈 GROWS      IOException / timeout
                    (looks       (finalizer   (steady        crash, silent slowdown
                     healthy!)    thread)      climb)         first, then sudden

Both Together       📈 GROWS     📈 HIGH      📈 GROWS      App dies much faster
```

### CPU Impact of a Resource Leak

```csharp
// When you do NOT call Dispose()...
// The object has a finalizer (~destructor)
// GC cannot free it in one cycle — it goes through finalization

// NORMAL object (no finalizer, properly disposed):
//   GC cycle 1 → object unreachable → freed immediately ✅
//   Cost: minimal

// Object WITH finalizer (not disposed):
//   GC cycle 1 → unreachable → moved to Finalization Queue
//   Finalizer thread → runs ~StreamReader() → releases handle
//   GC cycle 2 → NOW freed
//   Cost: 2x GC cycles + finalizer thread CPU time

// At scale — thousands of undisposed objects:
//   Finalization queue backs up
//   Finalizer thread runs constantly   → CPU spike 📈
//   GC runs more Gen2 collections      → CPU spike 📈
//   App pauses during GC               → latency increases
//   All requests slow down             → users notice
```

### Memory Impact of a Resource Leak

```csharp
// The C# wrapper object is tiny:
var fs = new FileStream("file.txt", FileMode.Open);
// FileStream on heap ≈ 200 bytes

// BUT because it has a finalizer and is NOT disposed:
//   Survives first GC → promoted to Gen1
//   Survives second GC → promoted to Gen2
//   Gen2 objects are collected RARELY (expensive full GC)
//   1000 leaked streams → 1000 objects stuck in Gen2
//   Gen2 heap grows → GC triggers more full collections
//   Full GC = stop-the-world pause on all threads
//   App freezes briefly on every full GC
//   As more objects accumulate → freezes get longer and more frequent
```

### The Degradation Curve

```
Performance
    │
100%│████████████████
    │                ████████
 80%│                        ████████
    │                                ████
 60%│                                    ████
    │                                        ████
 40%│                                            ██
    │                                              █
 20%│                                               █
    │                                                █
  0%└──────────────────────────────────────────────────▶ Time
    │                                                │
    Day 1                                        Crash ❌
    (deployed)

This curve is the classic signature of a resource leak.
Gradual slowdown → sudden crash → restart "fixes" it → repeat.
```

---

## 7. How It Shows Up in Production

```
Day 1 of deployment:
  Response time : 45ms    ✅
  RAM usage     : 200MB   ✅
  OS handles    : 150     ✅
  DB pool used  : 5/100   ✅
  Errors        : 0       ✅
  Team          : happy   😊

Day 3:
  Response time : 120ms   ⚠️  (slowing down)
  RAM usage     : 220MB   ✅  (looks fine — misleading!)
  OS handles    : 3,500   ⚠️  (quietly climbing)
  DB pool used  : 45/100  ⚠️  (almost half gone)
  Errors        : occasional timeout
  Team          : "hmm, weird" 🤔

Day 7:
  Response time : 800ms   ❌  (very slow)
  RAM usage     : 280MB   ✅  (still looks healthy — very misleading!)
  OS handles    : 9,200   ❌  (near the 10,000 limit)
  DB pool used  : 98/100  ❌  (almost exhausted)
  Errors        : frequent timeouts and connection errors
  Team          : worried 😟

Day 8, 3:17am:
  IOException: "Too many open files"   ❌  APP DOWN
  InvalidOperationException: "Connection pool exhausted" ❌
  On-call engineer gets paged 📟
  Restart at 3:30am — app recovers
  Problem will come back in ~7 days
  Team          : very unhappy 😤
```

> This exact pattern — **gradual degradation then sudden crash that a restart temporarily fixes** — is the unmistakable signature of a resource leak.
> A memory leak looks similar but RAM usage would be climbing in the monitoring graphs.
> A resource leak's RAM stays flat while handles climb — which is why it is so often missed.

---

## 8. Real World Code Examples

### FileStream — OS file handle leak

```csharp
// ❌ BAD — handle leaked on every call
public string ReadConfig(string path)
{
    var fs      = new FileStream(path, FileMode.Open);
    var reader  = new StreamReader(fs);
    return reader.ReadToEnd();
    // both fs and reader go out of scope undisposed
    // file stays locked until GC runs finalizer (unknown time)
}

// ✅ GOOD — handle released immediately
public string ReadConfig(string path)
{
    using var fs     = new FileStream(path, FileMode.Open);
    using var reader = new StreamReader(fs);
    return reader.ReadToEnd();
    // Dispose() called the moment method returns ✅
    // File unlocked immediately ✅
}
```

### SqlConnection — connection pool exhaustion

```csharp
// ❌ BAD — connection never returned to pool
public List<User> GetUsers()
{
    var conn = new SqlConnection(connectionString);
    conn.Open();
    var cmd    = new SqlCommand("SELECT * FROM Users", conn);
    var reader = cmd.ExecuteReader();
    return MapUsers(reader);
    // conn, cmd, reader — all undisposed
    // pool slot held forever ❌
}

// ✅ GOOD — connection returned to pool immediately
public List<User> GetUsers()
{
    using var conn   = new SqlConnection(connectionString);
    using var cmd    = new SqlCommand("SELECT * FROM Users", conn);
    conn.Open();
    using var reader = cmd.ExecuteReader();
    return MapUsers(reader);
    // all disposed when method returns ✅
    // pool slot freed immediately ✅
}
```

### HttpClient — socket exhaustion

```csharp
// ❌ BAD — new HttpClient per request, never disposed
public async Task<string> GetDataAsync(string url)
{
    var client = new HttpClient();
    return await client.GetStringAsync(url);
    // socket stays in TIME_WAIT state for ~240 seconds
    // under load → port exhaustion ❌
}

// ✅ GOOD — reuse a single HttpClient (recommended pattern)
// HttpClient is designed to be reused — one instance per app
private static readonly HttpClient _client = new HttpClient();

public async Task<string> GetDataAsync(string url)
{
    return await _client.GetStringAsync(url);
    // reuses existing connection pool ✅
    // no socket exhaustion ✅
}
```

### Azure Blob Stream — network connection leak

```csharp
// ❌ BAD — stream never disposed
public async Task<T> ReadBlobAsync<T>(BlobClient blob)
{
    var streamReader = new StreamReader(
        await blob.OpenReadAsync());
    var content = await streamReader.ReadToEndAsync();
    return JsonConvert.DeserializeObject<T>(content);
    // network connection to Azure stays open ❌
    // under load → Azure connection limit hit ❌
}

// ✅ GOOD — stream disposed immediately after use
public async Task<T> ReadBlobAsync<T>(BlobClient blob)
{
    using var streamReader = new StreamReader(
        await blob.OpenReadAsync());
    var content = await streamReader.ReadToEndAsync();
    return JsonConvert.DeserializeObject<T>(content);
    // connection closed the moment method returns ✅
}

// ✅ EVEN BETTER — use DownloadContentAsync for small blobs
public async Task<T> ReadBlobAsync<T>(BlobClient blob)
{
    var result = await blob.DownloadContentAsync();
    return result.Value.Content.ToObjectFromJson<T>();
    // SDK handles everything internally ✅
    // No streams to manage at all ✅
}
```

---

## 9. Your Specific Case — StreamReader Leak

```csharp
// This code leaks on EVERY single call ❌
var streamReader = new StreamReader(
    await asdasd.asdasdas(
        metadata.TenantObjectId,
        metadata.CapacityObjectId,
        metadata.FolderObjectId,
        metadata.ObjectId,
        subFolder,
        file,
        cancellationToken));

var content = await streamReader.ReadToEndAsync();
return JObject.Parse(content).ToObject<TExpectedType>();
// method returns — streamReader never disposed ❌
```

### What Leaks on Every Call

```
Call to this method
        │
        ├── Opens network stream to storage service  ← OS socket handle
        ├── Creates StreamReader wrapping it         ← managed wrapper
        ├── Reads content ✅
        ├── Returns result ✅
        └── StreamReader goes out of scope
                │
                └── Dispose() NEVER called ❌
                        │
                        ├── Network connection stays open ❌
                        ├── Socket handle held by OS ❌
                        └── GC finalizer runs eventually...
                                (unpredictable — could be minutes)
                                disposing=false → only unmanaged cleaned
```

### Impact Under Load

```
Service receives 50 requests/second:

After 10 seconds  →   500 open network connections
After 60 seconds  → 3,000 open network connections
After 3 minutes   → 9,000 open network connections
After 3.5 minutes → storage service rejects connections ❌
                    SocketException on every new request
                    Service is down
```

### The Fix — One Line Change

```csharp
// ✅ FIXED — using var guarantees Dispose() on method exit
using var streamReader = new StreamReader(
    await asdasd.asdasdas(
        metadata.TenantObjectId,
        metadata.CapacityObjectId,
        metadata.FolderObjectId,
        metadata.ObjectId,
        subFolder,
        file,
        cancellationToken));

var content = await streamReader.ReadToEndAsync();
return JObject.Parse(content).ToObject<TExpectedType>();
// Dispose() called here automatically ✅
// Network connection closed ✅
// Socket handle released ✅
// Works correctly even if an exception is thrown ✅
```

---

## 10. How to Detect Leaks

### Detecting Memory Leaks

```
Tools:
  dotMemory (JetBrains)      → best for .NET memory profiling
  Visual Studio Diagnostic   → built-in memory snapshot tool
  PerfView (Microsoft free)  → GC and memory analysis

Signs in code review:
  → Static collections that grow (Dictionary, List) with no eviction
  → Event subscriptions never unsubscribed
  → Caches with no expiry
  → Large objects held in long-lived scopes
```

### Detecting Resource Leaks

```
Tools:
  Process Explorer (SysInternals)  → see handle count per process
  Performance Monitor (perfmon)    → track "Handle Count" counter
  dotTrace (JetBrains)             → profiler shows finalizer queue depth
  Application Insights             → track exception rates over time

Signs in code review:
  → new FileStream / StreamReader / SqlConnection / HttpClient
    without using{} or .Dispose()
  → IDisposable objects created in a loop without using{}
  → Streams passed around and disposed "somewhere else" (easy to miss)
  → try/catch blocks where Dispose() is only in the happy path

Signs in production:
  → Handle count climbing in perfmon
  → "Too many open files" IOException
  → "Connection pool exhausted" InvalidOperationException
  → Performance degrades gradually, restart fixes it temporarily
```

### Quick Code Review Checklist

```csharp
// Run this mental check on any class that implements IDisposable:

// 1. Is it in a using{} block?
using (var x = new SomeDisposable()) { } ✅

// 2. Is it using var declaration (C# 8+)?
using var x = new SomeDisposable(); ✅

// 3. Is Dispose() called in finally block?
SomeDisposable x = null;
try { x = new SomeDisposable(); }
finally { x?.Dispose(); } ✅

// 4. None of the above?
var x = new SomeDisposable(); ❌ LEAK
```

---

## 11. The Fix — Golden Rules

```
RULE 1: If it implements IDisposable → ALWAYS use using{}
        No exceptions. No "I'll dispose it later."

RULE 2: using var (C# 8+) is your best friend
        One keyword, zero effort, guaranteed cleanup.

RULE 3: Dispose() is called even when exceptions are thrown
        using{} = try/finally under the hood. Always safe.

RULE 4: StreamReader owns its inner stream by default
        Disposing StreamReader also disposes the inner Stream.
        You only need one using.

RULE 5: HttpClient is the exception — reuse one instance
        Don't create and dispose HttpClient per request.
        Use IHttpClientFactory or a static instance.

RULE 6: In a loop, dispose inside the loop
        Each iteration should clean up before the next starts.

RULE 7: If you see new X() where X : IDisposable
        and there is no using → flag it in code review.
```

---

## 12. Quick Reference Cheat Sheet

### Memory Leak vs Resource Leak

| | Memory Leak | Resource Leak |
|---|---|---|
| What grows | RAM (heap) | OS handles / connections |
| RAM monitoring shows | 📈 Growing | ✅ Looks normal |
| Handle monitoring shows | ✅ Normal | 📈 Growing |
| CPU effect | High (GC thrashing) | Moderate (finalizer thread) |
| Crash type | OutOfMemoryException | IOException / pool exhaustion |
| How fast | Hours to days | Hours to days (under load: minutes) |
| Restart fixes it? | Temporarily | Temporarily |
| Root cause | Object references held too long | Dispose() never called |
| Fix | Remove stale references, fix caching | Add using{} / call Dispose() |

### Common IDisposable Types to Always Dispose

| Type | What leaks without Dispose | Consequence |
|---|---|---|
| `FileStream` | OS file handle | File stays locked |
| `StreamReader` | Inner stream handle | File / socket locked |
| `SqlConnection` | DB connection pool slot | Pool exhaustion |
| `SqlCommand` | Unmanaged command resources | Resource buildup |
| `HttpClient` | OS socket | Port exhaustion |
| `NetworkStream` | OS socket handle | Socket exhaustion |
| `BlobClient stream` | Network connection | Azure connection limit |
| `Timer` | OS timer handle | Timer never stops firing |
| `Mutex / Semaphore` | OS sync handle | Deadlocks |

### The One Rule

```
If you see this pattern anywhere in your codebase:

    var x = new SomethingDisposable();

And there is no using keyword anywhere near it → it is a leak.
Change it to:

    using var x = new SomethingDisposable();

That single keyword is the difference between a stable service
and a 3am on-call page.
```

---

*Guide covers: resource leak, memory leak, OS handles, file handle exhaustion,
connection pool exhaustion, socket exhaustion, GC finalizer thread, CPU and memory
effects, production degradation patterns, StreamReader leak, Azure Blob stream leak,
detection tools and golden rules.*
