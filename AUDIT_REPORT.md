# Appzuku Audit Report

## Executive summary

This project is a focused Android utility for stopping background apps, managing whitelist and blacklist rules, and enforcing background restrictions using either root or Shizuku. The app has a clear purpose and a practical feature set that solves a real problem for Android power users.

The codebase is functional and thoughtfully scoped, but the architecture is showing signs of stress. The main issues are not feature absence; they are maintainability, lifecycle safety, and privileged-command risk. The app is a capable utility, but it currently mixes UI logic, domain rules, and system access in ways that make it harder to evolve safely.

## Scope reviewed

Reviewed materials included:
- README.md
- app/build.gradle
- app/src/main/java/com/northmendo/Appzuku/MainActivity.java
- app/src/main/java/com/northmendo/Appzuku/BackgroundAppManager.java
- app/src/main/java/com/northmendo/Appzuku/ShellManager.java
- app/src/main/java/com/northmendo/Appzuku/AppModel.java
- app/src/main/java/com/northmendo/Appzuku/db/AppDatabase.java

## Product overview

The app is primarily a process-control utility. It allows users to:
- view background/running apps
- filter apps and protect selected ones
- kill selected processes or run auto-kill conditions
- use whitelist/blacklist modes
- apply background restrictions on Android 11+
- maintain kill history and statistics
- store settings for future sessions

This is a coherent product vision for a niche Android utility.

## Architecture overview

The codebase is structured around a few key components:

- UI layer: `MainActivity`, `SettingsActivity`, `BaseActivity`, and helper screens
- orchestration layer: `BackgroundAppManager`
- shell abstraction: `ShellManager`
- persistence: `SharedPreferences` and Room database (`AppDatabase`)
- domain model: `AppModel`

At a high level, the flow is:
- Activity renders the list and user controls
- `BackgroundAppManager` gathers package and RAM information
- `ShellManager` executes privileged commands using root or Shizuku
- user rules and app state are persisted in preferences and DAO-backed database tables

This is understandable and practical, but it is not yet a clean architecture for a privileged app.

## What the app does well

### Clear functional focus
The app knows exactly what it does. It is not trying to be a general-purpose Android suite; it is focused on background app control and optimization.

### Root and Shizuku support is a strong idea
Supporting both root and Shizuku is a good product decision because it broadens the audience and makes privileged access more flexible.

### Useful feature breadth for a utility app
The codebase supports a meaningful set of capabilities, including:
- quick kills
- protected apps
- whitelist/blacklist filtering
- scheduled background kills
- background restrictions
- RAM monitoring
- backup-style preference lists

### Structured persistence is present
The use of Room for kill statistics is a positive sign. It shows there is already a move toward structured persistence rather than only storing everything in volatile state.

## Main issues and critique

### 1. Privileged shell commands are too fragile
This is the most important issue.

`BackgroundAppManager` and `ShellManager` assemble shell commands dynamically, such as:
- `am force-stop <package>`
- `ps -A -o rss,name ...`
- `dumpsys activity activities`

The project defines a package-name regex, but it is not consistently enforced before issuing privileged commands. That means the app is relying mostly on trusted values from the package manager, which is good in most cases, but still too loose for a system-level tool.

Why this matters:
- shell command assembly is a common source of privilege-related bugs
- package names should be treated as validated input, not raw strings
- command-building logic should be centralized and strictly reviewed

Recommended solution:
- create a single `SystemCommandExecutor` or `ProcessCommandFactory`
- validate package names and command arguments centrally
- avoid raw concatenation where possible
- use explicit, typed command-building methods instead of freeform strings

### 2. The architecture is too monolithic for the complexity of the app
`MainActivity` is doing a large amount of work:
- permission management
- settings loading
- UI hookup
- list event handling
- app actions
- uninstall dialogs
- replacement notice logic
- app list updates

Likewise, `BackgroundAppManager` holds too many responsibilities:
- package discovery
- kill logic
- app filtering
- notifications
- widget update logic
- persistence access
- relaunch detection
- background restriction logic

This is a common anti-pattern in utility apps, but it makes debugging and future changes difficult.

Recommended solution:
- split responsibilities into repositories and use cases
- keep UI activities focused on rendering and action dispatching
- move policy/decision logic into dedicated classes
- introduce `ViewModel` + `LiveData`/`StateFlow` to manage screen state

### 3. `AppModel` mixes too many concerns
`AppModel` is acting as a data object, UI state holder, and business-rule container. It tracks:
- app metadata
- selection state
- protection status
- whitelist state
- background restriction state
- a derived status string

This is a sign that the app is evolving without a strict domain model. The result is harder testing and more inconsistent updates.

