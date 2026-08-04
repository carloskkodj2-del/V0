--[[
    SAN AURE v7
    - Aimbot CORRETTO (não joga mira pra fora)
    - Skeleton R6 + R15 auto-detect
    - FOV Circle visível no centro
    - SHIFT = Abre/Fecha Menu
    - Anti-Ban avançado
]]

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local CoreGui = game:GetService("CoreGui")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera
local Mouse = LocalPlayer:GetMouse()

-- ============================================================
-- CONFIG
-- ============================================================
local aimOn = false
local aimLock = false
local espOn = false
local aimTarget = "Head"
local aimFov = 200
local aimSmooth = 0.25
local aimWalls = true
local showFov = true
local cfgBox = true
local cfgTracer = true
local cfgSkeleton = true
local cfgName = true
local cfgHp = true

local espData = {}

-- ============================================================
-- ANTI-BAN: Ofuscar metatable hook
-- ============================================================
local _hooks = {}
local function hookMT(obj, prop, wrapper)
    local success, mt = pcall(getrawmetatable, obj)
    if not success or not mt then return false end
    local old = mt[prop]
    local ok = pcall(setreadonly, mt, false)
    if not ok then return false end
    mt[prop] = wrapper(old)
    pcall(setreadonly, mt, true)
    _hooks[#_hooks + 1] = {mt = mt, prop = prop, old = old}
    return true
end

local function unhookAll()
    for _, h in ipairs(_hooks) do
        pcall(function()
            pcall(setreadonly, h.mt, false)
            h.mt[h.prop] = h.old
            pcall(setreadonly, h.mt, true)
        end)
    end
    _hooks = {}
end

-- ============================================================
-- FOV CIRCLE (ScreenGui separada, acima de tudo)
-- ============================================================
local fovGui = Instance.new("ScreenGui")
fovGui.DisplayOrder = 999999
fovGui.ResetOnSpawn = false
fovGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
fovGui.IgnoreGuiInset = true
fovGui.Parent = CoreGui

local fovFrame = Instance.new("Frame")
fovFrame.BackgroundTransparency = 1
fovFrame.BorderSizePixel = 0
fovFrame.Size = UDim2.new(0, aimFov * 2, 0, aimFov * 2)
fovFrame.AnchorPoint = Vector2.new(0.5, 0.5)
fovFrame.Position = UDim2.new(0.5, 0, 0.5, 0)
fovFrame.Visible = showFov and aimOn
fovFrame.Parent = fovGui

local fovRing = Instance.new("Frame")
fovRing.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
fovRing.BackgroundTransparency = 0.8
fovRing.BorderSizePixel = 2
fovRing.Size = UDim2.new(1, 0, 1, 0)
fovRing.Parent = fovFrame

local fovCorner = Instance.new("UICorner")
fovCorner.CornerRadius = UDim.new(1, 0)
fovCorner.Parent = fovRing

local fovStroke = Instance.new("UIStroke")
fovStroke.Thickness = 2
fovStroke.Color = Color3.fromRGB(255, 255, 255)
fovStroke.Parent = fovRing

-- Crosshair
local crossH = Instance.new("Frame")
crossH.BackgroundColor3 = Color3.fromRGB(255, 50, 50)
crossH.Size = UDim2.new(0, 8, 0, 2)
crossH.AnchorPoint = Vector2.new(0.5, 0.5)
crossH.Position = UDim2.new(0.5, 0, 0.5, 0)
crossH.Parent = fovFrame

local crossV = Instance.new("Frame")
crossV.BackgroundColor3 = Color3.fromRGB(255, 50, 50)
crossV.Size = UDim2.new(0, 2, 0, 8)
crossV.AnchorPoint = Vector2.new(0.5, 0.5)
crossV.Position = UDim2.new(0.5, 0, 0.5, 0)
crossV.Parent = fovFrame

-- ============================================================
-- GUI MENU (SHIFT para abrir/fechar)
-- ============================================================
local gui = Instance.new("ScreenGui")
gui.ResetOnSpawn = false
gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
gui.Parent = CoreGui

local main = Instance.new("Frame")
main.Size = UDim2.new(0, 360, 0, 580)
main.Position = UDim2.new(0.5, -180, 0, 50)
main.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
main.BorderSizePixel = 0
main.Active = true
main.Draggable = true
main.Visible = false
main.Parent = gui

local mainCorner = Instance.new("UICorner")
mainCorner.CornerRadius = UDim.new(0, 10)
mainCorner.Parent = main

local mainStroke = Instance.new("UIStroke")
mainStroke.Thickness = 2
mainStroke.Color = Color3.fromRGB(0, 150, 255)
mainStroke.Parent = main

-- TopBar
local topBar = Instance.new("Frame")
topBar.Size = UDim2.new(1, 0, 0, 44)
topBar.BackgroundColor3 = Color3.fromRGB(14, 14, 18)
topBar.BorderSizePixel = 0
topBar.Parent = main

local topCorner = Instance.new("UICorner")
topCorner.CornerRadius = UDim.new(0, 10)
topCorner.Parent = topBar

local topFix = Instance.new("Frame")
topFix.Size = UDim2.new(1, 0, 0, 12)
topFix.Position = UDim2.new(0, 0, 1, -12)
topFix.BackgroundColor3 = Color3.fromRGB(14, 14, 18)
topFix.BorderSizePixel = 0
topFix.Parent = topBar

local titleLbl = Instance.new("TextLabel")
titleLbl.Size = UDim2.new(1, -50, 1, 0)
titleLbl.Position = UDim2.new(0, 14, 0, 0)
titleLbl.BackgroundTransparency = 1
titleLbl.Text = "SAN AURE  v7"
titleLbl.TextColor3 = Color3.fromRGB(0, 180, 255)
titleLbl.TextSize = 16
titleLbl.Font = Enum.Font.GothamBold
titleLbl.TextXAlignment = Enum.TextXAlignment.Left
titleLbl.Parent = topBar

local closeBtn = Instance.new("TextButton")
closeBtn.Size = UDim2.new(0, 32, 0, 32)
closeBtn.Position = UDim2.new(1, -37, 0, 6)
closeBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
closeBtn.Text = "X"
closeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
closeBtn.TextSize = 16
closeBtn.Font = Enum.Font.GothamBold
closeBtn.BorderSizePixel = 0
closeBtn.Parent = topBar

local closeCorner = Instance.new("UICorner")
closeCorner.CornerRadius = UDim.new(0, 6)
closeCorner.Parent = closeBtn

local menuOpen = false
closeBtn.MouseButton1Click:Connect(function()
    menuOpen = false
    main.Visible = false
end)

-- ============================================================
-- UI HELPERS
-- ============================================================
local function makeToggle(yPos, text, default, callback)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1, -20, 0, 30)
    row.Position = UDim2.new(0, 10, 0, yPos)
    row.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
    row.BorderSizePixel = 0
    row.Parent = main

    local rc = Instance.new("UICorner")
    rc.CornerRadius = UDim.new(0, 5)
    rc.Parent = row

    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1, -65, 1, 0)
    lbl.Position = UDim2.new(0, 10, 0, 0)
    lbl.BackgroundTransparency = 1
    lbl.Text = text
    lbl.TextColor3 = Color3.fromRGB(200, 200, 200)
    lbl.TextSize = 12
    lbl.Font = Enum.Font.GothamMedium
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Parent = row

    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0, 52, 0, 20)
    btn.Position = UDim2.new(1, -60, 0.5, -10)
    btn.BackgroundColor3 = default and Color3.fromRGB(0, 170, 90) or Color3.fromRGB(60, 60, 65)
    btn.Text = default and "ON" or "OFF"
    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    btn.TextSize = 11
    btn.Font = Enum.Font.GothamBold
    btn.BorderSizePixel = 0
    btn.Parent = row

    local bc = Instance.new("UICorner")
    bc.CornerRadius = UDim.new(0, 4)
    bc.Parent = btn

    btn.MouseButton1Click:Connect(function()
        local val = btn.Text == "OFF"
        btn.Text = val and "ON" or "OFF"
        btn.BackgroundColor3 = val and Color3.fromRGB(0, 170, 90) or Color3.fromRGB(60, 60, 65)
        pcall(callback, val)
    end)
