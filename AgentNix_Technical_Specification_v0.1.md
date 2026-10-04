# AgentNix — Technical Architecture & Research Specification

**Status:** Draft v0.1 — architecture and research specification  
**Purpose:** define what AgentNix is, what it must guarantee, what it must not attempt yet, and what must be researched before implementation begins.  
**Primary audience:** Codex or another engineering agent tasked with researching the design and producing an implementation plan.  
**Implementation status:** greenfield; no architectural decision in this document should be treated as permission to begin coding before the research phase is complete.

---

## 1. Executive summary

AgentNix is a declarative capability and environment reconciliation layer for autonomous AI agents. Its purpose is not to create a new agent harness and not to make a language model intrinsically more intelligent. Its purpose is to make the **body and environment of an existing agent predictable, reproducible, inspectable, and verifiably functional**.

The central hypothesis behind AgentNix comes from applying ideas from embodied cognition, ecological psychology, and enactivism to AI agents: an agent's effective cognition does not depend only on the model. It also depends on the body through which the model acts and on the environment in which that body operates. In practical agent engineering, this body includes tools, skills, MCP servers, configuration, runtime behavior, executable dependencies, filesystem access, and the interfaces that expose possible actions to the model.

Many current general-purpose agents are placed in environments where important capabilities are only partially known at runtime. A skill may refer to a binary that is missing. An MCP server may be configured but fail to start. A document-editing workflow may rely on a tool that is absent. A model may therefore spend tokens discovering the environment, searching for packages, retrying failed approaches, or improvising workflows that should have been deterministic. Larger models can sometimes brute-force their way through this uncertainty; smaller models degrade much faster.

AgentNix attempts to replace those hidden assumptions with explicit and testable capability contracts. A user should be able to declare something conceptually like:

```nix
programs.agentnix = {
  enable = true;

  agent = {
    type = "hermes";
    executable = "hermes";
  };

  capacities = {
    web-search.enable = true;
    pdf-authoring.enable = true;
  };

  runtime.type = "local";
};
```

and AgentNix should be able to answer, before the agent starts working:

- Is the requested agent actually installed and compatible?
- Was the procedural knowledge required by each capacity materialized where that agent can see it?
- Are all executable dependencies present in the **actual execution environment** used by the agent?
- Are required MCP servers registered and reachable?
- Are required permissions and secret references available without leaking secret values into the Nix store?
- Has any AgentNix-managed mutable resource been changed by the user or by the agent itself?
- Do health checks and smoke tests prove that the capability works instead of merely existing on paper?

The key design principle is:

> **AgentNix turns implicit environmental assumptions into explicit, reproducible, testable agent capabilities.**

---

# 2. Motivation and engineering thesis

## 2.1 The problem: opaque bodies and mysterious affordances

A general-purpose agent usually has at least four layers that determine what it can actually do:

1. the model;
2. the harness and its native tools;
3. procedural knowledge such as skills;
4. the execution environment and its available software/services.

A model may know that editing a PDF is possible while its shell contains none of the software required to do it reliably. A skill may instruct the agent to call a command that is not on `PATH`. An MCP configuration may point to a server that does not launch. The model may know that a browser or web-search system would solve the task while the current harness exposes neither. This creates environmental uncertainty that the model has to resolve cognitively during the task.

That is exactly the kind of work AgentNix should remove from the reasoning loop whenever it can be made deterministic.

The intended analogy is not that software agents literally have biological bodies. The useful engineering claim is narrower: **the interfaces and resources through which an agent can act alter the effective complexity of the task presented to the model**. If the action possibilities are stable and discoverable, the model can devote more of its computation to the user's problem rather than to reconstructing its own operational substrate.

## 2.2 Working hypothesis

AgentNix is built around the following working hypothesis:

> For a fixed agent and task, making capabilities, dependencies, interfaces, and runtime conditions explicit and reliable should reduce avoidable search, retries, tool misuse, and token expenditure while improving task consistency. The effect should become more important as model capability decreases.

This is an engineering hypothesis, not a theorem. AgentNix should eventually be benchmarked against unmanaged environments to test it empirically.

## 2.3 What AgentNix is not

AgentNix is **not**:

- a new autonomous-agent harness;
- an LLM router;
- a universal package manager intended to replace Nix;
- a marketplace for arbitrary community skills;
- a mechanism for rewriting the native tools embedded inside every supported harness;
- a promise that every agent will expose identical functionality;
- an attempt to force mutable agent memory into the immutable Nix store;
- a remote fleet-management platform in the MVP;
- a reason to install and maintain every supported agent itself.

---

# 3. Goals

AgentNix should eventually provide the following properties.

### 3.1 Declarative desired state

A user declares the agent, capacities, runtime, optional extra packages, and references to required credentials. The configuration describes **what should be available**, not a long imperative installation procedure.

### 3.2 Reproducible executable substrate

Where practical, executable dependencies should come from pinned Nix inputs and reproducible closures rather than ad-hoc package-manager commands executed during tasks.

### 3.3 Capability-oriented UX

The principal user-facing abstraction is a **capacity**, not a raw package list. A user should declare `pdf-authoring`, not manually reverse-engineer every executable and script that the capacity needs.

### 3.4 Agent independence through adapters

The AgentNix core must not hard-code Hermes paths, Codex TOML structure, OpenClaw conventions, or any other harness-specific detail. Harness-specific knowledge belongs in agent adapters.

### 3.5 Runtime independence through adapters

The same declared capacity should eventually be realizable in multiple execution environments where that is technically meaningful: local, OCI/Docker, Podman, and later other runtimes.

### 3.6 Safe coexistence with native and user state

AgentNix must not treat an agent profile as its private directory. Native skills, user-installed skills, user configuration, and agent-created procedural memory remain external unless explicitly adopted.

### 3.7 Mutable procedural memory

A skill seeded by AgentNix may become mutable runtime state. AgentNix must detect and preserve such mutation instead of silently replacing it during every rebuild.

### 3.8 Verification before cognition

AgentNix must provide health checks proving that requested capabilities are usable before the model discovers failures during a task.

### 3.9 No secret leakage into Nix

Secret values must never be intentionally embedded in Nix expressions, derivation inputs, generated store files, plans, logs, or state records.

### 3.10 Small-model friendliness

Official AgentNix capacities should minimize unnecessary context, tool-schema complexity, redundant instructions, and environmental uncertainty. Progressive disclosure is preferred over injecting complete manuals into the initial context.

---

# 4. Non-goals for the first implementation

