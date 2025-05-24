local TeleportService = game:GetService("TeleportService")
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local placeId = 107667487587870 -- Your Place ID for inter-place teleports
local leaveGuiEvent = ReplicatedStorage:WaitForChild("LeaveGuiEvent")

local INITIAL_COUNTDOWN_DURATION = 3 -- seconds
local MAX_PLAYERS_PER_ROOM = 5

local rooms = {}

-- Initialize rooms table for Room1 to Room5
for i = 1, 5 do
    local roomName = "Room" .. i
    local roomModel = workspace:WaitForChild(roomName)

    if roomModel then
        local barrierPartName = "Barrier" .. i
        local teleportPartName = "TeleportPart" .. i
        local exitPointName = "ExitPoint" .. i

        local barrierInstance = roomModel:WaitForChild(barrierPartName)
        local teleportPartInstance = roomModel:WaitForChild(teleportPartName)
        local exitPointInstance = roomModel:WaitForChild(exitPointName)
        
        -- PortalGui is a child of BarrierX
        local portalGui = barrierInstance:WaitForChild("PortalGui") 
        local playersImageFrame = portalGui:WaitForChild("PlayersImage")
        local statusFrame = portalGui:WaitForChild("StatusFrame")

        rooms[roomName] = {
            name = roomName,
            model = roomModel,
            barrier = barrierInstance,
            teleportPart = teleportPartInstance,
            exitPoint = exitPointInstance,
            portalGui = portalGui,

            playersImageFrame = playersImageFrame,
            avatarImageTemplate = playersImageFrame:WaitForChild("imageLabel"),

            statusTextLabel = statusFrame:WaitForChild("Status"),
            timeTextLabel = statusFrame:WaitForChild("time"), -- Corrected to lowercase 't'
            playersTextLabel = statusFrame:WaitForChild("players"), -- Corrected to lowercase 'p'

            players = {},
            maxPlayers = MAX_PLAYERS_PER_ROOM,
            timer = INITIAL_COUNTDOWN_DURATION,
            initialTimer = INITIAL_COUNTDOWN_DURATION,
            teleporting = false,
            isin = false,
            characterRemovingConnections = {}
        }
    else
        warn("RoomManager: Could not find model for " .. roomName)
    end
end

local function updateGui(room)
    if not room or not room.portalGui or not room.portalGui.Parent then
        -- warn("updateGui: Room or PortalGui not valid for room: " .. (room and room.name or "Unknown"))
        return
    end

	if room.playersTextLabel then
		room.playersTextLabel.Text = #room.players .. "/" .. room.maxPlayers
	end
	if room.timeTextLabel then
		room.timeTextLabel.Text = tostring(math.max(0, math.floor(room.timer)))
	end
	if room.statusTextLabel then
		if room.teleporting then
			room.statusTextLabel.Text = "TELEPORTING"
			room.statusTextLabel.TextColor3 = Color3.fromRGB(255, 0, 0)
			if room.timeTextLabel then room.timeTextLabel.Visible = false end
		else
            if #room.players > 0 then
			    room.statusTextLabel.Text = "GAME STARTING"
			    room.statusTextLabel.TextColor3 = Color3.fromRGB(255, 165, 0) -- Orange
            else
                room.statusTextLabel.Text = "WAITING"
                room.statusTextLabel.TextColor3 = Color3.fromRGB(0, 255, 0) -- Green
            end
			if room.timeTextLabel then room.timeTextLabel.Visible = true end
		end
	end
end

local function isPlayerInRoom(room, playerName)
	for _, name in room.players do
		if name == playerName then
			return true
		end
	end
	return false
end

local function cleanupPlayerAvatar(room, playerName)
	if room.playersImageFrame then
		local playerImageLabel = room.playersImageFrame:FindFirstChild(playerName .. "_ImageLabel")
		if playerImageLabel then
			playerImageLabel:Destroy()
		end
	end
end

