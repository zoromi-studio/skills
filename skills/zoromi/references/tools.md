# Zoromi API: tools and answers

Endpoint: `POST https://zoromi.studio/api/v1/mcp/skill`, JSON-RPC 2.0, MCP over Streamable HTTP.
Headers: `Content-Type: application/json`, `Accept: application/json, text/event-stream`, and
for account tools `Authorization: Bearer <key>` with the key from `ZOROMI_API_KEY`.

A successful call answers:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [{ "type": "text", "text": "<the same data as JSON text>" }],
    "structuredContent": { "...": "the data" },
    "isError": false
  }
}
```

Read `result.structuredContent`. When `result.isError` is `true`, `structuredContent` is
`{"error": "<code>", "message": "<words for the user>", "details": {...}}` instead.

Answers may carry more fields than listed here. Ignore the ones you do not need, and never
treat a missing field as zero.

## get_zoromi_overview

No key, no arguments.

- `service`, `what_zoromi_does` (list of sentences), `how_to_start` (list of sentences)
- `links`: `website`, `sign_up`, `pricing`, `account`, `instructions_for_ai`
- `tools`: each tool's `name`, `needs_api_key`, `summary`
- `spends_gems`: always `false` - pictures and clips made here are paid from the API balance in US
  dollars; `spends_api_balance` says whether making them is open to this connection

## get_video_prices

No key needed, no arguments. With a key, the prices are that account's own.

- `unit`: what one row's `gems` buys (a 5-second clip with sound)
- `rows`: one per model and resolution
  - `model`: the name the website shows for the model
  - `resolution`: for example `480p`, `720p`, `1080p`
  - `seconds`: the clip length the row is priced for
  - `gems`: Gems for that clip
  - `gems_per_second`
  - `was_gems`, `cut_pct`: present only while a price cut is shown
- `cut`: `{"min_pct", "max_pct"}` across the rows, or `null` when nothing is cut
- `clip_seconds`, `with_sound`, `note`
- `pricing_page`: link for the user

## get_plans_and_pricing

No key, no arguments.

- `plans`: each with `plan_id`, `monthly_usd`, `annual_usd`, `annual_available`,
  `gems_per_month`, `gems_rollover`, `features`, and `first_month`
  (`{"pct", "usd", "who"}`) when a first-month offer is listed
- `pay_as_you_go`: what an account without a plan gets
- `gems_packs`: each with `pack_id`, `usd`, `gems`
- `note`, `pricing_page`

## get_my_account

Key needed, no arguments.

- `account_id`, `plan`, `gems` (balance)
- `daily_gems_cap`, `daily_gems_spent_today`, `daily_gems_remaining_today`: the cap the
  account owner set for themselves; empty when they set none
- `api_key`: `name`, `key_prefix`, `scopes`, `daily_limit_usd_cents` (what the key may spend a day
  from the API balance, in US cents) of the key used for this call (never the key itself)
- `api_balance` (dollars and cents, as text), `api_balance_cents`: the API balance, in US dollars, separate from
  Gems; pictures and clips made here are paid from it
- `recent_api_entries`: its newest entries, each `what` (added, charged, held until the job ends, given
  back, refunded), `amount` and `at`. Use it when the user asks what they have or what something took.
  Both are left out when the balance cannot be read just now.
- `api_balance_on_hold`, `api_balance_on_hold_note`: present only while a payment dispute is open: that
  much is on hold, and nothing can be made through the API until the dispute ends
- `account_page`: link for the user

## list_my_books

Key needed. Arguments, all optional: `limit` (1-100, default 20), `offset` (default 0),
`query` (words in titles and text, up to 200 characters).

- `books`: newest first, each with `id`, `title`, `status`, `page_count`, `created_at`,
  `updated_at`, and when known `progress`, `has_cover`, `pdf_ready`, `language`, `open_url`
- `total`, `limit`, `offset`, `query`

Page through with `offset` while `offset + limit < total`.

## get_book

Key needed. Argument: `book_id` (from `list_my_books`).

- The book's fields as above, plus `pages`: each with `page_no`, `status`, `has_image`, and
  `text` (the first part of the page's text); `pages_with_picture`. The pictures themselves are
  seen on the website, through `open_url`
- `open_url`: link to the book on the website

## list_my_video_projects

Key needed. Arguments, all optional: `limit` (1-100, default 20), `offset` (default 0).

- `projects`: newest first, each with `id`, `title` or `name`, `status`, `created_at`,
  `updated_at`, `aspect_ratio`, `episode` when set, `segment_count`, and
  `segments_by_status` (for example `{"filmed": 3, "not_filmed": 7}`)
- `total`, `limit`, `offset`

## get_video_project

Key needed. Argument: `project_id` (from `list_my_video_projects`).

- The same fields as one entry of `list_my_video_projects`

## get_generation_status

Key needed. Argument: `task_id`, a job id the website or an earlier answer gave.

- `state`: upper case, such as `PENDING`, `STARTED`, `SUCCESS`, `FAILURE`. Stop checking at
  `SUCCESS` or `FAILURE`.
- `done`, `failed`, `has_result`: whether it finished, failed, and made something
- `result_url`: when done, a link to what it made that only the account owner can open; empty
  when it cannot be given here
- `view_url`: when done, the account owner's own page for it on Zoromi (signed in), with a
  download button. When `result_url` is empty, give the user `view_url` - `result_note` says so
- `error`: present when it failed, in words for the user
- `charge`: for a picture or clip made with `create_image` or `create_video_clip`, what it
  cost from the API balance - `charged` (in dollars and cents, as text), `charged_cents`, `state`
  (`held` until the job ends, `charged`, or `reversed` when it failed and the money went back)
  and `note`

When checking a running job again, wait at least 10 seconds between calls; a video clip
usually takes minutes. If the answer has `known: false`, or the job stays `PENDING` far longer
than such a job takes, the id is probably wrong: check it against the website or
`list_my_video_projects` instead of checking again and again.

## list_my_gems_charges

Key needed. Arguments, optional: `days` (1-90, default 30; 7 for "this week") and `limit` (1-200,
default 50).

- `summary`: the totals of those days from the account's charge records - `days`, `from`, `until`,
  `charged_gems`, `refunded_gems`, `net_gems`, `charges`. Give these numbers for any total; never add
  the entries up yourself. When they cannot be read, `summary_note` says so instead
- `days`
- `entries`: newest first, each with `id`, `created_at`, `kind`, `amount`, `charge_type`,
  `status`, and when known `title`, `description`, `book_id`, `project_id`, and `has_result`
  (it made something, seen on the usage page)
- `unit`: `Gems`
- `usage_page`: link for the user

## estimate_website_clips_cost

Key needed. Spends nothing, holds nothing and returns no id: the website prices the job again
and asks before it charges.

Arguments:

- `model`: optional; a model name exactly as `get_video_prices` lists it. Leave it out unless
  the user asked for a model: the usual one is used and the answer says which
- `resolution`: optional; one of the resolutions that model comes in. Left out, the usual one is
  used and the answer lists what the others cost
- `seconds_per_clip` (required): 1-30; a model that makes only some lengths rounds it
- `clips`: 1-200, default 1
- `aspect_ratio`: `9:16` (default, a vertical short drama), `16:9`, `1:1`, `4:3`, `3:4`, `21:9`
- `with_sound`: default `true`

Answer:

- `model`, `resolution`, `aspect_ratio`, `with_sound`, `clips`
- `seconds_per_clip`: the length each clip is priced at; when it differs from what was asked,
  `seconds_asked` and `seconds_note` say so
- `gems_per_clip`, `gems_total`: at this account's own prices
- `other_resolutions`, when the resolution was left out: each other resolution with its `gems`
  for the same job and a `note` saying how to ask for it. Offer it when the user wants it cheaper
- `your_gems`, `enough_gems`, and `daily_gems_remaining_today` when the owner set a daily cap
  (`enough_gems` counts that cap); absent when the balance could not be read
- `covers`: what the quote includes (filming only), `note`, `spends_gems: false`
- `pricing_page`: link for the user

Refusals inside the answer: `unsupported_resolution` (the model does not come in that
resolution; the message lists the ones it does), `cannot_price` (try again shortly). Bad or
missing arguments are a JSON-RPC `-32602` error.

## estimate_image_cost

Needs a key the account owner allowed to spend. Spends nothing. Pictures and clips made through
the API are paid in US dollars from the account's API balance, never in Gems.

Arguments:

- `prompt` (required): what the picture shows, up to 4000 characters
- `model`: optional; leave it out for the usual one (the tool lists the names it takes)
- `aspect_ratio`: `1:1` (default), `16:9`, `9:16`, `4:3`, `3:4`
- `quality`: `high` (default) or `basic`

One picture, made from the words alone.

Answer:

- `quote_id`: used once, by this key, for exactly these arguments; `expires_at`
- `price` (in dollars and cents, as text) and `cents`: the price of the picture, rounded up to the
  cent; `paid_from`
- `model`, `aspect_ratio`, `quality`, `pictures: 1`
- `api_balance`, `enough_balance`, `left_today_for_this_key`, `within_daily_limit`; absent when
  they could not be read. When the balance or the key's limit does not cover it, `next` says so
  and `account_page` is the link for the user
- `needs_approval_on_website`; when `true`, `approve_url`: the account owner approves the job
  on the website before it can run
- `next`, `pricing_page`, `charged: false`

Refusals inside the answer: `content_not_allowed` (the content policy does not allow the
request; nothing was charged), `content_check_unavailable` (try again shortly), `unknown_model`,
`unsupported_aspect_ratio`, `unsupported_quality`, `not_offered` (not sold through the API),
`cannot_price`, `spending_not_open`.

## create_image

Needs a key allowed to spend. **Charges the quoted price to the API balance.**

Arguments: exactly the arguments of the quote, plus `quote_id` and `confirmed_by_user`
(`true` only after the user said yes to the price).

The server checks the request against the content policy and prices the picture again; if
anything differs from the quote, it refuses and charges nothing. It also counts the picture
against the key's daily limit.

Answer: `task_id` (check it with `get_generation_status`, which then says what was charged), `price`,
`cents`, `paid_from`, `status: queued`, `next`, `if_it_fails`, `usage_page`. When the picture
finally fails, what it was charged goes back to the API balance.

## estimate_image_edit_cost

Needs a key the account owner allowed to spend. Spends nothing. Pictures made or edited through
the API are paid in US dollars from the account's API balance, never in Gems.

Arguments:

- `pictures` (required): 1 to 4 of the user's own pictures on Zoromi, each its id or its address
  on Zoromi (the address `get_generation_status` gives for a picture made here works). The first
  is the one edited; the others are references. Other web addresses are not accepted
- `prompt` (required): what to change, and what to keep, up to 4000 characters
- `model`: optional; leave it out for the usual one (the tool lists the names it takes)
- `aspect_ratio`: `1:1` (default), `16:9`, `9:16`, `4:3`, `3:4`; give the original's shape to
  keep it
- `quality`: `high` (default) or `basic`

Answer:

- `quote_id`: used once, by this key, for exactly these arguments; `expires_at`
- `price` (in dollars and cents, as text) and `cents`: the price of the edit, rounded up to the
  cent; `paid_from`
- `model`, `aspect_ratio`, `quality`, `pictures_in`, `pictures: 1`
- `api_balance`, `enough_balance`, `left_today_for_this_key`, `within_daily_limit`; absent when
  they could not be read. When the balance or the key's limit does not cover it, `next` says so
  and `account_page` is the link for the user
- `needs_approval_on_website`; when `true`, `approve_url`: the account owner approves the job
  on the website before it can run
- `next`, `pricing_page`, `charged: false`

Refusals inside the answer: `not_found` (not one of this account's pictures),
`address_not_accepted` (not an address on Zoromi), `too_many_pictures`, `pictures_missing`,
`content_not_allowed` (the content policy does not allow the request; nothing was charged),
`content_check_unavailable` (try again shortly), `unknown_model`, `unsupported_aspect_ratio`,
`unsupported_quality`, `not_offered` (not sold through the API), `cannot_price`,
`spending_not_open`, `edit_not_open`.

## edit_image

Needs a key allowed to spend. **Charges the quoted price to the API balance.**

Arguments: exactly the arguments of the quote, plus `quote_id` and `confirmed_by_user`
(`true` only after the user said yes to the price).

The server checks the request against the content policy, checks again that every picture is
still the user's own, keeps the user's word on the pictures with this job, and prices the edit
again; if anything differs from the quote, it refuses and charges nothing. It also counts the
edit against the key's daily limit.

Answer: `task_id` (check it with `get_generation_status`, which then gives `view_url` and says what
was charged), `price`, `cents`, `paid_from`, `status: queued`, `next`, `if_it_fails`,
`usage_page`. When the edit finally fails, what it was charged goes back to the API balance.

## estimate_video_clip_cost

Needs a key the account owner allowed to spend. Spends nothing.

Arguments:

- `model`, `resolution`: optional, as `get_video_prices` lists them; left out, the usual ones
  are used and the answer says which
- `seconds` (required): 1-30; a model that makes only some lengths rounds it
- `prompt` (required): what the clip shows, up to 4000 characters
- `aspect_ratio`: `9:16` (default), `16:9`, `1:1`, `4:3`, `3:4`, `21:9`
- `with_sound`: default `true`
- `first_frame`: optional, one of the user's own pictures on Zoromi the clip starts from - its id,
  or its address on Zoromi. Other web addresses are not accepted

Answer:

- `quote_id`: used once, by this key, for exactly these arguments; `expires_at`
- `price` (in dollars and cents, as text) and `cents`: the price of the clip, rounded up to the
  cent; `paid_from`
- `model`, `resolution`, `aspect_ratio`, `seconds`, `with_sound`, `first_frame`; `seconds_note`
  when the model rounds the length
- `other_resolutions`, when the resolution was left out: each other resolution sold through the
  API with its `price`
- `api_balance`, `enough_balance`, `left_today_for_this_key`, `within_daily_limit`, as for a
  picture
- `needs_approval_on_website`; when `true`, `approve_url`: the account owner approves the job
  on the website before it can run
- `next`, `pricing_page`, `charged: false`

Refusals inside the answer also include `content_not_allowed` (the content policy does not
allow the request; nothing was charged), `content_check_unavailable` (try again shortly),
`not_offered` (that model and resolution is not sold through the API), and
`not_found` (the picture is not one of this account's).

## create_video_clip

Needs a key allowed to spend. **Charges the quoted price to the API balance.**

Arguments: exactly the arguments of the quote, plus `quote_id` and `confirmed_by_user`
(`true` only after the user said yes to the price).

The server checks the picture to start from again and prices the clip again; if anything
differs from the quote, it refuses and charges nothing. It also counts the clip against the
key's daily limit.

Answer: `task_id` (check it with `get_generation_status`, which then says what was charged), `price`,
`cents`, `paid_from`, `status: queued`, `next`, `if_it_fails`, `usage_page`. When the clip
finally fails, what it was charged goes back to the API balance.

## estimate_website_story_video_cost

Key needed. Spends nothing.

Arguments: `clips` (1-200) and `seconds_per_clip` (1-30), both required; optional `model`,
`resolution` (as `get_video_prices` lists them; the usual ones otherwise), `cards` (0-60,
default 8: about how many people, places and things need a picture), `aspect_ratio` (default
`9:16`).

Answer: `gems_per_clip`, `filming_gems`, `gems_per_picture`, `pictures_gems`,
`estimated_total_gems`, `planning` (whether planning the script costs Gems), `your_gems`,
`enough_gems`, `note`, `pricing_page`. How many clips and pictures an episode really needs is
decided when the script is planned on the website.

## start_website_story_video

Needs a key allowed to spend. Spends nothing: it only makes the project. The steps the user
presses on the website are paid in Gems there, each showing its price first.

Arguments: `script` (required, up to 100,000 characters); optional `title` (the first step names
the project from the script otherwise), `model`, `resolution` (the usual ones otherwise), `aspect_ratio`
(default `9:16`), `seconds_per_clip` (one fixed length; otherwise chosen per clip),
`target_minutes`, `keep_my_writing` (`true` films the script as written), `dialogue_language`.

Answer: `project_id`, `open_url`, `on_the_website` (the steps the user presses, with the button
names), `planning`, `tell_the_user`, `spends_gems: false`. Nothing is run, drawn or filmed until
the user presses the buttons on the website.

## get_website_story_video_status

Key needed. Argument: `project_id`.

Answer: `stage` - `script` (press stage 1), `script_developed` (press stage 2, then stage 3),
`filming`, `clips_filmed` (press stage 5), `film_ready` - with `clips`, `clips_filmed`,
`clips_in_progress`, `films` (links to the finished film), `next_step` (what the user presses
next, in words to pass on), `open_url`, and `next_episode_url` once the film is ready.

## Error codes inside a tool answer

| `error` | Meaning |
|---|---|
| `api_key_required` | The tool needs a key and none was sent |
| `api_keys_not_open` | Keys are not open on the server yet |
| `not_found` | No such id in this account |
| `forbidden` | This account may not read that |
| `tool_failed` | A temporary problem; try once more after a few seconds |
| `tool_not_open` | This tool is switched off right now |
| `unsupported_resolution`, `cannot_price` | See estimate_website_clips_cost |
| `spending_not_allowed` | The key may not make pictures or clips; the account owner allows it on the account page |
| `spending_not_open` | Making pictures and clips through the API is not open yet |
| `not_offered` | That model, resolution or quality is not sold through the API |
| `address_not_accepted` | Only a picture's id or its address on Zoromi is accepted; other web addresses are not fetched |
| `pictures_missing`, `too_many_pictures` | An edit names 1 to 4 of the user's own pictures |
| `edit_not_open` | Editing pictures through the API is not open yet |
| `first_frame_not_open` | A clip cannot start from a picture through the API yet |
| `not_confirmed` | `confirmed_by_user` was not `true` |
| `quote_not_found`, `quote_other_key`, `quote_other_job`, `quote_used`, `quote_expired`, `quote_declined`, `price_changed` | The quote cannot be used; quote again and ask the user again |
| `approval_needed` | The owner approves it on the website first; `details.approve_url` |
| `daily_limit` | The key's daily limit would be passed |
| `not_enough_balance` | The API balance does not cover it; nothing was charged |
| `balance_on_hold` | The API balance is on hold while a payment dispute is open; nothing was charged |
| `already_started` | That quote's job was started before; nothing more was charged - check the job before making it again |
| `not_started` | It did not start; nothing was charged, or what was charged already went back |
| `start_unknown` | It is not known whether it started; check usage and projects before anything else |
| `script_missing` | No script was sent |
| `not_a_drama` | The project was not made from a script |

## HTTP refusals (before any tool runs)

The body is `{"code": "...", "message": "..."}`.

| Status | `code` | Meaning |
|---|---|---|
| 401 | `bad_authorization` | The header is not `Authorization: Bearer zk_live_...` |
| 401 | `invalid_api_key`, `api_key_revoked`, `api_key_expired` | The key does not work; the owner makes a new one on the account page |
| 401 | `account_unavailable` | The account behind the key cannot use the API |
| 403 | `origin_refused` | The request came from a browser page that may not call the API |
| 404 | `mcp_off` | The API is not open right now |
| 404 | `unknown_platform` | The address is wrong; use the endpoint above |
| 413 | `body_too_large` | The request is too big |
| 429 | `rate_limited` | Too many calls; wait the seconds in `Retry-After` |
| 503 | `api_keys_not_open` | Keys are not open on the server yet |
