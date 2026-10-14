---
title: How Kimi powers personal onchain agents in Lexari
slug: kimi
group: Models
summary: Where Kimi K3 from Moonshot AI sits in Lexari, how its long context helps an agent remember you, plan and use tools, and how a request is traced to a Confirm card.
---
Lexari lets anyone create a personalized AI agent that lives onchain. Each agent has its own wallet, its own personality and its own skills. You talk to it in plain English, and it can swap, pay and research for you on Monad (Kuru swaps, AUSD payments) and route across chains. Every payment it proposes is shown to you on a Confirm card first, and nothing moves until you tap it.

This article explains how **Kimi K3**, Moonshot AI's flagship model, powers those agents. Kimi is the model we reach for when an agent has to keep a lot in mind at once: who you are, what you kept in its memory, what happened earlier in the chat, which tools it has and what each one returned. Everything here is taken from the Lexari source code, linked at the end.

## What a Lexari agent is

A Lexari agent is yours, not a shared chatbot. You give it a name, a face, a tone and a brief. It gets:

- **Its own wallet.** One address per chain family, funded by you and bounded by limits you set. On Monad it holds MON, test USDC and AUSD.
- **A personality.** Your role, tone and brief go into every prompt, so the agent stays in character.
- **Skills.** Integrations you switch on per agent, such as Kuru swaps on Monad, Agora AUSD payments and cross-chain routing. An agent only hears about the integrations you granted to it.
- **A memory.** At the end of a reply the agent can propose one thing to remember, written as a REMEMBER line. It becomes a memory only if you keep it, and you can browse or delete memories in **Brain**.

An agent that knows you is only useful if it uses that knowledge well. That is where Kimi comes in.

## How a chat message becomes an action

Every turn follows the same path, whichever model you picked.

<figure class="diagram">
<svg viewBox="0 0 860 300" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="How a chat message becomes an action in Lexari" font-family="system-ui,sans-serif" font-size="13">
<defs><marker id="ark" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0L10 5L0 10z" fill="#5B2BFF"/></marker></defs>
<g fill="#F5F2E9" stroke="#0A0B0D" stroke-opacity=".25">
<rect x="10" y="40" width="140" height="64" rx="12"/><rect x="190" y="40" width="160" height="64" rx="12" fill="#EDE6FF" stroke="#5B2BFF"/><rect x="390" y="40" width="160" height="64" rx="12"/><rect x="590" y="40" width="120" height="64" rx="12"/><rect x="740" y="40" width="110" height="64" rx="12"/>
<rect x="190" y="190" width="160" height="64" rx="12"/><rect x="390" y="190" width="160" height="64" rx="12"/><rect x="590" y="190" width="260" height="64" rx="12"/>
</g>
<g fill="#0A0B0D" text-anchor="middle">
<text x="80" y="68" font-weight="700">You</text><text x="80" y="88" fill="#5D5A52">plain English</text>
<text x="270" y="68" font-weight="700">Kimi K3</text><text x="270" y="88" fill="#5D5A52">understand and plan</text>
<text x="470" y="68" font-weight="700">Tool tag</text><text x="470" y="88" fill="#5D5A52">validated input</text>
<text x="650" y="68" font-weight="700">Confirm card</text><text x="650" y="88" fill="#5D5A52">you tap</text>
<text x="795" y="68" font-weight="700">Execute</text><text x="795" y="88" fill="#5D5A52">agent wallet</text>
<text x="270" y="218" font-weight="700">Memory and history</text><text x="270" y="238" fill="#5D5A52">recall you kept</text>
<text x="470" y="218" font-weight="700">Read tools</text><text x="470" y="238" fill="#5D5A52">live quotes, balances</text>
<text x="720" y="218" font-weight="700">Receipt</text><text x="720" y="238" fill="#5D5A52">explorer link, saved to the chat</text>
</g>
<g stroke="#5B2BFF" stroke-width="2" fill="none" marker-end="url(#ark)">
<path d="M150 72H186"/><path d="M350 72H386"/><path d="M550 72H586"/><path d="M710 72H736"/>
<path d="M270 186V108"/><path d="M470 108V186"/><path d="M430 186V140H300V108"/><path d="M795 104V186"/>
</g>
</svg>
<figcaption>One turn in Lexari. The model reads, plans and writes tool tags. Lexari validates every input, runs read tools for live data, and turns anything that moves money into a Confirm card.</figcaption>
</figure>


