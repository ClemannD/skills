---
name: code-docs-style
description: Writes and edits code comments, doc comments, READMEs, and engineering docs in a plain engineer-to-engineer voice, without AI speak. Use when writing, reviewing, or rewriting any comment, docstring, README, guide, or design note, or when asked to "de-slop", "fix the AI speak", or "make this sound like an engineer wrote it".
---

# Code docs style

Write the way a senior engineer writes for another engineer: short, exact, and in the words people
already use for the thing. The reader is skimming between tasks. Every sentence must tell them
something the code doesn't.

## Voice

- **Use the standard term.** Normalize, serialize, deterministic, idempotent, retry, backoff,
  drop-in replacement, default. If engineers have a word for it, use that word.
- **Plain doesn't mean folksy.** Don't paraphrase a precise term into everyday prose. "Same input,
  same text" is worse than "serializes deterministically".
- **Don't invent vocabulary.** "The first arm of the rule", "the happy leg", "ratchet" make the
  reader decode your shorthand. Describe the behavior, or give the concept a real name in code.
- **Lead with what it is or does.** "Returns…", "Creates…", "Thrown when…", "Timeout in
  milliseconds." Not "This function is responsible for…" or "The helper that…".
- **Name the exact condition.** "Called for any error that isn't a `ValidationError` or a 4xx"
  beats "called for errors we didn't expect". The reader should be able to predict the behavior.
- **Give units and defaults.** "Timeout in milliseconds. Defaults to 10 seconds."
- **Claim only what is true.** "Every", "always", "never" and "all" are promises. Use them only when
  code or a test enforces them. Don't call a library what "every service uses" when adoption is
  opt-in.
- **Simple declarative sentences.** One fact per sentence, with its condition or consequence
  attached.

## AI speak to cut

- **Personification.** "The handler that answers every failure", "remembers each request". Say
  what the code does.
- **Circular descriptions.** "Declares a route the way every route is declared." Say what it adds.
- **Clipped aphorisms.** "Same routes, same text." Write the full sentence.
- **Grand framing.** "The single source of truth for all things config." Say what it loads and
  when.
- **Filler.** "It's worth noting", "essentially", "in order to", "seamlessly", "robust",
  "ensures". Delete.
- **Triplets and dash chains.** "Fast, safe, and simple — every time." Make one concrete claim.

## Comments

- One to three lines by default. A comment longer than the function it documents is almost always
  wrong.
- Keep only what the code can't say: a constraint from outside the file, a precedence choice that
  looks arbitrary, a deliberate no-op, a workaround for an external bug.
- Write doc comments in the language's doc syntax (`/** */`, docstrings) above the declaration.
- Cut:
  - restating the signature or types;
  - narrating the branches of the body;
  - naming other functions or files ("mirrors X", "same as Y"). They go stale. Share code
    instead. Issue IDs and external specs are fine;
  - "why not X" alternatives. Those go in the PR description or a decision record;
  - speculation about future callers.
- Test: read the comment, then the code. If the code told you the same thing as fast, delete the
  comment.

```ts
// Before
/** The handlers that answer every failure with a clean error. */
// After
/** Returns `notFound` and `onError` handlers that respond with an `ErrorBody`. */

// Before
/** Aborts a request that takes longer than this. */
// After
/** Per-request timeout in milliseconds. Defaults to 10 seconds. */

// Before
/** Keeps what an error body says when it is valid, and fills in the rest from the status. */
// After
/** Normalizes an error body to `ErrorBody`, deriving `code` from the HTTP status when it has none. */

// Before
// The request never went out, for example because getting the token threw. Pass that error on.
// After
// Never sent (e.g. `getToken` threw), so rethrow the original error.
```

```python
# Before
def retry(fn, attempts=3):
    """Robustly retries a flaky operation, ensuring resilience against transient failures."""

# After
def retry(fn, attempts=3):
    """Calls `fn` up to `attempts` times with exponential backoff. Re-raises the last error."""
```

## READMEs

A README says what the code can't: what the package is for, the rules a caller must follow, and
how to use or run it.

1. Open with one or two sentences: what it is, and any constraint that decides whether the reader
   can use it ("Safe to run in a browser: no Node built-ins.").
2. Show a short example with its import lines that works as written. Snippets beat prose.
3. Add sections only for what a caller must know, named after the topic: `## Errors`,
   `## Versions`, `## Generated code`. No "Overview", "Features" or "Conclusion".

Rules:

- Give a rule with its reason: "The check is byte for byte, so exclude the file from the
  formatter."
- Pair an instruction with what happens if it's ignored: "Tag every endpoint. An untagged endpoint
  ends up on the root object."
- State the runtime and setup the code assumes. Name only setups you tested.
- Don't promise what no test proves. If the README says "times out after 10 seconds" or "rolls back
  on failure", a test must show it, including the case most likely to break it.
- Show real output when it helps, such as an example response body.
- Cut tables or lists that copy the code (option tables, endpoint lists, file-by-file tours),
  features that don't exist yet, and boilerplate sections.

~~~md
# @acme/http-errors

A standard error body for HTTP APIs. A service that adopts it returns the same shape for every
failed request, so its clients handle errors one way. It's opt-in.

```json
{ "code": "conflict", "reason": "email-taken", "message": "That email is already registered." }
```

- `code` is one of a short, fixed list, and each code maps to an HTTP status. Clients switch on it.
- `reason` is optional. It names a specific failure that a caller handles differently.
- `message` is for people. It can change, so don't match on it.

## Adding a reason

- Add one only when a caller would act on it differently.
- Use lowercase words joined by hyphens, such as `email-taken`.
- Don't rename it after it ships. Callers compare the exact string.
~~~

## Other docs

Guides, design notes, runbooks and change descriptions use the same voice. Also:

- Open with the outcome or the answer, then the detail.
- Keep paragraphs short. Use lists only for real lists, such as steps or options.
- Prefer a snippet of what the reader writes now over a paragraph describing it.
- Keep summaries above the file level. Three or four bullets, not eight.
- Never invent numbers, sources, or results. Say what you don't know.

## Editing existing text

1. For each sentence, ask whether it tells the reader something the code or an earlier sentence
   doesn't. If not, delete it.
2. Replace personification, metaphors and invented terms with the standard term.
3. Check every "every", "always", "never" and "all" against the code.
4. Check that each guarantee has a test, or remove the claim.
5. Read it aloud. If you wouldn't say it to a teammate, rewrite it.
6. When asked, list each change as before and after.
