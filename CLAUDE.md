# Development loop

The main chat is the **orchestrator**. It talks to the user and runs the subagents in `.claude/agents/`. It does not write production code or tests itself; it delegates, passes context between agents, and keeps the user informed.

Subagents can't talk to each other or to the user. Pass everything an agent needs (the task, the plan, earlier reports) in its prompt.

For trivial tasks (typos, one-line fixes, questions), skip the loop and just do the work. When unsure, ask the user.

## Steps

1. **Plan**: run `planner` with the task.
   - If it returns open questions, ask the user and re-run the planner with the answers.
   - Show the user a short summary of the plan and whether tests are recommended. Wait for approval before continuing.

2. **Tests** (only if the plan or the user says tests are needed): run `go-test-writer` with the plan's Behavior and Tests sections. Tests are written before the code, from the plan, so they are expected to fail at first.

3. **Code**: run `coder` with the plan and the list of test files.

4. **Review**: run `reviewer` with the plan.
   - `changes requested`: send the blocking findings to `coder`, then review again.
   - At most **3** review rounds. If findings remain after that, stop and ask the user how to proceed.
   - If the coder reports a wrong test, ask the user whether to fix the test (via `go-test-writer`) or the plan.

5. **Attention**: run `attention-gatherer` on the final code.

6. **Report to the user**: what was built, checks that passed, minor review findings left open, and the attention list.

## Code style

All code-writing and reviewing agents follow the skills in `.claude/skills/`:

Code style:

- `minimal-comments`: comment only tricky or counter-intuitive code.
- `go-naming`: Go naming conventions.
- `go-nesting-depth`: nesting depth of 3 or less (strong recommendation).
- `go-oop`: encapsulation, behavior on the owning struct, polymorphism through interfaces.
- `go-interfaces`: interfaces over direct dependencies, declared in the file that uses them.

Architecture:

- `service-layers`: Controller, Usecase, Repository layers and entities.
- `go-train-di`: `@Injection`/`@Inject` codegen.
- `go-train-controller`: HTTP handlers on the framework's controller contract.
- `go-train-repository`: DB access on `store.Store[T]`.
