# android-leak-solver-skill

An AI agent skill that diagnoses Android memory leaks, finds the root cause, generates a surgical fix, and verifies its own reasoning — without human intervention.

Drop `SKILL.md` into any AI coding assistant (opencode, Claude Code, Cursor, Windsurf, etc.), paste a LeakCanary trace, and get:

- The **root cause** (not just the symptom — the actual lifecycle mismatch)
- A **surgical fix** targeting the root cause node
- A **mechanistic explanation** of why the fix breaks the retention chain
- A **self-verification checklist** the agent runs on its own output before showing you anything
- A **verify step** so you know exactly how to confirm the leak is gone

---

## Install

Paste this into your agent (Claude Code, opencode, Cursor, Windsurf, Gemini CLI — any tool):

```
Download https://raw.githubusercontent.com/joni-codes/android-leak-solver-skill/main/SKILL.md and install it as a skill/rule/context file in the correct location for your AI coding tool, then confirm where you placed it and how to activate it.
```

The agent knows what tool it's running in and will install it to the right place automatically.

---

## Usage

Once installed, tell your agent:

```
Load and apply the android-leak-solver skill. I will now paste a memory leak trace. Walk the full retention chain, identify the root cause node, classify the pattern, generate a surgical fix, verify your own reasoning using the self-check checklist, and output the result in the required format. Do not skip any phase.
```

Then paste your trace. The agent handles everything from there.

### Supported input formats

- **LeakCanary** — structured retention path output (recommended)
- **hprof heap dump** — from Android Studio Memory Profiler or `adb`
- **Logcat OOM** — `java.lang.OutOfMemoryError` with stack trace
- **Manual description** — describe the symptom if you have no trace

---

## What the agent covers

The skill contains a finite taxonomy of Android leak patterns. The agent classifies your leak against this list — it cannot hallucinate a novel pattern:

| Pattern | Description |
|---------|-------------|
| P1 | Activity/Fragment context held by singleton or static field |
| P2 | Non-static inner class / anonymous class holding implicit outer ref |
| P3 | Handler with pending messages outliving Activity |
| P4 | Listener registered but never unregistered |
| P5 | ViewModel holding Activity context directly |
| P6 | Coroutine or Thread without lifecycle scope |
| P7 | Fragment holding View reference after onDestroyView |
| P8 | Cursor / Stream / Connection not closed |
| P9 | Static View or Bitmap reference |
| P10 | RxJava / Flow / LiveData subscription not disposed |

---

## Why this is different from just asking an AI

Without the skill, AI assistants tend to:
- Stop at the symptom ("the Activity is leaking") instead of the root cause
- Give generic advice ("use WeakReference") without reading your actual trace
- Skip verifying whether the fix actually breaks the retention chain

This skill enforces:
- A full retention chain walk before any conclusion
- Causal root cause language tied to specific classes from your trace
- A 6-item self-verification checklist the agent runs on its own fix
- An explicit failure mode — if the trace is insufficient, the agent tells you exactly what command to run to get the missing info

---

## Example output

```
ANDROID LEAK ANALYSIS
══════════════════════════════════════════

TRACE FORMAT:    LeakCanary
PATTERN:         P2 — Non-static inner class / anonymous class
LEAKING OBJECT:  MainActivity — Activity#mDestroyed is true
GC ROOT:         Thread (HandlerThread)

ROOT CAUSE
──────────
MainActivity$fetchData$1 is a lambda defined inside MainActivity. Lambdas and
anonymous classes in Kotlin/Java capture an implicit reference to their enclosing
instance (this$0 = MainActivity). This lambda was posted as a Message callback to
a HandlerThread. The HandlerThread's MessageQueue is GC-root-reachable via its
Thread, so as long as the message is pending, MainActivity cannot be collected
after onDestroy.

FIX
───
File: MainActivity.kt

  private fun fetchData() {
-     handler.post {
-         updateUI(result)
-     }
+     val weakActivity = WeakReference(this)
+     handler.post {
+         weakActivity.get()?.updateUI(result)
+     }
  }

+ override fun onDestroy() {
+     super.onDestroy()
+     handler.removeCallbacksAndMessages(null)
+ }

WHY THIS WORKS
──────────────
WeakReference breaks the strong reference from the lambda to MainActivity. The GC
can now collect MainActivity after onDestroy regardless of pending messages.
removeCallbacksAndMessages(null) clears the queue immediately on destroy so the
lambda never executes against a dead Activity.

VERIFY
──────
1. Run LeakCanary — HandlerThread → MessageQueue → Message → MainActivity path should not appear.
2. Confirm fetchData still updates UI correctly when Activity is alive.
3. Confirm no NullPointerException from the weakActivity.get() call path.

WATCH OUT
─────────
If multiple handlers post to this Activity, each needs the same WeakReference
pattern and removeCallbacksAndMessages in onDestroy. One missed handler reproduces the leak.

SELF-CHECK RESULT
─────────────────
✓ Fix targets the lambda's implicit reference (root cause node), not MainActivity itself.
✓ WeakReference breaks the retention chain at the Message → MainActivity link.
✓ No alternate path remains — removeCallbacksAndMessages eliminates the pending message.
✓ Null-check on weakActivity.get() prevents NPE.
✓ VERIFY step confirms via LeakCanary path disappearance, not just "run the app".
```