The MVP should explicitly reject scope that is attractive but unnecessary for proving the architecture.

The first implementation should **not** attempt to:

- package every supported agent into nixpkgs or maintain AgentNix-specific packages for all agents;
- support every native skill shipped by every agent;
- import all skills from SkillMP, ClawHub, or other marketplaces;
- support VPS / split-host remote execution;
- solve bidirectional remote synchronization of procedural memory;
- automatically merge arbitrary user/agent modifications into updated upstream skills;
- rewrite or remove native harness tools such as `bash`, `read-file`, or `edit-file`;
- support every Linux distribution through `apt`, `dnf`, `pacman`, APK, AUR, etc.;
- become a secrets manager;
- become a general container orchestrator;
- guarantee compatibility with an agent version that its adapter has never been tested against.

---

# 5. Vocabulary and ontology

Precise terminology is important because the original conceptual model mixes objects that operate at different layers.

## 5.1 Agent

An existing autonomous-agent harness such as Hermes Agent, Codex CLI, OpenClaw, OpenCode, Claude Code, Pi, or another compatible system.

The model itself is not the agent in this specification. The model is one component used by the harness.

## 5.2 Native tool

An action interface implemented directly by the harness, for example shell execution, file reading, file editing, process control, or a built-in browser. Native tools are part of the harness body and should normally remain under the harness's ownership.

AgentNix may inspect native-tool availability when compatibility requires it, but it should not attempt to surgically redefine the harness's internal toolset in the MVP.

## 5.3 Skill

A directory of reusable procedural knowledge following the open Agent Skills convention where possible. At minimum it contains `SKILL.md`; it may additionally contain scripts, references, assets, templates, examples, or other resources.

Skills are not synonymous with capacities. A capacity may contain a skill, but it can also require MCP servers, packages, scripts, permissions, secrets, or runtime features.

## 5.4 Dependency / resource

A software package, executable, data file, browser binary, runtime library, or other resource required for a capacity to function.

A dependency does not automatically become visible to the model. For example, a Python library used internally by an MCP server is a dependency even if the agent never needs to know that it exists.

## 5.5 Affordance

An **action possibility available to the agent through its environment**.

Examples:

- an executable command that the agent can call;
- an MCP tool exposed to the harness;
- a browser interaction interface;
- a writable workspace;
- a native harness tool.

Not every dependency is an affordance. A hidden transitive library may be required for an affordance without being an affordance itself.

## 5.6 MCP server

A process or remote service speaking Model Context Protocol and exposing tools, resources, prompts, or other protocol capabilities to a compatible client/harness.

For AgentNix, the important distinction is primarily operational:

- **local MCP:** typically started as a local process, commonly over stdio;
- **remote MCP:** reached over a network transport such as Streamable HTTP and normally requires no local server installation, though it can require authentication or client configuration.

## 5.7 Capacity

The principal AgentNix abstraction.

A capacity is a **versioned, testable contract that bundles everything required to make a useful class of action reliably available to an agent**.

Conceptually:

```text
capacity
├── metadata
├── procedural knowledge / skill(s)
├── dependencies
├── exposed affordances
├── MCP definitions (optional)
├── scripts/resources/assets (optional)
├── runtime requirements
├── compatibility requirements
├── permissions
├── secret references
├── health checks
└── smoke tests / verification
```

Example: `pdf-authoring` should represent "the agent can reliably create and manipulate PDF documents", not merely "LibreOffice is installed".

## 5.8 Agent adapter

A harness-specific integration layer that translates AgentNix operations into the configuration and filesystem semantics of one agent.

## 5.9 Runtime

The environment in which the agent's actions actually execute and therefore where executable affordances must be present.

A package existing on the host does not satisfy a capacity if the agent executes commands inside an isolated container that cannot see that package.

## 5.10 Runtime adapter

A runtime-specific integration layer responsible for constructing, entering, or validating the environment in which commands and local services actually run.

## 5.11 Managed resource

Any file, configuration fragment, materialized skill, MCP registration, runtime artifact, or other state that AgentNix intentionally created or explicitly adopted and therefore tracks.

## 5.12 Managed mutable resource

A resource whose initial version may be derived from immutable declarative input but whose live copy is intentionally writable and may diverge through user or agent modification.

Agent-created or agent-edited skills are the most important example.

---

# 6. Core architectural principles and invariants

The following are not suggestions. They are intended to become architectural invariants unless research demonstrates that one is technically impossible or harmful.

## 6.1 AgentNix does not own what it did not create or explicitly adopt

Native agent state and user state must be treated as external by default.

If Hermes ships 50 skills and AgentNix adds two, AgentNix should not need to mirror, understand, or manage the original 50 merely to coexist with them.

## 6.2 A capacity owns its implementation closure

If `pdf-authoring` requires `libreoffice`, `qpdf`, helper scripts, and a skill, those requirements belong to the capacity definition.

A normal user should not have to manually repeat internal capacity dependencies in `extraPackages`.

## 6.3 Desired state and live mutable state are different domains

Nix is excellent for producing deterministic desired state and immutable closures. Agent profiles contain mutable state. AgentNix must represent both instead of pretending they are the same thing.

## 6.4 No destructive overwrite of drift without an explicit policy

If a managed mutable skill differs from the version AgentNix last materialized, an update must not silently erase the live version.

## 6.5 Agent-specific details stay behind adapters

The core should not know where Hermes stores skills or how another agent serializes MCP configuration.

## 6.6 Runtime-specific details stay behind runtime adapters

A capacity should express what it needs. The runtime adapter decides how those requirements become available in that runtime.

## 6.7 Verification must execute in the real action context

Checking `command -v qpdf` on the host is meaningless if the agent's terminal runs in a container. Health checks must execute in the same effective runtime the agent will use.

## 6.8 Native APIs and CLIs are preferred over direct config mutation

If a supported agent provides a stable command or API for adding an MCP server or installing a skill, the adapter should prefer it over parsing and rewriting the entire configuration file.

Direct configuration editing is a fallback.

## 6.9 Config mutation must be surgical

When direct editing is unavoidable, AgentNix must change only owned fields/fragments, preserve unrelated user state, validate the result, and write atomically.

## 6.10 Secrets are references, never derivation contents

Secret values must be resolved only at runtime from protected sources.

## 6.11 Plan before apply

AgentNix should be able to describe what it intends to change before making writes.

## 6.12 Idempotency

Applying the same desired configuration twice without external mutation should converge to no additional changes.

---

# 7. Two-plane architecture

