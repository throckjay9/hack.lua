-- script.lua
-- Roblox LocalScript for malicious script detection testing optimized for detecting heads with aimbot, improved GUI overlay with toggle shortcut
-- Features: ESP box, ESP line, aimbot locking on head, hologram, shoot on look/aim, auto kill
-- GUI can be toggled with H key
-- Must be run as LocalScript in Roblox environment

-- Services
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local LocalPlayer = Players.LocalPlayer
local Mouse = LocalPlayer:GetMouse()
local Camera = workspace.CurrentCamera

-- Variables
local enabledFeatures = {
    ESP = true,
    ESPLine = true,
    Aimbot = true,
    Hologram = true,
    ShootOnLook = true,
    ShootOnAim = true,
    AutoKill = false,
}

local settings = {
    AimFOV = 40,           -- degrees
    AimSmoothness = 0.3,   -- 0-1, lower means smoother/slower
    ShootInterval = 0.2,   -- seconds between shots
    HologramTransparency = 0.6,
    ESPBoxColor = Color3.fromRGB(0,255,0),
    ESPLineColor = Color3.fromRGB(255,0,0),
    ESPLineThickness = 1.5,
}

local lastShotTime = 0

-- UI Creation
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "MaliciousScriptDetectionUI"
ScreenGui.ResetOnSpawn = false
ScreenGui.IgnoreGuiInset = true
ScreenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")

local MainFrame = Instance.new("Frame")
MainFrame.Size = UDim2.new(0, 320, 0, 410)
MainFrame.Position = UDim2.new(0, 20, 0.5, -205)
MainFrame.BackgroundColor3 = Color3.fromRGB(30,30,30)
MainFrame.BorderSizePixel = 0
MainFrame.Visible = true
MainFrame.Parent = ScreenGui
MainFrame.Active = true
MainFrame.Draggable = true
MainFrame.ZIndex = 10

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1,0,0,40)
title.BackgroundTransparency = 1
title.Text = "Malicious Script Detection"
title.TextColor3 = Color3.new(1,1,1)
title.Font = Enum.Font.SourceSansBold
title.TextSize = 24
title.Parent = MainFrame

local UIContainer = Instance.new("ScrollingFrame")
UIContainer.Size = UDim2.new(1, -20, 1, -60)
UIContainer.Position = UDim2.new(0, 10, 0, 45)
UIContainer.BackgroundTransparency = 1
UIContainer.CanvasSize = UDim2.new(0,0,0,0)
UIContainer.Parent = MainFrame
UIContainer.ScrollBarThickness = 6
UIContainer.ZIndex = 10

local UIListLayout = Instance.new("UIListLayout")
UIListLayout.Parent = UIContainer
UIListLayout.SortOrder = Enum.SortOrder.LayoutOrder
UIListLayout.Padding = UDim.new(0,8)

-- Helper function to create toggles
local function CreateToggle(parent, text, defaultChecked, onToggle)
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, 0, 0, 35)
    frame.BackgroundTransparency = 1
    frame.Parent = parent

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(0.7, 0, 1, 0)
    label.BackgroundTransparency = 1
    label.Text = text
    label.TextColor3 = Color3.new(1,1,1)
    label.Font = Enum.Font.SourceSans
    label.TextSize = 18
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = frame

    local toggleButton = Instance.new("TextButton")
    toggleButton.Size = UDim2.new(0.3, -10, 0.8, 0)
    toggleButton.Position = UDim2.new(0.7, 10, 0.1, 0)
    toggleButton.AnchorPoint = Vector2.new(0,0)
    toggleButton.BackgroundColor3 = (defaultChecked and Color3.fromRGB(0,170,0) or Color3.fromRGB(170,0,0))
    toggleButton.Text = (defaultChecked and "ON" or "OFF")
    toggleButton.TextColor3 = Color3.new(1,1,1)
    toggleButton.Font = Enum.Font.SourceSansBold
    toggleButton.TextSize = 18
    toggleButton.Parent = frame

    toggleButton.MouseButton1Click:Connect(function()
        local isOn = toggleButton.Text == "ON"
        if isOn then
            toggleButton.Text = "OFF"
            toggleButton.BackgroundColor3 = Color3.fromRGB(170,0,0)
            onToggle(false)
        else
            toggleButton.Text = "ON"
            toggleButton.BackgroundColor3 = Color3.fromRGB(0,170,0)
            onToggle(true)
        end
    end)
end

