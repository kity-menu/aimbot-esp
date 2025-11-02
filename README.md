---------------------------------------------------------------------
-- SERVIÇOS BÁSICOS
---------------------------------------------------------------------
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

---------------------------------------------------------------------
-- CONFIGURAÇÕES INICIAIS
---------------------------------------------------------------------
local CHECK_INTERVAL = 1
local LINE_THICKNESS = 2
local RGB_SPEED = 2
local ESP_ENABLED = true
local AIMBOT_ENABLED = false
local isAiming = false

---------------------------------------------------------------------
-- VARIÁVEIS DE CONTROLE
---------------------------------------------------------------------
local highlights = {}
local lines = {}

---------------------------------------------------------------------
-- FUNÇÕES UTILITÁRIAS DO ESP
---------------------------------------------------------------------
local function createHighlight(character)
	local highlight = Instance.new("Highlight")
	highlight.FillTransparency = 0.75
	highlight.OutlineTransparency = 0
	highlight.Parent = character
	return highlight
end

local function createLine()
	local line = Drawing.new("Line")
	line.Thickness = LINE_THICKNESS
	line.Visible = true
	return line
end

local function isEnemy(player)
	if not LocalPlayer.Team or not player.Team then
		return true
	end
	return player.Team ~= LocalPlayer.Team
end

local function updateEsp()
	for _, player in ipairs(Players:GetPlayers()) do
		if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
			local enemy = isEnemy(player)
			local char = player.Character

			if not highlights[player] then
				local h = createHighlight(char)
				highlights[player] = h
			elseif highlights[player].Parent ~= char then
				highlights[player].Parent = char
			end

			highlights[player].Enabled = enemy and ESP_ENABLED

			if not lines[player] then
				lines[player] = createLine()
			end
		end
	end
end

local function onPlayerRemoving(player)
	if highlights[player] then
		highlights[player]:Destroy()
		highlights[player] = nil
	end
	if lines[player] then
		lines[player]:Remove()
		lines[player] = nil
	end
end

task.spawn(function()
	while true do
		if ESP_ENABLED then
			updateEsp()
		end
		task.wait(CHECK_INTERVAL)
	end
end)

Players.PlayerRemoving:Connect(onPlayerRemoving)

---------------------------------------------------------------------
-- LOOP DE ATUALIZAÇÃO ESP
---------------------------------------------------------------------
RunService.RenderStepped:Connect(function()
	if not ESP_ENABLED then
		for _, line in pairs(lines) do
			line.Visible = false
		end
		for _, highlight in pairs(highlights) do
			highlight.Enabled = false
		end
		return
	end

	local timeNow = tick() * RGB_SPEED
	local r = math.sin(timeNow) * 0.5 + 0.5
	local g = math.sin(timeNow + 2) * 0.5 + 0.5
	local b = math.sin(timeNow + 4) * 0.5 + 0.5
	local color = Color3.new(r, g, b)

	for player, highlight in pairs(highlights) do
		if highlight and highlight.Enabled then
			highlight.FillColor = color
			highlight.OutlineColor = color
		end
	end

	for player, line in pairs(lines) do
		if player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
			local root = player.Character.HumanoidRootPart
			local pos, onScreen = Camera:WorldToViewportPoint(root.Position)
			if onScreen then
				line.From = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y)
				line.To = Vector2.new(pos.X, pos.Y)
				line.Color = color
				line.Thickness = LINE_THICKNESS
				line.Visible = true
			else
				line.Visible = false
			end
		else
			line.Visible = false
		end
	end
end)

---------------------------------------------------------------------
-- FUNÇÕES DO AIMBOT
---------------------------------------------------------------------
local function getClosestTarget()
	local closestTarget = nil
	local smallestDistance = math.huge

	for _, targetPlayer in ipairs(Players:GetPlayers()) do
		if targetPlayer ~= LocalPlayer and targetPlayer.Character and targetPlayer.Character:FindFirstChild("Head") then
			if isEnemy(targetPlayer) then
				local head = targetPlayer.Character.Head
				local screenPos, onScreen = Camera:WorldToScreenPoint(head.Position)
				if onScreen then
					local distance = (Vector2.new(screenPos.X, screenPos.Y) - Camera.ViewportSize / 2).Magnitude
					if distance < smallestDistance then
						smallestDistance = distance
						closestTarget = head
					end
				end
			end
		end
	end

	return closestTarget
end

UserInputService.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton2 then
		isAiming = true
	end
end)

UserInputService.InputEnded:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton2 then
		isAiming = false
	end
end)

RunService.RenderStepped:Connect(function()
	if AIMBOT_ENABLED and isAiming then
		local target = getClosestTarget()
		if target then
			Camera.CFrame = CFrame.new(Camera.CFrame.Position, target.Position)
		end
	end
end)

---------------------------------------------------------------------
-- INTERFACE NEON RGB
---------------------------------------------------------------------
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "ESP_Interface"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = game:GetService("CoreGui")

