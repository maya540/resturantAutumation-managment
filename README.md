# Restaurant Ordering Bot (n8n + Telegram)

This is the first automation I built in n8n, and honestly the project where things started to click for me. It's a Telegram bot for a made-up restaurant called KD. Customers can ask about the menu, place an order, check on it, or cancel it, all by just chatting. No forms, no buttons.

I come from a backend background (PHP/Laravel), so I wanted to see how much of a "real" customer-support flow I could put together without writing a server from scratch. It turned out to be quite a lot.

## What it does

- Answers questions about the restaurant (menu, prices, location, opening hours) using a knowledge base, not the model's imagination
- Takes new orders: collects name, phone, address and the items, then reads the order back before saving it
- Looks up an existing order by its ID
- Cancels an order when the customer asks
- Replies in whatever language the customer wrote in, so Arabic and English both work
- Handles small talk ("hi", "good morning") without getting confused

## How it works

The flow starts at a Telegram Trigger. Every incoming message first goes through a small LLM chain whose only job is to decide who should handle it. It works like a receptionist sending the customer to the right employee, and it answers with one of three labels in JSON: `Questions Expert`, `Orders Expert` or `Generalist`.

A Switch node reads that label and sends the message down the matching path. Each path just sets a different system prompt, and then everything lands in one shared AI Agent. So there's a single agent, but it behaves differently depending on who the router picked. Anything the router can't classify falls back to the general path, so the bot never goes silent.

The agent has a few tools it can call:

- **Knowledge base**: a Supabase vector store (OpenAI embeddings) holding the menu and restaurant info. This is the RAG part.
- **Google Sheets, three tools**: one to add a new order, one to read an order by ID, one to update an order's status to cancelled
- **Code tool**: generates a random 4-digit order ID, so the model never makes one up on its own
- **Simple Memory**: keeps the conversation per Telegram chat ID, so the bot remembers what you ordered two messages ago

The final answer goes back to the customer through a Telegram "Send a text message" node.

```
Telegram Trigger
      │
      ▼
 Router (LLM chain) ──► Switch ──┬─► Questions prompt ─┐
                                 ├─► General prompt  ──┼─► AI Agent ─► Telegram reply
                                 └─► Orders prompt   ──┘      │
                                                              ├─ Knowledge base (Supabase)
                                                              ├─ Sheets: add / read / cancel
                                                              ├─ Code tool (order ID)
                                                              └─ Memory (per chat)
```

## Things I cared about while building it

**The bot shouldn't invent facts.** Early on it happily made up addresses and prices. The prompts now force it to check the knowledge base before answering anything about the restaurant, and to say "I don't have that information" when the answer isn't there.

**Orders need complete data.** The Orders prompt tells the agent not to save anything until it has the name, phone, address and items, and until the customer has confirmed the full order.

**Cancelling should be deliberate.** The bot asks for the order ID and confirms before it changes anything in the sheet.

## Setup

You'll need accounts or credentials for:

1. Telegram (a bot token from BotFather)
2. OpenRouter (for the chat models)
3. OpenAI (only for embeddings)
4. Supabase (a `documents` table set up as a vector store, with your menu loaded into it)
5. Google Sheets (a sheet with these columns: `Time`, `Item`, `Price`, `Order_id`, `Name`, `Phone_Number`, `Adress`, `Statue`)

Then:

1. Import `resturantMangement2.json` into n8n
2. Re-connect each credential, since they don't travel with the export
3. Point the three Google Sheets nodes at your own spreadsheet
4. Load your menu and restaurant info into the Supabase table
5. Activate the workflow and message your bot

## Known rough edges

I'd rather be upfront about these than pretend it's perfect:

- Order IDs are random 4-digit numbers, so two orders could get the same ID eventually. It's fine for a demo, but a real version should check for duplicates or use a proper sequence.
- The column names `Adress` and `Statue` are misspelled in my sheet, and the status values (`inkitchen`, `canceld`) are too. The workflow is consistent with itself, but I'd clean them up in a next version.
- Memory is a simple buffer, so it forgets older parts of a long conversation.
- The model choice is whatever was free or cheap on OpenRouter while I was testing. Swapping it is a one-node change, but answer quality will vary.
- There's no human handoff yet. If the bot gets stuck, nobody gets notified.

## What I'd add next

- A notification to the kitchen when a new order comes in
- Order status updates sent back to the customer
- Duplicate-ID protection
- A proper fallback to a real person

## What I learned

Mostly that the prompts matter as much as the nodes. Splitting the job between a router and specialists made the bot far more predictable than one giant prompt, and that was the biggest lesson from this build. I also got comfortable with tool-calling agents, vector stores and wiring Google Sheets in as a lightweight database.

If you have feedback or ideas, I'd love to hear them. I'm still learning and building more of these.