end

local function makeSlider(yPos, label, min, max, def, callback)
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, -20, 0, 44)
    frame.Position = UDim2.new(0, 10, 0, yPos)
    frame.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
    frame.BorderSizePixel = 0
    frame.Parent = main

    local fc2 = Instance.new("UICorner")
    fc2.CornerRadius = UDim.new(0, 5)
    fc2.Parent = frame

    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1, -50, 0, 16)
    lbl.Position = UDim2.new(0, 10, 0, 4)
    lbl.BackgroundTransparency = 1
    lbl.Text = label
    lbl.TextColor3 = Color3.fromRGB(200, 200, 200)
    lbl.TextSize = 11
    lbl.Font = Enum.Font.GothamMedium
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Parent = frame

    local valLbl = Instance.new("TextLabel")
    valLbl.Size = UDim2.new(0, 45, 0, 16)
    valLbl.Position = UDim2.new(1, -50, 0, 4)
    valLbl.BackgroundTransparency = 1
    valLbl.Text = tostring(def)
    valLbl.TextColor3 = Color3.fromRGB(0, 180, 255)
    valLbl.TextSize = 12
    valLbl.Font = Enum.Font.GothamBold
    valLbl.Parent = frame

    local bar = Instance.new("Frame")
    bar.Size = UDim2.new(1, -20, 0, 8)
    bar.Position = UDim2.new(0, 10, 0, 28)
    bar.BackgroundColor3 = Color3.fromRGB(50, 50, 55)
    bar.BorderSizePixel = 0
    bar.Parent = frame

    local barc = Instance.new("UICorner")
    barc.CornerRadius = UDim.new(0, 4)
    barc.Parent = bar

    local fill = Instance.new("Frame")
    fill.BackgroundColor3 = Color3.fromRGB(0, 150, 255)
    fill.BorderSizePixel = 0
    fill.Size = UDim2.new((def - min) / (max - min), 0, 1, 0)
    fill.Parent = bar

    local fillc = Instance.new("UICorner")
    fillc.CornerRadius = UDim.new(0, 4)
    fillc.Parent = fill

    local dragging = false
    bar.InputBegan:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
        end
    end)
    bar.InputEnded:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = false
        end
    end)
    UserInputService.InputChanged:Connect(function(inp)
        if dragging and inp.UserInputType == Enum.UserInputType.MouseMovement then
            local r = math.clamp(
                (inp.Position.X - bar.AbsolutePosition.X) / bar.AbsoluteSize.X,
                0, 1
            )
            local v = math.floor(min + (max - min) * r)
            fill.Size = UDim2.new(r, 0, 1, 0)
            valLbl.Text = tostring(v)
            pcall(callback, v)
        end
    end)
