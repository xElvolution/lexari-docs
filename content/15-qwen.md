---
title: How Qwen powers personal onchain agents in Lexari
slug: qwen
group: Models
summary: Where Qwen 3.8 Max sits in Lexari, how it turns a plain English request into checked, confirmable onchain steps, and what it brings to every agent.
---
Lexari lets anyone create a personalized AI agent that lives onchain. Each agent has its own wallet, its own personality and its own skills. You talk to it in plain English, and it can swap, pay and research for you on Monad (Kuru swaps, AUSD payments) and route across chains. Every payment it proposes is shown to you on a Confirm card first, and nothing moves until you tap it.

This article explains how **Qwen 3.8 Max**, Alibaba's strongest Qwen model served through Alibaba Cloud Model Studio, powers those agents: where it fits in the stack, what it actually does inside a turn, how one real request is traced from message to receipt, and what value it brought to the product. Everything here is taken from the Lexari source code, linked at the end.

## What a Lexari agent is

A Lexari agent is not a shared chatbot. When you make one, you give it a name, a face, a tone and a brief. It gets:

- **Its own wallet.** One address per chain family, funded by you and bounded by limits you set. On Monad that is a regular EVM address that holds MON, test USDC and AUSD.
- **A personality.** The role, tone and brief you wrote are part of every prompt it sees, so it answers like the agent you designed, not like a generic assistant.
- **Skills.** Integrations you switch on per agent: Kuru for swaps on Monad, Agora AUSD for payments, cross-chain routing, market data and more. An agent only hears about the integrations you granted to it.
- **A memory.** At the end of a reply the agent can propose one thing to remember. It is saved only if you keep it.

Your agent can read freely, but **it cannot spend on its own**. That single rule shapes how the model is used.

## How a chat message becomes an action

Every turn in Lexari follows the same path, whichever model you picked.

<figure class="diagram">
<svg viewBox="0 0 860 300" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="How a chat message becomes an action in Lexari" font-family="system-ui,sans-serif" font-size="13">
<defs><marker id="arq" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0L10 5L0 10z" fill="#5B2BFF"/></marker></defs>
<g fill="#F5F2E9" stroke="#0A0B0D" stroke-opacity=".25">
<rect x="10" y="40" width="140" height="64" rx="12"/><rect x="190" y="40" width="160" height="64" rx="12" fill="#EDE6FF" stroke="#5B2BFF"/><rect x="390" y="40" width="160" height="64" rx="12"/><rect x="590" y="40" width="120" height="64" rx="12"/><rect x="740" y="40" width="110" height="64" rx="12"/>
<rect x="190" y="190" width="160" height="64" rx="12"/><rect x="390" y="190" width="160" height="64" rx="12"/><rect x="590" y="190" width="260" height="64" rx="12"/>
</g>
<g fill="#0A0B0D" text-anchor="middle">
<text x="80" y="68" font-weight="700">You</text><text x="80" y="88" fill="#5D5A52">plain English</text>
<text x="270" y="68" font-weight="700">Qwen 3.8 Max</text><text x="270" y="88" fill="#5D5A52">understand and plan</text>
<text x="470" y="68" font-weight="700">Tool tag</text><text x="470" y="88" fill="#5D5A52">validated input</text>
<text x="650" y="68" font-weight="700">Confirm card</text><text x="650" y="88" fill="#5D5A52">you tap</text>
<text x="795" y="68" font-weight="700">Execute</text><text x="795" y="88" fill="#5D5A52">agent wallet</text>
<text x="270" y="218" font-weight="700">Memory and history</text><text x="270" y="238" fill="#5D5A52">recall you kept</text>
<text x="470" y="218" font-weight="700">Read tools</text><text x="470" y="238" fill="#5D5A52">live quotes, balances</text>
<text x="720" y="218" font-weight="700">Receipt</text><text x="720" y="238" fill="#5D5A52">explorer link, saved to the chat</text>
</g>
<g stroke="#5B2BFF" stroke-width="2" fill="none" marker-end="url(#arq)">
<path d="M150 72H186"/><path d="M350 72H386"/><path d="M550 72H586"/><path d="M710 72H736"/>
<path d="M270 186V108"/><path d="M470 108V186"/><path d="M430 186V140H300V108"/><path d="M795 104V186"/>
</g>
</svg>
<figcaption>One turn in Lexari. The model reads, plans and writes tool tags. Lexari validates every input, runs read tools for live data, and turns anything that moves money into a Confirm card.</figcaption>
</figure>