A purely declarative Nix implementation is insufficient for all of AgentNix because important target state is mutable and agent-owned configuration may live outside the Nix store. Conversely, replacing Nix with imperative scripts would destroy the reproducibility that motivates the project.

Therefore AgentNix should be designed as two cooperating planes.

```text
                     agentnix configuration
                             │
                             ▼
                  ┌──────────────────────┐
                  │ Declarative plane    │
                  │ Nix evaluation       │
                  │                      │
                  │ - resolve capacities │
                  │ - resolve packages   │
                  │ - build closures     │
                  │ - validate schema    │
                  │ - emit desired-state │
                  │   manifest           │
                  └──────────┬───────────┘
                             │
                       resolved plan
                             │
                             ▼
                  ┌──────────────────────┐
                  │ Reconciliation plane │
                  │ AgentNix reconciler  │
                  │                      │
                  │ - detect agent       │
                  │ - call adapters      │
                  │ - materialize skills │
                  │ - patch config       │
                  │ - manage live state  │
                  │ - run health checks  │
                  │ - record ownership   │
                  └───────┬──────────────┘
                          │
          ┌───────────────┼────────────────┐
          ▼               ▼                ▼
     agent adapter    runtime adapter   state registry
```

The declarative plane should remain pure enough that evaluating the configuration does not need to inspect secrets or mutate an agent profile.

The reconciliation plane is allowed to perform controlled side effects because its entire purpose is to reconcile the resolved desired state with mutable live state.

This architecture also permits AgentNix to run on non-NixOS distributions as long as Nix itself is available.

---

# 8. Desired-state configuration

The exact Nix API is intentionally not frozen yet. Codex must research the most ergonomic implementation pattern. The following expresses semantics, not final syntax.

```nix
programs.agentnix = {
  enable = true;

  profile = "default";

  agent = {
    type = "hermes";
    executable = "hermes";
    manageInstallation = false;
  };

  capacities = {
    web-search = {
      enable = true;
    };

    pdf-authoring = {
      enable = true;
    };
  };

  runtime = {
    type = "local";
  };

  extraPackages = with pkgs; [
    git
  ];
};
```

### Semantics

- `agent.type` selects the adapter.
- `agent.executable` identifies the already-installed harness.
- `manageInstallation = false` makes explicit that the MVP validates the harness but does not own its installation lifecycle.
- `capacities` declare useful abilities, not their internal packages.
- `runtime` identifies where actions execute.
- `extraPackages` is an escape hatch for user-provided affordances not owned by a capacity.

A later version may support multiple named profiles, multiple agents, capacity parameters, runtime variants, and remote topologies.

---

# 9. Capacity contracts

Each official capacity should be represented as a machine-readable contract.

Conceptually:

```nix
{
  name = "pdf-authoring";
  version = "1";
  description = "Create, inspect, transform, and validate PDF documents.";

  compatibility = {
    platforms = [ "linux" ];
    architectures = [ "x86_64" "aarch64" ];
  };

  skills = [ ./skill ];

  dependencies = with pkgs; [
    libreoffice
    qpdf
  ];

  affordances = [
    "command:libreoffice"
    "command:qpdf"
  ];

  permissions = {
    filesystemRead = true;
    filesystemWrite = true;
    network = false;
  };

  secretRefs = [ ];

  healthChecks = [
    { type = "command"; command = [ "libreoffice" "--version" ]; }
    { type = "command"; command = [ "qpdf" "--version" ]; }
  ];

  smokeTests = [
    # exact implementation TBD
  ];
}
```

The contract should answer at least:

1. What useful capacity is being promised?
2. What procedural knowledge is exposed to the agent?
3. What software/resources implement it?
4. Which affordances should become visible?
5. What runtime properties are required?
6. Which agents/runtime adapters can support it?
7. Which permissions are needed?
8. Which secrets are referenced?
9. How can AgentNix prove that it works?

A capacity should fail early if its contract cannot be satisfied.

---

# 10. Skills model

## 10.1 Standard

AgentNix should target the **open Agent Skills specification**, originally developed by Anthropic, rather than describing the format as an Anthropic-only feature.

Official AgentNix skills should use the standard directory model where appropriate:

```text
skill-name/
├── SKILL.md
├── scripts/
├── references/
├── assets/
└── ...
```

Progressive disclosure is a desired property. The model should know that a capacity exists without loading a complete manual into the initial context.

## 10.2 Official AgentNix skills

An AgentNix skill is not valuable merely because its prose is well-written. Official skills should encode tested workflows around deterministic tooling.

For example, a `pdf-authoring` skill should not say only "use Python to make a PDF". It should specify the tested workflow, which tools are available, when to use each, how to validate the result, common failure modes, and any scripts/resources supplied with the capacity.

## 10.3 Third-party marketplaces

Supporting arbitrary skills from public marketplaces is deliberately out of scope for the MVP.

The reason is architectural, not merely workload. Arbitrary community skills introduce an uncontrolled software supply chain into a project whose value proposition is predictability. Skills can contain executable scripts, hidden environmental assumptions, bugs, insecure instructions, or dependencies that conflict with the declared runtime.

A future community registry should therefore require explicit manifests, compatibility metadata, trust levels, validation, and preferably reproducible dependencies.

## 10.4 Native skills

AgentNix must not require a catalog of every skill shipped by every supported agent.

The default rule is:

> **Native skills are agent-owned and left untouched.**

AgentNix may expose optional adapter functionality for disabling a native capability if the harness itself supports a safe, documented mechanism. This is not a prerequisite for initial compatibility.

---

# 11. Ownership and state model

The state model is one of the most important parts of AgentNix.

Every relevant resource should belong conceptually to one of these classes:

```text
native state
    created/maintained by the agent distribution

user state
    created directly by the user or external tooling

agentnix immutable input
    source material produced by Nix / pinned repository content

agentnix managed state
    live state AgentNix owns and may reconcile exactly

managed mutable state
    state originally materialized by AgentNix but intentionally writable
    and therefore capable of becoming learned/user state

adopted state
    pre-existing state the user explicitly asked AgentNix to manage
```

AgentNix should never infer ownership merely because a file happens to live inside a directory it knows about.

## 11.1 State registry

The reconciler should maintain an explicit registry under an XDG-appropriate state directory, conceptually:

```text
$XDG_STATE_HOME/agentnix/
├── profiles/
│   └── default/
│       ├── resources.json
│       ├── last-plan.json
│       └── backups/
└── ...
```

The exact format is a research decision, but each mutable managed resource should record enough metadata to answer:

