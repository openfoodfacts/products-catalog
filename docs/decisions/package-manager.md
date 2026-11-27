# Choice of Package Manager

## Context and Problem Statement

A package manager keeps track of dependencies to ensure reproducible builds and a simple developer onboarding experience.

## Decision Drivers

- widely used
- supported by dependabot

## Considered Options

- [Poetry](https://python-poetry.org/)
- [uv](https://docs.astral.sh/uv/)

## Decision Outcome

Chosen option: "uv", because it is a more modern choice, is widely referenced in the FastAPI documentation and has been adopted on other newer Open Food Facts projects.

## Pros and Cons of the Options

### Poetry

- Good: Widely used
- Good: Used on some existing Open Food Facts projects
- Neutral: Supports plugins
- Bad: Can be slow

### uv

- Good: Modern, fast and well adopted
- Good: Used on newer Open Food Facts projects
- Good: Supports inline script metadata for standalone scripts

