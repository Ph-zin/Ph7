local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")

local lplr = Players.LocalPlayer
local camera = Workspace.CurrentCamera
local mouse = lplr:GetMouse()

local espEnabled = false
local aimbotEnabled = false
local teamCheckEnabled = false
local alliesList = {}
local fovEnabled = false
local fovRadius = 90
local fovCircle = nil
local aimbotSmoothness = 0.25
local espDrawings = {}

local aimbotTarget = "Head"
local legitHoldTime = 800
local currentTarget = nil
local targetAcquireTime = nil

local noRecoilEnabled = false
local speedEnabled = false
local currentSpeed = 16
local noclipEnabled = false

local silentAimEnabled = false
local spinBotEnabled = false
local spinBotSpeed = 10

local flickRequested = false
local flickTargetPos = nil
local silentAimActive = false
local silentAimOriginalCFrame = nil

local aimIndicatorLine = nil

local activeSlider = nil
local draggingFloat = false

local controllerShooting = false
local controllerAiming = false

local function isPlayerDead(player)
    local char = player.Character
    if not char then return true end
    local humanoid = char:FindFirstChild("Humanoid")
    return not humanoid or humanoid.Health <= 0
end

local function isAlly(player)
    for _, name in ipairs(alliesList) do if player.Name == name then return true end end
    return false
end

local function isSameTeam(player)
    if not teamCheckEnabled then return false end
    if lplr.Team == nil or player.Team == nil then return false end
    return lplr.Team == player.Team
end

local function isVisible(targetPos, targetCharacter)
    local char = lplr.Character
    if not char then return false end
    local origin = camera.CFrame.Position
    local direction = targetPos - origin
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Exclude
    params.FilterDescendantsInstances = {char, targetCharacter}
    local result = Workspace:Raycast(origin, direction, params)
    return result == nil
end

local function isWithinFOV(screenPos)
    if not fovEnabled then return true end
    local center = Vector2.new(camera.ViewportSize.X / 2, camera.ViewportSize.Y / 2)
    return (Vector2.new(screenPos.X, screenPos.Y) - center).Magnitude <= fovRadius
end

local function getTargetPosition(character, targetType)
    if not character then return nil end
    if targetType == "Head" then
        local head = character:FindFirstChild("Head")
        return head and head.Position or nil
    elseif targetType == "Chest" then
        local torso = character:FindFirstChild("UpperTorso") or character:FindFirstChild("Torso")
        return torso and torso.Position or nil
    end
    return nil
end

local function getBestTarget()
    if currentTarget then
        local targetChar = currentTarget.Character
        if targetChar and not isPlayerDead(currentTarget) then
            local checkType = "Head"
            if aimbotTarget == "Chest" and targetAcquireTime then
                if (tick() - targetAcquireTime) * 1000 < legitHoldTime then checkType = "Chest" end
            end
            local checkPos = getTargetPosition(targetChar, checkType)
            if checkPos then
                local screenPos, onScreen = camera:WorldToViewportPoint(checkPos)
                if onScreen and isWithinFOV(screenPos) and isVisible(checkPos, targetChar) then
                    local targetType = "Head"
                    if aimbotTarget == "Chest" and targetAcquireTime then
                        if (tick() - targetAcquireTime) * 1000 < legitHoldTime then targetType = "Chest" end
                    end
                    return getTargetPosition(targetChar, targetType), currentTarget
                end
            end
        end
        currentTarget = nil
        targetAcquireTime = nil
    end

    local bestPlayer = nil
    local bestDistToCenter = math.huge
    local bestDist3D = math.huge
    local bestHealth = math.huge
    local screenCenter = Vector2.new(camera.ViewportSize.X / 2, camera.ViewportSize.Y / 2)

    for _, v in pairs(Players:GetPlayers()) do
        if v == lplr then continue end
        if isPlayerDead(v) then continue end
        if isAlly(v) then continue end
        if teamCheckEnabled and isSameTeam(v) then continue end

        local checkPos = getTargetPosition(v.Character, "Head")
        if not checkPos then continue end

        local screenPos, onScreen = camera:WorldToViewportPoint(checkPos)
        if not onScreen or not isWithinFOV(screenPos) then continue end
        if not isVisible(checkPos, v.Character) then continue end

        local distToCenter = (Vector2.new(screenPos.X, screenPos.Y) - screenCenter).Magnitude
        local dist3D = (checkPos - camera.CFrame.Position).Magnitude
        local humanoid = v.Character and v.Character:FindFirstChild("Humanoid")
        local health = humanoid and humanoid.Health or 100

        local isBetter = false
        if distToCenter < bestDistToCenter - 0.1 then
            isBetter = true
        elseif math.abs(distToCenter - bestDistToCenter) <= 0.1 then
            if dist3D < bestDist3D - 1 then
                isBetter = true
            elseif math.abs(dist3D - bestDist3D) <= 1 then
                if health < bestHealth then
                    isBetter = true
                end
            end
        end

        if isBetter then
            bestDistToCenter = distToCenter
            bestDist3D = dist3D
            bestHealth = health
            bestPlayer = v
        end
    end

    if bestPlayer then
        currentTarget = bestPlayer
        targetAcquireTime = tick()
        local targetType = "Head"
        if aimbotTarget == "Chest" then
            targetType = (targetAcquireTime and (tick() - targetAcquireTime) * 1000 < legitHoldTime) and "Chest" or "Head"
        end
        return getTargetPosition(bestPlayer.Character, targetType), bestPlayer
    else
        currentTarget = nil
        targetAcquireTime = nil
        return nil, nil
    end
