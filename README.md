-- RayField-lite atualizado com função "Madeira Infinita" (LOCAL)
-- Use apenas no SEU jogo / ambiente de desenvolvimento
-- Coloque em StarterPlayer > StarterPlayerScripts

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")

local player = Players.LocalPlayer
local camera = workspace.CurrentCamera

-- ================= Configurações =================
local INFINITE_JUMP_ENABLED = false
local FLY_ENABLED = false
local ESP_ENABLED = false
local MADEIRA_ENABLED = false -- estado da madeira infinita (toggle pela tecla F)

local FLY_SPEED = 60
local SPAWN_RATE_MADEIRA = 0.12   -- segundos entre cada peça de madeira
local MADEIRA_LIFETIME = 12       -- tempo antes de destruir cada peça
local MADEIRA_SIZE = Vector3.new(2,2,0.5)
-- 50 cm = 0.5 metros; 1 stud ≈ 0.28m -> 0.5m ≈ 1.8 studs
local DIST_FRONT_STUDS = 1.8      -- distância à frente do jogador para spawn (≈50 cm)
local MAX_MADEIRA_LOCAL = 200     -- limite local para evitar travar o cliente

-- =================================================

-- util
local function isMobile()
    return UserInputService.TouchEnabled and not UserInputService.KeyboardEnabled
end

-- GUI (nome do painel = "os brabo")
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "os brabo"
screenGui.ResetOnSpawn = false
screenGui.Parent = player:WaitForChild("PlayerGui")

local frame = Instance.new("Frame")
frame.Size = UDim2.new(0,260,0,170)
frame.Position = UDim2.new(0,10,0,10)
frame.BackgroundTransparency = 0.2
frame.BackgroundColor3 = Color3.fromRGB(18,18,18)
frame.BorderSizePixel = 0
frame.Parent = screenGui

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1,0,0,28)
title.Position = UDim2.new(0,0,0,0)
title.BackgroundTransparency = 1
title.Text = "os brabo"
title.TextSize = 20
title.Font = Enum.Font.SourceSansBold
title.TextColor3 = Color3.fromRGB(255,200,80)
title.Parent = frame

local function makeToggle(text, y)
    local label = Instance.new("TextLabel", frame)
    label.Position = UDim2.new(0,10,0,y)
    label.Size = UDim2.new(0,150,0,28)
    label.BackgroundTransparency = 1
    label.Text = text
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.TextColor3 = Color3.fromRGB(230,230,230)
    label.Font = Enum.Font.SourceSans
    label.TextSize = 16

    local button = Instance.new("TextButton", frame)
    button.Position = UDim2.new(0,170,0,y)
    button.Size = UDim2.new(0,70,0,28)
    button.Text = "OFF"
    button.Font = Enum.Font.SourceSansBold
    button.TextSize = 14
    button.BackgroundColor3 = Color3.fromRGB(70,70,70)
    button.TextColor3 = Color3.fromRGB(255,180,80)

    return label, button
end

local _, btnInfinite = makeToggle("Infinite Jump", 36)
local _, btnFly = makeToggle("Fly (mobile+desktop)", 72)
local _, btnESP = makeToggle("ESP Players", 108)

local function setButtonState(btn, on)
    if on then
        btn.Text = "ON"
        btn.BackgroundColor3 = Color3.fromRGB(30,130,30)
    else
        btn.Text = "OFF"
        btn.BackgroundColor3 = Color3.fromRGB(70,70,70)
    end
end

-- ============ Infinite Jump ============
UserInputService.InputBegan:Connect(function(input, gp)
    if gp then return end
    if input.KeyCode == Enum.KeyCode.Space and INFINITE_JUMP_ENABLED then
        local char = player.Character
        if char then
            local hum = char:FindFirstChildOfClass("Humanoid")
            if hum then hum:ChangeState(Enum.HumanoidStateType.Jumping) end
        end
    end
end)

local function enableInfiniteJump() INFINITE_JUMP_ENABLED = true; setButtonState(btnInfinite, true) end
local function disableInfiniteJump() INFINITE_JUMP_ENABLED = false; setButtonState(btnInfinite, false) end

-- ============ Fly (cliente) ============
local moveDir = Vector3.new(0,0,0)
local upDown = 0

