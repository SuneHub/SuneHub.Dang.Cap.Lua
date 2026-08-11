local RS = game:GetService("RunService")
local UIS = game:GetService("UserInputService")
local Players = game:GetService("Players")
local VirtualUser = game:GetService("VirtualUser")
local StatsService = game:GetService("Stats")
local P = Players.LocalPlayer
local Camera = workspace.CurrentCamera
local Mouse = P:GetMouse()

local AP_Normal = nil
local Target = nil
local IgnoreTarget = nil
local lastTime = tick()
local startTime = tick()
local frameCount = 0

local State = {
    Anchor = false,
    Noclip = false,
    InfJump = false,
    AutoReset = false,
    ESP = false,
    AntiLag = false,
    AntiAFK = false,
    CameraLock = false,
    AimbotEnabled = false,
    WallCheck = false,
    FOVSize = 130
}


local originalCache = {}

local function setAntiLag(state)
    local Lighting = game:GetService("Lighting")
    local Terrain = workspace:FindFirstChildOfClass("Terrain")

    if state then
        if originalCache.GlobalShadows == nil then 
            originalCache.GlobalShadows = Lighting.GlobalShadows 
        end
        Lighting.GlobalShadows = false
        Lighting.FogEnd = 9e9

        for _, effect in pairs(Lighting:GetChildren()) do
            if effect:IsA("PostEffect") then
                if originalCache[effect] == nil then originalCache[effect] = effect.Enabled end
                effect.Enabled = false
            end
        end

        if Terrain then
            if originalCache.WaterWaveSize == nil then originalCache.WaterWaveSize = Terrain.WaterWaveSize end
            Terrain.WaterWaveSize = 0
            Terrain.WaterWaveSpeed = 0
        end

        for _, v in pairs(workspace:GetDescendants()) do
            if v:IsA("ParticleEmitter") or v:IsA("Trail") or v:IsA("Fire") or v:IsA("Smoke") or v:IsA("Sparkles") or v:IsA("Light") then
                if originalCache[v] == nil then originalCache[v] = v.Enabled end
                v.Enabled = false
            elseif v:IsA("BasePart") then
                if originalCache[v] == nil then
                    originalCache[v] = { Material = v.Material, CastShadow = v.CastShadow }
                end
                v.Material = Enum.Material.SmoothPlastic
                v.CastShadow = false
            elseif v:IsA("Decal") or v:IsA("Texture") then
                if originalCache[v] == nil then originalCache[v] = v.Transparency end
                v.Transparency = 1
            end
        end
    else
        if originalCache.GlobalShadows ~= nil then 
            Lighting.GlobalShadows = originalCache.GlobalShadows 
        end

        for _, effect in pairs(Lighting:GetChildren()) do
            if originalCache[effect] ~= nil then effect.Enabled = originalCache[effect] end
        end

        if Terrain and originalCache.WaterWaveSize ~= nil then
            Terrain.WaterWaveSize = originalCache.WaterWaveSize
            Terrain.WaterWaveSpeed = 10
        end

        for v, data in pairs(originalCache) do
            if typeof(v) == "Instance" and v.Parent then
                if type(data) == "table" then
                    v.Material = data.Material
                    v.CastShadow = data.CastShadow
                elseif typeof(data) == "boolean" then
                    v.Enabled = data
                elseif type(data) == "number" then
                    v.Transparency = data
                end
            end
        end
    end
end

local function cleanupESP()
    for _, plr in pairs(Players:GetPlayers()) do
        if plr.Character then
            if plr.Character:FindFirstChild("SuneAura") then plr.Character.SuneAura:Destroy() end
            if plr.Character:FindFirstChild("Head") and plr.Character.Head:FindFirstChild("SuneESP_Gui") then 
                plr.Character.Head.SuneESP_Gui:Destroy() 
            end
        end
    end
end

