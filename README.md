
--========================================================
-- BLACKHAT HUB | SMART AUTO - OPTIMIZED & FIXED AVATAR
-- Author: blackhat
--========================================================

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local CoreGui = game:GetService("CoreGui")

local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")

-- Güvenli Events ve Remotes Kontrolü
local Events = ReplicatedStorage:WaitForChild("Events", 10)
if not Events then
    warn("[Blackhat Hub] Events klasörü bulunamadı!")
    return
end

local ModeVote = Events:WaitForChild("ModeVote", 5)
local ReadyEvent = Events:WaitForChild("Ready", 5)
local PurchaseItem = ReplicatedStorage:WaitForChild("PurchaseItem", 5)

local SpawnPartner = Events:FindFirstChild("SpawnPartner")
local UpgradeEvent = Events:FindFirstChild("UpgradeEvent")

local CharacterRemotes = ReplicatedStorage:FindFirstChild("CharacterRemotes")
local ProtoClickEvent = CharacterRemotes and CharacterRemotes:FindFirstChild("ProtoBV1") and CharacterRemotes.ProtoBV1:FindFirstChild("Click")

--========================================================
-- ASSETS & CONFIG
--========================================================

local ICON_ID = "rbxthumb://type=Asset&id=103525855196551&w=150&h=150"

local Modes = {
    "Easy", "Normal", "Hard", "Infinity", "Strike Back", "Boss Rush", "Shattered Bonds"
}

local ModeIndex = 6
local SelectedMode = Modes[ModeIndex]

local AutoMode = false
local AutoReady = false
local AutoStartLoop = false

local AutoBuyTransform = false
local TransformTargets = {"TSM", "TTV"}
local TransformTargetIndex = 1
local SelectedTransformTarget = TransformTargets[TransformTargetIndex]

local AutoTPFreeze = false
local AutoTSMBattleArmor = false
local AutoTTVCinema = false

local AutoUpTechFullAll = false
local AutoAntiAFK = false
local AutoSpawn = false
local AutoReset = false
local AutoESP = false

-- VOID & POS STATE
local VoidActive = false
local VoidDepth = 50
local preVoidCFrame = nil

local savedPositions = {}
local positionCounter = 0

-- AUTO FARM & ANTI-VOID STATE
local UndergroundFarm = false
local UndergroundDistance = 6
local AutoPunch = false
local AntiVoid = true
local VoidYThreshold = -100
local TargetNames = {"bathub", "bathtub"}
local lastSafeSurfaceCFrame = nil

local VoteDelay = 2
local VoteCooldown = 8
local ReadyCooldown = 8

local PostTransformTweenDelay = 5
local TSMBattleArmorRetryDelay = 2
local TTVCinemaDelay = 1

local StartLoopDelay = 5
local SpawnDelay = 1.5

local TeleportTargets = {
    TSM = {
        Position = Vector3.new(-7202.05810546875, 2407.994384765625, 63.98310089111328),
        Rotation = CFrame.new()
    },
    TTV = {
        Position = Vector3.new(-7199.883789, -230.049576, 61.849308),
        Rotation = CFrame.Angles(math.rad(90), 0, 0)
    }
}

local function GetSelectedTeleportTarget()
    return TeleportTargets[SelectedTransformTarget] or TeleportTargets.TSM
end

local function GetSelectedTeleportPosition()
    return GetSelectedTeleportTarget().Position
end

local function GetSelectedTeleportCFrame()
    local target = GetSelectedTeleportTarget()
    return CFrame.new(target.Position) * target.Rotation
end

--========================================================
-- STATE
--========================================================

local State = {
    Cycle = 0,
    LastVote = 0,
    LastReady = 0,
    LastStart = 0,
    SequenceRunning = false,
    SetupRunning = false,
    ReadyDoneThisCycle = false,
    LastBuy = 0,
    LastTransform = 0,
    LastBattleArmorUpgrade = 0,
    LastCinemaUpgrade = 0,
    BuyTransformPurchasedThisCycle = false,
    BuyTransformDoneThisCycle = false,
    TransformInProgress = false,
    TransformCharacterSwap = false,
    TransformCompletedThisCycle = false,
    ResetDeadline = nil,
    Resetting = false,
    DeathConnection = nil,
    FreezeActive = false,
    NoclipActive = false,
    NoclipParts = {},
    FreezeSnapshot = nil,
    FreezeHeartbeat = nil
}

local UpdateStartLoopUI = nil
local CloseModePicker = function() end
local CloseTransformPicker = function() end

--========================================================
-- CHARACTER HELPERS
--========================================================

local function GetCharacter()
    local character = LocalPlayer.Character
    if not character then return nil, nil, nil end
    return character,
        character:FindFirstChild("HumanoidRootPart") or character:FindFirstChild("Torso"),
        character:FindFirstChildOfClass("Humanoid")
end

local function DisableNoclip()
    State.NoclipActive = false
    local character = LocalPlayer.Character
    if character then
        for part, oldCanCollide in pairs(State.NoclipParts) do
            if part and part.Parent then
                pcall(function() part.CanCollide = oldCanCollide end)
            end
        end
    end
    State.NoclipParts = {}
end

local function EnableNoclip()
    local character = LocalPlayer.Character
    if not character then return false end
    State.NoclipParts = {}
    State.NoclipActive = true

    for _, obj in ipairs(character:GetChildren()) do
        if obj:IsA("BasePart") then
            State.NoclipParts[obj] = obj.CanCollide
            pcall(function() obj.CanCollide = false end)
        end
    end
    return true
end