Recommended solution:
- split data models from UI state models
- move status calculations into dedicated policy objects or mappers
- reduce domain logic from plain model classes

### 4. SharedPreferences is too weak for configuration growth
The app stores rules like whitelists, blacklists, hidden apps, and background restrictions in `SharedPreferences` as string sets. That is convenient but fragile as the app grows.

This becomes more problematic if feature count expands because the configuration will become harder to validate and migrate.

Recommended solution:
- migrate to `DataStore`
- define typed preference models
- centralize configuration validation and migrations
- use clear schema versions and migration handling

### 5. Lifecycle and thread handling are not strong enough for a system app
There is an `ExecutorService` created in the activity and passed into managers, but there is no clearly visible lifecycle shutdown path. That can lead to orphaned threads or resource leaks.

The app also does a lot of background work and then posts results back to the UI thread using `Handler`. That is workable, but the lifecycle ownership is not explicit enough.

Recommended solution:
- scope executors and resources to a proper lifecycle owner
- move background work into `ViewModel`-owned coroutines or lifecycle-aware tasks
- register and unregister permission listeners in a clearly controlled lifecycle flow

### 6. Test coverage appears weak for a privileged utility
The project has useful functionality but not enough evidence of comprehensive automated safeguards. For a utility that calls privileged system commands, coverage is especially important.

This app should have:
- unit tests for decision logic and selection rules
- tests for background restriction states
- tests for permission fallback logic
- instrumentation tests around UI flows and common user actions

### 7. Build and environment validation is not yet fully reliable
I attempted a local debug build check, and the Gradle configuration failed in the environment with corrupted or unreadable cache metadata. That is not necessarily a source-code bug, but it is a sign that release verification should be more disciplined.

Recommended solution:
- clear/rebuild Gradle caches in a controlled way
- establish a repeatable CI build and lint pipeline
- enforce a clean build gate before release

## App critique

### Strengths
- Strong product focus
- Real utility value
- Root + Shizuku support is a practical choice
- Clear app intent and user flow
- Feature set is broad enough for the niche

### Weaknesses
- Too much logic in UI and manager classes
- Shell access is not robustly hardened
- state and domain logic are mixed together
- persistence is too basic for future growth
- lifecycle management is not fully disciplined
- test coverage is not strong enough for a privileged app

### Overall assessment

This is a functional app with a solid purpose, but it is architecturally closer to a prototype utility than a mature Android product. The code is readable and the ideas are sound, but it needs architectural cleanup and stronger safety controls to be reliable long-term.

## Architecture critique and suggested direction

### Current architecture

The current design is essentially:
- UI activity drives the experience
- manager class orchestrates system operations
- shell manager executes privileged commands
- shared preferences store user configuration
- Room DB stores app statistics

This structure is understandable but not ideal. It is the structure of a small, focused utility app that has grown over time without a formal architectural layer separation.

### Better architecture

A stronger version would look like this:

- UI layer
  - `MainActivity`, `SettingsActivity`
  - `ViewModel` objects for state exposure
- domain layer
  - `AppSelectionPolicy`
  - `KillRulesEngine`
  - `BackgroundRestrictionPolicy`
  - `PermissionResolver`
- data layer
  - `AppRepository`
  - `PreferencesRepository`
  - `AppStatsRepository`
- system layer
  - `RootAccessProvider`
  - `ShizukuAccessProvider`
  - `ShellCommandExecutor`
- background execution
  - `WorkManager` for scheduled tasks
  - lifecycle-aware handling for long-running actions

This would keep the same feature set but make the app easier to test, easier to reason about, and safer to maintain.

## Recommendations

### Priority 1: Security and shell safety
- centralize shell execution
- validate all input arguments
- reduce string concatenation in privileged commands
- add stronger error logging and recovery paths

### Priority 2: Architecture cleanup
- move business logic from Activity and manager classes
- introduce ViewModels and repositories
- separate data state from UI state

### Priority 3: Persistence modernization
- migrate settings to `DataStore`
- use typed preference models
- prepare a clean migration strategy

### Priority 4: Testing
- add unit tests for rules and policies
- add integration tests around permission fallback handling
- add UI tests for selection and blacklist/whitelist flows

### Priority 5: Release discipline
- use a clean CI pipeline
- build and lint in a reproducible environment
- verify all root/Shizuku flows with release checklists

## Final assessment

This is a valuable and focused utility app with a clear user problem, but its current implementation is more fragile than it should be. The app would benefit significantly from cleaner architecture, safer privileged-command handling, and stronger automated validation.

The key takeaway is simple: the app concept is strong, but the system boundary and architecture need to become more disciplined before it can be considered a robust long-term Android utility.
