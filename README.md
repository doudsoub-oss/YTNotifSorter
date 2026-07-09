# YTNotifSorter

A browser-console script that lists every YouTube channel you're subscribed to whose notification bell is set to **All**.

It reads the data straight from YouTube's own *Manage subscriptions* page, so there's nothing to install and no API key required.

## Usage

1. Sign in to your YouTube account.
2. Go to [youtube.com/feed/channels](https://www.youtube.com/feed/channels).
3. Open DevTools (`F12`) and switch to the **Console** tab.
4. Paste the script below and press `Enter`.

> **Note:** the first time you paste into the console, the browser may block it for security and ask you to type `allow pasting` before it lets you paste code.

The script scrolls to load all of your subscriptions, then prints and copies to your clipboard the list of channels set to **All**.

## Script

```js
(async () => {
  const ALL = 2;                                  // bell state to filter (see table below)
  const sleep = ms => new Promise(r => setTimeout(r, ms));

  // 1. Force YouTube to lazy-load every subscription
  let stable = 0, lastLen = 0;
  for (let i = 0; i < 300 && stable < 3; i++) {
    window.scrollTo(0, document.documentElement.scrollHeight);
    await sleep(1000);
    const len = JSON.stringify(ytInitialData).length;
    stable = (len === lastLen) ? stable + 1 : 0;
    lastLen = len;
  }
  window.scrollTo(0, 0);

  // 2. Pull channel name + current bell state out of ytInitialData
  const out = [], seen = new Set();
  const bellState = node => {
    let s;
    (function dig(o){
      if (s !== undefined || !o || typeof o !== 'object') return;
      const t = o.subscriptionNotificationToggleButtonRenderer;
      if (t && typeof t.currentStateId === 'number') { s = t.currentStateId; return; }
      for (const k in o) dig(o[k]);
    })(node);
    return s;
  };
  (function walk(o){
    if (!o || typeof o !== 'object') return;
    if (o.title && o.subscribeButton) {
      const name = o.title.simpleText || (o.title.runs||[]).map(r=>r.text).join('');
      const id   = o.channelId || o.navigationEndpoint?.browseEndpoint?.browseId;
      const st   = bellState(o);
      const key  = id || name;
      if (st !== undefined && !seen.has(key)) { seen.add(key); out.push({name, id, state: st}); }
    }
    for (const k in o) walk(o[k]);
  })(ytInitialData);

  // 3. Filter to the chosen state, print, and copy to clipboard
  window.__bells = out;
  const matches = out.filter(c => c.state === ALL).map(c => c.name).sort();
  console.log(`${matches.length} / ${out.length} channels with bell state ${ALL}:\n` + matches.join('\n'));
  copy(matches.join('\n'));
})();
```

## Notification states

Change the `ALL` variable at the top of the script to filter by a different bell state:

| Value | Meaning |
|-------|---------|
| `0`   | **None** — no notifications at all |
| `2`   | **All** — every notification |
| `3`   | **Personalized** — YouTube decides which notifications to send you |

These are the state IDs observed in practice; you may occasionally come across others. The full list of subscriptions and their states is kept in `window.__bells` after the script runs, so you can inspect any bucket manually.

## Notes

- The script relies on YouTube's internal page data (`ytInitialData`), so it may break whenever YouTube changes its front-end.
