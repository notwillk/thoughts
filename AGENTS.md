# Agent Instructions

## Primary Focus: PRD (Product Requirements Document)

**Unless explicitly stated otherwise, all user requests refer to the PRD (Product Requirements Document).**

When the user makes a request:
- **Assume they are talking about the PRD** in `prds/hostvault/PRD.md`
- **Implement the description** - actually modify the PRD to reflect the changes they describe
- Do not just discuss or analyze - make the actual changes to the document

### Exceptions
Only treat the following as NOT referring to the PRD:
- **Devcontainer** configuration (`.devcontainer/`)
- **Checksy** configurations (`*.checksy.yaml`)
- **Justfile** (build scripts)
- **Explicitly** mentioned file paths or other documents
- **Agent files** skills (`.agents/skills/*.md`) or hint files (e.g. `AGENTS.md`)

## Repository Structure

- **PRDs**: Located in `prds/` with a subdirectory for each project
  - Main PRD: `prds/secret-server/PRD.md`
  - Include documentation, images, and supporting files in the same directory
  - Prefer text formats over binary ones

- **Build Scripts**: Use `just` for running repo scripts (defined in `justfile`)

## Workflow

1. When user asks for a feature or change → **Update the PRD**
2. When user asks a question → Answer based on PRD content
3. When user mentions specific implementation files → Work on those files directly
4. When in doubt → Ask for clarification

## PRD Maintenance

- Keep the PRD as the single source of truth for product specifications
- Update relevant sections when adding new features
- Maintain consistent formatting and section numbering
- Ensure examples and code snippets are accurate

## Visualizations

- Prefer **Mermaid.js** for creating diagrams (block diagrams, flow charts, sequence diagrams, etc.)
- Mermaid diagrams are text-based, version-controllable, and render natively in most modern documentation platforms