- which capacity created it;
- which agent adapter materialized it;
- where it lives;
- source version/revision;
- source/upstream hash;
- hash of the version last applied;
- current/live hash when observed;
- ownership class;
- whether it is mutable;
- last successful verification time/result;
- compatibility metadata needed for migration.

Secret values must never be recorded here.

---

# 12. Managed mutable resources and reconciliation

## 12.1 Why normal Nix replacement is insufficient

Some agents use skills as procedural memory. They can create, patch, rewrite, and delete skills after learning workflows. If AgentNix exposes an official skill merely as an immutable symlink into `/nix/store`, such self-improvement becomes impossible. If AgentNix copies the skill into a writable directory but overwrites it on every apply, learned state is destroyed.

Therefore skills that are permitted to evolve must be **materialized** from immutable source into writable runtime state and then reconciled using drift detection.

## 12.2 Minimal three-hash model

For each mutable resource:

```text
upstreamHash
    hash of the currently declared source version

lastAppliedHash
    hash of the version AgentNix last materialized

liveHash
    hash of the current writable resource
```

### Safe update case

```text
liveHash == lastAppliedHash
```

No one modified the live copy since the last apply. If upstream changed, AgentNix may replace/update the resource atomically.

### Local/agent mutation case

```text
liveHash != lastAppliedHash
```

The live resource contains drift, which may represent user customization or learned procedural memory. AgentNix must not overwrite it silently.

### Divergent update case

```text
upstreamHash != lastAppliedHash
AND
liveHash != lastAppliedHash
```

Both upstream and live state changed. This is a conflict.

The MVP should **report the conflict and preserve live state** rather than attempting an unsafe automatic merge.

## 12.3 Deletion semantics

If a previously materialized mutable resource disappears, the deletion itself may be intentional state. AgentNix should classify it as drift rather than blindly recreating it under the default preservation policy.

A future policy system may allow:

- `preserve` — never overwrite drift automatically;
- `enforce` — restore declared state;
- `prompt` — interactive resolution;
- `merge` — future, resource-specific three-way merge where safe.

The MVP default should favor **preservation over destructive enforcement** for mutable procedural memory.

## 12.4 Explicit reset

Restoring an upstream skill over a mutated live skill must require an explicit command/policy such as a future `agentnix reset` or equivalent operation that clearly communicates data loss.

---

# 13. Agent adapters

Agent adapters are the boundary between AgentNix's generic model and the real harness ecosystem.

The core should communicate in generic operations. Each adapter translates those operations into the native semantics of its agent.

## 13.1 Minimum adapter responsibilities

A useful adapter interface may need operations conceptually similar to:

```text
detect()
getVersion()
checkCompatibility()
getCapabilities()
getSkillLocations()
materializeSkill(...)
removeOwnedSkill(...)
registerMcp(...)
unregisterOwnedMcp(...)
readOwnedConfigState(...)
validateConfig(...)
resolveExecutionTopology(...)
runInAgentExecutionContext(...)
healthCheck(...)
```

The exact API is a research task.

## 13.2 Capability matrix

Not all agents support the same extension mechanisms. An adapter should advertise support explicitly rather than allowing the core to assume it.

Conceptually:

```text
Hermes adapter
  skills: yes
  mutable skills: yes
  local MCP: yes
  remote MCP: research exact support/version
  delegated container terminal: yes/research

SomeOtherAgent adapter
  skills: yes
  mutable skills: no/unknown
  local MCP: yes
  remote MCP: yes
  delegated container terminal: no
```

A capacity that requires a feature unavailable in the selected adapter must fail during planning, not during an agent task.

## 13.3 Prefer native management interfaces

Adapter priority should be:

1. official stable CLI/API designed to make the desired change;
2. official include/plugin mechanism that lets AgentNix own an isolated fragment;
3. structured config editing of only the owned subtree;
4. full-file rewriting only as a last resort.

## 13.4 Direct config editing requirements

If an adapter must edit TOML, YAML, JSON, or another config format:

- back up before destructive migration;
- parse structurally rather than regex-replacing arbitrary text;
- preserve unrelated fields;
- preserve comments when the chosen parser and format make that feasible;
- edit only owned keys;
- write to a temporary file first;
- validate the result;
- replace atomically;
- support plan/diff output;
- detect concurrent/unexpected changes where practical.

---

# 14. Runtime model

A central mistake AgentNix must avoid is confusing **host installation** with **agent availability**.

The question is never merely "is package X installed?". The question is:

> **Can the agent invoke the required affordance from the execution context in which this agent actually acts?**

## 14.1 Execution topologies

At least two topologies exist in real agent systems.

### Co-located execution

The harness and its shell/tools execute in the same environment.

```text
runtime
├── agent
└── capacity dependencies
```

### Delegated execution

The harness is on the host but terminal/tool execution is delegated to another runtime such as a container.

```text
host
└── agent
      │
      └── terminal delegation
             │
             ▼
         container
         └── capacity dependencies
```

These topologies cannot be treated as equivalent. An agent adapter and runtime adapter may have to cooperate to determine where each dependency belongs.

## 14.2 Local runtime

The MVP should begin with the local runtime.

Research must determine the cleanest method for exposing a per-profile Nix closure to the agent without unnecessary global pollution. Possible approaches include a profile-specific environment, wrapper, activation mechanism, or adapter-supported environment configuration.

Whatever mechanism is selected must satisfy the invariant that `agentnix doctor` tests the same effective `PATH` and environment that agent commands will actually use.

## 14.3 OCI / Docker / Podman runtime

Container support should not begin by teaching AgentNix how to operate `apt`, `dnf`, `pacman`, APK, and AUR inside arbitrary distributions.

The preferred direction is to investigate constructing reproducible OCI/Docker-compatible images directly from Nix closures using nixpkgs container tooling. This keeps dependency resolution in Nix and avoids recreating a cross-distribution package manager.

Two eventual modes may be useful:

### Managed image

AgentNix constructs the execution image from the declared capacities and packages.

Properties:

- high reproducibility;
- known closure;
- good fit for official capacities;
- preferred mode when possible.

### External image

The user supplies an arbitrary base/image/runtime.

Properties:

- lower guarantees;
- compatibility must be verified;
- may require extra constraints;
- should be deferred until the architecture is stable.

## 14.4 mise

`mise` may be valuable as an optional provider for language/toolchain versions in environments where it is better suited than Nix for a specific workflow.

