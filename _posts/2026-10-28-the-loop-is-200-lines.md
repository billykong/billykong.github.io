---
layout: post
title: "The loop is 200 lines. Everything else is the harness."
description: "A minimal coding agent in 171 lines of Python on the Anthropic SDK: the loop, four tools, one permission rule, and a scripted model that tests it all without an API key. Then its first real fix, for under a tenth of a cent."
redirect_from: /saysno-loop/
---

[The first post][p1] in this series described a coding agent as a loop: build the context, call the model, check and run the tool it asks for, add the result, repeat. This post is that loop, written out. It's the bare starting point the rest of the series improves on, one measured failure at a time.

The whole agent is **171 lines of Python** on the Anthropic SDK, with no other dependencies. The code for this post is the [`m1` tag][m1] of [saysno][repo].

**In short:**

- **Four files:** the loop, four tools, one permission rule and a command-line prompt.
- **Small choices that matter:** an edit tool that refuses ambiguous edits, file reads a range at a time, paths that can't leave the project, and shell commands that need an allow list or a yes.
- **Tested without a model:** a scripted model replays fixed replies, so the loop runs in CI with no API key and no bill.
- **First real run:** Claude Haiku 5.5 fixed a planted bug in 4 model calls, for $0.0008. One of its commands was refused on the way, and it carried on without it.

## The shape

| File | Lines | What it does |
| --- | --- | --- |
| `loop.py` | 54 | Stages 1 to 4, model settings, a transcript of every run |
| `tools.py` | 69 | `read_file`, `edit_file`, `bash`, `grep` |
| `policy.py` | 19 | Which tool calls run, and which need a yes |
| `cli.py` | 27 | The prompt: `uv run saysno --dir PROJECT "the task"` |

(Non-blank, non-comment lines; a two-line `__main__.py` makes 171. CI fails if the total goes over 200.)

There's no framework. The SDK sends one request and returns one reply; everything else is visible here. (The Anthropic SDK also has a tool runner that drives this loop for you. Writing it by hand is the point of the exercise.)

## The loop

This is the heart of it, from [`loop.py`][loop]:

```python
def send(self, text):
    """Run one user request to completion; return the final reply."""
    self.messages.append({"role": "user", "content": text})
    while True:
        reply = self.client.messages.create(max_tokens=16000, system=SYSTEM, tools=TOOLS,
                                            messages=self.messages, **self.params)
        self.messages.append({"role": "assistant", "content": reply.content})
        if reply.stop_reason != "tool_use":
            return reply
        results = [self.call(b) for b in reply.content if b.type == "tool_use"]
        self.messages.append({"role": "user", "content": results})
```

(The logging and printing lines are left out here.)

Two details that are easy to get wrong:

- **Append the reply as it came back**, not just its text. The model's tool requests, and its thinking when thinking is on, have to go back to the API unchanged on the next call.
- **Return every tool result in one message.** A model can ask for several tools at once. Their results go back together, each matched to its request by ID.

The loop ends whenever the model stops asking for tools, whatever the reason: it finished, it ran out of output space, or it refused. There's no turn limit and no cost limit yet. Those come later in the series, after I've seen why they're needed.

## Stage 3: the only place that says no

```python
def call(self, block):
    """Stage 3: the harness decides whether the requested tool runs."""
    decision = policy.decide(block.name, block.input)
    if decision == "ask":
        decision = "allow" if self.ask(block.name, block.input) else "deny"
    if decision == "deny":
        output, is_error = "denied: the user did not allow this command", True
    else:
        try:
            output, is_error = self.tools.run(block.name, block.input), False
        except ToolError as e:
            output, is_error = f"error: {e}", True
    return {"type": "tool_result", "tool_use_id": block.id, "content": output, "is_error": is_error}
```

(Again without the logging lines.)

A refusal isn't a crash. It goes back to the model as an ordinary result, so the model can try something else. So does a tool error: a missing file or an ambiguous edit becomes text the model can read and act on.

The rule itself, from [`policy.py`][policy], is short:

