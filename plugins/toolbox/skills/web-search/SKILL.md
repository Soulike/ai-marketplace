---
name: web-search
description: >
  Search the web for current information using GitHub Copilot CLI.
  Use this skill whenever you need to look up real-time or recent information
  from the internet — stock prices, news, documentation, release notes,
  weather, sports scores, API references, or anything else that requires
  up-to-date web data. This replaces the built-in WebSearch tool. Trigger this
  skill whenever you would normally use WebSearch, or when the user asks you to
  "search", "look up", "find out", "what is the latest", "current price of",
  or any query that implies fetching live information from the web.
---

# Web Search via Copilot CLI

This skill provides web search capability through the GitHub Copilot CLI,
which has access to a `web_search` tool and can return sourced, up-to-date
results.

## How to use

Run the following command via Bash, replacing `<query>` with a clear,
specific search query. Always append the instruction to include source URLs
so you can dig deeper into the results if needed:

```bash
copilot --silent --allow-all-urls --allow-tool="web_search" --available-tools="web_search" --model gpt-5-mini -p "<query> Include all source URLs in your response."
```

The "Include all source URLs" suffix ensures copilot returns the URLs it
found, which you can then fetch using the **web-fetch** skill to dig deeper.

### Flags explained

| Flag | Purpose |
|---|---|
| `--silent` | Suppresses metadata noise (model info, token counts, tool lists) for clean output |
| `--allow-all-urls` | Permits fetching from any URL encountered during search |
| `--allow-tool="web_search"` | Grants permission to use the web search tool |
| `--available-tools="web_search"` | Restricts available tools to only web search (faster, cheaper) |
| `--model gpt-5-mini` | Uses a lightweight model — fast and costs 0 Premium requests |
| `-p "<query>"` | The search prompt |

## Tips for good results

- Be specific in your query. Include dates, names, and context.
- Include the current date in the query when searching for "today's" or
  "latest" information, since the copilot model may not know what today is.
- For complex research, run multiple searches with different queries rather
  than one broad query.
- The command may take 15-60 seconds depending on query complexity.
- Set an appropriate Bash timeout (e.g., 60000ms) since web searches can be
  slow.

## Example

```bash
copilot --silent --allow-all-urls --allow-tool="web_search" --available-tools="web_search" --model gpt-5-mini -p "What is the MSFT stock closing price on March 21, 2026? Include all source URLs in your response."
```

This would return something like:

> U.S. markets were closed on Saturday, March 21, 2026. The most recent close
> was Friday, March 20, 2026: MSFT closed at $389.02 per share.
> Sources: StatMuse (https://...), Google Finance (https://...)

You can then use the **web-fetch** skill on any of those URLs to get more
detailed information.
