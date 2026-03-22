# AGENTS.md - LinearAI Working Guide

This repository is reserved for Linear-oriented tooling and experiments. Right now it is mostly empty, so treat it as a deliberate placeholder rather than as a finished product.

## Goal

- provide a clean workspace for Linear integrations
- focus on useful free-tier workflows instead of building a generic clone
- keep the repo small and purposeful until the first real use case is defined

## Current Idea And Progress

- Intended purpose:
  connect to Linear and make practical use of its free-tier capabilities
- Current state:
  empty or effectively unused
- Progress level:
  idea stage only

## Initial Setup Requirements

- do not add a large framework before the first use case is defined
- start with one of these minimal setups:
  Node.js 20+ with TypeScript scripts
  or Python 3.11+ with a small API / CLI
- add a real `README.md` and `.env.example` before significant implementation
- expected secret:
  `LINEAR_API_KEY`

## Environments

- local development:
  scripts or a tiny local service
- staging:
  not defined
- production:
  not defined

## Dependencies

- Linear API
- optional local persistence:
  SQLite
- optional scheduling / automation:
  cron, task scheduler, or a lightweight job runner

## Backend Need

- backend required now:
  no
- backend likely later:
  yes, but only lightweight
- likely backend shape:
  small Node or FastAPI service for webhook intake, caching, or recurring sync jobs

## Development Plan

1. Define the first concrete workflow before writing code.
   Examples: daily digest, stale issue review, label hygiene, roadmap export, team activity summary.
2. Build read-only integrations first.
3. Add safe write actions only after the repo proves useful.
4. Add local persistence if repeated API calls or history views become necessary.
5. Add a UI only if the scripts become hard to operate from the command line.

## End Goal

The end goal is a focused Linear operations toolset that saves real time. If no clear recurring workflow emerges, keep this repo minimal or archive it instead of turning it into an unfocused side project.
