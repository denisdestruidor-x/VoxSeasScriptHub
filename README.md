local TweenService = game:GetService("TweenService")
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local player = Players.LocalPlayer
local character = player.Character or player.CharacterAdded:Wait()
local humanoid = character:FindFirstChild("Humanoid") or character:WaitForChild("Humanoid")

_G.EquipTool = false
_G.ToolName = "Combat"

function attack(npc)
	local args = {
		"Effects",
		1,
	}
	game:GetService("ReplicatedStorage")
		:WaitForChild("BetweenSides")
		:WaitForChild("Remotes")
		:WaitForChild("Events")
		:WaitForChild("ToolsEvent")
		:FireServer(unpack(args))

	local args = {
		"DealDamage",
		{
			CallTime = 1753518567.1118534,
			Results = {
				npc,
			},
			Combo = 1,
			DelayTime = 0.2,
		},
	}
	game:GetService("ReplicatedStorage")
		:WaitForChild("BetweenSides")
		:WaitForChild("Remotes")
		:WaitForChild("Events")
		:WaitForChild("CombatEvent")
		:FireServer(unpack(args))
end

function EquipTool(toolName)
	local tool = game.Players.LocalPlayer.Backpack:FindFirstChild(toolName)
	if tool then
		if not _G.EquipTool then
			_G.EquipTool = true
			game.Players.LocalPlayer.Character.Humanoid:EquipTool(tool)
		else
			_G.EquipTool = false
			tool.Parent = game.Players.LocalPlayer.Backpack
		end
	else
		print("Tool not found in backpack.")
	end
end

function KillNpc(npc)
	local distance = (npc.HumanoidRootPart.Position - game.Players.LocalPlayer.Character.HumanoidRootPart.Position).Magnitude
	local velocity = distance / 300
	local tweenInfo = TweenInfo.new(velocity, Enum.EasingStyle.Linear, Enum.EasingDirection.InOut)
	local tween = TweenService:Create(
		game.Players.LocalPlayer.Character.HumanoidRootPart,
		tweenInfo,
		{ CFrame = npc.HumanoidRootPart.CFrame * CFrame.new(0, 6, 0) }
	)
	_G.mon = npc.Name
	_G.Dis = CFrame.new(0, 6, 0)
	getgenv().auto = true
	
	local firstTime = true
	tween:Play()
	while getgenv().auto do
		task.wait(0.01)
		if _G.EquipTool == false then
			EquipTool(_G.ToolName)
		end
		game.Players.LocalPlayer.Character.HumanoidRootPart.CFrame = npc.HumanoidRootPart.CFrame * CFrame.new(0, 6, 0)
		attack(npc)
	end
end

KillNpc(workspace.Playability.Enemys["Foosha Village"].Bandit70)
