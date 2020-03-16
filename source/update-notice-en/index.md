---
title: Bandori Database Update Note
date: 2018-09-17 20:44:03
---

中文版请看[这里](/update-notice-zhcn)

### 0.9.0
- Quasar version to 1.0.0-rc4
- Data from **Chinese Mainland server** now available!
- Remove bad external links
- Remove update notice modal, **please refer this page for update notice**
- A "VIEW DETAIL" button is added to event and gacha card
- Filter button now have badge if you applied a custom filter
- "Live Achievement Reward" in "Music Detail" page displays as a modal
- "Current Event" page shows "Event Point Rewards", "Event Point Rewards" and "Stories"
- A refined card detail page special thanks to [@TimothyBu](https://github.com/timothybu)

### 0.8.2
- Added "Title Screen Image" page
- Fix the wrong order of displaying character birthday
- Fix the "next birthday" display of two or more characters whose birthday are on the same day
- Fix the wrong order of EN server events
- Gacha modal shows only pickup cards as default
- Use new cache policy for faster data loading
- All canvas with PIXI.js now handle touch event normally

### 0.8.0
- Event card on home page use the same countdown layout as gacha card
- Move "change server" from sidebar to "SETTINGS" button on right top corner
- Redesigned card list page
- In music list page, display band icon under the music name instead of on the jacket image
- Add "Return" button for some detail pages
- Redesigned card detail page, add a button for jumping to Live2D costume for current card (if exists)
- Single Frame Cartoon of Japan Server comes back
- Add "Tag" and "Release Date" to music list filter, default sort is "Release Date"
- Fix crash of beatmap player when playing some charts
- Fix wrong bonus info for current event

### 0.7.1
- Add band icon and rarity icon to card thumb
- Redesigned card list, no loading full card image for less data consumption and better performance
- Improve music list layout (thanks to @Cee)
- Add stamp list for each server (thanks to @Cee)
- Correct some korean translation (thanks to @thtl1999)
- Fix redundant description on gacha detail modal (thanks to @Cee and @alxnr)
- Fix wrong stretch of gacha display card on index page (thanks to @Cee and @alxnr)
- Fix slow loading of card list on gacha detail modal
- Add loading indicator to the place that need some time for loading
- Remove Big Picture Viewer on music detail page