It should **not** initially become AgentNix's general dependency manager. Nix remains the authoritative declarative substrate. `mise` can be reconsidered later as an adapter/provider when a concrete requirement justifies it.

## 14.5 VPS / remote execution

Remote execution is explicitly deferred.

The original problem remains real: skills/configuration may live beside the local harness while executable affordances live on a remote host. Supporting this correctly requires explicit control-plane, agent-host, and execution-host semantics plus synchronization/security design.

Do not approximate this with "SSH in and install things" during the MVP.

---

# 15. MCP integration

MCP is an important implementation mechanism for capacities but remains below the capacity abstraction.

## 15.1 Local MCP

A local MCP capacity may require:

- a pinned executable/server package;
- command and arguments;
- environment references;
- registration through the selected agent adapter;
- startup/handshake verification;
- optional subprocess lifecycle behavior.

For stdio servers, the executable must exist in the same effective environment from which the harness will spawn it.

## 15.2 Remote MCP

A remote MCP server generally does not require installing the server itself in the local runtime. AgentNix may still need to manage:

- endpoint configuration;
- protocol/transport compatibility;
- authentication references;
- network requirements;
- TLS expectations;
- client registration through the adapter;
- connectivity/handshake checks.

## 15.3 Protocol handling

AgentNix should not implement MCP itself unless a concrete future requirement demands it. Harnesses already act as MCP clients.

AgentNix's job is to configure, provide dependencies for, and verify the integration.

## 15.4 Version and transport compatibility

Adapters/capacities must not assume that every harness supports every MCP protocol version or transport. Planning should fail with a clear compatibility error when the declared MCP configuration cannot be represented by the selected agent.

---

# 16. Secrets and credentials

Secrets require a hard security boundary because values embedded in Nix expressions or derivations can enter the Nix store.

## 16.1 Absolute rule

> **No secret value may enter Nix evaluation, the Nix store, a generated desired-state manifest, logs, plans, or the AgentNix state registry.**

## 16.2 Secret references

Configuration should contain only references, for example conceptually:

```nix
capacities.some-service = {
  enable = true;

  secrets.token = {
    provider = "environment";
    key = "SERVICE_TOKEN";
  };
};
```

or:

```nix
secrets.token = {
  provider = "file";
  path = "/run/secrets/service-token";
};
```

Future providers may include integrations with tools such as sops/agenix, desktop keyrings, password managers, or a user-defined command. Those are references/resolvers, not reasons to serialize the secret into Nix.

## 16.3 Runtime injection

The reconciler/runtime adapter resolves the reference only when the process that needs the secret starts. Secret values should be redacted from diagnostic output.

## 16.4 Missing secret behavior

`agentnix plan` should be able to know that a secret reference is required without reading its value.

`agentnix doctor` may verify that the reference is resolvable but must not print the value.

---

# 17. Security and supply-chain policy

## 17.1 Official capacities are curated software

An official capacity can execute code and therefore must be treated like software, not like harmless prompt text.

Official capacities should aim for:

- pinned sources;
- reproducible dependencies;
- known licenses;
- reviewed scripts;
- explicit permissions;
- compatibility tests;
- health checks;
- minimal hidden network behavior;
- no unpinned `curl | sh`-style installation during tasks;
- no implicit `npx -y latest` in a reproducible path unless there is an explicit reason and warning.

## 17.2 Permissions metadata

Capacity contracts should declare expected permissions even if the MVP cannot enforce every permission technically.

Examples:

```text
network
filesystem read
filesystem write
browser
process spawning
container access
credential reference
```

This enables future policy enforcement and already improves inspection/auditing.

## 17.3 Mutable skill auditing

Because an autonomous agent may modify executable scripts inside mutable skills, future AgentNix versions should consider offering diffs/audit history for mutations.

The MVP only needs to detect drift and preserve it safely.

---

# 18. Health checks, smoke tests, and `agentnix doctor`

Verification is central to the AgentNix thesis.

An installed declaration is not equivalent to a functioning capacity.

AgentNix should provide a diagnostic command conceptually similar to:

```text
$ agentnix doctor

Profile: default
Agent: Hermes
Runtime: local

Agent
  ✓ executable found
  ✓ supported version
  ✓ adapter initialized

web-search
  ✓ skill materialized
  ✓ Hound executable available in agent runtime
  ✓ MCP registration present
  ✓ MCP server starts
  ✓ protocol handshake succeeds
  ✓ expected tools discovered

pdf-authoring
  ✓ skill materialized
  ✓ libreoffice available in agent runtime
  ✓ qpdf available in agent runtime
  ✓ smoke test passed

State
  ✓ no unmanaged overwrite required
  ! pdf-authoring skill has local modifications
    preserved; upstream update available
```

## 18.1 Layers of verification

A capacity may have several check levels:

### Presence check

Does the required file/executable/config entry exist?

### Version check

Does it satisfy compatibility constraints?

### Startup check

Can the server/process start successfully?

### Interface check

Can the expected MCP tools or commands actually be discovered/called?

### Functional smoke test

Can the capacity complete a minimal deterministic operation?

A capacity should use the cheapest check that proves the relevant guarantee. Expensive tests need not run on every invocation.

---

# 19. Planning, reconciliation, and CLI semantics

The exact CLI is not final, but the architecture should support a workflow similar to:

```text
agentnix plan
agentnix apply
agentnix status
agentnix doctor
agentnix diff
```

Potential later commands:

```text
agentnix reset
agentnix adopt
agentnix run
agentnix migrate
```

## 19.1 `plan`

Must be non-destructive. It should:

- evaluate desired state;
- resolve capacity graph;
- verify adapter/runtime compatibility that can be checked safely;
- inspect current ownership/state;
- report additions, updates, removals, conflicts, and drift;
- never reveal secret values.

## 19.2 `apply`

Reconciles safe changes. It must refuse or preserve resources when destructive conflict resolution would be required under the current policy.

## 19.3 `status`

Shows what AgentNix believes it owns, what is healthy, and what has drifted.

## 19.4 `diff`

Shows upstream-vs-live differences for managed mutable text resources where practical.

## 19.5 Failure semantics

A failed apply should leave the profile in a recoverable state. Multi-file changes should be ordered and backed up carefully enough that AgentNix can explain partial completion rather than silently leaving ambiguous ownership.

---

# 20. Configuration mutation safety

Configuration is one of the highest-risk parts of AgentNix because user profiles can contain important settings unrelated to this project.

Rules:

