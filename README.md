--[[
    Script Freecam + Aim Assist para Roblox (Mobile)
    Compatível com executors mobile (loadstring).
    Ativa uma interface touch completa com:
    - Freecam com joystick, botões de altura e área de rotação
    - Aim assist suave com FOV, raycast e suavização
    - Círculo de FOV opcional
    Comentários em português.
]]

-- Serviços
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local player = Players.LocalPlayer
local camera = Workspace.CurrentCamera

-- Aguarda o jogador e seu PlayerGui
local playerGui = player:WaitForChild("PlayerGui")

-- ======================== VARIÁVEIS DE ESTADO ========================
local freecamActive = false
local aimActive = false
local moveSpeed = 16            -- studs por segundo
local rotationSensitivity = 0.5 -- sensibilidade do arrasto
local aimSmoothing = 0.15       -- fator de interpolação (0 = instantâneo, 1 = sem movimento)
local fovCircleVisible = true
local upPressed = false
local downPressed = false
local joystickDirection = Vector2.new(0, 0) -- X: right/left, Y: forward/back

-- Estados de toque para áreas dedicadas
local rotationTouch = nil       -- InputObject do toque na área de rotação
local initialTouchPos = nil     -- posição inicial do toque de rotação
local initialCamCFrame = nil    -- CFrame da câmera no início do arrasto

local heartbeatConn = nil       -- conexão do RunService.Heartbeat

-- ======================== INTERFACE (GUI) ========================
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "FreecamAimAssist"
screenGui.ResetOnSpawn = false
screenGui.Parent = playerGui

-- Função auxiliar para criar botões rapidamente
local function createButton(name, text, position, size, parent)
    local btn = Instance.new("TextButton")
    btn.Name = name
    btn.Text = text
    btn.Position = position
    btn.Size = size
    btn.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    btn.Font = Enum.Font.SourceSansBold
    btn.TextSize = 14
    btn.BorderSizePixel = 0
    btn.AutoButtonColor = false
    btn.Parent = parent or screenGui
    return btn
end

-- Botão Freecam (ON/OFF) – canto superior direito
local freecamBtn = createButton("FreecamToggle", "Freecam OFF", 
    UDim2.new(1, -110, 0, 10), UDim2.new(0, 100, 0, 40))
freecamBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)

-- Botão Aim Assist (ON/OFF) – abaixo do freecam
local aimBtn = createButton("AimToggle", "Aim OFF", 
    UDim2.new(1, -110, 0, 60), UDim2.new(0, 100, 0, 40))
aimBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 200)

-- Área de velocidade (+/-) – canto superior esquerdo
local speedLabel = Instance.new("TextLabel")
speedLabel.Name = "SpeedLabel"
speedLabel.Text = "Vel: " .. moveSpeed
speedLabel.Position = UDim2.new(0, 10, 0, 10)
speedLabel.Size = UDim2.new(0, 80, 0, 30)
speedLabel.BackgroundTransparency = 1
speedLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
speedLabel.Font = Enum.Font.SourceSans
speedLabel.TextSize = 16
speedLabel.Parent = screenGui

local speedUpBtn = createButton("SpeedUp", "+", 
    UDim2.new(0, 100, 0, 10), UDim2.new(0, 30, 0, 30))
speedUpBtn.BackgroundColor3 = Color3.fromRGB(60, 60, 60)

local speedDownBtn = createButton("SpeedDown", "-", 
    UDim2.new(0, 140, 0, 10), UDim2.new(0, 30, 0, 30))
speedDownBtn.BackgroundColor3 = Color3.fromRGB(60, 60, 60)

-- Botão toggle do círculo FOV
local fovToggleBtn = createButton("FOVToggle", "FOV: ON", 
    UDim2.new(0, 180, 0, 10), UDim2.new(0, 70, 0, 30))
fovToggleBtn.BackgroundColor3 = Color3.fromRGB(80, 80, 80)

-- ======================== CÍRCULO FOV (Opcional) ========================
local fovCircle = Instance.new("Frame")
fovCircle.Name = "FOVCircle"
fovCircle.Size = UDim2.new(0, 200, 0, 200)
fovCircle.Position = UDim2.new(0.5, -100, 0.5, -100)
fovCircle.AnchorPoint = Vector2.new(0.5, 0.5)
fovCircle.BackgroundTransparency = 1
fovCircle.BorderSizePixel = 0
fovCircle.Visible = fovCircleVisible
fovCircle.Parent = screenGui

local stroke = Instance.new("UIStroke")
stroke.Thickness = 2
stroke.Color = Color3.fromRGB(0, 255, 0)
stroke.Transparency = 0.5
stroke.Parent = fovCircle

local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0.5, 0) -- transforma o quadrado em círculo
corner.Parent = fovCircle

-- ======================== JOYSTICK DE MOVIMENTO ========================
local joystickBase = Instance.new("Frame")
joystickBase.Name = "JoystickBase"
joystickBase.Size = UDim2.new(0, 150, 0, 150)
joystickBase.Position = UDim2.new(0, 20, 1, -170)
joystickBase.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
joystickBase.BackgroundTransparency = 0.85
joystickBase.BorderSizePixel = 0
joystickBase.Active = true -- recebe toques diretamente
joystickBase.ZIndex = 10
joystickBase.Parent = screenGui

