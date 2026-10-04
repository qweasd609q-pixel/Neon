--// NEON MENU v9 — LocalScript
--// Положи в StarterPlayer > StarterPlayerScripts

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UIS = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local Lighting = game:GetService("Lighting")
local TextChatService = game:GetService("TextChatService")
local StarterGui = game:GetService("StarterGui")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local VirtualUser = game:GetService("VirtualUser")
local Stats = game:GetService("Stats")

local LP = Players.LocalPlayer
local Camera = workspace.CurrentCamera
local IS_TOUCH = UIS.TouchEnabled
local PHONE = IS_TOUCH and not UIS.KeyboardEnabled

-- очистка при повторном запуске
do
	local pg = LP:WaitForChild("PlayerGui")
	local old = pg:FindFirstChild("DesignerMenu")
	if old then old:Destroy() end
	for _, n in ipairs({ "NeonSteps", "NeonLaser" }) do
		local f = workspace:FindFirstChild(n)
		if f then f:Destroy() end
	end
	local c = LP.Character
	if c then
		for _, v in ipairs(c:GetDescendants()) do
			if v.Name:sub(1, 8) == "NeonAura" or v.Name:sub(1, 7) == "NeonRib" then v:Destroy() end
		end
	end
end

----------------------------------------------------------------
-- Настройки
----------------------------------------------------------------
local S = {
	Fly = false, FlySpeed = 50,
	Speed = false, WalkSpeed = 32,
	Noclip = false,
	Bhop = false, BhopSpeed = 40,
	Spin = false, SpinSpeed = 3,
	SuperJump = false, JumpPower = 200,
	Gravity = false, GravityVal = 100,
	-- Игрок
	Invis = false, AntiFling = true, AntiAFK = true, AntiAFKJump = false,
	-- Эффекты
	Aura = false, AuraSize = 6, AuraRate = 40,
	Steps = false, StepLife = 15,
	Ribbon = false,
	FXRainbow = false, FXColor = Color3.fromRGB(150, 90, 255),
	-- Лазер
	Laser = false, LaserAlways = false, LaserRange = 200, LaserWidth = 0.35,
	LaserRainbow = false, LaserColor = Color3.fromRGB(255, 40, 40),
	-- Танцы
	DanceSpeed = 1,
	-- ESP
	ESP = false, ESPColor = Color3.fromRGB(255, 70, 90),
	ESPBox = true, ESPSkel = false, ESPName = true, ESPDist = true, ESPHealth = true, ESPFill = true,
	-- Аим
	Aim = false, PhoneMode = PHONE, AutoAim = PHONE,
	AimNearest = true, NearRange = 300, UseFOVLimit = false,
	FOV = 150, AimPower = 60,
	AimHead = true, ShowFOV = true, TeamCheck = false, WallCheck = false,
	-- Триггер-бот
	Trigger = false, TriggerAlways = true, TriggerDelay = 80, TriggerRadius = 15,
	-- Чат
	ForceChat = false,
	-- Визуал
	Fullbright = false, ShowStats = false,
	ClockTime = math.floor(Lighting.ClockTime * 10) / 10,
	SkyTint = Color3.fromRGB(255, 255, 255),
	CamFOV = math.clamp(math.floor(Camera.FieldOfView + 0.5), 30, 120),
}

local WHITE = Color3.fromRGB(255, 255, 255)
local ACCENT = Color3.fromRGB(140, 90, 255)
local BG = Color3.fromRGB(18, 18, 26)
local PANEL = Color3.fromRGB(26, 26, 38)
local ITEM = Color3.fromRGB(34, 34, 50)
local TEXT = Color3.fromRGB(235, 235, 245)
local SUB = Color3.fromRGB(150, 150, 170)

local function new(class, props, parent)
	local o = Instance.new(class)
	for k, v in pairs(props) do o[k] = v end
	o.Parent = parent
	return o
end

----------------------------------------------------------------
-- НЕБО (логика)
----------------------------------------------------------------
local origLight = {
	ClockTime = Lighting.ClockTime, Ambient = Lighting.Ambient,
	OutdoorAmbient = Lighting.OutdoorAmbient,
}
local atmos = Lighting:FindFirstChildOfClass("Atmosphere")
local atmosCreated = false
local origAtmos = atmos and {
	Color = atmos.Color, Decay = atmos.Decay, Density = atmos.Density,
	Haze = atmos.Haze, Glare = atmos.Glare,
}
local sky = Lighting:FindFirstChildOfClass("Sky")
local origStars = sky and sky.StarCount

local oldTint = Lighting:FindFirstChild("MenuTint")
if oldTint then oldTint:Destroy() end
local tint = new("ColorCorrectionEffect", { Name = "MenuTint" }, Lighting)

local function getAtmos()
	if not atmos then
		atmos = new("Atmosphere", {}, Lighting)
		atmosCreated = true
	end
	return atmos
end

local SKY_PRESETS = {
	{ name = "☀ День", time = 14, amb = Color3.fromRGB(120, 130, 150), color = Color3.fromRGB(190, 210, 240), decay = Color3.fromRGB(120, 150, 200), density = 0.3, haze = 1, stars = 0 },
	{ name = "🌇 Закат", time = 18.2, amb = Color3.fromRGB(150, 100, 90), color = Color3.fromRGB(255, 170, 110), decay = Color3.fromRGB(255, 110, 80), density = 0.35, haze = 2, stars = 0 },
	{ name = "🌙 Ночь", time = 0, amb = Color3.fromRGB(40, 45, 80), color = Color3.fromRGB(40, 50, 90), decay = Color3.fromRGB(20, 25, 60), density = 0.3, haze = 1.5, stars = 3000 },
	{ name = "💜 Неон", time = 23, amb = Color3.fromRGB(110, 50, 160), color = Color3.fromRGB(180, 60, 255), decay = Color3.fromRGB(255, 40, 170), density = 0.4, haze = 3, stars = 3000 },
	{ name = "🌫 Туман", time = 12, amb = Color3.fromRGB(150, 150, 150), color = Color3.fromRGB(200, 200, 200), decay = Color3.fromRGB(180, 180, 180), density = 0.6, haze = 4, stars = 0 },
}

local timeSet

local function applySky(p)
	if timeSet then timeSet(p.time) else Lighting.ClockTime = p.time end
	Lighting.Ambient = p.amb
	Lighting.OutdoorAmbient = p.amb
	local a = getAtmos()
	a.Color, a.Decay, a.Density, a.Haze, a.Glare = p.color, p.decay, p.density, p.haze, 0
	if sky then sky.StarCount = p.stars end
end

local function resetSky()
	if timeSet then timeSet(origLight.ClockTime) else Lighting.ClockTime = origLight.ClockTime end
	Lighting.Ambient = origLight.Ambient
	Lighting.OutdoorAmbient = origLight.OutdoorAmbient
	if atmosCreated and atmos then
		atmos:Destroy(); atmos = nil; atmosCreated = false
	elseif atmos and origAtmos then
		for k, v in pairs(origAtmos) do atmos[k] = v end
	end
	if sky and origStars then sky.StarCount = origStars end
	tint.TintColor = WHITE
	S.SkyTint = WHITE
end

----------------------------------------------------------------
-- FULLBRIGHT и ГРАВИТАЦИЯ (логика)
----------------------------------------------------------------
local fbOrig
local function applyFullbright(on)
	if on then
		if not fbOrig then
			fbOrig = {
				Brightness = Lighting.Brightness, Ambient = Lighting.Ambient,
				OutdoorAmbient = Lighting.OutdoorAmbient, FogEnd = Lighting.FogEnd,
				FogStart = Lighting.FogStart, GlobalShadows = Lighting.GlobalShadows,
			}
		end
		Lighting.Brightness = 2
		Lighting.Ambient = Color3.fromRGB(190, 190, 190)
		Lighting.OutdoorAmbient = Color3.fromRGB(190, 190, 190)
		Lighting.FogStart = 0
		Lighting.FogEnd = 1e6
		Lighting.GlobalShadows = false
	elseif fbOrig then
		for k, v in pairs(fbOrig) do Lighting[k] = v end
		fbOrig = nil
	end
end

task.spawn(function()
	while true do
		task.wait(0.5)
		if S.Fullbright then pcall(applyFullbright, true) end
	end
end)

local origGravity = workspace.Gravity
local gravApplied = false
RunService.Heartbeat:Connect(function()
	if S.Gravity then
		workspace.Gravity = S.GravityVal
		gravApplied = true
	elseif gravApplied then
		workspace.Gravity = origGravity
		gravApplied = false
	end
end)

----------------------------------------------------------------
-- СКИН (логика)
----------------------------------------------------------------
local skinStatus
local function setStatus(t) if skinStatus then skinStatus.Text = t end end

local backup
local function saveBackup(char)
	if backup and backup.char == char then return end
	local bc = char:FindFirstChildOfClass("BodyColors")
	local sh = char:FindFirstChildOfClass("Shirt")
	local pa = char:FindFirstChildOfClass("Pants")
	local gr = char:FindFirstChildOfClass("ShirtGraphic")
	backup = {
		char = char,
		bc = bc and bc:Clone(), shirt = sh and sh:Clone(),
		pants = pa and pa:Clone(), gfx = gr and gr:Clone(),
	}
end

local function clearFX(char)
	for _, v in ipairs(char:GetChildren()) do
		if v.Name == "SkinFX" then v:Destroy() end
	end
end

local function attach(char, baseName, size, cf, color, material)
	local base = char:FindFirstChild(baseName)
	if not base then return end
	local p = new("Part", {
		Name = "SkinFX", Size = size, Color = color,
		Material = material or Enum.Material.SmoothPlastic,
		CanCollide = false, CanQuery = false, CanTouch = false, Massless = true,
	}, char)
	new("Weld", { Part0 = base, Part1 = p, C0 = cf }, p)
	return p
end

local function applyColors(char, c)
	for _, cls in ipairs({ "Shirt", "Pants", "ShirtGraphic" }) do
		for _, v in ipairs(char:GetChildren()) do
			if v:IsA(cls) then v:Destroy() end
		end
	end
	local bc = char:FindFirstChildOfClass("BodyColors") or new("BodyColors", {}, char)
	bc.HeadColor3, bc.TorsoColor3 = c.head, c.torso
	bc.LeftArmColor3, bc.RightArmColor3 = c.arms, c.arms
	bc.LeftLegColor3, bc.RightLegColor3 = c.legs, c.legs
end

local BLACK = Color3.fromRGB(15, 15, 18)

local SKINS = {
	{
		name = "🟢 Ниндзя",
		colors = { head = Color3.fromRGB(235, 190, 140), torso = Color3.fromRGB(50, 230, 40), arms = Color3.fromRGB(225, 175, 125), legs = Color3.fromRGB(50, 230, 40) },
		fx = function(char, hs, torso)
			attach(char, "Head", Vector3.new(hs.X * 1.1, hs.Y * 0.45, hs.Z * 1.1), CFrame.new(0, hs.Y * 0.55, 0), BLACK)
			attach(char, "Head", Vector3.new(hs.X * 1.05, hs.Y * 0.5, hs.Z * 1.05), CFrame.new(0, -hs.Y * 0.25, 0), BLACK)
			local cf = CFrame.new(0, 0.2, 0.65) * CFrame.Angles(0, 0, math.rad(40))
			attach(char, torso, Vector3.new(0.12, 2.8, 0.06), cf * CFrame.new(0, 0.5, 0), Color3.fromRGB(170, 170, 180), Enum.Material.Metal)
			attach(char, torso, Vector3.new(0.18, 0.8, 0.18), cf * CFrame.new(0, -1.4, 0), BLACK)
		end,
	},
	{
		name = "🧢 Кепка",
		colors = { head = Color3.fromRGB(25, 25, 28), torso = Color3.fromRGB(20, 20, 24), arms = Color3.fromRGB(20, 20, 24), legs = Color3.fromRGB(55, 60, 70) },
		fx = function(char, hs)
			attach(char, "Head", Vector3.new(hs.X * 1.1, hs.Y * 0.4, hs.Z * 1.1), CFrame.new(0, hs.Y * 0.5, 0), BLACK)
			attach(char, "Head", Vector3.new(hs.X * 0.9, 0.1, hs.Z * 0.6), CFrame.new(0, hs.Y * 0.4, -hs.Z * 0.75), BLACK)
			attach(char, "Head", Vector3.new(hs.X * 1.15, hs.Y * 0.9, hs.Z * 0.5), CFrame.new(0, 0, hs.Z * 0.45), BLACK)
		end,
	},
	{
		name = "🪖 Ушанка",
		colors = { head = Color3.fromRGB(25, 25, 28), torso = Color3.fromRGB(18, 18, 22), arms = Color3.fromRGB(18, 18, 22), legs = Color3.fromRGB(45, 48, 55) },
		fx = function(char, hs)
			attach(char, "Head", Vector3.new(hs.X * 1.2, hs.Y * 0.5, hs.Z * 1.2), CFrame.new(0, hs.Y * 0.55, 0), Color3.fromRGB(30, 30, 34))
			attach(char, "Head", Vector3.new(0.25, 0.25, 0.1), CFrame.new(0, hs.Y * 0.6, -hs.Z * 0.62), Color3.fromRGB(220, 40, 40), Enum.Material.Neon)
			attach(char, "Head", Vector3.new(hs.X * 1.2, hs.Y * 1.5, hs.Z * 0.5), CFrame.new(0, -hs.Y * 0.3, hs.Z * 0.5), BLACK)
		end,
	},
	{
		name = "🕶 Жилет",
		colors = { head = Color3.fromRGB(25, 25, 28), torso = Color3.fromRGB(45, 45, 50), arms = Color3.fromRGB(130, 130, 135), legs = Color3.fromRGB(25, 25, 30) },
		fx = function(char, hs)
			attach(char, "Head", Vector3.new(hs.X * 1.15, hs.Y * 0.5, hs.Z * 1.15), CFrame.new(0, hs.Y * 0.55, 0), BLACK)
			attach(char, "Head", Vector3.new(hs.X * 1.05, hs.Y * 0.4, hs.Z * 0.4), CFrame.new(0, hs.Y * 0.2, -hs.Z * 0.5), BLACK)
		end,
	},
}

