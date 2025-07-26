local _ENV = (getgenv or getrenv or getfenv)()

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")
local Players = game:GetService("Players")

local Player = Players.LocalPlayer

local CombatEvent = ReplicatedStorage.BetweenSides.Remotes.Events.CombatEvent
local ToolEvent = ReplicatedStorage.BetweenSides.Remotes.Events.ToolsEvent
local Enemys = workspace.Playability.Enemys

local Connections = _ENV.rz_connections or {}
do
	_ENV.rz_connections = Connections

	for i = 1, #Connections do
		Connections[i]:Disconnect()
	end

	table.clear(Connections)
end

local function IsAlive(Character)
	if Character then
		local Humanoid = Character:FindFirstChildOfClass("Humanoid")
		return Humanoid and Humanoid.Health > 0
	end
end

local BodyVelocity
do
	BodyVelocity = Instance.new("BodyVelocity")
	BodyVelocity.Velocity = Vector3.zero
	BodyVelocity.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
	BodyVelocity.P = 1000

	if _ENV.tween_bodyvelocity then
		_ENV.tween_bodyvelocity:Destroy()
	end

	_ENV.tween_bodyvelocity = BodyVelocity

	local CanCollideObjects = {}

	local function AddObjectToBaseParts(Object)
		if Object:IsA("BasePart") and Object.CanCollide then
			table.insert(CanCollideObjects, Object)
		end
	end

	local function RemoveObjectsFromBaseParts(BasePart)
		local index = table.find(CanCollideObjects, BasePart)

		if index then
			table.remove(CanCollideObjects, index)
		end
	end

	local function NewCharacter(Character)
		table.clear(CanCollideObjects)

		for _, Object in Character:GetDescendants() do
			AddObjectToBaseParts(Object)
		end
		Character.DescendantAdded:Connect(AddObjectToBaseParts)
		Character.DescendantRemoving:Connect(RemoveObjectsFromBaseParts)
	end

	table.insert(Connections, Player.CharacterAdded:Connect(NewCharacter))
	task.spawn(NewCharacter, Player.Character)

	local function NoClipOnStepped(Character)
		if _ENV.OnFarm then
			for i = 1, #CanCollideObjects do
				CanCollideObjects[i].CanCollide = false
			end
		elseif Character.PrimaryPart and not Character.PrimaryPart.CanCollide then
			for i = 1, #CanCollideObjects do
				CanCollideObjects[i].CanCollide = true
			end
		end
	end

	local function UpdateVelocityOnStepped(Character)
		local BasePart = Character:FindFirstChild("UpperTorso")
		local Humanoid = Character:FindFirstChild("Humanoid")
		local BodyVelocity = _ENV.tween_bodyvelocity

		if _ENV.OnFarm and BasePart and Humanoid and Humanoid.Health > 0 then
			if BodyVelocity.Parent ~= BasePart then
				BodyVelocity.Parent = BasePart
			end
		elseif BodyVelocity.Parent then
			BodyVelocity.Parent = nil
		end

		if BodyVelocity.Velocity ~= Vector3.zero and (not Humanoid or not Humanoid.SeatPart or not _ENV.OnFarm) then
			BodyVelocity.Velocity = Vector3.zero
		end
	end

	table.insert(
		Connections,
		RunService.Stepped:Connect(function()
			local Character = Player.Character

			if IsAlive(Character) then
				UpdateVelocityOnStepped(Character)
				NoClipOnStepped(Character)
			end
		end)
	)
end

