-- RayField-like LocalScript (use apenas no SEU jogo / ambiente de desenvolvimento)
-- Coloque este LocalScript em StarterPlayer > StarterPlayerScripts

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local StarterGui = game:GetService("StarterGui")

local player = Players.LocalPlayer
local camera = workspace.CurrentCamera

-- Configs
local INFINITE_JUMP_ENABLED = false
local FLY_ENABLED = false
local ESP_ENABLED = false

local FLY_SPEED = 60 -- velocidade do voo (ajuste)
local MOBILE_BUTTON_SIZE = UDim2.new(0,100,0,60)

-- util
local function isMobile()
    return UserInputService.TouchEnabled and not UserInputService.KeyboardEnabled
end

-- cria GUI minimalista
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "RayFieldLite"
screenGui.ResetOnSpawn = false
screenGui.Parent = player:WaitForChild("PlayerGui")

local frame = Instance.new("Frame")
frame.Size = UDim2.new(0,240,0,140)
frame.Position = UDim2.new(0,10,0,10)
frame.BackgroundTransparency = 0.25
frame.BackgroundColor3 = Color3.fromRGB(20,20,20)
frame.BorderSizePixel = 0
frame.Parent = screenGui

local function makeToggle(text, y)
    local label = Instance.new("TextLabel", frame)
    label.Position = UDim2.new(0,8,0,y)
    label.Size = UDim2.new(0,140,0,30)
    label.BackgroundTransparency = 1
    label.Text = text
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.TextColor3 = Color3.new(1,1,1)
    label.Font = Enum.Font.SourceSans
    label.TextSize = 18

    local button = Instance.new("TextButton", frame)
    button.Position = UDim2.new(0,150,0,y)
    button.Size = UDim2.new(0,70,0,30)
    button.Text = "OFF"
    button.Font = Enum.Font.SourceSansBold
    button.TextSize = 16
    button.BackgroundColor3 = Color3.fromRGB(60,60,60)
    button.TextColor3 = Color3.fromRGB(255,180,80)

    return label, button
end

local _, btnInfinite = makeToggle("Infinite Jump", 8)
local _, btnFly = makeToggle("Fly (mobile+desktop)", 46)
local _, btnESP = makeToggle("ESP Players", 84)

-- status helpers
local function setButtonState(btn, on)
    if on then
        btn.Text = "ON"
        btn.BackgroundColor3 = Color3.fromRGB(30,110,30)
    else
        btn.Text = "OFF"
        btn.BackgroundColor3 = Color3.fromRGB(60,60,60)
    end
end

-- Infinite Jump Implementation
local function enableInfiniteJump()
    INFINITE_JUMP_ENABLED = true
    setButtonState(btnInfinite, true)
end
local function disableInfiniteJump()
    INFINITE_JUMP_ENABLED = false
    setButtonState(btnInfinite, false)
end

-- Força pular mesmo no ar quando tecla espaço for pressionada
UserInputService.InputBegan:Connect(function(input, gp)
    if gp then return end
    if input.KeyCode == Enum.KeyCode.Space and INFINITE_JUMP_ENABLED then
        local char = player.Character
        if char then
            local hum = char:FindFirstChildOfClass("Humanoid")
            if hum then
                -- chama ChangeState para forçar pulo
                hum:ChangeState(Enum.HumanoidStateType.Jumping)
            end
        end
    end
end)

-- Também faz jump no touch (mobile) — ao tocar na tela uma vez
if isMobile() then
    UserInputService.TouchTap:Connect(function()
        if INFINITE_JUMP_ENABLED then
            local char = player.Character
            if char then
                local hum = char:FindFirstChildOfClass("Humanoid")
                if hum then hum:ChangeState(Enum.HumanoidStateType.Jumping) end
            end
        end
    end)
end

-- Fly Implementation (cliente; usa AssemblyLinearVelocity para suavidade)
local flyVelocity = Vector3.new(0,0,0)
local flyBodyEnabled = false

local function enableFly()
    FLY_ENABLED = true
    setButtonState(btnFly, true)
end
local function disableFly()
    FLY_ENABLED = false
    setButtonState(btnFly, false)
end

-- controles de teclado para voo
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