1. AgentNix owns only the keys/fragments it created or explicitly adopted.
2. Removal of a capacity removes only resources recorded as owned by that capacity.
3. Never regenerate an entire user config from an incomplete internal model of the file.
4. Prefer separate include files if the agent supports them.
5. Backups are mandatory before migration that can lose information.
6. A malformed output must never replace a valid config.
7. If external changes occur between read and write, fail safely where detection is feasible.

---

# 21. Candidate official capacities

The following capacities are useful project directions, not all MVP commitments.

## 21.1 `web-search`

Goal: provide reliable web search/fetch/crawl behavior with minimal token waste and structured outputs.

Candidate implementation: Hound / master-fetch as a local MCP server, assuming packaging, architecture support, licenses, and reproducibility are validated during research.

Possible components:

```text
web-search
├── Agent Skill instructions
├── Hound MCP server
├── browser/runtime dependencies if full mode enabled
├── MCP registration
├── startup + handshake health checks
└── a small functional web-fetch/search smoke test
```

The skill should teach efficient use of the available MCP rather than duplicating the MCP schema in prose.

## 21.2 `pdf-authoring`

Goal: provide a tested workflow for PDF creation, inspection, transformation, and validation.

Research should determine the minimal high-quality toolchain. LibreOffice, qpdf, Python libraries, rendering/inspection tools, or a local service such as Stirling PDF are candidates; they should not all be included merely because they exist.

The capacity should be workflow-driven and benchmarked on representative tasks.

## 21.3 Document capacities

`docx-authoring`, `latex-authoring`, and related formats may become separate capacities or share common internal infrastructure. Research should determine whether one giant `documents` capacity would create unnecessary context/dependency weight compared with smaller specialized capacities.

## 21.4 `study`

Primarily procedural knowledge rather than software dependencies. It may compose other capacities such as web research and document generation without duplicating them.

## 21.5 `language-learning`

A more specialized pedagogical capacity. This should come later; it is useful as evidence that capacities do not have to be tool-heavy.

---

# 22. Capacity composition

Capacities should eventually be able to depend on other capacities without copying their implementation.

For example:

```text
study
├── optional dependency: web-search
└── optional dependency: document-authoring
```

The design must avoid loading all composed skill text eagerly. Dependency of implementation does not necessarily mean dependency of context.

Research should distinguish:

- **hard capacity dependency** — cannot function without it;
- **optional enhancement** — works without it but gains functionality;
- **shared resource dependency** — reuses a package/service but not the procedural skill.

---

# 23. Installation policy for agents

The MVP should **not manage installation of the agent harness itself**.

Reasons:

- agents evolve rapidly;
- upstream installation methods differ substantially;
- Nix packages may be missing, stale, or architecture-limited;
- packaging several large agents would become a second project;
- AgentNix can prove its core thesis without owning harness installation.

The selected adapter should instead perform detection:

```text
agent executable not found
→ fail clearly
→ report supported/recommended upstream installation method or documentation
```

A future `manageInstallation = true` mode can be explored per agent if maintainability is acceptable.

---

# 24. Proposed repository boundaries

Codex should research the final implementation language and packaging strategy before creating this structure, but the conceptual separation should resemble:

```text
agentnix/
├── flake.nix
├── flake.lock
├── nix/
│   ├── modules/
│   ├── capacities/
│   └── packages/
├── capacities/
│   ├── web-search/
│   └── pdf-authoring/
├── adapters/
│   ├── agents/
│   │   └── hermes/
│   └── runtimes/
│       └── local/
├── reconciler/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
├── docs/
└── README.md
```

This is not an instruction to create these directories immediately. It is a separation-of-concerns sketch.

A major research question is how much of the core should be Nix code versus a small conventional program used for state reconciliation, config parsing, diagnostics, and atomic operations.

---

# 25. MVP definition

The first milestone should be intentionally narrow.

## 25.1 Supported agent

**Hermes Agent only**, unless research identifies an unexpectedly severe blocker.

Reasons:

- it strongly exercises the mutable-skills problem;
- it is important to the motivating use cases;
- it supports skills and MCP-style extensibility;
- solving Hermes well is a stronger architectural test than beginning with a harness that has almost no mutable extension state.

## 25.2 Runtime

**Local runtime only.**

Container support begins after the local semantics and ownership model are correct.

## 25.3 Capacities

Exactly two serious capacities are enough:

1. `web-search`;
2. `pdf-authoring`.

They stress different parts of the architecture:

- `web-search` tests MCP registration, service dependencies, and procedural guidance;
- `pdf-authoring` tests executable affordances, files, scripts, workflow verification, and output validation.

## 25.4 MVP functionality

The MVP should demonstrate:

```text
Nix config
   ↓
capacity resolution
   ↓
dependency closure
   ↓
Hermes adapter
   ↓
writable skill materialization
   ↓
local runtime availability
   ↓
MCP/config integration when required
   ↓
state ownership registry
   ↓
health checks
   ↓
repeat apply without destructive overwrite
```

## 25.5 MVP success criteria

The MVP is successful if it can prove all of the following:

- a clean profile can acquire both declared capacities without manual configuration;
- the dependencies are actually usable from Hermes's action environment;
- `agentnix doctor` can prove both capacities are functional;
- an AgentNix-provided skill can be edited after installation;
- re-applying unchanged desired state does not erase that edit;
- an upstream skill update plus a local edit is detected as a conflict instead of overwriting the live copy;
- disabling a capacity removes only AgentNix-owned resources and does not remove unrelated Hermes/user state;
- no secret values are necessary for the MVP, but the design prevents future secrets from entering the Nix store;
- all important behavior has integration tests or reproducible fixtures.

---

# 26. Phase roadmap after MVP

The exact ordering may change after research.

## Phase 0 — research and architecture

No implementation beyond disposable experiments.

Outputs:

- architecture decision record;
- adapter feasibility report for Hermes;
- final configuration model;
- state/reconciliation design;
- implementation language decision;
- packaging strategy;
- test strategy;
- risk register;
- staged implementation plan.

## Phase 1 — Hermes + local runtime + two capacities

Build and test the MVP described above.

## Phase 2 — second agent adapter

Add an agent with substantially different configuration semantics, likely Codex or another important target. This tests whether the adapter abstraction is genuine rather than merely a renamed Hermes implementation.

## Phase 3 — managed OCI runtime

Use Nix-built reproducible container images/closures and validate execution-context health checks.

## Phase 4 — additional agents and capacities

Only after the abstraction survives two agents and two runtimes.

## Phase 5 — richer mutable-state workflows