end

local function requestFlick()
    if not silentAimEnabled then return end
    local targetPos, _ = getBestTarget()
    if targetPos then
        flickRequested = true
        flickTargetPos = targetPos
    end
end

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if input.UserInputType == Enum.UserInputType.Gamepad1 then
        if input.KeyCode == Enum.KeyCode.ButtonR2 then
            controllerShooting = true
            requestFlick()
        elseif input.KeyCode == Enum.KeyCode.ButtonL2 then
            controllerAiming = true
        end
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.Gamepad1 then
        if input.KeyCode == Enum.KeyCode.ButtonR2 then
            controllerShooting = false
        elseif input.KeyCode == Enum.KeyCode.ButtonL2 then
            controllerAiming = false
        end
    elseif input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        activeSlider = nil
        draggingFloat = false
    end
end)

UserInputService.WindowFocused:Connect(function()
    controllerShooting = false
    controllerAiming = false
end)

local function isShootingPressed()
    return controllerShooting
end

local function isAimingPressed()
    return controllerAiming
end

local function updateSpinBot()
    if not spinBotEnabled then return end
    local char = lplr.Character
    if not char then return end
    local rootPart = char:FindFirstChild("HumanoidRootPart")
    if not rootPart then return end
    local delta = spinBotSpeed * 2 * math.pi / 60
    rootPart.CFrame = rootPart.CFrame * CFrame.Angles(0, delta, 0)
end

local function setNoclip(state)
    local char = lplr.Character
    if char then
        for _, part in ipairs(char:GetDescendants()) do
            if part:IsA("BasePart") then
                part.CanCollide = not state
            end
        end
    end
end

local function createFOVCircle()
    if fovCircle then return end
    fovCircle = Drawing.new("Circle")
    fovCircle.Thickness = 1
    fovCircle.Color = Color3.fromRGB(255, 255, 255)
    fovCircle.Filled = false
    fovCircle.NumSides = 64
    fovCircle.Visible = false
end
local function updateFOVCircle()
    if not fovCircle then return end
    fovCircle.Visible = fovEnabled
    if not fovEnabled then return end
    fovCircle.Position = Vector2.new(camera.ViewportSize.X / 2, camera.ViewportSize.Y / 2)
    fovCircle.Radius = fovRadius

    local hasTarget = false
    if fovEnabled then
        for _, v in pairs(Players:GetPlayers()) do
            if v == lplr then continue end
            if isPlayerDead(v) then continue end
            if isAlly(v) then continue end
            if teamCheckEnabled and isSameTeam(v) then continue end
            local pos = getTargetPosition(v.Character, "Head")
            if not pos then continue end
            local screenPos, onScreen = camera:WorldToViewportPoint(pos)
            if onScreen and isWithinFOV(screenPos) and isVisible(pos, v.Character) then
                hasTarget = true
                break
            end
        end
    end
    fovCircle.Color = hasTarget and Color3.fromRGB(0, 255, 0) or Color3.fromRGB(255, 255, 255)
end

local function clearPlayerESP(player)
    local data = espDrawings[player]
    if data then
        if data.box then data.box:Remove() end
        if data.healthBar then data.healthBar:Remove() end
        if data.healthBg then data.healthBg:Remove() end
        for _, line in ipairs(data.skeletonLines or {}) do line:Remove() end
        espDrawings[player] = nil
    end
end

