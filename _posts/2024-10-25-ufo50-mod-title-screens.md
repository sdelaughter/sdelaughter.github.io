---
layout: post
title:  "UFO 50 Modding Guide: Title Screens"
date:   2024-10-25
tags:   game-dev, modding, UFO-50, GameMaker
---

I've recently started making mods for [UFO 50](https://50games.fun), a phenomenal game (really 50+ different games) that everyone should play.  There's a great [guide](https://discord.com/channels/525973026429206530/1297055481746952294) for getting started making UFO 50 mods on the game's official Discord server, but I thought it might be helpful to document some tips and techniques for modding specific aspects of the game, starting with title screens.  As an example, I'll walk through the process of adding an endless mode to the game Magic Garden.

All 50 sub-games (they are *much* more than "mini-games") use the same `oTitleScreens` object.  Each game is assigned one of 18 "menu styles" which define the basic layout of the title screen.  Magic Garden is assigned menu style 3 in line 10 of `gml_GlobalScript_scr27_Meta`:

```
global.mGameMenuStyle[argument0] = 3;
```

The main logic to build the title screen's structure from this style is in `gml_Object_oTitleScreens_Other_12`.  The portion used for menuStyle 3 starts on line 53:

```
else if (menuStyle == 3)
{
    menuOption[0] = scrStringManual("title_game_start", 0);
    menuOption[1] = scrStringManual("title_ghigh_scores", 0);
}
```

It adds two options to the menu, one for "START GAME" and one for "HIGH SCORES".  We can't simply add an extra option here, because that will change the title screens of *all* games using menu style 3.  Instead, we can define behavior for a new menu style down below.  Numbers 0-17 are already in use but we can choose any other value.  Let's go with 27 to match the game's ID.  Before the `else` condition that sets "INVALID MENU STYLE", add:

```
else if (menuStyle == 27)
{
    menuOption[0] = scrStringManual("title_game_start", 0);
    menuOption[1] = "ENDLESS";
    menuOption[2] = scrStringManual("title_ghigh_scores", 0);
}
```

We've copied the `menuStyle == 3` code above, but shifted the high scores entry down one and added a new "ENDLESS" entry in between.  Ideally we should probably add a new translation string for this and retrieve it using `scrStringManual()` like the other options do, but that's a topic for another post.  Hardcoding the string "ENDLESS" works fine for now.

Next we need to modify the game's metadata to use this new menu style, by simply changing line 10 in `gml_GlobalScript_scr27_Meta` to:

```
global.mGameMenuStyle[argument0] = 27;
```

Now we need to change what happens when selecting our new menu option.  This is handled farther down in `gml_Object_oTitleScreens_Other12`, where the `SUB_TITLE` substate is run.  When the confirm button is pressed, the default behavior for any menu style and selection is to start the game:

```
else
    scrSwitchSub(SUB_SELECTED)
```

Before that, there's a check to see if we're using menuStyle 3 and have the second option selected, and if so we switch to the high scores screen: 

```
else if (menuStyle == 3 && menuSel==1)
    scrSwitchState(STATE_HIGHSCORE);
```

We need to add a similar check for our new menuStyle, but remember that High Scores are now the third option instead of the second:

```
else if (menuStyle == 27 && menuSel==2)
    scrSwitchState(STATE_HIGHSCORE);
```

Since we haven't explicitly defined any behavior for the first two options ("START GAME" and "ENDLESS"), both of them will transition to the `SUB_SELECTED` substate.  This is handled even farther down in `gml_Object_oTitleScreens_Other12`.  Most of the code here is for setting the number of players, but it seems like a good place to set a flag for endless mode as well.  Just before `scrSwitchSub(SUB_BUFFER)`, we can add:

```
if (menuStyle == 27)
{
    if (menuSel == 0)
        global.endlessMode = false;
    else if (menuSel == 1)
        global.endlessMode = true;
}
```

For good measure, we should also initialize this new global variable to false in `gml_GlobalScript_scrInit`.  It would probably make more sense to set it as a variable of o27__Game, but Im not sure if that object has actually been created yet at this point.

Finally, we need to reference this global variable in the game to see if we should bypass the win condition.  This check happens in the `Other 10` event of o27_Player:

```
else if (o27__Game.rank == 3 && o27__Game.savedTotal >= o27__Game.GOLD_GOAL)
```

We just need to add another component to this conditional so that it only applies when `global.endlessMode` is false:

```
else if (o27__Game.rank == 3 && o27__Game.savedTotal >= o27__Game.GOLD_GOAL && global.endlessMode == false)
```

That's all there is to it, happy modding!