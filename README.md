# TESSA & COLE

**Token-Efficient Self-hosted Smart Agent & Context-Optimized Learning Engine**

*No more waking up to a massive API bill.*

---

## The Story

A friend called me on a Saturday morning. Not to catch up. To vent. He'd woken up to a $340 API bill. His AI assistant had hit an error loop overnight. Every retry loaded the entire system context, 20,000 tokens of tools, skills, memory, and instructions, just to fail again. Nine hundred calls. While he slept.

He's a senior engineer. Years of experience. Built systems at scale. And none of that mattered. The bill still hit him in his sleep.

Turns out he's not alone. Not even close. I started paying attention and this stuff is happening everywhere:

- A developer [burned $8,000 in 11 days](https://medium.com/artificial-intelligence-field-guide/we-burned-8-000-in-ai-api-costs-because-we-ignored-one-simple-signal-10a9706a6627) because a small tweak to a system prompt made responses longer. Nobody noticed until the invoice came.
- A team's AI chatbot [cost $14,000 in three weeks](https://medium.com/lets-code-future/our-ai-chatbot-cost-14k-in-openai-bills-before-i-fixed-these-5-mistakes-a9c4029c3cce). $670 per day. For a customer support bot.
- One developer [built a cost monitor after a $500 shock](https://medium.com/@reachbrt/i-built-an-advanced-openai-usage-monitor-after-a-500-bill-shock-now-open-source-b98fc89c2017) from what they thought was a small experiment.
- Another team's AI bill [jumped from $1,200 to $4,800 in a single month](https://medium.com/@nirbhaysingh1/our-ai-bill-was-4-800-last-month-nobody-knew-why-so-i-built-an-open-source-llm-cost-tracker-018cdbdf9a6b). Nobody knew why.
- A startup got [an $82,000 Gemini API bill in 48 hours](https://www.pointguardai.com/blog/when-a-stolen-ai-api-key-becomes-an-82-000-problem) after a key was stolen. Their normal spend was $180/month.
- On the OpenAI forums, a developer [posted "Massive spend today, WTF went wrong?"](https://community.openai.com/t/massive-spend-today-wtf-went-wrong/83191) after going from $0.50/day to $31 overnight.

These aren't beginners. These are experienced builders who set up their own AI systems, connected them to their tools, and tried to make something useful. And the reward for their ambition was an API bill that made them sick.

I started calling them **API-mares**. The nightmares you get from AI APIs.

Then it happened to me.

I checked my dashboard one morning. Ten dollars gone. Overnight. While I was sleeping. My setup on OpenClaw had been running fine for weeks. Nine AI agents, twenty skills, CRM, health tracking, meeting transcripts, the works. Then something glitched and the retry loop kicked in. Every retry loaded 18,000 tokens of context. For errors that went nowhere.

That morning I finally understood the problem. It wasn't the error loop. That was just the symptom. The real problem was that **every single message, no matter how simple, loaded everything.**

When I said "tell me a joke," my system loaded 18,000 tokens of context. Tools I didn't need. Skills I wasn't using. Memory from six months ago. The full blob. Every time.

When I asked "what time is my meeting?" 18,000 tokens. To look up a calendar entry.

When I processed a meeting transcript and it failed twice? 54,000 tokens of pure waste.

The math was brutal. 200 messages a day, 18,000 tokens each, 30 days a month. That's 108 million tokens. At the rates these models charge, that's $30 to $100 a month. And most of those tokens were completely unnecessary for the task at hand.

So I started optimizing. I trimmed the system prompt from 57,000 characters to 15,000. Switched models for routine tasks. Added circuit breakers. And it helped. The cost dropped from $8/day to under $1.

But I kept hitting a wall. OpenClaw (and every other AI gateway I looked at) loads the same blob of context on every single call. There's no way to say "for this message, only load the calendar" or "for this one, skip the tools and just use memory." The gateway doesn't know what you need. It just sends everything, every time.

I got tired of working around the problem. I wanted to solve it.

So I built TESSA & COLE.

Not a fork of OpenClaw. Not a plugin. A standalone system that can work by itself, or sit in front of OpenClaw (or Open WebUI) as a smart router. It reads the same files, shares the same memory, uses the same skills. But before anything hits the AI, TESSA & COLE ask the question that nobody else is asking:

**What do you actually need to answer this?**

---

## What TESSA & COLE Does

TESSA & COLE are a standalone AI command center. You can run it on its own, or put it in front of OpenClaw or Open WebUI as a smart router that decides what context to load before passing the request through. Same file formats, same memory directories, same skills. It's fully compatible but completely independent.

Every incoming message gets one question: *"What do you actually need to answer this?"*

- **"Tell me a joke"** = Zero context loaded. Cheapest model. Cost: $0.0001.
- **"What time is my meeting?"** = Don't even call the AI. Run a local calendar script. Cost: $0.00.
- **"Brief me on Acme Corp before my call"** = Pull just that contact's file and recent meetings. 3,000 tokens to a mid-tier model. Cost: $0.03.
- **"Analyze 20 years of journal entries"** = OK, now load the full context and use the best model. Cost: $0.50. But it's justified.

Simple idea. Took months to build. But the results are real:

| | Before | After |
|---|---|---|
| **"Tell me a joke"** | 18,000 tokens | 200 tokens |
| **Calendar lookup** | 18,000 tokens | 0 (local script) |
| **Meeting prep** | 18,000 tokens | 3,000 tokens |
| **Contact lookup** | 18,000 tokens | 0 (local script) |
| **Error retry x 3** | 54,000 tokens | 0 (circuit breaker stops it) |
| **Daily cost** | $8+ | Under $1 |
| **Monthly cost** | $240 | $15-25 |

70-95% reduction. Same quality. Often better, because the right model gets the right context instead of drowning in irrelevant information.

---

## Meet TESSA & COLE

**TESSA** is the **Token-Efficient Self-hosted Smart Agent.** She's the orchestrator. Decides what to load, which model to use, how to keep costs down. She's the reason your API bill drops 70-95%.

**COLE** is the **Context-Optimized Learning Engine.** He's the brain. Remembers things, learns from mistakes, gets smarter over time without you retraining anything. He's the reason the system never makes the same mistake twice.

Together they're your AI command center. Self-hosted. Private. Affordable.

---

## The Principles

These aren't abstract ideals. Every single one came from a real problem that cost real money or lost real data.

### 1. Minimum Viable Context
The core philosophy. Every token costs money. If you don't need it for this specific request, don't load it. Period.

*We had a 57,000-character system prompt loading on every single message. Cut it to 15,000 characters, a 74% reduction, by splitting it into "always loaded" and "loaded on demand." Daily cost dropped from $8 to under $1.*

### 2. Right Model for the Right Job
A joke doesn't need GPT-5. A calendar lookup doesn't need any model at all. A deep analysis of your journal might need the best model available. TESSA & COLE pick the right one automatically.

*Running everything through Gemini Pro cost $8/day. Switching routine tasks to Gemini Flash (8x cheaper) dropped it to under $1/day. No quality loss for those tasks.*

### 3. Fail Cheaply
When something goes wrong, stop retrying with the full context. A circuit breaker cuts off after 3 attempts. The $10 overnight nightmare never happens again.

*900 retry calls, each loading 18,000 tokens, while everyone was asleep. That was the morning that started all of this.*

### 4. Learn From Mistakes
TESSA & COLE capture every correction you make and every approach that works. Weekly, a cheap AI model distills them into rules. The system stops repeating the same mistakes, across all your agents.

*We corrected the same routing error 6 times before building an autolearning system. Now corrections are captured instantly and applied everywhere.*

### 5. Scripts, Not Conversations
When you ask "who is Sarah Chen?" don't send that to a $0.01 AI call. Run a local Python script that reads the contact file directly. Zero tokens. Instant answer.

*The AI was calling itself to look up contacts sitting in a file on disk. Every lookup cost money for no reason.*

### 6. Privacy by Architecture
Your data stays on your hardware. API calls send only the minimum context needed. Everything encrypted at rest. The AI sees what it needs for this specific task and nothing more.

*A social media agent accidentally included personal details in a public post because it had access to the full memory instead of just its own workspace.*

### 7. No More API-mares
Budget caps. Circuit breakers. Cost logging on every single call. Daily spending alerts. You know exactly where every dollar goes. Nothing runs away while you sleep.

*You know the story by now.*

---

## Who This Is For

TESSA & COLE are for the person who:

- **Set up an AI assistant** and got excited about what it could do, then got the bill
- **Wants AI to help run their life** (calendar, contacts, health, finances, projects) without paying enterprise prices
- **Values privacy** because your journal entries, health data, and business contacts shouldn't live on someone else's server
- **Isn't necessarily a developer** but is willing to follow a guide and run a few commands
- **Got burned by an API-mare** and wants to make sure it never happens again

You don't need a powerful computer. TESSA & COLE run on a $12/month cloud server. No GPU. No machine learning expertise. Just an internet connection and the willingness to take control of your AI stack.

---

## What It Can Do (When It's Built)

TESSA & COLE are being built in public. Here's what the finished system looks like:

- **Smart routing** that classifies every message and routes it to the cheapest option that can handle it well
- **9+ AI agents** for personal assistant, CRM, document processing, health tracking, social media, and more
- **20+ skills** for calendar, contacts, finance, transcripts, web search, document OCR, and more
- **Hybrid memory** with semantic search + keyword search + relationship graph. Ask "who works with Sarah at Acme Corp?" and get a real answer
- **Morning reports** so you wake up to a briefing with your calendar, todos, payment alerts, health flags, relationship reminders, and trends
- **Meeting prep** where "brief me on Sarah before my call" pulls just what you need
- **Document intelligence** that takes a PDF, transcript, or photo and extracts contacts, action items, health data, filing everything in the right place
- **Autolearning** that captures what works and what doesn't, getting smarter every week for $0.01/week
- **Magic links** to share a temporary chat link with someone, they talk to a scoped AI, you get the summary, link expires automatically
- **Encrypted everything** with LUKS vault, age encryption, encrypted directory overlays. Your data is yours

---

## The Journey

This project started as a personal system running on top of OpenClaw. It ran my life for months. Managing contacts, tracking health data, processing meeting transcripts, posting to social media, coordinating with family. Every principle in the documentation was tested against real daily use on a $12/month DigitalOcean droplet.

Now it's becoming open source. Not because it's finished, but because building in public is how the best tools get made.

The documentation in the `docs/` folder tells the full story:

| Document | What's Inside |
|----------|--------------|
| [**Part 1: Vision & Architecture**](docs/part1-vision-architecture.md) | The origin story, principles, and system overview |
| [**Part 2: Core Systems**](docs/part2-core-systems.md) | The Context Chef, Router, and how Minimum Viable Context actually works |
| [**Part 3: Knowledge & Memory**](docs/part3-knowledge-memory.md) | How the 7-phase hybrid search system evolved |
| [**Part 4: Skills, Security & Ops**](docs/part4-skills-security-ops.md) | Every skill, security layer, and operational lesson |
| [**Part 5: Build Plan & Reference**](docs/part5-build-plan-reference.md) | The build phases, data formats, and API design |
| [**Part 6: Dashboard**](docs/part6-dashboard.md) | The command center UI, design system, and screen layouts |

Each document is written with real production data, real cost numbers, and real lessons learned. No hand-waving. No "left as an exercise for the reader."

---

## Current Status

**Phase: Documentation & Architecture**

The architecture is complete and battle-tested on a production system. The open-source implementation is being built phase by phase. Watch this repo for updates, or read the docs to understand what's coming.

---

## Get Involved

This is early. If you've had your own API-mare, if you believe AI should be affordable and private, if you want to help build something that makes AI accessible to normal people, you're in the right place.

- Read the docs
- Open an issue with your thoughts
- Share your API-mare story
- Star the repo if this resonates

---

## License

AGPL v3. Free and open source forever. Even if someone offers it as a hosted service, they must share their modifications. Like Fedi, Mastodon, and Nextcloud. Freedom technology.

---

*TESSA & COLE: Token-Efficient Self-hosted Smart Agent & Context-Optimized Learning Engine.*

*Built by builders who got sick and tired of API-mares. And decided to fix it for everyone.*
