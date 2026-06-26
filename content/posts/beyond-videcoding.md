+++
author = "Andreas Lay"
title = "Beyond Vibecoding: Spec Driven Development with OpenSpec & Open Code Review"
date = "2026-06-26"
description = "LLM Driven Development Speedups While Keeping Code Quality High"
tags = ["openspec", "open-code-review", "claude", "spec-driven-development"]
categories = ["Spec-Driven"]
ShowToc = true
TocOpen = true
+++

{{< callout >}}
This blog post is based on my personal experience of adopting OpenSpec when building a new greenfield project & using for legacy codebases. It's a tool that genuinely improves my workflow and I believe many teams can benefit from its adoption.
{{< /callout >}}

## What's OpenSpec and Why Might You Need It?



[OpenSpec](https://github.com/Fission-AI/OpenSpec/blob/main/docs/getting-started.md) is basically a collection of skills / commands (read: markdown files) which will create specification files for you as the basis for you implementation. Think of it as a fancier `plan` mode!

[Open Code Review](https://github.com/alibaba/open-code-review) can help you ensure quality of both your specs & implementation by running multiple review agents in parallel. It can minimize false positive findings by letting the independent review agents discuss their findings.

**Why?** 

1. **Given a good spec state-of-the-art models can one-shot even complex tasks.** You can implement much quicker and guarantee quality at the same time. You might not be 10x faster, but 2-5x.

2. **Specs live in your repository.** You commit them along your code. If done right, you have a **full audit trail** of any decision made: Not just what is documented (that can be inferred from the code itself), but also the **why**. Your proposal and design are archived as markdown documents. So ideally, when you ask yourself in 12 months _Why did choose DuckDB over Pandas again?_, it's documented in the corresponding proposal & design.

**Prerequisites** 
1. You need experience, you need to know what you want to build, or at least be able to identify during the explore phase what you **don't** want. 

2. You need to be able to communicate your requirements and identify what context the LLM is missing if it goes off the rails. In cases where you know exactly what you want - as you might have built it many times before, with small variations, or generally just have the experience - you can build it fast & in better quality than you would have by hand.

## The 5-Step Spec-Driven Workflow

### TL;DR: explore → propose → review → apply → review


1. `/opsx:explore` your idea: Refine and ground in the codebase, give as much context as needed → **most of your time should be spend here!**
2. `/opsx:propose` your spec: Create specification docs - the basis for your implementation → **does not require humand input; the explore phase's output is its input**
3. `/ocr:review` your proposal: Catch unspecced edge cases, refine your spec → **needs human attention**
4. `/opsx:apply` your proposal: Implement your feature → **needs no or limited human attention**
5. `/ocr:review` your implementation: Ensure code quality & find bugs → **needs human attention**


### Step 1: /opsx:explore - Refine your idea with idea

Step 1 is where you should spend most of your time. When you have a general idea, but need to figure out what the best architecture is use `/opsx:explore`:

```bash
/opsx:explore I want to add Dagster to orchestrate this repo's jobs. Dagster should:
- Submit batch jobs
- Monitor / poll these jobs async
- Support AWS Batch & Azure Batch as production backends and Docker as a backend for local development
```

OpenSpec will ask dive deep into your codebase and continually ask you questions. The more context you give it here the better:

- Architecture guidelines or even the full architecture you have in mind
- Technologies to use or to consider
- Reference repositories
- Related tickets
- etc.

{{< callout >}}
Your idea here can be a big feature or component of your final product. It does not need to be small — if you want to break it up into smaller chunks, you can still ask it to do this at the end of your session. Your explore session can also be cross-repository if you have downstream or upstream dependencies.
{{< /callout >}}

Important things to keep in mind when exploring:

- Answer any question you are being asked. If a question does not make sense to you, then your agent is usually missing context that might be obvious to you but it cannot infer. Ask it to slow down, explain itself or give it more context that might clear up this question

- If a suggestion seems overly complex, call it out. Ask why it's needed, for easier alternatives and potential trade-offs

**Generally:** Explore mode works best when you 1. know what you want to build or 2. are good at recognosing when the LLM goes off in an overly complex direction. You need to be able to judge suggested architectures and overrule nonsensical LLM suggestions.

### Step 2: /opsx:propose - Create the Proposal / Specs

Once all open questions are answered and enough context was provided, `/opsx:explore` will tell you that you're now ready to create the proposal via `/opsx:propose`. It generates a change folder under `openspec/` with the proposal, design decisions, and task breakdown:

```
openspec/
└── dagster-orchestration/
    ├── .openspec.yaml
    ├── proposal.md
    ├── design.md
    ├── tasks.md
    └── specs/
        └── spec.md
```

I recommend:
- Read `proposal.md`. It should document the context: 
  - What's the current state?
  - Why do we need this?
  - Reference team internal discussions
  - etc.
- Read `design.md`. It documents your architecture decisions:
  - Review each enumerated design decision
  - Make sure everything marked as **deferred** is actually deferred and should really not be in scope of this proposal
  - Look at the open questions: Ideally there are no open questions as you have answered all of them before
- Take at least a quick look at `tasks.md`:
  - Your agent will walk through these steps one by one
  - Generally though if your `design.md` matches your expectation, `tasks.md` will be good as well


### Step 3: /ocr:review - Review the Proposal

As proposals are already very detailed, they're easy to review. I usually use [open-code-review](https://github.com/alibaba/open-code-review) for this:

```
/ocr:review the (staged) proposal / spec. Pay special attention to [whatever is relevant here]
```

It will spawn multiple review agents and you will receive one consolidated `final.md` review document. This review will often find:

- Gaps in your spec: Edge cases that are not cleary specced out
- Overconfident claims: If potential issues in the design are played down in `design.md` it will often be called out
- References to code that don't exist: Occasionally even after a thorough explore mode, your LLM might have made guesses on what should exist in your codebase but doesn't. It does not happen often, but if then it can be caught here
- Violations of your `CLAUDE.md`: While generally `explore` and `propose` know your `CLAUDE.md` and architecture guidelines, this review phase gives you the chance of finding any remaining violations

Make sure to answer any open questions in the corresponding `final.md` doc and update it accordingly. After having vetted `final.md` I will usually go into `/plan Address the review findings in final.md` and update the spec accordingly. This will save you time later on when reviewing the implementation.

{{< callout >}}
You might turn this proposal into a ticket if someone else is to pick it up. Another developer can continue with step 4 then. Attach the full spec to this ticket.
{{< /callout >}}

### Step 4: /opsx:apply - Implement your Spec

When you run e.g.

```
/opsx:apply <your-spec>
```

your agent will go through `tasks.md` and implement the required changes step by step. In my experience (using Opus 4.8) - if you have specced out properly! - this will one-shot the implementation. Usually there's not manual interaction required here, with one expection: If you have written your spec and in the meantime your codebase has evolved significantly. Your spec may contradict it own assumptions about your current codebase - you might have to respec or at least make some apply time decisions here.

### Step 5: /ocr:review - This time the Implementation

Even with a good spec you should review the implementation. Here you will catch:

- Deviations: Implementation doesn't match the design (usually small, but it happens)
- `CLAUDE.md` violations
- Code duplication & bugs

However you should have already caught most issues in your step 3 review before.

## Drawback: Token Usage

One thing to note: Both OpenSpec and Open Code Review eat up a lot of tokens. I've been using almost exclusively Opus 4.8 with it and have been averaging around 2mio tokens per day. However this includes building a greenfield project and many code reviews.

## So What's Your Job Now?

The bottleneck is now in the `explore` phase: With fuzzy requirements, your LLM (at least Opus 4.8) will often ask you hard questions here and usually they're very valid. Before you have your first spec you might have to spend a few hour in this mode if it's a reasonably complex features and you have to deal with a legacy codebase.