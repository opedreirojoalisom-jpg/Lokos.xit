-- Aimbot configuration
local config = {
    precision = 80, -- Headshot accuracy percentage (50-100%)
    fov = 30,       -- Field of view radius percentage (0-100%)
    smoothness = 0.1, -- Movement smoothness (lower = more precise)
    autoShoot = true, -- Auto-shoot when target locked
    sensitivity = 0.8, -- Mouse sensitivity adjustment
    targetDistance = 50, -- Max distance to lock targets
    keybind = "t" -- Toggle key
}

-- Setup variables
local player = game.Players.LocalPlayer
local character = player.Character or player.CharacterAdded:Wait()
local humanoid = character:WaitForChild("Humanoid")
local mouse = player:GetMouse()
local camera = workspace.CurrentCamera
local currentTarget = nil
local isAiming = false

-- Get closest valid target within FOV
local function findTarget()
    local enemies = {}
    
    for _, p in pairs(workspace:GetChildren()) do
        if p:IsA("Model") and p:FindFirstChild("Humanoid") and p ~= character then
            -- Skip if in same team
            if p.Team ~= character.Team then
                local root = p:FindFirstChild("HumanoidRootPart")
                if root then
                    local distance = (root.Position - camera.CFrame.Position).Magnitude
                    
                    -- Only consider targets within max distance
                    if distance <= config.targetDistance then
                        local screenPos, onScreen = camera:WorldToViewportPoint(root.Position)
                        
                        -- Check if target is within FOV
                        if onScreen then
                            local centerX = camera.ViewportSize.X / 2
                            local centerY = camera.ViewportSize.Y / 2
                            local dx = math.abs(screenPos.X - centerX)
                            local dy = math.abs(screenPos.Y - centerY)
                            
                            -- Check if within FOV circle
                            if dx <= (camera.ViewportSize.X * config.fov / 100) / 2 and 
                               dy <= (camera.ViewportSize.Y * config.fov / 100) / 2 then
                                
                                table.insert(enemies, {
                                    player = p,
                                    distance = distance,
                                    screenPos = screenPos
                                })
                            end
                        end
                    end
                end
            end
        end
    end
    
    -- Sort by distance
    table.sort(enemies, function(a, b) return a.distance < b.distance end)
    return #enemies > 0 and enemies[1] or nil
end

-- Calculate precise aim position with precision factor
local function calculateAimPosition(target)
    local head = target.player:FindFirstChild("Head")
    if not head then return nil end
    
    -- Calculate random offset based on precision
    local offset = Vector3.new()
    local precisionFactor = config.precision / 100
    
    -- Add precision-based randomness
    offset = Vector3.new(
        (math.random() * 2 - 1) * (1 - precisionFactor),
        (math.random() * 2 - 1) * (1 - precisionFactor),
        (math.random() * 2 - 1) * (1 - precisionFactor)
    )
    
    return head.Position + offset
end

-- Smoothly move mouse to aim position
local function smoothAim(target)
    local aimPos = calculateAimPosition(target)
    if not aimPos then return end
    
    local screenPos, onScreen = camera:WorldToViewportPoint(aimPos)
    
    if onScreen then
        -- Calculate mouse delta with sensitivity
        local deltaX = (screenPos.X - mouse.X) * config.sensitivity
        local deltaY = (screenPos.Y - mouse.Y) * config.sensitivity
        
        -- Apply smoothing
        deltaX = deltaX * config.smoothness
        deltaY = deltaY * config.smoothness
        
        -- Move mouse
        mouse.X = mouse.X + deltaX
        mouse.Y = mouse.Y + deltaY
        
        -- Auto-shoot if enabled
        if config.autoShoot then
            local shootButton = mouse.Button1Down
            if not shootButton then
                mouse.Button1Down:Fire()
            end
        end
    end
end

-- Toggle aimbot with keybind
mouse.KeyDown:Connect(function(key)
    if string.lower(key) == config.keybind then
        isAiming = not isAiming
        print("Aimbot " .. (isAiming and "enabled" or "disabled"))
    end
end)

-- Main aimbot loop
game:GetService("RunService").Heartbeat:Connect(function()
    if isAiming then
        local target = findTarget()
        if target then
            smoothAim(target)
        end
    end
end)

-- Create visual FOV indicator
local fovIndicator = Instance.new("ImageLabel")
fovIndicator.Image = "rbxasset://textures/ui/SelectionCircle.png"
fovIndicator.Size = UDim2.new(0, 0, 0, 0)
fovIndicator.Visible = false
fovIndicator.BackgroundTransparency = 1
fovIndicator.Parent = player:WaitForChild("PlayerGui")

-- Update FOV indicator size
local function updateFOVIndicator()
    if isAiming then
        local size = camera.ViewportSize
        local radius = math.min(size.X, size.Y) * config.fov / 100 / 2
        fovIndicator.Size = UDim2.new(0, radius * 2, 0, radius * 2)
        fovIndicator.Visible = true
    else
        fovIndicator.Visible = false
    end
end

-- Update indicator on resolution change
camera:GetPropertyChangedSignal("ViewportSize"):Connect(updateFOVIndicator)
updateFOVIndicator()

print("Aimbot loaded! Press T to toggle.")
