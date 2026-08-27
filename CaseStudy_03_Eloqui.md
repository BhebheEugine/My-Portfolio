# Case Study 03 — Eloqui
## Personal Android English Vocabulary Tutor

**Role:** Solo Developer  
**Type:** Native Android Application  
**Stack:** Kotlin · Jetpack Compose · MVVM · Room · Supabase · WorkManager  
**Platform:** Android  
**Timeline:** 2025 — Present  
**Status:** Built, signed, personal use  
**Availability:** Personal — not publicly distributed

---

## The Problem

I wanted to improve my English vocabulary systematically.

Every existing solution was either too passive (reading apps that expose you to words without testing retention), too gamified in ways that felt patronising (Duolingo-style streaks and cartoon characters), or required a subscription to features that should be basic.

More specifically, I wanted:
- Words presented in context, not in isolation
- AI-generated explanations that went beyond dictionary definitions
- Spaced repetition that actually adapted to what I was getting wrong
- Notifications that appeared when I was likely to be available — not just at arbitrary times
- An app that worked offline, because I do not always have data
- The ability to switch between AI providers without being locked into one

None of the available apps gave me all of this. So I built one.

---

## Why This Case Study Matters

Eloqui is the project that most directly reveals how I approach engineering when no one is watching.

There is no client. No deadline. No user base to impress. No grade. Just me, deciding what to build and how to build it — and choosing to do it properly anyway.

The architectural decisions in Eloqui — the MVVM separation, the Room database, the AIProvider abstraction — are more rigorous than many production applications I have seen from professional teams. Not because rigor was required, but because I find poorly structured code uncomfortable to work in, regardless of who the audience is.

---

## What I Built

### Architecture

Full MVVM (Model-View-ViewModel) with clean architecture separation across three layers.

**Why MVVM for a personal app?**  
Because the alternative — putting business logic in the UI layer — creates a codebase that is painful to extend. When I want to add a new AI provider, or change the spaced repetition algorithm, or add a new screen, MVVM means those changes are isolated. The ViewModel does not know what the UI looks like. The Repository does not know which ViewModel is calling it. Each layer has one job.

This is not over-engineering. Over-engineering would be adding microservices or a message queue. MVVM for an Android app is the minimum viable architecture for a codebase I want to still enjoy working in six months from now.

**Layer breakdown:**
- **UI Layer** — Jetpack Compose screens, observing ViewModel state via `StateFlow`
- **ViewModel Layer** — Business logic, state management, coroutine scope management
- **Data Layer** — Repository pattern abstracting Room (local) and Supabase (remote)

---

### Jetpack Compose UI

Ten+ screens built in Jetpack Compose.

Compose was the right choice for a solo developer building a new Android app in 2025. The declarative model means UI state is predictable and testable. The composable function model means UI components are reusable without the boilerplate of fragment transactions or view binding.

Key screens:
- **Home** — Daily word queue, progress indicator, streak
- **Word Detail** — Definition, AI explanation, example sentences, etymology
- **Quiz** — Spaced repetition quiz session with answer evaluation
- **Word List** — Browse all saved words with search and filter
- **Settings** — AI provider selection, API key entry, notification schedule
- **Stats** — Learning progress visualisation, accuracy by word category
- **Add Word** — Manual word entry with AI-assisted definition fetch

---

### Room — Local-First Storage

Room (SQLite wrapper) is the primary data store. The app is fully functional without any network connection.

The word database schema:

```
words
  id          INTEGER PRIMARY KEY
  term        TEXT NOT NULL
  definition  TEXT
  context     TEXT         -- the sentence where I first encountered this word
  ai_explanation TEXT      -- AI-generated contextual explanation
  difficulty  INTEGER      -- 1-5 scale, updated by spaced repetition algorithm
  next_review DATETIME     -- when spaced repetition schedules next review
  correct_count INTEGER
  incorrect_count INTEGER
  added_at    DATETIME
  last_seen   DATETIME

quiz_sessions
  id          INTEGER PRIMARY KEY
  started_at  DATETIME
  completed_at DATETIME
  words_reviewed INTEGER
  correct_count INTEGER

quiz_results
  id          INTEGER PRIMARY KEY
  session_id  INTEGER REFERENCES quiz_sessions(id)
  word_id     INTEGER REFERENCES words(id)
  was_correct BOOLEAN
  response_time_ms INTEGER
```

The spaced repetition algorithm updates `difficulty` and `next_review` after each quiz result. Words answered correctly get a longer review interval. Words answered incorrectly get a shorter one and their difficulty increases.

---

### AIProvider Abstraction

This is the most technically interesting component of Eloqui.

I wanted to use AI to generate contextual word explanations — not dictionary definitions, but explanations that show how a word functions in context, its nuance, its common misuses, and two or three memorable example sentences.

But I did not want to be locked into a single AI provider. Different models have different strengths, different pricing, and different availability. I also wanted to be able to switch providers without touching the rest of the application.

The solution is a clean abstraction:

```kotlin
interface AIProvider {
    suspend fun explainWord(term: String, context: String?): WordExplanation
    suspend fun generateExamples(term: String, count: Int): List<String>
    suspend fun evaluateAnswer(term: String, userAnswer: String): AnswerEvaluation
    val providerName: String
}
```

Three concrete implementations:

```kotlin
class GeminiProvider(private val apiKey: String) : AIProvider { ... }
class ClaudeProvider(private val apiKey: String) : AIProvider { ... }
class GrokProvider(private val apiKey: String) : AIProvider { ... }
```