local function updateESP()
    if not State.ESP then return end
    for _, plr in pairs(Players:GetPlayers()) do
        if plr ~= P and plr.Character and plr.Character:FindFirstChild("Head") then
            local char = plr.Character
            if not char:FindFirstChild("SuneAura") then
                local aura = Instance.new("Highlight", char)
                aura.Name = "SuneAura"
                aura.FillColor = Color3.new(1,1,1)
                aura.OutlineColor = Color3.new(1,1,1)
                aura.FillTransparency = 0.8
                aura.OutlineTransparency = 0.5
            end
            if not char.Head:FindFirstChild("SuneESP_Gui") then
                local gui = Instance.new("BillboardGui", char.Head)
                gui.Name = "SuneESP_Gui"
                gui.Size = UDim2.new(0, 100, 0, 50)
                gui.StudsOffset = Vector3.new(0, 2, 0)
                gui.AlwaysOnTop = true
                
                local txt = Instance.new("TextLabel", gui)
                txt.Size = UDim2.new(1, 0, 1, 0)
                txt.BackgroundTransparency = 1
                txt.TextColor3 = Color3.new(1,1,1)
                txt.TextStrokeTransparency = 0
                txt.Font = Enum.Font.Code
                txt.TextSize = 12
                txt.Name = "Info"
            end
            local hum = char:FindFirstChild("Humanoid")
            if hum then 
                char.Head.SuneESP_Gui.Info.Text = plr.Name.."\nHP: "..math.floor(hum.Health).."\nSPD: "..hum.WalkSpeed.." | JMP: "..hum.JumpPower 
            end
        end
    end
end

local function makeDraggable(dragHandle, frameToMove)
    frameToMove = frameToMove or dragHandle
    local dragging, dragStart, startPos
    dragHandle.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true; dragStart = input.Position; startPos = frameToMove.Position
        end
    end)
    UIS.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            local delta = input.Position - dragStart
            frameToMove.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)
    UIS.InputEnded:Connect(function(input) 
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then 
            dragging = false 
        end 
    end)
end

local guiHolder = (game:GetService("CoreGui") or P:WaitForChild("PlayerGui"))
local sg = Instance.new("ScreenGui", guiHolder)
sg.Name = "SuneHub_Extras"
sg.IgnoreGuiInset = true
sg.ResetOnSpawn = false

local statsContainer = Instance.new("Frame", sg)
statsContainer.Size = UDim2.new(0, 200, 0, 28)
statsContainer.Position = UDim2.new(1, -10, 0, 10)
statsContainer.AnchorPoint = Vector2.new(1, 0)
statsContainer.BackgroundTransparency = 1

local layout = Instance.new("UIListLayout", statsContainer)
layout.FillDirection = Enum.FillDirection.Horizontal
layout.HorizontalAlignment = Enum.HorizontalAlignment.Right
layout.VerticalAlignment = Enum.VerticalAlignment.Center
layout.Padding = UDim.new(0, 5)

local function createPillBox(txt, width)
    local f = Instance.new("Frame", statsContainer)
    f.Size = UDim2.new(0, width, 1, 0)
    f.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    f.BackgroundTransparency = 0.14
    f.BorderSizePixel = 0
    Instance.new("UICorner", f).CornerRadius = UDim.new(1, 0)
    
    local t = Instance.new("TextLabel", f)
    t.Size = UDim2.new(1, 0, 1, 0)
    t.BackgroundTransparency = 1
    t.TextColor3 = Color3.new(1, 1, 1)
    t.Font = Enum.Font.GothamBold
    t.TextSize = 10
    t.Text = txt
    return t
end

local fpsLabel = createPillBox("FPS: 0", 55)
local pingLabel = createPillBox("MS: 0", 55)
local timeLabel = createPillBox("00:00:00", 75)

local repo = "https://raw.githubusercontent.com/deividcomsono/Obsidian/main/"
local Library = loadstring(game:HttpGet(repo .. "Library.lua"))()
local ThemeManager = loadstring(game:HttpGet(repo .. "addons/ThemeManager.lua"))()
local SaveManager = loadstring(game:HttpGet(repo .. "addons/SaveManager.lua"))()