local function getSkeletonConnections(character)
    local connections = {}
    local parts = {
        Head = character:FindFirstChild("Head"),
        Torso = character:FindFirstChild("Torso"),
        UpperTorso = character:FindFirstChild("UpperTorso"),
        LowerTorso = character:FindFirstChild("LowerTorso"),
        HumanoidRootPart = character:FindFirstChild("HumanoidRootPart"),
        LeftUpperArm = character:FindFirstChild("LeftUpperArm"),
        LeftLowerArm = character:FindFirstChild("LeftLowerArm"),
        LeftHand = character:FindFirstChild("LeftHand"),
        RightUpperArm = character:FindFirstChild("RightUpperArm"),
        RightLowerArm = character:FindFirstChild("RightLowerArm"),
        RightHand = character:FindFirstChild("RightHand"),
        LeftUpperLeg = character:FindFirstChild("LeftUpperLeg"),
        LeftLowerLeg = character:FindFirstChild("LeftLowerLeg"),
        LeftFoot = character:FindFirstChild("LeftFoot"),
        RightUpperLeg = character:FindFirstChild("RightUpperLeg"),
        RightLowerLeg = character:FindFirstChild("RightLowerLeg"),
        RightFoot = character:FindFirstChild("RightFoot")
    }
    if parts.UpperTorso and parts.LowerTorso then
        if parts.Head and parts.UpperTorso then table.insert(connections, {parts.Head, parts.UpperTorso}) end
        if parts.UpperTorso and parts.LowerTorso then table.insert(connections, {parts.UpperTorso, parts.LowerTorso}) end
        if parts.UpperTorso and parts.LeftUpperArm then table.insert(connections, {parts.UpperTorso, parts.LeftUpperArm}) end
        if parts.LeftUpperArm and parts.LeftLowerArm then table.insert(connections, {parts.LeftUpperArm, parts.LeftLowerArm}) end
        if parts.LeftLowerArm and parts.LeftHand then table.insert(connections, {parts.LeftLowerArm, parts.LeftHand}) end
        if parts.UpperTorso and parts.RightUpperArm then table.insert(connections, {parts.UpperTorso, parts.RightUpperArm}) end
        if parts.RightUpperArm and parts.RightLowerArm then table.insert(connections, {parts.RightUpperArm, parts.RightLowerArm}) end
        if parts.RightLowerArm and parts.RightHand then table.insert(connections, {parts.RightLowerArm, parts.RightHand}) end
        if parts.LowerTorso and parts.LeftUpperLeg then table.insert(connections, {parts.LowerTorso, parts.LeftUpperLeg}) end
        if parts.LeftUpperLeg and parts.LeftLowerLeg then table.insert(connections, {parts.LeftUpperLeg, parts.LeftLowerLeg}) end
        if parts.LeftLowerLeg and parts.LeftFoot then table.insert(connections, {parts.LeftLowerLeg, parts.LeftFoot}) end
        if parts.LowerTorso and parts.RightUpperLeg then table.insert(connections, {parts.LowerTorso, parts.RightUpperLeg}) end
        if parts.RightUpperLeg and parts.RightLowerLeg then table.insert(connections, {parts.RightUpperLeg, parts.RightLowerLeg}) end
        if parts.RightLowerLeg and parts.RightFoot then table.insert(connections, {parts.RightLowerLeg, parts.RightFoot}) end
    elseif parts.Torso then
        if parts.Head and parts.Torso then table.insert(connections, {parts.Head, parts.Torso}) end
        local leftArm = character:FindFirstChild("Left Arm")
        local rightArm = character:FindFirstChild("Right Arm")
        local leftLeg = character:FindFirstChild("Left Leg")
        local rightLeg = character:FindFirstChild("Right Leg")
        if parts.Torso and leftArm then table.insert(connections, {parts.Torso, leftArm}) end
        if parts.Torso and rightArm then table.insert(connections, {parts.Torso, rightArm}) end
        if parts.Torso and leftLeg then table.insert(connections, {parts.Torso, leftLeg}) end
        if parts.Torso and rightLeg then table.insert(connections, {parts.Torso, rightLeg}) end
    end
    return connections
end

local function getESPColorForPlayer(player)
    if (aimbotEnabled or silentAimEnabled) and player == currentTarget then
        return Color3.new(1, 0, 0)
    end
    if isAlly(player) then return Color3.new(0, 1, 1) end
    if not teamCheckEnabled then return Color3.new(1, 1, 1) end
    if lplr.Team == nil or player.Team == nil then return Color3.new(1, 0, 0) end
    return (lplr.Team == player.Team) and Color3.new(0, 1, 0) or Color3.new(1, 0, 0)
end

local function updateESPForPlayer(player)
    if not espEnabled then clearPlayerESP(player) return end
    if isPlayerDead(player) then clearPlayerESP(player) return end
    local char = player.Character
    if not char then clearPlayerESP(player) return end
    local rootPart = char:FindFirstChild("HumanoidRootPart") or char:FindFirstChild("Torso")
    local headPart = char:FindFirstChild("Head")
    if not rootPart or not headPart then clearPlayerESP(player) return end
    local _, onScreen = camera:WorldToViewportPoint(rootPart.Position)
    local headPos = camera:WorldToViewportPoint(headPart.Position)
    local dist = (camera.CFrame.Position - rootPart.Position).Magnitude
    if not onScreen or dist > 2500 then clearPlayerESP(player) return end
    local humanoid = char:FindFirstChild("Humanoid")
    local health = humanoid and humanoid.Health or 0
    local maxHealth = humanoid and humanoid.MaxHealth or 100
    local healthPercent = math.clamp(health / maxHealth, 0, 1)

    local data = espDrawings[player]
    if not data then data = { box = nil, skeletonLines = {}, healthBar = nil, healthBg = nil }; espDrawings[player] = data end

    if not data.box then data.box = Drawing.new("Square"); data.box.Thickness = 2; data.box.Filled = false end
    local footPos = rootPart.Position - Vector3.new(0, 3, 0)
    local footScreen = camera:WorldToViewportPoint(footPos)
    local topY, bottomY = headPos.Y, footScreen.Y
    local height = bottomY - topY
    local width = height * 0.6
    local centerX = headPos.X
    data.box.Visible = true
    data.box.Color = getESPColorForPlayer(player)
    data.box.Position = Vector2.new(centerX - width/2, topY)
    data.box.Size = Vector2.new(width, height)

    local barWidth = 3
    local barX = centerX + width/2 + 2
    if not data.healthBg then
        data.healthBg = Drawing.new("Square")
        data.healthBg.Filled = true
        data.healthBg.Color = Color3.new(0.3, 0.3, 0.3)
    end
    data.healthBg.Visible = true
    data.healthBg.Size = Vector2.new(barWidth, height)
    data.healthBg.Position = Vector2.new(barX, topY)

    if not data.healthBar then
        data.healthBar = Drawing.new("Square")
        data.healthBar.Filled = true
    end
    local healthColor = Color3.new(1 - healthPercent, healthPercent, 0)
    data.healthBar.Visible = true
    data.healthBar.Color = healthColor
    data.healthBar.Size = Vector2.new(barWidth, height * healthPercent)
    data.healthBar.Position = Vector2.new(barX, topY + height * (1 - healthPercent))

    local connections = getSkeletonConnections(char)
    while #data.skeletonLines > #connections do table.remove(data.skeletonLines):Remove() end
    while #data.skeletonLines < #connections do
        local line = Drawing.new("Line"); line.Thickness = 2; line.Visible = true; table.insert(data.skeletonLines, line)
    end
    for i, conn in ipairs(connections) do
        local fromPart, toPart = conn[1], conn[2]
        if fromPart and toPart then
            local fromPos, fromVis = camera:WorldToViewportPoint(fromPart.Position)
            local toPos, toVis = camera:WorldToViewportPoint(toPart.Position)
            if fromVis and toVis then
                local line = data.skeletonLines[i]; line.Visible = true; line.Color = getESPColorForPlayer(player)
                line.From = Vector2.new(fromPos.X, fromPos.Y); line.To = Vector2.new(toPos.X, toPos.Y)
            else data.skeletonLines[i].Visible = false end
        else data.skeletonLines[i].Visible = false end
    end
