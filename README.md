# EawX Lua modifications guide

### Galactic Conquest Framework
Empire at war expanded uses Pox's eawx-galactic-framework which is free release, found here https://github.com/SvenMarcus/eawx-galactic-framework
it is a system that loads modules froma  plugin folder, resolves dependencies automatically and then updates them at a set time decided by plugin targets
it also includes two very useful and very powerful things
the crossplot - a library for communicating across different story plots
and the class script, to allow for object oriented programming in EaW

# Setting up a Plugin
Plugins for the galactic level are folders containing two files, an init.lua and the main plugin file, the folder containing these files must be in the eawx-plugins folder
Setting up plugins for gameobjects is just as easy, except in the eawx-plugins-gameobject-space, or eawx-plugins-gameobject-land directories
Plugin init files are setup like this 
```lua
require("eawx-std/plugintargets")
require("eawx-plugins/my-plugin/MyPlugin")

return {
    --the always() target ensures the plugin is updated every frame
    target = PluginTargets.always(),
    --if the required_planets tag is set to true, the plugin :update() function is repeated for every planet in mod every frame, with the planet class as parameter
    requires_planets = false,
    --Plugins this is dependent on
    dependencies = { "another-plugin" },
    --all plugins have self, and ctx, as parameters, then dependencies are unpacked as params in order
    init = function(self, ctx, the_other_plugin)
    --Return the plugin class from MyPlugin.lua
        return MyPlugin(the_other_plugin)
    end
}
```

Plugin classes are set up like this
```lua
require("eawx-std/class")

---@class MyPlugin
MyPlugin = class()
--classes when called automatically run the :new() function
function MyPlugin:new(args)
    self.args = args
end
--Update is called every frame if plugin target is always(), and takes param as planet class, if requires_planets is true
function MyPlugin:update()

end

return MyPlugin
```

Possible Plugin targets
plugintargets.always() - Ensures plugin is updated every frame
plugintargets.never() - Is never updated, used to react to observable events only, does not have :update() function
plugintargets.interval(time_int) updates the plugin every time_int seconds, time_int must be an integer number
plugintargets.story_flag("FLAG_NAME", playerwrapper) - uses the Check_Story_Flag function to check if flag is active, then allows update

The galactic conquest class is a large part of the framework, handling events such as production started, finished, cancelled, planet owner changed, and hero killed, as well as handling every planet in mod through a custom planet class, which is used in the update function of plugins that require planets to update