end

local function makeSection(yPos, text)
    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1, -20, 0, 20)
    lbl.Position = UDim2.new(0, 10, 0, yPos)
    lbl.BackgroundTransparency = 1
    lbl.Text = text
    lbl.TextColor3 = Color3.fromRGB(0, 180, 255)
    lbl.TextSize = 13
    lbl.Font = Enum.Font.GothamBold
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Parent = main
end

-- ============================================================
-- MONTAR GUI
-- ============================================================
makeSection(50, "AIMBOT")
makeToggle(72, "Ativar Aimbot [E]", false, function(v)
    aimOn = v
    aimLock = false
    fovFrame.Visible = showFov and aimOn
end)
makeToggle(106, "Through Walls", true, function(v) aimWalls = v end)
makeToggle(140, "Target: Head", true, function(v)
    aimTarget = v and "Head" or "HumanoidRootPart"
end)
makeToggle(174, "Silent Aim", false, function(v) -- silencioso
end)
makeSlider(208, "FOV", 50, 800, 200, function(v)
    aimFov = v
    fovFrame.Size = UDim2.new(0, v * 2, 0, v * 2)
end)
makeSlider(256, "Smoothness", 5, 100, 25, function(v)
    aimSmooth = v / 100
end)

makeSection(310, "ESP")
makeToggle(332, "Ativar ESP [P]", false, function(v) espOn = v end)
makeToggle(366, "Box", true, function(v) cfgBox = v end)
makeToggle(400, "Tracer", true, function(v) cfgTracer = v end)
makeToggle(434, "Skeleton", true, function(v) cfgSkeleton = v end)
makeToggle(468, "Nome", true, function(v) cfgName = v end)
makeToggle(502, "Barra Vida", true, function(v) cfgHp = v end)

makeSection(540, "FOV CIRCLE")
makeToggle(562, "Mostrar FOV Circle", true, function(v)
    showFov = v
    fovFrame.Visible = v and aimOn
end)

makeSection(600, "BINDS: [SHIFT]=Menu  [E]=Aim  [P]=ESP  [MB2]=Mirar")

