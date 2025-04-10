local Players = game:GetService("Players")
local Debris = game:GetService("Debris")
local TweenService = game:GetService("TweenService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local BaseAbility = {}
BaseAbility.__index = BaseAbility

function BaseAbility.new(player)
	local self = setmetatable({}, BaseAbility)
	self.Player = player
	self.Cooldown = 5
	self.LastUsed = 0
	self.Name = "Unnamed Ability"
	self.Damage = 0
	return self
end

function BaseAbility:CanUse()
	return tick() - self.LastUsed >= self.Cooldown
end

function BaseAbility:Activate()
	warn(self.Name .. " is missing an activation method.")
end

function BaseAbility:Use()
	if self:CanUse() then
		self.LastUsed = tick()
		self:Activate()
	end
end

function BaseAbility:CreateProjectile(startPos, direction, speed, size, color, lifetime)
	local part = Instance.new("Part")
	part.Shape = Enum.PartType.Ball
	part.Size = size or Vector3.new(2, 2, 2)
	part.Material = Enum.Material.Neon
	part.Color = color or Color3.new(1, 0.4, 0)
	part.CFrame = CFrame.new(startPos)
	part.CanCollide = false
	part.Anchored = false
	part.Velocity = direction.Unit * speed
	part.Parent = workspace
	Debris:AddItem(part, lifetime or 5)
	return part
end

function BaseAbility:CreateAOE(position, radius, damage)
	for _, obj in ipairs(workspace:GetDescendants()) do
		if obj:IsA("Humanoid") and obj.Parent:FindFirstChild("HumanoidRootPart") then
			local dist = (obj.Parent.HumanoidRootPart.Position - position).Magnitude
			if dist <= radius then
				obj:TakeDamage(damage)
			end
		end
	end
end

local Fireball = setmetatable({}, BaseAbility)
Fireball.__index = Fireball

function Fireball.new(player)
	local self = BaseAbility.new(player)
	setmetatable(self, Fireball)
	self.Cooldown = 6
	self.Name = "Fireball"
	self.Damage = 25
	return self
end

function Fireball:Activate()
	local char = self.Player.Character
	if not char or not char:FindFirstChild("Head") then return end

	local direction = char.Head.CFrame.LookVector
	local startPos = char.Head.Position + direction * 3
	local projectile = self:CreateProjectile(startPos, direction, 100, Vector3.new(2, 2, 2), Color3.fromRGB(255, 100, 0), 4)

	projectile.Touched:Connect(function(hit)
		if hit and hit.Parent and hit.Parent ~= char then
			local humanoid = hit.Parent:FindFirstChildOfClass("Humanoid")
			if humanoid then
				humanoid:TakeDamage(self.Damage)
			end
			self:CreateAOE(projectile.Position, 5, self.Damage / 2)
			projectile:Destroy()
		end
	end)
end

local Dash = setmetatable({}, BaseAbility)
Dash.__index = Dash

function Dash.new(player)
	local self = BaseAbility.new(player)
	setmetatable(self, Dash)
	self.Cooldown = 3
	self.Name = "Dash"
	return self
end

function Dash:Activate()
	local char = self.Player.Character
	if not char or not char:FindFirstChild("HumanoidRootPart") then return end

	local hrp = char.HumanoidRootPart
	local dashVector = hrp.CFrame.LookVector * 50
	local goal = hrp.Position + dashVector

	local tween = TweenService:Create(hrp, TweenInfo.new(0.2, Enum.EasingStyle.Sine, Enum.EasingDirection.Out), {Position = goal})
	tween:Play()
end

local Heal = setmetatable({}, BaseAbility)
Heal.__index = Heal

function Heal.new(player)
	local self = BaseAbility.new(player)
	setmetatable(self, Heal)
	self.Cooldown = 8
	self.Name = "Heal"
	self.HealAmount = 30
	return self
end

function Heal:Activate()
	local char = self.Player.Character
	if not char then return end
	local humanoid = char:FindFirstChildOfClass("Humanoid")
	if humanoid then
		humanoid.Health = math.min(humanoid.MaxHealth, humanoid.Health + self.HealAmount)
	end
end

Players.PlayerAdded:Connect(function(player)
	player.Chatted:Connect(function(msg)
		if msg == "!fireball" then
			local ability = Fireball.new(player)
			ability:Use()
		elseif msg == "!dash" then
			local ability = Dash.new(player)
			ability:Use()
		elseif msg == "!heal" then
			local ability = Heal.new(player)
			ability:Use()
		end
	end)
end)