Explore adoption, reset, manual conflict resolution, history, and perhaps safe three-way merges for specific resource types.

## Phase 6 — remote/split execution

Research control-plane / agent-host / execution-host architecture before implementing VPS support.

## Phase 7 — curated community ecosystem

Only if the official capacity format and security model are mature enough to accept third-party extensions safely.

---

# 27. Research questions Codex must answer before implementation

Codex must investigate these questions rather than guessing.

## 27.1 Nix integration

1. What is the best way to expose AgentNix as a reusable Nix/flake interface on NixOS and non-NixOS distributions?
2. Should AgentNix integrate with Home Manager, provide an independent flake/library, or support both?
3. What is the cleanest way to build a per-profile dependency environment without globally installing every capacity package?
4. How should a resolved Nix configuration emit a manifest consumable by the reconciler without introducing impurity?
5. Which `dockerTools`/OCI approach best fits later managed runtimes?

## 27.2 Reconciler implementation

1. Which implementation language minimizes complexity while providing safe TOML/YAML/JSON manipulation, hashing, atomic writes, filesystem operations, and good distribution through Nix?
2. How should transaction boundaries and rollback work?
3. Which XDG directories should contain config, state, cache, backups, and materialized resources?
4. What should the durable state schema look like, and how will migrations work?

## 27.3 Hermes adapter

Verify from current upstream documentation/source:

1. installation detection and version command;
2. exact skill discovery paths and precedence;
3. how bundled, user, external, and agent-created skills interact;
4. supported skill-management CLI/API operations;
5. MCP configuration methods and formats;
6. whether an isolated AgentNix-owned MCP config fragment is possible;
7. execution-environment/container behavior;
8. safe restart/reload semantics after config changes;
9. any built-in skill update logic that could conflict with AgentNix ownership;
10. compatibility/version boundaries the adapter should enforce.

## 27.4 Hound / web-search

1. current maintained source/repository status;
2. license;
3. Python and browser dependencies;
4. architecture support;
5. feasibility of packaging reproducibly with Nix;
6. stdio and HTTP modes;
7. startup/health behavior;
8. whether browser binaries can be provided reproducibly without using imperative installers;
9. exact tool surface and token overhead;
10. whether a smaller official AgentNix wrapper/skill is needed.

## 27.5 PDF workflow

Benchmark candidate toolchains instead of picking by familiarity.

At minimum test:

- text extraction;
- page rendering/visual verification;
- PDF generation;
- conversion from office formats;
- merging/splitting/reordering;
- metadata/structure inspection;
- failure behavior on malformed files;
- output quality.

The result should justify the minimal dependency set for the official capacity.

## 27.6 Agent Skills compatibility

1. verify the current open Agent Skills specification;
2. determine which metadata fields can be used portably;
3. identify harness-specific extensions that must stay in adapters rather than official cross-agent skills;
4. determine how validation should be integrated into AgentNix tests.

## 27.7 Security

1. confirm all paths by which strings can accidentally enter the Nix store;
2. design secret-reference validation without reading values during Nix evaluation;
3. define redaction rules for diagnostics;
4. define minimum trust requirements for official capacity scripts and MCP servers;
5. define how source hashes/locks are surfaced in `plan` and `status`.

---

# 28. Mandatory research deliverables before coding

Codex must **not begin production implementation immediately after reading this document**.

First produce a research package containing:

1. **Architecture assessment** — confirm, challenge, or refine this design.
2. **Current upstream verification** — Hermes, Agent Skills, Nix/container tooling, Hound, MCP assumptions.
3. **Decision log / ADRs** — especially for declarative-vs-reconciler boundaries, state storage, config editing, and implementation language.
4. **Risk register** — technical risks ranked by probability and impact.
5. **MVP implementation plan** — ordered, small milestones with explicit test criteria.
6. **Proposed repository structure** — only after architecture research.
7. **Test plan** — unit, integration, fixture, and end-to-end tests.
8. **Questions / contradictions** — anything in this specification that should be changed before coding.

Only after those deliverables have been reviewed should implementation begin.

---

# 29. Important unresolved design decisions

This document intentionally leaves some questions open.

### 29.1 Exact Nix API

The example configuration is semantic, not final syntax.

### 29.2 Reconciler language

Do not assume Python, Rust, Go, or another language without comparing trade-offs for this specific project.

### 29.3 Home Manager relationship

AgentNix should work outside NixOS, but whether Home Manager becomes an optional integration layer requires research.

### 29.4 Local dependency exposure

A package being present in a Nix closure is insufficient; it must be visible from the true agent execution context. The mechanism is not yet selected.

### 29.5 Mutable-state resolution UX

The MVP can preserve and report conflicts. The long-term user experience for merging/adopting/resetting state remains open.

### 29.6 Capacity granularity

Overly large capacities can recreate the token/dependency bloat AgentNix is meant to fight. Overly small capacities can create fragmentation and configuration overhead. This should be decided empirically.

---

# 30. Architectural test: what should AgentNix make impossible or obvious?

A useful architecture is one that eliminates entire classes of mistakes.

AgentNix should aim to make the following situations impossible or immediately visible:

- a skill references a required executable that the declared runtime does not contain;
- a local MCP is registered but its binary is missing;
- a remote MCP requires credentials but no credential reference exists;
- a capacity declares support for an agent adapter that cannot represent it;
- a rebuild silently erases a skill the agent has modified;
- disabling one capacity deletes unrelated native/user skills;
- an AgentNix config embeds a secret value that would reach the store;
- health checks run on the host while the agent acts elsewhere;
- an official capacity uses an unpinned dependency without making that impurity explicit;
- a config writer replaces a user's entire configuration just to manage one MCP entry;
- a capacity is considered healthy solely because files were copied successfully.

---

# 31. Long-term evaluation and research program

AgentNix originates from an engineering/cognitive hypothesis. The project should eventually evaluate that hypothesis rather than relying only on intuition.

Possible experiments:

## 31.1 Managed vs unmanaged environment

Give the same agent/model a task under:

- an unmanaged generic shell;
- an AgentNix capacity with verified dependencies/workflow.

Measure:

- task success rate;
- number of tool calls;
- failed tool calls;
- retries;
- total tokens;
- wall-clock time;
- output quality;
- manual intervention.

## 31.2 Model-size sensitivity

Repeat with multiple model capability levels.

Expected hypothesis: environmental regularization should provide greater relative benefit to smaller/weaker models because they have less spare reasoning capacity to recover from missing or ambiguous affordances.

## 31.3 Context-cost measurement