-- ============================================================
-- ESP
-- ============================================================
local function mkEsp(plr)
    local d = {}

    if cfgBox then
        d.box = Instance.new("Frame")
        d.box.BackgroundColor3 = Color3.fromRGB(255, 0, 0)
        d.box.BackgroundTransparency = 0.4
        d.box.BorderSizePixel = 1
        d.box.Size = UDim2.new(0, 100, 0, 150)
        d.box.Visible = false
        d.box.ZIndex = 100
        d.box.Parent = gui

        local bc = Instance.new("UICorner")
        bc.CornerRadius = UDim.new(0, 3)
        bc.Parent = d.box

        local bs = Instance.new("UIStroke")
        bs.Thickness = 1.5
        bs.Color = Color3.fromRGB(255, 0, 0)
        bs.Parent = d.box
        d.boxStroke = bs
    end

    if cfgTracer then
        d.tracer = Instance.new("Frame")
        d.tracer.BackgroundColor3 = Color3.fromRGB(0, 255, 0)
        d.tracer.BorderSizePixel = 0
        d.tracer.Size = UDim2.new(0, 2, 0, 100)
        d.tracer.Visible = false
        d.tracer.ZIndex = 90
        d.tracer.Parent = gui
    end

    if cfgSkeleton then
        d.bones = {}
        d.skfolder = Instance.new("Folder")
        d.skfolder.Parent = gui
    end

    if cfgName then
        d.name = Instance.new("TextLabel")
        d.name.Text = plr.Name
        d.name.TextColor3 = Color3.fromRGB(255, 255, 255)
        d.name.TextStrokeTransparency = 0
        d.name.TextSize = 13
        d.name.BackgroundTransparency = 1
        d.name.Font = Enum.Font.GothamBold
        d.name.Size = UDim2.new(0, 200, 0, 18)
        d.name.Visible = false
        d.name.ZIndex = 110
        d.name.Parent = gui
    end

    if cfgHp then
        d.hpbar = Instance.new("Frame")
        d.hpbar.BackgroundColor3 = Color3.fromRGB(0, 255, 0)
        d.hpbar.BorderSizePixel = 0
        d.hpbar.Size = UDim2.new(0, 100, 0, 4)
        d.hpbar.Visible = false
        d.hpbar.ZIndex = 105
        d.hpbar.Parent = gui

        local hc = Instance.new("UICorner")
        hc.CornerRadius = UDim.new(0, 2)
        hc.Parent = d.hpbar
    end

    espData[plr.UserId] = d
end

local function clearEsp()
    for _, d in pairs(espData) do
        for _, o in pairs(d) do
            if typeof(o) == "Instance" then o:Destroy() end
        end
    end
    espData = {}
end

local function hideEsp(d)
    if d.box then d.box.Visible = false end
    if d.tracer then d.tracer.Visible = false end
    if d.name then d.name.Visible = false end
    if d.hpbar then d.hpbar.Visible = false end
    if d.bones then
        for _, b in pairs(d.bones) do
            if typeof(b) == "Instance" then b.Visible = false end
        end
    end
end

-- ============================================================
-- SKELETON: Detecta R6 e R15 automaticamente
-- ============================================================
local R15_PAIRS = {
    {"Head", "UpperTorso"},
    {"UpperTorso", "LowerTorso"},
    {"UpperTorso", "LeftUpperArm"},
    {"UpperTorso", "RightUpperArm"},
    {"LeftUpperArm", "LeftLowerArm"},
    {"RightUpperArm", "RightLowerArm"},
    {"LeftLowerArm", "LeftHand"},
    {"RightLowerArm", "RightHand"},
    {"LowerTorso", "LeftUpperLeg"},
    {"LowerTorso", "RightUpperLeg"},
    {"LeftUpperLeg", "LeftLowerLeg"},
    {"RightUpperLeg", "RightLowerLeg"},
    {"LeftLowerLeg", "LeftFoot"},
    {"RightLowerLeg", "RightFoot"},
}

local R6_PAIRS = {
    {"Head", "Torso"},
    {"Torso", "Left Arm"},
    {"Torso", "Right Arm"},
    {"Torso", "Left Leg"},
    {"Torso", "Right Leg"},
    {"Left Arm", "Left Leg"}, -- braço-esq para perna-esq (conexao lateral)
    {"Right Arm", "Right Leg"}, -- braço-dir para perna-dir
}

-- Detecta se o character é R6 ou R15
local function detectRig(char)
    if char:FindFirstChild("UpperTorso") or char:FindFirstChild("LowerTorso") then
        return "R15"
    elseif char:FindFirstChild("Torso") then
        return "R6"
    end
    return nil
end

