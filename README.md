--[[
	LEGO PARKOUR + VISUAL DESTRUCTION + TOYBOX ENGINE V8 — Cinematic Toys & Drivable Vehicles
	Tipo: LocalScript
	Local: StarterPlayer > StarterPlayerScripts

	ARQUITETURA: rig visual SEPARADO (somente cliente).
	- O personagem R15 original continua sendo o corpo físico (movimento, câmera, controles, pulo).
	- Uma minifigura de peças (Part) é criada dentro do Character e presa ao HumanoidRootPart por UM Motor6D raiz.
	- As peças internas se ligam por Motor6D (articulações animadas) e Weld (detalhes fixos).
	- Todas as peças são Massless, sem colisão, sem toque e sem consulta (CanQuery=false).
	- O avatar original só é ocultado (Transparency local) DEPOIS de construir, validar e verificar o rig.
	- As animações são procedurais (Motor6D.Transform, um único loop de render). Nomes "MF_*" evitam
	  qualquer conflito com o Animator/Animate padrão.
	Base V3 preservada. V8 acrescenta modelos LEGO detalhados, interações com preparação/impacto/recuperação,
	VFX cinematográficos e carro/motocicleta dirigíveis com física local, direção, rodas, suspensão simplificada e HUD touch.
	Destruição apenas visual/local: não remove peças reais nem altera colisões compartilhadas.
	Status: edição estática; precisa ser executado e ajustado no Roblox Studio antes de considerar as mecânicas verificadas.
]]

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")
local TweenService = game:GetService("TweenService")
local Debris = game:GetService("Debris")

local player = Players.LocalPlayer
local TAG = "[LegoEngine]"

----------------------------------------------------------------------
-- CONFIG
----------------------------------------------------------------------
local CONFIG = {
	StartEnabled = true,
	UnitScale = 0.13,        -- studs por "mm" LEGO (minifigura ~5 studs de altura)
	EnableDoubleJump = true,
	WaitTimeout = 12,
	MaxRigParts = 140,
	FootOffsetTweak = 0,     -- ajuste manual da altura dos pés (studs; negativo = desce)
	PunchCooldown = 0.3,
	DoubleJumpMinAirTime = 0.12,
	ButtonBottom = 200,      -- distância dos botões de ação até a base da tela (acima do botão de pulo)
	ParkourScanDistance = 3.2,
	BreakScanRadius = 34,
	BreakScanInterval = 0.9,
	BreakMaxHighlights = 60,
	LedgeShimmySpeed = 2.1,
	WallRunDuration = 0.52,
	WallRunSpeed = 17,
	WallRunCooldown = 0.75,
	BreakHitDistance = 22,
	BreakRestoreSeconds = 12,
	StaminaMax = 100,
	StaminaRegeneration = 19,
	StaminaRegenerationDelay = 0.65,
	DashSpeed = 29,
	DashDuration = 0.24,
	DashCooldown = 0.68,
	DashStaminaCost = 22,
	RollDuration = 0.62,
	RollStaminaCost = 16,
	SlideStaminaCost = 9,
	VaultStaminaCost = 8,
	MantleStaminaCost = 12,
	WallRunStaminaPerSecond = 23,
	HangStaminaPerSecond = 10,
	SenseInterval = 0.12,
	ImpactScanInterval = 0.11,
	ImpactMinSpeed = 23,
	ImpactGlobalCooldown = 0.22,
	MaxLocalFX = 95,
	MaxCrackMarks = 120,
	MaxFragmentsPerImpact = 9,
	AutoRollOnHardLanding = true,
}

local K = CONFIG.UnitScale
local rad, abs, sin, cos, exp, clamp = math.rad, math.abs, math.sin, math.cos, math.exp, math.clamp
local ST = Enum.HumanoidStateType

local function lerp(a, b, t) return a + (b - a) * t end
local function easeOut(t) return 1 - (1 - t) * (1 - t) end
local function easeIn(t) return t * t end
local function smooth(t) return t * t * (3 - 2 * t) end

local ROOT_PIVOT = CFrame.new(0, 18 * K, 0)
local ROOT_PIVOT_INV = ROOT_PIVOT:Inverse()
local HEAD_CENTER = CFrame.new(0, 32.5, 0)

-- Ganchos para o futuro sistema de combate (o soco atual é APENAS visual)
local Hooks = {
	OnPunchStrike = nil, -- function(side: number (1=direita,-1=esquerda), rootPart: BasePart)
}

----------------------------------------------------------------------
-- Utilitários
----------------------------------------------------------------------
local function newBin()
	local bin = { items = {} }
	function bin:Add(item)
		table.insert(self.items, item)
		return item
	end
	function bin:Clean()
		local items = self.items
		self.items = {}
		for i = #items, 1, -1 do
			local item = items[i]
			local kind = typeof(item)
			if kind == "RBXScriptConnection" then
				item:Disconnect()
			elseif kind == "Instance" then
				local ok, e = pcall(function() item:Destroy() end)
				if not ok then warn(TAG .. " erro ao destruir: " .. tostring(e)) end
			elseif kind == "function" then
				local ok, e = pcall(item)
				if not ok then warn(TAG .. " erro na limpeza: " .. tostring(e)) end
			end
		end
	end
	return bin
end

-- Executa uma etapa preservando a mensagem de erro original (e o traceback no console)
local function safeCall(step, fn, ...)
	local args = table.pack(...)
	local trace = nil
	local ok, res = xpcall(function()
		return fn(table.unpack(args, 1, args.n))
	end, function(e)
		trace = debug.traceback(tostring(e), 2)
		return tostring(e)
	end)
	if not ok then
		warn(TAG .. " ERRO na etapa '" .. step .. "': " .. tostring(res))
		if trace then warn(trace) end
		return false, tostring(res)
	end
	return true, res
end

local function make(class, props, parent)
	local inst = Instance.new(class)
	for k, v in pairs(props) do inst[k] = v end
	if parent then inst.Parent = parent end
	return inst
end

----------------------------------------------------------------------
-- Cores e trajes
----------------------------------------------------------------------
local COL = {
	Skin = Color3.fromRGB(245, 205, 47),
	SkinDark = Color3.fromRGB(226, 181, 30),
	Black = Color3.fromRGB(27, 42, 52),
	White = Color3.fromRGB(244, 244, 244),
	Gold = Color3.fromRGB(239, 184, 56),
	Red = Color3.fromRGB(196, 40, 28),
	DarkRed = Color3.fromRGB(140, 30, 22),
	Blue = Color3.fromRGB(13, 105, 172),
	DarkBlue = Color3.fromRGB(32, 58, 86),
	Brown = Color3.fromRGB(105, 64, 40),
	LightBrown = Color3.fromRGB(128, 84, 55),
	DarkBrown = Color3.fromRGB(70, 43, 27),
	Tan = Color3.fromRGB(215, 197, 154),
	Khaki = Color3.fromRGB(158, 138, 94),
	DarkGray = Color3.fromRGB(60, 60, 64),
}

local OUTFITS = {
	{
		Name = "Explorador", Style = "Jacket",
		Torso = COL.Brown, Arms = COL.Brown, Legs = COL.Khaki, Boots = COL.DarkBrown, Sole = COL.Black,
		Shirt = COL.Tan, Collar = COL.DarkBrown, Belt = COL.Black, Buckle = COL.Gold,
		Pocket = COL.DarkBrown, Flap = COL.LightBrown, Stitch = COL.DarkBrown,
		Strap = true, StrapColor = COL.DarkBrown,
		Headgear = "Hat", HeadgearColor = COL.Brown, HatBand = COL.Tan,
	},
	{
		Name = "Herói", Style = "Hero",
		Torso = COL.Blue, Arms = COL.Blue, Legs = COL.DarkBlue, Boots = COL.Red, Sole = COL.Black,
		Belt = COL.Gold, Buckle = COL.Red, Accent = COL.Gold, Accent2 = COL.Red,
		Headgear = "Helmet", HeadgearColor = COL.Blue,
	},
	{
		Name = "Piloto", Style = "Jacket",
		Torso = COL.Red, Arms = COL.Red, Legs = COL.DarkGray, Boots = COL.Black, Sole = COL.DarkGray,
		Shirt = COL.White, Collar = COL.DarkRed, Belt = COL.Black, Buckle = COL.Gold,
		Pocket = COL.DarkRed, Flap = COL.Red, Stitch = COL.Black,
		Strap = false,
		Headgear = "Hair", HeadgearColor = COL.DarkBrown,
	},
}

----------------------------------------------------------------------
-- Interface (mobile)
----------------------------------------------------------------------
local C_NEUTRAL = Color3.fromRGB(225, 228, 235)
local C_OK = Color3.fromRGB(120, 225, 150)
local C_WARN = Color3.fromRGB(255, 205, 90)
local C_ERR = Color3.fromRGB(255, 110, 110)

local function buildUI()
	local playerGui = player:WaitForChild("PlayerGui")
	local old = playerGui:FindFirstChild("LegoEngineUI")
	if old then old:Destroy() end

	local gui = make("ScreenGui", {
		Name = "LegoEngineUI", ResetOnSpawn = false, DisplayOrder = 50, IgnoreGuiInset = false,
	})
	pcall(function() gui.ScreenInsets = Enum.ScreenInsets.DeviceSafeInsets end)
	gui.Parent = playerGui

	local panel = make("Frame", {
		Name = "Panel", Position = UDim2.new(0, 8, 0, 6), Size = UDim2.new(0, 216, 0, 0),
		AutomaticSize = Enum.AutomaticSize.Y, BackgroundColor3 = Color3.fromRGB(24, 26, 32),
		BackgroundTransparency = 0.15, BorderSizePixel = 0,
	}, gui)
	make("UICorner", { CornerRadius = UDim.new(0, 10) }, panel)
	make("UIPadding", {
		PaddingTop = UDim.new(0, 4), PaddingBottom = UDim.new(0, 6),
		PaddingLeft = UDim.new(0, 8), PaddingRight = UDim.new(0, 8),
	}, panel)
	make("UIListLayout", { Padding = UDim.new(0, 4), SortOrder = Enum.SortOrder.LayoutOrder }, panel)

	local header = make("TextButton", {
		Size = UDim2.new(1, 0, 0, 30), BackgroundTransparency = 1, Text = "LEGO ENGINE  ▾",
		TextColor3 = Color3.fromRGB(255, 214, 10), Font = Enum.Font.GothamBold, TextSize = 14,
		TextXAlignment = Enum.TextXAlignment.Left, LayoutOrder = 1,
	}, panel)

	local body = make("Frame", {
		Size = UDim2.new(1, 0, 0, 0), AutomaticSize = Enum.AutomaticSize.Y,
		BackgroundTransparency = 1, LayoutOrder = 2,
	}, panel)
	make("UIListLayout", { Padding = UDim.new(0, 4), SortOrder = Enum.SortOrder.LayoutOrder }, body)

	local function makeLabel(order)
		return make("TextLabel", {
			Size = UDim2.new(1, 0, 0, 0), AutomaticSize = Enum.AutomaticSize.Y, BackgroundTransparency = 1,
			Font = Enum.Font.GothamMedium, TextSize = 13, TextWrapped = true,
			TextXAlignment = Enum.TextXAlignment.Left, TextColor3 = C_NEUTRAL, Text = "", LayoutOrder = order,
		}, body)
	end
	local lblInit, lblBuild, lblAnim, lblErr = makeLabel(1), makeLabel(2), makeLabel(3), makeLabel(4)
	lblErr.Visible = false

	-- Barra compacta de fôlego; fica no painel para não cobrir o joystick/jump padrão.
	local staminaWrap = make("Frame", {
		Name = "StaminaBar", Size = UDim2.new(1, 0, 0, 18), BackgroundColor3 = Color3.fromRGB(52, 55, 64),
		BorderSizePixel = 0, LayoutOrder = 5, ClipsDescendants = true,
	}, body)
	make("UICorner", { CornerRadius = UDim.new(0, 6) }, staminaWrap)
	local staminaFill = make("Frame", {
		Name = "Fill", Size = UDim2.new(1, 0, 1, 0), BackgroundColor3 = Color3.fromRGB(93, 196, 133),
		BorderSizePixel = 0,
	}, staminaWrap)
	make("UICorner", { CornerRadius = UDim.new(0, 6) }, staminaFill)
	local staminaText = make("TextLabel", {
		Name = "Label", Size = UDim2.new(1, 0, 1, 0), BackgroundTransparency = 1,
		Text = "FÔLEGO 100%", TextColor3 = Color3.new(1, 1, 1), Font = Enum.Font.GothamBold,
		TextSize = 10, ZIndex = 2,
	}, staminaWrap)

	local function makeButton(text, order, height, color)
		local b = make("TextButton", {
			Size = UDim2.new(1, 0, 0, height), BackgroundColor3 = color, Font = Enum.Font.GothamBold,
			TextSize = 14, TextColor3 = Color3.new(1, 1, 1), Text = text, AutoButtonColor = true, LayoutOrder = order,
		}, body)
		make("UICorner", { CornerRadius = UDim.new(0, 8) }, b)
		return b
	end
	local btnToggle = makeButton("Transformar: ...", 6, 40, Color3.fromRGB(60, 120, 220))
	local btnOutfit = makeButton("Traje: ...", 7, 36, Color3.fromRGB(90, 90, 105))

	local function makeAction(text, xOffset, color, bottomOffset)
		local b = make("TextButton", {
			AnchorPoint = Vector2.new(1, 1), Position = UDim2.new(1, xOffset, 1, -(bottomOffset or CONFIG.ButtonBottom)),
			Size = UDim2.new(0, 58, 0, 58), BackgroundColor3 = color, Font = Enum.Font.GothamBold,
			TextSize = 11, TextColor3 = Color3.new(1, 1, 1), Text = text, Visible = false, ZIndex = 5,
		}, gui)
		make("UICorner", { CornerRadius = UDim.new(1, 0) }, b)
		make("UIStroke", {
			Thickness = 2, Color = Color3.new(1, 1, 1), Transparency = 0.4,
			ApplyStrokeMode = Enum.ApplyStrokeMode.Border,
		}, b)
		return b
	end
	local btnPunch = makeAction("SOCO", -10, Color3.fromRGB(205, 60, 50))
	local btnGesture = makeAction("GESTO", -72, Color3.fromRGB(60, 160, 90))
	local btnSlide = makeAction("SLIDE", -134, Color3.fromRGB(46, 116, 172))
	local btnParkour = makeAction("PARKOUR", -196, Color3.fromRGB(115, 86, 185))
	local btnBreak = makeAction("QUEBRAR", -258, Color3.fromRGB(105, 105, 112))
	-- Segunda fileira de ações: evita espalhar mais botões na mesma linha horizontal.
	local btnDash = makeAction("DASH", -10, Color3.fromRGB(211, 125, 45), CONFIG.ButtonBottom + 68)
	local btnRoll = makeAction("ROLAR", -72, Color3.fromRGB(44, 145, 130), CONFIG.ButtonBottom + 68)
	local btnDrop = makeAction("SOLTAR", -134, Color3.fromRGB(156, 82, 75), CONFIG.ButtonBottom + 68)
	btnDrop.Visible = false

	header.Activated:Connect(function()
		body.Visible = not body.Visible
		header.Text = body.Visible and "LEGO ENGINE  ▾" or "LEGO ENGINE  ▸"
	end)

	local ui = { Gui = gui, Toggle = btnToggle, Outfit = btnOutfit, Punch = btnPunch, Gesture = btnGesture, Slide = btnSlide, Parkour = btnParkour, Break = btnBreak, Dash = btnDash, Roll = btnRoll, Drop = btnDrop }
	function ui.setInit(text, color)
		lblInit.Text = "Inicialização: " .. text
		lblInit.TextColor3 = color or C_NEUTRAL
	end
	function ui.setBuild(text, color)
		lblBuild.Text = "Construção: " .. text
		lblBuild.TextColor3 = color or C_NEUTRAL
		print(TAG, "Construção:", text)
	end
	function ui.setAnim(text, color)
		lblAnim.Text = "Animação: " .. text
		lblAnim.TextColor3 = color or C_NEUTRAL
	end
	function ui.setError(step, msg)
		if step == nil then
			lblErr.Visible = false
			lblErr.Text = ""
		else
			lblErr.Text = "ERRO [" .. step .. "]: " .. tostring(msg)
			lblErr.TextColor3 = C_ERR
			lblErr.Visible = true
		end
	end
	function ui.setToggleText(text) btnToggle.Text = text end
	function ui.setOutfitText(text) btnOutfit.Text = text end
	local lastStaminaPercent = -1
	function ui.setStamina(value, maxValue)
		local ratio = math.clamp((maxValue > 0 and value / maxValue or 0), 0, 1)
		local percent = math.floor(ratio * 100 + 0.5)
		if percent == lastStaminaPercent then return end
		lastStaminaPercent = percent
		staminaFill.Size = UDim2.new(ratio, 0, 1, 0)
		staminaText.Text = "FÔLEGO " .. tostring(percent) .. "%"
		staminaFill.BackgroundColor3 = ratio < 0.22 and Color3.fromRGB(205, 87, 76)
			or (ratio < 0.48 and Color3.fromRGB(218, 169, 65) or Color3.fromRGB(93, 196, 133))
	end
	function ui.setActionsVisible(v)
		btnPunch.Visible = v
		btnGesture.Visible = v
		btnSlide.Visible = v
		btnParkour.Visible = v
		btnBreak.Visible = v
		btnDash.Visible = v
		btnRoll.Visible = v
		if not v then btnDrop.Visible = false end
	end
	function ui.setParkourText(text)
		btnParkour.Text = text
		btnDrop.Visible = (text == "SUBIR")
	end
	return ui
end

local UI
do
	local ok, res = pcall(buildUI)
	if ok then
		UI = res
	else
		warn(TAG .. " Falha ao criar a interface: " .. tostring(res))
		local dummyBtn = { Activated = { Connect = function() return { Disconnect = function() end } end } }
		UI = setmetatable({ Toggle = dummyBtn, Outfit = dummyBtn, Punch = dummyBtn, Gesture = dummyBtn, Slide = dummyBtn, Parkour = dummyBtn, Break = dummyBtn, Dash = dummyBtn, Roll = dummyBtn, Drop = dummyBtn }, {
			__index = function() return function() end end,
		})
	end
end

----------------------------------------------------------------------
-- Estado global
----------------------------------------------------------------------
local State = {
	Enabled = CONFIG.StartEnabled,
	OutfitIndex = 1,
	EmoteIdx = 0,
	Token = 0,         -- invalida pipelines antigos (respawn / toggle)
	Character = nil, Humanoid = nil, Root = nil,
	Rig = nil,
	Hidden = nil,      -- [Instance] = valores originais
	ToolHeld = false,
	LastTrace = nil,
}
local GlobalBin = newBin()
State.RigBin = newBin()

----------------------------------------------------------------------
-- Poses / animação (dados)
----------------------------------------------------------------------
local JOINT_NAMES = { "Root", "Waist", "Neck", "ShL", "ShR", "WrL", "WrR", "HipL", "HipR" }

local function newPose()
	local p = {}
	for _, n in ipairs(JOINT_NAMES) do
		p[n] = { rx = 0, ry = 0, rz = 0, tx = 0, ty = 0, tz = 0 }
	end
	return p
end

local function resetPose(p)
	for i = 1, #JOINT_NAMES do
		local j = p[JOINT_NAMES[i]]
		j.rx, j.ry, j.rz, j.tx, j.ty, j.tz = 0, 0, 0, 0, 0, 0
	end
end

local function newAnimState()
	return {
		target = newPose(), cur = newPose(),
		t = 0, phase = 0, landT = 0, landS = 0, wasAir = false, lastVy = 0, airTime = 0,
		plantW = 1, yaw = nil, yawRate = 0, action = nil, punchSide = -1, lastPunch = -10,
		doubleUsed = false, prevJumpReq = -10, flipT = nil, flipAngle = 0,
		deadT = 0, deadHidden = false, nextBlink = os.clock() + 2, blinkUntil = 0,
		label = "", lastCalib = 0, expr = "",
		peak = 0, skidT = 0, fidget = nil, nextFidget = os.clock() + 6, combo = 0, parkour = nil,
	}
end

----------------------------------------------------------------------
-- Construção do rig
----------------------------------------------------------------------
local function faceCF(x, y, thick, roll)
	local R = 5.0 -- raio da cabeça (mm)
	local sz = -math.sqrt(math.max(R * R - x * x, 0.01))
	local n = Vector3.new(x, 0, sz).Unit
	local pos = Vector3.new(x, y, sz) + n * (thick * 0.5 - 0.08)
	return CFrame.lookAt(pos, pos + n) * CFrame.Angles(0, 0, roll or 0)
end

local EXPR = {
	Happy      = { browY = 2.4, browRoll = 0,         eye = 1.0,  curve = 1.0,  halfW = 2.1, open = 0,   round = false },
	Determined = { browY = 2.2, browRoll = rad(24),   eye = 0.8,  curve = 0.2,  halfW = 1.8, open = 0,   round = false },
	Surprised  = { browY = 3.1, browRoll = rad(-6),   eye = 1.15, curve = 0,    halfW = 1.2, open = 1.9, round = true },
	Shout      = { browY = 2.1, browRoll = rad(26),   eye = 0.75, curve = -0.2, halfW = 2.0, open = 2.6, round = false },
	Laugh      = { browY = 2.7, browRoll = rad(-8),   eye = 0.6,  curve = 1.1,  halfW = 2.3, open = 1.6, round = false },
	KO         = { browY = 2.2, browRoll = rad(-18),  eye = 0.12, curve = -0.9, halfW = 1.9, open = 0,   round = false },
}

local function buildRig(character, hum, hrp, outfit)
	local rig = { Parts = {}, Rest = {}, JointOf = {}, Joints = {}, BaseC0 = {}, DriveMode = "C0", Active = false }
	local model = Instance.new("Model")
	model.Name = "LegoMinifigRig"
	rig.Model = model

	local g = hum.HipHeight + hrp.Size.Y * 0.5 + CONFIG.FootOffsetTweak
	rig.BaseG = g
	rig.GroundOffset = g
	rig.AppliedG = g
	local origin = hrp.CFrame * CFrame.new(0, -g, 0)

	local rp = RaycastParams.new()
	rp.FilterType = Enum.RaycastFilterType.Exclude
	rp.FilterDescendantsInstances = { character }
	rig.RayParams = rp

	local function toRest(c)
		local p = c.Position
		return CFrame.new(p.X * K, p.Y * K, p.Z * K) * (c - p)
	end

	local function weldTo(parentPart, childPart)
		local w = Instance.new("Weld")
		w.Name = "MF_Weld"
		w.Part0 = parentPart
		w.Part1 = childPart
		w.C0 = rig.Rest[parentPart]:Inverse() * rig.Rest[childPart]
		w.C1 = CFrame.new()
		w.Parent = childPart
		rig.JointOf[childPart] = w
		return w
	end

	-- size/cfmm em "mm". cylAxis: nil (bloco), "Y" (cilindro vertical), "Z" (disco virado p/ frente), "Face" (sem rotação extra)
	local function mk(name, size, cfmm, color, parentPart, cylAxis)
		local p = Instance.new("Part")
		p.Name = "MF_" .. name
		p.Anchored = false
		p.CanCollide = false
		p.CanTouch = false
		p.CanQuery = false
		p.Massless = true
		p.Material = Enum.Material.SmoothPlastic
		p.Color = color
		p.TopSurface = Enum.SurfaceType.Smooth
		p.BottomSurface = Enum.SurfaceType.Smooth
		local cf = cfmm
		if cylAxis then
			p.Shape = Enum.PartType.Cylinder -- eixo do cilindro = eixo X local; size = (comprimento, diâmetro, diâmetro)
			if cylAxis == "Y" then
				cf = cfmm * CFrame.Angles(0, 0, rad(90))
			elseif cylAxis == "Z" then
				cf = cfmm * CFrame.Angles(0, rad(90), 0)
			end
		end
		p.Size = size * K
		local rest = toRest(cf)
		p.CFrame = origin * rest
		p.Parent = model
		table.insert(rig.Parts, p)
		rig.Rest[p] = rest
		if parentPart then weldTo(parentPart, p) end
		return p
	end

	local function motor(key, name, parentPart, childPart, jointMM)
		local J = CFrame.new(jointMM.X * K, jointMM.Y * K, jointMM.Z * K)
		local m = Instance.new("Motor6D")
		m.Name = "MF_" .. name
		m.Part0 = parentPart
		m.Part1 = childPart
		m.C0 = rig.Rest[parentPart]:Inverse() * J
		m.C1 = rig.Rest[childPart]:Inverse() * J
		m.Parent = childPart
		rig.Joints[key] = m
		rig.BaseC0[key] = m.C0
		rig.JointOf[childPart] = m
		return m
	end

	local skin = outfit.Skin or COL.Skin

	-- Corpo principal (mm; origem = centro do chão entre os pés, frente = -Z)
	local hips = mk("Hips", Vector3.new(16, 5, 8), CFrame.new(0, 12.5, 0), outfit.Legs)
	local torso = mk("Torso", Vector3.new(16, 13, 8), CFrame.new(0, 21.5, 0), outfit.Torso)
	local head = mk("Head", Vector3.new(9, 10, 10), CFrame.new(0, 32.5, 0), skin, nil, "Y")
	rig.Hips, rig.Head, rig.Torso = hips, head, torso

	-- Braços e mãos em gancho
	local arms, hands = {}, {}
	for _, s in ipairs({ -1, 1 }) do
		local n = s < 0 and "L" or "R"
		local x = 10.4 * s
		local arm = mk("Arm" .. n, Vector3.new(7.8, 4.6, 4.6), CFrame.new(x, 23.4, 0), outfit.Arms, nil, "Y")
		local hand = mk("Hand" .. n, Vector3.new(1.6, 4.4, 4.4), CFrame.new(x, 18.7, 0), skin, nil, "Y")
		mk("HandBack" .. n, Vector3.new(3.4, 4.2, 1.2), CFrame.new(x, 15.8, 1.5), skin, hand)
		mk("HandTop" .. n, Vector3.new(3.4, 1.2, 4.2), CFrame.new(x, 17.3, 0), skin, hand)
		mk("HandBot" .. n, Vector3.new(3.4, 1.2, 4.2), CFrame.new(x, 14.3, 0), skin, hand)
		local att = Instance.new("Attachment")
		att.Name = (s > 0 and "Right" or "Left") .. "GripAttachment" -- ponto para armas/objetos futuros
		att.CFrame = rig.Rest[hand]:Inverse() * toRest(CFrame.new(x, 15.8, 0))
		att.Parent = hand
		arms[s], hands[s] = arm, hand
	end

	-- Pernas com botas, solas e biqueiras
	local legs = {}
	for _, s in ipairs({ -1, 1 }) do
		local n = s < 0 and "L" or "R"
		local x = 4.0 * s
		local leg = mk("Leg" .. n, Vector3.new(7.8, 11, 7.8), CFrame.new(x, 5.5, 0), outfit.Legs)
		mk("Boot" .. n, Vector3.new(7.95, 3.4, 8.05), CFrame.new(x, 1.7, 0), outfit.Boots, leg)
		mk("Sole" .. n, Vector3.new(7.9, 0.8, 9.6), CFrame.new(x, 0.4, -0.7), outfit.Sole, leg)
		mk("Toe" .. n, Vector3.new(7.6, 2.4, 1.5), CFrame.new(x, 1.6, -4.4), outfit.Boots, leg)
		legs[s] = leg
	end

	-- Detalhes do tronco (coordenadas locais ao tronco; frente = z -4)
	local function TCF(x, y, z, rollDeg)
		local c = CFrame.new(x, 21.5 + y, z)
		if rollDeg then c = c * CFrame.Angles(0, 0, rad(rollDeg)) end
		return c
	end

	if outfit.Style == "Jacket" then
		mk("Shirt", Vector3.new(5.2, 9.6, 0.4), TCF(0, 1.4, -4.12), outfit.Shirt, torso)
		mk("CollarL", Vector3.new(3.2, 1.0, 0.9), TCF(-2.4, 5.6, -4.3, -40), outfit.Collar, torso)
		mk("CollarR", Vector3.new(3.2, 1.0, 0.9), TCF(2.4, 5.6, -4.3, 40), outfit.Collar, torso)
		mk("StitchL", Vector3.new(0.4, 9.2, 0.25), TCF(-2.75, 1.4, -4.3), outfit.Stitch, torso)
		mk("StitchR", Vector3.new(0.4, 9.2, 0.25), TCF(2.75, 1.4, -4.3), outfit.Stitch, torso)
		for i, y in ipairs({ 4.0, 1.9, -0.2 }) do
			mk("Button" .. i, Vector3.new(0.35, 1.0, 1.0), TCF(0, y, -4.445), outfit.Stitch, torso, "Z")
		end
		for _, s in ipairs({ -1, 1 }) do
			local n = s < 0 and "L" or "R"
			mk("Pocket" .. n, Vector3.new(3.6, 3.0, 0.5), TCF(4.8 * s, -1.6, -4.17), outfit.Pocket, torso)
			mk("Flap" .. n, Vector3.new(3.8, 0.8, 0.6), TCF(4.8 * s, -0.2, -4.22), outfit.Flap, torso)
		end
		if outfit.Strap then
			mk("Strap", Vector3.new(1.8, 15.5, 0.6), TCF(0, 1.0, -4.55, -45), outfit.StrapColor, torso)
			mk("StrapBuckle", Vector3.new(2.4, 1.3, 0.35), TCF(-1.77, -0.77, -4.9, -45), outfit.Buckle, torso)
		end
	else
		mk("EmblemOuter", Vector3.new(5.6, 5.6, 0.5), TCF(0, 2.0, -4.17, 45), outfit.Accent, torso)
		mk("EmblemInner", Vector3.new(3.4, 3.4, 0.3), TCF(0, 2.0, -4.42, 45), outfit.Accent2, torso)
		mk("TrimL", Vector3.new(0.9, 10.5, 0.4), TCF(-6.2, 0.2, -4.12), outfit.Accent, torso)
		mk("TrimR", Vector3.new(0.9, 10.5, 0.4), TCF(6.2, 0.2, -4.12), outfit.Accent, torso)
		mk("CollarRing", Vector3.new(7.0, 1.0, 8.2), TCF(0, 6.0, 0), outfit.Accent2, torso)
	end
	mk("Belt", Vector3.new(16.3, 2.0, 8.3), TCF(0, -5.5, 0), outfit.Belt, torso)
	mk("Buckle", Vector3.new(3.4, 1.6, 0.5), TCF(0, -5.5, -4.35), outfit.Buckle, torso)
	mk("BuckleHole", Vector3.new(1.8, 0.7, 0.25), TCF(0, -5.5, -4.62), outfit.Belt, torso)

	-- Cabeça: pino, cabelo, chapéu ou capacete
	local hg = outfit.Headgear
	if hg == "Hair" then
		mk("HairTop", Vector3.new(3.0, 10.6, 10.6), CFrame.new(0, 37.8, 0), outfit.HeadgearColor, head, "Y")
		mk("HairBack", Vector3.new(10.4, 6.8, 3.4), CFrame.new(0, 34.4, 3.6), outfit.HeadgearColor, head)
		mk("HairStud", Vector3.new(1.6, 5.4, 5.4), CFrame.new(0, 40.1, 0), outfit.HeadgearColor, head, "Y")
	elseif hg == "Hat" then
		mk("HatBrim", Vector3.new(0.9, 15.0, 15.0), CFrame.new(0, 37.4, 0), outfit.HeadgearColor, head, "Y")
		mk("HatCrown", Vector3.new(4.4, 9.6, 9.6), CFrame.new(0, 39.8, 0), outfit.HeadgearColor, head, "Y")
		mk("HatBand", Vector3.new(1.1, 9.9, 9.9), CFrame.new(0, 38.4, 0), outfit.HatBand or COL.Tan, head, "Y")
	elseif hg == "Helmet" then
		mk("HelmCap", Vector3.new(3.2, 10.8, 10.8), CFrame.new(0, 37.6, 0), outfit.HeadgearColor, head, "Y")
		mk("HelmTop", Vector3.new(2.2, 9.6, 9.6), CFrame.new(0, 40.2, 0), outfit.HeadgearColor, head, "Y")
		mk("HelmFin", Vector3.new(1.0, 3.0, 6.4), CFrame.new(0, 42.2, 0), outfit.HeadgearColor, head)
		mk("HelmSideL", Vector3.new(1.2, 6.0, 4.4), CFrame.new(-5.6, 34.2, 0.8), outfit.HeadgearColor, head)
		mk("HelmSideR", Vector3.new(1.2, 6.0, 4.4), CFrame.new(5.6, 34.2, 0.8), outfit.HeadgearColor, head)
	else
		mk("Stud", Vector3.new(1.8, 6.0, 6.0), CFrame.new(0, 37.9, 0), skin, head, "Y")
	end

	-- Rosto (peças reaproveitadas para todas as expressões; sem decalques/IDs externos)
	local F = { Mouth = {} }
	local faceState = { expr = "Happy", blink = false }
	local function faceMk(name, size, color, disc)
		local p = mk(name, size, HEAD_CENTER, color, head, disc and "Face" or nil)
		return { Part = p, Weld = rig.JointOf[p] }
	end
	F.EyeL = faceMk("EyeL", Vector3.new(0.4, 1.6, 1.6), COL.Black, true)
	F.EyeR = faceMk("EyeR", Vector3.new(0.4, 1.6, 1.6), COL.Black, true)
	F.GlintL = faceMk("GlintL", Vector3.new(0.7, 0.55, 0.55), COL.White, true)
	F.GlintR = faceMk("GlintR", Vector3.new(0.7, 0.55, 0.55), COL.White, true)
	F.BrowL = faceMk("BrowL", Vector3.new(2.4, 0.55, 0.4), COL.Black, false)
	F.BrowR = faceMk("BrowR", Vector3.new(2.4, 0.55, 0.4), COL.Black, false)
	F.Nose = faceMk("Nose", Vector3.new(0.9, 1.2, 0.8), outfit.SkinDark or COL.SkinDark, false)
	for i = 1, 7 do
		F.Mouth[i] = faceMk("Mouth" .. i, Vector3.new(1, 0.5, 0.35), COL.Black, false)
	end
	F.MouthFill = faceMk("MouthFill", Vector3.new(0.35, 1.6, 1.6), COL.Black, true)

	local function placeFace(f, x, y, thick, roll, size, disc)
		local fcf = faceCF(x, y, thick, roll)
		if disc then fcf = fcf * CFrame.Angles(0, rad(90), 0) end
		f.Weld.C0 = rig.Rest[head]:Inverse() * toRest(HEAD_CENTER * fcf)
		f.Part.Size = size * K
	end

	local function applyFace()
		local e = EXPR[faceState.expr] or EXPR.Happy
		local eyeS = faceState.blink and 0.12 or e.eye
		local eyeSize = Vector3.new(0.4, 1.6 * eyeS, 1.6)
		placeFace(F.EyeL, -1.9, 0.7, 0.4, 0, eyeSize, true)
		placeFace(F.EyeR, 1.9, 0.7, 0.4, 0, eyeSize, true)
		placeFace(F.GlintL, -1.55, 0.7 + 0.4 * eyeS, 0.7, 0, Vector3.new(0.7, 0.55, 0.55), true)
		placeFace(F.GlintR, 2.25, 0.7 + 0.4 * eyeS, 0.7, 0, Vector3.new(0.7, 0.55, 0.55), true)
		local gt = eyeS > 0.3 and 0 or 1
		F.GlintL.Part.Transparency = gt
		F.GlintR.Part.Transparency = gt
		placeFace(F.BrowL, -2.1, e.browY, 0.4, -e.browRoll, Vector3.new(2.4, 0.55, 0.4), false)
		placeFace(F.BrowR, 2.1, e.browY, 0.4, e.browRoll, Vector3.new(2.4, 0.55, 0.4), false)
		placeFace(F.Nose, 0, -0.7, 0.8, 0, Vector3.new(0.9, 1.2, 0.8), false)
		local mouthY = -2.1
		local n = #F.Mouth
		local spacing = (2 * e.halfW) / (n - 1)
		for i = 1, n do
			local u = (i - 1) / (n - 1) * 2 - 1
			local x = u * e.halfW
			local y = mouthY + e.curve * u * u
			local slope = 2 * e.curve * u / e.halfW
			placeFace(F.Mouth[i], x, y, 0.35, math.atan(slope), Vector3.new(spacing * 1.35, 0.5, 0.35), false)
			F.Mouth[i].Part.Transparency = (e.open > 0 and e.round) and 1 or 0
		end
		if e.open > 0 then
			local w = e.round and e.open or e.open * 1.5
			placeFace(F.MouthFill, 0, mouthY - e.open * 0.15, 0.35, 0, Vector3.new(0.35, e.open, w), true)
			F.MouthFill.Part.Transparency = 0
		else
			F.MouthFill.Part.Transparency = 1
		end
	end

	function rig.SetExpression(name)
		if faceState.expr == name then return end
		faceState.expr = name
		applyFace()
	end
	function rig.SetBlink(b)
		if faceState.blink == b then return end
		faceState.blink = b
		applyFace()
	end
	rig.RefreshFace = applyFace
	applyFace()

	-- Articulações Motor6D (pivôs em mm no referencial do rig)
	motor("Waist", "Waist", hips, torso, Vector3.new(0, 15, 0))
	motor("Neck", "Neck", torso, head, Vector3.new(0, 28.2, 0))
	motor("ShL", "ShoulderL", torso, arms[-1], Vector3.new(-10.4, 25.8, 0))
	motor("ShR", "ShoulderR", torso, arms[1], Vector3.new(10.4, 25.8, 0))
	motor("WrL", "WristL", arms[-1], hands[-1], Vector3.new(-10.4, 19.5, 0))
	motor("WrR", "WristR", arms[1], hands[1], Vector3.new(10.4, 19.5, 0))
	motor("HipL", "HipL", hips, legs[-1], Vector3.new(-4.0, 10.5, 0))
	motor("HipR", "HipR", hips, legs[1], Vector3.new(4.0, 10.5, 0))

	-- Só agora o rig entra no personagem; o Motor6D raiz é criado no mesmo quadro (sem yield)
	model.Parent = character
	local rm = Instance.new("Motor6D")
	rm.Name = "MF_RootMotor"
	rm.Part0 = hrp
	rm.Part1 = hips
	rm.C0 = CFrame.new(0, -g, 0)
	rm.C1 = rig.Rest[hips]:Inverse()
	rm.Parent = hips
	rig.Joints.Root = rm
	rig.BaseC0.Root = rm.C0
	rig.JointOf[hips] = rm

	-- Aplica a pose de uma junta. Modo "C0" (padrão): o Animator não mexe em C0, então não sobrescreve.
	function rig.Drive(key, delta)
		local m = rig.Joints[key]
		if rig.DriveMode == "C0" then
			m.C0 = rig.BaseC0[key] * delta
		else
			m.Transform = delta
		end
	end

	rig.Anim = newAnimState()

	function rig.Destroy()
		rig.Active = false
		if model and model.Parent then model:Destroy() elseif model then model:Destroy() end
	end
	return rig