local function applySkin(preset)
	local char = LP.Character
	if not char or not char:FindFirstChild("Head") then setStatus("Персонаж не найден") return end
	saveBackup(char)
	clearFX(char)
	applyColors(char, preset.colors)
	local torso = char:FindFirstChild("UpperTorso") and "UpperTorso" or "Torso"
	preset.fx(char, char.Head.Size, torso)
	setStatus("Скин применён: " .. preset.name)
end

local function resetSkin()
	local char = LP.Character
	if not char then return end
	clearFX(char)
	if backup and backup.char == char then
		for _, v in ipairs(char:GetChildren()) do
			if v:IsA("BodyColors") or v:IsA("Shirt") or v:IsA("Pants") or v:IsA("ShirtGraphic") then v:Destroy() end
		end
		for _, v in pairs({ backup.bc, backup.shirt, backup.pants, backup.gfx }) do
			if v then v:Clone().Parent = char end
		end
		setStatus("Скин сброшен")
	else
		setStatus("Скин не менялся")
	end
end

local function wearUser(text)
	task.spawn(function()
		local char = LP.Character
		local hum = char and char:FindFirstChildOfClass("Humanoid")
		if not hum then setStatus("Персонаж не найден") return end
		setStatus("Загрузка...")
		local id = tonumber(text)
		if not id then
			local ok, res = pcall(Players.GetUserIdFromNameAsync, Players, text)
			if not ok then setStatus("Ник не найден") return end
			id = res
		end
		local ok, desc = pcall(Players.GetHumanoidDescriptionFromUserId, Players, id)
		if not ok then setStatus("Не удалось получить аватар") return end
		clearFX(char)
		local ok2 = pcall(hum.ApplyDescription, hum, desc)
		setStatus(ok2 and "Аватар надет" or "Игра не разрешила сменить аватар")
	end)
end

----------------------------------------------------------------
-- НЕВИДИМОСТЬ (логика, только на твоём экране)
----------------------------------------------------------------
local invisOrig = setmetatable({}, { __mode = "k" })

local function applyInvis(on)
	local char = LP.Character
	if not char then return end
	for _, v in ipairs(char:GetDescendants()) do
		if (v:IsA("BasePart") or v:IsA("Decal")) and v.Name ~= "HumanoidRootPart" then
			if on then
				if invisOrig[v] == nil then invisOrig[v] = v.Transparency end
				v.Transparency = 1
			elseif invisOrig[v] ~= nil then
				v.Transparency = invisOrig[v]
				invisOrig[v] = nil
			end
		end
	end
	local hum = char:FindFirstChildOfClass("Humanoid")
	if hum then
		hum.DisplayDistanceType = on and Enum.HumanoidDisplayDistanceType.None
			or Enum.HumanoidDisplayDistanceType.Viewer
	end
end

task.spawn(function()
	while true do
		task.wait(0.5)
		if S.Invis then pcall(applyInvis, true) end
	end
end)

----------------------------------------------------------------
-- АНИМАЦИИ (логика)
----------------------------------------------------------------
local animStatusLabel
local function animStatus(t) if animStatusLabel then animStatusLabel.Text = t end end

local ANIMS = {
	{ name = "🥷 Ниндзя", idle = { 656117400, 656118341 }, walk = 656121766, run = 656118852, jump = 656117878, fall = 656115606, climb = 656114359 },
	{ name = "🦸 Супергерой", idle = { 616111295, 616113536 }, walk = 616122287, run = 616117076, jump = 616115533, fall = 616108001, climb = 616104706 },
	{ name = "🤖 Робот", idle = { 616088211, 616089559 }, walk = 616095330, run = 616091570, jump = 616090535, fall = 616087089, climb = 616086039 },
	{ name = "🧟 Зомби", idle = { 616158929, 616160636 }, walk = 616168032, run = 616163682, jump = 616161997, fall = 616157476, climb = 616156119 },
	{ name = "🎨 Мультяшная", idle = { 742637544, 742638445 }, walk = 742640026, run = 742638842, jump = 742637942, fall = 742637151, climb = 742636889 },
	{ name = "🧛 Вампир", idle = { 1083445855, 1083450166 }, walk = 1083473930, run = 1083462077, jump = 1083455352, fall = 1083443587, climb = 1083439238 },
	{ name = "🪄 Левитация", idle = { 616006778, 616008087 }, walk = 616013216, run = 616010382, jump = 616008936, fall = 616005863, climb = 616003713 },
	{ name = "🧙 Маг", idle = { 707742142, 707855907 }, walk = 707897309, run = 707861613, jump = 707853694, fall = 707829716, climb = 707826056 },
	{ name = "⚔ Рыцарь", idle = { 657595757, 657568135 }, walk = 657552124, run = 657564596, jump = 658409194, fall = 657600338, climb = 658360781 },
	{ name = "😎 Стильная", idle = { 616136790, 616138447 }, walk = 616146177, run = 616140816, jump = 616139451, fall = 616134815, climb = 616133594 },
	{ name = "🐺 Оборотень", idle = { 1083195517, 1083214717 }, walk = 1083178339, run = 1083216690, jump = 1083218792, fall = 1083189019, climb = 1083182000 },
	{ name = "🏴‍☠️ Пират", idle = { 750781874, 750782770 }, walk = 750785693, run = 750783738, jump = 750782230, fall = 750780242, climb = 750779899 },
	{ name = "🧸 Игрушка", idle = { 782841498, 782845736 }, walk = 782843345, run = 782842708, jump = 782847020, fall = 782846423, climb = 782843869 },
	{ name = "🫧 Пузырь", idle = { 910004836, 910009958 }, walk = 910034870, run = 910025107, jump = 910016857, fall = 910001910, climb = 909997997 },
	{ name = "👴 Старец", idle = { 845397899, 845400520 }, walk = 845403856, run = 845386501, jump = 845398858, fall = 845396048, climb = 845392038 },
	{ name = "🚀 Астронавт", idle = { 891621366, 891633237 }, walk = 891667138, run = 891636393, jump = 891627522, fall = 891617961, climb = 891609353 },
}

local ANIM_SLOTS = {
	{ "idle", "Animation1" }, { "idle", "Animation2" }, { "walk", "WalkAnim" },
	{ "run", "RunAnim" }, { "jump", "JumpAnim" }, { "fall", "FallAnim" }, { "climb", "ClimbAnim" },
}

local animBackup, currentPack

local function animUrl(n) return "http://www.roblox.com/asset/?id=" .. tostring(n) end

local function slotValue(pack, slot)
	if slot[1] == "idle" then
		return pack.idle[slot[2] == "Animation1" and 1 or 2]
	end
	return pack[slot[1]]
end

local function findSlot(animate, slot)
	local f = animate:FindFirstChild(slot[1])
	return f and f:FindFirstChild(slot[2])
end

local function applyAnimation(pack, silent)
	local char = LP.Character
	local hum = char and char:FindFirstChildOfClass("Humanoid")
	local animate = char and char:FindFirstChild("Animate")
	if not (hum and animate) then
		if not silent then animStatus("У персонажа нет скрипта анимаций (в игре свои анимации)") end
		return
	end

	if not animBackup or animBackup.char ~= char then
		animBackup = { char = char, ids = {} }
		for _, s in ipairs(ANIM_SLOTS) do
			local a = findSlot(animate, s)
			if a then animBackup.ids[s[1] .. "/" .. s[2]] = a.AnimationId end
		end
	end

	for _, s in ipairs(ANIM_SLOTS) do
		local a = findSlot(animate, s)
		if a then
			local key = s[1] .. "/" .. s[2]
			if pack then
				a.AnimationId = animUrl(slotValue(pack, s))
			elseif animBackup.ids[key] then
				a.AnimationId = animBackup.ids[key]
			end
		end
	end

	currentPack = pack
	for _, t in ipairs(hum:GetPlayingAnimationTracks()) do t:Stop(0) end
	animate.Disabled = true
	task.wait()
	animate.Disabled = false

	if not silent then
		animStatus(pack and ("Анимация: " .. pack.name .. " (паки для R15)") or "Анимации сброшены")
	end
end

LP.CharacterAdded:Connect(function(char)
	task.spawn(function()
		char:WaitForChild("Humanoid", 10)
		if S.Invis then
			task.wait(0.6)
			pcall(applyInvis, true)
		end
		if currentPack then
			char:WaitForChild("Animate", 10)
			task.wait(0.6)
			pcall(applyAnimation, currentPack, true)
		end
	end)
end)

----------------------------------------------------------------
-- ТАНЦЫ (логика)
----------------------------------------------------------------
local danceStatusLabel
local danceTrack
local function danceStatus(t) if danceStatusLabel then danceStatusLabel.Text = t end end

local function stopDance()
	local char = LP.Character
	local hum = char and char:FindFirstChildOfClass("Humanoid")
	if danceTrack then
		pcall(function() danceTrack:Stop(0.2) end)
		danceTrack = nil
	end
	if hum then
		for _, t in ipairs(hum:GetPlayingAnimationTracks()) do
			local pr = t.Priority
			if pr == Enum.AnimationPriority.Action or pr == Enum.AnimationPriority.Action2
				or pr == Enum.AnimationPriority.Action3 or pr == Enum.AnimationPriority.Action4 then
				t:Stop(0.2)
			end
		end
	end
	danceStatus("Танец остановлен")
end

local function playEmote(name, label)
	local char = LP.Character
	local hum = char and char:FindFirstChildOfClass("Humanoid")
	if not hum then danceStatus("Персонаж не найден") return end
	if hum.RigType ~= Enum.HumanoidRigType.R15 then
		danceStatus("Эмоции работают только на R15")
		return
	end
	if danceTrack then pcall(function() danceTrack:Stop(0.1) end) danceTrack = nil end
	local ok, res = pcall(function() return hum:PlayEmote(name) end)
	danceStatus((ok and res) and ("Играет: " .. label) or "Игра не поддерживает эту эмоцию")
end

local function playCustom(text)
	local char = LP.Character
	local hum = char and char:FindFirstChildOfClass("Humanoid")
	if not hum then danceStatus("Персонаж не найден") return end
	local id = tonumber((tostring(text):gsub("%D", "")))
	if not id then danceStatus("Введи числовой ID анимации") return end

	local animator = hum:FindFirstChildOfClass("Animator") or new("Animator", {}, hum)
	if danceTrack then pcall(function() danceTrack:Stop(0.1) end) danceTrack = nil end

	local anim = new("Animation", { AnimationId = "rbxassetid://" .. id })
	local ok, track = pcall(function() return animator:LoadAnimation(anim) end)
	if not ok or not track then danceStatus("Не удалось загрузить анимацию") return end

	track.Looped = true
	track.Priority = Enum.AnimationPriority.Action4
	track:Play(0.2)
	track:AdjustSpeed(S.DanceSpeed)
	danceTrack = track
	danceStatus("Играет анимация " .. id)

	task.delay(1.2, function()
		if danceTrack == track and track.Length == 0 then
			danceStatus("Анимация не загрузилась (неверный ID или нет доступа)")
		end
	end)
end

----------------------------------------------------------------
-- АНТИ-АФК
----------------------------------------------------------------
LP.Idled:Connect(function()
	if not S.AntiAFK then return end
	pcall(function()
		VirtualUser:CaptureController()
		VirtualUser:ClickButton2(Vector2.new())
	end)
end)

