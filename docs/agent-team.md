# Agent team

To build Mona's Project Pulse dashboard, I am using a team of four custom agents defined under `.github/agents/`, orchestrated with GitHub Copilot CLI in a Codespace.

## Orchestrator
- **Model:** Claude Opus 4.7 (copilot)
- **Responsibility:** Coordinates the Planner, Coder, and Designer agents. Breaks the Project Pulse request into phases, assigns non-overlapping file scopes, decides what can run in parallel vs. sequentially, and reports the integrated outcome. Does not implement anything itself.
- **Definition:** `.github/agents/orchestrator.agent.md`

## Planner
- **Model:** Claude Opus 4.7 (copilot)
- **Responsibility:** Researches the repository, docs, dependencies, and edge cases, then produces an ordered implementation plan with file assignments, dependencies, parallelizable work, and validation expectations. Does not write code.
- **Definition:** `.github/agents/planner.agent.md`

## Coder
- **Model:** GPT-5.5 (copilot)
- **Responsibility:** Implements the Project Pulse application logic within the file scope assigned by the Orchestrator, including support files like `.vscode/launch.json` when assigned, following consistent, testable, deterministic patterns.
- **Definition:** `.github/agents/coder.agent.md`

## Designer
- **Model:** Gemini 3.1 Pro (copilot)
- **Responsibility:** Owns UI/UX, accessibility, information architecture, and visual design for the dashboard — project cards, status badges, priority treatment, responsive layout, and deterministic CSS hooks like `.dashboard` and `.project-card`.
- **Definition:** `.github/agents/designer.agent.md`

All four agents are used together via GitHub Copilot CLI running in a Codespace, which orchestrates delegation between them; none of the agents stage, commit, or push changes themselves — git operations remain under the learner's control.
