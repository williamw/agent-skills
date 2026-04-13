---
name: composed-method-pattern
description: >-
  Refactor TypeScript in a Kent Beck style: make code read like a story for humans.
  Use when functions are hard to follow, mix abstraction levels, or resist debugging.
---

# Composed Method (Implementation-Patterns Voice)

Code is for people.
Computers can run anything; humans must understand it.

A good function reads like a paragraph of intent.
A better one reads like a checklist.

## Core stance

- One function, one level of thought.
- Top-level methods describe **what**.
- Helpers handle **how**.
- Names carry meaning so comments stay rare.

## Control-panel intent

When a value is tuned often — enabled tools, prompt strategy, step budget,
feature flags — keep it visible in the top-level method.

A reader should see every operational knob without opening helpers.
Helpers execute decisions; the top-level method declares them.

Prefer direct declarations over wrapper functions when the inline
version is just as clear and avoids hiding choices behind a click.

## Naming is design

If the name is weak, the design is weak.

Prefer names that read well at the call site:

- `getQueryParam(req)` not `param(req)`
- `isGibberish(text)` not `gibberishCheck(text)`
- `hasValidEmail(input)` not `emailValidation(input)`
- `buildPayload(data)` not `payloadBuilder(data)`
- `ensureQuotaAvailable(user)` not `quotaCheck(user)`
- `normalizePhoneNumber(raw)` not `phoneFormatter(raw)`

Rules of thumb:

- **Actions** use verbs: `get`, `parse`, `build`, `fetch`, `save`, `normalize`
- **Booleans** read as questions: `is`, `has`, `can`, `should`
- **Guards** use intent verbs: `ensure`, `require`, `validate`

If a call sounds awkward when spoken aloud, rename it.

## When to apply

- You need to explain a function line-by-line.
- Intent is buried in mechanics (loops, parsing, API setup, formatting).
- Debugging requires holding too many details at once.
- The user asks for readability or cleanup.

## Refactoring recipe

1. Write the story of the function in plain language.
2. Mark lines as intent-level or mechanic-level.
3. Extract mechanic-level work into focused helpers.
4. Rename helpers until the top-level flow reads naturally.
5. Keep parameters narrow and types explicit.
6. Pull policy decisions back up to the top-level flow if they are operator-facing.
7. Stop when the main method is obvious at a glance.

## Example

### Before

```ts
export async function handle(req: Request) {
  const param = req.nextUrl.searchParams.get('q') ?? '';
  const gibberish = gibberishCheck(param);
  if (gibberish) return NextResponse.json({ ok: false });
  // mixed logic continues...
}
```

### After

```ts
export async function handle(req: Request) {
  const query = getQueryParam(req);
  if (isGibberish(query)) return invalidQueryResponse();
  // composed flow continues...
}
```

### Control-panel handler

Avoid:

```ts
export default async function handler(req, res) {
  // ...
  const tools = selectTools(); // what did we select?
  const result = await enrichContact(config, tools, firstname, lastname, email);
  // prompt strategy and step budget are buried inside enrichContact
}
```

Prefer:

```ts
export default async function handler(req, res) {
  // ...
  const tools = {
    think: thinkTool(),
    tavilySearch: tavilySearch(),
  };
  const systemPrompt = buildSystemPrompt(tools);
  const maxSteps = 8;

  const result = await enrichContact(
    config,
    tools,
    systemPrompt,
    maxSteps,
    firstname,
    lastname,
    email
  );
}
```

Every operational knob is visible. Comment out a tool, change a step budget,
swap a prompt — all without opening helpers.

## Done means

- A reader understands the top-level flow without opening helpers.
- Every function stays at one level of thought.
- Names tell the truth of the behavior.
- The code is easier to debug because the story is visible.

## Non-goals

- Not maximal extraction.
- Not pattern ceremony.
- Not clever terseness.
- Not helper layers that obscure operational choices.
- Clarity first.

## Note on testing

Better testability often follows from this style, but it is a side effect.
Primary objective: code that is easy for humans to read, reason about, and debug.
