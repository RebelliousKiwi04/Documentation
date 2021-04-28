# EaWX State Machine Guide

### Setting Up a new State machine

When setting up a new state machine instance for a GC there are several things that need to be done, the first of these is making a player agnostic story script for it, to start just copy the code below, changing the following line to your new GC id
```Lua
local gc_id = "NEWGCNAME"
```

```Lua
require("PGDebug")
require("PGStateMachine")
require("PGStoryMode")

require("eawx-util/StoryUtil")
require("eawx-std/EawXMod")

function Definitions()
    DebugMessage("%s -- In Definitions", tostring(Script))

    ServiceRate = 0.1

    StoryModeEvents = {Zoom_Zoom = Begin_GC}
end

function Begin_GC(message)
    if message == OnEnter then
        CONSTANTS = ModContentLoader.get("GameConstants")
        GameObjectLibrary = ModContentLoader.get("GameObjectLibrary")
        local plot = StoryUtil.GetPlayerAgnosticPlot()

        local plugin_list = ModContentLoader.get("InstalledPlugins")
        local context = {
            plot = plot
        }
        local gc_id = "PROGRESSIVE"

        ActiveMod = EawXMod(context, plugin_list, gc_id)
    elseif message == OnUpdate then
        ActiveMod:update()
    end
end
```

Save this File as something like GCPlayerAgnosticNewGCName.lua in Data/Scripts/Story and tie it to your new GC via XML

Now create a new file in this directory, with NEWGCNAME being the same name you put in the Player Agnostic file

State Machine: ``Scripts\Library\eawx-mod-id\eawx-plugins\statemachine-main\NEWGCNAME.lua``

This code below is the PROGRESSIVE.lua file for FotR, it decides transitions for eras in FotR progressives, and the effects for those transitions

```Lua
require("eawx-statemachine/EawXState")

---@param dsl dsl
return function(dsl)
    --Declares Policy, effect, and conditions from dsl
    local policy = dsl.policy
    local effect = dsl.effect
    local owned_by = dsl.conditions.owned_by
    --EaWXState.with_empty_policy() returns empty state, meaning it does nothing
    local initialize = EawXState.with_empty_policy()
    --Setup State and all following states
    local setup = EawXState(require("eawx-states/fotr-setup-state"))
    local era_two = EawXState(require("eawx-states/fotr-progressive-era-two"))
    local era_three = EawXState(require("eawx-states/fotr-progressive-era-three"))
    local era_four = EawXState(require("eawx-states/fotr-progressive-era-four"))


    --Sets up transition from initialize state, which is empty as decided above, to state setup, when 2 in game ticks have passed
    dsl.transition(initialize)
        :to(setup)
        :when(policy:timed(2))
        :end_()
    --Transitions from setup state, to era_two state if tech level hits 2
    dsl.transition(setup)
        :to(era_two)
        :when(policy:tech_level(2))
        :end_()
    --Transitions from setup to era 3 if tech level hits 3
    dsl.transition(setup)
        :to(era_three)
        :when(policy:tech_level(3))
        :end_()
    --Transitions from setup to era four if tech level hits 4
    dsl.transition(setup)
        :to(era_four)
        :when(policy:tech_level(4))
        :end_()


    --Transitions from era two, to era 3, after 1200 in game tics, and transfers all planets to empire if they are owned by Corporate_Sector, and sets tech level lto 3 for the given faction list 
    dsl.transition(era_two)
        :to(era_three)
        :when(policy:timed(1200))
        :with_effects(
            effect:transfer_planets(unpack(FindPlanet.Get_All_Planets()))
            :to_owner("Empire")
            :if_(owned_by("Corporate_Sector")),
            effect:set_tech_level(3)
            :for_factions({"Rebel", "Empire", "Pentastar", "Hutts", "Pirates", "Teradoc"})
        ):end_()
    
    --Transition from era 3 to era 4 after 1200 in game ticks, transferring all planets to empire if owned by the CSA, and setting tech level to 4 for given faction list
    dsl.transition(era_three)
        :to(era_four)
        :when(policy:timed(1200))
        :with_effects(
            effect:transfer_planets(unpack(FindPlanet.Get_All_Planets()))
            :to_owner("Empire")
            :if_(owned_by("Corporate_Sector")),
            effect:set_tech_level(4)
            :for_factions({"Rebel", "Empire", "Pentastar", "Hutts", "Pirates", "Teradoc"})
        ):end_()
        --Returns initialize state
    return initialize
end
```

