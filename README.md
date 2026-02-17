-- Performance monitoring
local function safeCall(func, ...)
    local start = tick()
    local success, result = pcall(func, ...)
    local duration = tick() - start
    
    if duration > 0.08 then
        warn("Slow function:", debug.info(func, "n") or "anonymous", "took", duration, "seconds")
    end
    
    return success, result
end

-- Cleanup previous execution
if _G.MySkyCleanup then
    pcall(_G.MySkyCleanup)
end

_G.MySkyCleanup = function()
    if _G.MySkyConnections then
        for _, conn in ipairs(_G.MySkyConnections) do
            pcall(function() conn:Disconnect() end)
        end
        _G.MySkyConnections = {}
    end
    
    -- Clean up parts
    for _, part in ipairs(workspace:GetChildren()) do
        if part.Name:find("myskyp") or part.Name:find("Cosmic") then
            pcall(function() part:Destroy() end)
        end
    end
    
    -- Clean up GUI
    local coreGui = game:GetService("CoreGui")
    local existingGui = coreGui:FindFirstChild("CloudSEMITP")
    if existingGui then
        pcall(function() existingGui:Destroy() end)
    end
end

-- Execute cleanup on script restart
pcall(_G.MySkyCleanup)

-- // ------------------------------------------------ //
-- //                  MAIN SCRIPT                     //
-- // ------------------------------------------------ //

-- Load services in background to prevent freezing
local Services = {}
local servicePromises = {}

-- Function to get services asynchronously
local function getService(serviceName)
    if Services[serviceName] then
        return Services[serviceName]
    end
    
    -- Load service in background
    if not servicePromises[serviceName] then
        servicePromises[serviceName] = task.spawn(function()
            Services[serviceName] = game:GetService(serviceName)
            servicePromises[serviceName] = nil
        end)
    end
    
    -- Wait for service if needed immediately
    while not Services[serviceName] do
        task.wait()
    end
    
    return Services[serviceName]
end

-- Load critical services first without freezing
local Players = getService("Players")
local LocalPlayer = Players.LocalPlayer

-- Connection storage
local connections = {}
_G.MySkyConnections = connections

print("Script loaded successfully for user:", LocalPlayer.Name)

-- Prevent duplicate execution
if _G.MyskypInstaSteal then 
    pcall(_G.MySkyCleanup)
    task.wait(0.1)
end
_G.MyskypInstaSteal = true

-- Configuration constants - Define positions for BOTH bases
local TP_POSITIONS = {
    BASE1 = {
        INFO_POS = CFrame.new(334.76, 55.334, 99.40),
        TELEPORT_POS = CFrame.new(-352.98, -7.30, 74.3),
        STAND_HERE_PART = CFrame.new(-334.76, -5.334, 99.40) * CFrame.new(0, 2.6, 0)
    },
    BASE2 = {
        INFO_POS = CFrame.new(334.76, 55.334, 19.17),
        TELEPORT_POS = CFrame.new(-352.98, -7.30, 45.76),
        STAND_HERE_PART = CFrame.new(-336.41, -5.34, 19.20) * CFrame.new(0, 2.6, 0)
    }
}

local TP_DELAY = 0.15

-- State variables
local TPSysEnabled = true
local DesyncActive = false

-- Performance variables
local lastTeleportTime = 0
local TELEPORT_COOLDOWN = 0.8
local lastMarkerUpdate = 0
local MARKER_UPDATE_INTERVAL = 0.4

-- Device detection
local UserInputService = getService("UserInputService")
local TweenService = getService("TweenService")
local IS_MOBILE = UserInputService.TouchEnabled
local IS_PC = not IS_MOBILE

print("Device detection:", IS_MOBILE and "MOBILE" or "PC")

-- Utility functions
local function cleanup()
    for _, conn in ipairs(connections) do
        pcall(function() conn:Disconnect() end)
    end
    connections = {}
end

