-- MadeiraServer
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")
local Debris = game:GetService("Debris")
local RunService = game:GetService("RunService")

local ToggleEvent = ReplicatedStorage:WaitForChild("ToggleMadeiraEvent")
local ModelsFolder = ReplicatedStorage:WaitForChild("Models")
local WoodModel = ModelsFolder:WaitForChild("WoodPiece") -- seu modelo (ver instruções)

-- CONFIGURAÇÕES
local SPAWN_INTERVAL = 0.12          -- segundos entre cada peça criada por jogador
local MAX_MADEIRA_PER_PLAYER = 150   -- máximo de peças ativas por jogador
local MADEIRA_LIFETIME = 25          -- tempo em segundos antes de remover cada peça
local FRONT_DISTANCE_STUDS = 1.8     -- distância à frente do jogador (~50 cm)
local MIN_DISTANCE = 1.2             -- distância minima permitida (segurança)
local MAX_DISTANCE = 200             -- distancia maxima permitida (segurança)

-- estado por jogador
local playerState = {} -- player -> {active = bool, parts = {}, spawnTask = thread}

local function ensureState(player)
    if not playerState[player] then
        playerState[player] = { active = false, parts = {}, spawnTask = nil, lastToggle = 0 }
    end
    return playerState[player]
end

local function clearOldPieces(state)
    while #state.parts > MAX_MADEIRA_PER_PLAYER do
        local old = table.remove(state.parts, 1)
        if old and old.Parent then
            old:Destroy()
        end
    end
end

local function spawnMadeiraForPlayer(player)
    local state = ensureState(player)
    -- proteção: só enquanto ativo
    while state.active and player and player.Parent do
        -- valida character
        local char = player.Character
        local root = char and (char:FindFirstChild("HumanoidRootPart") or char:FindFirstChildWhichIsA("BasePart"))
        if not root then
            wait(0.5)
            continue
        end

        -- calcula posição na frente do jogador (usando lookVector do HumanoidRootPart)
        local look = root.CFrame.LookVector
        local forward = Vector3.new(look.X, 0, look.Z)
        if forward.Magnitude == 0 then forward = Vector3.new(0,0,-1) end
        forward = forward.Unit

        local spawnPos = root.Position + forward * FRONT_DISTANCE_STUDS + Vector3.new(0, 2, 0) -- spawn um pouco acima do chão
        -- segurança: distância do spawn ao root
        local dist = (spawnPos - root.Position).Magnitude
        if dist < MIN_DISTANCE or dist > MAX_DISTANCE then
            -- se algo estranho, pula esse ciclo
            wait(SPAWN_INTERVAL)
            continue
        end

        -- clone o modelo e posicione (exige que WoodPiece tenha PrimaryPart definido)
        local ok, part = pcall(function()
            local clone = WoodModel:Clone()
            -- exige PrimaryPart no modelo
            if not clone.PrimaryPart then
                -- tenta definir primeiro descendente como PrimaryPart (fallback)
                local p = clone:FindFirstChildWhichIsA("BasePart", true)
                if p then clone.PrimaryPart = p end
            end
            if not clone.PrimaryPart then
                -- If still no PrimaryPart, cria uma Part simples como fallback
                local fallback = Instance.new("Part")
                fallback.Name = "WoodFallback"
                fallback.Size = Vector3.new(2,2,0.5)
                fallback.Position = spawnPos
                fallback.Anchored = false
                fallback.Material = Enum.Material.Wood
                fallback.Parent = workspace
                return fallback
            end
            -- posiciona o modelo
            clone:SetPrimaryPartCFrame(CFrame.new(spawnPos) * CFrame.Angles(math.rad(math.random(-10,10)), math.rad(math.random(0,360)), 0))
            clone.Parent = workspace

            -- garante que todas as peças do modelo não fiquem ancoradas
            for _, v in pairs(clone:GetDescendants()) do
                if v:IsA("BasePart") then
                    v.Anchored = false
                    v.CanCollide = true
                end
            end

            return clone
        end)
        if not ok or not part then
            -- erro na clonagem: esperar e continuar
            wait(SPAWN_INTERVAL)
            continue
        end

        -- aplicação de impulso inicial: tenta AssemblyLinearVelocity se disponível
        pcall(function()
            -- se for modelo com PrimaryPart
            if part:IsA("Model") and part.PrimaryPart then
                -- aplica velocidade em cada BasePart caso AssemblyLinearVelocity não funcione em modelo
                for _, b in ipairs(part:GetDescendants()) do
                    if b:IsA("BasePart") then
                        b:SetNetworkOwner(nil) -- servidor como owner
                        b.AssemblyLinearVelocity = Vector3.new(0, -20, 0)
                    end
                end
            else
                if part:IsA("BasePart") then
                    part:SetNetworkOwner(nil)
                    part.AssemblyLinearVelocity = Vector3.new(0, -20, 0)
                end
            end
        end)

        -- rastreia a peça para o player e limpa se exceder limite
        table.insert(state.parts, part)
        clearOldPieces(state)

        -- usa Debris para remover depois do tempo de vida
        Debris:AddItem(part, MADEIRA_LIFETIME)

        wait(SPAWN_INTERVAL)
    end
    state.spawnTask = nil
end

-- Toggle recebido do cliente
ToggleEvent.OnServerEvent:Connect(function(player, enable)
    if typeof(enable) ~= "boolean" then return end
    local state = ensureState(player)
    -- anti-spam: impede toggles rápidos
    local now = tick()
    if now - state.lastToggle < 0.15 then return end
    state.lastToggle = now

    state.active = enable
    if state.active then
        -- inicia spawn se já não estiver rodando
        if not state.spawnTask then
            -- spawn em thread separada
            state.spawnTask = coroutine.create(function()
                spawnMadeiraForPlayer(player)
            end)
            coroutine.resume(state.spawnTask)
        end
    else
        -- quando desativa, simplesmente para o loop; peças existentes ficam e serão limpas por Debris
        state.active = false
    end
end)

-- limpeza quando jogador sai
Players.PlayerRemoving:Connect(function(player)
    local state = playerState[player]
    if state then
        for _, p in ipairs(state.parts) do
            if p and p.Parent then p:Destroy() end
        end
        playerState[player] = nil
    end
end)