local function Unfreeze()
    State.FreezeActive = false
    if State.FreezeHeartbeat then
        pcall(function() State.FreezeHeartbeat:Disconnect() end)
        State.FreezeHeartbeat = nil
    end

    local _, hrp, humanoid = GetCharacter()
    if hrp then pcall(function() hrp.Anchored = false end) end

    if humanoid and State.FreezeSnapshot then
        pcall(function()
            humanoid.WalkSpeed = State.FreezeSnapshot.WalkSpeed
            humanoid.JumpPower = State.FreezeSnapshot.JumpPower
            humanoid.AutoRotate = State.FreezeSnapshot.AutoRotate
            humanoid.PlatformStand = State.FreezeSnapshot.PlatformStand
        end)
    end
    State.FreezeSnapshot = nil
    DisableNoclip()
end

local function NewCycle(reason)
    Unfreeze()
    State.Cycle += 1
    State.SequenceRunning = false
    State.SetupRunning = false
    State.ReadyDoneThisCycle = false
    State.LastReady = 0
    State.LastVote = 0
    State.LastBuy = 0
    State.LastTransform = 0
    State.LastBattleArmorUpgrade = 0
    State.LastCinemaUpgrade = 0
    State.BuyTransformPurchasedThisCycle = false
    State.BuyTransformDoneThisCycle = false
    State.TransformInProgress = false
    State.TransformCharacterSwap = false
    State.TransformCompletedThisCycle = false
end

--========================================================
-- MODE VOTE & READY
--========================================================

local function VoteMode(force)
    local now = os.clock()
    if not force and now - State.LastVote < VoteCooldown then return false end
    if ModeVote then pcall(function() ModeVote:FireServer(SelectedMode) end) end
    State.LastVote = now
    return true
end

local function SendReady(force)
    if State.ReadyDoneThisCycle and not force then return false end
    local now = os.clock()
    if not force and now - State.LastReady < ReadyCooldown then return false end

    local ok = pcall(function() if ReadyEvent then ReadyEvent:FireServer() end end)
    if ok then
        State.LastReady = now
        State.ReadyDoneThisCycle = true
    end
    return ok
end

--========================================================
-- BUY & TRANSFORM
--========================================================

local PLAYABLE_PROTOCOL_NAME = "Playable Protocol"

local function HasPlayableProtocol()
    local backpack = LocalPlayer:FindFirstChildOfClass("Backpack")
    if backpack and backpack:FindFirstChild(PLAYABLE_PROTOCOL_NAME) then return true end
    local character = LocalPlayer.Character
    if character and character:FindFirstChild(PLAYABLE_PROTOCOL_NAME) then return true end
    return false
end

local function WaitForPlayableProtocol(timeout)
    timeout = timeout or 8
    local deadline = os.clock() + timeout
    repeat
        if HasPlayableProtocol() then return true end
        task.wait(0.2)
    until os.clock() >= deadline or State.TransformCompletedThisCycle
    return HasPlayableProtocol()
end

local function BuyProtocol(force)
    if State.TransformCompletedThisCycle or State.BuyTransformDoneThisCycle then return true, true end
    if HasPlayableProtocol() then return true, true end

    local now = os.clock()
    if not force and now - State.LastBuy < 2 then return false, false end
    State.LastBuy = now

    if not PurchaseItem then return true, false end
    local ok, result = pcall(function()
        return PurchaseItem:InvokeServer("Player", PLAYABLE_PROTOCOL_NAME)
    end)
    if not ok or result == false then return true, false end

    return true, WaitForPlayableProtocol(8)
end

local function GetPlayableProtocolTool()
    local character = LocalPlayer.Character
    if character then
        local tool = character:FindFirstChild(PLAYABLE_PROTOCOL_NAME)
        if tool and tool:IsA("Tool") then return tool end
    end
    local backpack = LocalPlayer:FindFirstChildOfClass("Backpack")
    if backpack then
        local tool = backpack:FindFirstChild(PLAYABLE_PROTOCOL_NAME)
        if tool and tool:IsA("Tool") then return tool end
    end
    return nil
end

--========================================================
-- TP & FREEZE
--========================================================

local function TPFreeze()
    local character, hrp, humanoid = GetCharacter()
    if not character or not hrp or not humanoid or humanoid.Health <= 0 then return false end

    pcall(function()
        Unfreeze()
        State.FreezeSnapshot = {
            WalkSpeed = humanoid.WalkSpeed,
            JumpPower = humanoid.JumpPower,
            AutoRotate = humanoid.AutoRotate,
            PlatformStand = humanoid.PlatformStand
        }
        EnableNoclip()
        humanoid.AutoRotate = false
        humanoid.PlatformStand = false

        local targetPivot = GetSelectedTeleportCFrame()
        character:PivotTo(targetPivot)
        hrp.CFrame = targetPivot
        hrp.AssemblyLinearVelocity = Vector3.zero
        hrp.AssemblyAngularVelocity = Vector3.zero

        humanoid.WalkSpeed = 0
        humanoid.JumpPower = 0
        humanoid.AutoRotate = false
        humanoid.PlatformStand = true

        State.FreezeActive = true
        State.FreezeHeartbeat = RunService.Heartbeat:Connect(function()
            if not State.FreezeActive then return end
            local cChar, cHrp, cHum = GetCharacter()
            if cChar == character and cHrp and cHum and cHum.Health > 0 then
                pcall(function()
                    cChar:PivotTo(targetPivot)
                    cHrp.AssemblyLinearVelocity = Vector3.zero
                    cHrp.AssemblyAngularVelocity = Vector3.zero
                end)
            end
        end)
    end)
    return true