-- ============================================================
-- ORIGINAL TELEPORT PATH FUNCTION
-- ============================================================
local function TeleportPath(targetCFrame)
    if tick() - lastTeleportTime < TELEPORT_COOLDOWN then 
        return 
    end
    lastTeleportTime = tick()
    
    if not LocalPlayer.Character or not LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then 
        return 
    end
    
    local hrp = LocalPlayer.Character.HumanoidRootPart
    
    local distance = (hrp.Position - targetCFrame.Position).Magnitude
    if distance > 1000 then
        warn("Target too far for pathfinding:", distance)
        return
    end
    
    if distance < 50 then
        hrp.AssemblyLinearVelocity = Vector3.zero
        LocalPlayer.Character:PivotTo(targetCFrame)
        task.wait(TP_DELAY)
        return
    end

    task.spawn(function()
        local PathfindingService = getService("PathfindingService")
        local success, path = pcall(function()
            local path = PathfindingService:CreatePath({ 
                AgentCanJump = true, 
                AgentRadius = 3,
                AgentHeight = 5,
                AgentCanClimb = false
            })
            path:ComputeAsync(hrp.Position, targetCFrame.Position)
            return path
        end)
        
        if not success or not path then return end
        
        if path.Status == Enum.PathStatus.Success then
            local waypoints = path:GetWaypoints()
            local stepCount = math.min(4, #waypoints)
            
            for i = 1, stepCount do
                if not LocalPlayer.Character or not LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
                    break
                end
                
                local frac = i / stepCount
                local idx = math.clamp(math.floor(frac * #waypoints) + 1, 1, #waypoints)
                hrp.AssemblyLinearVelocity = Vector3.zero
                LocalPlayer.Character:PivotTo(CFrame.new(waypoints[idx].Position + Vector3.new(0, 2.5, 0)))
                task.wait(0.08)
            end
        end

        if LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
            LocalPlayer.Character:PivotTo(targetCFrame)
            task.wait(TP_DELAY)
        end
    end)
end

-- Determine which base the player is at
local function getCurrentBasePosition()
    if not LocalPlayer.Character or not LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
        return TP_POSITIONS.BASE1.INFO_POS
    end
    
    local hrp = LocalPlayer.Character.HumanoidRootPart
    local currentPos = hrp.Position
    
    local distToBase1 = (currentPos - TP_POSITIONS.BASE1.INFO_POS.Position).Magnitude
    local distToBase2 = (currentPos - TP_POSITIONS.BASE2.INFO_POS.Position).Magnitude
    
    if distToBase1 < distToBase2 then
        return TP_POSITIONS.BASE1.INFO_POS
    else
        return TP_POSITIONS.BASE2.INFO_POS
    end
end

-- Create marker
local function CreateMarker()
    local currentBasePos = getCurrentBasePosition()
    local markerPos = currentBasePos * CFrame.new(0, -3.2, 0)
    
    local part = workspace:FindFirstChild("myskypBest") or Instance.new("Part")
    part.Name = "myskypBest"
    part.Size = Vector3.new(1, 1, 1)
    part.CFrame = markerPos
    part.Anchored = true
    part.CanCollide = false
    part.Material = Enum.Material.Neon
    part.Color = Color3.fromRGB(0, 255, 255)
    part.Transparency = 0.3
    part.Parent = workspace

    if not part:FindFirstChild("MyskypBillboard") then
        local gui = Instance.new("BillboardGui")
        gui.Name = "MyskypBillboard"
        gui.AlwaysOnTop = true
        gui.Size = UDim2.new(0, 200, 0, 50)
        gui.ExtentsOffset = Vector3.new(0, 3, 0)
        gui.Parent = part
        
        local label = Instance.new("TextLabel")
        label.Name = "Text"
        label.BackgroundTransparency = 1
        label.Size = UDim2.new(1, 0, 1, 0)
        label.Font = Enum.Font.GothamBold
        label.TextSize = 18
        label.TextColor3 = Color3.new(1, 1, 1)
        label.Text = ""
        label.Parent = gui
    end
    
    return part
end

local Marker = CreateMarker()

-- Create cosmic indicators
local function createCosmicIndicator(name, position, color, text)
    local part = Instance.new("Part")
    part.Name = name
    part.Size = Vector3.new(3.8, 0.3, 3.8)
    part.Material = Enum.Material.Plastic
    part.Color = color
    part.Transparency = 0.57
    part.Anchored = true
    part.CanCollide = false
    part.Position = position
    part.Parent = workspace

    local billboard = Instance.new("BillboardGui")
    billboard.Size = UDim2.new(0, 200, 0, 50)
    billboard.StudsOffset = Vector3.new(0, 4, 0)
    billboard.AlwaysOnTop = true
    billboard.Parent = part

    local textLabel = Instance.new("TextLabel")
    textLabel.Size = UDim2.new(1, 0, 1, 0)
    textLabel.BackgroundTransparency = 1
    textLabel.Text = text
    textLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    textLabel.TextStrokeTransparency = 0.3
    textLabel.TextStrokeColor3 = color
    textLabel.Font = Enum.Font.GothamBold
    textLabel.TextSize = 18
    textLabel.Parent = billboard

    return part
end

task.spawn(function()
    createCosmicIndicator(
        "CosmicStandHereBase1",
        Vector3.new(-334.84, -5.40, 101.02),
        Color3.fromRGB(39, 39, 39),
        " STAND HERE (BASE 1) "
    )

    createCosmicIndicator(
        "CosmicTeleportHereBase1",
        Vector3.new(-352.98, -7.30, 74.3),
        Color3.fromRGB(39, 39, 39),
        " TELEPORT HERE (BASE 1) "
    )

    createCosmicIndicator(
        "CosmicStandHereBase2",
        Vector3.new(-334.84, -5.40, 19.20),
        Color3.fromRGB(39, 39, 39),
        " STAND HERE (BASE 2) "
    )

    createCosmicIndicator(
        "CosmicTeleportHereBase2",
        Vector3.new(-352.98, -7.30, 45.76),
        Color3.fromRGB(39, 39, 39),
        " TELEPORT HERE (BASE 2) "
    )
end)

-- ============================================================
-- STEAL PROMPT DETECTION
-- ============================================================

local function initializeEventConnections()
    local ProximityPromptService = getService("ProximityPromptService")
    local RunService = getService("RunService")
    
    local promptConn = ProximityPromptService.PromptButtonHoldEnded:Connect(function(prompt, who)
        if who ~= LocalPlayer then return end
        if prompt.Name ~= "Steal" and prompt.ActionText ~= "Steal" and prompt.ObjectText ~= "Steal" then return end
        
        warn("STEAL DETECTED")
        
        if not TPSysEnabled then
            warn("TP System OFF")
            return
        end
        
        local character = LocalPlayer.Character
        if not character then return end
        
        local hrp = character:FindFirstChild("HumanoidRootPart")
        if not hrp then return end
        
        local backpack = LocalPlayer:FindFirstChild("Backpack")
        if backpack then
            local carpet = backpack:FindFirstChild("Flying Carpet")
            if carpet and character:FindFirstChild("Humanoid") then
                character.Humanoid:EquipTool(carpet)
                task.wait(0.1)
            end
        end
        
        local currentBasePos = getCurrentBasePosition()
        
        if currentBasePos == TP_POSITIONS.BASE1.INFO_POS then
            hrp.CFrame = TP_POSITIONS.BASE1.TELEPORT_POS
            print("TPED To Base 1")
        else
            hrp.CFrame = TP_POSITIONS.BASE2.TELEPORT_POS
            print("TPED To Base 2")
        end
    end)
    table.insert(connections, promptConn)
    
    local heartbeatConn = RunService.Heartbeat:Connect(function(deltaTime)
        lastMarkerUpdate = lastMarkerUpdate + deltaTime
        if lastMarkerUpdate < MARKER_UPDATE_INTERVAL then return end
        lastMarkerUpdate = 0
        
        local character = LocalPlayer.Character
        if not character then return end
        
        local hrp = character:FindFirstChild("HumanoidRootPart")
        if not hrp then return end
        
        local currentBasePos = getCurrentBasePosition()
        local markerPos = currentBasePos * CFrame.new(0, -3.2, 0)
        
        if Marker and Marker.Parent then
            Marker.CFrame = markerPos
            
            local dist = (hrp.Position - currentBasePos.Position).Magnitude
            Marker.Color = dist < 7 and Color3.fromRGB(0, 255, 100) or Color3.fromRGB(255, 50, 50)
        end
    end)
    table.insert(connections, heartbeatConn)
    
    print("Event connections initialized")
end

task.spawn(initializeEventConnections)

-- ============================================================
-- OPTIMIZED ANTI-KICK SYSTEM
-- ============================================================

local KEYWORD = "You stole"
local KICK_MESSAGE = "EZZ"

local function hasKeyword(text)
    return typeof(text) == "string" and text:lower():find(KEYWORD, 1, true) ~= nil
end

local function watchObject(obj)
    if not (obj:IsA("TextLabel") or obj:IsA("TextButton") or obj:IsA("TextBox")) then return end
    
    if obj.Visible and obj.Text and hasKeyword(obj.Text) then
        task.spawn(function()
            pcall(function() LocalPlayer:Kick(KICK_MESSAGE) end)
        end)
        return
    end
    
    if obj.Visible then
        local conn = obj:GetPropertyChangedSignal("Text"):Connect(function()
            if hasKeyword(obj.Text) then
                task.spawn(function()
                    pcall(function() LocalPlayer:Kick(KICK_MESSAGE) end)
                end)
            end
        end)
        table.insert(connections, conn)
    end
end

local lastScanTime = 0
local SCAN_COOLDOWN = 0.5
local MAX_SCAN_TIME = 0.016

local function optimizedScanDescendants(parent)
    local currentTime = tick()
    if currentTime - lastScanTime < SCAN_COOLDOWN then return end
    
    local startTime = tick()
    if tick() - startTime > MAX_SCAN_TIME then
        warn("Scan would take too long, skipping...")
        return
    end
    
    lastScanTime = currentTime
    
    task.spawn(function()
        local textObjects = {}
        local batchSize = 15
        
        for _, obj in ipairs(parent:GetDescendants()) do
            if tick() - startTime > MAX_SCAN_TIME then
                warn("Scan timeout - stopping early")
                break
            end
            
            if obj:IsA("TextLabel") or obj:IsA("TextButton") or obj:IsA("TextBox") then
                if obj.Visible and obj.Text then
                    table.insert(textObjects, obj)
                end
            end
        end
        
        for i = 1, #textObjects, batchSize do
            local endIdx = math.min(i + batchSize - 1, #textObjects)
            for j = i, endIdx do
                watchObject(textObjects[j])
            end
            task.wait(0.05)
        end
        
        warn("Scan completed in " .. (tick() - startTime) .. " seconds, objects: " .. #textObjects)
    end)
end

local childAddedDebounce = false
local function debouncedWatchObject(obj)
    if not childAddedDebounce then
        childAddedDebounce = true
        watchObject(obj)
        childAddedDebounce = false
    end
end

task.spawn(function()
    local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")
    
    task.wait(2)
    optimizedScanDescendants(PlayerGui)
    
    local childAddedConn = PlayerGui.ChildAdded:Connect(function(gui)
        task.wait(0.1)
        if gui:IsA("ScreenGui") then
            optimizedScanDescendants(gui)
        else
            debouncedWatchObject(gui)
        end
    end)
    table.insert(connections, childAddedConn)
    
    local descendantAddedConn = PlayerGui.DescendantAdded:Connect(debouncedWatchObject)
    table.insert(connections, descendantAddedConn)
end)

task.spawn(function()
    local playerLeavingConn = Players.PlayerRemoving:Connect(function(player)
        if player == LocalPlayer then
            cleanup()
            _G.MyskypInstaSteal = false
            pcall(_G.MySkyCleanup)
        end
    end)
    table.insert(connections, playerLeavingConn)
end)

-- ============================================================
-- DESYNC FFLAG SYSTEM
-- ============================================================

local FFlags = {
    GameNetPVHeaderRotationalVelocityZeroCutoffExponent = -5000,
    LargeReplicatorWrite5 = true,
    LargeReplicatorEnabled9 = true,
    AngularVelociryLimit = 360,
    TimestepArbiterVelocityCriteriaThresholdTwoDt = 2147483646,
    S2PhysicsSenderRate = 15000,
    DisableDPIScale = true,
    MaxDataPacketPerSend = 2147483647,
    PhysicsSenderMaxBandwidthBps = 20000,
    TimestepArbiterHumanoidLinearVelThreshold = 21,
    MaxMissedWorldStepsRemembered = -2147483648,
    PlayerHumanoidPropertyUpdateRestrict = true,
    SimDefaultHumanoidTimestepMultiplier = 0,
    StreamJobNOUVolumeLengthCap = 2147483647,
    DebugSendDistInSteps = -2147483648,
    GameNetDontSendRedundantNumTimes = 1,
    CheckPVLinearVelocityIntegrateVsDeltaPositionThresholdPercent = 1,
    CheckPVDifferencesForInterpolationMinVelThresholdStudsPerSecHundredth = 1,
    LargeReplicatorSerializeRead3 = true,
    ReplicationFocusNouExtentsSizeCutoffForPauseStuds = 2147483647,
    CheckPVCachedVelThresholdPercent = 10,
    CheckPVDifferencesForInterpolationMinRotVelThresholdRadsPerSecHundredth = 1,
    GameNetDontSendRedundantDeltaPositionMillionth = 1,
    InterpolationFrameVelocityThresholdMillionth = 5,
    StreamJobNOUVolumeCap = 2147483647,
    InterpolationFrameRotVelocityThresholdMillionth = 5,
    CheckPVCachedRotVelThresholdPercent = 10,
    WorldStepMax = 30,
    InterpolationFramePositionThresholdMillionth = 5,
    TimestepArbiterHumanoidTurningVelThreshold = 1,
    SimOwnedNOUCountThresholdMillionth = 2147483647,
    GameNetPVHeaderLinearVelocityZeroCutoffExponent = -5000,
    NextGenReplicatorEnabledWrite4 = true,
    TimestepArbiterOmegaThou = 1073741823,
    MaxAcceptableUpdateDelay = 1,
    LargeReplicatorSerializeWrite4 = true
}

local desyncFirstActivation = true
local desyncPermanentlyActivated = false

local function applyFFlags(flags)
    for name, value in pairs(flags) do
        pcall(function()
            setfflag(tostring(name), tostring(value))
        end)
    end
end

local function respawn(plr)
    local char = plr.Character
    if char then
        local hum = char:FindFirstChildOfClass("Humanoid")
        if hum then
            hum:ChangeState(Enum.HumanoidStateType.Dead)
        end
        char:ClearAllChildren()
        local newChar = Instance.new("Model")
        newChar.Parent = workspace
        plr.Character = newChar
        task.wait()
        plr.Character = char
        newChar:Destroy()
    end
end

local function applyPermanentDesync()
    applyFFlags(FFlags)
    if desyncFirstActivation then
        respawn(LocalPlayer)
        desyncFirstActivation = false
    end
    desyncPermanentlyActivated = true
end

_G.MySkyCleanup = function()
    cleanup()
    
    for _, part in ipairs(workspace:GetChildren()) do
        if part.Name:find("myskyp") or part.Name:find("Cosmic") then
            pcall(function() part:Destroy() end)
        end
    end
    
    local CoreGui = getService("CoreGui")
    local existingGui = CoreGui:FindFirstChild("CloudSEMITP")
    if existingGui then
        pcall(function() existingGui:Destroy() end)
    end
    
    _G.MyskypInstaSteal = false
    _G.MySkyConnections = nil
end

-- ============================================================
-- EXECUTE TP FUNCTION - FUNÇÃO DO CÍRCULO
-- ============================================================

local function executeTP()
    if tick() - lastTeleportTime < TELEPORT_COOLDOWN then 
        warn("Teleport on cooldown! Wait", TELEPORT_COOLDOWN - (tick() - lastTeleportTime), "seconds")
        return false
    end
    
    lastTeleportTime = tick()
    
    local Character = LocalPlayer.Character
    if not Character then
        Character = LocalPlayer.CharacterAdded:Wait()
    end
    
    local Humanoid = Character:WaitForChild("Humanoid")
    local HRP = Character:WaitForChild("HumanoidRootPart")
    
    local Carpet = Character:FindFirstChild("Flying Carpet") or LocalPlayer.Backpack:FindFirstChild("Flying Carpet")
    if not Carpet then
        warn("Flying Carpet not found!")
        return false
    end
    
    local currentBasePos = getCurrentBasePosition()
    local isAtBase1 = currentBasePos == TP_POSITIONS.BASE1.INFO_POS
    
    print("[DEBUG] Current base detected:", isAtBase1 and "BASE 1" or "BASE 2")
    
    task.spawn(function()
        Humanoid:EquipTool(Carpet)
        
        if isAtBase1 then
            HRP.CFrame = CFrame.new(-351.49, -6.65, 113.72)
            task.wait(0.15)
            HRP.CFrame = CFrame.new(-378.14, -6.00, 26.43)
            task.wait(0.15)
            HRP.CFrame = CFrame.new(-334.80, -5.04, 18.90)
        else
            HRP.CFrame = CFrame.new(-352.54, -6.83, 6.66)
            task.wait(0.15)
            HRP.CFrame = CFrame.new(-372.90, -6.20, 102.00)
            task.wait(0.15)
            HRP.CFrame = CFrame.new(-335.08, -5.10, 101.40)
        end
    end)
    
    return true
end

-- ============================================================
-- UI MINIMALISTA - CLOUD SEMI-TP V1
-- ============================================================

-- Services
local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")

-- Local Player
local localPlayer = Players.LocalPlayer

-- Modern color palette
local colors = {
    bg = Color3.fromRGB(12, 12, 18),
    bgSecondary = Color3.fromRGB(20, 20, 28),
    bgTertiary = Color3.fromRGB(28, 28, 38),
    accent = Color3.fromRGB(100, 180, 255),
    accentLight = Color3.fromRGB(150, 210, 255),
    accentDark = Color3.fromRGB(70, 140, 220),
    success = Color3.fromRGB(80, 200, 150),
    error = Color3.fromRGB(255, 100, 100),
    warning = Color3.fromRGB(255, 180, 70),
    text = Color3.fromRGB(255, 255, 255),
    textSecondary = Color3.fromRGB(180, 190, 210),
    textMuted = Color3.fromRGB(130, 140, 160),
    border = Color3.fromRGB(45, 45, 55)
}

local function addRoundedCorners(instance, radius)
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, radius)
    corner.Parent = instance
    return corner
end

local function createCloudIcon(parent, size, position)
    local cloudIcon = Instance.new("Frame")
    cloudIcon.Name = "CloudIcon"
    cloudIcon.Size = UDim2.new(0, size, 0, size)
    cloudIcon.Position = position
    cloudIcon.BackgroundColor3 = colors.accentLight
    cloudIcon.BackgroundTransparency = 0.2
    cloudIcon.BorderSizePixel = 0
    cloudIcon.Parent = parent
    
    local corner1 = Instance.new("UICorner")
    corner1.CornerRadius = UDim.new(0, size * 0.4)
    corner1.Parent = cloudIcon
    
    local cloudPuff1 = Instance.new("Frame")
    cloudPuff1.Name = "CloudPuff1"
    cloudPuff1.Size = UDim2.new(0, size * 0.5, 0, size * 0.5)
    cloudPuff1.Position = UDim2.new(0, -size * 0.1, 0, -size * 0.1)
    cloudPuff1.BackgroundColor3 = colors.accentLight
    cloudPuff1.BackgroundTransparency = 0.2
    cloudPuff1.BorderSizePixel = 0
    cloudPuff1.Parent = cloudIcon
    
    local puffCorner1 = Instance.new("UICorner")
    puffCorner1.CornerRadius = UDim.new(0, size * 0.25)
    puffCorner1.Parent = cloudPuff1
    
    local cloudPuff2 = Instance.new("Frame")
    cloudPuff2.Name = "CloudPuff2"
    cloudPuff2.Size = UDim2.new(0, size * 0.4, 0, size * 0.4)
    cloudPuff2.Position = UDim2.new(0, size * 0.4, 0, -size * 0.15)
    cloudPuff2.BackgroundColor3 = colors.accentLight
    cloudPuff2.BackgroundTransparency = 0.2
    cloudPuff2.BorderSizePixel = 0
    cloudPuff2.Parent = cloudIcon
    
    local puffCorner2 = Instance.new("UICorner")
    puffCorner2.CornerRadius = UDim.new(0, size * 0.2)
    puffCorner2.Parent = cloudPuff2
    
    local cloudPuff3 = Instance.new("Frame")
    cloudPuff3.Name = "CloudPuff3"
    cloudPuff3.Size = UDim2.new(0, size * 0.45, 0, size * 0.45)
    cloudPuff3.Position = UDim2.new(0, size * 0.2, 0, size * 0.2)
    cloudPuff3.BackgroundColor3 = colors.accentLight
    cloudPuff3.BackgroundTransparency = 0.2
    cloudPuff3.BorderSizePixel = 0
    cloudPuff3.Parent = cloudIcon
    
    local puffCorner3 = Instance.new("UICorner")
    puffCorner3.CornerRadius = UDim.new(0, size * 0.225)
    puffCorner3.Parent = cloudPuff3
    
    return cloudIcon
end

local function createModernToggle(parent, name, text, defaultState, description)
    local container = Instance.new("Frame")
    container.Name = name
    container.Size = UDim2.new(1, -20, 0, 48)
    container.Position = UDim2.new(0, 10, 0, 0)
    container.BackgroundColor3 = colors.bgSecondary
    container.BorderSizePixel = 0
    container.Parent = parent
    
    addRoundedCorners(container, 8)
    
    local stroke = Instance.new("UIStroke")
    stroke.Color = colors.border
    stroke.Thickness = 1
    stroke.Transparency = 0.7
    stroke.Parent = container
    
    local contentFrame = Instance.new("Frame")
    contentFrame.Size = UDim2.new(1, -70, 1, 0)
    contentFrame.Position = UDim2.new(0, 12, 0, 0)
    contentFrame.BackgroundTransparency = 1
    contentFrame.Parent = container
    
    local titleLabel = Instance.new("TextLabel")
    titleLabel.Size = UDim2.new(1, 0, 0, 20)
    titleLabel.Position = UDim2.new(0, 0, 0, description and 8 or 14)
    titleLabel.BackgroundTransparency = 1
    titleLabel.Text = text
    titleLabel.Font = Enum.Font.GothamSemibold
    titleLabel.TextSize = 13
    titleLabel.TextColor3 = colors.text
    titleLabel.TextXAlignment = Enum.TextXAlignment.Left
    titleLabel.Parent = contentFrame
    
    if description then
        local descLabel = Instance.new("TextLabel")
        descLabel.Size = UDim2.new(1, 0, 0, 16)
        descLabel.Position = UDim2.new(0, 0, 0, 24)
        descLabel.BackgroundTransparency = 1
        descLabel.Text = description
        descLabel.Font = Enum.Font.Gotham
        descLabel.TextSize = 11
        descLabel.TextColor3 = colors.textSecondary
        descLabel.TextXAlignment = Enum.TextXAlignment.Left
        descLabel.Parent = contentFrame
    end
    
    local switchContainer = Instance.new("TextButton")
    switchContainer.Name = "SwitchContainer"
    switchContainer.Size = UDim2.new(0, 40, 0, 20)
    switchContainer.Position = UDim2.new(1, -52, 0.5, -10)
    switchContainer.BackgroundColor3 = defaultState and colors.success or colors.error
    switchContainer.BackgroundTransparency = 0.2
    switchContainer.BorderSizePixel = 0
    switchContainer.Text = ""
    switchContainer.AutoButtonColor = false
    switchContainer.Parent = container
    
    addRoundedCorners(switchContainer, 10)
    
    local knob = Instance.new("Frame")
    knob.Name = "Knob"
    knob.Size = UDim2.new(0, 16, 0, 16)
    knob.Position = defaultState and UDim2.new(1, -18, 0.5, -8) or UDim2.new(0, 2, 0.5, -8)
    knob.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    knob.BorderSizePixel = 0
    knob.Parent = switchContainer
    
    addRoundedCorners(knob, 8)
    
    return {
        container = container,
        switch = switchContainer,
        knob = knob,
        enabled = defaultState
    }
end

local function createModernButton(parent, name, text, color)
    local button = Instance.new("TextButton")
    button.Name = name
    button.Size = UDim2.new(1, -20, 0, 36)
    button.Position = UDim2.new(0, 10, 0, 0)
    button.BackgroundColor3 = color or colors.accent
    button.BackgroundTransparency = 0.1
    button.BorderSizePixel = 0
    button.Text = ""
    button.AutoButtonColor = false
    button.Parent = parent
    
    addRoundedCorners(button, 8)
    
    local stroke = Instance.new("UIStroke")
    stroke.Color = color or colors.accentLight
    stroke.Thickness = 1
    stroke.Transparency = 0.6
    stroke.Parent = button
    
    local textLabel = Instance.new("TextLabel")
    textLabel.Size = UDim2.new(1, 0, 1, 0)
    textLabel.BackgroundTransparency = 1
    textLabel.Text = text
    textLabel.Font = Enum.Font.GothamSemibold
    textLabel.TextSize = 12
    textLabel.TextColor3 = colors.text
    textLabel.Parent = button
    
    button.MouseEnter:Connect(function()
        TweenService:Create(button, TweenInfo.new(0.2), {
            BackgroundTransparency = 0
        }):Play()
    end)
    
    button.MouseLeave:Connect(function()
        TweenService:Create(button, TweenInfo.new(0.2), {
            BackgroundTransparency = 0.1
        }):Play()
    end)
    
    return button
end

local function createCloudUI()
    local playerGui = localPlayer:WaitForChild("PlayerGui")
    
    local oldGui = playerGui:FindFirstChild("CloudSEMITP")
    if oldGui then oldGui:Destroy() end
    
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "CloudSEMITP"
    screenGui.ResetOnSpawn = false
    screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    screenGui.DisplayOrder = 999
    screenGui.Parent = playerGui
    
    local mainFrame = Instance.new("Frame")
    mainFrame.Name = "MainFrame"
    mainFrame.Size = UDim2.new(0, 260, 0, IS_MOBILE and 280 or 320)
    mainFrame.Position = UDim2.new(1, -280, 0, 60)
    mainFrame.BackgroundColor3 = colors.bg
    mainFrame.BorderSizePixel = 0
    mainFrame.ClipsDescendants = true
    mainFrame.Parent = screenGui
    
    addRoundedCorners(mainFrame, 12)
    
    local border = Instance.new("UIStroke")
    border.Color = colors.accent
    border.Thickness = 1
    border.Transparency = 0.8
    border.Parent = mainFrame
    
    local header = Instance.new("Frame")
    header.Name = "Header"
    header.Size = UDim2.new(1, 0, 0, 48)
    header.BackgroundColor3 = colors.bgSecondary
    header.BorderSizePixel = 0
    header.Parent = mainFrame
    
    addRoundedCorners(header, 12)
    
    createCloudIcon(header, 28, UDim2.new(0, 10, 0.5, -14))
    
    local title = Instance.new("TextLabel")
    title.Name = "Title"
    title.Size = UDim2.new(1, -50, 0, 18)
    title.Position = UDim2.new(0, 45, 0, 8)
    title.BackgroundTransparency = 1
    title.Text = "CLOUD SEMI-TP V1"
    title.Font = Enum.Font.GothamBlack
    title.TextSize = 14
    title.TextColor3 = colors.text
    title.TextXAlignment = Enum.TextXAlignment.Left
    title.Parent = header
    
    local subtitle = Instance.new("TextLabel")
    subtitle.Name = "Subtitle"
    subtitle.Size = UDim2.new(1, -50, 0, 16)
    subtitle.Position = UDim2.new(0, 45, 0, 26)
    subtitle.BackgroundTransparency = 1
    subtitle.Text = "By @matlindo"
    subtitle.Font = Enum.Font.Gotham
    subtitle.TextSize = 10
    subtitle.TextColor3 = colors.textSecondary
    subtitle.TextXAlignment = Enum.TextXAlignment.Left
    subtitle.Parent = header
    
    local statusDot = Instance.new("Frame")
    statusDot.Name = "StatusDot"
    statusDot.Size = UDim2.new(0, 6, 0, 6)
    statusDot.Position = UDim2.new(1, -18, 0.5, -3)
    statusDot.BackgroundColor3 = colors.success
    statusDot.BorderSizePixel = 0
    statusDot.Parent = header
    
    addRoundedCorners(statusDot, 3)
    
    local content = Instance.new("Frame")
    content.Name = "Content"
    content.Size = UDim2.new(1, 0, 1, -48)
    content.Position = UDim2.new(0, 0, 0, 48)
    content.BackgroundTransparency = 1
    content.Parent = mainFrame
    
    local togglesContainer = Instance.new("Frame")
    togglesContainer.Name = "TogglesContainer"
    togglesContainer.Size = UDim2.new(1, 0, 0, IS_MOBILE and 120 or 150)
    togglesContainer.Position = UDim2.new(0, 0, 0, 0)
    togglesContainer.BackgroundTransparency = 1
    togglesContainer.Parent = content
    
    local teleportToggle = createModernToggle(
        togglesContainer,
        "TeleportToggle",
        "Teleport System",
        true,
        "Auto-teleport when stealing"
    )
    teleportToggle.container.Position = UDim2.new(0, 0, 0, 4)
    
    local workToggle = createModernToggle(
        togglesContainer,
        "WorkToggle",
        "Reset Bypass",
        false,
        "A FFlag Bypass"
    )
    workToggle.container.Position = UDim2.new(0, 0, 0, 56)
    
    local brainrotButton = createModernButton(
        content,
        "BrainrotButton",
        "TELEPORT TO BRAINROT",
        colors.accent
    )
    brainrotButton.Position = UDim2.new(0, 0, 0, 110)
    
    local statusBar = Instance.new("Frame")
    statusBar.Name = "StatusBar"
    statusBar.Size = UDim2.new(1, -20, 0, 1)
    statusBar.Position = UDim2.new(0, 10, 1, -50)
    statusBar.BackgroundColor3 = colors.border
    statusBar.BorderSizePixel = 0
    statusBar.Parent = content
    
    local statusText = Instance.new("TextLabel")
    statusText.Name = "StatusText"
    statusText.Size = UDim2.new(0.6, -10, 0, 18)
    statusText.Position = UDim2.new(0, 12, 1, -42)
    statusText.BackgroundTransparency = 1
    statusText.Text = "● System Ready"
    statusText.Font = Enum.Font.Gotham
    statusText.TextSize = 10
    statusText.TextColor3 = colors.success
    statusText.TextXAlignment = Enum.TextXAlignment.Left
    statusText.Parent = content
    
    local discordLabel = Instance.new("TextLabel")
    discordLabel.Name = "DiscordLabel"
    discordLabel.Size = UDim2.new(0.4, -10, 0, 18)
    discordLabel.Position = UDim2.new(0.6, 0, 1, -42)
    discordLabel.BackgroundTransparency = 1
    discordLabel.Text = "discord.gg/4dCs9nmaX"
    discordLabel.Font = Enum.Font.Gotham
    discordLabel.TextSize = 8
    discordLabel.TextColor3 = colors.textMuted
    discordLabel.TextXAlignment = Enum.TextXAlignment.Right
    discordLabel.Parent = content
    
    local cooldownFrame = Instance.new("Frame")
    cooldownFrame.Name = "CooldownFrame"
    cooldownFrame.Size = UDim2.new(1, -20, 0, 26)
    cooldownFrame.Position = UDim2.new(0, 10, 1, -22)
    cooldownFrame.BackgroundColor3 = colors.bgTertiary
    cooldownFrame.BackgroundTransparency = 0.2
    cooldownFrame.BorderSizePixel = 0
    cooldownFrame.Parent = content
    
    addRoundedCorners(cooldownFrame, 6)
    
    local cooldownText = Instance.new("TextLabel")
    cooldownText.Name = "CooldownText"
    cooldownText.Size = UDim2.new(1, -10, 1, 0)
    cooldownText.Position = UDim2.new(0, 5, 0, 0)
    cooldownText.BackgroundTransparency = 1
    cooldownText.Text = "✓ Teleport Ready"
    cooldownText.Font = Enum.Font.GothamMedium
    cooldownText.TextSize = 10
    cooldownText.TextColor3 = colors.success
    cooldownText.TextXAlignment = Enum.TextXAlignment.Left
    cooldownText.Parent = cooldownFrame
    
    mainFrame.Position = UDim2.new(1, -280, 0, -200)
    mainFrame.BackgroundTransparency = 1
    
    TweenService:Create(mainFrame, TweenInfo.new(0.5, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
        Position = UDim2.new(1, -280, 0, 60),
        BackgroundTransparency = 0
    }):Play()
    
    -- Teleport System toggle
    teleportToggle.switch.MouseButton1Click:Connect(function()
        teleportToggle.enabled = not teleportToggle.enabled
        
        TweenService:Create(teleportToggle.switch, TweenInfo.new(0.2), {
            BackgroundColor3 = teleportToggle.enabled and colors.success or colors.error
        }):Play()
        
        TweenService:Create(teleportToggle.knob, TweenInfo.new(0.2), {
            Position = teleportToggle.enabled and UDim2.new(1, -18, 0.5, -8) or UDim2.new(0, 2, 0.5, -8)
        }):Play()
        
        TPSysEnabled = teleportToggle.enabled
        
        if not TPSysEnabled then
            if Marker and Marker.Parent then
                pcall(function() Marker:Destroy() end)
            end
        else
            Marker = CreateMarker()
        end
        
        statusText.Text = teleportToggle.enabled and "● Teleport System Active" or "● Teleport System Disabled"
        statusText.TextColor3 = teleportToggle.enabled and colors.success or colors.textSecondary
    end)
    
    -- Reset Bypass toggle
    workToggle.switch.MouseButton1Click:Connect(function()
        if desyncPermanentlyActivated then
            statusText.Text = "● Already Activated"
            statusText.TextColor3 = colors.warning
            task.wait(1)
            statusText.Text = "● System Ready"
            statusText.TextColor3 = colors.success
            return
        end
        
        workToggle.switch.Active = false
        workToggle.switch.BackgroundTransparency = 0.5
        
        task.spawn(function()
            statusText.Text = "● Activating Bypass..."
            statusText.TextColor3 = colors.warning
            
            task.wait(1.5)
            statusText.Text = "● Almost done..."
            
            if desyncFirstActivation then
                respawn(LocalPlayer)
                desyncFirstActivation = false
                applyPermanentDesync()
            end
            
            task.wait(2)
            
            desyncPermanentlyActivated = true
            
            TweenService:Create(workToggle.switch, TweenInfo.new(0.2), {
                BackgroundColor3 = colors.success
            }):Play()
            
            TweenService:Create(workToggle.knob, TweenInfo.new(0.2), {
                Position = UDim2.new(1, -18, 0.5, -8)
            }):Play()
            
            statusText.Text = "● FFlag Bypass Active"
            statusText.TextColor3 = colors.success
        end)
    end)
    
    -- BOTÃO TELEPORT TO BRAINROT - AGORA EXECUTA A FUNÇÃO DO CÍRCULO
    brainrotButton.MouseButton1Click:Connect(function()
        if tick() - lastTeleportTime < TELEPORT_COOLDOWN then
            statusText.Text = "● On Cooldown"
            statusText.TextColor3 = colors.warning
            task.wait(0.5)
            statusText.Text = "● System Ready"
            statusText.TextColor3 = colors.success
            return
        end
        
        TweenService:Create(brainrotButton, TweenInfo.new(0.1), {
            BackgroundTransparency = 0.2,
            Size = UDim2.new(1, -24, 0, 34),
            Position = UDim2.new(0, 12, 0, brainrotButton.Position.Y.Offset + 1)
        }):Play()
        
        task.wait(0.1)
        
        TweenService:Create(brainrotButton, TweenInfo.new(0.1), {
            BackgroundTransparency = 0.1,
            Size = UDim2.new(1, -20, 0, 36),
            Position = UDim2.new(0, 10, 0, brainrotButton.Position.Y.Offset - 1)
        }):Play()
        
        -- EXECUTA A FUNÇÃO DO CÍRCULO
        executeTP()
        
        statusText.Text = "● Teleported!"
        statusText.TextColor3 = colors.success
        task.wait(1)
        statusText.Text = "● System Ready"
    end)
    
    task.spawn(function()
        while screenGui and screenGui.Parent do
            local remaining = TELEPORT_COOLDOWN - (tick() - lastTeleportTime)
            
            if remaining > 0 then
                cooldownText.Text = string.format("Cooldown: %.1fs", remaining)
                cooldownText.TextColor3 = colors.warning
            else
                cooldownText.Text = "✓ Teleport Ready"
                cooldownText.TextColor3 = colors.success
            end
            
            task.wait(0.1)
        end
    end)
    
    local dragging = false
    local dragStart, startPos
    
    header.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = mainFrame.Position
        end
    end)
    
    UserInputService.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            local delta = input.Position - dragStart
            mainFrame.Position = UDim2.new(
                startPos.X.Scale, 
                startPos.X.Offset + delta.X,
                startPos.Y.Scale, 
                startPos.Y.Offset + delta.Y
            )
        end
    end)
    
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = false
        end
    end)
    
    return screenGui
end

-- ============================================================
-- MOBILE TP BUTTON
-- ============================================================

local function createMobileTPButton()
    if not IS_MOBILE then return nil end
    
    local playerGui = LocalPlayer:WaitForChild("PlayerGui")
    
    local circleGui = Instance.new("ScreenGui")
    circleGui.Name = "MobileTPButton"
    circleGui.ResetOnSpawn = false
    circleGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    circleGui.DisplayOrder = 1000
    circleGui.Parent = playerGui
    
    local circleButton = Instance.new("TextButton")
    circleButton.Name = "TPCircle"
    circleButton.Size = UDim2.new(0, 60, 0, 60)
    circleButton.Position = UDim2.new(0.5, -30, 0.85, -30)
    circleButton.BackgroundColor3 = colors.accent
    circleButton.BackgroundTransparency = 0.1
    circleButton.BorderSizePixel = 0
    circleButton.Text = ""
    circleButton.Parent = circleGui
    
    addRoundedCorners(circleButton, 30)
    
    local icon = Instance.new("ImageLabel")
    icon.Name = "Icon"
    icon.Size = UDim2.new(0, 30, 0, 30)
    icon.Position = UDim2.new(0.5, -15, 0.5, -15)
    icon.BackgroundTransparency = 1
    icon.Image = "rbxassetid://3926305904"
    icon.ImageRectOffset = Vector2.new(964, 324)
    icon.ImageRectSize = Vector2.new(36, 36)
    icon.ImageColor3 = colors.text
    icon.Parent = circleButton
    
    local cooldownOverlay = Instance.new("Frame")
    cooldownOverlay.Name = "CooldownOverlay"
    cooldownOverlay.Size = UDim2.new(1, 0, 1, 0)
    cooldownOverlay.BackgroundColor3 = colors.bg
    cooldownOverlay.BackgroundTransparency = 0.5
    cooldownOverlay.BorderSizePixel = 0
    cooldownOverlay.Visible = false
    cooldownOverlay.Parent = circleButton
    
    addRoundedCorners(cooldownOverlay, 30)
    
    local cooldownText = Instance.new("TextLabel")
    cooldownText.Name = "CooldownText"
    cooldownText.Size = UDim2.new(1, 0, 1, 0)
    cooldownText.BackgroundTransparency = 1
    cooldownText.Text = ""
    cooldownText.Font = Enum.Font.GothamBold
    cooldownText.TextSize = 20
    cooldownText.TextColor3 = colors.text
    cooldownText.ZIndex = 2
    cooldownText.Parent = circleButton
    
    local function updateCooldownDisplay()
        local remaining = TELEPORT_COOLDOWN - (tick() - lastTeleportTime)
        
        if remaining > 0 then
            cooldownOverlay.Visible = true
            cooldownText.Text = tostring(math.ceil(remaining))
            circleButton.BackgroundColor3 = colors.error
        else
            cooldownOverlay.Visible = false
            cooldownText.Text = ""
            circleButton.BackgroundColor3 = colors.accent
        end
    end
    
    task.spawn(function()
        while circleGui and circleGui.Parent do
            updateCooldownDisplay()
            task.wait(0.1)
        end
    end)
    
    circleButton.MouseButton1Click:Connect(function()
        if tick() - lastTeleportTime < TELEPORT_COOLDOWN then
            return
        end
        
        TweenService:Create(circleButton, TweenInfo.new(0.1), {
            Size = UDim2.new(0, 55, 0, 55),
            Position = UDim2.new(0.5, -27.5, 0.85, -27.5)
        }):Play()
        
        task.wait(0.1)
        
        TweenService:Create(circleButton, TweenInfo.new(0.1), {
            Size = UDim2.new(0, 60, 0, 60),
            Position = UDim2.new(0.5, -30, 0.85, -30)
        }):Play()
        
        executeTP()
        updateCooldownDisplay()
    end)
    
    return circleGui
end

-- Initialize UI
task.spawn(function()
    task.wait(0.5)
    createCloudUI()
    
    if IS_MOBILE then
        task.wait(0.5)
        createMobileTPButton()
    end
end)

print("Cloud SEMI-TP V1 loaded for: " .. (IS_MOBILE and "MOBILE" or "PC"))
print("By @matlindo | Discord: discord.gg/4dCs9nmaX")
print("Botão TELEPORT TO BRAINROT agora executa a função do círculo!")
