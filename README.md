# CustomItemLib

A modding library for Haste: Broken Worlds that allows for easier custom item creation directly in C# to reduce clutter and enable features such as type checking and IntelliSense.


## Features

- Factory methods for creating ItemInstance and PlayerStats objects with optional parameters to reduce clutter
- A subscribable event that allows for better compatibility between custom item mods


## Setup

This library can be used via the steam workshop as a dependency or compiled from source directly in your mod (see LICENSE file). If you choose to include it directly in your mod, remember to change the namespace to avoid namespace collisions.
By default, this library uses the `CustomItemLib` namespace.
Note: This library does not provide any method of making or loading item meshes. To add cutom models to your items, see the 3d models section in https://github.com/Haste-Team/HastePlugins/blob/main/README.md.


## Usage

The primary feature of CustomItemLib is the `ItemFactory.AddItemToDatabase` method, which acts very similarly to the native Haste method that imports a JSON item file. This method must be provided an `itemName` argument, which will be the name of the item you either edit (if the item already exists) or create (if it doesn't). Any parameters not included will default to the original item's value or to a default value when creating a new item. A full list of parameters will be included later. Here is an example of a simple edit to the cooldown of the Rocket Boots item.

```csharp
ItemFactory.AddItemToDatabase(
    itemName: "Active_Boost", // Internal itemName of Rocket Boots
    cooldown: 3f              // 3 second cooldown
);
```

<br>

#### Custom Items
The same can be done to create a new item. In a few places in Haste's codebase, enums are used like the `ItemTriggerType` below rather than the ints used in the JSON files. If you are unfamiliar with the enums, I would recommend keeping them open on a decompiler like dnSpy or ILSpy. The example below creates a new item which will trigger continuously every 3 seconds. We will get to effects later, for now this item does nothing when it triggers.

```csharp
ItemFactory.AddItemToDatabase(
    itemName: "ExampleItem",
    triggerType: ItemTriggerType.Continious, // Item will trigger continuously, as often as it is allowed to
    cooldown: 3f                             // However, its cooldown will only allow it to trigger every 3 seconds
);
```

<br>

#### Tooltips and Item Descriptions
Items have four different string fields that are used to create the tooltip that appears when you hover over one. In the first example, because we were modifying an item that already exists, we don't have to include these parameters, and the library will fall back to the original item's values. In the second example, because we created a new item without including them, the library uses a default warning string. These use Unity's LocalizedString class instead of normal strings, however Haste has helpfully provided a UnlocalizedString class which we will use below. 

```csharp
ItemFactory.AddItemToDatabase(
    itemName: "ExampleItem",
    title: new UnlocalizedString("Example Item"),                                 // The title displayed at the top of the item tooltip
    triggerDescription: new UnlocalizedString("This item's trigger description"), // Item trigger description
    description: new UnlocalizedString("This item's description"),                // Item Description (I don't know which description is which at the moment)
    flavorText: new UnlocalizedString("Some flavour text about this item")        // The flavour text at the bottom of the item tooltop
);
```

Note 1: To use localization features, see the localization section in https://github.com/Haste-Team/HastePlugins/blob/main/README.md and use LocalizedStrings in place of the unlocalized versions below.
Note 2: Haste supports Unity Rich Text. You can use this to create coloured (and otherwise modified) strings.

<br>

#### Simple Stat Effects
So far, the items we've created haven't actually done anything, so let's create a simple item that increases our max energy when we have it. Here we see the other factory method provided by this library, `ItemFactory.CreatePlayerStats`. Haste's native `PlayerStats` class is intended to be serialized into, and therefore doesn't assign default values to its fields. This would normally leave us the responsibility of initializing each field with a new PlayerStat object, even if we aren't assigning any meaningful values. `ItemFactory.CreatePlayerStats` takes care of this boilerplate by assigning a default PlayerStat object to any fields you don't specify. In the example below, we create a PlayerStats that adds 50 max energy to the player. You can also set a multiplier instead of / in addition to the baseValue. Any fields in a PlayerStat that aren't specified will use a default value. See the PlayerStats class definition or an exported item JSON for a list of modifiable stats.

```csharp
ItemFactory.AddItemToDatabase(
    itemName: "ExampleItem",
    stats: ItemFactory.CreatePlayerStats(maxEnergy: new PlayerStat { baseValue = 50f }) // Add 50 max energy to the player when they have this item
);
```

Note 1: Keep in mind that `ItemFactory.CreatePlayerStats` returns a `PlayerStats` object, while `ItemFactory.AddItemToDatabase` returns nothing.
Note 2: There are two different similarly named native classes here. `PlayerStats` is a native class containing multiple `PlayerStat` fields.

<br>

#### Triggered Effects
While the stats field is great for simple effects, most items in Haste have effects that are triggered by something. Below we will create an object that gives the player 1 energy whenever they pick up a spark using the `triggerType` parameter we saw earlier and a new parameter called `effects`. The effects parameter contains a list of `ItemEffect` instances, or more accurately, a list of instances of classes derived from ItemEffect. The game provides a few of these, which you can find in your decompiler by selecting "Analyze" on the ItemEffect class and checking the "Subtypes" section. Here we will use the `AddVariable_Effect` class to add energy to the player.

```csharp
ItemFactory.AddItemToDatabase(
    itemName: "ExampleItem",
    triggerType: ItemTriggerType.GetResource,                                     // ItemTriggerType.GetResource triggers when the player picks up a spark
    effects: new List<ItemEffect>
    {
        new AddVariable_Effect { amount = 1f, variableType = VariableType.Energy} // AddVariable_Effect adds an effect to the player
    }
);
```

Note 1: `effects` can contain multiple ItemEffect objects. When an item triggers, it triggers ALL of its ItemEffects.
Note 2: You can create your own `ItemEffect` classes to do just about anything by deriving the native ItemEffect class. (Guide coming sometime in the future)

<br>

#### Trigger Conditions
Many items in Haste also have a condition that must be met for their effect to trigger. These are defined with the `triggerConditions`, which contains a list of instances of classes derived from the native class `ItemTrigger`, very similarly to the effects from before. Below we'll add a simple `ChanceCheck_IT` trigger condition that only lets the item trigger 50% of the time.

```csharp
ItemFactory.AddItemToDatabase(
    itemName: "ExampleItem",
    triggerType: ItemTriggerType.GetResource,                                     // ItemTriggerType.GetResource triggers when the player picks up a spark
    triggerConditions: new List<ItemTrigger>
    {
        new ChanceCheck_IT { probability = 0.5f }                                 // ChanceCheck_IT only allows the item to trigger sometimes at random
    },
    effects: new List<ItemEffect>
    {
        new AddVariable_Effect { amount = 1f, variableType = VariableType.Energy} // AddVariable_Effect adds an effect to the player
    }
);
```

Note 1: `triggerConditions` can contain multiple ItemTrigger objects. Items will only trigger if ALL of their ItemTriggers return true.
Note 2: You can create your own `ItemTrigger` classes in the same way as before with ItemEffect. (Guide coming sometime in the future)

<br>

#### ItemsLoaded Event / Full Example
Unfortunately, you can't just put these method calls in your LandfallPlugin's static constructor, because plugins are loaded before the ItemDatabase is initialized. Normally, you would need to use a MonoMod hook or something similar to load the items at the right time (and you still can if you prefer). However, CustomItemLib provides an event that you can subscribe to to load your items when the game would normally load items, to ensure proper loading and provide some extra compatibility between custom item mods. Below is a full example of the recommended structure when using CustomItemLib.

```csharp
using Landfall.Modding;
using Landfall.Haste;
using CustomItemLib;

namespace ExampleItems
{
    [LandfallPlugin]
    public class ExampleItems
    {
        static ExampleItems()
        {
            ItemFactory.ItemsLoaded += LoadCustomItems; // Subscribe your method to ItemFactory.ItemsLoaded so it runs when the ItemDatabase loads items
            // Your plugin's other initialization code
        }
        private static void LoadCustomItems() // Put all of your AddItemToDatabase calls in a method that you can then subscribe to ItemsLoaded (seen above)
        {
            ItemFactory.AddItemToDatabase( // A custom item
                itemName: "ExampleItem1",
                triggerType: ItemTriggerType.Continious,
                cooldown: 3f
            );

            ItemFactory.AddItemToDatabase( // Another custom item
                itemName: "ExampleItem2",
                triggerType: ItemTriggerType.GetResource,
                effects: new List<ItemEffect>
                {
                    new AddVariable_Effect { amount = 1f, variableType = VariableType.Energy}
                }
            );
        }
    }
}
```

<br>

#### Exhaustive Example
Putting everything we've learned together, let's recreate the Brittle Breastplate item from Haste. Keep in mind, this is an extreme case for a complicated item, default item modifications and simpler custom items will rarely be as long as this. Because Brittle Breastplate already has localization for it's tooltip strings, we'll try using LocalizedStrings this time instead of UnlocalizedStrings.

```csharp
ItemFactory.AddItemToDatabase(
    itemName: "BrittleBreastplate", // The actual itemName is MaxHealthButDieOnOkOrBadLanding, but we're making a custom copy
    rarity: Rarity.Epic,
    itemTags: new List<ItemTag>     // I think these are just for flavour, I don't know if they even have an effect ingame
    {
        ItemTag.Object,
        ItemTag.Fantasy,
        ItemTag.Captain,
        ItemTag.Heir,
        ItemTag.Agency
    },
    title: new LocalizedString("Items", "MaxHealthButDieOnOkOrBadLanding_title"), // Reference the actual Brittle Breastplate's title
    triggerDescription: new LocalizedString("Items", "MaxHealthButDieOnOkOrBadLanding_triggerDesc"), // and triggerDescription
    description: new LocalizedString("Items", "MaxHealthButDieOnOkOrBadLanding_desc"),               // and description
    flavorText: new LocalizedString("Items", "MaxHealthButDieOnOkOrBadLanding_flavor"),              // and flavorText
    cooldown: 1f,
    triggerType: ItemTriggerType.Landing, // Brittle Breastplate's damage is triggered by landing
    triggerConditions: new List<ItemTrigger>
    {
        new LandingType_IT
        {
            landingType = LandingType.Ok,                           // Lets the item trigger if the landing was Ok
            compareType = LandingType_IT.CompareType.WorseOrSameAs, // or worse
        }
    },
    effects: new List<ItemEffect>
    {
        new AddVariable_Effect { amount = 50f, variableType = VariableType.Damage } // Deal 50 damage to the player when triggered
    },
    stats: ItemFactory.CreatePlayerStats(maxHealth: new PlayerStat { multiplier = 2.5f }) // Give the player +150% max health when they have this
);
```

Note: `ItemFactory.AddItemToDatabase` includes a parameter `autoUnlocked` which determines whether to automatically unlock the item for the player. This is set to true by default, but you can set it to false if your mod involves player progression and you want the player to have to unlock the item the normal way.

<br>

#### Parameters and Default Values
Below is a full list of parameters available with `ItemFactory.AddItemToDatabase` and their default values.

```csharp
string itemName;                                                    // itemName is required
bool autoUnlocked = true;                                           // Whether to unlock the item automatically
bool minorItem = false;                                             // Whether the item is considered a "minorItem", which can't appear in shops
Rarity rarity = Rarity.Common;                                      // The item's rarity
List<ItemTag> itemTags = new List<ItemTag>();                       // The item's tags (seemingly cosmetic)
LocalizedString title = new UnlocalizedString("Missing Title");     // The item's tooltip title
bool usesTriggerDescription = false;                                // Not sure what this does
LocalizedString triggerDescription = new UnlocalizedString("Item is missing trigger description") // One of the descriptions (I'm not sure the difference)
bool usesEffectDescription = false;                                 // Not sure what this does either
LocalizedString description = new UnlocalizedString("Item is missing description"); // One of the descriptions (I'm not sure the difference)
LocalizedString flavorText = new UnlocalizedString("Dalil didn't explain what this item is..."); // The item's flavour text at the bottom of the tooltip
float iconScaleModifier = 1f;                                       // Not sure, maybe for scaling meshes that are the wrong size?
bool statsAfterTrigger = false;                                     // No idea what this does
float cooldown = 0f;                                                // The minimum time between this item's triggers
ItemTriggerType triggerType = ItemTriggerType.None;                 // The type of event that causes this item to attempt to trigger
List<ItemTrigger> triggerConditions = new List<ItemTrigger> { };    // The item's trigger conditions
List<ItemEffect> effects = new List<ItemEffect> { };                // The item's effects when triggered
PlayerStats stats = ItemFactory.CreatePlayerStats();                   // The PlayerStats object that will be added to the player when they get the item
UnityEvent effectEvent = new UnityEvent();                          // Used as an alternative to effects and trigger conditions. Not fully understood yet
UnityEvent keyDownEvent = new UnityEvent();                         // Used as an alternative to effects and trigger conditions. Not fully understood yet
UnityEvent keyUpEvent = new UnityEvent();                           // Used as an alternative to effects and trigger conditions. Not fully understood yet
```

Note: While some parameters map to `ItemInstance` fields that aren't fully understood by the community yet, they should still work as intended if you figure them out, so feel free to test them if you have any ideas.


## License

MIT License (see LICENSE file)