local function disconnectCharacterRemoving(room, playerName)
	if room.characterRemovingConnections[playerName] then
		room.characterRemovingConnections[playerName]:Disconnect()
		room.characterRemovingConnections[playerName] = nil
	end
end

local function removePlayerFromRoomInternal(room, playerName)
	for i, name in room.players do
		if name == playerName then
			table.remove(room.players, i)
			return true
		end
	end
	return false
end

local function resetRoom(room)
	room.players = {}
	room.isin = false
	room.timer = room.initialTimer
	room.teleporting = false
	for playerName, conn in room.characterRemovingConnections do
		if conn then conn:Disconnect() end
	end
	room.characterRemovingConnections = {}
	if room.playersImageFrame then
		for _, child in room.playersImageFrame:GetChildren() do
			if child:IsA("ImageLabel") and child.Name ~= room.avatarImageTemplate.Name then
				child:Destroy()
			end
		end
	end
	updateGui(room)
end

local function removePlayerFromRoom(room, playerName)
	if removePlayerFromRoomInternal(room, playerName) then
		cleanupPlayerAvatar(room, playerName)
		disconnectCharacterRemoving(room, playerName)
		
		if #room.players == 0 and not room.teleporting then
			room.isin = false
			room.timer = room.initialTimer 
		end
		updateGui(room)
		return true
	end
	return false
end

local function addPlayerToRoom(room, player)
	if not player or not player.Character then
		warn("AddPlayerToRoom: Invalid player or character for player: " .. (player and player.Name or "nil"))
		return false
	end

    -- Check if player is already in any room
    for _, otherRoom in rooms do
        if isPlayerInRoom(otherRoom, player.Name) then
            -- warn(player.Name .. " tried to join " .. room.name .. " but is already in " .. otherRoom.name)
            return false
        end
    end

	if room.teleporting then 
		-- print(player.Name .. " tried to join " .. room.name .. " but it's teleporting.")
		return false 
	end
	if #room.players >= room.maxPlayers then 
		-- print(player.Name .. " tried to join " .. room.name .. " but it's full.")
		return false 
	end

	local char = player.Character
	table.insert(room.players, player.Name)

	local humanoidRootPart = char:FindFirstChild("HumanoidRootPart")
	if humanoidRootPart and room.teleportPart then
		humanoidRootPart.CFrame = room.teleportPart.CFrame * CFrame.new(0, 3, 0) -- Elevate slightly to avoid floor clipping
	else
		warn("AddPlayerToRoom: Could not move player " .. player.Name .. ". Missing HumanoidRootPart or room.teleportPart for room " .. room.name)
	end

	if room.playersImageFrame and room.avatarImageTemplate then
		cleanupPlayerAvatar(room, player.Name) 
		local newImageLabel = room.avatarImageTemplate:Clone()
        local success, userId = pcall(function() return Players:GetUserIdFromNameAsync(player.Name) end)
        if success and userId then
		    newImageLabel.Image = "rbxthumb://type=AvatarHeadShot&id=" .. userId .. "&w=100&h=100"
        else
            warn("AddPlayerToRoom: Could not get UserId for " .. player.Name)
            newImageLabel.Image = "" -- Default or placeholder
        end
		newImageLabel.Name = player.Name .. "_ImageLabel"
		newImageLabel.Visible = true
		newImageLabel.Parent = room.playersImageFrame
	else
		warn("AddPlayerToRoom: Missing playersImageFrame or avatarImageTemplate for room " .. room.name)
	end
	
	disconnectCharacterRemoving(room, player.Name)
	local conn = player.CharacterRemoving:Connect(function()
		removePlayerFromRoom(room, player.Name)
	end)
	room.characterRemovingConnections[player.Name] = conn
	
	if not room.isin then 
		room.isin = true
		room.timer = room.initialTimer
	end
    
    leaveGuiEvent:FireClient(player) 
	updateGui(room)
	return true
end

