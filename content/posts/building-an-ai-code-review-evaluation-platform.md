---
title: "Building an AI Code Review Evaluation Platform"
date: 2026-07-02T00:00:00+08:00
tags: ["ai", "code review", "engineering"]
author: "Azuan"
draft: true
---

Code review has always been one of the bottlenecks for software engineers. At [loveholidays](https://www.loveholidays.com/), we have been using a lot of AI in our daily workflows. Whether a PR is written with the help of AI or not, someone still needs to review it.

Then I read a [thread by Intercom](https://x.com/gregolsent/status/2039788736569885114) on how they use AI not just for code review, but also to approve it. Sounds risky, but also very interesting.

<p align="center">
  <img src="/building-an-ai-code-review-evaluation-platform/intercom-code-review.png" alt="Intercom code review" width="480">
</p>

<i>Disclaimer: This post is not an endorsement. This was just an experiment based on my understanding of how something like that could work. And it was also only possible because OpenAI gave us a three month trial. During the process, I burned through a few billion tokens. It was a great learning experience, but definitely not cheap.</i>

## 1. Yet another code review tools

There are a lot of code review tools out there. I'm sure some of them are great, even better than what I'm trying to build. But one thing that I'm trying to do here is to be able to run the tools with my existing ChatGPT Codex subscription (either personal or enterprise plan). I really, really don't want to pay for yet another subscriptions.

## 2. Coding agent

> Almost all coding agent will have a non-interactive mode. With Codex, you can trigger it via `codex exec`. I ran all sorts of automation with this approach.

I decided to use `codex exec` to run the code review. Using only the `git diff` would be a lot faster and cheaper, but I personally do not like that approach. Context matters a lot.

When I review a pull request, I rarely look at the diff in isolation. I open related files, check how the surrounding code works, look for similar patterns, and think about how the change could break existing behaviour. I wanted the AI reviewer to be able to do the same.

The basic flow looks like this:

1. Clone the pull request into a new `git worktree` location.
   - `git worktree` makes it easier to run multiple evaluations in parallel.
2. Run `codex exec` with a custom prompt, rules, and output format.
   - Run this in a sandbox, your own machine, or any environment you are comfortable with, as long as you understand the risks and limitations.
3. Ask the agent to return a structured JSON output.

Example output:

```json
{
  "verdict": "rejected",
  "reviews": [
    {
      "file": "src/file_1.ts",
      "line": 42,
      "comment": "This will break feature A.",
      "suggestion": "Consider rewriting it this way."
    }
  ]
}
```

With this approach, Codex can inspect relevant files and gather context before performing the review. Because the output is structured, I can also build a simple web interface around the system.

<p align="center">
  <img src="/building-an-ai-code-review-evaluation-platform/codex-usage.png" alt="codex-usage" width="800">
</p>


## Mock review

The next step is to answer a question, how can I test the code review system quickly?

Similar to the Twitter thread, I decided to use past pull requests as the source of truth. This is obviously simplified, but a pull request has a known lifecycle. At a specific commit, the expected outcome is usually clear. Should this be approved, or should it be blocked until the author makes changes?

With that in mind, I used previous pull request data as fixtures for mock reviews.

| Pull Request | Commit hash | Expected | Tags    | Note                 |
| ------------ | ----------- | -------- | ------- | -------------------- |
| PR#1         | aaa         | Approved | booking |                      |
| PR#2         | bbb         | Rejected | booking |                      |
| PR#2         | ccc         | Approved | booking |                      |
| PR#3         | ddd         | Rejected | payment | _Caused an incident_ |

Some examples:

- PR#1
  - A 10x engineer requested a review and everything was LGTM.
- PR#2
  - The author requested a review at commit `bbb`, and reviewers left comments.
  - The author updated the pull request at commit `ccc`, and it was finally approved.
- PR#3
  - This pull request was originally approved, but it accidentally caused an incident.
  - In the mock review, I want this PR to be rejected. Thanks to my team lead, who asked about this scenario.

<p align="center">
  <img src="/building-an-ai-code-review-evaluation-platform/eval-fixtures.png" alt="eval fixtures" width="800">
</p>



## Comparing evals

My next question was, what prompt produces the most reliable review for this type of pull request?

With the coding agent and mock review setup in place, I could test different prompts and compare the results.

Prompt A:

> You are a generic senior software engineer. Review this PR and make no mistakes.

Result: 50% accuracy

| Pull Request | Commit hash | Tags    | Expected | Actual   |
| ------------ | ----------- | ------- | -------- | -------- |
| PR#1         | aaa         | booking | Approved | Approved |
| PR#2         | bbb         | booking | Rejected | Approved |
| PR#2         | ccc         | booking | Approved | Approved |
| PR#3         | ddd         | payment | Rejected | Approved |

Prompt B:

> You are a senior software engineer for the post-booking team.

Result: 75% accuracy

| Pull Request | Commit hash | Tags    | Expected | Actual   |
| ------------ | ----------- | ------- | -------- | -------- |
| PR#1         | aaa         | booking | Approved | Approved |
| PR#2         | bbb         | booking | Rejected | Rejected |
| PR#2         | ccc         | booking | Approved | Approved |
| PR#3         | ddd         | payment | Rejected | Approved |

This is where the eval setup became really useful. Instead of guessing whether one prompt was better than another, I could run both against the same historical examples and compare the result.

I cannot stress enough how valuable this step is. With evals, I can measure whether the changes are moving in the right direction in a more deterministic way.

Notice the `tags` in the mock review table. The idea is that every team and domain is different. A booking-related pull request may need a different prompt, checklist, or priority from a payment-related pull request. With tags, I can run targeted evals for specific areas and tune the reviewer accordingly.

<p align="center">
  <img src="/building-an-ai-code-review-evaluation-platform/eval-runs.png" alt="eval runs" width="800">
</p>

## Eval automation

What is the right prompt? Ideally, I don't want to search for the best prompt myself. So instead, the system will be able to reiterate itself and improve it's own code review prompt. It is far from perfect, but some prompt managed to get 88% accuracy, up from 50% initial start.

If you use coding agents long enough, you will realise that the harness matters too, not just the model or reasoning level.

Because of that, I expanded the code review system to support other harnesses. One of my favourites is [`pi`](https://github.com/earendil-works/pi), which can be run through the `pi --print` command in a similar way.

Once the mock review system has been setup, it became much easier to compare different variables:

- `codex` vs `pi`
- one model vs another model
- different reasoning levels
- generic prompts vs team-specific prompts
- full repository context vs more constrained context
- cross repository context (backend, frontend, infrastructure)

This made the system less about one specific agent and more about a repeatable evaluation workflow.

<p align="center">
  <img src="/building-an-ai-code-review-evaluation-platform/eval-harness.png" alt="eval harness" width="800">
</p>

## Cost

Measuring cost is also important. For any coding agent execution, a session ID is generated. Combined with [ccusage](https://github.com/ccusage/ccusage), I can check the token usage and cost for each code review.

For example:

```bash
ccusage session --id 4aa0a585-b135-47e4-bdfb-c07bb958942d
```

One reason `pi` is interesting is that it is highly customisable and less bloated by default, which can make it cheaper to run. With this setup, I can test whether there is any truth to that theory.

For example, if one harness is 30% cheaper but still stays within an acceptable accuracy range, it might be worth pursuing. Cost must be measured together with review quality.

<p align="center">
  <img src="/building-an-ai-code-review-evaluation-platform/review-cost.png" alt="review cost" width="800">
</p>

# Beyond automation

## Learning opportunity

Code review is often a two-way conversation. There are times when I need help understanding code, especially when it touches an area I do not have much experience with yet.

I could pair with the pull request author, but sometimes I just have a few quick questions about the PR. Just like GitHub/GitLab code review flow, I can click on any line of code in the web UI and start a conversation. And the coding agent can explain the relevant context, answer questions, and help me understand the change in seconds/minutes.

To me, even if I never fully trust the system to auto-approve pull requests, this is still useful tools. It can act as a review companion to summarize risky areas, explaining unfamiliar code, and helping reviewers ask better questions.

<p align="center">
  <img src="/building-an-ai-code-review-evaluation-platform/bot-qa.png" alt="bot Q&A" width="800">
</p>

## Risks and limitations

There are still plenty of risks here, which I won't talk about. Mainly because there's just too many of them. But doesn't mean it's bad, or wrong. It's a great learning opportunity

# Final thoughts

This started as a part-time project and was fully vibe coded. Using Codex CLI, gpt-5.5 medium. Even so, I was quite impressed with the result.

I am still building it, and I run it locally when I need to. Partly, that is because our three-month trial has ended and we now have to start paying for usage. Although, I do not think that is a bad thing. The trial gave me a chance to build a few useful tools, and this code review platform is one of them.