UserInputService.InputBegan:Connect(function(input, gp)
    if gp then return end
    if not FLY_ENABLED then return end
    if input.UserInputType == Enum.UserInputType.Keyboard then
        local kc = input.KeyCode
        if kc == Enum.KeyCode.W then moveDir = moveDir + Vector3.new(0,0,-1) end
        if kc == Enum.KeyCode.S then moveDir = moveDir + Vector3.new(0,0,1) end
        if kc == Enum.KeyCode.A then moveDir = moveDir + Vector3.new(-1,0,0) end
        if kc == Enum.KeyCode.D then moveDir = moveDir + Vector3.new(1,0,0) end
        if kc == Enum.KeyCode.Space then upDown = upDown + 1 end
        if kc == Enum.KeyCode.LeftControl or kc == Enum.KeyCode.LeftShift then upDown = upDown - 1 end
    end
end)
UserInputService.InputEnded:Connect(function(input, gp)
    if gp then return end
    if not FLY_ENABLED then return end
    if input.UserInputType == Enum.UserInputType.Keyboard then
        local kc = input.KeyCode
        if kc == Enum.KeyCode.W then moveDir = moveDir - Vector3.new(0,0,-1) end
        if kc == Enum.KeyCode.S then moveDir = moveDir - Vector3.new(0,0,1) end
        if kc == Enum.KeyCode.A then moveDir = moveDir - Vector3.new(-1,0,0) end
        if kc == Enum.KeyCode.D then moveDir = moveDir - Vector3.new(1,0,0) end
        if kc == Enum.KeyCode.Space then upDown = upDown - 1 end
        if kc == Enum.KeyCode.LeftControl or kc == Enum.KeyCode.LeftShift then upDown = upDown + 1 end
    end
end)

RunService.Heartbeat:Connect(function(dt)
    if not FLY_ENABLED then return end
    local char = player.Character
    if not char then return end
    local root = char:FindFirstChild("HumanoidRootPart")
    if not root then return end

    local camCF = workspace.CurrentCamera.CFrame
    local forward = camCF.LookVector
    forward = Vector3.new(forward.X, 0, forward.Z)
    if forward.Magnitude > 0 then forward = forward.Unit end
    local right = Vector3.new(camCF.RightVector.X, 0, camCF.RightVector.Z)
    if right.Magnitude > 0 then right = right.Unit end

    local dir = (forward * moveDir.Z) + (right * moveDir.X) + Vector3.new(0, upDown, 0)
    if dir.Magnitude > 0 then
        local targetVel = dir.Unit * FLY_SPEED
        local currentVel = root.AssemblyLinearVelocity
        local newVel = currentVel:Lerp(targetVel, math.clamp(dt*8, 0, 1))
        root.AssemblyLinearVelocity = Vector3.new(newVel.X, newVel.Y, newVel.Z)
    else
        local currentVel = root.AssemblyLinearVelocity
        root.AssemblyLinearVelocity = currentVel:Lerp(Vector3.new(0,0,0), math.clamp(dt*3,0,1))
    end
end)

local function enableFly() FLY_ENABLED = true; setButtonState(btnFly, true) end
local function disableFly() FLY_ENABLED = false; setButtonState(btnFly, false) end

-- ============ ESP (Highlight) ============
local highlights = {}
local function enableESP()
    ESP_ENABLED = true
    setButtonState(btnESP, true)
    for _, pl in ipairs(Players:GetPlayers()) do
        if pl ~= player and pl.Character then
            if not highlights[pl] then
                local h = Instance.new("Highlight")
                h.Adornee = pl.Character
                h.FillTransparency = 0.6
                h.OutlineTransparency = 0.5
                h.Parent = workspace
                highlights[pl] = h
            end
        end
    end
end

local function disableESP()
    ESP_ENABLED = false
    setButtonState(btnESP, false)
    for pl,h in pairs(highlights) do
        if h and h.Parent then h:Destroy() end
    end
    highlights = {}
end

Players.PlayerAdded:Connect(function(pl)
    pl.CharacterAdded:Connect(function(char)
        if ESP_ENABLED and pl ~= player then
            wait(0.1)
            if char then
                if not highlights[pl] then
                    local h = Instance.new("Highlight")
                    h.Adornee = char
                    h.FillTransparency = 0.6
                    h.OutlineTransparency = 0.5
                    h.Parent = workspace
                    highlights[pl] = h
                end
            end
        end
    end)
end)
Players.PlayerRemoving:Connect(function(pl)
    if highlights[pl] then
        pcall(function() highlights[pl]:Destroy() end)
        highlights[pl] = nil
    end
end)

-- ============ MADEIRA INFINITA (LOCAL) ============
local madeiraParts = {}
local spawnCoroutine = nil

