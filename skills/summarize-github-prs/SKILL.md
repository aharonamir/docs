---
name: summarize-github-prs
description: Summarize open GitHub pull requests for one repository and one or more GitHub usernames into a single synthesized paragraph. Use when the user asks to summarize PRs, open pull requests, GitHub PR work, or contributions for a repo and named users.
---

# Summarize GitHub PRs

## Purpose

Create one concise paragraph that summarizes the themes across open GitHub pull requests for a given repository and one or more usernames.

## Inputs To Identify

- Repository, either as `owner/repo` or a GitHub PRs URL such as `https://github.com/owner/repo/pulls`.
- One or more GitHub usernames.
- PR state, defaulting to `open` unless the user asks for merged, closed, or all PRs.
- Output shape, defaulting to a single paragraph unless the user asks for bullets or per-user summaries.

## Workflow

1. Query GitHub for each user:

   ```bash
   curl -s 'https://api.github.com/search/issues?q=repo:OWNER/REPO+type:pr+state:open+author:USERNAME&per_page=50'
   ```

2. Verify the `total_count` for each user and handle pagination if any count is above 50.

3. Extract, for every PR:
   - PR number and URL
   - title
   - labels
   - useful body text, especially the "What does this PR do / why do we need it" section

4. Ignore PR template boilerplate, checklist text, repeated "Paired" links, and generic issue-closing text.

5. Group related work by technical theme, not by author, unless the user explicitly asks for user-specific attribution.

6. Write a single paragraph that:
   - names the repository
   - summarizes the main themes across all matching PRs
   - mentions notable CI/conflict status only if it materially changes the interpretation
   - avoids listing every PR number unless the user asks for traceability
   - is concrete enough to convey what the PRs actually change

## Summary Style

- Keep it to 3-5 sentences.
- Do not state the usernames unless the user asks for attribution.
- Prefer compact engineering nouns: observability, error handling, async safety, verification, benchmark execution, tooling, context preservation.
- Avoid vague phrasing like "various improvements" unless followed by concrete examples.
- If no PRs are found, say that directly and specify the repo, users, and state searched.

## Example

User request:

```text
Summarize open PRs for https://github.com/openJiuwen-ai/agent-core/pulls by alice and bob in one paragraph.
```

Answer shape:

```text
The open PRs for `openJiuwen-ai/agent-core` focus on hardening the agent runtime for long-running autonomous work: they improve error taxonomy and boundary exception handling, expose more tracing and live activity events, reduce event-loop stalls, preserve task requirements across compression, and add verification loops for benchmark/CI repair. Overall, the work makes agent execution easier to inspect, less prone to repeated failure loops, and better suited for unattended evaluation workflows.
```