task.spawn(function()
	while true do
		task.wait(60)
		if S.AntiAFK then
			pcall(function()
				VirtualUser:CaptureController()
				VirtualUser:Button2Down(Vector2.new(0, 0), workspace.CurrentCamera.CFrame)
				task.wait(0.1)
				VirtualUser:Button2Up(Vector2.new(0, 0), workspace.CurrentCamera.CFrame)
			end)
			if S.AntiAFKJump and not S.Fly then
				local hum = LP.Character and LP.Character:FindFirstChildOfClass("Humanoid")
				if hum and hum.Health > 0 then
					pcall(function() hum:ChangeState(Enum.HumanoidStateType.Jumping) end)
				end
			end
		end
	end
end)

----------------------------------------------------------------
-- ЭФФЕКТЫ: аура, следы, лента (видны только тебе)
----------------------------------------------------------------
local auraObjs = {}
local function destroyAura()
	for _, o in pairs(auraObjs) do
		if typeof(o) == "Instance" then pcall(function() o:Destroy() end) end
	end
	auraObjs = {}
end

local function ensureAura(char, root)
	if auraObjs.part and auraObjs.part.Parent == char then return end
	destroyAura()
	local pt = new("Part", {
		Name = "NeonAuraPart", Transparency = 1, CanCollide = false, CanQuery = false, CanTouch = false,
		Massless = true, Size = Vector3.new(S.AuraSize, S.AuraSize * 1.4, S.AuraSize), CFrame = root.CFrame,
	}, char)
	new("WeldConstraint", { Part0 = root, Part1 = pt }, pt)
	local em = new("ParticleEmitter", {
		Name = "NeonAuraEmitter",
		Texture = "rbxasset://textures/particles/sparkles_main.dds",
		Lifetime = NumberRange.new(0.9, 1.7),
		Speed = NumberRange.new(0.5, 3),
		SpreadAngle = Vector2.new(180, 180),
		Rotation = NumberRange.new(0, 360),
		RotSpeed = NumberRange.new(-80, 80),
		LightEmission = 1, LightInfluence = 0, Rate = S.AuraRate,
		Acceleration = Vector3.new(0, 3, 0),
		Transparency = NumberSequence.new({ NumberSequenceKeypoint.new(0, 0.15), NumberSequenceKeypoint.new(1, 1) }),
		Size = NumberSequence.new({ NumberSequenceKeypoint.new(0, 0.9), NumberSequenceKeypoint.new(1, 0) }),
	}, pt)
	pcall(function()
		em.Shape = Enum.ParticleEmitterShape.Box
		em.ShapeStyle = Enum.ParticleEmitterShapeStyle.Volume
		em.ShapeInOut = Enum.ParticleEmitterShapeInOut.Outward
	end)
	local lt = new("PointLight", { Name = "NeonAuraLight", Brightness = 1.5, Range = 12 }, pt)
	auraObjs = { part = pt, emitter = em, light = lt }
end

local ribbon = {}
local function destroyRibbon()
	for _, o in pairs(ribbon) do
		if typeof(o) == "Instance" then pcall(function() o:Destroy() end) end
	end
	ribbon = {}
end

local function ensureRibbon(root)
	if ribbon.trail and ribbon.trail.Parent == root then return end
	destroyRibbon()
	local a0 = new("Attachment", { Name = "NeonRibA0", Position = Vector3.new(0, 1, 0) }, root)
	local a1 = new("Attachment", { Name = "NeonRibA1", Position = Vector3.new(0, -1, 0) }, root)
	local tr = new("Trail", {
		Name = "NeonRibTrail", Attachment0 = a0, Attachment1 = a1,
		Lifetime = 1.4, MinLength = 0.05, LightEmission = 1, LightInfluence = 0, FaceCamera = true,
		Transparency = NumberSequence.new({ NumberSequenceKeypoint.new(0, 0.1), NumberSequenceKeypoint.new(1, 1) }),
		WidthScale = NumberSequence.new({ NumberSequenceKeypoint.new(0, 1), NumberSequenceKeypoint.new(1, 0.2) }),
	}, root)
	ribbon = { a0 = a0, a1 = a1, trail = tr }
end

local steps, stepFolder = {}, nil
local lastStepPos, stepSide, stepHue = nil, 1, 0
local MAX_STEPS = 200

local function getStepFolder()
	if not stepFolder or not stepFolder.Parent then
		stepFolder = new("Folder", { Name = "NeonSteps" }, workspace)
	end
	return stepFolder
end

local function clearSteps()
	for _, p in ipairs(steps) do pcall(function() p:Destroy() end) end
	steps = {}
	lastStepPos = nil
end

local function addStep(root, char, color)
	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	params.FilterDescendantsInstances = { char, getStepFolder() }
	local res = workspace:Raycast(root.Position, Vector3.new(0, -7, 0), params)
	if not res then return false end

	local n = res.Normal
	local look = root.CFrame.LookVector
	local fwd = look - n * look:Dot(n)
	if fwd.Magnitude < 0.05 then
		fwd = (math.abs(n.Y) > 0.9 and Vector3.xAxis or Vector3.yAxis):Cross(n)
	end
	fwd = fwd.Unit

	local side = root.CFrame.RightVector * 0.7 * stepSide
	local cf = CFrame.fromMatrix(res.Position + side + n * 0.08, n, fwd)

	local p = new("Part", {
		Name = "NeonStep", Shape = Enum.PartType.Cylinder, Size = Vector3.new(0.12, 1.5, 0.9),
		Anchored = true, CanCollide = false, CanQuery = false, CanTouch = false, CastShadow = false,
		Material = Enum.Material.Neon, Color = color, Transparency = 0.15, CFrame = cf,
	}, getStepFolder())

	table.insert(steps, p)
	if #steps > MAX_STEPS then
		local old = table.remove(steps, 1)
		if old then pcall(function() old:Destroy() end) end
	end

	task.delay(S.StepLife, function()
		if p.Parent then
			TweenService:Create(p, TweenInfo.new(1.2), { Transparency = 1 }):Play()
			task.wait(1.25)
			if p.Parent then p:Destroy() end
		end
	end)
	return true
end

local fxHue = 0
local function fxColor(dt)
	if S.FXRainbow then
		fxHue = (fxHue + dt * 0.25) % 1
		return Color3.fromHSV(fxHue, 0.9, 1)
	end
	return S.FXColor
end

local function fxStep(dt)
	local char = LP.Character
	local root = char and char:FindFirstChild("HumanoidRootPart")
	local hum = char and char:FindFirstChildOfClass("Humanoid")
	if not (root and hum) then return end
	local col = fxColor(dt)

	if S.Aura then
		ensureAura(char, root)
		local sz = S.AuraSize
		local pt = auraObjs.part
		if pt.Size.X ~= sz then pt.Size = Vector3.new(sz, sz * 1.4, sz) end
		auraObjs.emitter.Color = ColorSequence.new(col)
		auraObjs.emitter.Rate = S.AuraRate
		auraObjs.light.Color = col
		auraObjs.light.Range = sz * 2.5
	elseif auraObjs.part then
		destroyAura()
	end

	if S.Ribbon then
		ensureRibbon(root)
		ribbon.trail.Color = ColorSequence.new(col)
	elseif ribbon.trail then
		destroyRibbon()
	end

	if S.Steps then
		local pos = root.Position
		if not lastStepPos or (pos - lastStepPos).Magnitude >= 2.8 then
			local c = col
			if S.FXRainbow then
				stepHue = (stepHue + 0.05) % 1
				c = Color3.fromHSV(stepHue, 0.9, 1)
			end
			if addStep(root, char, c) then
				lastStepPos = pos
				stepSide = -stepSide
			end
		end
	end
end

local warnedFX = false
RunService.Heartbeat:Connect(function(dt)
	local ok, err = pcall(fxStep, dt)
	if not ok and not warnedFX then
		warnedFX = true
		warn("[NeonMenu] FX: " .. tostring(err))
	end
end)

----------------------------------------------------------------
-- UI: основа
----------------------------------------------------------------
local function round(parent, r)
	return new("UICorner", { CornerRadius = UDim.new(0, r or 8) }, parent)
end
local function stroke(parent, color, t)
	return new("UIStroke", { Color = color, Thickness = t or 1, ApplyStrokeMode = Enum.ApplyStrokeMode.Border }, parent)
end

local gui = new("ScreenGui", {
	Name = "DesignerMenu", ResetOnSpawn = false, IgnoreGuiInset = true,
	ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
}, LP:WaitForChild("PlayerGui"))

local espLayer = new("Frame", {
	Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Active = false,
}, gui)

local main = new("Frame", {
	Size = UDim2.fromOffset(480, 350), AnchorPoint = Vector2.new(0.5, 0.5),
	Position = UDim2.fromScale(0.5, 0.5), BackgroundColor3 = BG, BorderSizePixel = 0,
}, gui)
round(main, 14)
stroke(main, ACCENT, 1.5)
new("UIGradient", {
	Color = ColorSequence.new(Color3.fromRGB(24, 20, 40), Color3.fromRGB(14, 14, 22)), Rotation = 90,
}, main)

local uiScale = new("UIScale", {}, main)
local function updateScale()
	local vp = gui.AbsoluteSize
	uiScale.Scale = math.clamp(math.min(vp.X / 520, vp.Y / 390), 0.55, 1)
end
updateScale()
gui:GetPropertyChangedSignal("AbsoluteSize"):Connect(updateScale)

local orb
local function setOpen(v)
	main.Visible = v
	if orb then orb.Text = v and "✕" or "✦" end
end

----------------------------------------------------------------
-- КНОПКА-ОРБ (закрытое состояние)
----------------------------------------------------------------
local BTN = 54

local dock = new("Frame", {
	Size = UDim2.fromOffset(BTN, BTN), Position = UDim2.new(0, 8, 0.5, -BTN / 2),
	BackgroundTransparency = 1, ZIndex = 10,
}, gui)
local dockScale = new("UIScale", {}, dock)

local glow = new("Frame", {
	AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
	Size = UDim2.fromScale(1.5, 1.5), BackgroundColor3 = ACCENT,
	BackgroundTransparency = 0.78, ZIndex = 1,
}, dock)
round(glow, 100)
TweenService:Create(glow, TweenInfo.new(1.5, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true), {
	Size = UDim2.fromScale(1.95, 1.95), BackgroundTransparency = 0.93,
}):Play()

local orbBG = new("Frame", {
	Size = UDim2.fromScale(1, 1), BackgroundColor3 = WHITE, ZIndex = 2,
}, dock)
round(orbBG, BTN)
local orbGrad = new("UIGradient", {
	Color = ColorSequence.new({
		ColorSequenceKeypoint.new(0, Color3.fromRGB(130, 80, 255)),
		ColorSequenceKeypoint.new(1, Color3.fromRGB(255, 80, 190)),
	}),
}, orbBG)
local orbStroke = stroke(orbBG, WHITE, 2.5)
orbStroke.Transparency = 0.15

orb = new("TextButton", {
	Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Text = "✦", TextColor3 = WHITE,
	Font = Enum.Font.GothamBold, TextSize = 26, AutoButtonColor = false, ZIndex = 3,
}, dock)

local badge = new("TextLabel", {
	Size = UDim2.fromOffset(20, 20), Position = UDim2.new(1, -16, 0, -4),
	BackgroundColor3 = Color3.fromRGB(255, 70, 110), Text = "0", TextColor3 = WHITE,
	Font = Enum.Font.GothamBold, TextSize = 12, Visible = false, ZIndex = 4,
}, dock)
round(badge, 10)
stroke(badge, BG, 2)

RunService.Heartbeat:Connect(function()
	orbGrad.Rotation = (os.clock() * 60) % 360
end)

local COUNT_KEYS = {
	"Fly", "Speed", "Noclip", "SuperJump", "Gravity", "Bhop", "Spin", "Invis", "ESP", "Aim",
	"Trigger", "Aura", "Steps", "Ribbon", "Laser", "Fullbright",
}
task.spawn(function()
	while true do
		local n = 0
		for _, k in ipairs(COUNT_KEYS) do
			if S[k] then n += 1 end
		end
		badge.Visible = n > 0
		badge.Text = tostring(n)
		task.wait(0.3)
	end
end)