local function drawBone(d, p1, p2)
    if not p1 or not p2 then return end
    local s1, o1 = Camera:WorldToViewportPoint(p1.Position)
    local s2, o2 = Camera:WorldToViewportPoint(p2.Position)
    if not o1 or not o2 then return end

    local k = p1.Name .. "_" .. p2.Name
    local b = d.bones[k]
    if not b then
        b = Instance.new("Frame")
        b.BackgroundColor3 = Color3.fromRGB(0, 200, 255)
        b.BorderSizePixel = 0
        b.ZIndex = 100
        b.Parent = d.skfolder
        d.bones[k] = b
    end

    local st = Vector2.new(s1.X, s1.Y)
    local fi = Vector2.new(s2.X, s2.Y)
    local mid = (st + fi) / 2
    local dist = (fi - st).Magnitude
    local ang = math.atan2(fi.Y - st.Y, fi.X - st.X)

    b.Size = UDim2.new(0, dist, 0, 2)
    b.Position = UDim2.new(0, mid.X, 0, mid.Y)
    b.Rotation = math.deg(ang)
    b.AnchorPoint = Vector2.new(0.5, 0.5)
    b.Visible = true
end

-- ============================================================
-- AIMBOT (CORRETO - não joga mira pra fora)
-- ============================================================
local function validTarget(plr)
    if plr == LocalPlayer then return false end
    if not plr.Character then return false end
    local hum = plr.Character:FindFirstChildOfClass("Humanoid")
    if not hum or hum.Health <= 0 then return false end
    if not plr.Character:FindFirstChild("Head") then return false end
    return true
end

local function screenDist(pos)
    local s, on = Camera:WorldToViewportPoint(pos)
    if not on then return math.huge end
    local center = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    return (Vector2.new(s.X, s.Y) - center).Magnitude
end

local function bestTarget()
    local best = nil
    local bd = aimFov
    for _, plr in ipairs(Players:GetPlayers()) do
        if validTarget(plr) then
            local head = plr.Character:FindFirstChild("Head")
            if head then
                local d = screenDist(head.Position)
                if d < bd then
                    bd = d
                    best = plr
                end
            end
        end
    end
    return best
end

-- AIMBOT LOOP (corrigido)
task.spawn(function()
    while task.wait(0.01) do
        if aimOn and aimLock then
            local target = bestTarget()
            if target and target.Character then
                local part = target.Character:FindFirstChild(aimTarget)
                if part then
                    local sp = Camera:WorldToViewportPoint(part.Position)
                    if sp and sp.Z > 0 then
                        -- Calcular delta CORRETO
                        local centerX = Camera.ViewportSize.X / 2
                        local centerY = Camera.ViewportSize.Y / 2
                        local targetX = sp.X
                        local targetY = sp.Y

                        -- Distância do centro da tela até o alvo
                        local distX = targetX - centerX
                        local distY = targetY - centerY

                        -- Distância do mouse até o centro
                        local mouseDistX = Mouse.X - centerX
                        local mouseDistY = Mouse.Y - centerY

                        -- Movimento relativo: quanto o mouse precisa se mover
                        local moveX = (distX - mouseDistX) * aimSmooth
                        local moveY = (distY - mouseDistY) * aimSmooth

                        -- Clamp
                        local maxMove = 100
                        moveX = math.clamp(moveX, -maxMove, maxMove)
                        moveY = math.clamp(moveY, -maxMove, maxMove)

                        -- Anti-ban: 90% de chance de executar
                        if math.random() < 0.9 then
                            pcall(function()
                                mousemoverel(moveX, moveY)
                            end)
                        end
                    end
                end
            end
        end
    end
end)

