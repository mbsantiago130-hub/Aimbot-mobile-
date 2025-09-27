-- AutoAimToggleAndAssist (LocalScript) -- put inside AimUI ScreenGui
-- Requires: ReplicatedStorage.RequestLockOn, ConfirmLockOn, RequestFire
-- Assumes you already have a button named "AimbotToggleV3Button" in the same ScreenGui
-- and a "ReticleFrame" UI element (optional).

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TweenService = game:GetService("TweenService")

local player = Players.LocalPlayer
local gui = script.Parent
local camera = workspace.CurrentCamera

local RequestLockOn = ReplicatedStorage:WaitForChild("RequestLockOn")
local ConfirmLockOn = ReplicatedStorage:WaitForChild("ConfirmLockOn")
local RequestFire = ReplicatedStorage:WaitForChild("RequestFire")

-- UI references (create or find)
local toggleButton = gui:FindFirstChild("AimbotToggleV3Button")
if not toggleButton then
    toggleButton = Instance.new("TextButton")
    toggleButton.Name = "AimbotToggleV3Button"
    toggleButton.Size = UDim2.fromOffset(140,34)
    toggleButton.Position = UDim2.new(0,10,0,10)
    toggleButton.Text = "Aimbot GUI v3: OFF"
    toggleButton.Parent = gui
end

local reticle = gui:FindFirstChild("ReticleFrame")

-- Config
local SCAN_INTERVAL = 0.18
local LOCK_RANGE = 140
local MAX_ANGLE_RAD = math.rad(22)
local SMOOTH_SPEED_SCAN = 7       -- smoothing while automatic lock active
local SMOOTH_SPEED_AIM = 22       -- stronger smoothing when preparing to shoot
local LOCK_DURATION = 4
local FIRE_KEY = Enum.UserInputType.MouseButton1
local MAX_AIM_ASSIST_TIME = 0.30  -- seconds to try to align camera before firing
local AIM_ANGLE_THRESHOLD = math.rad(1.5) -- stop aiming when under this angle

-- State
local autoAimEnabled = false
local lastScan = 0
local lockedTargetPlayer = nil -- updated via ConfirmLockOn
local locking = false
local lockEndsAt = 0

-- Utility: find candidate target similar to previous functions
local function getClosestTarget()
    if not camera then return nil end
    local best, bestScore = nil, math.huge
    for _, p in pairs(Players:GetPlayers()) do
        if p ~= player and p.Character and p.Character:FindFirstChild("HumanoidRootPart") and p.Character:FindFirstChildOfClass("Humanoid") then
            local hum = p.Character:FindFirstChildOfClass("Humanoid")
            if hum and hum.Health > 0 then
                local hrp = p.Character.HumanoidRootPart
                local dir = hrp.Position - camera.CFrame.Position
                local dist = dir.Magnitude
                if dist <= LOCK_RANGE then
                    local angle = math.acos( math.clamp(camera.CFrame.LookVector:Dot(dir.unit), -1, 1) )
                    if angle <= MAX_ANGLE_RAD then
                        local score = dist + angle * 100
                        if score < bestScore then
                            bestScore = score
                            best = p
                        end
                    end
                end
            end
        end
    end
    return best
end

-- ConfirmLockOn handler (server authoritative): track locked target and duration
ConfirmLockOn.OnClientEvent:Connect(function(success, targetUserId, duration)
    if not success then
        lockedTargetPlayer = nil
        locking = false
        if reticle then reticle.Visible = false end
        return
    end
    local target = Players:GetPlayerByUserId(targetUserId)
    if target and target.Character and target.Character:FindFirstChild("HumanoidRootPart") then
        lockedTargetPlayer = target
        locking = true
        lockEndsAt = tick() + (duration or LOCK_DURATION)
        if reticle then
            reticle.Visible = true
        end
    end
end)