do -- перетаскивание, примагничивание к краю, тап
	local drag, moved, startIn, startPos = false, false, nil, nil
	local function isPress(i)
		return i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch
	end
	orb.InputBegan:Connect(function(i)
		if isPress(i) then
			drag, moved = true, false
			startIn, startPos = i.Position, dock.Position
		end
	end)
	UIS.InputChanged:Connect(function(i)
		if drag and (i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch) then
			local d = i.Position - startIn
			if d.Magnitude > 8 then moved = true end
			if moved then
				dock.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + d.X, startPos.Y.Scale, startPos.Y.Offset + d.Y)
			end
		end
	end)
	UIS.InputEnded:Connect(function(i)
		if drag and isPress(i) then
			drag = false
			if moved then
				local vp = gui.AbsoluteSize
				local ax, ay = dock.AbsolutePosition.X, dock.AbsolutePosition.Y
				local tx = (ax + BTN / 2 < vp.X / 2) and 8 or (vp.X - BTN - 8)
				local ty = math.clamp(ay, 8, math.max(vp.Y - BTN - 8, 8))
				dock.Position = UDim2.fromOffset(ax, ay)
				TweenService:Create(dock, TweenInfo.new(0.3, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
					Position = UDim2.fromOffset(tx, ty),
				}):Play()
			else
				dockScale.Scale = 0.85
				TweenService:Create(dockScale, TweenInfo.new(0.3, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
					Scale = 1,
				}):Play()
				setOpen(not main.Visible)
			end
		end
	end)
end

UIS.InputBegan:Connect(function(i, gp)
	if not gp and i.KeyCode == Enum.KeyCode.RightShift then setOpen(not main.Visible) end
end)

----------------------------------------------------------------
-- Счётчик FPS / пинг
----------------------------------------------------------------
local statsLabel = new("TextLabel", {
	Size = UDim2.fromOffset(150, 20), AnchorPoint = Vector2.new(0.5, 0),
	Position = UDim2.new(0.5, 0, 0, 50), BackgroundColor3 = BG, BackgroundTransparency = 0.3,
	Text = "", TextColor3 = WHITE, Font = Enum.Font.GothamBold, TextSize = 12,
	Visible = false, ZIndex = 9,
}, gui)
round(statsLabel, 6)
stroke(statsLabel, ACCENT, 1)

do
	local acc, n = 0, 0
	RunService.RenderStepped:Connect(function(dt)
		acc += dt
		n += 1
		if acc >= 0.5 then
			local fps = math.floor(n / acc + 0.5)
			acc, n = 0, 0
			local ping = 0
			pcall(function()
				ping = math.floor(Stats.Network.ServerStatsItem["Data Ping"]:GetValue())
			end)
			statsLabel.Visible = S.ShowStats
			statsLabel.Text = ("FPS %d  •  %d мс"):format(fps, ping)
		end
	end)
end

----------------------------------------------------------------
-- Окно меню
----------------------------------------------------------------
local title = new("Frame", { Size = UDim2.new(1, -50, 0, 42), BackgroundTransparency = 1 }, main)
new("TextLabel", {
	Size = UDim2.new(1, -20, 1, 0), Position = UDim2.fromOffset(16, 0), BackgroundTransparency = 1,
	Text = "✦  NEON  MENU", TextColor3 = TEXT, Font = Enum.Font.GothamBold, TextSize = 16,
	TextXAlignment = Enum.TextXAlignment.Left,
}, title)

local closeBtn = new("TextButton", {
	Size = UDim2.fromOffset(30, 30), Position = UDim2.new(1, -38, 0, 6),
	BackgroundColor3 = Color3.fromRGB(220, 60, 80), Text = "✕", TextColor3 = WHITE,
	Font = Enum.Font.GothamBold, TextSize = 15, ZIndex = 5,
}, main)
round(closeBtn, 8)
closeBtn.MouseButton1Click:Connect(function() setOpen(false) end)

do -- перетаскивание окна
	local dragging, dragStart, startPos
	title.InputBegan:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
			dragging, dragStart, startPos = true, i.Position, main.Position
		end
	end)
	UIS.InputChanged:Connect(function(i)
		if dragging and (i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch) then
			local d = i.Position - dragStart
			main.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + d.X, startPos.Y.Scale, startPos.Y.Offset + d.Y)
		end
	end)
	UIS.InputEnded:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
			dragging = false
		end
	end)
end

local tabBar = new("ScrollingFrame", {
	Size = UDim2.new(0, 120, 1, -54), Position = UDim2.fromOffset(10, 44),
	BackgroundColor3 = PANEL, BorderSizePixel = 0, ScrollBarThickness = 0,
	CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.Y,
}, main)
round(tabBar, 10)
new("UIListLayout", {
	Padding = UDim.new(0, 5), HorizontalAlignment = Enum.HorizontalAlignment.Center,
	SortOrder = Enum.SortOrder.LayoutOrder,
}, tabBar)
new("UIPadding", { PaddingTop = UDim.new(0, 8), PaddingBottom = UDim.new(0, 8) }, tabBar)

local content = new("Frame", {
	Size = UDim2.new(1, -150, 1, -54), Position = UDim2.fromOffset(140, 44),
	BackgroundColor3 = PANEL, BorderSizePixel = 0,
}, main)
round(content, 10)

local pages, tabButtons = {}, {}

local function selectTab(name)
	for n, p in pairs(pages) do p.Visible = (n == name) end
	for n, b in pairs(tabButtons) do
		TweenService:Create(b, TweenInfo.new(0.2), {
			BackgroundColor3 = (n == name) and ACCENT or ITEM,
			TextColor3 = (n == name) and WHITE or SUB,
		}):Play()
	end
end

local function addTab(name, icon)
	local btn = new("TextButton", {
		Size = UDim2.new(1, -16, 0, 34), BackgroundColor3 = ITEM, Text = icon .. " " .. name,
		TextColor3 = SUB, Font = Enum.Font.GothamMedium, TextSize = 13, AutoButtonColor = false,
	}, tabBar)
	round(btn, 8)
	tabButtons[name] = btn
	local page = new("ScrollingFrame", {
		Size = UDim2.new(1, -16, 1, -16), Position = UDim2.fromOffset(8, 8),
		BackgroundTransparency = 1, BorderSizePixel = 0, ScrollBarThickness = 3,
		ScrollBarImageColor3 = ACCENT, CanvasSize = UDim2.new(),
		AutomaticCanvasSize = Enum.AutomaticSize.Y, Visible = false,
	}, content)
	new("UIListLayout", { Padding = UDim.new(0, 8), SortOrder = Enum.SortOrder.LayoutOrder }, page)
	pages[name] = page
	btn.MouseButton1Click:Connect(function() selectTab(name) end)
	return page
end

----------------------------------------------------------------
-- UI: компоненты
----------------------------------------------------------------
local function addToggle(page, text, key, onChange)
	local row = new("Frame", { Size = UDim2.new(1, -4, 0, 40), BackgroundColor3 = ITEM }, page)
	round(row, 8)
	new("TextLabel", {
		Size = UDim2.new(1, -70, 1, 0), Position = UDim2.fromOffset(12, 0), BackgroundTransparency = 1,
		Text = text, TextColor3 = TEXT, Font = Enum.Font.GothamMedium, TextSize = 13,
		TextXAlignment = Enum.TextXAlignment.Left, TextWrapped = true,
	}, row)
	local pill = new("Frame", {
		Size = UDim2.fromOffset(42, 22), Position = UDim2.new(1, -54, 0.5, -11),
		BackgroundColor3 = Color3.fromRGB(60, 60, 80),
	}, row)
	round(pill, 11)
	local knob = new("Frame", {
		Size = UDim2.fromOffset(16, 16), Position = UDim2.fromOffset(3, 3), BackgroundColor3 = WHITE,
	}, pill)
	round(knob, 8)
	local click = new("TextButton", { Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Text = "" }, row)

	local function render()
		local on = S[key]
		TweenService:Create(pill, TweenInfo.new(0.2), { BackgroundColor3 = on and ACCENT or Color3.fromRGB(60, 60, 80) }):Play()
		TweenService:Create(knob, TweenInfo.new(0.2), { Position = on and UDim2.fromOffset(23, 3) or UDim2.fromOffset(3, 3) }):Play()
	end
	click.MouseButton1Click:Connect(function()
		S[key] = not S[key]
		render()
		if onChange then onChange(S[key]) end
	end)
	render()
end

local function addSlider(page, text, key, minV, maxV, step, onChange)
	local row = new("Frame", { Size = UDim2.new(1, -4, 0, 66), BackgroundColor3 = ITEM }, page)
	round(row, 8)
	new("TextLabel", {
		Size = UDim2.new(1, -150, 0, 34), Position = UDim2.fromOffset(12, 0), BackgroundTransparency = 1,
		Text = text, TextColor3 = TEXT, Font = Enum.Font.GothamMedium, TextSize = 13,
		TextXAlignment = Enum.TextXAlignment.Left, TextWrapped = true,
	}, row)

	local function arrow(sym, x)
		local b = new("TextButton", {
			Size = UDim2.fromOffset(28, 26), Position = UDim2.new(1, x, 0, 6),
			BackgroundColor3 = ACCENT, Text = sym, TextColor3 = WHITE,
			Font = Enum.Font.GothamBold, TextSize = 18,
		}, row)
		round(b, 6)
		return b
	end
	local left = arrow("‹", -118)
	local val = new("TextLabel", {
		Size = UDim2.fromOffset(50, 26), Position = UDim2.new(1, -86, 0, 6), BackgroundTransparency = 1,
		Text = "", TextColor3 = TEXT, Font = Enum.Font.GothamBold, TextSize = 13,
	}, row)
	local right = arrow("›", -36)

	local bar = new("Frame", {
		Size = UDim2.new(1, -28, 0, 8), Position = UDim2.fromOffset(14, 48),
		BackgroundColor3 = Color3.fromRGB(55, 55, 75),
	}, row)
	round(bar, 4)
	local fill = new("Frame", { Size = UDim2.fromScale(0, 1), BackgroundColor3 = ACCENT }, bar)
	round(fill, 4)
	local knob = new("Frame", {
		Size = UDim2.fromOffset(20, 20), AnchorPoint = Vector2.new(0.5, 0.5),
		Position = UDim2.fromScale(0, 0.5), BackgroundColor3 = WHITE,
	}, bar)
	round(knob, 10)
	stroke(knob, ACCENT, 2)
	local hit = new("TextButton", {
		Size = UDim2.new(1, -10, 0, 30), Position = UDim2.fromOffset(5, 37), BackgroundTransparency = 1, Text = "",
	}, row)

	local function set(v)
		v = minV + math.floor((v - minV) / step + 0.5) * step
		v = math.clamp(math.floor(v * 100 + 0.5) / 100, minV, maxV)
		S[key] = v
		val.Text = tostring(v)
		local a = (v - minV) / (maxV - minV)
		fill.Size = UDim2.fromScale(a, 1)
		knob.Position = UDim2.fromScale(a, 0.5)
		if onChange then onChange(v) end
	end

	local dragging = false
	local function fromX(x)
		local a = math.clamp((x - bar.AbsolutePosition.X) / bar.AbsoluteSize.X, 0, 1)
		set(minV + a * (maxV - minV))
	end
	hit.InputBegan:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
			dragging = true
			page.ScrollingEnabled = false
			fromX(i.Position.X)
		end
	end)
	UIS.InputChanged:Connect(function(i)
		if dragging and (i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch) then
			fromX(i.Position.X)
		end
	end)
	UIS.InputEnded:Connect(function(i)
		if dragging and (i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch) then
			dragging = false
			page.ScrollingEnabled = true
		end
	end)

	local function hold(btn, d)
		btn.MouseButton1Down:Connect(function()
			local held = true
			local c1; c1 = btn.MouseButton1Up:Connect(function() held = false c1:Disconnect() end)
			local c2; c2 = btn.MouseLeave:Connect(function() held = false c2:Disconnect() end)
			set(S[key] + d * step)
			task.delay(0.4, function()
				while held do set(S[key] + d * step) task.wait(0.05) end
			end)
		end)
	end
	hold(left, -1)
	hold(right, 1)

	set(S[key])
	return set
end

local function addColorPicker(page, text, key, colors, onChange)
	local row = new("Frame", { Size = UDim2.new(1, -4, 0, 76), BackgroundColor3 = ITEM }, page)
	round(row, 8)
	new("TextLabel", {
		Size = UDim2.new(1, -24, 0, 30), Position = UDim2.fromOffset(12, 2), BackgroundTransparency = 1,
		Text = text, TextColor3 = TEXT, Font = Enum.Font.GothamMedium, TextSize = 13,
		TextXAlignment = Enum.TextXAlignment.Left,
	}, row)
	local holder = new("Frame", {
		Size = UDim2.new(1, -24, 0, 30), Position = UDim2.fromOffset(12, 36), BackgroundTransparency = 1,
	}, row)
	new("UIListLayout", { FillDirection = Enum.FillDirection.Horizontal, Padding = UDim.new(0, 6) }, holder)

	local swatches = {}
	local function render()
		for c, s in pairs(swatches) do s.Thickness = (c == S[key]) and 2.5 or 0 end
	end
	for _, c in ipairs(colors) do
		local b = new("TextButton", {
			Size = UDim2.fromOffset(26, 26), BackgroundColor3 = c, Text = "", AutoButtonColor = false,
		}, holder)
		round(b, 13)
		swatches[c] = stroke(b, WHITE, 0)
		b.MouseButton1Click:Connect(function()
			S[key] = c
			render()
			if onChange then onChange(c) end
		end)
	end
	render()
