---
name: zoromi
description: Read live Zoromi (zoromi.studio) prices and plans for short videos, vertical short dramas and illustrated books, and, with the user's own ZOROMI_API_KEY, their Gems balance, daily cap, books, video projects, job status, recent charges and a quote for filming video clips; and, with a key the user allowed to spend, make one picture or one video clip, paid in US dollars from their API balance, after the user says yes to its quoted price, or start a short drama from a script that the user then finishes on the website. Use when the user mentions Zoromi or zoromi.studio, asks what a short drama, video clip or illustrated book would cost there, asks about their own Zoromi account, books, projects, jobs or Gems, or asks for a picture, a video clip or a short drama made there. Prices, plans and what Zoromi does need no key; which other tools are open, the server's tools/list says, so call it first and ask for a key only when it lists tools that need one. Nothing is charged until the user approves the exact price.
---

# Zoromi for AI agents

Zoromi (<https://zoromi.studio>) turns a story or an idea into an illustrated book or a short
video (script, shots, voices, music and the clips), and runs a theater of short dramas.
This skill lets you answer questions about prices, plans and the user's own account, and
make one picture or one video clip for the user, by calling the Zoromi API.

Field-by-field answers for every tool: [references/tools.md](references/tools.md).

## What is open

The server opens its tools in stages. Before you offer anything, call `tools/list` and use only the tools it lists.

- **Always open, with no key:** `get_zoromi_overview`, `get_video_prices` and `get_plans_and_pricing`.
- **When `tools/list` shows them:** the tools that read an account (balance, books, projects, jobs, charges) and
  the tools that make something. A tool that is not open yet answers "This tool is not open yet."

While `tools/list` shows only the three open tools:

- Do not ask the user for an API key just to try: while only these three tools are listed, a key changes nothing.
- If the user asks about their own account, or wants a picture, a video or a drama made from here, say plainly that this is not
  open from an assistant yet, and give them the website link from `get_zoromi_overview`: they can do it there today.
- Answer price and plan questions as usual.

The sections below about keys, account tools and making a picture or a clip apply once `tools/list` shows those tools.

## Rules

1. **Never quote a price from memory.** Prices change. Call `estimate_website_clips_cost` (with a
   key), `get_video_prices` or `get_plans_and_pricing` every time and quote what they return,
   with the pricing page link they return.
2. **Spend only after the user says yes to the exact price.** Only `create_image`,
   `create_video_clip` and `edit_image` spend, and they are paid in US dollars from the account's API
   balance, which is separate from its Gems. Get a quote with `estimate_image_cost`,
   `estimate_video_clip_cost` or `estimate_image_edit_cost`,
   tell the user what it makes and what it costs (`price`), and wait for a clear yes.
   Set `confirmed_by_user` to `true` only then, never on your own.
   Send exactly the quoted arguments. If the price changed or the quote expired, quote again
   and ask again. Never start the same job twice: after `start_unknown`, check the API balance's
   latest entries in `get_my_account` (`recent_api_entries`) and `get_generation_status` first. A
   clip may start from one of the user's own pictures on Zoromi (`first_frame`, its id or its
   address on Zoromi); other web addresses are not accepted. Books and whole dramas are made on the
   website for now: give the user the link from
   `get_zoromi_overview`.
   When a tool answers `content_not_allowed`, the content policy does not allow the request:
   tell the user so, and do not reword it to get it past the policy.
3. **Keep the API key secret.** Read it from the `ZOROMI_API_KEY` environment variable. Never
   print it, echo it, write it into a file in the project, put it in a commit, or paste it into
   the chat. Send it only to `zoromi.studio`.