end

----------------------------------------------------------------------
-- Validação
----------------------------------------------------------------------
local function isFiniteCF(c)
	local x, y, z = c.X, c.Y, c.Z
	if x ~= x or y ~= y or z ~= z then return false end
	if abs(x) > 1e5 or abs(y) > 1e5 or abs(z) > 1e5 then return false end
	local l = c.LookVector
	return l.X == l.X and l.Y == l.Y and l.Z == l.Z
end

local function validateRigStatic(rig, character, hrp)
	if not rig.Model or rig.Model.Parent ~= character then
		error("Modelo da minifigura não está dentro do personagem", 0)
	end
	if #rig.Parts > CONFIG.MaxRigParts then
		error("Peças demais no rig: " .. #rig.Parts .. " (máx " .. CONFIG.MaxRigParts .. ")", 0)
	end
	for _, n in ipairs(JOINT_NAMES) do
		local m = rig.Joints[n]
		if not m or not m.Parent then error("Junta ausente: " .. n, 0) end
		if not m.Part0 or not m.Part1 then error("Junta sem Part0/Part1: " .. n, 0) end
		if not isFiniteCF(m.C0) or not isFiniteCF(m.C1) then error("C0/C1 inválido na junta " .. n, 0) end
		if not m.Part1:IsDescendantOf(rig.Model) then error("Part1 fora do rig na junta " .. n, 0) end
		if m.Part0 ~= hrp and not m.Part0:IsDescendantOf(rig.Model) then
			error("Part0 inválida na junta " .. n, 0)
		end
	end
	for _, p in ipairs(rig.Parts) do
		local j = rig.JointOf[p]
		if not j or not j.Parent or j.Part1 ~= p or not j.Part0 then error("Peça sem junta válida: " .. p.Name, 0) end
		if p.Anchored then error("Peça ancorada: " .. p.Name, 0) end
		local s = p.Size
		if not (s.X > 0 and s.Y > 0 and s.Z > 0) then error("Tamanho inválido: " .. p.Name, 0) end
		local cur, guard = p, 0
		while cur ~= hrp do
			local jj = rig.JointOf[cur]
			if not jj or not jj.Part0 then error("Cadeia de juntas quebrada em " .. cur.Name, 0) end
			cur = jj.Part0
			guard += 1
			if guard > 20 then error("Ciclo de juntas em " .. p.Name, 0) end
		end
	end
end

local function verifyRigPhysics(rig, hrp, hum)
	local rm = rig.Joints.Root
	local tol = 0.75 + hrp.AssemblyLinearVelocity.Magnitude * 0.05
	for _, part in ipairs({ rig.Hips, rig.Torso, rig.Head }) do
		local expected = hrp.CFrame * rm.C0 * rm.Transform * rig.Rest[rig.Hips]
		if part ~= rig.Hips then
			expected = hrp.CFrame * rm.C0 * rig.Rest[part]
		end
		local d = (part.Position - expected.Position).Magnitude
		if d > tol then
			error(("Peça %s fora do lugar (%.2f studs)"):format(part.Name, d), 0)
		end
	end
	if rig.Hips.AssemblyRootPart ~= hrp.AssemblyRootPart then
		error("Rig não pertence ao mesmo assembly do HumanoidRootPart", 0)
	end
	if hum.Health <= 0 or hum:GetState() == ST.Dead then
		error("Humanoid sem vida durante a validação", 0)
	end
end

----------------------------------------------------------------------
-- Ocultar / restaurar o avatar original (somente local)
----------------------------------------------------------------------
local function isRigInstance(inst)
	return inst.Name == "LegoMinifigRig" or inst:FindFirstAncestor("LegoMinifigRig") ~= nil
end

local function shouldHide(inst)
	if isRigInstance(inst) then return false end
	if inst:FindFirstAncestorWhichIsA("Tool") then return false end
	return inst:IsA("BasePart") or inst:IsA("Decal")
end

local function hideOne(inst)
	local H = State.Hidden
	if not H or H[inst] then return end
	if inst:IsA("BasePart") then
		H[inst] = { Transparency = inst.Transparency, CastShadow = inst.CastShadow }
		inst.Transparency = 1
		inst.CastShadow = false
	elseif inst:IsA("Decal") then
		H[inst] = { Transparency = inst.Transparency }
		inst.Transparency = 1
	end
end

local function hideOriginal(character)
	State.Hidden = {}
	for _, inst in ipairs(character:GetDescendants()) do
		if shouldHide(inst) then hideOne(inst) end
	end
	State.RigBin:Add(character.DescendantAdded:Connect(function(inst)
		if State.Rig and shouldHide(inst) then hideOne(inst) end
	end))
end

local function restoreOriginal()
	local H = State.Hidden
	State.Hidden = nil
	if not H then return end
	for inst, rec in pairs(H) do
		if inst.Parent then
			local ok, e = pcall(function()
				inst.Transparency = rec.Transparency
				if rec.CastShadow ~= nil then inst.CastShadow = rec.CastShadow end
			end)
			if not ok then warn(TAG .. " falha ao restaurar " .. inst.Name .. ": " .. tostring(e)) end
		end
	end
end

----------------------------------------------------------------------
-- Poses
----------------------------------------------------------------------
local function ov(j, f, v, w) j[f] = j[f] + (v - j[f]) * w end

local function legDrop(rx, rz)
	return 10.5 * cos(rx) * cos(rz) + (rx > 0 and 5.4 or 4.1) * abs(sin(rx)) + 3.9 * abs(sin(rz))
end

local function poseGround(A, T, speed, yawRate, dt)
	local t = A.t
	local runB = smooth(clamp((speed - 7) / 6, 0, 1))
	local moveW = clamp(speed / 3, 0, 1)
	local idle = 1 - moveW
	if speed > 0.6 then
		A.phase += (speed / lerp(4.0, 5.4, runB)) * 2 * math.pi * dt
	end
	local ph = A.phase
	local s = sin(ph)
	local legAmp = rad(lerp(34, 58, runB)) * moveW
	T.HipL.rx = s * legAmp
	T.HipR.rx = -s * legAmp
	T.HipL.rz = -rad(2)
	T.HipR.rz = rad(2)
	local armAmp = rad(lerp(30, 64, runB)) * moveW
	T.ShL.rx = -s * armAmp
	T.ShR.rx = s * armAmp
	local turnK = clamp(speed / 4, 0, 1)
	local abd = rad(4) + rad(10) * runB * moveW + clamp(abs(yawRate) * 0.02, 0, rad(16)) * turnK
		+ rad(1.5) * sin(t * 1.3) * idle
	T.ShL.rz = -abd
	T.ShR.rz = abd
	T.WrL.rx = -T.ShL.rx * 0.3
	T.WrR.rx = -T.ShR.rx * 0.3
	local lean = -(rad(3) + rad(12) * runB) * moveW
	local turnLean = clamp(yawRate * 0.025, -rad(12), rad(12)) * turnK
	T.Waist.rx = lean
	T.Waist.ry = s * rad(5 + 7 * runB) * moveW - clamp(yawRate * 0.02, -rad(10), rad(10)) * turnK
		+ sin(t * 0.8) * rad(1.6) * idle
	T.Waist.rz = turnLean
	T.Neck.rx = -lean * 0.7 + sin(ph * 2) * rad(2.5) * moveW + sin(t * 0.9) * rad(2) * idle
	T.Neck.ry = clamp(yawRate * 0.03, -rad(20), rad(20)) * turnK
		+ (sin(t * 0.55) * rad(14) + sin(t * 1.3) * rad(4)) * idle
	T.Neck.rz = -turnLean * 0.6
	T.Root.ty = sin(t * 2.4) * 0.015 * idle
	T.Root.rz = -s * rad(3 + 2 * runB) * moveW -- balanço lateral característico
	T.Waist.rz += s * rad(3.5) * moveW
	if State.ToolHeld then -- pose de "segurar objeto" (visual)
		T.ShR.rx = rad(78)
		T.ShR.rz = rad(4)
		T.WrR.rx = -rad(8)
	end
end

local function poseAir(A, T, vy)
	local t = A.t
	local jw = clamp(vy / 12, 0, 1)
	local fw = 1 - jw
	local flail = sin(t * 13)
	local armRx = lerp(rad(118), rad(150), jw)
	T.ShL.rx = armRx + flail * rad(16) * fw
	T.ShR.rx = armRx - flail * rad(16) * fw
	T.ShL.rz = -lerp(rad(32), rad(14), jw)
	T.ShR.rz = lerp(rad(32), rad(14), jw)
	T.WrL.rx = -rad(10)
	T.WrR.rx = -rad(10)
	local kick = sin(t * 9) * rad(24) * fw
	T.HipL.rx = kick + rad(28) * jw
	T.HipR.rx = -kick - rad(14) * jw
	T.HipL.rz = -rad(8)
	T.HipR.rz = rad(8)
	T.Waist.rx = rad(6) * jw - rad(3) * fw
	T.Neck.rx = rad(10) * jw
	T.Neck.ry = sin(t * 6) * rad(6) * fw
end

local function poseSeated(T)
	T.HipL.rx = rad(88)
	T.HipR.rx = rad(88)
	T.HipL.rz = -rad(3)
	T.HipR.rz = rad(3)
	T.ShL.rx = rad(35)
	T.ShR.rx = rad(35)
	T.ShL.rz = -rad(6)
	T.ShR.rz = rad(6)
end

local function poseClimb(A, T, vy, dt)
	A.phase += clamp(abs(vy) / 3, 0, 3) * math.pi * dt
	local s = sin(A.phase)
	T.ShL.rx = rad(130) + s * rad(25)
	T.ShR.rx = rad(130) - s * rad(25)
	T.HipL.rx = rad(15) - s * rad(30)
	T.HipR.rx = rad(15) + s * rad(30)
	T.Waist.rx = -rad(5)
end

local function poseDead(A, T)
	local u = clamp(A.deadT / 1.1, 0, 1)
	local e = easeOut(u)
	local arc = sin(u * math.pi)
	T.Neck.ty = (arc * 18 + e * 4) * K
	T.Neck.ry = u * rad(540)
	T.Neck.rx = u * rad(120)
	T.Waist.ty = arc * 5 * K
	T.Waist.rx = u * rad(40)
	for _, s in ipairs({ -1, 1 }) do
		local sh = s < 0 and T.ShL or T.ShR
		local wr = s < 0 and T.WrL or T.WrR
		local hp = s < 0 and T.HipL or T.HipR
		sh.tx = s * e * 16 * K
		sh.ty = (arc * 10 - u * 6) * K
		sh.rx = u * rad(500)
		sh.rz = s * rad(60) * u
		wr.tx = s * e * 8 * K
		wr.ty = -u * 8 * K
		wr.rx = u * rad(720)
		hp.tx = s * e * 8 * K
		hp.ty = (arc * 6 - u * 8) * K
		hp.rz = s * rad(40) * u
		hp.rx = u * rad(200)
	end
end

local function overlayLanding(T, A)
	local w = smooth(A.landT) * A.landS
	T.Root.ty += -1.6 * K * w
	T.Root.rx += -rad(6) * w
	T.HipL.rz += -rad(18) * w
	T.HipR.rz += rad(18) * w
	T.Waist.rx += -rad(16) * w
	T.Neck.rx += rad(10) * w
	T.ShL.rz += -rad(20) * w
	T.ShR.rz += rad(20) * w
	T.ShL.rx += rad(15) * w
	T.ShR.rx += rad(15) * w
end

local function overlayPunch(T, act)
	local t, side = act.t, act.side
	local armRx, yaw, lean, pw
	if t < 0.12 then -- antecipação
		local u = easeOut(t / 0.12)
		armRx = lerp(0, -rad(60), u); yaw = lerp(0, -rad(24), u); lean = lerp(0, rad(4), u); pw = t / 0.22
	elseif t < 0.22 then -- golpe
		local u = easeIn((t - 0.12) / 0.10)
		armRx = lerp(-rad(60), rad(95), u); yaw = lerp(-rad(24), rad(32), u); lean = lerp(rad(4), -rad(10), u); pw = t / 0.22
	elseif t < 0.30 then -- impacto
		armRx = rad(95); yaw = rad(32); lean = -rad(10); pw = 1
	else -- retorno
		local u = smooth(clamp((t - 0.30) / 0.22, 0, 1))
		armRx = lerp(rad(95), 0, u); yaw = lerp(rad(32), 0, u); lean = lerp(-rad(10), 0, u); pw = 1 - u
	end
	local arm = side == 1 and T.ShR or T.ShL
	local other = side == 1 and T.ShL or T.ShR
	local wr = side == 1 and T.WrR or T.WrL
	arm.rx = armRx
	arm.rz = side * rad(5)
	wr.rx = -rad(8)
	other.rx = rad(38) * pw
	other.rz = -side * rad(14)
	T.Waist.ry = yaw * side
	T.Waist.rx = lean
	T.Waist.rz = 0
	T.Neck.ry = -yaw * side * 0.6
	T.Neck.rx = -lean * 0.5
	local front = side == 1 and T.HipL or T.HipR
	local back = side == 1 and T.HipR or T.HipL
	front.rx = rad(18) * pw
	back.rx = -rad(12) * pw
	T.Root.rx = -rad(5) * pw
end

local function overlayEmote(T, act)
	local t = act.t
	local w = clamp(t / 0.12, 0, 1) * clamp((act.dur - t) / 0.2, 0, 1)
	local hop = abs(sin(t * math.pi * 3.2))
	local flap = sin(t * 18)
	ov(T.ShL, "rx", rad(158) + flap * rad(14), w)
	ov(T.ShR, "rx", rad(158) - flap * rad(14), w)
	ov(T.ShL, "rz", -rad(24), w)
	ov(T.ShR, "rz", rad(24), w)
	ov(T.WrL, "rx", 0, w)
	ov(T.WrR, "rx", 0, w)
	ov(T.HipL, "rx", 0, w)
	ov(T.HipR, "rx", 0, w)
	ov(T.HipL, "rz", -rad(14) * hop, w)
	ov(T.HipR, "rz", rad(14) * hop, w)
	ov(T.Root, "ty", hop * 7 * K, w)
	ov(T.Waist, "rx", 0, w)
	ov(T.Waist, "rz", sin(t * 9) * rad(7), w)
	ov(T.Neck, "ry", sin(t * 11) * rad(16), w)
	ov(T.Neck, "rx", rad(8), w)
end

local function overlayFlip(T, u)
	local w = sin(clamp(u, 0, 1) * math.pi)
	ov(T.HipL, "rx", rad(80), w)
	ov(T.HipR, "rx", rad(80), w)
	ov(T.ShL, "rx", rad(70), w)
	ov(T.ShR, "rx", rad(70), w)
	ov(T.Waist, "rx", -rad(15), w)
end

----------------------------------------------------------------------
-- Animações extras (estilo jogos LEGO)
----------------------------------------------------------------------
local EMOTES = { "Comemorar", "Acenar", "Dançar" }
local FIDGETS = { "look", "shrug", "stretch", "tap" }

local function overlayKick(T, act)
	local t = act.t
	local k, lean
	if t < 0.15 then
		local u = easeOut(t / 0.15)
		k = lerp(0, -rad(30), u); lean = lerp(0, rad(6), u)
	elseif t < 0.28 then
		local u = easeIn((t - 0.15) / 0.13)
		k = lerp(-rad(30), rad(88), u); lean = lerp(rad(6), rad(14), u)
	elseif t < 0.38 then
		k = rad(88); lean = rad(14)
	else
		local u = smooth(clamp((t - 0.38) / 0.24, 0, 1))
		k = lerp(rad(88), 0, u); lean = lerp(rad(14), 0, u)
	end
	T.HipR.rx = k
	T.HipL.rx = -rad(6)
	T.HipR.rz = rad(4)
	T.Waist.rx = lean
	T.Neck.rx = -lean * 0.6
	T.ShL.rz = -rad(38)
	T.ShR.rz = rad(38)
	T.ShL.rx = rad(25)
	T.ShR.rx = -rad(20)
end

local function overlayWave(T, act)
	local t = act.t
	local w = clamp(t / 0.15, 0, 1) * clamp((act.dur - t) / 0.2, 0, 1)
	ov(T.ShR, "rx", rad(20), w)
	ov(T.ShR, "rz", rad(115) + sin(t * 12) * rad(20), w)
	ov(T.WrR, "rx", sin(t * 12) * rad(10), w)
	ov(T.Neck, "rz", -rad(8), w)
	ov(T.Waist, "rz", rad(3), w)
	ov(T.Root, "ty", abs(sin(t * 6)) * 0.5 * K, w)
end

local function overlayDance(T, act)
	local t = act.t
	local w = clamp(t / 0.15, 0, 1) * clamp((act.dur - t) / 0.2, 0, 1)
	local s = sin(t * 8)
	ov(T.ShL, "rx", rad(90) + s * rad(70), w)
	ov(T.ShR, "rx", rad(90) - s * rad(70), w)
	ov(T.ShL, "rz", -rad(20), w)
	ov(T.ShR, "rz", rad(20), w)
	ov(T.HipL, "rx", s * rad(28), w)
	ov(T.HipR, "rx", -s * rad(28), w)
	ov(T.Waist, "ry", s * rad(28), w)
	ov(T.Waist, "rz", cos(t * 8) * rad(8), w)
	ov(T.Neck, "ry", -s * rad(20), w)
	ov(T.Root, "ty", abs(sin(t * 8)) * 3 * K, w)
	ov(T.Root, "rz", s * rad(5), w)
end

local function overlayEmoteKind(T, act)
	if act.name == "Acenar" then
		overlayWave(T, act)
	elseif act.name == "Dançar" then
		overlayDance(T, act)
	else
		overlayEmote(T, act)
	end
end

local function overlaySkid(T, A)
	local w = smooth(A.skidT)
	ov(T.Waist, "rx", rad(16), w)
	ov(T.Neck, "rx", -rad(8), w)
	ov(T.ShL, "rx", rad(55), w)
	ov(T.ShR, "rx", rad(55), w)
	ov(T.ShL, "rz", -rad(25), w)
	ov(T.ShR, "rz", rad(25), w)
	ov(T.HipL, "rx", rad(34), w)
	ov(T.HipR, "rx", -rad(14), w)
	ov(T.Root, "rx", rad(6), w)
end

local function overlayFidget(T, f)
	local u = clamp(f.t / f.dur, 0, 1)
	local w = clamp(sin(u * math.pi) * 2.2, 0, 1)
	if f.kind == "look" then
		ov(T.Neck, "ry", sin(u * math.pi * 2) * rad(50), w)
		ov(T.Waist, "ry", sin(u * math.pi * 2) * rad(14), w)
	elseif f.kind == "shrug" then
		ov(T.ShL, "rz", -rad(55), w)
		ov(T.ShR, "rz", rad(55), w)
		ov(T.ShL, "rx", rad(15), w)
		ov(T.ShR, "rx", rad(15), w)
		ov(T.Neck, "rz", sin(u * math.pi * 4) * rad(8), w)
	elseif f.kind == "stretch" then
		ov(T.ShL, "rx", rad(172), w)
		ov(T.ShR, "rx", rad(172), w)
		ov(T.ShL, "rz", -rad(14), w)
		ov(T.ShR, "rz", rad(14), w)
		ov(T.Waist, "rx", rad(10), w)
		ov(T.Neck, "rx", rad(12), w)
	else -- tap: bate o pé
		local tap = abs(sin(f.t * 11))
		ov(T.HipR, "rx", rad(18) * tap, w)
		ov(T.Neck, "rx", rad(4) * sin(f.t * 11), w)
		ov(T.ShL, "rx", -rad(10), w)
		ov(T.ShR, "rx", -rad(10), w)
	end
end

----------------------------------------------------------------------
-- PARKOUR + DESTRUIÇÃO VISUAL LOCAL (V4)
----------------------------------------------------------------------
local Parkour = {
	mode = "None", -- None, Slide, Hang, Mantle, WallRun
	endsAt = 0, cooldown = 0, direction = nil, hangPosition = nil,
	wallNormal = nil, lastShimmy = 0,
	controlHumanoid = nil, savedWalkSpeed = nil, savedAutoRotate = nil,
	stamina = CONFIG.StaminaMax, lastStaminaUse = 0, lastStaminaUpdate = os.clock(),
	dashCooldown = 0, lastImpactScan = 0, lastImpactAt = 0, lastImpactPart = nil,
	movementStartedAt = 0, actionSerial = 0, lastKnownGround = nil,
}
local Destruction = { Entries = {}, FX = nil, scanAt = 0, running = true }
local isHardProtectedPart
local isLocalEffectPart
local flatDirection
local PROTECTED_WORDS = {
	"floor", "ground", "baseplate", "spawn", "checkpoint", "foundation", "pillar", "column",
	"support", "lobby", "terrain", "boundary", "hitbox", "walkway", "platform", "road",
	"bridge", "staircase", "stair", "building", "structure", "tower", "invisible wall",
}
local LIKELY_WORDS = {
	"breakable", "crate", "barrel", "box", "wood", "glass", "cabinet", "chair", "table",
	"lamp", "fence", "door", "window", "prop", "debris", "rock", "sign", "shelf", "panel",
}

local function destroyIfAlive(inst)
	if inst and inst.Parent then pcall(function() inst:Destroy() end) end
end

local function ensureFXFolder()
	if Destruction.FX and Destruction.FX.Parent then return Destruction.FX end
	local folder = Instance.new("Folder")
	folder.Name = "LegoLocalFX_V6"
	folder.Parent = Workspace
	Destruction.FX = folder
	return folder
end

local function clearDestruction()
	for part, entry in pairs(Destruction.Entries) do
		if part and part.Parent and entry.originalLTM ~= nil then
			pcall(function() part.LocalTransparencyModifier = entry.originalLTM end)
		end
		destroyIfAlive(entry.highlight)
		for _, crack in ipairs(entry.cracks or {}) do destroyIfAlive(crack) end
	end
	Destruction.Entries = {}
	destroyIfAlive(Destruction.FX)
	Destruction.FX = nil
	Destruction.scanAt = 0
end

local function saveMovementControl(hum)
	if Parkour.controlHumanoid == hum then return end
	Parkour.controlHumanoid = hum
	Parkour.savedWalkSpeed = hum.WalkSpeed
	Parkour.savedAutoRotate = hum.AutoRotate
end

local function restoreMovementControl()
	local hum = Parkour.controlHumanoid
	if hum and hum.Parent then
		if Parkour.savedWalkSpeed ~= nil then hum.WalkSpeed = Parkour.savedWalkSpeed end
		if Parkour.savedAutoRotate ~= nil then hum.AutoRotate = Parkour.savedAutoRotate end
	end
	Parkour.controlHumanoid = nil
	Parkour.savedWalkSpeed = nil
	Parkour.savedAutoRotate = nil
end

local function resetParkour(restore)
	if restore then restoreMovementControl() end
	Parkour.mode = "None"
	Parkour.endsAt = 0
	Parkour.direction = nil
	Parkour.hangPosition = nil
	Parkour.wallNormal = nil
	UI.setParkourText("PARKOUR")
	if State.Rig and State.Rig.Anim then State.Rig.Anim.parkour = nil end
end

local function getRayParams()
	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	local exclude = {}
	if State.Character then table.insert(exclude, State.Character) end
	if Destruction.FX then table.insert(exclude, Destruction.FX) end
	params.FilterDescendantsInstances = exclude
	params.IgnoreWater = true
	return params
end

--------------------------------------------------------------------
-- V6: CONTEXT SENSOR + STAMINA CORE
-- Estes sensores são amostrados em baixa frequência; eles não criam peças
-- no cenário e não percorrem o mapa inteiro.
--------------------------------------------------------------------
local SurfaceSense = {
	updatedAt = 0,
	frontLow = nil,
	frontMid = nil,
	frontHigh = nil,
	leftWall = nil,
	rightWall = nil,
	groundAhead = nil,
	groundBelow = nil,
	headClear = true,
	landingClear = false,
	obstacleHeight = math.huge,
	wallSide = "None",
	state = "Unknown",
	sequence = 0,
}

local function rayAt(origin, direction, params)
	if direction.Magnitude < 0.001 then return nil end
	local ok, result = pcall(function()
		return Workspace:Raycast(origin, direction, params or getRayParams())
	end)
	if ok then return result end
	return nil
end

local function isCharacterPart(part)
	return part ~= nil and State.Character ~= nil and part:IsDescendantOf(State.Character)
end

local function isSolidCandidate(part)
	if not part or not part:IsA("BasePart") or not part.Parent then return false end
	if isCharacterPart(part) then return false end
	if part.CanQuery == false or part.CanCollide == false then return false end
	if part.Transparency >= 0.98 then return false end
	return true
end

local function obstacleHeightFromHit(hit, root, hum)
	if not hit or not root or not hum then return math.huge end
	local footY = root.Position.Y - hum.HipHeight - root.Size.Y * 0.5
	return hit.Position.Y - footY
end

local function hasBoxClearance(center, size, extraIgnore)
	local params = OverlapParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	local ignore = {}
	if State.Character then table.insert(ignore, State.Character) end
	if Destruction and Destruction.FX then table.insert(ignore, Destruction.FX) end
	if extraIgnore then
		for _, inst in ipairs(extraIgnore) do table.insert(ignore, inst) end
	end
	params.FilterDescendantsInstances = ignore
	params.MaxParts = 48
	local ok, parts = pcall(function()
		return Workspace:GetPartBoundsInBox(center, size, params)
	end)
	if not ok or type(parts) ~= "table" then return false end
	for _, part in ipairs(parts) do
		if isSolidCandidate(part) then return false end
	end
	return true
end

local function canOccupyStandingPosition(position, hrp, hum)
	if not (position and hrp and hum) then return false end
	local center = CFrame.new(position + Vector3.new(0, hum.HipHeight + hrp.Size.Y * 0.5, 0))
	local size = Vector3.new(math.max(1.6, hrp.Size.X + 0.7), math.max(3.6, hum.HipHeight * 2 + hrp.Size.Y), math.max(1.6, hrp.Size.Z + 0.7))
	return hasBoxClearance(center, size)
end

local function findGroundAt(position, distance, params)
	return rayAt(position + Vector3.new(0, 1.5, 0), Vector3.new(0, -(distance + 1.5), 0), params)
end

local function refreshSurfaceSense(force)
	local now = os.clock()
	if not force and now - SurfaceSense.updatedAt < CONFIG.SenseInterval then return SurfaceSense end
	SurfaceSense.updatedAt = now
	SurfaceSense.sequence += 1
	local root, hum = State.Root, State.Humanoid
	if not (root and root.Parent and hum and hum.Parent) then
		SurfaceSense.state = "NoCharacter"
		return SurfaceSense
	end

	local params = getRayParams()
	local facing = flatDirection()
	local right = Vector3.new(-facing.Z, 0, facing.X)
	local origin = root.Position
	SurfaceSense.frontLow = rayAt(origin + Vector3.new(0, 0.05, 0), facing * 3.4, params)
	SurfaceSense.frontMid = rayAt(origin + Vector3.new(0, 1.15, 0), facing * 2.8, params)
	SurfaceSense.frontHigh = rayAt(origin + Vector3.new(0, 2.3, 0), facing * 2.4, params)
	SurfaceSense.leftWall = rayAt(origin + Vector3.new(0, 0.8, 0), -right * 1.7, params)
	SurfaceSense.rightWall = rayAt(origin + Vector3.new(0, 0.8, 0), right * 1.7, params)
	SurfaceSense.groundBelow = rayAt(origin, Vector3.new(0, -math.max(5.5, hum.HipHeight + 4), 0), params)
	SurfaceSense.groundAhead = findGroundAt(origin + facing * 2.2, math.max(5, hum.HipHeight + 2), params)
	SurfaceSense.obstacleHeight = obstacleHeightFromHit(SurfaceSense.frontLow, root, hum)
	SurfaceSense.headClear = SurfaceSense.frontHigh == nil
	SurfaceSense.landingClear = hasBoxClearance(CFrame.new(origin + facing * 2.4 + Vector3.new(0, 1.6, 0)), Vector3.new(1.6, 3.2, 1.6))

	local left = SurfaceSense.leftWall
	local rightHit = SurfaceSense.rightWall
	if left and left.Normal.Y < 0.28 then SurfaceSense.wallSide = "Left"
	elseif rightHit and rightHit.Normal.Y < 0.28 then SurfaceSense.wallSide = "Right"
	else SurfaceSense.wallSide = "None" end

	local state = hum:GetState()
	if state == ST.Freefall or state == ST.Jumping then
		SurfaceSense.state = "Air"
	elseif SurfaceSense.frontLow and SurfaceSense.frontHigh then
		SurfaceSense.state = "Wall"
	elseif SurfaceSense.frontLow then
		SurfaceSense.state = "LowObstacle"
	elseif SurfaceSense.groundAhead == nil then
		SurfaceSense.state = "DropAhead"
	else
		SurfaceSense.state = "Open"
	end
	return SurfaceSense
end

local function spendStamina(amount, reason)
	amount = math.max(0, tonumber(amount) or 0)
	if Parkour.stamina + 0.001 < amount then
		UI.setAnim("Fôlego insuficiente para " .. tostring(reason or "parkour"), C_WARN)
		return false
	end
	Parkour.stamina = math.max(0, Parkour.stamina - amount)
	Parkour.lastStaminaUse = os.clock()
	UI.setStamina(Parkour.stamina, CONFIG.StaminaMax)
	return true
end

local function updateStamina(dt)
	local now = os.clock()
	local drain = 0
	if Parkour.mode == "WallRun" then drain = CONFIG.WallRunStaminaPerSecond
	elseif Parkour.mode == "Hang" then drain = CONFIG.HangStaminaPerSecond end
	if drain > 0 then
		Parkour.stamina = math.max(0, Parkour.stamina - drain * dt)
		if Parkour.stamina <= 0 then
			if Parkour.mode == "Hang" then
				-- Sem fôlego, solta a borda em vez de prender o jogador indefinidamente.
				local hum, root = State.Humanoid, State.Root
				resetParkour(true)
				if root and root.Parent then root.AssemblyLinearVelocity = Vector3.new(0, -3, 0) end
				if hum and hum.Parent then hum:ChangeState(ST.Freefall) end
			elseif Parkour.mode == "WallRun" then
				Parkour.mode = "None"; Parkour.endsAt = 0; Parkour.wallNormal = nil
			end
		end
	elseif now - Parkour.lastStaminaUse > CONFIG.StaminaRegenerationDelay then
		local hum = State.Humanoid
		local grounded = hum and hum.Parent and hum.FloorMaterial ~= Enum.Material.Air
		local regen = CONFIG.StaminaRegeneration * (grounded and 1 or 0.42)
		if Parkour.mode == "None" then
			Parkour.stamina = math.min(CONFIG.StaminaMax, Parkour.stamina + regen * dt)
		end
	end
	UI.setStamina(Parkour.stamina, CONFIG.StaminaMax)
end

local function canStartAction(modeName)
	local hum, root, rig = State.Humanoid, State.Root, State.Rig
	if State.InVehicle then return false end
	if not (hum and root and rig and rig.Active and hum.Health > 0) then return false end
	if hum.Sit or hum:GetState() == ST.Dead then return false end
	if Parkour.mode == "Mantle" or Parkour.mode == "WallRun" or Parkour.mode == "Dash" or Parkour.mode == "Roll" then return false end
	if Parkour.mode == "Hang" and modeName ~= "Mantle" and modeName ~= "Drop" then return false end
	return true
end

local function nextActionSerial()
	Parkour.actionSerial += 1
	return Parkour.actionSerial
end

flatDirection = function()
	local hrp, hum = State.Root, State.Humanoid
	if not hrp then return Vector3.new(0, 0, -1) end
	local dir = hum and hum.MoveDirection or Vector3.zero
	if dir.Magnitude < 0.2 then dir = hrp.CFrame.LookVector end
	dir = Vector3.new(dir.X, 0, dir.Z)
	if dir.Magnitude < 0.05 then dir = Vector3.new(0, 0, -1) end
	return dir.Unit
end

local function setParkourAnimation(kind, duration)
	local rig = State.Rig
	if not (rig and rig.Anim) then return end
	rig.Anim.action = nil
	rig.Anim.fidget = nil
	rig.Anim.parkour = { kind = kind, t = 0, dur = duration or 1 }
end

local function finishSlide()
	if Parkour.mode ~= "Slide" then return end
	restoreMovementControl()
	Parkour.mode = "None"
	Parkour.endsAt = 0
end

local function startSlide()
	if State.InVehicle then return end
	local hum, hrp, rig = State.Humanoid, State.Root, State.Rig
	if not (hum and hrp and rig and rig.Active and hum.Health > 0) then return end
	local now = os.clock()
	if now < Parkour.cooldown or Parkour.mode == "Hang" or Parkour.mode == "Mantle" then return end
	local state = hum:GetState()
	local grounded = state ~= ST.Freefall and state ~= ST.Jumping and state ~= ST.FallingDown and state ~= ST.Swimming
	local direction = flatDirection()
	local velocity = hrp.AssemblyLinearVelocity
	local speed = Vector3.new(velocity.X, 0, velocity.Z).Magnitude
	if not grounded or speed < 5 or hum.Sit then
		UI.setAnim("Slide: corra e pressione novamente", C_WARN)
		return
	end
	if Parkour.mode == "Slide" then return end
	if not spendStamina(CONFIG.SlideStaminaCost, "slide") then return end
	saveMovementControl(hum)
	Parkour.mode = "Slide"
	Parkour.direction = direction
	Parkour.endsAt = now + 0.72
	Parkour.cooldown = now + 1.05
	Parkour.movementStartedAt = now
	nextActionSerial()
	hum.WalkSpeed = math.max(7, (Parkour.savedWalkSpeed or hum.WalkSpeed) * 0.48)
	local boost = math.min(25, math.max(15, speed + 5))
	hrp.AssemblyLinearVelocity = direction * boost + Vector3.new(0, math.min(velocity.Y, 0), 0)
	setParkourAnimation("slide", 0.72)
end

local function beginMantle()
	if State.InVehicle then return end
	local hum, hrp = State.Humanoid, State.Root
	if not (hum and hrp and hum.Health > 0) then return end
	if Parkour.mode == "Hang" and not spendStamina(CONFIG.MantleStaminaCost, "subida") then
		return
	end
	local dir = Parkour.direction or flatDirection()
	restoreMovementControl()
	Parkour.mode = "Mantle"
	Parkour.endsAt = os.clock() + 0.72
	Parkour.hangPosition = nil
	Parkour.movementStartedAt = os.clock()
	nextActionSerial()
	local v = hrp.AssemblyLinearVelocity
	hrp.AssemblyLinearVelocity = Vector3.new(dir.X * 12, math.max(23, v.Y), dir.Z * 12)
	hum:ChangeState(ST.Jumping)
	setParkourAnimation("mantle", 0.82)
	UI.setParkourText("PARKOUR")
end

--------------------------------------------------------------------
-- V6 ACTIONS: DASH, ROLL, DROP FROM LEDGE
--------------------------------------------------------------------
local function dropFromLedge()
	if State.InVehicle then return false end
	if Parkour.mode ~= "Hang" then return false end
	local hum, root = State.Humanoid, State.Root
	restoreMovementControl()
	Parkour.mode = "None"
	Parkour.endsAt = 0
	Parkour.hangPosition = nil
	Parkour.wallNormal = nil
	if root and root.Parent then
		local dir = Parkour.direction or flatDirection()
		root.AssemblyLinearVelocity = Vector3.new(dir.X * 2, -4, dir.Z * 2)
	end
	if hum and hum.Parent then hum:ChangeState(ST.Freefall) end
	if State.Rig and State.Rig.Anim then State.Rig.Anim.parkour = nil end
	UI.setParkourText("PARKOUR")
	UI.setAnim("Borda solta", C_WARN)
	return true
end

local function startDash()
	if not canStartAction("Dash") then
		if Parkour.mode == "Hang" then dropFromLedge() end
		return
	end
	local hum, root, rig = State.Humanoid, State.Root, State.Rig
	local now = os.clock()
	if now < Parkour.dashCooldown or now < Parkour.cooldown then return end
	if not spendStamina(CONFIG.DashStaminaCost, "dash") then return end
	local direction = flatDirection()
	local velocity = root.AssemblyLinearVelocity
	local currentSpeed = Vector3.new(velocity.X, 0, velocity.Z).Magnitude
	local targetSpeed = math.max(CONFIG.DashSpeed, currentSpeed + 7)
	if hum.FloorMaterial == Enum.Material.Air then
		targetSpeed = math.min(targetSpeed, CONFIG.DashSpeed + 2)
	end
	root.AssemblyLinearVelocity = Vector3.new(direction.X * targetSpeed, velocity.Y, direction.Z * targetSpeed)
	Parkour.mode = "Dash"
	Parkour.direction = direction
	Parkour.endsAt = now + CONFIG.DashDuration
	Parkour.dashCooldown = now + CONFIG.DashCooldown
	Parkour.cooldown = now + 0.18
	Parkour.movementStartedAt = now
	nextActionSerial()
	setParkourAnimation("dash", 0.42)
	UI.setAnim("Dash", C_OK)
end

local function startRoll(automatic)
	if not canStartAction("Roll") then return false end
	local hum, root = State.Humanoid, State.Root
	if not (hum and root) then return false end
	local now = os.clock()
	local grounded = hum.FloorMaterial ~= Enum.Material.Air
	local velocity = root.AssemblyLinearVelocity
	local speed = Vector3.new(velocity.X, 0, velocity.Z).Magnitude
	if not automatic and (not grounded or speed < 3.2) then
		UI.setAnim("Rolamento: corra primeiro", C_WARN)
		return false
	end
	if now < Parkour.cooldown then return false end
	if not spendStamina(CONFIG.RollStaminaCost, "rolamento") then return false end
	local direction = flatDirection()
	saveMovementControl(hum)
	Parkour.mode = "Roll"
	Parkour.direction = direction
	Parkour.endsAt = now + CONFIG.RollDuration
	Parkour.cooldown = now + 0.95
	Parkour.movementStartedAt = now
	nextActionSerial()
	hum.WalkSpeed = math.max(8, (Parkour.savedWalkSpeed or hum.WalkSpeed) * 0.72)
	local push = automatic and math.max(8, speed * 0.62) or math.max(11, speed + 1.5)
	root.AssemblyLinearVelocity = Vector3.new(direction.X * push, math.min(velocity.Y, 0), direction.Z * push)
	setParkourAnimation("roll", CONFIG.RollDuration)
	UI.setAnim(automatic and "Rolamento de impacto" or "Rolamento", C_OK)
	return true
end

local function resetActionResources()
	Parkour.stamina = CONFIG.StaminaMax
	Parkour.lastStaminaUse = 0
	Parkour.lastStaminaUpdate = os.clock()
	Parkour.dashCooldown = 0
	Parkour.lastImpactScan = 0
	Parkour.lastImpactAt = 0
	Parkour.lastImpactPart = nil
	UI.setStamina(Parkour.stamina, CONFIG.StaminaMax)
end

local function tryWallJump()
	local hum, hrp = State.Humanoid, State.Root
	if not (hum and hrp and hum.Health > 0) then return false end
	local now = os.clock()

	-- Pular durante a corrida na parede deve impulsionar para fora, mesmo com cooldown ativo.
	if Parkour.mode == "WallRun" and Parkour.wallNormal then
		if not spendStamina(8, "wall-jump") then return false end
		local normal = Vector3.new(Parkour.wallNormal.X, 0, Parkour.wallNormal.Z)
		if normal.Magnitude < 0.1 then normal = -flatDirection() else normal = normal.Unit end
		local current = hrp.AssemblyLinearVelocity
		Parkour.mode = "None"
		Parkour.endsAt = 0
		Parkour.wallNormal = nil
		Parkour.cooldown = now + 0.32
		hrp.AssemblyLinearVelocity = normal * 22 + Vector3.new(0, math.max(22, current.Y + 10), 0)
		setParkourAnimation("walljump", 0.58)
		UI.setAnim("Wall-jump: salto para longe da parede", C_OK)
		return true
	end

	if now < Parkour.cooldown then return false end
	local st = hum:GetState()
	if st ~= ST.Freefall and st ~= ST.Jumping then return false end
	local dir = flatDirection()
	local params = getRayParams()
	local hit = Workspace:Raycast(hrp.Position + Vector3.new(0, 0.4, 0), dir * 2.6, params)
	if not hit or hit.Normal.Y > 0.45 then return false end
	if not spendStamina(12, "wall-jump") then return false end
	local away = Vector3.new(hit.Normal.X, 0, hit.Normal.Z)
	if away.Magnitude < 0.1 then away = -dir else away = away.Unit end
	local v = hrp.AssemblyLinearVelocity
	hrp.AssemblyLinearVelocity = away * 20 + Vector3.new(0, math.max(22, v.Y + 9), 0)
	setParkourAnimation("walljump", 0.58)
	Parkour.cooldown = now + 0.32
	return true
end

local function tryWallRun()
	local hum, hrp = State.Humanoid, State.Root
	if not (hum and hrp and hum.Health > 0) then return false end
	local now = os.clock()
	if now < Parkour.cooldown or Parkour.mode ~= "None" then return false end
	if Parkour.stamina < 12 then return false end
	local st = hum:GetState()
	if st ~= ST.Freefall and st ~= ST.Jumping then return false end
	local move = hum.MoveDirection
	local flatMove = Vector3.new(move.X, 0, move.Z)
	if flatMove.Magnitude < 0.35 then return false end
	local params = getRayParams()
	local origin = hrp.Position + Vector3.new(0, 0.5, 0)
	local hit = Workspace:Raycast(origin, flatMove.Unit * 2.0, params)
	if not hit or math.abs(hit.Normal.Y) > 0.25 then return false end
	local normal = Vector3.new(hit.Normal.X, 0, hit.Normal.Z)
	if normal.Magnitude < 0.1 then return false end
	normal = normal.Unit
	local tangent = Vector3.new(-normal.Z, 0, normal.X)
	if tangent:Dot(flatMove.Unit) < 0 then tangent = -tangent end
	local v = hrp.AssemblyLinearVelocity
	if not spendStamina(10, "wall-run") then return false end
	Parkour.mode = "WallRun"
	Parkour.endsAt = now + CONFIG.WallRunDuration
	Parkour.direction = tangent
	Parkour.wallNormal = normal
	Parkour.cooldown = now + CONFIG.WallRunCooldown
	hrp.AssemblyLinearVelocity = tangent * CONFIG.WallRunSpeed + Vector3.new(0, math.max(3, v.Y + 2), 0)
	setParkourAnimation("wallrun", CONFIG.WallRunDuration)
	UI.setAnim("Wall-run lateral", C_OK)
	return true
end

local function performParkour()
	if State.InVehicle then return end
	local hum, hrp, rig = State.Humanoid, State.Root, State.Rig
	if not (hum and hrp and rig and rig.Active and hum.Health > 0) then return end
	if Parkour.mode == "Hang" then beginMantle(); return end
	if Parkour.mode == "Slide" or Parkour.mode == "Mantle" or Parkour.mode == "WallRun" or Parkour.mode == "Dash" or Parkour.mode == "Roll" then return end
	if tryWallRun() then return end
	if tryWallJump() then return end
	local now = os.clock()
	if now < Parkour.cooldown then return end
	if Parkour.stamina < CONFIG.VaultStaminaCost then
		UI.setAnim("Fôlego baixo", C_WARN)
		return
	end
	refreshSurfaceSense(true)
	local dir = flatDirection()
	local params = getRayParams()
	local groundY = hrp.Position.Y - hum.HipHeight - hrp.Size.Y * 0.5
	local lowOrigin = hrp.Position - Vector3.new(0, hum.HipHeight + hrp.Size.Y * 0.5, 0) + Vector3.new(0, 0.55, 0)
	local lowHit = Workspace:Raycast(lowOrigin, dir * CONFIG.ParkourScanDistance, params)
	local highHit = Workspace:Raycast(hrp.Position + Vector3.new(0, 1.75, 0), dir * CONFIG.ParkourScanDistance, params)
	local topOrigin = hrp.Position + dir * 2.25 + Vector3.new(0, 5.5, 0)
	local topHit = Workspace:Raycast(topOrigin, Vector3.new(0, -8.0, 0), params)
	local topHeight = topHit and (topHit.Position.Y - groundY) or math.huge
	local topStandClear = topHit and canOccupyStandingPosition(topHit.Position, hrp, hum) or false
	if lowHit and lowHit.Normal.Y < 0.55 and topHit and topHit.Normal.Y > 0.55 and topHeight > 0.25 and topHeight <= 5.3 then
		Parkour.direction = dir
		if topHeight <= 1.7 and not highHit and topStandClear then
			if not spendStamina(CONFIG.VaultStaminaCost, "vault") then return end
			local v = hrp.AssemblyLinearVelocity
			hrp.AssemblyLinearVelocity = Vector3.new(dir.X * math.max(17, Vector3.new(v.X,0,v.Z).Magnitude + 5), 13, dir.Z * math.max(17, Vector3.new(v.X,0,v.Z).Magnitude + 5))
			hum:ChangeState(ST.Jumping)
			setParkourAnimation("vault", 0.62)
			Parkour.cooldown = now + 0.45
		else
			-- Pendura ligeiramente abaixo da borda e trava só o controle local enquanto segura.
			if not spendStamina(5, "agarrar borda") then return end
			local target = topHit.Position - dir * 1.18 - Vector3.new(0, 2.25, 0)
			if (target - hrp.Position).Magnitude > 5.8 then
				UI.setAnim("Parkour: borda fora de alcance", C_WARN)
				return
			end
			saveMovementControl(hum)
			Parkour.mode = "Hang"
			Parkour.direction = dir
			Parkour.hangPosition = target
			Parkour.movementStartedAt = now
			nextActionSerial()
			Parkour.endsAt = 0
			hum.WalkSpeed = 0
			hum.AutoRotate = false
			hrp.AssemblyLinearVelocity = Vector3.zero
			hum:ChangeState(ST.Physics)
			setParkourAnimation("hang", 9999)
			UI.setParkourText("SUBIR")
		end
		return
	end
	if lowHit and lowHit.Normal.Y < 0.55 and not highHit then
		-- Obstáculo baixo: só atravessa quando há espaço livre para a chegada.
		local landingPos = hrp.Position + dir * 2.25
		if not hasBoxClearance(CFrame.new(landingPos + Vector3.new(0, 1.5, 0)), Vector3.new(1.6, 3.0, 1.6)) then
			UI.setAnim("Vault bloqueado: sem espaço seguro", C_WARN)
			return
		end
		if not spendStamina(CONFIG.VaultStaminaCost, "vault") then return end
		local v = hrp.AssemblyLinearVelocity
		local push = math.max(16, Vector3.new(v.X,0,v.Z).Magnitude + 3)
		hrp.AssemblyLinearVelocity = Vector3.new(dir.X * push, 11, dir.Z * push)
		hum:ChangeState(ST.Jumping)
		setParkourAnimation("vault", 0.58)
		Parkour.cooldown = now + 0.42
	else
		UI.setAnim("Parkour: nenhuma borda segura detectada", C_WARN)
	end
end

local updateTelemetry, applyImpactToObjectInFront, getContextSuggestion

local function updateParkourMotion(dt)
	local hum, hrp = State.Humanoid, State.Root
	if not (hum and hrp and hum.Parent and hrp.Parent and hum.Health > 0) then
		if Parkour.mode ~= "None" then resetParkour(true) end
		return
	end
	local now = os.clock()
	refreshSurfaceSense(false)
	updateStamina(dt)
	updateTelemetry()
	local rootVelocity = hrp.AssemblyLinearVelocity
	local rootSpeed = Vector3.new(rootVelocity.X, 0, rootVelocity.Z).Magnitude
	applyImpactToObjectInFront(rootSpeed)
	if (Parkour.mode == "Slide" or Parkour.mode == "WallRun" or Parkour.mode == "Dash" or Parkour.mode == "Roll") and now >= Parkour.endsAt then
		if Parkour.mode == "Slide" then finishSlide()
		elseif Parkour.mode == "Roll" then restoreMovementControl(); Parkour.mode = "None"; Parkour.endsAt = 0
		elseif Parkour.mode == "Dash" then
			local endingVelocity = hrp.AssemblyLinearVelocity
			hrp.AssemblyLinearVelocity = Vector3.new(endingVelocity.X * 0.58, endingVelocity.Y, endingVelocity.Z * 0.58)
			Parkour.mode = "None"; Parkour.endsAt = 0
		else Parkour.mode = "None"; Parkour.endsAt = 0; Parkour.wallNormal = nil end
	elseif Parkour.mode == "Mantle" and now >= Parkour.endsAt then
		Parkour.mode = "None"
		Parkour.endsAt = 0
	end
	if Parkour.mode == "Dash" then
		-- O impulso inicial do dash é aplicado uma única vez; não teletransporta nem ancora o rig.
		local velocity = hrp.AssemblyLinearVelocity
		if velocity.Magnitude < 2 then Parkour.mode = "None"; Parkour.endsAt = 0 end
	end
	if Parkour.mode == "WallRun" then
		local normal = Parkour.wallNormal
		if not normal then Parkour.mode = "None"; return end
		local wall = Workspace:Raycast(hrp.Position + Vector3.new(0, 0.35, 0), -normal * 1.65, getRayParams())
		if not wall or math.abs(wall.Normal.Y) > 0.3 then
			Parkour.mode = "None"; Parkour.endsAt = 0; Parkour.wallNormal = nil
		else
			local v = hrp.AssemblyLinearVelocity
			local tangent = Parkour.direction or Vector3.new(0, 0, -1)
			hrp.AssemblyLinearVelocity = Vector3.new(tangent.X * CONFIG.WallRunSpeed, math.max(-5, v.Y - 5 * dt), tangent.Z * CONFIG.WallRunSpeed)
		end
	end
	if Parkour.mode == "Hang" and Parkour.hangPosition then
		-- O jogador pode mover-se lateralmente enquanto a borda continuar existindo.
		local move = hum.MoveDirection
		local tangent = Vector3.new(-(Parkour.direction or Vector3.new(0,0,-1)).Z, 0, (Parkour.direction or Vector3.new(0,0,-1)).X)
		if move.Magnitude > 0.25 and now - Parkour.lastShimmy > 0.08 then
			local side = move:Dot(tangent)
			if math.abs(side) > 0.2 then
				local shift = tangent * math.sign(side) * CONFIG.LedgeShimmySpeed * dt
				local candidate = Parkour.hangPosition + shift
				local ledgeRay = Workspace:Raycast(candidate + (Parkour.direction or Vector3.new(0,0,-1)) * 0.9 + Vector3.new(0,1.0,0), Vector3.new(0,-2.2,0), getRayParams())
				if ledgeRay and ledgeRay.Normal.Y > 0.55 then Parkour.hangPosition = candidate end
				Parkour.lastShimmy = now
			end
		end
		-- Correção de velocidade suave, sem ancorar nem teleportar o personagem.
		local delta = Parkour.hangPosition - hrp.Position
		if delta.Magnitude > 7 then
			resetParkour(true)
			hum:ChangeState(ST.Freefall)
			return
		end
		local correction = delta * 9 - hrp.AssemblyLinearVelocity * 0.35 + Vector3.new(0, Workspace.Gravity * dt * 0.8, 0)
		if correction.Magnitude > 17 then correction = correction.Unit * 17 end
		hrp.AssemblyLinearVelocity = correction
	end
end

local function overlayParkour(T, p)
	local t = p.t or 0
	local duration = math.max(0.01, p.dur or 0.7)
	local u = clamp(t / duration, 0, 1)
	local w = p.kind == "hang" and 1 or sin(u * math.pi)
	if p.kind == "slide" then
		ov(T.Waist, "rx", rad(24), 0.95)
		ov(T.Neck, "rx", -rad(13), 0.9)
		ov(T.HipL, "rx", rad(72), 0.95)
		ov(T.HipR, "rx", rad(54), 0.95)
		ov(T.ShL, "rx", rad(54), 0.85); ov(T.ShR, "rx", rad(48), 0.85)
		ov(T.ShL, "rz", -rad(18), 0.8); ov(T.ShR, "rz", rad(18), 0.8)
	elseif p.kind == "vault" then
		ov(T.Waist, "rx", -rad(20), w)
		ov(T.ShL, "rx", rad(145), w); ov(T.ShR, "rx", rad(145), w)
		ov(T.ShL, "rz", -rad(22), w); ov(T.ShR, "rz", rad(22), w)
		ov(T.HipL, "rx", rad(78), w); ov(T.HipR, "rx", rad(70), w)
		ov(T.Neck, "rx", rad(8), w)
	elseif p.kind == "hang" then
		local sway = sin(t * 3) * rad(3)
		ov(T.ShL, "rx", rad(160) + sway, 1); ov(T.ShR, "rx", rad(160) - sway, 1)
		ov(T.ShL, "rz", -rad(15), 1); ov(T.ShR, "rz", rad(15), 1)
		ov(T.HipL, "rx", rad(26) + sin(t * 4) * rad(5), 1)
		ov(T.HipR, "rx", rad(34) - sin(t * 4) * rad(5), 1)
		ov(T.Waist, "rx", -rad(12), 1); ov(T.Neck, "rx", rad(8), 1)
	elseif p.kind == "mantle" then
		ov(T.ShL, "rx", rad(145), w); ov(T.ShR, "rx", rad(145), w)
		ov(T.Waist, "rx", -rad(28), w)
		ov(T.HipL, "rx", rad(90) * w, w); ov(T.HipR, "rx", rad(75) * w, w)
		ov(T.Neck, "rx", -rad(8), w)
	elseif p.kind == "wallrun" then
		local cycle = sin(t * 17)
		ov(T.Waist, "rx", -rad(18), 0.9)
		ov(T.Waist, "rz", rad(12), 0.8)
		ov(T.ShL, "rx", rad(82) + cycle * rad(18), 0.95)
		ov(T.ShR, "rx", rad(38) - cycle * rad(15), 0.95)
		ov(T.HipL, "rx", rad(62) - cycle * rad(24), 0.95)
		ov(T.HipR, "rx", rad(38) + cycle * rad(20), 0.95)
		ov(T.Neck, "rx", rad(8), 0.7)
	elseif p.kind == "dash" then
		local drive = sin(u * math.pi)
		ov(T.Waist, "rx", rad(30), 0.96)
		ov(T.Waist, "rz", rad(5) * (1 - u), 0.85)
		ov(T.Neck, "rx", -rad(20), 0.9)
		ov(T.ShL, "rx", rad(108), 0.9); ov(T.ShR, "rx", rad(72), 0.9)
		ov(T.ShL, "rz", -rad(28), 0.8); ov(T.ShR, "rz", rad(24), 0.8)
		ov(T.HipL, "rx", rad(35) + drive * rad(18), 0.9)
		ov(T.HipR, "rx", rad(48) - drive * rad(16), 0.9)
	elseif p.kind == "walljump" then
		ov(T.ShL, "rx", rad(150), w); ov(T.ShR, "rx", rad(150), w)
		ov(T.ShL, "rz", -rad(32), w); ov(T.ShR, "rz", rad(32), w)
		ov(T.HipL, "rx", rad(45), w); ov(T.HipR, "rx", -rad(25), w)
		ov(T.Waist, "rx", -rad(12), w)
	elseif p.kind == "roll" then
		ov(T.Waist, "rx", rad(42), w)
		ov(T.HipL, "rx", rad(86), w); ov(T.HipR, "rx", rad(82), w)
		ov(T.ShL, "rx", rad(95), w); ov(T.ShR, "rx", rad(95), w)
		ov(T.Neck, "rx", -rad(18), w)
	end
end

local function classifyDestructible(part)
	if not part or not part:IsA("BasePart") or not part.Parent then return false end
	if part:GetAttribute("LegoBreakable") == false then return false end
	-- Names/ancestrais estruturalmente protegidos têm prioridade até sobre atributos explícitos.
	local protectedLower = string.lower(part.Name)
	for _, protectedWord in ipairs(PROTECTED_WORDS) do
		if string.find(protectedLower, protectedWord, 1, true) then return false end
	end
	if part:GetAttribute("LegoBreakable") == true then
		if not part.Anchored or not part.CanCollide or part.Transparency > 0.35 then return false end
		return true
	end
	if not part.Anchored or not part.CanCollide or part.Transparency > 0.35 or (State.Character and part:IsDescendantOf(State.Character)) then return false end
	local name = string.lower(part.Name)
	local explicit = string.find(name, "breakable", 1, true) ~= nil
	for _, word in ipairs(PROTECTED_WORDS) do
		if string.find(name, word, 1, true) then return false end
	end
	local ancestor = part.Parent
	for _ = 1, 3 do
		if not ancestor or ancestor == Workspace then break end
		local an = string.lower(ancestor.Name)
		if string.find(an, "breakable", 1, true) then explicit = true end
		for _, word in ipairs({"baseplate", "terrain", "foundation", "building_shell", "main_structure", "collision", "hitbox"}) do
			if string.find(an, word, 1, true) then return false end
		end
		ancestor = ancestor.Parent
	end
	local size = part.Size
	local maxSide = math.max(size.X, size.Y, size.Z)
	local minSide = math.min(size.X, size.Y, size.Z)
	local volume = size.X * size.Y * size.Z
	if maxSide > 10 or volume > 300 or minSide < 0.12 then return false end
	if maxSide > 6 and minSide < 0.55 then return false end -- evita la maioria dos pisos/placas finas
	local score = explicit and 4 or 0
	for _, word in ipairs(LIKELY_WORDS) do
		if string.find(name, word, 1, true) then score += 2; break end
	end
	local material = part.Material
	if material == Enum.Material.Wood or material == Enum.Material.WoodPlanks or material == Enum.Material.Glass
		or material == Enum.Material.Metal or material == Enum.Material.DiamondPlate or material == Enum.Material.CorrodedMetal
		or material == Enum.Material.Brick or material == Enum.Material.Concrete or material == Enum.Material.Cobblestone then
		score += 1
	end
	if maxSide <= 4.5 and volume <= 90 and minSide >= 0.2 then score += 1 end
	if part:IsA("MeshPart") and not explicit and score < 3 then return false end
	return score >= 2
end

local function makeHighlight(part)
	local folder = ensureFXFolder()
	-- SelectionBox permite uma linha fina configurável; Highlight não tem espessura ajustável.
	local box = Instance.new("SelectionBox")
	box.Name = "LegoBreakableOutline"
	box.Adornee = part
	box.Color3 = Color3.fromRGB(132, 136, 144)
	box.LineThickness = 0.025
	box.SurfaceTransparency = 1
	box.Transparency = 0.18
	box.Parent = folder
	return box
end

local function scanDestructibles()
	local hrp = State.Root
	if not (hrp and hrp.Parent and State.Character) then return end
	ensureFXFolder()
	local now = os.clock()
	if now < Destruction.scanAt then return end
	Destruction.scanAt = now + CONFIG.BreakScanInterval
	local params = OverlapParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	params.FilterDescendantsInstances = { State.Character, Destruction.FX }
	params.MaxParts = 160
	local ok, parts = pcall(function() return Workspace:GetPartBoundsInRadius(hrp.Position, CONFIG.BreakScanRadius, params) end)
	if not ok or type(parts) ~= "table" then return end
	local added = 0
	for _, part in ipairs(parts) do
		if added >= CONFIG.BreakMaxHighlights then break end
		if not Destruction.Entries[part] and not isHardProtectedPart(part) and classifyDestructible(part) then
			local count = 0
			for _ in pairs(Destruction.Entries) do count += 1 end
			if count < CONFIG.BreakMaxHighlights then
				local originalLTM = part.LocalTransparencyModifier
				Destruction.Entries[part] = { highlight = makeHighlight(part), damage = 0, cracks = {}, originalLTM = originalLTM, lastHit = 0 }
				added += 1
			end
		end
	end
	for part, entry in pairs(Destruction.Entries) do
		if not part.Parent or (part.Position - hrp.Position).Magnitude > CONFIG.BreakScanRadius + 12 then
			if part.Parent and entry.originalLTM ~= nil then pcall(function() part.LocalTransparencyModifier = entry.originalLTM end) end
			destroyIfAlive(entry.highlight)
			for _, crack in ipairs(entry.cracks) do destroyIfAlive(crack) end
			Destruction.Entries[part] = nil
		end
	end
end

local function spawnFragments(part, position, normal, amount)
	local folder = ensureFXFolder()
	for i = 1, amount do
		local shard = Instance.new("Part")
		shard.Name = "LocalDebrisShard"
		shard.Size = Vector3.new(0.13 + math.random() * 0.22, 0.13 + math.random() * 0.2, 0.13 + math.random() * 0.22)
		shard.Color = part.Color
		shard.Material = part.Material
		shard.Anchored = true
		shard.CanCollide = false; shard.CanTouch = false; shard.CanQuery = false
		shard.CastShadow = false
		shard.CFrame = CFrame.new(position + Vector3.new((math.random()-0.5)*0.6, math.random()*0.4, (math.random()-0.5)*0.6))
		shard.Parent = folder
		local n = Vector3.new(math.random()-0.5, 0.3 + math.random(), math.random()-0.5)
		if normal then n += normal * 0.35 end
		local goal = { CFrame = shard.CFrame + n.Unit * (0.8 + math.random()*1.3), Transparency = 1 }
		local tween = TweenService:Create(shard, TweenInfo.new(0.48 + math.random()*0.18, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), goal)
		tween:Play()
		task.delay(0.8, function() destroyIfAlive(shard) end)
	end
end

local function createCrackMarks(part, hitPosition, normal, entry)
	if #entry.cracks >= 10 then return end
	local globalCrackCount = 0
	for _, tracked in pairs(Destruction.Entries) do
		globalCrackCount += #(tracked.cracks or {})
		if globalCrackCount >= CONFIG.MaxCrackMarks then return end
	end
	local folder = ensureFXFolder()
	local face = CFrame.lookAt(hitPosition + normal * 0.025, hitPosition + normal)
	local available = math.min(3, 10 - #entry.cracks, CONFIG.MaxCrackMarks - globalCrackCount)
	for i = 1, available do
		local length = 0.22 + math.random() * 0.48
		local mark = Instance.new("Part")
		mark.Name = "LocalCrackMark"
		mark.Size = Vector3.new(length, 0.025, 0.025)
		mark.Color = Color3.fromRGB(48, 49, 52)
		mark.Material = Enum.Material.SmoothPlastic
		mark.Anchored = true
		mark.CanCollide = false; mark.CanTouch = false; mark.CanQuery = false
		mark.CastShadow = false
		local ox = (math.random()-0.5) * 0.52
		local oy = (math.random()-0.5) * 0.52
		mark.CFrame = face * CFrame.new(ox, oy, 0) * CFrame.Angles(0, 0, math.rad(math.random(-65,65)))
		mark.Parent = folder
		table.insert(entry.cracks, mark)
	end
end

local function restoreDamageLater(part, entry)
	local stamp = os.clock()
	entry.restoreStamp = stamp
	task.delay(CONFIG.BreakRestoreSeconds, function()
		if Destruction.Entries[part] ~= entry or entry.restoreStamp ~= stamp then return end
		if part.Parent and entry.originalLTM ~= nil then pcall(function() part.LocalTransparencyModifier = entry.originalLTM end) end
		for _, crack in ipairs(entry.cracks) do destroyIfAlive(crack) end
		entry.cracks = {}
		entry.damage = 0
		entry.lastHit = 0
		if entry.highlight and entry.highlight.Parent then
			entry.highlight.Color3 = Color3.fromRGB(132,136,144)
			entry.highlight.Transparency = 0.18
		end
	end)
end

--------------------------------------------------------------------
-- V6: LOCAL MATERIAL RESPONSE / PARTICLE BUDGET / IMPACT DAMAGE
-- Efeitos produzidos aqui são objetos locais criados por um LocalScript.
-- Nenhuma BasePart original é destruída, desancorada ou alterada no servidor.
--------------------------------------------------------------------
local LocalFXBudget = {
	created = 0,
	lastSweep = 0,
	lastGlobalBurst = 0,
	lastPartHit = setmetatable({}, { __mode = "k" }),
	activeAnchors = {},
	maxActive = CONFIG.MaxLocalFX,
}

local MATERIAL_RESPONSE = {
	[Enum.Material.Wood] = { threshold = 2, color = Color3.fromRGB(155, 105, 66), dust = Color3.fromRGB(174, 134, 88), shards = 6, soundClass = "wood" },
	[Enum.Material.WoodPlanks] = { threshold = 2, color = Color3.fromRGB(155, 105, 66), dust = Color3.fromRGB(174, 134, 88), shards = 7, soundClass = "wood" },
	[Enum.Material.Glass] = { threshold = 1, color = Color3.fromRGB(169, 225, 231), dust = Color3.fromRGB(180, 230, 237), shards = 9, soundClass = "glass" },
	[Enum.Material.Metal] = { threshold = 4, color = Color3.fromRGB(145, 151, 160), dust = Color3.fromRGB(174, 182, 194), shards = 5, soundClass = "metal" },
	[Enum.Material.DiamondPlate] = { threshold = 4, color = Color3.fromRGB(139, 148, 157), dust = Color3.fromRGB(169, 179, 188), shards = 5, soundClass = "metal" },
	[Enum.Material.CorrodedMetal] = { threshold = 4, color = Color3.fromRGB(133, 104, 79), dust = Color3.fromRGB(155, 124, 95), shards = 5, soundClass = "metal" },
	[Enum.Material.Concrete] = { threshold = 4, color = Color3.fromRGB(134, 134, 130), dust = Color3.fromRGB(165, 163, 157), shards = 7, soundClass = "stone" },
	[Enum.Material.Brick] = { threshold = 4, color = Color3.fromRGB(151, 80, 61), dust = Color3.fromRGB(170, 115, 91), shards = 7, soundClass = "stone" },
	[Enum.Material.Cobblestone] = { threshold = 4, color = Color3.fromRGB(125, 126, 124), dust = Color3.fromRGB(157, 158, 154), shards = 7, soundClass = "stone" },
	[Enum.Material.Plastic] = { threshold = 3, color = Color3.fromRGB(160, 165, 174), dust = Color3.fromRGB(183, 187, 195), shards = 5, soundClass = "plastic" },
	[Enum.Material.SmoothPlastic] = { threshold = 3, color = Color3.fromRGB(160, 165, 174), dust = Color3.fromRGB(183, 187, 195), shards = 5, soundClass = "plastic" },
}

local function getMaterialResponse(part)
	if not part then return { threshold = 3, color = Color3.fromRGB(155,155,155), dust = Color3.fromRGB(170,170,170), shards = 5, soundClass = "generic" } end
	local response = MATERIAL_RESPONSE[part.Material]
	if response then return response end
	return { threshold = 3, color = part.Color, dust = part.Color:Lerp(Color3.fromRGB(220,220,220), 0.35), shards = 5, soundClass = "generic" }
end

local function pruneLocalFX()
	local now = os.clock()
	if now - LocalFXBudget.lastSweep < 1.0 then return end
	LocalFXBudget.lastSweep = now
	for i = #LocalFXBudget.activeAnchors, 1, -1 do
		local item = LocalFXBudget.activeAnchors[i]
		if not item or not item.Parent then table.remove(LocalFXBudget.activeAnchors, i) end
	end
	LocalFXBudget.created = #LocalFXBudget.activeAnchors
end

local function trackFXAnchor(anchor)
	pruneLocalFX()
	if LocalFXBudget.created >= LocalFXBudget.maxActive then
		local oldest = table.remove(LocalFXBudget.activeAnchors, 1)
		destroyIfAlive(oldest)
		LocalFXBudget.created = math.max(0, LocalFXBudget.created - 1)
	end
	table.insert(LocalFXBudget.activeAnchors, anchor)
	LocalFXBudget.created += 1
	return anchor
end

local function newFXAnchor(position, name)
	local folder = ensureFXFolder()
	local anchor = Instance.new("Part")
	anchor.Name = name or "LocalImpactFXAnchor"
	anchor.Size = Vector3.new(0.12, 0.12, 0.12)
	anchor.CFrame = CFrame.new(position)
	anchor.Transparency = 1
	anchor.Anchored = true
	anchor.CanCollide = false
	anchor.CanTouch = false
	anchor.CanQuery = false
	anchor.CastShadow = false
	anchor.Massless = true
	anchor.Parent = folder
	trackFXAnchor(anchor)
	return anchor
end

local function emitMaterialDust(part, position, normal, amount, colorOverride)
	if not position then return end
	local now = os.clock()
	if now - LocalFXBudget.lastGlobalBurst < 0.025 then return end
	LocalFXBudget.lastGlobalBurst = now
	local response = getMaterialResponse(part)
	local anchor = newFXAnchor(position, "LocalMaterialBurst")
	local attachment = Instance.new("Attachment")
	attachment.Name = "BurstAttachment"
	attachment.Parent = anchor
	local emitter = Instance.new("ParticleEmitter")
	emitter.Name = "DustAndChips"
	emitter.Texture = "rbxasset://textures/particles/smoke_main.dds"
	emitter.Color = ColorSequence.new(colorOverride or response.dust)
	emitter.LightEmission = response.soundClass == "glass" and 0.25 or 0.02
	emitter.Rate = 0
	emitter.Lifetime = NumberRange.new(0.22, 0.55)
	emitter.Speed = NumberRange.new(2.5, response.soundClass == "glass" and 8.5 or 5.5)
	emitter.Drag = 5
	emitter.SpreadAngle = Vector2.new(42, 42)
	emitter.Rotation = NumberRange.new(0, 360)
	emitter.RotSpeed = NumberRange.new(-120, 120)
	emitter.Size = NumberSequence.new({
		NumberSequenceKeypoint.new(0, 0.13),
		NumberSequenceKeypoint.new(0.35, response.soundClass == "glass" and 0.18 or 0.28),
		NumberSequenceKeypoint.new(1, 0),
	})
	emitter.Transparency = NumberSequence.new({
		NumberSequenceKeypoint.new(0, 0.18),
		NumberSequenceKeypoint.new(0.7, 0.35),
		NumberSequenceKeypoint.new(1, 1),
	})
	emitter.Parent = attachment
	local emitCount = math.clamp(math.floor(amount or 6), 1, 14)
	pcall(function() emitter:Emit(emitCount) end)
	Debris:AddItem(anchor, 1.15)
	return anchor
end

local function spawnImpactRing(position, normal, color, scale)
	if not position or not normal then return end
	local anchor = newFXAnchor(position + normal * 0.025, "LocalImpactRing")
	anchor.Transparency = 1
	local ring = Instance.new("Part")
	ring.Name = "FadingImpactRing"
	ring.Shape = Enum.PartType.Cylinder
	ring.Material = Enum.Material.Neon
	ring.Color = color or Color3.fromRGB(175, 180, 190)
	ring.Size = Vector3.new(0.035, 0.22, 0.22)
	ring.Anchored = true
	ring.CanCollide = false
	ring.CanTouch = false
	ring.CanQuery = false
	ring.CastShadow = false
	local up = normal.Unit
	local base = CFrame.lookAt(position + up * 0.035, position + up)
	ring.CFrame = base * CFrame.Angles(math.rad(90), 0, 0)
	ring.Parent = ensureFXFolder()
	local targetScale = math.clamp(scale or 1, 0.65, 2.5)
	local tween = TweenService:Create(ring, TweenInfo.new(0.34, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
		Size = Vector3.new(0.025, 0.22 * targetScale, 0.22 * targetScale), Transparency = 1,
	})
	tween:Play()
	Debris:AddItem(ring, 0.42)
	Debris:AddItem(anchor, 0.45)
end

local function spawnMaterialFragments(part, position, normal, response, power)
	local baseCount = response and response.shards or 5
	local glassExtra = response and response.soundClass == "glass"
	local capForBase = math.max(3, CONFIG.MaxFragmentsPerImpact - (glassExtra and 2 or 0))
	local count = math.clamp(baseCount + math.floor((power or 1) - 1), 3, capForBase)
	spawnFragments(part, position, normal, count)
	if glassExtra then
		-- Duas lascas maiores reforçam visualmente o vidro sem criar colisão.
		local folder = ensureFXFolder()
		for i = 1, 2 do
			local shard = Instance.new("Part")
			shard.Name = "LocalGlassChip"
			shard.Size = Vector3.new(0.06, 0.22 + math.random() * 0.18, 0.025)
			shard.Color = response.color
			shard.Material = Enum.Material.Glass
			shard.Transparency = 0.28
			shard.Anchored = true
			shard.CanCollide = false
			shard.CanTouch = false
			shard.CanQuery = false
			shard.CastShadow = false
			shard.CFrame = CFrame.new(position) * CFrame.Angles(math.random(), math.random(), math.random())
			shard.Parent = folder
			local direction = ((normal or Vector3.yAxis) + Vector3.new(math.random() - 0.5, math.random() * 0.6, math.random() - 0.5)).Unit
			local tween = TweenService:Create(shard, TweenInfo.new(0.55, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
				CFrame = shard.CFrame + direction * (0.8 + math.random() * 1.2), Transparency = 1,
			})
			tween:Play()
			Debris:AddItem(shard, 0.65)
		end
	end
end

local function ensureDestructionEntry(part)
	local entry = Destruction.Entries[part]
	if entry then return entry end
	if not classifyDestructible(part) then return nil end
	local count = 0
	for _ in pairs(Destruction.Entries) do count += 1 end
	if count >= CONFIG.BreakMaxHighlights then return nil end
	entry = {
		highlight = makeHighlight(part), damage = 0, cracks = {},
		originalLTM = part.LocalTransparencyModifier, lastHit = 0,
		lastBurst = 0, material = part.Material, wasDamaged = false,
	}
	Destruction.Entries[part] = entry
	return entry
end

local function applyLocalDamageV6(part, hitPosition, normal, power, source)
	if not (part and part.Parent and part:IsA("BasePart")) then return false, "Peça inválida" end
	if isHardProtectedPart(part) or isLocalEffectPart(part) or not classifyDestructible(part) then
		return false, "Objeto protegido/incerto"
	end
	local now = os.clock()
	local lastPart = LocalFXBudget.lastPartHit[part] or 0
	if now - lastPart < 0.16 then return false, "Impacto repetido ignorado" end
	LocalFXBudget.lastPartHit[part] = now
	local entry = ensureDestructionEntry(part)
	if not entry then return false, "Limite local de contornos atingido" end
	if not hitPosition then hitPosition = part.Position end
	if not normal or normal.Magnitude < 0.01 then normal = Vector3.yAxis end
	normal = normal.Unit
	local response = getMaterialResponse(part)
	local strength = math.clamp(tonumber(power) or 1, 0.5, 3)
	local add = strength >= 2.1 and 2 or 1
	entry.damage += add
	entry.lastHit = now
	entry.lastBurst = now
	entry.wasDamaged = true
	createCrackMarks(part, hitPosition, normal, entry)
	emitMaterialDust(part, hitPosition, normal, 5 + math.floor(strength * 2), response.dust)
	spawnMaterialFragments(part, hitPosition, normal, response, strength)
	spawnImpactRing(hitPosition, normal, response.color, 0.8 + strength * 0.18)

	local threshold = response.threshold
	if part:GetAttribute("LegoBreakable") == true then threshold = math.max(1, threshold - 1) end
	if entry.damage >= threshold then
		-- Ocultação somente local. A colisão e o objeto real continuam intactos.
		part.LocalTransparencyModifier = math.max(entry.originalLTM or 0, 0.72)
		if entry.highlight and entry.highlight.Parent then
			entry.highlight.Color3 = Color3.fromRGB(184, 188, 196)
			entry.highlight.Transparency = 0.34
		end
		UI.setAnim("Dano visual local aplicado (" .. response.soundClass .. ")", C_OK)
	else
		UI.setAnim("Dano visual " .. tostring(entry.damage) .. "/" .. tostring(threshold), C_OK)
	end
	restoreDamageLater(part, entry)
	return true, entry.damage >= threshold and "Damaged" or "Cracked"
end

applyImpactToObjectInFront = function(speed)
	local root = State.Root
	if not (root and root.Parent) then return end
	local now = os.clock()
	if now - Parkour.lastImpactScan < CONFIG.ImpactScanInterval then return end
	Parkour.lastImpactScan = now
	if speed < CONFIG.ImpactMinSpeed then return end
	local velocity = root.AssemblyLinearVelocity
	local horizontal = Vector3.new(velocity.X, 0, velocity.Z)
	if horizontal.Magnitude < CONFIG.ImpactMinSpeed then return end
	local hit = rayAt(root.Position + Vector3.new(0, 0.35, 0), horizontal.Unit * 1.65, getRayParams())
	if not hit or not hit.Instance or not classifyDestructible(hit.Instance) then return end
	if now - Parkour.lastImpactAt < CONFIG.ImpactGlobalCooldown then return end
	Parkour.lastImpactAt = now
	applyLocalDamageV6(hit.Instance, hit.Position, hit.Normal, math.clamp(horizontal.Magnitude / 15, 1, 2.5), "Ram")
end

--------------------------------------------------------------------
-- Extra guard rules for local visuals. Explicit attributes are respected,
-- but never override hard-protected names such as floor/spawn/foundation.
--------------------------------------------------------------------
isHardProtectedPart = function(part)
	if not part or not part:IsA("BasePart") then return true end
	local name = string.lower(part.Name)
	for _, word in ipairs(PROTECTED_WORDS) do
		if string.find(name, word, 1, true) then return true end
	end
	local ancestor = part.Parent
	for _ = 1, 5 do
		if not ancestor or ancestor == Workspace then break end
		local an = string.lower(ancestor.Name)
		for _, word in ipairs(PROTECTED_WORDS) do
			if string.find(an, word, 1, true) then return true end
		end
		for _, word in ipairs({"terrain", "main_structure", "structural", "collision", "spawnarea", "criticalgeometry"}) do
			if string.find(an, word, 1, true) then return true end
		end
		ancestor = ancestor.Parent
	end
	return false
end

isLocalEffectPart = function(part)
	if not part or not Destruction.FX then return false end
	return part:IsDescendantOf(Destruction.FX)
end

local function canCreateMoreLocalFX()
	pruneLocalFX()
	return LocalFXBudget.created < LocalFXBudget.maxActive
end

local function cleanupExpiredLocalEntry(part, entry)
	if entry and entry.highlight and not entry.highlight.Parent then entry.highlight = nil end
	if entry and entry.cracks then
		for i = #entry.cracks, 1, -1 do
			local crack = entry.cracks[i]
			if not crack or not crack.Parent then table.remove(entry.cracks, i) end
		end
	end
	if not part or not part.Parent then
		Destruction.Entries[part] = nil
	end
end

local function getDestructionDiagnostics()
	local entries = 0
	local damaged = 0
	local outlineCount = 0
	local crackCount = 0
	for part, entry in pairs(Destruction.Entries) do
		if part and part.Parent then
			entries += 1
			if (entry.damage or 0) > 0 then damaged += 1 end
			if entry.highlight and entry.highlight.Parent then outlineCount += 1 end
			crackCount += #(entry.cracks or {})
		end
	end
	return {
		entries = entries,
		damaged = damaged,
		outlines = outlineCount,
		cracks = crackCount,
		fx = LocalFXBudget.created,
		running = Destruction.running,
	}
end

--------------------------------------------------------------------
-- Parkour telemetry helpers. These are intentionally lightweight and
-- can be used later for tuning without printing every frame.
--------------------------------------------------------------------
local ParkourTelemetry = {
	lastMode = "None",
	modeStartedAt = 0,
	lastReportAt = 0,
	actions = 0,
	failedScans = 0,
	lastReason = "",
	lastGroundMaterial = Enum.Material.Air,
	lastHintAt = 0,
	lastContextMessageAt = 0,
}

local function noteParkourMode(mode, reason)
	if ParkourTelemetry.lastMode ~= mode then
		ParkourTelemetry.lastMode = mode
		ParkourTelemetry.modeStartedAt = os.clock()
		ParkourTelemetry.actions += 1
		ParkourTelemetry.lastReason = tostring(reason or "")
	end
end

updateTelemetry = function()
	local hum = State.Humanoid
	if hum and hum.Parent then ParkourTelemetry.lastGroundMaterial = hum.FloorMaterial end
	noteParkourMode(Parkour.mode, ParkourTelemetry.lastReason)
	local now = os.clock()
	if now - (ParkourTelemetry.lastHintAt or 0) > 2.3 then
		ParkourTelemetry.lastHintAt = now
		local hint = getContextSuggestion()
		if Parkour.mode == "None" and hint ~= "Livre: corra, deslize ou use DASH" then
			-- Só publica dicas contextuais; não sobrescreve mensagens de dano/erro a cada quadro.
			if now - (ParkourTelemetry.lastContextMessageAt or 0) > 3.8 then
				ParkourTelemetry.lastContextMessageAt = now
				UI.setAnim(hint, C_NEUTRAL)
			end
		end
	end
	if now - ParkourTelemetry.lastReportAt > 8 then
		ParkourTelemetry.lastReportAt = now
		-- Resumo silencioso por padrão; só avisa quando há um erro repetido.
		if ParkourTelemetry.failedScans >= 5 then
			warn(TAG .. " vários scans falharam; último estado=" .. tostring(SurfaceSense.state))
			ParkourTelemetry.failedScans = 0
		end
	end
end

--------------------------------------------------------------------
-- Safe context suggestions. This doesn't trigger movement automatically;
-- the player keeps control and can choose when to use parkour actions.
--------------------------------------------------------------------
getContextSuggestion = function()
	local sense = refreshSurfaceSense(false)
	local hum = State.Humanoid
	if not hum or not hum.Parent then return "Aguardando personagem" end
	if Parkour.mode == "Hang" then return "Na borda: SUBIR ou SOLTAR" end
	if Parkour.mode == "Slide" then return "Deslizando" end
	if Parkour.mode == "WallRun" then return "Corrida na parede: pressione PARKOUR para saltar" end
	if Parkour.mode == "Dash" then return "Dash ativo" end
	if Parkour.mode == "Roll" then return "Rolamento" end
	if sense.state == "LowObstacle" and sense.landingClear then return "Obstáculo baixo: PARKOUR para vault" end
	if sense.state == "Wall" and sense.wallSide ~= "None" then return "Parede lateral: PARKOUR no ar para wall-run" end
	if sense.state == "DropAhead" then return "Queda à frente: avalie o salto" end
	if hum.FloorMaterial == Enum.Material.Air then return "No ar: agarre uma borda com PARKOUR" end
	return "Livre: corra, deslize ou use DASH"
end

--------------------------------------------------------------------
-- Functional regression probes (manual, safe, non-destructive).
-- These tests never modify the live character or real map parts.
--------------------------------------------------------------------
local function runLocalSelfChecks()
	local results = {}
	local function record(name, passed, details)
		table.insert(results, { name = name, passed = passed == true, details = tostring(details or "") })
	end

	record("Stamina bounds", Parkour.stamina >= 0 and Parkour.stamina <= CONFIG.StaminaMax, Parkour.stamina)
	record("Action mode known", ({None=true, Slide=true, Hang=true, Mantle=true, WallRun=true, Dash=true, Roll=true})[Parkour.mode] == true, Parkour.mode)
	record("FX folder local name", Destruction.FX == nil or Destruction.FX.Name == "LegoLocalFX_V6", Destruction.FX and Destruction.FX.Name or "not created")
	record("Outline cap", getDestructionDiagnostics().outlines <= CONFIG.BreakMaxHighlights, getDestructionDiagnostics().outlines)
	record("Fragment cap", CONFIG.MaxFragmentsPerImpact <= 12, CONFIG.MaxFragmentsPerImpact)
	local failed = 0
	for _, result in ipairs(results) do
		if not result.passed then failed += 1 end
	end
	return #results, failed, results
end

local function logLocalSelfChecks()
	local total, failed, results = runLocalSelfChecks()
	print(TAG .. " V6 self-check: " .. tostring(total - failed) .. "/" .. tostring(total) .. " passaram")
	for _, result in ipairs(results) do
		if not result.passed then warn(TAG .. " self-check falhou: " .. result.name .. " — " .. result.details) end
	end
	return failed == 0
end


--------------------------------------------------------------------
-- Destruction visibility helpers and material-aware stress transitions.
-- Each function is defensive because maps can remove/reparent parts at runtime.
--------------------------------------------------------------------
local function safelyGetPartAttribute(part, name)
	if not part or not part.Parent then return nil end
	local ok, result = pcall(function() return part:GetAttribute(name) end)
	return ok and result or nil
end

local function isCandidateFromAttribute(part)
	return safelyGetPartAttribute(part, "LegoBreakable") == true
end

local function getPartSizeClass(part)
	if not part then return "Unknown" end
	local size = part.Size
	local maxSide = math.max(size.X, size.Y, size.Z)
	local volume = size.X * size.Y * size.Z
	if maxSide <= 1.6 and volume <= 3 then return "SmallProp" end
	if maxSide <= 4 and volume <= 35 then return "MediumProp" end
	if maxSide <= 7 and volume <= 125 then return "LargeProp" end
	return "Oversized"
end

local function canShowOutlineFor(part)
	if not part or not part.Parent or not part:IsA("BasePart") then return false end
	if isHardProtectedPart(part) or isLocalEffectPart(part) then return false end
	if part.Transparency > 0.35 or not part.CanCollide then return false end
	if State.Character and part:IsDescendantOf(State.Character) then return false end
	return classifyDestructible(part)
end

local function getVisibleTarget(maxDistance)
	local camera = Workspace.CurrentCamera
	if not camera then return nil end
	local params = getRayParams()
	local ray = Workspace:Raycast(camera.CFrame.Position, camera.CFrame.LookVector * (maxDistance or CONFIG.BreakHitDistance), params)
	if ray and ray.Instance and ray.Instance:IsA("BasePart") then return ray.Instance, ray end
	return nil
end

local function getOutlineState(part)
	local entry = part and Destruction.Entries[part]
	if not entry then return "NotTracked" end
	if entry.damage and entry.damage > 0 then return "Damaged" end
	if entry.highlight and entry.highlight.Parent then return "Outlined" end
	return "Tracked"
end

local function setOutlineDamaged(entry, damaged)
	if not entry or not entry.highlight or not entry.highlight.Parent then return end
	if damaged then
		entry.highlight.Color3 = Color3.fromRGB(184, 188, 196)
		entry.highlight.Transparency = 0.34
	else
		entry.highlight.Color3 = Color3.fromRGB(132, 136, 144)
		entry.highlight.Transparency = 0.18
	end
end

local function updateOutlineColors()
	for part, entry in pairs(Destruction.Entries) do
		if not part or not part.Parent then
			destroyIfAlive(entry.highlight)
			Destruction.Entries[part] = nil
		elseif entry.highlight and entry.highlight.Parent then
			setOutlineDamaged(entry, (entry.damage or 0) > 0)
		end
	end
end

local function restorePartVisualOnly(part, entry)
	if not entry then return end
	if part and part.Parent and entry.originalLTM ~= nil then
		pcall(function() part.LocalTransparencyModifier = entry.originalLTM end)
	end
	for _, crack in ipairs(entry.cracks or {}) do destroyIfAlive(crack) end
	entry.cracks = {}
	entry.damage = 0
	entry.lastHit = 0
	entry.wasDamaged = false
	setOutlineDamaged(entry, false)
end

local function resetAllDamageVisuals()
	for part, entry in pairs(Destruction.Entries) do
		restorePartVisualOnly(part, entry)
	end
end

local function countNearbyDestructibles()
	local count = 0
	local damaged = 0
	local root = State.Root
	if not root then return count, damaged end
	for part, entry in pairs(Destruction.Entries) do
		if part and part.Parent and (part.Position - root.Position).Magnitude <= CONFIG.BreakScanRadius then
			count += 1
			if (entry.damage or 0) > 0 then damaged += 1 end
		end
	end
	return count, damaged
end

local function getDamageThreshold(part)
	local response = getMaterialResponse(part)
	if isCandidateFromAttribute(part) then return math.max(1, response.threshold - 1) end
	return response.threshold
end

local function applyDamageGlow(entry, current, threshold)
	if not entry or not entry.highlight or not entry.highlight.Parent then return end
	local ratio = math.clamp(current / math.max(1, threshold), 0, 1)
	entry.highlight.Color3 = Color3.fromRGB(
		math.floor(132 + 50 * ratio),
		math.floor(136 + 52 * ratio),
		math.floor(144 + 52 * ratio)
	)
	entry.highlight.Transparency = 0.18 + ratio * 0.18
end

local function describeBreakable(part)
	if not part then return "Nenhum objeto" end
	local response = getMaterialResponse(part)
	local sizeClass = getPartSizeClass(part)
	return tostring(part.Name) .. " / " .. tostring(response.soundClass) .. " / " .. sizeClass
end

local function getSafeDamagePoint(part)
	if not part then return nil, Vector3.yAxis end
	local camera = Workspace.CurrentCamera
	if camera then
		local target, hit = getVisibleTarget(CONFIG.BreakHitDistance)
		if target == part and hit then return hit.Position, hit.Normal end
	end
	return part.Position, Vector3.yAxis
end

local function inspectDestructionTarget()
	local part = getVisibleTarget(CONFIG.BreakHitDistance)
	if not part then return nil, "Nada na mira" end
	if not canShowOutlineFor(part) then return part, "Protegido ou incerto" end
	local entry = ensureDestructionEntry(part)
	if not entry then return part, "Limite visual atingido" end
	return part, getOutlineState(part)
end

local function localDamagePreview(part)
	if not part then return false end
	local position, normal = getSafeDamagePoint(part)
	if not position then return false end
	local entry = ensureDestructionEntry(part)
	if not entry then return false end
	spawnImpactRing(position, normal, Color3.fromRGB(139, 143, 151), 0.65)
	return true
end

local function auditLocalDestructionState()
	local diagnostics = getDestructionDiagnostics()
	local nearby, damaged = countNearbyDestructibles()
	return {
		tracked = diagnostics.entries,
		damaged = damaged,
		nearby = nearby,
		outlines = diagnostics.outlines,
		cracks = diagnostics.cracks,
		localFX = diagnostics.fx,
		maxOutlines = CONFIG.BreakMaxHighlights,
		maxFX = CONFIG.MaxLocalFX,
	}
end

local function breakTarget()
	local cam = Workspace.CurrentCamera
	if not cam then return end
	local params = getRayParams()
	local hit = Workspace:Raycast(cam.CFrame.Position, cam.CFrame.LookVector * CONFIG.BreakHitDistance, params)
	if not hit or not hit.Instance or not hit.Instance:IsA("BasePart") then
		UI.setAnim("Destruição: mire em um objeto contornado", C_WARN)
		return
	end
	local part = hit.Instance
	if isHardProtectedPart(part) or isLocalEffectPart(part) then
		UI.setAnim("Objeto estrutural/protegido", C_WARN)
		return
	end
	local ok, result = applyLocalDamageV6(part, hit.Position, hit.Normal, 1, "Button")
	if not ok then UI.setAnim(result or "Objeto protegido/incerto", C_WARN) end
end

local function startLocalSubsystem()
	Destruction.running = true
	task.spawn(function()
		while Destruction.running and script.Parent do
			if State.Character and State.Root and State.Enabled then
				local ok, err = pcall(scanDestructibles)
				if not ok then warn(TAG .. " scanner de objetos falhou: " .. tostring(err)) end
			end
			task.wait(0.25)
		end
	end)
end

----------------------------------------------------------------------
-- V6 FIELD GUIDE / INPUT MAP
-- MOBILE:
--   SLIDE   : deslize baixo com impulso; só ativa em movimento e no chão.
--   PARKOUR : tenta vault, agarrar borda ou wall-run conforme contexto.
--   SUBIR   : aparece no lugar de PARKOUR enquanto pendurado.
--   SOLTAR  : aparece na segunda fileira enquanto pendurado.
--   DASH    : impulso curto, limitado por fôlego e cooldown.
--   ROLAR   : rolamento manual em movimento; também pode ocorrer ao pousar forte.
--   QUEBRAR : aplica dano VISUAL local apenas na peça mirável e classificada.
--
-- PC test keys (as ações mobile continuam sendo o caminho principal):
--   F = soco, G = gesto, LeftControl = slide, E = parkour,
--   LeftShift = dash, X = rolamento.
--
-- PERFORMANCE BUDGET:
--   * Um único loop RenderStepped para as juntas do rig.
--   * Sensores contextuais executam a cada SenseInterval, não a cada render.
--   * A varredura de destrutíveis é incremental e limitada por BreakMaxHighlights.
--   * Fragmentos e partículas são locais, não colidem e são removidos automaticamente.
--   * Nenhum script altera o Parent, Anchored, Size ou CanCollide da geometria real.
--
-- LIMITAÇÕES INTENCIONAIS:
--   * Classificação por heurística não conhece a intenção de todo mapa; dúvida = protegido.
--   * Objetos unidos/modelos inteiros podem precisar de atributo LegoBreakable = true.
--   * Atributo explícito não supera os nomes/ancestrais protegidos nem peças não colidíveis.
--   * Ações locais do personagem usam velocidade no cliente; validação anticheat do jogo pode
--     limitar dash/parkour em experiências que impõem velocidades personalizadas.
----------------------------------------------------------------------

----------------------------------------------------------------------
-- Loop de animação (um único ciclo por quadro)
----------------------------------------------------------------------
local function setAnimLabel(A, text)
	if A.label ~= text then
		A.label = text
		UI.setAnim(text, C_OK)
	end
end

local function updateRig(dt)
	local rig = State.Rig
	if not rig or not rig.Active then return end
	local hum, hrp = State.Humanoid, State.Root
	if not (hum and hum.Parent and hrp and hrp.Parent) then return end -- personagem saindo (respawn)
	if not rig.Model.Parent then error("O modelo da minifigura foi removido do personagem", 0) end

	dt = math.min(dt, 0.1)
	local A = rig.Anim
	local now = os.clock()
	updateParkourMotion(dt)
	A.t += dt

	local vel = hrp.AssemblyLinearVelocity
	local speed = math.sqrt(vel.X * vel.X + vel.Z * vel.Z)
	local vy = vel.Y
	local state = hum:GetState()
	local dead = state == ST.Dead or hum.Health <= 0
	local inAir = state == ST.Freefall or state == ST.Jumping
	local seated = state == ST.Seated
	local climbing = state == ST.Climbing
	local airLike = inAir or state == ST.Swimming or state == ST.FallingDown or state == ST.Ragdoll
		or state == ST.Physics or state == ST.Flying
	local grounded = not (airLike or seated or climbing or dead)

	-- Pouso
	if inAir then
		A.lastVy = vy
		A.airTime += dt
	else
		if A.wasAir and not dead then
			local impact = -A.lastVy
			if impact > 6 then
				A.landT = 1
				A.landS = clamp(impact / 45, 0.4, 1)
				if impact > 30 and Parkour.mode == "None" then
					if CONFIG.AutoRollOnHardLanding and Parkour.stamina >= CONFIG.RollStaminaCost then
						startRoll(true)
					else
						A.parkour = { kind = "roll", t = 0, dur = 0.62 }
					end
				end
			end
		end
		A.airTime = 0
	end
	A.wasAir = inAir
	A.landT = math.max(0, A.landT - dt / 0.32)
	if grounded or seated or climbing then A.doubleUsed = false end

	-- Taxa de giro (reação em mudança de direção)
	local look = hrp.CFrame.LookVector
	local yaw = math.atan2(-look.X, -look.Z)
	if A.yaw then
		local d = yaw - A.yaw
		if d > math.pi then d -= 2 * math.pi elseif d < -math.pi then d += 2 * math.pi end
		A.yawRate += (d / math.max(dt, 1 / 240) - A.yawRate) * (1 - exp(-14 * dt))
	end
	A.yaw = yaw

	-- Calibração dos pés: raio para baixo quando parado no chão (barato: 4x por segundo)
	if state == ST.Running and speed < 1.5 and now - A.lastCalib > 0.25 then
		A.lastCalib = now
		local res = Workspace:Raycast(hrp.Position, Vector3.new(0, -(rig.BaseG + 4), 0), rig.RayParams)
		if res and abs(res.Distance - rig.BaseG) <= 1.5 then
			rig.GroundOffset = lerp(rig.GroundOffset, res.Distance + CONFIG.FootOffsetTweak, 0.5)
		end
	end
	if abs(rig.GroundOffset - rig.AppliedG) > 0.002 then
		rig.AppliedG = rig.GroundOffset
		rig.BaseC0.Root = CFrame.new(0, -rig.AppliedG, 0)
		if rig.DriveMode ~= "C0" then rig.Joints.Root.C0 = rig.BaseC0.Root end
	end

	local T = A.target
	resetPose(T)
	local expr = "Happy"
	local label = "Parado"
	local fast = false
	local plantTarget = 0
	local act = A.action

	if dead then
		A.action = nil
		act = nil
		A.deadT += dt
		label = "Desativado"
		expr = "KO"
		fast = true
		poseDead(A, T)
		if A.deadT > 1.5 and not A.deadHidden then
			A.deadHidden = true
			for _, p in ipairs(rig.Parts) do p.Transparency = 1 end
		end
	else
		if A.deadT > 0 then -- reviveu
			A.deadT = 0
			if A.deadHidden then
				A.deadHidden = false
				for _, p in ipairs(rig.Parts) do p.Transparency = 0 end
				rig.RefreshFace()
			end
		end

		if act then
			act.t += dt
			if act.kind == "punch" and not act.fired and act.t >= (act.strikeAt or 0.12) then
				act.fired = true
				if Hooks.OnPunchStrike then
					local ok, e = pcall(Hooks.OnPunchStrike, act.side, hrp, act.variant)
					if not ok then warn(TAG .. " Hooks.OnPunchStrike falhou: " .. tostring(e)) end
				end
			end
			if act.t >= act.dur or (act.kind == "emote" and (airLike or speed > (act.name == "Comemorar" and 10 or 4))) then
				A.action = nil
				act = nil
			end
		end

		if A.flipT then
			A.flipT += dt / 0.5
			if A.flipT >= 1 then
				A.flipT = nil
				A.flipAngle = 0
			else
				A.flipAngle = -smooth(A.flipT) * 2 * math.pi
			end
		end

		-- Parada brusca (derrapada) e fidgets de idle
		A.peak = math.max(speed, A.peak - 25 * dt)
		if grounded and A.skidT <= 0 and A.peak > 12 and speed < A.peak * 0.45 then
			A.skidT = 1
			A.peak = speed
		end
		if not grounded then A.skidT = 0 end
		A.skidT = math.max(0, A.skidT - dt / 0.45)
		local idleNow = grounded and speed < 0.8 and not act and A.skidT <= 0 and A.landT <= 0
		if A.fidget then
			A.fidget.t += dt
			if not idleNow or A.fidget.t >= A.fidget.dur then
				A.fidget = nil
				A.nextFidget = now + 5 + math.random() * 5
			end
		elseif idleNow and now >= A.nextFidget then
			local kind = FIDGETS[math.random(#FIDGETS)]
			A.fidget = { kind = kind, t = 0, dur = (kind == "stretch" and 2.2) or (kind == "tap" and 1.6) or 1.8 }
		elseif not idleNow then
			A.nextFidget = math.max(A.nextFidget, now + 4)
		end

		if seated then
			poseSeated(T)
			label = "Sentado"
		elseif climbing then
			poseClimb(A, T, vy, dt)
			label = "Escalando"
		elseif airLike then
			poseAir(A, T, vy)
			label = vy > 1 and "Pulando" or "Caindo"
			expr = vy < -22 and "Surprised" or "Happy"
		else
			plantTarget = 1
			poseGround(A, T, speed, A.yawRate, dt)
			if speed > 12 then label = "Correndo"
			elseif speed > 0.8 then label = "Caminhando" end
			if A.landT > 0 then
				overlayLanding(T, A)
				label = "Pouso"
			end
			if A.skidT > 0 then
				overlaySkid(T, A)
				label = "Derrapando"
			end
			if A.fidget then
				overlayFidget(T, A.fidget)
				label = "Idle: " .. A.fidget.kind
			end
		end

		if A.flipT then overlayFlip(T, A.flipT) end
		if act then
			if act.kind == "punch" then
				if act.variant == "kick" then
					overlayKick(T, act)
					label = "Chute"
				else
					overlayPunch(T, act)
					label = "Soco"
				end
				if act.t > 0.06 and act.t < 0.42 then expr = "Shout" end
			else
				overlayEmoteKind(T, act)
				label = "Gesto: " .. act.name
				expr = act.name == "Acenar" and "Happy" or "Laugh"
			end
		end
		if act or A.flipT or A.landT > 0 or A.skidT > 0 then fast = true end
	end

	if A.parkour then
		A.parkour.t += dt
		if A.parkour.kind ~= "hang" and A.parkour.t >= (A.parkour.dur or 0.7) then
			A.parkour = nil
		else
			overlayParkour(T, A.parkour)
			label = ({slide="Deslizando", vault="Vault", hang="Agarrado na borda", mantle="Subindo borda", walljump="Wall-jump", wallrun="Wall-run", roll="Rolamento"})[A.parkour.kind] or "Parkour"
			fast = true
		end
	end

	-- Suavização exponencial
	local alpha = 1 - exp(-(fast and 34 or 16) * dt)
	local C = A.cur
	for i = 1, #JOINT_NAMES do
		local jn = JOINT_NAMES[i]
		local c, t = C[jn], T[jn]
		c.rx += (t.rx - c.rx) * alpha
		c.ry += (t.ry - c.ry) * alpha
		c.rz += (t.rz - c.rz) * alpha
		c.tx += (t.tx - c.tx) * alpha
		c.ty += (t.ty - c.ty) * alpha
		c.tz += (t.tz - c.tz) * alpha
	end

	-- Pés plantados no chão: abaixa/levanta o quadril conforme o ângulo real das pernas
	local plant = (math.max(legDrop(C.HipL.rx, C.HipL.rz), legDrop(C.HipR.rx, C.HipR.rz)) - 10.5) * K
	A.plantW += (plantTarget - A.plantW) * (1 - exp(-12 * dt))

	local rc = C.Root
	rig.Drive("Root", CFrame.new(rc.tx, rc.ty + plant * A.plantW, rc.tz) * ROOT_PIVOT
		* CFrame.Angles(rc.rx + A.flipAngle, rc.ry, rc.rz) * ROOT_PIVOT_INV)
	for i = 2, #JOINT_NAMES do
		local jn = JOINT_NAMES[i]
		local c = C[jn]
		rig.Drive(jn, CFrame.new(c.tx, c.ty, c.tz) * CFrame.Angles(c.rx, c.ry, c.rz))
	end

	-- Rosto
	if not dead then
		if A.blinkUntil > 0 then
			if now >= A.blinkUntil then
				A.blinkUntil = 0
				rig.SetBlink(false)
				A.nextBlink = now + 2 + math.random() * 3
			end
		elseif now >= A.nextBlink and expr == "Happy" then
			A.blinkUntil = now + 0.12
			rig.SetBlink(true)
		end
	end
	if not A.deadHidden then rig.SetExpression(expr) end
	setAnimLabel(A, label .. " [" .. tostring(rig.DriveMode) .. "]")
end

----------------------------------------------------------------------
-- Ações do jogador
----------------------------------------------------------------------
local function startPunch()
	if State.InVehicle then return end
	local rig = State.Rig
	if not (rig and rig.Active) then return end
	local A = rig.Anim
	local now = os.clock()
	if A.deadT > 0 or now - A.lastPunch < CONFIG.PunchCooldown then return end
	if now - A.lastPunch > 1.0 then A.combo = 0 end
	A.lastPunch = now
	A.combo += 1
	if A.combo >= 3 then -- 3º golpe do combo: chute
		A.combo = 0
		A.action = { kind = "punch", variant = "kick", t = 0, dur = 0.62, side = 1, strikeAt = 0.18, fired = false }
	else
		A.punchSide = -A.punchSide -- alterna direita/esquerda
		A.action = { kind = "punch", variant = "punch", t = 0, dur = 0.55, side = A.punchSide, strikeAt = 0.12, fired = false }
	end
end

local function startEmote()
	if State.InVehicle then return end
	local rig = State.Rig
	if not (rig and rig.Active) then return end
	local A = rig.Anim
	if A.deadT > 0 then return end
	State.EmoteIdx = State.EmoteIdx % #EMOTES + 1
	local name = EMOTES[State.EmoteIdx]
	A.action = { kind = "emote", name = name, t = 0, dur = name == "Dançar" and 3.2 or 2.0 }
end

local function onJumpRequest()
	if State.InVehicle then return end
	if not CONFIG.EnableDoubleJump then return end
	local rig = State.Rig
	local now = os.clock()
	if not (rig and rig.Active) then return end
	local A = rig.Anim
	local gap = now - A.prevJumpReq
	A.prevJumpReq = now
	if gap < 0.18 then return end -- exige um novo toque (ignora o botão mantido)
	if Parkour.mode == "Hang" then beginMantle(); return end
	if tryWallJump() then return end
	local hum, hrp = State.Humanoid, State.Root
	if not (hum and hrp and hum.Health > 0) then return end
	local jumpState = hum:GetState()
	if jumpState ~= ST.Freefall and jumpState ~= ST.Jumping then return end
	if A.doubleUsed or A.airTime < CONFIG.DoubleJumpMinAirTime then return end
	A.doubleUsed = true
	local jv
	if hum.UseJumpPower then jv = hum.JumpPower else jv = math.sqrt(2 * Workspace.Gravity * hum.JumpHeight) end
	local v = hrp.AssemblyLinearVelocity
	hrp.AssemblyLinearVelocity = Vector3.new(v.X, jv * 0.95, v.Z)
	A.flipT = 0
end

----------------------------------------------------------------------
-- Ativação / desativação
----------------------------------------------------------------------
local function refreshToggle()
	UI.setToggleText(State.Enabled and "Transformar: ON" or "Transformar: OFF")
end
local function refreshOutfit()
	UI.setOutfitText("Traje: " .. OUTFITS[State.OutfitIndex].Name)
end

local function deactivateRig(restore)
	local rig = State.Rig
	State.Rig = nil
	State.RigBin:Clean()
	if rig then rig.Destroy() end
	if restore then restoreOriginal() else State.Hidden = nil end
	State.ToolHeld = false
	UI.setActionsVisible(false)
end

local function sweepStrayRigs(character)
	for _, c in ipairs(character:GetChildren()) do
		if c.Name == "LegoMinifigRig" and not (State.Rig and State.Rig.Model == c) then
			c:Destroy()
		end
	end
end

local function failBuild(step, msg, rig, replacing, character)
	warn(TAG .. " Falha ao construir [" .. step .. "]: " .. tostring(msg))
	local full = (not replacing) or (rig ~= nil and State.Rig == rig)
	if rig then rig.Destroy() end
	if full then deactivateRig(true) end
	if character then sweepStrayRigs(character) end
	UI.setBuild("Falha ao construir: " .. tostring(msg), C_ERR)
	UI.setError(step, msg)
	UI.setAnim("—", C_NEUTRAL)
end

local function waitForParts(character, token)
	local deadline = os.clock() + CONFIG.WaitTimeout
	local function remaining() return math.max(0.1, deadline - os.clock()) end
	local hum = character:WaitForChild("Humanoid", remaining())
	if not hum then error("Humanoid não apareceu em " .. CONFIG.WaitTimeout .. "s", 0) end
	local hrp = character:WaitForChild("HumanoidRootPart", remaining())
	if not hrp then error("HumanoidRootPart não apareceu em " .. CONFIG.WaitTimeout .. "s", 0) end
	if hum.RigType ~= Enum.HumanoidRigType.R15 then
		error("Este sistema exige R15 (RigType atual: " .. tostring(hum.RigType) .. ")", 0)
	end
	for _, name in ipairs({ "LowerTorso", "UpperTorso", "Head" }) do
		if not character:WaitForChild(name, remaining()) then error("Parte R15 ausente: " .. name, 0) end
	end
	while not player:HasAppearanceLoaded() and os.clock() < deadline do
		if token ~= State.Token then return nil end
		task.wait(0.1)
	end
	if not player:HasAppearanceLoaded() then
		warn(TAG .. " aparência não terminou de carregar em " .. CONFIG.WaitTimeout .. "s; continuando mesmo assim")
	end
	return { hum = hum, hrp = hrp }
end

local function installRig(rig, character)
	local old = State.Rig
	if not old then
		hideOriginal(character)
		State.ToolHeld = character:FindFirstChildOfClass("Tool") ~= nil
		State.RigBin:Add(character.ChildAdded:Connect(function(c)
			if c:IsA("Tool") then State.ToolHeld = true end
		end))
		State.RigBin:Add(character.ChildRemoved:Connect(function(c)
			if c:IsA("Tool") then State.ToolHeld = character:FindFirstChildOfClass("Tool") ~= nil end
		end))
		local errHandler = function(e)
			State.LastTrace = debug.traceback(tostring(e), 2)
			return tostring(e)
		end
		State.RigBin:Add(RunService.RenderStepped:Connect(function(dt)
			local ok, err = xpcall(updateRig, errHandler, dt)
			if not ok then State.OnAnimError(err) end
		end))
	end
	rig.Active = true
	State.Rig = rig
	if old and old ~= rig then old.Destroy() end
end

-- Move a perna esquerda 40° e confere se a peça realmente se moveu. Tenta C0 e depois Transform.
local function testJointDrive(rig)
	local leg = rig.Joints.HipL.Part1
	local restRel = rig.Rest[rig.Hips]:Inverse() * rig.Rest[leg]
	local test = CFrame.Angles(rad(40), 0, 0)
	for _, mode in ipairs({ "C0", "Transform" }) do
		rig.DriveMode = mode
		rig.Drive("HipL", test)
		RunService.Heartbeat:Wait()
		RunService.Heartbeat:Wait()
		local rel = rig.Hips.CFrame:ToObjectSpace(leg.CFrame)
		local moved = rel.UpVector:Dot(restRel.UpVector) < 0.95
		rig.Drive("HipL", CFrame.new())
		if moved then
			RunService.Heartbeat:Wait()
			return
		end
	end
	rig.DriveMode = nil
	error("As juntas não responderam à animação (nem por C0 nem por Transform)", 0)
end

local function finalCheck(rig, character, hum)
	if not rig.Model.Parent then error("Modelo da minifigura sumiu após instalar", 0) end
	if State.Rig ~= rig or not rig.Active then error("Rig não ficou registrado como ativo", 0) end
	if hum.Parent ~= character or hum.Health <= 0 then error("Humanoid inválido após instalar", 0) end
	local count = 0
	for _ in pairs(State.Hidden or {}) do count += 1 end
	if count == 0 then error("Nenhuma peça original foi ocultada", 0) end
end

local function runPipeline(token, replacing)
	local character = player.Character
	if not character then
		UI.setInit("Aguardando personagem", C_WARN)
		return
	end
	UI.setError(nil)

	if not replacing then
		deactivateRig(true)
		sweepStrayRigs(character)
		UI.setInit("Aguardando personagem", C_WARN)
		UI.setBuild("—", C_NEUTRAL)
		UI.setAnim("—", C_NEUTRAL)
		local ok, res = safeCall("Aguardando personagem", waitForParts, character, token)
		if token ~= State.Token then return end
		if not ok then return failBuild("Aguardando personagem", res, nil, false, character) end
		if not res then return end
		State.Character, State.Humanoid, State.Root = character, res.hum, res.hrp
		UI.setInit("Personagem detectado", C_OK)
	end

	local hum, hrp = State.Humanoid, State.Root
	if State.Character ~= character or not hum or not hrp then
		return failBuild("Aguardando personagem", "personagem inválido", nil, replacing, character)
	end

	UI.setBuild("Criando peças", C_WARN)
	local ok, rig = safeCall("Criando peças", buildRig, character, hum, hrp, OUTFITS[State.OutfitIndex])
	if not ok then return failBuild("Criando peças", rig, nil, replacing, character) end

	UI.setBuild("Validando juntas", C_WARN)
	local ok2, err2 = safeCall("Validando juntas", validateRigStatic, rig, character, hrp)
	if not ok2 then return failBuild("Validando juntas", err2, rig, replacing, character) end

	RunService.Heartbeat:Wait()
	RunService.Heartbeat:Wait()
	if token ~= State.Token then rig.Destroy() return end

	UI.setBuild("Aplicando transformação", C_WARN)
	local ok3, err3 = safeCall("Aplicando transformação", verifyRigPhysics, rig, hrp, hum)
	if not ok3 then return failBuild("Aplicando transformação", err3, rig, replacing, character) end

	UI.setBuild("Testando animação das juntas", C_WARN)
	local okT, errT = safeCall("Testando animação das juntas", testJointDrive, rig)
	if not okT then return failBuild("Testando animação das juntas", errT, rig, replacing, character) end
	if token ~= State.Token then rig.Destroy() return end

	local ok4, err4 = safeCall("Instalando rig", installRig, rig, character)
	if not ok4 then return failBuild("Instalando rig", err4, rig, replacing, character) end

	local ok5, err5 = safeCall("Confirmação final", finalCheck, rig, character, hum)
	if not ok5 then return failBuild("Confirmação final", err5, rig, replacing, character) end

	UI.setInit("Personagem OK", C_OK)
	UI.setBuild("Minifigura LEGO ativa (" .. OUTFITS[State.OutfitIndex].Name .. ", " .. #rig.Parts .. " peças)", C_OK)
	UI.setActionsVisible(true)
end

State.OnAnimError = function(msg)
	task.defer(function()
		resetParkour(true)
		clearDestruction()
		warn(TAG .. " Falha na animação: " .. tostring(msg))
		if State.LastTrace then warn(State.LastTrace) end
		State.Enabled = false
		State.Token += 1
		deactivateRig(true)
		refreshToggle()
		UI.setBuild("Desativada por erro de animação (avatar restaurado)", C_ERR)
		UI.setError("Animação", msg)
		UI.setAnim("Falha: " .. tostring(msg), C_ERR)
	end)
end

----------------------------------------------------------------------
-- LEGO TOYBOX V7 — 41 BRINQUEDOS LOCAIS, MENU TOUCH E AÇÕES VISUAIS
-- Tudo deste módulo é client-side: não dá ferramentas ao servidor,
-- não causa dano, não muda a velocidade do Humanoid e não modifica o mapa.
----------------------------------------------------------------------
local ToyKit = {
	CurrentId = nil,
	CurrentModel = nil,
	FXFolder = nil,
	Gui = nil,
	ActiveProjectiles = 0,
	MaxProjectiles = 12,
	Status = "Escolha um brinquedo no menu.",
	Category = "TODOS",
	Search = "",
	Connections = {},
}

local TOY_ITEMS = {
	{Id="laser", Name="Pistola laser LEGO", Icon="🔫", Cat="FERRAMENTAS", Mount="hand", Action="laser", Color=Color3.fromRGB(65,220,255), Desc="Dispara um raio de luz."},
	{Id="saber", Name="Sabre de luz", Icon="⚔️", Cat="FERRAMENTAS", Mount="hand", Action="saber", Color=Color3.fromRGB(60,220,255), Desc="Liga a lâmina, faz cortes luminosos e pode ser guardado para desligar."},
	{Id="football", Name="Bola de futebol", Icon="⚽", Cat="ESPORTES", Mount="hand", Action="ball", Color=Color3.fromRGB(245,245,245), Desc="Arremesse e veja a bola quicar localmente."},
	{Id="basketball", Name="Bola de basquete", Icon="🏀", Cat="ESPORTES", Mount="hand", Action="ball", Color=Color3.fromRGB(225,112,43), Desc="Lance a bola com gravidade e quique visual."},
	{Id="tennis", Name="Raquete de tênis", Icon="🎾", Cat="ESPORTES", Mount="hand", Action="tennis", Color=Color3.fromRGB(190,235,55), Desc="Bate numa bolinha de tênis luminosa."},
	{Id="mini_car", Name="Carro esportivo pilotável", Icon="🏎️", Cat="VEÍCULOS", Mount="hand", Action="vehicle", Color=Color3.fromRGB(220,55,45), Desc="Invoca um carro em escala de personagem, com aceleração, direção, freio, suspensão visual, rodas rotativas e controles touch."},
	{Id="mini_bike", Name="Motocicleta pilotável", Icon="🏍️", Cat="VEÍCULOS", Mount="hand", Action="vehicle", Color=Color3.fromRGB(45,120,235), Desc="Invoca uma motocicleta dirigível com direção, aceleração, freio, inclinação, rodas rotativas e controles touch."},
	{Id="skateboard", Name="Skate LEGO", Icon="🛹", Cat="VEÍCULOS", Mount="feet", Action="trick", Color=Color3.fromRGB(245,155,40), Desc="Skate visual com faíscas de manobra, sem alterar movimento."},
	{Id="boomerang", Name="Bumerangue", Icon="🪃", Cat="FERRAMENTAS", Mount="hand", Action="boomerang", Color=Color3.fromRGB(245,180,45), Desc="Voa para a frente e retorna visualmente."},
	{Id="grapple", Name="Lançador de cabo", Icon="🪝", Cat="FERRAMENTAS", Mount="hand", Action="grapple", Color=Color3.fromRGB(175,180,190), Desc="Lança um cabo visual até a superfície apontada; não puxa o jogador."},
	{Id="paint", Name="Lançador de tinta", Icon="🎨", Cat="FERRAMENTAS", Mount="hand", Action="paint", Color=Color3.fromRGB(245,55,155), Desc="Lança uma esfera de tinta e um respingo local."},
	{Id="hammer", Name="Martelo de espuma", Icon="🔨", Cat="FERRAMENTAS", Mount="hand", Action="swing", Color=Color3.fromRGB(255,90,90), Desc="Golpe visual cartunesco com estrelas de impacto."},
	{Id="wand", Name="Varinha mágica", Icon="🪄", Cat="CRIATIVO", Mount="hand", Action="magic", Color=Color3.fromRGB(195,95,255), Desc="Lança um orbe mágico e partículas brilhantes."},
	{Id="camera", Name="Câmera instantânea", Icon="📷", Cat="CRIATIVO", Mount="hand", Action="flash", Color=Color3.fromRGB(245,245,245), Desc="Dispara um flash visual e estrelinhas."},
	{Id="boombox", Name="Rádio LEGO", Icon="📻", Cat="CRIATIVO", Mount="hand", Action="pulse", Color=Color3.fromRGB(70,220,175), Desc="Pulsa luzes e notas visuais; não depende de áudio externo."},
	{Id="guitar", Name="Guitarra", Icon="🎸", Cat="CRIATIVO", Mount="hand", Action="music", Color=Color3.fromRGB(220,65,70), Desc="Toca uma animação visual com notas flutuantes."},
	{Id="yoyo", Name="Yo-yo", Icon="🪀", Cat="ESPORTES", Mount="hand", Action="yoyo", Color=Color3.fromRGB(80,185,255), Desc="O disco sai por um fio visual e volta."},
	{Id="water", Name="Lançador de água", Icon="💦", Cat="FERRAMENTAS", Mount="hand", Action="water", Color=Color3.fromRGB(50,165,255), Desc="Dispara gotículas azuis locais."},
	{Id="freeze", Name="Raio congelante", Icon="❄️", Cat="FERRAMENTAS", Mount="hand", Action="freeze", Color=Color3.fromRGB(130,235,255), Desc="Lança cristais de gelo visuais."},
	{Id="firework", Name="Lança-fogos", Icon="🎆", Cat="CRIATIVO", Mount="hand", Action="firework", Color=Color3.fromRGB(255,80,180), Desc="Lança um foguete visual que explode em luzes."},
	{Id="confetti", Name="Canhão de confete", Icon="🎉", Cat="CRIATIVO", Mount="hand", Action="confetti", Color=Color3.fromRGB(255,210,50), Desc="Explosão de confetes coloridos sem dano."},
	{Id="slingshot", Name="Estilingue", Icon="🪨", Cat="FERRAMENTAS", Mount="hand", Action="slingshot", Color=Color3.fromRGB(160,115,75), Desc="Arremessa uma bolinha cartunesca."},
	{Id="shield", Name="Escudo de energia", Icon="🛡️", Cat="ACESSÓRIOS", Mount="hand", Action="shield", Color=Color3.fromRGB(70,150,255), Desc="Cria uma bolha protetora puramente visual."},
	{Id="jetpack", Name="Mini jetpack", Icon="🚀", Cat="ACESSÓRIOS", Mount="back", Action="jetpack", Color=Color3.fromRGB(255,120,50), Desc="Jatos decorativos; não dá impulso nem altera a velocidade."},
	{Id="wings", Name="Asas LEGO", Icon="🪽", Cat="ACESSÓRIOS", Mount="back", Action="wings", Color=Color3.fromRGB(175,205,255), Desc="Asas decorativas com pequenas partículas."},
	{Id="fishing", Name="Vara de pesca", Icon="🎣", Cat="FERRAMENTAS", Mount="hand", Action="fishing", Color=Color3.fromRGB(90,175,100), Desc="Lança linha e boia visual; não exige água real."},
	{Id="magnet", Name="Superímã", Icon="🧲", Cat="FERRAMENTAS", Mount="hand", Action="magnet", Color=Color3.fromRGB(235,65,70), Desc="Atrai orbes de luz até o jogador, sem mexer em objetos reais."},
	{Id="rocket", Name="Foguete de brinquedo", Icon="🛰️", Cat="CRIATIVO", Mount="hand", Action="rocket", Color=Color3.fromRGB(245,245,245), Desc="Foguetinho visual que sobe e vira uma chuva de estrelas."},
	{Id="potion", Name="Poção colorida", Icon="🧪", Cat="CRIATIVO", Mount="hand", Action="potion", Color=Color3.fromRGB(65,230,135), Desc="Arremessa um frasco de luz que solta bolhas coloridas."},
	{Id="bubbles", Name="Soprador de bolhas", Icon="🫧", Cat="CRIATIVO", Mount="hand", Action="bubbles", Color=Color3.fromRGB(120,230,255), Desc="Cria bolhas que sobem e desaparecem."},
	{Id="lantern", Name="Lanterna LEGO", Icon="🔦", Cat="ACESSÓRIOS", Mount="hand", Action="lantern", Color=Color3.fromRGB(255,225,125), Desc="Liga/desliga luz local no acessório."},
	{Id="telescope", Name="Mini telescópio", Icon="🔭", Cat="CRIATIVO", Mount="hand", Action="telescope", Color=Color3.fromRGB(90,145,210), Desc="Marca visualmente o ponto observado sem alterar a câmera."},
	{Id="drum", Name="Baquetas", Icon="🥁", Cat="CRIATIVO", Mount="hand", Action="drum", Color=Color3.fromRGB(230,175,95), Desc="Batida visual com ondas coloridas."},
	{Id="pickaxe", Name="Picareta de brinquedo", Icon="⛏️", Cat="FERRAMENTAS", Mount="hand", Action="swing", Color=Color3.fromRGB(155,180,195), Desc="Faíscas cartunescas no ponto mirado; não minera o mapa."},
	{Id="hoverboard", Name="Hoverboard LEGO", Icon="🛹", Cat="VEÍCULOS", Mount="feet", Action="trick", Color=Color3.fromRGB(65,220,205), Desc="Prancha decorativa com trilha de luz; sem impulso."},
	{Id="flag", Name="Bandeira de corrida", Icon="🏁", Cat="ACESSÓRIOS", Mount="hand", Action="flag", Color=Color3.fromRGB(240,240,240), Desc="Acena e lança confetes de corrida."},
	{Id="drone", Name="Mini drone", Icon="🚁", Cat="VEÍCULOS", Mount="hand", Action="drone", Color=Color3.fromRGB(120,210,245), Desc="Orbita alguns segundos à frente e retorna; puramente visual."},
	{Id="spring", Name="Botas mola", Icon="🌀", Cat="ACESSÓRIOS", Mount="feet", Action="trick", Color=Color3.fromRGB(240,120,55), Desc="Molas decorativas e faíscas; não muda salto nem velocidade."},
	{Id="candy", Name="Doce gigante", Icon="🍭", Cat="CRIATIVO", Mount="hand", Action="candy", Color=Color3.fromRGB(255,95,170), Desc="Arremessa um doce de brinquedo que solta estrelas."},
	{Id="portal", Name="Disco portal", Icon="🌀", Cat="CRIATIVO", Mount="hand", Action="portal", Color=Color3.fromRGB(150,70,255), Desc="Cria dois anéis decorativos; não teletransporta ninguém."},
	{Id="horn", Name="Buzina LEGO", Icon="📣", Cat="CRIATIVO", Mount="hand", Action="pulse", Color=Color3.fromRGB(255,190,60), Desc="Pulso cônico de luz sem áudio externo."},
}

local TOY_PARTS = {
	laser={{"Grip",Vector3.new(.20,.22,.35),CFrame.new(0,-.08,.03),Color3.fromRGB(38,43,54),"Block"},{"Body",Vector3.new(.31,.20,.48),CFrame.new(0,.04,-.17),Color3.fromRGB(66,77,94),"Block"},{"Barrel",Vector3.new(.13,.13,.43),CFrame.new(0,.04,-.54),Color3.fromRGB(25,29,36),"Block"},{"Emitter",Vector3.new(.16,.16,.08),CFrame.new(0,.04,-.79),Color3.fromRGB(65,230,255),"Ball"}},
	 saber={{"Hilt",Vector3.new(.16,.18,.43),CFrame.new(0,-.07,.02),Color3.fromRGB(175,180,190),"Cylinder"},{"Ring",Vector3.new(.23,.08,.08),CFrame.new(0,-.07,-.12),Color3.fromRGB(245,185,60),"Cylinder"},{"ToyBlade",Vector3.new(.11,.11,1.2),CFrame.new(0,-.02,-.77),Color3.fromRGB(70,220,255),"Block"}},
	 tennis={{"RacketGrip",Vector3.new(.12,.30,.10),CFrame.new(0,-.30,0),Color3.fromRGB(45,48,60),"Block"},{"RacketNeck",Vector3.new(.16,.18,.09),CFrame.new(0,-.10,0),Color3.fromRGB(80,90,105),"Block"}},
	 mini_car={{"Chassis",Vector3.new(.76,.20,.95),CFrame.new(0,0,0),Color3.fromRGB(210,45,42),"Block"},{"Cabin",Vector3.new(.47,.27,.39),CFrame.new(0,.20,-.05),Color3.fromRGB(60,180,240),"Block"},{"WheelL1",Vector3.new(.18,.18,.12),CFrame.new(-.36,-.09,-.27)*CFrame.Angles(0,0,math.rad(90)),Color3.fromRGB(25,25,30),"Cylinder"},{"WheelR1",Vector3.new(.18,.18,.12),CFrame.new(.36,-.09,-.27)*CFrame.Angles(0,0,math.rad(90)),Color3.fromRGB(25,25,30),"Cylinder"},{"WheelL2",Vector3.new(.18,.18,.12),CFrame.new(-.36,-.09,.30)*CFrame.Angles(0,0,math.rad(90)),Color3.fromRGB(25,25,30),"Cylinder"},{"WheelR2",Vector3.new(.18,.18,.12),CFrame.new(.36,-.09,.30)*CFrame.Angles(0,0,math.rad(90)),Color3.fromRGB(25,25,30),"Cylinder"}},
	 mini_bike={{"Frame",Vector3.new(.18,.15,.74),CFrame.new(0,0,0),Color3.fromRGB(45,120,235),"Block"},{"Seat",Vector3.new(.25,.08,.24),CFrame.new(0,.14,.13),Color3.fromRGB(30,30,35),"Block"},{"FrontWheel",Vector3.new(.13,.30,.30),CFrame.new(0,-.12,-.35),Color3.fromRGB(28,28,33),"Cylinder"},{"RearWheel",Vector3.new(.13,.30,.30),CFrame.new(0,-.12,.34),Color3.fromRGB(28,28,33),"Cylinder"},{"Handlebar",Vector3.new(.40,.06,.06),CFrame.new(0,.20,-.31),Color3.fromRGB(180,185,195),"Block"}},
	 skateboard={{"Deck",Vector3.new(1.1,.10,.36),CFrame.new(0,0,0),Color3.fromRGB(245,145,40),"Block"},{"Grip",Vector3.new(.90,.02,.28),CFrame.new(0,.06,0),Color3.fromRGB(45,45,55),"Block"},{"WheelL",Vector3.new(.15,.14,.14),CFrame.new(-.33,-.08,.12),Color3.fromRGB(30,30,35),"Cylinder"},{"WheelR",Vector3.new(.15,.14,.14),CFrame.new(.33,-.08,.12),Color3.fromRGB(30,30,35),"Cylinder"}},
	 hoverboard={{"Board",Vector3.new(.90,.12,.42),CFrame.new(0,0,0),Color3.fromRGB(35,70,90),"Block"},{"Glow",Vector3.new(.72,.05,.30),CFrame.new(0,-.07,0),Color3.fromRGB(65,230,210),"Block"},{"EngineL",Vector3.new(.18,.14,.18),CFrame.new(-.31,-.12,0),Color3.fromRGB(60,175,255),"Ball"},{"EngineR",Vector3.new(.18,.14,.18),CFrame.new(.31,-.12,0),Color3.fromRGB(60,175,255),"Ball"}},
	 jetpack={{"TankL",Vector3.new(.20,.48,.20),CFrame.new(-.17,0,.05),Color3.fromRGB(225,80,48),"Cylinder"},{"TankR",Vector3.new(.20,.48,.20),CFrame.new(.17,0,.05),Color3.fromRGB(225,80,48),"Cylinder"},{"Pack",Vector3.new(.42,.38,.17),CFrame.new(0,0,.14),Color3.fromRGB(62,67,78),"Block"},{"NozzleL",Vector3.new(.12,.22,.12),CFrame.new(-.15,-.24,.08),Color3.fromRGB(255,185,70),"Cylinder"},{"NozzleR",Vector3.new(.12,.22,.12),CFrame.new(.15,-.24,.08),Color3.fromRGB(255,185,70),"Cylinder"}},
	 wings={{"WingL",Vector3.new(.50,.65,.12),CFrame.new(-.30,0,.12)*CFrame.Angles(0,0,math.rad(25)),Color3.fromRGB(175,205,255),"Block"},{"WingR",Vector3.new(.50,.65,.12),CFrame.new(.30,0,.12)*CFrame.Angles(0,0,math.rad(-25)),Color3.fromRGB(175,205,255),"Block"},{"Center",Vector3.new(.22,.35,.14),CFrame.new(0,0,.05),Color3.fromRGB(245,230,175),"Block"}},
	 guitar={{"Body",Vector3.new(.50,.55,.13),CFrame.new(0,-.06,0),Color3.fromRGB(200,55,65),"Ball"},{"Neck",Vector3.new(.13,.70,.12),CFrame.new(0,.47,-.06)*CFrame.Angles(math.rad(-12),0,0),Color3.fromRGB(120,78,48),"Block"},{"Head",Vector3.new(.22,.20,.13),CFrame.new(0,.85,-.06),Color3.fromRGB(220,185,115),"Block"}},
	 fishing={{"Rod",Vector3.new(.10,.10,.95),CFrame.new(0,.15,-.26)*CFrame.Angles(math.rad(-18),0,0),Color3.fromRGB(83,135,85),"Block"},{"Reel",Vector3.new(.22,.22,.16),CFrame.new(0,-.05,.05),Color3.fromRGB(185,190,195),"Cylinder"}},
	 flag={{"Pole",Vector3.new(.07,.80,.07),CFrame.new(0,.36,0),Color3.fromRGB(195,195,205),"Block"},{"Cloth",Vector3.new(.50,.31,.035),CFrame.new(.23,.55,0),Color3.fromRGB(245,245,245),"Block"},{"Stripe",Vector3.new(.50,.08,.045),CFrame.new(.23,.55,-.025),Color3.fromRGB(230,55,55),"Block"}},
	 shield={{"Rim",Vector3.new(.72,.78,.13),CFrame.new(0,0,0),Color3.fromRGB(70,150,255),"Cylinder"},{"Core",Vector3.new(.49,.54,.08),CFrame.new(0,0,-.06),Color3.fromRGB(125,215,255),"Cylinder"}},
}

local ITEM_BY_ID = {}
for _, def in ipairs(TOY_ITEMS) do ITEM_BY_ID[def.Id] = def end

local function toyMake(className, properties, parent)
	local obj = Instance.new(className)
	for key, value in pairs(properties) do obj[key] = value end
	obj.Parent = parent
	return obj
end

local function ensureToyFX()
	if ToyKit.FXFolder and ToyKit.FXFolder.Parent then return ToyKit.FXFolder end
	local f = Instance.new("Folder")
	f.Name = "LegoToyFX_Local"
	f.Parent = Workspace
	ToyKit.FXFolder = f
	return f
end

local function getToyAim(maxDistance)
	local camera = Workspace.CurrentCamera
	if not camera then return Vector3.new(0, 0, -1), Vector3.new(0, 0, -70), nil end
	local origin = camera.CFrame.Position
	local direction = camera.CFrame.LookVector * (maxDistance or 75)
	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	local ignore = {State.Character, ToyKit.FXFolder, ToyKit.CurrentModel}
	params.FilterDescendantsInstances = ignore
	params.IgnoreWater = false
	local result = Workspace:Raycast(origin, direction, params)
	return camera.CFrame.LookVector, result and result.Position or (origin + direction), result
end

local function toyPart(parent, name, size, cf, color, shape, material, anchored, transparency)
	-- Limite de FX locais para evitar crescimento ilimitado de peças no celular.
	if ToyKit.FXFolder and parent == ToyKit.FXFolder then
		local children = parent:GetChildren()
		if #children >= 720 then
			-- Mantém modelos persistentes (veículos) e limpa primeiro os efeitos temporários mais antigos.
			local removed = 0
			for _, child in ipairs(children) do
				if removed >= 90 then break end
				if child:IsA("BasePart") and (string.find(child.Name, "Toy", 1, true) or string.find(child.Name, "Action", 1, true) or string.find(child.Name, "Spark", 1, true)) then
					child:Destroy()
					removed += 1
				end
			end
		end
	end
	local p = Instance.new("Part")
	p.Name = name
	p.Size = size
	p.CFrame = cf
	p.Color = color
	p.Material = material or Enum.Material.SmoothPlastic
	p.Anchored = anchored == true
	p.CanCollide = false
	p.CanTouch = false
	p.CanQuery = false
	p.Massless = true
	p.TopSurface = Enum.SurfaceType.Smooth
	p.BottomSurface = Enum.SurfaceType.Smooth
	p.Transparency = transparency or 0
	if shape == "Ball" or shape == "Cylinder" then p.Shape = Enum.PartType[shape] end
	p.Parent = parent
	return p
end

---------------------------------------------------------------------
-- V8 MODEL DETAIL FACTORY: high-detail hard-surface LEGO-like props
-- The equipped prop is the only full detail model kept on the avatar.
---------------------------------------------------------------------
local function addToyDetailPack(def, model, basis)
	local detailCount = 0
	local rootColor = def.Color or Color3.new(1,1,1)
	local dark = Color3.fromRGB(28,32,42)
	local metal = Color3.fromRGB(155,165,180)
	local glow = rootColor:Lerp(Color3.new(1,1,1), .28)
	local function part(name, size, x, y, z, color, material, shape, rx, ry, rz, transparency)
		detailCount += 1
		local cf = basis * CFrame.new(x or 0, y or 0, z or 0) * CFrame.Angles(rx or 0, ry or 0, rz or 0)
		return toyPart(model, "V8Detail_" .. name .. "_" .. detailCount, size, cf, color or rootColor, shape or "Block", material or Enum.Material.SmoothPlastic, false, transparency or 0)
	end
	local function ribs(axis, count, startValue, step, size, color, y, z)
		for i = 1, count do
			local p = startValue + (i-1)*step
			if axis == "x" then part("Rib", size, p, y, z, color, Enum.Material.Metal) else part("Rib", size, 0, p, z, color, Enum.Material.Metal) end
		end
	end
	local function ring(name, z, radius, thickness, color, material)
		for i = 1, 10 do
			local a = math.pi*2*i/10
			part(name, Vector3.new(thickness, thickness, thickness*1.7), math.cos(a)*radius, math.sin(a)*radius, z, color, material or Enum.Material.Metal, "Block", 0, 0, a)
		end
	end

	if def.Id == "laser" then
		part("PowerCell", Vector3.new(.20,.16,.24), .19,.02,.02, Color3.fromRGB(48,58,72), Enum.Material.Metal)
		part("PowerCellGlow", Vector3.new(.16,.045,.12), .20,.105,.02, glow, Enum.Material.Neon)
		part("TopRail", Vector3.new(.09,.045,.54), 0,.155,-.22, metal, Enum.Material.Metal)
		part("SightBase", Vector3.new(.16,.07,.10), 0,.19,-.34,dark,Enum.Material.Metal)
		part("SightDot", Vector3.new(.055,.055,.055), 0,.235,-.36,glow,Enum.Material.Neon,"Ball")
		for i=1,6 do part("BarrelRing",Vector3.new(.19,.19,.032),0,.04,-.36-i*.075,metal,Enum.Material.Metal,"Cylinder") end
		for i=1,7 do part("GripGroove",Vector3.new(.23,.025,.035),0,-.105+i*.031,.11,dark,Enum.Material.Metal) end
		for side=-1,1,2 do
			part("SideRail",Vector3.new(.045,.07,.38),side*.18,.055,-.20,metal,Enum.Material.Metal)
			for i=1,4 do part("Vent",Vector3.new(.02,.025,.045),side*.205,.055,-.31+i*.075,glow,Enum.Material.Neon) end
			part("SideScrew",Vector3.new(.045,.045,.045),side*.205,-.005,-.05,metal,Enum.Material.Metal,"Ball")
		end
	elseif def.Id == "saber" then
		for i=1,8 do part("HiltGrip",Vector3.new(.18,.035,.035),0,-.16+i*.035,.08,dark,Enum.Material.Metal) end
		for i=1,5 do part("EmitterCoil",Vector3.new(.21,.035,.035),0,-.02,-.21-i*.035,glow,Enum.Material.Neon,"Cylinder") end
		part("ActivationButton",Vector3.new(.075,.055,.045),.10,.015,-.03,Color3.fromRGB(245,70,70),Enum.Material.Neon)
		part("Pommel",Vector3.new(.20,.20,.11),0,-.07,.23,metal,Enum.Material.Metal,"Cylinder")
		part("EmitterCrown",Vector3.new(.24,.24,.07),0,-.02,-.31,glow,Enum.Material.Neon,"Cylinder")
		for i=1,8 do local a=i*math.pi/4; part("EmitterBolt",Vector3.new(.035,.035,.035),math.cos(a)*.105,-.02+math.sin(a)*.105,-.35,metal,Enum.Material.Metal,"Ball") end
	elseif def.Id == "tennis" then
		-- Oval head rendered as a ring of short polymer beams plus tensioned strings.
		for i=1,18 do
			local a=2*math.pi*i/18
			part("RacketFrame",Vector3.new(.09,.07,.075),math.cos(a)*.29,.26+math.sin(a)*.34,0,Color3.fromRGB(75,135,75),Enum.Material.SmoothPlastic,"Block",0,0,-a)
		end
		for i=-3,3 do part("StringVertical",Vector3.new(.018,.51,.018),i*.055,.27,-.043,Color3.fromRGB(235,245,220),Enum.Material.SmoothPlastic) end
		for i=-4,4 do part("StringHorizontal",Vector3.new(.42,.018,.018),0,.27+i*.048,-.044,Color3.fromRGB(235,245,220),Enum.Material.SmoothPlastic) end
		part("GripBand",Vector3.new(.135,.10,.12),0,-.31,0,Color3.fromRGB(230,230,225),Enum.Material.Fabric)
	elseif def.Id == "football" or def.Id == "basketball" then
		local r = .205
		local seam = def.Id == "basketball" and Color3.fromRGB(45,35,30) or Color3.fromRGB(38,40,45)
		for i=1,8 do
			local a = i*math.pi/4
			local x,z = math.cos(a)*r*.70,math.sin(a)*r*.70
			part("BallPanel",Vector3.new(.072,.018,.046),x,.09,z,seam,Enum.Material.SmoothPlastic,"Block",0,-a,a*.12)
		end
		for _, sign in ipairs({-1,1}) do
			part("BallSeam",Vector3.new(.02,.36,.02),sign*.08,0,0,seam,Enum.Material.SmoothPlastic,"Block",0,0,sign*.32)
			part("BallSeam",Vector3.new(.36,.02,.02),0,sign*.08,0,seam,Enum.Material.SmoothPlastic)
		end
		for i=1,8 do local a=i*math.pi/4; part("GripDot",Vector3.new(.025,.025,.025),math.cos(a)*.16,math.sin(a)*.11,.13,seam,Enum.Material.SmoothPlastic,"Ball") end
	elseif def.Id == "camera" then
		part("LensBarrel",Vector3.new(.25,.25,.18),0,.015,-.20,metal,Enum.Material.Metal,"Cylinder")
		part("LensGlass",Vector3.new(.21,.21,.035),0,.015,-.31,Color3.fromRGB(45,120,205),Enum.Material.Glass,"Cylinder",0,0,0,.15)
		part("LensSpecular",Vector3.new(.055,.055,.025),-.045,.055,-.335,Color3.fromRGB(210,245,255),Enum.Material.Neon,"Ball")
		part("FlashBox",Vector3.new(.12,.075,.045),.13,.13,-.10,Color3.fromRGB(255,245,205),Enum.Material.Neon)
		part("Screen",Vector3.new(.18,.12,.025),0,.11,.17,Color3.fromRGB(55,95,115),Enum.Material.Glass)
		for i=1,3 do part("ScreenLine",Vector3.new(.13,.012,.012),0,.14-i*.025,.15,glow,Enum.Material.Neon) end
		for i=1,4 do part("GripRidge",Vector3.new(.025,.14,.025),-.19+i*.032,-.015,.03,dark,Enum.Material.Metal) end
		part("Shutter",Vector3.new(.07,.045,.05),-.12,.15,-.08,Color3.fromRGB(235,65,55),Enum.Material.Neon)
	elseif def.Id == "guitar" then
		for i=1,6 do part("GuitarString",Vector3.new(.012,.68,.012),-.075+i*.03,.33,-.085,Color3.fromRGB(225,220,195),Enum.Material.Metal) end
		for i=1,8 do part("Fret",Vector3.new(.15,.012,.018),0,.10+i*.072,-.095,metal,Enum.Material.Metal) end
		for i=1,4 do local a=i*math.pi/2; part("Knob",Vector3.new(.055,.055,.035),math.cos(a)*.13,-.08+math.sin(a)*.12,-.095,metal,Enum.Material.Metal,"Ball") end
		part("PickGuard",Vector3.new(.19,.20,.018),.08,-.07,-.09,Color3.fromRGB(245,205,110),Enum.Material.SmoothPlastic)
	elseif def.Id == "hammer" or def.Id == "pickaxe" then
		for i=1,9 do part("HandleWrap",Vector3.new(.14,.025,.15),0,-.28+i*.065,.01,dark,Enum.Material.Fabric) end
		part("HeadCore",Vector3.new(.52,.21,.25),0,.23,0,metal,Enum.Material.Metal)
		part("HeadCapL",Vector3.new(.12,.30,.30),-.28,.23,0,rootColor,Enum.Material.Metal)
		part("HeadCapR",Vector3.new(.12,.30,.30),.28,.23,0,rootColor,Enum.Material.Metal)
		for side=-1,1,2 do for i=1,3 do part("HeadBolt",Vector3.new(.045,.045,.045),side*.26,.18+i*.05,-.14,Color3.fromRGB(225,225,230),Enum.Material.Metal,"Ball") end end
	elseif def.Id == "paint" or def.Id == "freeze" or def.Id == "water" or def.Id == "slingshot" then
		part("PressureCanister",Vector3.new(.24,.36,.22),.18,.04,.04,Color3.fromRGB(55,65,80),Enum.Material.Metal,"Cylinder")
		part("CanisterLabel",Vector3.new(.25,.13,.235),.18,.045,-.075,rootColor,Enum.Material.Neon)
		part("GaugeRim",Vector3.new(.12,.12,.07),.05,.13,-.16,metal,Enum.Material.Metal,"Cylinder")
		part("GaugeFace",Vector3.new(.08,.08,.075),.05,.13,-.20,Color3.fromRGB(235,245,250),Enum.Material.SmoothPlastic,"Cylinder")
		part("NozzleOuter",Vector3.new(.20,.20,.15),0,.07,-.45,metal,Enum.Material.Metal,"Cylinder")
		part("NozzleCore",Vector3.new(.105,.105,.09),0,.07,-.54,glow,Enum.Material.Neon,"Cylinder")
		for i=1,7 do part("VentRib",Vector3.new(.24,.025,.035),.18,-.13+i*.034,.03,dark,Enum.Material.Metal) end
	elseif def.Id == "boombox" or def.Id == "horn" or def.Id == "drum" then
		for side=-1,1,2 do
			part("SpeakerOuter",Vector3.new(.22,.22,.065),side*.13,.02,-.16,dark,Enum.Material.Metal,"Cylinder")
			part("SpeakerCone",Vector3.new(.16,.16,.07),side*.13,.02,-.20,metal,Enum.Material.Metal,"Cylinder")
			part("SpeakerCore",Vector3.new(.055,.055,.075),side*.13,.02,-.24,rootColor,Enum.Material.Neon,"Ball")
		end
		for i=1,5 do part("TopKnob",Vector3.new(.04,.04,.04),-.15+i*.06,.15,-.04,metal,Enum.Material.Metal,"Cylinder") end
		part("Display",Vector3.new(.18,.075,.03),0,.10,.17,Color3.fromRGB(40,210,160),Enum.Material.Neon)
	elseif def.Id == "jetpack" then
		for side=-1,1,2 do
			for i=1,5 do part("FuelBand",Vector3.new(.22,.035,.22),side*.17,-.17+i*.075,.06,metal,Enum.Material.Metal,"Cylinder") end
			part("NozzleLip",Vector3.new(.16,.08,.16),side*.16,-.28,.08,Color3.fromRGB(255,170,55),Enum.Material.Neon,"Cylinder")
			part("WarningLight",Vector3.new(.055,.055,.055),side*.17,.22,-.05,glow,Enum.Material.Neon,"Ball")
		end
	elseif def.Id == "lantern" then
		for i=1,8 do local a=math.pi*2*i/8; part("CageBar",Vector3.new(.025,.30,.025),math.cos(a)*.11,0,math.sin(a)*.11,metal,Enum.Material.Metal) end
		part("TopCap",Vector3.new(.25,.06,.25),0,.18,0,metal,Enum.Material.Metal,"Cylinder")
		part("BottomCap",Vector3.new(.25,.06,.25),0,-.18,0,metal,Enum.Material.Metal,"Cylinder")
	elseif def.Id == "drone" then
		for x=-1,1,2 do for z=-1,1,2 do
			part("Arm",Vector3.new(.48,.06,.08),x*.21,.04,z*.18,metal,Enum.Material.Metal, "Block",0,x*z*math.rad(22),0)
			part("Rotor",Vector3.new(.38,.025,.07),x*.36,.09,z*.31,rootColor,Enum.Material.Neon,"Block",0,0,math.rad(18*x*z))
			part("Motor",Vector3.new(.12,.05,.12),x*.36,.07,z*.31,dark,Enum.Material.Metal,"Cylinder")
		end end
		part("CameraLens",Vector3.new(.12,.12,.07),0,-.08,-.08,Color3.fromRGB(65,200,255),Enum.Material.Glass,"Ball")
	elseif def.Id == "shield" then
		for i=1,16 do local a=i*math.pi/8; part("ShieldRimBolt",Vector3.new(.045,.045,.045),math.cos(a)*.30,math.sin(a)*.34,-.09,metal,Enum.Material.Neon,"Ball") end
		part("ShieldEmblem",Vector3.new(.22,.28,.055),0,.02,-.13,Color3.fromRGB(225,245,255),Enum.Material.Neon)
	elseif def.Id == "wings" then
		for side=-1,1,2 do
			for i=1,5 do part("Feather",Vector3.new(.13,.32-i*.025,.08),side*(.12+i*.095),-.04-i*.035,.04,rootColor:Lerp(Color3.new(1,1,1),i*.08),Enum.Material.SmoothPlastic,"Block",0,0,side*math.rad(20)) end
		end
	elseif def.Id == "fishing" then
		for i=1,8 do part("RodGripRib",Vector3.new(.14,.025,.13),0,-.12+i*.035,.09,dark,Enum.Material.Metal) end
		part("ReelHandle",Vector3.new(.08,.22,.08),.14,-.03,.05,metal,Enum.Material.Metal)
		part("ReelKnob",Vector3.new(.12,.10,.10),.17,.08,.05,rootColor,Enum.Material.SmoothPlastic,"Ball")
	elseif def.Id == "portal" then
		for i=1,18 do local a=i*math.pi/9; part("PortalRune",Vector3.new(.045,.10,.035),math.cos(a)*.17,math.sin(a)*.17,-.08,glow,Enum.Material.Neon,"Block",0,0,a) end
	elseif def.Id == "bubbles" then
		part("SoapTank",Vector3.new(.23,.30,.22),.12,-.05,.05,Color3.fromRGB(80,160,190),Enum.Material.Glass,"Block",0,0,0,.18)
		part("BubbleNozzle",Vector3.new(.32,.10,.10),0,.02,-.28,metal,Enum.Material.Metal,"Cylinder")
		part("BubbleRing",Vector3.new(.18,.18,.04),0,.02,-.38,rootColor,Enum.Material.Neon,"Cylinder",0,0,0,.10)
	else
		-- General premium greebles for every remaining item: rails, fasteners, status lights and grip plates.
		for side=-1,1,2 do
			part("SideArmor",Vector3.new(.045,.16,.30),side*.14,0,.03,metal,Enum.Material.Metal)
			for i=1,4 do part("GripRidge",Vector3.new(.18,.022,.025),0,-.12+i*.052,.07,dark,Enum.Material.Metal) end
			part("StatusLED",Vector3.new(.055,.055,.055),side*.13,.11,-.12,glow,Enum.Material.Neon,"Ball")
		end
		for i=1,4 do part("Fastener",Vector3.new(.035,.035,.035),-.09+i*.06,.12,.12,metal,Enum.Material.Metal,"Ball") end
	end

	-- Extra configurable micro-panels: makes each model feel assembled, not one smooth primitive.
	for i=1,4 do
		local side = i%2 == 0 and 1 or -1
		local z = -.17 + math.floor((i-1)/2)*.26
		part("MicroPanel",Vector3.new(.065,.025,.085),side*.145,.14,z, i%2==0 and glow or dark, i%2==0 and Enum.Material.Neon or Enum.Material.Metal)
	end
	return detailCount
end

local function makeToyProp(def)
	local rig = State.Rig
	if not rig or not rig.Active or not rig.Model or not rig.Model.Parent then return nil, "Ative a minifigura LEGO primeiro." end
	local mountName = (def.Mount == "back" and "MF_Torso") or ((def.Mount == "feet") and "MF_Hips") or "MF_HandR"
	local mount = rig.Model:FindFirstChild(mountName)
	if not mount or not mount:IsA("BasePart") then return nil, "Não encontrei o ponto de fixação do brinquedo." end
	local model = Instance.new("Model")
	model.Name = "LegoToy_" .. def.Id
	local offset = CFrame.new(0, -0.06, -0.18)
	if def.Mount == "back" then offset = CFrame.new(0, 0.0, 0.43) end
	if def.Mount == "feet" then offset = CFrame.new(0, -0.43, 0.03) end
	local rootCF = mount.CFrame * offset
	local specs = TOY_PARTS[def.Id]
	if not specs then
		local rootSize = Vector3.new(.22,.22,.35)
		specs = {{"Handle",rootSize,CFrame.new(),Color3.fromRGB(45,48,60),"Block"}}
		if def.Id == "football" or def.Id == "basketball" or def.Id == "tennis" then
			specs[1] = {"Ball",Vector3.new(.40,.40,.40),CFrame.new(),def.Color,"Ball"}
		elseif def.Id == "boomerang" then
			specs = {{"WingL",Vector3.new(.55,.10,.12),CFrame.new(-.20,0,0)*CFrame.Angles(0,0,math.rad(30)),def.Color,"Block"},{"WingR",Vector3.new(.55,.10,.12),CFrame.new(.20,0,0)*CFrame.Angles(0,0,math.rad(-30)),def.Color,"Block"}}
		elseif def.Id == "bubbles" then
			specs = {{"Pipe",Vector3.new(.18,.18,.36),CFrame.new(),Color3.fromRGB(125,205,225),"Cylinder"},{"BubbleEnd",Vector3.new(.25,.25,.08),CFrame.new(0,0,-.20),def.Color,"Ball"}}
		elseif def.Id == "lantern" then
			specs = {{"Handle",Vector3.new(.18,.40,.18),CFrame.new(0,.1,0),Color3.fromRGB(65,68,76),"Block"},{"Lamp",Vector3.new(.24,.24,.24),CFrame.new(0,-.12,-.05),def.Color,"Ball"}}
		elseif def.Id == "telescope" then
			specs = {{"Tube",Vector3.new(.22,.22,.68),CFrame.new(0,0,-.08),Color3.fromRGB(70,130,195),"Cylinder"},{"Lens",Vector3.new(.28,.28,.10),CFrame.new(0,0,-.43),Color3.fromRGB(95,225,255),"Cylinder"}}
		elseif def.Id == "drone" then
			specs = {{"Body",Vector3.new(.30,.16,.30),CFrame.new(),Color3.fromRGB(65,75,90),"Block"},{"RotorL",Vector3.new(.22,.05,.22),CFrame.new(-.24,.05,0),def.Color,"Cylinder"},{"RotorR",Vector3.new(.22,.05,.22),CFrame.new(.24,.05,0),def.Color,"Cylinder"}}
		elseif def.Id == "spring" then
			specs = {{"Sole",Vector3.new(.50,.13,.40),CFrame.new(0,-.13,0),Color3.fromRGB(40,45,52),"Block"},{"Spring",Vector3.new(.18,.34,.18),CFrame.new(0,.08,0),def.Color,"Cylinder"}}
		elseif def.Id == "portal" then
			specs = {{"Disc",Vector3.new(.35,.35,.12),CFrame.new(),def.Color,"Cylinder"},{"Center",Vector3.new(.18,.18,.14),CFrame.new(0,0,-.04),Color3.fromRGB(40,15,85),"Ball"}}
		elseif def.Id == "guitar" then
			specs = TOY_PARTS.guitar
		elseif def.Id == "fishing" then
			specs = TOY_PARTS.fishing
		elseif def.Id == "flag" then
			specs = TOY_PARTS.flag
		end
	end
	local root
	for index, spec in ipairs(specs) do
		local name, size, rel, color, shape = spec[1], spec[2], spec[3], spec[4], spec[5]
		local p = toyPart(model, "Toy_" .. name, size, rootCF * rel, color or def.Color, shape, Enum.Material.SmoothPlastic, false, 0)
		if index == 1 then root = p; model.PrimaryPart = root end
	end
	if not root then model:Destroy(); return nil, "O modelo visual ficou vazio." end
	addToyDetailPack(def, model, root.CFrame)
	for _, child in ipairs(model:GetChildren()) do
		if child:IsA("BasePart") and child ~= root then
			local weld = Instance.new("WeldConstraint")
			weld.Part0, weld.Part1, weld.Parent = root, child, root
		end
	end
	local attach = Instance.new("Weld")
	attach.Name = "ToyHandWeld"
	attach.Part0, attach.Part1 = mount, root
	attach.C0, attach.C1 = offset, CFrame.new()
	attach.Parent = root
	model.Parent = State.Character
	if def.Id == "lantern" then
		local lamp = model:FindFirstChild("Toy_Lamp")
		if lamp then toyMake("PointLight", {Name="ToyLanternLight",Color=def.Color,Brightness=1.4,Range=12,Enabled=true}, lamp) end
	end
	if def.Id == "saber" then
		local blade = model:FindFirstChild("Toy_ToyBlade")
		if blade then blade.Transparency = 1; blade.Material = Enum.Material.Neon end
	end
	if def.Id == "shield" then
		for _, p in ipairs(model:GetChildren()) do if p:IsA("BasePart") then p.Material = Enum.Material.Neon; p.Transparency = (p.Name == "Toy_Rim") and .15 or .35 end end
	end
	return model
end

local exitVehicle -- assigned by the V8 driving module below

local function destroyToyObject(obj)
	if obj and obj.Parent then pcall(function() obj:Destroy() end) end
end

local function clearEquippedToy(message)
	if exitVehicle then exitVehicle("Item guardado") end
	destroyToyObject(ToyKit.CurrentModel)
	ToyKit.CurrentModel = nil
	ToyKit.CurrentId = nil
	ToyKit.HandJoint = nil
	ToyKit.HandJointHome = nil
	if ToyKit.UseButton then ToyKit.UseButton.Visible = false end
	if message then ToyKit.Status = message end
	if ToyKit.StatusLabel then ToyKit.StatusLabel.Text = ToyKit.Status end
	if ToyKit.refreshCards then ToyKit.refreshCards() end
end

local function equipToy(id)
	if ToyKit.UseBusy then ToyKit.Status = "Espere a animação terminar."; if ToyKit.StatusLabel then ToyKit.StatusLabel.Text = ToyKit.Status end; return end
	if exitVehicle then exitVehicle("Trocando de item") end
	local def = ITEM_BY_ID[id]
	if not def then return end
	if ToyKit.CurrentId == id then
		clearEquippedToy(def.Name .. " guardado.")
		return
	end
	if not State.Enabled or not State.Rig or not State.Rig.Active then
		ToyKit.Status = "Ative a transformação LEGO antes de pegar itens."
		if ToyKit.StatusLabel then ToyKit.StatusLabel.Text = ToyKit.Status end
		return
	end
	destroyToyObject(ToyKit.CurrentModel)
	ToyKit.CurrentModel = nil
	ToyKit.CurrentId = nil
	local model, err = makeToyProp(def)
	if not model then
		ToyKit.Status = tostring(err or "Não foi possível equipar.")
	else
		ToyKit.CurrentModel = model
		ToyKit.CurrentId = id
		ToyKit.HandJoint = model:FindFirstChild("ToyHandWeld", true)
		ToyKit.HandJointHome = ToyKit.HandJoint and ToyKit.HandJoint.C0 or CFrame.new(0,-0.06,-0.18)
		ToyKit.Status = def.Name .. " equipado. Ação com preparação, impacto e recuperação: toque em USAR."
	end
	if ToyKit.StatusLabel then ToyKit.StatusLabel.Text = ToyKit.Status end
	if ToyKit.UseButton then
		ToyKit.UseButton.Visible = model ~= nil
		ToyKit.UseButton.Text = "USAR\n" .. string.upper(def.Name:sub(1,12))
		ToyKit.UseButton.BackgroundColor3 = Color3.fromRGB(60,150,225)
	end
	if ToyKit.refreshCards then ToyKit.refreshCards() end
end

local function toyBurst(position, color, count, scale)
	local folder = ensureToyFX()
	count = math.clamp(count or 8, 1, 24)
	scale = scale or 1
	for i = 1, count do
		local orb = toyPart(folder, "ToySpark", Vector3.new(.09,.09,.09) * scale, CFrame.new(position), color, "Ball", Enum.Material.Neon, true, 0)
		local randomDir = Vector3.new(math.random(-100,100)/100, math.random(-20,120)/100, math.random(-100,100)/100)
		if randomDir.Magnitude < .1 then randomDir = Vector3.new(0,1,0) end
		local distance = math.random(12,42)/10 * scale
		local tween = TweenService:Create(orb, TweenInfo.new(math.random(25,70)/100, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {Position=position + randomDir.Unit * distance, Transparency=1, Size=Vector3.new(.015,.015,.015)})
		tween:Play()
		Debris:AddItem(orb, .85)
	end
end

local function toyBeam(startPos, endPos, color, thickness, life)
	local delta = endPos - startPos
	if delta.Magnitude < .05 then return end
	local beam = toyPart(ensureToyFX(), "ToyLightTrail", Vector3.new(thickness or .09, thickness or .09, delta.Magnitude), CFrame.lookAt((startPos+endPos)*.5, endPos), color, "Block", Enum.Material.Neon, true, .05)
	TweenService:Create(beam, TweenInfo.new(life or .30, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {Transparency=1, Size=Vector3.new(.015,.015,delta.Magnitude)}):Play()
	Debris:AddItem(beam, (life or .30)+.1)
end

local function getFXRayParams()
	local p = RaycastParams.new()
	p.FilterType = Enum.RaycastFilterType.Exclude
	p.FilterDescendantsInstances = {State.Character, ToyKit.FXFolder, ToyKit.CurrentModel}
	p.IgnoreWater = false
	return p
end

local function impactVisual(position, normal, color, mode)
	toyBurst(position, color, mode == "confetti" and 24 or 14, mode == "firework" and 1.6 or 1.15)
	if mode == "swing" then
		for ring=1,3 do
			local arc=toyPart(ensureToyFX(),"ToyImpactArc",Vector3.new(.07,.07,.8+ring*.35),CFrame.new(position+normal*.12)*CFrame.Angles(math.rad(20*ring),math.rad(ring*26),math.rad(70)),color,"Block",Enum.Material.Neon,true,.12)
			TweenService:Create(arc,TweenInfo.new(.26+ring*.07,Enum.EasingStyle.Quart,Enum.EasingDirection.Out),{Transparency=1,Size=Vector3.new(.015,.015,1.6+ring*.5)}):Play(); Debris:AddItem(arc,.5)
		end
	end
	if mode == "paint" then
		local spot = toyPart(ensureToyFX(), "LocalPaintSplash", Vector3.new(.34,.05,.34), CFrame.lookAt(position, position+normal), color, "Cylinder", Enum.Material.Neon, true, .18)
		Debris:AddItem(spot, 2.8)
	elseif mode == "freeze" then
		for i=1,4 do
			local shard = toyPart(ensureToyFX(), "ToyIceShard", Vector3.new(.06,.42,.06), CFrame.new(position) * CFrame.Angles(math.random(),math.random(),math.random()), color, "Block", Enum.Material.Glass, true, .15)
			TweenService:Create(shard,TweenInfo.new(.6),{Transparency=1,Size=Vector3.new(.01,.01,.01)}):Play()
			Debris:AddItem(shard,.7)
		end
	elseif mode == "portal" then
		for i=1,2 do
			local ring=toyPart(ensureToyFX(),"ToyPortalRing",Vector3.new(.75,.75,.08),CFrame.lookAt(position+normal*.15,position+normal),color,"Cylinder",Enum.Material.Neon,true,.15)
			TweenService:Create(ring,TweenInfo.new(.7),{Size=Vector3.new(1.8,1.8,.04),Transparency=1}):Play()
			Debris:AddItem(ring,.8)
		end
	end
end

local function untrackToyConnection(conn)
	for i = #ToyKit.Connections, 1, -1 do
		if ToyKit.Connections[i] == conn then table.remove(ToyKit.Connections, i) end
	end
end

local function launchToyProjectile(def, mode, options)
	options = options or {}
	if ToyKit.ActiveProjectiles >= ToyKit.MaxProjectiles then
		ToyKit.Status = "Muitos efeitos ativos; espere um instante."
		if ToyKit.StatusLabel then ToyKit.StatusLabel.Text = ToyKit.Status end
		return
	end
	local camera = Workspace.CurrentCamera
	if not camera then return end
	local direction, target = getToyAim(85)
	local origin = camera.CFrame.Position + camera.CFrame.LookVector * 1.5
	local speed = options.Speed or 74
	local partSize = options.Size or Vector3.new(.20,.20,.42)
	local shape = options.Shape or "Ball"
	local orb = toyPart(ensureToyFX(), "ToyProjectile_" .. def.Id, partSize, CFrame.new(origin), options.Color or def.Color, shape, options.Material or Enum.Material.Neon, true, 0)
	local projectileDecor = {}
	if def.Id == "football" or def.Id == "basketball" or def.Id == "tennis" then
		local baseRadius = math.max(.10, partSize.X*.46)
		local seam = def.Id == "basketball" and Color3.fromRGB(38,30,26) or (def.Id == "football" and Color3.fromRGB(35,37,42) or Color3.fromRGB(225,245,180))
		for i=1,(def.Id == "tennis" and 4 or 10) do
			local a=i*math.pi*2/(def.Id == "tennis" and 4 or 10)
			local rel=CFrame.new(math.cos(a)*baseRadius*.62,math.sin(a)*baseRadius*.62,baseRadius*.42)*CFrame.Angles(0,0,a)
			local d=toyPart(ensureToyFX(),"ToyBallSeam",Vector3.new(baseRadius*.58,.018,.018),CFrame.new(origin)*rel,seam, "Block", Enum.Material.SmoothPlastic,true,0)
			table.insert(projectileDecor,{Part=d,Rel=rel})
		end
		for i=1,6 do
			local a=i*math.pi/3
			local rel=CFrame.new(math.cos(a)*baseRadius*.72,math.sin(a)*baseRadius*.72,-baseRadius*.36)
			local d=toyPart(ensureToyFX(),"ToyBallPanel",Vector3.new(.045,.035,.025)*math.max(.8,baseRadius*5),CFrame.new(origin)*rel,seam,"Block",Enum.Material.SmoothPlastic,true,0)
			table.insert(projectileDecor,{Part=d,Rel=rel})
		end
	end
	ToyKit.ActiveProjectiles += 1
	local velocity = direction * speed + Vector3.new(0, options.Lift or 0, 0)
	local gravity = options.Gravity or 0
	local lifetime = options.Life or 1.6
	local startTime = os.clock()
	local returning = options.Returning == true
	local bounced = 0
	local lastPos = origin
	local finished = false
	local conn
	local returnStart = nil
	local returnFrom = nil
	local function finish(pos, normal, didImpact)
		if finished then return end
		finished = true
		if conn then conn:Disconnect(); untrackToyConnection(conn) end
		if didImpact then impactVisual(pos, normal or Vector3.yAxis, options.Color or def.Color, mode) end
		destroyToyObject(orb)
		for _,entry in ipairs(projectileDecor) do if entry.Part and entry.Part.Parent then entry.Part:Destroy() end end
		ToyKit.ActiveProjectiles = math.max(0, ToyKit.ActiveProjectiles - 1)
	end
	conn = RunService.Heartbeat:Connect(function(dt)
		if not orb.Parent or not State.Character or not State.Character.Parent then finish(lastPos, Vector3.yAxis, false); return end
		local elapsed = os.clock() - startTime
		if elapsed > lifetime then finish(orb.Position, Vector3.yAxis, false); return end
		if returning and elapsed > .48 then
			if not returnStart then returnStart = elapsed; returnFrom = orb.Position end
			local returnAlpha = math.clamp((elapsed-returnStart)/.55,0,1)
			local playerRoot = State.Root
			local home = playerRoot and (playerRoot.Position + Vector3.new(0,1,0)) or origin
			local pos = returnFrom:Lerp(home, returnAlpha)
			local delta = pos - orb.Position
			if delta.Magnitude > .01 then orb.CFrame = CFrame.lookAt(pos, pos+delta.Unit) end
			lastPos = pos
			if returnAlpha >= 1 then toyBurst(home, options.Color or def.Color, 4, .55); finish(home, Vector3.yAxis, false) end
			return
		end
		velocity += Vector3.new(0,-gravity,0)*dt
		local nextPos = lastPos + velocity*dt
		local cast = Workspace:Raycast(lastPos, nextPos-lastPos, getFXRayParams())
		if cast then
			if options.Bounce and bounced < 2 then
				bounced += 1
				velocity = (velocity - 2*velocity:Dot(cast.Normal)*cast.Normal)*.58 + Vector3.new(0,options.BounceLift or 7,0)
				lastPos = cast.Position + cast.Normal*.08
				orb.CFrame = CFrame.new(lastPos)
			else
				finish(cast.Position, cast.Normal, true)
			end
		else
			lastPos = nextPos
			orb.CFrame = CFrame.new(nextPos, nextPos + velocity.Unit)
		end
	end)
	table.insert(ToyKit.Connections, conn)
	if #projectileDecor>0 then
		local follower=RunService.Heartbeat:Connect(function()
			if not orb.Parent then if follower then follower:Disconnect(); untrackToyConnection(follower) end; return end
			local cf=orb.CFrame
			local spin=CFrame.Angles(os.clock()*3,os.clock()*5,os.clock()*2)
			for _,entry in ipairs(projectileDecor) do if entry.Part and entry.Part.Parent then entry.Part.CFrame=cf*spin*entry.Rel end end
		end)
		table.insert(ToyKit.Connections,follower)
	end
	Debris:AddItem(orb, lifetime + .5)
end

local function getToyOrigin()
	local camera = Workspace.CurrentCamera
	if camera then return camera.CFrame.Position + camera.CFrame.LookVector*1.1 end
	return State.Root and State.Root.Position or Vector3.zero
end

---------------------------------------------------------------------
-- V8 DRIVABLE VEHICLES
-- Independent vehicle movement, ground sampling, obstacle probes,
-- rotating wheels, mobile hold-controls and clean exit/restore path.
-- Foot movement stats are restored on exit; walking speed is never buffed.
---------------------------------------------------------------------
local VehicleState = {
	Active = false,
	Model = nil,
	Kind = nil,
	Speed = 0,
	Yaw = 0,
	Y = 0,
	VerticalSpeed = 0,
	WheelRotation = 0,
	Steer = 0,
	Input = {Throttle=false, Brake=false, Left=false, Right=false, Handbrake=false},
	Connections = {},
	BodyParts = {},
	Wheels = {},
	SavedHumanoid = nil,	SavedHumanoidProps = nil,
	SavedRoot = nil,
	SpeedLabel = nil,
	SavedItemTransparency = {},
	ControlGui = nil,
	StatusTime = 0,
}

local function vehicleRayParams()
	local p = RaycastParams.new()
	p.FilterType = Enum.RaycastFilterType.Exclude
	local exclude = {State.Character, ToyKit.FXFolder, VehicleState.Model}
	p.FilterDescendantsInstances = exclude
	p.IgnoreWater = false
	return p
end

local function addVehicleBodyPart(model, partList, base, name, size, localCF, color, material, shape, transparency)
	local p = toyPart(model, name, size, base * localCF, color or Color3.fromRGB(210,45,42), shape or "Block", material or Enum.Material.SmoothPlastic, true, transparency or 0)
	p.CanCollide = false; p.CanTouch = false; p.CanQuery = false
	table.insert(partList, {Part=p, LocalCF=localCF, Kind="body"})
	return p
end

local function addWheelAssembly(model, wheelList, base, name, centerCF, radius, width, color, isFront)
	local wheel = {Name=name, CenterCF=centerCF, Radius=radius, Width=width, IsFront=isFront, Parts={}, BaseColor=color}
	local function wpart(partName, size, rel, partColor, material, shape, transparency)
		local p = toyPart(model, name .. "_" .. partName, size, base * centerCF * rel, partColor or Color3.fromRGB(28,29,33), shape or "Block", material or Enum.Material.SmoothPlastic, true, transparency or 0)
		p.CanCollide=false; p.CanTouch=false; p.CanQuery=false
		table.insert(wheel.Parts, {Part=p, Rel=rel})
		return p
	end
	-- Cylinder axes are laid across the wheel, producing a visibly thick tire.
	wpart("Tire", Vector3.new(radius*2,radius*2,width), CFrame.Angles(0,0,math.pi/2), Color3.fromRGB(22,24,28), Enum.Material.SmoothPlastic, "Cylinder")
	wpart("Sidewall", Vector3.new(radius*1.70,radius*1.70,width*1.05), CFrame.new((width*.52),0,0)*CFrame.Angles(0,0,math.pi/2), Color3.fromRGB(38,40,45), Enum.Material.SmoothPlastic, "Cylinder")
	wpart("InnerRim", Vector3.new(radius*1.25,radius*1.25,width*1.10), CFrame.new(width*.60,0,0)*CFrame.Angles(0,0,math.pi/2), Color3.fromRGB(150,158,170), Enum.Material.Metal, "Cylinder")
	wpart("Hub", Vector3.new(radius*.78,radius*.78,width*1.18), CFrame.new(width*.64,0,0)*CFrame.Angles(0,0,math.pi/2), color, Enum.Material.Metal, "Cylinder")
	wpart("AxleCap", Vector3.new(radius*.28,radius*.28,width*1.24), CFrame.new(width*.70,0,0)*CFrame.Angles(0,0,math.pi/2), Color3.fromRGB(235,238,242), Enum.Material.Metal, "Cylinder")
	for i=1,8 do
		local a=math.pi*2*i/8
		local y,z=math.cos(a)*radius*.32,math.sin(a)*radius*.32
		wpart("Spoke", Vector3.new(width*.16,radius*.62,width*.10), CFrame.new(width*.69,y,z)*CFrame.Angles(a,0,0), Color3.fromRGB(205,210,218), Enum.Material.Metal, "Block")
		wpart("Bolt", Vector3.new(width*.12,width*.12,width*.12), CFrame.new(width*.73,math.cos(a)*radius*.48,math.sin(a)*radius*.48), Color3.fromRGB(240,240,245), Enum.Material.Metal, "Ball")
	end
	if isFront then
		wpart("BrakeDisc", Vector3.new(radius*1.05,radius*1.05,width*.14), CFrame.new(width*.48,0,0)*CFrame.Angles(0,0,math.pi/2), Color3.fromRGB(115,125,138), Enum.Material.Metal, "Cylinder")
	end
	table.insert(wheelList, wheel)
	return wheel
end

local function buildCarVehicle(model, base, def)
	local body, wheels = {}, {}
	local red = def.Color or Color3.fromRGB(205,42,40)
	local dark = Color3.fromRGB(28,31,38)
	local glass = Color3.fromRGB(65,155,195)
	local silver = Color3.fromRGB(170,180,190)
	local function b(n,s,x,y,z,c,m,rx,ry,rz,tr)
		return addVehicleBodyPart(model,body,base,n,s,CFrame.new(x,y,z)*CFrame.Angles(rx or 0,ry or 0,rz or 0),c,m or Enum.Material.SmoothPlastic,"Block",tr)
	end
	-- Six-stud sport chassis, deliberately open cockpit to fit the LEGO rider.
	b("ChassisPan",Vector3.new(2.75,.34,5.85),0,.72,0,dark,Enum.Material.Metal)
	b("SideSillL",Vector3.new(.27,.28,4.45),-1.37,.72,.12,red)
	b("SideSillR",Vector3.new(.27,.28,4.45),1.37,.72,.12,red)
	b("FrontNose",Vector3.new(2.72,.38,1.45),0,.82,-2.22,red)
	b("HoodUpper",Vector3.new(2.30,.12,1.15),0,1.07,-2.28,red)
	b("RearDeck",Vector3.new(2.62,.30,1.18),0,.84,2.16,red)
	b("RearSpoiler",Vector3.new(2.15,.12,.25),0,1.35,2.62,dark,Enum.Material.Metal)
	b("SpoilerStandL",Vector3.new(.10,.32,.10),-.78,1.20,2.57,silver,Enum.Material.Metal)
	b("SpoilerStandR",Vector3.new(.10,.32,.10),.78,1.20,2.57,silver,Enum.Material.Metal)
	b("FrontBumper",Vector3.new(2.86,.22,.24),0,.58,-3.02,dark,Enum.Material.Metal)
	b("RearBumper",Vector3.new(2.86,.22,.24),0,.58,2.98,dark,Enum.Material.Metal)
	b("FrontSplitter",Vector3.new(2.72,.06,.52),0,.46,-2.92,Color3.fromRGB(18,20,24),Enum.Material.Metal)
	b("RearDiffuser",Vector3.new(2.30,.08,.40),0,.45,2.88,dark,Enum.Material.Metal)
	-- Hood stripes, panels and vents.
	for side=-1,1,2 do
		b("RacingStripe",Vector3.new(.12,.026,1.02),side*.44,1.139,-2.25,Color3.fromRGB(245,235,205),Enum.Material.SmoothPlastic)
		for i=1,6 do b("HoodVent",Vector3.new(.22,.028,.075),side*.80,1.145,-2.63+i*.14,dark,Enum.Material.Metal) end
		b("FrontFender",Vector3.new(.40,.25,1.20),side*1.34,.95,-1.63,red)
		b("RearFender",Vector3.new(.40,.25,1.15),side*1.34,.95,1.67,red)
		b("MirrorArm",Vector3.new(.18,.08,.20),side*1.42,1.18,-.96,dark,Enum.Material.Metal)
		b("MirrorGlass",Vector3.new(.22,.10,.17),side*1.48,1.22,-.97,silver,Enum.Material.Glass,.0,0,0,.12)
		b("DoorAccent",Vector3.new(.025,.22,1.12),side*1.515,.86,.06,Color3.fromRGB(120,30,28),Enum.Material.Metal)
		b("DoorHandle",Vector3.new(.04,.045,.24),side*1.54,.98,.26,silver,Enum.Material.Metal)
		b("SideSignal",Vector3.new(.035,.07,.18),side*1.53,.96,-1.01,Color3.fromRGB(255,170,45),Enum.Material.Neon)
	end
	-- Open cockpit, bucket seat and a carefully detailed steering wheel.
	b("CockpitFloor",Vector3.new(1.70,.10,1.75),0,.95,.33,Color3.fromRGB(36,39,47),Enum.Material.Metal)
	b("SeatBase",Vector3.new(.78,.18,.74),0,1.12,.65,Color3.fromRGB(36,39,48),Enum.Material.SmoothPlastic)
	b("SeatBack",Vector3.new(.78,.85,.20),0,1.55,1.01,Color3.fromRGB(53,57,67),Enum.Material.SmoothPlastic,math.rad(-7),0,0)
	b("SeatHeadrest",Vector3.new(.52,.27,.18),0,1.99,1.08,red)
	b("SeatPad",Vector3.new(.63,.07,.55),0,1.235,.61,Color3.fromRGB(75,80,90))
	b("DashPanel",Vector3.new(1.85,.22,.25),0,1.43,-.70,dark,Enum.Material.Metal,math.rad(-8),0,0)
	b("DashScreen",Vector3.new(.44,.15,.025),0,1.56,-.77,Color3.fromRGB(60,230,165),Enum.Material.Neon)
	for i=1,5 do b("DashButton",Vector3.new(.07,.05,.045),-.56+i*.23,1.56,-.77,i%2==0 and red or silver,Enum.Material.Neon) end
	b("SteeringColumn",Vector3.new(.10,.40,.10),0,1.47,-.42,silver,Enum.Material.Metal,math.rad(-30),0,0)
	b("SteeringWheel",Vector3.new(.42,.42,.07),0,1.60,-.50,dark,Enum.Material.Metal)
	for i=1,10 do local a=i*math.pi/5; b("SteeringGrip",Vector3.new(.04,.04,.075),math.cos(a)*.20,1.60+math.sin(a)*.20,-.54,red,Enum.Material.Metal,0,0,a) end
	-- Windscreen and pillars; the top remains open for the avatar's head.
	b("WindshieldGlass",Vector3.new(2.12,.62,.075),0,1.68,-1.08,glass,Enum.Material.Glass,math.rad(-16),0,0,.25)
	b("WindshieldFrame",Vector3.new(2.26,.08,.14),0,1.99,-1.02,silver,Enum.Material.Metal,math.rad(-16),0,0)
	b("RearBraceL",Vector3.new(.10,.55,.12),-.92,1.46,1.48,silver,Enum.Material.Metal,math.rad(-16),0,0)
	b("RearBraceR",Vector3.new(.10,.55,.12),.92,1.46,1.48,silver,Enum.Material.Metal,math.rad(-16),0,0)
	-- Headlights, tail lamps, grilles and exhaust tips.
	for side=-1,1,2 do
		b("HeadlightHousing",Vector3.new(.55,.20,.08),side*.88,.92,-2.98,dark,Enum.Material.Metal)
		b("Headlight",Vector3.new(.40,.12,.05),side*.88,.93,-3.035,Color3.fromRGB(235,250,255),Enum.Material.Neon)
		b("TailLamp",Vector3.new(.43,.17,.07),side*.91,.91,2.78,Color3.fromRGB(255,38,38),Enum.Material.Neon)
		b("ExhaustOuter",Vector3.new(.20,.17,.28),side*.76,.53,3.00,silver,Enum.Material.Metal)
		b("ExhaustCore",Vector3.new(.11,.10,.30),side*.76,.53,3.04,Color3.fromRGB(45,50,60),Enum.Material.Metal)
		b("Canard",Vector3.new(.50,.07,.35),side*1.24,.56,-2.84,dark,Enum.Material.Metal,0,0,side*math.rad(8))
	end
	-- Wheel packages with actual visual rotation and steering; front axle is steerable.
	for _, z in ipairs({-2.05,2.05}) do
		for _, x in ipairs({-1.52,1.52}) do
			local front = z < 0
			addWheelAssembly(model,wheels,base,(front and "Front" or "Rear")..(x<0 and "Left" or "Right").."Wheel",CFrame.new(x,.74,z),.76,.42,front and silver or red,front)
		end
	end
	return body, wheels, {Length=6.5,Width=3.5, RideHeight=2.05, MaxSpeed=27, Acceleration=15.5, ReverseSpeed=9, TurnRate=1.10, BrakeRate=24, RideZ=.45}
end

local function buildMotorcycle(model, base, def)
	local body, wheels = {}, {}
	local blue = def.Color or Color3.fromRGB(45,120,235)
	local dark = Color3.fromRGB(24,27,32)
	local silver = Color3.fromRGB(165,175,185)
	local function b(n,s,x,y,z,c,m,rx,ry,rz,tr)
		return addVehicleBodyPart(model,body,base,n,s,CFrame.new(x,y,z)*CFrame.Angles(rx or 0,ry or 0,rz or 0),c,m or Enum.Material.SmoothPlastic,"Block",tr)
	end
	-- Motorcycle geometry: large rubber wheels, chassis, engine fins, fuel tank, fork, controls.
	addWheelAssembly(model,wheels,base,"RearMotorcycleWheel",CFrame.new(0,.72,1.28),.72,.30,dark,false)
	addWheelAssembly(model,wheels,base,"FrontMotorcycleWheel",CFrame.new(0,.72,-1.42),.72,.30,dark,true)
	b("MainFrame",Vector3.new(.22,.24,1.55),0,1.03,-.02,blue,Enum.Material.Metal,math.rad(-7),0,0)
	b("LowerFrame",Vector3.new(.18,.18,1.28),0,.80,.05,silver,Enum.Material.Metal,math.rad(11),0,0)
	b("SwingArmL",Vector3.new(.10,.14,1.10),-.18,.83,.70,silver,Enum.Material.Metal,math.rad(-18),0,0)
	b("SwingArmR",Vector3.new(.10,.14,1.10),.18,.83,.70,silver,Enum.Material.Metal,math.rad(-18),0,0)
	b("EngineBlock",Vector3.new(.72,.60,.66),0,1.02,.27,Color3.fromRGB(82,88,98),Enum.Material.Metal)
	for i=1,7 do b("EngineCoolingFin",Vector3.new(.78,.035,.70),0,.78+i*.055,.27,dark,Enum.Material.Metal) end
	b("EngineCoverL",Vector3.new(.10,.39,.40),-.40,1.05,.28,blue,Enum.Material.Metal,0,0,math.rad(-8))
	b("EngineCoverR",Vector3.new(.10,.39,.40),.40,1.05,.28,blue,Enum.Material.Metal,0,0,math.rad(8))
	b("FuelTank",Vector3.new(.70,.47,.93),0,1.44,-.43,blue,Enum.Material.SmoothPlastic,math.rad(-9),0,0)
	b("TankStripe",Vector3.new(.17,.045,.72),0,1.685,-.44,Color3.fromRGB(240,242,245),Enum.Material.Neon,math.rad(-9),0,0)
	b("TankBadge",Vector3.new(.23,.13,.04),0,1.45,-.12,Color3.fromRGB(245,205,58),Enum.Material.Neon)
	b("SeatBase",Vector3.new(.54,.16,.86),0,1.48,.47,dark,Enum.Material.SmoothPlastic,math.rad(-5),0,0)
	b("SeatRear",Vector3.new(.48,.12,.31),0,1.56,.84,dark,Enum.Material.SmoothPlastic)
	b("TailFairing",Vector3.new(.45,.28,.66),0,1.47,1.02,blue,Enum.Material.SmoothPlastic,math.rad(-10),0,0)
	b("TailLight",Vector3.new(.28,.11,.05),0,1.50,1.36,Color3.fromRGB(255,35,40),Enum.Material.Neon)
	b("RearMudguard",Vector3.new(.46,.11,1.20),0,1.18,1.31,dark,Enum.Material.SmoothPlastic,math.rad(8),0,0)
	b("FrontMudguard",Vector3.new(.48,.10,1.08),0,1.29,-1.40,blue,Enum.Material.SmoothPlastic,math.rad(-6),0,0)
	for side=-1,1,2 do
		b("FrontFork",Vector3.new(.10,.85,.11),side*.18,1.14,-1.36,silver,Enum.Material.Metal,math.rad(-15),0,0)
		b("ForkGuard",Vector3.new(.15,.32,.14),side*.18,.86,-1.38,blue,Enum.Material.SmoothPlastic,math.rad(-15),0,0)
		b("FootPeg",Vector3.new(.32,.07,.12),side*.38,.84,.25,dark,Enum.Material.Metal)
		b("MirrorStem",Vector3.new(.055,.42,.055),side*.23,1.89,-1.06,silver,Enum.Material.Metal,math.rad(-15),0,side*math.rad(8))
		b("Mirror",Vector3.new(.24,.16,.07),side*.29,2.08,-1.10,dark,Enum.Material.Metal,0,0,side*math.rad(12))
		b("TurnSignal",Vector3.new(.14,.12,.14),side*.39,1.53,-1.12,Color3.fromRGB(255,170,40),Enum.Material.Neon)
		b("BrakeDisc",Vector3.new(.09,.86,.86),side*.20,.72,1.28,silver,Enum.Material.Metal,0,0,math.pi/2)
		b("ExhaustCan",Vector3.new(.18,.20,.95),side*.34,.91,.70,silver,Enum.Material.Metal,math.rad(90),0,0)
		b("ExhaustTip",Vector3.new(.20,.20,.18),side*.34,.91,1.14,dark,Enum.Material.Metal,math.rad(90),0,0)
	end
	b("HandleStem",Vector3.new(.13,.38,.13),0,1.72,-1.06,silver,Enum.Material.Metal,math.rad(-15),0,0)
	b("HandleBar",Vector3.new(1.05,.09,.10),0,1.89,-1.18,dark,Enum.Material.Metal)
	b("GripL",Vector3.new(.26,.10,.13),-.52,1.89,-1.18,Color3.fromRGB(35,36,41),Enum.Material.SmoothPlastic)
	b("GripR",Vector3.new(.26,.10,.13),.52,1.89,-1.18,Color3.fromRGB(35,36,41),Enum.Material.SmoothPlastic)
	b("ThrottleSwitch",Vector3.new(.16,.07,.15),.38,1.96,-1.18,blue,Enum.Material.Neon)
	b("HeadlightHousing",Vector3.new(.47,.38,.25),0,1.69,-1.48,dark,Enum.Material.Metal)
	b("HeadlightGlass",Vector3.new(.36,.27,.06),0,1.69,-1.62,Color3.fromRGB(220,245,255),Enum.Material.Neon,0,0,0,.05)
	b("FrontNumberPlate",Vector3.new(.47,.25,.045),0,1.35,-1.63,Color3.fromRGB(245,245,245),Enum.Material.SmoothPlastic,math.rad(-8),0,0)
	b("NumberMark",Vector3.new(.27,.08,.025),0,1.36,-1.66,blue,Enum.Material.Neon)
	for i=1,4 do b("TankVent",Vector3.new(.25,.035,.04),-.18+i*.09,1.46,-.70,dark,Enum.Material.Metal) end
	return body, wheels, {Length=4.6,Width=1.8,RideHeight=2.03,MaxSpeed=31,Acceleration=18,ReverseSpeed=8,TurnRate=1.32,BrakeRate=27,RideZ=.25}
end

local function setVehicleHudStatus(text)
	ToyKit.Status = text
	if ToyKit.StatusLabel then ToyKit.StatusLabel.Text = text end
end

local function removeVehicleControlGui()
	if VehicleState.ControlGui then
		VehicleState.ControlGui:Destroy()
		VehicleState.ControlGui = nil
	end
	VehicleState.SpeedLabel = nil
	for i=#VehicleState.Connections,1,-1 do
		local c=VehicleState.Connections[i]
		VehicleState.Connections[i]=nil
		pcall(function() c:Disconnect() end)
	end
end

local function restoreVehicleDriver()
	local humanoid, root = VehicleState.SavedHumanoid, VehicleState.SavedRoot
	if root and root.Parent then
		root.AssemblyLinearVelocity = Vector3.zero
		root.AssemblyAngularVelocity = Vector3.zero
		if VehicleState.Model and VehicleState.Model.Parent then
			root.CFrame = root.CFrame * CFrame.new(VehicleState.Kind == "mini_bike" and 1.65 or 2.0, 0, 0)
		else
			root.CFrame = root.CFrame * CFrame.new(2,0,0)
		end
	end
	if humanoid and humanoid.Parent then
		local saved=VehicleState.SavedHumanoidProps or {}
		humanoid.PlatformStand = saved.PlatformStand == true
		humanoid.AutoRotate = saved.AutoRotate ~= false
		humanoid.WalkSpeed = saved.WalkSpeed or 16
		humanoid.JumpPower = saved.JumpPower or 50
		humanoid.JumpHeight = saved.JumpHeight or 7.2
		humanoid.Sit = false
		pcall(function() humanoid:ChangeState(Enum.HumanoidStateType.GettingUp) end)
	end
	for part, transparency in pairs(VehicleState.SavedItemTransparency) do
		if part and part.Parent then part.Transparency = transparency end
	end
	VehicleState.SavedItemTransparency = {}
end

exitVehicle = function(reason)
	if not VehicleState.Active then return end
	VehicleState.Active=false
	State.InVehicle=false
	if UI and UI.setActionsVisible then UI.setActionsVisible(State.Enabled and State.Rig ~= nil and State.Rig.Active) end
	for _, key in ipairs({"Throttle","Brake","Left","Right","Handbrake"}) do VehicleState.Input[key]=false end
	if VehicleState.Loop then
		local conn=VehicleState.Loop; VehicleState.Loop=nil
		untrackToyConnection(conn)
		pcall(function() conn:Disconnect() end)
	end
	removeVehicleControlGui()
	restoreVehicleDriver()
	if VehicleState.Model and VehicleState.Model.Parent then
		local pos=VehicleState.Model:GetPivot().Position
		toyBurst(pos, VehicleState.Kind == "mini_bike" and Color3.fromRGB(75,190,255) or Color3.fromRGB(255,175,65), 18, 1.2)
		VehicleState.Model:Destroy()
	end
	VehicleState.Model=nil
	VehicleState.SavedHumanoid=nil; VehicleState.SavedHumanoidProps=nil; VehicleState.SavedRoot=nil
	VehicleState.BodyParts={}; VehicleState.Wheels={}; VehicleState.Definition=nil
	VehicleState.Speed=0; VehicleState.VerticalSpeed=0
	if ToyKit.CurrentModel and ToyKit.CurrentModel.Parent then
		for part, transparency in pairs(VehicleState.SavedItemTransparency) do if part.Parent then part.Transparency=transparency end end
	end
	setVehicleHudStatus(reason or "Veículo guardado. A velocidade normal a pé foi restaurada.")
	if ToyKit.UseButton and ToyKit.CurrentId then ToyKit.UseButton.Text="USAR\n"..string.upper(ITEM_BY_ID[ToyKit.CurrentId].Name:sub(1,12)) end
end

local function createVehicleControlGui(kind)
	local pg=player:FindFirstChildOfClass("PlayerGui") or player:WaitForChild("PlayerGui")
	local gui=Instance.new("ScreenGui")
	gui.Name="LegoDrivableVehicleControls"
	gui.ResetOnSpawn=false; gui.DisplayOrder=90; gui.IgnoreGuiInset=false
	pcall(function() gui.ScreenInsets=Enum.ScreenInsets.DeviceSafeInsets end)
	gui.Parent=pg; VehicleState.ControlGui=gui
	if ToyKit.Gui then local panel=ToyKit.Gui:FindFirstChild("InventoryPanel"); if panel then panel.Visible=false end end
	local function makeButton(name,label,pos,size,anchor,color,flag)
		local b=Instance.new("TextButton")
		b.Name=name; b.Text=label; b.Font=Enum.Font.GothamBold; b.TextScaled=true; b.TextColor3=Color3.fromRGB(250,250,255)
		b.BackgroundColor3=color; b.BackgroundTransparency=.10; b.BorderSizePixel=0
		b.AnchorPoint=anchor or Vector2.new(0,1); b.Position=pos; b.Size=size; b.AutoButtonColor=true; b.Parent=gui
		local corner=Instance.new("UICorner"); corner.CornerRadius=UDim.new(0,15); corner.Parent=b
		local stroke=Instance.new("UIStroke"); stroke.Color=Color3.fromRGB(220,230,245); stroke.Transparency=.45; stroke.Thickness=1.4; stroke.Parent=b
		if flag then
			local function press(input)
				if input.UserInputType==Enum.UserInputType.Touch or input.UserInputType==Enum.UserInputType.MouseButton1 then VehicleState.Input[flag]=true end
			end
			local function release(input)
				if input.UserInputType==Enum.UserInputType.Touch or input.UserInputType==Enum.UserInputType.MouseButton1 then VehicleState.Input[flag]=false end
			end
			table.insert(VehicleState.Connections,b.InputBegan:Connect(press))
			table.insert(VehicleState.Connections,b.InputEnded:Connect(release))
			b.Activated:Connect(function() VehicleState.Input[flag]=true; task.delay(.12,function() if VehicleState.Active then VehicleState.Input[flag]=false end end) end)
		end
		return b
	end
	local left=makeButton("SteerLeft","◀",UDim2.new(0,18,1,-100),UDim2.new(0,72,0,62),Vector2.new(0,1),Color3.fromRGB(53,66,91),"Left")
	local right=makeButton("SteerRight","▶",UDim2.new(0,98,1,-100),UDim2.new(0,72,0,62),Vector2.new(0,1),Color3.fromRGB(53,66,91),"Right")
	local brake=makeButton("VehicleBrake","FREIO\n◼",UDim2.new(1,-242,1,-100),UDim2.new(0,92,0,62),Vector2.new(1,1),Color3.fromRGB(156,63,63),"Brake")
	local gas=makeButton("VehicleThrottle",kind=="mini_bike" and "ACELERAR\n▲" or "ACELERAR\n▲",UDim2.new(1,-24,1,-100),UDim2.new(0,116,0,76),Vector2.new(1,1),Color3.fromRGB(38,126,88),"Throttle")
	local handbrake=makeButton("VehicleHandbrake",kind=="mini_bike" and "FREIO\nTRASEIRO" or "DERRAPAR",UDim2.new(1,-130,1,-188),UDim2.new(0,104,0,48),Vector2.new(1,1),Color3.fromRGB(125,91,45),"Handbrake")
	local exit=makeButton("ExitVehicle","SAIR ✕",UDim2.new(1,-18,0,16),UDim2.new(0,100,0,42),Vector2.new(1,0),Color3.fromRGB(130,49,55),nil)
	exit.Activated:Connect(function() exitVehicle("Você saiu do veículo.") end)
	local hint=Instance.new("TextLabel")
	hint.Name="VehicleHint"; hint.AnchorPoint=Vector2.new(.5,0); hint.Position=UDim2.new(.5,0,0,16); hint.Size=UDim2.new(0,250,0,42)
	hint.BackgroundColor3=Color3.fromRGB(18,22,30); hint.BackgroundTransparency=.24; hint.BorderSizePixel=0
	hint.Text=kind=="mini_bike" and "MOTO • SEGURE ACELERAR • A/D OU ◀ ▶" or "CARRO • SEGURE ACELERAR • A/D OU ◀ ▶"
	hint.TextColor3=Color3.fromRGB(245,248,255); hint.TextSize=12; hint.Font=Enum.Font.GothamBold; hint.TextWrapped=true; hint.Parent=gui
	local hc=Instance.new("UICorner"); hc.CornerRadius=UDim.new(0,10); hc.Parent=hint
	VehicleState.SpeedLabel=hint
end

local function createVehicle(def, spawnPosition, direction)
	local model=Instance.new("Model")
	model.Name=def.Id=="mini_bike" and "LegoPlayerMotorcycle_V8_Local" or "LegoPlayerSportsCar_V8_Local"
	local yaw=math.atan2(-direction.X,-direction.Z)
	local base=CFrame.new(spawnPosition)*CFrame.Angles(0,yaw,0)
	local params=vehicleRayParams()
	local ground=Workspace:Raycast(spawnPosition+Vector3.new(0,8,0),Vector3.new(0,-24,0),params)
	local groundedY=ground and ground.Position.Y or spawnPosition.Y
	base=CFrame.new(spawnPosition.X,groundedY,spawnPosition.Z)*CFrame.Angles(0,yaw,0)
	model.Parent=ensureToyFX()
	local body,wheels,driveSpec
	if def.Id=="mini_bike" then body,wheels,driveSpec=buildMotorcycle(model,base,def) else body,wheels,driveSpec=buildCarVehicle(model,base,def) end
	local primary=body[1] and body[1].Part
	if primary then model.PrimaryPart=primary end
	VehicleState.Model=model; VehicleState.Kind=def.Id; VehicleState.Definition=def
	VehicleState.BodyParts=body; VehicleState.Wheels=wheels; VehicleState.DriveSpec=driveSpec
	VehicleState.Speed=0; VehicleState.Yaw=yaw; VehicleState.Y=groundedY; VehicleState.VerticalSpeed=0; VehicleState.WheelRotation=0
	VehicleState.Input={Throttle=false,Brake=false,Left=false,Right=false,Handbrake=false}
	VehicleState.Steer=0; VehicleState.Active=true
	-- Save the real humanoid settings and freeze normal walking only while mounted.
	local hum,root=State.Humanoid,State.Root
	if not hum or not root or hum.Health<=0 then model:Destroy(); VehicleState.Active=false; VehicleState.Model=nil; return false end
	-- Cancel hanging/sliding before mounting, then capture untouched walking settings.
	resetParkour(true)
	VehicleState.SavedHumanoid=hum
	VehicleState.SavedHumanoidProps={PlatformStand=hum.PlatformStand,AutoRotate=hum.AutoRotate,WalkSpeed=hum.WalkSpeed,JumpPower=hum.JumpPower,JumpHeight=hum.JumpHeight}
	VehicleState.SavedRoot=root
	State.InVehicle=true
	if UI and UI.setActionsVisible then UI.setActionsVisible(false) end
	VehicleState.SavedItemTransparency={}
	if ToyKit.CurrentModel then
		for _,p in ipairs(ToyKit.CurrentModel:GetDescendants()) do if p:IsA("BasePart") then VehicleState.SavedItemTransparency[p]=p.Transparency; p.Transparency=1 end end
	end
	hum.WalkSpeed=0; hum.JumpPower=0; hum.JumpHeight=0; hum.AutoRotate=false; hum.PlatformStand=true
	root.AssemblyLinearVelocity=Vector3.zero; root.AssemblyAngularVelocity=Vector3.zero
	root.CFrame=base*CFrame.new(0,driveSpec.RideHeight,driveSpec.RideZ)
	pcall(function() hum:ChangeState(Enum.HumanoidStateType.Physics) end)
	createVehicleControlGui(def.Id)
	return true
end

local function vehicleImpact(position, color)
	toyBurst(position,color,32,1.35)
	local ring=toyPart(ensureToyFX(),"VehicleImpactRing",Vector3.new(.10,.10,.10),CFrame.new(position),color,"Ball",Enum.Material.Neon,true,.15)
	TweenService:Create(ring,TweenInfo.new(.38,Enum.EasingStyle.Quad,Enum.EasingDirection.Out),{Size=Vector3.new(2.6,.08,2.6),Transparency=1}):Play()
	Debris:AddItem(ring,.45)
end

local function updateVehicle(dt)
	if not VehicleState.Active then return end
	local model=VehicleState.Model; local root=VehicleState.SavedRoot; local hum=VehicleState.SavedHumanoid and VehicleState.SavedHumanoid
	if not model or not model.Parent or not root or not root.Parent or not hum or not hum.Parent or hum.Health<=0 then exitVehicle("Veículo encerrado porque o personagem não está disponível."); return end
	local spec=VehicleState.DriveSpec
	local input=VehicleState.Input
	local throttle=(input.Throttle and 1 or 0) - (input.Brake and 1 or 0)
	local maxForward=spec.MaxSpeed; local maxReverse=spec.ReverseSpeed
	if input.Handbrake then
		VehicleState.Speed=VehicleState.Speed*math.max(0,1-dt*4.8)
	else
		if throttle>0 then VehicleState.Speed=math.min(maxForward,VehicleState.Speed+spec.Acceleration*dt)
		elseif throttle<0 then
			if VehicleState.Speed>1.5 then VehicleState.Speed=math.max(0,VehicleState.Speed-spec.BrakeRate*dt)
			else VehicleState.Speed=math.max(-maxReverse,VehicleState.Speed-spec.Acceleration*.62*dt) end
		else
			local drag=1.15+(math.abs(VehicleState.Speed)*.035)
			if VehicleState.Speed>0 then VehicleState.Speed=math.max(0,VehicleState.Speed-drag*dt) elseif VehicleState.Speed<0 then VehicleState.Speed=math.min(0,VehicleState.Speed+drag*dt) end
		end
	end
	local steering=(input.Right and 1 or 0)-(input.Left and 1 or 0)
	local speedFactor=math.clamp(math.abs(VehicleState.Speed)/6,.18,1)
	VehicleState.Steer=lerp(VehicleState.Steer,steering,.14)
	local turnRate=spec.TurnRate*speedFactor
	if VehicleState.Speed<-.2 then turnRate=-turnRate end
	VehicleState.Yaw-=VehicleState.Steer*turnRate*dt
	if VehicleState.Kind=="mini_bike" then
		-- Lean visuals are applied to the model orientation without rolling the driver's root.
		VehicleState.Lean=lerp(VehicleState.Lean or 0,-VehicleState.Steer*math.rad(14)*speedFactor,.13)
	else
		VehicleState.Lean=lerp(VehicleState.Lean or 0,-VehicleState.Steer*math.rad(4)*speedFactor,.12)
	end
	local yawCF=CFrame.Angles(0,VehicleState.Yaw,0)
	local forward=yawCF.LookVector
	local oldPos=Vector3.new(root.Position.X,VehicleState.Y,root.Position.Z)
	local nextX=oldPos.X+forward.X*VehicleState.Speed*dt
	local nextZ=oldPos.Z+forward.Z*VehicleState.Speed*dt
	local params=vehicleRayParams()
	local sweepOrigin=oldPos+Vector3.new(0,1.05,0)
	local nextDelta=Vector3.new(nextX-oldPos.X,0,nextZ-oldPos.Z)
	if nextDelta.Magnitude>.02 then
		local frontHit=Workspace:Raycast(sweepOrigin,forward*(math.min(math.abs(VehicleState.Speed)*dt+spec.Length*.33, spec.Length*.58)),params)
		local sideHit=Workspace:Raycast(sweepOrigin+Vector3.new(0,.55,0),forward*(math.min(math.abs(VehicleState.Speed)*dt+spec.Length*.25,spec.Length*.52)),params)
		if frontHit or sideHit then
			local hit=frontHit or sideHit
			VehicleState.Speed=VehicleState.Speed>0 and 0 or VehicleState.Speed*.15
			nextX=oldPos.X; nextZ=oldPos.Z
			if os.clock()-VehicleState.StatusTime>.45 then vehicleImpact(hit.Position,VehicleState.Kind=="mini_bike" and Color3.fromRGB(75,180,255) or Color3.fromRGB(255,155,65)); VehicleState.StatusTime=os.clock() end
		end
	end
	local targetGround=Workspace:Raycast(Vector3.new(nextX,oldPos.Y+7,nextZ),Vector3.new(0,-18,0),params)
	if targetGround and targetGround.Normal.Y>.25 then
		local targetY=targetGround.Position.Y
		VehicleState.Y=lerp(VehicleState.Y,targetY,math.clamp(dt*18,0,1))
		VehicleState.VerticalSpeed=0
	else
		VehicleState.VerticalSpeed=math.max(-38,VehicleState.VerticalSpeed-Workspace.Gravity*dt)
		VehicleState.Y+=VehicleState.VerticalSpeed*dt
	end
	VehicleState.WheelRotation += (VehicleState.Speed/math.max(.3,VehicleState.Kind=="mini_bike" and .72 or .76))*dt
	local bodyCF=CFrame.new(nextX,VehicleState.Y,nextZ)*yawCF*CFrame.Angles(0,0,VehicleState.Lean or 0)
	for _,entry in ipairs(VehicleState.BodyParts) do
		if entry.Part and entry.Part.Parent then entry.Part.CFrame=bodyCF*entry.LocalCF end
	end
	for _,wheel in ipairs(VehicleState.Wheels) do
		local steer=wheel.IsFront and -VehicleState.Steer*.36 or 0
		local wheelCF=bodyCF*wheel.CenterCF*CFrame.Angles(0,steer,0)*CFrame.Angles(VehicleState.WheelRotation,0,0)
		for _,entry in ipairs(wheel.Parts) do if entry.Part and entry.Part.Parent then entry.Part.CFrame=wheelCF*entry.Rel end end
	end
	-- The rider follows the vehicle seat. Leaving restores the original humanoid properties.
	local rideCF=CFrame.new(nextX,VehicleState.Y,nextZ)*yawCF*CFrame.new(0,spec.RideHeight,spec.RideZ)
	root.CFrame=rideCF
	root.AssemblyLinearVelocity=Vector3.zero; root.AssemblyAngularVelocity=Vector3.zero
	if VehicleState.SpeedLabel then
		local kmh=math.floor(math.abs(VehicleState.Speed)*3.6+0.5)
		VehicleState.SpeedLabel.Text=string.format("%s  •  %d km/h\nACELERE • VIRE • FREIE",VehicleState.Kind=="mini_bike" and "MOTO" or "CARRO",kmh)
	end
	if os.clock()-VehicleState.StatusTime>.35 and ToyKit.StatusLabel then
		local kmh=math.floor(math.abs(VehicleState.Speed)*3.6+0.5)
		ToyKit.StatusLabel.Text=string.format("%s • %d km/h • acelere, freie e vire",VehicleState.Kind=="mini_bike" and "MOTO" or "CARRO",kmh)
	end
end

local function spawnMiniVehicle(def)
	if VehicleState.Active then exitVehicle("Veículo guardado."); return end
	local direction=Workspace.CurrentCamera and Workspace.CurrentCamera.CFrame.LookVector or (State.Root and State.Root.CFrame.LookVector) or Vector3.new(0,0,-1)
	local flat=Vector3.new(direction.X,0,direction.Z)
	if flat.Magnitude<.05 then flat=Vector3.new(0,0,-1) else flat=flat.Unit end
	local root=State.Root
	if not root then setVehicleHudStatus("Não foi possível invocar o veículo sem personagem ativo."); return end
	local start=root.Position+flat*(def.Id=="mini_bike" and 3.4 or 4.4)
	local ray=Workspace:Raycast(start+Vector3.new(0,8,0),Vector3.new(0,-22,0),vehicleRayParams())
	if ray then start=Vector3.new(start.X,ray.Position.Y,start.Z) end
	local ok=createVehicle(def,start,flat)
	if not ok then setVehicleHudStatus("Falha ao criar o veículo; personagem restaurado."); return end
	local loop=RunService.Heartbeat:Connect(function(dt)
		local okStep,err=xpcall(updateVehicle,debug.traceback,math.min(dt,.05))
		if not okStep then warn(TAG.." erro no veículo: "..tostring(err)); exitVehicle("O veículo foi parado após um erro.") end
	end)
	VehicleState.Loop=loop
	table.insert(ToyKit.Connections,loop)
	-- Keyboard support in addition to mobile touch controls.
	table.insert(VehicleState.Connections,UserInputService.InputBegan:Connect(function(input,processed)
		if processed or not VehicleState.Active then return end
		if input.KeyCode==Enum.KeyCode.W or input.KeyCode==Enum.KeyCode.Up then VehicleState.Input.Throttle=true
		elseif input.KeyCode==Enum.KeyCode.S or input.KeyCode==Enum.KeyCode.Down then VehicleState.Input.Brake=true
		elseif input.KeyCode==Enum.KeyCode.A or input.KeyCode==Enum.KeyCode.Left then VehicleState.Input.Left=true
		elseif input.KeyCode==Enum.KeyCode.D or input.KeyCode==Enum.KeyCode.Right then VehicleState.Input.Right=true
		elseif input.KeyCode==Enum.KeyCode.Space then VehicleState.Input.Handbrake=true
		elseif input.KeyCode==Enum.KeyCode.Escape then exitVehicle("Você saiu do veículo.") end
	end))
	table.insert(VehicleState.Connections,UserInputService.InputEnded:Connect(function(input,processed)
		if input.KeyCode==Enum.KeyCode.W or input.KeyCode==Enum.KeyCode.Up then VehicleState.Input.Throttle=false
		elseif input.KeyCode==Enum.KeyCode.S or input.KeyCode==Enum.KeyCode.Down then VehicleState.Input.Brake=false
		elseif input.KeyCode==Enum.KeyCode.A or input.KeyCode==Enum.KeyCode.Left then VehicleState.Input.Left=false
		elseif input.KeyCode==Enum.KeyCode.D or input.KeyCode==Enum.KeyCode.Right then VehicleState.Input.Right=false
		elseif input.KeyCode==Enum.KeyCode.Space then VehicleState.Input.Handbrake=false end
	end))
	setVehicleHudStatus(def.Id=="mini_bike" and "Motocicleta pronta: acelere, incline nas curvas, freie ou toque SAIR." or "Carro pronto: acelere, vire, freie e use DERRAPAR para reduzir a velocidade.")
	if ToyKit.UseButton then ToyKit.UseButton.Text="VEÍCULO ATIVO\nSAIR PELO HUD" end
end

local function useToyInstant()
	local def = ToyKit.CurrentId and ITEM_BY_ID[ToyKit.CurrentId]
	if not def or not ToyKit.CurrentModel or not ToyKit.CurrentModel.Parent then
		ToyKit.Status = "Pegue um brinquedo no menu primeiro."
		if ToyKit.StatusLabel then ToyKit.StatusLabel.Text = ToyKit.Status end
		return
	end
	local root = State.Root
	local hand = State.Rig and State.Rig.Model and State.Rig.Model:FindFirstChild("MF_HandR")
	local origin = (hand and hand.Position) or getToyOrigin()
	local direction, target, result = getToyAim(75)
	local action = def.Action
	if action == "laser" then
		launchToyProjectile(def,"spark",{Color=def.Color,Size=Vector3.new(.12,.12,.75),Speed=105,Life=1.1})
	elseif action == "saber" then
		local blade = ToyKit.CurrentModel:FindFirstChild("Toy_ToyBlade")
		if blade and blade.Transparency > .5 then
			blade.Transparency = 0
			ToyKit.Status = "Sabre ligado. Toque em USAR novamente para fazer um corte; guarde o item para desligá-lo."
			toyBurst(origin,def.Color,7,.6)
		elseif blade then
			toyBeam(origin,target,def.Color,.16,.22)
			toyBurst(target,def.Color,8,.7)
		else
			toyBeam(origin,target,def.Color,.20,.22); toyBurst(target,def.Color,8,.7)
		end
	elseif action == "ball" then
		launchToyProjectile(def,"confetti",{Color=def.Color,Size=Vector3.new(.38,.38,.38),Shape="Ball",Speed=40,Gravity=34,Lift=7,Life=3.5,Bounce=true,BounceLift=9,Material=Enum.Material.SmoothPlastic})
	elseif action == "tennis" then
		launchToyProjectile(def,"spark",{Color=def.Color,Size=Vector3.new(.17,.17,.17),Shape="Ball",Speed=80,Gravity=15,Lift=3,Life=1.8,Bounce=true,Material=Enum.Material.Neon})
	elseif action == "vehicle" then
		spawnMiniVehicle(def)
	elseif action == "trick" then
		local pos = root and root.Position or origin
		toyBurst(pos+Vector3.new(0,-1.2,0),def.Color,15,.7)
		for i=1,3 do toyBeam(pos+Vector3.new(-.5+i*.5,-1.05,0),pos+Vector3.new(-.5+i*.5,-1.4,-1.1),def.Color,.045,.45) end
		ToyKit.Status = "Manobra visual! Movimento do personagem intacto."
	elseif action == "boomerang" then
		launchToyProjectile(def,"spark",{Color=def.Color,Size=Vector3.new(.48,.08,.18),Shape="Block",Speed=48,Lift=5,Gravity=0,Life=1.8,Returning=true})
	elseif action == "grapple" then
		toyBeam(origin,target,def.Color,.045,.85); toyBurst(target,def.Color,6,.7)
		ToyKit.Status = "Cabo visual lançado; sem puxar o personagem."
	elseif action == "paint" then
		launchToyProjectile(def,"paint",{Color=def.Color,Size=Vector3.new(.24,.24,.24),Shape="Ball",Speed=66,Gravity=8,Lift=2,Life=1.7})
	elseif action == "swing" then
		toyBeam(origin,target,def.Color,.10,.18); impactVisual(target,result and result.Normal or Vector3.yAxis,def.Color,"swing")
	elseif action == "magic" then
		launchToyProjectile(def,"spark",{Color=def.Color,Size=Vector3.new(.24,.24,.24),Shape="Ball",Speed=55,Lift=9,Gravity=2,Life=2.0})
	elseif action == "flash" then
		local flash=toyPart(ensureToyFX(),"CameraFlash",Vector3.new(.15,.15,.15),CFrame.new(origin),Color3.new(1,1,1),"Ball",Enum.Material.Neon,true,0)
		local light=toyMake("PointLight",{Color=Color3.new(1,1,1),Brightness=8,Range=22},flash)
		toyBurst(origin,Color3.new(1,1,1),14,1.2); Debris:AddItem(flash,.18)
	elseif action == "pulse" then
		toyBurst(origin,def.Color,14,1.2); toyBeam(origin,target,def.Color,.06,.6)
	elseif action == "music" or action == "drum" then
		for i=1,4 do toyBurst(origin+Vector3.new((i-2.5)*.28,.35+i*.12,0),Color3.fromHSV((i*.18)%1,.8,1),5,.6) end
	elseif action == "yoyo" then
		launchToyProjectile(def,"spark",{Color=def.Color,Size=Vector3.new(.22,.22,.22),Shape="Ball",Speed=45,Lift=5,Gravity=4,Life=1.8,Returning=true})
	elseif action == "water" then
		launchToyProjectile(def,"water",{Color=def.Color,Size=Vector3.new(.11,.16,.11),Shape="Ball",Speed=65,Lift=2,Gravity=11,Life=1.3})
	elseif action == "freeze" then
		launchToyProjectile(def,"freeze",{Color=def.Color,Size=Vector3.new(.16,.42,.16),Shape="Block",Speed=72,Lift=1,Gravity=0,Life=1.3})
	elseif action == "firework" then
		launchToyProjectile(def,"firework",{Color=def.Color,Size=Vector3.new(.14,.50,.14),Shape="Cylinder",Speed=38,Lift=15,Gravity=-2,Life=1.4})
	elseif action == "confetti" then
		toyBurst(target,Color3.fromHSV(math.random(),.8,1),20,1.15); toyBeam(origin,target,def.Color,.045,.30)
	elseif action == "slingshot" then
		launchToyProjectile(def,"spark",{Color=def.Color,Size=Vector3.new(.20,.20,.20),Shape="Ball",Speed=85,Lift=3,Gravity=16,Life=2})
	elseif action == "shield" then
		local pos = root and root.Position or origin
		for i=1,10 do local angle=(i/10)*math.pi*2; local p=pos+Vector3.new(math.cos(angle)*1.25,1+math.sin(angle*2)*.25,math.sin(angle)*1.25); toyBurst(p,def.Color,2,.45) end
		ToyKit.Status = "Escudo visual ativado; não altera dano ou colisões."
	elseif action == "jetpack" then
		local pos = root and root.Position or origin
		toyBurst(pos+Vector3.new(-.25,-.7,.4),Color3.fromRGB(255,120,45),8,.8); toyBurst(pos+Vector3.new(.25,-.7,.4),Color3.fromRGB(80,190,255),8,.8)
		ToyKit.Status = "Jatos decorativos ativados; nenhum impulso foi aplicado."
	elseif action == "wings" then
		local pos = root and root.Position or origin
		toyBurst(pos+Vector3.new(-.8,.5,.3),def.Color,9,.7); toyBurst(pos+Vector3.new(.8,.5,.3),def.Color,9,.7)
	elseif action == "fishing" then
		local bobber=target+Vector3.new(0,.2,0); toyBeam(origin,bobber,def.Color,.025,1.2); local b=toyPart(ensureToyFX(),"ToyBobber",Vector3.new(.16,.16,.16),CFrame.new(bobber),Color3.fromRGB(255,75,75),"Ball",Enum.Material.SmoothPlastic,true,0); TweenService:Create(b,TweenInfo.new(.6,Enum.EasingStyle.Sine,Enum.EasingDirection.InOut,2,true),{Position=bobber+Vector3.new(0,.28,0)}):Play(); Debris:AddItem(b,2.2)
	elseif action == "magnet" then
		local pos = target
		for i=1,6 do
			local orb=toyPart(ensureToyFX(),"ToyMagnetOrb",Vector3.new(.12,.12,.12),CFrame.new(pos+Vector3.new(math.random(-10,10)/10,math.random(0,10)/10,math.random(-10,10)/10)),def.Color,"Ball",Enum.Material.Neon,true,0)
			local home=root and root.Position or origin
			TweenService:Create(orb,TweenInfo.new(.8,Enum.EasingStyle.Quad,Enum.EasingDirection.In),{Position=home,Transparency=1}):Play(); Debris:AddItem(orb,.9)
		end
	elseif action == "rocket" then
		launchToyProjectile(def,"firework",{Color=def.Color,Size=Vector3.new(.18,.50,.18),Shape="Cylinder",Speed=24,Lift=25,Gravity=-1,Life=1.4})
	elseif action == "potion" then
		launchToyProjectile(def,"confetti",{Color=def.Color,Size=Vector3.new(.27,.35,.27),Shape="Ball",Speed=44,Lift=4,Gravity=24,Life=2.1,Bounce=true,BounceLift=5})
	elseif action == "bubbles" then
		for i=1,8 do local p=origin+direction*1.5+Vector3.new(math.random(-10,10)/10,math.random(-5,8)/10,math.random(-10,10)/10); local bubble=toyPart(ensureToyFX(),"ToyBubble",Vector3.new(.12,.12,.12)*math.random(7,16)/10,CFrame.new(p),Color3.fromRGB(130,225,255),"Ball",Enum.Material.Glass,true,.30); TweenService:Create(bubble,TweenInfo.new(math.random(9,16)/10,Enum.EasingStyle.Sine,Enum.EasingDirection.Out),{Position=p+Vector3.new(0,math.random(2,5),0),Transparency=1,Size=bubble.Size*1.4}):Play(); Debris:AddItem(bubble,1.8) end
	elseif action == "lantern" then
		local lamp=ToyKit.CurrentModel:FindFirstChild("Toy_Lamp")
		local light=lamp and lamp:FindFirstChild("ToyLanternLight")
		if light then light.Enabled = not light.Enabled; ToyKit.Status = light.Enabled and "Lanterna ligada." or "Lanterna desligada." end
	elseif action == "telescope" then
		local marker=toyPart(ensureToyFX(),"TelescopeMarker",Vector3.new(.8,.05,.8),CFrame.new(target)*CFrame.Angles(math.rad(90),0,0),def.Color,"Cylinder",Enum.Material.Neon,true,.15)
		TweenService:Create(marker,TweenInfo.new(1),{Size=Vector3.new(1.6,.02,1.6),Transparency=1}):Play(); Debris:AddItem(marker,1.1)
	elseif action == "flag" then
		toyBurst(origin,def.Color,8,.6); toyBurst(origin+direction*.8,Color3.fromRGB(245,220,75),6,.5)
	elseif action == "drone" then
		launchToyProjectile(def,"spark",{Color=def.Color,Size=Vector3.new(.28,.18,.28),Shape="Ball",Speed=20,Lift=8,Gravity=0,Life=2.6,Returning=true})
	elseif action == "candy" then
		launchToyProjectile(def,"confetti",{Color=def.Color,Size=Vector3.new(.25,.38,.25),Shape="Ball",Speed=38,Lift=7,Gravity=17,Life=2.3,Bounce=true,BounceLift=4})
	elseif action == "portal" then
		impactVisual(target,result and result.Normal or Vector3.yAxis,def.Color,"portal")
	elseif action == "swing" then
		impactVisual(target,result and result.Normal or Vector3.yAxis,def.Color,"swing")
	end
	if ToyKit.StatusLabel then ToyKit.StatusLabel.Text = ToyKit.Status end
end


---------------------------------------------------------------------
-- V8 CINEMATIC INTERACTION LAYER
-- Every use now has anticipation, charge, an action beat, impact and recovery.
---------------------------------------------------------------------
local TOY_ACTION_PROFILE = {
	laser={Windup=.72,Recovery=.50,Pose=CFrame.Angles(math.rad(-8),0,math.rad(-8)),Charge=Color3.fromRGB(70,225,255)},
	saber={Windup=.62,Recovery=.66,Pose=CFrame.Angles(math.rad(-25),math.rad(-5),math.rad(-40)),Charge=Color3.fromRGB(60,225,255)},
	ball={Windup=.54,Recovery=.62,Pose=CFrame.Angles(math.rad(18),math.rad(-10),math.rad(30)),Charge=Color3.fromRGB(245,245,245)},
	tennis={Windup=.48,Recovery=.45,Pose=CFrame.Angles(math.rad(-15),0,math.rad(-35)),Charge=Color3.fromRGB(190,235,55)},
	vehicle={Windup=.92,Recovery=.44,Pose=CFrame.Angles(math.rad(-10),math.rad(6),math.rad(12)),Charge=Color3.fromRGB(255,185,60)},
	trick={Windup=.58,Recovery=.56,Pose=CFrame.Angles(math.rad(10),0,math.rad(18)),Charge=Color3.fromRGB(65,220,205)},
	boomerang={Windup=.46,Recovery=.55,Pose=CFrame.Angles(math.rad(10),math.rad(-10),math.rad(-28)),Charge=Color3.fromRGB(245,180,45)},
	grapple={Windup=.70,Recovery=.50,Pose=CFrame.Angles(math.rad(-8),math.rad(10),math.rad(-18)),Charge=Color3.fromRGB(175,180,190)},
	paint={Windup=.68,Recovery=.49,Pose=CFrame.Angles(math.rad(-8),0,math.rad(-12)),Charge=Color3.fromRGB(245,55,155)},
	swing={Windup=.52,Recovery=.58,Pose=CFrame.Angles(math.rad(-30),math.rad(-8),math.rad(-48)),Charge=Color3.fromRGB(255,115,95)},
	magic={Windup=.92,Recovery=.62,Pose=CFrame.Angles(math.rad(10),0,math.rad(22)),Charge=Color3.fromRGB(195,95,255)},
	flash={Windup=.70,Recovery=.58,Pose=CFrame.Angles(math.rad(-10),0,math.rad(-10)),Charge=Color3.fromRGB(255,255,235)},
	pulse={Windup=.60,Recovery=.52,Pose=CFrame.Angles(math.rad(-5),0,math.rad(-20)),Charge=Color3.fromRGB(70,220,175)},
	music={Windup=.48,Recovery=.75,Pose=CFrame.Angles(math.rad(15),math.rad(8),math.rad(-24)),Charge=Color3.fromRGB(220,65,70)},
	yoyo={Windup=.54,Recovery=.62,Pose=CFrame.Angles(math.rad(12),0,math.rad(-20)),Charge=Color3.fromRGB(80,185,255)},
	water={Windup=.58,Recovery=.42,Pose=CFrame.Angles(math.rad(-4),0,math.rad(-10)),Charge=Color3.fromRGB(50,165,255)},
	freeze={Windup=.82,Recovery=.55,Pose=CFrame.Angles(math.rad(-4),0,math.rad(-10)),Charge=Color3.fromRGB(130,235,255)},
	firework={Windup=1.03,Recovery=.72,Pose=CFrame.Angles(math.rad(-18),0,math.rad(-10)),Charge=Color3.fromRGB(255,80,180)},
	confetti={Windup=.70,Recovery=.72,Pose=CFrame.Angles(math.rad(-16),0,math.rad(-20)),Charge=Color3.fromRGB(255,210,50)},
	slingshot={Windup=.78,Recovery=.56,Pose=CFrame.Angles(math.rad(10),0,math.rad(-40)),Charge=Color3.fromRGB(160,115,75)},
	shield={Windup=.78,Recovery=.76,Pose=CFrame.Angles(math.rad(-6),0,math.rad(8)),Charge=Color3.fromRGB(70,150,255)},
	jetpack={Windup=.58,Recovery=.45,Pose=CFrame.Angles(math.rad(10),0,0),Charge=Color3.fromRGB(255,120,50)},
	wings={Windup=.66,Recovery=.60,Pose=CFrame.Angles(math.rad(12),0,math.rad(10)),Charge=Color3.fromRGB(175,205,255)},
	fishing={Windup=.72,Recovery=.65,Pose=CFrame.Angles(math.rad(-10),0,math.rad(-28)),Charge=Color3.fromRGB(90,175,100)},
	magnet={Windup=.70,Recovery=.54,Pose=CFrame.Angles(math.rad(-5),0,math.rad(-12)),Charge=Color3.fromRGB(235,65,70)},
	rocket={Windup=.94,Recovery=.68,Pose=CFrame.Angles(math.rad(-18),0,math.rad(-8)),Charge=Color3.fromRGB(245,245,245)},
	potion={Windup=.66,Recovery=.58,Pose=CFrame.Angles(math.rad(10),0,math.rad(-20)),Charge=Color3.fromRGB(65,230,135)},
	bubbles={Windup=.48,Recovery=.52,Pose=CFrame.Angles(math.rad(-5),0,math.rad(-16)),Charge=Color3.fromRGB(120,230,255)},
	lantern={Windup=.42,Recovery=.40,Pose=CFrame.Angles(math.rad(-6),0,math.rad(-8)),Charge=Color3.fromRGB(255,225,125)},
	telescope={Windup=.76,Recovery=.58,Pose=CFrame.Angles(math.rad(-8),0,math.rad(-12)),Charge=Color3.fromRGB(90,145,210)},
	drum={Windup=.45,Recovery=.80,Pose=CFrame.Angles(math.rad(-20),0,math.rad(-28)),Charge=Color3.fromRGB(230,175,95)},
	flag={Windup=.46,Recovery=.72,Pose=CFrame.Angles(math.rad(15),0,math.rad(-26)),Charge=Color3.fromRGB(240,240,240)},
	drone={Windup=.65,Recovery=.48,Pose=CFrame.Angles(math.rad(-5),0,math.rad(-8)),Charge=Color3.fromRGB(120,210,245)},
	candy={Windup=.58,Recovery=.58,Pose=CFrame.Angles(math.rad(12),0,math.rad(-26)),Charge=Color3.fromRGB(255,95,170)},
	portal={Windup=1.05,Recovery=.76,Pose=CFrame.Angles(math.rad(8),0,math.rad(-15)),Charge=Color3.fromRGB(150,70,255)},
}
local DEFAULT_ACTION_PROFILE={Windup=.62,Recovery=.55,Pose=CFrame.Angles(math.rad(-8),0,math.rad(-12)),Charge=Color3.fromRGB(130,210,255)}

local function getToyHoldPosition()
	if ToyKit.CurrentModel and ToyKit.CurrentModel.PrimaryPart and ToyKit.CurrentModel.PrimaryPart.Parent then return ToyKit.CurrentModel.PrimaryPart.Position end
	local hand=State.Rig and State.Rig.Model and State.Rig.Model:FindFirstChild("MF_HandR")
	return hand and hand.Position or getToyOrigin()
end

local function createChargeSequence(def, duration, color)
	local fxModel=Instance.new("Model")
	fxModel.Name="V8ActionCharge_"..def.Id
	fxModel.Parent=ensureToyFX()
	local rings,orbs={},{}
	local function fixedPart(name,size)
		local p=toyPart(fxModel,name,size,CFrame.new(0,-1000,0),color,"Block",Enum.Material.Neon,true,.12)
		return p
	end
	for ring=1,3 do
		local count=20
		for i=1,count do
			local p=fixedPart("ActionRingSegment",Vector3.new(.055,.055,.17+ring*.025))
			table.insert(rings,{Part=p,Ring=ring,Index=i,Count=count})
		end
	end
	for i=1,16 do
		local p=fixedPart("ActionEnergyOrb",Vector3.new(.075,.075,.075))
		table.insert(orbs,{Part=p,Index=i,Count=16})
	end
	local started=os.clock(); local stopped=false; local conn
	conn=RunService.Heartbeat:Connect(function()
		if stopped or not fxModel.Parent then if conn then conn:Disconnect() end; return end
		local elapsed=os.clock()-started
		local alpha=math.clamp(elapsed/math.max(.1,duration),0,1)
		local center=getToyHoldPosition()+Vector3.new(0,.05,0)
		for _,entry in ipairs(rings) do
			local r=entry.Ring
			local angle=entry.Index/entry.Count*math.pi*2+elapsed*(r%2==0 and -4.7 or 3.5)*r
			local radius=(.28+r*.14)+alpha*(.35+r*.10)
			local y=(r-2)*(.10+alpha*.08)+math.sin(angle*2+elapsed*5)*.035
			local point=center+Vector3.new(math.cos(angle)*radius,y,math.sin(angle)*radius)
			local tangent=Vector3.new(-math.sin(angle),0,math.cos(angle))
			entry.Part.CFrame=CFrame.lookAt(point,point+tangent)
			entry.Part.Transparency=.16+alpha*.36
		end
		for _,entry in ipairs(orbs) do
			local angle=entry.Index/entry.Count*math.pi*2+elapsed*(5.1+(entry.Index%3))
			local r=.46+alpha*.58+math.sin(elapsed*6+entry.Index)*.06
			local point=center+Vector3.new(math.cos(angle)*r,math.sin(angle*1.8+elapsed*2)*(.22+alpha*.22),math.sin(angle)*r)
			entry.Part.CFrame=CFrame.new(point)*CFrame.Angles(elapsed*2,elapsed*3,elapsed)
			entry.Part.Transparency=.03+alpha*.35
		end
		if elapsed>=duration then
			for _,entry in ipairs(rings) do if entry.Part.Parent then entry.Part.Transparency=.65 end end
		end
	end)
	table.insert(ToyKit.Connections,conn)
	return function()
		stopped=true
		if conn then conn:Disconnect(); untrackToyConnection(conn) end
		if fxModel and fxModel.Parent then fxModel:Destroy() end
	end
end

local function richImpactPulse(def, position)
	local color=def.Color
	for ringIndex=1,3 do
		local ringModel=Instance.new("Model")
		ringModel.Name="V8ImpactWave_"..def.Id
		ringModel.Parent=ensureToyFX()
		local segments=24
		local y=.16+ringIndex*.045
		for i=1,segments do
			local a=i/segments*math.pi*2
			local p=toyPart(ringModel,"ImpactArc",Vector3.new(.055,.055,.20),CFrame.new(position+Vector3.new(math.cos(a)*.1,y,math.sin(a)*.1)),color,"Block",Enum.Material.Neon,true,.12)
			local startAngle=a
			local startRadius=.15
			task.spawn(function()
				local duration=.34+ringIndex*.08
				local began=os.clock()
				while p.Parent and os.clock()-began<duration do
					local t=(os.clock()-began)/duration
					local radius=startRadius+(1.05+ringIndex*.40)*t
					local angle=startAngle+ringIndex*.18*t
					local point=position+Vector3.new(math.cos(angle)*radius,y*(1-t*.45),math.sin(angle)*radius)
					p.CFrame=CFrame.lookAt(point,point+Vector3.new(-math.sin(angle),0,math.cos(angle)))
					p.Transparency=.10+t*.90
					task.wait()
				end
				if p.Parent then p:Destroy() end
			end)
		end
		Debris:AddItem(ringModel,.9)
	end
	toyBurst(position,color,24,1.18)
end

local function animateHeldToy(profile, windup, recovery)
	local joint=ToyKit.HandJoint
	local home=ToyKit.HandJointHome
	if not joint or not joint.Parent or not home then task.wait(windup); return end
	local pose=home*profile.Pose
	local first=TweenService:Create(joint,TweenInfo.new(math.max(.12,windup*.70),Enum.EasingStyle.Quint,Enum.EasingDirection.Out),{C0=pose})
	first:Play()
	local remain=math.max(0,windup-windup*.70)
	task.wait(remain)
	-- Hold the prepared pose for a readable anticipation beat.
	if joint.Parent then
		local hold=TweenService:Create(joint,TweenInfo.new(.08,Enum.EasingStyle.Linear),{C0=pose})
		hold:Play()
	end
end

local function useToy()
	if VehicleState and VehicleState.Active then
		setVehicleHudStatus("Você está dirigindo. Use SAIR no painel do veículo para desmontar.")
		return
	end
	local def=ToyKit.CurrentId and ITEM_BY_ID[ToyKit.CurrentId]
	if not def or not ToyKit.CurrentModel or not ToyKit.CurrentModel.Parent then
		ToyKit.Status="Pegue um item no menu primeiro."
		if ToyKit.StatusLabel then ToyKit.StatusLabel.Text=ToyKit.Status end
		return
	end
	if ToyKit.UseBusy then return end
	ToyKit.UseBusy=true
	local id=def.Id
	local profile=TOY_ACTION_PROFILE[def.Action] or DEFAULT_ACTION_PROFILE
	local windup=profile.Windup or .62
	local recovery=profile.Recovery or .55
	if def.Action=="saber" then
		local blade=ToyKit.CurrentModel:FindFirstChild("Toy_ToyBlade")
		if blade and blade.Transparency<.5 then windup=.48; recovery=.72 end
	end
	if ToyKit.UseButton then ToyKit.UseButton.Text="PREPARANDO...\n"..string.upper(def.Name:sub(1,12)); ToyKit.UseButton.BackgroundColor3=Color3.fromRGB(129,83,182) end
	ToyKit.Status="Preparando "..def.Name.."..."
	if ToyKit.StatusLabel then ToyKit.StatusLabel.Text=ToyKit.Status end
	local cancelCharge=createChargeSequence(def,windup,profile.Charge or def.Color)
	task.spawn(function()
		local currentJoint=ToyKit.HandJoint
		local home=ToyKit.HandJointHome
		local ok,err=xpcall(function()
			animateHeldToy(profile,windup,recovery)
			if cancelCharge then cancelCharge(); cancelCharge=nil end
			if not State.Character or not State.Character.Parent or ToyKit.CurrentId~=id or not ToyKit.CurrentModel or not ToyKit.CurrentModel.Parent then return end
			local impactPos=getToyHoldPosition()
			richImpactPulse(def,impactPos)
			-- Execute the real action after the visible charge/anticipation.
			useToyInstant()
			-- Recovery is intentionally readable: a second activation cannot spam the action.
			local joint=ToyKit.HandJoint
			local rest=ToyKit.HandJointHome
			if joint and joint.Parent and rest then
				local recover=TweenService:Create(joint,TweenInfo.new(math.max(.18,recovery*.65),Enum.EasingStyle.Back,Enum.EasingDirection.Out),{C0=rest})
				recover:Play()
			end
			task.wait(recovery)
		end,debug.traceback)
		if cancelCharge then pcall(cancelCharge) end
		if not ok then warn(TAG.." interação V8 falhou: "..tostring(err)) end
		if currentJoint and currentJoint.Parent and home then pcall(function() currentJoint.C0=home end) end
		ToyKit.UseBusy=false
		if ToyKit.UseButton then
			ToyKit.UseButton.BackgroundColor3=Color3.fromRGB(60,150,225)
			ToyKit.UseButton.Text=VehicleState.Active and "VEÍCULO ATIVO\nSAIR PELO HUD" or (ToyKit.CurrentId and ("USAR\n"..string.upper((ITEM_BY_ID[ToyKit.CurrentId].Name):sub(1,12))) or "USAR")
		end
	end)
end

local function buildToyInventoryUI()
	local pg = player:WaitForChild("PlayerGui")
	local old = pg:FindFirstChild("LegoToyboxUI")
	if old then old:Destroy() end
	local gui = toyMake("ScreenGui",{Name="LegoToyboxUI",ResetOnSpawn=false,DisplayOrder=70,IgnoreGuiInset=false,ZIndexBehavior=Enum.ZIndexBehavior.Sibling},pg)
	pcall(function() gui.ScreenInsets = Enum.ScreenInsets.DeviceSafeInsets end)
	ToyKit.Gui = gui
	local openButton = toyMake("TextButton",{Name="OpenToybox",AnchorPoint=Vector2.new(1,0),Position=UDim2.new(1,-10,0,12),Size=UDim2.new(0,126,0,38),BackgroundColor3=Color3.fromRGB(35,43,60),Text="🎒 BRINQUEDOS",TextColor3=Color3.fromRGB(255,225,90),Font=Enum.Font.GothamBold,TextSize=12,AutoButtonColor=true,ZIndex=10},gui)
	toyMake("UICorner",{CornerRadius=UDim.new(0,10)},openButton)
	toyMake("UIStroke",{Color=Color3.fromRGB(100,150,220),Thickness=1.4,Transparency=.12},openButton)
	local useButton = toyMake("TextButton",{Name="UseEquippedToy",AnchorPoint=Vector2.new(1,1),Position=UDim2.new(1,-12,1,-345),Size=UDim2.new(0,78,0,54),BackgroundColor3=Color3.fromRGB(60,150,225),Text="USAR",TextColor3=Color3.new(1,1,1),Font=Enum.Font.GothamBold,TextSize=10,TextWrapped=true,Visible=false,ZIndex=10},gui)
	toyMake("UICorner",{CornerRadius=UDim.new(0,15)},useButton)
	toyMake("UIStroke",{Color=Color3.fromRGB(230,245,255),Thickness=1.5,Transparency=.25},useButton)
	local panel = toyMake("Frame",{Name="InventoryPanel",AnchorPoint=Vector2.new(.5,.5),Position=UDim2.fromScale(.5,.5),Size=UDim2.new(.88,0,.72,0),BackgroundColor3=Color3.fromRGB(23,27,36),BackgroundTransparency=.04,Visible=false,ZIndex=20},gui)
	panel.SizeConstraint = Enum.SizeConstraint.RelativeXY
	toyMake("UISizeConstraint",{MinSize=Vector2.new(280,350),MaxSize=Vector2.new(510,650)},panel)
	toyMake("UICorner",{CornerRadius=UDim.new(0,14)},panel)
	toyMake("UIStroke",{Color=Color3.fromRGB(87,130,185),Thickness=1.7,Transparency=.1},panel)
	toyMake("UIPadding",{PaddingTop=UDim.new(0,8),PaddingBottom=UDim.new(0,8),PaddingLeft=UDim.new(0,9),PaddingRight=UDim.new(0,9)},panel)
	local title = toyMake("TextLabel",{Name="Title",Size=UDim2.new(1,-44,0,30),BackgroundTransparency=1,Text="LEGO TOYBOX  •  41 ITENS",TextColor3=Color3.fromRGB(255,220,75),Font=Enum.Font.GothamBold,TextSize=14,TextXAlignment=Enum.TextXAlignment.Left,ZIndex=21},panel)
	local close = toyMake("TextButton",{Name="Close",AnchorPoint=Vector2.new(1,0),Position=UDim2.new(1,0,0,0),Size=UDim2.new(0,38,0,30),BackgroundColor3=Color3.fromRGB(75,80,92),Text="✕",TextColor3=Color3.new(1,1,1),Font=Enum.Font.GothamBold,TextSize=16,ZIndex=22},panel)
	toyMake("UICorner",{CornerRadius=UDim.new(0,8)},close)
	local categories = toyMake("ScrollingFrame",{Name="Categories",Position=UDim2.new(0,0,0,35),Size=UDim2.new(1,0,0,32),BackgroundTransparency=1,BorderSizePixel=0,ScrollBarThickness=2,ScrollingDirection=Enum.ScrollingDirection.X,CanvasSize=UDim2.new(0,0,0,0),AutomaticCanvasSize=Enum.AutomaticSize.X,ZIndex=21},panel)
	local categoryLayout = toyMake("UIListLayout",{FillDirection=Enum.FillDirection.Horizontal,Padding=UDim.new(0,5),SortOrder=Enum.SortOrder.LayoutOrder},categories)
	local categoryNames = {"TODOS","FERRAMENTAS","ESPORTES","VEÍCULOS","CRIATIVO","ACESSÓRIOS"}
	local categoryButtons = {}
	for i, name in ipairs(categoryNames) do
		local button = toyMake("TextButton",{Name="Cat_"..i,Size=UDim2.new(0,92,1,0),BackgroundColor3=Color3.fromRGB(55,65,83),Text=name,TextColor3=Color3.fromRGB(235,240,250),Font=Enum.Font.GothamBold,TextSize=10,LayoutOrder=i,ZIndex=22},categories)
		toyMake("UICorner",{CornerRadius=UDim.new(0,7)},button)
		categoryButtons[name]=button
	end
	local search = toyMake("TextBox",{Name="Search",Position=UDim2.new(0,0,0,72),Size=UDim2.new(1,0,0,34),BackgroundColor3=Color3.fromRGB(42,47,60),Text="",PlaceholderText="Buscar brinquedo...",PlaceholderColor3=Color3.fromRGB(165,172,185),TextColor3=Color3.new(1,1,1),Font=Enum.Font.Gotham,TextSize=12,ClearTextOnFocus=false,ZIndex=21},panel)
	toyMake("UICorner",{CornerRadius=UDim.new(0,8)},search)
	local list = toyMake("ScrollingFrame",{Name="ToyList",Position=UDim2.new(0,0,0,112),Size=UDim2.new(1,0,1,-166),BackgroundColor3=Color3.fromRGB(30,34,45),BackgroundTransparency=.15,BorderSizePixel=0,ScrollBarThickness=5,CanvasSize=UDim2.new(0,0,0,0),ZIndex=21},panel)
	toyMake("UICorner",{CornerRadius=UDim.new(0,9)},list)
	toyMake("UIPadding",{PaddingTop=UDim.new(0,6),PaddingBottom=UDim.new(0,6),PaddingLeft=UDim.new(0,6),PaddingRight=UDim.new(0,6)},list)
	local grid = toyMake("UIGridLayout",{CellSize=UDim2.new(.5,-5,0,72),CellPadding=UDim2.new(0,6,0,6),SortOrder=Enum.SortOrder.LayoutOrder},list)
	local status = toyMake("TextLabel",{Name="Status",AnchorPoint=Vector2.new(0,1),Position=UDim2.new(0,0,1,0),Size=UDim2.new(1,0,0,45),BackgroundTransparency=1,Text=ToyKit.Status,TextColor3=Color3.fromRGB(200,220,245),Font=Enum.Font.Gotham,TextSize=11,TextWrapped=true,TextXAlignment=Enum.TextXAlignment.Left,ZIndex=21},panel)
	ToyKit.UseButton = useButton
	ToyKit.StatusLabel = status
	ToyKit.CardHolder = list
	ToyKit.CategoryButtons = categoryButtons
	ToyKit.CardButtons = {}

	local function createCard(def, order)
		local card = toyMake("TextButton",{Name="Item_"..def.Id,Size=UDim2.new(.5,-5,0,72),BackgroundColor3=Color3.fromRGB(49,55,70),Text="",AutoButtonColor=true,LayoutOrder=order,ZIndex=22},list)
		toyMake("UICorner",{CornerRadius=UDim.new(0,9)},card)
		local stroke = toyMake("UIStroke",{Color=Color3.fromRGB(86,96,119),Thickness=1,Transparency=.2},card)
		local icon = toyMake("TextLabel",{Position=UDim2.new(0,5,0,3),Size=UDim2.new(0,33,1,-6),BackgroundTransparency=1,Text=def.Icon,TextSize=20,Font=Enum.Font.Gotham,ZIndex=23},card)
		local name = toyMake("TextLabel",{Position=UDim2.new(0,40,0,5),Size=UDim2.new(1,-45,0,30),BackgroundTransparency=1,Text=def.Name,TextColor3=Color3.fromRGB(245,247,250),TextWrapped=true,TextSize=10,Font=Enum.Font.GothamBold,TextXAlignment=Enum.TextXAlignment.Left,ZIndex=23},card)
		local use = toyMake("TextLabel",{Position=UDim2.new(0,40,1,-23),Size=UDim2.new(1,-45,0,18),BackgroundTransparency=1,Text="TOQUE PARA PEGAR",TextColor3=Color3.fromRGB(185,200,220),TextSize=9,Font=Enum.Font.Gotham,ZIndex=23,TextXAlignment=Enum.TextXAlignment.Left},card)
		card.Activated:Connect(function() equipToy(def.Id) end)
		ToyKit.CardButtons[def.Id] = {Button=card, Stroke=stroke, Hint=use, Item=def}
	end
	for i, def in ipairs(TOY_ITEMS) do createCard(def,i) end
	local function refreshCards()
		local visibleCount = 0
		local searchText = string.lower(ToyKit.Search or "")
		for _, def in ipairs(TOY_ITEMS) do
			local row = ToyKit.CardButtons[def.Id]
			local categoryOk = ToyKit.Category == "TODOS" or def.Cat == ToyKit.Category
			local searchOk = searchText == "" or string.find(string.lower(def.Name),searchText,1,true) or string.find(string.lower(def.Desc),searchText,1,true)
			local visible = categoryOk and searchOk
			row.Button.Visible = visible
			if visible then visibleCount += 1; row.Button.LayoutOrder = visibleCount end
			local equipped = ToyKit.CurrentId == def.Id
			row.Button.BackgroundColor3 = equipped and Color3.fromRGB(41,91,79) or Color3.fromRGB(49,55,70)
			row.Stroke.Color = equipped and Color3.fromRGB(105,245,165) or Color3.fromRGB(86,96,119)
			row.Stroke.Thickness = equipped and 2 or 1
			row.Hint.Text = equipped and "EQUIPADO • TOQUE PARA GUARDAR" or "TOQUE PARA PEGAR"
			row.Hint.TextColor3 = equipped and Color3.fromRGB(135,255,185) or Color3.fromRGB(185,200,220)
		end
		list.CanvasSize = UDim2.new(0,0,0,grid.AbsoluteContentSize.Y+12)
		title.Text = "LEGO TOYBOX  •  "..tostring(#TOY_ITEMS).." BRINQUEDOS"
	end
	ToyKit.refreshCards = refreshCards
	grid:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function() list.CanvasSize = UDim2.new(0,0,0,grid.AbsoluteContentSize.Y+12) end)
	for name, button in pairs(categoryButtons) do
		button.Activated:Connect(function()
			ToyKit.Category = name
			for category, b in pairs(categoryButtons) do b.BackgroundColor3 = (category == name) and Color3.fromRGB(70,115,180) or Color3.fromRGB(55,65,83) end
			refreshCards()
		end)
	end
	search:GetPropertyChangedSignal("Text"):Connect(function() ToyKit.Search = search.Text; refreshCards() end)
	openButton.Activated:Connect(function() panel.Visible = not panel.Visible end)
	close.Activated:Connect(function() panel.Visible = false end)
	useButton.Activated:Connect(useToy)
	categoryButtons.TODOS.BackgroundColor3 = Color3.fromRGB(70,115,180)
	refreshCards()
	return gui
end

local function clearToyEffects()
	if exitVehicle then exitVehicle("Efeitos locais limpos.") end
	for i = #ToyKit.Connections, 1, -1 do
		local conn = ToyKit.Connections[i]
		ToyKit.Connections[i] = nil
		pcall(function() conn:Disconnect() end)
	end
	if ToyKit.FXFolder then
		for _, child in ipairs(ToyKit.FXFolder:GetChildren()) do child:Destroy() end
	end
	ToyKit.ActiveProjectiles = 0
end

ToyKit.Gui = buildToyInventoryUI()


----------------------------------------------------------------------
-- Ligações de eventos
----------------------------------------------------------------------
local function onCharacterAdded(char)
	clearEquippedToy("Novo personagem: brinquedo guardado.")
	clearToyEffects()
	resetParkour(true)
	resetActionResources()
	clearDestruction()
	State.Token += 1
	local token = State.Token
	deactivateRig(false)
	State.Character, State.Humanoid, State.Root = nil, nil, nil
	if not State.Enabled then
		UI.setBuild("Desativada (avatar original)", C_NEUTRAL)
		return
	end
	task.spawn(runPipeline, token, false)
end

GlobalBin:Add(player.CharacterAdded:Connect(onCharacterAdded))
GlobalBin:Add(player.CharacterRemoving:Connect(function()
	clearEquippedToy("Personagem saindo: brinquedo guardado.")
	clearToyEffects()
	resetParkour(true)
	clearDestruction()
	State.Token += 1
	deactivateRig(false)
	UI.setInit("Aguardando personagem", C_WARN)
end))

GlobalBin:Add(UI.Toggle.Activated:Connect(function()
	State.Enabled = not State.Enabled
	State.Token += 1
	local token = State.Token
	refreshToggle()
	UI.setError(nil)
	if State.Enabled then
		task.spawn(runPipeline, token, false)
	else
		resetParkour(true)
		resetActionResources()
		clearDestruction()
		clearEquippedToy("Transformação desativada; brinquedo guardado.")
		clearToyEffects()
		deactivateRig(true)
		UI.setBuild("Desativada — avatar original restaurado", C_NEUTRAL)
		UI.setAnim("—", C_NEUTRAL)
	end
end))

GlobalBin:Add(UI.Outfit.Activated:Connect(function()
	clearEquippedToy("Troca de traje: brinquedo guardado.")
	State.OutfitIndex = State.OutfitIndex % #OUTFITS + 1
	refreshOutfit()
	if State.Enabled and State.Rig then
		State.Token += 1
		task.spawn(runPipeline, State.Token, true)
	end
end))

GlobalBin:Add(UI.Punch.Activated:Connect(startPunch))
GlobalBin:Add(UI.Gesture.Activated:Connect(startEmote))
GlobalBin:Add(UI.Slide.Activated:Connect(startSlide))
GlobalBin:Add(UI.Parkour.Activated:Connect(performParkour))
GlobalBin:Add(UI.Break.Activated:Connect(breakTarget))
GlobalBin:Add(UI.Dash.Activated:Connect(startDash))
GlobalBin:Add(UI.Roll.Activated:Connect(function() startRoll(false) end))
GlobalBin:Add(UI.Drop.Activated:Connect(dropFromLedge))
GlobalBin:Add(UserInputService.JumpRequest:Connect(onJumpRequest))
GlobalBin:Add(UserInputService.InputBegan:Connect(function(input, processed)
	if processed then return end
	if input.KeyCode == Enum.KeyCode.U then
		if ToyKit.Gui then
			local panel = ToyKit.Gui:FindFirstChild("InventoryPanel")
			if panel then panel.Visible = not panel.Visible end
		end
	elseif input.KeyCode == Enum.KeyCode.Q then
		useToy()
	end
end))

GlobalBin:Add(UserInputService.InputBegan:Connect(function(input, processed) -- atalhos para testar no PC
	if processed then return end
	if input.KeyCode == Enum.KeyCode.F then startPunch()
	elseif input.KeyCode == Enum.KeyCode.G then startEmote()
	elseif input.KeyCode == Enum.KeyCode.LeftControl then startSlide()
	elseif input.KeyCode == Enum.KeyCode.E then performParkour()
	elseif input.KeyCode == Enum.KeyCode.LeftShift then startDash()
	elseif input.KeyCode == Enum.KeyCode.X then startRoll(false) end
end))

script.Destroying:Connect(function()
	clearEquippedToy("Script encerrado.")
	clearToyEffects()
	Destruction.running = false
	resetAllDamageVisuals()
	resetParkour(true)
	clearDestruction()
	State.Token += 1
	deactivateRig(true)
	GlobalBin:Clean()
	if UI.Gui then UI.Gui:Destroy() end
end)

-- Início
startLocalSubsystem()
resetActionResources()
logLocalSelfChecks()
refreshToggle()
refreshOutfit()
UI.setInit("V8: aguardando personagem", C_WARN)
ToyKit.Status = "Toybox V8: " .. tostring(#TOY_ITEMS) .. " itens detalhados. Ações com carga/recuperação; carro e moto dirigíveis."
if ToyKit.StatusLabel then ToyKit.StatusLabel.Text = ToyKit.Status end
if player.Character then
	task.spawn(function() onCharacterAdded(player.Character) end)
end
if not State.Enabled then
	UI.setBuild("Desativada (avatar original)", C_NEUTRAL)
end
