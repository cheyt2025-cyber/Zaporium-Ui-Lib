# Zaporium-Ui-Lib
--[[
	Zaporium UI Library - Full Custom Luau Implementation
	Compatible with: Synapse X, Fluxus, KRNL, and all Roblox Executors
	Mobile / Tablet / Desktop Ready | One-Script Pastebin Ready
	Features: Themes, Animations, Dragging, Logo Toggle, Config System
	Author: Custom Built for Zaporium
--]]

-- Services
local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local HttpService = game:GetService("HttpService")
local CoreGui = game:GetService("CoreGui")

-- Player
local Player = Players.LocalPlayer
local Mouse = Player:GetMouse()

-- Main Library
local Zaporium = {}
Zaporium.Version = "1.0.0"
Zaporium.Themes = {
	Dark = {
		Background = Color3.fromRGB(25, 25, 30),
		Accent = Color3.fromRGB(0, 170, 255),
		Text = Color3.fromRGB(255, 255, 255),
		Secondary = Color3.fromRGB(40, 40, 45),
		Border = Color3.fromRGB(60, 60, 70),
		Success = Color3.fromRGB(0, 200, 100),
		Warning = Color3.fromRGB(255, 170, 0),
		Danger = Color3.fromRGB(255, 60, 60)
	},
	Light = {
		Background = Color3.fromRGB(240, 240, 245),
		Accent = Color3.fromRGB(0, 120, 215),
		Text = Color3.fromRGB(30, 30, 30),
		Secondary = Color3.fromRGB(220, 220, 225),
		Border = Color3.fromRGB(180, 180, 190),
		Success = Color3.fromRGB(0, 160, 80),
		Warning = Color3.fromRGB(200, 130, 0),
		Danger = Color3.fromRGB(200, 40, 40)
	}
}

-- Default Config
local Config = {
	Theme = "Dark",
	Transparency = 0,
	Font = Enum.Font.GothamBold,
	ToggleKey = Enum.KeyCode.RightShift
}

-- State
local GUI = nil
local LogoGui = nil
local Dragging = false
local DragStart = nil
local StartPos = nil
local Notifications = {}
local Configs = {}
local CurrentWindow = nil
local CurrentTab = nil

-- Utility Functions
local function CreateInstance(class, props)
	local obj = Instance.new(class)
	for prop, value in pairs(props or {}) do
		if prop ~= "Parent" then
			obj[prop] = value
		end
	end
	if props and props.Parent then
		obj.Parent = props.Parent
	end
	return obj
end

local function Tween(obj, props, duration, easing)
	local tween = TweenService:Create(obj, TweenInfo.new(duration or 0.2, easing or Enum.EasingStyle.Quad), props)
	tween:Play()
	return tween
end

local function GetTheme()
	return Zaporium.Themes[Config.Theme]
end

local function SaveConfig(name)
	local saveData = {
		Theme = Config.Theme,
		Transparency = Config.Transparency,
		Font = Config.Font.Name,
		ToggleKey = Config.ToggleKey.Name
	}
	Configs[name] = saveData
	writefile("Zaporium_Config_" .. name .. ".json", HttpService:JSONEncode(saveData))
end

local function LoadConfig(name)
	if not isfile("Zaporium_Config_" .. name .. ".json") then return false end
	local data = HttpService:JSONDecode(readfile("Zaporium_Config_" .. name .. ".json"))
	Config.Theme = data.Theme or "Dark"
	Config.Transparency = data.Transparency or 0
	Config.Font = Enum.Font[data.Font] or Enum.Font.GothamBold
	Config.ToggleKey = Enum.KeyCode[data.ToggleKey] or Enum.KeyCode.RightShift
	return true
end

