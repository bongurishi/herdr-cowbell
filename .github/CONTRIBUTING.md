# Contributing to herdr-cowbell

Thanks for your interest in contributing to herdr-cowbell!

Whether you're fixing a bug, improving documentation, or proposing a feature, contributions are welcome. Please follow the guidelines below to keep development safe and consistent.

## Development setup

### Requirements

- Node.js 18 or newer.
- npm is not required; the project has no external dependencies or build step.
- Linux or macOS.
- herdr installed for integration and smoke testing.

### Get the source

```sh
git clone https://github.com/dvoraj75/herdr-cowbell.git
cd herdr-cowbell
```

Create a working branch before making changes:

```sh
git switch -c fix/short-description
```

There is no dependency installation or build step.

## Running tests

Run the complete test suite using Node.js's built-in test runner:

```sh
node --test
```

Run tests after making changes and before opening a pull request.

If you add or modify functionality, include tests covering the relevant behavior.

## Sandbox safety — mandatory

**Never run herdr against your live session while developing or testing herdr-cowbell.**

Cowbell renames terminal tabs and interacts with the herdr server. Testing against a live session can modify your actual tab names and interfere with your workflow.

Always use an isolated sandbox herdr server.

Before testing:

1. Set up a separate herdr sandbox environment.
2. Ensure it does not share your live session or server.
3. Use isolated configuration and state directories.
4. Run the plugin and integration tests only against the sandbox.
5. Verify that your live session remains untouched.

See [PLAN.md](../PLAN.md), under **Testing**, for the project's sandbox setup instructions.

Do not skip this requirement, even for small changes.

## Smoke testing

In addition to unit tests, changes affecting plugin integration should pass the herdr smoke test.

### Run the smoke test locally

1. Open the repository's CI workflow under `.github/workflows/`.
2. Locate the step named **herdr smoke test**.
3. Review the exact setup and command used by that step.
4. Reproduce the same smoke-test procedure locally using your isolated sandbox.
5. Never substitute your live herdr session for the sandbox.

The CI workflow is the source of truth for the smoke-test command and required environment.

### CI and weekly canary

The project also uses a weekly canary against the latest herdr release.

When changing plugin integration, daemon lifecycle, hooks, actions, or tab-renaming behavior:

- Check the regular CI smoke-test result.
- Check the weekly latest-herdr canary when available.
- Investigate compatibility failures before merging.
- Do not claim compatibility with a version that has not been verified.

If the canary exposes a regression, include the failure details in your pull request.

## Branch conventions

Use short, descriptive branch names.

| Prefix | Purpose | Example |
|---|---|---|
| `feature/` | New functionality | `feature/custom-status-icons` |
| `fix/` | Bug fixes | `fix/tab-rename-conflict` |
| `docs/` | Documentation | `docs/update-installation` |
| `test/` | Test improvements | `test/config-validation` |
| `refactor/` | Internal improvements | `refactor/daemon-state` |

Keep changes focused. Avoid combining unrelated features, fixes, and refactoring in one pull request.

## Commit conventions

Use Conventional Commits.

Format:

```text
type: short description
```

Common types:

| Type | Purpose |
|---|---|
| `feat` | New functionality |
| `fix` | Bug fixes |
| `docs` | Documentation changes |
| `test` | Test changes |
| `refactor` | Internal code improvements |
| `chore` | Maintenance tasks |

Examples:

```text
feat: add custom status icon support
fix: prevent duplicate tab icons
docs: update installation instructions
test: cover preset validation
refactor: simplify daemon state handling
```

Use clear descriptions and keep each commit focused.

## Code guidelines

- Follow the existing project structure and coding style.
- Keep the plugin lightweight and dependency-free.
- Avoid introducing unnecessary dependencies or build tools.
- Validate configuration input and handle errors safely.
- Avoid changing existing behavior unless the change requires it.
- Add or update tests for changed functionality.
- Preserve compatibility with the documented minimum herdr version.
- Document user-visible changes.

## Pull request process

Before opening a pull request:

1. Make sure your branch is up to date.
2. Run `node --test`.
3. Run the relevant smoke test in an isolated sandbox.
4. Verify that you have not interacted with your live herdr session.
5. Update `CHANGELOG.md` when the change requires a changelog entry.
6. Review your changes for unrelated modifications.

Your pull request should include:

- A link to the related issue.
- A clear description of what changed and why.
- Tests performed and their results.
- Confirmation that the sandbox safety check was completed.
- Confirmation that `CHANGELOG.md` was updated, or an explanation if no update is needed.

## Reporting bugs and requesting features

- Use GitHub Issues to report bugs and request enhancements.
- Use GitHub Discussions for questions and general support.
- For security vulnerabilities, follow [SECURITY.md](../SECURITY.md).

Please provide enough information for maintainers to reproduce and investigate reported problems.

## Code of Conduct

All contributors are expected to follow our [Code of Conduct](../CODE_OF_CONDUCT.md).

Thank you for helping improve herdr-cowbell!
