# YTNotifSorter
A script to display all the youtube chanel you follow which have all the notifications enabled.

## Usage
- First you need to be connected to your youtube account
- Go to youtube.com/feed/channels
- Open DevTools (F12)
- Go to the console
- Finally paste this script :
*The console might give you an error message telling to write `allow pasting` before being able to actually paste some code to it*

You can change the value of the `ALL` variable to display the other state of notifications.
- **0** = YT chanel from which you recieve **NO** notificaitons at all
- **2** = YT chanel from which you recieve **ALL** the notificaitons
- **3** = YT chanel that youtube recommends you through notifications

```
(async () => {
  const ALL = 2;                                  // bell state : "0" = "None", 2 = "All", 3 = "Personalized"
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

  // 3. Filter to "All", print, and copy to clipboard
  window.__bells = out;
  const allChannels = out.filter(c => c.state === ALL).map(c => c.name).sort();
  console.log(`${allChannels.length} / ${out.length} channels set to "All":\n` + allChannels.join('\n'));
  copy(allChannels.join('\n'));
  console.log('\nCopied to clipboard.');
})();
```
