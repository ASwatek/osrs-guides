# A Guide For Tick Manipulation Mechanics In Skilling
Throughout this guide, we'll introduce skilling mechanics and tick-by-tick descriptions of optimal skilling methods. This text is not intended as a guide to perform one particular method, but rather as a guide to broadly understand skilling. 

Few of the ideas presented here are due to the writers. Thanks to Bea5, Drew, Fraser, GeChallengeM, Henke, Jamal, Julia, Jukebox Romeo, Nechs, Port Khazard, Tannerdino, and all of The Summit for their explanations and helpful discussions.

### Table of Contents

 - [**Ticks via Henke's model**](#tick-manipulation-i-ticks-via-henkes-model)
    - [Client input](#client-input)
    - [NPC and player turns](#npc-and-player-turns)
    - [Elements of turns](#elements-of-turns)
       - [Stalls](#stalls)
       - [The queue](#the-queue)
       - [Timers](#timers)
       - [The area queue](#the-area-queue)
       - [Interactions](#interactions-with-objectsitems-and-npcsplayers)
 - [**The Skilling Tick**](#tick-manipulation-ii-the-skilling-tick)
    - [Early examples](#early-examples)
    - [Inventory actions](#inventory-actions)
    - [Eating](#eating)
    - [Flinching](#flinching)
    - [Review of specialized mechanics](#review-of-specialized-mechanics)
 - [**Stalls**](#tick-manipulation-iii-stalls)
    - [Stalls after movement](#stalls-after-movement)
    - [Stalls on successful rolls](#stalls-on-successful-rolls)
    - [Stalls in cooking](#stalls-in-cooking)
 - [**Movement and pathing**](#tick-manipulation-v-movement-and-pathing)

## Tick Manipulation I: Ticks via Henke's Model

The shortest unit of time in Old School Runescape is one game tick, or just _tick_ for short. This means that describing the state of the game for each tick provides a complete and perfect description. While this is true, it's important that the description is read out at the same place during each tick. This is because actions are executed each tick, so the game state incrementally changes as time progresses within a tick. Henke's model provides a powerful framework to understand the order of execution of actions within each tick.

````
Server tick:
	for each player:
		Process client input

	for each npc:
		Stalls end
		Timers
		Queue
		Interaction with objects/items
		Movement
		Interactions with players/npcs

	for each player:
		Stalls end
		# Close interface if strong command queued
		Queue
		Timers
		Area queue
		Interaction with objects/items
		Movement
		Interaction with players/npcs
		# Close interface if trying to log
````

At the highest level of complexity, Henke's model asserts that: first, _client input_ is evaluated for each player; second, each nonplayer character, or npc, takes its _turn_ in order of npc ID; last, each player takes its _turn_ in order of player ID, or pid.

### Client input

Clicks on one tick are not processed until client input on the next tick. This can sometimes create an appearance of lag. We show this in the clip below.

<div style="text-align:center"><img src="https://i.imgur.com/OEadZs3.gif" alt='Opening bank interface' width=500>

On the tick that we clicked the bank, we also right clicked the range. Had the bank interface appeared instantly, it would not have been possible for us to right click the range.

At most ten commands will execute in client input during any tick. For example, if twenty-eight unclean herbs are clicked on then an adjacent and reachable tile is clicked on in the same tick, then ten herbs will be cleaned on the next tick, then ten herbs will be cleaned on the next tick, then the final eight herbs will be cleaned and movement will happen on the next tick. Note, rarely only the last ten commands will be executed on the tick after the clicks.

### NPC And player turns

After client input is processed, all non player characters (or npcs) will take their turns. Following that, all players will take their turns. An immediate consequence of this ordering is that PVMers typically do not have to be mindful of their player ID, whereas PVPers do, since in PVP our enemy could have their turn either before or after ours.

An interesting way to take advantage of the ordering of client input and the turns is through using mithril seeds to attack an npc while the npc is unable to attack back, such as in the clip below. Note that mithril seeds move the player during client input at the same time that the click on them is processed.

<div style="text-align:center"><img src="https://i.imgur.com/7FEB1fG.gif" alt='Mith seed to avoid damage' width=500>

Below, we write out the actions that are occurring on the server, in terms of Henke's model.

 - **Tick 1**: During our character's turn, both move from under KQ and attack KQ.
 - **Tick 2**: During our client input, move to underneath KQ via mithril seeds.
 - **Tick 3**: During our character's turn, pick the flowers.

Our character is never in range of KQ during one of her turns, since her turn on **Tick 1** is before our actions and her turn on **Tick 2** is after our action. 

### Elements of turns

Within each turn, the terms which appear (such as 'timer' and 'queue ') are categories of commands, and the ordering of the these categories denotes the ordering of the execution of the corresponding commands within a turn. We will mostly focus here on the player turns, however do notice that this ordering is not the same for npcs and players.

#### Stalls

When our character is stalled, our client input and our turn is completely blocked: our timers don't go off, our queue doesn't evaluate, we can't interact with anything, we can't move, and no client input is accepted. An example of a stall comes from teleporting. When we click a teleport in the normal spellbook, a stall starts in client input on the next tick. The stall ends four ticks later, when our coordinates change to our destination.

#### The queue

The queue is a driver of much behavior in game. In this section, we'll first focus on commands unrelated to skilling to initially develop our understanding. Commands in the queue evaluate so that the first that come in are the first that come out. 

A basic example of a queued command is hitsplats. On a turn that a player or npc interacts with an enemy to attack, a command to deal damage is put into the enemy's queue, to be evaluated the next time the enemy has a turn. Notice the asymmetry in this example due to the turn ordering: npcs deal damage to players on the same tick they attack, whereas players deal damage to npcs on the tick after they attack. This can be seen by looking carefully at Henke's model: [this](https://i.imgur.com/BRUE6wn.png) is what happens when the npc attacks, and [this](https://i.imgur.com/i3g7kXw.png) is what happens when the player attacks. Since the player's turn is late in a tick, when the player attacks, the npcs next turn will be on the following tick.

A clear illustration of the position of the queue as being in our turn is provided by trying to kill ourselves with a locator orb and a zamorak brew. Both of these do damage based on our hitpoints at the moment our click on them is processed; however, zamorak brews do their damage in client input, while locator orbs queue their damage. At 11 hitpoints, if we click on a zamorak brew then a locator orb on the same tick, the following clip happens.

<div style="text-align:center"><img src="https://i.imgur.com/serp8Qp.gif" alt='Zammy brew then locator orb' width=500>

Below we provide an alternative view of the clip by writing out what's occurring on the server in text. All of the actions are executed within one tick.

During client input:
 - Zammy brew checks current hp (11), and damage (1) is calculated
 - Zammy brew deals 1 damage
 - Locator orb checks current hp (11-1=10), and damage (9) is calculated
 - Locator orb sends 9 damage to queue

During the queue on our turn:
 - Locator orb deals 9 damage

The situation is different if we perform the clicks in the opposite order, which we see in the clip below.

<div style="text-align:center"><img src="https://i.imgur.com/HitPpiN.gif" alt='Locator orb then zammy brew' width=500>

Below, we again provide an alternative view of the gif by writing out what's occurring on the server in text. After our health hits zero, our death is put into our queue to be evaluated on the next tick.

During client input:
 - Locator orb checks current hp (11), and damage (10) is calculated
 - Locator orb sends 10 damage command to our queue
 - Zammy brew checks current hp (11), and damage (1) is calculated
 - Zammy brew deals 1 damage

During the queue on our turn:
 - Locator orb deals 10 damage, and our health hits 0

There are three types of commands: weak, normal, and strong. Weak commands get deleted by any actions that interrupts. Since having a strong command in our queue will interrupt us every tick, weak commands get deleted when there is a strong command queued. Note that interfaces also get closed by interruptions, which is written earlier than the queue in Henke's model. An example of a command being deleted appears often in wintertodt. Most fletches there are weak commands, while the damage is a strong command. Almost all client input also interrupts: a good quick test for whether a command is weak in the queue is whether rearranging items in our inventory deletes it.

We now discuss the use of the queue in skilling. Weak commands are used most commonly in skilling.

Skilling actions not based on interactions tend happen in the queue. For example, making herb tar uses a weak command in our queue: in client input on the tick after we click a low leveled herb onto swamp tar, the skilling tick will be set if the skilling timer is nonpositive, then a command to complete the make on the skilling tick will be placed as a weak command in our queue. Recall that we can quickly test that making herb tar is indeed a weak command since rearranging our inventory cancels the action.

The queue is also used for repetitive actions in skilling, such as in fletching, potion making, cooking, wintertodt brazier feeding, smithing, and crafting. For most of these, the first action occurs in client input if the first action happens on the tick after the click, then later actions occur from the queue. In all of these examples, the commands are weak.

#### Timers

During the category timers, commands which are to be executed periodically are executed. For example, cannons do all of their work once a tick during timers. Prayer drain also occurs during timers. Poison "goes off" during timers and puts a hitsplat in our queue, which is evaluated on the next tick.

#### The area queue

The area queue is similar but separate from the queue. It still evaluates on a first in, first out basis, but it only takes in commands related to area effects based on standing on particular tiles. When our character is on certain tiles, called _area tiles_, commands are put into our area queue. Note, when an interface is open, movement is blocked on area tiles: therefore a good test for whether a tile is an area tile is to walk across it with an interface open.

A standard example of the area queue can be seen in the wilderness. Overlays such as the wilderness level and multicombat indicators are controlled by the area queue. Below, we see an example of blocking the wilderness level indicator from updating.

<div style="text-align:center"><img src="https://i.imgur.com/fly1kmc.gif" alt='Teleporting in 31 wildy' width=500>

We now describe the actions depicted in the clip.
 - **Tick 1**: During our turn, move from level 31 wilderness to level 30 wilderness
 - **Tick 2**: During client input, glory teleport starts a stall
 - **Tick 3**:
 - **Tick 4**:
 - **Tick 5**: During our turn, the stall ends and our character is in Edgeville

After we left the level 31 wilderness on **Tick 1** but before our area queue ran on **Tick 2**, we teleported and started a stall. This made our overlay never update to level 30.

#### Interactions with objects/items and npcs/players

Interactions are one of the most useful categories for skilling. They're separated into two types: interactions with object or items and interactions with npcs or players. In between the two, movement occurs, which has many noticeable consequences in game. One of them is that interaction with a banker (an npc) happens sooner than interaction with a bank booth (an object) when moving up to them, as shown in the clip below.

<div style="text-align:center"><img src="https://i.imgur.com/5UBIyFI.gif" alt='Interactions with bank booth/banker' width=500>

In the first case, we move then interact with the banker in the same tick. In the second case, we move on a tick then interact with the bank booth on the following tick. The same behavior can be seen when moving up to a tree (an object) to woodcut or when moving up to a fishing spot (an npc) to fish.

Interactions can be used to produce an optimal method to cook. It's well known that karambwans can be cooked once per tick; however, it's less widely known that the property of karambwans that makes 1t cooking possible is that raw karambwans can produce more than one cooked product. Since raw beef and raw bear meat share this property, they can also be used to 1t cook. This holds even in f2p where no second option explicitly appears, as shown in the clip below.

<div style="text-align:center"><img src="https://i.imgur.com/5JiU7Mp.gif" alt='1t cooking' width=500>

We now describe in text the commands shown in the clip.
 - **Tick 1**: In client input, we start interacting with the fire. During our turn, in interactions with objects, a choice interface opens.
 - **Tick 2**: In client input, our choice is processed and the cook happens. Later in client input, we start interacting with the fire again. During our turn, in interactions with objects, a choice interface opens.
 - **Tick 3**: In client input, the cook happens according to our choice, and then we start interacting with the fire again. During our turn, in interactions with objects, a choice interface opens.

Notice that the first raw beef is cooked before the next raw beef begins to be processed. Each tick, we restart interacting with the fire since otherwise later cooks would happen more slowly from our queue.

## Tick Manipulation II: The Skilling Tick

Most tick manipulation in skilling amounts to carefully controlling the so-called skilling tick. There's two relevant variables for this: One is a `global tick counter` that starts at 0 on a server reboot and increments by one every tick before any player's client input and another is a player-specific variable called the `skilling tick` that specifies a value of the global tick counter where a skilling action can occur.

Sometimes people refer to the skilling tick minus the global tick counter as the _skilling timer_, which can be conceptually convenient since it counts down by one every tick and the skilling action occurs when the skilling timer "goes off" and hits 0. We will refer to the skilling timer most of the times that we mention setting the value of the skilling tick: for convenience, we sometimes write that we are delaying the skilling timer by _k+1_ in place of writing that we are setting the skilling timer to _k_. This is helpful because there are _k+1_ ticks between when the timer is set and when it goes off.

In most skilling examples, which skilling action occurs when the skilling timer goes off is determined through what we're interacting with. We can only interact with one entity (object, item, npc, or player) at a time. 

A first example of the use of the skilling tick can be found in woodcutting, which standardly is a 4 tick action. Let's set the stage: suppose that the global tick counter is 998 and the skilling tick is 900. This means in part that it's been 98 ticks since we've completed our last skilling action. On global tick 998, let's say we click a reachable tree while there's one tile between our character and the tree. Then, on global tick 999, our click is processed in client input where our interaction is set to the tree; later in the tick, during our turn, we move one tile to be adjacent to the tree. On global tick 1000, during our turn, we interact with the tree, and, since our skilling tick is less than the global tick counter, our skilling tick is set to 1003. (Put differently, this sets the skilling timer to 3, or delays the skilling timer by 4.) On global ticks 1001 and 1002, we interact with the tree during our turn largely with no effect. On global tick 1003, which is the value of the skilling tick, our interaction with the tree during our turn produces a roll. If the roll succeeds, we get a log and there's a possibility of the tree falling.

### Early examples

While mining, we roll for an ore against any rock that we're interacting with during the skilling tick. An immediate consequence of this is that stopping interacting with a rock then restarting interacting with a rock has the same effect as continually interacting with a rock. This can be cleverly used while mining iron in the mining guild to buy an extra tick for reaction time, as shown below.

<div style="text-align:center"><img src="https://i.imgur.com/AxUdwnW.gif" alt="Bea's mining guild iron" width=500>

Using a dragon pickaxe and crystal pickaxe provides a chance of delaying the skilling timer by two ticks instead of the usual three ticks. We control for this by always moving to another rock after two ticks: when our pickaxe delays our skilling timer by two ticks, we move to an undepleted rock to set the skilling tick again; when our pickaxe delays our skilling timer by three ticks, we deplete the rock we just moved to, and then move back to the old rock to set the skilling tick again. We also drop ore in client input before interacting with a rock to keep space in our inventory.

The skilling tick is just a number, so it doesn't know what kind of action set the skilling tick. In the iron mining example, the skilling tick both got set from an iron rock and was used to mine an iron rock; however, this consistency was not needed. We can also set the skilling tick using one action and then complete a distinct action during the skilling tick. This strategy is behind almost every optimal skilling method in the game. A first example of this is from setting the skilling timer using woodcutting (which is 4 ticks) then completing a barbarian fishing action (which is by default 5 ticks), as shown in the clip below.

<div style="text-align:center"><img src="https://i.imgur.com/FL2K5q7.gif" alt='Tree fishing' width=500>

In doing this method, we click on the tree on the same tick as the roll from fishing: on the next tick, this makes us interact with the tree when the skilling timer gets delayed. This method, known as _tree fishing_, was optimal near the start of Old School, before more ways to set the skilling tick were known.

Combat does also use the skilling tick, but it uses the skilling tick in a different way than skills like fishing, mining, and woodcutting. While we are interacting with an npc or player, we attack them whenever the skilling timer is nonpositive. On the same tick as we attack, our skilling timer gets to set to our weapon attack speed minus one. This behavior is different than in fishing, mining, and woodcutting, where the skilling timer gets set on the tick after the skilling action, and we can use it to speed up fishing even further.

<div style="text-align:center"><img src="https://i.imgur.com/95cO4aX.gif" alt="Jamal's 3t fishing with darts" width=500>

In the clip, we attack an alt with a dart while our attack options are set to rapid, and then start interacting with the fishing spot. Below we describe this in more detail.
 - **Tick 1**: The skilling timer decrements to -1. During client input, we start interacting with our alt. During our turn, in interactions with players, we attack our alt with rapid darts, setting our skilling timer to 2. 
 - **Tick 2**: The skilling timer decrements to 1. During client input, we start interacting with the fishing spot.
 - **Tick 3**: The skilling timer decrements to 0. During our turn, in interactions with npcs, we get a roll for a fish.

In this method, we stop interacting with the fishing spot on **Tick 1** so that our skilling tick instead gets set from the darts.

### Inventory actions

A major breakthrough in skilling occurred when the community found inventory items which could delay the skilling timer without consuming any items. This was done around the time of the Skilling Cups with the discovery that making herb tar, which takes three ticks, uses the skilling tick. With this, nearly all of fishing, mining, and woodcutting actions could now be done every three ticks. These inventory actions and later eats delay the skilling timer during client input, which makes some of the following methods possible.

An example of 3t skilling with barbarian fishing is below, where we use a knife on a teak log rather than an herb on swamp tar, although these are essentially equivalent from the perspective of the skilling tick.

<div style="text-align:center"><img src="https://i.imgur.com/pAPlYsy.gif" alt='3t Barbarian fishing with knife-log' width=500>

Below we describe in text the actions in the clip.
 - **Tick 1**: The skilling timer decrements to -1. During client input, we start making a teak stock, which removes our interaction with the fishing spot, sets our skilling timer to 2, and puts a weak command in our queue to make a teak stock.
 - **Tick 2**: The skilling timer decrements to 1. During client input, we start interacting with the fishing spot. This deletes the teak stock command from our queue.
 - **Tick 3**: The skilling timer decrements to 0. During our turn, in interactions with npcs, we get a roll for a fish.

This rhythm repeats: notice that on **Tick 1** we crucially interrupted our fishing with a knife-log to set the skilling timer using the knife log rather than the fishing spot. We could have also started interacting with the fishing spot on **Tick 1**.

The knife-log used above set the skilling timer after the skilling timer had gone off. Most actions which can change the skilling tick similarly have this _after the skilling timer has gone off_ condition. One way to make sense of this is through an example: while interacting with a fishing spot every tick, the skilling tick change condition is passed every tick, so to make getting rolls possible, the skilling tick change must only happen _after_ the roll, which occurs on the skilling tick.

### Eating

Not all actions which change the skilling tick have this _after the skilling timer has gone off_ condition, though. 

Eating most food will add three ticks to the skilling tick. We'll case such food items _3t foods_. For example, if the skilling tick is in the past, eating a 3t food won't necessarily bring the skilling tick into the future; however, when the skilling timer is -1, eating (for example) a roe or caviar moves the skilling timer to 2, which is the same as effect as from a knife-log. Sometimes this is referred to as eating "continuing cycles". An example is shown below.

<div style="text-align:center"><img src="https://i.imgur.com/WayNBjJ.gif" alt='Cut-eat 3t barb fishing' width=500>

This method is mechanically similar to the previous 3t fishing with a knife-log example, with some additions.
 - **Tick 1**: The skilling timer decrements to -1. During client input, we eat a roe or caviar, which removes our interaction with the fishing spot and adds three to our skilling timer to make it -1+3=2. Later in client input, we process a fish into a roe or caviar. Later in client input, we start interacting with the fishing spot.
 - **Tick 2**: The skilling timer decrements to 1.
 - **Tick 3**: The skilling timer decrements to 0. During our turn, in interactions with npcs, we get a roll for a fish.

In this version of 3t fishing, on **Tick 1**, we also cut a fish in client input to produce more food to eat and some cooking experience. We also start interacting with the fishing spot on **Tick 1** rather than on **Tick 2**, even though either would work. An interesting variant of this method includes one more action: we pick up a fish on **Tick 1** (by interacting with a ground fish) then start interacting with the fishing spot on **Tick 2**. The result of this is that we have more fish to cut, giving more cooking experience.

Some food items, such as karambwans, add two ticks to the skilling tick, rather than three ticks like most other foods. A typical example of this kind of food is karambwans. This shorter delay can be used to our advantage to skill faster. We can't using karambwans to 2t skill since they can only be eaten at most once every three ticks. However, we can use karambwans to extend 3t fishing into 2.5t fishing, where we alternate between getting a roll in three ticks and a roll in two ticks.

<div style="text-align:center"><img src="https://i.imgur.com/hAIRd8S.gif" alt='2.5t barb fishing' width=500>

Below, we describe the actions shown in clip as text.
 - **Tick 1**: The skilling timer decrements to -1. During client input, we start making a teak stock, which removes our interaction with the fishing spot, sets our skilling timer to 2, and puts a weak command in our queue to make a teak stock.
 - **Tick 2**: The skilling timer decrements to 1. During client input, we start interacting with the fishing spot. This deletes the teak stock command from our queue.
 - **Tick 3**: The skilling timer decrements to 0. During our turn, in interactions with npcs, we get a roll for a fish.
 - **Tick 4**: The skilling timer decrements to -1. During client input, we eat a karambwan which sets the skilling timer to -1+2=1. Next in client input, we start interacting with the fishing spot.
 - **Tick 5**: The skilling timer decrements to 0.  During our turn, in interactions with npcs, we get a roll for a fish.

A cleverly timed second karambwan eat can be put into the 2.5t method to make skilling even faster. To do the method, we will eat the second karambwan two ticks after a roll, rather than one tick after like for the first karambwan. Unfortunately, this method cannot be used at barbarian fishing, which has a mechanic where the first interaction with the fishing spot will never produce a roll.

Here's the method shown for 2.33t willows. The method is called that because we get 3 rolls every 7 ticks, making the average number of ticks per roll equal to 2 1/3.

<div style="text-align:center"><img src='https://i.imgur.com/Nw2M8ty.gif' src="https://i.imgur.com/dG0kl78.gif" alt="Port Khazard's 2.33t method" width=500>

Below, we describe the actions shown in clip as text.
 - **Tick 1**: During client input, we start making a teak stock, which removes our interaction with the tree, sets our skilling timer to 2, and puts a weak command in our queue to make a teak stock.
 - **Tick 2**: The skilling timer decrements to 1. During client input, we start interacting with the tree. This deletes the teak stock command from our queue.
 - **Tick 3**: The skilling timer decrements to 0. During our turn, in interactions with objects, we get a roll for a log.
 - **Tick 4**: The skilling timer decrements to -1. During client input, we eat a karambwan which sets the skilling timer to -1+2=1. Next in client input, we start interacting with the tree.
 - **Tick 5**: The skilling timer decrements to 0.  During our turn, in interactions with objects, we get a roll for a fish.
 - **Tick 6**: The skilling timer decrements to -1. During client input, we remove our interaction with the tree.
 - **Tick 7**: The skilling timer decrements to -2. During client input, we start interacting with the tree. Next in client input, we start interacting with the tree. During our turn, in interaction with objects, we get a roll for a log.

### Flinching

Being attacked can effect the skilling tick. In more detail, while our auto-retaliate is on, we are not interacting with anything, and our skilling tick is in the past, being attacked will set our skilling tick to half of our attack speed, rounded down. This is called being _flinched_. This mechanic was likely designed for use in combat, but it is used in several optimal skilling methods.

With a 2 tick or 3 tick weapon, being flinched will make the skilling tick be the next tick. While being attacked every other tick, this can make the skilling tick be set to be the next tick every other tick, as shown in the example below.

<div style="text-align:center"><img src="https://i.imgur.com/BQR6DDl.gif" alt='2t prif teaks' width=500>

This method is 2t teaks, and was the optimal method for woodcutting experience and the pet before Fossil Island introduced a spot which allows for 1.5t teaks. The method involves a two tick rhythm, described in text below.
 - **Tick 1**: The skilling timer decrements to -1. In client input, we remove our interaction with the tree. During a rabbits turn, it attacks us and puts damage into our queue. During our turn, when our queue evalutes, we recieve the damage, which sets our skilling timer to 1 and makes us start interacting with the rabbit.
 - **Tick 2**: The skilling timer decrements to 0. In client input,  we start interacting with the tree. During our turn, in interactions with objects, we to get a roll a log. 

The interaction with the tree at the beginning of **Tick 1** can be removed in a number of ways, including via a no-movement path (as in the clip), log dropping, or dart fletching. Notice that we are at no point interacting with a rabbit during any skilling tick, which keeps us from shooting a dart.

Essentially the same method is also done for swordfish harpooning, as shown below.

<div style="text-align:center"><img src="https://i.imgur.com/5QyQZVc.gif" alt='2t pisc swordfish harpooning' width=500>

Interestingly, in this variant of the method, we don't need to make any clicks to cancel our interaction with the fishing spot. This is because harpoon fishing spots (like most others) force you to stop interacting with them after one tick before the skilling timer has gone off (except for when the skilling timer is 4, which allows for afk fishing). This force stop of our interaction with the fishing spot is the same reason why the rhythm for 3t swordfish fishing is different than the rhythm for 3t barbarian fishing: we still have to start interacting with the fishing spot on the skilling tick.

### Review of specialized mechanics

In this section, we'll review and collate the specialized mechanics which were introduced above. These are mechanics beyond just keeping track of the skilling tick which we should keep in mind while thinking about skilling methods.

The skilling timer delay can happen either after the action or on the action. Fishing, mining, and woodcutting actions happen on the skilling tick, and then the next interaction (while the skilling timer is negative) sets the skilling timer. However, combat and firemaking actions happen when the skilling timer is nonpositive, and the skilling timer delay happens on the same tick as the action. This difference is intrinsically notable but can also be combined to produce a method like PVP world 3t fishing, shown previously.

Inventory actions like making herb tar, wood stocks (via knife-logging), kebbit claws, or battlestaves (via celastrus bark) don't complete when we start them on the skilling tick. This is because the check to delay their skilling timer is whether the skilling timer is _nonpositive_. This is unlike fishing, mining, or woodcutting, where the check to delay their skilling timer is whether the skilling timer is _negative_. Another example of unusual behavior can be seen in snow gathering: there, the skilling timer is delayed by three whenever the skilling timer is less than two.

The first interaction can have different behavior than later interactions. We saw this with barbarian fishing, where we cannot get rolls during the first interaction with a fishing spot, even if the first interaction is during the skilling tick. Other examples of this behavior appear at fly fishing, pike fishing, and boulder mining in volcanic mine.

In fishing, sometimes when the skilling timer is nonnegative we are forced to stop interacting with the fishing spot after one tick. We saw this during 2t swordfish, where this mechanic made us not need to click off the fishing spot. Barbarian fishing, fly fishing, pike fishing do not have this mechanic at all, while lava eel fishing has a variant of it.

Most new skilling activities in the game unfortunately do not use the skilling tick. In fishing, the examples are Karambwan, infernal eel, minnow, and sacred eel fishing. In mining, examples include mining for runecrafting in Zeah.

This section has focused on the skilling tick, but there are other "named" ticks which function in much the same way. Particularly prominent examples are the alchemy tick (which several normal spellbook spells use), the eating tick (which control which ticks we can eat non-karambwan food), the karambwan eating tick, the pot sipping tick, and the bone bury tick. For an example of delaying these ticks, note that sipping a potion will block eating, karambwan eating, and potion sipping for three ticks. This is because sipping a potion delays the eating, karambwan eating, and the pot sipping tick. However, a non-karambwan food item will only the eating tick, so karambwans and potions can still be immediately consumed. The reason why karambwans can't be used to 2t at barbarian fishing is, in more detail, because karambwans delay the karambwan eating timer by three ticks, despite delaying the skilling timer by two ticks.

## Tick Manipulation III: Stalls

Stalls appear often during skilling, sometimes to block increased experience rates and sometimes to improve experience rates.

There are of course immediate examples of unhelpful stalls. Agility obstacles stall our character while we cross them. This means that no amount of ingenuity will ever speed up crossing an obstacle, as we're blocked during crossing. Another example is picking up a box trap, which produces a three tick stall. Being stalled here means that a trap can be reset in at fewest three ticks, which is only possible if we can set up a box trap on the turn that our stall ends. However, this isn't possible, so the fastest a trap can be reset in practice is in four ticks.

Another example where a stall slows us down is below. However, here, an understanding of mechanics saves us a tick compared to a naive method. Notice that our bank interface opens on the same tick that a stall ends.

<div style="text-align:center"><img src="https://i.imgur.com/us896Tz.gif" alt='Interface opening after stall' width=500>

We first process the bank click in client input, which sets the object we're interacting with to be the bank, then we process the spell click in client input, which does the spell action and produces a stall for three ticks. Note, the spell crucially does not remove our interaction. Three ticks later, the stall ends at the beginning of our turn, then the bank interface opens during 'interact with objects/items'. This saves a tick over casting the spell then clicking the bank on the tick the stall ends. Note that clicking the bank a tick before the stall ends will not be processed since the stall ends after client input.

### Stalls after movement

Some objects produce a one tick stall before fully executing our interaction with them after moving in the previous tick. For example, changing levels via most ladders in game is slowed down by one tick due to this stall. There are primarily two places where this stall appears in skilling: while woodcutting player-farmed trees or mining rocks. For example, when afk mining, this stall slows down experience when moving in between rocks: it forces us to take five ticks until a roll instead of the usual four ticks (one for movement, three for the skilling timer delay from mining).

This stall can be used to help us by giving up to two rolls on a skilling tick, rather than just one roll. Below we show a clip of 1.5t teaks on fossil island, which makes use of this stall.

<div style="text-align:center"><img src="https://i.imgur.com/LoHe2Ae.gif" alt='fossil island 1.5t teaks' width=500>

 - **Tick 1**: In client input, we start making herb tar, which removes our interaction with the tree, sets our skilling timer to 2, and puts a weak command in our queue to make a teak stock. Next in client input, we path to move, which deletes the teak stock command. During our turn, in movement, we move.
 - **Tick 2**: In client input, we start interacting the tree. During our turn, in interaction with objects, a stall starts.
 - **Tick 3**: During our turn, the stall ends. Later during our turn, in interaction with objects, we get a roll for a log.

Significantly, our interaction with the tree was paused when the stall began, so our interaction completes when the stall ends. This finishing of our interaction from **Tick 2** is the source of the additional roll. 

Unfortunately, this method can easily go wrong. If we interact with a tree two ticks after moving rather than one, the stall won't happen and we will lose out on one roll. Further, if we move on the tick after setting the skilling tick with (say) herb tar, we will be stalled when we would have interacted with the tree during the skilling tick, giving no rolls at all.

The same method is commonly done at sandstone and granite in the quarry. There we still can get two rolls, but the second roll only offers us a second chance for when the first roll fails. This is perhaps the only example in game of tick manipulation being used to increase the probability of successfully gathering a resource in a tick.

<div style="text-align:center"><img src="https://i.imgur.com/qTXZQgk.gif" alt='3 tick 4 granite' width=500>

### Stalls on successful rolls

Most mining rocks produce a one tick stall after successfully rolling a resource. The effects of this are most well known for gem mining, where herb tar is used to produce a resource every four ticks instead of the usual three. Iron, granite, and sandstone do not have this stall. The motherlode mine only has a one tick stall for the first resource gathered per click to interact.

### Stalls in cooking

Another example of an unhelpful stall can be found when cooking food besides raw beef, raw bear meat, and raw karambwans. The 1t cooking method doesn't work for other food items due to a stall which forces the first cook of these raw food items to take four ticks. However, since the stall only appears when there's more than one of the same raw food in our inventory, it's still possible to accelerate cooking these other food items by directly circumventing this problem. Indeed, we can 2t cook by only cooking with one raw food in our inventory at a time, which we show in the clip below.

<div style="text-align:center"><img src="https://i.imgur.com/3netQ5Y.gif" alt='2t cooking' width=500>

Below, we describe the actions shown in clip as text.
- **Tick 1**: In client input, we withdraw a shark from our bank. Next in client input, we start interacting with the stove. During our turn, in interaction with objects, we cook the shark.
- **Tick 2**: In client input, we start interacting with the bank. During our turn, in interaction with objects, an interface to our bank opens.

Every so often, in client input on **Tick 1**, we first deposit our inventory.

## Tick Manipulation IV: Movement and Pathing

There are two major aspects of movement that need to be understood for skilling. The first involves being able to keep track of the position of npcs and players across ticks, and the second involves controlling which tiles are obstructed or not. 

One example is moving the rabbits between trees at prifddinas teaks without disturbing their attack schedule, as shown below.

<div style="text-align:center"><img src="https://i.imgur.com/xbDfTia.gif" alt='Movement from the third tree at prif teaks' width=500>

We first drag the rabbit with lower ID (so whose turn is first) over to the desired tile, then unobstruct the tile by running over it so that the second rabbit can also reach the tile.

Another example is the rat setup for 2t swordfish. Below is an optimal strategy for this.

<div style="text-align:center"><img src="https://i.imgur.com/G3BlmCc.gif" alt='Rat setup at 2t swordfish' width=500>

We first gets the rats horizontal by running away from the coast then back, careful to walk to the side of the stack on the way back to keep the tile next to us obstructed. Second, we move to the side of the rat line up on the tick after being attacked. This makes the second (farther south) rat attack us two ticks after the first. Third and finally, we take a diagonal step toward the coast to leave undisturbed the rat set up. The diagonal step keeps them undisturbed since they both move to the only tile adjacent to our character that is one tile away from their former position.