local function createMadeiraPart(spawnPos)
    local part = Instance.new("Part")
    part.Name = "Madeira"
    part.Size = MADEIRA_SIZE
    part.Position = spawnPos
    part.Anchored = false
    part.CanCollide = true
    part.Material = Enum.Material.Wood
    part.TopSurface = Enum.SurfaceType.Smooth
    part.BottomSurface = Enum.SurfaceType.Smooth
    part.Parent = workspace

    -- aplica uma pequena velocidade para "cair"
    local bv = Instance.new("BodyVelocity")
    bv.MaxForce = Vector3.new(1e5,1e5,1e5)
    bv.Velocity = Vector3.new(0, -20, 0)
    bv.Parent = part
    delay(0.12, function() if bv and bv.Parent then bv:Destroy() end end)

    -- rastreia localmente
    table.insert(madeiraParts, part)
    if #madeiraParts > MAX_MADEIRA_LOCAL then
        local old = table.remove(madeiraParts, 1)
        if old and old.Parent then old:Destroy() end
    end

    -- limpeza automática
    delay(MADEIRA_LIFETIME, function()
        if part and part.Parent then part:Destroy() end
    end)
end

local function getSpawnPositionInFront()
    local char = player.Character
    local root = char and (char:FindFirstChild("HumanoidRootPart") or char:FindFirstChildWhichIsA("BasePart"))
    local origin
    local lookVector
    if root then
        origin = root.Position
        -- usa a direção da câmera para ficar natural
        lookVector = workspace.CurrentCamera and workspace.CurrentCamera.CFrame.LookVector or root.CFrame.LookVector
    else
        origin = workspace.CurrentCamera.CFrame.Position
        lookVector = workspace.CurrentCamera.CFrame.LookVector
    end
    local spawnPos = origin + (Vector3.new(lookVector.X, 0, lookVector.Z).Unit * DIST_FRONT_STUDS) + Vector3.new(0, 2, 0)
    -- pequeno jitter para espalhar
    local jitter = Vector3.new((math.random()-0.5)*1, 0, (math.random()-0.5)*1)
    return spawnPos + jitter
end

local function startMadeiraLoop()
    if spawnCoroutine then return end
    spawnCoroutine = true
    spawn(function()
        while spawnCoroutine do
            if not MADEIRA_ENABLED then break end
            local pos = getSpawnPositionInFront()
            createMadeiraPart(pos)
            wait(SPAWN_RATE_MADEIRA)
        end
        spawnCoroutine = nil
    end)
end

local function stopMadeiraLoop()
    spawnCoroutine = false
end

-- Toggle tecla F para ativar/desativar madeira infinita
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end

    -- tecla F = toggle madeira infinita
    if input.UserInputType == Enum.UserInputType.Keyboard and input.KeyCode == Enum.KeyCode.F then
        MADEIRA_ENABLED = not MADEIRA_ENABLED
        if MADEIRA_ENABLED then
            startMadeiraLoop()
            -- aviso rápido
            local hint = Instance.new("Hint")
            hint.Text = "Madeira infinita: ATIVADA"
            hint.Parent = player:WaitForChild("PlayerGui")
            delay(1.5, function() if hint and hint.Parent then hint:Destroy() end end)
        else
            stopMadeiraLoop()
            local hint = Instance.new("Hint")
            hint.Text = "Madeira infinita: DESATIVADA"
            hint.Parent = player:WaitForChild("PlayerGui")
            delay(1.2, function() if hint and hint.Parent then hint:Destroy() end end)
        end
    end
end)

-- ============ Botões GUI ============
btnInfinite.MouseButton1Click:Connect(function()
    if INFINITE_JUMP_ENABLED then disableInfiniteJump() else enableInfiniteJump() end
end)
btnFly.MouseButton1Click:Connect(function()
    if FLY_ENABLED then disableFly() else enableFly() end
end)
btnESP.MouseButton1Click:Connect(function()
    if ESP_ENABLED then disableESP() else enableESP() end
end)

-- inicializa estados visuais
setButtonState(btnInfinite, INFINITE_JUMP_ENABLED)
setButtonState(btnFly, FLY_ENABLED)
setButtonState(btnESP, ESP_ENABLED)

-- garante criar controles mobile de fly ao spawn se necessário (simples)
player.CharacterAdded:Connect(function()
    -- nada extra por enquanto, apenas garante que futuras calls funcionem
end)