end

local function clearAllESP()
    for player, _ in pairs(espDrawings) do clearPlayerESP(player) end
end

Players.PlayerRemoving:Connect(function(player)
    clearPlayerESP(player)
    if currentTarget == player then currentTarget = nil; targetAcquireTime = nil end
end)

local screenGui = Instance.new("ScreenGui")
screenGui.Name = "Ph7"
screenGui.ResetOnSpawn = false
screenGui.Parent = lplr:WaitForChild("PlayerGui")

local floatButton = Instance.new("ImageButton")
floatButton.Size = UDim2.new(0, 45, 0, 45)
floatButton.Position = UDim2.new(0, 10, 0, 10)
floatButton.Image = "rbxassetid://124447288284831"
floatButton.BackgroundColor3 = Color3.fromRGB(40, 0, 0)
floatButton.BackgroundTransparency = 0.3
floatButton.BorderSizePixel = 0
floatButton.ZIndex = 10
floatButton.Parent = screenGui
Instance.new("UICorner", floatButton).CornerRadius = UDim.new(0.5, 0)

floatButton.MouseEnter:Connect(function() floatButton.BackgroundTransparency = 0 end)
floatButton.MouseLeave:Connect(function() floatButton.BackgroundTransparency = 0.3 end)

local dragFloatStart, floatStartPos

floatButton.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        draggingFloat = true
        activeSlider = nil
        dragFloatStart = Vector2.new(mouse.X, mouse.Y)
        floatStartPos = floatButton.Position
    end
end)

floatButton.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        draggingFloat = false
    end
end)

RunService.RenderStepped:Connect(function()
    if draggingFloat then
        local delta = Vector2.new(mouse.X, mouse.Y) - dragFloatStart
        floatButton.Position = UDim2.new(floatStartPos.X.Scale, floatStartPos.X.Offset + delta.X, floatStartPos.Y.Scale, floatStartPos.Y.Offset + delta.Y)
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if activeSlider and input.UserInputType == Enum.UserInputType.MouseMovement then
        local trackAbsPos = activeSlider.track.AbsolutePosition.X
        local trackSize = activeSlider.track.AbsoluteSize.X
        local relativeX = math.clamp(mouse.X - trackAbsPos, 0, trackSize)
        local percent = relativeX / trackSize
        local newValue = activeSlider.min + percent * (activeSlider.max - activeSlider.min)
        newValue = math.clamp(math.floor(newValue / activeSlider.step + 0.5) * activeSlider.step, activeSlider.min, activeSlider.max)
        if newValue ~= activeSlider.value then
            activeSlider.value = newValue
            activeSlider.callback(newValue)
            activeSlider.updateUI(newValue)
        end
    end
end)

local mainFrame = Instance.new("Frame")
mainFrame.Size = UDim2.new(0, 320, 0, 270)
mainFrame.Position = UDim2.new(0.5, -160, 0.2, 0)
mainFrame.BackgroundColor3 = Color3.fromRGB(12, 0, 0)
mainFrame.BorderSizePixel = 0
mainFrame.Active = true
mainFrame.Draggable = true
mainFrame.Visible = false
mainFrame.Parent = screenGui
Instance.new("UICorner", mainFrame).CornerRadius = UDim.new(0, 12)

local titleBar = Instance.new("Frame")
titleBar.Size = UDim2.new(1, 0, 0, 36)
titleBar.BackgroundColor3 = Color3.fromRGB(20, 0, 0)
titleBar.BorderSizePixel = 0
titleBar.Parent = mainFrame
Instance.new("UICorner", titleBar).CornerRadius = UDim.new(0, 12)

local titleLabel = Instance.new("TextLabel")
titleLabel.Size = UDim2.new(1, -40, 1, 0)
titleLabel.Position = UDim2.new(0, 16, 0, 0)
titleLabel.BackgroundTransparency = 1
titleLabel.Text = "Ph7"
titleLabel.TextColor3 = Color3.fromRGB(255, 50, 50)
titleLabel.TextSize = 20
titleLabel.Font = Enum.Font.GothamBold
titleLabel.TextXAlignment = Enum.TextXAlignment.Left
titleLabel.Parent = titleBar

local closeBtn = Instance.new("TextButton")
closeBtn.Size = UDim2.new(0, 28, 0, 28)
closeBtn.Position = UDim2.new(1, -32, 0, 4)
closeBtn.Text = "X"
closeBtn.TextColor3 = Color3.new(1, 0.3, 0.3)
closeBtn.BackgroundTransparency = 1
closeBtn.TextSize = 18
closeBtn.Font = Enum.Font.GothamBold
closeBtn.Parent = titleBar
closeBtn.MouseButton1Click:Connect(function() mainFrame.Visible = false end)

