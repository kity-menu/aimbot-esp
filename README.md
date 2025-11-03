-- Serviços
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

-- Config / Estados
local LINE_THICKNESS = 2
local RGB_SPEED = 2

local ESP_ENABLED = true
local AIMBOT_ENABLED = false
local NOCOLIP_ENABLED = false
local FLY_ENABLED = false
local FLY_SPEED = 70
local WALK_SPEED = 16
local WALK_ENABLED = true

-- Containers ESP
local lines, cornerBoxes, healthBars, nameTags = {}, {}, {}, {}

-- ESP helpers
local function newLine()
	local L = Drawing.new("Line")
	L.Visible = false
	L.Thickness = LINE_THICKNESS
	return L
end

local function newSquare()
	local S = Drawing.new("Square")
	S.Visible = false
	S.Filled = false
	S.Thickness = 1.5
	return S
end

local function newText(text,size)
	local T = Drawing.new("Text")
	T.Visible = false
	T.Text = text or ""
	T.Size = size or 14
	T.Center = true
	T.Outline = true
	return T
end

local function newFilledRect()
	local R = Drawing.new("Square")
	R.Visible = false
	R.Filled = true
	return R
end

local function isEnemy(player)
	if not LocalPlayer.Team or not player.Team then return true end
	return player.Team ~= LocalPlayer.Team
end

local function ensureVisuals(p)
	if not lines[p] then lines[p] = newLine() end
	if not cornerBoxes[p] then cornerBoxes[p] = newSquare() end
	if not healthBars[p] then healthBars[p] = newFilledRect() end
	if not nameTags[p] then nameTags[p] = newText(p.Name,16) end
end

Players.PlayerRemoving:Connect(function(p)
	if lines[p] then lines[p]:Remove() lines[p]=nil end
	if cornerBoxes[p] then cornerBoxes[p]:Remove() cornerBoxes[p]=nil end
	if healthBars[p] then healthBars[p]:Remove() healthBars[p]=nil end
	if nameTags[p] then nameTags[p]:Remove() nameTags[p]=nil end
end)

-- ESP update loop
RunService.RenderStepped:Connect(function()
	local t = tick()*RGB_SPEED
	local color = Color3.new(math.sin(t)*0.5+0.5, math.sin(t+2)*0.5+0.5, math.sin(t+4)*0.5+0.5)
	for _,p in ipairs(Players:GetPlayers()) do
		if p ~= LocalPlayer and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
			ensureVisuals(p)
			local char = p.Character
			local root = char.HumanoidRootPart
			local pos2, onscreen = Camera:WorldToViewportPoint(root.Position)
			if onscreen and ESP_ENABLED then
				local screen = Vector2.new(pos2.X,pos2.Y)
				-- Linha
				local line = lines[p]
				line.From = Vector2.new(Camera.ViewportSize.X/2,Camera.ViewportSize.Y)
				line.To = screen
				line.Color = color
				line.Visible = true

				-- Corner Box
				local boxSize = 60
				local box = cornerBoxes[p]
				box.Size = Vector2.new(boxSize,boxSize)
				box.Position = Vector2.new(screen.X-boxSize/2,screen.Y-boxSize/2)
				box.Color = color
				box.Visible = true

				-- Nome acima
				local name = nameTags[p]
				name.Text = p.Name
				name.Size = 16
				name.Position = Vector2.new(screen.X,screen.Y-boxSize/2-12)
				name.Color = Color3.new(1,1,1)
				name.Visible = true

				-- Barra de vida horizontal abaixo
				local humanoid = char:FindFirstChildOfClass("Humanoid")
				local hb = healthBars[p]
				if humanoid then
					local percent = math.clamp(humanoid.Health/humanoid.MaxHealth,0,1)
					hb.Size = Vector2.new(boxSize*percent,4)
					hb.Position = Vector2.new(screen.X-boxSize/2,screen.Y+boxSize/2+4)
					hb.Color = Color3.new(1-percent,percent,0)
					hb.Visible = true
				else
					hb.Visible = false
				end
			else
				if lines[p] then lines[p].Visible=false end
				if cornerBoxes[p] then cornerBoxes[p].Visible=false end
				if nameTags[p] then nameTags[p].Visible=false end
				if healthBars[p] then healthBars[p].Visible=false end
			end
		end
	end
end)

-- WalkSpeed
local function applyWalkSpeed(speed)
	local hum = LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
	if hum then hum.WalkSpeed = speed end
end

-- Noclip
local function applyNoclip()
	if not NOCOLIP_ENABLED then return end
	local char = LocalPlayer.Character
	if char then
		for _,part in ipairs(char:GetDescendants()) do
			if part:IsA("BasePart") then
				part.CanCollide = false
			end
		end
	end
end
RunService.Stepped:Connect(applyNoclip)