end

--========================================================
-- LOOPS (NOCLIP / VOID HOVER / ANTI-VOID)
--========================================================

RunService.Stepped:Connect(function()
    if VoidActive or UndergroundFarm then
        local char = LocalPlayer.Character
        if char then
            for _, part in ipairs(char:GetChildren()) do
                if part:IsA("BasePart") then
                    part.CanCollide = false
                end
            end
        end
    end
end)

RunService.Heartbeat:Connect(function()
    if VoidActive then
        local _, hrp = GetCharacter()
        if hrp and preVoidCFrame then
            hrp.AssemblyLinearVelocity = Vector3.zero
            hrp.AssemblyAngularVelocity = Vector3.zero
            hrp.CFrame = CFrame.new(hrp.Position.X, preVoidCFrame.Position.Y - VoidDepth, hrp.Position.Z)
        end
    end
end)

task.spawn(function()
    while task.wait(0.5) do
        if not UndergroundFarm and not VoidActive then
            local character = LocalPlayer.Character
            if character and character:FindFirstChild("HumanoidRootPart") then
                local hrp = character.HumanoidRootPart
                local ray = workspace:Raycast(hrp.Position, Vector3.new(0, -20, 0))
                if ray then lastSafeSurfaceCFrame = hrp.CFrame end
            end
        end
    end
end)

task.spawn(function()
    while task.wait(0.3) do
        if AntiVoid and not UndergroundFarm and not VoidActive then
            pcall(function()
                local character = LocalPlayer.Character
                if character and character:FindFirstChild("HumanoidRootPart") then
                    local hrp = character.HumanoidRootPart
                    if hrp.Position.Y < VoidYThreshold then
                        hrp.AssemblyLinearVelocity = Vector3.zero
                        hrp.CFrame = lastSafeSurfaceCFrame or CFrame.new(hrp.Position.X, 50, hrp.Position.Z)
                    end
                end
            end)
        end
    end
end)

--========================================================
-- AUTO FARM
--========================================================

local function getClosestTarget()
    local aiFolder = workspace:FindFirstChild("AI") or workspace
    local character = LocalPlayer.Character
    if not character or not character:FindFirstChild("HumanoidRootPart") then return nil end
    
    local myPos = character.HumanoidRootPart.Position
    local closestTarget = nil
    local shortestDistance = math.huge

    local children = aiFolder:GetChildren()
    for i = 1, #children do
        local obj = children[i]
        local isValid = false
        local lowerName = string.lower(obj.Name)
        
        for j = 1, #TargetNames do
            if string.find(lowerName, TargetNames[j]) then
                isValid = true
                break
            end
        end

        if isValid then
            local humanoid = obj:FindFirstChildOfClass("Humanoid")
            local hrp = obj:FindFirstChild("HumanoidRootPart") or obj.PrimaryPart

            if humanoid and humanoid.Health > 0 and hrp then
                local dist = (hrp.Position - myPos).Magnitude
                if dist < shortestDistance then
                    shortestDistance = dist
                    closestTarget = obj
                end
            end
        end
    end
    return closestTarget
end

task.spawn(function()
    local wasUnderground = false

    while task.wait(0.05) do
        if UndergroundFarm then
            pcall(function()
                local target = getClosestTarget()
                local character = LocalPlayer.Character
                
                if character and character:FindFirstChild("HumanoidRootPart") then
                    local hrp = character.HumanoidRootPart
                    if target then
                        local targetHrp = target:FindFirstChild("HumanoidRootPart") or target.PrimaryPart
                        local targetHumanoid = target:FindFirstChildOfClass("Humanoid")

                        if targetHrp and targetHumanoid and targetHumanoid.Health > 0 then
                            wasUnderground = true
                            hrp.CFrame = CFrame.new(targetHrp.Position - Vector3.new(0, UndergroundDistance, 0), targetHrp.Position)

                            if not hrp:FindFirstChild("UndergroundBV") then
                                local bv = Instance.new("BodyVelocity")
                                bv.Name = "UndergroundBV"
                                bv.Velocity = Vector3.zero
                                bv.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
                                bv.Parent = hrp
                            end
                        end
                    else
                        if wasUnderground then
                            wasUnderground = false
                            local bv = hrp:FindFirstChild("UndergroundBV")
                            if bv then bv:Destroy() end
                            if lastSafeSurfaceCFrame then hrp.CFrame = lastSafeSurfaceCFrame end
                        end
                    end
                end
            end)
        else
            if wasUnderground then
                wasUnderground = false
                pcall(function()
                    local character = LocalPlayer.Character
                    if character and character:FindFirstChild("HumanoidRootPart") then
                        local bv = character.HumanoidRootPart:FindFirstChild("UndergroundBV")
                        if bv then bv:Destroy() end
                        if lastSafeSurfaceCFrame then character.HumanoidRootPart.CFrame = lastSafeSurfaceCFrame end
                    end
                end)
            end
            task.wait(0.3)
        end
    end
end)

--========================================================
-- CHARACTER LIFECYCLE
--========================================================

local function HookCharacter(character)
    local humanoid = character:WaitForChild("Humanoid", 15)
    if not humanoid then return end

    if State.DeathConnection then pcall(function() State.DeathConnection:Disconnect() end) end
    State.DeathConnection = humanoid.Died:Connect(function()
        State.ResetDeadline = nil
        State.SequenceRunning = false
        State.SetupRunning = false
        State.FreezeActive = false
        task.wait(1)
        NewCycle("death")
    end)
end