local tabButtons = {}
local tabs = {"AIMBOT", "VISUAL", "MISC"}
local tabFrames = {}

local tabBar = Instance.new("Frame")
tabBar.Size = UDim2.new(1, 0, 0, 32)
tabBar.Position = UDim2.new(0, 0, 0, 36)
tabBar.BackgroundColor3 = Color3.fromRGB(16, 0, 0)
tabBar.BorderSizePixel = 0
tabBar.Parent = mainFrame

for i, name in ipairs(tabs) do
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0, 100, 1, -4)
    btn.Position = UDim2.new(0, 5 + (i-1)*105, 0, 2)
    btn.BackgroundColor3 = i == 1 and Color3.fromRGB(180, 0, 0) or Color3.fromRGB(35, 0, 0)
    btn.Text = name
    btn.TextColor3 = Color3.new(1, 1, 1)
    btn.TextSize = 12
    btn.Font = Enum.Font.GothamBold
    btn.BorderSizePixel = 0
    btn.Parent = tabBar
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 6)
    table.insert(tabButtons, btn)

    local frame = Instance.new("ScrollingFrame")
    frame.Size = UDim2.new(1, -10, 1, -74)
    frame.Position = UDim2.new(0, 5, 0, 70)
    frame.BackgroundTransparency = 1
    frame.BorderSizePixel = 0
    frame.CanvasSize = UDim2.new(0, 0, 0, 0)
    frame.ScrollBarThickness = 4
    frame.ScrollingEnabled = true
    frame.AutomaticCanvasSize = Enum.AutomaticSize.Y
    frame.Visible = (i == 1)
    frame.Parent = mainFrame
    table.insert(tabFrames, frame)

    local layout = Instance.new("UIListLayout")
    layout.Padding = UDim.new(0, 6)
    layout.HorizontalAlignment = Enum.HorizontalAlignment.Center
    layout.SortOrder = Enum.SortOrder.LayoutOrder
    layout.Parent = frame
    Instance.new("UIPadding", frame).PaddingTop = UDim.new(0, 6)
    Instance.new("UIPadding", frame).PaddingBottom = UDim.new(0, 6)

    btn.MouseButton1Click:Connect(function()
        for j, b in ipairs(tabButtons) do
            b.BackgroundColor3 = Color3.fromRGB(35, 0, 0)
            tabFrames[j].Visible = false
        end
        btn.BackgroundColor3 = Color3.fromRGB(180, 0, 0)
        tabFrames[i].Visible = true
    end)
end

floatButton.MouseButton1Click:Connect(function() mainFrame.Visible = not mainFrame.Visible end)

local function createToggle(frame, text, callback)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0, 290, 0, 30)
    btn.BackgroundColor3 = Color3.fromRGB(30, 0, 0)
    btn.Text = "  " .. text .. ": OFF"
    btn.TextColor3 = Color3.new(0.9, 0.9, 0.9)
    btn.TextSize = 12
    btn.Font = Enum.Font.Gotham
    btn.TextXAlignment = Enum.TextXAlignment.Left
    btn.BorderSizePixel = 0
    btn.Parent = frame
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 6)

    local state = false
    btn.MouseButton1Click:Connect(function()
        state = not state
        btn.Text = "  " .. text .. ": " .. (state and "ON" or "OFF")
        btn.BackgroundColor3 = state and Color3.fromRGB(120, 0, 0) or Color3.fromRGB(30, 0, 0)
        callback(state)
    end)
    return btn
end