-- AIMBOT
local isAiming = false
local function getClosestTarget()
	local closest=nil
	local smallest=math.huge
	for _,p in ipairs(Players:GetPlayers()) do
		if p~=LocalPlayer and p.Character and p.Character:FindFirstChild("Head") and isEnemy(p) then
			local head = p.Character.Head
			local screenPos,onScreen = Camera:WorldToViewportPoint(head.Position)
			if onScreen then
				local dist = (Vector2.new(screenPos.X,screenPos.Y)-Camera.ViewportSize/2).Magnitude
				if dist<smallest then
					smallest=dist
					closest=head
				end
			end
		end
	end
	return closest
end

UserInputService.InputBegan:Connect(function(i)
	if i.UserInputType==Enum.UserInputType.MouseButton2 then isAiming=true end
end)
UserInputService.InputEnded:Connect(function(i)
	if i.UserInputType==Enum.UserInputType.MouseButton2 then isAiming=false end
end)

RunService.RenderStepped:Connect(function()
	if AIMBOT_ENABLED and isAiming then
		local t=getClosestTarget()
		if t then
			Camera.CFrame = CFrame.new(Camera.CFrame.Position, t.Position)
		end
	end
end)

-- FLY
local BV
local flyInput={W=false,A=false,S=false,D=false,Up=false,Down=false}
local function enableFly()
	local char = LocalPlayer.Character
	if not char then return end
	local hrp = char:FindFirstChild("HumanoidRootPart")
	if not hrp then return end
	if not BV then
		BV = Instance.new("BodyVelocity")
		BV.MaxForce = Vector3.new(1e5,1e5,1e5)
		BV.P = 1e4
		BV.Velocity = Vector3.new(0,0,0)
		BV.Parent = hrp
	end
end
local function disableFly()
	if BV then BV:Destroy() BV=nil end
end
RunService.RenderStepped:Connect(function()
	if FLY_ENABLED and BV and LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
		local hrp = LocalPlayer.Character.HumanoidRootPart
		local camCF = Camera.CFrame
		local moveVec = Vector3.new()
		if flyInput.W then moveVec = moveVec + camCF.LookVector end
		if flyInput.S then moveVec = moveVec - camCF.LookVector end
		if flyInput.A then moveVec = moveVec - camCF.RightVector end
		if flyInput.D then moveVec = moveVec + camCF.RightVector end
		if flyInput.Up then moveVec = moveVec + Vector3.yAxis end
		if flyInput.Down then moveVec = moveVec - Vector3.yAxis end
		if moveVec.Magnitude>0 then moveVec = moveVec.Unit * FLY_SPEED end
		BV.Velocity = moveVec
	end
end)
UserInputService.InputBegan:Connect(function(input, processed)
	if processed then return end
	if input.KeyCode==Enum.KeyCode.W then flyInput.W=true end
	if input.KeyCode==Enum.KeyCode.S then flyInput.S=true end
	if input.KeyCode==Enum.KeyCode.A then flyInput.A=true end
	if input.KeyCode==Enum.KeyCode.D then flyInput.D=true end
	if input.KeyCode==Enum.KeyCode.Space then flyInput.Up=true end
	if input.KeyCode==Enum.KeyCode.LeftControl or input.KeyCode==Enum.KeyCode.C then flyInput.Down=true end
end)
UserInputService.InputEnded:Connect(function(input)
	if input.KeyCode==Enum.KeyCode.W then flyInput.W=false end
	if input.KeyCode==Enum.KeyCode.S then flyInput.S=false end
	if input.KeyCode==Enum.KeyCode.A then flyInput.A=false end
	if input.KeyCode==Enum.KeyCode.D then flyInput.D=false end
	if input.KeyCode==Enum.KeyCode.Space then flyInput.Up=false end
	if input.KeyCode==Enum.KeyCode.LeftControl or input.KeyCode==Enum.KeyCode.C then flyInput.Down=false end
end)