local function teleportPlayers(room)
	if #room.players == 0 then
		resetRoom(room)
		return
	end

	room.teleporting = true
	updateGui(room)

	local playersToTeleportObjects = {}
	local currentPlayersInRoomNames = {} 

	for _, playerName in room.players do
		local player = Players:FindFirstChild(playerName)
		if player then
			table.insert(playersToTeleportObjects, player)
			table.insert(currentPlayersInRoomNames, playerName)
		else
			cleanupPlayerAvatar(room, playerName)
			disconnectCharacterRemoving(room, playerName)
		end
	end
    
    room.players = currentPlayersInRoomNames

	if #playersToTeleportObjects == 0 then
		resetRoom(room)
		return
	end

	local success, result
	local attempt = 0
	local maxAttempts = 3
	
	while attempt < maxAttempts and not success do
		attempt = attempt + 1
		success, result = pcall(function()
			return TeleportService:ReserveServer(placeId)
		end)
		if not success then
			warn("Failed to reserve server for room " .. room.name .. " (attempt " .. attempt .. "): " .. tostring(result))
			if attempt < maxAttempts then task.wait(1) end
		end
	end

	if not success then
		warn("Max attempts reached for ReserveServer for room " .. room.name .. ". Teleportation cancelled.")
		room.teleporting = false
		room.timer = room.initialTimer
		updateGui(room)
		return
	end
	
	local privateServerCode = result
	
	success, result = pcall(function()
		TeleportService:TeleportToPrivateServer(placeId, privateServerCode, playersToTeleportObjects)
	end)

	if not success then
		warn("TeleportToPrivateServer failed for room " .. room.name .. ": " .. tostring(result))
		room.teleporting = false
		room.timer = room.initialTimer
		updateGui(room)
		return
	end

	task.wait(3) 
	resetRoom(room)
end


-- Initialize rooms and connect events
for roomNameKey, roomData in rooms do
	if not roomData.barrier or not roomData.teleportPart or not roomData.exitPoint or 
	   not roomData.statusTextLabel or not roomData.timeTextLabel or not roomData.playersTextLabel or
	   not roomData.playersImageFrame or not roomData.avatarImageTemplate then
		warn("Room '" .. roomData.name .. "' is missing one or more required components. Skipping setup for this room.")
		continue
	end

	resetRoom(roomData) 

	roomData.barrier.Touched:Connect(function(hit)
		local char = hit.Parent
		if char then
			local player = Players:GetPlayerFromCharacter(char)
			if player then
                if player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
				    addPlayerToRoom(roomData, player)
                end
			end
		end
	end)
end

leaveGuiEvent.OnServerEvent:Connect(function(player)
	if not player then return end

	for roomNameKey, roomData in rooms do
		if isPlayerInRoom(roomData, player.Name) then
			if player.Character then
				local humanoidRootPart = player.Character:FindFirstChild("HumanoidRootPart")
				if humanoidRootPart and roomData.exitPoint then
					humanoidRootPart.Anchored = false
                    local targetCFrame = CFrame.new(roomData.exitPoint.Position + Vector3.new(0, 3, 0))
					player.Character:PivotTo(targetCFrame)
				end
			end
			removePlayerFromRoom(roomData, player.Name)
            if #roomData.players == 0 and not roomData.teleporting then
                roomData.isin = false
                roomData.timer = roomData.initialTimer
                updateGui(roomData)
            end
			break 
		end
	end
end)

RunService.Heartbeat:Connect(function(deltaTime)
	for roomNameKey, roomData in rooms do
		if roomData.teleporting then continue end

		if #roomData.players > 0 then
			if not roomData.isin then 
				roomData.isin = true
				roomData.timer = roomData.initialTimer
			end
			
			roomData.timer = roomData.timer - deltaTime
			updateGui(roomData)

			if roomData.timer <= 0 then
				teleportPlayers(roomData)
			end
		else 
			if roomData.isin then 
				roomData.isin = false
				roomData.timer = roomData.initialTimer
				updateGui(roomData)
			end
		end
	end
end)