if LocalPlayer.Character then task.spawn(function() HookCharacter(LocalPlayer.Character) end) end

LocalPlayer.CharacterAdded:Connect(function(character)
    local wasTransformSwap = State.TransformCharacterSwap
    State.TransformCharacterSwap = false

    Unfreeze()
    State.ResetDeadline = nil
    State.SequenceRunning = false
    State.SetupRunning = false

    task.spawn(function()
        HookCharacter(character)
        if not wasTransformSwap then
            NewCycle("respawn")
            task.wait(1.2)
        end
    end)
end)

--========================================================
-- AUTO UP TECH & ANTI AFK & AUTO SPAWN
--========================================================

local StatTypes = {"STRENGTH", "TECH", "MORALE"}
local ScrapTypes = {"Enhanced Scrap", "Improved Scrap", "Normal Scrap"}

task.spawn(function()
    while task.wait(0.2) do
        if AutoUpTechFullAll and UpgradeEvent then
            for i = 1, #StatTypes do
                for j = 1, #ScrapTypes do
                    if not AutoUpTechFullAll then break end
                    pcall(function() UpgradeEvent:FireServer(StatTypes[i], ScrapTypes[j]) end)
                    task.wait(0.05)
                end
            end
        end
    end
end)

task.spawn(function()
    while task.wait(1) do
        if AutoAntiAFK then
            pcall(function()
                local vu = game:GetService("VirtualUser")
                vu:CaptureController()
                vu:ClickButton2(Vector2.new())
            end)
        end
    end
end)

task.spawn(function()
    while task.wait(SpawnDelay) do
        if AutoSpawn and SpawnPartner then
            pcall(function() SpawnPartner:FireServer("Ultimate Camcorder Strider") end)
        end
    end
end)

--========================================================
-- OPTIMIZED ESP
--========================================================

local function ClearAllESP()
    for _, v in ipairs(workspace:GetDescendants()) do
        if v.Name == "Blackhat_ESP" or v.Name == "Blackhat_ESP_Text" then
            v:Destroy()
        end
    end
end

local function ApplyESP(model)
    local humanoid = model:FindFirstChildOfClass("Humanoid")
    local hrp = model:FindFirstChild("HumanoidRootPart") or model.PrimaryPart
    if not humanoid or not hrp or humanoid.Health <= 0 then return end

    if not model:FindFirstChild("Blackhat_ESP") then
        local hl = Instance.new("Highlight")
        hl.Name = "Blackhat_ESP"
        hl.FillColor = Color3.fromRGB(255, 45, 45)
        hl.OutlineColor = Color3.fromRGB(255, 255, 255)
        hl.FillTransparency = 0.5
        hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
        hl.Adornee = model
        hl.Parent = model
    end

    local textGui = hrp:FindFirstChild("Blackhat_ESP_Text")
    if not textGui then
        textGui = Instance.new("BillboardGui")
        textGui.Name = "Blackhat_ESP_Text"
        textGui.Adornee = hrp
        textGui.Size = UDim2.new(0, 180, 0, 40)
        textGui.StudsOffset = Vector3.new(0, 3.5, 0)
        textGui.AlwaysOnTop = true

        local textLabel = Instance.new("TextLabel")
        textLabel.Name = "InfoLabel"
        textLabel.Size = UDim2.new(1, 0, 1, 0)
        textLabel.BackgroundTransparency = 1
        textLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
        textLabel.TextStrokeTransparency = 0
        textLabel.TextSize = 11
        textLabel.Font = Enum.Font.GothamBold
        textLabel.RichText = true
        textLabel.Parent = textGui
        textGui.Parent = hrp
    end

    local label = textGui:FindFirstChild("InfoLabel")
    if label then
        label.Text = string.format("%s\n<font color=\"#00FF88\">HP: %d/%d</font>", model.Name, math.floor(humanoid.Health), math.floor(humanoid.MaxHealth))
    end
end

task.spawn(function()
    while task.wait(0.4) do
        if AutoESP then
            local folder = workspace:FindFirstChild("AI") or workspace
            for _, obj in ipairs(folder:GetChildren()) do
                if obj:IsA("Model") and not Players:GetPlayerFromCharacter(obj) then
                    ApplyESP(obj)
                end
            end
        else
            ClearAllESP()
        end
    end
end)

--========================================================
-- USER INTERFACE (BLACKHAT HUB)
--========================================================

local TargetGui = CoreGui
pcall(function() if gethui then TargetGui = gethui() end end)

local old = TargetGui:FindFirstChild("BLACKHAT_HUB")
if old then old:Destroy() end

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "BLACKHAT_HUB"
ScreenGui.ResetOnSpawn = false
ScreenGui.IgnoreGuiInset = true
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
ScreenGui.Parent = TargetGui

local Window = Instance.new("Frame")
Window.Name = "Window"
Window.Size = UDim2.fromOffset(330, 420)
Window.Position = UDim2.new(0.5, -165, 0.5, -210)
Window.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
Window.BorderSizePixel = 0
Window.ClipsDescendants = true
Window.Parent = ScreenGui

local WindowCorner = Instance.new("UICorner")
WindowCorner.CornerRadius = UDim.new(0, 10)
WindowCorner.Parent = Window

local WindowStroke = Instance.new("UIStroke")
WindowStroke.Color = Color3.fromRGB(0, 230, 150)
WindowStroke.Thickness = 1.2
WindowStroke.Transparency = 0.5
WindowStroke.Parent = Window

-- Header
local Header = Instance.new("Frame")
Header.Size = UDim2.new(1, 0, 0, 44)
Header.BackgroundColor3 = Color3.fromRGB(35, 35, 42)
Header.BorderSizePixel = 0
Header.Parent = Window