-- Helper function to create sliders
local function CreateSlider(parent, text, min, max, default, decimals, onChange)
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, 0, 0, 55)
    frame.BackgroundTransparency = 1
    frame.Parent = parent

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, 0, 0, 20)
    label.BackgroundTransparency = 1
    label.Text = text .. ": " .. tostring(default)
    label.TextColor3 = Color3.new(1,1,1)
    label.Font = Enum.Font.SourceSans
    label.TextSize = 16
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = frame

    local slider = Instance.new("TextBox")
    slider.Size = UDim2.new(1, 0, 0, 25)
    slider.Position = UDim2.new(0, 0, 0, 27)
    slider.Text = tostring(default)
    slider.ClearTextOnFocus = false
    slider.TextColor3 = Color3.new(1,1,1)
    slider.BackgroundColor3 = Color3.fromRGB(50,50,50)
    slider.Font = Enum.Font.SourceSans
    slider.TextSize = 18
    slider.Parent = frame

    local function updateValue(val)
        local num = tonumber(val)
        if num and num >= min and num <= max then
            onChange(num)
            label.Text = text .. ": " .. string.format("%."..decimals.."f", num)
            slider.Text = string.format("%."..decimals.."f", num)
            slider.TextColor3 = Color3.new(1,1,1)
        else
            slider.TextColor3 = Color3.fromRGB(255,100,100)
        end
    end

    slider.FocusLost:Connect(function(enterPressed)
        if enterPressed then
            updateValue(slider.Text)
        else
            slider.Text = string.format("%."..decimals.."f", onChange and (function()
                return settings[text:gsub("%s","")] or default
            end)() or default)
            slider.TextColor3 = Color3.new(1,1,1)
        end
    end)
end

-- Populate the UI container with toggles
CreateToggle(UIContainer, "ESP", enabledFeatures.ESP, function(v) enabledFeatures.ESP = v end)
CreateToggle(UIContainer, "ESP Line", enabledFeatures.ESPLine, function(v) enabledFeatures.ESPLine = v end)
CreateToggle(UIContainer, "Aimbot", enabledFeatures.Aimbot, function(v) enabledFeatures.Aimbot = v end)
CreateToggle(UIContainer, "Hologram", enabledFeatures.Hologram, function(v) enabledFeatures.Hologram = v end)
CreateToggle(UIContainer, "ShootOnLook", enabledFeatures.ShootOnLook, function(v) enabledFeatures.ShootOnLook = v end)
CreateToggle(UIContainer, "ShootOnAim", enabledFeatures.ShootOnAim, function(v) enabledFeatures.ShootOnAim = v end)
CreateToggle(UIContainer, "AutoKill", enabledFeatures.AutoKill, function(v) enabledFeatures.AutoKill = v end)

-- Populate the UI container with sliders
CreateSlider(UIContainer, "AimFOV", 5, 180, settings.AimFOV, 0, function(v) settings.AimFOV = v end)
CreateSlider(UIContainer, "AimSmoothness", 0, 1, settings.AimSmoothness, 2, function(v) settings.AimSmoothness = v end)
CreateSlider(UIContainer, "ShootInterval", 0.05, 1, settings.ShootInterval, 2, function(v) settings.ShootInterval = v end)
CreateSlider(UIContainer, "HologramTransparency", 0, 1, settings.HologramTransparency, 2, function(v) settings.HologramTransparency = v end)

-- Utility functions
local function getCharacterRoot(char)
    return char:FindFirstChild("HumanoidRootPart") or char.PrimaryPart
end

local function isAlive(player)
    local char = player.Character
    if char then
        local humanoid = char:FindFirstChildOfClass("Humanoid")
        return humanoid and humanoid.Health > 0
    end
    return false
end

-- ESP Boxes and Lines management
local espBoxes = {}
local espLines = {}

local function CreateESPBox(player)
    if espBoxes[player] and espBoxes[player].Parent then return end
    if not player.Character then return end
    local rootPart = getCharacterRoot(player.Character)
    if not rootPart then return end

    local billboard = Instance.new("BillboardGui")
    billboard.Name = "ESPBox"
    billboard.Adornee = rootPart
    billboard.AlwaysOnTop = true
    billboard.Size = UDim2.new(4, 0, 6, 0)
    billboard.StudsOffset = Vector3.new(0, 3, 0)
    billboard.Parent = ScreenGui

    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1,0,1,0)
    frame.BackgroundColor3 = settings.ESPBoxColor
    frame.BorderColor3 = Color3.new(0,0,0)
    frame.BorderSizePixel = 2
    frame.BackgroundTransparency = 0.65
    frame.Parent = billboard

    espBoxes[player] = billboard
end

local function RemoveESPBox(player)
    if espBoxes[player] then
        espBoxes[player]:Destroy()
        espBoxes[player] = nil
    end
end

local function CreateESPLine(player)
    if espLines[player] then return end
    local line = Instance.new("Frame")
    line.Name = "ESPLine"
    line.AnchorPoint = Vector2.new(0,0.5)
    line.BackgroundColor3 = settings.ESPLineColor
    line.BorderSizePixel = 0
    line.ZIndex = 10
    line.Size = UDim2.new(0, 1, 0, settings.ESPLineThickness)
    line.Parent = ScreenGui

    espLines[player] = line
end