local PlayerTP
do
	local TweenCreator = {}
	do
		TweenCreator.__index = TweenCreator

		local tweens = {}
		local EasingStyle = Enum.EasingStyle.Linear

		function TweenCreator.new(obj, time, prop, value)
			local self = setmetatable({}, TweenCreator)

			self.tween = TweenService:Create(obj, TweenInfo.new(time, EasingStyle), { [prop] = value })
			self.tween:Play()
			self.value = value
			self.object = obj

			if tweens[obj] then
				tweens[obj]:destroy()
			end

			tweens[obj] = self
			return self
		end

		function TweenCreator:destroy()
			self.tween:Pause()
			self.tween:Destroy()

			tweens[self.object] = nil
			setmetatable(self, nil)
		end

		function TweenCreator:stopTween(obj)
			if obj and tweens[obj] then
				tweens[obj]:destroy()
			end
		end
	end

	local function TweenStopped()
		if not BodyVelocity.Parent and IsAlive(Player.Character) then
			TweenCreator:stopTween(Player.Character:FindFirstChild("HumanoidRootPart"))
		end
	end

	local lastCFrame = nil
	local lastTeleport = 0
	local TweenSpeed = 75

	PlayerTP = function(TargetCFrame)
		if not IsAlive(Player.Character) or not Player.Character.PrimaryPart then
			return false
		elseif (tick() - lastTeleport) <= 1 and lastCFrame == TargetCFrame then
			return false
		end

		local Character = Player.Character
		local Humanoid = Character.Humanoid
		local PrimaryPart = Character.PrimaryPart

		if Humanoid.Sit then
			Humanoid.Sit = false
			return
		end

		lastTeleport = tick()
		lastCFrame = TargetCFrame
		_ENV.OnFarm = true

		local teleportPosition = TargetCFrame.Position
		local Distance = (PrimaryPart.Position - teleportPosition).Magnitude

		if Distance < TweenSpeed then
			PrimaryPart.CFrame = TargetCFrame
			return TweenCreator:stopTween(PrimaryPart)
		end

		TweenCreator.new(PrimaryPart, Distance / TweenSpeed, "CFrame", TargetCFrame)
	end

	table.insert(Connections, BodyVelocity:GetPropertyChangedSignal("Parent"):Connect(TweenStopped))
end

local CurrentTime = workspace:GetServerTimeNow()

local function DealDamage(Enemies)
	CurrentTime += 1

	local Combo = 4
	ToolEvent:FireServer("Effects", Combo)
	CombatEvent:FireServer("DealDamage", {
		CallTime = CurrentTime,
		DelayTime = 0,
		Combo = Combo,
		Results = Enemies,
	})
end

local function GetClosestEnemies()
	local Islands = Enemys:GetChildren()

	local DistanceFromIsland, ClosestIsland = math.huge

	for i = 1, #Islands do
		local Island = Islands[i]
		local Model = Island:FindFirstChild("Humanoid", true)

		if Model and Model.RootPart then
			local Distance = Player:DistanceFromCharacter(Model.RootPart.Position)

			if Distance < DistanceFromIsland then
				DistanceFromIsland, ClosestIsland = Distance, Island
			end
		end
	end

	if ClosestIsland then
		local _Enemies = ClosestIsland:GetChildren()
		local Enemies = {}

		for i = 1, #_Enemies do
			local Enemy = _Enemies[i]
			local RootPart = Enemy:FindFirstChild("HumanoidRootPart")

			if Enemy:GetAttribute("Respawned") and Player:DistanceFromCharacter(RootPart.Position) < 2500 then
				table.insert(Enemies, Enemy)
			end
		end

		return Enemies
	end
end

local function BringEnemies(Enemies, Target)
	for _, Enemy in Enemies do
		local RootPart = Enemy:FindFirstChild("HumanoidRootPart")

		if RootPart then
			RootPart.Size = Vector3.one * 40
			RootPart.CFrame = Target
		end
	end

	pcall(sethiddenproperty, Player, "SimulationRadius", math.huge)
end

local Libary =
	loadstring(game:HttpGet("https://raw.githubusercontent.com/tlredz/Library/refs/heads/main/V5/Source.lua"))()
local Window = Libary:MakeWindow({ "Vox seas Hub", "discord: none", "rz-VoxSeas.json" })

local MainTab = Window:MakeTab({ "Farm", "Home" })

do
	MainTab:AddSection("Farming")
	MainTab:AddToggle({
		"Kill Closests Mobs",
		false,
		function(Value)
			_ENV.OnFarm = Value

			while task.wait() and _ENV.OnFarm do
				local Enemies = GetClosestEnemies()
				if not Enemies or #Enemies == 0 then
					continue
				end

				for _, Enemy in Enemies do
					local HumanoidRootPart = Enemy:FindFirstChild("HumanoidRootPart")

					if HumanoidRootPart then
						PlayerTP(HumanoidRootPart.CFrame + Vector3.yAxis * 10)
						DealDamage(Enemies)
						break
					end
				end
			end
		end,
	})
end