local function createSlider(frame, text, min, max, default, step, callback, formatStr, bigStep)
    bigStep = bigStep or step * 10
    formatStr = formatStr or "%d"
    local container = Instance.new("Frame")
    container.Size = UDim2.new(0, 290, 0, 46)
    container.BackgroundColor3 = Color3.fromRGB(25, 0, 0)
    container.BorderSizePixel = 0
    container.Parent = frame
    Instance.new("UICorner", container).CornerRadius = UDim.new(0, 8)

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, -10, 0, 16)
    label.Position = UDim2.new(0, 8, 0, 4)
    label.BackgroundTransparency = 1
    label.Text = text .. ": " .. string.format(formatStr, default)
    label.TextColor3 = Color3.new(200, 200, 200)
    label.TextSize = 11
    label.Font = Enum.Font.Gotham
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = container

    local btnMinus = Instance.new("TextButton")
    btnMinus.Size = UDim2.new(0, 22, 0, 18)
    btnMinus.Position = UDim2.new(0, 10, 0, 24)
    btnMinus.Text = "-"
    btnMinus.TextColor3 = Color3.new(1,1,1)
    btnMinus.BackgroundColor3 = Color3.fromRGB(40, 0, 0)
    btnMinus.BorderSizePixel = 0
    btnMinus.Font = Enum.Font.GothamBold
    btnMinus.TextSize = 14
    btnMinus.Parent = container
    Instance.new("UICorner", btnMinus).CornerRadius = UDim.new(0, 4)

    local track = Instance.new("Frame")
    track.Size = UDim2.new(1, -76, 0, 6)
    track.Position = UDim2.new(0, 38, 0, 30)
    track.BackgroundColor3 = Color3.fromRGB(60, 0, 0)
    track.BorderSizePixel = 0
    track.Active = true
    track.Parent = container
    Instance.new("UICorner", track).CornerRadius = UDim.new(1, 0)

    local fill = Instance.new("Frame")
    fill.Size = UDim2.new(0, 0, 1, 0)
    fill.BackgroundColor3 = Color3.fromRGB(180, 0, 0)
    fill.BorderSizePixel = 0
    fill.Parent = track
    Instance.new("UICorner", fill).CornerRadius = UDim.new(1, 0)

    local knob = Instance.new("TextButton")
    knob.Size = UDim2.new(0, 16, 0, 16)
    knob.Position = UDim2.new(0, 0, 0, -5)
    knob.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    knob.Text = ""
    knob.BorderSizePixel = 0
    knob.Parent = track
    Instance.new("UICorner", knob).CornerRadius = UDim.new(1, 0)

    local btnPlus = Instance.new("TextButton")
    btnPlus.Size = UDim2.new(0, 22, 0, 18)
    btnPlus.Position = UDim2.new(1, -32, 0, 24)
    btnPlus.Text = "+"
    btnPlus.TextColor3 = Color3.new(1,1,1)
    btnPlus.BackgroundColor3 = Color3.fromRGB(40, 0, 0)
    btnPlus.BorderSizePixel = 0
    btnPlus.Font = Enum.Font.GothamBold
    btnPlus.TextSize = 14
    btnPlus.Parent = container
    Instance.new("UICorner", btnPlus).CornerRadius = UDim.new(0, 4)

    local sliderData = {
        track = track,
        min = min,
        max = max,
        step = step,
        bigStep = bigStep,
        value = default,
        callback = callback,
        updateUI = function(val)
            local percent = (val - min) / (max - min)
            local maxOffset = track.AbsoluteSize.X - knob.AbsoluteSize.X
            knob.Position = UDim2.new(0, percent * maxOffset, 0, -5)
            fill.Size = UDim2.new(percent, 0, 1, 0)
            label.Text = text .. ": " .. string.format(formatStr, val)
        end
    }

    sliderData.updateUI(default)

    local function setValue(newValue)
        newValue = math.clamp(math.floor(newValue / step + 0.5) * step, min, max)
        if newValue ~= sliderData.value then
            sliderData.value = newValue
            callback(newValue)
            sliderData.updateUI(newValue)
        end
    end

    btnMinus.MouseButton1Click:Connect(function() setValue(sliderData.value - bigStep) end)
    btnPlus.MouseButton1Click:Connect(function() setValue(sliderData.value + bigStep) end)

    local function updateFromMouse()
        local trackAbsPos = sliderData.track.AbsolutePosition.X
        local trackSize = sliderData.track.AbsoluteSize.X
        local relativeX = math.clamp(mouse.X - trackAbsPos, 0, trackSize)
        local percent = relativeX / trackSize
        local newValue = sliderData.min + percent * (sliderData.max - sliderData.min)
        setValue(newValue)
    end

    knob.MouseButton1Down:Connect(function()
        activeSlider = sliderData
        draggingFloat = false
        updateFromMouse()
    end)
    track.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            activeSlider = sliderData
            draggingFloat = false
            updateFromMouse()
        end
    end)

    return container
end

local aimFrame = tabFrames[1]
createToggle(aimFrame, "Aimbot", function(on)
    aimbotEnabled = on
    if not on then currentTarget = nil; targetAcquireTime = nil end
end)
createSlider(aimFrame, "Suavidade", 0, 1, aimbotSmoothness, 0.01, function(v) aimbotSmoothness = v end, "%.2f", 0.05)

local targetBtn = Instance.new("TextButton")
targetBtn.Size = UDim2.new(0, 290, 0, 30)
targetBtn.BackgroundColor3 = Color3.fromRGB(30, 0, 0)
targetBtn.Text = "  Modo: RAGE"
targetBtn.TextColor3 = Color3.new(0.9, 0.9, 0.9)
targetBtn.TextSize = 12
targetBtn.Font = Enum.Font.Gotham
targetBtn.TextXAlignment = Enum.TextXAlignment.Left
targetBtn.BorderSizePixel = 0
targetBtn.Parent = aimFrame
Instance.new("UICorner", targetBtn).CornerRadius = UDim.new(0, 6)
targetBtn.MouseButton1Click:Connect(function()
    if aimbotTarget == "Head" then
        aimbotTarget = "Chest"
        targetBtn.Text = "  Modo: LEGIT"
    else
        aimbotTarget = "Head"
        targetBtn.Text = "  Modo: RAGE"
    end
    currentTarget = nil
    targetAcquireTime = nil
end)

createSlider(aimFrame, "Tempo Legit", 0, 3000, legitHoldTime, 50, function(v) legitHoldTime = v end, "%d", 100)
createToggle(aimFrame, "Silent Aim", function(on) silentAimEnabled = on end)

local visFrame = tabFrames[2]
createToggle(visFrame, "ESP", function(on) espEnabled = on; if not on then clearAllESP() end end)
createToggle(visFrame, "FOV", function(on) fovEnabled = on; updateFOVCircle() end)
createSlider(visFrame, "Raio FOV", 0, 400, fovRadius, 1, function(v) fovRadius = v; updateFOVCircle() end, "%d", 10)