local HeaderCorner = Instance.new("UICorner")
HeaderCorner.CornerRadius = UDim.new(0, 10)
HeaderCorner.Parent = Header

local HeaderLogo = Instance.new("ImageLabel")
HeaderLogo.Size = UDim2.fromOffset(26, 26)
HeaderLogo.Position = UDim2.fromOffset(10, 9)
HeaderLogo.BackgroundTransparency = 1
HeaderLogo.Image = ICON_ID
HeaderLogo.Parent = Header

local Title = Instance.new("TextLabel")
Title.BackgroundTransparency = 1
Title.Size = UDim2.new(0, 150, 0, 20)
Title.Position = UDim2.fromOffset(42, 6)
Title.Text = "Blackhat Hub <font color=\"#00FFAA\">| SMART AUTO</font>"
Title.RichText = true
Title.TextColor3 = Color3.new(1, 1, 1)
Title.TextSize = 13
Title.Font = Enum.Font.GothamBold
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.ZIndex = 3
Title.Parent = Header

local SubTitle = Instance.new("TextLabel")
SubTitle.BackgroundTransparency = 1
SubTitle.Size = UDim2.new(0, 150, 0, 12)
SubTitle.Position = UDim2.fromOffset(42, 24)
SubTitle.Text = "by blackhat"
SubTitle.TextColor3 = Color3.fromRGB(160, 160, 175)
SubTitle.TextSize = 9
SubTitle.Font = Enum.Font.GothamMedium
SubTitle.TextXAlignment = Enum.TextXAlignment.Left
SubTitle.ZIndex = 3
SubTitle.Parent = Header

local ModeButton
local ModeLabel = Instance.new("TextLabel")
ModeLabel.BackgroundTransparency = 1
ModeLabel.Size = UDim2.fromOffset(115, 25)
ModeLabel.Position = UDim2.new(0, 168, 0, 9)
ModeLabel.Text = "Current Mode: " .. SelectedMode
ModeLabel.TextColor3 = Color3.fromRGB(185, 185, 185)
ModeLabel.TextSize = 9
ModeLabel.Font = Enum.Font.Gotham
ModeLabel.TextXAlignment = Enum.TextXAlignment.Center
ModeLabel.ZIndex = 3
ModeLabel.Parent = Header

-- Floating Show Button
local ShowButton = Instance.new("ImageButton")
ShowButton.Name = "ShowButton"
ShowButton.Size = UDim2.fromOffset(46, 46)
ShowButton.Position = UDim2.new(0, 12, 0.5, -23)
ShowButton.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
ShowButton.BorderSizePixel = 0
ShowButton.Visible = false
ShowButton.ZIndex = 100
ShowButton.Parent = ScreenGui

local ShowCorner = Instance.new("UICorner")
ShowCorner.CornerRadius = UDim.new(0, 12)
ShowCorner.Parent = ShowButton

local ShowStroke = Instance.new("UIStroke")
ShowStroke.Color = Color3.fromRGB(0, 230, 150)
ShowStroke.Thickness = 1.5
ShowStroke.Transparency = 0.3
ShowStroke.Parent = ShowButton

local ShowIcon = Instance.new("ImageLabel")
ShowIcon.Size = UDim2.fromOffset(28, 28)
ShowIcon.Position = UDim2.new(0.5, -14, 0.5, -14)
ShowIcon.BackgroundTransparency = 1
ShowIcon.Image = ICON_ID
ShowIcon.ZIndex = 101
ShowIcon.Parent = ShowButton

-- Drag Floating Button
do
    local draggingShow = false
    local dragStartShow, startPosShow
    ShowButton.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            draggingShow = true
            dragStartShow = input.Position
            startPosShow = ShowButton.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then draggingShow = false end
            end)
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if draggingShow and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            local delta = input.Position - dragStartShow
            ShowButton.Position = UDim2.new(startPosShow.X.Scale, startPosShow.X.Offset + delta.X, startPosShow.Y.Scale, startPosShow.Y.Offset + delta.Y)
        end
    end)
end

-- Hide Button
local HideButton = Instance.new("TextButton")
HideButton.Name = "HideButton"
HideButton.Size = UDim2.fromOffset(34, 30)
HideButton.Position = UDim2.new(1, -40, 0, 7)
HideButton.BackgroundColor3 = Color3.fromRGB(50, 50, 58)
HideButton.BorderSizePixel = 0
HideButton.Text = "—"
HideButton.TextColor3 = Color3.fromRGB(235, 235, 235)
HideButton.TextSize = 12
HideButton.Font = Enum.Font.GothamBold
HideButton.ZIndex = 10
HideButton.Parent = Header

Instance.new("UICorner", HideButton).CornerRadius = UDim.new(0, 6)

HideButton.MouseButton1Click:Connect(function()
    CloseModePicker()
    if CloseTransformPicker then CloseTransformPicker() end
    Window.Visible = false
    ShowButton.Visible = true
end)

ShowButton.MouseButton1Click:Connect(function()
    Window.Visible = true
    ShowButton.Visible = false
end)

-- Window Drag
do
    local dragging = false
    local dragStart, startPos, dragInput
    Header.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = Window.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then dragging = false end
            end)
        end
    end)
    Header.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then dragInput = input end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if dragging and input == dragInput then
            local delta = input.Position - dragStart
            Window.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)
end

