# Migration guide
This guide lists general differences between version 0.9.8 and version 1.0.0+. It's not an exhaustive list; please refer to API documentation for more detailed information.

| **0.9.8-**									| **1.0.0+**			|
| :-------  									| :------- 				|
| `SMODS.INIT`									| Removed. Move all code inside `SMODS.INIT` to top-level scope |
| `SMODS.Deck`									| `SMODS.Back` 			|
| `SMODS.Sprite`								| `SMODS.Atlas` 		|
| `SMODS.Tarot, SMODS.Planet, SMODS.Spectral` 	| `SMODS.Consumable`	|
| `slug`										| `key` 				|
| `SMODS.Joker:new(name, slug, ...):register()` | `SMODS.Joker { key = 'key', name = 'Name', ... }` |
| `SMODS.findModByID('id')`						| `SMODS.current_mod` (available during load) / `SMODS.Mods['id']` |
| `loc_def()`									| `loc_vars()` - overhauled and more consistent interface |
| `G.localization.descriptions.Foo.bar = baz`   | <pre>function SMODS.current_mod.process_loc_text()<br>&emsp;    G.localization.descriptions.Foo.bar = baz<br>end</pre> |
# API Documentation
## General remarks
All Steamodded APIs are built on an Object Oriented Programming engine. As such, Steamodded objects share some common methods and parameters, described below.
## Creating an object
Create an object by calling a class, for example `SMODS.Joker`, with a single table parameter. Each class may have required fields and may provide some default values.

Your table must have a `key` field, which must be a unique string. Don't worry about collisions with other mods&mdash;your mod's prefix will always be prepended to `key`. With a few exceptions, you don't have to worry about this.
```lua
-- Skeleton for creating an object
SMODS.Class {
	key = 'key',
	other_param = 0,
	loc_txt = {
		-- ...
	},
}
``` 

## Common parameters
### `name`
Used by the game to identify certain objects, but Steamodded doesn't use it at all. You can ignore it.