end

local function addGrid(page, items)
	local f = new("Frame", {
		Size = UDim2.new(1, -4, 0, 0), AutomaticSize = Enum.AutomaticSize.Y, BackgroundTransparency = 1,
	}, page)
	new("UIGridLayout", {
		CellSize = UDim2.new(0.5, -4, 0, 36), CellPadding = UDim2.fromOffset(8, 8),
		SortOrder = Enum.SortOrder.LayoutOrder,
	}, f)
	for _, it in ipairs(items) do
		local b = new("TextButton", {
			BackgroundColor3 = ITEM, Text = it.text, TextColor3 = TEXT,
			Font = Enum.Font.GothamMedium, TextSize = 13,
		}, f)
		round(b, 8)
		stroke(b, ACCENT, 1)
		b.MouseButton1Click:Connect(it.cb)
	end
end

local function addNote(page, text, h)
	return new("TextLabel", {
		Size = UDim2.new(1, -4, 0, h or 34), BackgroundTransparency = 1, Text = text, TextColor3 = SUB,
		Font = Enum.Font.Gotham, TextSize = 12, TextWrapped = true,
		TextXAlignment = Enum.TextXAlignment.Left, TextYAlignment = Enum.TextYAlignment.Top,
	}, page)
end

local function addInput(page, placeholder, btnText, cb, clear)
	local row = new("Frame", { Size = UDim2.new(1, -4, 0, 42), BackgroundColor3 = ITEM }, page)
	round(row, 8)
	local box = new("TextBox", {
		Size = UDim2.new(1, -110, 0, 30), Position = UDim2.fromOffset(8, 6),
		BackgroundColor3 = BG, PlaceholderText = placeholder, Text = "", TextColor3 = TEXT,
		PlaceholderColor3 = SUB, Font = Enum.Font.Gotham, TextSize = 13, ClearTextOnFocus = false,
	}, row)
	round(box, 6)
	local b = new("TextButton", {
		Size = UDim2.fromOffset(88, 30), Position = UDim2.new(1, -96, 0, 6), BackgroundColor3 = ACCENT,
		Text = btnText, TextColor3 = WHITE, Font = Enum.Font.GothamBold, TextSize = 13,
	}, row)
	round(b, 6)
	local function submit()
		if box.Text ~= "" then
			local t = box.Text
			if clear then box.Text = "" end
			cb(t)
		end
	end
	b.MouseButton1Click:Connect(submit)
	box.FocusLost:Connect(function(enter)
		if enter and clear then submit() end
	end)
end

----------------------------------------------------------------
-- Вкладки
----------------------------------------------------------------
local pMove = addTab("Движение", "🚀")
local pPlayer = addTab("Игрок", "🛡")
local pFX = addTab("Эффекты", "✨")
local pLaser = addTab("Лазер", "🔥")
local pDance = addTab("Танцы", "💃")
local pESP = addTab("ESP", "👁")
local pAim = addTab("Аим", "🎯")
local pTP = addTab("Телепорт", "📍")
local pChat = addTab("Чат", "💬")
local pAnim = addTab("Анимации", "🕺")
local pVis = addTab("Визуал", "🌌")
local pSkin = addTab("Скин", "🧍")

local PALETTE = {
	Color3.fromRGB(255, 70, 90), Color3.fromRGB(255, 150, 40), Color3.fromRGB(255, 230, 60),
	Color3.fromRGB(80, 235, 120), Color3.fromRGB(60, 220, 230), Color3.fromRGB(70, 130, 255),
	Color3.fromRGB(150, 90, 255), Color3.fromRGB(255, 100, 220), Color3.fromRGB(255, 255, 255),
}

local refreshESP
local camFovTouched = false

-- Движение
addToggle(pMove, "Полёт (WASD, Space/Ctrl или джойстик + ▲▼)", "Fly")
addSlider(pMove, "Скорость полёта", "FlySpeed", 10, 500, 10)
addToggle(pMove, "Ускорение бега", "Speed")
addSlider(pMove, "Скорость бега", "WalkSpeed", 16, 500, 4)
addToggle(pMove, "Сквозь стены (NoClip)", "Noclip")
addToggle(pMove, "Супер-прыжок", "SuperJump")
addSlider(pMove, "Сила прыжка (макс. 1000)", "JumpPower", 50, 1000, 10)
addToggle(pMove, "Своя гравитация (лунные прыжки)", "Gravity")
addSlider(pMove, "Гравитация (196 = обычная, меньше = легче)", "GravityVal", 5, 300, 5)
addToggle(pMove, "Бхоп (ПК: держи Space, телефон: авто)", "Bhop")
addSlider(pMove, "Скорость бхопа", "BhopSpeed", 16, 200, 2)
addToggle(pMove, "Крутилка игрока (спин)", "Spin")
addSlider(pMove, "Обороты в секунду", "SpinSpeed", 0.5, 20, 0.5)

-- Игрок
addToggle(pPlayer, "Невидимость (видна только тебе)", "Invis", function(v)
	pcall(applyInvis, v)
end)
addToggle(pPlayer, "Анти-флинг (защита от выбрасывания)", "AntiFling")
addNote(pPlayer, "Анти-флинг: другие игроки проходят сквозь тебя и не могут толкнуть, резкие скачки скорости гасятся.", 48)
addToggle(pPlayer, "Анти-АФК (не кикает за простой)", "AntiAFK")
addToggle(pPlayer, "Микро-прыжок раз в минуту (если игра сама кикает)", "AntiAFKJump")
addNote(pPlayer, "Анти-АФК убирает стандартный кик Роблокса за 20 минут простоя. Свои АФК-системы игр могут требовать микро-прыжок.", 60)

-- Эффекты
addToggle(pFX, "Аура вокруг игрока", "Aura")
addSlider(pFX, "Размер ауры", "AuraSize", 2, 16, 1)
addSlider(pFX, "Плотность ауры", "AuraRate", 5, 150, 5)
addToggle(pFX, "Следы (остаются на земле)", "Steps")
addSlider(pFX, "Сколько живут следы, сек", "StepLife", 2, 60, 1)
addToggle(pFX, "Лента за спиной", "Ribbon")
addToggle(pFX, "Радужные цвета", "FXRainbow")
addColorPicker(pFX, "Цвет эффектов", "FXColor", PALETTE)
addGrid(pFX, { { text = "🧹 Стереть следы", cb = clearSteps } })
addNote(pFX, "Эффекты видны только тебе (другие игроки их не видят).", 34)

-- Лазер
addToggle(pLaser, "Лазер из глаз (только визуал)", "Laser")
addToggle(pLaser, "Стрелять всегда (выкл = держи E / кнопку 🔥)", "LaserAlways")
addSlider(pLaser, "Длина луча, м", "LaserRange", 30, 500, 10)
addSlider(pLaser, "Толщина луча", "LaserWidth", 0.1, 1.5, 0.05)
addToggle(pLaser, "Радужный лазер", "LaserRainbow")
addColorPicker(pLaser, "Цвет лазера", "LaserColor", PALETTE)
addNote(pLaser, "Лазер — чистый визуал: виден только тебе, никого не толкает и не ранит. Работает, когда меню закрыто.", 48)

-- Танцы
do
	local emotes = {
		{ "💃 Танец 1", "dance" }, { "🕺 Танец 2", "dance2" }, { "🪩 Танец 3", "dance3" },
		{ "👋 Помахать", "wave" }, { "🙌 Ура", "cheer" }, { "😂 Смех", "laugh" }, { "👉 Показать", "point" },
	}
	local items = {}
	for _, e in ipairs(emotes) do
		table.insert(items, { text = e[1], cb = function() playEmote(e[2], e[1]) end })
	end
	table.insert(items, { text = "⏹ Стоп", cb = stopDance })
	addGrid(pDance, items)
	addInput(pDance, "ID анимации (свой танец)", "Играть", playCustom)
	addSlider(pDance, "Скорость своей анимации", "DanceSpeed", 0.2, 3, 0.1, function(v)
		if danceTrack then pcall(function() danceTrack:AdjustSpeed(v) end) end
	end)
	danceStatusLabel = addNote(pDance, "Эмоции работают на R15, танец прерывается, если пойдёшь. Твой танец видят и другие игроки.", 48)
end

-- ESP
addToggle(pESP, "Включить ESP", "ESP", function() refreshESP() end)
addToggle(pESP, "Заливка силуэта через стены", "ESPFill", function() refreshESP() end)
addToggle(pESP, "Коробки", "ESPBox")
addToggle(pESP, "Скелеты", "ESPSkel")
addToggle(pESP, "Ники", "ESPName")
addToggle(pESP, "Дистанция", "ESPDist")
addToggle(pESP, "Здоровье", "ESPHealth")
addColorPicker(pESP, "Цвет ESP", "ESPColor", PALETTE, function() refreshESP() end)

-- Аим + триггер
addNote(pAim, "⚠ Аим и триггер работают, когда меню ЗАКРЫТО (✕ или ✦).", 34)
addToggle(pAim, "Аим", "Aim")
addToggle(pAim, "Режим телефона (круг по центру, кнопки 🎯 🔫)", "PhoneMode")
addToggle(pAim, "Авто-аим (без кнопки / ПКМ)", "AutoAim")
addToggle(pAim, "Целиться в ближайшего (по расстоянию)", "AimNearest")
addSlider(pAim, "Макс. дистанция до цели, м", "NearRange", 20, 2000, 10)
addToggle(pAim, "Ограничить кругом FOV", "UseFOVLimit")
addSlider(pAim, "Размер FOV (круг)", "FOV", 30, 800, 10)
addSlider(pAim, "Сила наводки (100 = мгновенно)", "AimPower", 1, 100, 1)
addToggle(pAim, "Целиться в голову (выкл = в тело)", "AimHead")
addToggle(pAim, "Проверка стен (выкл = аим сквозь стены)", "WallCheck")
addToggle(pAim, "Показывать круг FOV", "ShowFOV")
addToggle(pAim, "Не целиться в команду", "TeamCheck")
addToggle(pAim, "Триггер-бот", "Trigger")
addToggle(pAim, "Триггер всегда (выкл = по кнопке T / 🔫)", "TriggerAlways")
addSlider(pAim, "Задержка между выстрелами, мс", "TriggerDelay", 0, 500, 10)
addSlider(pAim, "Радиус триггера, px (0 = точно в прицел)", "TriggerRadius", 0, 80, 1)
addNote(pAim, "ПК: аим — зажми ПКМ (или авто-аим). Телефон: круг FOV по центру экрана, аим — кнопка 🎯 или авто-аим.", 48)

-- Телепорт + наблюдение
local tpStatus
do
	local top = new("Frame", { Size = UDim2.new(1, -4, 0, 36), BackgroundTransparency = 1 }, pTP)
	local refresh = new("TextButton", {
		Size = UDim2.new(0.5, -4, 1, 0), BackgroundColor3 = ACCENT, Text = "↻ Обновить",
		TextColor3 = WHITE, Font = Enum.Font.GothamBold, TextSize = 13,
	}, top)
	round(refresh, 8)
	local stopSpec = new("TextButton", {
		Size = UDim2.new(0.5, 0, 1, 0), Position = UDim2.new(0.5, 0, 0, 0), BackgroundColor3 = ITEM,
		Text = "⏹ Выйти из 👁", TextColor3 = TEXT, Font = Enum.Font.GothamBold, TextSize = 13,
	}, top)
	round(stopSpec, 8)
	stroke(stopSpec, ACCENT, 1)

	tpStatus = addNote(pTP, "Нажми на ник — телепорт, на 👁 — наблюдать за игроком", 32)
	local list = new("Frame", {
		Size = UDim2.new(1, -4, 0, 0), AutomaticSize = Enum.AutomaticSize.Y, BackgroundTransparency = 1,
	}, pTP)
	new("UIListLayout", { Padding = UDim.new(0, 6), SortOrder = Enum.SortOrder.LayoutOrder }, list)

	local function tpTo(p)
		local c = p.Character
		local r = c and c:FindFirstChild("HumanoidRootPart")
		local mine = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
		if r and mine then
			mine.CFrame = r.CFrame * CFrame.new(0, 2, 3)
			tpStatus.Text = "Телепорт к: " .. p.DisplayName
		else
			tpStatus.Text = "Игрок недоступен"
		end
	end

	local function spectate(p)
		local c = p.Character
		local h = c and c:FindFirstChildOfClass("Humanoid")
		if h then
			Camera.CameraSubject = h
			tpStatus.Text = "Наблюдаешь за: " .. p.DisplayName
		else
			tpStatus.Text = "Игрок недоступен"
		end
	end

	local function stopSpectate()
		local h = LP.Character and LP.Character:FindFirstChildOfClass("Humanoid")
		if h then Camera.CameraSubject = h end
		tpStatus.Text = "Камера вернулась к тебе"
	end
	stopSpec.MouseButton1Click:Connect(stopSpectate)

	local function rebuild()
		for _, v in ipairs(list:GetChildren()) do
			if not v:IsA("UIListLayout") then v:Destroy() end
		end
		for _, p in ipairs(Players:GetPlayers()) do
			if p ~= LP then
				local row = new("Frame", { Size = UDim2.new(1, 0, 0, 34), BackgroundTransparency = 1 }, list)
				local b = new("TextButton", {
					Size = UDim2.new(1, -46, 1, 0), BackgroundColor3 = ITEM,
					Text = p.DisplayName .. "  (@" .. p.Name .. ")", TextColor3 = TEXT,
					Font = Enum.Font.GothamMedium, TextSize = 13,
				}, row)
				round(b, 8)
				stroke(b, ACCENT, 1)
				b.MouseButton1Click:Connect(function() tpTo(p) end)
				local eye = new("TextButton", {
					Size = UDim2.fromOffset(40, 34), Position = UDim2.new(1, -40, 0, 0),
					BackgroundColor3 = ACCENT, Text = "👁", TextColor3 = WHITE,
					Font = Enum.Font.GothamBold, TextSize = 16,
				}, row)
				round(eye, 8)
				eye.MouseButton1Click:Connect(function() spectate(p) end)
			end
		end
	end
	refresh.MouseButton1Click:Connect(rebuild)
	Players.PlayerAdded:Connect(function() task.wait(0.5) rebuild() end)
	Players.PlayerRemoving:Connect(function() task.defer(rebuild) end)
	rebuild()