it has several attributes that are accessible via ctx.galactic_conquest
galactic_conquest.HumanPlayer - is human player playerwrapper
galactic_conquest.Planets - table of all planets, using planet class, can be arrayed into using string, like this Planets["PLANET_NAME"]
galactic_conquest.Events.EventName - Events that can be listened to via the Observable class, like this ctx.galactic_conquest.GalacticHeroKilled:attach_listener(self.function_to_trigger,self --[[ self is just parameters for listener function, in the case of hero killed, it's hero name]]) 

Functions
get_all_planets_by_faction(playerwrapper) - Returns list of planets owned by faction, with key being string
get_number_of_planets_owned_by_faction(playerwrapper) - returns integer for num of owned planets

Events
PlanetOwnerChanged - Listener Args: planet, new_owner_name, old_owner_name
GalacticProductionStarted - Listener Args: planet, game_object_type_name
GalacticProductionFinished - Listener Args: planet, game_object_type_name
GalacticProductionCanceled - Listener Args: planet, game_object_type_name
GalacticHeroKilled - Listener args: hero name
TacticalBattleStarting - no listener args
TacticalBattleEnding - no listener args


# The Planet class
the galactic_conquest class wraps all planets ina  custom class, accessible via ctx.galactic_conquest.Planets and is used individually as update parameter for plugins that require planets

the planet class has several methods
Planet:get_owner() - returns playerwrapper
Planet:get_game_object() - returns planet GameObjectWrapper
Planet:get_name() - returns name string in all caps
Planet:has_structure(structure_name) - returns true if structure is present, false otherwise

# Initialising the framework
Initialising the framework is generally done in EaWX in the player agnostic story script file, it is run via the EaWXMod Script, which loads the pluginloader, and galactic_conquest class

for example this is a demo file that will initialise the framework with no context, (in EaWX we have plot for context which is Player_Agnostic_Plot.xml, accessible via ctx.plot)
```lua
require("PGDebug")
require("PGStateMachine")
require("PGStoryMode")

require("eawx-std/EawXMod")

function Definitions()
    DebugMessage("%s -- In Definitions", tostring(Script))

    ServiceRate = 0.1

    StoryModeEvents = {Universal_Story_Start = Begin_GC}
end

function Begin_GC(message)
    if message == OnEnter then
        -- The context table allows you to pass variables to
        -- the init() function of your plugins
        local context = {}

        ActiveMod = EawXMod(context)
    elseif message == OnUpdate then
        ActiveMod:update()
    end
end
```

any variables defined in the context table will be accessible from plugins, like this
```lua
context ={
    test= "ThisIsAString"
}

--...and in a plugin you can print this variale permentantly via
StoryUtil.ShowScreenText(ctx.test, -1)
```

#Using gameobject plugins
Gameobject plugins in EaWX are loaded automatically by checking the scripts entry of any unit in GameObjectLibrary.lua, for example here is a dummy unit entry in GameObjectLibrary
```lua
["IMPERIAL_BOARDING_SHUTTLE"] = {
  Scripts = {
    "multilayer",
    "boarding"
   },
  Fighters = {}
},```
this unit will load the 'multilayer' and 'boarding' plugins on load
GameObject plugins are similar to Galactic plugins in almost every way, apart from the fact that they dont load the galactic conquest class, and dont have a requires_planets tag

#Single file plugins

Plugins can be coded in a single file, like the following, and initialised in the table, here is a one file plugin with only an update funcion

```lua
require("eawx-std/plugintargets")

return {
    target = PluginTargets.interval(45)
    init = function(self, ctx)
        return {
            update = function(self)
                -- do something
            end
        }
    end
}```
the plugin will do the update function every 45 seconds, but nothing else, it is useful for loading documentation such as the Government display

# Using events in plugins with the Observable class

Here is an example that shows how to use events from the galactic_conquest class in a plugin, this plugin will return the cost of a unit on completion of production

the init file:
```lua
require("eawx-std/plugintargets")
require("eawx-plugins/productionfinished/productionfinishedlistener")
return {
    target = PluginTargets.never()
    init = function(self, ctx)
        return ProductionFinishedListener(ctx.galactic_conquest)
    end
}
```
the main plugin file:
```lua
require("eawx-std/class")

---@class ProductionFinishedListener
ProductionFinishedListener = class()

---@param galactic_conquest GalacticConquest
function ProductionFinishedListener:new(galactic_conquest)
    self.human_player = galactic_conquest.HumanPlayer
    self.total_amount_of_objects = 0
    --Attatch a listener to the event using Observable methods, you can also use detach_listener with the exact same arguments to detach a listener from events
    galactic_conquest.Events.GalacticProductionFinished:attach_listener(self.on_production_finished, self)
end

---We don't like to lose money, so we return it to the player on build completion
---@param planet Planet
---@param object_type_name string
--Is called from the listener above, when production of a unit finishes
function ProductionFinishedListener:on_production_finished(planet, object_type_name)
    if not planet:get_owner().Is_Human() then
        return
    end

    self.total_amount_of_objects = self.total_amount_of_objects + 1

    local object_type = Find_Object_Type(object_type_name)
    if object_type then
        local cost = object_type.Get_Build_Cost()
        self.human_player.Give_Money(cost)
    end
end
```
# Creating an Observable class that can be listened to
Using the Observable class you can also listen to events from other plugins with listeners, an example plugin setup you can use to listen to events from other pluginsis below
Observable plugin
init:
```lua
require("eawx-std/plugintargets")
require("eawx-plugins/listenertest/listenertest")
return {
    target = PluginTargets.always()
    init = function(self, ctx)
        return ListenerTest()
    end
}

```
```lua

require("eawx-std/Observable")--Required for observable plugins
require("eawx-std/class")
--@class ListenerTest
--Plugins can be, extended by the Observable class, meaning it inherits the methods of the observable class
ListenerTest = class(Observable)

---@param ctx Context
function ListenerTest:new(ctx)
--Another way you can listen to individual events is by declaring an observable variable
    self.PluginUpdated = Observable()
end

function ListenerTest:update()
--Triggers any listeners that are listening to the PluginUpdated variable with optional args
   self.PluginUpdate:notify(optional_args)
   --Triggers ant listeners listening to the ListenerTest class with optional args
   self:notify(optional_args)
end

return ListenerTest

```
and the plugin listening to it:
```lua
require("eawx-std/plugintargets")
require("eawx-plugins/ListeningPlugin/ListeningPlugin")
return {
    target = PluginTargets.never()
        dependencies = {"listenertest"},--has listener test as dependency
    init = function(self, ctx, listenertest)
        return ListeningPlugin(listenertest)
    end
}

```
```lua

require("eawx-std/class")
--@class ListeningPlugin
ListeningPlugin = class()

---@param listenertest ListenerTest
function ListeningPlugin:new(listenertest)
    self.listenertest = listenertest
        --Attach listener to class that triggers on_class_update function with args
    self.listenertest:attach_listener(self.on_class_update,self)
    --Attach listener to pluginUpdate variable
    self.listenertest.PluginUpdate:attach_listener(self.on_plugin_update,self)
end
--Triggered on listenertest class notify
function ListeningPlugin:on_class_update(optional_args)
    --Do something
end
--Triggered on listenertest.PluginUpdate notify
function ListeningPlugin:on_plugin_update(optional_args)
    --Do something
end

return ListeningPlugin

```
# Using the crossplot
The crossplot can be used to great effect to bring variable across story plot, one big example in EawX is baording, where the boarding plugin publishes a boarded unit to the crossplot, where gamescoring.lua (Data/Scripts/Miscallaneous) listens to it, and pushes it through the boarding handler, which gamescoring sends to teh boardingListener plugin after tactical battle end, and events which publish data from gamescoring to the galactic_conquest class to trigger event listeners

an example of how to communicate from stroy plot to story plot is here:
Plot 1:
```lua
require("eawx-crossplot/crossplot")

function Definitions()
    DebugMessage("%s -- In Definitions", tostring(Script))

    ServiceRate = 0.1

    StoryModeEvents = {Universal_Story_Start = Begin_GC}
end

function Begin_GC(message)
    if message == OnEnter then
        crossplot:galactic()
        crossplot:subscribe("sub_test", Receive)
    elseif message == OnUpdate then
        crossplot:update()
    end
end

function Receive(text_entry)
    Game_Message(text_entry)
end
```
Plot 2:
```lua
require("eawx-crossplot/crossplot")

function Definitions()
    DebugMessage("%s -- In Definitions", tostring(Script))

    ServiceRate = 0.1

    StoryModeEvents = {Universal_Story_Start = Begin_GC}
end

function Begin_GC(message)
    if message == OnEnter then
        crossplot:galactic()
        Sleep(1) -- sleep to make sure the other plot had time to subscribe
        crossplot:publish("sub_test", "TEXT_FACTION_EMPIRE")
    elseif message == OnUpdate then
        crossplot:update()
    end
end
```

Plot one subscribes to the crossplot instance "sub_test" triggering the Recieve function with the args given by Plot 2, in the publish line "TEXT_FACTION_EMPIRE", triggering the recieve function in Plot 1 printing Empire, to the droid log, 

NOTE: The crossplot must be updated every frame to ensure it publishes and recives data correctly
NOTE: you must subscribe to a crossplot instance before it is published, or the subscription will never recieve the data


# Using EaWX utilities
Empire at war expanded has a variety of different utilities all stored in the eawx-util folder, under Data/Scripts/Library/eawx-util
they include ChangeOwnerUtilities, CachedEquationEvaluator, Comparators, GalacticUtil, PlanetTable, PopulatePlanetUtilities, and StoryUtil
all of which contain a variety of extremely useful utility functions that one can use to great extent if they know how to

### CachedEquationEvaluator.lua
functions from this file can be accessed from any lua script using ``require("eawx-util/CachedEquationEvaluator)``

this utility is used in FreeStores as an easy way to evaluate perceptions and store equation results for future use, for example to evaluate a perception you could do
```lua
CachedEquationEvaluator:evaluate(equation_name, faction, object)
```
and it would return the result to the entry, whilst also storing it in a cache to be accessed at later time if necessary, you can reset it using ``CachedEquationEvaluator:reset()``

### ChangeOwnerUtilities.lua
functions from this file can be accessed from any lua script using ``require("eawx-util/ChangeOwnerUtilities)``
This utility file contains functions that are used to easily change planet owner, fufilling certain conditions as well

``ChangePlanetOwnerAndRetreat(planets, newOwner)``
can take a table of, or single GameObjectWrapper for a planet as the planets parameter, and retreats all units from the planet to a planet friendly to the old owner, and then changes planet owner to the newOwner parameter

ChangePlanetOwnerAndReplace
This function is effectively the same as the above, but changes despawns all units on the planet, and replaces them with the exact same units for the newOwner parameter
it can be run using ``ChangePlanetOwnerAndRetreat(planets, newOwner)``

``get_friendly_units_on_planet(player,planet)``
returns a table containing all friendly units on a planet

``get_relevant_object(unit)``
returns the object necessary to spawn the unit on the galactic map, if unit is in company it will return company, otherwise returns GameObjectWrapper

``spawn_units_on_friendly_location(unitTypes, unitOwners)``
param unitTypes table, unitOwners table
spawns units on planet friendly to the index of the unit in the unitOwners Table, if owner is human, shows text in top left saying where units retreated to

``is_valid_category(unit)``
param unit GameObjectWrapper
Returns true if unit has a category that allows it to move to a different planet

``Destroy_Planet(planet)``
param planet Planet GameObjectWrapper
destroys planet using Dummy Planet Killer

### GalacticUtil.lua
functions from this file can be accessed from any lua script using ``require("eawx-util/GalacticUtil)``
this file contains a variety of functions that are useful for doing things on the galactic level, such as spawning things, finding friendly planet, and finding planets
the most notable usage of this is spawning units on galactic map after boarding

``GalacticUtil.spawn(data)``
param data, table
table must have at least three variables in it, location, objects, and owner, or returns error
optional variables, fallback, can be true or false, determines if is spawning at predetermined location or friendly planet
spawns objects at location or friendly planet for owner and returns unit list

``GalacticUtil.in_galactic_mode()``
returns true if in galactic

``GalacticUtil.find_friendly_planet(player)``
param player PlayerWrapper or string
returns random friendly planet to player or if none available returns nil

``GalacticUtil.get_all_planets_without_dummies()``
returns table of all planets without any dummies

### PopulatePlanetUtilities.lua
functions from this file can be accessed from any lua script using ``require("eawx-util/GalacticUtil)``
this file contains one primary function most notably used in revolts, for changing a planet owner and populating with random units
``ChangePlanetOwnerAndPopulate(planet, newOwner, combat_power, random_population)``
param planet Gameobjectwrapper
param newOwner PlayerWrapper
param combat_power integer
param random_population, bool

function changes planet owner to newOwner, retreating any existing units off planet, and spawns random fleet depending on random_population bool to meet inserted combat power value, potential unit table sare decided in UnitSpawnerTables.lua

### StoryUtil.lua
functions from this file can be accessed from any lua script using ``require("eawx-util/StoryUtil)``
this file is the most extensive file of utility functions and contains a variety of very useful and very powerful utilities

``StoryUtil.ShowScreenText(textId, time, var, color, teletype)``
param textID, string or textfile dat identifier
param time, int
param var, variable to fill %s in string
param color, RGB value
param teletype bool
The ShowScreenText function shows text in the top left corner of the screen, which since the petro patch can now take any string as TextId, can be any RGB color, -1 is for text to display infinitely
``StoryUtil.RemoveScreenText(textId)``
param textID, string or textfile dat identifier
the RemoveScreenText function removes any shown text in the top left if the given string or ID is present

``StoryUtil.LockUnit(unitType)``
param unitType GameObjectTypeWrapper

locks the given unit for all factions, as opposed to PlayerWrapper.Lock_Tech(unitType) which only locks for given player

``StoryUtil.LockList(faction,lock_list)``
param faction, string
param lock_list table of unit names
Locks all units in list for only given faction

``StoryUtil.UnlockList(faction, unlock_list)``
Same as above, but unlocks instead of locks

``StoryUtil.SetPlanetRestricted(planet, status)``
param planet GameObjectWrapper
param status string

locks access to given planet (not sure what status does yet need to find out)

``StoryUtil.EnableBuildableUnit(unitType)``
param unitType GameObjectTypeWrapper
Enables a buildable unit, only works for units that already appear darkened out on the build bar

``StoryUtil.DisableBuildableUnit(unitType)``
param unitType GameObjectTypeWrapper
Makes a unit non buildable by darkening it on the build bar, so it still appears, but isn't buildable

``StoryUtil.SetTechLevel(player, level)``
param player PlayerWrapper
param level integer
Sets tech level for given player to level - has built in support for rebel, so will increase tech level 1 more if rebel to keep in same scale with all other factions

``StoryUtil.GetGenericEvent()``
Grabs generic XML event, can be used for triggering custom XML events completely via lua

``StoryUtil.TriggerGenericEvent()``
Triggers generic XML event, useful if params have been setwith lua beforehand

``StoryUtil.ResetGenericEvent()``
Resets reward and event parameters of generic XML event

``StoryUtil.ValidGlobalValue(val)``
param val, variable
returns true if variable exists and is not empty string

``StoryUtil.FindFriendlyPlanet(player)``
param player playerwrapper or string
returns random planet friendly to player or nil

``StoryUtil.CheckFriendlyPlanet(planet, player)``
param planet GameObjectWrapper
param player PlayerWrapper
returns true if planet is owner by player and has no enemy forces on it

``StoryUtil.FindEnemyPlanet(player)``
param player PlayerWrapper or string
returns random enemy planet to player

``StoryUtil.GetAllPlanetsWithoutDummies()``
returns table of all planets with no dummies present

``StoryUtil.SpawnAtSafePlanet(planet_name, player, spawn_location_table, spawn_list)``
param planet_name, string
param player, PlayerWrapper or string
param spawn_location_table, table
param spawn_list, table of unit names

Spawns unit on given planet if it exists in spawn_location_table, if enemy is present or player is not owner, spawns on random planet, if planet_name doesn't return true in spawn_location_table, returns nil, mainly used in story scripts

``StoryUtil.GetSafePlanetTable()``
Returns table of all planets in GC

``StoryUtil.ReplaceAtLocation(original, upgrade)``
param original string
param upgrade string
replaces original object with upgrade object for orignial owner at original location

``StoryUtil.ChangeAIPlayer(player, ai_type)``
param player, string
param ai_type, string
changes AI for player to type given in ai_type

``StoryUtil.Multimedia(textId, time, speech_name, movie_name, int, faction)``
param textId, textID or string
param time, int
param speech_name, audio event name
param movie_name, movie name in movies.xml
param int, integer
param faction, faction - optional

shows text for given time, plays speech and movie, for faction, or human if no given faction

# State Machine
After TR 3.1 and FotR 1.1 EaWX has moved away from standard story scripting and instead story scripting is now handled by a State Machine.
In order to set up the system in a GC you'll need to create a new player agnostic GC file for the GC, for this you can just Copy GCPlayerAgnostic and rename it, 
you then edit this line to be whatever id you want
```Lua
local gc_id = "NEWGCNAME"
```
this is the tag that will be used for the name of the event handler, and state files at these locations

State Machine: Scripts\Library\eawx-mod-id\eawx-plugins\statemachine-main\NEWGCNAME.lua

the most basic form of the state machine defines eras, and a way to transition between them, it uses credits to oload into different eras, meaning the max amount of eras is 11
due to the nature of the state machine it can be in only one state at a time, but a GC can have multiple statemachines initialised at once, here is a dummy state thand gives all corporate sector planets to empire, and transitions all factions to tech 3
```Lua
return function(dsl)
    local policy = dsl.policy
    local effect = dsl.effect
    local owned_by = dsl.conditions.owned_by

    local initialize = EawXState.with_empty_policy() -- Returns empty state file, does nothing on call, update, and exit
    local setup = EawXState(require("eawx-states/statefilename"))--Location of setup state
    
    dsl.transition(initialize )
        :to(setup )
        :when(policy:timed(20))
        :with_effects(
            effect:transfer_planets(unpack(FindPlanet.Get_All_Planets()))
            :to_owner("Empire")
            :if_(owned_by("Corporate_Sector")),
            effect:set_tech_level(3)
            :for_factions({"Rebel", "Empire", "Pentastar", "Hutts", "Pirates", "Teradoc"})
        ):end_()
      
    return initialize
end```

There are several different types of transition policy
Timed: Example : :when(policy:timed(20))
Hero Death: :when(policy:hero_dies("Boba_Fett_Team"))
Object Construction :when(policy:object_constructed("Thrawn_regicide_dummy"))

There are also oseveral different types of effects these can have
Transfer Planet OwnerShip:
```lua
            effect:transfer_planets(planets) -- Can have multiple, must be concatenated strings
            :to_owner("Name")
            :if_(owned_by("Name")), --Not needed, but is helpful
```
Setting Tech Level:
```lua
    effect:set_tech_level(integer)
    :for_factions(list)

State File Layout:
```lua
return {
    --Triggered on state entered
    on_enter = function(self, state_context)

    end,
    --Run once per frame
    on_update = function(self, state_context)

    end,
    --run on state exit
    on_exit = function(self, state_context)

    end

}```

#Event Handler
Event handler is a state agnostic class that creates and runs specific events, they can be fired by states, or anything that is capable of using the crossplot, you can have the publishing file
publish information to an event, and trigger it
Each GC has its own event handler

Event Handler: Scripts\Library\eawx-mod-id\eawx-plugins\event-handler\NEWGCNAME.lua
A basic event manager setup looks like this

```lua
require("eawx-std/class")
--These are only needed if the event is one of the following
require("eawx-events/GenericResearch")
require("eawx-events/GenericSwap")
require("eawx-events/GenericConquer")


EventManager = class()
--takes galactic conquest class, human player, and table of planets as parameters
function EventManager:new(galactic_conquest, human_player, planets)
    --Store parameters as variables in class
    self.galactic_conquest = galactic_conquest
    self.human_player = human_player
    self.planets = planets

end
--Triggered each frame
function EventManager:update()
    
end
--Returns eventmanager class
return EventManager
```

Research Events can be set up like this
```lua
    self.researchtest = GenericResearch(
        self.galactic_conquest, -- used for accessing events
        "RESEACH_NAME", -- used for crossplot and story tags
        object_name,
        player,
        {list of items to unlock},
        {list of items to lock},
        {list of items to spawn},
        "Planet_To_Spawn_Items_At",
        "CONNECTED_EVENT_TO_TRIGGER"
    )
```
Swap events are set up like this
```lua
    self.swaptestevent = GenericSwap(
        "EVENT_TAG",
        "FACTION_NAME",
        {Objects to be replaced},
        {objects to spawn}
    )
```

Conquer events are set up like this 
```lua
    self.conquertest = GenericConquer(
        self.galactic_conquest,
        "EVENT_NAME",
        "PLANET_NAME", -- must be capitalised
        "CONQUERING_FACTION"
        {Reward items},
        {unlock list},
        {lock list},
        "CONNECTED_EVENT_TO_TRIGGER"
    )
```

video and multimedia events can be made for events in the EventLogRepository.xml file under XML/Conquests/Events with the EVENTNAME_STARTED, EVENTNAME_COMPLETED
they will need the AI_NOTIFICATION event triggers
multimedia events will automatically use the TEXT_SPEECH_EVENTNAME_STARTED, or _FINISHED or _FAILED for conquering string names