There are several different types of transition policy
The below transitions state when 20 in game ticks hav epassed
Timed: Example : ``:when(policy:timed(20))``
Transitions to next state when boba fett dies
Hero Death: ``:when(policy:hero_dies("Boba_Fett_Team"))``
Transitions to next state when thrawn regicide dummy is built
Object Construction ``:when(policy:object_constructed("Thrawn_regicide_dummy"))``

There are also oseveral different types of effects these can have
Transfer Planet OwnerShip:
```lua
            effect:transfer_planets(planets) -- Can have multiple, must be concatenated strings, like this "PLANET1,"PLANETTWO"
            :to_owner("Name")
            :if_(owned_by("Name")), --Not needed, but is helpful
```
Setting Tech Level:
```lua
    effect:set_tech_level(integer)--Sets tech level to given integer for given faction list
    :for_factions(list) --List must be table, like this {"Rebel", "Empire", "Pentastar", "Hutts", "Pirates", "Teradoc"}

State File Layout:
The below is an empty state, which can easily be set in your setup script with this 
``    local initialize = EawXState.with_empty_policy()
``

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

Below is the fotr progressive era 2 state, with comments for examples

```Lua

require("eawx-util/StoryUtil")
require("PGStoryMode")
require("PGSpawnUnits")

return {
    on_enter = function(self, state_context)
        --Set current era global value !!!!!REQUIRED OR THINGS WILL BREAK!!! namely fighters
        GlobalValue.Set("CURRENT_ERA", 2)
        --Sets venator events bool to false - Used for venator research
        self.VenatorEvents = false
        --Returns table of all active planets
        self.Active_Planets = StoryUtil.GetSafePlanetTable()
        --Grabs time state was entered
        self.entry_time = GetCurrentTime()

        --If you have entered era 2 within 5 ticks of game starting, then give message from palpatine if player is empire, dooku if player is CIS
        if self.entry_time <= 5 then
            if Find_Player("local") == Find_Player("Empire") then

                StoryUtil.Multimedia("TEXT_STORY_INTRO_PROGRESSIVE_REPUBLIC_PALPATINE_ERA_2", 15, nil, "PalpatineFotR_Loop", 0)
            elseif Find_Player("local") == Find_Player("Rebel") then

                StoryUtil.Multimedia("TEXT_STORY_INTRO_PROGRESSIVE_CIS_DOOKU_ERA_2", 15, nil, "Dooku_Loop", 0)
            end

            --Grab era 2 spawns table from file, and spawn them at safe planet from active planets table for faction
            self.Starting_Spawns = require("eawx-mod-fotr/spawn-sets/EraTwoStartSet")
            for faction, herolist in pairs(self.Starting_Spawns) do
                for planet, spawnlist in pairs(herolist) do
                    StoryUtil.SpawnAtSafePlanet(planet, Find_Player(faction), self.Active_Planets, spawnlist)  
                end
            end

            --Grab static fleet spawn sets
			self.Static_Spawns = require("eawx-mod-fotr/spawn-sets/EraTwoStaticFleetSet")
            --Iterate through all static fleet spawns, spawning them only for AI players at enemy planets
            for faction, unitlist in pairs(self.Static_Spawns) do
                if (Find_Player(faction) == Find_Player("EMPIRE")) and (Find_Player("REBEL").Is_Human()) then
                    for planet, spawnlist in pairs(unitlist) do
                        StoryUtil.SpawnAtSafePlanet(planet, Find_Player(faction), self.Active_Planets, spawnlist)  
                    end
                elseif (Find_Player(faction) ~= Find_Player("EMPIRE")) and (Find_Player("EMPIRE").Is_Human()) then
                    for planet, spawnlist in pairs(unitlist) do
                        StoryUtil.SpawnAtSafePlanet(planet, Find_Player(faction), self.Active_Planets, spawnlist)  
                    end
                end
            end
        end
    end,
    --Triggered each frame
    on_update = function(self, state_context)
        --If player has been in state for more than 400 ticks and venator research hasn't happened, then trigger venator research
        local current = GetCurrentTime() - self.entry_time
        if (current >= 400) and (self.VenatorEvents == false) then
            self.VenatorEvents = true
            crossplot:publish("VENATOR_RESEARCH", "empty")
        end
    end,
    --Triggered on state exit
    on_exit = function(self, state_context)
    end
}
```

### Event Handler
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

Research Events can be set up like this from inside the event manager
```lua
        self.PhaseIIResearch = GenericResearch(self.galactic_conquest,
        "PHASE_TWO_RESEARCH", -- Name Of Research, used for crossplot and story tags
        "Dummy_Research_Clone_Trooper_II", -- Research Object
         "Empire", --Player name
        {"Clonetrooper_Phase_Two_Team", "BARC_Squad", "ARC_Phase_Two_Team"}, -- List of items to unlock
        {"Clonetrooper_Phase_One_Team", "Speeder_Bike_Squad", "ARC_Phase_One_Team"},--List of items to lock
         nil,--List of items to spawn, for this event there are none so nil
          nil, --Planet to spawn items at, in this case nothing to spawn so nil
        {"CLONE_UPGRADES"}) -- Connected event to trigger, is crossplot event
