# Scrap Mechanic Blueprint Import-Export Guide

This repository is attended to guide you into the implementation of the importation and exportation commands, 
allowing you to import and export your contraception blueprints from creative mode to survival mode.

The fact that commands `/import` and `/export` doesn't work natively is that they are not implemented in the creative mode, and not bound without the dev-mode enabled.

## Compatibility

You can check the [build ID of the last public release in SteamDB](https://steamdb.info/app/387990/depots/)

Working for build ID : `6956308` (30th june 2021)

## Requirements

- A working version of *Scrap Mechanic* (on *Steam*)
- Text editor (that can preserve file encoding and line separator) for Windows, the native notepad or [Notepad++](https://notepad-plus-plus.org/downloads/) are good choices.

## Implementing the commands

### 1/ Open Scrap Mechanic installation folder

While being in your *Steam* game library, do `right-click` on `Scrap Mechanic`, click `Properties...`, select `LOCAL FILES` and click `Browse...`

Explorer will open your *Scrap Mechanic* path, typically the directory will be something like: 
```
C:\Program Files (x86)\Steam\steamapps\common\Scrap Mechanic
```

### 2/ Open creative and survival game files

Open the file named `CreativeGame.lua` in the following directory:  

```
[...]\Steam\steamapps\common\Scrap Mechanic\Data\Scripts\game\CreativeGame.lua
```

In parallel, open the second file named `SurvivalGame.lua` in the following directory:

```
[...]\Steam\steamapps\common\Scrap Mechanic\Survival\Scripts\game\SurvivalGame.lua
```
<sup>Note that this file is in `Survival/` directory instead of `Data/`.</sup>

### 3/ Implement import-export commands for creative mode

Search a function named `CreativeGame.client_onCreate` (around line: 47).

Edit the function as shown below: 

```lua
function CreativeGame.client_onCreate( self )
    -- add this function call
    self:bindImportExportCmds()
    
    -- [... rest of the function code, left untouched ...]
end
```

After the `end` keyword of this function, add the following functions: 

```lua
function CreativeGame.bindImportExportCmds( self )
	sm.game.bindChatCommand( "/export", { { "string", "name", false } }, "cl_onChatImportExportCmds", "Exports blueprint $SURVIVAL_DATA/LocalBlueprints/<name>.blueprint" )
	sm.game.bindChatCommand( "/import", { { "string", "name", false } }, "cl_onChatImportExportCmds", "Imports blueprint from survival $SURVIVAL_DATA/LocalBlueprints/<name>.blueprint" )
end

function CreativeGame.cl_onChatImportExportCmds( self, params )
	if params[1] == "/export" then
		local rayCastValid, rayCastResult = sm.localPlayer.getRaycast( 100 )
		if rayCastValid and rayCastResult.type == "body" then
			local importParams = {
				name = params[2],
				body = rayCastResult:getBody()
			}
			self.network:sendToServer( "sv_exportCreation", importParams )
		end
	elseif params[1] == "/import" then
		local rayCastValid, rayCastResult = sm.localPlayer.getRaycast( 100 )
		if rayCastValid then
			local importParams = {
				world = sm.localPlayer.getPlayer().character:getWorld(),
				name = params[2],
				position = rayCastResult.pointWorld
			}
			self.network:sendToServer( "sv_importCreation", importParams )
		end
	end
end

function CreativeGame.sv_exportCreation( self, params )
	local obj = sm.json.parseJsonString( sm.creation.exportToString( params.body ) )
	sm.json.save( obj, "$SURVIVAL_DATA/LocalBlueprints/"..params.name..".blueprint" )
end

function CreativeGame.sv_importCreation( self, params )
	local status, error = pcall( sm.creation.importFromFile( params.world, "$SURVIVAL_DATA/LocalBlueprints/" .. params.name .. ".blueprint", params.position ) )

	if status == false then
		self.network:sendToClients("client_showMessage", "An error occurred (some parts are survival-only and cannot be imported to creative mode)")
	end
end
```

Save the file.

### 4/ Implement import-export commands for survival mode

Search a function named `SurvivalGame.client_onCreate` (around line: 99).

Edit the function as shown below: 

```lua
function SurvivalGame.client_onCreate( self )
    -- add this function call
    self:bindImportExportCmds()
    
    -- [... rest of the function code, left untouched ...]
end
```

After the `end` keyword of this function, add the following function: 

```lua
function SurvivalGame.bindImportExportCmds( self )
	sm.game.bindChatCommand( "/import", { { "string", "name", false } }, "cl_onChatCommand", "Imports blueprint $SURVIVAL_DATA/LocalBlueprints/<name>.blueprint" )
	sm.game.bindChatCommand( "/export", { { "string", "name", false } }, "cl_onChatCommand", "Exports blueprint $SURVIVAL_DATA/LocalBlueprints/<name>.blueprint" )
end
```

Save the file.

### 5/ Using the commands in game

If you've correctly implemented the functions the game should be running fine. 

Export a creation: 
- Start a creative game
- Build your contraception
- Point your aim at your creation (your creation **must be detached of the world** or other creation)
- Open the chat (`enter`) and type the following command: `/export MyCreationBlueprint`
- Quit the creative game

Import a creation: 
- Start a survival game
- Point your aim at a position near you
- Open the chat (`enter`) and type the following command: `/import MyCreationBlueprint`
- Your contraception should spawn nearby