```python
ALLOW = ["ls", "cat", "head", "tail", "wc", "grep", "find", "pwd", "pytest", "python -m pytest",
         "git status", "git diff", "git log", "git show"]
# Anything that chains, redirects or substitutes can hide a second command.
SHELL_META = re.compile(r"[;&|`$<>\n]")
```

File tools always run, because they're confined to the project directory (below). A shell command runs without asking only if it starts with something on the allow list *and* contains no shell metacharacters. Without the second check, `ls; rm -rf ~` would pass as "starts with `ls`". Anything else asks the person at the keyboard.

This rule is deliberately cautious, and string matching on shell commands is a speed bump, not a wall. The real wall is a sandbox, which comes later. As the next post shows, it's also far too strict, in a way I didn't expect.

## The four tools

The tools are where most of the small design decisions live. From [`tools.py`][tools]:

- **`read_file(path, offset, limit)`** returns numbered lines, 200 at a time by default, and says how many lines are left. The model reads what it needs instead of loading whole files. This is rung 3 of the context ladder from the first post, in its simplest form.
- **`edit_file(path, old, new)`** replaces exact text, and refuses unless that text appears exactly once:

  ```python
  count = text.count(old) if old else 0
  if count != 1:
      raise ToolError(f"old text found {count} times in {path}; it must match exactly once")
  ```

  Rewriting whole files is the obvious alternative, and the dangerous one: a model that rewrites a 400-line file to change one line can silently drop the other 399. An exact replace fails loudly instead, and the message says how to fix the request.
- **`bash(command)`** runs in the project directory with a two-minute timeout, and returns the exit code with the output.
- **`grep(pattern, path)`** returns `path:line:text`, so the model can find code without reading the repository.

Every path goes through one check:

```python
def path(self, p):
    full = (self.root / p).resolve()
    if not full.is_relative_to(self.root):
        raise ToolError(f"{p} is outside the project directory")
    return full
```

`resolve()` follows `..` and symbolic links before the check, so `../outside.txt` and `sub/../../x` are both refused.

## Testing an agent without a model

An agent's tests shouldn't need an API key or cost money. saysno's tests use a **scripted model**: a stand-in client that replays a fixed list of replies, built from the SDK's real message types.

```python
client = ScriptedClient(
    [text("Looking."), tool("grep", pattern="def mean")],
    [tool("read_file", path="stats.py")],
    [tool("edit_file", path="stats.py", old="(len(values) - 1)", new="len(values)")],
    [tool("bash", command="python -m pytest -q")],
    [text("Fixed the off-by-one in mean().")],
)
```

That test runs the real loop and real tools against a copy of a small buggy project. It checks that the bug is fixed, that each tool result went back matched to its request, and that the transcript recorded every step. Other tests check that a refused command never runs, that tool errors reach the model, that paths can't escape, and that the agent stays within its line budget. That's 32 tests at this stage, and none of them need the network.

## The first run

The first real task is one planted bug. `mean()` divides by `len(values) - 1`, so the average of 10 and 20 comes out as 30. The ticket says only: *"The price summary is wrong: for prices 10 and 20 it reports a mean of 30. Fix it."*

Claude Haiku 5.5, with thinking off:

```text
I'll start by finding the price summary code.
→ grep {"pattern": "mean|summary"}
→ bash {"command": "ls -R | head -100"}
  run `ls -R | head -100`? [y/N]
The `ls` command was denied, but grep gave me what I need. Let me read `stats.py`.
→ read_file {"path": "stats.py"}
The bug is on line 5: the mean divides by `len(values) - 1` instead of `len(values)`.
→ edit_file {"path": "stats.py", "old": "    return sum(values) / (len(values) - 1)", "new": "    return sum(values) / len(values)"}
→ bash {"command": "python -m pytest -q"}
Fixed. [...] The test suite passes (3 passed).
```

It took **4 model calls, 5,369 input tokens and 569 output tokens: about $0.0008.** The full transcript is [in the repository][transcript].

Two things to notice. The model asked for two tools in one reply, a search and a directory listing. The listing had a pipe in it, so the harness asked first. The answer was no, and the model carried on with what the search had found. And `python -m pytest -q` was on the allow list, so it ran without asking.

One run on an easy bug shows the loop works, not how well. For that, it needs a set of tasks and a score, and a way to run it without anyone at the keyboard to answer "y/N". That's the next post: twelve planted bugs, two models, and the shell rule above turning out to say no to almost everything.

[p1]: /saysno/
[repo]: https://github.com/billykong/saysno
[m1]: https://github.com/billykong/saysno/tree/m1
[loop]: https://github.com/billykong/saysno/blob/m1/saysno/loop.py
[policy]: https://github.com/billykong/saysno/blob/m1/saysno/policy.py
[tools]: https://github.com/billykong/saysno/blob/m1/saysno/tools.py
[transcript]: https://github.com/billykong/saysno/blob/m1/results/m1/first-task-haiku.jsonl
