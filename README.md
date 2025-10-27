-- MadeiraClient (LocalScript)
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local UserInputService = game:GetService("UserInputService")
local Players = game:GetService("Players")

local ToggleEvent = ReplicatedStorage:WaitForChild("ToggleMadeiraEvent")
local player = Players.LocalPlayer

local madeiraOn = false
local canToggle = true

local function showHint(text, time)
    local gui = Instance.new("Hint")
    gui.Text = text
    gui.Parent = player:WaitForChild("PlayerGui")
    delay(time or 1.4, function()
        if gui and gui.Parent then gui:Destroy() end
    end)
end

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.UserInputType == Enum.UserInputType.Keyboard and input.KeyCode == Enum.KeyCode.F and canToggle then
        canToggle = false
        madeiraOn = not madeiraOn
        -- envia toggle pro servidor
        ToggleEvent:FireServer(madeiraOn)
        if madeiraOn then
            showHint("Madeira infinita: ATIVADA", 1.5)
        else
            showHint("Madeira infinita: DESATIVADA", 1.2)
        end
        wait(0.12) -- pequena proteção local contra spam
        canToggle = true
    end
end)