Measure the startup/context cost of official AgentNix skills and MCP schemas. A capacity that saves runtime errors but floods every session with thousands of unnecessary tokens would violate the project's small-harness principle.

## 31.4 Capacity regression tests

Official capacities should accumulate representative tasks so updates to skills, packages, or MCP servers can be tested against previous working behavior.

---

# 32. Design philosophy in one example

Suppose the user enables `web-search`.

A poor implementation would do this:

```text
"Here is a long SKILL.md explaining web research.
Hopefully curl, a browser, search APIs, and parsers exist.
The agent can figure it out."
```

AgentNix should instead aim for:

```text
web-search capacity declared
    ↓
compatible agent adapter verified
    ↓
Hound (or selected backend) supplied reproducibly
    ↓
required browser/runtime resources supplied
    ↓
MCP server registered through adapter
    ↓
small procedural skill materialized
    ↓
MCP handshake succeeds
    ↓
expected tool surface discovered
    ↓
functional smoke test succeeds
    ↓
agent starts knowing the capacity exists
```

The model should reason about **the user's question**, not about whether its own body was assembled correctly.

---

# 33. Final project definition

AgentNix should be understood as:

> **A declarative, agent-agnostic capability and environment reconciliation system built on Nix, designed to provide autonomous AI agents with predictable, reproducible, inspectable, and verified action environments while preserving native, user, and learned mutable state.**

Its distinctive contribution is not merely installing packages or copying skills. It is the combination of:

- declarative desired state;
- capability contracts;
- reproducible dependency closures;
- agent adapters;
- runtime adapters;
- ownership-aware reconciliation;
- mutable procedural-memory preservation;
- health checks and smoke tests;
- safe secret references;
- and a design explicitly optimized to reduce environmental uncertainty presented to the model.

If these properties cannot be preserved, the implementation should be reconsidered rather than silently weakening them.

---

# Appendix A — Initial MVP capacity sketch

```text
Profile: default
Agent: Hermes (externally installed)
Runtime: local

Capacity: web-search
  Skill:
    agentnix-web-search
  MCP:
    candidate: Hound
  Runtime resources:
    hound server
    browser resources if required by selected mode
  Verification:
    executable/version
    MCP startup
    handshake
    expected tools
    one deterministic fetch/search fixture

Capacity: pdf-authoring
  Skill:
    agentnix-pdf-authoring
  Runtime resources:
    determined by benchmark/research
  Verification:
    executable versions
    generate/transform fixture
    inspect resulting PDF
    optional render comparison
```

---

# Appendix B — Example ownership scenario

Before AgentNix:

```text
~/.hermes/skills/
├── research-native/        # Hermes-owned
├── devops-native/          # Hermes-owned
└── my-personal-skill/      # user-owned
```

After enabling two capacities:

```text
~/.hermes/skills/
├── research-native/        # Hermes-owned; untouched
├── devops-native/          # Hermes-owned; untouched
├── my-personal-skill/      # user-owned; untouched
├── agentnix-web-search/    # managed mutable
└── agentnix-pdf-authoring/ # managed mutable
```

Hermes later edits `agentnix-web-search`.

The AgentNix registry records drift. A later apply does **not** replace it silently even if a new upstream skill version exists.

---

# Appendix C — Example reconciliation states

```text
CLEAN
  live == lastApplied
  upstream == lastApplied
  action: none

SAFE_UPGRADE
  live == lastApplied
  upstream != lastApplied
  action: atomic update allowed

LOCAL_DRIFT
  live != lastApplied
  upstream == lastApplied
  action: preserve; report local/agent modification

DIVERGED
  live != lastApplied
  upstream != lastApplied
  action: preserve; report conflict

MISSING_MUTABLE
  resource previously existed but live copy is missing
  action: treat as drift under default policy; do not silently recreate

UNMANAGED_COLLISION
  desired target path already exists but registry says AgentNix does not own it
  action: fail safely; require alternate target or explicit adoption
```

---

# Appendix D — External technical facts to verify during Phase 0

These were checked while drafting this specification but must still be re-verified by Codex against current upstream documentation/source immediately before implementation because the ecosystem changes quickly.

1. **Agent Skills** currently specifies a required `SKILL.md`, optional `scripts/`, `references/`, and `assets/`, and progressive disclosure.  
   Source: https://github.com/agentskills/agentskills/blob/main/docs/specification.mdx

2. **Hermes Agent** currently treats `~/.hermes/skills/` as its primary skill source and permits the agent to create/update/delete skills as procedural memory. Its bundled-skill synchronization is documented as respecting local edits/deletions.  
   Sources:  
   https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/skills.md  
   https://github.com/NousResearch/hermes-agent/blob/main/website/docs/reference/skills-catalog.md

3. **Nix secrets:** the Nix store is readable by local users and secrets should not be placed in it; runtime protected-file approaches are recommended.  
   Source: https://nix.dev/manual/nix/2.34/store/secrets

4. **nixpkgs `dockerTools`** can build reproducible Docker-compatible images directly from Nix derivations/closures, including layered/streamed variants, without requiring Docker to perform the build operations.  
   Source: https://nixos.org/manual/nixpkgs/stable/

5. **MCP transport direction:** local process-based stdio and remote Streamable HTTP are the important baseline transports to research for adapters; exact currently supported protocol versions must be discovered per agent.  
   Sources: https://modelcontextprotocol.io/ and current official SDK/specification documentation.

6. **Hound/master-fetch** currently advertises local keyless web search/fetch/crawl through MCP with stdio and HTTP modes, but packaging, current maintained upstream, browser requirements, architecture coverage, and reproducibility must be independently verified before making it an official dependency.  
   Source checked during drafting: https://github.com/dondai1234/master-fetch

---

# Appendix E — Instruction to Codex

Read this specification as a set of **requirements, hypotheses, invariants, and research questions**, not as a request to immediately generate a codebase.

Your first task is to challenge the architecture using current upstream source code and documentation. Identify incorrect assumptions, missing edge cases, and places where the design can be simplified without weakening its guarantees.

Do not optimize for producing many files quickly. Optimize for finding the smallest architecture that can prove the AgentNix thesis with one agent, one runtime, and two real capacities.

When an implementation choice is uncertain, investigate it and record the trade-off rather than silently choosing whatever is easiest to code.

The project owner is learning software architecture through this project and intends to delegate much of the mechanical coding. Therefore your research and implementation plan must make architectural boundaries, data flow, ownership, failure behavior, and tests understandable rather than hiding them behind generated code.