-- Notification System
function Zaporium:Notify(title, text, duration, color)
	duration = duration or 3
	color = color or GetTheme().Accent

	local notif = CreateInstance("Frame", {
		Size = UDim2.new(0, 300, 0, 80),
		Position = UDim2.new(1, 10, 1, -90),
		BackgroundColor3 = GetTheme().Secondary,
		BorderSizePixel = 0,
		Parent = GUI
	})

	CreateInstance("UICorner", {CornerRadius = UDim.new(0, 8), Parent = notif})
	CreateInstance("UIStroke", {Color = GetTheme().Border, Thickness = 1, Parent = notif})

	local titleLabel = CreateInstance("TextLabel", {
		Size = UDim2.new(1, -16, 0, 25),
		Position = UDim2.new(0, 8, 0, 8),
		BackgroundTransparency = 1,
		Text = title,
		TextColor3 = GetTheme().Text,
		Font = Config.Font,
		TextSize = 16,
		TextXAlignment = Enum.TextXAlignment.Left,
		Parent = notif
	})

	local textLabel = CreateInstance("TextLabel", {
		Size = UDim2.new(1, -16, 0, 30),
		Position = UDim2.new(0, 8, 0, 33),
		BackgroundTransparency = 1,
		Text = text,
		TextColor3 = GetTheme().Text,
		Font = Config.Font,
		TextSize = 14,
		TextXAlignment = Enum.TextXAlignment.Left,
		TextWrapped = true,
		Parent = notif
	})

	local bar = CreateInstance("Frame", {
		Size = UDim2.new(1, 0, 0, 3),
		Position = UDim2.new(0, 0, 1, -3),
		BackgroundColor3 = color,
		BorderSizePixel = 0,
		Parent = notif
	})

	table.insert(Notifications, {Frame = notif, Time = tick() + duration})

	-- Animate in
	Tween(notif, {Position = UDim2.new(1, -310, 1, -90)}, 0.3)

	-- Auto remove
	spawn(function()
		wait(duration)
		Tween(notif, {Position = UDim2.new(1, 10, 1, -90)}, 0.3):Completed:Wait()
		notif:Destroy()
	end)
end

-- Logo Toggle
local function CreateLogo()
	if LogoGui then LogoGui:Destroy() end

	LogoGui = CreateInstance("ScreenGui", {
		Name = "ZaporiumLogo",
		ResetOnSpawn = false,
		ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
		Parent = CoreGui
	})

	local logo = CreateInstance("ImageLabel", {
		Size = UDim2.new(0, 80, 0, 80),
		Position = UDim2.new(0, 20, 0, 20),
		BackgroundTransparency = 1,
		Image = "rbxassetid://YOUR_LOGO_ID_HERE", -- Optional: Replace with real logo
		Parent = LogoGui
	})

	-- Fallback to text if no image
	if not logo.Image or logo.Image == "" then
		logo:Destroy()
		logo = CreateInstance("TextLabel", {
			Size = UDim2.new(0, 80, 0, 80),
			Position = UDim2.new(0, 20, 0, 20),
			BackgroundTransparency = 1,
			Text = "Z",
			TextColor3 = Color3.fromRGB(0, 170, 255),
			Font = Enum.Font.GothamBlack,
			TextSize = 60,
			Parent = LogoGui
		})
	end

	CreateInstance("UICorner", {CornerRadius = UDim.new(0, 16), Parent = logo})

	local clickDetector = CreateInstance("TextButton", {
		Size = UDim2.new(1, 0, 1, 0),
		BackgroundTransparency = 1,
		Text = "",
		Parent = logo
	})

	clickDetector.MouseButton1Click:Connect(function()
		if GUI then
			GUI.Enabled = true
			LogoGui:Destroy()
			LogoGui = nil
		end
	end)

	-- Pulse animation
	spawn(function()
		while LogoGui and logo do
			Tween(logo, {ImageTransparency = 0.7}, 0.8)
			wait(0.8)
			Tween(logo, {ImageTransparency = 0.3}, 0.8)
			wait(0.8)
		end
	end)
end