local alliesBtn = Instance.new("TextButton")
alliesBtn.Size = UDim2.new(0, 290, 0, 30)
alliesBtn.BackgroundColor3 = Color3.fromRGB(30, 0, 0)
alliesBtn.Text = "  Gerenciar Aliados"
alliesBtn.TextColor3 = Color3.new(0.9, 0.9, 0.9)
alliesBtn.TextSize = 12
alliesBtn.Font = Enum.Font.Gotham
alliesBtn.TextXAlignment = Enum.TextXAlignment.Left
alliesBtn.BorderSizePixel = 0
alliesBtn.Parent = visFrame
Instance.new("UICorner", alliesBtn).CornerRadius = UDim.new(0, 6)
alliesBtn.MouseButton1Click:Connect(function() toggleAlliesMenu() end)

createToggle(visFrame, "Team Check", function(on) teamCheckEnabled = on end)

local miscFrame = tabFrames[3]
createToggle(miscFrame, "SpinBot", function(on) spinBotEnabled = on end)
createSlider(miscFrame, "Vel. Spin", 1, 30, spinBotSpeed, 1, function(v) spinBotSpeed = v end, "%d", 2)
createToggle(miscFrame, "Velocidade", function(on)
    speedEnabled = on
    if not on and lplr.Character then
        local humanoid = lplr.Character:FindFirstChild("Humanoid")
        if humanoid then humanoid.WalkSpeed = 16 end
    end
end)
createSlider(miscFrame, "Valor Vel.", 16, 100, currentSpeed, 1, function(v)
    currentSpeed = v
    if speedEnabled and lplr.Character then
        local humanoid = lplr.Character:FindFirstChild("Humanoid")
        if humanoid and humanoid.Health > 0 then humanoid.WalkSpeed = v end
    end
end, "%d", 5)
createToggle(miscFrame, "NoClip", function(on) noclipEnabled = on; setNoclip(on) end)
createToggle(miscFrame, "No Recoil", function(on) noRecoilEnabled = on end)

function toggleAlliesMenu()
    local existing = screenGui:FindFirstChild("AlliesSubmenu")
    if existing then existing:Destroy(); return end

    local sub = Instance.new("Frame")
    sub.Name = "AlliesSubmenu"
    sub.Size = UDim2.new(0, 200, 0, 130)
    sub.Position = UDim2.new(0.5, -100, 0.5, -65)
    sub.BackgroundColor3 = Color3.fromRGB(20, 0, 0)
    sub.BorderSizePixel = 0
    sub.Draggable = true
    sub.Active = true
    sub.Parent = screenGui
    Instance.new("UICorner", sub).CornerRadius = UDim.new(0, 10)

    local titleBar = Instance.new("TextLabel")
    titleBar.Size = UDim2.new(1, 0, 0, 22)
    titleBar.BackgroundColor3 = Color3.fromRGB(28, 0, 0)
    titleBar.Text = "Aliados"
    titleBar.TextColor3 = Color3.new(1,1,1)
    titleBar.TextSize = 11
    titleBar.Font = Enum.Font.GothamBold
    titleBar.Parent = sub
    Instance.new("UICorner", titleBar).CornerRadius = UDim.new(0, 10)

    local closeSub = Instance.new("TextButton")
    closeSub.Size = UDim2.new(0, 22, 0, 22)
    closeSub.Position = UDim2.new(1, -24, 0, 0)
    closeSub.Text = "X"
    closeSub.TextColor3 = Color3.new(1,0,0)
    closeSub.BackgroundTransparency = 1
    closeSub.TextSize = 13
    closeSub.Font = Enum.Font.GothamBold
    closeSub.Parent = sub
    closeSub.MouseButton1Click:Connect(function() sub:Destroy() end)

    local playerListFrame = Instance.new("ScrollingFrame")
    playerListFrame.Size = UDim2.new(0, 180, 0, 86)
    playerListFrame.Position = UDim2.new(0.5, -90, 0, 26)
    playerListFrame.BackgroundColor3 = Color3.fromRGB(22, 0, 0)
    playerListFrame.BorderSizePixel = 0
    playerListFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
    playerListFrame.ScrollBarThickness = 4
    playerListFrame.ScrollingEnabled = true
    playerListFrame.AutomaticCanvasSize = Enum.AutomaticSize.Y
    playerListFrame.Parent = sub
    Instance.new("UICorner", playerListFrame).CornerRadius = UDim.new(0, 6)

    local function refreshUI()
        for _, child in ipairs(playerListFrame:GetChildren()) do
            if child:IsA("Frame") then child:Destroy() end
        end
        local yOffset = 3
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= lplr then
                local item = Instance.new("Frame")
                item.Size = UDim2.new(1, -8, 0, 22)
                item.Position = UDim2.new(0, 4, 0, yOffset)
                item.BackgroundColor3 = Color3.fromRGB(35, 0, 0)
                item.Parent = playerListFrame
                Instance.new("UICorner", item).CornerRadius = UDim.new(0, 4)

                local nameLabel = Instance.new("TextLabel")
                nameLabel.Size = UDim2.new(0, 100, 1, 0)
                nameLabel.Position = UDim2.new(0, 4, 0, 0)
                nameLabel.BackgroundTransparency = 1
                nameLabel.Text = player.Name
                nameLabel.TextColor3 = Color3.new(1,1,1)
                nameLabel.TextXAlignment = Enum.TextXAlignment.Left
                nameLabel.TextSize = 9
                nameLabel.Font = Enum.Font.Gotham
                nameLabel.Parent = item

                local allyFlag = isAlly(player)
                local toggleBtn = Instance.new("TextButton")
                toggleBtn.Size = UDim2.new(0, 48, 1, -2)
                toggleBtn.Position = UDim2.new(1, -50, 0, 1)
                toggleBtn.Text = allyFlag and "Rem." or "Add"
                toggleBtn.BackgroundColor3 = allyFlag and Color3.fromRGB(100, 0, 0) or Color3.fromRGB(0, 80, 0)
                toggleBtn.TextColor3 = Color3.new(1,1,1)
                toggleBtn.TextSize = 9
                toggleBtn.Font = Enum.Font.GothamBold
                toggleBtn.Parent = item
                Instance.new("UICorner", toggleBtn).CornerRadius = UDim.new(0, 4)

                toggleBtn.MouseButton1Click:Connect(function()
                    if allyFlag then
                        for i, name in ipairs(alliesList) do if name == player.Name then table.remove(alliesList, i); break end end
                    else
                        table.insert(alliesList, player.Name)
                    end
                    refreshUI()
                end)
                yOffset += 25
            end
        end
        playerListFrame.CanvasSize = UDim2.new(0, 0, 0, yOffset + 6)
    end
    refreshUI()