All three are at full commercial parity — not placeholder stubs with TODO comments. Each one makes real API calls to its respective service, parses the response into the shared `WordExplanation` data class, and handles errors consistently.

The Settings screen lets me switch between providers and enter API keys in the app UI. The `AIRepository` injects whichever provider is currently selected.

**Why this matters architecturally:**  
Adding a fourth provider — say, a locally-running Ollama instance — requires implementing the `AIProvider` interface and registering it in the dependency graph. Zero changes to any ViewModel, Repository, or UI screen. That is the correct outcome of a well-designed abstraction.

---

### WorkManager — Intelligent Notifications

Three notifications per day. But not at fixed times.

WorkManager schedules notification workers with time constraints that match realistic availability windows:
- Morning session: 7:00–8:30 AM
- Midday session: 12:30–1:30 PM
- Evening session: 8:00–9:30 PM

Each worker checks:
1. Was the previous session completed? If yes, show the next one.
2. Is there a quiz due (based on spaced repetition `next_review` dates)? If yes, prioritise that notification.
3. Has the user already seen a notification in the last 2 hours? If yes, skip.

WorkManager handles this reliably across Android's battery optimisation systems — which aggressively kill background processes on most Android manufacturers' devices. Using `setExpedited()` for time-sensitive notifications and `setRequiresBatteryNotLow()` for non-urgent ones keeps the app well-behaved without draining battery.

---

### Supabase Cloud Sync

Room handles local storage. Supabase handles cloud backup and restoration.

The sync strategy is one-way from the device to the cloud, triggered on significant events (completing a quiz session, adding a new word, manually requesting a sync). This avoids the complexity of bidirectional sync conflict resolution for a single-user app.

The sync gives me: ability to restore my word database when I change phones, and ability to view my learning statistics from a browser if I want to.

There is no authentication layer. API keys are entered directly in the Settings screen and stored in Android's encrypted SharedPreferences. This is appropriate for a personal app where I am the only user.

---

## Technical Decisions I Would Make Differently

**Spaced repetition algorithm**  
I implemented a simplified version of SM-2 (the algorithm used by Anki and most spaced repetition apps). With more time, I would implement a more sophisticated model that accounts for response time (not just correctness), word difficulty clustering, and optimal review session length.

**No authentication**  
The no-authentication decision made the app simpler to build but means the Supabase data is not truly private — anyone with the API key from the APK could read or write to the database. For a personal app this is acceptable. For a distributed app, this would need to be rethought entirely.

**AI response caching**  
Currently, every AI explanation is generated fresh on demand. Adding a local cache in Room for generated explanations would reduce API calls significantly and make the app faster for words I look at frequently. This is the next planned improvement.

---

## What I Learned

**Technical:**
- Jetpack Compose's `remember` and `rememberSaveable` have different lifecycles and using the wrong one causes state loss on configuration changes — a subtle bug that only appears when rotating the screen or switching apps
- WorkManager constraints interact with Android's Doze mode in non-obvious ways on certain manufacturers (particularly Xiaomi and Samsung) — testing on multiple physical devices is essential for notification reliability
- The `AIProvider` abstraction showed me that designing for replaceability from the start costs very little extra time and pays off immediately the first time you want to swap something out
- Room's `@Transaction` annotation is essential for operations that touch multiple tables — without it, a crash mid-operation can leave the database in an inconsistent state

**Personal:**
- Building something purely for yourself removes a specific kind of pressure (shipping, users, feedback) but introduces another (you have to actually want to use it, or you will not finish it)
- The apps I find myself most proud of are the ones where the internal structure is clean, regardless of what the user sees. Eloqui has no users, but I would not be embarrassed to show any experienced Android developer the codebase.

---

## What Eloqui Demonstrates to a Hiring Manager

**1. I apply professional engineering practices to personal work.**  
MVVM, clean architecture, Room, and the AIProvider abstraction are not things I reached for because I had to. I reached for them because they produce better software. That habit does not switch off when I am working on a team project.

**2. I understand Android development at a non-tutorial depth.**  
WorkManager lifecycle constraints, Compose state management, Room transactions, encrypted SharedPreferences, Expo EAS vs native build signing — these are the kinds of details that separate someone who has done Android tutorials from someone who has shipped a signed Android application.

**3. I think about extensibility.**  
The AIProvider interface is the clearest example. No one asked me to support three AI providers. I did it because I could see that having a single hardcoded dependency on one AI service was a design that would require significant rework the moment I wanted to change anything. Good software anticipates change without over-engineering for changes that will never come.

**4. I build for myself, which means I build with genuine standards.**  
There is no one to show this to. No portfolio reviewer is going to diff my Room schema. The fact that the schema is well-normalised, the fact that the AIProvider abstraction is clean, the fact that the WorkManager logic handles edge cases — these exist because I found the alternative unsatisfying, not because anyone told me to.

---

## Stack Summary

| Layer | Technology |
|---|---|
| Language | Kotlin |
| UI Framework | Jetpack Compose |
| Architecture | MVVM + Clean Architecture |
| Local Storage | Room (SQLite) |
| Cloud Sync | Supabase |
| Background Work | WorkManager |
| AI Providers | Gemini, Claude, Grok (via AIProvider abstraction) |
| Build | Gradle, release signing configured |
| Target Platform | Android |