local Options = Library.Options
local Toggles = Library.Toggles

local Window = Library:CreateWindow({
    Title = "SuneHub v1.03",
    Footer = "Pepsi Pepsi",
    Icon = "crown",
    NotifySide = "Right",
    ShowCustomCursor = true,
})

local Tabs = {
    Movement = Window:AddTab("Movement", "menu"),
    Visual   = Window:AddTab("Visual", "eye"),
    Utility  = Window:AddTab("Utility", "wrench"),
    Camera   = Window:AddTab("Camera", "camera"),
    ["UI Settings"] = Window:AddTab("UI Settings", "settings"),
}

local MovementGroup = Tabs.Movement:AddLeftGroupbox("Movement Controls")
local VisualGroup   = Tabs.Visual:AddLeftGroupbox("Visual Settings")
local UtilityGroup  = Tabs.Utility:AddLeftGroupbox("Utility Features")
local CameraGroup   = Tabs.Camera:AddLeftGroupbox("Aimlock & Camera")

MovementGroup:AddToggle("Anchor", {
    Text = "Anchor",
    Default = false,
    Callback = function(Value)
        State.Anchor = Value
        if State.Anchor then
            AP_Normal = P.Character and P.Character:FindFirstChild("HumanoidRootPart") and P.Character.HumanoidRootPart.CFrame or nil
        else
            AP_Normal = nil
        end
    end,
})

MovementGroup:AddToggle("AutoReset", {
    Text = "Ragdoll Reset",
    Default = false,
    Callback = function(Value)
        State.AutoReset = Value
    end,
})

MovementGroup:AddToggle("Noclip", {
    Text = "Noclip",
    Default = false,
    Callback = function(Value)
        State.Noclip = Value
        if not State.Noclip and P.Character then
            for _, v in pairs(P.Character:GetDescendants()) do
                if v:IsA("BasePart") then v.CanCollide = true end
            end
        end
    end,
})

MovementGroup:AddToggle("InfJump", {
    Text = "Infinite Jump",
    Default = false,
    Callback = function(Value)
        State.InfJump = Value
    end,
})

MovementGroup:AddToggle("AntiAFK", {
    Text = "Anti AFK",
    Default = false,
    Callback = function(Value)
        State.AntiAFK = Value
    end,
})

MovementGroup:AddButton({
    Text = "Load SuneHub Fly",
    Func = function()
        loadstring(game:HttpGet("https://raw.githubusercontent.com/SuneHub/SuneHub.Dang.Cap.Lua/refs/heads/main/SuneHub.Fly.lua"))()
            
    end,
})

local speedValue = 16
local jumpValue = 50

local SpeedGroup = Tabs.Movement:AddRightGroupbox("Walk")
SpeedGroup:AddSlider("SpeedSlider", {
    Text = "Walk Speed",
    Default = 16,
    Min = 1,
    Max = 500,
    Rounding = 0,
    Compact = false,
    Callback = function(Value)
        speedValue = Value
    end,
})

local JumpGroup = Tabs.Movement:AddRightGroupbox("Jump")
JumpGroup:AddSlider("JumpSlider", {
    Text = "Jump Power",
    Default = 50,
    Min = 1,
    Max = 500,
    Rounding = 0,
    Compact = false,
    Callback = function(Value)
        jumpValue = Value
    end,
})

game:GetService("RunService").Stepped:Connect(function()
    local char = game.Players.LocalPlayer.Character
    if char and char:FindFirstChildOfClass("Humanoid") then
        local hum = char:FindFirstChildOfClass("Humanoid")
        hum.WalkSpeed = speedValue
        hum.UseJumpPower = true
        hum.JumpPower = jumpValue
    end
end)

game:GetService("RunService").Stepped:Connect(function()
    local char = game.Players.LocalPlayer.Character
    if char and char:FindFirstChildOfClass("Humanoid") then
        char:FindFirstChildOfClass("Humanoid").WalkSpeed = speedValue
    end
end)

