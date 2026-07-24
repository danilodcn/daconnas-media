# Repository Guidelines

## Code Quality

- Keep cyclomatic complexity low.
- Prefer small functions with a single responsibility.
- Use early returns to avoid deeply nested control flow.
- Favor declarative configuration over procedural code when practical.
- Do not introduce abstractions unless they reduce real complexity.

## Language

- Write all code, identifiers, filenames, CLI messages, and technical documentation in English.

## Comments

- Add comments only when they explain a non-obvious decision or constraint.
- Keep comments concise and always write them in English.
- Do not add comments that merely restate the code.

## Scripts

- Keep shell scripts small, linear, and focused on command orchestration.
- Use Python instead of shell when branching, state management, validation, error handling, or data processing makes a shell script difficult to understand or maintain.
