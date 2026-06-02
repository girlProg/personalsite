---
title: "ci/cd telegram notifications"
date: 2026-05-22
categories: [uncategorized]
---

so I've been having amaro in production break, and i would only find out when i see the github email hours later or the users send me a message asking whats happening.  
sso i thought why not have telegram message me each time deployment or tests dont pass. With a little googling, turns out appleboy has a telegram github action

so i got to work with BotFather, and got a new bot up and running. Added secrets to github and modified my yaml file to include this:

```
- name: Notify on failure
  if: failure()
  uses: appleboy/telegram-action@master
  with:
    to: ${{ secrets.TELEGRAM_CHAT_ID }}
    token: ${{ secrets.TELEGRAM_BOT_TOKEN }}
    message: |
      ❌ CI failed on amaro
      Branch: ${{ github.ref_name }}
      Commit: ${{ github.event.head_commit.message }}
      By: ${{ github.actor }}
      Run: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
```

cause i was eager to test it, so i added anothr one for on success:

```
- name: Notify on success
  if: success()
  uses: appleboy/telegram-action@master
  with:
    to: ${{ secrets.TELEGRAM_CHAT_ID }}
    token: ${{ secrets.TELEGRAM_BOT_TOKEN }}
    message: |
      ✅ CI passed on amaro
      Branch: ${{ github.ref_name }}
      Commit: ${{ github.event.head_commit.message }}
      By: ${{ github.actor }}
```

and lo and behold!

![](https://softcopyofme.yedi.com.ng/wp-content/uploads/2026/05/IMG_8263-1024x419.jpeg)

*Telegram notification*

so handy, this was all up and running within 10 minutes. Definitely worth adding to the pipeline!