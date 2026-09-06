# PraiseWorld

A shared page for two people: task board with a visible split of who is carrying what, medication check-in, a nature-trip quiz, and drafting tools.

## Deploy to GitHub Pages

Run this from inside the project folder. The `.gitignore` keeps `CLAUDE.md` and
`CLAUDE-root.md` out of the repo — those are working notes and they name health
details and an employer. Check `git status` before the first commit and make sure
neither is listed.

```bash
git init
git branch -M main
git add .gitignore index.html README.md
git status
```

If `git status` shows only those three files, commit and push:

```bash
git commit -m "PraiseWorld: initial deploy"
git remote add origin https://github.com/prashant7122/PraiseWorld.git
git push -u origin main
```

Create the repo `PraiseWorld` on GitHub first (green **New** button), then in the repo go to
**Settings → Pages → Source: Deploy from a branch → Branch: `main` / `root` → Save**.

It goes live at `https://prashant7122.github.io/PraiseWorld/` within a couple of minutes.

Make the repo **private** if you don't want the contents public. Private repos need
GitHub Pro for Pages; otherwise use Netlify or Cloudflare Pages, which host private
sites free — drag the folder onto their dashboard and it deploys.

## What ships in the file, and what doesn't

`index.html` is deliberately impersonal. No names, no employers, no health details,
no diagnoses, nothing that identifies either of you. The starter tasks are generic
home and interiors admin.

Everything personal is typed in once at runtime and stays in your saved state:

- **Names and context**, at the bottom of the page. The context box is free text and
  is what the AI drafts read, so put the useful background there — work, what is
  going on, what she is dealing with.
- Everything you add to the board, the medication list and the notes.

That means the repo can be public without publishing anything about either of you.
Keep it that way: if you add a task or a prompt with real details in it, put it in
the app, not in the file.

## Two things about deploying

**Storage.** Inside Claude the page uses shared storage, so you both see the same
board. On GitHub Pages there is no server, so it falls back to each browser's own
local storage — your phone and her phone will hold separate copies. To make it
genuinely shared you need a backend. Cheapest working options: a free Supabase
project, or Cloudflare Workers KV.

**The AI features.** Unload, the trip planner, the quote checker and the
message-drafting tools call the Anthropic API. That call cannot go directly from a
web page, because it needs an API key and putting a key in front-end code makes it
public the moment you push. You need a tiny proxy that holds the key server-side.

A Cloudflare Worker that does it:

```js
export default {
  async fetch(request, env) {
    const cors = {
      "Access-Control-Allow-Origin": "https://prashant7122.github.io",
      "Access-Control-Allow-Headers": "Content-Type",
      "Access-Control-Allow-Methods": "POST, OPTIONS"
    };
    if (request.method === "OPTIONS") return new Response(null, { headers: cors });
    const body = await request.text();
    const r = await fetch("https://api.anthropic.com/v1/messages", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "x-api-key": env.ANTHROPIC_API_KEY,
        "anthropic-version": "2023-06-01"
      },
      body
    });
    return new Response(await r.text(), {
      status: r.status,
      headers: { ...cors, "Content-Type": "application/json" }
    });
  }
};
```

Set `ANTHROPIC_API_KEY` as a Worker secret, deploy, then open the site, scroll to the
bottom and click **AI endpoint**. Paste the Worker URL. Reload.

Until you do that, everything else works — board, load bar, medication, quiz,
destination rankings, books, briefs. Only the buttons that generate text will fail.
