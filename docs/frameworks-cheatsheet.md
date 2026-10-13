# Prompt frameworks: quick reference

A short reference for the five frameworks used in this challenge. It's not
part of the graded deliverable.

## R-T-F (Role · Task · Format)

**What it is:** the simplest framework. Who the model is, what to do, and
what shape the answer takes.
**Best for:** short, well-defined tasks with a single output artifact, like a
Dockerfile, a script, or an email.
**Avoid when:** there's a lot of input data, or the answer requires reasoning
through trade-offs.

```
Role: You are a <expert>.
Task: <what to produce, with the key constraints>.
Format: <output shape: code only / table / bullet list>.
```

## T-A-G (Task · Action · Goal)

**What it is:** what you need, how to do it, and *why*. The Goal is the
purpose, which helps the model decide edge cases.
**Best for:** analysis or reports where the purpose shapes the answer, like a
cost report or a data query for an audience.
**Avoid when:** you need a specific persona or a strict step-by-step method.

```
Task: <what needs to be delivered>.
Action: <the steps / rules / data to use>.
Goal: <who will use it and for what; what success looks like>.
```

## B-A-B (Before · After · Bridge)

**What it is:** the current state, the desired state, and how to get from one
to the other.
**Best for:** transformations, like modernizing, refactoring, migrating, or
fixing something that exists.
**Avoid when:** there's no "before", meaning you're creating something from
scratch.

```
Before: <current state + what's wrong with it, concretely>.
After: <the target state + its requirements>.
Bridge: <what to do to get there + the deliverable format>.
```

## C-A-R-E (Context · Action · Result · Example)

**What it is:** the situation, what to do, what the deliverable looks like,
and a reference to imitate.
**Best for:** output that must match an existing standard or style, like a
module in the house style or a document following a template.
**Avoid when:** you have no good example to give. Without one, it's just
T-A-G with extra steps.

```
Context: <situation + constraints/rules>.
Action: <what to build>.
Result: <exact deliverable: files, sections, format>.
Example: <reference + what to copy from it>.
```

## R-I-S-E (Role · Input · Steps · Expectation)

**What it is:** who the model is, the data it gets, the method to follow, and
the expected result.
**Best for:** procedures and multi-step analysis with lots of input, like
runbooks, investigations, or diagnostics.
**Avoid when:** the task is simple. The structure becomes overhead.

```
Role: <expert + audience>.
Input: <all the data/artifacts, pasted verbatim>.
Steps: <the analysis method, in order>.
Expectation: <the output format + required sections + criteria>.
```

## Choosing quickly

| Use | When the task is… | Signals in the brief | Typical output | Common pitfall |
|---|---|---|---|---|
| **R-T-F** | Small and clear, with one artifact | "Create a…", few inputs, no trade-offs | Dockerfile, script, email | Cramming lots of data into a thin Task |
| **T-A-G** | Driven by a purpose or audience | A stakeholder, a target number, a decision to support | Report, query, summary | Writing Goal as a format spec instead of a purpose |
| **B-A-B** | Changing something that already exists | "Modernize", "migrate", "fix", a legacy artifact | Refactored code or config, migration plan | A Before that says "old" instead of naming the problems |
| **C-A-R-E** | Following a standard or example | "In the style of", an internal standard, a template | Module or document in the house format | Pasting the Example without saying what to copy |
| **R-I-S-E** | Heavy on data, needing a method | Many artifacts, logs and metrics, a procedure | Runbook, investigation, diagnosis | Steps written as vague topics instead of a method |

## Rules that apply to every framework

1. **Paste source data verbatim.** Paraphrasing is how details get flipped.
2. **Put each piece in its own slot.** Rules go in Context or Input, the
   deliverable goes in Format, Result, or Expectation.
3. **Name the specifics.** Files, thresholds, versions, formats. Anything you
   leave open, the model decides.
4. **End with "list any assumptions you made".** It shows you the decisions
   you left open.
