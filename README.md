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
    for _, part in ipairs(workspace:GetChildren()) do
        if part.Name:find("myskyp") then
            pcall(function() part:Destroy() end)
        end
    end
    local coreGui = game:GetService("CoreGui")
    local existingGui = coreGui:FindFirstChild("CloudSEMITP")
    if existingGui then
        pcall(function() existingGui:Destroy() end)
    end
end
pcall(_G.MySkyCleanup)

-- Services
local Services = {}
local servicePromises = {}
local function getService(serviceName)
    if Services[serviceName] then return Services[serviceName] end
    if not servicePromises[serviceName] then
        servicePromises[serviceName] = task.spawn(function()
            Services[serviceName] = game:GetService(serviceName)
            servicePromises[serviceName] = nil
        end)
    end
    while not Services[serviceName] do task.wait() end
    return Services[serviceName]
end

local Players = getService("Players")
local LocalPlayer = Players.LocalPlayer
local connections = {}
_G.MySkyConnections = connections

print("Script loaded for:", LocalPlayer.Name)

if _G.MyskypInstaSteal then
    pcall(_G.MySkyCleanup)
    task.wait(0.1)
end
_G.MyskypInstaSteal = true

-- Teleport positions
local TP_POSITIONS = {
    BASE1 = {
        INFO_POS = CFrame.new(334.76, 55.334, 99.40),
        TELEPORT_POS = CFrame.new(-352.98, -7.30, 74.3),
    },
    BASE2 = {
        INFO_POS = CFrame.new(334.76, 55.334, 19.17),
        TELEPORT_POS = CFrame.new(-352.98, -7.30, 45.76),
    }
}

local TP_DELAY = 0.15
local TPSysEnabled = true
local lastTeleportTime = 0
local TELEPORT_COOLDOWN = 0.8
local lastMarkerUpdate = 0
local MARKER_UPDATE_INTERVAL = 0.4

local UserInputService = getService("UserInputService")
local TweenService = getService("TweenService")
local IS_MOBILE = UserInputService.TouchEnabled
local IS_PC = not IS_MOBILE

print("Device:", IS_MOBILE and "MOBILE" or "PC")

local function cleanup()
    for _, conn in ipairs(connections) do
        pcall(function() conn:Disconnect() end)
    end
    connections = {}
end