VisualGroup:AddToggle("ESP", {
    Text = "ESP",
    Default = false,
    Callback = function(Value)
        State.ESP = Value
        if not State.ESP then cleanupESP() end
    end,
})

VisualGroup:AddToggle("AntiLag", {
    Text = "Anti Lag",
    Default = false,
    Callback = function(Value)
        setAntiLag(Value)
    end,
})

local TargetName = ""
local TargetESPEnabled = false

local TargetESPGroup = Tabs.Visual:AddRightGroupbox("Target ESP")

TargetESPGroup:AddInput("TargetNameInput", {
    Text = "Name Target",
    Default = "",
    Placeholder = "Tên hoặc DisplayName...",
    Numeric = false,
    Finished = false,
    Callback = function(Value)
        TargetName = Value
    end,
})

TargetESPGroup:AddToggle("TargetESPToggle", {
    Text = "Bật ESP Mục tiêu",
    Default = false,
    Callback = function(Value)
        TargetESPEnabled = Value
        if not Value then
            for _, player in pairs(game.Players:GetPlayers()) do
                if player.Character then
                    if player.Character:FindFirstChild("TargetHighlight") then
                        player.Character.TargetHighlight:Destroy()
                    end
                    if player.Character:FindFirstChild("Head") and player.Character.Head:FindFirstChild("TargetESP_Gui") then
                        player.Character.Head.TargetESP_Gui:Destroy()
                    end
                end
            end
        end
    end,
})


local targetName = ""

UtilityGroup:AddInput("TeleportInput", {
    Text = "Nhập tên người chơi",
    Default = "",
    Placeholder = "Name...",
    Numeric = false,
    Finished = false,
    Callback = function(Value)
        targetName = Value
    end,
})

UtilityGroup:AddToggle("AutoTele", {
    Text = "Auto Teleport",
    Default = false,
    Callback = function(Value)
        State.AutoTele = Value
    end,
})

UtilityGroup:AddButton({
    Text = "Dịch chuyển (Teleport)",
    Func = function()
        if targetName == "" then return end
        
        local LocalPlayer = game.Players.LocalPlayer
        local char = LocalPlayer.Character
        if not char or not char:FindFirstChild("HumanoidRootPart") then return end

        local search = targetName:lower()
        for _, target in pairs(game.Players:GetPlayers()) do
            if target ~= LocalPlayer then
                local pName = target.Name:lower()
                local dName = target.DisplayName:lower()
                
                if pName:find(search, 1, true) or dName:find(search, 1, true) then
                    if target.Character and target.Character:FindFirstChild("HumanoidRootPart") then
                        char.HumanoidRootPart.CFrame = target.Character.HumanoidRootPart.CFrame
                        break
                    end
                end
            end
        end
    end,
})

local SelectedTargetPlayer = nil
local PlayerMap = {}
local LastPlayerSig = ""
local lastPlayerCheck = 0

local SelectedTargetPlayer = nil
local PlayerMap = {}
local LastPlayerSig = ""
local FilterText = ""
local lastPlayerCheck = 0

CameraGroup:AddToggle("CameraLock", {
    Text = "Camera Lock (Aimlock)",
    Default = false,
    Callback = function(Value)
        State.CameraLock = Value
    end
})

CameraGroup:AddInput("SearchInput", {
    Text = "Put Name:",
    Callback = function(Value)
        FilterText = Value:lower()
    end
})

local TargetDropdown = CameraGroup:AddDropdown("TargetSelectDropdown", {
    Values = {"Đang tải..."},
    Default = 1,
    Multi = false,
    Text = "Choose Target",
    Callback = function(Value)
        if PlayerMap[Value] then
            SelectedTargetPlayer = PlayerMap[Value]
        else
            SelectedTargetPlayer = nil
        end
    end,
})

