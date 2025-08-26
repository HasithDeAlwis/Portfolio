---
date: '2025-08-20T10:30:00-04:00'
draft: false
title: 'How I merged 1k+ lines of Code into the Grafana Labs Ecosystem with AI'
description: "See an example PR I merged in and how I worked with AI to speedup the contributing process while still learning"
---

Up until two months ago, I was very adverse against AI. I believed it lowered our ability to think critically, be creative, and to write good scalable code. I still believe this! But for better or for worse, we should learn how to use it effectively and in a manner such that we still LEARN. 

I strongly advise against people using AI for personal projects, but for work and contributing to large codebases, AI can be a helpful tool that allows you to optimize workflows and make your company more money.

## What did I merge? 

I've merged over 5+ PRs in the Grafana Lab's ecosystem. Here are a couple: 
1. [Shareable Dashboard w/Span Filters](https://github.com/grafana/grafana/pull/107363)
2. [Show Values in Timeseries/Trends](https://github.com/grafana/grafana/pull/108090)
3. [waitForResponse API](https://github.com/grafana/k6/pull/5002)

All of these PRs were written using AI ASSISTANCE, if you use AI to write the majority of your code in complex repos like Grafana, you will end up wasting a lot of time. AI can write code if you are...
- Writing docs
- Scaffolding unit tests for BE business logic
- Building dumb components

Notice how from the examples I listed, these tasks are *isolated*. As soon as you introduce complex state management (ie. Redux), or multithreading, or working with multiple services, AI is much less effective. 

# Example Walkthrough
*Note: All examples will be using GitHub Copilot using Claude Sonnet 4. IMO, if you know how to prompt engineer effectively, almost all models work similarly. The prompt responses are very long so I will be paraphrasing the responses, but giving the exact prompts sent to the model.*

## What is the problem?
For the example, I'll walk through the [Shareable Dashboard w/Span Filters](https://github.com/grafana/grafana/pull/107363) PR. 

In Grafana's Explore trace view, when a user applied span filters (for example by service name, span name, duration, tags, etc.), those filters were stored only in the local component state. This meant:
- **Filters were lost if you refreshed the page or navigated away.**
- **You couldn’t bookmark or share a URL with your specific filter configuration.**
- **Browser navigation (back/forward) did not preserve the filter state.**
- **It was hard to collaborate, since you couldn’t send someone else a link with your filter settings.**

### What This PR Did

This PR implemented **URL synchronization for span filters** in the Explore trace view. In summary:

- **When you apply span filters in Explore, those settings are now encoded into the URL.**  
  - If you refresh, navigate away and return, or use back/forward, your filters are restored.
  - You can copy/paste/share a link with all your filter settings embedded.
- **This new behavior is only for Explore.**  
  - Dashboard “Traces” panels keep their old behavior (local state, not URL-synced) for backwards compatibility.

## Ideation Stage

The ideation stage is where AI is the most useful—it helps onboard you onto a new part of the codebase. This is where you should get an idea of HOW you are going to approach this problem and what problems you might run into. 

The first prompt I gave was: 

`Your goal is to provide a comprehensive report on how Grafana handles global state management. Give a real walkthrough on how global state is created, rendered, and updated. Think step by step.`

This gave me the Redux foundation I needed, but more importantly, it revealed *some* Grafana's specific patterns around URL state synchronization. The response showed me that Grafana uses Redux for Explore state management with specific actions for managing URL state.

My follow-up prompt was more targeted:

`Show me exactly how Grafana's Explore view currently handles URL state for query parameters. I need to understand the data flow from URL → Redux → Component and back. Include specific file paths and function names.`

This revealed the key insight: Explore already had URL sync infrastructure via the Redux store in `public/app/features/explore/state/main.ts`. The span filters weren't using it—they were just living in local component state via a custom `useSearch` hook that only used `useState`.

Here's the crucial part that most developers miss when using AI: **I didn't ask AI to solve the problem yet**. I was still in discovery mode. My third prompt dug deeper into the existing patterns:

`Find all instances where Grafana components sync local state to URL state. Show me 3 different examples with their implementation patterns. Focus on components that handle filtering or search functionality.`

This taught me about Grafana's state management strategies. I learned that Explore uses Redux with URL synchronization, while dashboard panels use local state for backward compatibility. I also discovered that Grafana has a pattern of using context-aware hooks—the same hook can behave differently based on whether it's used in Explore or dashboard contexts.

**The key insight from this stage**: The problem wasn't that span filters were broken—it was that the `useSearch` hook in the trace view was hardcoded to use local `useState`, regardless of context. The solution was to make it context-aware: use Redux + URL sync when in Explore (if `exploreId` provided), but keep local state behavior for dashboard panels.

At this point, I was fairly certain that I had come up with an effective plan to implement the PR, but I was still confused. How did Grafana approach serializing JSON to a URL and deserializing a URL. So I asked: 

`Show me an example of how Grafana handles serializing and deserializing an object from a URL in the Explore page. Be explicit with file names and paths and show code snippets that outline the process.`

This gave me a clear insight that I otherwise would have missed. There already exist logic to serialize the global state to and from a URL. Copilot was able to give me the exact files and this stopped me from writing an existing service.

At this point, I had a clear architectural plan without having written a single line of code. But to make Copilot more effective, I asked it to synthesize the plan:

`Your goal is to create a comprehensive report of how to approach adding span filters to URL state. Write down all implementation details and outline the potential edge cases`

Giving a plan that looked roughly like so:
1. **Modify the `useSearch` hook** to detect context via `exploreId` parameter
2. **When in Explore context**: Connect to Redux store and leverage existing URL sync
3. **When in Dashboard context**: Keep existing local state behavior for backward compatibility  
4. **Update the Redux actions** in `public/app/features/explore/state/main.ts` to handle span filter state
5. **Ensure URL encoding/decoding** works for the complex filter objects

The beauty of this approach was that it required minimal changes—I wasn't rebuilding the entire filtering system, just making the existing hook context-aware. As one reviewer noted.

This is where AI truly shines, not in writing code, but in helping you understand existing systems so you can make informed architectural decisions that leverage existing patterns rather than reinventing the wheel.

## Code Writing Stage

As I stated before, I like to avoid AI for writing code; however, it's useful for certain situations (ie. writing simple business logic). Here are some prompts I gave Copilot: 
1. `I have a React hook called useSearch that currently uses useState to manage filter state. I need to make it context-aware: if an 'exploreId' prop is provided, use Redux dispatch actions instead of local state. If no exploreId, keep existing behavior. Show me the conditional logic pattern.`
2. `Update the URL serilization logic to prune empty arrays.`
3. `Create default span filter constants and use them for all tests and in business logic`

Notice: I did not give it large overarching tasks, instead I asked Copilot to do small focused tasks. Copilot already has context of the plan, so it's able to execute these small snippets with high code quality. 

## Code Review Stage

This is the easiest part! Simply ask Copilot to review the code you wrote and to explicitly note the good and the bad with the code. 

`Outline the key benefits of the code written. What was good and what was bad. If you could, what changes would you make? Think step by step.`

After this, it's up to you as the engineer to implement or not implement the changes. 

## Overview

1. Understand the code base. Treat AI as a peer that understands the codebase much more than you
2. Ask for a plan
3. Give Copilot small and easy to consume tasks w/the plan in context
4. Review the code.

# Techniques
These are prompt engineering techniques I have found useful and backed by various research studies including this [Cornell University study](https://arxiv.org/abs/2402.14837). This is a little blurb, but if you don't want to read the whole study, find equivalent information [here](https://www.promptingguide.ai/techniques).

1. **Few-Shot Prompting**
    - Give concrete examples from the actual codebase you're working with
    - Paste real function signatures, existing patterns, or similar implementations

2. **Chain-of-Thought**
    - Notice how I asked for "comprehensive reports" or to "think step-by-step". This encourages deep reasoning and reveals the model's understanding of complex systems

3. **Self-Consistency** 
    - Ask the AI model to review its own code and identify potential issues
    - Follow up with: "What could go wrong with this implementation?"

4. **Context Priming**
    - Always include relevant file paths, existing function names, and architectural constraints in your prompts
    - The more context you provide, the more accurate the response

# Conclusion
AI is a great tool, but use it responsibly and effectively, or you'll just be creating technical debt.