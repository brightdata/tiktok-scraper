# For coding agents working in this repository

Read this before changing anything. Every line below was verified against the
live API, the installed SDK (`brightdata-sdk` 2.5.2) or the Bright Data CLI
0.4.0 on 2026-10-20. Where the API reference pages disagree with the live API,
the live API is what is written here.

## Auth

- The SDK reads `BRIGHTDATA_API_TOKEN` from the environment, from a `.env` file
  found by searching upward from its own install folder (so the project root,
  when the virtualenv is inside the project), or from the Bright Data CLI login.
  Setting it, or running `bdata login` once, is the one step a human must do;
  never paste a key into code.
- Always construct the client as `SyncBrightDataClient(auto_create_zones=False)`.
  Left on, the SDK tries to create Web Unlocker and SERP zones on startup. This
  repository never uses a zone, and zone creation fails on accounts without a
  payment method.

## How the API behaves

- Every SDK call is an asynchronous job: trigger, poll, return. Discovery of a
  profile's recent posts took 303 to 532 seconds, and once ran past 600. The
  SDK gives up at 240 by default and returns `success=False`,
  `status="timeout"`, no rows. Pass `timeout=900` on every discovery call.
- `search.tiktok.posts_by_profile(url, num_of_posts=N)` returns the full post
  record, 44 fields. `scrape.tiktok.posts_by_profile_fast(url)` returns the
  same posts with 36 fields, faster, but the SDK sends no `num_of_posts`, so it
  bills every post the account has. Do not use it for a capped run.
- Many URLs in one call is one job. Prefer that over a loop. For large runs use
  `posts_trigger`, `posts_status`, `posts_fetch` and keep the snapshot id.
- One credit per record. Comments cost one per comment, and the SDK cannot cap
  them: check `comment_count` on the post first. Raw HTTP with
  `limit_per_input` does cap them; 3 on a 344-comment post returned 3 rows.
- The SDK has no TikTok Shop methods. Shop is raw HTTP or the CLI. Its other
  discovery calls, `search.tiktok.profiles` and `posts_by_url`, take no limit.
- A handle that does not exist comes back as a row whose error is
  "User is banned". An empty date window comes back as a row whose error is
  `EMPTY_WINDOW`, error_code `dead_page`. Match the message, not the code.
- `account_id` is the handle. `profile_username` is the display name.
- Input field names, as the API validates them. Several API reference pages
  are wrong:
  - posts discover by URL takes `URL`, upper case. `url` is a 400:
    "This input should not contain a url field".
  - Shop discovery is `discover_by=category`, `keyword` or `shop`. The pages'
    `category_url` and `shop_url` are a 400: "Incorrect discovery collector
    id Available types: category, keyword, shop".
  - Shop by keyword takes `keyword` and optional `domain`. `num_of_products`
    is a 400.
  - Shop by store takes `url`, not `shop_url`. Shop by category takes
    `category_url` only.
  - Shop discovery is slow: 18 to 20 minutes for two products by keyword or
    by category.
  - Profiles discover by search URL requires `country`.
  - `bdata pipelines info <dataset_id> <method>` prints the live input schema.
- The schema changes without notice. Never hardcode a field list.
- The full documentation index, one `.md` page per entry:
  https://docs.brightdata.com/llms.txt

## Working here

- `pytest` runs offline and needs no token. `ruff check .` must pass.
- CI installs from the README's own commands on an empty machine. A weekly
  workflow executes every fenced block in the README against the real API, and
  the same run also does a smaller live check.
  A snippet must be complete and paste-able on its own, and its shown output
  must be real.
- The field table in the README sits between `<!-- fields:start -->` and
  `<!-- fields:end -->` and is regenerated weekly. Do not edit it by hand.
- The "last verified" badge line at the top of the README is rewritten by the
  weekly run. Do not edit it by hand.
- Keep it small. Do not add retries, deduplication, scheduling, databases,
  async examples or concurrency.
