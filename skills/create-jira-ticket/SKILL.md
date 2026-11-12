---
name: create-jira-ticket
version: 2.0.0
description: Create a JIRA ticket from user instructions via the Atlassian MCP connector. Uses project from the current branch when possible, lists project epics, recommends the best epic, asks confirmation before creating, and links Figma designs.
argument-hint: "describe the ticket to create"
---

Create a JIRA ticket from the user’s instructions using the **Atlassian MCP connector** (`mcp__claude_ai_Atlassian__*` tools; load their schemas with ToolSearch first). Prefer the **same project as the current branch** when it can be inferred from a JIRA key in the branch name; otherwise ask which project. Load **epics** for that project, **infer the best epic** for the new work, show a **preview**, and obtain **explicit confirmation** before creating the issue.

**Usage:**
- `/create-jira-ticket` — Pass the ticket description in the same message as `$ARGUMENTS` (summary intent, bug vs feature, acceptance criteria, links, etc.)

**Instructions:**

1. **Resolve the Atlassian site:**
   - Call `getAccessibleAtlassianResources` once and reuse its `cloudId` on every later call. AgXeed's site is `https://agxeed.atlassian.net`.
   - If the connector is missing or unauthenticated, tell the user to connect the Atlassian connector in claude.ai and **STOP**.

2. **Determine the JIRA project key:**
   - Run `git branch --show-current`. If detached HEAD, note it but continue if a project can still be chosen.
   - Match a JIRA issue key in the branch name with pattern `[A-Z][A-Z0-9]+-[0-9]+` (case-insensitive; normalize to uppercase). If found, set **project key** to the prefix before `-` (e.g. `PROJ-123` → `PROJ`).
   - If no key is found, list projects (`discover` "list jira projects", then `executeRead`) and use `AskUserQuestion` so the user picks the **project key**.

3. **Parse instructions from `$ARGUMENTS`:**
   - If empty, use `AskUserQuestion` to collect what ticket to create, then continue.
   - Infer where possible:
     - **Summary** (short, imperative title)
     - **Issue type** (Bug, Story, Task, etc.) — from wording (“fix”, “broken” → Bug; “add”, “implement” → Story/Task)
     - **Description** in Markdown with clear sections (for bugs: what’s wrong, steps, expected vs actual; for stories/tasks: goal and acceptance criteria as `- [ ]` items). `createJiraIssue` converts Markdown to ADF.
     - Optional: **priority**, **labels**, **Figma URLs** (`https://www.figma.com/...`)
     - If the user **names a specific epic** (key like `PROJ-100` or title), treat that as a **manual epic preference** for step 5.

4. **List epics in the project:**
   - Call `searchJiraIssuesUsingJql` with `jql: "project = <PROJECT-KEY> AND issuetype = Epic ORDER BY created DESC"`, `fields: ["summary", "status"]`, `maxResults: 50`.
   - If empty, check the project's issue types (`discover` "listJiraProjectIssueTypesMetadata") for a differently named epic type and retry once; if still empty, report that no epics were found.
   - Collect each epic’s **key**, **summary**, and **status**.

5. **Recommend an epic:**
   - If the user specified a **manual epic** in step 3 and that key exists in the list (or `getJiraIssue` shows it is an Epic), use that as **recommended epic**.
   - Else if there are **no epics**, set **recommended epic** to none and skip matching.
   - Else compare the new ticket’s **summary + description** (keywords, product area, components) to each epic **summary**. Pick the strongest thematic match; prefer epics that are **not Done/Closed** when ties exist.
   - Write a **one-sentence rationale** (e.g. “Matches ‘Checkout’ epic because the work describes payment flow.”).

6. **Confirmation preview (mandatory):**
   - Show a clear preview:
     - Project, issue type, summary
     - **Recommended epic** (key + summary + rationale), or “None — no epics in project” / “None — user declined parent”
     - Short description outline
     - **Figma URLs** if any
   - Use `AskUserQuestion` so the user can:
     - **Create** with the recommended epic
     - **Choose another epic** from the list (show keys + summaries)
     - **Create without epic parent**
     - **Edit** — user revises instructions; return to step 3
     - **Cancel** — **STOP**
   - Call `createJiraIssue` only after the user picks one of the “create” paths.

7. **Create the issue:**
   - Call `createJiraIssue` with `cloudId`, `projectKey`, `issueType`, `summary`, `description` (Markdown), and optional `priority`/`labels`.
   - **Parent / epic:** pass `parent: "<EPIC-KEY>"` unless the issue type is **Epic**. If the call is rejected, follow its `repairHint` and retry once; if still blocked, create without parent and tell the user to set the epic in Jira.
   - Report the new **key** and browse URL `https://agxeed.atlassian.net/browse/<KEY>`.

8. **Link Figma designs (if provided):**
   - The Figma for Jira **Add Design** panel reads an app-owned issue property, and the MCP connector only writes properties prefixed `atlassian-mcp.`, so the panel can't be filled from here.
   - Put each Figma URL in the description under a **Design** heading, and add it as a web link: `discover` "createJiraIssueRemoteIssueLink", then `executeWrite` with the URL and a title naming the frame.
   - Tell the user they can still attach the design to the Figma panel with **Add Design** in Jira.

9. **Edge cases:**
   - Required custom fields: if create fails with validation errors, show the fields and `allowedValues` from the response, ask the user for values, then retry.

**Summary output for the user:** project, new issue key, URL, epic used (or none), and how the Figma links were added.