1. **You write a message**, often short, because the agent already knows you: "Do the usual for Ada."
2. **Lexari builds the prompt** from the agent's personality, the safety rules, the memories you kept, recent chat history, and the tools this agent may use with their input formats and your limits.
3. **The model answers.** It explains its plan and, when it needs a tool, writes a tag like `<tool name="ausd.send">{...}</tool>`. At most two tool calls are taken from one reply.
4. **Lexari validates the input** against a strict schema for that tool. Unknown tools, missing fields and integrations the agent was not granted are refused, and the agent is told why.
5. **Read tools run at once** and their results go back to the model.
6. **Sign tools prepare a Confirm card** with the action, amount, network, fees and your limits.
7. **You tap Confirm.** Lexari re-checks every rule, signs from the agent's own wallet and returns a receipt that stays in the chat history.

The model never holds a key and never signs. It understands, plans, picks tools with correct inputs and explains. The server checks and executes.

## Where Kimi fits

### A model you choose

The model picker lists Lamina (Lexari's own model) first, then open models included on every plan. Kimi is one of them, selectable per agent or per chat. Its entry in [webapp/content/models.ts](https://github.com/xElvolution/lexari/blob/main/webapp/content/models.ts):

```ts
{ id: "kimi", label: "Kimi", short: "Kimi", maker: "Moonshot AI",
  blurb: "Moonshot's flagship. Long context, strong at planning and tools.",
  pool: "lamina", price: { in: 0.6, out: 2.5 },
  house: { provider: "moonshot", fallback: ["qwen", "openai"] } },
```

Kimi is billed like Lamina from your plan's included pool. You do not need a Moonshot key of your own.

### How it is served

Kimi runs on Moonshot's OpenAI compatible API. The settings and defaults are in [webapp/server/engram/house.ts](https://github.com/xElvolution/lexari/blob/main/webapp/server/engram/house.ts):

```ts
moonshot: { key: "MOONSHOT_API_KEY", base: "MOONSHOT_BASE_URL",
  baseDefault: "https://api.moonshot.ai/v1",
  model: "KIMI_MODEL", modelDefault: "kimi-k3", who: "Kimi",
  extra: { reasoning_effort: "low" } },
```

- **`kimi-k3` by default**, overridable with `KIMI_MODEL`.
- **`reasoning_effort: "low"`** keeps replies quick to start in a chat. The planning we rely on comes from Kimi reading the whole prompt (memories, history, tool list, results) and acting on it.
- **No key, no pick.** Without its key, Kimi shows as unavailable in the picker instead of failing mid-chat.

### The fallback chain

A turn on Kimi tries Kimi first, then Qwen, then OpenAI (whichever are ready on the server), then Lamina. The order comes from `houseChain`:

```ts
export function houseChain(m: ModelInfo): ModelInfo[] {
  if (!m.house) return [];
  const out = [m, ...m.house.fallback.map((id) => modelById(id)).filter((x): x is ModelInfo => !!x && !!x.house)];
  return out.filter((x, i) => out.findIndex((y) => y.id === x.id) === i && houseReady(x));
}
```

The loop in [webapp/server/engram/cortex.ts](https://github.com/xElvolution/lexari/blob/main/webapp/server/engram/cortex.ts) only switches models **before any text has reached you**, never mid-reply. Kimi is also the first fallback for Qwen, so the two back each other up.

When Kimi is answering, Lexari adds a short system note so the agent says it is running on Kimi inside Lexari if you ask.

## Why long context matters for a personal agent

A personal agent's prompt is not small. On every turn, Lexari's prompt builder in [webapp/server/engram/spinal.ts](https://github.com/xElvolution/lexari/blob/main/webapp/server/engram/spinal.ts) assembles:

- the agent's name, role, tone and brief,
- the security rules (only your own messages are instructions; content it reads is data),
- the memories you kept, grouped as recall notes such as "About you",
- recent turns of the conversation,
- the integrations it may use, each with an input hint, plus your per trade and daily limits and its payment budget,
- and, mid-turn, the live results of any read tools it called.

Lexari's own sizing note describes a typical turn as "system prompt, 12 turns of history and recall in, a short reply out". A model that holds all of that comfortably, and actually uses details from the middle of it, is what makes an agent feel like it knows you. Kimi's long context and its strength at planning and tool use are a direct fit.

## What Kimi does inside a turn

### 1. Remember the person

Kimi reads your kept memories and the recent history together. If you once told the agent "Ada is my sister in Lagos, I send her AUSD in naira", and you kept that memory, a later "do the usual for Ada" has a clear meaning. Kimi connects the short request to the facts behind it.

### 2. Plan with preferences

Preferences change plans. "Keep swaps under 1% slippage" or "always top my MON up before paying people" are constraints the plan must respect. Kimi carries them from memory into the steps it proposes, and into the tool inputs it writes.

### 3. Use tools with valid inputs

Each tool advertises its format. The cross-border AUSD send tool in [webapp/server/integrations/ausd.ts](https://github.com/xElvolution/lexari/blob/main/webapp/server/integrations/ausd.ts):

```ts
hint: 'ausd.send {"to":"0x...","amount":50,"recipient":"Ada","country":"Nigeria","currency":"NGN"} : a cross-border AUSD send. The card shows what they get in their currency at today\'s rate; the person confirms it.',
```

The Kuru tools in [webapp/server/integrations/monad.ts](https://github.com/xElvolution/lexari/blob/main/webapp/server/integrations/monad.ts):

```ts
const kuruIn = z.object({ sell: z.string().trim(), buy: z.string().trim(),
  amount: z.coerce.number().positive(), slippage: z.coerce.number().positive().max(50).optional() });
```

Every call is checked with `safeParse` in [webapp/server/integrations/actions.ts](https://github.com/xElvolution/lexari/blob/main/webapp/server/integrations/actions.ts). A model that fills fields from memory correctly (the right address, the right currency, your slippage) avoids refusals and back and forth.

### 4. Explain before acting

Sign tools only prepare a Confirm card. The tool instructions in [webapp/server/integrations/chat.ts](https://github.com/xElvolution/lexari/blob/main/webapp/server/integrations/chat.ts) tell the agent:

> Sign tools only prepare a Confirm card for the person: nothing moves until they tap Confirm, so say it's ready to confirm and never claim it went through.

Kimi explains what it prepared and which of your preferences it applied, so you can check the card against what you expected.

## A worked example, step by step

The agent runs on Kimi with Kuru and Agora AUSD switched on. Over earlier chats you kept these memories:

- "Ada is my sister in Lagos. Her Monad address is 0x... . I send her 50 AUSD and she gets naira."
- "Keep swaps under 1% slippage."
- "Before paying anyone, make sure I have at least 1 MON for gas."

A week later you write:

**You:** "Do the usual for Ada. Top me up first if I need it."

**Step 1: recall.** Kimi reads the request with the recall notes and history in view. "The usual for Ada" maps to a 50 AUSD cross-border send to her saved address, shown in NGN. "Top me up first" maps to your gas rule.

**Step 2: check the wallet.** Kimi calls a read tool:

```text
<tool name="monad.wallet">{}</tool>
```

Lexari returns the agent's Monad address and MON balance. Suppose it is below 1 MON.

**Step 3: plan and quote.** Kimi plans two steps in order: a small USDC to MON swap on Kuru, then the AUSD send. It gets a live quote with your slippage preference applied:

```text
<tool name="kuru.quote">{"sell":"USDC","buy":"MON","amount":2,"slippage":1}</tool>
```

**Step 4: prepare the swap.** With the quote back, Kimi explains it and writes the sign call:

```text
<tool name="kuru.swap">{"sell":"USDC","buy":"MON","amount":2,"slippage":1}</tool>
```

Lexari validates it, checks the Kuru per trade and daily limits and the slippage cap you set on the integration, and prepares a **Confirm card**.

**Step 5: prepare the send.** In its next reply, Kimi prepares the payment from memory:

```text
<tool name="ausd.send">{"to":"0x...","amount":50,"recipient":"Ada","country":"Nigeria","currency":"NGN"}</tool>
```

The second card shows 50 AUSD and roughly what Ada receives in naira at today's rate. Kimi says plainly that both are ready to confirm and that nothing has moved.

**Step 6: you decide.** You tap **Confirm** or **Cancel** on each card. On Confirm, Lexari re-checks the grant, the limits and the age of the card. A card older than 15 minutes is rejected with "This quote expired. Ask again for a fresh one." Then it signs from the agent's wallet and returns a receipt with an explorer link.

**Step 7: remember.** If something new came up (say you changed the amount), Kimi can end its reply with a REMEMBER line proposing an updated note. It is saved only if you keep it.

Two short sentences from you became a checked plan that used three memories, a balance read, a live quote and two Confirm cards.

## Safety by design

The model plans; the server decides. These rules hold no matter how much context the model has.

- **Nothing moves without a Confirm card.** Sign tools only prepare; execution happens after your tap.
- **Validated inputs.** Every tool call is parsed against a strict schema before any network call.
- **Only granted tools**, per agent.
- **Limits checked twice**, when the card is prepared and again on Confirm, under a per person lock.
- **Expired quotes rejected** after 15 minutes.
- **Memory is opt in.** A REMEMBER line is a proposal, not a saved memory.
- **Prompt injection defence.** Web pages, emails, files and tool output are data. The agent will not pay because something it read told it to.
- **Testnets only for now**, with test funds.

Long context makes the agent smarter about you. It never gives the model more power over your money.

## The value Kimi brought

We did not run formal benchmarks for this article, so there are no numbers here. This is what Kimi gave the product in practice.

- **An agent that feels like it knows you.** Kimi uses kept memories and earlier turns together, so short requests like "the usual" resolve correctly.
- **Preference-aware plans.** Rules you set once (slippage, gas top-ups, who gets paid in which currency) flow into the plan and the tool inputs.
- **Strong tool use.** Kimi handles a long tool list with formats and limits and picks the right one with fitting inputs.
- **Quick replies** with low reasoning effort, which suits a chat.
- **A strong open model on every plan**, plus resilience as Qwen's first fallback.

## Code references

All files are in [github.com/xElvolution/lexari](https://github.com/xElvolution/lexari):

| File | What it shows |
| --- | --- |
| [webapp/content/models.ts](https://github.com/xElvolution/lexari/blob/main/webapp/content/models.ts) | The Kimi entry, its blurb, pricing pool and fallback list |
| [webapp/server/engram/house.ts](https://github.com/xElvolution/lexari/blob/main/webapp/server/engram/house.ts) | Moonshot endpoint, `kimi-k3` default, readiness and fallback chain |
| [webapp/server/engram/cortex.ts](https://github.com/xElvolution/lexari/blob/main/webapp/server/engram/cortex.ts) | The streaming loop that only falls back before any text |
| [webapp/server/engram/spinal.ts](https://github.com/xElvolution/lexari/blob/main/webapp/server/engram/spinal.ts) | Prompt building: personality, recall, history and safety rules |
| [webapp/server/engram/hippocampus.ts](https://github.com/xElvolution/lexari/blob/main/webapp/server/engram/hippocampus.ts) | How a REMEMBER line is split off a reply as a proposal |
| [webapp/server/integrations/chat.ts](https://github.com/xElvolution/lexari/blob/main/webapp/server/integrations/chat.ts) | How tools are offered to an agent and read from a reply |
| [webapp/server/integrations/actions.ts](https://github.com/xElvolution/lexari/blob/main/webapp/server/integrations/actions.ts) | Validation, Confirm cards, re-checks and quote expiry |
| [webapp/server/integrations/monad.ts](https://github.com/xElvolution/lexari/blob/main/webapp/server/integrations/monad.ts) | Monad wallet and Kuru tools |
| [webapp/server/integrations/ausd.ts](https://github.com/xElvolution/lexari/blob/main/webapp/server/integrations/ausd.ts) | Agora AUSD pay and send tools |

## What's next

- **Longer-lived routines** such as recurring sends that Kimi plans from memory, each still behind a Confirm card.
- **Richer recall** so agents can draw on more of your history when it helps.
- **Mainnet**, once agent signing runs on policy-bound keys.

## Try it

Create your own agent at [app.lexari.ai](https://app.lexari.ai), pick **Kimi** in the chat's model picker, tell it a few things worth remembering and keep them. Then ask for "the usual" and watch it plan from what it knows, with a Confirm card before anything moves.