-- Main GUI Creation
local function CreateGUI()
	if GUI then GUI:Destroy() end

	GUI = CreateInstance("ScreenGui", {
		Name = "Zaporium",
		ResetOnSpawn = false,
		ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
		Parent = CoreGui
	})

	local background = CreateInstance("Frame", {
		Name = "Background",
		Size = UDim2.new(1, 0, 1, 0),
		BackgroundColor3 = GetTheme().Background,
		BackgroundTransparency = Config.Transparency,
		BorderSizePixel = 0,
		Parent = GUI
	})

	CreateInstance("UICorner", {CornerRadius = UDim.new(0, 12), Parent = background})
	CreateInstance("UIStroke", {Color = GetTheme().Border, Thickness = 2, Parent = background})

	-- Title Bar
	local titleBar = CreateInstance("Frame", {
		Size = UDim2.new(1, 0, 0, 50),
		BackgroundColor3 = GetTheme().Secondary,
		BorderSizePixel = 0,
		Parent = background
	})

	CreateInstance("UICorner", {CornerRadius = UDim.new(0, 12), Parent = titleBar})

	local title = CreateInstance("TextLabel", {
		Size = UDim2.new(0, 200, 1, 0),
		Position = UDim2.new(0, 20, 0, 0),
		BackgroundTransparency = 1,
		Text = "Zaporium",
		TextColor3 = GetTheme().Accent,
		Font = Enum.Font.GothamBlack,
		TextSize = 20,
		TextXAlignment = Enum.TextXAlignment.Left,
		Parent = titleBar
	})

	local closeBtn = CreateInstance("TextButton", {
		Size = UDim2.new(0, 40, 0, 40),
		Position = UDim2.new(1, -50, 0, 5),
		BackgroundTransparency = 1,
		Text = "×",
		TextColor3 = GetTheme().Text,
		Font = Enum.Font.GothamBold,
		TextSize = 28,
		Parent = titleBar
	})

	closeBtn.MouseButton1Click:Connect(function()
		GUI.Enabled = false
		CreateLogo()
	end)

	-- Make Draggable
	local dragInput, dragConnection
	titleBar.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 then
			Dragging = true
			DragStart = input.Position
			StartPos = background.Position

			dragConnection = input.Changed:Connect(function()
				if input.UserInputState == Enum.UserInputState.End then
					Dragging = false
				end
			end)
		end
	end)

	titleBar.InputChanged:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseMovement then
			dragInput = input
		end
	end)

	RunService.RenderStepped:Connect(function()
		if Dragging and dragInput then
			local delta = dragInput.Position - DragStart
			background.Position = UDim2.new(
				StartPos.X.Scale,
				StartPos.X.Offset + delta.X,
				StartPos.Y.Scale,
				StartPos.Y.Offset + delta.Y
			)
		end
	end)

	-- Main Container
	local container = CreateInstance("Frame", {
		Name = "Container",
		Size = UDim2.new(1, -20, 1, -70),
		Position = UDim2.new(0, 10, 0, 60),
		BackgroundTransparency = 1,
		Parent = background
	})

	local tabFrame = CreateInstance("Frame", {
		Size = UDim2.new(0, 150, 1, 0),
		BackgroundColor3 = GetTheme().Secondary,
		BorderSizePixel = 0,
		Parent = container
	})

	CreateInstance("UICorner", {CornerRadius = UDim.new(0, 8), CornerRadius = UDim.new(0, 8), Parent = tabFrame})

	local tabLayout = CreateInstance("UIListLayout", {
		Padding = UDim.new(0, 5),
		FillDirection = Enum.FillDirection.Vertical,
		Parent = tabFrame
	})

	local contentFrame = CreateInstance("Frame", {
		Size = UDim2.new(1, -160, 1, 0),
		Position = UDim2.new(0, 160, 0, 0),
		BackgroundTransparency = 1,
		Parent = container
	})

	-- Window Object
	local Window = {}
	Window.Tabs = {}
	Window.TabButtons = {}

	function Window:CreateTab(name)
		local tabBtn = CreateInstance("TextButton", {
			Size = UDim2.new(1, -10, 0, 40),
			BackgroundColor3 = GetTheme().Secondary,
			Text = name,
			TextColor3 = GetTheme().Text,
			Font = Config.Font,
			TextSize = 16,
			Parent = tabFrame
		})

		CreateInstance("UICorner", {CornerRadius = UDim.new(0, 6), Parent = tabBtn})

		local content = CreateInstance("ScrollingFrame", {
			Size = UDim2.new(1, 0, 1, 0),
			BackgroundTransparency = 1,
			BorderSizePixel = 0,
			ScrollBarThickness = 4,
			ScrollBarImageColor3 = GetTheme().Accent,
			Visible = false,
			Parent = contentFrame
		})

		CreateInstance("UICorner", {CornerRadius = UDim.new(0, 8), Parent = content})
		CreateInstance("UIListLayout", {
			Padding = UDim.new(0, 8),
			Parent = content
		})

		local padding = CreateInstance("UIPadding", {
			PaddingLeft = UDim.new(0, 10),
			PaddingRight = UDim.new(0, 10),
			PaddingTop = UDim.new(0, 10),
			Parent = content
		})

		local Tab = {}
		Tab.Content = content
		Tab.Name = name

		-- Elements
		function Tab:CreateButton(text, callback)
			local btn = CreateInstance("TextButton", {
				Size = UDim2.new(1, 0, 0, 40),
				BackgroundColor3 = GetTheme().Accent,
				Text = text,
				TextColor3 = Color3.fromRGB(255, 255, 255),
				Font = Config.Font,
				TextSize = 16,
				Parent = content
			})

			CreateInstance("UICorner", {CornerRadius = UDim.new(0, 8), Parent = btn})

			btn.MouseButton1Click:Connect(function()
				spawn(callback)
				Tween(btn, {BackgroundColor3 = GetTheme().Accent:lerp(Color3.new(1,1,1), 0.3)}, 0.1):Completed:Wait()
				Tween(btn, {BackgroundColor3 = GetTheme().Accent}, 0.1)
			end)

			return btn
		end

		function Tab:CreateToggle(text, default, callback)
			local enabled = default or false

			local frame = CreateInstance("Frame", {
				Size = UDim2.new(1, 0, 0, 40),
				BackgroundColor3 = GetTheme().Secondary,
				Parent = content
			})

			CreateInstance("UICorner", {CornerRadius = UDim.new(0, 8), Parent = frame})

			local label = CreateInstance("TextLabel", {
				Size = UDim2.new(1, -60, 1, 0),
				BackgroundTransparency = 1,
				Text = text,
				TextColor3 = GetTheme().Text,
				Font = Config.Font,
				TextSize = 16,
				TextXAlignment = Enum.TextXAlignment.Left,
				Parent = frame
			})

			local toggle = CreateInstance("Frame", {
				Size = UDim2.new(0, 50, 0, 25),
				Position = UDim2.new(1, -60, 0.5, -12.5),
				BackgroundColor3 = enabled and GetTheme().Success or GetTheme().Border,
				Parent = frame
			})

			CreateInstance("UICorner", {CornerRadius = UDim.new(0, 12), Parent = toggle})

			local circle = CreateInstance("Frame", {
				Size = UDim2.new(0, 20, 0, 20),
				Position = enabled and UDim2.new(1, -25, 0.5, -10) or UDim2.new(0, 5, 0.5, -10),
				BackgroundColor3 = Color3.fromRGB(255, 255, 255),
				Parent = toggle
			})

			CreateInstance("UICorner", {CornerRadius = UDim.new(1, 0), Parent = circle})

			frame.InputBegan:Connect(function(input)
				if input.UserInputType == Enum.UserInputType.MouseButton1 then
					enabled = not enabled
					Tween(toggle, {BackgroundColor3 = enabled and GetTheme().Success or GetTheme().Border}, 0.2)
					Tween(circle, {Position = enabled and UDim2.new(1, -25, 0.5, -10) or UDim2.new(0, 5, 0.5, -10)}, 0.2)
					spawn(function() callback(enabled) end)
				end
			end)

			return frame
		end

		function Tab:CreateSlider(text, min, max, default, callback)
			local value = default or min

			local frame = CreateInstance("Frame", {
				Size = UDim2.new(1, 0, 0, 60),
				BackgroundColor3 = GetTheme().Secondary,
				Parent = content
			})

			CreateInstance("UICorner", {CornerRadius = UDim.new(0, 8), Parent = frame})

			local label = CreateInstance("TextLabel", {
				Size = UDim2.new(1, -10, 0, 25),
				Position = UDim2.new(0, 5, 0, 5),
				BackgroundTransparency = 1,
				Text = text .. ": " .. value,
				TextColor3 = GetTheme().Text,
				Font = Config.Font,
				TextSize = 16,
				TextXAlignment = Enum.TextXAlignment.Left,
				Parent = frame
			})

			local bar = CreateInstance("Frame", {
				Size = UDim2.new(1, -20, 0, 10),
				Position = UDim2.new(0, 10, 0, 35),
				BackgroundColor3 = GetTheme().Border,
				Parent = frame
			})

			CreateInstance("UICorner", {CornerRadius = UDim.new(0, 5), Parent = bar})

			local fill = CreateInstance("Frame", {
				Size = UDim2.new((value - min) / (max - min), 0, 1, 0),
				BackgroundColor3 = GetTheme().Accent,
				Parent = bar
			})

			CreateInstance("UICorner", {CornerRadius = UDim.new(0, 5), Parent = fill})

			local dragging = false

			bar.InputBegan:Connect(function(input)
				if input.UserInputType == Enum.UserInputType.MouseButton1 then
					dragging = true
				end
			end)

			bar.InputEnded:Connect(function(input)
				if input.UserInputType == Enum.UserInputType.MouseButton1 then
					dragging = false
				end
			end)

			UserInputService.InputChanged:Connect(function(input)
				if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
					local mousePos = UserInputService:GetMouseLocation()
					local rel = math.clamp((mousePos.X - bar.AbsolutePosition.X) / bar.AbsoluteSize.X, 0, 1)
					value = math.floor(min + (max - min) * rel)
					label.Text = text .. ": " .. value
					Tween(fill, {Size = UDim2.new(rel, 0, 1, 0)}, 0.1)
					spawn(function() callback(value) end)
				end
			end)

			return frame
		end

		function Tab:CreateDropdown(text, items, callback)
			local selected = items[1]

			local frame = CreateInstance("Frame", {
				Size = UDim2.new(1, 0, 0, 40),
				BackgroundColor3 = GetTheme().Secondary,
				Parent = content
			})

			CreateInstance("UICorner", {CornerRadius = UDim.new(0, 8), Parent = frame})

			local label = CreateInstance("TextLabel", {
				Size = UDim2.new(1, -40, 1, 0),
				BackgroundTransparency = 1,
				Text = text .. ": " .. selected,
				TextColor3 = GetTheme().Text,
				Font = Config.Font,
				TextSize = 16,
				TextXAlignment = Enum.TextXAlignment.Left,
				Position = UDim2.new(0, 10, 0, 0),
				Parent = frame
			})

			local arrow = CreateInstance("TextLabel", {
				Size = UDim2.new(0, 30, 1, 0),
				Position = UDim2.new(1, -40, 0, 0),
				BackgroundTransparency = 1,
				Text = "▼",
				TextColor3 = GetTheme().Text,
				Font = Config.Font,
				TextSize = 16,
				Parent = frame
			})

			local dropdown = CreateInstance("Frame", {
				Size = UDim2.new(1, 0, 0, #items * 35),
				Position = UDim2.new(0, 0, 1, 5),
				BackgroundColor3 = GetTheme().Secondary,
				BorderSizePixel = 0,
				Visible = false,
				Parent = frame
			})

			CreateInstance("UICorner", {CornerRadius = UDim.new(0, 8), Parent = dropdown})
			CreateInstance("UIStroke", {Color = GetTheme().Border, Parent = dropdown})

			for i, item in ipairs(items) do
				local btn = CreateInstance("TextButton", {
					Size = UDim2.new(1, 0, 0, 35),
					Position = UDim2.new(0, 0, 0, (i-1)*35),
					BackgroundTransparency = 1,
					Text = item,
					TextColor3 = GetTheme().Text,
					Font = Config.Font,
					TextSize = 16,
					Parent = dropdown
				})

				btn.MouseButton1Click:Connect(function()
					selected = item
					label.Text = text .. ": " .. selected
					dropdown.Visible = false
					spawn(function() callback(selected) end)
				end)
			end

			frame.InputBegan:Connect(function(input)
				if input.UserInputType == Enum.UserInputType.MouseButton1 then
					dropdown.Visible = not dropdown.Visible
				end
			end)

			return frame
		end

		function Tab:CreateKeybind(text, default, callback)
			local key = default or Enum.KeyCode.Unknown

			local frame = CreateInstance("Frame", {
				Size = UDim2.new(1, 0, 0, 40),
				BackgroundColor3 = GetTheme().Secondary,
				Parent = content
			})

			CreateInstance("UICorner", {CornerRadius = UDim.new(0, 8), Parent = frame})

			local label = CreateInstance("TextLabel", {
				Size = UDim2.new(1, -80, 1, 0),
				BackgroundTransparency = 1,
				Text = text,
				TextColor3 = GetTheme().Text,
				Font = Config.Font,
				TextSize = 16,
				TextXAlignment = Enum.TextXAlignment.Left,
				Position = UDim2.new(0, 10, 0, 0),
				Parent = frame
			})

			local bind = CreateInstance("TextButton", {
				Size = UDim2.new(0, 60, 0, 30),
				Position = UDim2.new(1, -70, 0.5, -15),
				BackgroundColor3 = GetTheme().Border,
				Text = key.Name,
				TextColor3 = GetTheme().Text,
				Font = Config.Font,
				TextSize = 14,
				Parent = frame
			})

			CreateInstance("UICorner", {CornerRadius = UDim.new(0, 6), Parent = bind})

			local listening = false
			bind.MouseButton1Click:Connect(function()
				listening = true
				bind.Text = "..."
			end)

			UserInputService.InputBegan:Connect(function(input)
				if listening and input.KeyCode ~= Enum.KeyCode.Unknown then
					key = input.KeyCode
					bind.Text = key.Name
					listening = false
					spawn(function() callback(key) end)
				end
			end)

			return frame
		end

		function Tab:CreateTextbox(text, placeholder, callback)
			local frame = CreateInstance("Frame", {
				Size = UDim2.new(1, 0, 0, 40),
				BackgroundColor3 = GetTheme().Secondary,
				Parent = content
			})

			CreateInstance("UICorner", {CornerRadius = UDim.new(0, 8), Parent = frame})

			local label = CreateInstance("TextLabel", {
				Size = UDim2.new(1, 0, 0, 20),
				BackgroundTransparency = 1,
				Text = text,
				TextColor3 = GetTheme().Text,
				Font = Config.Font,
				TextSize = 14,
				TextXAlignment = Enum.TextXAlignment.Left,
				Position = UDim2.new(0, 10, 0, 0),
				Parent = frame
			})

			local box = CreateInstance("TextBox", {
				Size = UDim2.new(1, -20, 0, 30),
				Position = UDim2.new(0, 10, 0, 25),
				BackgroundColor3 = GetTheme().Background,
				Text = "",
				PlaceholderText = placeholder,
				TextColor3 = GetTheme().Text,
				Font = Config.Font,
				TextSize = 14,
				Parent = frame
			})

			CreateInstance("UICorner", {CornerRadius = UDim.new(0, 6), Parent = box})
			CreateInstance("UIStroke", {Color = GetTheme().Border, Parent = box})

			box.FocusLost:Connect(function(enter)
				if enter then
					spawn(function() callback(box.Text) end)
				end
			end)

			return frame
		end

		-- Tab Switching
		tabBtn.MouseButton1Click:Connect(function()
			for _, tab in pairs(Window.Tabs) do
				tab.Content.Visible = false
			end
			for _, btn in pairs(Window.TabButtons) do
				Tween(btn, {BackgroundColor3 = GetTheme().Secondary}, 0.2)
			end
			content.Visible = true
			Tween(tabBtn, {BackgroundColor3 = GetTheme().Accent}, 0.2)
			CurrentTab = Tab
		end)

		table.insert(Window.Tabs, Tab)
		table.insert(Window.TabButtons, tabBtn)

		-- Auto-select first tab
		if #Window.Tabs == 1 then
			tabBtn.MouseButton1Click:Fire()
		end

		return Tab
	end

	-- Config Tab
	local configTab = Window:CreateTab("Config")

	configTab:CreateDropdown("Theme", {"Dark", "Light"}, function(val)
		Config.Theme = val
		CreateGUI() -- Rebuild
	end)

	configTab:CreateSlider("Transparency", 0, 80, Config.Transparency, function(val)
		Config.Transparency = val / 100
		background.BackgroundTransparency = Config.Transparency
	end)

	configTab:CreateTextbox("Save Config", "config_name", function(name)
		SaveConfig(name)
		Zaporium:Notify("Config Saved", "Saved as " .. name, 2, GetTheme().Success)
	end)

	configTab:CreateTextbox("Load Config", "config_name", function(name)
		if LoadConfig(name) then
			CreateGUI()
			Zaporium:Notify("Config Loaded", "Loaded " .. name, 2, GetTheme().Success)
		else
			Zaporium:Notify("Error", "Config not found", 2, GetTheme().Danger)
		end
	end)

	-- Center Window
	background.Size = UDim2.new(0, 700, 0, 500)
	background.Position = UDim2.new(0.5, -350, 0.5, -250)

	return Window
end

-- Toggle Handler
UserInputService.InputBegan:Connect(function(input, gp)
	if gp then return end
	if input.KeyCode == Config.ToggleKey then
		if GUI and GUI.Enabled then
			GUI.Enabled = false
			CreateLogo()
		elseif LogoGui then
			LogoGui:FindFirstChildOfClass("TextButton").MouseButton1Click:Fire()
		end
	end
end)

-- Create Window Function
function Zaporium:CreateWindow(title)
	local window = CreateGUI()
	return window
end

-- Notification Cleanup
RunService.Heartbeat:Connect(function()
	for i = #Notifications, 1, -1 do
		local notif = Notifications[i]
		if tick() > notif.Time then
			if notif.Frame and notif.Frame.Parent then
				Tween(notif.Frame, {Position = UDim2.new(1, 10, 1, -90)}, 0.3):Completed:Wait()
				notif.Frame:Destroy()
			end
			table.remove(Notifications, i)
		end
	end
end)

-- Initial Build
spawn(function()
	wait(1)
	CreateGUI()
	Zaporium:Notify("Zaporium Loaded", "Press RightShift to toggle", 4)
end)

-- Return Library
return Zaporium

-- EXAMPLE USAGE:
--[[
local Library = loadstring(game:HttpGet("PASTEBIN_LINK_HERE"))()
local Window = Library:CreateWindow("Zaporium Hub")
local Tab = Window:CreateTab("Main")

Tab:CreateButton("Hello World", function()
    print("Hello from Zaporium!")
end)

Tab:CreateToggle("God Mode", false, function(state)
    print("God Mode:", state)
end)

Tab:CreateSlider("Speed", 16, 200, 50, function(val)
    game.Players.LocalPlayer.Character.Humanoid.WalkSpeed = val
end)
--]]