-- Scrolling Content
local Content = Instance.new("ScrollingFrame")
Content.Size = UDim2.new(1, -12, 1, -52)
Content.Position = UDim2.fromOffset(6, 48)
Content.BackgroundTransparency = 1
Content.BorderSizePixel = 0
Content.ScrollBarThickness = 3
Content.ScrollBarImageColor3 = Color3.fromRGB(0, 230, 150)
Content.ScrollingDirection = Enum.ScrollingDirection.Y
Content.AutomaticCanvasSize = Enum.AutomaticSize.Y
Content.CanvasSize = UDim2.fromOffset(0, 0)
Content.Parent = Window

local Padding = Instance.new("UIPadding")
Padding.PaddingLeft = UDim.new(0, 5)
Padding.PaddingRight = UDim.new(0, 5)
Padding.PaddingTop = UDim.new(0, 4)
Padding.PaddingBottom = UDim.new(0, 8)
Padding.Parent = Content

local Layout = Instance.new("UIListLayout")
Layout.Padding = UDim.new(0, 5)
Layout.SortOrder = Enum.SortOrder.LayoutOrder
Layout.Parent = Content

local function AddButton(text)
    local button = Instance.new("TextButton")
    button.Size = UDim2.new(1, 0, 0, 34)
    button.BackgroundColor3 = Color3.fromRGB(42, 44, 54)
    button.BorderSizePixel = 0
    button.TextColor3 = Color3.new(1, 1, 1)
    button.TextSize = 11
    button.Font = Enum.Font.GothamMedium
    button.Text = text
    button.ZIndex = 5
    button.Parent = Content
    Instance.new("UICorner", button).CornerRadius = UDim.new(0, 7)
    return button
end

local function SetToggle(button, name, enabled)
    button.Text = name .. ": " .. (enabled and "ON" or "OFF")
    button.BackgroundColor3 = enabled and Color3.fromRGB(0, 160, 100) or Color3.fromRGB(42, 44, 54)
end

local function SetModeText()
    ModeLabel.Text = "Current Mode: " .. SelectedMode
    ModeButton.Text = "MODE"
end

--========================================================
-- CONTROLS ATTACHMENT
--========================================================

local AutoModeButton = AddButton("Auto Mode: OFF")
local AutoReadyButton = AddButton("Auto Ready: OFF")
local StartLoopButton = AddButton("Start Loop: OFF")
local BuyTransformButton = AddButton("Auto Buy + Transform: OFF")
local TSMBattleArmorButton = AddButton("TSMBattleArmor: OFF")
local TTVCinemaButton = AddButton("TTVCinema: OFF")
local AutoTPButton = AddButton("Auto Tween / Freeze: OFF")
local UndergroundFarmButton = AddButton("Auto Farm: OFF")

local DepthContainer = Instance.new("Frame")
DepthContainer.Size = UDim2.new(1, 0, 0, 34)
DepthContainer.BackgroundColor3 = Color3.fromRGB(35, 37, 47)
DepthContainer.BorderSizePixel = 0
DepthContainer.Parent = Content
Instance.new("UICorner", DepthContainer).CornerRadius = UDim.new(0, 6)

local DepthTextLabel = Instance.new("TextLabel")
DepthTextLabel.Size = UDim2.new(1, -100, 1, 0)
DepthTextLabel.Position = UDim2.fromOffset(10, 0)
DepthTextLabel.BackgroundTransparency = 1
DepthTextLabel.Text = "Farm Depth: " .. tostring(UndergroundDistance) .. " studs"
DepthTextLabel.TextColor3 = Color3.fromRGB(220, 220, 220)
DepthTextLabel.TextSize = 10
DepthTextLabel.Font = Enum.Font.GothamBold
DepthTextLabel.TextXAlignment = Enum.TextXAlignment.Left
DepthTextLabel.Parent = DepthContainer

local DepthUpBtn = Instance.new("TextButton")
DepthUpBtn.Size = UDim2.fromOffset(36, 22)
DepthUpBtn.Position = UDim2.new(1, -86, 0.5, -11)
DepthUpBtn.BackgroundColor3 = Color3.fromRGB(50, 150, 100)
DepthUpBtn.Text = "+"
DepthUpBtn.TextColor3 = Color3.new(1, 1, 1)
DepthUpBtn.TextSize = 11
DepthUpBtn.Font = Enum.Font.GothamBold
DepthUpBtn.Parent = DepthContainer
Instance.new("UICorner", DepthUpBtn).CornerRadius = UDim.new(0, 5)

local DepthDownBtn = Instance.new("TextButton")
DepthDownBtn.Size = UDim2.fromOffset(36, 22)
DepthDownBtn.Position = UDim2.new(1, -44, 0.5, -11)
DepthDownBtn.BackgroundColor3 = Color3.fromRGB(150, 50, 50)
DepthDownBtn.Text = "-"
DepthDownBtn.TextColor3 = Color3.new(1, 1, 1)
DepthDownBtn.TextSize = 11
DepthDownBtn.Font = Enum.Font.GothamBold
DepthDownBtn.Parent = DepthContainer
Instance.new("UICorner", DepthDownBtn).CornerRadius = UDim.new(0, 5)

local AutoPunchButton = AddButton("Auto Punch: OFF")
local AntiVoidButton = AddButton("🛡️ Anti-Void Protection: ON")
local VoidHoverButton = AddButton("Void Hover: OFF")

local VoidDepthContainer = Instance.new("Frame")
VoidDepthContainer.Size = UDim2.new(1, 0, 0, 34)
VoidDepthContainer.BackgroundColor3 = Color3.fromRGB(35, 37, 47)
VoidDepthContainer.BorderSizePixel = 0
VoidDepthContainer.Parent = Content
Instance.new("UICorner", VoidDepthContainer).CornerRadius = UDim.new(0, 6)

