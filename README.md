local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local LocalPlayer = Players.LocalPlayer

local REACH_DISTANCE = 35
local UPDATE_INTERVAL = 0.05
local CACHE_LIFETIME = 0.5

local remoteEventsCache = {}
local lastCacheUpdate = 0

local function getRemoteEvents()
    local currentTime = tick()
    
    if remoteEventsCache.events and (currentTime - lastCacheUpdate) < CACHE_LIFETIME then
        return remoteEventsCache.events
    end
    
    local events = {}
    local searchLocations = {
        game:GetService("ReplicatedStorage"),
        game:GetService("ReplicatedFirst"),
        workspace
    }
    
    for _, location in ipairs(searchLocations) do
        for _, v in ipairs(location:GetDescendants()) do
            if (v:IsA("RemoteEvent") or v:IsA("RemoteFunction")) and 
               (v.Name:lower():find("ball") or v.Name:lower():find("steal")) then
                table.insert(events, v)
            end
        end
    end
    
    remoteEventsCache.events = events
    lastCacheUpdate = currentTime
    
    return events
end

local function findNearbyBallsForSteal()
    local character = LocalPlayer.Character
    if not character then return {} end
    
    local hrp = character:FindFirstChild("HumanoidRootPart")
    if not hrp then return {} end
    
    local nearbyBalls = {}
    local checkedBalls = {}
    
    -- Checking if the ball is within reach
    if workspace:FindFirstChild("Basketball") then
        local basketballFolder = workspace.Basketball
        
        if basketballFolder:FindFirstChild("Ball") and basketballFolder.Ball:IsA("BasePart") then
            local ball = basketballFolder.Ball
            local distance = (hrp.Position - ball.Position).Magnitude
            if distance <= REACH_DISTANCE and not checkedBalls[ball] then
                table.insert(nearbyBalls, ball)
                checkedBalls[ball] = true
            end
        end
    end
    
    return nearbyBalls
end

local function stealBall(ball)
    local character = LocalPlayer.Character
    if not character then return end
    
    local hrp = character:FindFirstChild("HumanoidRootPart")
    if not hrp then return end

    -- Simulate stealing action by using the ball's RemoteEvent or touch interactions
    local stealEvent = getRemoteEvents()
    for _, event in pairs(stealEvent) do
        -- Trigger the steal action here, for example, fire the remote event
        pcall(function()
            event:FireServer(ball)
        end)
    end
end

local function enableBallSteal()
    local lastUpdate = tick()
    local updateInterval = UPDATE_INTERVAL
    
    local triggeredBalls = {}
    
    RunService.Heartbeat:Connect(function()
        if tick() - lastUpdate < updateInterval then return end
        lastUpdate = tick()
        
        local character = LocalPlayer.Character
        if not character then return end
        
        local hrp = character:FindFirstChild("HumanoidRootPart")
        if not hrp then return end
        
        local nearbyBalls = findNearbyBallsForSteal()
        
        for _, ball in ipairs(nearbyBalls) do
            local distance = (hrp.Position - ball.Position).Magnitude
            
            if distance <= REACH_DISTANCE then
                local ballId = tostring(ball)
                local currentTime = tick()
                
                -- Trigger the steal action
                if not triggeredBalls[ballId] then
                    stealBall(ball)
                    triggeredBalls[ballId] = currentTime
                end
            end
        end
    end)
end

enableBallSteal()

return {
    setReachDistance = function(distance)
        REACH_DISTANCE = distance
    end,
    setPerformance = function(level)
        if level == 1 then
            UPDATE_INTERVAL = 0.1
            CACHE_LIFETIME = 1.0
        elseif level == 2 then
            UPDATE_INTERVAL = 0.05
            CACHE_LIFETIME = 0.5
        elseif level == 3 then
            UPDATE_INTERVAL = 0.03
            CACHE_LIFETIME = 0.3
        end
    end
}