end

-- Чат
local function forceChat()
	pcall(function() StarterGui:SetCoreGuiEnabled(Enum.CoreGuiType.Chat, true) end)
	pcall(function()
		local w = TextChatService:FindFirstChildOfClass("ChatWindowConfiguration")
		if w then w.Enabled = true end
		local i = TextChatService:FindFirstChildOfClass("ChatInputBarConfiguration")
		if i then i.Enabled = true end
	end)
end

task.spawn(function()
	while true do
		if S.ForceChat then forceChat() end
		task.wait(1)
	end
end)

do
	addToggle(pChat, "Показать чат Роблокс (включить принудительно)", "ForceChat", function(v)
		if v then forceChat() end
	end)

	local log = new("ScrollingFrame", {
		Size = UDim2.new(1, -4, 0, 150), BackgroundColor3 = BG, BorderSizePixel = 0,
		ScrollBarThickness = 3, ScrollBarImageColor3 = ACCENT, CanvasSize = UDim2.new(),
		AutomaticCanvasSize = Enum.AutomaticSize.Y,
	}, pChat)
	round(log, 8)
	new("UIListLayout", { Padding = UDim.new(0, 2), SortOrder = Enum.SortOrder.LayoutOrder }, log)
	new("UIPadding", {
		PaddingLeft = UDim.new(0, 6), PaddingRight = UDim.new(0, 6), PaddingTop = UDim.new(0, 4),
	}, log)

	local msgs, order = {}, 0
	local function addChatMsg(text)
		order += 1
		local l = new("TextLabel", {
			Size = UDim2.new(1, 0, 0, 0), AutomaticSize = Enum.AutomaticSize.Y,
			BackgroundTransparency = 1, Text = text, TextColor3 = TEXT, Font = Enum.Font.Gotham,
			TextSize = 12, TextWrapped = true, TextXAlignment = Enum.TextXAlignment.Left,
			LayoutOrder = order,
		}, log)
		table.insert(msgs, l)
		if #msgs > 60 then
			local old = table.remove(msgs, 1)
			old:Destroy()
		end
		task.defer(function() log.CanvasPosition = Vector2.new(0, 1e6) end)
	end

	local chatStatus = addNote(pChat, "Сообщения чата игры появятся выше. Напиши и нажми «Отправить».", 34)

	local function clean(s)
		s = tostring(s or "")
		s = s:gsub("<[^>]+>", "")
		s = s:gsub("&lt;", "<")
		s = s:gsub("&gt;", ">")
		s = s:gsub("&quot;", '"')
		s = s:gsub("&apos;", "'")
		s = s:gsub("&amp;", "&")
		return s
	end

	local newChat = TextChatService.ChatVersion == Enum.ChatVersion.TextChatService

	local function sendChat(text)
		local ok = false
		if newChat then
			local channels = TextChatService:FindFirstChild("TextChannels")
			local ch = channels and (channels:FindFirstChild("RBXGeneral") or channels:FindFirstChildOfClass("TextChannel"))
			if ch then
				ok = pcall(function() ch:SendAsync(text) end)
			end
		else
			local ev = ReplicatedStorage:FindFirstChild("DefaultChatSystemChatEvents")
			local say = ev and ev:FindFirstChild("SayMessageRequest")
			if say then
				ok = pcall(function() say:FireServer(text, "All") end)
			end
		end
		chatStatus.Text = ok and "✔ Отправлено" or "✖ Не удалось отправить (в игре отключён стандартный чат)"
	end

	addInput(pChat, "Сообщение...", "Отправить", sendChat, true)

	if newChat then
		pcall(function()
			TextChatService.MessageReceived:Connect(function(m)
				local who = clean(m.PrefixText)
				if who == "" and m.TextSource then who = m.TextSource.Name .. ":" end
				addChatMsg((who ~= "" and (who .. " ") or "") .. clean(m.Text))
			end)
		end)
	else
		local function hook(p)
			p.Chatted:Connect(function(t) addChatMsg(p.DisplayName .. ": " .. t) end)
		end
		for _, p in ipairs(Players:GetPlayers()) do hook(p) end
		Players.PlayerAdded:Connect(hook)
	end
end

-- Анимации
do
	local items = {}
	for _, p in ipairs(ANIMS) do
		table.insert(items, { text = p.name, cb = function() applyAnimation(p) end })
	end
	table.insert(items, { text = "↺ Сброс анимации", cb = function() applyAnimation(nil) end })
	addGrid(pAnim, items)
	animStatusLabel = addNote(pAnim, "Выбери пак анимаций. Работает для R15; сохраняется после респавна.", 48)
end

-- Визуал
addToggle(pVis, "Fullbright (убрать темноту и туман)", "Fullbright", function(v)
	pcall(applyFullbright, v)
end)
addToggle(pVis, "Счётчик FPS и пинга", "ShowStats")
addSlider(pVis, "Угол обзора камеры (макс. 120)", "CamFOV", 30, 120, 1, function()
	camFovTouched = true
end)
camFovTouched = false
timeSet = addSlider(pVis, "Время суток", "ClockTime", 0, 24, 0.1, function(v)
	Lighting.ClockTime = v
end)
do
	local items = {}
	for _, p in ipairs(SKY_PRESETS) do
		table.insert(items, { text = p.name, cb = function() applySky(p) end })
	end
	table.insert(items, { text = "↺ Сброс неба", cb = resetSky })
	addGrid(pVis, items)
end
addColorPicker(pVis, "Оттенок неба / экрана", "SkyTint", PALETTE, function(c)
	tint.TintColor = c
end)

-- Скин
do
	local items = {}
	for _, p in ipairs(SKINS) do
		table.insert(items, { text = p.name, cb = function() applySkin(p) end })
	end
	table.insert(items, { text = "↺ Сброс скина", cb = resetSkin })
	addGrid(pSkin, items)
end
addInput(pSkin, "Ник или UserId", "Надеть", wearUser)
skinStatus = addNote(pSkin, "Скин виден только тебе. Пресеты — цвета и простые детали, похожие на твои фото.", 48)

selectTab("Движение")

----------------------------------------------------------------
-- Кнопки на экране (телефон) + полёт
----------------------------------------------------------------
local upHeld, downHeld = false, false
local aimHeld, trigHeld, laserHeld = false, false, false
local spinAngle = 0

local function makeHoldBtn(sym, pos, setter)
	local b = new("TextButton", {
		Size = UDim2.fromOffset(56, 56), Position = pos,
		BackgroundColor3 = ACCENT, BackgroundTransparency = 0.2, Text = sym, TextColor3 = WHITE,
		Font = Enum.Font.GothamBold, TextSize = 24, Visible = false, ZIndex = 10,
	}, gui)
	round(b, 28)
	b.InputBegan:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.Touch or i.UserInputType == Enum.UserInputType.MouseButton1 then setter(true) end
	end)
	b.InputEnded:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.Touch or i.UserInputType == Enum.UserInputType.MouseButton1 then setter(false) end
	end)
	return b
end
local flyUpBtn = makeHoldBtn("▲", UDim2.new(1, -76, 0.5, -70), function(v) upHeld = v end)
local flyDownBtn = makeHoldBtn("▼", UDim2.new(1, -76, 0.5, 0), function(v) downHeld = v end)
local aimBtn = makeHoldBtn("🎯", UDim2.new(1, -150, 0.5, -70), function(v) aimHeld = v end)
local trigBtn = makeHoldBtn("🔫", UDim2.new(1, -150, 0.5, 0), function(v) trigHeld = v end)
local laserBtn = makeHoldBtn("🔥", UDim2.new(1, -76, 0.5, 70), function(v) laserHeld = v end)

UIS.InputBegan:Connect(function(i, gp)
	if gp then return end
	if i.KeyCode == Enum.KeyCode.T then trigHeld = true end
	if i.KeyCode == Enum.KeyCode.E then laserHeld = true end
end)
UIS.InputEnded:Connect(function(i)
	if i.KeyCode == Enum.KeyCode.T then trigHeld = false end
	if i.KeyCode == Enum.KeyCode.E then laserHeld = false end
end)

local bv, bg
local function stopFly()
	if bv then bv:Destroy() bv = nil end
	if bg then bg:Destroy() bg = nil end
	local hum = LP.Character and LP.Character:FindFirstChildOfClass("Humanoid")
	if hum then hum.PlatformStand = false end
end

RunService.RenderStepped:Connect(function(dt)
	Camera = workspace.CurrentCamera
	if not Camera then return end
	local showFly = S.Fly and IS_TOUCH
	flyUpBtn.Visible, flyDownBtn.Visible = showFly, showFly
	aimBtn.Visible = S.PhoneMode and S.Aim and not S.AutoAim
	trigBtn.Visible = S.PhoneMode and S.Trigger and not S.TriggerAlways
	laserBtn.Visible = IS_TOUCH and S.Laser and not S.LaserAlways

	local char = LP.Character
	local root = char and char:FindFirstChild("HumanoidRootPart")
	local hum = char and char:FindFirstChildOfClass("Humanoid")

	if S.Fly and root and hum then
		if not bv or bv.Parent ~= root then
			stopFly()
			bv = new("BodyVelocity", { MaxForce = Vector3.one * 1e6, Velocity = Vector3.zero }, root)
			bg = new("BodyGyro", { MaxTorque = Vector3.one * 1e6, P = 1e4 }, root)
		end
		hum.PlatformStand = true
		local cf = Camera.CFrame
		local d = Vector3.zero
		if not UIS:GetFocusedTextBox() then
			if UIS:IsKeyDown(Enum.KeyCode.W) then d += cf.LookVector end
			if UIS:IsKeyDown(Enum.KeyCode.S) then d -= cf.LookVector end
			if UIS:IsKeyDown(Enum.KeyCode.D) then d += cf.RightVector end
			if UIS:IsKeyDown(Enum.KeyCode.A) then d -= cf.RightVector end
			if UIS:IsKeyDown(Enum.KeyCode.Space) then d += Vector3.yAxis end
			if UIS:IsKeyDown(Enum.KeyCode.LeftControl) then d -= Vector3.yAxis end
		end
		if upHeld then d += Vector3.yAxis end
		if downHeld then d -= Vector3.yAxis end
		if hum.MoveDirection.Magnitude > 0 then
			local rel = cf:VectorToObjectSpace(hum.MoveDirection)
			d += cf.LookVector * (-rel.Z) + cf.RightVector * rel.X
		end
		bv.Velocity = d.Magnitude > 0 and d.Unit * S.FlySpeed or Vector3.zero

		if S.Spin then
			spinAngle += math.pi * 2 * S.SpinSpeed * dt
			bg.CFrame = CFrame.new(root.Position) * CFrame.Angles(0, spinAngle, 0)
		else
			bg.CFrame = cf
		end
	elseif bv then
		stopFly()
	end
end)

----------------------------------------------------------------
-- СКОРОСТЬ / ПРЫЖОК / БХОП / СПИН / АНТИ-ФЛИНГ (скорости)
----------------------------------------------------------------
local wasSpeed, wasSpin = false, false
local sj = nil
local lastSafe, lastChar = nil, nil

