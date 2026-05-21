# Achievements
Create an optimized **Achievement System** directly into the core that can be customised via SCP. Managing the entire logic of adding, completing and tracking the achievement via player props and then recall it customized into gumps.

## Suggested Setup in SCP
We can imagine to create an achievement same as an item, that can be recalled into the player via its index.

Example:
```js
[ACHIEVEMENT 1]
NAME=Orcs Master
TYPE=Kill
OBJECT=Single                               // Single for "chardef", Multi for "subsection"
CATEGORY=Hunting                            // Category where the achievement sits
TARGET=500                                  // Creatures to kill
POINTS=10                                   // Reward
ITEMS=i_gold 500, i_platemail_chest 1       // Items reward in backpack

ON=@Completion
    // This could be a custom trigger when a player complete the achievements
    // for example we can set for a specific achievements to set young=0
```

## The Player Props
The player will have props like `src.achievement.<ID>.actual` to check against `serv.achievement.<ID>.target`