### `loc_txt`
Most objects display a text description, and some objects need to display additional text in the collection and other places. The `loc_txt` field defines these pieces of text. You can find an in-depth explanation of `loc_txt` and other ways to load description strings on [this page](https://github.com/Steamodded/smods/wiki/Localization).

### `unlocked`
Sets the default unlock state of an object. If set to `false`, your object won't be obtainable until it's unlocked; make sure to implement an unlock condition.

### `discovered`
Sets the default discovery state of an object. If set to `true`, your object can be viewed in the collection without needing to find it in a run.

### `no_collection`
If set to `true`, this object will not show up in the collection.

### `config`
Put initial values for your object in `config`. Cards representing your object have an `ability` table, whose initial value is a copy of `config`, but can change during the game. Defaults to `{}` when not specified.
- Only specific keys are copied from `config`; define an `extra` table inside `config` to make sure your initial values aren't lost.
	```lua
	config = {
		extra = {
			custom_value = 10,
			another_value = 'something',
		},
	}
	``` 

### `prefix_config`
Defining this table gives you control over where prefixes should be added to keys you specify. The default behavior is to add a class prefix (if it exists) and your mod prefix.
- Supported keys:
	- `key`
	- `atlas`: Includes all atlas-related fields like `hc_atlas` and `lc_atlas` on suits and ranks
	- `shader`
	- `card_key`
	- `above_stake`
	- `applied_stakes`: Also supports options per index
- Each key can be set to a table or the value `false`. Effects:
	- `mod`: Setting this to `false` removes your mod prefix.
	- `class`: Setting this to `false` removes the class prefix.
	- If `false` is used as the value instead of a table, no prefixes are applied.
- Set `prefix_config = false` to apply no prefixes to any key.
- Example:
	```lua
	{
		prefix_config = {
			key = { mod = false },
			atlas = false, 
			applied_stakes = {
				[1] = { mod = false },
				[3] = false,
			},
		},
	}
	```

### `dependencies`
A list of one or more mod IDs. Your object will only be loaded when all specified mods are present. This is useful for cross-mod compatibility, but not for dependencies that are required for your mod to function properly. In that case, add a dependency to your mod header.

### `display_size`
Changes the display size of cards by scaling them by a factor relative to pixel size (default: `{ w = 71, h = 95 }`). 

### `pixel_size`
Changes how large the sprite of this card is considered, useful for smaller sprites like Half Joker (default: `{ w = 71, h = 95 }`).

## Taking ownership
You may need to modify vanilla objects or objects from another mod. Use the `take_ownership` function to modify an existing object; then, you can use all of Steamodded's API functions on it. Each key-value pair of the provided table overwrites the object's value, while the rest of the object is left intact. Objects you take ownership of have your mod's badge added to them, unless you suppress this with the `silent` argument.
```lua
SMODS.Joker:take_ownership('joker', -- object key (class prefix not required)
    { -- table of properties to change from the existing object
	cost = 5,
	calculate = function(self, card, context)
		-- more on this later
	end
    },
    true -- silent | suppresses mod badge
)
```

## API functions
Each class's API functions are explained on that class's Wiki page. The following lists parameter names common to these API functions.
| Identifier 	| Meaning 		|
| :--------- 	| :--------- 	|
| `self`		| The table this method is defined on. This is generally a prototype object that can't keep track of state. |
| `card`		| An instance of the game's `Card` class. Keeps track of state in an `ability` field. |

# Calculating Objects
Objects can have a `calculate` function defined on them to allow them to react to different events happening in the game. The main time for this to happen is during **scoring**, but there are other contexts that the game calculates effects. There is a full list of contexts available at the end of this guide.

## Creating a Calculate Function
To create a calculate function on your object, add this code to your object definition.
```lua
calculate = function(self, card, context)
	-- calculation code goes in here
end,
```
In this function, `self` refers to the center object that is used to create cards of this type, `card` refers to the actual card you are calculating on, and `context` refers to the table of values sent to the calculate. You will need to use `card` and `context` to calculate any effects within your object.
There are three steps to writing the calculation code within your function.
1. Checking for the correct context
2. Any logic/effects you want to do
3. Returning any effects that need to be handled

### Step 1: Context Checks
All code within your calculate function should be inside a **context check**. This statement will ensure that your effect will open happen in the timing that you want it to happen. There is a full list of contexts at the end of this guide, but here are some common ones you might want to use.
- `if context.joker_main then` The main scoring timing of jokers
- `if context.cardarea == G.play and context.main_scoring` The main scoring of played cards *(used for modifiers to cards)*
- `if context.before then` For effects that happen in the scoring loop but before anything is scored
- `if context.final_scoring_step then`For effects that modify the score after all cards have been scored
- `if context.cardarea == G.play and context.repetition then` For adding repetitions to played cards

### Step 2: Logic/Effects
In this step, you need to add your actual logic for calculation. Values that are defined in your object's config can be access by using `card.ability.`, for example, here is some code that increases the chips an object will give out by 10 every time the effect is evaluated.
```lua
card.ability.extra.chips = card.ability.extra.chips + card.ability.extra.chip_gain
```

### Step 3: Returning Effects
Any effects that you want to be evaluated will need to be returned in a table. To score the chips from Step 2, you should use this return table.
```lua
return {
	chips = card.ability.extra.chips
}
```
There are a range of different keys that you can return in this table.
- `chips`,`mult`,`xmult`, `dollars` - scores these values *(automatically adds a message to the card that is being scored)*
- `swap` - swaps current chips and mult values with each other
- `level_up` - levels up the played hand by the number returned
- `saved` - used during `context.end_of_round` to prevent game over
- `message` - used to return a custom message
	- will automatically be put on the scored card unless `message_card` is also returned
	- colour of message background will be `G.C.FILTER` unless `colour` is returned
	- sound will be `generic1` unless `sound` is returned
- `func` - return a function to be called at the correct timing *(advanced)*
- `extra` - an extra table set out the same as this one *(advanced)*

If you do not want the default messages to be shown when adding `chips` etc. there are 2 ways around this
1. Use `remove_default_message = true` in your return table to ignore the message completely
2. Use `chip_message = 'text'` to change the text *(this maintains default timings)*
___
Putting all this together, we get the following function that will increment chips each time the object is scored, and then score the new amount of chips.
```lua
calculate = function(self, card, context)
	if context.joker_main then
		card.ability.extra.chips = card.ability.extra.chips + card.ability.extra.chip_gain
		return {
			chips = card.ability.extra.chips
		}
	end
end
```
Here is another example for a Joker that increases the amount of mult it gives when you play a Flush.
```lua
calculate = function(self, card, context)
	-- Check if we have played a Flush before we do any scoring and increment the chips
	if context.before and next(context.poker_hands['Flush']) then
		card.ability.extra.mult = card.ability.extra.mult + card.ability.extra.mult_gain
		return {
			message = 'Upgraded!',
			colour = G.C.RED
		}
	end
	-- Add the chips in main scoring context
	if context.joker_main then
		return {
			mult = card.ability.extra.mult
		}
	end
```

> [!NOTE]
> If you want to evaluate effects outside of the return table, use `SMODS.calculate_effect({effects}, card)`,
> where `{effects}` is a table like the return table of a calculate function, and `card` is the card that is being evaluated.

## Contexts
Detailed here is a list of all contexts that are sent to calculate functions, as well as a unique identifier to use to reference them. As each context is sent to different areas, use the following logic statement to make sure you are calculating in the intended area.
```lua
if context.cardarea == G.play then -- replace G.play with G.jokers/G.hand as appropriate
```
---
### Main Scoring Loop
This context is used for effects that happen before scoring begins. 
```lua
if context.before and context.cardarea == G.play then
{
	cardarea = G.jokers, -- G.play, G.hand, (G.deck and G.discard optionally enabled)
	full_hand = G.play.cards,
	scoring_hand = scoring_hand,
	scoring_name = text,
	poker_hands = poker_hands,
	before = true
}
```
---
This context is used for the effects from playing cards when they are scored. This is used for enhancements, editions and seals (or any other card modifiers)
```lua
if context.main_scoring and context.cardarea == G.play then
{
	cardarea = G.play, -- G.hand, (G.deck and G.discard optionally enabled)
	full_hand = G.play.cards,
	scoring_hand = scoring_hand,
	scoring_name = text,
	poker_hands = poker_hands,
	main_scoring = true
}
```
---
This context is used for triggering joker effects on playing cards. 
```lua
if context.individual and context.cardarea == G.play then
{
	cardarea = G.play, -- G.hand, (G.deck and G.discard optionally enabled)
	full_hand = G.play.cards,
	scoring_hand = scoring_hand,
	scoring_name = text,
	poker_hands = poker_hands,
	individual = true,
	other_card = card
}
```
---
This context is used for adding repetitions to playing cards. 
```lua
if context.repetition and context.cardarea == G.play then
{
	cardarea = G.play, -- G.hand, G.deck and G.discard optionally enabled
	full_hand = G.play.cards,
	scoring_hand = scoring_hand,
	scoring_name = text,
	poker_hands = poker_hands,
	repetition = true,
	card_effects = effects -- this is the table of effects that has been calculated
}
```
---
This context is used for triggering editions on jokers before they score. *(used for chips/mult)*
```lua
if context.pre_joker then
{
	cardarea = G.jokers,
	full_hand = G.play.cards,
	scoring_hand = scoring_hand,
	scoring_name = text,
	poker_hands = poker_hands,
	edition = true,
	pre_joker = true
}
```
---
This context is used for triggering normal scoring effects on **jokers**. 
```lua
if context.joker_main then
{
	cardarea = G.jokers,
	full_hand = G.play.cards,
	scoring_hand = scoring_hand,
	scoring_name = text,
	poker_hands = poker_hands,
	joker_main = true,
}
```
---
This context is used for triggering joker effects from other jokers. 
```lua
if context.other_joker then
{
	full_hand = G.play.cards,
	scoring_hand = scoring_hand,
	scoring_name = text,
	poker_hands = poker_hands,
	other_joker = card, --changes to other_consumeable as it iterates over consumeables
}
```
---
This context is used for triggering editions on jokers after they score. *(used for xmult)*
```lua
if context.post_joker then
{
	cardarea = G.jokers,
	full_hand = G.play.cards,
	scoring_hand = scoring_hand,
	scoring_name = text,
	poker_hands = poker_hands,
	edition = true,
	post_joker = true
}
```
---
This context is used for any effects after cards have been calculated but before the score is totalled. 
```lua
if context.final_scoring_step and context.cardarea == G.play then
{
	cardarea = G.jokers, -- G.play, G.hand, (G.deck and G.discard optionally enabled)
	full_hand = G.play.cards,
	scoring_hand = scoring_hand,
	scoring_name = text,
	poker_hands = poker_hands,
	final_scoring_step = true,
}
```
---
This context is used for marking cards to be destroyed. 
```lua
if context.destroy_card and context.cardarea == G.play then
{
	cardarea = G.play, -- G.hand, (G.deck and G.discard optionally enabled)
	full_hand = G.play.cards,
	scoring_hand = scoring_hand,
	scoring_name = text,
	poker_hands = poker_hands,
	destroy_card = card,
	destroying_card = card -- only when calculating in G.play
}
```
> [!TIP]
> Your return table to destroy a card without a message should look like
> `return { remove = true }`
---
This context is used for effects on removing cards. 
```lua
if context.remove_playing_cards then
{
	cardarea = G.jokers, -- G.play, G.hand, (G.deck and G.discard optionally enabled)
	scoring_hand = scoring_hand,
	remove_playing_cards = true,
	removed = cards_destroyed
}
```
> [!WARNING]
> This is also called after discarding a hand without the `scoring_hand` part of the context.
---
This context is used for effects after scoring. 
```lua
if context.after and context.cardarea == G.play then
{
	cardarea = G.jokers, -- G.play, G.hand, (G.deck and G.discard optionally enabled)
	full_hand = G.play.cards,
	scoring_hand = scoring_hand,
	scoring_name = text,
	poker_hands = poker_hands,
	after = true,
}
```
---
### Other Contexts
This context is used for trigger effects on playing a hand debuffed by the blind. 
```lua
if context.debuffed_hand and context.cardarea == G.play then
{
	cardarea = G.jokers, -- G.play, G.hand, (G.deck and G.discard optionally enabled)
	full_hand = G.play.cards,
	scoring_hand = scoring_hand,
	scoring_name = text,
	poker_hands = poker_hands,
	debuffed_hand = true,
}
```
---
This context is used for end of round effects. 
```lua
if context.end_of_round and context.cardarea == G.jokers then
{
	cardarea = G.jokers, -- G.hand, (G.deck and G.discard optionally enabled)
	end_of_round = true,
	game_over = game_over -- true or false
}
```
---
This context is used for effects on cards from jokers at the end of the round. 
```lua
if context.end_of_round and context.individual then
{
	cardarea = G.hand, -- (G.deck and G.discard optionally enabled)
	end_of_round = true,
	individual = true,
	other_card = card
}
```
---
This context is used for repetitions on cards at the end of the round. 
```lua
if context.end_of_round and context.repetition then
{
	cardarea = G.hand, -- (G.deck and G.discard optionally enabled)
	end_of_round = true,
	repetition = true,
	other_card = card,
	card_effects = effects -- table of effects for the card
}
```
---
This context is used for effects when the blind is started. 
```lua
if context.setting_blind then
{
	cardarea = G.jokers, -- G.hand, (G.deck and G.discard optionally enabled)
	setting_blind = true,
	blind = G.GAME.round_resets.blind
}
```
---
This context is used for effects before discarding. 
```lua
if context.pre_discard and context.cardarea == G.play then
{
	cardarea = G.jokers, -- G.hand, (G.deck and G.discard optionally enabled)
	full_hand = G.hand.highlighted,
	pre_discard = true,
	hook = hook -- true when 'The Hook' discards cards
}
```
---
This context is used for effects on discarded cards. 
```lua
if context.discard then
{
	cardarea = G.jokers, -- G.hand, (G.deck and G.discard optionally enabled)
	full_hand = G.hand.highlighted,
	discard = true
	other_card = G.hand.highlighted[i]
}
```
> [!TIP]
> Return `{ remove = true }` to destroy cards in this context.
---
This context is used for effects after opening a booster. 
```lua
if context.open_booster then
{
	cardarea = G.jokers, -- G.hand, (G.deck and G.discard optionally enabled)
	open_booster = true,
	card = self -- the booster pack being opened
}
```
---
This context is used for effects after skipping a booster. 
```lua
if context.skipping_booster then
{
	cardarea = G.jokers, -- G.hand, (G.deck and G.discard optionally enabled)
	skipping_booster = true,
}
```
---
This context is used for effects after buying a card. 
```lua
if context.buying_card then
{
	cardarea = G.jokers, -- G.hand, (G.deck and G.discard optionally enabled)
	buying_card = true,
	card = self -- the card being bought
}
```
The card being bought also calculates itself with context `{ buying_card = true, card = c1 }`.

---
This context is used for effects after selling a card. 
```lua
if context.selling_card then
{
	cardarea = G.jokers, -- G.hand, (G.deck and G.discard optionally enabled)
	selling_card = true,
	card = card -- the card being sold
}
```
The card being sold also calculates itself with context `{ selling_self = true }`.

---
This context is used for effects after rerolling the shop. 
```lua
if context.reroll_shop then
{
	cardarea = G.jokers, -- G.hand, (G.deck and G.discard optionally enabled)
	reroll_shop = true,
}
```
---
This context is used for effects after leaving the shop. 
```lua
if context.ending_shop then
{
	cardarea = G.jokers, -- G.hand, (G.deck and G.discard optionally enabled)
	ending_shop = true,
}
```
---
This context is used for effects after drawing the first hand of a blind. 
```lua
if context.first_hand_drawn then
{
	cardarea = G.jokers, -- G.hand, (G.deck and G.discard optionally enabled)
	first_hand_drawn = true,
}
```
---
This context is used for effects after drawing a hand. 
```lua
if context.hand_drawn then
{
	cardarea = G.jokers, -- G.hand, (G.deck and G.discard optionally enabled)
	hand_drawn = true,
}
```
---
This context is used for effects after using a consumable 
```lua
if context.using_consumeable then
{
	cardarea = G.jokers, -- G.hand, (G.deck and G.discard optionally enabled)
	using_consumeable = true,
	consumeable = card, -- the consumable being used
	area = G.consumeables, -- the area the card was used from (could be G.shop_jokers, G.pack_cards)
}
```
---
This context is used for effects after skipping a blind. 
```lua
if context.skip_blind then
{
	cardarea = G.jokers, -- G.hand, (G.deck and G.discard optionally enabled)
	skip_blind = true,
}
```
---
This context is used for effects after adding a card to your deck. 
```lua
if context.playing_card_added then
{
	cardarea = G.jokers, -- G.hand, (G.deck and G.discard optionally enabled)
	playing_card_added = true,
	cards = cards -- the cards being added (sometimes is true when used from vanilla items)
}
```
---
This context is used for applying quantum enhancements. 
```lua
if context.check_enhancement then
{
	cardarea = G.jokers, -- G.hand, (G.deck and G.discard optionally enabled)
	check_enhancement = true,
	other_card = card,
	no_blueprint = true
}
```
---
This context is used for effects after a Joker has triggered. 
```lua
if context.post_trigger then
{
	cardarea = G.jokers, -- G.hand, (G.deck and G.discard optionally enabled)
	post_trigger = true,
	blueprint_card = context.blueprint_card, -- if applicable
	other_card = card, -- the card that triggered
	other_context = context, -- the context the trigger happened on
	other_ret = ret -- the return table from the trigger
}
```




> [!NOTE]
> All calculation of jokers that evaluates no secondary card is called with `main_eval = true` in the context table.
> All joker triggers are called with again when the joker is **retriggered** with
> `retrigger_joker = retrigger_card` added to the context table.
# The Balatro Event System

You can make a new event and dispatch it like so 

```lua
G.E_MANAGER:add_event(Event({}))
```

Which is an event which does nothing :tada:

However, you probably want to do something in this event. Let's cover some of the properties you can pass to the event handler


# Properties
These can be passed in a table to the Event function. 

trigger - string:
- "immediate" (default) - Runs as soon as possible
- "after" - Will run after a set amount of time. Also see delay
- "condition" - Will not finish until the condition is met. For most cases, you probably just want to use immediate with a check (see func). See ref_table, ref_value and stop_val for how to set this.
- "ease" - Used to interpolate values. Useful for score incrementing or movement. Also see ref_table, ref_value and ease_to for how to set this.
- "before" - Will run immediately, but if it doesn't complete before the set amount of time, it will be cancelled. Also see delay

blocking - boolean: Whether or not this event may block other events. Default is true

blockable - boolean: Whether or not this event may be blocked by other events is true

func - function: The function to call for the event. When this is called depends on the trigger type.
    This function takes no arguments. 
    It returns whether it completed. If true, then the event is over, if false, then it will be called again next time the event manager processes events (the next frame).
    It's behavior for each trigger is as follows:
- immediate - Called when events are next processed
- after - Called when the time is over
- condition - Behaves like immediate. Providing a function will overwrite the default condition behaviour. If you want to do stuff conditionally and use a function, then just do your check and return false if the condition is not satisfied.
- ease - Called each frame with the current value of the ease. The function defintion is a bit different here. The first argument is the current value of the ease. The return value is then set to the value stored in the table. The default function just returns the current value.
- "before" - Called immediately. If the event does not complete after the delay, then it is automatically canceled.

delay - number: The time to take, in seconds. Used for after, ease and before. Note this value is effected by the game speed option. Default is 0.

ref_table - table: The table to check for the condition. Used for condition and ease. No default

ref_value - string: The key in ref_table to get the value. Used for condition and ease. No default.

stop_val - any: The value to stop at. Used for condition. No default.

ease_to - number: The value to end at. Used for ease. No default.

ease - string: The type of ease to use. Used for ease. Default is "lerp". Can be any of "lerp", "elastic" or "quad".

# Examples
Great, now you've been info dumped, let's show you some more practical examples.

The below code will run the function `print("Hello, world!")` as soon as the event is processed. Make sure to remember to return true at the end of the function, otherwise it will be called again next frame.

```lua
G.E_MANAGER:add_event(Event({
    func = function() 
        print("Hello, world!")
        return true 
    end
}))
```

Now of course, you might be wondering, why wouldn't I just call the function directly? Well, the real power of this is when used in tandom with other events. If you run this event in the  calculate_joker context, you'll notice that it won't run right away, but only after the joker(s) before it are done doing their stuff. 

If you remember from earlier, there are blockable and blocking properties on events. This is what allows this to happen. The previous calculations are blocking, and your calculation is blockable, so it will wait until the blocking events are done before running.

This also works for after events. For example, if you want to run the function after 5 seconds, you can do this:

```lua
G.E_MANAGER:add_event(Event({
    trigger = "after", 
    delay = 5, 
    func = function() 
        print("Hello, world!") 
        return true 
    end
}))
```

(you may notice this might be less than 5 seconds. This is because the delay is effected by the game speed option)

Also note that this event will block for those 5 seconds, so other events will also wait your 5 seconds, just like the immediate event.

Now, let's say you want to block until a certain condition is met. You can do this by checking your condition, and returning false if it is not met. For example, if you want to wait until the player has 2 hands left, you can do this:

```lua
G.E_MANAGER:add_event(Event({
    func = function() 
        if G.GAME.current_round.hands_left ~= 2 then
            return false
        end
        print("Hello, world!")
        return true
    end,
    blocking = false
}))
```

Note the blocking false I made. This may be nessicary if what you are waiting for is blockable (you can lock up the game if you're not careful).

Now let's get to the last main type. Easing. This is used for interpolating values. For example, if you want to increase the round score by 1000 over 5 seconds, you can do this:

```lua
G.E_MANAGER:add_event(Event({
    trigger = "ease",
    delay = 5,
    ref_table = G.GAME,
    ref_value = "chips",
    ease_to = G.GAME.chips + 1000,
}))
```

and with that, you should have everything you need to use events. However, I'll cover some more stuff just in case you want more.

# Further Reading
So far we have just been adding events to the base queue. However, there are a few other queues for us to use.

## Queues
When calling G.E_MANAGER:add_event, you can pass in a second argument for the queue. This is string corresponding to the queue type (default is "base"). The queues are as follows:

- base - The default queue. This is where most events should go.
- other - Was once used for displaying a video in the demo. Since it's not used, it makes for a prime candidate for putting stuff in, if you want to use blocking but not block the base queue.

The rest are all mostly for super specific things, but are documented here for completeness.

- unlock - This queue is for when you unlock new stuff. This allows it to keep showing new items when you close the last one.
- tutorial - This is used for the tutorial. Allows it to manage some of it's stuff. Also let's all the tutorial stuff be cleared at once.
- achievement - This is used for unlocking achievements. Allows the ui to show one at a time.

add_event also has a third argument, front. This is a boolean that will put the event at the front of the queue. Useful to block stuff already in the queue.

## Clearing the queue
The event manager also has another function you may use.
G.E_MANAGER:clear_queue(queue, exception)

Passing it with no options will clear all the queues. If a queue is passed, it will clear that queue. If an exception is passed, it will not clear all queues, except the one passed. You can avoid being cleared by setting the no_delete property to true.

## Repeating on a timer
Here is an example I made to run an event every 5 seconds. This does use a bit of hacking to get working, but I figured it might be handy.

```lua
local event
event = Event {
    blockable = false,
    blocking = false,
    pause_force = true,
    no_delete = true,
    trigger = "after",
    delay = 5,
    func = function()
        print("Hi mom!")
        event.start_timer = false
    end
}
G.E_MANAGER:add_event(event)
```

This works by storing a reference to the event, then setting the start_timer property to false. This will cause the event to think it never started the timer, and as such restart it after running the event, causing it to repeat every 5 seconds.

This event also set's some properties to make sure it doesn't get block, block other events, and run even if the game is paused. If you want to make sure you can stop this event, then you could add some check for some variable then return if it's true or smth.


## More properites
These are some more event properties you can use, but aren't really that useful in most situations.

start_timer - boolean: If true, then it will use the starting time of the event being created. Otherwise this will start them timer the first time the event is processed. Default is false.

no_delete - boolean: If true, then when clearing the event queue, this event will not be deleted. You probably don't want to use this unless you want to have an event that never stops. Default is false.

force_pause - boolean: If true, then the event will run even if the game is paused. This only has an effect if your command was created when the game wasn't paused. Default is false.

timer - string: The name of the timer to use. Default is "REAL" of created while paused (or with force_pause) otherwise "TOTAL". Can take any of the keys in G.TIMERS.

***
*Guide written by WilsontheWolf*
_This guide is missing any contexts added by RetriggerAPI_

# Score Evaluation

The process of score evaluation in Balatro is quite linear.
There are well-defined stages that go one after another, but the actual implementation is somewhat convoluted.
Perhaps one of the better ways to understand most of this process is by reading the code of [Divvy's Simulation](https://github.com/DivvyCr/Balatro-Simulation/blob/main/src/Engine.lua).
It's a perfect replication of the game's score evaluation but rewritten in a clearer manner.
Nevertheless, the rest of this guide explains the high-level details of this process.

## Set-Up

Once you play a hand, the game must first determine what cards are actually scoring.
For instance, if you play `A228`, only the pair of `2s` will be scored (unless you have the Splash joker!)

For this, the game uses `G.FUNCS.get_poker_hand_info(..)`, which takes an array of Card objects and returns a tuple of five values in this order:

 - The name of the play (eg. 'High Card', 'Pair', or 'Straight')
 - The localized name of the play
 - A table mapping play names (eg. 'High Card') to arrays of Card objects; it contains *all possible plays* out of the played hand, so a 'Two Pair' play will also have at least two 'Pair' values.<br>
   Eg. `poker_hands["Pair"] == { {2, 2}, {Q, Q} }`, but instead of numbers and letters, there will be Card objects.
 - An array of *scoring* Card objects (eg. only the pair of `2s` out of `A228`)
 - The special name of the play (eg. 'Royal Flush') or its standard name if no special name exists

```lua
-- Example usage:
local hand_name, localized_hand_name, poker_hands, scoring_hand, customised_hand_name = G.FUNCS.get_poker_hand_info(..)
```

Now, as I alluded, the scoring cards may be different in special circumstances, such as when 'Splash' is present.
Therefore, if it *is* present, the game adds all cards to scoring cards.
(It also adds any Stone cards during this process, even if no 'Splash' is present)

## Check for Blind Debuffs

The game will now check whether the played hand is actually debuffed by the blind.
For instance, if you play fewer than five cards under the Psychic.

If the hand *is debuffed*, then the game will calculate the effects of all jokers with the special `context` argument containing `debuffed_hand = true`.
In the vanilla game, only 'Matador' actually does anything with this.

If the hand is *not debuffed*, then the game will carry out its full score evaluation, explained further below.

## The Context

First, however, it is important to understand the `context` argument that is used throughout the whole process.
The `context` is just a special argument that is passed to all objects that influence score evaluation.

### Bare-Minimum Context

All context arguments will always contain the following:

 - `cardarea`: which CardArea is being evaluated (either `G.jokers`, `G.hand`, or `G.play`)
 - `full_hand`: an array of all played Card objects (incl. non-scoring)
 - `scoring_hand`: an array of all scored Card objects
 - `scoring_name`: the name of the play (eg. 'High Card', or 'Straight')
 - `poker_hands`: the table mapping play names to Card objects (explained earlier)

```lua
-- Example minimal context:
local minimal_context = {
   cardarea = G.jokers,
   full_hand = G.play.cards,
   scoring_hand = scoring_hand, -- from G.FUNCS.get_poker_hand_info(..)
   scoring_name = hand_name,    -- from G.FUNCS.get_poker_hand_info(..)
   poker_hands = poker_hands    -- from G.FUNCS.get_poker_hand_info(..)
}
```

### More Context

Lastly, the context is further extended with particular flags and/or data, which signify  and help during the different stages of score evaluation.
One of these flags was mentioned earlier &ndash; `debuffed_hand = true` &ndash; so the full context would look like this:

```lua
-- Example context:
local context = {
   cardarea = G.jokers,
   full_hand = G.play.cards,
   scoring_hand = scoring_hand, -- from G.FUNCS.get_poker_hand_info(..)
   scoring_name = hand_name,    -- from G.FUNCS.get_poker_hand_info(..)
   poker_hands = poker_hands,   -- from G.FUNCS.get_poker_hand_info(..)
   debuffed_hand = true
}
```

### The Purpose of Context

Context enables the strict order of evaluation that Balatro relies upon.
Consider that upgrade jokers like 'Hiker' and 'Green Joker' are evaluated before anything else &ndash; it would be confusing otherwise.
Similarly, the 'DNA' joker must copy a card up-front.
Then, there are also jokers that do something for each scored card like 'Hiker' and 'Odd Todd', as opposed to jokers that do something after all cards were evaluated like 'Green Joker' and 'Hologram'.

Notice how some jokers appear multiple times?
Each time, there is a different *context*.
This is why `context` is vital.

On top of that, consider jokers like 'Blueprint' and 'Brainstorm'.
They must somehow know what joker they are replicating, so the `context` will also contain data (in this case, it would be `min_context + {other_joker = JOKER_OBJ}`)

## Stages of Evaluation

Hence, there are multiple stages within score evaluation:

 1. **Before** Stage<br>
   For any jokers that need to do something *before* score evaluation.<br>
   Sets `context.cardarea = G.jokers` and `context.before = true`
 2. **Score** Initialisation Stage<br>
   Sets chips and mult to those associated with the current hand level.
 3. **Blind** Effects Stage<br>
   For any blind effects like 'Flint' halving initial chips and mult.<br>
   This is handled by `Blind:modify_hand(..)` in game.
 4. **Scoring-Cards** Evaluation Stage<br>
   Evaluates each Card in `context.scoring_hand`, with `context.cardarea = G.play`.<br>
   See 'Card Evaluation' section below.
 5. **Held-Cards** Evaluation Stage<br>
   Evaluates each Card in `G.hand.cards`, with `context.cardarea = G.hand`.<br>
   See 'Card Evaluation' section below.
 6. **Global** Joker Effects Stage<br>
   For any jokers that do something *after* all cards have been evaluated.<br>
   Sets `context.cardarea = G.jokers` and `context.joker_main = true`
 7. **Consumable** Effects Stage<br>
   For any consumable effects (due to 'Observatory' for example).<br>
   This is a very short and custom stage.
 8. **Deck** Effects Stage<br>
   For any deck effects like 'Plasma' merging chips and mult.<br>
   This is handled by `Back:trigger_effect(..)` in game, with `context.final_scoring_step = true`
 9. Card **Destruction** Stage<br>
   Self-explanatory.
 10. **After** Stage<br>
    For any jokers that need to do something *after* a hand is played like 'Loyalty Card.<br>
    Sets `context.cardarea = G.jokers` and `context.after = true`
   
## Card Evaluation

Stages **4** and **5** go through each scored/held card and do the following:

 0. If the card is debuffed, do nothing.
 1. Collect repetitions from seals
   Sets `context.other_card = [CARD]` and `context.repetition = true` and `context.repetition_only = true`
 2. Collect repetitions from jokers
   Sets `context.other_card = [CARD]` and `context.repetition = true`
 3. For each collected repetition:
   - Evaluate the Card object via `eval_card([CARD], context)`
   - For each joker, evaluate joker effects via `[JOKER]:calculate_joker(context)`
    Sets `context.other_card = [CARD]` and `context.individual = true`
   - The return values of the above evaluations comprise a table that contains some fields described below.

```lua
-- Table with possible fields, returned from each card evaluation above:
{
  chips = 0,  -- Chips to add
  mult = 0,   -- Mult to add
  x_mult = 1, -- Mult multiplier
  h_mult = 1, -- TODO

  message = nil, -- TODO

  -- TODO: Difference between 'dollars' vs 'p_dollars'?
  dollars = 0,   -- Dollars to add
  p_dollars = 0, -- Dollars to add

  extra.chip_mod = 0, -- Chips to add
  extra.mult_mod = 0, -- Mult to add
  extra.swap = nil,   -- Should swap chips and mult?
  extra.func = nil,   -- TODO

  -- Effects due to Card's edition:
  edition.chip_mod = 0,   -- Chips to add
  edition.mult_mod = 0,   -- Mult to add
  edition.x_mult_mod = 0  -- Mult multiplier
}
```

## Joker Evaluation

Stage **6** goes through each joker and applies its global effect, if any.
For each joker:

 1. Evaluate its edition via `eval_card([JOKER], context)`<br>
   Sets `context.edition = true`
 2. Evaluate its effect via `eval_card([JOKER], context)`<br>
   Sets `context.joker_main = true`
 3. Evaluate any replications of this joker (by jokers like 'Blueprint'); see below.<br>
   Sets `context.other_joker = [JOKER]`
   
```lua
-- Joker-on-Joker simplified code
-- Assume we are currently evaluating the effects of 'current_joker'

context.other_joker = current_joker

for _, another_joker in ipairs(G.jokers.cards) do
   -- Yes, we pass 'current_joker' as context to all other jokers.
   another_joker:calculate_joker(context)
end
```

## Destruction

Lastly, the game will check if any scored cards or jokers need to be destroyed.
For each scored card, the game will first check if any joker destroys it via:

```lua
[JOKER]:calculate_joker({destroying_card = [CARD], full_hand = G.play.cards})
```

Then, the game will check if any Glass cards break.
Any destroyed cards are saved in the array `cards_destroyed`.

Then, the game will go through all jokers again, checking if any jokers were triggered due to a card being destroyed:

```lua
eval_card([CARD], {cardarea = G.jokers, remove_playing_cards = true, removed = cards_destroyed})
```


# Other Evaluation

Jokers are also evaluated after other player actions, not just when a hand is played.
The two main actions are discarding cards and winning the round, which have dedicated sections below.
All other actions are listed in the 'Everything Else' section at the bottom, for quick reference.

## Discard

Once you discard a hand, the game also does multiple stages of evaluation:

 1. **Before** Stage (Held Cards)<br>
   Sets `context.pre_discard = true` and `context.full_hand = G.hand.highlighted` (ie. which cards will be discarded)<br>
   *Also*, it sets `context.hook = true` if 'The Hook' blind is active.
    1. Evaluates **each held card** with that context.
    2. Evaluates **each joker** with that context.
 2. **Discard** Stage<br>
   Sets `context.discard = true` and `context.full_hand = G.hand.highlighted` (ie. which cards are discarded)
    1. Evaluates **each held card** with that context.
    2. Evaluates **each joker** with that context and also `context.other_card = discarded_card` (ie. it's evaluated *for each discarded card*)<br>
      This checks if any joker returns `remove = true`, like 'Trading Card' in vanilla.
 3. Card **Destruction** Stage<br>
   Evaluates **each joker** with `context.cardarea = G.jokers`<br>
   Sets `context.remove_playing_cards = true` and `context.removed = cards_destroyed`

## End of Round

Once you win (or lose) a round, the game also does multiple stages of evaluation:

 1. **Game Over** Effects<br>
   Sets `context.end_of_round = true` and `game_over = true` if the player just lost the the game<br>
   This checks if any joker returns `saved = true`, like 'Mr. Bones' in vanilla.
 2. **Held-Cards** Evaluation Stage<br>
   Sets `context.cardarea = G.hand` and `context.end_of_round = true`<br>
   This follows the same steps described in the 'Card Evaluation' section above.
 
> [!NOTE]
> For **End of Round** effects that trigger once, your joker should use:<br>
> `if context.end_of_round and not context.repetition and not context.individual then ...`

## Everything Else

Whenever any of the events below take place, *each joker* will be evaluated via `[JOKER]:calculate_joker(context)`.
All changes to context are mentioned alongside the event:

 - Round-related events
   - **New Round**: sets `context.setting_blind = true` and `context.blind = G.GAME.round_resets.blind` (ie. the type of blind)<br>
    Used by 'Madness', 'Burglar', and others in vanilla
   - **Drawing First Hand**: sets `context.first_hand_drawn = true`<br>
    Used by 'DNA', 'Trading Card', and others in vanilla
   - **Skipping a Blind**: sets `context.skip_blind = true`<br>
    Used only by 'Throwback' in vanilla
 - Set-up modifications
   - **Adding a Playing Card**: sets `context.playing_card_added = true` and `context.cards = cards` (ie. which cards added)<br>
    Used only by 'Hologram' in vanilla
   - **Using a Consumable**: sets `context.using_consumeable = true` and `context.consumeable = card` (ie. which consumable)<br>
    Used by 'Fortune Teller', 'Glass Joker', and others in vanilla
   - **Selling a Joker/Consumable**: sets `context.selling_card = true` and `context.card = card` (ie. which card)<br>
    *Also*, if selling a joker, that joker is evaluated with `context.selling_self = true`<br>
    Used only by 'Campfire' and 'Luchador' in vanilla
 - Shop-related events
   - **Buying Anything**: sets `context.buying_card = true` and `context.card = card` (ie. a joker/consumable/card/voucher object)<br>
    *Also*, if buying a joker, that joker is evaluated with the same context as above.
   - **Opening a Booster Pack**: sets `context.open_booster = true` and `context.card = booster` (ie. which booster)<br>
    Used only by 'Hallucination' in vanilla
   - **Skipping a Booster**: sets `context.skipping_booster = true`<br>
    Used only by 'Red Card' in vanilla
   - **Rerolling Shop**: sets `context.reroll_shop = true`<br>
    Used only by 'Flash Card' in vanilla
   - **Leaving Shop**: sets `context.ending_shop = true`<br>
    Used only by 'Perkeo' in vanilla

***
*Guide written by Divvy and Eremel*
Welcome to the Steamodded wiki! The following pages form an assembly of documentation and guides for Steamodded.
# How to install Steamodded
## Step 1: Anti-virus setup
*Skip ahead to step 2 if you are not using any anti-virus software.*

Steamodded relies on a runtime code injector in order to modify the game. Because it functions similarly to a Trojan, it is often **incorrectly** flagged as malware by anti-virus systems. Rest assured that Lovely is **not malicious** in any way. It is fully open source, so you can convince yourself that it works exactly as promised and does nothing else. You can even build it yourself if you're still unsure. In order to get Lovely running properly, you will have to whitelist Balatro's installation folder in whatever anti-virus software you may be using. You may also need to temporarily disable real-time protection to avoid having files deleted while moving them around.
### Example: steps for Windows Defender
Differences may occur if you're using different software.
1. Navigate to the game's directory by right-clicking the game in Steam, hovering "Manage", and selecting "Browse local files". Copy the file path of this directory.
2. Open Windows Security and navigate to `Virus & threat protection > Manage settings`.
3. Disable `Real-time protection`.
4. Scroll down to `Add or remove exclusions` and confirm if prompted.
5. Add a folder exclusion. When an explorer window opens, paste the path you copied in step 1 into the address bar and confirm.

## Step 2: Installing Lovely
Now you're ready to install the **Lovely injector**. Please follow the installation instructions for your operating system [here](https://github.com/ethangreen-dev/lovely-injector?tab=readme-ov-file#manual-installation), then return to this page and continue with Step 3. If your browser is blocking your download, use Firefox instead.

## Step 3: Installing Steamodded
**If you previously installed Steamodded without Lovely,** you must first remove that installation by verifying your game files on Steam: `Library > Balatro > Properties > Installed Files > Verify integrity of game files`.

### Method (3a): Direct download
*This method requires no further tools. If you know what Git is, skip ahead to Method (3b).*
1. Click [here](https://github.com/Steamodded/smods/archive/refs/heads/main.zip) to download the latest source code. (*No up-to-date releases exist as of now. This guide will be updated when they do.*)
  > [!IMPORTANT]
  > We recently reworked some stuff in the latest commits that has caused incompatibility with lots of mods. Regular users probably want to download the [old-calc](https://github.com/Steamodded/smods/archive/refs/tags/old-calc.zip) version instead of the main one. Mod dev's should use the main commit to update their mods.
  > When following steps with this build, the folders will be named `smods-old-calc` instead of `smods-main`.
2. Extract the download zip file.
3. In your file explorer, navigate to Balatro's save directory: **Windows:** `%AppData%/Balatro`; **Mac:** `~/Library/Application Support/Balatro`; **Linux (WINE/Proton):** `~/.local/share/Steam/steamapps/compatdata/2379780/pfx/drive_c/users/steamuser/AppData/Roaming/Balatro`.
  > [!IMPORTANT]
  > On Windows systems, just paste `%AppData%/Balatro` into the explorer path bar or the run dialog to open the folder. `%AppData%` has special meaning and you do not need to manually replace it.
  > <details>
  > <summary>Click me for Example Video</summary>
  > 
  > [Screencast_20241231_162107.webm](https://github.com/user-attachments/assets/12b76bed-fb0b-4e49-ae57-4ca12b6f1727)
  > 
  > </details>

  > [!NOTE]
  > On Linux systems with a snap installation of Steam, the save directory is slightly different: `~/snap/steam/common/.local/share/Steam/steamapps/compatdata/2379780/pfx/drive_c/users/steamuser/AppData/Roaming/Balatro`
4. Create a folder named `Mods` if it doesn't already exist. Open your `Mods` folder.
5. Inside the extracted zip file, you should find a directory named `smods-main`. Move this interior folder into your `Mods` folder. Ensure there is more than a single folder inside.
6. To update Steamodded later, delete the `smods-main` folder and repeat steps 1-5.

### Method (3b): Using the command line
*This method requires [Git](https://git-scm.com/downloads). If you have completed Method (3a), please skip this step. Your installation is complete.*
1. Navigate to Balatro's save directory: **Windows:** `cd %AppData%/Balatro`; **Mac:** `cd ~/Library/Application Support/Balatro`; **Linux (WINE/Proton):** `cd ~/.local/share/Steam/steamapps/compatdata/2379780/pfx/drive_c/users/steamuser/AppData/Roaming/Balatro`.

2. Paste the following lines: 
```shell
mkdir Mods
cd Mods
git clone https://github.com/Steamodded/smods.git

```
3. To update Steamodded later, navigate back to the `smods` directory and run `git pull`.
# Localization
Steamodded offers multiple ways to load strings for in-game descriptions and other labels into the game. This page offers an overview of each option and when you should use them.

## Localization files (recommended)
This method follows the ways of the base game: instead of attaching descriptions directly to each object, all localization strings for the same language are combined into one file. Each such file should be found in the location `localization/locale.lua` relative to your mod's root directory. `locale` can be the key of any language found in Balatro itself or added through [SMODS.Language](https://github.com/Steamodded/smods/wiki/SMODS.Language).

- `default`: Used as a fallback when no text was found for the selected language.
- `de`: German
- `en-us`: English
- `es_419`: Spanish (Latin America)
- `es_ES`: Spanish (Spain)
- `fr`: French
- `id`: Indonesian
- `it`: Italian
- `ja`: Japanese
- `ko`: Korean
- `nl`: Dutch
- `pl`: Polish
- `pt_BR`: Portuguese
- `ru`: Russian
- `zh_CN`: Chinese (simplified)
- `zh_TW`: Chinese (traditional)

The following is a skeleton for the base structure of a localization file. Entries without content may be omitted. Each found localization string is copied into `G.localization` when the languages are matching.
```lua
return {
    descriptions = {
        Back={},
        Blind={},
        Edition={},
        Enhanced={},
        Joker={},
        Other={},
        Planet={},
        Spectral={},
        Stake={},
        Tag={},
        Tarot={},
        Voucher={},
    },
    misc = {
        achievement_descriptions={},
        achievement_names={},
        blind_states={},
        challenge_names={},
        collabs={},
        dictionary={},
        high_scores={},
        labels={},
        poker_hand_descriptions={},
        poker_hands={},
        quips={},
        ranks={},
        suits_plural={},
        suits_singular={},
        tutorial={},
        v_dictionary={},
        v_text={},
    },
}
```
### Adding your descriptions
A localization file containing a single description may look something like this:
```lua
return {
    descriptions = {
        -- this key should match the set ("object type") of your object,
        -- e.g. Voucher, Tarot, or the key of a modded consumable type
        Joker = {
            -- this should be the full key of your object, including any prefixes
            j_mod_joker = {
                name = 'Name',
                text = {
                    'This is the first line of this description',
                    'This is the second line of this description',
                },
                -- only needed when this object is locked by default
                unlock = {
                    'This is a condition',
                    'for unlocking this card',
                },
            },
        },
    },
}
```

## `loc_txt`
Alternatively, you can define a `loc_txt` table on each object to create its description. This is more limited than the above option because it does not allow creating multiple descriptions for the same object or strings that don't belong to an object at all. The effort of maintaining translations of your mod while using this method is significantly higher. 

You can provide a single table for every language, or provide a subtable for each language. If the currently selected language isn't provided, the `default` subtable, the English (`'en_us'`) subtable, or `loc_txt` itself will be used as defaults, in that order. Refer to your object's documentation for what fields need to go in `loc_txt`.

The following are examples of valid `loc_txt` tables:
-
```lua
{ name = 'Name', text = { 'This is example text' } }
```
-
```lua
    {
        ['en-us'] = {
            name = 'Name',
            text = { 'Example', 'text', 'on', 'five', 'lines'},
        },
        ['fr'] = {
            -- French translation
        },
        ['nl'] = {
            -- Dutch translation
        },
        ['default'] = {
            -- a different default text
            -- leave this empty to allow
            -- custom languages to provide
            -- their own translation
        }
    }
```
-
```lua
{
    ['en-us'] = {
        -- Sometimes, more text is required than just a name and a description
        label = 'Label',
        name = 'Name',
        text = { 'A moderately long', 'description of', 'your effect.' },
    }
}
```
-
```lua
{
    -- The simplest option: use a single table
    label = 'Label',
    name = 'Name',
    text = { 'A moderately long', 'description of', 'your effect.' },
}
```

## `mod.process.loc_text`
Finally, it is possible to define additional localization strings even when not using localization files. Because the localization sometimes gets reloaded in-game, this can't be done by simply adding the strings to `G.localization` upon loading your mod. Instead, you should define a function on your mod object that puts the strings you need into place. It should look something like this:
```lua
SMODS.current_mod.process_loc_text = function()
    G.localization.misc.labels.mod_mylabel = 'My Label' -- assigning to G.localization directly
    SMODS.process_loc_text(G.localization.misc.labels, 'mod_anotherlabel', { -- Util function to handle multiple languages.
        ['en-us'] = 'English label',
        ['de'] = 'Deutsches Label',
        ['it'] = 'Etichetta italiana',
    })
end
```

# Text formatting
```lua
{
    name = 'Example name',
    text = {
        'This is {C:attention}coloured{} text',
        '{s:0.8}This is scaled down, {s:1.5}this is scaled up',
        'This text can {V:1}change colours{}!',
        'This is {E:1}floating DynaText',
        'This is {E:2}spaced out DynaText',
        '{X:mult,C:white} X#1# {} Mult',
        '{T:v_telescope}This will display a tooltip when hovered',
    },
}
```

# Localization functions
To create dynamic descriptions, you need to create functions that define how they behave.
## `loc_vars`
`loc_vars` functions are a simple to use but versatile tool for any type of description.
```lua
SMODS.Consumable {
    -- `self`: The object this function is being called on (the card prototype)
    -- `info_queue`: A table that stores tooltips to be displayed alongside the description
    -- `card`: The card being described. If no card is available because this is for a tooltip,
    -- a fake table that looks like a card will be provided (See: duck typing)
    -- NOTE: card may belong to a different class depending on what object this is defined on;
    -- e.g. a `Tag` is passed if this is defined on a `SMODS.Tag` object.
    loc_vars = function(self, info_queue, card)
        -- Add tooltips by appending to info_queue
        info_queue[#info_queue+1] = G.P_CENTERS.m_stone -- Add a description of the Stone enhancement
        info_queue[#info_queue+1] = G.P_CENTERS.j_stone -- Add a description of Stone Joker
        -- all keys in this return table are optional
        return {
            vars = {
                card.ability.extra.mult, -- replace #1# in the description with this value
                card.ability.extra.mult_gain, -- replace #2# in the description with this value
                colours = {
                    G.C.SECONDARY_SET.Tarot, -- colour text formatted with {V:1}
                    HEX(card.ability.extra.colour_string), -- colour text formatted with {V:2}
                },
            },
            key = self.key..'_alt', -- Use an alternate description key (pulls from G.localization.descriptions[self.set][key])
            set = 'Spectral', -- Use an alternate description set (G.localization.descriptions[set][key or self.key])
            scale = 1.2, -- Change the base text scale of the description
            text_colour = G.C.RED, -- Change the default text colour (when no other colour is being specified)
            background_colour = G.C.BLACK, -- Change the default background colour
            main_start = nil, -- Table of UIElements inserted before the main description (-> "Building a UI")
            main_end = nil, -- Table of UIElements inserted after the main description
        }
    end,
}
```

## `locked_loc_vars`
This function is called for descriptions of locked cards. It behaves identically to regular `loc_vars`.

## `generate_ui` *(advanced)*
The base implementation of this function acts as a wrapper to `loc_vars`. Overriding it allows you to freely define the UI of a card description. Function signature:
```lua
    {
        generate_ui = function(self, info_queue, card, desc_nodes, specific_vars, full_UI_table)
            -- `self`, `info_queue`, `card`: See loc_vars
            -- `desc_nodes`: A table to place UIElements into to be displayed in the current description box
            -- `specific_vars`: Variables passed from outside the current `generate_ui` call. Can be ignored for modded objects
            -- `full_UI_table`: A table representing the main description box. 
            -- Mainly used for checking if this is the main description or a tooltip, or manipulating the main description from tooltips.
            -- This function need not return anything, its effects should be applied by modifying desc_nodes
        end,
    }
```
# Logging

Steamodded implements 6 levels of logging, each with a different purpose. The levels are as follows:

- `TRACE`: Very detailed information, typically of interest only when diagnosing problems.
- `DEBUG`: Detailed information, typically of interest only when diagnosing problems.
- `INFO`: Confirmation that things are working as expected.
- `WARN`: An indication that something unexpected happened, or indicative of some problem in the near future. The
  software is still working as expected.
- `ERROR`: Due to a more serious problem, the software has not been able to perform some function.
- `FATAL`: A serious error, indicating that the program itself may be unable to continue running.

## Logging usage

All logging functions follow the same pattern, with the first argument being the message, and the second argument being
the logger name. This argument order is consistent across all logging functions. The logger name is optional and will
default to `DefaultLogger` but it is strongly recommended that you use it for clarity purposes.

### Trace

#### Lua

```lua
sendTraceMessage("This is a trace message", "MyTraceLogger")
```

#### Output

```log
2024-04-04 22:58:45 :: TRACE :: MyTraceLogger :: This is a trace message
```

### Debug

#### Lua

```lua
sendDebugMessage("This is a debug message", "MyDebugLogger")
```

#### Output

```log
2024-04-04 22:58:45 :: TRACE :: MyDebugLogger :: This is a debug message
```

### Info

#### Lua

```lua
sendInfoMessage("This is an info message", "MyInfoLogger")
```

#### Output

```log
2024-04-04 22:58:45 :: INFO  :: MyInfoLogger :: This is an info message
```

### Warn

#### Lua

```lua
sendWarnMessage("This is a warn message", "MyWarnLogger")
```

#### Output

```log
2024-04-04 22:58:45 :: WARN  :: MyWarnLogger :: This is a warn message
```

### Error

#### Lua

```lua
sendErrorMessage("This is an error message", "MyErrorLogger")
```

#### Output

```log
2024-04-04 22:58:45 :: ERROR :: MyErrorLogger :: This is an error message
```

### Fatal

#### Lua

```lua
sendFatalMessage("This is a fatal message", "MyFatalLogger")
```

#### Output

```log
2024-04-04 22:58:45 :: FATAL :: MyFatalLogger :: This is a fatal message
```

### Additional notes

A logger name can be any string, but it is recommended to use a descriptive name to make it easier to identify where
the log message is coming from.

An additional function `sendMessageToConsole` is available to send a log message with a custom log level. This function
is not recommended for general use.

You can, with this function, implements your own log levels, but it is not recommended to do so, as the debug console do
not support custom log levels by default. You will have to modify the debug console by yourself to support custom log
levels. If you want to do so, you'll find the function signature in
the [debug.lua](https://github.com/Steamodded/smods/blob/main/src/logging.lua#L39-L49) file, and the log
supported in the console on
the [tk_debug_window.py](https://github.com/Steamodded/smods/blob/main/tk_debug_window.py#L9-L16) file.

Keep in mind that other people may not have the same modifications as you, so it is recommended to use the default log
levels.

## Debug Console

The debug console is a tool that allows you to see the logs in real-time, filter them, and search for specific strings
within the logs. It is a very useful tool for debugging and diagnosing problems.

### Opening the Debug Console

The debug console requires python 3.6+ to be installed on your system. To open the debug console,
execute `tk_debug_window.py` in [the steamodded repo](https://github.com/Steamodded/smods/blob/main/tk_debug_window.py).

The debug console was not tested with python 3.6, but it _should_ work with it. It is recommended to use the latest
python release.

### Using the Debug Console

You need to have the debug console open before you start the game. The console will show all logs in real-time.

### Features

- You can filter logs by logger name.
- You can filter logs by log level (only a single level, or the level selected and above).
- You can search for a specific string within the logs shown.
- You can save the logs to a file using the top left dropdown.
- You can clear the logs all the logs received from the console.

### Shortcuts

- `Ctrl + F`: Focus the search bar.
- `Enter` when focused on the search bar: Go to the next search result.
- `Esc` when focused on the search bar or the logger filter: Focus back into the logs.
- `Shift + Enter` when focused on the search bar: Go to the previous search result.
- `Ctrl + S`: Save all logs to a file.
- `Ctrl + Shift + S`: Save filtered logs to a file.
- `Ctrl + D`: Clear all logs.
- `Ctrl + L`: Focus the logger filter.

## Issues

Please open an issue on the [GitHub repository](https://github.com/Steamodded/smods/issues/new) if you encounter
any problems with the debug console.
# Mod functions
## `mod.config_tab`
Your mod can have an optional config page that is accessed through the Mods menu. 

### Setting Up Your Config
Create the structure of your config in a file named `config.lua` stored in your mod directory. SMODS will take this and assign it as the config for your mod, handling modifications and reloading internally. Your `config.lua` file should follow a structure like this:
```lua
return {
	["setting_1"] = true,
	["setting_2"] = {
		["option_1"] = 50,
		["option_2"] = 3,
		["option_3"] = false,
	}
}
```

You can access your config inside your mod by using this set up `local config = SMODS.current_mod.config`. Following this, using `config.setting_1` would give you the value of setting_1 in your config.

### Creating a Config Tab
To create a **config tab**, use this block of code.
```lua
SMODS.current_mod.config_tab = function()
	return {n = G.UIT.ROOT, config = {
		-- config values here, see 'Building a UI' page
	}, nodes = {
		-- work your UI wizardry here, see 'Building a UI' page
	}}
end
```

## `mod.extra_tabs`
You may want to create additional pages besides your config tab.

### Creating Additional Tabs
To create **additional tabs**, use this block of code.
```lua
SMODS.current_mod.extra_tabs = function()
	return {
		{
			label = 'My Label',
			tab_definition_function = function()
				-- works in the same way as mod.config_tab
				return {n = G.UIT.ROOT, config = {
					-- config values here, see 'Building a UI' page
				}, nodes = {
					-- work your UI wizardry here, see 'Building a UI' page
				}}
			end,
		},
		-- insert more tables with the same structure here
	}
end
```

## `mod.custom_collection_tabs`
This sets up additional collection pages to be accessed through the 'Other' button.
```lua
SMODS.current_mod.custom_collection_tabs = function()
	return {
		{
			button = UIBox_button({
				-- calls `G.FUNCS.your_collection_something` when pressed, define accordingly
				button = 'your_collection_something', 
				id = 'your_collection_something',
				-- Displayed label on the button (using non-localized strings also works)
				label = {localize('b_your_label')},
				-- optional; should have numeric 'tally' and 'of' values (for discovery counts)
				count = G.DISCOVER_TALLIES['something'], 
				-- optional; minimum width of your button
				minw = 5,
			})
		},
		-- add more buttons here
	}
end
G.FUNCS.your_collection_something()
	G.SETTINGS.paused = true
  	G.FUNCS.overlay_menu{
    	definition = create_UIBox_your_collection_something(), -- this is the actual UI definition function
  	}
end
-- define `create_UIBox_your_collection_something()` to create the collection, see 'Building a UI'
```

## Advanced Mod descriptions
Default mod descriptions defined within your mod's metadata supports basic text wrapping, but no further formatting like changing the colour and scale of text or inserting variables, as well as localization. By using [Localization files](https://github.com/Steamodded/smods/wiki/Localization#localization-files-recommended), you can create a description for your mods with support for all the formatting of other descriptions. Your description should be placed in `G.localization.descriptions.Mod[mod_id]`.

### `mod.description_loc_vars`
To change your description dynamically through variables and alternate keys or specify a default text colour and scale, you can define this function on your mod object. This function behaves like [`loc_vars`](https://github.com/Steamodded/smods/wiki/Localization#loc_vars) on other objects.
```lua
SMODS.current_mod.description_loc_vars = function(self)
	return {
		vars = { 'some var', colours = {HEX('123ABC') }}, -- text variables (e.g. #1#) and colour variables (e.g. {V:1})
		key = 'alternate_key', -- Get description from G.localization.descriptions.Mod[key] instead
		scale = 1.1, -- Change text scale, default 1
		text_colour = HEX('1A2B3C'), -- Default text colour if no colour control is active
		background_colour = HEX('9876547F')
	}
end
```

### `mod.custom_ui`
This function can be used to manipulate your mod's description tab arbitarily. It receives a table of nodes as an argument, you can modify this table to insert additional elements or modify existing ones. See also: [Building a UI](https://github.com/Steamodded/smods/wiki/UI-Guide).

## `mod.debug_info`
This is a property that can be set by mods to show debug information on the crash screen.
- Setting this to a string will display that string under your mod in the crash screen.
- Setting this to a table of strings will display a list with each key and its value under your mod in the crash screen.

## In-game functionality
You may want to have some functionality happen with your mod installed independently of some other object being present. For cases like this, defining these mod-scope functions is a good alternative to destructively modifying the underlying vanilla function and helps preserve compatibility with other mods.

### `mod.reset_game_globals(run_start)`
Set up global game values that reset each round, similar to vanilla jokers like Castle, The Idol, Ancient Joker, and Mail-In Rebate. `run_start` indicates if the function is being called at the start of the run.
# Metadata
The (new) standard way to specify your mod's metadata is in a separate JSON file in your mod folder, as per the following specification:
```js
// Mods/your_mod/your_mod.json
{
	"id": "your_mod", // [required] ! Must be unique. "Steamodded", "Lovely" and "Balatro" are disallowed.
	"name": "Your Mod", // [required]
	"author": ["You", "Someone else"], // [required]
	"description": "A description of your mod.", // [required] ! To use more advanced typesetting, specify your description as a localization entry at G.localization.descriptions.Mod[id]
	"prefix": "prefix", // [required] ! Must be unique. This prefix is added to the keys of all objects your mod registers. UNLIKE LEGACY HEADERS, THERE IS NO DEFAULT VALUE.
	"main_file": "main.lua", // [required] ! This is the entry point of your mod. The specified file (including .lua extension) will be executed when your mod is loaded.
	"priority": -20, // [default: 0] ! Mods are loaded in order from lowest to highest priority value.
	"badge_colour": "FF230A", // [default: 666665] ! Background colour for your mod badge. Must be a valid hex color with 6 or 8 digits (RRGGBB or RRGGBBAA)
	"badge_text_colour": "ABC123", // [default: FFFFFF] ! Text colour for your mod badge.
	"display_name": "YM", // [default: <name>] ! Displayed text on your mod badge.
	"version": "1.0.0", // ! Must follow a version format of (major).(minor).(patch)(rev). rev starting with ~ indicates a beta/pre-release version.
	"dependencies": [
		"Steamodded (>=1.*)", // Allows any version past a 1.0.0 stable version (but disallows 1.0.0 beta versions)
		"Lovely (>=0.6)", // Allows all versions past 0.6.0 stable, including future beta versions and major version breaks
		"SomeMod (==1.0.*)", // Allows all versions past 1.0.0 stable that are 1.0.x (1.1 and later are disallowed)
		"SomeOtherMod (==1.0.0~)", // Allows 1.0.0 versions of any revision, beta or not.
		"Balatro (==1.0.1m)", // Allows only the specified version and revision.
		"OneMoreMod (>>1.0~g) (<<2~)", // << and >> are used for versions strictly less/greater than the specified one.
		// NOTE: The << operator should be used in combination with the ~ beta wildcard in order to exclude pre-release major version changes.
		"IRanOutOfIdeas (==1.*~)", // Combination of wildcard symbols. All 1.x versions are allowed, including beta.
		"Talisman | TalismanReplacement" // Multiple different mods can be used to fulfill this dependency. May want to use `provides` instead
	], // ! All mods in the list must be installed and loaded (and must fulfill version requirements), else this mod will not load.
	"conflicts": [
		"Talisman (>=1.1) (<<2~)" // Same format as for dependencies, except alternatives (|) are disallowed.
	], // ! No mods in the list (that fulfill version restrictions) may be installed, else this mod will not load.
	"provides": [
		"SomeAPIMod (1.0)"
	], // ! Use this if your mod is able to stand in for a different mod and fulfill dependencies on it. This allows the usage of a different ID so both mods can coexist. If you don't specify a valid version, your mod's version is used instead.
	"dump_loc": false // !! Not for use in distributions. Writes all localization changes made on startup to a file, for conversion from a legacy system.
}
```
Template for copying:
```json
{
	"id": "",
	"name": "",
	"display_name": "",
	"author": [""],
	"description": "",
	"prefix": "",
	"main_file": "",
	"priority": 0,
	"badge_colour": "",
	"badge_text_colour": "",
	"version": "",
	"dependencies": [],
}
```
## File header (outdated)
Using the legacy file header system is still supported, though switching to metadata files is encouraged. Steamodded will recognize your mod in this system only if the first line in your mod file is EXACTLY `--- STEAMODDED HEADER`. If you are transitioning away from using this method, make sure to remove the header to prevent Steamodded from trying to load your mod twice.

Your mod can also contain the following lines. These lines describe information about your mod and how Steamodded should load it.
- Required:
	- `--- MOD_NAME: Example Mod`
	- `--- MOD_ID: ExampleMod` (**Must be unique and without spaces**)
	- `--- MOD_AUTHOR: [You, AnotherDev, AnotherOtherDev]` (**Brackets are required**)
	- `--- MOD_DESCRIPTION: A description of your mod.` (**No line breaks, text is wrapped automatically.**)
- Optional:
	- `--- PRIORITY: -100` (**Negative values go first, positive values go last**)
	- `--- BADGE_COLOR: 123456` or `--- BADGE_COLOUR: ABCDEF`
	- `--- DISPLAY_NAME: Example` (**Shown on mod badges instead of your mod's name**)
	- `--- DEPENDENCIES: [Steamodded>=1.0.0~BETA, Mod1, Mod2>=1.0.0, Mod3<=1.7.5, Mod4>=1.0.0<=2.0]`
	- `--- CONFLICTS: [Mod5, Mod6<=0.9.9, Mod7>=0.6.2, Mod8<=1.0>=0.3.7]`
	- `--- PREFIX: example` (**Must be unique. Defaults to the first 4 letters, lowercase, of your mod's ID.**)
	- `--- VERSION: 1.0.0`
These lines can be specified in any order.

# API Documentation: SMODS.Achievement
- **Required parameters:**
	- `key`,
    - `loc_txt` or localization entry [(reference)](https://github.com/Steamodded/smods/wiki/Localization)
        - Rather than a `text` table, `loc_txt` should contain a `description` table. When using localization files, the name should be placed in `misc.achievement_names[key]`, the description should be placed in `misc.achievement_descriptions[key]`.
- **Optional parameters** *(defaults)*:
    - `atlas = 'Joker', pos = { x = 1, y = 0 }, hidden_pos = { x = 0, y = 0 }` [(reference)](https://github.com/Steamodded/smods/wiki/SMODS.Atlas#applying-textures-to-cards)
        - `pos` is used when the achievement has been earned, `hidden_pos` if it has not been earned.
    - `earned`: Achievement is considered "earned". Achievement will stay earned on loaded profiles unless `reset_on_startup` is true
    - `reset_on_startup`: Unearns the achievement if already earned on profile load
    - `bypass_all_unlocked = false`: Achievement cannot be earned if "Unlock All" button was pressed on current profile
    - `hidden_name = true`: Hides the name of the achievement if not earned
    - `hidden_text`: Hides the description of the achievement if not earned

## API methods
- `unlock_condition(self, args) -> bool`
    - Runs every time `check_for_unlock` is called. Returns true if the achievement should be earned. 
# API Documentation: `SMODS.Atlas`
This class allows you to use custom spritesheets ("Atlases") or replace existing ones. Your mod must be located in its own subdirectory of the `Mods` folder. Due to Balatro's pixel smoothing setting, it is required to provide both a single and double resolution image file. The file structure should look something like this:
```bash
Mods
└──NegateTexturePack
	├── NegateTexturePack.lua
	└── assets
		├── 1x
		│   ├── BlindChips-negate.png
		│   ├── Jokers-negate.png
		│   └── boosters-negate.png
		└── 2x
			├── BlindChips-negate.png
			├── Jokers-negate.png
			└── boosters-negate.png
```
- **Required parameters:**
	- `key`
	- `px`: the width of each individual sprite at single resolution, in pixels.
	- `py`: the height of each individual sprite at single resolution, in pixels.
	- `path`: the image file's name, including the extension (e.g. `'Jokers-negate.png'`).
		- If you want to use different sprites depending on the selected language, you can also provide a table:
		```lua
		path = {
			['default'] = 'Jokers.png', -- use this for any languages not specified
			['zh_CN'] = 'Jokers-zh-CN.png',
			['ja'] = 'Jokers-ja.png',
		}
		```
- **Optional parameters** *(defaults)*:
	- `atlas_table = 'ASSET_ATLAS'`
		- Use `ASSET_ATLAS` for non-animated sprites
		- Use `ANIMATION_ATLAS` for animated sprites
		- Use `ASSET_IMAGES` for other images
	- `frames`: for animated sprites, you must provide the number of frames of the animation. Each row should contain one animation, with each column showing one frame.
	- `raw_key`: Set this to `true` to prevent the loader from adding your mod prefix to the `key`. Useful for replacing sprites from the base game or other mods.
	- `language`: Restrict your atlas to a specific locale. Useful for introducing localized sprites while leaving other languages intact.
	- `disable_mipmap`: Disable mipmap being applied to this texture. Might remove artifacts on smaller textures.

## Applying textures to cards
For objects of any class that have a visual representation in-game, you can assign a sprite from your atlas by setting `atlas` to the key of your atlas and `pos` to the position of the sprite on this atlas (`{ x = 0, y = 0 }` refers to the top-left corner). Example:
```lua
SMODS.Joker {
	key = 'my_joker',
	atlas = 'my_atlas',
	pos = { x = 1, y = 1 } -- second row, second colum
}
```
# API Documentation: `SMODS.Back`
**Class prefix:** `b`
- **Required parameters:**
	- `key`,
	- `loc_txt` or localization entry [(reference)](https://github.com/Steamodded/smods/wiki/Localization)
- **Optional parameters** *(defaults)*:
    - `atlas = 'centers', pos = { x = 0, y = 0 }` [(reference)](https://github.com/Steamodded/smods/wiki/SMODS.Atlas#applying-textures-to-cards)
    - `config = {}, unlocked = true, discovered = false, no_collection, prefix_config, dependencies, display_size, pixel_size` [(reference)](https://github.com/Steamodded/smods/wiki/API-Documentation#common-parameters)


## API methods
- `calculate(self, back, context)` [(reference)](https://github.com/Steamodded/smods/wiki/Calculate-Functions)
    - This method is called from `Back:trigger_effect()` and incorporated into the standard calculation pipeline. This does not apply to vanilla `trigger_effect` functionality, which can be used as normal from this function by checking for `context.context == 'eval'` or `context.context == 'final_scoring_step` respectively.
    - **Defining a `trigger_effect` function on your deck is deprecated.**
- `loc_vars, locked_loc_vars, generate_ui` [(reference)](https://github.com/Steamodded/smods/wiki/Localization#Localization-functions)
- `apply(self, back)`
    - Apply modifiers at the start of a run. If you want to modify the starting deck, you must use events to do so.
- `in_pool(self, args) -> bool, { allow_duplicates = bool }`
	- Define custom logic for when a card is allowed to spawn. A card can spawn if `in_pool` returns true and all other checks are met.
	- `allow_duplicates` allows this card to spawn when one already exists, even without Showman.
	- When called from `generate_card_ui`, the `_append` key is passed as `args.source`.
- `check_for_unlock(self, args) -> bool`
	- Configure unlock conditions. Refer to the function `check_for_unlock` in Balatro's code for more information.
# API Documentation: `SMODS.Blind`
**Class prefix:** `bl`
- **Required parameters:**
	- `key`
	- `loc_txt` or localization entry [(reference)](https://github.com/Steamodded/smods/wiki/Localization)
- **Optional parameters** *(defaults)*
	- `atlas = 'blind_chips', pos = { x = 0, y = 0 }` [(reference)](https://github.com/Steamodded/smods/wiki/SMODS.Atlas#applying-textures-to-cards)
		- Any atlas specified here must set `atlas_table = 'ANIMATION_ATLAS'`. The `y` value determines the row to use for the animation. The `x` value is ignored and cycles through each frame of the animation.
	- `discovered = false, no_collection, prefix_config, dependencies` [(reference)](https://github.com/Steamodded/smods/wiki/API-Documentation#common-parameters)
	- `dollars = 5`: Amount of money obtained when defeated.
	- `mult = 2`: Required score relative to the Ante's Base score.
	- `boss`: Marks this Blind as a Boss Blind and specifies on which Antes it can appear (`{ min = 1, max = 10 }`). `max` is an artifact and not functional. Use `in_pool` instead for advanced conditions.
		- `boss.showdown`: Marks this Blind as a Final Boss Blind that shows up on every multiple of the winning Ante. `min` is ignored, use `in_pool` to restrict spawning.
	- `boss_colour`: Sets the background colour to use while playing this Blind (e.g. `HEX('56789A')`)
	- `debuff = {}`: Configure vanilla Blind effects with these fields:
		- Disallowing hands in full:
			- `hand = { ['Hand Type'] = true}` for a specific set of hands,
			- `h_size_ge = n`, must play at least `n` cards,
			- `h_size_le = n`, must play at most `n` cards.
		- Debuffing cards:
			- `suit = 'Hearts'` for one specific suit,
			- `value = '2'` for one specific rank,
			- `nominal = n` for all ranks scoring `n` base chips,
			- `is_face = true` for all face cards.
		- These effects are ignored if you specify a `debuff_hand` or `debuff_card` function respectively.
	- `ignore_showdown_check`: Enabling this allows `in_pool` to be evaluated regardless of whether a showdown Boss Blind was requested or not.
	- `vars = {}`: variables for the Blind's description in the collection. Fallback if `collection_loc_vars` isn't set.

## API methods
In all of the following methods, use the global variable `G.GAME.blind` to
refer to the current blind. (The base game uses `self` to refer to the current blind in `Blind:foo()`.)
- `set_blind(self)`
	- Effects that activate when this Blind is selected
- `disable(self)`
	- Reverting effects when this Blind gets disabled
- `defeat(self)`
	- Reverting effects when this Blind is defeated
- `drawn_to_hand(self)`
	- Effects that activate when cards are drawn to hand
- `press_play(self)`
	- Effects that activate when a hand is played
- `recalc_debuff(self, card, from_blind) -> bool`
	- Determines what cards should be debuffed by this Blind
- `debuff_hand(self, cards, hand, handname, check) -> bool`
	- Determines if a hand is disallowed by this Blind
- `stay_flipped(self, area, card) -> bool`
	- Determines what cards are drawn face down
- `modify_hand(self, cards, poker_hands, text, mult, hand_chips) -> number, number, bool`
	- Used for modifications on the base score of played hands. Expected return values in order are:
		- The modified value of `mult`
		- The modified value of `hand_chips`
		- A boolean value indicating whether any values were changed
- `get_loc_debuff_text(self) -> string`
	- Allows modifying text displayed for debuff warnings on invalid hands
- `loc_vars(self) -> { vars ?= table, key ?= string }` [(reference)](https://github.com/Steamodded/smods/wiki/Localization#Localization-functions)
	- Due to various constraints, the functionality of `loc_vars` on blinds is very limited. Only `vars` and `key` returns are supported, and no `info_queue` exists.
- `collection_loc_vars(self) -> { vars ?= table, key ?= string }`
	- Used for passing variables to Blind descriptions when viewing the collection. If not defined, the game will use the `vars` field on your object.
- `in_pool(self) -> bool`
	- For implementing advanced restrictions on when a Blind may appear in a run. This puts `boss.min` and `boss.max` restrictions out of effect.
	- Unless `ignore_showdown_check` is set, this function will not be evaluated in the following cases:
		- A showdown Boss Blind should appear, but this Blind is a regular Boss Blind.
		- A regular Boss Blind should appear, but this Blind is a showdown Boss Blind.
# API Documentation: `SMODS.Booster`
**Class prefix:** `p`
- **Required parameters:**
	- `key`,
	- `loc_txt` or localization entry [(reference)](https://github.com/Steamodded/smods/wiki/Localization)
        - `loc_txt` can contain an additional `group_name` string. This is the bottom text while on the pack opening screen.
        - With localization files, this text should be stored in `misc.dictionary[group_key or 'k_booster_group_'..key]`
- **Optional parameters** *(defaults)*:
    - `atlas = 'Booster', pos = { x = 0, y = 0 }` [(reference)](https://github.com/Steamodded/smods/wiki/SMODS.Atlas#applying-textures-to-cards)
    - `config, discovered = false, no_collection, prefix_config, dependencies, display_size, pixel_size` [(reference)](https://github.com/Steamodded/smods/wiki/API-Documentation#common-parameters)
        - The default `config` table is `{ extra = 3, choose = 1 }`; `extra` is the amount of cards in the pack, `choose` is the amount of choices.
        - Note: `unlocked` on boosters is currently unsupported.
    - `group_key`: Key to the group name (see above) of this booster. Useful when multiple booster packs share the same group name.
	- `cost = 4`,
    - `weight = 1`: Determines how freqently the pack appears in the shop.
    - `draw_hand = false`: If this is `true`, draw playing cards to hand when this pack is opened.
    - `kind`: Used for grouping packs together. For example, this can be used in `get_pack()` to generate a booster pack of a given type.
    - `select_card`: Set to string of card area, `consumeables`, to save cards from the pack instead of using them.

## API methods
- `loc_vars, generate_ui` [(reference)](https://github.com/Steamodded/smods/wiki/Localization#Localization-functions)
- `create_card(self, card, i) -> table|Card`
	- Creates the cards inside of the booster pack. `card` is the booster pack card, `i` is the position of the card to be generated. If the returned table is not a `Card`, it is passed into [`SMODS.create_card`](https://github.com/Steamodded/smods/wiki/Utility#mod-facing-utilities).
- `update_pack(self, dt)`
	- Handles booster pack UI when opened. 
- `ease_background_colour(self)`
	- Changes background colour. 
- `particles(self)`
	- Handles particle effects. 
- `create_UIBox(self) -> table`
	- Returns the booster's UIBox. 
- `set_ability(self, card, initial, delay_sprites)`
	- Set up initial ability values or manipulate sprites in an advanced way.
- `in_pool(self, args) -> bool, { allow_duplicates = bool }`
	- Define custom logic for when a card is allowed to spawn. A card can spawn if `in_pool` returns true and all other checks are met.
	- `allow_duplicates` allows this card to spawn when one already exists, even without Showman.
	- When called from `generate_card_ui`, the `_append` key is passed as `args.source`.
- `update(self, card, dt)`
	- For actions that happen every frame.
- `set_sprites(self, card, front)`
	- For advanced sprite manipulation that happens when a card is created or loaded.
- `load(self, card, card_table, other_card)`
	- For modifications to sprites or the card itself when a run is reloaded.
- `set_badges(self, card, badges)`
	- Add additional badges, leaving existing badges intact. This function doesn't return; add badges by appending to `badges`.
	- Avoid overwriting existing elements. It will cause text to appear on the top left corner of your screen instead.
	- Function for creating badges: `create_badge(_string, _badge_col, _text_col, scaling)`
		- `_string`: Text displayed on the badge.
		- `_badge_col = G.C.GREEN`: Background colour.
		- `_text_col = G.C.WHITE`: Text colour.
		- `_scaling = 1`: Relative size of the badge.
	- Example:
	```lua
	{
		set_badges = function(self, card, badges)
			badges[#badges+1] = create_badge(localize('k_your_string'), G.C.RED, G.C.BLACK, 1.2 )
		end,
	}
	```
- `set_card_type_badge(self, card, badges)`
	- Same as `set_badges`, but bypasses creation of the card type / rarity badge, allowing you to replace it with a custom one.
- `draw(self, card, layer)`
	- Draws the sprite and shader of the card.

## Utility function
- `SMODS.Booster:take_ownership_by_kind(kind, obj, silent)`
    - Finds all booster packs with matching `kind` and calls `SMODS.Booster:take_ownership()` for each one with arguments `obj, silent` passed through.
# API Documentation: `SMODS.Challenge`
- **Required parameters:**
    - `key`
    - `loc_txt` or localization entry [(reference)](https://github.com/Steamodded/smods/wiki/Localization)
        - The only supported field is `name`. In localization files, it is set by referring to `misc.challenge_names[key]`.
- **Optional parameters** *(defaults)*:
    - `rules`: Custom rules and modifiers for the challenge.
        - `rules.custom`: Expects a list of tables with an `id` and optionally a `value` field (defaults to `true`). Sets `G.GAME.modifiers[id] = value`. Text for each rule should be stored in `G.localization.misc.v_text['ch_c_'..id]`, `value` is passed as a variable. The following custom rule keys are defined by the base game:
            - `all_eternal`, `chips_dollar_cap`, `daily`, `debuff_played_cards`, `discard_cost`, `flipped_cards`, `inflation`, `minus_hand_size_per_X_dollar`, `no_extra_hand_money`, `no_interest`, `no_reward`, `no_reward_specific`, `no_shop_jokers`, `none`, `set_eternal_ante`, `set_joker_slots_ante`, `set_seed`.
        - `rules.modifiers`: Expects a list of tables with an `id` and a `value` field. Sets each corresponding base modifier to the given value. The following modifiers are supported:
            - `dollars`, `discards`, `hands`, `reroll_cost`, `joker_slots`, `consumable_slots`, `hand_size`.
    - `jokers`: Expects a list of tables that represent jokers added at the start of the run. Each table can have the following fields:
        - `id` (required): The key of the joker to create.
        - `edition`: The edition of the joker, if any, given without the `e_` prefix.
        - `eternal`: If the joker is eternal.
        - `pinned`: If the joker is pinned.
    - `consumeables`: Behaves like `jokers`, but for consumables.
        - Supports all fields of `jokers` except `pinned`.
    - `vouchers`: Behaves like `jokers`, but for vouchers redeemed at the start of the run.
        - Supports the same fields as `consumeables`, but `edition` and `eternal` have no functional effect beyond displaying in the preview UI.
    - `restrictions`: Contains information about objects that are banned in this challenge.
        - `restrictions.banned_cards`: Expects a list of tables with keys to ban in their `id` fields.
            - This can be used to ban jokers, consumables, vouchers and booster packs. 
            - If a table has an `ids` field containing a list of center keys, only `id` is shown as banned in the challenge UI, but all of the `ids` are banned.
        - `restrictions.banned_tags`: Expects a list of tables with valid tag keys in their `id` fields.
        - `restrictions.banned_other`: Expects a list of tables with valid keys in their `id` field and a `type` string with the value `'blind'`.
            - Despite the name, the UI for this only supports using this to ban blinds.
            - Functionally, all three options achieve the same task of adding specified keys to `G.GAME.banned_keys`.
    - `deck`: Defines the challenge's deck.
        - `deck.type = 'Challenge Deck'`: The deck type for this challenge. It is not recommended to change this value.
        - `deck.cards`: Defines the cards present in the deck using a list of *control tables*. Control tables have the following structure:
            - `s`: Suit of the card, given by its `card_key`.
            - `r`: Rank of the card, given by its `card_key`.
            - `e`: Enhancement of the card, given by its key.
            - `d`: Edition of the card, given by its key without the `e_` prefix.
            - `g`: Seal of the card, given by its key. *(The key for this option is based on Gold Seals being the only available seals in the demo version.)* 
        - `deck.yes_ranks`, `deck.yes_suits`: Can be used only if no `cards` table is specified. Expects a key-indexed table of ranks/suits by their `card_key` and acts as a whitelist, i.e., it includes only cards of those ranks/suits in the starting deck.
        - `deck.no_ranks`, `deck.no_suits`: Same as `yes_ranks` and `yes_suits`, but acts as a blacklist, i.e., the specified ranks/suits are excluded.
        - `deck.enhancement`: Can be used only if no `cards` table is specified. Given an enhancement by its key, apply it to all cards in the starting deck.
        - `deck.edition`: Can be used only if no `cards` table is specified. Given an edition by its key without the `e_` prefix, apply it to all cards in the starting deck.
        - `deck.seal`: Can be used only if no `cards` table is specified. Given a seal by its key, apply it to all cards in the starting deck.

## API methods
- `unlocked(self) -> bool`
    - Defines when the challenge should be unlocked (or not).
# API Documentation: `SMODS.Consumable`
**Class prefix:** `c`
- **Required parameters:**
	- `key`,
    - `set` (`'Tarot'`, `'Planet'`, `'Spectral'` or the key of a modded consumable type)
	- `loc_txt` or localization entry [(reference)](https://github.com/Steamodded/smods/wiki/Localization)
- **Optional parameters** *(defaults)*:
    - `atlas = 'Tarot', pos = { x = 0, y = 0 }` [(reference)](https://github.com/Steamodded/smods/wiki/SMODS.Atlas#applying-textures-to-cards)
    - `config = {}, unlocked = true, discovered = false, no_collection, prefix_config, dependencies, display_size, pixel_size` [(reference)](https://github.com/Steamodded/smods/wiki/API-Documentation#common-parameters)
	- `cost = 3`,
    - `hidden` for legendary consumables like The Soul. For further customization:
		- `soul_set = 'Spectral'`, legendaries may appear when a card of either their own set or this one are being generated.
		- `soul_rate = 0.003`, determines how likely this legendary is to replace each card in a pack
		- `can_repeat_soul`, allows repeats of this card as if Showman were present

## API methods
- `calculate(self, card, context)` [(reference)](https://github.com/Steamodded/smods/wiki/Calculate-Functions)
- `loc_vars, locked_loc_vars, generate_ui` [(reference)](https://github.com/Steamodded/smods/wiki/Localization#Localization-functions)
- `use(self, card, area, copier)`
	- Defines the behavior of a consumable when used. *(The `copier` argument is a remnant from an effect no longer present in the game and can be ignored.)*
- `can_use(self, card) -> bool`
	- Determines whether a consumable is currently able to be used.
- `keep_on_use(self, card) -> bool`
	- Allows a used card to stay where it is or be moved to the consumables area from a booster pack instead of getting destroyed.
- `set_ability(self, card, initial, delay_sprites)`
	- Set up initial ability values or manipulate sprites in an advanced way.
- `add_to_deck(self, card, from_debuff)`
	- Modify the game state when this card is obtained.
	- Cards are considered added when they become undebuffed (in this case, `from_debuff` will be true).
- `remove_from_deck(self, card, from_debuff)`
	- Modify the game state when this card is sold, destroyed, or otherwise removed.
	- Cards are considered removed when debuffed (in this case, `from_debuff` will be true).
- `in_pool(self, args) -> bool, { allow_duplicates = bool }`
	- Define custom logic for when a card is allowed to spawn. A card can spawn if `in_pool` returns true and all other checks are met.
	- `allow_duplicates` allows this card to spawn when one already exists, even without Showman.
	- When called from `generate_card_ui`, the `_append` key is passed as `args.source`.
- `update(self, card, dt)`
	- For actions that happen every frame.
- `set_sprites(self, card, front)`
	- For advanced sprite manipulation that happens when a card is created or loaded.
- `load(self, card, card_table, other_card)`
	- For modifications to sprites or the card itself when a run is reloaded.
- `check_for_unlock(self, args) -> bool`
	- Configure unlock conditions. Refer to the function `check_for_unlock` in Balatro's code for more information.
- `set_badges(self, card, badges)`
	- Add additional badges, leaving existing badges intact. This function doesn't return; add badges by appending to `badges`.
	- Avoid overwriting existing elements. It will cause text to appear on the top left corner of your screen instead.
	- Function for creating badges: `create_badge(_string, _badge_col, _text_col, scaling)`
		- `_string`: Text displayed on the badge.
		- `_badge_col = G.C.GREEN`: Background colour.
		- `_text_col = G.C.WHITE`: Text colour.
		- `_scaling = 1`: Relative size of the badge.
	- Example:
	```lua
	{
		set_badges = function(self, card, badges)
			badges[#badges+1] = create_badge(localize('k_your_string'), G.C.RED, G.C.BLACK, 1.2 )
		end,
	}
	```
- `set_card_type_badge(self, card, badges)`
	- Same as `set_badges`, but bypasses creation of the card type / rarity badge, allowing you to replace it with a custom one.
- `draw(self, card, layer)`
	- Draws the sprite and shader of the card.
# API Documentation: `SMODS.DeckSkin`
This API extends the game's Friends of Jimbo collabs screen to create new deck skins for all suits, including modded ones.

Note: Atlases in this class are not automatically prefixed.

- **Required parameters:**
    - `key`
    - `suit`: The suit this skin applies to.
    - `loc_txt` or localization entry [(reference)](https://github.com/Steamodded/smods/wiki/Localization)
    - `palettes`: A list of tables with the following values:
        - `ranks`: A list of ranks the skin provides sprites for.
        - `display_ranks`: A list of ranks to show in the preview.
        - `atlas`: Atlas for the cards.
        - `pos_style = 'deck'`: Determines how to access sprite positions.
            - As a string:
                - `'deck'`: Use the `pos` table of the playing card.
                - `'suit'`: `y` position is always zero, `x` position is taken from the card's `pos`.
                - `'collab'`: Use the base game's `G.COLLABS.pos`.
                - `'ranks'`: Use the palette's rank list for the `x` position.
            - As a table:
                - `fallback_style`: Any of the above strings. Is used when a rank that isn't specified is loaded.
                - `[rank]`: Using a rank as a key, it takes a table with an atlas and pos.
        - `colour`: Replaces the suit's colour in the `G.C` table with this one when skin is applied.
        - `suit_icon`: A table defining the icon for the suit
            - `atlas`: An atlas with the icons
            - `pos = 0`: The position for the icon. If set to a number, it will use y = pos, x = suit's vanilla icon pos. If it's a table [see reference](https://github.com/Steamodded/smods/wiki/SMODS.Atlas#applying-textures-to-cards).
	    - `loc_txt` or localization entry [(reference)](https://github.com/Steamodded/smods/wiki/Localization)

# API Documentation: `SMODS.Edition`
**Class prefix:** `e`
- **Required parameters:**
	- `key`
	- `shader`: the shader key for your shader. `shader = false` is allowed. This will create edition with no shader.
	- `loc_txt` or localization entry [(reference)](https://github.com/Steamodded/smods/wiki/Localization)
		- `loc_txt` should contain an additional `label` string. It is used on badges, while `name` is displayed at the top of info boxes. For use with localization files, this label should be placed in `misc.labels` **(without the `e_` prefix)**.
- **Optional parameters** *(defaults)*:
	- `atlas = 'Joker', pos = { x = 0, y = 0 }` [(reference)](https://github.com/Steamodded/smods/wiki/SMODS.Atlas#applying-textures-to-cards)
		- This defines the card to draw the edition on in the collection.
	- `config = {}, unlocked = true, discovered = false, no_collection, prefix_config, dependencies` [(reference)](https://github.com/Steamodded/smods/wiki/API-Documentation#common-parameters)
		- The following base values for `config` are supported and will be applied/scored automatically:
		```lua
			{
				chips = 10,
				mult = 10,
				x_mult = 2,
				p_dollars = 3,
				card_limit = 2,
			}
		```
	- `in_shop = false`: Whether the edition can spawn naturally in the shop/booster packs.
	- `weight = 0`: The weighting of the edition, see below for more details.
	- `extra_cost`: The extra cost applied to cards in the shop with the edition.
	- `apply_to_float = false`: Whether the shader should be applied to floating sprites or not.
	- `badge_colour = G.C.DARK_EDITION`: Used to set a custom badge colour.
	- `sound = { sound = "foil1", per = 1.2, vol = 0.4 }`: Used to set a custom sound when the edition is applied.
	- `disable_shadow`: Disables shadow drawn under the card.
	- `disable_base_shader = false`: Whether the base shader should be applied (`booster` for Booster packs and Spectral cards, `voucher` for Vouchers and Invisible Joker, `dissolve` otherwise). Enable this if your shader modifies card transparency or shape in any way. Example:<br/>![image](https://github.com/user-attachments/assets/c7b32385-e486-40c2-9a83-c8a09a67185c)

## API methods
- `loc_vars` [(reference)](https://github.com/Steamodded/smods/wiki/Localization#Localization-functions)
	- Only `vars`, `key` and `set` return values are currently supported for editions.
- `get_weight(self) ->  number `
	- Used to modify the weight of edition on certain conditions.
- `on_apply(card) -> void`
	- Used to modify Card when edition is applied
- `on_remove(card) -> void`
	- Used to modify Card when edition is removed
- `on_load(card) -> void`
	- Used to modify Card with edition when it is loaded from save file.
- `draw(self, card, layer)`
	- Draws the edition's shader. `self.shader` is drawn by default when this is absent.
## Other information
### SMODS.Shader
A shader is required for a custom edition.
- **Required parameters:**
	- `key`
	- `path`: The file name of your shader. Shaders must be stored in `assets/shaders/` and be a `.fs` file.
		- The shader's `key`, file name (without extension) and the shader name used in the GLSL file **must be identical.**

### API methods
- `send_vars(sprite, @nullable card) -> table`
	- Used to send extra vars to the shader. Card may be `nil` if shader is not applied to a Card. Returned table entries are sent to the shader via `Shader:send(key, value)`. Usage example may be found in `example_mods/Mods/EditionExamples` - [here](https://github.com/Steamodded/examples/blob/master/Mods/EditionExamples/EditionExamples.lua#L126) and [here](https://github.com/Steamodded/examples/blob/master/Mods/EditionExamples/assets/shaders/gold.fs#L24)


#### Working with Shaders
[ionized.fs](https://github.com/Steamodded/examples/blob/master/Mods/EditionExamples/assets/shaders/ionized.fs) has shader code explanation with comments.
For a general guide, look at [LOVE2D introduction to shaders](https://blogs.love2d.org/content/beginners-guide-shaders).

If you want to see vanilla Balatro shaders, unzip the Balatro.exe and go to `resources/shaders` folder.

To see values for default externs, check out `engine/sprite.lua` -> `Sprite:draw_shader`.


#### Useful shaders resources
- [The book of shaders](https://thebookofshaders.com) - beginner friendly introduction to shaders.
- [GLSL Editor](https://patriciogonzalezvivo.github.io/glslEditor/) - preview your fragment shaders live.
- [GLSL Image Processing System](https://github.com/kajott/GIPS/releases) - preview your fragment shaders live. The program additionally provides some built-in shaders as examples.
- [Inigo Quilez articles](https://iquilezles.org/articles/) - in-depth articles on algorithms and techniques you could use in shaders. A lot of those are for 3D, but there's some 2D stuff as well.
- [Shadertoy](https://www.shadertoy.com) - tons of shaders from other people to learn from. A lot of them are pretty complex and 3D, but you can find simple 2D ones.

Note: in all resources the language is slightly different from LOVE2D shaders language, but the logic works the same way.


### Weight System
The default game editions have the following weights
```
negative = 3
polychrome = 3 (takes negative's weight in some circumstances)
holographic = 14
foil = 20
```
The base chance for an edition to appear in the shop is 4%, thus there is a 2% chance for a foil joker to appear. Adding custom editions to the shop maintains this 4% chance. Therefore, if you add an edition with a weight of 40, it will have a 2% chance to spawn, and foil will be reduced to 1% chance.

When the Hone and Glow Up vouchers are purchased, the weights of the base editions are adjusted. Custom editions do not have this bonus applied by default. This means that the vouchers do not increase the edition chances to 8%/16%, but rather increase the chances of the default editions by 2X and 4X respectively. To add your edition to the Hone and Glow Up functionality, add this to your edition definition.
```lua
get_weight = function(self)
	return G.GAME.edition_rate * self.weight
end
```

### Edition methods
- `Card:set_edition(edition, immediate, silent)`
	- `edition`, `nil` removes edition, `key` of edition as a string
	- `immediate`, *boolean*
	- `silent`, *boolean*
Use this function to set the edition of a card

- `poll_edition(_key, _mod, _no_neg, _guaranteed, _options)` *(defaults)*
	- `_key = edition_generic`, *string* - key value for a random seed
	- `_mod = 1`, *number* - scale of chance against base card
	- `_no_neg = false`, *boolean* - disables negative edition chance *(chance is added to polychrome)*
	- `_guaranteed = false`, *boolean* - disables base card
	- `_options = all editions marked in_shop`, *table* - List of editions to poll. Two variations.
	Option 1 - list of keys for included editions. This method respects defined weights,
	```lua
	{"e_foil", "e_holo","e_negative"}
	```
	Option 2 - table of keys and weights. Used to override default weights,
	```lua
	{
		e_foil = 1,
		e_holo = 1,
		e_negative = 1
	}
	```
# API Documentation: SMODS.Enhancement
**Class prefix:** `m`
- **Required parameters:**
	- `key`
    - `loc_txt` or localization entry [(reference)](https://github.com/Steamodded/smods/wiki/Localization)
- **Optional parameters** *(defaults)*:
	- `atlas = 'centers', pos = { x = 0, y = 0 }` [(reference)](https://github.com/Steamodded/smods/wiki/SMODS.Atlas#applying-textures-to-cards)
    - `config = {}, no_collection, prefix_config, dependencies, display_size, pixel_size` [(reference)](https://github.com/Steamodded/smods/wiki/API-Documentation#common-parameters)
        - The following base values for `config` are supported and will be scored automatically:
        ```lua
		{
			bonus,
            bonus_chips,
			mult,
			x_mult,
			p_dollars,
            h_mult,
            h_x_mult,
		}
	    ```
        - Note: `discovered` and `unlocked` on enhancements are currently unsupported.
    - `replace_base_card`: If `true`, don't draw base card sprite or give base card chips.
    - `no_rank`: If `true`, enhanced card has no rank.
    - `no_suit`: If `true`, enhanced card has no suit.
    - `overrides_base_rank`: If `true`, enhancement cannot be generated by Grim, Familiar and Incantation *(enhancements with `no_rank` set to true are automatically assigned this property)*.
    - `any_suit`: If `true`, enhanced card counts as any suit.
    - `always_scores`: If `true`, enhanced card always counts in scoring.
	- `weight`: The weighting of the enhancement, follows same rules as other weighted objects *(default weight is 5)*.

## API methods
- `calculate(self, card, context)` [(reference)](https://github.com/Steamodded/smods/wiki/Calculate-Functions)
- `loc_vars, locked_loc_vars, generate_ui` [(reference)](https://github.com/Steamodded/smods/wiki/Localization#Localization-functions)
- `get_weight(self) ->  number `
	- Used to modify the weight of enhancement on certain conditions.
- `set_ability(self, card, initial, delay_sprites)`
	- Set up initial ability values or manipulate sprites in an advanced way.
-  `in_pool(self, args) -> bool, { allow_duplicates = bool }`
	- Define custom logic for when a card is allowed to spawn. A card can spawn if `in_pool` returns true and all other checks are met.
	- When called from `generate_card_ui`, the `_append` key is passed as `args.source`.
- `update(self, card, dt)`
	- For actions that happen every frame.
- `set_sprites(self, card, front)`
	- For advanced sprite manipulation that happens when a card is created or loaded.
- `set_badges(self, card, badges)`
	- Add additional badges, leaving existing badges intact. This function doesn't return; add badges by appending to `badges`.
	- Avoid overwriting existing elements. It will cause text to appear on the top left corner of your screen instead.
	- Function for creating badges: `create_badge(_string, _badge_col, _text_col, scaling)`
		- `_string`: Text displayed on the badge.
		- `_badge_col = G.C.GREEN`: Background colour.
		- `_text_col = G.C.WHITE`: Text colour.
		- `_scaling = 1`: Relative size of the badge.
	- Example:
	```lua
	{
		set_badges = function(self, card, badges)
			badges[#badges+1] = create_badge(localize('k_your_string'), G.C.RED, G.C.BLACK, 1.2 )
		end,
	}
	```
- `set_card_type_badge(self, card, badges)`
	- Same as `set_badges`, but bypasses creation of the card type / rarity badge, allowing you to replace it with a custom one.
- `draw(self, card, layer)`
	- Draws the sprite and shader of the card.

## Util methods
- `SMODS.poll_enhancement(args)` *(defaults)*
	- `args.key`, the key used to generate the seed
    - `args.type_key`, an optional key used to generate the specific enhancement seed
    - `args.mod`, multiplying modifier to the base rate (40%)
    - `args.guaranteed`, if `true`, enhancement is guaranteed
    - `args.options`, can be used to provide fixed options to the pool
        - `{ keys }`, a table of keys as strings, respects original weight values
        - `{ {key = string, weight = number} }`, a table of tables, with key and weight pairs
- `SMODS.get_enhancements(card, extra_only)`: Returns a table indexed by keys of enhancements that the given card has.
    - `Card:calculate_joker` is called for each joker with `context = { check_enhancement = true, other_card = card }`, expecting return tables in the same format to add extra enhancements. No other ways to give a card multiple enhancements are currently supported.
    - If `extra_only == true`, the card's base enhancement is excluded.
- `SMODS.has_enhancement(card, key)`: Returns `true` if the given card has the specified enhancement, either as its natural enhancement or an extra enhancement from jokers.
- `SMODS.has_no_suit(card)`: Returns true if a card doesn't have any suit due to its enhancements (e.g., Stone Cards).
- `SMODS.has_any_suit(card)`: Returns true if a card can be used as any suit due to its enhancements (e.g., Wild Cards).
    - Cards with enhancement effects both for having no suit and for having any suit can be used as any suit.
- `SMODS.has_no_rank(card)`: Returns true if a card doesn't have any rank due to its enhancements (e.g., Stone Cards).
- `SMODS.always_scores(card)`: Returns true if a card always scores due to its enhancements (e.g., Stone Cards).
# API Documentation: `SMODS.Joker`
**Class prefix:** `j`
- **Required parameters:**
	- `key`,
	- `loc_txt` or localization entry [(reference)](https://github.com/Steamodded/smods/wiki/Localization)
- **Optional parameters** *(defaults)*:
    - `atlas = 'Joker', pos = { x = 0, y = 0 }` [(reference)](https://github.com/Steamodded/smods/wiki/SMODS.Atlas#applying-textures-to-cards)
    - `config = {}, unlocked = true, discovered = false, no_collection, prefix_config, dependencies, display_size, pixel_size` [(reference)](https://github.com/Steamodded/smods/wiki/API-Documentation#common-parameters)
	- `rarity = 1`
        - `1` = Common, `2` = Uncommon, `3` = Rare, `4` = Legendary
        - [Modded rarities](https://github.com/Steamodded/smods/wiki/SMODS.Rarity) are specified using their key (including your mod prefix)
	- `cost = 3`,
	- `blueprint_compat = false` (**Purely cosmetic, you need to define your Joker's blueprint behavior in code**),
	- `eternal_compat = true`,
	- `perishable_compat = true`,
	- `<sticker>_compat` for any modded stickers

## API methods
- `calculate(self, card, context)` [(reference)](https://github.com/Steamodded/smods/wiki/Calculate-Functions)
- `loc_vars, locked_loc_vars, generate_ui` [(reference)](https://github.com/Steamodded/smods/wiki/Localization#Localization-functions)
- `calc_dollar_bonus(self, card) -> number`
	- For awarding money at the end of the round (e.g. Delayed Gratification, Cloud Nine)
- `set_ability(self, card, initial, delay_sprites)`
	- Set up initial ability values or manipulate sprites in an advanced way.
- `add_to_deck(self, card, from_debuff)`
	- Modify the game state when this card is obtained.
	- Cards are considered added when they become undebuffed (in this case, `from_debuff` will be true).
- `remove_from_deck(self, card, from_debuff)`
	- Modify the game state when this card is sold, destroyed, or otherwise removed.
	- Cards are considered removed when debuffed (in this case, `from_debuff` will be true).
- `in_pool(self, args) -> bool, { allow_duplicates = bool }`
	- Define custom logic for when a card is allowed to spawn. A card can spawn if `in_pool` returns true and all other checks are met.
	- `allow_duplicates` allows this card to spawn when one already exists, even without Showman.
	- When called from `generate_card_ui`, the `_append` key is passed as `args.source`.
- `update(self, card, dt)`
	- For actions that happen every frame.
- `set_sprites(self, card, front)`
	- For advanced sprite manipulation that happens when a card is created or loaded.
- `load(self, card, card_table, other_card)`
	- For modifications to sprites or the card itself when a run is reloaded.
- `check_for_unlock(self, args) -> bool`
	- Configure unlock conditions. Refer to the function `check_for_unlock` in Balatro's code for more information.
- `set_badges(self, card, badges)`
	- Add additional badges, leaving existing badges intact. This function doesn't return; add badges by appending to `badges`.
	- Avoid overwriting existing elements. It will cause text to appear on the top left corner of your screen instead.
	- Function for creating badges: `create_badge(_string, _badge_col, _text_col, scaling)`
		- `_string`: Text displayed on the badge.
		- `_badge_col = G.C.GREEN`: Background colour.
		- `_text_col = G.C.WHITE`: Text colour.
		- `_scaling = 1`: Relative size of the badge.
	- Example:
	```lua
	{
		set_badges = function(self, card, badges)
			badges[#badges+1] = create_badge(localize('k_your_string'), G.C.RED, G.C.BLACK, 1.2 )
		end,
	}
	```
- `set_card_type_badge(self, card, badges)`
	- Same as `set_badges`, but bypasses creation of the card type / rarity badge, allowing you to replace it with a custom one.
- `draw(self, card, layer)`
	- Draws the sprite and shader of the card.
# API Documentation: `SMODS.Keybind`
- **Required parameters:**
	- `key_pressed`: The key that needs to be pressed for this keybind to activate. Keycodes are documented [here](https://love2d.org/wiki/KeyConstant).
    - `action(self)`: Function to be called when the keybind is triggered.
- **Optional parameters** *(defaults)*:
    - `event = 'pressed'`: Defines when the keybind should trigger.
        - `'pressed'`: When the key is pressed.
        - `'released'`: When the key is released.
        - `'held'`: When the key has been held for a specified amount of time.
    - `held_duration = 1`: How long (in seconds) the key must be held before triggering when `event == 'held'`.
    - `held_keys = {}`: Set of keycodes that must also be pressed for the keybind to activate. 
    - `key`: Because there is no need to explicitly refer to a keybind elsewhere, it is not required to specify an object key here. The default value is the amount of previously registered keybinds as a string.
# API Documentation: `SMODS.Language`
- **Required parameters:**
	- `key` (does not get prefixed by default)
    - `label`: The label displayed on the language selection screen
- **Optional parameters** *(defaults)*:
	- `font = 1`: When a number is specified, use the corresponding font provided by the game.
        - 1: m6x11plus (Latin alphabet)
        - 2: NotoSansSC-Bold (Simplified Chinese)
        - 3: NotoSansTC-Bold (Traditional Chinese)
        - 4: NotoSansKR-Bold (Korean)
        - 5: NotoSansJP-Bold (Japanese)
        - 6: NotoSans-Bold (Used in-game for Russian)
        - 7: Also m6x11plus (for some reason)
        - 8: GoNotoCurrent-Bold (unused asset)
        - 9: GoNotoCJKCore (unused asset)
        - You can also specify a table to use your own font. It should look something the following. The file is expected to be found in the `assets/fonts` subdirectory within your mod.
        ```lua
        {
            file = "myfont.ttf",
            render_scale = G.TILESIZE*10,
            TEXT_HEIGHT_SCALE = 0.83,
            TEXT_OFFSET = {x=10,y=-20},
            FONTSCALE = 0.1,
            squish = 1,
            DESCSCALE = 1
        },
        ```
    - `loc_key`: Treats the language with the given key as a base for this one, keeping any unchanged localization strings intact and adding changes from your localization and fonts, where applicable.

You should place a localization file for your language's translation in a file at `localization/[key].lua` within your mod files, where `[key]` is the language's key. Steamodded will also load files named this way from other mods if the language is active. Non-existent entries default to English text.
# API Documentation: `SMODS.ObjectType`
- **Required parameters:**
	- `key`
- **Optional parameters** *(defaults)*:
	- `default`: Fallback card when object pool is empty
	- `cards`: List of keys to centers to auto-inject into this ObjectType
		-  Expects a list of keys like this:
		```lua
			{
				["j_foo_joker"] = true
				["c_bar_tarot"] = true
			}
		```
	- `rarities` allows cards to show up at different rates.
		-  Expects a list of tables like this:
		```lua
			{
				key = '',
				rate = 0.5,
			}
		```
		- The sum of all rates is normalized internally, so you don't need to concern yourself with making these values add up to `1`. Not defining a rate will use the default rate assigned by the rarity. 

## API Documentation: `SMODS.ConsumableType`
This is a subclass of `SMODS.ObjectType`. All values and functions tied to `SMODS.ObjectType` work for this class. 
- **Required parameters:**
	- `key`
	- `primary_colour`
	- `secondary_colour`
- **Optional parameters** *(defaults)*:
	- `loc_txt`, Skeleton:
	```lua
		{
			name = '', -- used on card type badges
			collection = '', -- label for the button to access the collection
			undiscovered = { -- description for undiscovered cards in the collection
				name = '',
				text = { '' },
			},
		}
	```
	- `collection_rows = { 6, 6 }`: Customize the collection for this card type. Each value indicates a row with the specified amount of cards.
	- `shop_rate`: Setting a numerical value for `shop_rate` enables cards of this type to appear in the shop at the specified rate.
		- This sets a default rate, which can be modified by accessing `G.GAME[key:lower() .. '_rate']` during a run.

## API methods
- `ObjectType.inject_card(self, center)`
	- Used for modifying any registered cards of this type, or adding them to additional pools.
- `ObjectType.delete_card(self, center)`
	- Used for removing cards from additional pools when deleted.
- `ConsumableType.create_UIBox_your_collection(self) -> table`
	- Returns the UIBox of the ConsumableType's collections menu. 

# API Documentation: `SMODS.UndiscoveredSprite`
For consumable types, a sprite for undiscovered objects can be registered. Otherwise, the default undiscovered Joker sprite is used. The keys of both objects should match exactly. Remember to use `prefix_config.atlas = false` if you want to reference an atlas from the base game.
- **Required parameters:**
	- `key`
	- `atlas`
	- `pos`
   - **Optional parameters:**
  	- `no_overlay`: disables the floating ? sprite from undiscovered objects
  	- `overlay_pos`: customize the floating ? sprite using your atlas. Expects `{x = 0, y = 0}`
# API Documentation: `SMODS.PokerHand`
- **Required parameters:**
	- `key`
	- `mult`, `chips`: Base Mult and Chips
    - `l_mult`, `l_chips`: Amount of Mult/Chips scaling per hand level
    - `example`: Hand example to show in the Run Info tab, format:
    ```lua
        {
            { 'S_K', false }, -- King of Spades, does not score
            { 'S_9', true }, -- 9 of Spades, scores
            { 'D_9', true }, -- 9 of Diamonds, scores
            { 'H_6', false }, -- 6 of Hearts, does not score
            { 'D_3', false } -- 3 of Diamonds, does not score
        }
    ```
    - `evaluate`
    - `loc_txt` or localization entry [(reference)](https://github.com/Steamodded/smods/wiki/Localization)
        - Rather than a `text` table, `loc_txt` should contain a `description` table. The description is displayed when viewing the hand in the Run Info menu. When using localization files, the name should be placed in `misc.poker_hands[key]`, the description should be placed in `misc.poker_hand_descriptions[key]`.
- **Optional parameters** *(defaults)*:
    - `prefix_config, dependencies` [(reference)](https://github.com/Steamodded/smods/wiki/API-Documentation#common-parameters)
	- `visible = true`: Is this hand visible in the poker hands menu from the start, or is it hidden until played for the first time?
    - `above_hand`: Sets the position in the hands menu above the specified hands. By default, hands are ordered by the product of their chips and mult.
    - `order_offset`: If this numeric parameter is specified, add its value to the product of chips and mult for the purpose of ordering.

## API methods
- `evaluate(parts, hand) -> table`
    - This function is used to determine if the played cards contain this hand, and which cards it's made up of.
    - See the next section for an explanation of parts.
    - The returned table should be an array that contains a table of all scoring cards, e.g.,
        - `{ hand }` if all cards score,
        - `{ SMODS.merge_lists(parts._flush, parts._straight) }` for all scoring cards that are part of a straight or flush, or
        - `{}` if the cards don't contain this hand.

## Utility functions
- `SMODS.merge_lists(...) -> table`
    - Takes any amount of parts and flattens them into an array containing all scoring cards present in any of the parts. This is particularly useful for composite hands.

# API Documentation: `SMODS.PokerHandPart`
This utility class allows easily re-using the same poker hand constructs to build composite hands without much boilerplate code. Before all hand types get evaluated, each part gets executed and added to `parts` for access in the `evaluate` functions.
- **Required parameters:**
    - `key`: *Keep in mind that your mod prefix is prepended to this!*
    - `func(hand) -> table`: This expects the same return value format as `evaluate` on poker hands.
- **Pre-existing parts:**
    - `parts._2, parts._3, parts._4, parts._5` **contain all groups of at least 2/3/4/5 cards that share the same rank.**
        - `parts._all_pairs` **provides a list of all cards that are part of at least one pair in the hand.** (This is equivalent to `SMODS.merge_lists(parts._2)`.)
        - `parts._flush` **corresponds to a Flush being present in the hand.**
        - `parts._straight` **corresponds to a Straight being present in the hand.**
        - `parts._highest` **holds the card with the highest nominal value**.
# API Documentation: `SMODS.Rarity`
- **Required parameters:**
    - `key`
    - `loc_txt` or localization entry [(reference)](https://github.com/Steamodded/smods/wiki/Localization)
        - The only supported field is `name`. In localization files, it must be set as both `misc.labels['k_'..key:lower()]` and `misc.dictionary['k_'..key:lower()]`.
- **Optional parameters** *(defaults)*:
    - `pools`: Table with a list of ObjectType keys to add this rarity to. Skeleton:
    ```lua
    {
        ["Joker"] = true, --uses self.default_rate when polled
        ["Joker"] = { rate = 0.7 },
    }
    ```
    - `badge_colour`: Colour of the rarity's badge.
    - `default_weight`: Setting a numerical value for `default_weight` enables cards with this rarity to appear in the shop at the specified weight.
        - This sets a default weight, which can be modified by accessing `G.GAME[key:lower() .. '_mod']` during a run.

## API methods
- `get_weight(self, weight, object_type) -> number`
    - Returns weight. Use for finer control over the rarity's weight. 
- `gradient(self, dt)`
    - Used to control the badge colour gradient.

## Util methods
- `SMODS.poll_rarity(_pool_key, _rand_key) -> key`
    - Polls all rarities tied to `_pool_key` and returns rarity key. 
# API Documentation: `SMODS.Seal`
- **Required parameters:**
	- `key`,
    - `loc_txt` or localization entry [(reference)](https://github.com/Steamodded/smods/wiki/Localization)
        - For use with localization file, the description should be set as `descriptions.Other[key:lower()..'_seal']`.
		- `loc_txt` should contain an additional `label` string. It is used on badges, while `name` is displayed at the top of info boxes. For use with localization files, this label should be set as `misc.labels[key:lower()..'_seal']`.
- **Optional parameters** *(defaults)*:
    - `atlas = 'Joker', pos = { x = 0, y = 0 }` [(reference)](https://github.com/Steamodded/smods/wiki/SMODS.Atlas#applying-textures-to-cards)
    - `config = {}, discovered = false, no_collection, prefix_config, dependencies` [(reference)](https://github.com/Steamodded/smods/wiki/API-Documentation#common-parameters)
        - Values in `config` are copied to `card.ability.seal` when the seal is applied to `card`.
    - `badge_colour = HEX('FFFFFF')`
    - `sound = { sound = 'gold_seal', per = 1.2, vol = 0.4 }`: The sound that should play when the seal is applied to a card.
        - `sound`: The key of the sound to play.
        - `per`: The pitch at which the sound should be played.
        - `vol`: The volume at which the sound should be played.

## API methods
- `calculate(self, card, context)` [(reference)](https://github.com/Steamodded/smods/wiki/Calculate-Functions)
- `loc_vars, generate_ui` [(reference)](https://github.com/Steamodded/smods/wiki/Localization#Localization-functions)
- `get_p_dollars(self, card) -> number`
    - Gives money when a card with this seal is played.
- `update(self, card, dt)`
    - For actions that happen every frame.
- `draw(self, card, layer)`
	- Draws the sprite and shader of the seal.
# API Documentation: `SMODS.Sound`
This class allows you to play custom sounds and music tracks or replace existing ones. Your mod must be located in its own subdirectory of the `Mods` folder. The file structure should look something like this:
```bash
Mods
└──Cryptid
	├── Cryptid.lua
	└── assets
		└── sounds
			├── e_mosaic.ogg
			├── music_big.ogg
			└── music_jimball.ogg
```
- **Required parameters:**
	- `key`
    - `path`: The sound file's name, including the extension (e.g. `'music_jimball.ogg'`).
		- Supported file types include `*.ogg`, `*.wav` and `*.mp3`. It is not recommended to use MP3 files.
		- If you want to use different sound files depending on the selected language, you can also provide a table:
		```lua
		path = {
			['default'] = 'my_sound.ogg',
			['nl'] = 'my_sound-nl.ogg',
			['pl'] = 'my_sound_pl.ogg',
		}
		```
- **Optional parameters** *(defaults)*:
	- `prefix_config, dependencies` [(reference)](https://github.com/Steamodded/smods/wiki/API-Documentation#common-parameters)
    - `pitch = 0.7`: Specify a custom pitch shift for music tracks.
	- `volume = 0.6`: Specify a custom volume for music tracks.
		- These modifiers do not apply to regularly played sounds. Specify them as arguments to `play_sound` instead.
	- `replace`: Replace the specified sound with this one whenever it is played. Behaves like `self:create_replace_sound(replace)`.
	- `sync`: For music tracks only - configuration for synchronizing different music tracks.
		- Default behavior: sync with all tracks that aren't configured otherwise.
		- `sync = false`: Do not sync with anything.
		- If provided a table, only try to sync with keys that correspond to a true value, e.g.:
			```lua
			sync = {
				['mymusic1'] = true,
				['mymusic2'] = true,
			}
			```
		- You can use metatables to exclude certain tracks but include everything else:
			```lua
			sync = setmetatable({
				['somemusic1'] = false,
				['somemusic2'] = false,
				['somemusic3'] = false,
			}, { __index = function() return true end })
			```
		- In any case, syncing is only possible when both tracks agree on it. It is recommended synced tracks have the same length

## API methods
- `select_music_track(self) -> number`
	- This function is called each frame to decide what music to play. Return values that are not `nil`, `false` or a number are converted to zero. The music track that returned the highest value is to be played.
		- Your track's sound code must contain the string `music` if you are using this.

## Utility functions
- `create_replace_sound(self, replace)`
	- If `replace` is a string, indefinitely replace it with this one until it is replaced again.
	- If `replace` is a table, the key of the sound being replaced is `replace.key`. Specify a number `replace.times` to temporarily replace it that amount of times. Additional arguments may be passed as `replace.args`.
- [STATIC] `SMODS.Sound:create_stop_sound(key, times)`
	- Supress sounds with the given sound code `times` amount of times or indefinitely.
- [STATIC] `SMODS.Sound:register_global()`
	- Register all sound files found in the `assets/sounds` directory of the current mod. Use the file names without their extensions as keys.
- [STATIC] `SMODS.Sound:get_current_music()`
	- Polls `select_music_track` on all sound objects that have it, returns the key of the music to play.
	- You may override this function to take full control of the played music.

## Vanilla music tracks
The following overview provides a list of music tracks present in the base game and when they play. Custom music with conditions overrides all of these.
- `music1`: default track. Plays when none of the below apply
- `music2`: booster pack music. Plays for all vanilla boosters except Celestial Packs. Plays for all booster packs that have a sparkles effect or don't have a meteor effect.
- `music3`: booster pack music. Plays for Celestial Packs and any other packs with the meteor effect (but no sparkles effect).
- `music4`: shop music. Plays in the shop, outside of booster packs.
- `music5`: boss music. Plays during Boss Blinds.
# API Documentation: `SMODS.Stake`
**Class prefix:** `stake`
- **Required parameters:**
	- `key`
    - `applied_stakes`: An array of keys of stakes that should also be applied when this stake is active. This is evaluated recursively to include all stakes applied by applied stakes, so you usually don't need to specify multiple stakes here.
    - `loc_txt` or localization entry [(reference)](https://github.com/Steamodded/smods/wiki/Localization)
        - `loc_txt` should contain a `sticker` table that also consists of a `name` and `text`. It is used for the tooltip of the stake's win sticker.
        - When using localization files, the stake description should be placed in `descriptions.Stake[key]`, while the sticker description should be placed in `descriptions.Other[key:sub(7)..'_sticker']`, i.e., the `stake_` prefix is removed.
> [!IMPORTANT]
> An extra line that lists applied stakes is appended at the end of the description only when `loc_txt` is used. If you are using localization files, you should add this yourself.
- **Optional parameters** *(defaults)*:
    - `atlas = 'chips', pos = { x = 0, y = 0 }` [(reference)](https://github.com/Steamodded/smods/wiki/SMODS.Atlas#applying-textures-to-cards
    - `sticker_atlas, sticker_pos`: The atlas and position to use for this stake's win sticker.
    - `unlocked = false, prefix_config, dependencies` [(reference)](https://github.com/Steamodded/smods/wiki/API-Documentation#common-parameters)
        - If `unlocked` is set to `false`, the stake is unlocked by first winning a run on each of the `applied_stakes`.
    - `colour = [white]`: The colour used for this stake in the stake selection column.
    - `above_stake`: The stake's key that this stake should appear directly above in the list. By default, your stake will be placed at the top of the list.
> [!NOTE] 
> Key prefixing is applied to `applied_stakes` and `above_stake` by default. If you want your stake above a stake from the base game or other mods, this can be adjusted by using `prefix_config`. [(reference)](https://github.com/Steamodded/smods/wiki/API-Documentation#common-parameters)

## API methods
- `modifiers()`
    - Used for applying changes to the game state when your stake is applied at the start of a run.
# API Documentation: `SMODS.Sticker`
- **Required parameters:**
	- `key`
	- `loc_txt` or localization entry [(reference)](https://github.com/Steamodded/smods/wiki/Localization)
- **Optional parameters** *(defaults)*:
	- `atlas = 'stickers', pos = { x = 0, y = 0 }` [(reference)](https://github.com/Steamodded/smods/wiki/SMODS.Atlas#applying-textures-to-cards)
	- `badge_colour`: Colour of this sticker's badge.
    - `hide_badge`: If set to `true`, no badge is shown for this sticker.
	- `default_compat`: Default compatibility with cards. If `true`, all cards can have this sticker unless otherwise specified.
	- `compat_exceptions`: Array of keys of centers that have non-default compatibility with this sticker.
		- `default_compat = true`: sticker cannot be applied to centers in `compat_exceptions`
		- `default_compat = false`: sticker can only be applied to centers in `compat_exceptions`
	- `sets`, list of pools that this sticker is allowed to be applied on, format: 
	```lua
		{
			Joker = true
		}
	```
	- `rate = 0.3`: Chance of the sticker applying on an eligible card
	- `needs_enable_flag`: If set to `true`, this sticker requires `G.GAME.modifiers['enable_'..self.key]` to be `true` before it can be applied.

## API methods
- `calculate(self, card, context)` [(reference)](https://github.com/Steamodded/smods/wiki/Calculate-Functions)
- `loc_vars` [(reference)](https://github.com/Steamodded/smods/wiki/Localization#Localization-functions)
	- Due to some constraints, the functionality of `loc_vars` on stickers is limited. Out of all possible return values, only `vars`, `key` and `set` are supported.
- `should_apply(self, card, center, area, bypass_roll) -> bool`
	- Returns true if the sticker can be applied to the card. 
	- `bypass_roll` skips the RNG check and only looks for compatibility
- `apply(self, card, val)`
	- Handles applying and removing the sticker
	- Sets `card.ability[self.key]` to `val` by default. 
- `draw(self, card, layer)`
	- Draws the sprite and shader of the sticker.
# API Documentation: `SMODS.Suit`
- **Required parameters:**
	- `key`
    - `card_key`: Used to create keys for playing cards, formatted like `S_R`, where `S` is the suit's and `R` is the rank's card key. Your mod prefix gets prepended by default.
    - `pos`: This is a partial `pos` table that only needs a `y` coordinate. As such, your atlas should organize suits in rows.
    - `ui_pos`: Sprite position of the miniature suit symbol used in deck view.
    - `loc_txt` or localization entry [(reference)](https://github.com/Steamodded/smods/wiki/Localization)
        - `loc_txt` should contain a `singular` and `plural` string only. When using localization files, assign to `misc.suits_singular[key]` and `misc.suits_plural[key]` respectively.
- **Optional parameters** *(defaults)*:
    - `lc_atlas = 'cards_1'`: Atlas to use when high-contrast cards are disabled.
    - `hc_atlas = 'cards_2'`: Atlas to use when high-contrast cards are enabled.
    - `lc_ui_atlas = 'ui_1'`: Atlas for miniature suit symbols when high-contrast cards are disabled.
    - `hc_ui_atlas = 'ui_2'`: Atlas for miniature suit symbols when high-contrast cards are enabled.
    - `lc_colour = [white]`: Text colour when high-contrast cards are disabled.
    - `hc_colour = [white]`: Text colour when high-contrast cards are enabled.
    
# API Documentation: `SMODS.Rank`
- **Required parameters:**
	- `key`
    - `card_key`: Used to create keys for playing cards, formatted like `S_R`, where `S` is the suit's and `R` is the rank's card key. Your mod prefix gets prepended by default.
    - `pos`: This is a partial `pos` table that only needs an `x` coordinate. As such, your atlas should organize ranks in columns.
    - `nominal`: The amount of chips this rank should score.
    - `loc_txt` or localization entry [(reference)](https://github.com/Steamodded/smods/wiki/Localization)
        - `loc_txt` should contain only a `name` string. For localization files, place this string in `misc.ranks[key]`.
- **Optional parameters** *(defaults)*:
    - `lc_atlas = 'cards_1', hc_atlas = 'cards_2'`: Atlas to use for low-contrast and high-contrast settings respectively. Use the same atlas key if you don't have separate high contrast textures. [(reference)](https://github.com/Steamodded/smods/wiki/SMODS.Atlas#applying-textures-to-cards)
    - `shorthand = key`: Short descriptor used in deck preview.
    - `face_nominal`: Numeric value (normally between 0 and 1) that determines the displayed order of ranks with the same nominal value.
    - `face = false`: whether this rank counts as a face card.
    - `next = {}`: The keys contained in the `next` table are considered to come after this card, e.g. for the purpose of evaluating straights.
    - `strength_effect = { fixed = 1 }`: Determines how cards of this rank behave when Strength is used.
        - If `strength_effect.fixed` is a numeric value and `next` indexed at that value is a valid rank's key, always convert into that rank.
        - If `strength_effect.random` is `true`, choose a random rank from `next`.
        - If `strength_effect.ignore` is `true` or none of the above apply, do nothing.
    - `straight_edge = false`: If this is `true`, this card behaves like an Ace in straights, i.e., it can only be used as the lowest- or highest-order card in the hand.
    - `suit_map = { Hearts = 0, Clubs = 1, Diamonds = 2, Spades = 3 }`
        - For any suit keys present as keys in `suit_map`, prefer using this rank's atlas over the suit's atlas. The value at the suit's key will be used as each sprite's `x` position instead of the one specified by the suit.
        - This indicates that you provide sprites for certain suits with your rank. Combinations of suits and ranks where neither side supports the other, blank sprites are used instead.

## API methods
These methods are available for both suits and ranks.
- `loc_vars` [(reference)](https://github.com/Steamodded/smods/wiki/Localization#Localization-functions)
    - This method provides very limited functionality compared to its counterpart on other classes. It has no support for any return values, but it does allow you to add tooltips to `info_queue`.
- `draw(self, card, layer)`
	- Draws additional sprites or shaders on cards.

## Utility: pools and randomness
- Suits and Ranks have special support for `in_pool` with an extended argument signature: `in_pool(self, args)`. `args.initial_deck` indicates when a starting deck is being generated. This can be used to set rules for when your cards should be added to starting decks independently of whether they can show up in other places. 
- Even though `SMODS.Suits` and `SMODS.Ranks` are unordered tables, it is possible to feed them directly into `pseudorandom_element` to get a random suit or rank while respecting `in_pool`. Example: `local rank = pseudorandom_element(SMODS.Ranks, pseudoseed('myrank'))`.
# API Documentation: `SMODS.Tag`
**Class prefix:** `tag`
- **Required parameters:**
    - `key`
    - `loc_txt` or localization entry [(reference)](https://github.com/Steamodded/smods/wiki/Localization)
- **Optional parameters** *(defaults)*:
    - `atlas = 'tags', pos = { x = 0, y = 0 }` [(reference)](https://github.com/Steamodded/smods/wiki/SMODS.Atlas#applying-textures-to-cards)
    - `config = {}, discovered = false, no_collection, prefix_config, dependencies` [(reference)](https://github.com/Steamodded/smods/wiki/API-Documentation#common-parameters) 
    - `min_ante`: Minimum ante needed for this tag to appear. Use in_pool for more advanced spawn restrictions instead.

## API methods
- `loc_vars, generate_ui` [(reference)](https://github.com/Steamodded/smods/wiki/Localization#Localization-functions)
- `in_pool(self, args) -> bool, { allow_duplicates = bool }`
	- Define custom logic for when a tag is allowed to spawn. A tag can spawn if `in_pool` returns true and all other checks are met.
	- When called from `generate_card_ui`, the `_append` key is passed as `args.source`.
- `apply(self, tag, context)`
    - This function defines the tag's behavior when triggering. Unlike vanilla tags, this function is not restricted by `self.config.type` matching `context.type`, meaning you have to check context manually.
        - To trigger the tag, you should call `tag:yep(message, colour, func)` and set `tag.triggered = true`.
        - `message` is the string to display upon trigger,
        - `colour` is the colour of the message box,
        - `func` is a function that gets executed as an event before the used tag gets removed. This function should always return `true`.
- `set_ability(self, tag)`
    - This function allows you to store any additional information your tag may need when it is created. Values should be stored within `tag.ability`.

# API Documentation: `SMODS.Voucher`
**Class prefix:** `v`
- **Required parameters:**
	- `key`,
	- `loc_txt` or localization entry [(reference)](https://github.com/Steamodded/smods/wiki/Localization)
- **Optional parameters** *(defaults)*:
    - `atlas = 'Voucher', pos = { x = 0, y = 0 }` [(reference)](https://github.com/Steamodded/smods/wiki/SMODS.Atlas#applying-textures-to-cards)
    - `config = {}, unlocked = true, discovered = false, no_collection, prefix_config, dependencies, display_size, pixel_size` [(reference)](https://github.com/Steamodded/smods/wiki/API-Documentation#common-parameters)
	- `cost = 10`,
    - `requires`: specify a list of one or more vouchers by their **full key** (e.g. `'v_grabber'` for vanilla vouchers, or `'v_pref_myvoucher'` for a modded voucher from the mod with prefix `'pref'`)

## API methods
- `calculate(self, card, context)` [(reference)](https://github.com/Steamodded/smods/wiki/Calculate-Functions)
- `loc_vars, locked_loc_vars, generate_ui` [(reference)](https://github.com/Steamodded/smods/wiki/Localization#Localization-functions)
- `redeem(self, card)`
	- Defines the behavior of a Voucher when redeemed.
- `set_ability(self, card, initial, delay_sprites)`
	- Set up initial ability values or manipulate sprites in an advanced way.
- `add_to_deck(self, card, from_debuff)`
	- Modify the game state when this card is obtained.
	- Cards are considered added when they become undebuffed (in this case, `from_debuff` will be true).
- `remove_from_deck(self, card, from_debuff)`
	- Modify the game state when this card is sold, destroyed, or otherwise removed.
	- Cards are considered removed when debuffed (in this case, `from_debuff` will be true).
- `in_pool(self, args) -> bool, { allow_duplicates = bool }`
	- Define custom logic for when a card is allowed to spawn. A card can spawn if `in_pool` returns true and all other checks are met.
	- `allow_duplicates` allows this card to spawn when one already exists, even without Showman.
	- When called from `generate_card_ui`, the `_append` key is passed as `args.source`.
- `update(self, card, dt)`
	- For actions that happen every frame.
- `set_sprites(self, card, front)`
	- For advanced sprite manipulation that happens when a card is created or loaded.
- `load(self, card, card_table, other_card)`
	- For modifications to sprites or the card itself when a run is reloaded.
- `check_for_unlock(self, args) -> bool`
	- Configure unlock conditions. Refer to the function `check_for_unlock` in Balatro's code for more information.
- `set_badges(self, card, badges)`
	- Add additional badges, leaving existing badges intact. This function doesn't return; add badges by appending to `badges`.
	- Avoid overwriting existing elements. It will cause text to appear on the top left corner of your screen instead.
	- Function for creating badges: `create_badge(_string, _badge_col, _text_col, scaling)`
		- `_string`: Text displayed on the badge.
		- `_badge_col = G.C.GREEN`: Background colour.
		- `_text_col = G.C.WHITE`: Text colour.
		- `_scaling = 1`: Relative size of the badge.
	- Example:
	```lua
	{
		set_badges = function(self, card, badges)
			badges[#badges+1] = create_badge(localize('k_your_string'), G.C.RED, G.C.BLACK, 1.2 )
		end,
	}
	```
- `set_card_type_badge(self, card, badges)`
	- Same as `set_badges`, but bypasses creation of the card type / rarity badge, allowing you to replace it with a custom one.
- `draw(self, card, layer)`
	- Draws the sprite and shader of the card.
# Basic Structure

The user interface of Balatro most closely resembles the CSS `flexbox` layout, albeit with fewer features.
In Balatro, the basic building block of the UI is `UIElement` (aka. node), which is a table with the following fields:

 - `n`: stands for 'node (type)' and determines how the element will handle its contents
 - `config`: the properties of the node, akin to CSS properties
 - `nodes`: the nodes *inside* this node (aka. children nodes)

A basic example looks like this:

```lua
-- Example node:
{n = G.UIT.C, config = {align = "cm", padding = 0.1}, nodes = {
   {n = G.UIT.T, config = {text = "Hello, world!", colour = G.C.UI.TEXT_LIGHT, scale = 0.5}}
}}
```

Now, these `UIElement` objects would be very finicky to render and update one-by-one.
Therefore, they are packaged into `UIBox` objects.
Each `UIBox` is a complete UI feature like a menu, or a button.
Note that a `UIBox` can have other `UIBox` objects inside it &ndash; a `UIBox` for a menu will usually contain some `UIBox` objects for buttons (among many `UIElement` objects).
Hopefully, all of this will become more clear as this guide continues.

> [!TIP]
> As a metaphor, you can think of `UIElement` as a function, and `UIBox` as a class.<br>
> In reality, both of them are objects, but the structure/hierarchy between them is similar.

## Node Types

The only values that the `n` field can take:

 - `G.UIT.ROOT`: the top-level node of *every* `UIBox`; there's exactly one of these per `UIBox`.
 - `G.UIT.R`: a **Row** node, which arranges its child nodes *horizontally*.
 - `G.UIT.C`: a **Column** node, which arranges its child nodes *vertically*.
 - `G.UIT.T`: a **Text** node, which displays text.<br>
   This node must contain `text`, `colour`, and `scale` in its `config`.
 - `G.UIT.O`: an **Object** node, which displays a special game object.<br>
   Game objects include `Sprite`, `Card`, `CardArea`, `DynaText` (ie. dynamic text), and more.
 - `G.UIT.B`: a **Box** node, which is used as a spacer; it *ignores any child nodes*.
   This node must contain `h` and `w` in its `config`.
 - `G.UIT.S`: a **Slider** node, which contains a slider input.<br>
   You should probably use `create_slider(..)` instead.
 - `G.UIT.I`: a text **Input** node, which contains text input.

*Example*
The following code uses a combination of columns and rows to build up the basic structure of the UI. It can then have other elements added to add content.
```lua
{n = G.UIT.ROOT, config = {r = 0.1, minw = 8, minh = 6, align = "tm", padding = 0.2, colour = G.C.BLACK}, nodes = {
	{n = G.UIT.C, config = {minw=4, minh=4, colour = G.C.MONEY, padding = 0.15}, nodes = {
		{n = G.UIT.R, config = {minw=2, minh=2, colour = G.C.RED, padding = 0.15}, nodes = {
			{n = G.UIT.C, config = {minw=1, minh=1, colour = G.C.BLUE, padding = 0.15}},
			{n = G.UIT.C, config = {minw=1, minh=1, colour = G.C.BLUE, padding = 0.15}}
		}},
		
		{n = G.UIT.R, config = {minw=2, minh=1, colour = G.C.RED, padding = 0.15}, nodes = {
			{n = G.UIT.C, config = {minw=1, minh=1, colour = G.C.BLUE, padding = 0.15}},
			{n = G.UIT.C, config = {minw=1, minh=1, colour = G.C.BLUE, padding = 0.15}}
		}}
	}}
}}
```

![UI Example](https://i.imgur.com/0HMQObb.png)

## Node Configuration

The `config` field expects a table of configuration entries.
There are *a lot* of different keys that the game recognises in different contexts, but there are only a few that you need to know in order to build working UIs:

 - `align`: where the child nodes are placed within the current node; it always consists of *exactly two letters*:<br>
   the first letter is for *vertical* alignment: either `t` for Top, `c` for Center, or `b` for Bottom;<br>
   the second letter is for *horizontal* alignment: either `l` for Left, `m` for Middle, or `r` for Right.
 - `h`, `minh`, `maxh`: the Height of this node (either fixed, minimum, or maximum)
 - `w`, `minw`, `maxw`: the Width of this node (either fixed, minimum, or maximum)
 - `padding`: the extra space inside the edges of the current node.<br>
   Standard values are `0.05` or `0.1`.
 - `r`: the Roundness of corners of the current node.<br>
   Standard value is `0.1`.
 - `colour`: the fill colour of the current node.<br>
   Standard values are stored in the `G.C` table; fully custom values can be used with `HEX("00000000")` (taking RGBA values).<br>
   The British spelling is important!
 - `no_fill`: set to `true` for no fill.
 - `outline`: the thickness of the outline of the current node.<br>
   Standard value is `???`.
 - `outline_colour`: the colour of the outline of the current node.<br>
   See `colour` for possible values, above.
 - `emboss`: how much the current node is *raised* out of its parent node.<br>
   Standard value is `0.05`.
 - `hover`: set to `true` to render the current node as hovering *above* its parent node.
 - `shadow`: set to `true` to render the shadow of the current node; only shows-up under hovering nodes.
 - `juice`: set to `true` to apply the `juice_up` animation to the current node once it loads in.

Some advanced configuration keys include:

 - `id`: set a custom ID to the current node.<br>
   This will allow you to find this specific node via `[UIBOX]:get_UIE_by_ID([ID])`;
   note that you must already know some `UIBox` element that *contains* this node.<br>
   This is only useful in very specific cases, and for simple UIs you should not need to use this.
 - `instance_type`: set the layer that the current node is drawn on, either:<br>
   `NODE`, `MOVEABLE`, `UIBOX` (w/o `attention_text`), `CARDAREA`, `CARD`, `UI_BOX` (w/ `attention_text`), `ALERT`, or `POPUP`<br>
   These are ordered from lowest to highest layer.
 - `ref_table`: a table containing some data that is relevant to the current node.<br>
   This is used to pass data to UI nodes or between UI-related functions.
 - `ref_value`: a string corresponding to a key in the current node's `ref_table`.<br>
   This is always used in conjunction with `ref_table` to access the relevant value by key.
 - `func`: set a function that will be called when the current node is being drawn.<br>
   Its value is a string of the function name; the function itself must be stored in `G.FUNCS`.
 - `button`: set a function that will be called when the current node is clicked on.<br>
   Its value is a string of the function name; the function itself must be stored in `G.FUNCS`.
 - `tooltip`: add a tooltip when the current node is hovered over by a mouse/controller.<br>
   Its value is a table: `{title = "", text = {"Line1", "Line2"}}`.
 - `detailed_tooltip`: similar to tooltip above
 
> [!TIP]
> The **`ref_table`** and **`ref_value`** fields are extremely custom.
> Therefore, whenever you encounter them, you must manually trace what data they access and how it is used.

Configuration options for Text nodes (ie. `G.UIT.T`):

 - `text`: set the string to display.<br>
   Alternatively, you can set text via the `ref_table` and `ref_value` combination; for example:<br>
   `{n=G.UIT.T, config={ref_table = G.GAME.current_round, ref_value = "reroll_cost", scale = 0.75, colour = G.C.WHITE}`
 - `scale`: set a multiplier to text size.
 - `colour`: set the text colour.<br>
   Standard values are `G.C.UI.TEXT_LIGHT`, `G.C.UI.TEXT_DARK`, and `G.C.UI.TEXT_INACTIVE`,<br>
   but all values applicable to `colour` can also go here (see above).
 - `vert`: set to `true` to draw the text vertically (ie. sideways).
 
 > [!TIP]
 > The text contents of a `G.UIT.T` node **cannot be changed** interactively.<br>
 > You must either: (1) delete and re-create the node with new text, or (2) use a `DynaText` object.
 
Configuration optoins for Object nodes (ie. `G.UIT.O`):

 - `object`: set the object to render.<br>
   This is a literal game object, like `CardArea`.
 - `role`: set the relationship of the object to the UI.<br>
   Its value is a table containing at least: `{role_type = [TYPE]}` where type is either `"Major"`, `"Minor"`, or `"Glued"`.<br>
   You probably don't need to use this.

# How to Build UI

At this point, you should have a general idea about the structure of Balatro's UI.
Here are a few tips to actually making changes to it.

## The Basics

The most important thing to understand is the parent-child relationship between nodes.
A parent Row/Column node will arrange *all of its direct children* in Rows/Columns.
The children will be equally distributed (unless any specific width/height configuration is specified).

```lua
{n=G.UIT.C, config={align = "cm"}, nodes={      -- 0
  {n=G.UIT.R, config={align = "cr"}, nodes={}}, -- 1
  {n=G.UIT.R, config={align = "cm"}, nodes={}}, -- 2
  {n=G.UIT.R, config={align = "cl"}, nodes={}}  -- 3
}}

-- Result:
--   -------------------
--   |    1|  2  |3    |
--   -------------------
-- The WHOLE box containing 1,2,3 is Element 0
-- The contents of Elements 1,2,3 will go inside their respective small boxes.
-- The contents of Elements 1/3 will be right/left aligned, due to configuration.
```

```lua
{n=G.UIT.R, config={align = "cm"}, nodes={    -- 0
  {n=G.UIT.C, config={align = "cm"}, nodes={}}, -- 1
  {n=G.UIT.C, config={align = "cm"}, nodes={    -- 2
    {n=G.UIT.R, config={align = "cm"}, nodes={}}, -- 3
    {n=G.UIT.R, config={align = "cm"}, nodes={}}  -- 4
  }}
}}

-- Result:
--   -----------------
--   |       1       |
--   -----------------
--   |   3   |   4   |
--   -----------------
-- The WHOLE box is Element 0
-- The row containing 3,4 is Element 2
```

The biggest tip with regards to making a UI is to always **alternate Row nodes and Column nodes for each UI 'level'**.
The way that the UI is drawn, similar to CSS `flexbox`, stretches each UI element either in the Row direction or the Column direction.
Therefore, to make everything center-aligned, you need to stretch UI elements in both directions.

```lua
-- Always alternate Row and Column nodes between UI 'levels' (to keep center alignment):
{n=G.UIT.C, config={}, nodes={
  {n=G.UIT.R, config={}, nodes={
    {n=G.UIT.C, config={}, nodes={...}},
    {n=G.UIT.C, config={}, nodes={...}}
  }},
  {n=G.UIT.R, config={}, nodes={...}},
  {n=G.UIT.R, config={}, nodes={
    {n=G.UIT.C, config={}, nodes={
      {n=G.UIT.R, config={}, nodes={...}},
      {n=G.UIT.R, config={}, nodes={...}},
    }}
  }}
}}
```

## Creating a `UIBox`

A `UIBox` is a very useful tool if you need to create **interactive UIs**,
because a `UIBox` can be updated by deleting it and creating a new `UIBox` with updated values.
This is how Balatro's menus are updated.

It is created via a constructor and then used as a UI Object:

```lua
-- A simple UIBox being created:
local my_menu = UIBox({
   definition = my_menu_function(...),
   config = {type = "cm", ...}
})

-- A UIBox must be placed in an Object node!
local my_menu_node = {n=G.UIT.O, config={object = my_menu}}
```

The vital part of creating a `UIBox` is providing a **definition function**.
This function must simply **return a Root node** (ie. `G.UIT.ROOT`),
although it will most likely contain at least a few child nodes that hold some content.

```lua
function my_menu_function(menu_name)
   return {n=G.UIT.ROOT, config={align = "cm"}, nodes={
      -- Use a Row node to arrange the contents in rows:
      {n=G.UIT.R, config={align = "cm"}, nodes={
         -- Use a wrapper Column node to hold the menu name:
         {n=G.UIT.C, config={align = "cm"}, nodes={
            {n=G.UIT.T, config={text = menu_name, colour = G.C.UI.TEXT_LIGHT, scale = 0.5}}
         }},
         -- Use a wrapper Column node to hold the first menu row:
         {n=G.UIT.C, config={align = "cm"}, nodes={
            -- Menu Row 1 contents...
         }},
         -- Use a wrapper Column node to hold the second menu row:
         {n=G.UIT.C, config={align = "cm"}, nodes={
            -- Menu Row 2 contents...
         }},
         -- Etc...
      }}
   }}
end
```

## Interaction

The last vital component of Balatro's UI is interaction, and it is governed by buttons.
Consider the following button element:

```lua
{n=G.UIT.C, config={button = "my_button", my_data={1, 2, 3}}, nodes={
  {n=G.UIT.T, config={text = "Press Me!", ...}}
}}
```

Once this button is pressed, the game will call `G.FUNCS.my_button(e)`, where `e` is the whole button UI Element. That means you can access `e.config.my_data`, `e.nodes`, etc.

As a last example, here is one possible pattern to update the contents of a `UIBox`:

```lua
function G.FUNCS.my_update_menu(e)
  -- Get the menu UIBox object:
  local my_menu_uibox = e.config.my_data.menu_uibox
  -- Get the parent of the menu UIBox, because we want to delete and re-create the menu:
  local menu_wrap = my_menu_uibox.parent
  
  -- Delete the current menu UIBox:
  menu_wrap.config.object:remove()
  -- Create the new menu UIBox:
  menu_wrap.config.object = UIBox({
    definition = my_menu_function(e.config.my_data),
    config = {parent = menu_wrap, type = "cm"} -- You MUST specify parent!
  })
  -- Update the UI:
  menu_wrap.UIBox:recalculate()
end
```

## Helper Functions

The game's UI elements should ideally be consistent, which is why there are a few helper functions that
create and/or organise `UIBox` elements from templates.
   
```lua
UIBox_button(args)
create_toggle(args)
create_slider(args)
create_text_input(args)
simple_text_container(_loc, args)
create_UIBox_generic_options(args)
create_option_cycle(args)
```

These are not exhaustive, and you will have to figure out exactly how to use them by yourself.
This is meant to be an introductory guide, not Balatro's UI API documentation.

## The Details

The last couple sections may already seem quite complex.
It took hours of exploring the game's UI and experimenting with custom UIs to truly understand the dynamics of it all.
Unfortunately, there are a lot of different functions, tables, keys, and values that are used sporadically,
so the best anyone can do is to try different things and find patterns.

> [!IMPORTANT]
> In reality, the best way to learn Balatro's UI is to **copy its existing UI elements**.
> Everything you could ever need to design/change already exists in the game &ndash; you just need to find where it is defined and then explore how/why it works.

Once again, two good places to get a feel for building UI are [Divvy's Preview](https://github.com/DivvyCr/Balatro-Preview/blob/main/src/Interface.lua) and [Divvy's History](https://github.com/DivvyCr/Balatro-History/tree/main/src/UI) mods.

***
*Guide written by DivvyC, original notes by Eremel*
# Utility functions
Steamodded provides utility functions that extend or replace vanilla functionality or provide other useful tools you may need when making your mod. This page contains information on these functions. Check out `src/utils.lua` if you'd like to learn more about a function's implementation.

## Debugging 
- `inspect(table) -> string`
    - Given an input table, outputs a shallow mapping of keys to values as a string. Gives no further information on subtables.
- `inspectDepth(table) -> string`
    - Given an input table, recursively creates a string representation up to a depth of 5.
- `inspectFunction(func) -> string`
    - Given an input function, outputs details on its name, source, line of definition and number of upvalues.
- `serialize(t, indent) -> string`
    - Given an input table, creates a string containing Lua code that evaluates to the serializable part of `t`. Information may be lost, and circular references are not resolved.
- `tprint(t, indent) -> string`
    - Recursively stringifies an input table into pseudo-valid Lua code, leaving non-serializable values and tables above a depth of 5 into their default string representations.

## Number formatting
- `round_number(num_precision) -> number`
    - Rounds the input number to a given amount of decimal places.
- `format_ui_value(value) -> string`
    - Stringifies the input if it is not a number. Otherwise return the number as it should be displayed in the UI (e.g. scientific notation).

## Randomness
- `SMODS.poll_rarity(_pool_key, _rand_key) -> string|number` 
    - Generates a random rarity for the given pool (the only vanilla pool that supports rarities is `'Joker'`). If `_rand_key` is present, use it as a seed.
- `SMODS.poll_seal(args) -> string?`
    - Checks if a seal should be generated, and generates one according to the specified arguments if so. The following arguments are supported:
    - `key` - Randomness seed for the check whether to generate a seal, defaults to `'stdseal'`.
    - `mod` - Multiplier to the base rate at which seals appear.
    - `guaranteed` - If this is `true`, always generate a seal.
    - `options` - A table of possible seals to generate. Defaults to all available seals.
    - `type_key` - Randomness seed for the roll which seal to generate.
- `SMODS.poll_enhancement(args) -> string?`
    - Checks if an enhancement should be generated, and generates one according to the specified arguments if so. The following arguments are supported: 
    - `key` - Randomness seed for the check whether to generate an enhancement, defaults to `'std_enhance'`.
    - `mod` - Multiplier to the base rate at which enhancements appear.
    - `guaranteed` - If this is `true`, always generate an enhancement.
    - `options` - A table of possible enhancements to generate. Defaults to all available enhancements.
    - `type_key` - Randomness seed for the roll which enhancement to generate.


## Mod-facing utilities
These functions facilitate specific tasks that many mods may use but may be harder to achieve when implemented individually. Some replace base game functions to create a more usable interface.
- `SMODS.find_mod(id) -> table`
    - Returns a list of mods that either match the given mod ID or provide it, and are enabled and loaded. 
    - The result of `next(SMODS.find_mod(id))` can be used to determine if a mod is present, akin to finding cards with `SMODS.find_card`.
- `SMODS.load_file(path, id) -> function`
    - Given a path to a file in the context of the mod currently being loaded, loads the file contents and returns them as a function. If this function is called after the mod loading stage, a mod's `id` must be specified in order to find the correct file.
    - Return type is the same as [love.filesystem.load](https://love2d.org/wiki/love.filesystem.load) and [loadfile](https://www.lua.org/manual/5.1/manual.html#pdf-loadfile).
    - Example usage: `assert(SMODS.load_file('jokers.lua'))()`
- `SMODS.juice_up_blind()`
    - Plays a 'juice up' animation on the Blind chip.
- `SMODS.eval_this(card, effects)`
    - Given a `Card` object and a table that could be returned from a `calculate` function, evaluates the specified effects on this card while allowing the `calculate` function to continue on. You'll need to use this if your joker shows multiple consecutive messages in the main calculation phase.
    - Supported keys in `effects` include `mult_mod, chip_mod, Xmult_mod, message`, but you can extend the function to include your own effects.
- `SMODS.change_base(card, suit, rank) -> bool`
    - Given a `Card` representing a playing card, changes the card's suit, rank, or both. Either argument may be omitted to retain the original suit or rank.
    - This function returns `false` if it fails. It is recommended to always wrap calls to it in `assert` so errors don't go unnoticed.
    - Examples: `assert(SMODS.change_base(card, 'Hearts'))` converts a card into Hearts. `assert(SMODS.change_base(card, nil, 'Ace'))` converts a card into an Ace. `assert(SMODS.change_base(card, 'Hearts', 'Ace'))` converts a card into an Ace of Hearts.
- `SMODS.find_card(key, count_debuffed) -> table`
    - This function replaces `find_joker`. It operates using keys instead of names, which avoids overlap between mods.
    - Returns an array of all jokers or consumables in the player's possession with the given key. Debuffed cards count only if `count_debuffed` is true.
- `SMODS.add_card(t) -> Card`
    - This function replaces `add_joker`. It takes the same input parameters as `SMODS.create_card` (below) and additionally emplaces the generated card into its area and evaluates `add_to_deck` effects.
- `SMODS.create_card(t) -> Card`
    - This function replaces `create_card`. It provides a cleaner interface to the same functionality. The argument to this function should always be a table. The following fields are supported:
    - `set` - The card type to be generated, e.g. `'Joker'`, `'Tarot'`, `'Spectral'`
    - `area` - The card area this will be emplaced into, e.g. `G.jokers`, `G.consumeables`. Default values are determined based on `set`.
    - `legendary` - Set this to `true` to generate a card of Legendary rarity.
    - `rarity` - If this is specified, skip rarity polling and use this instead of a chance roll between 0 and 1.
        - Under vanilla conditions, values up to 0.7 indicate Common rarity, values above 0.7 and up to 0.95 indicate Uncommon rarity, and values above 0.95 indicate Rare rarity.
    - `skip_materialize` - If this is `true`, a `start_materialize` animation will not be shown.
    - `soulable` - If this is `true`, hidden Soul-type cards are allowed to replace the generated card. Usually, this is the case for cards generated for booster packs.
    - `key` - If this is specified, generate a card with the given key instead of the random one.
    - `key_append` - An additional RNG seeding string. This should be used to put different sources of card generation on different queues.
    - `no_edition` - If this is `true`, the generated card is guaranteed to have no randomly generated edition.
    - `edition`, `enhancement`, `seal` - Applies the specified modifier to the card.
    - `stickers` - This should be an array of sticker keys. Applies all specified stickers to the card.
- `SMODS.debuff_card(card, debuff, source)`
    - Allows manually setting and removing debuffs from cards.
    - `source` should be a unique identifier string. You must use the same source to remove a previously set debuff.
    - If `debuff` is the string `'reset'`, remove debuffs from all sources handled by this function.
    - If `debuff` is the string `'prevent_debuff'`, blocks all possible debuffs on the card until removed.
    - If `debuff` is another truthy value, set a debuff on the card.
    - If `debuff` is a falsy values, remove this debuff.
- `SMODS.recalc_debuff(card)`
    - Recalculates the debuff state of a card.
- `SMODS.merge_lists(...) -> table`
    - Takes any number of 2D arrays. Flattens the input into a 1D array that contains all elements once in the order they first appear and removes any duplicates.
    - This can be used to merge results from poker hand evaluation to create a combined hand.
- `SMODS.get_blind_amount(ante) -> number`
    - Provides score requirements for higher stages of ante scaling. If you want to implement your own scaling rules, you should modify this function.
- `SMODS.stake_from_index(index) -> string`
    - Given an index from the Stake pool, return the corresponding key, or `'error'` if it doesn't exist.
- `time(func, ...) -> number`
    - Calls an input function with any given additional arguments: `func(...)` and returns the time the function took to execute in milliseconds. The return value from `func` is lost.
## Internal utilities
These functions are used internally and may be of use to you if you're modifying Steamodded's injection process.
- `SMODS.save_d_u(o)`
    - Saves the default discovery and unlock states of an object into a separate field for later use before they are overwritten with profile data.
- `SMODS.SAVE_UNLOCKS()`
    - Modified from base game code. Sets discovery and unlock data according to profile state after injection.
- `SMODS.process_loc_text(ref_table, ref_value, loc_txt, key)`
    - Saves a localization entry (usually consisting of a name and a description) into the game's dictionary, with the ability to handle a multi-language format. The `loc_txt` table should either have locale indexing at the top layer or none at all. If the table holds multiple entries, `key` can be used to choose a key one layer down.
    - Example: `SMODS.process_loc_text(G.localization.descriptions.Joker, 'my_joker_key', loc_txt)`
- `SMODS.handle_loc_file(path)`
    - Given a mod's path, reads any present localization files and adds entries to the dictionary.
- `SMODS.insert_pool(pool, center, replace)`
    - If `replace` is true or `center` has been taken ownership of, look for an entry in `pool` with the same key and replace it. Otherwise, append `center` to the end of `pool`.
- `SMODS.remove_pool(pool, key)`
    - Find an entry in pool with this `key` and remove it if found.
- `SMODS.create_loc_dump()`
    - Dumps localization entries created during injection into a single file, for purposes of converting to a file-based localization system.
- `SMODS.create_mod_badges(obj, badges)`
    - UI code for adding mod badges to an object.
- `SMODS.merge_defaults(t, defaults) -> t`
    - Starting with `t`, insert any key-value pairs from `defaults` that don't already exist in `t` into `t`. Modifies `t` and returns it as the result of the merge.
    - `nil` entries are interpreted as an empty table; `false` inputs count as a table where every possible key maps to `false`. Therefore, `t == nil` is weak and falls back to defaults, while `t == false` explicitly ignores `defaults` in its entirety. Due to this, this function may not always return a table.
    - This is used for controlling when keys should receive prefixes.
- `convert_save_data()`
    - Adjusts values in save data to make the save compatible with Steamodded's way of storing deck wins by stake key instead of index.
- `SMODS.restart_game()`
    - Restarts the game.
# Making Your First Mod
This page is designed to help you find what is needed when first making a mod. If you are just looking on more clarifcation on something specific, you should check the sidebar to see if it's covered, or ask in the [Balatro Discord](https://discord.gg/balatro).

# First Steps 
- If you haven't already [Install Steamodded](https://github.com/Steamodded/smods/wiki).
- Checkout the [Mod Metadata](https://github.com/Steamodded/smods/wiki/Mod-Metadata) page for how to get your mod detected by Steamodded.
- Checkout the [API Documentation](https://github.com/Steamodded/smods/wiki/API-Documentation) page for information on the basics of Steamodded's api.
- For adding content, check the **Game Objects** part of the sidebar, which lists every object SMODS can create.

# Useful resources
- [The Lua Reference Manual](https://www.lua.org/manual/5.1/) and [Programming in Lua](https://www.lua.org/pil/contents.html) are very useful resources to familarize yourself with lua (the game's programming language).
- Often, something you want to do has already been implemented in the base game. Familiarizing yourself with the game's code is an important step to learn Balatro modding. To get Balatro's source code, extract the game's executable file with [7-zip](https://www.7-zip.org/). For Mac, find `Balatro.love` inside `Balatro.app` and rename it to `Balatro.zip`, then extract `Balatro.zip`. A handful of vanilla jokers have been reimplemented in a [Steamodded example mod](https://github.com/Steamodded/examples/tree/master/Mods/ExampleJokersMod) for reference.
- It can also be useful to look at code from other mod creators.
  - Steamodded has some [Example Mods](https://github.com/Steamodded/examples/tree/master/Mods).
  - The best place to find other mods is in the official [Balatro Discord](https://discord.gg/balatro).
- Lovely is a tool that lets you patch the balatro source code, and since it's necessary for Steamodded, Steamodded mods can take advantage of it too. See [Lovely's patch documentation](https://github.com/ethangreen-dev/lovely-injector?tab=readme-ov-file#patches).

  * [Home](https://github.com/Steamodded/smods/wiki)
***
Game Objects
  * [API Documentation](https://github.com/Steamodded/smods/wiki/API-Documentation)
  * [SMODS.Achievement](https://github.com/Steamodded/smods/wiki/SMODS.Achievement)
  * [SMODS.Atlas](https://github.com/Steamodded/smods/wiki/SMODS.Atlas)
  * [SMODS.Blind](https://github.com/Steamodded/smods/wiki/SMODS.Blind)
  * SMODS.Center
    * [SMODS.Back](https://github.com/Steamodded/smods/wiki/SMODS.Back)
    * [SMODS.Booster](https://github.com/Steamodded/smods/wiki/SMODS.Booster)
    * [SMODS.Consumable](https://github.com/Steamodded/smods/wiki/SMODS.Consumable)
    * [SMODS.Edition](https://github.com/Steamodded/smods/wiki/SMODS.Edition)
    * [SMODS.Enhancement](https://github.com/Steamodded/smods/wiki/SMODS.Enhancement)
    * [SMODS.Joker](https://github.com/Steamodded/smods/wiki/SMODS.Joker)
    * [SMODS.Voucher](https://github.com/Steamodded/smods/wiki/SMODS.Voucher)
  * [SMODS.Challenge](https://github.com/Steamodded/smods/wiki/SMODS.Challenge)
  * [SMODS.DeckSkin](https://github.com/Steamodded/smods/wiki/SMODS.DeckSkin)
  * [SMODS.Keybind](https://github.com/Steamodded/smods/wiki/SMODS.Keybind)
  * [SMODS.Language](https://github.com/Steamodded/smods/wiki/SMODS.Language)
  * [SMODS.ObjectType](https://github.com/Steamodded/smods/wiki/SMODS.ObjectType)
  * [SMODS.PokerHand](https://github.com/Steamodded/smods/wiki/SMODS.PokerHand)
  * [SMODS.Rarity](https://github.com/Steamodded/smods/wiki/SMODS.Rarity)
  * [SMODS.Seal](https://github.com/Steamodded/smods/wiki/SMODS.Seal)
  * [SMODS.Sound](https://github.com/Steamodded/smods/wiki/SMODS.Sound)
  * [SMODS.Stake](https://github.com/Steamodded/smods/wiki/SMODS.Stake)
  * [SMODS.Sticker](https://github.com/Steamodded/smods/wiki/SMODS.Sticker)
  * [SMODS.Suit and SMODS.Rank](https://github.com/Steamodded/smods/wiki/SMODS.Suit-and-SMODS.Rank)
  * [SMODS.Tag](https://github.com/Steamodded/smods/wiki/SMODS.Tag)
***
Guides
  * [Your First Mod](https://github.com/Steamodded/smods/wiki/Your-First-Mod)
  * [Mod Metadata](https://github.com/Steamodded/smods/wiki/Mod-Metadata)
  * [Calculate Functions](https://github.com/Steamodded/smods/wiki/calculate_functions)
  * [Logging](https://github.com/Steamodded/smods/wiki/Logging)
  * [Event Manager](https://github.com/Steamodded/smods/wiki/Guide-%E2%80%90-Event-Manager)
  * [Localization](https://github.com/Steamodded/smods/wiki/Localization)
  * [Mod functions](https://github.com/Steamodded/smods/wiki/Mod-functions)
  * [UI Structure](https://github.com/Steamodded/smods/wiki/UI-Guide)
  * [Utility Functions](https://github.com/Steamodded/smods/wiki/Utility)

 ***

Found an issue, or want to add something? Submit a PR to [the Wiki repo](https://github.com/Steamodded/Wiki).
