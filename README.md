# Ralph

An iterative AI agent loop for [Amp](https://ampcode.com). Ralph runs your prompt over and over until the job is done, the heat death of the universe, or you hit max iterations. Whichever comes first.

## What is Ralph?

Ralph is what happens when you realize AI agents are pretty good at fixing their own mistakes if you just... keep asking them to.

Named after Ralph Wiggum from The Simpsons, it embodies the philosophy of cheerful persistence in the face of repeated failure. Like Ralph, it doesn't get discouraged. It just keeps going. Eventually, something works.

```
┌─────────────────────────────────────────┐
│           Ralph Loop                    │
│                                         │
│  ┌──────┐    ┌──────┐    ┌──────────┐   │
│  │Prompt│───▶│ Amp  │───▶│ Done?    │   │
│  └──────┘    └──────┘    └────┬─────┘   │
│      ▲                        │         │
│      │         Nope           │         │
│      └────────────────────────┘         │
│                                         │
│              Yep ──▶ 🎉                 │
└─────────────────────────────────────────┘
```

## Installation

```bash
# Add to your PATH:
export PATH="$PATH:/path/to/ralph"

# Or create a symlink:
ln -s /path/to/ralph/ralph /usr/local/bin/ralph
```

**Requirements:**
- `amp` CLI installed and authenticated
- `jq` for JSON processing
- `git` (optional, for checkpointing your regrets)

## Quick Start

```bash
# Simple inline prompt
ralph start -p "Build a REST API for todos. Output RALPH_STATUS:done when complete."

# With a prompt file
ralph start -f prompts/my-task.md -d "API implementation"

# With test-based completion (recommended for the trust-impaired)
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

Start a new Ralph loop. Watch it go. Maybe get coffee.

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
| `--no-git` | | Disable git checkpointing (live dangerously) |
| `--checkpoint-every <n>` | | Git checkpoint frequency (default: 1) |
| `--job-id <id>` | | Custom job ID |

### `ralph resume [jobId]`

Resume a paused or failed job. Hope springs eternal.

### `ralph status [jobId]`

Show status of a job. "Still working" is the most common answer.

### `ralph list`

List recent jobs with their status.

### `ralph cancel <jobId>`

Show mercy and cancel a running job.

### `ralph tail [jobId] [-f]`

View job logs. Use `-f` to follow in real-time like a nervous parent.

## Completion Conditions

Ralph needs to know when to stop. Left unchecked, it will loop forever like a golden retriever chasing its tail.

### 1. Status String (Default)

Amp outputs a magic phrase when done:

```
RALPH_STATUS:done
```

Customize with `--done-string`:

```bash
ralph start -p "..." --done-string "I_AM_FINISHED_TRUST_ME"
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

This is the "trust but verify" option. Highly recommended.

## Verification (Because AI Lies)

Amp might claim completion but not actually finish the work. Shocking, I know. Use verification to keep it honest:

### Expected Files

Require specific files to exist before accepting completion:

```bash
# Single file
ralph start -p "Create a config file" --expect-files "config.yaml"

# Multiple files (comma-separated)
ralph start -p "Build API with tests" --expect-files "api.go,api_test.go"

# Glob patterns
ralph start -p "Create Go tests" --expect-files "*_test.go"
```

### Verification Command

Run a command that must succeed:

```bash
# Tests must pass (not just claim to pass)
ralph start -p "Fix all tests" --verify-cmd "go test ./..."

# Linting must pass
ralph start -p "Clean up code" --verify-cmd "npm run lint"

# Types must check
ralph start -p "Fix type errors" --verify-cmd "tsc --noEmit"
```

### How Verification Works

1. Amp signals completion (outputs `RALPH_STATUS:done`)
2. Ralph runs verification checks
3. If verification **passes**: Job completes 🎉
4. If verification **fails**: Ralph continues, telling Amp "nice try, but no"

This prevents the classic "I'm done!" / "No you're not" loop from happening silently.

## Writing Effective Prompts

The quality of your prompt determines everything. Garbage in, infinite loop out.

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

Always use `--max-iterations`. Unless you enjoy explaining your AWS bill.

```bash
ralph start -f task.md --max-iterations 30 --max-minutes 60
```

Include fallback instructions:

```
After 20 iterations, if not complete:
- Document what's blocking progress
- List what was attempted
- Suggest alternative approaches
- Output RALPH_STATUS:done with a summary

(Better to admit defeat than loop until the sun explodes)
```

## Git Integration

Ralph automatically creates git checkpoints after each iteration. This is your "oh no" insurance policy.

```bash
# Disable git checkpoints
ralph start -p "..." --no-git

# Checkpoint every 5 iterations instead of every iteration
ralph start -p "..." --checkpoint-every 5
```

## File Structure

```
.amp/
  ralph/
    jobs/
      ralph-2025-01-11T10-32-45_api-impl.json   # Job state
    logs/
      ralph-2025-01-11T10-32-45_api-impl.log   # Job logs (the saga)
```

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `RALPH_DIR` | `.amp/ralph` | Base directory for Ralph data |
| `MAX_LOG_SIZE` | `10485760` | Max log file size (10MB) |
| `MAX_LOG_BACKUPS` | `3` | Number of rotated logs to keep |

### Long-running Task (Overnight to keep your house warm)

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

1. **Iteration > Perfection**: First attempts are usually wrong. That's fine. Try again.

2. **Failures Are Data**: Each failed iteration teaches something. Probably.

3. **Persistence Wins**: Keep trying. Eventually the tests pass or you hit max iterations.

4. **Trust but Verify**: Use `--done-cmd`. AI is optimistic about its own abilities.

5. **Operator Skill Matters**: Ralph is only as good as your prompts. Sorry.

## When to Use Ralph

**Good for:**
- Well-defined tasks with clear success criteria
- Tasks requiring iteration (getting tests to pass)
- Greenfield implementations you can walk away from
- Tasks with automatic verification (tests, linters, type checks)
- Going to lunch while code writes itself

**Not good for:**
- Tasks requiring human judgment or taste
- One-shot operations (just use Amp directly)
- Tasks with unclear success criteria ("make it better")
- Production debugging (please don't)

## Troubleshooting

### Job gets stuck

```bash
# Check status
ralph status

# View recent logs
ralph tail

# Cancel and try again with different parameters
ralph cancel <jobId>
ralph start -f same-prompt.md --max-iterations 20
```

### Amp not outputting completion signal

Be more explicit in your prompt:

```
IMPORTANT: You MUST include one of these lines in EVERY response:
- RALPH_STATUS:done (if task is complete)
- RALPH_STATUS:working (if still in progress)

I cannot stress this enough. Please. I'm begging you.
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

Inspired by [Geoffrey Huntley's](https://ghuntley.com/) Ralph Wiggum technique for Claude Code.

## License

MIT