local function CheckAndUpdatePlayerList()
    local list = {}
    local newMap = {}
    local sig = FilterText

    for _, plr in pairs(game.Players:GetPlayers()) do
        if plr ~= game.Players.LocalPlayer then
            local playerName = plr.Name:lower()
            local displayName = plr.DisplayName:lower()
            
            if FilterText == "" or playerName:find(FilterText, 1, true) or displayName:find(FilterText, 1, true) then
                local label = plr.DisplayName .. " (@" .. plr.Name .. ")"
                table.insert(list, label)
                newMap[label] = plr
                sig = sig .. plr.Name .. ";"
            end
        end
    end

    if sig ~= LastPlayerSig then
        LastPlayerSig = sig
        PlayerMap = newMap

        if #list == 0 then
            table.insert(list, "No one")
        end

        TargetDropdown:SetValues(list)

        if SelectedTargetPlayer and not SelectedTargetPlayer:IsDescendantOf(game.Players) then
            SelectedTargetPlayer = nil
        end
    end
end

local AimBotGroup = Tabs.Camera:AddRightGroupbox("Aimbot Settings")

AimBotGroup:AddToggle("AimbotToggle", {
    Text = "Bật Aimbot",
    Default = false,
    Callback = function(Value)
        State.AimbotEnabled = Value
    end
})

AimBotGroup:AddToggle("WallCheck", {
    Text = "Kiểm tra tường (Wall Check)",
    Default = false,
    Callback = function(Value)
        State.WallCheck = Value
    end
})

AimBotGroup:AddToggle("TeamCheck", {
    Text = "Kiểm tra đồng đội (Team Check)",
    Default = false,
    Callback = function(Value)
        State.TeamCheck = Value
    end
})

AimBotGroup:AddSlider("FOVSize", {
    Text = "Kích thước FOV",
    Default = 130,
    Min = 50,
    Max = 600,
    Rounding = 0,
    Callback = function(Value)
        State.FOVSize = Value
    end
})


ThemeManager:SetLibrary(Library)
SaveManager:SetLibrary(Library)
SaveManager:IgnoreThemeSettings()
SaveManager:SetIgnoreIndexes({})
ThemeManager:SetFolder("SuneHub")
SaveManager:SetFolder("SuneHub/configs")
SaveManager:BuildConfigSection(Tabs["UI Settings"])
ThemeManager:ApplyToTab(Tabs["UI Settings"])

UIS.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed or not State.CameraLock then return end
    if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
        local tapPos = Vector2.new(input.Position.X, input.Position.Y)
        local closestDist = math.huge
        local newTarget = nil
        for _, plr in pairs(Players:GetPlayers()) do
            if plr ~= P and plr.Character and plr.Character:FindFirstChild("HumanoidRootPart") then
                local hum = plr.Character:FindFirstChild("Humanoid")
                if hum and hum.Health > 0 then
                    local vector, onScreen = Camera:WorldToScreenPoint(plr.Character.HumanoidRootPart.Position)
                    if onScreen then
                        local dist = (tapPos - Vector2.new(vector.X, vector.Y)).Magnitude
                        if dist < closestDist and dist < 60 then
                            closestDist = dist
                            newTarget = plr.Character.HumanoidRootPart
                        end
                    end
                end
            end
        end
        if newTarget and Target ~= newTarget then
            Target = newTarget
            IgnoreTarget = nil
        end
    end
end)

RS.Stepped:Connect(function()
    if State.Noclip and P.Character then
        for _, v in pairs(P.Character:GetDescendants()) do
            if v:IsA("BasePart") then v.CanCollide = false end
        end
    end
end)

local TargetPartName = "Head"
local Prediction = 0.135
local Smoothing = 0.3

local TargetPartName = "Head"
local Prediction = 0.135

local Prediction = 0.135
local Smoothing = 0.3

