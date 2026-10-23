# 01: tool-agent

**Repo:** [JTorres87/tool-agent](https://github.com/JTorres87/tool-agent) · **Release:** v0.1.0

A CLI agent that helps you choose an LLM and plan what it will cost: "Which Claude model with a 1M context window is cheapest for 50,000 requests a day?" It answers by calling typed tools over a SQLite catalog of Anthropic, OpenAI and xAI models, and every answer is checked before you see it.

## Why I built it this way

I wrote the agent loop by hand, on the Anthropic SDK with no framework. Frameworks make the loop's decisions for you: what goes back into context, when to stop, what happens when a tool fails. I wanted to make each of those decisions myself before handing them to a framework.

## How the loop works

1. Send the whole conversation, plus the tool definitions, to the model (Claude Haiku).
2. If it asks for tools, run every requested call, validate the inputs against Pydantic models, and send **all** the results back in **one** message. A tool error goes back to the model as an error result, not up the stack.
3. Repeat until the model answers, or until the step limit stops it.
4. The answer is structured output, a `FinalAnswer`: the reply, the tool calls it relied on, and the estimate it chose for cost questions.

**How a tool result re-enters context:** as a `tool_result` block in the next user message, tied to its call's `tool_use_id`. The loop keeps the model's whole turn, including its tool calls, in the history; dropping them would break the pairing.

**How the loop ends:** the model stops asking for tools (`stop_reason` isn't `tool_use`), or the step guard ends it after 10 steps.

## Two checks on every answer

1. **Citation check** (free, in code): every cited tool call happened, and the chosen estimate matches what that call returned, number for number.
2. **Verifier** (Claude Sonnet): reads the question, the answer and only the cited results, and rejects claims they don't support. A rejected answer goes back to the agent with the problems. After two rejections it's returned flagged with the verifier's concerns.

A verified answer costs about $0.01. The agent on Haiku costs under a third of a cent; the rest is the check.

## What broke, and what I learned

**1. The model couldn't see what I asked it to cite.**
Haiku cited tool names ("list_models") or made-up ids on its first answer **every time**, so the citation check forced a retry on every question. Rewording the instruction changed nothing: 5 failures in 5 runs both before and after. The cause: tool-call ids are attached by the API, and the model only reads the tool's output. Writing each id into its tool result took failures to **0 in 5**, saved a model call per question, and cut agent tokens by about a third.

> Lesson: before rewording a prompt, check whether the model can see the thing you're asking about.

**2. My own check blocked the right answer.**
"What would claude-ultra cost?" came back unverified after two rejections. The verifier was right each time. My citation rule banned citing failed tool calls, but the failed lookup ("Unknown model_id") was the evidence that the model doesn't exist. A second bug made it worse: after a rejection the agent made a new call but kept citing the old one, because the feedback didn't list the call ids. Fix: failed calls can be cited (and are marked as errors for the verifier), and both feedback messages list every call so far.

> Lesson: test the "not found" paths, not just the happy path. A strict check can be the bug.

**3. The verifier catches what code can't.**
The citation check proves a number came from a tool. It can't tell whether a sentence about that number is true. Real catches:

- "Haiku 5.5 scores lower than Sonnet 5 (43.4 vs 38.2)": 43.4 is higher. Rejected and corrected.
- "All OpenAI models support tools and reasoning": a filtered search can't show that. Rejected and corrected.

**4. The verifier's cost is its input, not its effort.**
Low, medium and high effort gave the right verdict on all three test answers, at the same cost. About 2,500 tokens of evidence is nearly the whole bill; a "correct" verdict is 16 output tokens. With a 42-model list as evidence, a check reads about 7,000 tokens. The lever is smaller evidence, not lower effort.

## The chat session

The same loop runs inside a conversation. Follow-ups see earlier answers and can cite their tool calls: "What would the cheapest of *those* cost?" reused the earlier search instead of repeating it. A turn that fails partway is rolled back, because a history that ends on a tool call with no result makes the API reject every later request.

## Three ways agents like this fail in production

1. **Unsupported claims that look grounded.** The numbers come from tools, but the sentences around them are wrong. Here, a second model checks claims against the evidence.
2. **Broken conversation state.** A crash between a tool call and its result poisons the history. Here, failed turns are rolled back.
3. **Retry loops that burn money.** A check that can never be satisfied keeps the agent retrying. Here, a step limit and a cap on verifier rejections stop it, and the answer comes back flagged instead.

## Next: v0.2.0

Logging and evals (count every failure type; turn the verifier's test answers into a labeled set), retries with backoff and a fallback model, prompt caching with before-and-after costs, smaller verifier evidence, streaming, and a reasoning model vs. a fast model on the same questions.