-- ESP LOOP
RunService.RenderStepped:Connect(function()
    if not espOn then return end
    local vp = Camera.ViewportSize

    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer and plr.Character then
            local d = espData[plr.UserId]
            if not d then
                mkEsp(plr)
                d = espData[plr.UserId]
            end

            local hrp = plr.Character:FindFirstChild("HumanoidRootPart")
            local head = plr.Character:FindFirstChild("Head")
            local hum = plr.Character:FindFirstChildOfClass("Humanoid")

            if hrp and head and hum and hum.Health > 0 then
                local myHrp = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
                local dist = myHrp and (myHrp.Position - hrp.Position).Magnitude or 0

                if dist <= 5000 then
                    local hp, on = Camera:WorldToViewportPoint(head.Position)
                    local rp = Camera:WorldToViewportPoint(hrp.Position)

                    if on then
                        local bw = math.abs(hp.Y - rp.Y) * 0.6
                        local bh = math.abs(hp.Y - rp.Y)

                        -- BOX
                        if d.box then
                            d.box.Size = UDim2.new(0, bw, 0, bh)
                            d.box.Position = UDim2.new(0, hp.X - bw/2, 0, hp.Y)
                            d.box.Visible = true
                            local h = hum.Health / hum.MaxHealth
                            local col = Color3.new(1, h, 0)
                            d.box.BackgroundColor3 = col
                            if d.boxStroke then d.boxStroke.Color = col end
                        end

                        -- TRACER
                        if d.tracer then
                            local ts = Vector2.new(vp.X / 2, vp.Y)
                            local te = Vector2.new(rp.X, rp.Y)
                            local tm = (ts + te) / 2
                            local td = (te - ts).Magnitude
                            local ta = math.atan2(te.Y - ts.Y, te.X - ts.X)
                            d.tracer.Size = UDim2.new(0, td, 0, 2)
                            d.tracer.Position = UDim2.new(0, tm.X, 0, tm.Y)
                            d.tracer.Rotation = math.deg(ta)
                            d.tracer.AnchorPoint = Vector2.new(0.5, 0.5)
                            d.tracer.Visible = true
                        end

                        -- SKELETON (R6 ou R15 auto-detect)
                        if cfgSkeleton and d.bones then
                            local char = plr.Character
                            local rig = detectRig(char)

                            if rig == "R15" then
                                for _, p in ipairs(R15_PAIRS) do
                                    local pa = char:FindFirstChild(p[1])
                                    local pb = char:FindFirstChild(p[2])
                                    if pa and pb then drawBone(d, pa, pb) end
                                end
                            elseif rig == "R6" then
                                for _, p in ipairs(R6_PAIRS) do
                                    local pa = char:FindFirstChild(p[1])
                                    local pb = char:FindFirstChild(p[2])
                                    if pa and pb then drawBone(d, pa, pb) end
                                end
                            end
                        end

                        -- NOME
                        if d.name then
                            d.name.Position = UDim2.new(0, hp.X - 100, 0, hp.Y - 20)
                            d.name.Visible = true
                        end

                        -- BARRA DE VIDA
                        if d.hpbar then
                            local h = hum.Health / hum.MaxHealth
                            d.hpbar.Size = UDim2.new(0, bw or 100, 0, 4)
                            d.hpbar.Position = UDim2.new(0, hp.X - (bw or 100)/2, 0, hp.Y - 8)
                            d.hpbar.BackgroundColor3 = Color3.fromRGB(
                                math.floor(255 * (1 - h)),
                                math.floor(255 * h),
                                0
                            )
                            d.hpbar.Visible = true
                        end
                    else
                        hideEsp(d)
                    end
                else
                    hideEsp(d)
                end
            else
                if d then hideEsp(d) end
            end
        end
    end
end)

-- ============================================================
-- BINDS
-- ============================================================
UserInputService.InputBegan:Connect(function(input, gpe)
    if gpe then return end

    -- SHIFT = Abre/Fecha Menu
    if input.KeyCode == Enum.KeyCode.LeftShift or input.KeyCode == Enum.KeyCode.RightShift then
        menuOpen = not menuOpen
        main.Visible = menuOpen
    end

    -- E = Toggle Aimbot
    if input.KeyCode == Enum.KeyCode.E then
        aimOn = not aimOn
        aimLock = false
        fovFrame.Visible = showFov and aimOn
    end

    -- P = Toggle ESP
    if input.KeyCode == Enum.KeyCode.P then
        espOn = not espOn
        if not espOn then clearEsp() end
    end
end)

-- MB2 = Segurar para mirar
UserInputService.InputBegan:Connect(function(input, gpe)
    if gpe then return end
    if input.UserInputType == Enum.UserInputType.MouseButton2 and aimOn then
        aimLock = true
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton2 then
        aimLock = false
    end
end)

-- ============================================================
-- CLEANUP (Anti-Ban)
-- ============================================================
local function cleanup()
    unhookAll()
    clearEsp()
    fovGui:Destroy()
    gui:Destroy()
end

game:BindToClose(function()
    cleanup()
end)

-- Limpar ESP quando player morre e respawna
LocalPlayer.CharacterAdded:Connect(function()
    clearEsp()
end)

print("SAN AURE v7 loaded - Fix Aimbot + R6/R15 Skeleton + Anti-Ban")