RS.RenderStepped:Connect(function()
    if tick() - lastPlayerCheck >= 1 then
        lastPlayerCheck = tick()
        CheckAndUpdatePlayerList()
    end

    if State.ESP then updateESP() end

    if State.CameraLock and SelectedTargetPlayer then
        if SelectedTargetPlayer:IsDescendantOf(game.Players) then
            local char = SelectedTargetPlayer.Character
            if char then
                local head = char:FindFirstChild("Head")
                local hum = char:FindFirstChildOfClass("Humanoid")

                if head and hum and hum.Health > 0 then
                    local velocity = head.AssemblyLinearVelocity or head.Velocity or Vector3.new(0, 0, 0)
                    local predictedPos = head.Position + (velocity * Prediction)
                    
                    
                    if P.Character and P.Character:FindFirstChild("HumanoidRootPart") then
                        local hrp = P.Character.HumanoidRootPart
                        local bodyLookPos = Vector3.new(predictedPos.X, hrp.Position.Y, predictedPos.Z)
                        hrp.CFrame = CFrame.lookAt(hrp.Position, bodyLookPos)
                    end

                    
                    local isFirstPerson = (Camera.CFrame.Position - Camera.Focus.Position).Magnitude < 1.5
                    if isFirstPerson then
                        local camPos = Camera.CFrame.Position
                        local targetCFrame = CFrame.lookAt(camPos, predictedPos)
                        Camera.CFrame = Camera.CFrame:Lerp(targetCFrame, Smoothing)
                    end
                end
            end
        else
            SelectedTargetPlayer = nil
        end
    end

    if State.Anchor and AP_Normal and P.Character and P.Character:FindFirstChild("HumanoidRootPart") then 
        P.Character.HumanoidRootPart.CFrame = AP_Normal 
    end
    
    frameCount += 1
    if tick() - lastTime >= 1 then
        fpsLabel.Text = "FPS: "..frameCount
        pingLabel.Text = "MS: "..math.floor(StatsService.Network.ServerStatsItem["Data Ping"]:GetValue())
        local elapsed = tick() - startTime
        timeLabel.Text = string.format("%02d:%02d:%02d", math.floor(elapsed/3600), math.floor((elapsed%3600)/60), math.floor(elapsed%60))
        frameCount = 0; lastTime = tick()
    end
end)

local Teams = game:GetService("Teams")
local SkyLines = {}

local FOVCircle = nil
if typeof(Drawing) == "table" and Drawing.new then
    FOVCircle = Drawing.new("Circle")
    FOVCircle.Color = Color3.fromRGB(255, 255, 255)
    FOVCircle.Thickness = 1
    FOVCircle.Radius = 130
    FOVCircle.Filled = false
    FOVCircle.Visible = false
end

local function RemoveLine(plr)
    if SkyLines[plr] then
        pcall(function()
            SkyLines[plr].Visible = false
            SkyLines[plr]:Remove()
        end)
        SkyLines[plr] = nil
    end
end

local function ClearAllLines()
    for plr, line in pairs(SkyLines) do
        pcall(function()
            line.Visible = false
            line:Remove()
        end)
    end
    table.clear(SkyLines)
end

game.Players.PlayerRemoving:Connect(function(plr)
    RemoveLine(plr)
end)

local function GetSkyLine(plr)
    if not SkyLines[plr] then
        if typeof(Drawing) == "table" and Drawing.new then
            local line = Drawing.new("Line")
            line.Thickness = 2
            line.Transparency = 1
            line.Visible = false
            SkyLines[plr] = line
        end
    end
    return SkyLines[plr]
end

local function GetPlayerColor(plr)
    if plr.Team then
        return plr.TeamColor.Color
    end
    return Color3.fromRGB(255, 50, 50)
end

local function IsSameTeam(plr)
    if not State.TeamCheck then return false end
    local teamList = Teams:GetTeams()
    if #teamList <= 1 then return false end
    return plr.Team ~= nil and plr.Team == P.Team
end

local function IsVisible(targetPart)
    if not State.WallCheck then return true end
    local char = P.Character
    if not char then return false end
    local params = RaycastParams.new()
    params.FilterDescendantsInstances = {char, Camera}
    params.FilterType = Enum.RaycastFilterType.Exclude
    local origin = Camera.CFrame.Position
    local direction = targetPart.Position - origin
    local result = workspace:Raycast(origin, direction, params)
    return result and result.Instance:IsDescendantOf(targetPart.Parent)