-- TeleportPath (mantida)
local function TeleportPath(targetCFrame)
    if tick() - lastTeleportTime < TELEPORT_COOLDOWN then return end
    lastTeleportTime = tick()
    if not LocalPlayer.Character or not LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then return end
    local hrp = LocalPlayer.Character.HumanoidRootPart
    local distance = (hrp.Position - targetCFrame.Position).Magnitude
    if distance > 1000 then return end
    if distance < 50 then
        hrp.AssemblyLinearVelocity = Vector3.zero
        LocalPlayer.Character:PivotTo(targetCFrame)
        task.wait(TP_DELAY)
        return
    end
    task.spawn(function()
        local PathfindingService = getService("PathfindingService")
        local success, path = pcall(function()
            local path = PathfindingService:CreatePath({ AgentCanJump = true, AgentRadius = 3, AgentHeight = 5 })
            path:ComputeAsync(hrp.Position, targetCFrame.Position)
            return path
        end)
        if not success or not path then return end
        if path.Status == Enum.PathStatus.Success then
            local waypoints = path:GetWaypoints()
            local stepCount = math.min(4, #waypoints)
            for i = 1, stepCount do
                if not LocalPlayer.Character or not LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then break end
                local idx = math.clamp(math.floor((i/stepCount)*#waypoints)+1, 1, #waypoints)
                hrp.AssemblyLinearVelocity = Vector3.zero
                LocalPlayer.Character:PivotTo(CFrame.new(waypoints[idx].Position + Vector3.new(0,2.5,0)))
                task.wait(0.08)
            end
        end
        if LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
            LocalPlayer.Character:PivotTo(targetCFrame)
            task.wait(TP_DELAY)
        end
    end)
end

local function getCurrentBasePosition()
    if not LocalPlayer.Character or not LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
        return TP_POSITIONS.BASE1.INFO_POS
    end
    local hrp = LocalPlayer.Character.HumanoidRootPart
    local pos = hrp.Position
    local d1 = (pos - TP_POSITIONS.BASE1.INFO_POS.Position).Magnitude
    local d2 = (pos - TP_POSITIONS.BASE2.INFO_POS.Position).Magnitude
    return d1 < d2 and TP_POSITIONS.BASE1.INFO_POS or TP_POSITIONS.BASE2.INFO_POS
end

-- Marker
local function CreateMarker()
    local currentBasePos = getCurrentBasePosition()
    local markerPos = currentBasePos * CFrame.new(0, -3.2, 0)
    local part = workspace:FindFirstChild("myskypBest") or Instance.new("Part")
    part.Name = "myskypBest"
    part.Size = Vector3.new(1,1,1)
    part.CFrame = markerPos
    part.Anchored = true
    part.CanCollide = false
    part.Material = Enum.Material.Neon
    part.Color = Color3.fromRGB(0,255,255)
    part.Transparency = 0.3
    part.Parent = workspace
    if not part:FindFirstChild("MyskypBillboard") then
        local gui = Instance.new("BillboardGui")
        gui.Name = "MyskypBillboard"
        gui.AlwaysOnTop = true
        gui.Size = UDim2.new(0,200,0,50)
        gui.ExtentsOffset = Vector3.new(0,3,0)
        gui.Parent = part
        local label = Instance.new("TextLabel")
        label.Name = "Text"
        label.BackgroundTransparency = 1
        label.Size = UDim2.new(1,0,1,0)
        label.Font = Enum.Font.GothamBold
        label.TextSize = 18
        label.TextColor3 = Color3.new(1,1,1)
        label.Text = ""
        label.Parent = gui
    end
    return part
end

local Marker = CreateMarker()

-- Steal detection
local function initEvents()
    local ProximityPromptService = getService("ProximityPromptService")
    local RunService = getService("RunService")
    local promptConn = ProximityPromptService.PromptButtonHoldEnded:Connect(function(prompt, who)
        if who ~= LocalPlayer then return end
        if prompt.Name ~= "Steal" and prompt.ActionText ~= "Steal" and prompt.ObjectText ~= "Steal" then return end
        if not TPSysEnabled then return end
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
        else
            hrp.CFrame = TP_POSITIONS.BASE2.TELEPORT_POS
        end
    end)
    table.insert(connections, promptConn)
    local heartbeatConn = RunService.Heartbeat:Connect(function(dt)
        lastMarkerUpdate = lastMarkerUpdate + dt
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
            Marker.Color = dist < 7 and Color3.fromRGB(0,255,100) or Color3.fromRGB(255,50,50)
        end
    end)
    table.insert(connections, heartbeatConn)
end
task.spawn(initEvents)

-- Anti-kick
local KEYWORD = "You stole"
local KICK_MESSAGE = "EZZ"
local function hasKeyword(t) return type(t)=="string" and t:lower():find(KEYWORD,1,true)~=nil end
local function watchObject(obj)
    if not (obj:IsA("TextLabel") or obj:IsA("TextButton") or obj:IsA("TextBox")) then return end
    if obj.Visible and obj.Text and hasKeyword(obj.Text) then
        task.spawn(function() pcall(function() LocalPlayer:Kick(KICK_MESSAGE) end) end)
        return
    end
    if obj.Visible then
        local conn = obj:GetPropertyChangedSignal("Text"):Connect(function()
            if hasKeyword(obj.Text) then
                task.spawn(function() pcall(function() LocalPlayer:Kick(KICK_MESSAGE) end) end)
            end
        end)
        table.insert(connections, conn)
    end
end

local lastScan = 0
local SCAN_COOLDOWN = 0.5
local function scanDescendants(parent)
    if tick() - lastScan < SCAN_COOLDOWN then return end
    lastScan = tick()
    task.spawn(function()
        local objs = {}
        for _, v in ipairs(parent:GetDescendants()) do
            if v:IsA("TextLabel") or v:IsA("TextButton") or v:IsA("TextBox") then
                if v.Visible and v.Text then table.insert(objs, v) end
            end
        end
        for i = 1, #objs, 15 do
            for j = i, math.min(i+14, #objs) do watchObject(objs[j]) end
            task.wait(0.05)
        end
    end)
end

local debounce = false
local function debouncedWatch(o)
    if not debounce then debounce = true; watchObject(o); debounce = false end
end

task.spawn(function()
    local pg = LocalPlayer:WaitForChild("PlayerGui")
    task.wait(2)
    scanDescendants(pg)
    local conn1 = pg.ChildAdded:Connect(function(g)
        task.wait(0.1)
        if g:IsA("ScreenGui") then scanDescendants(g) else debouncedWatch(g) end
    end)
    table.insert(connections, conn1)
    local conn2 = pg.DescendantAdded:Connect(debouncedWatch)
    table.insert(connections, conn2)
end)

task.spawn(function()
    local conn = Players.PlayerRemoving:Connect(function(p)
        if p == LocalPlayer then cleanup(); _G.MyskypInstaSteal = false; pcall(_G.MySkyCleanup) end
    end)
    table.insert(connections, conn)
end)

-- FFlags
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
local desyncFirst = true
local desyncPerm = false

local function applyFFlags(t) for k,v in pairs(t) do pcall(function() setfflag(tostring(k), tostring(v)) end) end end
local function respawn(plr)
    local c = plr.Character
    if c then
        local h = c:FindFirstChildOfClass("Humanoid")
        if h then h:ChangeState(Enum.HumanoidStateType.Dead) end
        c:ClearAllChildren()
        local nc = Instance.new("Model")
        nc.Parent = workspace
        plr.Character = nc
        task.wait()
        plr.Character = c
        nc:Destroy()
    end
end
local function applyPermDesync()
    applyFFlags(FFlags)
    if desyncFirst then respawn(LocalPlayer); desyncFirst = false end
    desyncPerm = true
end

-- Execute TP (função do círculo)
local function executeTP()
    if tick() - lastTeleportTime < TELEPORT_COOLDOWN then return false end
    lastTeleportTime = tick()
    local char = LocalPlayer.Character
    if not char then char = LocalPlayer.CharacterAdded:Wait() end
    local hum = char:WaitForChild("Humanoid")
    local hrp = char:WaitForChild("HumanoidRootPart")
    local carpet = char:FindFirstChild("Flying Carpet") or LocalPlayer.Backpack:FindFirstChild("Flying Carpet")
    if not carpet then return false end
    local isBase1 = getCurrentBasePosition() == TP_POSITIONS.BASE1.INFO_POS
    task.spawn(function()
        hum:EquipTool(carpet)
        if isBase1 then
            hrp.CFrame = CFrame.new(-351.49, -6.65, 113.72); task.wait(0.15)
            hrp.CFrame = CFrame.new(-378.14, -6.00, 26.43); task.wait(0.15)
            hrp.CFrame = CFrame.new(-334.80, -5.04, 18.90)
        else
            hrp.CFrame = CFrame.new(-352.54, -6.83, 6.66); task.wait(0.15)
            hrp.CFrame = CFrame.new(-372.90, -6.20, 102.00); task.wait(0.15)
            hrp.CFrame = CFrame.new(-335.08, -5.10, 101.40)
        end
    end)
    return true
end

-- ================== UI MODERNA E COMPACTA ==================
local localPlayer = Players.LocalPlayer
local colors = {
    bg = Color3.fromRGB(10,10,15),
    bg2 = Color3.fromRGB(18,18,25),
    bg3 = Color3.fromRGB(25,25,35),
    accent = Color3.fromRGB(100,180,255),
    accentLight = Color3.fromRGB(150,210,255),
    success = Color3.fromRGB(80,200,150),
    error = Color3.fromRGB(255,100,100),
    warning = Color3.fromRGB(255,180,70),
    text = Color3.fromRGB(255,255,255),
    textSec = Color3.fromRGB(180,190,210),
    border = Color3.fromRGB(45,45,55)
}
local function round(inst, r) local c = Instance.new("UICorner"); c.CornerRadius = UDim.new(0,r); c.Parent = inst; return c end

-- Ícone de nuvem (imagem)
local function createCloudIcon(parent, size, pos)
    local img = Instance.new("ImageLabel")
    img.Name = "CloudIcon"
    img.Size = UDim2.new(0,size,0,size)
    img.Position = pos
    img.BackgroundTransparency = 1
    img.Image = "rbxassetid://3570695787"
    img.ImageColor3 = colors.accentLight
    img.ImageTransparency = 0.1
    img.ScaleType = Enum.ScaleType.Fit
    img.Parent = parent
    return img
end

-- Toggle padrão (ambos iguais)
local function createToggle(parent, name, text, default, desc)
    local frame = Instance.new("Frame")
    frame.Name = name
    frame.Size = UDim2.new(1, -20, 0, 44)
    frame.Position = UDim2.new(0,10,0,0)
    frame.BackgroundColor3 = colors.bg2
    frame.BorderSizePixel = 0
    frame.Parent = parent
    round(frame, 8)
    local stroke = Instance.new("UIStroke")
    stroke.Color = colors.border
    stroke.Thickness = 1
    stroke.Transparency = 0.7
    stroke.Parent = frame

    local content = Instance.new("Frame")
    content.Size = UDim2.new(1, -70, 1, 0)
    content.Position = UDim2.new(0,12,0,0)
    content.BackgroundTransparency = 1
    content.Parent = frame

    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1,0,0,20)
    title.Position = UDim2.new(0,0,0, desc and 6 or 12)
    title.BackgroundTransparency = 1
    title.Text = text
    title.Font = Enum.Font.GothamSemibold
    title.TextSize = 13
    title.TextColor3 = colors.text
    title.TextXAlignment = Enum.TextXAlignment.Left
    title.Parent = content

    if desc then
        local dsc = Instance.new("TextLabel")
        dsc.Size = UDim2.new(1,0,0,14)
        dsc.Position = UDim2.new(0,0,0,22)
        dsc.BackgroundTransparency = 1
        dsc.Text = desc
        dsc.Font = Enum.Font.Gotham
        dsc.TextSize = 10
        dsc.TextColor3 = colors.textSec
        dsc.TextXAlignment = Enum.TextXAlignment.Left
        dsc.Parent = content
    end

    local switch = Instance.new("TextButton")
    switch.Name = "Switch"
    switch.Size = UDim2.new(0,38,0,18)
    switch.Position = UDim2.new(1, -48, 0.5, -9)
    switch.BackgroundColor3 = default and colors.success or colors.error
    switch.BackgroundTransparency = 0.2
    switch.BorderSizePixel = 0
    switch.Text = ""
    switch.AutoButtonColor = false
    switch.Parent = frame
    round(switch, 9)

    local knob = Instance.new("Frame")
    knob.Name = "Knob"
    knob.Size = UDim2.new(0,14,0,14)
    knob.Position = default and UDim2.new(1, -16, 0.5, -7) or UDim2.new(0,2,0.5,-7)
    knob.BackgroundColor3 = Color3.new(1,1,1)
    knob.BorderSizePixel = 0
    knob.Parent = switch
    round(knob, 7)

    return { frame = frame, switch = switch, knob = knob, enabled = default }
end

-- Botão moderno (para Teleport To Brainrot)
local function createButton(parent, name, text, color)
    local btn = Instance.new("TextButton")
    btn.Name = name
    btn.Size = UDim2.new(1, -20, 0, 34)
    btn.Position = UDim2.new(0,10,0,0)
    btn.BackgroundColor3 = color or colors.accent
    btn.BackgroundTransparency = 0.1
    btn.BorderSizePixel = 0
    btn.Text = ""
    btn.AutoButtonColor = false
    btn.Parent = parent
    round(btn, 8)
    local stroke = Instance.new("UIStroke")
    stroke.Color = color or colors.accentLight
    stroke.Thickness = 1
    stroke.Transparency = 0.6
    stroke.Parent = btn
    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1,0,1,0)
    label.BackgroundTransparency = 1
    label.Text = text
    label.Font = Enum.Font.GothamSemibold
    label.TextSize = 12
    label.TextColor3 = colors.text
    label.Parent = btn
    btn.MouseEnter:Connect(function() TweenService:Create(btn, TweenInfo.new(0.2), {BackgroundTransparency = 0}):Play() end)
    btn.MouseLeave:Connect(function() TweenService:Create(btn, TweenInfo.new(0.2), {BackgroundTransparency = 0.1}):Play() end)
    return btn
end

-- UI principal (tamanho reduzido)
local function buildUI()
    local pg = localPlayer:WaitForChild("PlayerGui")
    local old = pg:FindFirstChild("CloudSEMITP")
    if old then old:Destroy() end
    local gui = Instance.new("ScreenGui")
    gui.Name = "CloudSEMITP"
    gui.ResetOnSpawn = false
    gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    gui.DisplayOrder = 999
    gui.Parent = pg

    local main = Instance.new("Frame")
    main.Name = "Main"
    main.Size = UDim2.new(0, 240, 0, 290) -- mais compacto
    main.Position = UDim2.new(1, -260, 0, 60)
    main.BackgroundColor3 = colors.bg
    main.BorderSizePixel = 0
    main.ClipsDescendants = true
    main.Parent = gui
    round(main, 12)

    local border = Instance.new("UIStroke")
    border.Color = colors.accent
    border.Thickness = 1
    border.Transparency = 0.8
    border.Parent = main

    -- Header
    local hdr = Instance.new("Frame")
    hdr.Name = "Header"
    hdr.Size = UDim2.new(1,0,0,44)
    hdr.BackgroundColor3 = colors.bg2
    hdr.BorderSizePixel = 0
    hdr.Parent = main
    round(hdr, 12)

    createCloudIcon(hdr, 24, UDim2.new(0,8,0.5,-12))

    local tit = Instance.new("TextLabel")
    tit.Name = "Title"
    tit.Size = UDim2.new(1, -40, 0, 18)
    tit.Position = UDim2.new(0,36,0,6)
    tit.BackgroundTransparency = 1
    tit.Text = "CLOUD SEMI-TP V1"
    tit.Font = Enum.Font.GothamBlack
    tit.TextSize = 13
    tit.TextColor3 = colors.text
    tit.TextXAlignment = Enum.TextXAlignment.Left
    tit.Parent = hdr

    local sub = Instance.new("TextLabel")
    sub.Name = "Subtitle"
    sub.Size = UDim2.new(1, -40, 0, 14)
    sub.Position = UDim2.new(0,36,0,24)
    sub.BackgroundTransparency = 1
    sub.Text = "By @matlindo"
    sub.Font = Enum.Font.Gotham
    sub.TextSize = 9
    sub.TextColor3 = colors.textSec
    sub.TextXAlignment = Enum.TextXAlignment.Left
    sub.Parent = hdr

    local dot = Instance.new("Frame")
    dot.Size = UDim2.new(0,5,0,5)
    dot.Position = UDim2.new(1, -16, 0.5, -2.5)
    dot.BackgroundColor3 = colors.success
    dot.BorderSizePixel = 0
    dot.Parent = hdr
    round(dot, 2.5)

    -- Conteúdo
    local content = Instance.new("Frame")
    content.Name = "Content"
    content.Size = UDim2.new(1,0,1,-44)
    content.Position = UDim2.new(0,0,0,44)
    content.BackgroundTransparency = 1
    content.Parent = main

    -- Toggles
    local togFrame = Instance.new("Frame")
    togFrame.Name = "Toggles"
    togFrame.Size = UDim2.new(1,0,0,100)
    togFrame.Position = UDim2.new(0,0,0,4)
    togFrame.BackgroundTransparency = 1
    togFrame.Parent = content

    local tpToggle = createToggle(togFrame, "TPToggle", "Teleport System", true, "Auto-teleport when stealing")
    tpToggle.frame.Position = UDim2.new(0,0,0,0)

    local resetToggle = createToggle(togFrame, "ResetToggle", "Reset Bypass", false, "A Reset+FFlag Bypass") -- descrição alterada
    resetToggle.frame.Position = UDim2.new(0,0,0,48)

    -- Botão Teleport To Brainrot (agora com a função do círculo)
    local brainBtn = createButton(content, "BrainBtn", "TELEPORT TO BRAINROT", colors.accent)
    brainBtn.Position = UDim2.new(0,0,0,112) -- ajustado

    -- Status bar
    local statBar = Instance.new("Frame")
    statBar.Size = UDim2.new(1, -20, 0, 1)
    statBar.Position = UDim2.new(0,10,1,-48)
    statBar.BackgroundColor3 = colors.border
    statBar.BorderSizePixel = 0
    statBar.Parent = content

    local statTxt = Instance.new("TextLabel")
    statTxt.Name = "Status"
    statTxt.Size = UDim2.new(0.6, -10, 0, 16)
    statTxt.Position = UDim2.new(0,12,1,-42)
    statTxt.BackgroundTransparency = 1
    statTxt.Text = "● System Ready"
    statTxt.Font = Enum.Font.Gotham
    statTxt.TextSize = 9
    statTxt.TextColor3 = colors.success
    statTxt.TextXAlignment = Enum.TextXAlignment.Left
    statTxt.Parent = content

    local disc = Instance.new("TextLabel")
    disc.Name = "Discord"
    disc.Size = UDim2.new(0.4, -10, 0, 16)
    disc.Position = UDim2.new(0.6,0,1,-42)
    disc.BackgroundTransparency = 1
    disc.Text = "discord.gg/4dCs9nmaX"
    disc.Font = Enum.Font.Gotham
    disc.TextSize = 7
    disc.TextColor3 = colors.textSec
    disc.TextXAlignment = Enum.TextXAlignment.Right
    disc.Parent = content

    -- Cooldown
    local cold = Instance.new("Frame")
    cold.Name = "Cooldown"
    cold.Size = UDim2.new(1, -20, 0, 24)
    cold.Position = UDim2.new(0,10,1,-22)
    cold.BackgroundColor3 = colors.bg3
    cold.BackgroundTransparency = 0.2
    cold.BorderSizePixel = 0
    cold.Parent = content
    round(cold, 5)

    local coldTxt = Instance.new("TextLabel")
    coldTxt.Name = "Text"
    coldTxt.Size = UDim2.new(1, -8, 1, 0)
    coldTxt.Position = UDim2.new(0,4,0,0)
    coldTxt.BackgroundTransparency = 1
    coldTxt.Text = "✓ Teleport Ready"
    coldTxt.Font = Enum.Font.GothamMedium
    coldTxt.TextSize = 9
    coldTxt.TextColor3 = colors.success
    coldTxt.TextXAlignment = Enum.TextXAlignment.Left
    coldTxt.Parent = cold

    -- Animação de entrada
    main.Position = UDim2.new(1, -260, 0, -200)
    main.BackgroundTransparency = 1
    TweenService:Create(main, TweenInfo.new(0.5, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
        Position = UDim2.new(1, -260, 0, 60),
        BackgroundTransparency = 0
    }):Play()

    -- Funcionalidade dos toggles
    tpToggle.switch.MouseButton1Click:Connect(function()
        tpToggle.enabled = not tpToggle.enabled
        TweenService:Create(tpToggle.switch, TweenInfo.new(0.2), {BackgroundColor3 = tpToggle.enabled and colors.success or colors.error}):Play()
        TweenService:Create(tpToggle.knob, TweenInfo.new(0.2), {Position = tpToggle.enabled and UDim2.new(1, -16, 0.5, -7) or UDim2.new(0,2,0.5,-7)}):Play()
        TPSysEnabled = tpToggle.enabled
        if not TPSysEnabled then
            if Marker and Marker.Parent then pcall(function() Marker:Destroy() end) end
        else
            Marker = CreateMarker()
        end
        statTxt.Text = tpToggle.enabled and "● Teleport System Active" or "● Teleport System Disabled"
        statTxt.TextColor3 = tpToggle.enabled and colors.success or colors.textSec
    end)

    resetToggle.switch.MouseButton1Click:Connect(function()
        if desyncPerm then
            statTxt.Text = "● Already Activated"
            statTxt.TextColor3 = colors.warning
            task.wait(1)
            statTxt.Text = "● System Ready"
            statTxt.TextColor3 = colors.success
            return
        end
        resetToggle.switch.Active = false
        resetToggle.switch.BackgroundTransparency = 0.5
        task.spawn(function()
            statTxt.Text = "● Activating Bypass..."
            statTxt.TextColor3 = colors.warning
            task.wait(1.5)
            statTxt.Text = "● Almost done..."
            if desyncFirst then
                respawn(LocalPlayer)
                desyncFirst = false
                applyPermDesync()
            end
            task.wait(2)
            desyncPerm = true
            TweenService:Create(resetToggle.switch, TweenInfo.new(0.2), {BackgroundColor3 = colors.success}):Play()
            TweenService:Create(resetToggle.knob, TweenInfo.new(0.2), {Position = UDim2.new(1, -16, 0.5, -7)}):Play()
            statTxt.Text = "● Reset+FFlag Bypass Active"
            statTxt.TextColor3 = colors.success
        end)
    end)

    -- Botão Teleport To Brainrot (função do círculo)
    brainBtn.MouseButton1Click:Connect(function()
        if tick() - lastTeleportTime < TELEPORT_COOLDOWN then
            statTxt.Text = "● On Cooldown"
            statTxt.TextColor3 = colors.warning
            task.wait(0.5)
            statTxt.Text = "● System Ready"
            statTxt.TextColor3 = colors.success
            return
        end
        -- feedback visual
        TweenService:Create(brainBtn, TweenInfo.new(0.1), {
            BackgroundTransparency = 0.2,
            Size = UDim2.new(1, -24, 0, 32),
            Position = UDim2.new(0,12,0, brainBtn.Position.Y.Offset+1)
        }):Play()
        task.wait(0.1)
        TweenService:Create(brainBtn, TweenInfo.new(0.1), {
            BackgroundTransparency = 0.1,
            Size = UDim2.new(1, -20, 0, 34),
            Position = UDim2.new(0,10,0, brainBtn.Position.Y.Offset-1)
        }):Play()
        executeTP()
        statTxt.Text = "● Teleported!"
        statTxt.TextColor3 = colors.success
        task.wait(1)
        statTxt.Text = "● System Ready"
    end)

    -- Atualização do cooldown
    task.spawn(function()
        while gui and gui.Parent do
            local rem = TELEPORT_COOLDOWN - (tick() - lastTeleportTime)
            if rem > 0 then
                coldTxt.Text = string.format("Cooldown: %.1fs", rem)
                coldTxt.TextColor3 = colors.warning
            else
                coldTxt.Text = "✓ Teleport Ready"
                coldTxt.TextColor3 = colors.success
            end
            task.wait(0.1)
        end
    end)

    -- Arrastar
    local drag = false; local ds, sp
    hdr.InputBegan:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
            drag = true; ds = i.Position; sp = main.Position
        end
    end)
    UserInputService.InputChanged:Connect(function(i)
        if drag and (i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch) then
            local delta = i.Position - ds
            main.Position = UDim2.new(sp.X.Scale, sp.X.Offset + delta.X, sp.Y.Scale, sp.Y.Offset + delta.Y)
        end
    end)
    UserInputService.InputEnded:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then drag = false end
    end)
end

-- Iniciar UI (sem botão mobile)
task.spawn(function()
    task.wait(0.5)
    buildUI()
end)

print("Cloud SEMI-TP V1 carregado (versão compacta, sem marcadores no mapa)")
print("By @matlindo | Discord: discord.gg/4dCs9nmaX")
print("Botão TELEPORT TO BRAINROT faz a função do círculo")