local Frame = Instance.new("Frame")
Frame.Size = UDim2.new(0, 280, 0, 230)
Frame.Position = UDim2.new(0.02, 0, 0.3, 0)
Frame.BackgroundColor3 = Color3.new(0, 0, 0)
Frame.BackgroundTransparency = 0.3
Frame.BorderSizePixel = 0
Frame.Parent = ScreenGui
Instance.new("UICorner", Frame).CornerRadius = UDim.new(0, 15)

local UIStroke = Instance.new("UIStroke", Frame)
UIStroke.Thickness = 2
UIStroke.Color = Color3.new(1, 0, 0)

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, 0, 0, 30)
Title.BackgroundTransparency = 1
Title.Text = "✨ Painel Neon RGB ✨"
Title.Font = Enum.Font.GothamBold
Title.TextSize = 18
Title.TextColor3 = Color3.new(1, 1, 1)
Title.Parent = Frame

local ToggleESP = Instance.new("TextButton")
ToggleESP.Size = UDim2.new(1, -20, 0, 30)
ToggleESP.Position = UDim2.new(0, 10, 0, 40)
ToggleESP.Text = "ESP: ON"
ToggleESP.Font = Enum.Font.GothamBold
ToggleESP.TextSize = 16
ToggleESP.BackgroundColor3 = Color3.new(0.1, 0.1, 0.1)
ToggleESP.TextColor3 = Color3.new(0, 1, 0)
ToggleESP.Parent = Frame
Instance.new("UICorner", ToggleESP).CornerRadius = UDim.new(0, 10)

local ToggleAimbot = Instance.new("TextButton")
ToggleAimbot.Size = UDim2.new(1, -20, 0, 30)
ToggleAimbot.Position = UDim2.new(0, 10, 0, 75)
ToggleAimbot.Text = "Aimbot: OFF"
ToggleAimbot.Font = Enum.Font.GothamBold
ToggleAimbot.TextSize = 16
ToggleAimbot.BackgroundColor3 = Color3.new(0.1, 0.1, 0.1)
ToggleAimbot.TextColor3 = Color3.new(1, 0, 0)
ToggleAimbot.Parent = Frame
Instance.new("UICorner", ToggleAimbot).CornerRadius = UDim.new(0, 10)

local SpeedLabel = Instance.new("TextLabel")
SpeedLabel.Size = UDim2.new(1, -20, 0, 25)
SpeedLabel.Position = UDim2.new(0, 10, 0, 115)
SpeedLabel.Text = "Velocidade RGB: " .. RGB_SPEED
SpeedLabel.Font = Enum.Font.Gotham
SpeedLabel.TextSize = 14
SpeedLabel.BackgroundTransparency = 1
SpeedLabel.TextColor3 = Color3.new(1, 1, 1)
SpeedLabel.Parent = Frame

local SpeedBox = Instance.new("TextBox")
SpeedBox.Size = UDim2.new(1, -20, 0, 25)
SpeedBox.Position = UDim2.new(0, 10, 0, 140)
SpeedBox.Text = tostring(RGB_SPEED)
SpeedBox.Font = Enum.Font.Gotham
SpeedBox.TextSize = 14
SpeedBox.BackgroundColor3 = Color3.new(0.1, 0.1, 0.1)
SpeedBox.TextColor3 = Color3.new(1, 1, 1)
SpeedBox.Parent = Frame
Instance.new("UICorner", SpeedBox).CornerRadius = UDim.new(0, 8)

---------------------------------------------------------------------
-- INTERAÇÕES
---------------------------------------------------------------------
ToggleESP.MouseButton1Click:Connect(function()
	ESP_ENABLED = not ESP_ENABLED
	if ESP_ENABLED then
		ToggleESP.Text = "ESP: ON"
		ToggleESP.TextColor3 = Color3.new(0, 1, 0)
	else
		ToggleESP.Text = "ESP: OFF"
		ToggleESP.TextColor3 = Color3.new(1, 0, 0)
	end
end)

ToggleAimbot.MouseButton1Click:Connect(function()
	AIMBOT_ENABLED = not AIMBOT_ENABLED
	if AIMBOT_ENABLED then
		ToggleAimbot.Text = "Aimbot: ON"
		ToggleAimbot.TextColor3 = Color3.new(0, 1, 0)
	else
		ToggleAimbot.Text = "Aimbot: OFF"
		ToggleAimbot.TextColor3 = Color3.new(1, 0, 0)
	end
end)

SpeedBox.FocusLost:Connect(function()
	local val = tonumber(SpeedBox.Text)
	if val then
		RGB_SPEED = val
		SpeedLabel.Text = "Velocidade RGB: " .. val
	end
end)

---------------------------------------------------------------------
-- EFEITO RGB DO PAINEL
---------------------------------------------------------------------
RunService.RenderStepped:Connect(function()
	local t = tick() * 2
	local r = math.sin(t) * 0.5 + 0.5
	local g = math.sin(t + 2) * 0.5 + 0.5
	local b = math.sin(t + 4) * 0.5 + 0.5
	UIStroke.Color = Color3.new(r, g, b)
	Title.TextColor3 = Color3.new(r, g, b)
end)