end

local function GetClosestPlayerInFOV()
    local closestPlayer = nil
    local shortestDistance = State.FOVSize or 130
    local viewportCenter = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)

    for _, player in ipairs(game.Players:GetPlayers()) do
        if player ~= P and player.Character and not IsSameTeam(player) then
            local head = player.Character:FindFirstChild("Head")
            local hum = player.Character:FindFirstChildOfClass("Humanoid")

            if head and hum and hum.Health > 0 then
                local screenPos, onScreen = Camera:WorldToViewportPoint(head.Position)
                if onScreen then
                    local targetPos2D = Vector2.new(screenPos.X, screenPos.Y)
                    local distance = (targetPos2D - viewportCenter).Magnitude

                    if distance <= shortestDistance then
                        if IsVisible(head) then
                            shortestDistance = distance
                            closestPlayer = player
                        end
                    end
                end
            end
        end
    end

    return closestPlayer
end

RS:UnbindFromRenderStep("SuneAimbotLogic")
RS:BindToRenderStep("SuneAimbotLogic", Enum.RenderPriority.Camera.Value + 1, function()
    local viewportCenter = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)

    if FOVCircle then
        FOVCircle.Position = viewportCenter
        FOVCircle.Radius = State.FOVSize or 130
        FOVCircle.Visible = State.AimbotEnabled or false
    end

    if not State.AimbotEnabled then
        ClearAllLines()
        return
    end

    for plr, line in pairs(SkyLines) do
        if not game.Players:FindFirstChild(plr.Name) then
            RemoveLine(plr)
        end
    end

    for _, player in ipairs(game.Players:GetPlayers()) do
        if player ~= P then
            local line = GetSkyLine(player)
            if line then
                local char = player.Character
                local head = char and char:FindFirstChild("Head")
                local hum = char and char:FindFirstChildOfClass("Humanoid")

                if head and hum and hum.Health > 0 and not IsSameTeam(player) then
                    local headPos = head.Position
                    local skyPos = headPos + Vector3.new(0, 100, 0)

                    local head2D, headOnScreen = Camera:WorldToViewportPoint(headPos)
                    local sky2D, skyOnScreen = Camera:WorldToViewportPoint(skyPos)

                    if head2D.Z > 0 then
                        line.From = Vector2.new(head2D.X, head2D.Y)
                        line.To = Vector2.new(sky2D.X, sky2D.Y)
                        line.Color = GetPlayerColor(player)
                        line.Visible = true
                    else
                        line.Visible = false
                    end
                else
                    line.Visible = false
                end
            end
        end
    end

    local target = GetClosestPlayerInFOV()
    if target and target.Character and target.Character:FindFirstChild("Head") then
        local head = target.Character.Head
        local velocity = head.AssemblyLinearVelocity or head.Velocity or Vector3.new(0, 0, 0)
        local predictedPos = head.Position + (velocity * 0.135)

        local currentCFrame = Camera.CFrame
        local targetCFrame = CFrame.lookAt(currentCFrame.Position, predictedPos)
        Camera.CFrame = currentCFrame:Lerp(targetCFrame, 0.3)
    end
end)



UIS.JumpRequest:Connect(function() 
    if State.InfJump and P.Character and P.Character:FindFirstChild("Humanoid") then 
        P.Character.Humanoid:ChangeState("Jumping") 
    end 
end)

task.spawn(function()
    while true do
        task.wait(60)
        if State.AntiAFK then 
            VirtualUser:CaptureController()
            VirtualUser:ClickButton2(Vector2.new()) 
        end
    end
end)

