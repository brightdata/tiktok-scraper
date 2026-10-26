# tiktok-scraper

[![Live check](https://github.com/brightdata/tiktok-scraper/actions/workflows/live.yml/badge.svg)](https://github.com/brightdata/tiktok-scraper/actions/workflows/live.yml)
[![last check failed](https://img.shields.io/badge/last%20check%20failed-26%20Oct%202026-red)](https://github.com/brightdata/tiktok-scraper/actions/workflows/live.yml) <!-- verified: rewritten by the daily run -->

[Quickstart](#quickstart) · [Command](#or-run-it-as-a-command) · [Endpoints](#the-rest-of-the-api) · [Data](#the-data) · [Errors](#when-it-fails) · [Coding agents](#coding-agents) · [Docs](https://docs.brightdata.com/products/scrapers/tiktok/introduction) · [Support](#support)

TikTok profiles, posts, comments and TikTok Shop products as JSON, in Python.
No TikTok login, no browser. Built on the
[Bright Data TikTok Scraper API](https://brightdata.com/products/web-scraper/tiktok?utm_source=github).

Uses the [Bright Data Python SDK](https://github.com/brightdata/sdk-python).
Full API docs:
[TikTok Scraper API](https://docs.brightdata.com/products/scrapers/tiktok/introduction).

Also here: a one-command CLI for recent posts, and the
[Bright Data CLI](#coding-agents), which needs no Python at all.

## Quickstart

Python 3.10 or newer.

```bash
pip install brightdata-sdk
export BRIGHTDATA_API_TOKEN=YOUR_API_KEY
```

Get a token from the
[Bright Data control panel](https://brightdata.com/cp/setting/users). A `.env`
file in the project root works too, as long as the virtualenv is inside the
project.

Or skip the token. Run `npx -p @brightdata/cli bdata login` once: it opens a
browser, and from then on the SDK finds the stored credentials on its own, for
you and for any coding agent working in that terminal. Agents cannot click
through the login, so do it yourself first. The same CLI scrapes TikTok
directly; see [Coding agents](#coding-agents).

No account yet? [Create one](https://brightdata.com/cp/start); new accounts get
[5,000 free credits a month](https://docs.brightdata.com/general/account/billing-and-pricing/free-tier).

```python
from brightdata import SyncBrightDataClient

with SyncBrightDataClient(auto_create_zones=False) as client:
    result = client.search.tiktok.posts_by_profile(
        "https://www.tiktok.com/@tiktok", num_of_posts=5, timeout=900
    )
    if not result.success:
        raise SystemExit(f"{result.status}: {result.error}")
    for post in result.data:
        print(post["digg_count"], post["comment_count"], post["url"])
```

```
12300 3222 https://www.tiktok.com/@tiktok/video/7692114317151423775
12000 5091 https://www.tiktok.com/@tiktok/video/7694797676306615583
9428 1391 https://www.tiktok.com/@tiktok/video/7691013160425327903
14800 3613 https://www.tiktok.com/@tiktok/video/7693184538704416031
11200 1821 https://www.tiktok.com/@tiktok/video/7691450773619625247
```

Expect five to ten minutes: the API runs a discovery job and the SDK waits for
it. One [credit](https://brightdata.com/pricing/web-scraper) per post.
`digg_count` is TikTok's name for likes.

Three things in that snippet are not optional.

`auto_create_zones=False` stops the SDK creating zones on startup. The zones
are for Web Unlocker and SERP, two other Bright Data products this scraper
never touches, and zone creation fails on accounts without a payment method
([sdk-python#57](https://github.com/brightdata/sdk-python/issues/57)).

`timeout=900`. The SDK gives up at 240 seconds by default. This call took 303
to 532 seconds on 20 Oct 2026, and once ran past 600. At 240 it returns
`success=False` and no data.

`num_of_posts` caps the run, and the bill. Without it discovery collects every
post the account has.

## Or run it as a command

The command in this repo does the same for several accounts and writes one
JSON file.

```bash
pip install git+https://github.com/brightdata/tiktok-scraper
tiktok-scraper tiktok khaby.lame
```

```
Fetching up to 5 recent posts per account, for: tiktok, khaby.lame
Usually two to six minutes each. One credit per post, 5,000 free per month.
asking  @tiktok...
got     @tiktok: 5 posts
asking  @khaby.lame...
got     @khaby.lame: 5 posts

Saved 10 posts as JSON to tiktok.json (44 fields per post)
```

In a terminal the `asking` lines are replaced by a spinner and a running
clock, updating in place, so you can see it is working and how long it has
been going.

```
--limit N    posts per account, default 5, minimum 1
--out PATH   output file, default tiktok.json
```

`python -m tiktok_scraper` works too.

Import it instead of running it, for `ok`, `note` and `error` per account
instead of raw rows. `scrape` never raises for one bad account; check `ok`
before reading `posts`:

```python
from tiktok_scraper import scrape

for outcome in scrape(["tiktok", "zz_not_a_real_account_zz"], limit=1):
    if outcome.ok:
        print(f"{outcome.handle}: {len(outcome.posts)} posts")
    else:
        print(f"{outcome.handle} failed: {outcome.error}")
```

```
tiktok: 1 posts
zz_not_a_real_account_zz failed: User is banned
```

An account that does not exist comes back as "User is banned". An account
with nothing in the period searched is a success with no posts. The reason
lands in `note`, not `error`.

## The rest of the API

The Bright Data TikTok Scraper API has 12 endpoints, each with its own page in
the [API reference](https://docs.brightdata.com/api-reference/scrapers/social-media-apis/tiktok-profiles-collect-by-url).
Every snippet below is complete and needs only `brightdata-sdk`, or nothing at
all for raw HTTP: paste it as is. Every one of them runs in Actions each
Monday, a smaller check runs every other day, and the badge at the top is the
latest result.

| you have | want | how |
| --- | --- | --- |
| a profile URL | recent posts, 44 fields | `client.search.tiktok.posts_by_profile(url, num_of_posts=5, timeout=900)` |
| a profile URL | recent posts, 36 fields, faster | raw HTTP, Posts by Profile Fast API. The SDK's `posts_by_profile_fast` sends no `num_of_posts`, so it bills every post |
| a profile URL | the profile | `client.scrape.tiktok.profiles(url)` |
| a TikTok search URL | matching profiles | raw HTTP, Profiles `discover_by=search_url` |
| post URLs | those posts | `client.scrape.tiktok.posts([url, url])` |
| a keyword | matching posts | `client.search.tiktok.posts_by_keyword("street food", num_of_posts=2, timeout=900)` |
| a tag, music or explore URL | posts on that page | raw HTTP, Posts `discover_by=url` |
| a post URL | its comments | `client.scrape.tiktok.comments(url)`, one credit per comment |
| a TikTok Shop product URL | the product | raw HTTP, Shop collect |
| a Shop category URL, keyword or store URL | products | raw HTTP, Shop `discover_by=category`, `keyword` or `shop` |

Every one of these is an asynchronous job. The SDK triggers it, polls, and
returns when it is ready. Error rows, like the "User is banned" one above,
appear because the SDK asks for them with `include_errors=true`. The API
default is off.

`search` calls return a list in `result.data`. `scrape` calls with one URL
return one record as a dict, and several as a list.

### Two accounts, one job

A list of URLs is one job, not one per account.

```python
from brightdata import SyncBrightDataClient

with SyncBrightDataClient(auto_create_zones=False) as client:
    result = client.search.tiktok.posts_by_profile(
        ["https://www.tiktok.com/@tiktok", "https://www.tiktok.com/@khaby.lame"],
        num_of_posts=1,
        timeout=900,
    )
    for post in result.data:
        print(post["account_id"], post["url"])
```

```
khaby.lame https://www.tiktok.com/@khaby.lame/video/7697307919943060767
tiktok https://www.tiktok.com/@tiktok/video/7694797676306615583
```

`account_id` is the handle. `profile_username` is the display name, "TikTok"
or "Khabane lame", not the handle.

### Trigger now, fetch later

For anything bigger than a few accounts, do not block a process for an hour.
Trigger, keep the snapshot id, fetch when ready. Snapshots stay downloadable
for 30 days.

```python
import time

from brightdata import SyncBrightDataClient

with SyncBrightDataClient(auto_create_zones=False) as client:
    job = client.scrape.tiktok.posts_trigger("https://www.tiktok.com/@tiktok/video/7680721699171601694")
    print("snapshot:", job.snapshot_id)
    while (status := client.scrape.tiktok.posts_status(job.snapshot_id)) not in ("ready", "failed"):
        time.sleep(5)
    print("status:", status)
    record = client.scrape.tiktok.posts_fetch(job.snapshot_id)[0]
    print("fetched:", record["url"], "likes:", record["digg_count"])
```

```
snapshot: sd_mvgcsmg815mdlpv6kj
status: ready
fetched: https://www.tiktok.com/@tiktok/video/7680721699171601694 likes: 22900
```

### A date window

```python
from brightdata import SyncBrightDataClient

with SyncBrightDataClient(auto_create_zones=False) as client:
    result = client.search.tiktok.posts_by_profile(
        "https://www.tiktok.com/@tiktok",
        num_of_posts=3,
        start_date="09-01-2026",  # MM-DD-YYYY
        end_date="09-30-2026",
        timeout=900,
    )
    for post in result.data:
        print(post["create_time"], post["play_count"], post["url"])
```

```
2026-09-25T20:13:19.000Z 449600 https://www.tiktok.com/@tiktok/video/7689568478570843422
2026-09-29T17:39:33.000Z 264700 https://www.tiktok.com/@tiktok/video/7691013160425327903
2026-09-30T21:57:30.000Z 346500 https://www.tiktok.com/@tiktok/video/7691450773619625247
```

A window with nothing in it comes back as one row whose `error` is "There are
no public posts in the profile for the specified period". The command treats
that row as zero posts, not a failure.

### A profile, by URL

```python
from brightdata import SyncBrightDataClient

with SyncBrightDataClient(auto_create_zones=False) as client:
    profile = client.scrape.tiktok.profiles("https://www.tiktok.com/@tiktok").data
    print(profile["account_id"], "followers:", profile["followers"], "videos:", profile["videos_count"])
```

```
tiktok followers: 96400000 videos: 1510
```

### Posts by keyword

```python
from brightdata import SyncBrightDataClient

with SyncBrightDataClient(auto_create_zones=False) as client:
    result = client.search.tiktok.posts_by_keyword("street food", num_of_posts=2, timeout=900)
    for post in result.data:
        print(post["account_id"], post["play_count"], post["url"])
```

```
hughabroad 6600000 https://www.tiktok.com/@hughabroad/video/7571414421311065346
biteswithlily 11500000 https://www.tiktok.com/@biteswithlily/video/7598108209903127829
```

### Comments on a post

One credit per comment, and `comments()` has no way to ask for fewer. It also
gives up at 240 seconds by default, and a two-comment post once took 243, so
pass `timeout=900`. Check
`comment_count` on the post first: the TikTok post used elsewhere in this
README has more than 5,000. The post below has two. To cap a big one, use raw
HTTP with `limit_per_input`; 3 on a 344-comment post returned 3 rows.

```python
from brightdata import SyncBrightDataClient

with SyncBrightDataClient(auto_create_zones=False) as client:
    url = "https://www.tiktok.com/@cooking__alone/video/7565981443512339742"
    comments = client.scrape.tiktok.comments(url, timeout=900).data
    print(len(comments), "comments; first:", repr(comments[0]["comment_text"][:60]))
```

```
2 comments; first: 'GfcgUZxolnnnmmmmmmmm mmm.'
```

### TikTok Shop and discovery, over raw HTTP

The SDK has no TikTok Shop methods, and its other discovery calls take no
limit. These go straight to the API, with the standard library only. This path
reads `BRIGHTDATA_API_TOKEN` only, not the CLI login.

```python
import json
import os
import time
import urllib.request

API = "https://api.brightdata.com/datasets/v3"
HEADERS = {
    "Authorization": f"Bearer {os.environ['BRIGHTDATA_API_TOKEN']}",
    "Content-Type": "application/json",
}


def call(path, body=None):
    data = None if body is None else json.dumps(body).encode()
    request = urllib.request.Request(API + path, data=data, headers=HEADERS)
    with urllib.request.urlopen(request) as response:
        return json.load(response)


product = "https://shop.tiktok.com/us/pdp/wireless-bluetooth-earphones-low-latency-gaming-gift/1732339489808880487"
job = call("/trigger?dataset_id=gd_m45m1u911dsa4274pi&include_errors=true", [{"url": product}])
while call(f"/progress/{job['snapshot_id']}")["status"] not in ("ready", "failed"):
    time.sleep(10)
record = call(f"/snapshot/{job['snapshot_id']}?format=json")[0]
if "error" in record:
    raise SystemExit(record["error"])
print(record["title"], record["final_price"], record["currency"], record["store_name"])
```

```
Wireless Bluetooth Earphones, Black TWS Gaming Headset 13.99 USD Saeme
```

Every other endpoint is the same `/trigger` call with a different dataset, a
`type=discover_new&discover_by=...` pair, and a different input. Add
`&limit_per_input=N` to cap a discovery: without it a keyword or a category
returns, and bills, every match. Each input below was accepted by the live API
on 2026-10-20. Several differ from the API reference pages, and the live API
wins:

| endpoint | dataset_id | discover_by | input |
| --- | --- | --- | --- |
| Shop product | `gd_m45m1u911dsa4274pi` | none | `{"url"}`, a `shop.tiktok.com/us/pdp/...` URL |
| Shop by keyword | `gd_m45m1u911dsa4274pi` | `keyword` | `{"keyword"}`. `num_of_products` is rejected |
| Shop by category | `gd_m45m1u911dsa4274pi` | `category`, not `category_url` | `{"category_url"}` |
| Shop by store | `gd_m45m1u911dsa4274pi` | `shop`, not `shop_url` | `{"url"}`, not `shop_url` |
| Posts by profile, Fast API, 36 fields | `gd_m7n5v2gq296pex2f5m` | none | `{"url", "num_of_posts"}` |
| Posts on a tag or music page | `gd_lu702nij2f790tmv9h` | `url` | `{"URL"}`, upper case. `url` is rejected |
| Profiles from a search | `gd_l1villgoiiidt09ci` | `search_url` | `{"search_url", "country"}`, both required |

`bdata pipelines info <dataset_id> <method>` prints the input fields the API
validates, which is how the table above was checked. Discovery is slow: shop
discovery by keyword and by category took 18 to 20 minutes for two products.

## The data

The fields most people want:

```
url  create_time  description  hashtags  digg_count  comment_count  play_count  account_id
```

`digg_count` is likes, `collect_count` is saves, and `share_count` arrives as a
string. The code hardcodes no field list. Whatever the API returns lands in
`result.data`, and in the command's file.

<!-- fields:start -->
<details>
<summary>All 44 fields, with type and description</summary>

Regenerated every day from the dataset schema, via
`client.datasets.tiktok_posts.get_metadata()`, so it cannot go stale. A
post carries the fields that apply to it; the sample file has 44
of these 44.

| field | type | description |
| --- | --- | --- |
| `url` | url | Unique post link |
| `post_id` | text | Identification for each post |
| `description` | text | Text accompanying the post |
| `create_time` | date | Post creation timestamp |
| `digg_count` | number | Likes received |
| `share_count` | text | Number of shares |
| `collect_count` | number | Number of times saved |
| `comment_count` | number | Number of comments |
| `play_count` | number | Video play count |
| `video_duration` | number | Duration of the video |
| `hashtags` | array | Included hashtags |
| `original_sound` | text | Sound used, if any |
| `profile_id` | text | Unique user identifier |
| `profile_username` | text | User's username |
| `profile_url` | url | User's profile link |
| `profile_avatar` | url | User's profile picture |
| `profile_biography` | text | User's bio |
| `preview_image` | url | Post preview image |
| `post_type` | text | Type of post (video, image, text) |
| `discovery_input` | object | Discovery input value |
| `offical_item` | boolean | Indicates if the video is an official item |
| `secu_id` | text | Security identifier for the profile |
| `original_item` | boolean | Indicates if the video is original |
| `shortcode` | text | Shortcode for the video |
| `width` | number | Width of the video |
| `ratio` | text | Aspect ratio of the video |
| `video_url` | url | URL of the video, URLs that are generated are session-based only |
| `music` | object | Information about the music used in the video |
| `cdn_url` | url | URL link to media |
| `is_verified` | boolean | Indicates whether the TikTok account is verified (boolean: True/False) |
| `account_id` | text | Account ID |
| `carousel_images` | array | List of images for 'carousel'-type TikTok posts |
| `tagged_user` | array | Tagged user |
| `profile_followers` | number | Number of followers of the post Author |
| `tt_chain_token` | text | A session-related token used by TikTok to track and validate user requests |
| `region` | text | The region column indicates the video's country of origin (e.g., US, UK) |
| `commerce_info` | text | Information indicating if the post includes shopping, promotional, or affiliate content |
| `subtitle_url` | url | URL for the subtitles |
| `subtitle_format` | text | Format of the subtitles |
| `country` | text | Some profiles are restricted by location. Please select a country to continue(from discovery) |
| `subtitle_info` | array | Subtitle info |
| `cdn_link` | text | URL link to media |
| `num_share_count` | number | Number of shares |
| `comments` | array | Comments of the post |

</details>
<!-- fields:end -->

<details>
<summary>The start of a real output file, from <code>tiktok-scraper tiktok --limit 1</code></summary>

```json
{
  "generated_at": "2026-10-20T07:39:08.192204+00:00",
  "handles": [
    {
      "handle": "tiktok",
      "posts": [
        {
          "url": "https://www.tiktok.com/@tiktok/video/7694797676306615583",
          "post_id": "7694797676306615583",
          "description": "this summer gave us pop songs, glam routines, and a few main-character habits. @McCall Mirabella breaks down this summer's souvenirs \ud83d\udc2c\u2600\ufe0f\nMcCall Mirabella breaks down the key souvenirs of this summer, from the pop songs we can't stop listening to and glam routines to the main-character habits we all adopted. \ud83d\udc2c\u2600\ufe0f\nThis summer was more than just a season; it was a collection of memories and trends that defined the moment. From the songs that were on repeat to the specific routines we all adopted, the summer of 2024 left its mark. McCall Mirabella breaks down the key souvenirs from this summer, capturing the essence of the season.\n\nThe soundtrack of the summer was undeniably pop, with catchy tunes that became the backdrop for countless moments. Alongside the music, we saw a rise in \"glam routines,\" where self-care and beauty became a central part of the daily experience. These weren't just simple tasks; they were rituals that felt special and indulgent.\n\nPerhaps the most defining trend was the adoption of \"main-character habits.\" This concept encourages living life as if you're the protagonist of your own story, embracing confidence, style, and a sense of purpose in everyday actions. McCall Mirabella's breakdown explores how these pop songs, glam routines, and main-character habits came together to create the unique souvenirs of this summer. \ud83d\udc2c\u2600\ufe0f",
          "create_time": "2026-10-09T22:25:11.000Z",
          "digg_count": 12000,
          "share_count": "1260",
          "collect_count": 1229,
          "comment_count": 5094,
          "play_count": 1400000,
          "video_duration": 49,
          "original_sound": "TikTok: original sound",
          "profile_id": "107955",
          "profile_username": "TikTok",
  ...
```

The whole file, one post with every field, is
[examples/sample_output.json](examples/sample_output.json).

</details>

## When it fails

| you see | what it means |
| --- | --- |
| `API token required but not found.` | Exit 2, before any request. Set the token. |
| `failed  @name: User is banned` | Exit 1. No such account, usually a typo. TikTok says "banned" for an account that does not exist. |
| `got     @name: 0 posts, the account has no public posts in the period searched` | Exit 0, and correct. The API reports an empty window as an error row. |
| `failed  @name: timeout` | Exit 1. A request gives up after 900 seconds. Run it again. |

Any failure exits 1, so a run is safe to gate a script on.

From the SDK, the same conditions look like this:

| you see | what it means |
| --- | --- |
| `AuthenticationError: Unauthorized (401)` | The token is set but wrong. |
| `result.success` is `False`, `result.status` is `"timeout"` | The SDK gave up waiting, 240 seconds by default. Pass `timeout=900`, or run it again. |
| a row in `result.data` with an `error` key | The API's answer for one input: an empty window, or no such account. The other rows are fine. |
| HTTP 400 `validation_error` on raw HTTP | An input field the API does not accept. The message names it; compare with the table above. |

## Coding agents

No Python, nothing installed. Paste both lines; the first opens a browser
once, or use `bdata login --device` over SSH and in CI:

```bash
npx -p @brightdata/cli bdata login
npx -p @brightdata/cli bdata pipelines run gd_m7n5v2gq296pex2f5m collect_by_url --url "https://www.tiktok.com/@tiktok" --num_of_posts 2 --pretty
```

The CLI runs a scraper by its dataset ID, every TikTok one including Shop and
discovery. `bdata pipelines search tiktok` lists their IDs and methods, and
`--limit-per-input N` caps a discovery, 10 by default.

`npx skills add brightdata/skills` teaches Claude Code, Cursor and Codex these
commands and the docs, so plain language works afterwards. Full guide:
[Bright Data for your coding agent](https://docs.brightdata.com/quickstart-coding-agent).

No terminal, for a hosted assistant? The
[Bright Data MCP server](https://github.com/brightdata/brightdata-mcp#which-tool-to-use)
has TikTok tools in its `social` group, which is off unless you ask for it:

    https://mcp.brightdata.com/mcp?token=YOUR_API_TOKEN&groups=social

An agent can also open the account itself, no signup form:
[agent registration](https://brightdata.com/auth.md). Everything else Bright
Data connects to, from LangChain to Zapier and n8n:
[integrations](https://docs.brightdata.com/integrations/introduction).

<details>
<summary>TikTok commands for the CLI, each run on 20 Oct 2026</summary>

Prefix each command with `npx -p @brightdata/cli`, or install once with
`npm i -g @brightdata/cli`. Each prints JSON and costs one credit per record.

Six scrapers, each run with `bdata pipelines run <dataset_id> <method>`:
profiles `gd_l1villgoiiidt09ci`, posts `gd_lu702nij2f790tmv9h`, comments
`gd_lkf2st302ap89utw5k`, the Posts by Profile Fast API `gd_m7n5v2gq296pex2f5m`
and TikTok Shop `gd_m45m1u911dsa4274pi`.

```bash
bdata pipelines run gd_l1villgoiiidt09ci collect_by_url "https://www.tiktok.com/@tiktok" --pretty
```

1 record, 35 fields, 33 seconds. `"account_id": "tiktok"`, `"followers": 96300000`.

```bash
bdata pipelines run gd_m7n5v2gq296pex2f5m collect_by_url --url "https://www.tiktok.com/@tiktok" --num_of_posts 2 --pretty
```

2 records, 34 fields, 70 seconds. Without `--num_of_posts` it returns, and
bills, every post the account has.

```bash
bdata pipelines run gd_lu702nij2f790tmv9h collect_by_url "https://www.tiktok.com/@tiktok/video/7680721699171601694" --format csv -o posts.csv
```

11 minutes. The CSV header carries the same field names as the table above.

```bash
bdata pipelines run gd_lu702nij2f790tmv9h discover_by_keyword --search_keyword "street food" --limit-per-input 2 --pretty
```

2 records, 41 fields, 6.5 minutes.

```bash
bdata pipelines run gd_lkf2st302ap89utw5k collect_by_url "https://www.tiktok.com/@cooking__alone/video/7565981443512339742" --timeout 900 --pretty
```

2 records, 16 fields, 2 minutes. One credit per comment, so check the post's
`comment_count` first. The same command without `--timeout 900` once gave up
at the CLI's 600-second default.

```bash
bdata pipelines run gd_m45m1u911dsa4274pi collect_by_url "https://shop.tiktok.com/us/pdp/wireless-bluetooth-earphones-low-latency-gaming-gift/1732339489808880487" --pretty
```

1 product, 44 fields, 2 minutes. Shop discovery works the same way with
`discover_by_keyword --keyword`, `discover_by_category --category_url` or
`discover_by_shop`, but took 18 to 20 minutes for two products, past the
600-second default: add `--timeout 1500`, or `--async` and fetch later with
`bdata pipelines results <snapshot_id>`.

`bdata pipelines info <dataset_id> <method>` shows a scraper's input fields.
`bdata pipelines run --help` shows `--format json|csv|ndjson|jsonl`, `-o FILE`,
`--timeout` and `--async`.

</details>

## Support

Bugs in this repo:
[open an issue](https://github.com/brightdata/tiktok-scraper/issues), and
[CONTRIBUTING.md](CONTRIBUTING.md) says what to put in it.
Anything about the API, your account or your credits:
[Bright Data support](https://brightdata.zendesk.com/hc/en-us/requests/new).

## License

[Apache 2.0](LICENSE).

## Node.js version

The same TikTok scraper is available for Node.js: [brightdata/tiktok-scraper-node](https://github.com/brightdata/tiktok-scraper-node).