```
Swap events are set up like this
```lua
    self.CloneSwap = GenericSwap("CLONE_UPGRADES",--Event name
     "EMPIRE",--Faction name
        {"Fordo", "Cody", "Rex"},--Objects to despawn
        {"Fordo2_Team", "Cody2_Team", "Rex2_Team"})--Objects to spawn
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
        "CONNECTED_EVENT_TO_TRIGGER" -- Is crossplot event
    )
```
video and multimedia events can be made for events in the EventLogRepository.xml file under XML/Conquests/Events with the EVENT_NAME_STARTED, and EVENT_NAME_COMPLETED tags
they will need the AI_NOTIFICATION event triggers, though note these are not compulsary
multimedia events will automatically use the TEXT_SPEECH_EVENTNAME_STARTED, or _FINISHED or _FAILED for conquering string names

Note that the EventLogRepository.xml file will need to be linked to the GC in the GC XML plot files

# Using EaWX utilities
EaWX has a variety of different utility files containing functions that can be extremely useful for a variety of different things, the most important of these are below

### ChangeOwnerUtilities.lua
functions from this file can be accessed from any lua script using ``require("eawx-util/ChangeOwnerUtilities)``
This utility file contains functions that are used to easily change planet owner, fufilling certain conditions as well

``ChangePlanetOwnerAndRetreat(planets, newOwner)``
can take a table of, or single GameObjectWrapper for a planet as the planets parameter, and retreats all units from the planet to a planet friendly to the old owner, and then changes planet owner to the newOwner parameter

ChangePlanetOwnerAndReplace
This function is effectively the same as the above, but changes despawns all units on the planet, and replaces them with the exact same units for the newOwner parameter
it can be run using ``ChangePlanetOwnerAndRetreat(planets, newOwner)``

``Destroy_Planet(planet)``
param planet Planet GameObjectWrapper
destroys planet using Dummy Planet Killer

### GalacticUtil.lua
functions from this file can be accessed from any lua script using ``require("eawx-util/GalacticUtil)``
this file contains a variety of functions that are useful for doing things on the galactic level, such as spawning things, finding friendly planet, and finding planets
the most notable usage of this is spawning units on galactic map after boarding, I would personally advise using the functions in storyutil, or base game functions instead of these

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


#EaWX Galactic Framework
For information, and documentation, as well as guides on how to use and modify the eawx-galactic-framework, view the readme in the repository below
https://github.com/SvenMarcus/eawx-galactic-framework
