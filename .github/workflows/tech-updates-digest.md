---
emoji: 📰
name: Tech Updates Digest
description: Collect the latest Azure, GitHub Changelog, and OpenAI news updates every 12 hours and record them in Index.html through a pull request.
intent: Keep Index.html current with newly published Azure, GitHub, and OpenAI updates, delivered only as a pull request and never as a direct push to the default branch.
on:
  schedule: every 12h
  workflow_dispatch:
permissions:
  contents: read
strict: true
network:
  allowed:
    - defaults
    - azure.microsoft.com
    - "*.azure.microsoft.com"
    - www.microsoft.com
    - github.blog
    - "*.github.blog"
    - openai.com
    - "*.openai.com"
tools:
  web-fetch:
  bash:
    - "curl"
safe-outputs:
  create-pull-request:
    title-prefix: "[updates] "
    labels: [automation, updates]
    draft: true
    if-no-changes: "ignore"
    allowed-files:
      - "Index.html"
---

# Tech Updates Digest

## Task

Collect product and news updates published in the **last 12 hours** (the reporting window ends at the current run time) from these sources, and record them in `Index.html`:

- Azure updates: <https://azure.microsoft.com/en-us/update> (feed: <https://azure.microsoft.com/en-us/updates/feed/>, which redirects to <https://www.microsoft.com/releasecommunications/api/v2/azure/rss>)
- GitHub Changelog: <https://github.blog/changelog/> (feed: <https://github.blog/changelog/feed/>)
- OpenAI news: <https://openai.com/news/> (feed: <https://openai.com/news/rss.xml>)

The Azure and OpenAI listing pages are rendered client-side, so neither `web-fetch` nor `curl` can execute the JavaScript that populates them. Always fetch the feed URL for a source first and only fall back to the HTML page if the feed is unavailable.

Steps:

1. Fetch each source. Try the `web-fetch` tool first. If `web-fetch` is not available in this runtime, or the call fails, retry with `curl` from the shell, for example:

   ```
   curl --fail --location --show-error --silent --max-time 60 "<url>"
   ```

   `curl` must go through the sandbox proxy, so keep the ambient `HTTPS_PROXY` / `HTTP_PROXY` environment variables intact — do not unset them or pass `--noproxy`. Do not run connectivity probes against unrelated domains (for example `example.com`); only test the exact source feed and fallback page URLs listed above. Treat a denial on an unrelated domain as non-actionable. Only report `missing_tool` after **all three sources** fail with both `web-fetch` and `curl`, and include the exact per-source errors in the reason; always continue with the remaining sources instead of aborting the run.
2. Extract each update's title, publication date, canonical link, and a brief 1-2 sentence summary of what changed.
3. Read `Index.html` in the repository root if it exists. Treat the update links already present in the file as the deduplication key: skip any update whose link is already recorded.
4. Write the result to `Index.html`:
   - If `Index.html` does not exist, create it as a complete, valid HTML5 document titled "Tech Updates" with a short intro and one section per source (Azure, GitHub Changelog, OpenAI).
   - If it exists, append the new updates to the matching source section, newest first, preserving all existing content and the document structure.
   - Each entry must include the update title as a link to the source URL, the publication date, and the brief summary.
   - Escape any text taken from the sources so the generated HTML stays well-formed, and do not include scripts, iframes, or remote assets.
5. Modify only `Index.html`. Do not change any other file.

## Safe Outputs

- Publish the change with the `create-pull-request` safe output. Use a title summarizing how many updates were added and a body listing each added update with its source and link. Never push to the default branch directly.
- Call `noop` with a short reason when there are no new updates in the window, or when every discovered update is already recorded in `Index.html`.