1. **You write a message.** For example: "Swap 20 USDC to MON on Kuru, then send 5 AUSD to Ada."
2. **Lexari builds the prompt.** It combines the agent's personality, safety rules, the memories you kept, recent chat history and a list of the tools this agent is allowed to use, with each tool's input format and your spending limits.
3. **The model answers.** It explains its plan in plain words and, when it needs a tool, writes a tag such as `<tool name="kuru.quote">{...}</tool>`. At most two tool calls are taken from one reply.
4. **Lexari validates the input.** Each tool has a strict schema. A call that is missing details, names a tool that does not exist, or uses an integration the agent was not granted is refused, and the agent is told why.
5. **Read tools run right away** and their live results (a quote, a balance) go back to the model so it can answer with real numbers.
6. **Sign tools only prepare a Confirm card.** The card shows what will happen, the amount, the network, fees and your limits.
7. **You tap Confirm.** Lexari checks every rule again, signs from the agent's own wallet and returns a receipt with an explorer link that stays in the chat history.

The model never holds a key and never signs. Its job is to understand you, plan the steps, pick the right tools with correct inputs and explain clearly. Lexari's server does the checking and the executing.

## Where Qwen fits

### A model you choose

Lexari's model picker lists Lamina (Lexari's own model) first, then a set of open models included on every plan. Qwen is one of them. You can pick it per agent or per chat, and the chat header shows which model is answering. The entry is defined in [webapp/content/models.ts](https://github.com/xElvolution/lexari/blob/main/webapp/content/models.ts):

```ts
{ id: "qwen", label: "Qwen", short: "Qwen", maker: "Alibaba",
  blurb: "Alibaba's strongest Qwen. Careful multi-step reasoning.",
  pool: "lamina", price: { in: 1.2, out: 6 },
  house: { provider: "dashscope", fallback: ["kimi", "openai"] } },
```

Because it sits in the included pool, using Qwen is billed like Lamina from your plan. You never need your own Alibaba Cloud key.

### How it is served

Qwen runs through Alibaba Cloud Model Studio (DashScope) on its OpenAI compatible endpoint. The server reads its settings from the environment, with these defaults, in [webapp/server/engram/house.ts](https://github.com/xElvolution/lexari/blob/main/webapp/server/engram/house.ts):

```ts
dashscope: { key: "DASHSCOPE_API_KEY", base: "DASHSCOPE_BASE_URL",
  baseDefault: "https://dashscope-intl.aliyuncs.com/compatible-mode/v1",
  model: "QWEN_MODEL", modelDefault: "qwen3.8-max", who: "Qwen",
  extra: { enable_thinking: false } },
```

A few choices are worth calling out:

- **`qwen3.8-max` by default**, overridable with `QWEN_MODEL` so the team can move to a newer Qwen without a code change.
- **The international Model Studio endpoint** by default, overridable with `DASHSCOPE_BASE_URL`.
- **`enable_thinking: false`.** Replies stream straight to the person. The careful step by step behavior we rely on comes from the model itself and from the way Lexari's prompt lays out tools and rules, not from a separate hidden thinking phase that would delay the first word.
- **If the key is missing**, Qwen is listed as unavailable in the picker and cannot be chosen. Nothing breaks silently.

### The fallback chain

Providers have bad minutes. Lexari plans for that. A turn on Qwen tries Qwen first, then its fallbacks that are ready on the server (Kimi, then OpenAI), and finally Lamina. The order comes from `houseChain` in house.ts, and the loop lives in [webapp/server/engram/cortex.ts](https://github.com/xElvolution/lexari/blob/main/webapp/server/engram/cortex.ts):

```ts
for (const m of chain) {
  const t = houseTarget(m);
  if (!t) continue;
  let any = false;
  try {
    for await (const tok of streamByo(withNote, t, { signal, maxTokens, onUsage })) { any = true; yield tok; }
    return;
  } catch (e) {
    // Only switch before any text reached the person, never mid-reply.
    if (any || signal?.aborted) throw e;
  }
}
```

The important rule is in the comment: **a fallback only happens before any text has reached you.** You never see half an answer from one model glued to half an answer from another.

When Qwen is answering, Lexari also adds a short system note so the agent is honest about it: if asked which model it is, it says it is running on Qwen inside Lexari.

## What Qwen does inside a turn

Qwen's job in an agent can be broken into four parts.

### 1. Understand the request

People do not write in API calls. They write "swap twenty bucks of USDC into MON and then pay Ada five". Qwen has to pull out the actions, the assets, the amounts, the venue and the recipient, and notice what is missing (Ada's address, for example) instead of guessing.

### 2. Plan the steps

A request with two money actions has an order and dependencies. The swap needs a quote first. The payment needs a valid address. Lexari takes at most two tool calls from one reply, so the model has to decide which steps happen now and which happen after a result comes back. This is where Qwen's careful multi-step reasoning matters most: it sequences the work instead of firing everything at once.

### 3. Choose tools with valid inputs

Each tool advertises a short hint with its input format. For example, the AUSD payment tool in [webapp/server/integrations/ausd.ts](https://github.com/xElvolution/lexari/blob/main/webapp/server/integrations/ausd.ts):

```ts
hint: 'ausd.pay {"to":"0x...","amount":5,"note":"lunch"} : pay someone in Agora AUSD from your Monad wallet. The person confirms it.',
```

and its schema:

```ts
const payIn = z.object({ to: z.string().trim(), amount: z.coerce.number().positive(),
  note: z.string().max(120).optional() }).passthrough();
```

The Kuru tools in [webapp/server/integrations/monad.ts](https://github.com/xElvolution/lexari/blob/main/webapp/server/integrations/monad.ts) take:

```ts
const kuruIn = z.object({ sell: z.string().trim(), buy: z.string().trim(),
  amount: z.coerce.number().positive(), slippage: z.coerce.number().positive().max(50).optional() });
```

Qwen writes the JSON. Lexari checks it with `safeParse` in [webapp/server/integrations/actions.ts](https://github.com/xElvolution/lexari/blob/main/webapp/server/integrations/actions.ts) before anything runs:

```ts
const parsed = tool.input.safeParse(input ?? {});
if (!parsed.success) {
  return { kind: "refused", tool: tool.name,
    text: `the ${info.name} request was missing details (${parsed.error.issues[0]?.path.join(".") || "input"})` };
}
```

A model that reads the format carefully produces fewer refusals and fewer round trips. When one does happen, the refusal names the missing field, and the agent can ask you for it.

### 4. Explain before acting

Sign tools never execute from the chat. They prepare a Confirm card, and the agent is told so directly in the tool instructions from [webapp/server/integrations/chat.ts](https://github.com/xElvolution/lexari/blob/main/webapp/server/integrations/chat.ts):

> Sign tools only prepare a Confirm card for the person: nothing moves until they tap Confirm, so say it's ready to confirm and never claim it went through.

Qwen's part is to say, in plain words, what it prepared and why, so the card makes sense the moment you see it.

## A worked example, step by step

Here is how one request moves through an agent running on Qwen. The agent has Kuru and Agora AUSD switched on, and its Monad wallet holds test USDC and AUSD.

**You:** "Swap 20 USDC to MON on Kuru, then send 5 AUSD to Ada."

**Step 1: read the request.** Qwen identifies two money actions in order: a swap (sell USDC, buy MON, amount 20, venue Kuru) and a payment (5 AUSD to Ada). It checks the chat history and the memories you kept for Ada's address. If it does not find one, it plans to ask rather than invent one.

**Step 2: get a live quote.** Qwen writes a read tool call:

```text
<tool name="kuru.quote">{"sell":"USDC","buy":"MON","amount":20}</tool>
```

Lexari validates the input against `kuruIn`, confirms the agent has the Kuru grant, runs the quote on Monad testnet and hands the result back to the model.

**Step 3: prepare the swap.** With a real quote in hand, Qwen tells you what it found and writes the sign tool call:

```text
<tool name="kuru.swap">{"sell":"USDC","buy":"MON","amount":20}</tool>
```

Lexari validates it again, checks the per trade and daily limits for Kuru and your slippage setting, and prepares a **Confirm card**: "Swap 20 USDC for MON on Kuru", the amount in coins and dollars, the minimum you will receive, the network and how much of today's limit is used. Nothing has moved.

**Step 4: the payment.** If Ada's address is known, Qwen prepares the second step:

```text
<tool name="ausd.pay">{"to":"0x...","amount":5,"note":"for Ada"}</tool>
```

That becomes a second Confirm card. If her address is not known, Qwen asks you for it instead of producing a call that would fail validation.

**Step 5: you decide.** You tap **Confirm** on each card, or **Cancel**. On Confirm, Lexari re-checks everything under a per person lock: the grant is still live, the limits still hold, and the quote is not stale. A card older than 15 minutes is rejected with "This quote expired. Ask again for a fresh one." Then it signs from the agent's Monad wallet.

**Step 6: receipts and history.** Each confirmed action returns a receipt with an explorer link, and the outcome is written back into the chat. Next time you say "do the same as last week", that history is there for the model to read.

The pattern is the same for a cross-chain route: a read tool for the route quote, then a sign tool that becomes a card.

## Safety by design

The model is powerful, so Lexari does not trust it with money. The rules are enforced on the server, not in the prompt.

- **Nothing moves without a Confirm card.** Sign tools only prepare. Execution happens in `confirmAction`, after your tap.
- **Validated inputs.** Every tool call is parsed against a strict schema. Missing or malformed fields are refused before any network call.
- **Only granted tools.** An agent can only use integrations you added and granted to that agent. Anything else is refused.
- **Limits checked twice.** Per trade and daily limits, slippage and agent budgets are checked when the card is prepared and again when you confirm.
- **Expired quotes rejected.** A card that sits too long cannot be confirmed on old prices.
- **Prompt injection defence.** Text from web pages, emails, files and tool output is data, never instructions. The agent will not pay because something it read asked it to.
- **Testnets only for now.** Execution runs on devnet and testnets with test funds while agent signing moves to policy-bound keys.

Qwen's careful reasoning makes the agent pleasant and accurate. These rules make it safe regardless of which model is answering.

## The value Qwen brought

We did not run formal benchmarks for this article, so there are no numbers here. What we can describe is what Qwen gave the product in practice.

- **Ordered plans for multi-step money requests.** "Do this, then that" requests are where agents usually go wrong. Qwen's strength in careful multi-step reasoning fits Lexari's flow, where a quote comes before a swap and a valid address comes before a payment.
- **Inputs that fit the schema.** Qwen follows the tool formats closely, which is exactly what a strict validator rewards.
- **Clear explanations before the card.** The agent says what it is about to prepare in plain English, so the Confirm card is never a surprise.
- **A strong open model on every plan.** Through Model Studio, Lexari can offer Qwen 3.8 Max to every user without asking them for an API key.
- **Resilience.** As part of the fallback chain, Qwen also backs up Kimi, so an agent keeps answering when one provider is down.

## Code references

All files are in [github.com/xElvolution/lexari](https://github.com/xElvolution/lexari):

| File | What it shows |
| --- | --- |
| [webapp/content/models.ts](https://github.com/xElvolution/lexari/blob/main/webapp/content/models.ts) | The Qwen entry, its blurb, pricing pool and fallback list |
| [webapp/server/engram/house.ts](https://github.com/xElvolution/lexari/blob/main/webapp/server/engram/house.ts) | Model Studio endpoint, `qwen3.8-max` default, readiness and fallback chain |
| [webapp/server/engram/cortex.ts](https://github.com/xElvolution/lexari/blob/main/webapp/server/engram/cortex.ts) | The streaming loop that tries each model and only falls back before any text |
| [webapp/server/engram/spinal.ts](https://github.com/xElvolution/lexari/blob/main/webapp/server/engram/spinal.ts) | Prompt building and the safety rules every agent follows |
| [webapp/server/integrations/chat.ts](https://github.com/xElvolution/lexari/blob/main/webapp/server/integrations/chat.ts) | How tools are offered to an agent and how tags are read from a reply |
| [webapp/server/integrations/actions.ts](https://github.com/xElvolution/lexari/blob/main/webapp/server/integrations/actions.ts) | Input validation, Confirm card preparation, re-checks and quote expiry |
| [webapp/server/integrations/monad.ts](https://github.com/xElvolution/lexari/blob/main/webapp/server/integrations/monad.ts) | Kuru quote and swap tools on Monad |
| [webapp/server/integrations/ausd.ts](https://github.com/xElvolution/lexari/blob/main/webapp/server/integrations/ausd.ts) | Agora AUSD pay and send tools |

## What's next

- **More multi-step flows** that chain a quote, a swap and a payment into one clear set of cards.
- **Richer cross-chain routing** that Qwen can plan across, with the same Confirm card at every step.
- **Mainnet**, once agent signing runs on policy-bound keys.

## Try it

Create your own agent at [app.lexari.ai](https://app.lexari.ai), open the model picker in the chat and choose **Qwen**. Ask it to quote a swap on Kuru or pay a friend in AUSD, and watch it plan the steps and hand you a Confirm card before anything moves.