RunService.Heartbeat:Connect(function(dt)
	local char = LP.Character
	local hum = char and char:FindFirstChildOfClass("Humanoid")
	local root = char and char:FindFirstChild("HumanoidRootPart")
	if not hum or not root then return end

	if S.Speed then
		hum.WalkSpeed = S.WalkSpeed
		wasSpeed = true
	elseif wasSpeed then
		hum.WalkSpeed = 16
		wasSpeed = false
	end

	if S.SuperJump then
		if not sj or sj.hum ~= hum then
			sj = { hum = hum, use = hum.UseJumpPower, power = hum.JumpPower }
		end
		hum.UseJumpPower = true
		hum.JumpPower = S.JumpPower
	elseif sj then
		if sj.hum == hum then
			hum.UseJumpPower = sj.use
			hum.JumpPower = sj.power
		end
		sj = nil
	end

	if S.Bhop and not S.Fly and hum.Health > 0 then
		local typing = UIS:GetFocusedTextBox() ~= nil
		local wants = (not typing and UIS:IsKeyDown(Enum.KeyCode.Space))
			or (IS_TOUCH and hum.MoveDirection.Magnitude > 0)
		if wants then
			if hum.FloorMaterial ~= Enum.Material.Air then
				hum:ChangeState(Enum.HumanoidStateType.Jumping)
			elseif hum.MoveDirection.Magnitude > 0 then
				local v = root.AssemblyLinearVelocity
				local h = hum.MoveDirection * S.BhopSpeed
				root.AssemblyLinearVelocity = Vector3.new(h.X, v.Y, h.Z)
			end
		end
	end

	if S.Spin then
		wasSpin = true
		hum.AutoRotate = false
		if not S.Fly then
			root.CFrame = root.CFrame * CFrame.Angles(0, math.pi * 2 * S.SpinSpeed * dt, 0)
		end
	elseif wasSpin then
		wasSpin = false
		hum.AutoRotate = true
	end

	-- анти-флинг: гасим резкие скачки скорости и вращения
	if lastChar ~= char then lastChar = char lastSafe = nil end
	if S.AntiFling and hum.Health > 0 then
		local lim = 120
		if S.Fly then lim = math.max(lim, S.FlySpeed * 1.6) end
		if S.Speed then lim = math.max(lim, S.WalkSpeed * 1.6) end
		if S.Bhop then lim = math.max(lim, S.BhopSpeed * 1.6) end
		if S.SuperJump then lim = math.max(lim, S.JumpPower * 1.6) end
		local v = root.AssemblyLinearVelocity
		local horiz = Vector3.new(v.X, 0, v.Z).Magnitude
		local angLim = S.Spin and 600 or 60
		if horiz > lim or v.Y > lim or root.AssemblyAngularVelocity.Magnitude > angLim then
			root.AssemblyLinearVelocity = Vector3.zero
			root.AssemblyAngularVelocity = Vector3.zero
			if lastSafe then root.CFrame = lastSafe end
		else
			lastSafe = root.CFrame
		end
	end
end)

----------------------------------------------------------------
-- NOCLIP + АНТИ-ФЛИНГ (чужие игроки не сталкиваются с тобой)
----------------------------------------------------------------
local wasNoclip = false
RunService.Stepped:Connect(function()
	local char = LP.Character
	if char then
		if S.Noclip then
			wasNoclip = true
			for _, v in ipairs(char:GetDescendants()) do
				if v:IsA("BasePart") then v.CanCollide = false end
			end
		elseif wasNoclip then
			wasNoclip = false
			for _, n in ipairs({ "Head", "UpperTorso", "Torso", "HumanoidRootPart" }) do
				local p = char:FindFirstChild(n)
				if p then p.CanCollide = true end
			end
		end
	end

	if S.AntiFling then
		for _, p in ipairs(Players:GetPlayers()) do
			local c = p ~= LP and p.Character
			if c then
				for _, v in ipairs(c:GetDescendants()) do
					if v:IsA("BasePart") and v.CanCollide then v.CanCollide = false end
				end
			end
		end
	end
end)

----------------------------------------------------------------
-- ESP: Highlight (заливка через стены)
----------------------------------------------------------------
local highlights = {}

local function clearHighlight(p)
	if highlights[p] then highlights[p]:Destroy() highlights[p] = nil end
end

local function applyHighlight(p)
	if p == LP then return end
	local char = p.Character
	if not (S.ESP and S.ESPFill) or not char then clearHighlight(p) return end
	local h = highlights[p]
	if not h or h.Parent ~= char then
		clearHighlight(p)
		h = new("Highlight", {
			Adornee = char,
			DepthMode = Enum.HighlightDepthMode.AlwaysOnTop,
			FillTransparency = 0.5, OutlineTransparency = 0,
		}, char)
		highlights[p] = h
	end
	h.FillColor = S.ESPColor
	h.OutlineColor = S.ESPColor
end

refreshESP = function()
	for _, p in ipairs(Players:GetPlayers()) do
		pcall(applyHighlight, p)
	end
end

task.spawn(function()
	while true do
		task.wait(1)
		pcall(refreshESP)
	end
end)

----------------------------------------------------------------
-- ESP: коробки, скелеты, ник, дистанция, здоровье
----------------------------------------------------------------
local R15 = {
	{ "Head", "UpperTorso" }, { "UpperTorso", "LowerTorso" },
	{ "UpperTorso", "LeftUpperArm" }, { "LeftUpperArm", "LeftLowerArm" }, { "LeftLowerArm", "LeftHand" },
	{ "UpperTorso", "RightUpperArm" }, { "RightUpperArm", "RightLowerArm" }, { "RightLowerArm", "RightHand" },
	{ "LowerTorso", "LeftUpperLeg" }, { "LeftUpperLeg", "LeftLowerLeg" }, { "LeftLowerLeg", "LeftFoot" },
	{ "LowerTorso", "RightUpperLeg" }, { "RightUpperLeg", "RightLowerLeg" }, { "RightLowerLeg", "RightFoot" },
}
local R6 = {
	{ "Head", "Torso" }, { "Torso", "Left Arm" }, { "Torso", "Right Arm" },
	{ "Torso", "Left Leg" }, { "Torso", "Right Leg" },
}

local espObjs = {}
local warnedESP = false

local function makeESP()
	local o = {}
	o.box = new("Frame", { BackgroundTransparency = 1, Visible = false }, espLayer)
	o.boxStroke = new("UIStroke", { Thickness = 1.5, Color = S.ESPColor }, o.box)
	o.hpBack = new("Frame", { BackgroundColor3 = Color3.new(0, 0, 0), BorderSizePixel = 0, Visible = false }, espLayer)
	o.hpFill = new("Frame", {
		AnchorPoint = Vector2.new(0, 1), Position = UDim2.fromScale(0, 1),
		Size = UDim2.fromScale(1, 1), BorderSizePixel = 0, BackgroundColor3 = Color3.fromRGB(80, 255, 100),
	}, o.hpBack)
	o.name = new("TextLabel", {
		Size = UDim2.fromOffset(140, 14), BackgroundTransparency = 1, Visible = false,
		Font = Enum.Font.GothamBold, TextSize = 12, TextColor3 = WHITE, TextStrokeTransparency = 0.4,
	}, espLayer)
	o.info = new("TextLabel", {
		Size = UDim2.fromOffset(140, 14), BackgroundTransparency = 1, Visible = false,
		Font = Enum.Font.GothamMedium, TextSize = 11, TextColor3 = WHITE, TextStrokeTransparency = 0.4,
	}, espLayer)
	o.lines = {}
	for i = 1, 14 do
		o.lines[i] = new("Frame", {
			AnchorPoint = Vector2.new(0.5, 0.5), BorderSizePixel = 0, Visible = false,
			BackgroundColor3 = S.ESPColor,
		}, espLayer)
	end
	return o
end

local function hideESP(o)
	o.box.Visible = false
	o.hpBack.Visible = false
	o.name.Visible = false
	o.info.Visible = false
	for _, l in ipairs(o.lines) do l.Visible = false end
end

local function destroyESP(p)
	local o = espObjs[p]
	if not o then return end
	pcall(function()
		o.box:Destroy(); o.hpBack:Destroy(); o.name:Destroy(); o.info:Destroy()
		for _, l in ipairs(o.lines) do l:Destroy() end
	end)
	espObjs[p] = nil
end

local function setLine(f, a, b)
	local d = b - a
	f.Size = UDim2.fromOffset(d.Magnitude, 1.5)
	f.Position = UDim2.fromOffset((a.X + b.X) / 2, (a.Y + b.Y) / 2)
	f.Rotation = math.deg(math.atan2(d.Y, d.X))
	f.BackgroundColor3 = S.ESPColor
	f.Visible = true
end

local function updatePlayerESP(p)
	local o = espObjs[p]
	if o and not o.box.Parent then
		destroyESP(p)
		o = nil
	end
	if not o then
		o = makeESP()
		espObjs[p] = o
	end

	local char = p.Character
	local hum = char and char:FindFirstChildOfClass("Humanoid")
	local root = char and (char:FindFirstChild("HumanoidRootPart") or char.PrimaryPart)
	local head = char and (char:FindFirstChild("Head") or root)
	local alive = root and head and (not hum or hum.Health > 0)

	if not (S.ESP and alive) then
		hideESP(o)
		return
	end

	local rp, onScreen = Camera:WorldToViewportPoint(root.Position)
	if not (onScreen and rp.Z > 0) then
		hideESP(o)
		return
	end

	local top = Camera:WorldToViewportPoint(head.Position + Vector3.new(0, 0.8, 0))
	local bot = Camera:WorldToViewportPoint(root.Position - Vector3.new(0, 3, 0))
	local h = math.max(bot.Y - top.Y, 8)
	local w = h * 0.55
	local cx = rp.X
	local x0 = cx - w / 2

	o.box.Visible = S.ESPBox
	if S.ESPBox then
		o.box.Position = UDim2.fromOffset(x0, top.Y)
		o.box.Size = UDim2.fromOffset(w, h)
		o.boxStroke.Color = S.ESPColor
	end

	local hp = hum and hum.Health or 100
	local maxhp = hum and hum.MaxHealth or 100
	local ratio = math.clamp(hp / math.max(maxhp, 1), 0, 1)
	o.hpBack.Visible = S.ESPHealth
	if S.ESPHealth then
		o.hpBack.Position = UDim2.fromOffset(x0 - 6, top.Y)
		o.hpBack.Size = UDim2.fromOffset(3, h)
		o.hpFill.Size = UDim2.fromScale(1, ratio)
		o.hpFill.BackgroundColor3 = Color3.fromHSV(ratio * 0.33, 1, 1)
	end

	o.name.Visible = S.ESPName
	if S.ESPName then
		o.name.Text = p.DisplayName
		o.name.Position = UDim2.fromOffset(cx - 70, top.Y - 16)
	end

	local parts = {}
	if S.ESPDist then
		table.insert(parts, math.floor((Camera.CFrame.Position - root.Position).Magnitude) .. " м")
	end
	if S.ESPHealth then
		table.insert(parts, "❤ " .. math.floor(hp))
	end
	o.info.Visible = #parts > 0
	if #parts > 0 then
		o.info.Text = table.concat(parts, "  ")
		o.info.Position = UDim2.fromOffset(cx - 70, bot.Y + 2)
	end

	if S.ESPSkel then
		local list = char:FindFirstChild("UpperTorso") and R15 or R6
		for i, l in ipairs(o.lines) do
			local pair = list[i]
			if pair then
				local a = char:FindFirstChild(pair[1])
				local b = char:FindFirstChild(pair[2])
				if a and b then
					local pa = Camera:WorldToViewportPoint(a.Position)
					local pb = Camera:WorldToViewportPoint(b.Position)
					if pa.Z > 0 and pb.Z > 0 then
						setLine(l, Vector2.new(pa.X, pa.Y), Vector2.new(pb.X, pb.Y))
					else
						l.Visible = false
					end
				else
					l.Visible = false
				end
			else
				l.Visible = false
			end
		end
	else
		for _, l in ipairs(o.lines) do l.Visible = false end
	end
end

local function updateESP()
	for _, p in ipairs(Players:GetPlayers()) do
		if p ~= LP then
			local ok, err = pcall(updatePlayerESP, p)
			if not ok and not warnedESP then
				warnedESP = true
				warn("[NeonMenu] ESP: " .. tostring(err))
			end
		end
	end
end

local function hookPlayer(p)
	p.CharacterAdded:Connect(function()
		task.wait(0.3)
		pcall(applyHighlight, p)
	end)
end
for _, p in ipairs(Players:GetPlayers()) do hookPlayer(p) end
Players.PlayerAdded:Connect(hookPlayer)
Players.PlayerRemoving:Connect(function(p)
	clearHighlight(p)
	destroyESP(p)
end)

