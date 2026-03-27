---
name: web-fetch
description: >
  Fetch and read the content of a web page URL using curl and html2text.
  Use this skill whenever you need to read or extract content from a specific
  URL — following up on search result links, reading documentation pages,
  checking release notes, viewing articles, or fetching any web page content.
  This replaces the built-in WebFetch tool. Trigger this skill whenever you
  would normally use WebFetch, or when the user gives you a URL to read, or
  when you have a URL from a web search that you want to dig deeper into.
---

# Web Fetch via curl + html2text

This skill fetches web page content using `curl` and converts it to readable
plain text using `html2text`. It is a lightweight, dependency-free
replacement for the built-in `WebFetch` tool.

## How to use

Run the following command via Bash, replacing `<url>` with the target URL:

```bash
curl -sL "<url>" | html2text
```

### Flags explained

| Flag | Purpose |
|---|---|
| `-s` | Silent mode — no progress bar or error messages |
| `-L` | Follow redirects automatically |

### Limiting output length

Web pages can be very long. To avoid overwhelming context, truncate the
output when you only need the beginning of a page:

```bash
curl -sL "<url>" | html2text | head -n 200
```

Adjust the line count as needed. For very large pages, start with a smaller
number and fetch more if needed.

## Tips

- If you need to send custom headers (e.g., a User-Agent), use
  `curl -sL -H "User-Agent: Mozilla/5.0" "<url>"`.
- Some sites block `curl` without a User-Agent header. If you get an empty or
  403 response, try adding one.
- For JSON APIs, skip `html2text` and use `curl -sL "<url>"` directly.
- Pair with the **web-search** skill: search first to find relevant URLs,
  then fetch specific pages to read them in full.

## Example

```bash
curl -sL "https://bun.sh/docs/cli/install" | html2text | head -n 200
```