local function RemoveESPLine(player)
    if espLines[player] then
        espLines[player]:Destroy()
        espLines[player] = nil
    end
end

-- Returns head position and onScreen, nil if unavailable
local function getHeadScreenPos(player)
    if not player.Character then return nil, false end
    local head = player.Character:FindFirstChild("Head")
    if not head then return nil, false end
    return Camera:WorldToViewportPoint(head.Position)
end

-- Returns the bottom center screen position for line start
local function getScreenBottomCenter()
    local size = Camera.ViewportSize
    return Vector2.new(size.X/2, size.Y)
end

RunService.RenderStepped:Connect(function()
    for _, player in pairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and isAlive(player) then
            if enabledFeatures.ESP then
                CreateESPBox(player)
            else
                RemoveESPBox(player)
            end

            if enabledFeatures.ESPLine then
                CreateESPLine(player)
            else
                RemoveESPLine(player)
            end
        else
            RemoveESPBox(player)
            RemoveESPLine(player)
        end
    end

    if enabledFeatures.ESPLine then
        local startPos = getScreenBottomCenter()
        for player, line in pairs(espLines) do
            local screenPos, onScreen = getHeadScreenPos(player)
            if onScreen then
                local endPos = Vector2.new(screenPos.X, screenPos.Y)
                local direction = endPos - startPos
                local length = direction.Magnitude
                if length < 2 then
                    line.Visible = false
                else
                    line.Visible = true
                    line.Position = UDim2.new(0, startPos.X, 0, startPos.Y)
                    line.Size = UDim2.new(0, length, 0, settings.ESPLineThickness)
                    line.Rotation = math.deg(math.atan2(direction.Y, direction.X))
                end
            else
                line.Visible = false
            end
        end
    end
end)

local function getClosestPlayerToMouse()
    local closestDistance = math.huge
    local closestPlayer = nil
    local mousePos = Vector2.new(Mouse.X, Mouse.Y)
    for _, player in pairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and isAlive(player) then
            local char = player.Character
            local root = getCharacterRoot(char)
            if root then
                local screenPos, onScreen = Camera:WorldToViewportPoint(root.Position)
                if onScreen then
                    local delta = Vector2.new(screenPos.X, screenPos.Y) - mousePos
                    local dist = delta.Magnitude
                    if dist < closestDistance then
                        closestDistance = dist
                        closestPlayer = player
                    end
                end
            end
        end
    end
    if closestDistance <= settings.AimFOV then
        return closestPlayer
    else
        return nil
    end
end

local function aimbot()
    if not enabledFeatures.Aimbot then return end
    if not isAlive(LocalPlayer) then return end
    local target = getClosestPlayerToMouse()
    if target and target.Character then
        local head = target.Character:FindFirstChild("Head")
        if head then
            local camCF = Camera.CFrame
            local targetPos = head.Position

            local direction = (targetPos - camCF.Position).Unit
            local lookVector = camCF.LookVector

            local smoothDir = lookVector:Lerp(direction, settings.AimSmoothness)
            Camera.CFrame = CFrame.new(camCF.Position, camCF.Position + smoothDir)
        end
    end
end

local function shoot()
    local now = tick()
    if now - lastShotTime < settings.ShootInterval then return end
    lastShotTime = now

    -- Implement actual shooting action for your game here
    print("[Action] Shoot triggered")
end

local function isLookingAt(player)
    local mouseTarget = Mouse.Target
    if not mouseTarget or not player.Character then return false end
    return mouseTarget:IsDescendantOf(player.Character)
end

local function isAimingAt(player)
    return isLookingAt(player)
end

local function performShootOnLookAim()
    if not (enabledFeatures.ShootOnLook or enabledFeatures.ShootOnAim) then return end
    if not isAlive(LocalPlayer) then return end

    for _, player in pairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and isAlive(player) then
            if enabledFeatures.ShootOnLook and isLookingAt(player) then
                shoot()
                return
            end
            if enabledFeatures.ShootOnAim and isAimingAt(player) then
                shoot()
                return
            end
        end
    end
end

local function autoKill()
    if not enabledFeatures.AutoKill then return end
    if not isAlive(LocalPlayer) then return end

    for _, player in pairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and isAlive(player) then
            print("[AutoKill] Attempting kill on "..player.Name)
            -- Insert game specific kill logic here
        end
    end
end

RunService.RenderStepped:Connect(function()
    if enabledFeatures.Aimbot then
        aimbot()
    end

    performShootOnLookAim()
    autoKill()
end)

-- Shortcut: Toggle GUI visibility with Right Control
UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if input.KeyCode == Enum.KeyCode.RightControl then
        MainFrame.Visible = not MainFrame.Visible
    end
end)

print("Script loaded. Press Right Control to toggle GUI visibility.")

if input.KeyCode == Enum.KeyCode.RightControl then

if input.KeyCode == Enum.KeyCode.M then