-- GUI
local function CreateGui()
	local ScreenGui = Instance.new("ScreenGui")
	ScreenGui.Name="ControlPanel"
	ScreenGui.ResetOnSpawn=false
	ScreenGui.Parent=game:GetService("CoreGui")

	local Main=Instance.new("Frame",ScreenGui)
	Main.Size=UDim2.new(0,420,0,360)
	Main.Position=UDim2.new(0.03,0,0.15,0)
	Main.BackgroundColor3=Color3.fromRGB(18,18,18)
	Instance.new("UICorner",Main).CornerRadius=UDim.new(0,12)
	Instance.new("UIStroke",Main).Thickness=2

	local Title=Instance.new("TextLabel",Main)
	Title.Size=UDim2.new(1,0,0,34)
	Title.Position=UDim2.new(0,0,0,0)
	Title.BackgroundTransparency=1
	Title.Text="✔ Reals Xiter"
	Title.Font=Enum.Font.GothamBold
	Title.TextSize=18
	Title.TextColor3=Color3.fromRGB(235,235,235)

	local TabsBar=Instance.new("Frame",Main)
	TabsBar.Size=UDim2.new(1,-20,0,36)
	TabsBar.Position=UDim2.new(0,10,0,40)
	TabsBar.BackgroundTransparency=1

	local function makeTab(text,x)
		local btn=Instance.new("TextButton",TabsBar)
		btn.Size=UDim2.new(0,120,0,28)
		btn.Position=UDim2.new(0,10+x,0,4)
		btn.Text=text
		btn.Font=Enum.Font.GothamBold
		btn.TextSize=14
		btn.BackgroundColor3=Color3.fromRGB(40,40,40)
		btn.TextColor3=Color3.fromRGB(200,200,200)
		Instance.new("UICorner",btn).CornerRadius=UDim.new(0,8)
		return btn
	end

	local tabESP=makeTab("ESP",0)
	local tabAimbot=makeTab("Aimbot",130)
	local tabMiscs=makeTab("Miscs",260)

	local ContentRoot=Instance.new("Frame",Main)
	ContentRoot.Size=UDim2.new(1,-20,1,-90)
	ContentRoot.Position=UDim2.new(0,10,0,80)
	ContentRoot.BackgroundTransparency=1

	local ESPPage=Instance.new("Frame",ContentRoot)
	ESPPage.Size=UDim2.new(1,0,1,0)
	ESPPage.BackgroundTransparency=1

	local AimbotPage=Instance.new("Frame",ContentRoot)
	AimbotPage.Size=UDim2.new(1,0,1,0)
	AimbotPage.BackgroundTransparency=1
	AimbotPage.Visible=false

	local MiscsPage=Instance.new("Frame",ContentRoot)
	MiscsPage.Size=UDim2.new(1,0,1,0)
	MiscsPage.BackgroundTransparency=1
	MiscsPage.Visible=false

	-- ESP Toggle
	local espBtn=Instance.new("TextButton",ESPPage)
	espBtn.Size=UDim2.new(0,160,0,32)
	espBtn.Position=UDim2.new(0,0,0,0)
	espBtn.Text=ESP_ENABLED and "ESP: ON" or "ESP: OFF"
	espBtn.Font=Enum.Font.GothamBold
	espBtn.TextSize=16
	espBtn.TextColor3=ESP_ENABLED and Color3.new(0,1,0) or Color3.new(1,0,0)
	espBtn.BackgroundColor3=Color3.fromRGB(45,45,45)
	Instance.new("UICorner",espBtn).CornerRadius=UDim.new(0,8)
	espBtn.MouseButton1Click:Connect(function()
		ESP_ENABLED=not ESP_ENABLED
		espBtn.Text=ESP_ENABLED and "ESP: ON" or "ESP: OFF"
		espBtn.TextColor3=ESP_ENABLED and Color3.new(0,1,0) or Color3.new(1,0,0)
	end)

	-- AIMBOT Toggle
	local aimBtn=Instance.new("TextButton",AimbotPage)
	aimBtn.Size=UDim2.new(0,160,0,32)
	aimBtn.Position=UDim2.new(0,0,0,0)
	aimBtn.Text=AIMBOT_ENABLED and "Aimbot: ON" or "Aimbot: OFF"
	aimBtn.Font=Enum.Font.GothamBold
	aimBtn.TextSize=16
	aimBtn.TextColor3=AIMBOT_ENABLED and Color3.new(0,1,0) or Color3.new(1,0,0)
	aimBtn.BackgroundColor3=Color3.fromRGB(45,45,45)
	Instance.new("UICorner",aimBtn).CornerRadius=UDim.new(0,8)
	aimBtn.MouseButton1Click:Connect(function()
		AIMBOT_ENABLED=not AIMBOT_ENABLED
		aimBtn.Text=AIMBOT_ENABLED and "Aimbot: ON" or "Aimbot: OFF"
		aimBtn.TextColor3=AIMBOT_ENABLED and Color3.new(0,1,0) or Color3.new(1,0,0)
	end)

	-- Miscs: Noclip
	local noclipBtn=Instance.new("TextButton",MiscsPage)
	noclipBtn.Size=UDim2.new(0,160,0,32)
	noclipBtn.Position=UDim2.new(0,0,0,0)
	noclipBtn.Text=NOCOLIP_ENABLED and "Noclip: ON" or "Noclip: OFF"
	noclipBtn.Font=Enum.Font.GothamBold
	noclipBtn.TextSize=16
	noclipBtn.TextColor3=NOCOLIP_ENABLED and Color3.new(0,1,0) or Color3.new(1,0,0)
	noclipBtn.BackgroundColor3=Color3.fromRGB(45,45,45)
	Instance.new("UICorner",noclipBtn).CornerRadius=UDim.new(0,8)
	noclipBtn.MouseButton1Click:Connect(function()
		NOCOLIP_ENABLED = not NOCOLIP_ENABLED
		noclipBtn.Text = NOCOLIP_ENABLED and "Noclip: ON" or "Noclip: OFF"
		noclipBtn.TextColor3 = NOCOLIP_ENABLED and Color3.new(0,1,0) or Color3.new(1,0,0)
	end)

	-- Miscs: WalkSpeed
	local walkBtn=Instance.new("TextButton",MiscsPage)
	walkBtn.Size=UDim2.new(0,160,0,32)
	walkBtn.Position=UDim2.new(0,0,0,44)
	walkBtn.Text=WALK_ENABLED and "WalkSpeed: ON" or "WalkSpeed: OFF"
	walkBtn.Font=Enum.Font.GothamBold
	walkBtn.TextSize=16
	walkBtn.TextColor3=WALK_ENABLED and Color3.new(0,1,0) or Color3.new(1,0,0)
	walkBtn.BackgroundColor3=Color3.fromRGB(45,45,45)
	Instance.new("UICorner",walkBtn).CornerRadius=UDim.new(0,8)
	walkBtn.MouseButton1Click:Connect(function()
		WALK_ENABLED = not WALK_ENABLED
		walkBtn.Text = WALK_ENABLED and "WalkSpeed: ON" or "WalkSpeed: OFF"
		walkBtn.TextColor3 = WALK_ENABLED and Color3.new(0,1,0) or Color3.new(1,0,0)
		applyWalkSpeed(WALK_ENABLED and WALK_SPEED or 16)
	end)

	local walkBox=Instance.new("TextBox",MiscsPage)
	walkBox.Position=UDim2.new(0,170,0,44)
	walkBox.Size=UDim2.new(0,80,0,32)
	walkBox.Text=tostring(WALK_SPEED)
	walkBox.BackgroundColor3=Color3.fromRGB(40,40,40)
	Instance.new("UICorner",walkBox).CornerRadius=UDim.new(0,6)
	walkBox.FocusLost:Connect(function()
		local v=tonumber(walkBox.Text)
		if v and v>0 and v<=500 then
			WALK_SPEED=v
			if WALK_ENABLED then applyWalkSpeed(WALK_SPEED) end
		else walkBox.Text=tostring(WALK_SPEED) end
	end)

	-- Miscs: Fly
	local flyBtn=Instance.new("TextButton",MiscsPage)
	flyBtn.Size=UDim2.new(0,160,0,32)
	flyBtn.Position=UDim2.new(0,0,0,88)
	flyBtn.Text=FLY_ENABLED and "Fly: ON" or "Fly: OFF"
	flyBtn.Font=Enum.Font.GothamBold
	flyBtn.TextSize=16
	flyBtn.TextColor3=FLY_ENABLED and Color3.new(0,1,0) or Color3.new(1,0,0)
	flyBtn.BackgroundColor3=Color3.fromRGB(45,45,45)
	Instance.new("UICorner",flyBtn).CornerRadius=UDim.new(0,8)
	flyBtn.MouseButton1Click:Connect(function()
		FLY_ENABLED=not FLY_ENABLED
		flyBtn.Text=FLY_ENABLED and "Fly: ON" or "Fly: OFF"
		flyBtn.TextColor3=FLY_ENABLED and Color3.new(0,1,0) or Color3.new(1,0,0)
		if FLY_ENABLED then enableFly() else disableFly() end
	end)

	local flyBox=Instance.new("TextBox",MiscsPage)
	flyBox.Position=UDim2.new(0,170,0,88)
	flyBox.Size=UDim2.new(0,80,0,32)
	flyBox.Text=tostring(FLY_SPEED)
	flyBox.BackgroundColor3=Color3.fromRGB(40,40,40)
	Instance.new("UICorner",flyBox).CornerRadius=UDim.new(0,6)
	flyBox.FocusLost:Connect(function()
		local v=tonumber(flyBox.Text)
		if v and v>0 and v<=1000 then
			FLY_SPEED=v
		else flyBox.Text=tostring(FLY_SPEED) end
	end)

	-- Tabs
	tabESP.MouseButton1Click:Connect(function()
		ESPPage.Visible=true
		AimbotPage.Visible=false
		MiscsPage.Visible=false
	end)
	tabAimbot.MouseButton1Click:Connect(function()
		ESPPage.Visible=false
		AimbotPage.Visible=true
		MiscsPage.Visible=false
	end)
	tabMiscs.MouseButton1Click:Connect(function()
		ESPPage.Visible=false
		AimbotPage.Visible=false
		MiscsPage.Visible=true
	end)

	return ScreenGui
end

CreateGui()
applyWalkSpeed(WALK_ENABLED and WALK_SPEED or 16)