local knob = Instance.new("Frame")
knob.Name = "Knob"
knob.Size = UDim2.new(0, 50, 0, 50)
knob.Position = UDim2.new(0.5, -25, 0.5, -25)
knob.BackgroundColor3 = Color3.fromRGB(200, 200, 200)
knob.BackgroundTransparency = 0.5
knob.BorderSizePixel = 0
knob.ZIndex = 11
knob.Parent = joystickBase

-- ======================== BOTÕES DE ALTURA ========================
local upBtn = createButton("UpBtn", "Sobe", 
    UDim2.new(0, 20, 0.5, -80), UDim2.new(0, 60, 0, 60))
upBtn.BackgroundColor3 = Color3.fromRGB(60, 60, 60)

local downBtn = createButton("DownBtn", "Desce", 
    UDim2.new(0, 20, 0.5, 20), UDim2.new(0, 60, 0, 60))
downBtn.BackgroundColor3 = Color3.fromRGB(60, 60, 60)

-- ======================== ÁREA DE ROTAÇÃO (ARRASSTO) ========================
local rotationArea = Instance.new("Frame")
rotationArea.Name = "RotationArea"
rotationArea.Size = UDim2.new(1, 0, 1, 0)
rotationArea.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
rotationArea.BackgroundTransparency = 0.9
rotationArea.BorderSizePixel = 0
rotationArea.Active = true
rotationArea.ZIndex = 0  -- atrás dos outros elementos
rotationArea.Parent = screenGui

-- ======================== FUNÇÕES DE APOIO ========================
-- Retorna a posição da cabeça do inimigo mais próximo (ou nil)
local function getNearestEnemyHead()
    local localChar = player.Character
    if not localChar then return nil end
    local localTeam = player.Team
    
    local camPos = camera.CFrame.Position
    local camLook = camera.CFrame.LookVector
    local bestHead = nil
    local bestDist = 500 -- distância máxima
    
    for _, otherPlayer in ipairs(Players:GetPlayers()) do
        if otherPlayer ~= player then
            local otherChar = otherPlayer.Character
            if otherChar then
                -- Verifica time
                if localTeam and otherPlayer.Team == localTeam then continue end
                local humanoid = otherChar:FindFirstChild("Humanoid")
                if humanoid and humanoid.Health > 0 then
                    local head = otherChar:FindFirstChild("Head")
                    if head then
                        local headPos = head.Position
                        local dir = (headPos - camPos)
                        local dist = dir.Magnitude
                        if dist < bestDist then
                            -- Verifica FOV (120° = 60° de cada lado)
                            local angle = math.acos(math.clamp(camLook:Dot(dir.Unit), -1, 1))
                            if angle <= math.rad(60) then
                                -- Raycast de visibilidade
                                local rayParams = RaycastParams.new()
                                rayParams.FilterDescendantsInstances = {localChar, otherChar}
                                rayParams.FilterType = Enum.RaycastFilterType.Blacklist
                                local rayResult = Workspace:Raycast(camPos, dir, rayParams)
                                if not rayResult then -- sem obstrução
                                    bestHead = headPos
                                    bestDist = dist
                                end
                            end
                        end
                    end
                end
            end
        end
    end
    return bestHead
end

-- Limpa tudo e restaura a câmera
local function cleanup()
    if heartbeatConn then
        heartbeatConn:Disconnect()
        heartbeatConn = nil
    end
    -- Restaura câmera ao padrão
    if freecamActive then
        freecamActive = false
        if player.Character then
            local hum = player.Character:FindFirstChild("Humanoid")
            if hum then
                camera.CameraType = Enum.CameraType.Custom
                camera.CameraSubject = hum
            else
                camera.CameraType = Enum.CameraType.Custom
                camera.CameraSubject = player.Character
            end
        else
            camera.CameraType = Enum.CameraType.Custom
        end
    end
    -- Remove GUI
    if screenGui then
        screenGui:Destroy()
    end
end

-- Garantir limpeza ao remover o script
script.Destroying:Connect(cleanup)

-- ======================== CONEXÕES DE EVENTOS DA GUI ========================
-- Alterna Freecam
freecamBtn.Activated:Connect(function()
    freecamActive = not freecamActive
    if freecamActive then
        freecamBtn.Text = "Freecam ON"
        freecamBtn.BackgroundColor3 = Color3.fromRGB(50, 200, 50)
        -- Desacopla câmera
        camera.CameraType = Enum.CameraType.Scriptable
        -- Não define CameraSubject para evitar conflitos
    else
        freecamBtn.Text = "Freecam OFF"
        freecamBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
        -- Reacopla câmera
        if player.Character then
            local hum = player.Character:FindFirstChild("Humanoid")
            if hum then
                camera.CameraType = Enum.CameraType.Custom
                camera.CameraSubject = hum
            else
                camera.CameraType = Enum.CameraType.Custom
                camera.CameraSubject = player.Character
            end
        else
            camera.CameraType = Enum.CameraType.Custom
        end
    end
end)