4. **Check the answer, not the HTTP status.** A tool can answer HTTP 200 and still report a
   problem: read `result.isError` before `result.structuredContent`. See
   [Errors](#errors).
5. **Use the links the tools return** (pricing page, account page, a book's `open_url`)
   instead of making up URLs.
6. **Write to the user in plain words.** Gems are Zoromi's credits on the website; the API
   balance is in US dollars. An answer such as "3 of 10 segments are filmed" is better than raw
   field names.

## Setup

### The API key

Skip this section while account tools are not open (see [What is open](#what-is-open)).

Prices, plans and `get_zoromi_overview` need no key. Everything about an account needs that
account owner's key, which starts with `zk_live_`. The owner makes it on their Zoromi account
page (`get_zoromi_overview` returns the link); keys come with some plans, and the page says so
if theirs does not include them yet. If the user has no key yet, answer what you can without
one and tell them where to make it.

The user sets it once, outside the project:

```bash
# macOS / Linux: add to ~/.zshrc or ~/.bashrc
export ZOROMI_API_KEY='zk_live_...'
```

```powershell
# Windows PowerShell (current user, persists)
[Environment]::SetEnvironmentVariable('ZOROMI_API_KEY', 'zk_live_...', 'User')
```

If a command shows `ZOROMI_API_KEY` is empty, ask the user to set it; do not ask them to paste
the key into the chat.

### Option A: connect as an MCP server (Claude Code and other MCP clients)

The API is an MCP server over Streamable HTTP at `https://zoromi.studio/api/v1/mcp/skill`.
In Claude Code, put this in the project's `.mcp.json`. The key stays in the environment:
Claude Code fills in `${ZOROMI_API_KEY}` when it connects, so the file holds no secret.

```json
{
  "mcpServers": {
    "zoromi": {
      "type": "http",
      "url": "https://zoromi.studio/api/v1/mcp/skill",
      "headers": { "Authorization": "Bearer ${ZOROMI_API_KEY}" }
    }
  }
}
```

The tools then appear by name (`get_video_prices` and so on) and you call them directly.

### Option B: call it with curl (any agent)

Every call is one HTTP POST of a JSON-RPC `tools/call` message. There is no session and no
handshake to do first. This shell function makes each call one line. It sends the key through
standard input (`-H @-`), so the key never appears in the command line or the process list.

```sh
# usage: zoromi <tool_name> ['<json arguments>']
zoromi() {
  tool=${1:?usage: zoromi <tool_name> [json-arguments]}
  args=${2:-}; [ -n "$args" ] || args='{}'
  body=$(printf '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"%s","arguments":%s}}' "$tool" "$args")
  url=${ZOROMI_API_URL:-https://zoromi.studio/api/v1/mcp/skill}
  if [ -n "${ZOROMI_API_KEY:-}" ]; then
    printf 'Authorization: Bearer %s\n' "$ZOROMI_API_KEY" |
      curl -sS -X POST "$url" -H 'Content-Type: application/json' \
        -H 'Accept: application/json, text/event-stream' -H @- --data-binary "$body"
  else
    curl -sS -X POST "$url" -H 'Content-Type: application/json' \
      -H 'Accept: application/json, text/event-stream' --data-binary "$body"
  fi
  echo
}

zoromi get_video_prices
zoromi list_my_books '{"limit": 5}'
```

- It needs only `curl`. `jq` is handy to read answers (`... | jq .result.structuredContent`)
  but not required.
- Windows: run it in Git Bash or WSL. In PowerShell, `curl` is another command; use
  `curl.exe` and write the body to a file (`--data-binary "@body.json"`) to avoid quoting
  problems.
- To list every tool with its argument schema: send
  `{"jsonrpc":"2.0","id":1,"method":"tools/list"}` the same way.

## Tools

| Tool | Key | Arguments | What it answers |
|---|---|---|---|
| `get_zoromi_overview` | no | none | What Zoromi makes, how to start, links (website, sign up, pricing, account) |
| `get_video_prices` | no | none | Live Gems per 5-second clip with sound, per model and resolution, and any current cut |
| `get_plans_and_pricing` | no | none | Live plans (monthly / annual price, Gems granted), pay-as-you-go, Gems packs |
| `get_my_account` | yes | none | Plan, Gems balance, today's daily cap, spent and remaining today, and the API balance with its latest entries |
| `list_my_books` | yes | `limit` 1-100, `offset`, `query` | The user's illustrated books, newest first |
| `get_book` | yes | `book_id` | One book: status and each page's status, image and opening text |
| `list_my_video_projects` | yes | `limit` 1-100, `offset` | The user's video projects, newest first, with segments counted by state |
| `get_video_project` | yes | `project_id` | One video project and its segments counted by state |
| `get_generation_status` | yes | `task_id` | One generation job: queued, running, done or failed; when done, `view_url` (the user's own page to see and download it) and for a picture made here what it was charged |
| `list_my_gems_charges` | yes | `days` 1-90 (default 30), `limit` 1-200 | Recent Gems charges and refunds, newest first, and `summary`: the totals of those days |
| `estimate_website_clips_cost` | yes | `model`, `resolution`, `seconds_per_clip`, `clips`, `aspect_ratio`, `with_sound` | Gems for filming that many clips at this account's prices, and whether the balance covers it. Spends nothing |
| `estimate_image_cost` | spending key | `prompt`, and optional `model`, `aspect_ratio`, `quality` | The price of exactly this one picture and a `quote_id` (short-lived, used once). Spends nothing |
| `create_image` | spending key | the same, plus `quote_id` and `confirmed_by_user` | Makes the quoted picture and **charges its price to the API balance**; answers a `task_id` |
| `estimate_image_edit_cost` | spending key | `pictures` (1-4 of the user's own pictures on Zoromi), `prompt`, and optional `model`, `aspect_ratio`, `quality` | The price of exactly this edit and a `quote_id` (short-lived, used once). Spends nothing |
| `edit_image` | spending key | the same, plus `quote_id` and `confirmed_by_user` | Makes the quoted edit and **charges its price to the API balance**; answers a `task_id` |
| `estimate_video_clip_cost` | spending key | `model`, `resolution`, `seconds`, `prompt`, `aspect_ratio`, `with_sound`, and optionally a picture to start from, `first_frame` | The price of exactly this one clip and a `quote_id` (short-lived, used once). Spends nothing |
| `create_video_clip` | spending key | the same, plus `quote_id` and `confirmed_by_user` | Makes the quoted clip and **charges its price to the API balance**; answers a `task_id` |
| `estimate_website_story_video_cost` | yes | `model`, `resolution`, `clips`, `seconds_per_clip`, `cards`, `aspect_ratio` | An estimate for one episode: filming plus pictures. Spends nothing |
| `start_website_story_video` | spending key | `script`, and optional `title`, `model`, `resolution`, `aspect_ratio`, `seconds_per_clip`, `target_minutes`, `keep_my_writing`, `dialogue_language` | Makes the project on the website from the script and answers `open_url` and the buttons the user presses there. Spends nothing |
| `get_website_story_video_status` | yes | `project_id` | Which step the drama is at, clips filmed, the film's link when ready, what to press next |

A "spending key" is one the account owner allowed to make pictures and clips on the account
page, with a daily limit in dollars they picked. What it makes is paid from the account's API
balance, which the owner tops up on the account page; it is separate from Gems, and neither pays
for the other. `estimate_image_cost`, `create_image`, `estimate_image_edit_cost`, `edit_image`,
`estimate_video_clip_cost` and `create_video_clip` are not listed for other keys.

With a key, `get_video_prices` answers with that account's own prices, which a plan can
change. Without a key it answers with the public prices.

## Common requests

**"What would a short drama of N clips cost?"** With a key, call `estimate_website_clips_cost` with
the model and resolution (names exactly as `get_video_prices` lists them), the seconds per
clip and the number of clips; a vertical drama is `aspect_ratio` `9:16`, the default. Report
`gems_total`, `gems_per_clip` and `enough_gems`, and say what `covers` says: the quote is for
filming only. Pictures of the characters are charged as well, and separate voiceovers or music
if the user adds them; the website shows the exact total before anything is spent. If the model rounds the
length, the answer has `seconds_note`; pass it on. Without a key, call `get_video_prices`
and estimate the filming as the total seconds times `gems_per_second` for that model and
resolution, showing the arithmetic.

**"Which plan should I take?"** Call `get_plans_and_pricing`, compare the Gems each plan grants
with the estimate above, and link the pricing page it returns.

**"How many Gems do I have?"** `get_my_account`. Also mention the daily cap and what is
left of it today when the account has one.

**"Is my video done?"** `list_my_video_projects`, then `get_video_project` with the project's
`id`. Report the segments counted by state. For one job id the user already has, use
`get_generation_status`.

**"Where is my book?"** `list_my_books` (use `query` to search by title), then
`get_book`. Give the book's `open_url`.

**"What did I spend?"** `list_my_gems_charges` with `days` for the period the user asks about (7 for a
week). Give the `summary` totals it returns - never add the entries up yourself - and link the usage page.
Pictures and clips made through the API are paid from the API balance in US dollars, not Gems: give
those from `get_my_account` (`api_balance` and `recent_api_entries`).

**"Make me a picture of ..."** With a spending key:

1. Call `estimate_image_cost` with the prompt (who or what, where, the light, the style) and, if
   the user asked for one, the frame shape as `aspect_ratio`. Leave `model` and `quality` out
   unless the user asked.
2. Tell the user what will be made and its `price`, and ask. If `enough_balance` is `false`,
   say the API balance does not cover it and give them `account_page` to add to it.
3. After a clear yes, call `create_image` with the same arguments, the `quote_id` and
   `confirmed_by_user: true`.
4. Check `get_generation_status` with the `task_id` every 5 seconds or more until `SUCCESS` or
   `FAILURE`, then give the user `view_url` (their own page on Zoromi to see and download it; the
   picture's `result_url` too when there is one) and what `charge` says was charged. When a
   picture finally fails, what it was charged goes back to the API balance.

**"Change this picture ..."** With a spending key:

1. The picture must be one of the user's own on Zoromi: its id, or the address Zoromi gave for it.
   A picture made with `create_image` works: pass the address `get_generation_status` answered
   when it was done. Other web addresses are not accepted; do not upload or fetch anything yourself.
2. Call `estimate_image_edit_cost` with the pictures (the first is the one edited, up to three more
   are references), what to change and what to keep as the `prompt`, and the shape to keep as
   `aspect_ratio`. Tell the user what will be made and its `price`, and ask.
3. After a clear yes, call `edit_image` with the same arguments, the `quote_id` and
   `confirmed_by_user: true`. Check `get_generation_status` with the `task_id` every 5 seconds or
   more until `SUCCESS` or `FAILURE`, then give the user `view_url` and what `charge` says was
   charged. When an edit finally fails, what it was charged goes back to the API balance.

**"Make me a video clip of ..."** With a spending key:

1. Call `estimate_video_clip_cost` with the seconds and the prompt (who, where, what happens, the
   camera). To start from one of the user's own pictures on Zoromi, add it as `first_frame` (its id
   or its address on Zoromi). Add the model
   and resolution (names as `get_video_prices` lists them) only when the user asked for them:
   left out, the usual ones are used and the answer says which.
2. Tell the user what will be made and its `price`, and ask. If `enough_balance` is `false`, say
   the API balance does not cover it and give them `account_page`. When the answer has
   `other_resolutions`, you may mention a cheaper one in a word. If the answer has `approve_url`, the
   job costs more than an assistant may start: send the user that link to approve it on the
   website, and go on only after they say they approved.
3. After a clear yes, call `create_video_clip` with the same arguments, the `quote_id` and
   `confirmed_by_user: true`.
4. Check `get_generation_status` with the `task_id` every 10 seconds or more until `SUCCESS` or
   `FAILURE`, then give the user `view_url` - their own page on Zoromi to watch and download the
   clip; do not wait for another link - and what `charge` says was charged. When a clip finally
   fails, what it was charged goes back to the API balance.

Without a spending key, estimate as above and give the website link.

**"Make a short drama from this script."** The user presses the steps on the website; you
start it and follow it.

1. If they want a price first, call `estimate_website_story_video_cost` and say it is an estimate.
2. With a spending key, call `start_website_story_video` with the script as the user gave it.
3. Tell the user plainly that **the next steps happen on the website, pressed by them**: give
   them `open_url` and the steps in `on_the_website`, word for word, including the button names.
   Say that every button that spends Gems shows the price first. Do not say you are making the
   drama yourself, and do not wait silently: the project does not move until they press.
4. When they ask how it is going, call `get_website_story_video_status` and pass on `next_step`. When
   `stage` is `film_ready`, give them the film link from `films`, and for another episode with the
   same people, `next_episode_url` and the button named in `next_step`.

Without a spending key, estimate with `estimate_website_story_video_cost` and send the user to the website.

**"Make me a book."** Not through this API yet. Give the website link from
`get_zoromi_overview`.

## Errors

| What you see | Meaning | What to do |
|---|---|---|
| `result.isError: true`, `structuredContent.error: "api_key_required"` | This tool needs a key | Ask the user to set `ZOROMI_API_KEY` |
| `result.isError: true`, `"not_found"` | No such id in this account | List again and use an `id` from the list |
| `result.isError: true`, `"tool_failed"` | Temporary problem | Wait a few seconds and try once more |
| `result.isError: true`, `"tool_not_open"` | This tool is switched off right now | Say so; use the website meanwhile |
| JSON-RPC `error` with code `-32602` | Bad arguments | Fix them; `tools/list` shows each tool's schema |
| HTTP 401 with `code` such as `invalid_api_key`, `api_key_revoked`, `api_key_expired` | The key does not work | Ask the user to check it or make a new one on their account page |
| HTTP 429 with a `Retry-After` header | Too many calls | Wait that many seconds; do not loop |
| HTTP 404 with `code: "mcp_off"`, or HTTP 503 `api_keys_not_open` | The API is not open right now | Tell the user; use the website meanwhile |
| HTTP 413 `body_too_large` | The request is too big | Send shorter arguments |
| `"spending_not_allowed"` | This key may not make pictures or clips | The account owner allows it on the account page |
| `"not_confirmed"` | `confirmed_by_user` was not `true` | Ask the user; call again only after their yes |
| `"price_changed"`, `"quote_expired"`, `"quote_used"`, `"quote_other_job"` | The quote no longer fits | Quote again and ask the user again |
| `"approval_needed"` | Above the approval line | Send the user `details.approve_url`; wait for their approval |
| `"daily_limit"` | The key's daily limit would be passed | Tell the user; the owner can change it on the account page |
| `"not_enough_balance"` | The API balance does not cover it; nothing was charged | Tell the user to add to the API balance on the account page |
| `"balance_on_hold"` | The API balance is on hold during a payment dispute; nothing was charged | Tell the user; adding to it does not help until the dispute ends |
| `"already_started"` | That quote's job was started before; nothing more was charged | Check `get_generation_status` or `list_my_video_projects`; do not quote and make it again blindly |
| `"not_offered"` | That model, resolution or quality is not sold through the API | Offer another one, or the website |
| `"start_unknown"` | It is not known whether the clip started | Check the API balance's latest entries in `get_my_account` (`recent_api_entries`); never run it again blindly |
| `"not_a_drama"` | That project was not made from a script | Use `get_video_project` for it instead |

Call one tool at a time and reuse answers you already have; the price list does not change
from one minute to the next.
