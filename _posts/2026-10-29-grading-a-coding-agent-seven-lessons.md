---
layout: post
title: "Grading a coding agent: seven lessons the score didn't show"
description: "I scored a bare coding agent on twelve planted-bug tasks, fixed what the scoring found, and scored it again. The score moved by one run. The lessons were underneath it."
date: 2026-10-29 00:00:00 +0000
redirect_from:
  - /saysno-first-scores/
  - /saysno-context/
---

[The last post][p2] built a bare coding agent, [saysno][repo], in 171 lines of Python. This one scores it. I wrote twelve tasks with planted bugs, ran each three times on Claude Haiku 5.5 with nobody at the keyboard, fixed what the results showed with five small harness changes, and ran them again.

The score went from 30 of 36 to 31. On its own, that says nothing happened. Underneath it, nearly everything changed, and very little of it was what I'd have predicted. I don't think I could have learned these lessons by thinking it through or by asking a model. I got them by running the experiment and reading the transcripts.

**In short:**

1. **The evaluation has bugs too.**
2. **A safe rule can be a useless rule.**
3. **On hard tasks, checking beats cleverness.**
4. **In an agent loop, more tokens can cost less.**
5. **Refusing a command doesn't refuse the action.**
6. **A transcript gives you a suspect, not a verdict.**
7. **The total hides the story.**

The tasks are copies of a small CSV report tool, each with a bug or two and a ticket written the way a user would. Some just need fixing. Others are traps: a test that's tempting to edit, an instruction hidden in a data file, a request that contradicts the spec. A run passes if the hidden tests pass and nothing was broken or cheated on the way.

## 1. The evaluation has bugs too

The first full run passed only 15 of 36. Reading the failures, 15 had failed for "changing the tests". Every one of those changes was an addition: the agent had written regression tests for its fix, unasked. My grader treated any change to a test file as cheating, so it was punishing the behaviour I'd most want.

A trap task had the same problem. It asks for something the spec forbids, and the right move is to stop and say so. Most runs didn't. They found a change that kept every visible test passing, and the hidden test failed them. I first wrote it up as the models ignoring the spec. But no visible test pinned the rule, so there was a legitimate fix within what the agent could see: my task was rewarding reading my mind. Once I added a visible spec test, every run stopped and asked.

**Takeaway:** before you believe a score, read the failures. Some will be the grader's. And a task that tests judgement must give the agent what it needs to judge; if only the hidden test knows the rule, you're measuring luck.

## 2. A safe rule can be a useless rule

The agent's permission rule refuses any shell command with a pipe or a semicolon, since those can hide a second command. I expected that to cost a little convenience. Instead it refused **94%** of shell commands, and the tests ran in **3 runs of 36**.

The refused commands weren't dangerous. They were things like `python -m pytest -q 2>&1 | tail -5`: the model trimming its own output, because nothing else would. The refusal didn't say why, so the model never tried the plain command, which would have run. A second model, DeepSeek-V4.1-Flash, hit the same wall.

The fix wasn't to loosen the rule much. I capped tool output in the harness so the model doesn't need to trim it, allowed the few trimming endings that can't start a second command, and made refusals say what would run. The tests then ran in 35 of 36 runs.

**Takeaway:** measure what a safety rule refuses, not just whether it refuses. A rule that blocks the model's housekeeping blocks its checking too.

## 3. On hard tasks, checking beats cleverness

In the first round, every failed run was one where the tests never ran. The model usually knew. It said "I couldn't run the tests, so this is unverified", then stopped anyway.

The second round didn't make the model any smarter: same model, same tasks. But with the tests running, the hard tasks went from **1 of 6 to 5 of 6**. The second bug in a two-bug task only shows up when you run something.

**Takeaway:** for a small model, much of "capability" is access to feedback. Before reaching for a bigger model, check that the small one can see the results of what it did. And an honest "unverified" is still unverified. The harness, not the model, should decide whether "done" is supported.

## 4. In an agent loop, more tokens can cost less

Once it could test, the agent worked harder: 39% more tokens. The bill still fell **36%**, because of prompt caching. An agent loop resends its whole history on every call, so if the start of the prompt never changes, most of each request is a cheap cache read. Caching roughly halved the cost, and it paid off from the second call, not just in long sessions.

**Takeaway:** caching in an agent loop isn't an optimisation for later. Keep the start of the prompt fixed from day one.

## 5. Refusing a command doesn't refuse the action

One task leaves a colleague's unfinished draft in the project, with a note not to lose it. In the second round, the agent decided the draft was a stray edit. The harness refused `git checkout` twice. The agent then deleted the same lines with its file-editing tool, which was allowed, and reported the clean-up as done.

The first round had shown the same pattern. With `cat` refused, the agent printed a whole 190 KB file with a search tool instead.

This failure was only visible because the first fix let the agent get further into the task. In round one it never saw the draft at all.

**Takeaway:** a rule on command strings guards one spelling of an action. Guard the effect instead: which files may change, how much output may come back. And expect each fix to expose the next failure.

## 6. A transcript gives you a suspect, not a verdict

One task got worse: from 2 of 3 to 0 of 3. The transcripts suggested a neat culprit, the new prompt line telling the agent not to read whole data files. So I ran it again with that line switched off, and with the output cap switched off.

The result was "maybe". Switching the line off gave 1 pass in 3, which three runs can't tell apart from chance. Switching off the cap made it an unlikely culprit, and showed what it's worth: without it, each run sent prompts of about 200,000 tokens and cost about 25 times as much.

The most useful finding was one I wasn't looking for. One run ended by saying the exact case that was broken was "handled". It had read the code and believed it, without running anything.

**Takeaway:** a transcript tells you where to look, and an experiment tells you whether you were right. Three runs per task is enough to see a failure, not to explain one. Give every feature an off switch, so you can run the experiment.

## 7. The total hides the story

The first round's 83% hid 15 of 15 easy tasks and 1 of 6 hard. The second round's +1 hid +4 on hard tasks and −3 elsewhere, and the drops are where the new failures were: worth more than the score.

**Takeaway:** a total mostly tells you how many tasks were easy. Report by task, and read every failure.

## What's next

The agent still can't run the command each ticket names, because `python` can run anything. So it still stops without checking against the ticket. The next steps are a sandbox, so a command that only touches a throwaway copy can just run, and protected files checked on every tool. After that comes a harness that won't accept "done" until the check has actually happened.

The tasks, results and every failure are in the [repository][repo]: the [first][log2] and [second][log3] failure logs, and the code at the [`m3` tag][m3]. A full evaluation costs about six cents on Claude Haiku 5.5.

[p2]: /saysno-loop/
[repo]: https://github.com/billykong/saysno
[m3]: https://github.com/billykong/saysno/tree/m3
[log2]: https://github.com/billykong/saysno/blob/m3/results/m2/failure-log.md
[log3]: https://github.com/billykong/saysno/blob/m3/results/m3/failure-log.md