local function createOrUpdateTargetESP(player)
    local char = player.Character
    if not char then return end
    
    local head = char:FindFirstChild("Head")
    local humanoid = char:FindFirstChildOfClass("Humanoid")
    if not head or not humanoid then return end

    local highlight = char:FindFirstChild("TargetHighlight")
    if not highlight then
        highlight = Instance.new("Highlight")
        highlight.Name = "TargetHighlight"
        highlight.Adornee = char
        highlight.FillColor = Color3.fromRGB(255, 0, 0)
        highlight.OutlineColor = Color3.fromRGB(255, 255, 255)
        highlight.Parent = char
    end

    local gui = head:FindFirstChild("TargetESP_Gui")
    local label
    if not gui then
        gui = Instance.new("BillboardGui")
        gui.Name = "TargetESP_Gui"
        gui.Adornee = head
        gui.Size = UDim2.new(0, 200, 0, 50)
        gui.StudsOffset = Vector3.new(0, 3, 0)
        gui.AlwaysOnTop = true
        gui.Parent = head

        label = Instance.new("TextLabel")
        label.Name = "InfoLabel"
        label.Size = UDim2.new(1, 0, 1, 0)
        label.BackgroundTransparency = 1
        label.TextColor3 = Color3.fromRGB(255, 255, 255)
        label.TextStrokeTransparency = 0
        label.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
        label.TextSize = 14
        label.Font = Enum.Font.SourceSansBold
        label.Parent = gui
    else
        label = gui:FindFirstChild("InfoLabel")
    end

    if label then
        local hp = math.floor(humanoid.Health)
        local maxHp = math.floor(humanoid.MaxHealth)
        local speed = math.floor(humanoid.WalkSpeed)
        
        label.Text = string.format("%s\nHP: %d/%d | Speed: %d", player.DisplayName, hp, maxHp, speed)
    end
end

local function removeTargetESP(player)
    if player.Character then
        if player.Character:FindFirstChild("TargetHighlight") then
            player.Character.TargetHighlight:Destroy()
        end
        if player.Character:FindFirstChild("Head") and player.Character.Head:FindFirstChild("TargetESP_Gui") then
            player.Character.Head.TargetESP_Gui:Destroy()
        end
    end
end

game:GetService("RunService").Heartbeat:Connect(function()
    if TargetESPEnabled and TargetName ~= "" then
        local search = TargetName:lower()
        for _, player in pairs(game.Players:GetPlayers()) do
            if player ~= game.Players.LocalPlayer then
                if player.Character then
                    local isTarget = player.Name:lower():find(search, 1, true) or player.DisplayName:lower():find(search, 1, true)
                    if isTarget then
                        createOrUpdateTargetESP(player)
                    else
                        removeTargetESP(player)
                    end
                end
            end
        end
    else
        for _, player in pairs(game.Players:GetPlayers()) do
            removeTargetESP(player)
        end
    end
end)


game:GetService("RunService").Heartbeat:Connect(function()
    if State.AutoTele and targetName ~= "" then
        local lp = game.Players.LocalPlayer
        if lp.Character and lp.Character:FindFirstChild("HumanoidRootPart") then
            local search = targetName:lower()
            for _, target in pairs(game.Players:GetPlayers()) do
                if target ~= lp and target.Character and target.Character:FindFirstChild("HumanoidRootPart") then
                    if target.Name:lower():find(search, 1, true) or target.DisplayName:lower():find(search, 1, true) then
                        lp.Character.HumanoidRootPart.CFrame = target.Character.HumanoidRootPart.CFrame
                        break
                    end
                end
            end
        end
    end
end)

task.spawn(function()
    while true do
        task.wait(0.1)
        if State.AutoReset and P.Character and P.Character:FindFirstChild("Humanoid") and P.Character:FindFirstChild("HumanoidRootPart") then
            local hum = P.Character.Humanoid
            local state = hum:GetState()
            if state ~= Enum.HumanoidStateType.Flying and state ~= Enum.HumanoidStateType.Physics then
                if math.abs(P.Character.HumanoidRootPart.CFrame.UpVector.Y) < 0.72 then hum.Health = 0 end
            end
        end
    end
end)