-- Background scanning loop: when auto-aim enabled, request locks periodically
spawn(function()
    while true do
        local dt = task.wait(0.06)
        if autoAimEnabled then
            lastScan = lastScan + dt
            if lastScan >= SCAN_INTERVAL then
                lastScan = 0
                local candidate = getClosestTarget()
                if candidate then
                    -- request server validation
                    pcall(function() RequestLockOn:FireServer(candidate.UserId) end)
                end
            end
        else
            task.wait(0.18)
        end
    end
end)

-- Smooth camera towards a target HRP over multiple frames, using dt-based exponential lerp
-- returns when angle <= threshold or timeout reached
local function aimAssistToTarget(targetHRP, speed, timeout, angleThreshold)
    if not camera or not targetHRP then return end
    local startTime = tick()
    local completed = false
    local conn
    conn = RunService.RenderStepped:Connect(function(dt)
        if not camera or not targetHRP or not targetHRP.Parent then
            completed = true
            conn:Disconnect()
            return
        end
        local camC = camera.CFrame
        local desired = CFrame.new(camC.Position, targetHRP.Position)
        local alpha = 1 - math.exp(-speed * dt)
        local newC = camC:Lerp(desired, alpha)
        camera.CFrame = CFrame.new(camC.Position, (newC * CFrame.new(0,0,-1)).Position)

        -- compute current angle to target
        local dir = (targetHRP.Position - camera.CFrame.Position).Unit
        local ang = math.acos(math.clamp(camera.CFrame.LookVector:Dot(dir), -1, 1))
        if ang <= (angleThreshold or AIM_ANGLE_THRESHOLD) then
            completed = true
            conn:Disconnect()
            return
        end
        if tick() - startTime >= (timeout or MAX_AIM_ASSIST_TIME) then
            completed = true
            conn:Disconnect()
            return
        end
    end)
    -- yield until finished
    while not completed do task.wait() end
end

-- Fire handler: if auto-aim enabled and we have a locked target, aim a little then fire.
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.UserInputType == FIRE_KEY then
        -- If auto-aim ON and we have a locked target, do short aim assist before firing
        if autoAimEnabled and locking and lockedTargetPlayer and lockedTargetPlayer.Character and lockedTargetPlayer.Character:FindFirstChild("HumanoidRootPart") then
            local hrp = lockedTargetPlayer.Character.HumanoidRootPart
            -- quick aim smoothing (blocking short time): higher speed for snappy feel
            aimAssistToTarget(hrp, SMOOTH_SPEED_AIM, MAX_AIM_ASSIST_TIME, AIM_ANGLE_THRESHOLD)
            -- After aiming, send fire request to server (server will validate)
            local camC = camera.CFrame
            pcall(function() RequestFire:FireServer(lockedTargetPlayer.UserId, camC.Position, camC.LookVector) end)
        else
            -- Not auto-aiming or no lock: send normal fire request (server should validate)
            local camC = camera.CFrame
            pcall(function() RequestFire:FireServer(nil, camC.Position, camC.LookVector) end)
        end
    end
end)

-- Toggle button behavior
local function setButtonVisual(on)
    toggleButton.Text = "Aimbot GUI v3: " .. (on and "ON" or "OFF")
    local color = on and Color3.fromRGB(24,120,70) or Color3.fromRGB(40,40,40)
    TweenService:Create(toggleButton, TweenInfo.new(0.12), {BackgroundColor3 = color}):Play()
    if reticle then
        reticle.Visible = on and (locking == true) or false
    end
end

toggleButton.MouseButton1Click:Connect(function()
    autoAimEnabled = not autoAimEnabled
    setButtonVisual(autoAimEnabled)
    -- If enabling and we already have target candidate, request lock immediately
    if autoAimEnabled then
        local candidate = getClosestTarget()
        if candidate then
            pcall(function() RequestLockOn:FireServer(candidate.UserId) end)
        end
    else
        -- turning off: hide reticle
        if reticle then reticle.Visible = false end
    end
end)

-- Ensure UI initial state
setButtonVisual(autoAimEnabled)
