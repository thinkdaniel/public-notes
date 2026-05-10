# Conventional Commits

## What are Conventional Commits?

Conventional Commits is a specification for adding human and machine-readable meaning to commit messages. It provides an easy set of rules for creating an explicit commit history, making it easier to write automated tools on top of it.

The Conventional Commits specification is inspired by, and based heavily on, the Angular Commit Message Guidelines.

## Structure

The commit message should be structured as follows:

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

### Components Breakdown

- **type**: A noun describing the kind of change (required)
- **scope**: A noun describing the section of the codebase affected (optional)
- **description**: A short summary of the code changes (required)
- **body**: A longer explanation of the motivation for the change (optional)
- **footer**: Information about breaking changes or issue references (optional)

## Common Types

| Type       | Description                                               | Example                                         |
| ---------- | --------------------------------------------------------- | ----------------------------------------------- |
| `feat`     | A new feature for the user                                | `feat: add user authentication`                 |
| `fix`      | A bug fix                                                 | `fix: resolve login timeout issue`              |
| `docs`     | Documentation changes                                     | `docs: update API documentation`                |
| `style`    | Code style changes (formatting, missing semicolons, etc.) | `style: fix indentation in user service`        |
| `refactor` | Code changes that neither fix a bug nor add a feature     | `refactor: extract validation logic to utility` |
| `perf`     | Performance improvements                                  | `perf: optimize database queries`               |
| `test`     | Adding or updating tests                                  | `test: add unit tests for user service`         |
| `build`    | Changes to build system or dependencies                   | `build: update webpack configuration`           |
| `ci`       | Changes to CI/CD configuration                            | `ci: add GitHub Actions workflow`               |
| `chore`    | Maintenance tasks                                         | `chore: update dependencies`                    |
| `revert`   | Reverting a previous commit                               | `revert: revert "feat: add user auth"`          |

## Examples

### Basic Examples

```bash
# Feature addition
feat: add shopping cart functionality

# Bug fix
fix: prevent racing of requests

# Documentation update
docs: correct spelling of CHANGELOG

# Style change
style: remove empty line

# Refactoring
refactor: move user validation to separate module

# Performance improvement
perf: improve caching mechanism for user data
```

### Examples with Scope

```bash
# Scope specifies the area of change
feat(auth): add OAuth2 integration
fix(ui): resolve button alignment issue
docs(api): add endpoint documentation
test(user): add integration tests for user creation
```

### Examples with Body

```bash
feat: add user profile management

Allow users to update their profile information including
name, email, and profile picture. The changes are validated
on both client and server side.

Closes #123
```

### Breaking Changes

```bash
feat!: send an email to the customer when a product is shipped

BREAKING CHANGE: The shipping notification feature now requires
an email service to be configured. See migration guide for details.
```

## Benefits of Using Conventional Commits

### 1. **Automated Versioning**

- Automatically determine semantic version bumps
- Generate changelogs from commit messages
- Create releases based on commit types

### 2. **Clear Communication**

- Standardized format makes commits easier to read
- Team members instantly understand the nature of changes
- Historical context is preserved

### 3. **Better Tooling Integration**

- CI/CD pipelines can make decisions based on commit types
- Automated testing can be triggered by specific types
- Code review tools can categorize changes

### 4. **Improved Project Management**

- Link commits to issues and pull requests
- Track feature development progress
- Generate release notes automatically

### 5. **Enhanced Collaboration**

- Consistent format across team members
- Easier code reviews
- Better understanding of project evolution

## How to Enforce Conventional Commits

### 1. Git Hooks with Commitizen

Install Commitizen to guide commit message creation:

```bash
# Install globally
npm install -g commitizen

# Install adapter
npm install -g cz-conventional-changelog

# Configure
echo '{ "path": "cz-conventional-changelog" }' > ~/.czrc
```

Usage:

```bash
# Instead of git commit, use:
git cz
```

### 2. Commit Message Validation with commitlint

Install and configure commitlint:

```bash
# Install commitlint
npm install --save-dev @commitlint/{config-conventional,cli}

# Create configuration
echo "module.exports = {extends: ['@commitlint/config-conventional']}" > commitlint.config.js
```

Add to package.json:

```json
{
  "husky": {
    "hooks": {
      "commit-msg": "commitlint -E HUSKY_GIT_PARAMS"
    }
  }
}
```

### 3. GitHub Actions Validation

Create `.github/workflows/commitlint.yml`:

```yaml
name: Commitlint
on: [push, pull_request]

jobs:
  commitlint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      - uses: wagoid/commitlint-github-action@v5
```

### 4. Pre-commit Hooks

Install pre-commit and configure:

```bash
# Install pre-commit
pip install pre-commit

# Create .pre-commit-config.yaml
cat > .pre-commit-config.yaml << EOF
repos:
  - repo: https://github.com/compilerla/conventional-pre-commit
    rev: v2.1.1
    hooks:
      - id: conventional-pre-commit
        stages: [commit-msg]
        args: [optional, feat, fix, docs, style, refactor, perf, test, build, ci, chore, revert]
EOF

# Install the hooks
pre-commit install --hook-type commit-msg
```

### 5. IDE Integration

#### VS Code Extensions

- **Conventional Commits**: Helps write conventional commit messages
- **Git Commit Plugin**: Provides commit message templates

#### JetBrains IDEs

- **Git Commit Template Plugin**: Provides conventional commit templates

### 6. Team Guidelines

Create a `CONTRIBUTING.md` file with commit guidelines:

```markdown
## Commit Message Format

We follow the Conventional Commits specification. Each commit message should be structured as follows:

<type>[optional scope]: <description>

Examples:

- feat: add user authentication
- fix: resolve memory leak in cache
- docs: update installation guide
```

## Advanced Usage

### Semantic Release Integration

Configure automatic releases based on conventional commits:

```bash
npm install --save-dev semantic-release
```

Create `.releaserc.json`:

```json
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    "@semantic-release/changelog",
    "@semantic-release/npm",
    "@semantic-release/git",
    "@semantic-release/github"
  ]
}
```

### Custom Types

You can define custom types for your project:

```javascript
// commitlint.config.js
module.exports = {
  extends: ["@commitlint/config-conventional"],
  rules: {
    "type-enum": [
      2,
      "always",
      [
        "feat",
        "fix",
        "docs",
        "style",
        "refactor",
        "perf",
        "test",
        "build",
        "ci",
        "chore",
        "revert",
        "security",
        "deps", // Custom types
      ],
    ],
  },
};
```

## Best Practices

1. **Keep descriptions concise**: Aim for 50 characters or less
2. **Use imperative mood**: "add feature" not "added feature"
3. **Don't capitalize the first letter** of the description
4. **No period at the end** of the description
5. **Use the body for context**: Explain the "why" not the "what"
6. **Reference issues**: Use "Closes #123" or "Fixes #456"
7. **Mark breaking changes**: Use `!` after type or `BREAKING CHANGE:` in footer

## Common Mistakes to Avoid

- ❌ `Fix bug` → ✅ `fix: resolve null pointer exception in user service`
- ❌ `Added new feature` → ✅ `feat: add user profile management`
- ❌ `fix: Fix login issue.` → ✅ `fix: resolve login timeout issue`
- ❌ `updated docs` → ✅ `docs: update API documentation`

## Conclusion

Conventional Commits provide a standardized way to write commit messages that benefit both humans and automated tools. By adopting this specification, teams can improve their development workflow, automate version management, and maintain a clear project history.

The key to success is consistency and team adoption. Start with basic enforcement tools and gradually introduce more sophisticated automation as the team becomes comfortable with the format.
