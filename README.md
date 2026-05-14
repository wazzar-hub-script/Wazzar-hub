# Wazzar-hub
local scriptString = [[
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TeleportService = game:GetService("TeleportService")
local RunService = game:GetService("RunService")
local HttpService = game:GetService("HttpService")

local player = Players.LocalPlayer
local character = player.Character or player.CharacterAdded:Wait()
local humanoidRootPart = character:WaitForChild("HumanoidRootPart")

local remoteEvents = ReplicatedStorage:WaitForChild("Remotes")
local serverHopCooldown = 60 -- seconds cooldown between server hops

-- Configurations
local farmTargets = {
    "Full Moon", -- enemy names for farming
    "Bones",
    "Mirage",
    "Trial"
}

local farmRadius = 100 -- radius to search for enemies
local autoFarmEnabled = true
local autoServerHopEnabled = true

-- Utility functions
local function getEnemiesByName(names)
    local enemies = {}
    for _, npc in pairs(workspace.Enemies:GetChildren()) do
        if table.find(names, npc.Name) and npc:FindFirstChild("Humanoid") and npc.Humanoid.Health > 0 then
            table.insert(enemies, npc)
        end
    end
    return enemies
end

local function moveToPosition(position)
    local pathfindingService = game:GetService("PathfindingService")
    local path = pathfindingService:CreatePath()
    path:ComputeAsync(humanoidRootPart.Position, position)
    if path.Status == Enum.PathStatus.Success then
        local waypoints = path:GetWaypoints()
        for _, waypoint in pairs(waypoints) do
            humanoidRootPart.CFrame = CFrame.new(waypoint.Position + Vector3.new(0, 3, 0))
            wait(0.1)
        end
    else
        humanoidRootPart.CFrame = CFrame.new(position + Vector3.new(0, 3, 0))
    end
end

local function attackEnemy(enemy)
    if enemy and enemy:FindFirstChild("Humanoid") and enemy.Humanoid.Health > 0 then
        -- Move close to enemy
        moveToPosition(enemy.HumanoidRootPart.Position)
        -- Attack loop
        while enemy.Humanoid.Health > 0 and autoFarmEnabled do
            -- Fire attack remote event (example, adjust to actual game remotes)
            if remoteEvents:FindFirstChild("Attack") then
                remoteEvents.Attack:FireServer()
            end
            wait(0.5)
        end
    end
end

local function farm()
    while autoFarmEnabled do
        local enemies = getEnemiesByName(farmTargets)
        if #enemies == 0 then
            wait(2)
        else
            for _, enemy in pairs(enemies) do
                attackEnemy(enemy)
                if not autoFarmEnabled then break end
            end
        end
        wait(1)
    end
end

local function getServerList()
    local servers = {}
    local placeId = game.PlaceId
    local cursor = nil
    repeat
        local url = "https://games.roblox.com/v1/games/"..placeId.."/servers/Public?sortOrder=Asc&limit=100"
        if cursor then
            url = url .. "&cursor=" .. cursor
        end
        local response = HttpService:GetAsync(url)
        local data = HttpService:JSONDecode(response)
        for _, server in pairs(data.data) do
            if server.playing < server.maxPlayers then
                table.insert(servers, server.id)
            end
        end
        cursor = data.nextPageCursor
    until not cursor
    return servers
end

local function serverHop()
    while autoServerHopEnabled do
        wait(serverHopCooldown)
        local servers = getServerList()
        if #servers > 0 then
            local randomServer = servers[math.random(1, #servers)]
            TeleportService:TeleportToPlaceInstance(game.PlaceId, randomServer, player)
            break
        end
    end
end

local function autoTrial()
    while autoFarmEnabled do
        local trialNpc = workspace:FindFirstChild("Trial")
        if trialNpc and trialNpc:FindFirstChild("Humanoid") and trialNpc.Humanoid.Health > 0 then
            attackEnemy(trialNpc)
        end
        wait(5)
    end
end

local function autoFindMirage()
    while autoFarmEnabled do
        local mirage = workspace:FindFirstChild("Mirage")
        if mirage and mirage:FindFirstChild("Humanoid") and mirage.Humanoid.Health > 0 then
            attackEnemy(mirage)
        end
        wait(5)
    end
end

-- Start farming and server hopping in parallel
spawn(farm)
spawn(autoFindMirage)
spawn(autoTrial)
spawn(serverHop)
]]

local func = loadstring(scriptString)
if func then
    func()
else
    error("Failed to load script string")
end