-- Mobile fly buttons
local upButton, downButton
local function createMobileFlyControls()
    if not isMobile() then return end
    upButton = Instance.new("TextButton", screenGui)
    upButton.Size = MOBILE_BUTTON_SIZE
    upButton.Position = UDim2.new(1,-110,1,-140)
    upButton.Text = "Fly+"
    upButton.BackgroundTransparency = 0.35

    downButton = Instance.new("TextButton", screenGui)
    downButton.Size = MOBILE_BUTTON_SIZE
    downButton.Position = UDim2.new(1,-110,1,-70)
    downButton.Text = "Fly-"
    downButton.BackgroundTransparency = 0.35

    upButton.TouchStarted:Connect(function() upDown = 1 end)
    upButton.TouchEnded:Connect(function() upDown = 0 end)
    downButton.TouchStarted:Connect(function() upDown = -1 end)
    downButton.TouchEnded:Connect(function() upDown = 0 end)
end

local function destroyMobileFlyControls()
    if upButton then upButton:Destroy(); upButton = nil end
    if downButton then downButton:Destroy(); downButton = nil end
end

-- aplica o movimento de voo localmente
RunService.Heartbeat:Connect(function(dt)
    if not FLY_ENABLED then return end
    local char = player.Character
    if not char then return end
    local root = char:FindFirstChild("HumanoidRootPart")
    if not root then return end

    -- calcula direção baseada na câmera
    local camCF = workspace.CurrentCamera.CFrame
    local forward = camCF.LookVector
    forward = Vector3.new(forward.X, 0, forward.Z).Unit
    if forward ~= forward then forward = Vector3.new(0,0,-1) end -- NaN guard

    local right = camCF.RightVector
    right = Vector3.new(right.X, 0, right.Z).Unit
    if right ~= right then right = Vector3.new(1,0,0) end

    local dir = (forward * moveDir.Z) + (right * moveDir.X) + Vector3.new(0, upDown, 0)
    if dir.Magnitude > 0 then
        local targetVel = dir.Unit * FLY_SPEED
        -- suaviza a transição
        local currentVel = root.AssemblyLinearVelocity
        local newVel = currentVel:Lerp(targetVel, math.clamp(dt*8, 0, 1))
        root.AssemblyLinearVelocity = Vector3.new(newVel.X, newVel.Y, newVel.Z)
    else
        -- desacelera gradualmente
        local currentVel = root.AssemblyLinearVelocity
        root.AssemblyLinearVelocity = currentVel:Lerp(Vector3.new(0,0,0), math.clamp(dt*3,0,1))
    end
end)

-- ESP Implementation (usa Highlight para clareza)
local highlights = {}

local function enableESP()
    ESP_ENABLED = true
    setButtonState(btnESP, true)
    -- cria highlight para cada jogador (exceto você)
    for _, pl in ipairs(Players:GetPlayers()) do
        if pl ~= player and pl.Character and pl.Character:FindFirstChild("HumanoidRootPart") then
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

-- atualizar highlights quando jogadores entrarem/sairem/respawnarem
Players.PlayerAdded:Connect(function(pl)
    pl.CharacterAdded:Connect(function(char)
        if ESP_ENABLED and pl ~= player then
            wait(0.1)
            if char and char:FindFirstChildWhichIsA("BasePart") then
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

-- botões clicáveis
btnInfinite.MouseButton1Click:Connect(function()
    if INFINITE_JUMP_ENABLED then disableInfiniteJump() else enableInfiniteJump() end
end)
btnFly.MouseButton1Click:Connect(function()
    if FLY_ENABLED then
        disableFly()
        destroyMobileFlyControls()
    else
        enableFly()
        if isMobile() then createMobileFlyControls() end
    end
end)
btnESP.MouseButton1Click:Connect(function()
    if ESP_ENABLED then disableESP() else enableESP() end
end)

-- Ao spawnar, garante que mobile controls apareçam se Fly já ativado
player.CharacterAdded:Connect(function()
    if FLY_ENABLED and isMobile() then createMobileFlyControls() end
end)

-- inicializa botões no estado OFF
setButtonState(btnInfinite, INFINITE_JUMP_ENABLED)
setButtonState(btnFly, FLY_ENABLED)
setButtonState(btnESP, ESP_ENABLED)