end

createFOVCircle()

-- Flick no Stepped (antes da física)
RunService.Stepped:Connect(function()
    if flickRequested and flickTargetPos then
        silentAimOriginalCFrame = camera.CFrame
        camera.CFrame = CFrame.lookAt(camera.CFrame.Position, flickTargetPos)
        silentAimActive = true
        flickRequested = false
    end
end)

RunService.RenderStepped:Connect(function()
    -- arrasto do botão flutuante tratado em outra conexão, não aqui
    for _, v in pairs(Players:GetPlayers()) do if v ~= lplr then updateESPForPlayer(v) end end

    if currentTarget and isPlayerDead(currentTarget) then
        currentTarget = nil
        targetAcquireTime = nil
    end

    if noRecoilEnabled and isShootingPressed() then
        local char = lplr.Character
        if char then
            for _, tool in ipairs(char:GetChildren()) do
                if tool:IsA("Tool") then
                    pcall(function()
                        tool.Recoil = 0
                        tool.RecoilX = 0
                        tool.RecoilY = 0
                        tool.RecoilAmount = 0
                        tool.RecoilStrength = 0
                        tool.RecoilAngle = 0
                        tool.RecoilRadius = 0
                        tool.RecoilPunch = 0
                        tool.RecoilRecovery = 0
                        tool.RecoilResetTime = 0
                        tool.RecoilSpeed = 0
                        tool.MaxRecoil = 0
                        tool.MinRecoil = 0
                        tool.Spread = 0
                        tool.MinSpread = 0
                        tool.MaxSpread = 0
                        tool.CurrentSpread = 0
                        tool.SpreadValue = 0
                        tool.Accuracy = 100
                        tool.BulletSpread = 0
                        tool.CameraRecoil = 0
                        tool.CameraRecoilX = 0
                        tool.CameraRecoilY = 0
                    end)
                end
            end
            local humanoid = char:FindFirstChild("Humanoid")
            if humanoid then
                humanoid.CameraOffset = Vector3.new(0, 0, 0)
            end
        end
        if camera:FindFirstChild("CameraRecoil") then
            camera.CameraRecoil = 0
        end
    end

    if noclipEnabled and lplr.Character then setNoclip(true) end

    if aimbotEnabled and not silentAimEnabled and (isShootingPressed() or isAimingPressed()) then
        local targetPos, _ = getBestTarget()
        if targetPos then
            local targetCF = CFrame.lookAt(camera.CFrame.Position, targetPos)
            camera.CFrame = camera.CFrame:Lerp(targetCF, aimbotSmoothness)
        end
    end

    -- restaura câmera após flick
    if silentAimActive and silentAimOriginalCFrame then
        camera.CFrame = silentAimOriginalCFrame
        silentAimActive = false
        silentAimOriginalCFrame = nil
        flickTargetPos = nil
    end

    -- linha indicadora
    if aimbotEnabled or silentAimEnabled then
        local targetPos, _ = getBestTarget()
        if targetPos then
            if not aimIndicatorLine then
                aimIndicatorLine = Drawing.new("Line")
                aimIndicatorLine.Thickness = 1
                aimIndicatorLine.Color = Color3.fromRGB(0, 150, 255)
            end
            aimIndicatorLine.From = Vector2.new(camera.ViewportSize.X/2, camera.ViewportSize.Y/2)
            local screenPos = camera:WorldToViewportPoint(targetPos)
            aimIndicatorLine.To = Vector2.new(screenPos.X, screenPos.Y)
            aimIndicatorLine.Visible = true
        else
            if aimIndicatorLine then
                aimIndicatorLine:Remove()
                aimIndicatorLine = nil
            end
        end
    else
        if aimIndicatorLine then
            aimIndicatorLine:Remove()
            aimIndicatorLine = nil
        end
    end

    if speedEnabled and lplr.Character then
        local humanoid = lplr.Character:FindFirstChild("Humanoid")
        if humanoid and humanoid.Health > 0 then humanoid.WalkSpeed = currentSpeed end
    end

    updateFOVCircle()
    updateSpinBot()
end)

screenGui.AncestryChanged:Connect(function()
    if not screenGui.Parent then
        if fovCircle then fovCircle:Remove() end
        if aimIndicatorLine then aimIndicatorLine:Remove() end
        clearAllESP()
    end
end)