-- Alterna Aim Assist
aimBtn.Activated:Connect(function()
    aimActive = not aimActive
    if aimActive then
        aimBtn.Text = "Aim ON"
        aimBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
    else
        aimBtn.Text = "Aim OFF"
        aimBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 200)
    end
end)

-- Ajuste de velocidade
speedUpBtn.Activated:Connect(function()
    moveSpeed = math.min(moveSpeed + 4, 100)
    speedLabel.Text = "Vel: " .. moveSpeed
end)
speedDownBtn.Activated:Connect(function()
    moveSpeed = math.max(moveSpeed - 4, 2)
    speedLabel.Text = "Vel: " .. moveSpeed
end)

-- Alterna círculo FOV
fovToggleBtn.Activated:Connect(function()
    fovCircleVisible = not fovCircleVisible
    fovCircle.Visible = fovCircleVisible
    fovToggleBtn.Text = fovCircleVisible and "FOV: ON" or "FOV: OFF"
end)

-- Botões de altura (SOBE / DESCE)
upBtn.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.Touch then
        upPressed = true
    end
end)
upBtn.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.Touch then
        upPressed = false
    end
end)
downBtn.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.Touch then
        downPressed = true
    end
end)
downBtn.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.Touch then
        downPressed = false
    end
end)

-- ======================== JOYSTICK (toque dedicado) ========================
local joystickRadius = 50 -- raio máximo dentro da base (150x150 -> centro em 75,75)
joystickBase.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.Touch then
        -- centraliza o knob visualmente
        knob.Position = UDim2.new(0.5, -25, 0.5, -25)
        joystickDirection = Vector2.new(0, 0)
    end
end)
joystickBase.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.Touch then
        local pos = input.Position - joystickBase.AbsolutePosition - joystickBase.AbsoluteSize/2
        local clamped = Vector2.new(math.clamp(pos.X, -joystickRadius, joystickRadius),
            math.clamp(pos.Y, -joystickRadius, joystickRadius))
        -- Atualiza knob visual
        knob.Position = UDim2.new(0, clamped.X + joystickBase.AbsoluteSize.X/2 - 25, 
            0, clamped.Y + joystickBase.AbsoluteSize.Y/2 - 25)
        -- Calcula direção: X->direita/esquerda, Y->frente/trás (inverte Y da tela)
        joystickDirection = Vector2.new(clamped.X / joystickRadius, -clamped.Y / joystickRadius)
    end
end)
joystickBase.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.Touch then
        knob.Position = UDim2.new(0.5, -25, 0.5, -25)
        joystickDirection = Vector2.new(0, 0)
    end
end)

-- ======================== ÁREA DE ROTAÇÃO (arrasto) ========================
rotationArea.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.Touch and freecamActive then
        rotationTouch = input
        initialTouchPos = input.Position
        initialCamCFrame = camera.CFrame
    end
end)
rotationArea.InputChanged:Connect(function(input)
    if input == rotationTouch and freecamActive then
        local delta = input.Position - initialTouchPos
        -- Sensibilidade aplicada ao ângulo (graus por pixel)
        local yaw = delta.X * rotationSensitivity
        local pitch = delta.Y * rotationSensitivity
        -- Aplica rotação: yaw (eixo Y global), pitch (eixo X local)
        local newCF = initialCamCFrame * CFrame.Angles(0, math.rad(-yaw), 0) 
                     * CFrame.Angles(math.rad(-pitch), 0, 0)
        camera.CFrame = newCF
    end
end)
rotationArea.InputEnded:Connect(function(input)
    if input == rotationTouch then
        rotationTouch = nil
        initialTouchPos = nil
        initialCamCFrame = nil
    end
end)

-- ======================== LOOP PRINCIPAL (movimento + aim) ========================
local function onHeartbeat(deltaTime)
    if not freecamActive then return end

    local camCF = camera.CFrame
    -- Movimento relativo à câmera (joystick)
    local moveDir = joystickDirection -- X: right/left, Y: forward/back
    local forward = camCF.LookVector
    local right = camCF.RightVector
    local up = Vector3.new(0, 1, 0)

    local worldMove = (forward * moveDir.Y + right * moveDir.X) * moveSpeed * deltaTime
    worldMove = worldMove + up * ((upPressed and 1 or 0) - (downPressed and 1 or 0)) * moveSpeed * deltaTime

    camera.CFrame = camCF + worldMove

    -- Aim assist
    if aimActive then
        local headPos = getNearestEnemyHead()
        if headPos then
            local targetLook = (headPos - camCF.Position).Unit
            local desiredCFrame = CFrame.lookAt(camCF.Position, camCF.Position + targetLook)
            camera.CFrame = camCF:Lerp(desiredCFrame, aimSmoothing)
        end
    end
end

-- Inicia o loop
heartbeatConn = RunService.Heartbeat:Connect(onHeartbeat)

-- ======================== FINALIZAÇÃO ========================
-- (cleanup já registrada no Destroying)
print("FreecamAimAssist carregado com sucesso. Use os botões na tela.")
