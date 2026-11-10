print("[SUPERMAN TESTE] Executou")

local ok, erro = pcall(function()
    local jogador = game:GetService("Players").LocalPlayer
    assert(jogador, "LocalPlayer indisponível")

local gui = Instance.new("ScreenGui")
    gui.Name = "TesteSuperman"
    gui.ResetOnSpawn = false
    gui.IgnoreGuiInset = true
    gui.DisplayOrder = 1000000

local texto = Instance.new("TextLabel")
    texto.Size = UDim2.new(0, 320, 0, 100)
    texto.Position = UDim2.new(0.5, -160, 0.5, -50)
    texto.BackgroundColor3 = Color3.fromRGB(15, 15, 20)
    texto.TextColor3 = Color3.fromRGB(255, 255, 255)
    texto.TextSize = 22
    texto.Text = "SUPERMAN: UI FUNCIONANDO"
    texto.Parent = gui

gui.Parent = jogador:WaitForChild("PlayerGui", 10)
    print("[SUPERMAN TESTE] UI criada")
end)

if not ok then
    warn("[SUPERMAN TESTE] ERRO: " .. tostring(erro))
end
