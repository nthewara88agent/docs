# Spec Kit: GitHub's Framework for Spec-Driven Development

*Published: February 14, 2026*

## Introduction

GitHub has released [Spec Kit](https://github.com/github/spec-kit), an open-source toolkit that fundamentally changes how we approach software development with AI. Instead of "vibe coding" where you throw prompts at an AI and hope for the best, Spec Kit introduces **Spec-Driven Development (SDD)** — a structured methodology that puts specifications at the centre of your workflow.

## What is Spec-Driven Development?

For decades, code has been king. Specifications were scaffolding — something we built and discarded once the "real work" of coding began. PRDs guided development, design docs informed implementation, diagrams visualised architecture. But these were always subordinate to the code itself.

**Spec-Driven Development flips this relationship.**

Specifications don't serve code — code serves specifications. Your PRD isn't just a guide; it's the source that *generates* your implementation. Technical plans aren't documents that inform coding; they're precise definitions that *produce* code.

This shift eliminates the gap between specification and implementation. When specifications generate code, there's no translation loss — only transformation.

## Why Does This Matter Now?

Three trends make Spec-Driven Development both possible and necessary:

1. **AI capabilities have reached a threshold** — Natural language specifications can now reliably generate working code. This isn't about replacing developers; it's about amplifying their effectiveness.

2. **Software complexity is exploding** — Modern systems integrate dozens of services, frameworks, and dependencies. Keeping all pieces aligned through manual processes is increasingly difficult.

3. **The pace of change accelerates** — Requirements change rapidly. Pivoting is no longer exceptional — it's expected. Traditional development treats changes as disruptions; SDD treats them as normal workflow.

## How Spec Kit Works

Spec Kit provides a CLI tool (`specify`) and a set of slash commands that work with popular AI coding agents like Claude Code, Cursor, GitHub Copilot, Windsurf, and many others.

### The Workflow

1. **Install the CLI**
   ```bash
   uv tool install specify-cli --from git+https://github.com/github/spec-kit.git
   ```

2. **Initialise your project**
   ```bash
   specify init my-project --ai claude
   ```

3. **Establish project principles** (`/speckit.constitution`)
   Define your governing principles — code quality standards, testing requirements, UX consistency, performance targets.

4. **Create the specification** (`/speckit.specify`)
   Describe *what* you want to build and *why*. Focus on outcomes, not implementation details.

5. **Create a technical plan** (`/speckit.plan`)
   Provide your tech stack preferences. The AI generates an implementation plan with documented rationale for every decision.

6. **Break down into tasks** (`/speckit.tasks`)
   Generate actionable task lists from your implementation plan.

7. **Execute implementation** (`/speckit.implement`)
   The AI builds your feature according to the plan.

### Available Commands

| Command | Description |
|---------|-------------|
| `/speckit.constitution` | Create project principles and development guidelines |
| `/speckit.specify` | Define requirements and user stories |
| `/speckit.plan` | Create technical implementation plans |
| `/speckit.tasks` | Generate actionable task lists |
| `/speckit.implement` | Build the feature according to plan |
| `/speckit.clarify` | Clarify underspecified areas before planning |
| `/speckit.analyze` | Cross-artifact consistency analysis |
| `/speckit.checklist` | Generate quality checklists ("unit tests for English") |

## Supported AI Agents

Spec Kit works with a wide range of AI coding tools:

- **Claude Code** ✅
- **GitHub Copilot** ✅
- **Cursor** ✅
- **Windsurf** ✅
- **Gemini CLI** ✅
- **Codex CLI** ✅
- **Amazon Q Developer CLI** ⚠️ (limited slash command support)
- And many more including Amp, Roo Code, Kilo Code, and Qoder

## Core Philosophy

### Specifications as the Lingua Franca
The specification becomes the primary artifact. Code is just its expression in a particular language and framework.

### Executable Specifications
Specifications must be precise, complete, and unambiguous enough to generate working systems.

### Continuous Refinement
Consistency validation happens continuously. AI analyses specifications for ambiguity, contradictions, and gaps as an ongoing process.

### Bidirectional Feedback
Production reality informs specification evolution. Metrics, incidents, and operational learnings become inputs for specification refinement.

### Branching for Exploration
Generate multiple implementation approaches from the same specification to explore different optimisation targets — performance, maintainability, UX, cost.

## Practical Example

Let's say you want to build a photo album application:

**Step 1: Specify**
```
/speckit.specify Build an application that helps me organise photos in separate albums. 
Albums are grouped by date and can be reorganised by drag and drop. Albums cannot be 
nested. Within each album, photos display in a tile-like interface.
```

**Step 2: Plan**
```
/speckit.plan Use Vite with minimal libraries. Vanilla HTML, CSS, and JavaScript 
where possible. Images stored locally with metadata in SQLite.
```

**Step 3: Tasks & Implement**
```
/speckit.tasks
/speckit.implement
```

The AI handles the translation from your intent to working code, with full traceability back to your original requirements.

## Benefits for Teams

- **Version-controlled specifications** — Treat specs like code, with branches, reviews, and merges
- **Consistency at scale** — Organisational constraints (database standards, auth requirements, deployment policies) integrate automatically
- **Rapid pivots** — Change a requirement in the PRD, and affected implementation plans update systematically
- **Parallel exploration** — Generate multiple implementations to explore different approaches

## Getting Started

```bash
# Install
uv tool install specify-cli --from git+https://github.com/github/spec-kit.git

# Check your environment
specify check

# Create a new project
specify init my-project --ai claude

# Or initialise in current directory
specify init . --ai copilot
```

## Conclusion

Spec Kit represents a shift in how we think about software development. Instead of starting with code and hoping it matches intent, we start with clear specifications and generate code that inherently aligns with our goals.

This isn't about removing developers from the equation — it's about freeing them to focus on creativity, critical thinking, and the hard problems that matter. The mechanical translation from "what we want" to "how it's built" becomes systematic and reliable.

If you're working with AI coding assistants and want more predictable, traceable outcomes, Spec Kit is worth exploring.

---

**Resources:**
- [GitHub Repository](https://github.com/github/spec-kit)
- [Video Overview](https://www.youtube.com/watch?v=a9eR1xsfvHg)
- [Documentation](https://github.github.io/spec-kit/)
