# Ralph for Amp

An iterative AI agent loop system for [Amp](https://ampcode.com). Ralph creates self-referential feedback loops where Amp repeatedly works on a task until completion.

## What is Ralph?

Ralph is a development methodology based on continuous AI agent loops. Named after Ralph Wiggum from The Simpsons, it embodies the philosophy of persistent iteration despite setbacks.

**Core concept**: A simple loop that repeatedly feeds Amp a prompt, allowing it to iteratively improve its work until a completion condition is met. Each iteration sees the modified files from previous iterations, creating a self-referential feedback loop.

```
┌─────────────────────────────────────────┐
│           Ralph Loop                    │
│                                         │
│  ┌──────┐    ┌──────┐    ┌──────────┐   │
│  │Prompt│───▶│ Amp  │───▶│Check Done│   │
│  └──────┘    └──────┘    └────┬─────┘   │
│      ▲                        │         │
│      │         No             │         │
│      └────────────────────────┘         │
│                                         │
│              Yes ──▶ Exit               │
└─────────────────────────────────────────┘
```

## Installation

```bash
# Clone or copy the ralph directory
# Add to your PATH:
export PATH="$PATH:/path/to/ralph"

# Or create a symlink:
ln -s /path/to/ralph/ralph /usr/local/bin/ralph
```

**Requirements:**
- `amp` CLI installed and authenticated
- `jq` for JSON processing
- `git` (optional, for checkpointing)

## Quick Start

```bash
# Simple inline prompt
ralph start -p "Build a REST API for todos with CRUD operations and tests. Output RALPH_STATUS:done when complete."

# With a prompt file
ralph start -f prompts/my-task.md -d "API implementation"

# With test-based completion
ralph start -p "Fix all failing tests" --done-cmd "npm test"

# Check status
ralph status

# Follow logs
ralph tail -f

# Resume if interrupted
ralph resume
```

## Commands

### `ralph start [options]`

Start a new Ralph loop.

| Option | Short | Description |
|--------|-------|-------------|
| `--prompt <text>` | `-p` | Inline prompt text |
| `--prompt-file <path>` | `-f` | Path to prompt file |
| `--description <text>` | `-d` | Job description (used in job ID) |
| `--max-iterations <n>` | `-i` | Maximum iterations (default: 50) |
| `--max-minutes <n>` | `-t` | Maximum runtime in minutes (default: 120) |
| `--done-string <text>` | `-s` | Completion signal (default: `RALPH_STATUS:done`) |
| `--done-file <path>` | | Complete when this file exists |
| `--done-cmd <command>` | | Complete when this command succeeds |
| `--no-git` | | Disable git checkpointing |
| `--checkpoint-every <n>` | | Git checkpoint frequency (default: 1) |
| `--job-id <id>` | | Custom job ID |

### `ralph resume [jobId]`

Resume a paused or failed job. If no job ID is provided, resumes the most recent job.

### `ralph status [jobId]`

Show status of a job. If no job ID is provided, shows the most recent job.

### `ralph list`

List recent jobs with their status.

### `ralph cancel <jobId>`

Cancel a running job.

### `ralph tail [jobId] [-f]`

View job logs. Use `-f` to follow in real-time.

## Completion Conditions

Ralph supports three types of completion detection:

### 1. Status String (Default)

Amp outputs a special line when done:

```
RALPH_STATUS:done
```

This is the most reliable method. Customize with `--done-string`:

```bash
ralph start -p "..." --done-string "TASK_COMPLETE"
```

### 2. File Marker

Complete when a specific file exists:

```bash
ralph start -p "Create a .done file when finished" --done-file ".done"
```

### 3. Command Success

Complete when a command exits with code 0:

```bash
ralph start -p "Fix all tests" --done-cmd "npm test"
ralph start -p "Build the project" --done-cmd "cargo build --release"
```

### Combining Conditions

All conditions use "any" mode by default - the loop stops when ANY condition is satisfied.

## Verification (Trust but Verify)

Amp might claim completion but not actually finish the work. Use verification to ensure requirements are met:

### Expected Files

Require specific files to exist before accepting completion:

```bash
# Single file
ralph start -p "Create a config file" --expect-files "config.yaml"

# Multiple files (comma-separated)
ralph start -p "Build API with tests" --expect-files "api.go,api_test.go"

# Glob patterns
ralph start -p "Create Go tests" --expect-files "*_test.go"
ralph start -p "Build React components" --expect-files "src/components/*.tsx"
```

### Verification Command

Run a command that must succeed for completion:

```bash
# Tests must pass
ralph start -p "Fix all tests" --verify-cmd "go test ./..."

# Linting must pass
ralph start -p "Clean up code" --verify-cmd "npm run lint"

# Build must succeed
ralph start -p "Fix type errors" --verify-cmd "tsc --noEmit"
```

### Combining Verification

Use both for maximum confidence:

```bash
ralph start \
  -p "Build a REST API with comprehensive tests" \
  --expect-files "*_test.go,go.mod,main.go" \
  --verify-cmd "go test ./... && go build"
```

### How Verification Works

1. Amp signals completion (outputs `RALPH_STATUS:done`)
2. Ralph runs verification checks:
   - Are all expected files present?
   - Does the verification command pass?
3. If verification **passes**: Job completes successfully
4. If verification **fails**: Ralph continues iterating, telling Amp it didn't actually finish

This prevents false completions where Amp claims success but didn't deliver.

## Writing Effective Prompts

### 1. Clear Completion Criteria

❌ Bad:
```
Build a todo API and make it good.
```

✅ Good:
```
Build a REST API for todos.

Requirements:
- CRUD endpoints (GET, POST, PUT, DELETE)
- Input validation
- Error handling
- Tests with >80% coverage

When ALL requirements are met, output: RALPH_STATUS:done
If still working, output: RALPH_STATUS:working
```

### 2. Incremental Goals

❌ Bad:
```
Create a complete e-commerce platform.
```

✅ Good:
```
Build an e-commerce API in phases:

Phase 1: User authentication (JWT, tests)
Phase 2: Product catalog (CRUD, search)
Phase 3: Shopping cart (add/remove items)
Phase 4: Order processing

Complete each phase with tests before moving on.
Output RALPH_STATUS:done when all phases complete.
```

### 3. Self-Correction Instructions

```
Implement feature X using TDD:

1. Write failing tests first
2. Implement the feature
3. Run tests: npm test
4. If tests fail, debug and fix
5. Refactor if needed
6. Repeat until all tests pass

Output RALPH_STATUS:done when all tests pass.
```

### 4. Safety Limits

Always use `--max-iterations` as a safety net:

```bash
ralph start -f task.md --max-iterations 30 --max-minutes 60
```

Include fallback instructions in your prompt:

```
After 20 iterations, if not complete:
- Document what's blocking progress
- List what was attempted
- Suggest alternative approaches
- Output RALPH_STATUS:done with a summary
```

## Git Integration

Ralph automatically creates git checkpoints after each iteration (unless disabled with `--no-git`).

Benefits:
- Track exactly what changed in each iteration
- Easy to revert if something goes wrong
- Audit trail of AI-generated changes

```bash
# Disable git checkpoints
ralph start -p "..." --no-git

# Checkpoint every 5 iterations instead of every iteration
ralph start -p "..." --checkpoint-every 5
```

Commit messages follow this format:
```
ralph: iteration 5 - API implementation

Job: ralph-2025-01-11T10-32-45_api-implementation
```

## File Structure

```
.amp/
  ralph/
    jobs/
      ralph-2025-01-11T10-32-45_api-impl.json   # Job state
    logs/
      ralph-2025-01-11T10-32-45_api-impl.log   # Job logs
```

### Job State File

Each job has a JSON state file containing:
- Job metadata (ID, description, timestamps)
- Prompt configuration
- Completion conditions
- Safety limits
- Iteration metrics
- Git checkpoint info

This enables:
- **Resume**: Pick up where you left off after interruption
- **Status**: Check progress at any time
- **Debugging**: Inspect what happened in each iteration

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `RALPH_DIR` | `.amp/ralph` | Base directory for Ralph data |
| `MAX_LOG_SIZE` | `10485760` | Max log file size (10MB) |
| `MAX_LOG_BACKUPS` | `3` | Number of rotated logs to keep |

## Examples

### Build a Feature

```bash
ralph start -p "
Add user authentication to the Express API:

1. Install passport and passport-jwt
2. Create User model with email/password
3. Add /auth/register endpoint
4. Add /auth/login endpoint returning JWT
5. Add middleware to protect routes
6. Write tests for all auth endpoints

Run tests after each change: npm test
Output RALPH_STATUS:done when all tests pass.
" -d "user-auth" --done-cmd "npm test" --max-iterations 25
```

### Fix Bugs

```bash
ralph start -p "
Fix all TypeScript errors in the project.

1. Run: npx tsc --noEmit
2. Fix each error one at a time
3. Repeat until no errors

Output RALPH_STATUS:done when tsc exits cleanly.
" --done-cmd "npx tsc --noEmit"
```

### Refactoring

```bash
ralph start -f prompts/refactor-auth.md \
  --description "auth-refactoring" \
  --max-iterations 40 \
  --max-minutes 90 \
  --done-cmd "npm test && npm run lint"
```

### Long-running Task (Overnight)

```bash
ralph start -f prompts/big-migration.md \
  --description "database-migration" \
  --max-iterations 100 \
  --max-minutes 480 \
  --checkpoint-every 5

# Check progress from another terminal
ralph status
ralph tail -f
```

## Philosophy

Ralph embodies several key principles:

1. **Iteration > Perfection**: Don't aim for perfect on first try. Let the loop refine the work.

2. **Failures Are Data**: Each failed iteration provides information. Use it to improve.

3. **Operator Skill Matters**: Success depends on writing good prompts, not just having a good model.

4. **Persistence Wins**: Keep trying until success. The loop handles retry logic automatically.

5. **Trust but Verify**: Use completion commands (`--done-cmd`) to objectively verify success.

## When to Use Ralph

**Good for:**
- Well-defined tasks with clear success criteria
- Tasks requiring iteration (getting tests to pass)
- Greenfield implementations you can walk away from
- Tasks with automatic verification (tests, linters, type checks)

**Not good for:**
- Tasks requiring human judgment or design decisions
- One-shot operations
- Tasks with unclear success criteria
- Production debugging (use targeted debugging instead)

## Troubleshooting

### Job gets stuck

```bash
# Check status
ralph status

# View recent logs
ralph tail

# Cancel and resume with adjusted parameters
ralph cancel <jobId>
ralph start -f same-prompt.md --max-iterations 20
```

### Amp not outputting completion signal

Add explicit instructions in your prompt:
```
CRITICAL: You MUST include one of these lines in EVERY response:
- RALPH_STATUS:done (if task is complete)
- RALPH_STATUS:working (if still in progress)
```

### Git checkpoint failures

```bash
# Run without git
ralph start -p "..." --no-git

# Or fix git state first
git status
git stash  # if needed
```

## Credits

Inspired by [Geoffrey Huntley's](https://ghuntley.com/ralph/) Ralph Wiggum technique for Claude Code.

## License

MIT