local DepthLabel = Instance.new("TextLabel")
DepthLabel.Size = UDim2.new(1, -100, 1, 0)
DepthLabel.Position = UDim2.fromOffset(10, 0)
DepthLabel.BackgroundTransparency = 1
DepthLabel.Text = "Void Hover Depth: " .. tostring(VoidDepth)
DepthLabel.TextColor3 = Color3.fromRGB(220, 220, 220)
DepthLabel.TextSize = 10
DepthLabel.Font = Enum.Font.GothamBold
DepthLabel.TextXAlignment = Enum.TextXAlignment.Left
DepthLabel.Parent = VoidDepthContainer

local UpBtn = Instance.new("TextButton")
UpBtn.Size = UDim2.fromOffset(36, 22)
UpBtn.Position = UDim2.new(1, -86, 0.5, -11)
UpBtn.BackgroundColor3 = Color3.fromRGB(50, 150, 100)
UpBtn.Text = "+"
UpBtn.TextColor3 = Color3.new(1, 1, 1)
UpBtn.TextSize = 11
UpBtn.Font = Enum.Font.GothamBold
UpBtn.Parent = VoidDepthContainer
Instance.new("UICorner", UpBtn).CornerRadius = UDim.new(0, 5)

local DownBtn = Instance.new("TextButton")
DownBtn.Size = UDim2.fromOffset(36, 22)
DownBtn.Position = UDim2.new(1, -44, 0.5, -11)
DownBtn.BackgroundColor3 = Color3.fromRGB(150, 50, 50)
DownBtn.Text = "-"
DownBtn.TextColor3 = Color3.new(1, 1, 1)
DownBtn.TextSize = 11
DownBtn.Font = Enum.Font.GothamBold
DownBtn.Parent = VoidDepthContainer
Instance.new("UICorner", DownBtn).CornerRadius = UDim.new(0, 5)

local PosListContainer = Instance.new("Frame")
PosListContainer.Size = UDim2.new(1, 0, 0, 0)
PosListContainer.BackgroundTransparency = 1
PosListContainer.AutomaticSize = Enum.AutomaticSize.Y
PosListContainer.Parent = Content

local PosLayout = Instance.new("UIListLayout")
PosLayout.Padding = UDim.new(0, 4)
PosLayout.SortOrder = Enum.SortOrder.LayoutOrder
PosLayout.Parent = PosListContainer

local SavePosButton = AddButton("Save Current Position")
local ClearPosButton = AddButton("Clear Saved Positions")

local AutoESPButton = AddButton("ESP: OFF")
local AutoUpTechButton = AddButton("Auto Up Tech Full All: OFF")
local AutoAntiAFKButton = AddButton("Anti AFK: OFF")
local AutoSpawnButton = AddButton("Auto Spawn: OFF")
local AutoResetButton = AddButton("Auto Reset 1-2h: OFF")
ModeButton = AddButton("MODE")

--========================================================
-- EVENT HANDLERS
--========================================================

AutoModeButton.MouseButton1Click:Connect(function()
    AutoMode = not AutoMode
    SetToggle(AutoModeButton, "Auto Mode", AutoMode)
    if AutoMode then VoteMode(true) end
end)

AutoReadyButton.MouseButton1Click:Connect(function()
    AutoReady = not AutoReady
    SetToggle(AutoReadyButton, "Auto Ready", AutoReady)
end)

StartLoopButton.MouseButton1Click:Connect(function()
    AutoStartLoop = not AutoStartLoop
    if UpdateStartLoopUI then UpdateStartLoopUI(AutoStartLoop) end
end)

BuyTransformButton.MouseButton1Click:Connect(function()
    AutoBuyTransform = not AutoBuyTransform
    SetToggle(BuyTransformButton, "Auto Buy + Transform", AutoBuyTransform)
end)

TSMBattleArmorButton.MouseButton1Click:Connect(function()
    AutoTSMBattleArmor = not AutoTSMBattleArmor
    SetToggle(TSMBattleArmorButton, "TSMBattleArmor", AutoTSMBattleArmor)
end)

TTVCinemaButton.MouseButton1Click:Connect(function()
    AutoTTVCinema = not AutoTTVCinema
    SetToggle(TTVCinemaButton, "TTVCinema", AutoTTVCinema)
end)

AutoTPButton.MouseButton1Click:Connect(function()
    AutoTPFreeze = not AutoTPFreeze
    SetToggle(AutoTPButton, "Auto Tween / Freeze", AutoTPFreeze)
    if AutoTPFreeze then TPFreeze() else Unfreeze() end
end)

UndergroundFarmButton.MouseButton1Click:Connect(function()
    UndergroundFarm = not UndergroundFarm
    SetToggle(UndergroundFarmButton, "Auto Farm", UndergroundFarm)
end)

DepthUpBtn.MouseButton1Click:Connect(function()
    UndergroundDistance = math.clamp(UndergroundDistance + 1, 3, 100)
    DepthTextLabel.Text = "Farm Depth: " .. tostring(UndergroundDistance) .. " studs"
end)

DepthDownBtn.MouseButton1Click:Connect(function()
    UndergroundDistance = math.clamp(UndergroundDistance - 1, 3, 100)
    DepthTextLabel.Text = "Farm Depth: " .. tostring(UndergroundDistance) .. " studs"
end)

AutoPunchButton.MouseButton1Click:Connect(function()
    AutoPunch = not AutoPunch
    SetToggle(AutoPunchButton, "Auto Punch", AutoPunch)
end)