----------------------------------------------------------------
-- АИМ + ТРИГГЕР
----------------------------------------------------------------
local fovCircle = new("Frame", {
	AnchorPoint = Vector2.new(0.5, 0.5), BackgroundTransparency = 1, Visible = false,
}, gui)
round(fovCircle, 1000)
stroke(fovCircle, ACCENT, 2)

local aimLabel = new("TextLabel", {
	Size = UDim2.fromOffset(260, 16), AnchorPoint = Vector2.new(0.5, 0), BackgroundTransparency = 1,
	Font = Enum.Font.GothamBold, TextSize = 12, TextColor3 = WHITE, TextStrokeTransparency = 0.4,
	Text = "", Visible = false,
}, gui)

local function aimCenter()
	if S.PhoneMode then
		return Camera.ViewportSize / 2
	end
	return UIS:GetMouseLocation()
end

local function isVisible(part, targetChar)
	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	params.FilterDescendantsInstances = { LP.Character, targetChar }
	local origin = Camera.CFrame.Position
	return workspace:Raycast(origin, part.Position - origin, params) == nil
end

local function isEnemy(p)
	if p == LP then return false end
	if S.TeamCheck and p.Team ~= nil and p.Team == LP.Team then return false end
	return true
end

local function playerFromInstance(inst)
	local m = inst
	while m and m ~= workspace do
		local p = Players:GetPlayerFromCharacter(m)
		if p then return p, m end
		m = m.Parent
	end
	return nil
end

local function getAimPart(char)
	if S.AimHead then
		return char:FindFirstChild("Head") or char:FindFirstChild("HumanoidRootPart")
	end
	return char:FindFirstChild("UpperTorso") or char:FindFirstChild("Torso") or char:FindFirstChild("HumanoidRootPart")
end

local function getTarget(center)
	local myRoot = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
	local base = myRoot and myRoot.Position or Camera.CFrame.Position

	local best, bestScore, bestPlayer, bestDist = nil, math.huge, nil, 0
	for _, p in ipairs(Players:GetPlayers()) do
		local char = isEnemy(p) and p.Character
		if char then
			local hum = char:FindFirstChildOfClass("Humanoid")
			local part = getAimPart(char)
			if hum and part and hum.Health > 0 then
				local worldDist = (part.Position - base).Magnitude
				local pos, onScreen = Camera:WorldToViewportPoint(part.Position)
				local screenDist = (onScreen and pos.Z > 0)
					and (Vector2.new(pos.X, pos.Y) - center).Magnitude or math.huge

				local ok, score
				if S.AimNearest then
					ok = worldDist <= S.NearRange and (not S.UseFOVLimit or screenDist <= S.FOV)
					score = worldDist
				else
					ok = screenDist <= S.FOV
					score = screenDist
				end

				if ok and score < bestScore and (not S.WallCheck or isVisible(part, char)) then
					best, bestScore, bestPlayer, bestDist = part, score, p, worldDist
				end
			end
		end
	end
	return best, bestPlayer, bestDist
end

local function enemyInCrosshair(center)
	local myChar = LP.Character

	local ray = Camera:ViewportPointToRay(center.X, center.Y)
	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	params.FilterDescendantsInstances = { myChar }
	local res = workspace:Raycast(ray.Origin, ray.Direction * 2000, params)
	if res then
		local pl, model = playerFromInstance(res.Instance)
		if pl and isEnemy(pl) then
			local hum = model:FindFirstChildOfClass("Humanoid")
			if hum and hum.Health > 0 then return true end
		end
	end

	if S.TriggerRadius > 0 then
		for _, p in ipairs(Players:GetPlayers()) do
			local char = isEnemy(p) and p.Character
			if char then
				local hum = char:FindFirstChildOfClass("Humanoid")
				if hum and hum.Health > 0 then
					for _, n in ipairs({ "Head", "UpperTorso", "Torso", "HumanoidRootPart" }) do
						local part = char:FindFirstChild(n)
						if part then
							local pos, onScreen = Camera:WorldToViewportPoint(part.Position)
							if onScreen and pos.Z > 0
								and (Vector2.new(pos.X, pos.Y) - center).Magnitude <= S.TriggerRadius
								and isVisible(part, char) then
								return true
							end
						end
					end
				end
			end
		end
	end
	return false
end

local function fire(center)
	local ok = pcall(function()
		local vim = game:GetService("VirtualInputManager")
		vim:SendMouseButtonEvent(center.X, center.Y, 0, true, game, 0)
		task.delay(0.03, function()
			pcall(function() vim:SendMouseButtonEvent(center.X, center.Y, 0, false, game, 0) end)
		end)
	end)
	if not ok then
		local char = LP.Character
		local tool = char and char:FindFirstChildOfClass("Tool")
		if tool then
			tool:Activate()
			task.delay(0.05, function()
				if tool.Parent then tool:Deactivate() end
			end)
		end
	end
end

local lastShot = 0
local function runTrigger(center)
	if not S.Trigger then return false end
	if not (S.TriggerAlways or trigHeld) then return false end
	if os.clock() - lastShot < S.TriggerDelay / 1000 then return false end
	if enemyInCrosshair(center) then
		lastShot = os.clock()
		fire(center)
		return true
	end
	return false
end

----------------------------------------------------------------
-- ЛАЗЕР ИЗ ГЛАЗ (чистый визуал, только на твоём экране)
----------------------------------------------------------------
local laserFolder, laserParts

local function getLaserFolder()
	if not laserFolder or not laserFolder.Parent then
		laserFolder = new("Folder", { Name = "NeonLaser" }, workspace)
	end
	return laserFolder
end

local function ensureLaser()
	if laserParts and laserParts.beamL.Parent then return end
	local f = getLaserFolder()
	local function mk(shape, size)
		return new("Part", {
			Anchored = true, CanCollide = false, CanQuery = false, CanTouch = false, CastShadow = false,
			Material = Enum.Material.Neon, Shape = shape, Size = size, Color = S.LaserColor, Transparency = 1,
		}, f)
	end
	local p = {
		beamL = mk(Enum.PartType.Block, Vector3.new(0.3, 0.3, 1)),
		beamR = mk(Enum.PartType.Block, Vector3.new(0.3, 0.3, 1)),
		glowL = mk(Enum.PartType.Ball, Vector3.new(0.6, 0.6, 0.6)),
		glowR = mk(Enum.PartType.Ball, Vector3.new(0.6, 0.6, 0.6)),
		impact = mk(Enum.PartType.Ball, Vector3.new(1, 1, 1)),
	}
	p.sparks = new("ParticleEmitter", {
		Texture = "rbxasset://textures/particles/sparkles_main.dds",
		Lifetime = NumberRange.new(0.25, 0.55), Speed = NumberRange.new(6, 16),
		SpreadAngle = Vector2.new(180, 180), LightEmission = 1, LightInfluence = 0,
		Rate = 90, Enabled = false,
		Size = NumberSequence.new({ NumberSequenceKeypoint.new(0, 0.6), NumberSequenceKeypoint.new(1, 0) }),
		Transparency = NumberSequence.new({ NumberSequenceKeypoint.new(0, 0), NumberSequenceKeypoint.new(1, 1) }),
	}, p.impact)
	p.light = new("PointLight", { Brightness = 3, Range = 14, Enabled = false }, p.impact)
	laserParts = p
end

local function hideLaser()
	if not laserParts then return end
	for _, v in pairs(laserParts) do
		if typeof(v) == "Instance" and v:IsA("BasePart") then v.Transparency = 1 end
	end
	laserParts.sparks.Enabled = false
	laserParts.light.Enabled = false
end

local function placeBeam(beam, a, b, w, col)
	local len = (b - a).Magnitude
	if len < 0.1 then
		beam.Transparency = 1
		return
	end
	beam.Size = Vector3.new(w, w, len)
	beam.CFrame = CFrame.lookAt(a, b) * CFrame.new(0, 0, -len / 2)
	beam.Color = col
	beam.Transparency = 0.05
end

local function laserStep(dt)
	local firing = S.Laser and (S.LaserAlways or laserHeld) and not main.Visible
	local char = LP.Character
	local head = char and char:FindFirstChild("Head")
	if not (firing and head) then
		hideLaser()
		return
	end

	ensureLaser()
	local center = aimCenter()
	local ray = Camera:ViewportPointToRay(center.X, center.Y)
	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	params.FilterDescendantsInstances = { char, getLaserFolder(), workspace:FindFirstChild("NeonSteps") }
	local res = workspace:Raycast(ray.Origin, ray.Direction * S.LaserRange, params)
	local target = res and res.Position or (ray.Origin + ray.Direction * S.LaserRange)

	local t = os.clock()
	local col = S.LaserRainbow and Color3.fromHSV((t * 0.6) % 1, 1, 1) or S.LaserColor
	local w = S.LaserWidth * (1 + 0.15 * math.sin(t * 30))

	local hs = head.Size
	local eyeL = head.CFrame:PointToWorldSpace(Vector3.new(-hs.X * 0.2, hs.Y * 0.08, -hs.Z * 0.5))
	local eyeR = head.CFrame:PointToWorldSpace(Vector3.new(hs.X * 0.2, hs.Y * 0.08, -hs.Z * 0.5))

	local p = laserParts
	placeBeam(p.beamL, eyeL, target, w, col)
	placeBeam(p.beamR, eyeR, target, w, col)

	for _, g in ipairs({ { p.glowL, eyeL }, { p.glowR, eyeR } }) do
		g[1].Size = Vector3.one * (w * 2.4)
		g[1].Position = g[2]
		g[1].Color = col
		g[1].Transparency = 0.1
	end

	if res then
		p.impact.Size = Vector3.one * (w * 4 * (1 + 0.2 * math.sin(t * 40)))
		p.impact.Position = target
		p.impact.Color = col
		p.impact.Transparency = 0.15
		p.sparks.Color = ColorSequence.new(col)
		p.sparks.Enabled = true
		p.light.Color = col
		p.light.Enabled = true
	else
		p.impact.Transparency = 1
		p.sparks.Enabled = false
		p.light.Enabled = false
	end
end

----------------------------------------------------------------
-- Основной цикл аима
----------------------------------------------------------------
local warnedAim, warnedLaser = false, false
local function aimStep(dt)
	local center = aimCenter()
	local menuOpen = main.Visible

	local circleOn = S.Aim and S.ShowFOV and (not S.AimNearest or S.UseFOVLimit)
	fovCircle.Visible = circleOn
	fovCircle.Size = UDim2.fromOffset(S.FOV * 2, S.FOV * 2)
	fovCircle.Position = UDim2.fromOffset(center.X, center.Y)

	local labelText = ""
	if menuOpen and (S.Aim or S.Trigger) then
		labelText = "закрой меню (✕), чтобы работало"
	else
		if S.Trigger then
			local shot = runTrigger(center)
			labelText = shot and "🔫 огонь" or "🔫 триггер вкл"
		end

		if S.Aim then
			local active = S.AutoAim or aimHeld or UIS:IsMouseButtonPressed(Enum.UserInputType.MouseButton2)
			if active then
				local target, pl, dist = getTarget(center)
				if target then
					labelText = "🎯 " .. pl.DisplayName .. " (" .. math.floor(dist) .. " м)"
					local cf = Camera.CFrame
					local pos = cf.Position
					local goal = CFrame.lookAt(pos, target.Position)
					local a = math.clamp(S.AimPower / 100, 0.01, 1)
					local alpha = 1 - (1 - a) ^ (dt * 60)
					Camera.CFrame = CFrame.new(pos) * cf.Rotation:Lerp(goal.Rotation, alpha)
				else
					labelText = S.AimNearest and "нет цели рядом" or "нет цели в круге"
				end
			else
				labelText = S.PhoneMode and "аим: держи 🎯" or "аим: держи ПКМ"
			end
		end
	end

	aimLabel.Visible = labelText ~= "" and (S.Aim or S.Trigger)
	aimLabel.Text = labelText
	aimLabel.Position = UDim2.fromOffset(center.X, center.Y + (circleOn and S.FOV or 24) + 6)
end

pcall(RunService.UnbindFromRenderStep, RunService, "NeonAim")
RunService:BindToRenderStep("NeonAim", Enum.RenderPriority.Camera.Value + 1, function(dt)
	Camera = workspace.CurrentCamera
	if not Camera then return end

	if camFovTouched then Camera.FieldOfView = S.CamFOV end

	updateESP()

	local ok, err = pcall(aimStep, dt)
	if not ok and not warnedAim then
		warnedAim = true
		warn("[NeonMenu] Aim: " .. tostring(err))
	end

	local ok2, err2 = pcall(laserStep, dt)
	if not ok2 and not warnedLaser then
		warnedLaser = true
		warn("[NeonMenu] Laser: " .. tostring(err2))
	end
end)
