# Tool Quality Standards

Standards for generating CLI tools and wrappers.

## Audience

Tools are built for AI assistants to operate. Design for structured, parseable output.

## Required

- **Stack:** Bun + TypeScript. Not Node.js, not Python.
- **Architecture:** Library-first — core logic in `lib/`, thin CLI wrapper in `cli.ts`
- **Args:** Manual parsing (`process.argv`), no frameworks (no yargs, no commander)
- **Output:** Always JSON, structured `{success, data, error}` pattern
- **Types:** Strict TypeScript, no `any`
- **Errors:** JSON to stderr, non-zero exit code
- **Help:** `--help` flag with usage, commands, options, examples

## Preferred

- **Dependencies:** Zero when possible. When required, pin exact versions (no `^` or `~`)

## Anti-Patterns

- Argument parsing frameworks
- Plain text output
- Mixed CLI and library concerns
- Hardcoded paths
- Caret/tilde version ranges