AntiVoidButton.MouseButton1Click:Connect(function()
    AntiVoid = not AntiVoid
    SetToggle(AntiVoidButton, "🛡️ Anti-Void Protection", AntiVoid)
end)

VoidHoverButton.MouseButton1Click:Connect(function()
    VoidActive = not VoidActive
    SetToggle(VoidHoverButton, "Void Hover", VoidActive)
    local _, hrp = GetCharacter()
    if VoidActive then
        if hrp then preVoidCFrame = hrp.CFrame end
    else
        if hrp and preVoidCFrame then
            hrp.AssemblyLinearVelocity = Vector3.zero
            hrp.CFrame = preVoidCFrame
            preVoidCFrame = nil
        end
    end
end)

UpBtn.MouseButton1Click:Connect(function()
    VoidDepth = math.clamp(VoidDepth + 10, 0, 1000)
    DepthLabel.Text = "Void Hover Depth: " .. tostring(VoidDepth)
end)

DownBtn.MouseButton1Click:Connect(function()
    VoidDepth = math.clamp(VoidDepth - 10, 0, 1000)
    DepthLabel.Text = "Void Hover Depth: " .. tostring(VoidDepth)
end)

local function AddPositionUI(posData)
    local item = Instance.new("Frame")
    item.Size = UDim2.new(1, 0, 0, 34)
    item.BackgroundColor3 = Color3.fromRGB(35, 37, 47)
    item.BorderSizePixel = 0
    item.Parent = PosListContainer
    Instance.new("UICorner", item).CornerRadius = UDim.new(0, 6)

    local nameLabel = Instance.new("TextLabel")
    nameLabel.Size = UDim2.new(0.5, 0, 1, 0)
    nameLabel.Position = UDim2.fromOffset(10, 0)
    nameLabel.BackgroundTransparency = 1
    nameLabel.Text = posData.name
    nameLabel.TextColor3 = Color3.fromRGB(255, 220, 180)
    nameLabel.TextSize = 10
    nameLabel.Font = Enum.Font.GothamBold
    nameLabel.TextXAlignment = Enum.TextXAlignment.Left
    nameLabel.Parent = item

    local tpBtn = Instance.new("TextButton")
    tpBtn.Size = UDim2.fromOffset(42, 22)
    tpBtn.Position = UDim2.new(1, -100, 0.5, -11)
    tpBtn.BackgroundColor3 = Color3.fromRGB(0, 140, 200)
    tpBtn.Text = "TP"
    tpBtn.TextColor3 = Color3.new(1, 1, 1)
    tpBtn.TextSize = 10
    tpBtn.Font = Enum.Font.GothamBold
    tpBtn.Parent = item
    Instance.new("UICorner", tpBtn).CornerRadius = UDim.new(0, 5)

    local delBtn = Instance.new("TextButton")
    delBtn.Size = UDim2.fromOffset(50, 22)
    delBtn.Position = UDim2.new(1, -54, 0.5, -11)
    delBtn.BackgroundColor3 = Color3.fromRGB(180, 50, 60)
    delBtn.Text = "Delete"
    delBtn.TextColor3 = Color3.new(1, 1, 1)
    delBtn.TextSize = 10
    delBtn.Font.GothamBold = Enum.Font.GothamBold
    delBtn.Parent = item
    Instance.new("UICorner", delBtn).CornerRadius = UDim.new(0, 5)

    tpBtn.MouseButton1Click:Connect(function()
        local _, hrp = GetCharacter()
        if hrp then hrp.CFrame = posData.cframe end
    end)

    delBtn.MouseButton1Click:Connect(function()
        for index, p in ipairs(savedPositions) do
            if p.id == posData.id then
                table.remove(savedPositions, index)
                break
            end
        end
        item:Destroy()
    end)
end

SavePosButton.MouseButton1Click:Connect(function()
    local _, hrp = GetCharacter()
    if hrp then
        positionCounter = positionCounter + 1
        local newPos = { id = positionCounter, cframe = hrp.CFrame, name = "Position " .. tostring(positionCounter) }
        table.insert(savedPositions, newPos)
        AddPositionUI(newPos)
    end
end)

ClearPosButton.MouseButton1Click:Connect(function()
    savedPositions = {}
    positionCounter = 0
    for _, child in ipairs(PosListContainer:GetChildren()) do
        if child:IsA("Frame") then child:Destroy() end
    end
end)

AutoESPButton.MouseButton1Click:Connect(function()
    AutoESP = not AutoESP
    SetToggle(AutoESPButton, "ESP", AutoESP)
    if not AutoESP then ClearAllESP() end
end)

AutoUpTechButton.MouseButton1Click:Connect(function()
    AutoUpTechFullAll = not AutoUpTechFullAll
    SetToggle(AutoUpTechButton, "Auto Up Tech Full All", AutoUpTechFullAll)
end)

AutoAntiAFKButton.MouseButton1Click:Connect(function()
    AutoAntiAFK = not AutoAntiAFK
    SetToggle(AutoAntiAFKButton, "Anti AFK", AutoAntiAFK)
end)

AutoSpawnButton.MouseButton1Click:Connect(function()
    AutoSpawn = not AutoSpawn
    SetToggle(AutoSpawnButton, "Auto Spawn", AutoSpawn)
end)

AutoResetButton.MouseButton1Click:Connect(function()
    AutoReset = not AutoReset
    SetToggle(AutoResetButton, "Auto Reset 1-2h", AutoReset)
end)

print("[Blackhat Hub] Script Execution Completed Successfully!")
