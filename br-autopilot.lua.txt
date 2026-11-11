--[[
    British Railways Autopilot (Simplified Controls) - v4.11
    Direct traction/service-brake control with a measured-speed feedback controller.
    - Verified standstill: requires exact 0 MPH standstill, waits 1.2s pause, then opens doors
    - Native door verification: polls game's native door state, retries until confirmed OPEN before entering boarding
    - Accurate station stopping at platform & car stop markers
    - Timetable departure & passenger boarding (20s min + scheduled departure)
    - Buffer & Red signal detection
    - Fixed premature Terminus detection at penultimate station
--]]
if STATE and STATE.onCleanup then
    STATE.onCleanup(function()
        local guard = workspace:FindFirstChild("BR_Autopilot_RunningGuard")
        if guard then guard:Destroy() end
    end)
end
local runOnActor = run_on_actor or function(actor, src)
    return loadstring(src)()
end
runOnActor(game:GetService("ReplicatedFirst").src, [==[
local CONFIG = {
    -- Hotkey to toggle autopilot ON / OFF
    ToggleKey = Enum.KeyCode.P,
    -- Speed settings (MPH)
    MaxCruisingSpeed = 125,      -- Maximum speed cap (track limit will govern)
    CautionSpeed = 25,           -- Speed under Caution (Yellow) signal
    MarkerSearchSpeed = 30,      -- Fallback approach speed while the exact marker is unavailable
    StationCrawlSpeed = 10,      -- Fallback ONLY at a platform with no exact marker/buffer data
    -- Stopping Distances (Miles: 1 mile = 2640 studs)
    MarkerStopDist = 0.003,      -- Distance to stop at Car Stop marker board (~8 studs)
    BufferStopDist = 0.015,      -- Distance to stop before buffer stop (~40 studs)
    SignalStopDist = 0.015,      -- Distance to stop before Red signal (~40 studs)
    -- Station & Passenger Timings (Seconds)
    DoorOpenDelay = 1.2,         -- Wait time at complete standstill (0 MPH) before opening doors
    DoorRetryInterval = 10,      -- One toggle attempt every 10s at a stationary station stop
    -- The game's timetable/boarding interlocks decide whether a toggle is accepted.
    DoorCloseWait = 3.5,         -- Seconds for door closing animation & locking
    -- Safety & Automation
    AntiAFK = true,              -- Keep the client active on LocalPlayer.Idled
    AutoAWS = true,              -- Auto-acknowledge AWS warnings immediately
    AutoDRA = true,              -- Auto-reset DRA if active
    AutoNextRoute = true,        -- Auto-select next/reverse route at terminus
    PreferReverseRoute = true,   -- Prefer reverse route back to origin station
    AutoDeclineGuard = false,     -- Auto-decline incoming guard requests to prevent popups from blocking UI
    -- Automatic role selection: driver movement or guard procedure/ticketing.
    AutoGuard = true,
    AutoGuardRequests = true,
    GuardRequestSearchInterval = 5,
    GuardRequestInterval = 15,  -- Minimum delay between different requests
    GuardRequestTimeout = 15,
    GuardRequestRetryBackoff = 120, -- Do not immediately request a rejected train again
    AutoDriverGuard = true,    -- Driver side of guarded-train door/bell procedures
    DriverGuardActionInterval = 0.75,
    GuardSpawnRecovery = true, -- Let the native menu recover after a failed guard spawn
    GuardSpawnTimeout = 15,    -- Never wait forever on Request Accepted
    GuardAutoDispatch = true,
    GuardAutoTickets = true,
    GuardActionInterval = 0.75,
    GuardTicketInterval = 1.0,
    GuardAutoPanel = true,      -- Find and open the panel when guard duties require it
    GuardAutoTeleport = true,   -- Move through native carriage-relative navigation
    GuardPanelSearchInterval = 1.0,
    GuardTeleportInterval = 2.0, -- Retry only after verifying position / seat state
    GuardPanelRange = 4.0,       -- Native panel proximity, in studs
    -- Dynamic Train Physics & Braking
    DynamicPhysicsBraking = true, -- Use native/profile physics; false uses the conservative fixed deceleration
    DynamicBrakingMargin = 1.12, -- Reserve on the service-brake stopping model; keep >= 1
    BrakeResponseTime = 0.20,    -- Seconds of controller/brake response, plus the measured loop time
    ControlInterval = 0.05,      -- Braking control period (seconds)
    StopToleranceMeters = 0.5,   -- Capture the stop point within this distance, then command exact zero
    FallbackDecel = 1.35,        -- m/s^2 if train physics is unavailable
    InitialBrakeEfficiency = 0.85, -- Fraction of conservative profile braking; learned at full service
    BrakeResponseLag = 0.65,    -- Reserve for application/traction rundown (seconds)
    SpeedGain = 0.85,           -- Speed error to acceleration, 1/s
    AccelerationDamping = 0.20, -- Damp measured acceleration around a steady speed
    CruiseSpeedGain = 0.30,     -- Gentler speed correction while cruising
    BrakeReleaseInterval = 0.65, -- Stronger braking is immediate; release partial brakes steadily
    BrakeNotchHysteresis = 0.18,
    ThrottleNotchInterval = 0.65, -- Minimum time between ordinary traction notch changes
    NotchHysteresis = 0.70,     -- Schmitt threshold, in notches (must exceed 0.5)
    ThrottleSlewPerSecond = 0.25,
    TractionLookAhead = 1.2,    -- Cut traction before rundown carries the train above target
    MinimumCoastSeconds = 2.0,
    CruiseRestartBandMPH = 1.0, -- Coast until this far below target, then reapply power
    CruiseUpperBandMPH = 0.15,
    CoastLeadMPH = 0.15,
    CruiseBrakeStartMPH = 0.15, -- Braking hysteresis relative to the permitted limit
    CruiseBrakeReleaseMPH = 0.15,
    CruiseMarginMPH = 0.7,      -- Aim below the speed limit to allow for response/steps
    SpeedDeadbandMPH = 0.25,
    ServiceBrake = 0.8,         -- Native full service; 1.0 is emergency braking
    BrakeSampleWindow = 0.6,
    BrakeSettleTime = 1.0,      -- Learn only sustained full-service applications

    AdaptiveBraking = true,
}
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local VirtualInputManager = game:GetService("VirtualInputManager")
local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")
-- Singleton Guard: Terminate any previous running instance immediately
local oldGuard = workspace:FindFirstChild("BR_Autopilot_RunningGuard")
if oldGuard then
    oldGuard:Destroy()
    task.wait(0.2)
end
local GuardInstance = Instance.new("Folder")
GuardInstance.Name = "BR_Autopilot_RunningGuard"
GuardInstance.Parent = workspace
local isRunning = true
GuardInstance.Destroying:Connect(function()
    isRunning = false
end)
-- Clean up previous GUIs
for _, g in ipairs(PlayerGui:GetChildren()) do
    if g.Name == "BR_Autopilot_HUD" then
        g:Destroy()
    end
end
local Modules = game:GetService("ReplicatedFirst").src.Modules
local GameplayStates = require(game:GetService("ReplicatedFirst").src.Data.GameplayStates)
local function isGuarding()
    return GameplayStates.isGuarding == true and GameplayStates.guardedTrain ~= nil
end
local TrainHandling = require(Modules.Gameplay.TrainHandling)
local Inputs = require(Modules.Core.Inputs)
local Triggers = require(Modules.Core.Triggers)
local InputHandler = require(Modules.Core.Inputs.Handler)
local DriverGuardHandler = require(Modules.Gameplay.Guarding.driverHandler)
local utils = require(game:GetService("ReplicatedStorage").Modules.utils)
-- The game's guard join waits forever for Character/PrimaryPart. Bound that
-- wait before entering its original initializer so its normal failure path
-- can close the accepted-request screen and return to the menu.
local GuardJoinStatus = {Pending = false, Deadline = 0, LastError = ""}

local function makeGuardSpawnStart(nativeStart, dependencies)
    return function(trainName, ...)
        if not dependencies.enabled() then return nativeStart(trainName, ...) end
        local deadline = dependencies.now() + dependencies.timeout()
        GuardJoinStatus.Pending, GuardJoinStatus.Deadline = true, deadline
        GuardJoinStatus.LastError = ""
        local reason = "Waiting for character"
        while dependencies.alive() do
            local ready, message, invalid = dependencies.ready(trainName)
            if ready then
                GuardJoinStatus.Pending = false
                return nativeStart(trainName, ...)
            end
            reason = message or reason
            if invalid or dependencies.now() >= deadline then break end
            dependencies.sleep(0.1)
        end
        GuardJoinStatus.Pending = false
        GuardJoinStatus.LastError = reason
        dependencies.failed(reason)
        return false -- Native Accept handler already restores the menu on false.
    end
end

local GuardingModule = require(Modules.Gameplay.Guarding)
local nativeGuardStart = GuardingModule.startGuarding
local boundedGuardStart = makeGuardSpawnStart(nativeGuardStart, {
    now = os.clock,
    sleep = task.wait,
    alive = function() return isRunning end,
    enabled = function() return CONFIG.GuardSpawnRecovery end,
    timeout = function() return CONFIG.GuardSpawnTimeout end,
    ready = function(trainName)
        if GameplayStates.isDriving then return false, "Already driving", true end
        local character = LocalPlayer.Character
        if not character or not character.Parent or not character.PrimaryPart
            or not character:FindFirstChildOfClass("Humanoid") then
            return false, "Guard character did not spawn"
        end
        local trains = workspace:FindFirstChild("Trains")
        local hidden = game:GetService("ReplicatedStorage"):FindFirstChild("HiddenTrainStorage")
        local train = (trains and trains:FindFirstChild(trainName))
            or (hidden and hidden:FindFirstChild(trainName))
        if train and train:GetAttribute("Removing") then return false, "Train despawned", true end
        if not train or train:GetAttribute("Guard") ~= LocalPlayer.Name then
            return false, "Waiting for guard assignment"
        end
        return true
    end,
    failed = function(reason)
        warn("[BR Autopilot] Guard join cancelled: " .. reason .. ". Returning to menu.")
    end,
})
GuardingModule.startGuarding = boundedGuardStart
GuardInstance.Destroying:Connect(function()
    if GuardingModule.startGuarding == boundedGuardStart then
        GuardingModule.startGuarding = nativeGuardStart
    end
end)


-- The native acceptance handler closes Guard/back, assuming the request
-- was sent from the Guard menu. Use that same entry point before auto-requesting.
local GuardMenu = {Pages = nil, Open = nil, NextInit = 0, SyncedTrain = nil}

local function resolveGuardMenu(now)
    if GuardMenu.Pages and GuardMenu.Open then return true end
    if now < GuardMenu.NextInit then return false end
    GuardMenu.NextInit = now + 5
    local events = game:GetService("ReplicatedStorage"):FindFirstChild("Events")
    local remote = events and events:FindFirstChild("GuardReq")
    if not remote then return false end
    for _, connection in ipairs(getconnections(remote.OnClientEvent)) do
        if type(connection.Function) == "function" then
            local pages, map = nil, nil
            for _, value in pairs(getupvalues(connection.Function)) do
                if type(value) == "table" then
                    if value.Main and value.Guard and value.back and value.Background then pages = value end
                    if value.isPending and value.requestAccepted and value.selectedTrain then map = value end
                end
            end
            if pages and map then
                local open = getfenv(connection.Function).guardButton
                if type(open) == "function" then
                    GuardMenu.Pages, GuardMenu.Open = pages, open
                    return true
                end
            end
        end
    end
    return false
end

local function prepareGuardRequestMenu(now)
    if not resolveGuardMenu(now) then return false end
    if GuardMenu.Pages.Main:get() == 0 then GuardMenu.Open() end
    return GuardMenu.Pages.Main:get() == 1 and GuardMenu.Pages.Guard:get() == 0
end

local function finishGuardMenuTransition(now, train)
    if GuardMenu.SyncedTrain == train then return end
    if not resolveGuardMenu(now) then return end
    -- Repair an already-active guard session once. Later manual menu openings remain usable.
    GuardMenu.Pages.Main:set(1)
    GuardMenu.Pages.Guard:set(1)
    GuardMenu.Pages.back:set(1)
    GuardMenu.SyncedTrain = train
end

local guardModule = DriverGuardHandler

local function readUpvalues(callback)
    if type(callback) ~= "function" then return {} end
    local ok, values = pcall(getupvalues, callback)
    return ok and type(values) == "table" and values or {}
end

local function hasGetter(value)
    return type(value) == "table" and type(value.get) == "function"
end

local function getCurrentTrainData(upvalues)
    upvalues = upvalues or readUpvalues(TrainHandling.trainSpawned)
    for _, value in pairs(upvalues) do
        if type(value) == "table" and type(value.speed) == "number"
            and typeof(value.model) == "Instance" then
            return value
        end
    end
end

local function readDriverUpvalues()
    local upvalues = readUpvalues(TrainHandling.trainSpawned)
    local signals, activityQueue
    for _, value in pairs(upvalues) do
        if type(value) == "table" then
            local info = value.signalInfo
            if type(info) == "table" and hasGetter(info.signalId) and hasGetter(info.signalType)
                and hasGetter(info.signalState) and hasGetter(info.dist) then
                signals = info
            end
            if type(value.routeRenderStepUpdate) == "function"
                and type(value.getNextStop) == "function"
                and type(value.getNextDeparture) == "function"
                and type(value.getNextActivity) == "function" then
                activityQueue = value
            end
        end
    end
    return getCurrentTrainData(upvalues), signals, activityQueue
end
local liveTrackLimit = 60
pcall(function()
    Triggers:add("LimitChanged", function(newLimit)
        if typeof(newLimit) == "number" and newLimit > 0 then
            liveTrackLimit = newLimit
        end
    end)
end)
local function getLiveTrackLimit(td)
    local panel = PlayerGui:FindFirstChild("UI") and PlayerGui.UI:FindFirstChild("Drive") and PlayerGui.UI.Drive:FindFirstChild("Panel")
    if panel then
        for _, c in ipairs(panel:GetChildren()) do
            for _, d in ipairs(c:GetDescendants()) do
                if d:IsA("TextLabel") and d.TextColor3.R > 0.8 and d.TextColor3.G < 0.5 then
                    local num = tonumber(d.Text)
                    if num and num > 0 then
                        liveTrackLimit = num
                        return num
                    end
                end
            end
        end
    end
    if liveTrackLimit and liveTrackLimit > 0 then
        return liveTrackLimit
    end
    if td and td.limit and td.limit > 0 then
        return td.limit
    end
    return 60
end
local State = {
    Enabled = true,
    LastRole = "NONE",
    ControlTime = 0,
    CurrentPhase = "CRUISING",
    StationPhase = "NONE", -- "NONE", "STOPPING", "OPENING_DOORS", "BOARDING", "CLOSING_DOORS", "DEPARTED"
    TargetSpeed = 0,
    CurrentSpeed = 0,
    ExactSpeed = math.huge, -- Keep emergency braking until an actual speed sample is available.
    SpeedLimit = 15,
    NextStop = "None",
    NextStopDist = 999,
    ScheduledDepTime = 0,
    CurrentSignalId = "None",
    CurrentSignalState = "Clear",
    CurrentSignalDist = 999,
    FullStopTime = 0,
    DoorAttemptTime = nil,
    DoorStandstillSince = nil,
    DoorClosedSince = nil,
    BoardingStartTime = 0,
    DoorsOpen = false,
    CurrentStopStation = "",
    CurrentStopIsTerminus = false,
    DepartedStation = "",
    OriginStation = "",
    TerminusStation = "",
    RouteModel = nil,
    RouteSignature = nil,
    LastRouteCheckTime = 0,
    TrainClass = "Unknown",
    TrainMass = 0,
    TrainBrakingForce = 0,
    DecelRate = 1.35, -- m/s^2 (~3.0 mph/s)
    AccelRate = 1.0,  -- m/s^2
    DynBrakeDist = 0, -- miles needed to stop from current speed
    ControlInterval = CONFIG.ControlInterval,
    LastControlTime = nil,
    BrakeReferenceSpeed = 0,
    ThrottleLevel = 0,
    BrakeLevel = 0,
    MeasuredAcceleration = 0,
    MeasuredDecel = 0,
    BrakeResponseSeconds = CONFIG.BrakeResponseLag,
}
local TractionControl = {
    Notch = 0, Level = 0, LastChange = -math.huge,
    Coasting = false, CoastSince = 0, Braking = false,
    LastUpdate = nil,
}
local function resetTractionControl()
    TractionControl.Notch, TractionControl.Level = 0, 0
    TractionControl.LastChange = -math.huge
    TractionControl.LastUpdate = nil
    TractionControl.Coasting, TractionControl.Braking = false, false
    TractionControl.CoastSince = 0
end
local BrakeControl = {LastChange = -math.huge}
local BrakeFeedback = {
    Decel = nil,
    Response = CONFIG.BrakeResponseLag,
}

local function resetBrakeFeedback(keepLearning)
    if not keepLearning then
        resetTractionControl()
        BrakeControl.LastChange = -math.huge
        BrakeFeedback.Decel = nil
        BrakeFeedback.Response = CONFIG.BrakeResponseLag
    end
    BrakeFeedback.LastTime = nil
    BrakeFeedback.LastSpeed = nil
    BrakeFeedback.WindowTime = nil
    BrakeFeedback.WindowSpeed = nil
    BrakeFeedback.MinBrake = nil
    BrakeFeedback.MaxThrottle = 0
    BrakeFeedback.WindowDuration = 0
    BrakeFeedback.StrongSince = nil
    BrakeFeedback.BrakingSince = nil
    State.MeasuredDecel = 0
    State.MeasuredAcceleration = 0
    State.BrakeResponseSeconds = BrakeFeedback.Response
end

-- C is the native combined controller used by Simplified. S and tap
-- controllers use separate values. Never write a target speed to the game.
local function getAppliedEffort()
    if Inputs:getThrottleType() == "C" then
        local value = Inputs:getCBTC()
        return math.clamp(value, 0, 1), math.clamp(-value, 0, 1)
    end
    local throttle = Inputs:getThrottle()
    if Inputs.config.useTapOutput or Inputs:getThrottleType() == "T" then
        throttle = Inputs.tap.state:get() ~= 0 and Inputs.tap.outputThrottle:get() or 0
    end
    return math.clamp(throttle, 0, 1), math.clamp(Inputs:getBrake(), 0, 1)
end

local function mustPreserveEmergency(existingBrake, nativeLocked, speed)
    if nativeLocked then return true end
    -- Native SPAD/AWS/manual emergency owns throttleLocked. Once it releases
    -- at a standstill, the old brake value alone must not relatch itself.
    local stopped = type(speed) == "number" and speed == speed and math.abs(speed) < 0.001
    return existingBrake > 0.81 and not stopped
end

local function applyEffort(throttle, brake)
    throttle = math.clamp(throttle or 0, 0, 1)
    brake = math.clamp(brake or 0, 0, CONFIG.ServiceBrake)
    local _, existingBrake = getAppliedEffort()
    -- Keep native door/SPAD/emergency interlocks. Inputs:clear() would
    -- clear throttleLocked, so only discard the manual movement flags.
    if mustPreserveEmergency(existingBrake, Inputs:getThrottleLocked(), State.ExactSpeed) then
        throttle = 0
        brake = math.max(brake, existingBrake)
    end
    if brake > 0 then throttle = 0 end
    local throttleSteps, brakeSteps = Inputs:getThrottleSteps(), Inputs:getBrakeSteps()
    if throttle > 0 and throttleSteps > 0 then
        throttle = math.clamp(math.max(1, math.round(throttle * throttleSteps)) / throttleSteps, 0, 1)
    end
    if brake <= CONFIG.ServiceBrake and brakeSteps > 1 then
        local step = CONFIG.ServiceBrake / (brakeSteps - 1)
        local raw = brake / step
        local requested = math.clamp(math.ceil(raw - 0.000001), 0, brakeSteps - 1)
        local current = math.round(existingBrake / step)
        local now = State.ControlTime or os.clock()
        local chosen = requested
        if requested < current then
            local nextNotch = requested == 0 and 0 or current - 1
            if now - BrakeControl.LastChange < CONFIG.BrakeReleaseInterval
                or (requested > 0 and raw > nextNotch - CONFIG.BrakeNotchHysteresis) then
                chosen = current
            else
                chosen = nextNotch
            end
        end
        brake = math.clamp(chosen * step, 0, CONFIG.ServiceBrake)
        if math.abs(brake - existingBrake) > 0.00001 then BrakeControl.LastChange = now end
    end
    if brake > 0 then throttle = 0 end
    for _, key in ipairs({"throttle", "brake", "applyBrake", "releaseBrake",
        "CBTCDrag", "VRDrag", "cabThrottleDrag", "UIThrottleDrag", "UIBrakeDrag",
        "UITapDrag", "drag3D"}) do
        Inputs.currentInputs[key] = false
    end
    local combined = throttle > 0 and throttle or -brake
    if math.abs(Inputs:getCBTC() - combined) > 0.00001 then Inputs:setCBTC(combined) end
    if math.abs(Inputs:getThrottle() - throttle) > 0.00001 then Inputs:setThrottle(throttle) end
    if math.abs(Inputs:getBrake() - brake) > 0.00001 then Inputs:setBrake(brake) end
    if throttle == 0 then
        Inputs:resetTap()
    elseif Inputs:getThrottleType() == "T" then
        Inputs.tap.outputThrottle:set(throttle)
        Inputs.tap.state:set(3) -- Tap HOLD, not another increase every frame.
        Inputs.tap.displayVal:set(3)
    end
    State.ThrottleLevel, State.BrakeLevel = getAppliedEffort()
end

local function holdStopped()
    resetTractionControl()
    State.TargetSpeed = 0
    State.BrakeReferenceSpeed = 0
    applyEffort(0, CONFIG.ServiceBrake)
end

local function pressKey(keyCode)
    pcall(function()
        VirtualInputManager:SendKeyEvent(true, keyCode, false, game)
        task.wait(0.06)
        VirtualInputManager:SendKeyEvent(false, keyCode, false, game)
    end)
end

local DriverGuardControl = {
    Train = nil, Guard = nil, LastAction = -math.huge,
    Active = false, SeenProcedure = false,
}

local function readDriverGuardContext(td)
    if not GameplayStates.isDriving or isGuarding() then return nil end
    if not td then td = getCurrentTrainData() end
    local model = td and td.model
    local guard = model and model:GetAttribute("Guard")
    if type(guard) ~= "string" or guard == "" then return nil end
    return {
        train = model, guard = guard,
        stage = DriverGuardHandler:getCurrentStage() or {},
        doors = DriverGuardHandler:getDoorState(),
    }
end


local function getDoorsStateNative()
    local ok, isOpen, atPlat, toggleFn, known = pcall(function()
        local Handler = require(Modules.Core.Inputs.Handler)
        local keyb = Handler._inputMap and Handler._inputMap.began and Handler._inputMap.began.keyboard
        local callbacks = keyb and keyb[Enum.KeyCode.T]
        local entry = callbacks and callbacks[1]
        local fn = type(entry) == "table" and entry.fn or entry
        if type(fn) == "function" then
            local values = getupvalues(fn)
            return values[1] == true, values[2] == true, values[3], type(values[1]) == "boolean"
        end
        return false, false, nil, false
    end)
    if not ok then isOpen, atPlat, toggleFn, known = false, false, nil, false end
    local guarded = readDriverGuardContext()
    if guarded then
        return guarded.doors == "Open" or guarded.doors == "Ready", atPlat, toggleFn, true
    end
    if known then return isOpen, atPlat, toggleFn, true end

    -- A failed native read is UNKNOWN, not proof that the doors are shut.
    local lockedOK, locked = pcall(function()
        return guardModule and guardModule.getDoorState and guardModule:getDoorState() == "Locked"
    end)
    if lockedOK and locked then return false, atPlat, toggleFn, true end
    local uiOK, uiOpen = pcall(function()
        local ui = PlayerGui:FindFirstChild("UI")
        local drive = ui and ui:FindFirstChild("Drive")
        local buttons = drive and drive:FindFirstChild("Buttons")
        local btn = buttons and buttons:FindFirstChild("Toggle Doors", true)
        local textBtn = btn and (btn:IsA("TextButton") and btn or btn:FindFirstChildOfClass("TextButton"))
        if textBtn then
            return textBtn.BackgroundColor3 == Color3.fromRGB(203, 137, 71)
        end
        return nil
    end)
    if uiOK and uiOpen ~= nil then return uiOpen, atPlat, toggleFn, true end
    return false, atPlat, toggleFn, false
end

local function toggleDoors()
    local _, _, toggleFn = getDoorsStateNative()
    if type(toggleFn) == "function" then
        local sent = pcall(toggleFn)
        -- Return from THIS function, not merely from inside pcall: otherwise
        -- one retry can toggle twice and undo the successful first action.
        if sent then return true end
    end
    local sent = pcall(function()
        local ui = PlayerGui:FindFirstChild("UI")
        local drive = ui and ui:FindFirstChild("Drive")
        local buttons = drive and drive:FindFirstChild("Buttons")
        local btn = buttons and buttons:FindFirstChild("Toggle Doors", true)
        local textBtn = btn and (btn:IsA("TextButton") and btn or btn:FindFirstChildOfClass("TextButton"))
        if not textBtn or not firesignal then error("Door button unavailable") end
        firesignal(textBtn.Activated)
    end)
    if sent then return true end
    pressKey(Enum.KeyCode.T)
    return false -- A key was attempted, but successful opening is checked separately.
end

local function isDoorsLocked()
    local guarded = readDriverGuardContext()
    if guarded then return guarded.doors == "Locked" end
    local isOpen, _, _, known = getDoorsStateNative()
    if isOpen or not known then return false end
    local ok, result = pcall(function()
        if guardModule and guardModule.getDoorState then
            return guardModule:getDoorState() == "Locked"
        end
        return nil
    end)
    if ok and result ~= nil then return result end
    return known and not isOpen
end

local function setupAntiAFK()
    if not CONFIG.AntiAFK then return nil end
    local lastPulse = -math.huge
    local ok, connection = pcall(function()
        return LocalPlayer.Idled:Connect(function()
            if not isRunning or not CONFIG.AntiAFK or os.clock() - lastPulse < 30 then return end
            lastPulse = os.clock()
            local pulseOK, err = pcall(function()
                local virtualUser = game:GetService("VirtualUser")
                virtualUser:CaptureController()
                local camera = workspace.CurrentCamera
                virtualUser:ClickButton2(Vector2.new(0, 0), camera and camera.CFrame or CFrame.new())
            end)
            if not pulseOK then warn("[BR Autopilot] Anti-AFK input unavailable: " .. tostring(err)) end
        end)
    end)
    if ok then return connection end
    warn("[BR Autopilot] Anti-AFK setup failed: " .. tostring(connection))
    return nil
end
local antiAfkConn = setupAntiAFK()

local function resolveButton(inst)
    if not inst then return nil end
    if inst:IsA("GuiButton") then return inst end
    return inst:FindFirstChildWhichIsA("GuiButton", true)
end
local function clickButton(button)
    if not button then
        return
    end

    pcall(function()
        if firesignal then
            firesignal(button.Activated)
        end
    end)

    pcall(function()
        if getconnections then
            for _, connection in ipairs(getconnections(button.Activated)) do
                connection:Fire()
            end
        end
    end)

    pcall(function()
        if firesignal then
            firesignal(button.MouseButton1Click)
        end
    end)

    pcall(function()
        if getconnections then
            for _, connection in ipairs(getconnections(button.MouseButton1Click)) do
                connection:Fire()
            end
        end
    end)
end

local function getActiveEndOfRoutePopup(ui)
    if not ui then
        return nil
    end

    for _, child in ipairs(ui:GetChildren()) do
        if child.Name == "EndOfRoutePopup" and child.Visible then
            local glassFrame = child:FindFirstChild("GlassFrame")
            if glassFrame and glassFrame.Visible and glassFrame.AbsoluteSize.Y > 50 then
                return child
            end
        end
    end

    return nil
end

local function getActiveRoutePicker(ui)
    if not ui then
        return nil
    end

    for _, child in ipairs(ui:GetChildren()) do
        if child.Name == "RoutePicker" and child.Visible then
            local contentOverlay = child:FindFirstChild("ContentOverlay")
            if contentOverlay and contentOverlay.Visible and child.AbsoluteSize.Y > 100 then
                return child
            end
        end
    end

    return nil
end

local function autoDeclineGuardRequests()
    if not CONFIG.AutoDeclineGuard then
        return
    end

    pcall(function()
        for _, gui in ipairs(PlayerGui:GetChildren()) do
            if gui:IsA("ScreenGui") and gui.Enabled then
                for _, descendant in ipairs(gui:GetDescendants()) do
                    if descendant:IsA("TextLabel") and descendant.Visible and descendant.Text ~= "" then
                        local text = string.lower(descendant.Text)
                        local isGuardRequest =
                            string.find(text, "guard")
                            and (
                                string.find(text, "request")
                                or string.find(text, "wants to be")
                                or string.find(text, "кондуктор")
                                or string.find(text, "запрос")
                            )

                        if isGuardRequest then
                            local container = descendant:FindFirstAncestorWhichIsA("Frame") or descendant.Parent
                            if container then
                                for _, button in ipairs(container:GetDescendants()) do
                                    if button:IsA("GuiButton") and button.Visible then
                                        local label = button:FindFirstChildWhichIsA("TextLabel")
                                        local buttonText = string.lower(
                                            button:IsA("TextButton") and button.Text
                                            or (label and label.Text or button.Name)
                                        )
                                        local isDecline =
                                            string.find(buttonText, "decline")
                                            or string.find(buttonText, "reject")
                                            or string.find(buttonText, "отклон")
                                            or string.find(buttonText, "cancel")
                                            or string.find(buttonText, "deny")
                                            or string.find(buttonText, "no")
                                        local isRedButton =
                                            button.BackgroundColor3.R > 0.6
                                            and button.BackgroundColor3.G < 0.4

                                        if isDecline or isRedButton then
                                            clickButton(button)
                                            return
                                        end
                                    end
                                end
                            end
                        end
                    end
                end
            end
        end
    end)
end

local lastRouteActionTime = 0

local function activateRouteButton(button, now)
    if not button then
        return false
    end

    lastRouteActionTime = now
    clickButton(button)
    return true
end

local function getDropdown(list)
    return list and list:FindFirstChildWhichIsA("ScrollingFrame", true)
end

local function openDropdownIfClosed(list, dropdown, now)
    if not dropdown or dropdown.AbsoluteSize.Y >= 50 then
        return false
    end

    local header = resolveButton(list:FindFirstChild("Header", true))
    return activateRouteButton(header, now)
end

local function findDropdownButton(dropdown, matcher)
    local firstButton = nil
    local matchedButton = nil

    for _, child in ipairs(dropdown:GetChildren()) do
        if
            child:IsA("GuiObject")
            and child.Name ~= "UIListLayout"
            and child.Name ~= "UIPadding"
        then
            local button = resolveButton(child)
            if button then
                firstButton = firstButton or button

                local label = child:FindFirstChildWhichIsA("TextLabel", true)
                local text = label and label.Text or child.Name

                if matcher and matcher(text) then
                    matchedButton = button
                    break
                end
            end
        end
    end

    return matchedButton, firstButton
end

local function selectDestination(destinationList, now)
    if not destinationList then
        return
    end

    local dropdown = getDropdown(destinationList)
    if openDropdownIfClosed(destinationList, dropdown, now) then
        return
    end

    if not dropdown or dropdown.AbsoluteSize.Y < 50 then
        return
    end

    local targetButton, firstButton = findDropdownButton(dropdown, function(text)
        return
            State.OriginStation ~= ""
            and string.find(
                string.lower(text),
                string.lower(State.OriginStation),
                1,
                true
            ) ~= nil
    end)

    local destinationButton =
        (CONFIG.PreferReverseRoute and targetButton)
        or firstButton

    activateRouteButton(destinationButton, now)
end

local function selectPattern(patternList, now)
    local dropdown = getDropdown(patternList)
    if openDropdownIfClosed(patternList, dropdown, now) then
        return
    end

    if not dropdown or dropdown.AbsoluteSize.Y < 50 then
        return
    end

    local preferredButton, firstButton = findDropdownButton(dropdown, function(text)
        local lowerText = string.lower(text)
        return
            not string.find(lowerText, "empty")
            and not string.find(lowerText, "ecs")
    end)

    activateRouteButton(preferredButton or firstButton, now)
end

local function startSelectedRoute(continueButton, now)
    if not activateRouteButton(continueButton, now) then
        return
    end

    if State.TerminusStation ~= "" then
        State.OriginStation = State.TerminusStation
        State.TerminusStation = ""
    end

    resetBrakeFeedback(true)
    State.DoorAttemptTime = nil
    State.DoorStandstillSince = nil
    State.DoorClosedSince = nil
    State.DoorsOpen = false
    State.CurrentStopStation = ""
    State.CurrentStopIsTerminus = false
    State.StationPhase = "NONE"
    State.CurrentPhase = "CRUISING"
    State.DepartedStation = ""
    State.FullStopTime = 0
end

local function autoSelectNextRoute(popup, picker)
    autoDeclineGuardRequests()

    local now = os.clock()
    if now - lastRouteActionTime < 0.6 then
        return
    end

    if popup then
        local chooseButton = resolveButton(
            popup:FindFirstChild("ChooseNewRoute", true)
        )
        if activateRouteButton(chooseButton, now) then
            return
        end
    end

    if not picker then
        return
    end

    local contentOverlay = picker:FindFirstChild("ContentOverlay", true)
    if not contentOverlay then
        return
    end

    local destinationList = contentOverlay:FindFirstChild("DestList")
    local patternList = contentOverlay:FindFirstChild("PatternList")
    local buttonFrame = contentOverlay:FindFirstChild("ButtonFrame")

    if not patternList then
        selectDestination(destinationList, now)
        return
    end

    local continueFrame = buttonFrame and buttonFrame:FindFirstChild("Continue")
    local continueButton = resolveButton(continueFrame)
    local spawnText = continueButton and continueButton:FindFirstChild("SpawnText")
    local isReady = spawnText and spawnText.TextTransparency < 0.2

    if not isReady then
        selectPattern(patternList, now)
        return
    end

    startSelectedRoute(continueButton, now)
end

-- Create HUD
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "BR_Autopilot_HUD"
ScreenGui.ResetOnSpawn = false
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Size = UDim2.new(0, 320, 0, 210)
MainFrame.Position = UDim2.new(0, 20, 0, 80)
MainFrame.BackgroundColor3 = Color3.fromRGB(20, 22, 28)
MainFrame.BackgroundTransparency = 0.15
MainFrame.BorderSizePixel = 0
MainFrame.Active = true
MainFrame.Draggable = true
MainFrame.Parent = ScreenGui
local UICorner = Instance.new("UICorner")
UICorner.CornerRadius = UDim.new(0, 10)
UICorner.Parent = MainFrame
local UIStroke = Instance.new("UIStroke")
UIStroke.Color = Color3.fromRGB(60, 65, 80)
UIStroke.Thickness = 1.2
UIStroke.Parent = MainFrame
local TitleBar = Instance.new("Frame")
TitleBar.Name = "TitleBar"
TitleBar.Size = UDim2.new(1, 0, 0, 34)
TitleBar.BackgroundColor3 = Color3.fromRGB(30, 34, 44)
TitleBar.BackgroundTransparency = 0.2
TitleBar.BorderSizePixel = 0
TitleBar.Parent = MainFrame
local TitleCorner = Instance.new("UICorner")
TitleCorner.CornerRadius = UDim.new(0, 10)
TitleCorner.Parent = TitleBar
local TitleLabel = Instance.new("TextLabel")
TitleLabel.Size = UDim2.new(1, -95, 1, 0)
TitleLabel.Position = UDim2.new(0, 12, 0, 0)
TitleLabel.BackgroundTransparency = 1
TitleLabel.Text = "BR AUTOPILOT (SIMPLIFIED)"
TitleLabel.TextColor3 = Color3.fromRGB(230, 235, 245)
TitleLabel.TextSize = 13
TitleLabel.FontFace = Font.fromName("Montserrat", Enum.FontWeight.Bold)
TitleLabel.TextXAlignment = Enum.TextXAlignment.Left
TitleLabel.Parent = TitleBar
local ToggleBtn = Instance.new("TextButton")
ToggleBtn.Name = "ToggleBtn"
ToggleBtn.Size = UDim2.new(0, 75, 0, 24)
ToggleBtn.Position = UDim2.new(1, -82, 0, 5)
ToggleBtn.BackgroundColor3 = Color3.fromRGB(45, 180, 80)
ToggleBtn.Text = "ON [P]"
ToggleBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
ToggleBtn.TextSize = 11
ToggleBtn.FontFace = Font.fromName("Montserrat", Enum.FontWeight.Bold)
ToggleBtn.Parent = TitleBar
local BtnCorner = Instance.new("UICorner")
BtnCorner.CornerRadius = UDim.new(0, 6)
BtnCorner.Parent = ToggleBtn
local Content = Instance.new("Frame")
Content.Name = "Content"
Content.Size = UDim2.new(1, -20, 1, -45)
Content.Position = UDim2.new(0, 10, 0, 40)
Content.BackgroundTransparency = 1
Content.Parent = MainFrame
local function createRow(name, yPos)
    local lbl = Instance.new("TextLabel")
    lbl.Name = name
    lbl.Size = UDim2.new(1, 0, 0, 20)
    lbl.Position = UDim2.new(0, 0, 0, yPos)
    lbl.BackgroundTransparency = 1
    lbl.TextColor3 = Color3.fromRGB(200, 205, 220)
    lbl.TextSize = 11
    lbl.FontFace = Font.fromName("Montserrat", Enum.FontWeight.Medium)
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Parent = Content
    return lbl
end
local StatusLabel = createRow("StatusLabel", 0)
local SpeedRow = createRow("SpeedRow", 24)
local StationRow = createRow("StationRow", 48)
local SignalRow = createRow("SignalRow", 72)
local TimeRow = createRow("TimeRow", 96)
local PhysicsRow = createRow("PhysicsRow", 120)
pcall(function()
    ScreenGui.Parent = PlayerGui
end)
local function setAutopilot(enabled)
    State.Enabled = enabled
    resetBrakeFeedback(false)
    State.DoorAttemptTime = nil
    State.DoorStandstillSince = nil
    State.DoorClosedSince = nil
    State.DoorsOpen = false
    State.CurrentStopStation = ""
    State.CurrentStopIsTerminus = false
    if enabled then
        ToggleBtn.BackgroundColor3 = Color3.fromRGB(45, 180, 80)
        ToggleBtn.Text = "ON [P]"
        State.CurrentPhase = "INITIALIZING"
        State.StationPhase = "NONE"
        State.FullStopTime = 0
    else
        ToggleBtn.BackgroundColor3 = Color3.fromRGB(200, 60, 60)
        ToggleBtn.Text = "OFF [P]"
        State.CurrentPhase = "DISABLED"
        State.StationPhase = "NONE"
        State.FullStopTime = 0
        if GameplayStates.isDriving and not isGuarding() then
            pcall(holdStopped)
        end
    end
end
ToggleBtn.MouseButton1Click:Connect(function()
    setAutopilot(not State.Enabled)
end)
local keyConn = UserInputService.InputBegan:Connect(function(input, gpe)
    if not gpe and input.KeyCode == CONFIG.ToggleKey then
        setAutopilot(not State.Enabled)
    end
end)
local METERS_PER_MILE = 1609.344
local MPS_PER_MPH = 0.44704

local function isFiniteNumber(value)
    return type(value) == "number" and value == value and math.abs(value) < math.huge
end

local function positiveNumber(value, fallback)
    return isFiniteNumber(value) and value > 0 and value or fallback
end

local function hasUsableDistance(distance)
    return isFiniteNumber(distance) and distance < 900
end

local TrainPhysics = {
    Mass = 175000,
    BrakingForce = 236250,
    MaxSpeed = 100,
    ServiceDecel = CONFIG.FallbackDecel, -- m/s^2
    MaxAccel = 1.0,      -- m/s^2
    Class = "Unknown",
    HasProfile = false,
    LastCheck = 0,
    Train = nil,
    PhysicsObject = nil,
}

local function updateTrainPhysics(td)
    if not td then return end
    local now = os.clock()
    local tp = td.trainPhysics
    if TrainPhysics.Train ~= td or TrainPhysics.PhysicsObject ~= tp then
        resetBrakeFeedback(false)
        TrainPhysics.Train = td
        TrainPhysics.PhysicsObject = tp
        TrainPhysics.Mass = 175000
        TrainPhysics.BrakingForce = 236250
        TrainPhysics.ServiceDecel = CONFIG.FallbackDecel
        TrainPhysics.MaxAccel = 1.0
        TrainPhysics.MaxSpeed = 100
        TrainPhysics.Class = "Unknown"
        TrainPhysics.HasProfile = false
        TrainPhysics.LastCheck = 0
    end
    if now - TrainPhysics.LastCheck < 1.0 and TrainPhysics.HasProfile then
        return
    end
    TrainPhysics.LastCheck = now

    if tp and tp.profile then
        local prof = tp.profile
        TrainPhysics.Mass = positiveNumber(prof.mass, TrainPhysics.Mass)
        TrainPhysics.BrakingForce = positiveNumber(prof.braking, TrainPhysics.BrakingForce)
        TrainPhysics.MaxSpeed = prof.maxSpeed or td.maxSpeed or TrainPhysics.MaxSpeed
        TrainPhysics.Class = td.class or "Train"
        TrainPhysics.HasProfile = true

        -- Physics formula from game's physicsHandler:
        -- friction = 0.001 * mass * 9.81
        -- serviceBrakeForce = braking * 0.8 * 1.25 * 0.75 (worst speed fade)
        -- serviceDecel = (friction + serviceBrakeForce) / mass
        local friction = 0.001 * TrainPhysics.Mass * 9.81
        local serviceBrake = TrainPhysics.BrakingForce * CONFIG.ServiceBrake * 1.25 * 0.75
        TrainPhysics.ServiceDecel = (friction + serviceBrake) / TrainPhysics.Mass

        local pow = prof.power and prof.power[1]
        local pSrc = pow and (pow.OHLE or pow.ThirdRail or pow.Diesel)
        local powerWatts = pSrc and pSrc.PowerOutput or 1000000
        TrainPhysics.MaxAccel = math.clamp(powerWatts * 0.1 / TrainPhysics.Mass, 0.1, 3.0)
    elseif td.maxSpeed then
        TrainPhysics.MaxSpeed = td.maxSpeed
        TrainPhysics.Class = td.class or "Train"
    end

    State.TrainClass = TrainPhysics.Class
    State.TrainMass = TrainPhysics.Mass
    State.TrainBrakingForce = TrainPhysics.BrakingForce
    State.DecelRate = TrainPhysics.ServiceDecel
    State.AccelRate = TrainPhysics.MaxAccel
end

local function getUsableBrakeDecel()
    local nominal = CONFIG.DynamicPhysicsBraking and TrainPhysics.ServiceDecel or CONFIG.FallbackDecel
    nominal = positiveNumber(nominal, CONFIG.FallbackDecel)
    local learned = CONFIG.AdaptiveBraking and BrakeFeedback.Decel
    return math.max(0.1, math.min(nominal,
        learned or nominal * CONFIG.InitialBrakeEfficiency))
end

local function observeBrakeResponse(speedMph, now)
    local f = BrakeFeedback
    local throttle, brake = getAppliedEffort()
    local dt = f.LastTime and now - f.LastTime or 0
    if not State.Enabled or not isFiniteNumber(speedMph) or dt <= 0 or dt > 0.8
        or (f.LastSpeed and math.abs(speedMph - f.LastSpeed) > math.max(10, 20 * dt)) then
        resetBrakeFeedback(true)
        f.LastTime, f.LastSpeed = now, speedMph
        f.WindowTime, f.WindowSpeed = now, speedMph
        return
    end
    local acceleration = (speedMph - f.LastSpeed) * MPS_PER_MPH / dt
    State.MeasuredAcceleration = State.MeasuredAcceleration * 0.8 + acceleration * 0.2
    f.LastTime, f.LastSpeed = now, speedMph
    local strong = brake >= CONFIG.ServiceBrake - 0.01 and brake <= 0.81
        and throttle < 0.01 and speedMph >= 8
    if strong then f.StrongSince = f.StrongSince or now else f.StrongSince = nil end
    f.MinBrake = math.min(f.MinBrake or brake, brake)
    f.MaxThrottle = math.max(f.MaxThrottle or 0, throttle)
    local elapsed = now - f.WindowTime
    if elapsed < CONFIG.BrakeSampleWindow then return end
    local measured = (f.WindowSpeed - speedMph) * MPS_PER_MPH / elapsed
    State.MeasuredDecel = math.max(0, measured)
    local nominal = positiveNumber(CONFIG.DynamicPhysicsBraking and TrainPhysics.ServiceDecel,
        CONFIG.FallbackDecel)
    if CONFIG.AdaptiveBraking and f.StrongSince
        and f.WindowTime - f.StrongSince >= CONFIG.BrakeSettleTime
        and f.MinBrake >= CONFIG.ServiceBrake - 0.01 and f.MaxThrottle < 0.01
        and measured > 0.08 and measured < nominal * 2 then
        local sample = math.clamp(measured * 0.95, 0.1, nominal)
        local old = f.Decel or nominal * CONFIG.InitialBrakeEfficiency
        f.Decel = old + (sample - old) * (sample < old and 0.45 or 0.05)
    end
    f.WindowTime, f.WindowSpeed = now, speedMph
    f.MinBrake, f.MaxThrottle = nil, 0
end

local function getServiceStoppingMeters(speedMph)
    local v = math.max(0, speedMph) * MPS_PER_MPH
    local a = getUsableBrakeDecel()
    local meters = v * v / (2 * a)
    local tp = CONFIG.DynamicPhysicsBraking and TrainPhysics.PhysicsObject
    if tp and tp.getStoppingDistance and v > 0 then
        local ok, native = pcall(function() return tp:getStoppingDistance(speedMph, CONFIG.ServiceBrake) end)
        if ok and isFiniteNumber(native) and native > 0 then
            meters = math.max(meters, native)
        end
    end
    return meters
end

local function getDynamicBrakingDistMiles(fromMph, toMph)
    toMph = math.max(0, toMph or 0)
    fromMph = math.max(0, fromMph or 0)
    if fromMph <= toMph then return 0 end
    local meters = math.max(0, getServiceStoppingMeters(fromMph) - getServiceStoppingMeters(toMph))
    local responseTime = math.max(0, CONFIG.BrakeResponseTime) + CONFIG.BrakeResponseLag
        + math.max(CONFIG.ControlInterval, State.ControlInterval)
    return (meters * math.max(1, CONFIG.DynamicBrakingMargin)
        + fromMph * MPS_PER_MPH * responseTime) / METERS_PER_MILE
end

local function getDynamicSafeSpeed(distMiles, finalMph, maxMph)
    finalMph = math.max(0, finalMph or 0)
    maxMph = math.max(finalMph, maxMph or CONFIG.MaxCruisingSpeed)
    if distMiles <= 0 then return finalMph end
    if getDynamicBrakingDistMiles(maxMph, finalMph) <= distMiles then
        return maxMph
    end

    -- Bisection stays consistent even if native braking varies with speed.
    local low, high = finalMph, maxMph
    for _ = 1, 18 do
        local mid = (low + high) * 0.5
        if getDynamicBrakingDistMiles(mid, finalMph) <= distMiles then
            low = mid
        else
            high = mid
        end
    end
    return low
end

local function getStopTargetMph(ctx, distanceMiles, clearanceMiles)
    local remaining = distanceMiles - clearanceMiles
    if remaining <= CONFIG.StopToleranceMeters / METERS_PER_MILE then return 0, 0, 0 end
    local cap = math.max(0, math.min(ctx.speedLimit, ctx.maxSpeed, CONFIG.MaxCruisingSpeed))
    local reference = getDynamicSafeSpeed(remaining, 0, cap)
    local current = math.max(0, ctx.currentSpeed)
    local stepMeters = math.min(remaining * METERS_PER_MILE,
        math.max(0.1, current * MPS_PER_MPH * 0.1))
    local ahead = getDynamicSafeSpeed(math.max(0, remaining - stepMeters / METERS_PER_MILE), 0, cap)
    local slope = math.max(0, (reference - ahead) * MPS_PER_MPH / stepMeters)
    local plannedDecel = math.min(getUsableBrakeDecel(), slope * current * MPS_PER_MPH)
    -- This is the actual permitted speed curve. The effort controller supplies
    -- its deceleration directly; there is no artificial Arcade speed command.
    return reference, reference, plannedDecel
end

local function selectCruiseThrottle(requested, now)
    local control = TractionControl
    local steps = Inputs:getThrottleSteps()
    if steps <= 0 then
        -- Continuous controllers get a bounded slew, rather than discrete notches.
        local dt = math.clamp(now - (control.LastUpdate or now), 0, 0.25)
        control.Level = (control.Level or 0) + math.clamp(requested - (control.Level or 0),
            -CONFIG.ThrottleSlewPerSecond * dt, CONFIG.ThrottleSlewPerSecond * dt)
        control.LastUpdate = now
        return control.Level
    end
    local previous = math.clamp(control.Notch or 0, 0, steps)
    local wanted = math.clamp(requested * steps, 0, steps)
    local nextNotch = previous
    -- Schmitt thresholds around each notch prevent boundary noise from toggling it.
    if now - control.LastChange >= CONFIG.ThrottleNotchInterval then
        if wanted > previous + CONFIG.NotchHysteresis then
            nextNotch = math.min(steps, previous + 1)
        elseif wanted < previous - CONFIG.NotchHysteresis then
            nextNotch = math.max(1, previous - 1)
        end
    end
    if previous == 0 then nextNotch = 1 end
    if nextNotch ~= previous then control.LastChange = now end
    control.Notch, control.Level, control.LastUpdate = nextNotch, nextNotch / steps, now
    return control.Level
end

local function coastCruise(now)
    if not TractionControl.Coasting then TractionControl.CoastSince = now end
    TractionControl.Coasting = true
    TractionControl.Notch, TractionControl.Level = 0, 0
    TractionControl.LastChange, TractionControl.LastUpdate = now, now
    applyEffort(0, 0)
end

local function driveToSpeed(ctx, targetMph, plannedDecel)
    State.ControlTime = ctx.controlTime or os.clock()
    if targetMph <= 0 then holdStopped(); return end
    if Inputs:getThrottleLocked() then
        resetTractionControl()
        State.CurrentPhase = "WAITING FOR NATIVE BRAKE RELEASE"
        State.TargetSpeed = 0
        applyEffort(0, CONFIG.ServiceBrake)
        return
    end
    local speed = math.max(0, ctx.currentSpeed)
    local errorMps = (targetMph - speed) * MPS_PER_MPH
    local holdThrottle = 0
    local tp = TrainPhysics.PhysicsObject
    if tp and tp.getThrottleNeededForConstantSpeed then
        local ok, value = pcall(function() return tp:getThrottleNeededForConstantSpeed(speed) end)
        if ok and isFiniteNumber(value) then holdThrottle = math.clamp(value, 0, 1) end
    end
    local now = ctx.controlTime or os.clock()
    local control = TractionControl
    local acceleration = State.MeasuredAcceleration
    local predictedSpeed = speed + math.max(0, acceleration) * CONFIG.TractionLookAhead / MPS_PER_MPH
    local ceiling = math.min(ctx.speedLimit or math.huge, ctx.maxSpeed or math.huge,
        CONFIG.MaxCruisingSpeed, targetMph + CONFIG.CruiseMarginMPH)

    if (plannedDecel or 0) > 0.001 then
        -- The stopping trajectory always takes priority over cruise dwell times.
        control.Braking, control.Coasting = false, false
        local desiredAccel = CONFIG.SpeedGain * errorMps - plannedDecel
        if desiredAccel < -0.04 then
            control.Notch, control.Level = 0, 0
            applyEffort(0, CONFIG.ServiceBrake * math.clamp(-desiredAccel / getUsableBrakeDecel(), 0, 1))
        elseif speed > targetMph + CONFIG.SpeedDeadbandMPH then
            control.Notch, control.Level = 0, 0
            applyEffort(0, 0)
        elseif desiredAccel > 0.02 or math.abs(errorMps) <= CONFIG.SpeedDeadbandMPH * MPS_PER_MPH then
            local requested = math.clamp(holdThrottle + desiredAccel / positiveNumber(TrainPhysics.MaxAccel, 1), 0, 1)
            applyEffort(selectCruiseThrottle(requested, now), 0)
        else
            control.Notch, control.Level = 0, 0
            applyEffort(0, 0)
        end
        return
    end

    if speed > ceiling + CONFIG.CruiseBrakeStartMPH then control.Braking = true end
    if control.Braking then
        if speed > targetMph + CONFIG.CruiseBrakeReleaseMPH then
            control.Coasting = true
            control.CoastSince = now
            control.Notch, control.Level = 0, 0
            control.LastChange, control.LastUpdate = now, now
            applyEffort(0, CONFIG.ServiceBrake * math.clamp(
                (speed - targetMph) * MPS_PER_MPH * CONFIG.SpeedGain / getUsableBrakeDecel(), 0, 1))
            return
        end
        control.Braking = false
        coastCruise(now)
        return
    end

    local steps = Inputs:getThrottleSteps()
    local minimumThrottle = steps > 0 and 1 / steps or 0.02
    local underMinimumPower = holdThrottle < minimumThrottle * 0.5
    local coastAt = underMinimumPower and targetMph - CONFIG.CoastLeadMPH
        or targetMph + CONFIG.CruiseUpperBandMPH
    if predictedSpeed >= coastAt then
        coastCruise(now)
        return
    end
    if control.Coasting then
        local restartSpeed = targetMph - CONFIG.CruiseRestartBandMPH
        if speed > restartSpeed or predictedSpeed > restartSpeed
            or now - control.CoastSince < CONFIG.MinimumCoastSeconds then
            coastCruise(now)
            return
        end
        control.Coasting = false
        control.LastChange = now - CONFIG.ThrottleNotchInterval
    end
    local desiredAccel = CONFIG.CruiseSpeedGain * errorMps
        - CONFIG.AccelerationDamping * acceleration
    local requested = math.clamp(holdThrottle
        + desiredAccel / positiveNumber(TrainPhysics.MaxAccel, 1), minimumThrottle, 1)
    applyEffort(selectCruiseThrottle(requested, now), 0)
end


local function updateSignalState(sInfo, hasBuffer, bufferDist)
    local rawSignalId = sInfo and sInfo.signalId:get()
    local rawSignalType = sInfo and sInfo.signalType:get()
    local rawSignalState = sInfo and sInfo.signalState:get()
    local rawSignalDist = sInfo and sInfo.dist:get()

    if type(rawSignalId) ~= "string" or type(rawSignalType) ~= "string"
        or not isFiniteNumber(rawSignalState) or not isFiniteNumber(rawSignalDist) then
        State.CurrentSignalId = "None"
        State.CurrentSignalDist = 999
        State.CurrentSignalState = "UNAVAILABLE"
        return false, false, false, 999, false
    end

    local validSignal =
        rawSignalType ~= ""
        and rawSignalType ~= "None"
        and rawSignalId ~= ""
        and rawSignalId ~= "None"

    local isRed = false
    local isYellow = false

    if validSignal then
        State.CurrentSignalId = rawSignalId
        State.CurrentSignalDist = rawSignalDist

        if rawSignalType == "ShuntSignal" then
            isRed = rawSignalState == 0
        elseif rawSignalType == "TwoAspect" then
            isRed = rawSignalState == 0
        else
            if rawSignalState == 0 then
                isRed = true
            elseif rawSignalState == 1 then
                isYellow = true
            end
        end

        State.CurrentSignalState =
            isRed and "DANGER (RED)"
            or (isYellow and "CAUTION (YELLOW)" or "CLEAR (GREEN)")
    elseif hasBuffer then
        State.CurrentSignalId = "BUFFER STOP"
        State.CurrentSignalDist = bufferDist
        State.CurrentSignalState = "BUFFER AHEAD"
    else
        State.CurrentSignalId = "None"
        State.CurrentSignalDist = 999
        State.CurrentSignalState = "CLEAR"
    end

    return validSignal, isRed, isYellow, rawSignalDist, true
end

local function syncDriverRoute(td)
    local model = td and td.model
    local routeJSON = model and model:GetAttribute("FullRoute")
    if type(routeJSON) ~= "string" or routeJSON == "" then return end
    local ok, route = pcall(function() return game:GetService("HttpService"):JSONDecode(routeJSON) end)
    if not ok or type(route) ~= "table" or #route == 0 then return end
    local stops = {}
    for _, stop in ipairs(route) do
        if type(stop) ~= "table" then return end
        table.insert(stops, tostring(stop.id or stop.name or "") .. (stop.turnaround and ":R" or ""))
    end
    -- Server delay/score updates rewrite route JSON during a stop; they are not a new route.
    local signature = table.concat(stops, ">")
    if State.RouteModel == model and State.RouteSignature == signature then return end
    State.RouteModel, State.RouteSignature = model, signature
    -- A new timetable on the same train must not inherit its previous terminus stop.
    State.StationPhase = "NONE"
    State.CurrentStopStation, State.CurrentStopIsTerminus = "", false
    State.DepartedStation, State.OriginStation, State.TerminusStation = "", "", ""
    State.FullStopTime, State.BoardingStartTime = 0, 0
    State.DoorAttemptTime, State.DoorStandstillSince, State.DoorClosedSince = nil, nil, nil
    State.DoorsOpen = false
    resetBrakeFeedback(true)
end

local function readRuntimeSnapshot()
    local now = os.clock()
    if State.LastControlTime then
        State.ControlInterval = math.max(CONFIG.ControlInterval,
            now - State.LastControlTime, State.ControlInterval * 0.9)
    end
    State.LastControlTime = now
    local currentTrainData, sInfo, ActivityQueue = readDriverUpvalues()
    syncDriverRoute(currentTrainData)
    updateTrainPhysics(currentTrainData)
    local currentSpeed = currentTrainData and currentTrainData.speed or 0
    local speedLimit = getLiveTrackLimit(currentTrainData)
    local maxSpeed = positiveNumber(currentTrainData and currentTrainData.maxSpeed, 100)

    observeBrakeResponse(currentSpeed, now)
    State.CurrentSpeed = math.round(currentSpeed)
    State.ExactSpeed = currentSpeed
    State.SpeedLimit = speedLimit
    State.DynBrakeDist = getDynamicBrakingDistMiles(currentSpeed, 0)

    local doorsOpenNative, atPlatformNative, _, doorsStateKnown = getDoorsStateNative()

    local stationValue = ActivityQueue and readUpvalues(ActivityQueue.routeRenderStepUpdate)[9]
    local stationText = hasGetter(stationValue) and stationValue:get()
    if type(stationText) ~= "string" then stationText = "None" end
    local stationTextAtPlatform =
        stationText == "At Platform"
        or stationText == "Arrived"
    local atPlatform =
        stationTextAtPlatform
        or atPlatformNative

    local stationDistance = 999
    if stationTextAtPlatform then
        stationDistance = 0
    else
        local miles =
            stationText:match("([%d%.]+)%s*miles")
            or stationText:match("([%d%.]+)%s*mi")
        if miles then
            stationDistance = tonumber(miles) or 999
        end
    end

    local hasMarker = sInfo and hasGetter(sInfo.renderMarker) and sInfo.renderMarker:get() or false
    local markerDistance = sInfo and hasGetter(sInfo.markerDist) and sInfo.markerDist:get() or 999
    local hasBuffer = sInfo and hasGetter(sInfo.renderBuffer) and sInfo.renderBuffer:get() or false
    local bufferDistance = sInfo and hasGetter(sInfo.bufferDist) and sInfo.bufferDist:get() or 999

    local validSignal, isRed, isYellow, signalDistance, signalDataReady =
        updateSignalState(sInfo, hasBuffer, bufferDistance)

    local nextStop = ActivityQueue and ActivityQueue:getNextStop()
    local nextDeparture = ActivityQueue and ActivityQueue:getNextDeparture()
    local nextActivity = ActivityQueue and ActivityQueue:getNextActivity()

    local currentOrNextStation =
        (nextActivity and nextActivity.location)
        or (nextStop and nextStop.location)
        or "En Route"

    State.NextStop = currentOrNextStation
    State.NextStopDist = stationDistance

    local currentEpochTime = utils:time()
    local departureEpochTime =
        (nextDeparture and nextDeparture.epochT)
        or (nextActivity and nextActivity.epochT)
        or (currentEpochTime + 20)

    State.ScheduledDepTime = departureEpochTime

    local isStopActivity =
        nextActivity
        and (
            nextActivity.action == "Stop"
            or nextActivity.action == "Depart"
            or nextActivity.action == "Terminate"
            or nextActivity.action == "Turnaround"
        )

    local isTerminusActivity =
        nextActivity
        and (
            nextActivity.action == "Terminate"
            or nextActivity.action == "Turnaround"
        )
        or false

    if isTerminusActivity and nextActivity and nextActivity.location then
        State.TerminusStation = nextActivity.location
    end

    if
        State.OriginStation == ""
        and currentOrNextStation ~= "En Route"
        and currentOrNextStation ~= "None"
        and not isTerminusActivity
    then
        State.OriginStation = currentOrNextStation
    end

    -- Detect if ActivityQueue already advanced to the NEXT station down the line
    -- while we are still handling the current station stop (e.g. penultimate station).
    local activityAdvancedToNextStation =
        State.StationPhase ~= "NONE"
        and State.StationPhase ~= "DEPARTED"
        and State.CurrentStopStation ~= ""
        and nextActivity ~= nil
        and nextActivity.location ~= nil
        and nextActivity.location ~= State.CurrentStopStation

    local isAtActivityStation =
        not activityAdvancedToNextStation
        and (
            stationTextAtPlatform
            or stationDistance <= 0.05
        )

    -- Native Depart at the same location supersedes the old arrival/terminus.
    if nextActivity and nextActivity.action == "Depart"
        and nextActivity.location == State.CurrentStopStation then
        State.CurrentStopIsTerminus = false
    end

    local isTerminus =
        State.CurrentStopIsTerminus
        or (isTerminusActivity and isAtActivityStation)

    if State.StationPhase == "DEPARTED" then
        local clearDistance =
            not atPlatform
            and not atPlatformNative
            and stationDistance > 0.08

        local newStation =
            nextActivity
            and nextActivity.location ~= State.DepartedStation
            and not atPlatform
            and not atPlatformNative

        if clearDistance or newStation then
            State.StationPhase = "NONE"
            State.FullStopTime = 0
            State.DoorAttemptTime = nil
            State.DoorStandstillSince = nil
            State.DoorClosedSince = nil
            State.DoorsOpen = false
            State.CurrentStopStation = ""
            State.CurrentStopIsTerminus = false
        end
    end

    return {
        currentSpeed = currentSpeed,
        driverGuard = readDriverGuardContext(currentTrainData),
        controlTime = now,
        signalDataReady = signalDataReady,
        trainAvailable = currentTrainData and isFiniteNumber(currentTrainData.speed)
            and isFiniteNumber(currentTrainData.maxSpeed) and currentTrainData.maxSpeed > 0,
        speedLimit = speedLimit,
        maxSpeed = maxSpeed,

        doorsOpenNative = doorsOpenNative,
        doorsStateKnown = doorsStateKnown,
        atPlatformNative = atPlatformNative,
        stationTextAtPlatform = stationTextAtPlatform,
        atPlatform = atPlatform,
        stationDistance = stationDistance,

        hasMarker = hasMarker,
        markerDistance = markerDistance,
        hasBuffer = hasBuffer,
        bufferDistance = bufferDistance,

        validSignal = validSignal,
        isRed = isRed,
        isYellow = isYellow,
        signalDistance = signalDistance,

        nextActivity = nextActivity,
        currentOrNextStation = currentOrNextStation,
        currentEpochTime = currentEpochTime,
        departureEpochTime = departureEpochTime,
        isStopActivity = isStopActivity,
        isTerminusActivity = isTerminusActivity,
        isAtActivityStation = isAtActivityStation,
        isTerminus = isTerminus,
    }
end

local function tryAutoSelectNextRoute()
    if not CONFIG.AutoNextRoute then
        return
    end

    local ui = PlayerGui:FindFirstChild("UI")
    local activePopup = getActiveEndOfRoutePopup(ui)
    local activePicker = getActiveRoutePicker(ui)

    if not (activePopup or activePicker) then
        return
    end

    if os.clock() - State.LastRouteCheckTime < 0.5 then
        return
    end

    State.LastRouteCheckTime = os.clock()
    State.CurrentPhase = "AUTO-SELECTING NEXT ROUTE"
    autoSelectNextRoute(activePopup, activePicker)
end

local function isAtStationStopPoint(ctx)
    local tolerance = CONFIG.StopToleranceMeters / METERS_PER_MILE
    if ctx.hasBuffer and hasUsableDistance(ctx.bufferDistance)
        and ctx.bufferDistance <= CONFIG.BufferStopDist + tolerance then
        return true
    end
    if ctx.hasMarker and hasUsableDistance(ctx.markerDistance)
        and ctx.markerDistance <= CONFIG.MarkerStopDist + tolerance then
        return true
    end
    return not ctx.hasMarker
        and not ctx.hasBuffer
        and ctx.atPlatformNative
        and (ctx.stationTextAtPlatform or ctx.stationDistance <= 0.15)
end

local function handleDriverGuardStation(ctx)
    local info = ctx.driverGuard
    local control = DriverGuardControl
    if not info then
        control.Active, control.SeenProcedure = false, false
        if State.StationPhase == "GUARD_PROCEDURE" then
            State.StationPhase = ctx.doorsOpenNative and "BOARDING" or "STOPPING"
            State.DoorAttemptTime, State.DoorClosedSince = nil, nil
        end
        return false
    end
    if control.Train ~= info.train or control.Guard ~= info.guard then
        control.Train, control.Guard = info.train, info.guard
        control.LastAction, control.Active, control.SeenProcedure = -math.huge, false, false
    end
    if math.abs(ctx.currentSpeed) >= 0.001 then return false end
    local stage = info.stage or {}
    local action = stage.Action
    if State.StationPhase == "DEPARTED" and not action and info.doors == "Locked" then return false end
    local stationStop = action ~= nil or info.doors ~= "Locked"
        or (State.StationPhase ~= "NONE" and State.StationPhase ~= "DEPARTED")
        or (ctx.isStopActivity and (ctx.atPlatform or ctx.atPlatformNative))
    if not stationStop then return false end
    if not control.Active then
        control.Active, control.SeenProcedure = true, false
        State.DoorClosedSince = nil
    end
    if action or info.doors ~= "Locked" then control.SeenProcedure = true end
    State.StationPhase = "GUARD_PROCEDURE"
    holdStopped()
    if State.CurrentStopStation == "" and ctx.isAtActivityStation
        and ctx.currentOrNextStation ~= "En Route" and ctx.currentOrNextStation ~= "None" then
        State.CurrentStopStation = ctx.currentOrNextStation
        State.CurrentStopIsTerminus = ctx.isTerminusActivity
    end
    if info.doors == "Open" or info.doors == "Ready" then State.DoorsOpen = true end
    if action then
        State.DoorClosedSince = nil
        if action == "Wait" then
            State.CurrentPhase = "WAITING FOR GUARD / " .. tostring(stage.For or "PROCEDURE")
            return true -- Only native guard/server messages complete a wait.
        end
        local callback = action == "SignalBell" and "Signal Bell"
            or ((action == "UnlockDoors" or action == "LockDoors") and "Toggle Doors")
        State.CurrentPhase = action == "SignalBell" and "ACKNOWLEDGING GUARD BELL"
            or "DRIVER / " .. tostring(action)
        if not callback or not CONFIG.AutoDriverGuard then return true end
        local now = ctx.controlTime or os.clock()
        if now - control.LastAction < CONFIG.DriverGuardActionInterval then return true end
        control.LastAction = now
        -- Named callbacks respect Drive mode and run the exact same code as T,
        -- including buzzer animation, stage completion and the native bell.
        local current = DriverGuardHandler:getCurrentStage() or {}
        if current.Action == action then InputHandler:runCallback(callback) end
        return true
    end
    if info.doors ~= "Locked" or Inputs:getThrottleLocked() or not control.SeenProcedure then
        State.DoorClosedSince = nil
        State.CurrentPhase = (info.doors == "Open" or info.doors == "Ready")
            and "BOARDING / GUARD PROCEDURE" or "WAITING FOR GUARD RELEASE"
        return true
    end
    local now = ctx.controlTime or os.clock()
    State.DoorClosedSince = State.DoorClosedSince or now
    State.CurrentPhase = "GUARD RELEASED / DEPARTURE CHECK"
    if now - State.DoorClosedSince < CONFIG.DoorCloseWait then return true end
    State.StationPhase = "DEPARTED"
    State.CurrentPhase = "DEPARTING"
    State.DepartedStation = State.CurrentStopStation ~= "" and State.CurrentStopStation or ctx.currentOrNextStation
    State.CurrentStopStation, State.CurrentStopIsTerminus = "", false
    State.DoorsOpen, State.FullStopTime = false, 0
    control.Active, control.SeenProcedure = false, false
    return false -- Normal red-signal/buffer constraints still apply to departure.
end


local function retryStationDoors(ctx)
    local stoppedAtPlatform = math.abs(ctx.currentSpeed) < 0.001
        and (ctx.atPlatformNative or ctx.atPlatform)
    if not State.Enabled or not stoppedAtPlatform or State.StationPhase == "DEPARTED" then
        State.DoorStandstillSince = nil
        return
    end
    local now = os.clock()
    State.DoorStandstillSince = State.DoorStandstillSince or now
    if now - State.DoorStandstillSince < CONFIG.DoorOpenDelay then return end

    -- At the actual terminus, once doors are confirmed open, keep them open for route completion.
    if ctx.isTerminus and State.StationPhase == "BOARDING" and ctx.doorsStateKnown and ctx.doorsOpenNative then
        return
    end

    -- A completed close must not be undone during the locking/departure wait.
    if State.DoorsOpen and ctx.doorsStateKnown and not ctx.doorsOpenNative and isDoorsLocked() then
        return
    end
    if State.DoorAttemptTime and now - State.DoorAttemptTime < CONFIG.DoorRetryInterval then return end
    State.DoorAttemptTime = now -- Shared across OPENING, BOARDING and CLOSING.
    toggleDoors() -- Let the game's timetable reject an early close.
end

local function handleStation(ctx)
    local atStopPoint = isAtStationStopPoint(ctx)
    local stoppedAtPlatform = math.abs(ctx.currentSpeed) < 0.001
        and (ctx.atPlatformNative or ctx.atPlatform)
    local isCurrentStationTarget =
        ctx.stationTextAtPlatform
        or ctx.stationDistance <= 0.15
        or ctx.hasMarker
        or ctx.hasBuffer
        or ctx.doorsOpenNative
    local phase = State.StationPhase
    local isHandlingStation = phase ~= "NONE"
    local shouldStartStop = phase == "NONE" and ctx.isStopActivity
        and isCurrentStationTarget
        and (atStopPoint or stoppedAtPlatform)
    if not (isHandlingStation or shouldStartStop) then
        State.DoorStandstillSince = nil
        return false
    end

    -- Native departure completion can happen between our snapshots (or while reloading).
    -- Do not wait for an open-door state that the game has already advanced past.
    local departedNatively = State.CurrentStopStation ~= "" and ctx.nextActivity
        and ctx.nextActivity.location ~= State.CurrentStopStation
        and ctx.doorsStateKnown and not ctx.doorsOpenNative
        and not ctx.atPlatformNative and isDoorsLocked()
    if departedNatively then
        State.StationPhase = "DEPARTED"
        State.CurrentPhase = "DEPARTING"
        State.DepartedStation = State.CurrentStopStation
        State.CurrentStopStation, State.CurrentStopIsTerminus = "", false
        State.DoorsOpen, State.FullStopTime = false, 0
        State.DoorAttemptTime, State.DoorStandstillSince, State.DoorClosedSince = nil, nil, nil
        return false
    end

    retryStationDoors(ctx)

    if phase == "NONE" then
        State.StationPhase = "STOPPING"
        State.CurrentPhase = "BRAKING TO 0 MPH"
        State.FullStopTime = 0
        if ctx.isAtActivityStation and ctx.currentOrNextStation ~= "En Route" and ctx.currentOrNextStation ~= "None" then
            State.CurrentStopStation = ctx.currentOrNextStation
            State.CurrentStopIsTerminus = ctx.isTerminusActivity
        else
            State.CurrentStopStation = ""
            State.CurrentStopIsTerminus = false
        end
        holdStopped()

    elseif phase == "STOPPING" then
        holdStopped()
        if State.CurrentStopStation == "" and ctx.isAtActivityStation and ctx.currentOrNextStation ~= "En Route" and ctx.currentOrNextStation ~= "None" then
            State.CurrentStopStation = ctx.currentOrNextStation
            State.CurrentStopIsTerminus = ctx.isTerminusActivity
        end
        if stoppedAtPlatform then
            if State.FullStopTime == 0 then State.FullStopTime = os.clock() end
            local elapsed = os.clock() - State.FullStopTime
            if elapsed < CONFIG.DoorOpenDelay then
                State.CurrentPhase = string.format("STANDSTILL (PAUSE %.1fs)", CONFIG.DoorOpenDelay - elapsed)
            else
                State.StationPhase = "OPENING_DOORS"
            end
        else
            State.CurrentPhase = "BRAKING TO 0 MPH"
            State.FullStopTime = 0
        end

    elseif phase == "OPENING_DOORS" then
        holdStopped()
        State.CurrentPhase = "OPENING DOORS (10s RETRY)"
        if ctx.doorsStateKnown and ctx.doorsOpenNative then
            State.StationPhase = "BOARDING"
            State.BoardingStartTime = os.clock()
            State.DoorsOpen = true
        end

    elseif phase == "BOARDING" then
        holdStopped()
        if ctx.doorsStateKnown and not ctx.doorsOpenNative then
            -- The game accepted the close. Do not reopen on the next retry
            -- because a local timetable or minimum-boarding timer disagrees.
            State.StationPhase = "CLOSING_DOORS"
            State.DoorClosedSince = nil
            State.CurrentPhase = "CLOSING DOORS"
        elseif ctx.isTerminus then
            State.CurrentPhase = CONFIG.AutoNextRoute and "TERMINUS (WAITING FOR NEXT ROUTE)" or "TERMINUS REACHED"
        else
            State.CurrentPhase = "BOARDING (10s DOOR RETRY)"
        end

    elseif phase == "CLOSING_DOORS" then
        holdStopped()
        State.CurrentPhase = "CLOSING DOORS (10s RETRY)"
        if ctx.doorsStateKnown and not ctx.doorsOpenNative and isDoorsLocked() then
            State.DoorClosedSince = State.DoorClosedSince or os.clock()
            if os.clock() - State.DoorClosedSince >= CONFIG.DoorCloseWait then
                State.StationPhase = "DEPARTED"
                State.CurrentPhase = "DEPARTING"
                State.DepartedStation = State.CurrentStopStation ~= "" and State.CurrentStopStation or ctx.currentOrNextStation
                State.CurrentStopStation = ""
                State.CurrentStopIsTerminus = false
                State.DoorsOpen = false
                State.FullStopTime = 0
            end
        else
            State.DoorClosedSince = nil
        end

    elseif phase == "DEPARTED" then
        return false -- Departure still obeys red signals and buffers.
    end
    return true
end

local function getBufferTarget(ctx)
    if ctx.hasBuffer and hasUsableDistance(ctx.bufferDistance) then
        local command, reference, decel = getStopTargetMph(ctx, ctx.bufferDistance, CONFIG.BufferStopDist)
        return command, "BUFFER BRAKING", reference, decel
    end
end

local function getStationTarget(ctx)
    if State.StationPhase ~= "NONE" or not ctx.isStopActivity then return nil end

    if ctx.hasMarker and hasUsableDistance(ctx.markerDistance) then
        -- A platform flag is NOT a reason to force 10 MPH: use the marker.
        local command, reference, decel = getStopTargetMph(ctx, ctx.markerDistance, CONFIG.MarkerStopDist)
        return command, "STATION BRAKING", reference, decel
    end
    if ctx.hasBuffer and hasUsableDistance(ctx.bufferDistance) then
        return nil -- The exact buffer constraint is already considered.
    end

    -- A rounded distance to the station is not the distance to its car-stop
    -- board. Approach cautiously until the exact board appears; do not pretend
    -- to guarantee a precise stop from this coarse/missing measurement.
    if ctx.atPlatform and (ctx.stationTextAtPlatform or ctx.stationDistance <= 0.15) then
        return CONFIG.StationCrawlSpeed, "LOCATING STOP MARKER"
    end
    if hasUsableDistance(ctx.stationDistance) then
        return getDynamicSafeSpeed(ctx.stationDistance, CONFIG.MarkerSearchSpeed,
            math.max(CONFIG.MarkerSearchSpeed, math.min(ctx.speedLimit, ctx.maxSpeed,
                CONFIG.MaxCruisingSpeed))), "STATION APPROACH"
    end
end

local function getRedSignalTarget(ctx)
    -- Only brake for a red signal before the station; equality does not qualify.
    if ctx.validSignal and ctx.isRed and hasUsableDistance(ctx.signalDistance)
        and hasUsableDistance(ctx.stationDistance)
        and ctx.stationDistance > ctx.signalDistance then
        local command, reference, decel = getStopTargetMph(ctx, ctx.signalDistance, CONFIG.SignalStopDist)
        return command, "RED SIGNAL BRAKING", reference, decel
    end
end

local function getYellowSignalTarget(ctx)
    if ctx.validSignal and ctx.isYellow and ctx.signalDistance <= 0.25 then
        return CONFIG.CautionSpeed, "CAUTION (YELLOW)"
    end
end

local function runAutopilot(ctx)
    if not State.Enabled then
        return
    end

    if
        CONFIG.AutoDRA
        and Inputs.states
        and Inputs.states.DRA
        and Inputs.states.DRA:get()
    then
        pcall(function()
            Inputs:setDRA()
        end)
    end

    if not ctx.trainAvailable then
        holdStopped()
        State.CurrentPhase = "WAITING FOR TRAIN"
        return
    end
    if Inputs:getThrottleType() == "SpeedSet" then
        holdStopped()
        State.CurrentPhase = "SELECT SIMPLIFIED CONTROLS"
        return
    end
    if not ctx.signalDataReady then
        holdStopped()
        State.CurrentPhase = "WAITING FOR SIGNAL DATA"
        return
    end
    if handleDriverGuardStation(ctx) then return end
    if ctx.driverGuard and State.StationPhase == "DEPARTED" then
        -- Continue with the normal movement constraints below.
    elseif not ctx.driverGuard and handleStation(ctx) then
        return
    end
    local finalTarget = math.max(0, math.min(ctx.speedLimit, ctx.maxSpeed, CONFIG.MaxCruisingSpeed) - CONFIG.CruiseMarginMPH)
    local finalDecel = 0
    State.CurrentPhase = "CRUISING"

    State.BrakeReferenceSpeed = finalTarget
    local function consider(targetMph, phase, reference, decel)
        if targetMph and targetMph <= finalTarget then
            if targetMph < finalTarget or targetMph == 0 then
                State.CurrentPhase = phase
            end
            finalTarget = math.max(0, targetMph)
            State.BrakeReferenceSpeed = reference or finalTarget
            finalDecel = decel or 0
        end
    end

    -- Apply every constraint. A distant station/buffer must never mask a
    -- nearer stop point or red signal; issue just one movement command.
    consider(getBufferTarget(ctx))
    consider(getStationTarget(ctx))
    consider(getRedSignalTarget(ctx))
    consider(getYellowSignalTarget(ctx))
    State.TargetSpeed = finalTarget
    driveToSpeed(ctx, finalTarget, finalDecel)
end

local function updateHud(ctx)
    local phaseColor = State.Enabled and (
        State.CurrentPhase:find("RED") and "rgb(255, 100, 100)"
        or State.CurrentPhase:find("BRAKING") and "rgb(255, 180, 80)"
        or State.CurrentPhase:find("STANDSTILL") and "rgb(255, 220, 80)"
        or State.CurrentPhase:find("OPENING") and "rgb(255, 200, 100)"
        or State.CurrentPhase:find("BOARDING") and "rgb(100, 200, 255)"
        or State.CurrentPhase:find("CLOSING") and "rgb(200, 150, 255)"
        or State.CurrentPhase:find("CAUTION") and "rgb(255, 210, 100)"
        or "rgb(100, 255, 150)"
    ) or "rgb(160, 160, 170)"

    StatusLabel.Text = string.format(
        "Phase: <font color=\"%s\"><b>%s</b></font>",
        phaseColor,
        State.CurrentPhase
    )
    StatusLabel.RichText = true

    SpeedRow.Text = string.format(
        "Speed: %d | Aim: %.1f | Limit: %d MPH",
        State.CurrentSpeed,
        State.TargetSpeed,
        State.SpeedLimit
    )

    local distanceDisplay = ctx.stationTextAtPlatform and "At Platform" or (
        ctx.hasMarker and string.format("%.02f mi (Marker)", ctx.markerDistance)
        or (
            ctx.stationDistance < 900
            and string.format("%.02f mi", ctx.stationDistance)
            or "En Route"
        )
    )
    StationRow.Text = string.format("Next: %s  (%s)", State.NextStop, distanceDisplay)

    local signalColor =
        (ctx.validSignal and ctx.isRed) and "rgb(255, 90, 90)"
        or (
            (ctx.validSignal and ctx.isYellow) and "rgb(255, 210, 80)"
            or (
                ctx.hasBuffer and "rgb(255, 150, 100)"
                or "rgb(90, 255, 120)"
            )
        )

    local signalDistanceText =
        State.CurrentSignalDist < 900
        and string.format("%.02f mi", State.CurrentSignalDist)
        or "None"

    SignalRow.Text = string.format(
        "Signal: %s  [<font color=\"%s\">%s</font>]  (%s)",
        State.CurrentSignalId,
        signalColor,
        State.CurrentSignalState,
        signalDistanceText
    )
    SignalRow.RichText = true

    local departureDelta = ctx.departureEpochTime - ctx.currentEpochTime
    local timeText =
        departureDelta > 0
        and string.format("+%ds", departureDelta)
        or "Due / Ready"

    TimeRow.Text = string.format(
        "Schedule Dep: %s  (Status: %s)",
        os.date("%H:%M:%S", ctx.departureEpochTime),
        timeText
    )

    if PhysicsRow then
        if CONFIG.DynamicPhysicsBraking then
            PhysicsRow.Text = string.format(
                "Power: %d%% | Brake: %d%% | Stop: %.1fm",
                math.round(State.ThrottleLevel * 100),
                math.round(State.BrakeLevel * 100),
                (State.DynBrakeDist or 0) * METERS_PER_MILE
            )
        else
            PhysicsRow.Text = string.format("Power: %d%% | Brake: %d%%",
                math.round(State.ThrottleLevel * 100), math.round(State.BrakeLevel * 100))
        end
        PhysicsRow.RichText = true
    end
end

-- Guard actions use the game's procedure, including driver/server waits.
local GuardAuto: {[string]: any} = {
    LastAction = 0, LastTicket = 0, NextInit = 0,
    Checked = setmetatable({}, {__mode = "k"}),
    Accepted = 0, Rejected = 0, Train = nil,
}

local function initGuardAuto(now)
    if GuardAuto.Handler then return true end
    if now < GuardAuto.NextInit then return false end
    GuardAuto.NextInit = now + 5
    local ok, modules = pcall(function()
        require(Modules.Gameplay.Guarding)
        local ticketing = Modules.FusionUIs_BR.Guard.Ticketing
        return {
            Handler = require(Modules.Gameplay.Guarding.guardHandler),
            Buttons = require(Modules.Gameplay.Guarding.buttonFunctions),
            Panels = require(Modules.FusionUIs_BR.Guard.Panels),
            PassengerUI = require(Modules.Gameplay.TrainFunctions.processPassengerUIs),
            Navigation = require(Modules.Gameplay.Explore.passengerNavigation),
            TrainUpdates = require(Modules.Gameplay.TrainFunctions.trainUpdates),
            TicketStates = require(ticketing.TicketStates),
            Ticketing = require(ticketing),
            GetTicketData = require(ticketing.GetTicketData),
            Overheads = require(ticketing.OverheadUIs),
            Packets = require(game:GetService("ReplicatedStorage").Data.packets),
        }
    end)
    if not ok then
        State.CurrentPhase = "GUARD / MODULES UNAVAILABLE"
        return false
    end
    for key, value in pairs(modules) do GuardAuto[key] = value end
    return true
end

local function guardPanelDistance(attachment, character)
    return ((attachment.WorldPosition - character:GetPivot().Position) * Vector3.new(1, 0, 1)).Magnitude
end

local function findGuardPanel(train, now)
    local character = LocalPlayer.Character
    if not character or not character.PrimaryPart then return nil end
    local selectedDoor = GuardAuto.PassengerUI.currentLocalDoor
    if selectedDoor and not selectedDoor:IsDescendantOf(train) then return nil end
    local savedPanel, savedDoor = GuardAuto.PanelAttachment, GuardAuto.PanelDoor
    if savedPanel and savedPanel:IsDescendantOf(train) and savedDoor and savedDoor:IsDescendantOf(train)
        and (not selectedDoor or savedDoor == selectedDoor)
        and (selectedDoor == savedDoor or savedDoor:GetAttribute("CanGuardOpen")) then
        return savedDoor, savedPanel
    end
    if now < (GuardAuto.NextPanelSearch or 0) then return nil end
    GuardAuto.NextPanelSearch = now + CONFIG.GuardPanelSearchInterval
    local bestDoor, bestPanel, bestScore = nil, nil, math.huge
    for _, item in ipairs(train:GetDescendants()) do
        -- Default panels are generated from LeaveTrain; their native callback
        -- uses that original attachment, not the generated visual marker.
        if item:IsA("Attachment") and (item.Name == "GuardPanel" or item.Name == "LeaveTrain") then
            local path = item.Parent
            local car = path and path.Parent
            if path and path:IsA("BasePart") and car then
                local link = item:FindFirstChild("DoorValue")
                local door = link and link:IsA("ObjectValue") and link.Value
                    or GuardAuto.PassengerUI.findDoor(item, car)
                if door and door:IsDescendantOf(train)
                    and (not selectedDoor or door == selectedDoor)
                    and (door == selectedDoor or door:GetAttribute("CanGuardOpen")) then
                    local distance = guardPanelDistance(item, character)
                    local inRange = CONFIG.GuardAutoPanel and CONFIG.GuardAutoTeleport
                        or distance < CONFIG.GuardPanelRange
                    local score = distance + (item.Name == "LeaveTrain" and 0.2 or 0)
                    if inRange and score < bestScore then
                        bestDoor, bestPanel, bestScore = door, item, score
                    end
                end
            end
        end
    end
    GuardAuto.PanelDoor, GuardAuto.PanelAttachment = bestDoor, bestPanel
    return bestDoor, bestPanel
end

local function teleportGuardToPanel(train, door, attachment, now)
    if not CONFIG.GuardAutoPanel or not CONFIG.GuardAutoTeleport or not State.Enabled
        or not isGuarding() or GameplayStates.guardedTrain ~= train
        or GameplayStates.currentTrain ~= train then return false end
    local character = LocalPlayer.Character
    local path = attachment.Parent
    if not character or not character.PrimaryPart or not path or not path:IsA("BasePart")
        or not path:IsDescendantOf(train) or not door:IsDescendantOf(train) then return false end
    if now - (GuardAuto.LastPanelTeleport or -math.huge) < CONFIG.GuardTeleportInterval then return false end
    GuardAuto.LastPanelTeleport = now
    if character:GetAttribute("Seated") then
        -- Use the game's normal unseat request; the next attempt moves only
        -- after the seat state actually clears.
        local events = game:GetService("ReplicatedStorage"):FindFirstChild("Events")
        local seatEvent = events and events:FindFirstChild("SeatPlayer")
        if seatEvent and seatEvent:IsA("RemoteEvent") then seatEvent:FireServer() end
        return false
    end
    local navigation = GuardAuto.Navigation
    local z = attachment.Position.Z
    local x, floor = navigation:calculateXYFromZ(path, z)
    if not path:GetAttribute("SolveY") then floor = 0 end
    local relative = CFrame.new(x or 0, (character:GetAttribute("HeightOffset") or 2) + (floor or 0), z)
    -- Stay inside the corridor and update the native carriage-relative state;
    -- PivotTo alone would be undone by the passenger movement render loop.
    navigation:setCurrentPart(path)
    navigation:setRelativePos(relative)
    character:PivotTo(path.CFrame:ToWorldSpace(relative))
    GuardAuto.Packets.SetLocalPos.send({isWalking = false, pos = relative, part = path})
    Triggers:call("CurrentTrainUpdate")
    return true
end

local function ensureGuardPanel(train, now)
    now = now or os.clock()
    if not State.Enabled or not isGuarding() or GameplayStates.guardedTrain ~= train
        or GameplayStates.currentTrain ~= train then return false end
    local door, attachment = findGuardPanel(train, now)
    if not door then
        State.CurrentPhase = "GUARD / SEARCHING FOR AVAILABLE PANEL"
        return false
    end
    local character = LocalPlayer.Character
    if not character or not character.PrimaryPart then return false end
    if guardPanelDistance(attachment, character) >= CONFIG.GuardPanelRange
        or GuardAuto.Navigation:getCurrentPart() ~= attachment.Parent then
        State.CurrentPhase = "GUARD / MOVING TO PANEL"
        teleportGuardToPanel(train, door, attachment, now)
        return false
    end
    local panels = GuardAuto.Panels.panelStates
    local localDoor = GuardAuto.PassengerUI.currentLocalDoor
    if localDoor and localDoor ~= door then return false end
    if not panels.Opened:get() or not localDoor then
        if now - (GuardAuto.LastPanelAction or -math.huge) < CONFIG.GuardActionInterval then return false end
        GuardAuto.LastPanelAction = now
        Triggers:call("_Guarding/PanelUpdate", attachment, attachment:GetAttribute("GuardPanel") or "Default")
        if not localDoor then GuardAuto.Buttons.openPanel(door)
        else GuardAuto.Handler:stageCompleted("OpenPanel") end
        State.CurrentPhase = "GUARD / OPENING PANEL"
        return false
    end
    if not GuardAuto.TicketStates.isNearPanel:get() then return false end
    if CONFIG.GuardAutoPanel and not GuardAuto.Handler:isPanelUnlocked() then
        if now - (GuardAuto.LastPanelAction or -math.huge) >= CONFIG.GuardActionInterval then
            GuardAuto.LastPanelAction = now
            GuardAuto.Buttons.unlock()
        end
        State.CurrentPhase = "GUARD / UNLOCKING PANEL"
        return false
    end
    return true
end


local function ticketDecision(data)
    if type(data) ~= "table" or type(data.validity) ~= "table"
        or type(data.validity.Valid) ~= "boolean" then return nil end
    return not data.validity.Valid
end

local function getTicketCandidate(train)
    -- Each native overhead exists only for an unchecked passenger. Its click
    -- closure retains the exact train, seat, seed and journey data sent by the game.
    for _, ui in ipairs(PlayerGui:GetChildren()) do
        if ui:IsA("BillboardGui") and ui.Name == "TicketOverheadUI" and ui.Adornee then
            local passenger = ui.Adornee.Parent
            if passenger and passenger.Parent and not GuardAuto.Checked[passenger] then
                local button = ui:FindFirstChildOfClass("TextButton")
                if button then
                    for _, connection in ipairs(getconnections(button.Activated)) do
                        if type(connection.Function) == "function" then
                            for _, data in pairs(getupvalues(connection.Function)) do
                                if type(data) == "table" and data.trainName == train.Name
                                    and type(data.ticketSeed) == "number"
                                    and type(data.originId) == "string" and type(data.destId) == "string"
                                    and typeof(data.seat) == "Instance" and data.seat:IsDescendantOf(train) then
                                    return passenger, data
                                end
                            end
                        end
                    end
                end
            end
        end
    end
    return nil
end

local function checkGuardTicket(train, now)
    if not CONFIG.GuardAutoTickets or now - GuardAuto.LastTicket < CONFIG.GuardTicketInterval
        or GuardAuto.Ticketing.currentCheckData then return end
    GuardAuto.LastTicket = now
    local passenger, source = getTicketCandidate(train)
    if not passenger then return end
    local data = GuardAuto.GetTicketData(passenger, source.originId, source.destId, source.ticketSeed, source.seat)
    local reject = ticketDecision(data)
    local id = passenger:GetAttribute("ID")
    if reject == nil or type(id) ~= "number" then return end
    -- Recheck the role after reading the ticket before making a submission.
    if not isGuarding() or GameplayStates.guardedTrain ~= train then return end
    GuardAuto.Packets.TicketChecked.send({
        id = id, wasCorrect = data.validity.Valid == (not reject), removeFromTrain = reject,
    })
    GuardAuto.Checked[passenger] = true
    GuardAuto.Overheads:checked(passenger)
    local count = GuardAuto.TicketStates.correctChecks
    count:set(count:get() + 1) -- Same local counter update as the native submit button.
    if reject then GuardAuto.Rejected = GuardAuto.Rejected + 1
    else GuardAuto.Accepted = GuardAuto.Accepted + 1 end
end

local guardActions = {
    OpenLocal = true, CloseLocal = true, UnlockDoors = true, LockDoors = true,
    SignalBell = true, UnlockPanel = true, ClosePanel = true,
}

local function runGuardAuto(now)
    finishGuardMenuTransition(now, GameplayStates.guardedTrain)
    State.CurrentPhase = "GUARD / INITIALIZING"
    if not initGuardAuto(now) then return end
    local train = GameplayStates.guardedTrain
    if GuardAuto.Train ~= train then
        GuardAuto.Train = train
        GuardAuto.Checked = setmetatable({}, {__mode = "k"})
        GuardAuto.Accepted, GuardAuto.Rejected = 0, 0
        GuardAuto.LastAction, GuardAuto.LastTicket = 0, 0
        GuardAuto.PanelDoor, GuardAuto.PanelAttachment = nil, nil
        GuardAuto.NextPanelSearch = 0
        GuardAuto.LastPanelTeleport, GuardAuto.LastPanelAction = -math.huge, -math.huge
    end
    local parts = GuardAuto.TrainUpdates.parts[train.Name]
    local speed = parts and parts.speed
    State.CurrentSpeed = type(speed) == "number" and math.abs(speed) or 0
    local stage = GuardAuto.Handler:getCurrentStage() or {}
    local action = stage.Action
    local doors = GuardAuto.Handler:getDoorState()
    if not State.Enabled or not CONFIG.AutoGuard then
        State.CurrentPhase = "GUARD / DISABLED"
        return
    end
    if GameplayStates.currentTrain ~= train then
        State.CurrentPhase = "GUARD / BOARD YOUR TRAIN"
        return
    end
    if CONFIG.GuardAutoPanel and action
        and not (action == "Wait" and stage.For == "Departed") then
        if not ensureGuardPanel(train, now) then return end
    end
    if type(speed) ~= "number" then
        State.CurrentPhase = "GUARD / WAITING FOR TRAIN DATA"
        return
    end
    if math.abs(speed) > 0.5 then
        State.CurrentPhase = "GUARD / CHECKING TICKETS"
        if doors == "Locked" then checkGuardTicket(train, now) end
        return
    end
    if action == "Wait" then
        State.CurrentPhase = "GUARD / WAIT " .. tostring(stage.For or "DRIVER")
        return -- Native driver/server messages advance these stages.
    end
    if not action then
        State.CurrentPhase = doors == "Open" and "GUARD / BOARDING - WAIT TO CLOSE" or "GUARD / WAITING FOR STATION"
        return
    end
    State.CurrentPhase = "GUARD / " .. tostring(action)
    if not CONFIG.GuardAutoDispatch or not guardActions[action]
        or now - GuardAuto.LastAction < CONFIG.GuardActionInterval then return end
    GuardAuto.LastAction = now
    if not ensureGuardPanel(train, now) then return end
    if not GuardAuto.Handler:isPanelUnlocked() then
        GuardAuto.Buttons.unlock()
        return
    end
    if action == "SignalBell" then
        GuardAuto.Buttons.signalBell() -- Completes the stage and sends the native buzzer.
    else
        GuardAuto.Handler:stageCompleted(action)
    end
end

local function updateGuardHud()
    TitleLabel.Text = "BR AUTOPILOT / GUARD"
    StatusLabel.RichText = false
    StatusLabel.Text = State.CurrentPhase
    SpeedRow.Text = string.format("Guard | Speed: %.1f MPH", State.CurrentSpeed)
    local train = GameplayStates.guardedTrain
    StationRow.Text = "Train: " .. (train and train.Name or "None")
    SignalRow.RichText = false
    SignalRow.Text = string.format("Tickets: %d accepted | %d rejected", GuardAuto.Accepted, GuardAuto.Rejected)
    TimeRow.Text = "Dispatch: " .. (CONFIG.GuardAutoDispatch and "AUTO" or "OFF")
    PhysicsRow.Text = "Tickets between stations | P = ON/OFF"
end


local GuardRequest = GuardingModule._BR_AutoGuardRequestState or {
    Pending = nil, SentAt = 0, AcceptedUntil = 0, NextScan = os.clock() + 3,
    NextInit = 0, LastError = 0, Status = "SEARCHING", Target = nil, Blocked = {},
    Map = nil, Trains = nil, Catalog = nil, Remote = nil, Connection = nil,
    LengthCache = {},
}
-- Preserve the outstanding request and its original send time across reloads.
GuardingModule._BR_AutoGuardRequestState = GuardRequest

local function guardUnitCars(unit, catalog)
    local parts = string.split(unit, ".")
    local trainId = (parts[1] or ""):gsub("%-f$", "")
    local definition = catalog[trainId]
    if not definition then return nil end
    local variant = #parts >= 3 and definition.variants and definition.variants[parts[2]] or nil
    local liveries = variant and variant.liveries or definition.variants
    local liveryName = parts[#parts]
    local livery = nil
    if type(liveries) == "table" then
        for _, value in pairs(liveries) do
            if type(value) == "table" and value.name == liveryName then livery = value; break end
        end
    end
    local length = (livery and livery.length) or (variant and variant.length) or definition.length or 1
    local count = tonumber(length)
    return count and count > 0 and count or nil
end

local function chooseGuardRequestCandidate(candidates, blocked, now, epoch)
    local eligible = {}
    for _, candidate in ipairs(candidates) do
        if candidate.allowed and not candidate.guard and not candidate.self
            and (not candidate.cooldown or candidate.cooldown <= epoch)
            and (blocked[candidate.name] or 0) <= now then
            table.insert(eligible, candidate)
        end
    end
    table.sort(eligible, function(a, b)
        local aSemi = a.line:lower():gsub("[^%a]", "") == "semifast"
        local bSemi = b.line:lower():gsub("[^%a]", "") == "semifast"
        if aSemi ~= bSemi then return aSemi end
        if a.cars ~= b.cars then return a.cars > b.cars end
        if a.length ~= b.length then return a.length > b.length end
        if a.looking ~= b.looking then return a.looking end
        return a.name < b.name
    end)
    return eligible[1]
end

local function finishGuardRequest(state, now, status)
    if state.Pending then state.Blocked[state.Pending] = now + CONFIG.GuardRequestRetryBackoff end
    state.Pending = nil
    state.Status = status
    state.NextScan = math.max(state.NextScan, now + CONFIG.GuardRequestInterval)
end

local function receiveGuardRequestResponse(state, action, target, now)
    if not state.Pending then return end
    if action == "Accept" then
        if type(target) == "string" and target ~= state.Pending then return end
        finishGuardRequest(state, now, "JOINING")
        state.AcceptedUntil = now + CONFIG.GuardSpawnTimeout + 10
    elseif action == "Reject" or action == "Fail" or action == "Cooldown"
        or action == "Despawned" or action == "TimeOutGuard" then
        finishGuardRequest(state, now, action == "Reject" and "REJECTED" or "RETRY LATER")
    end
end

local function stepGuardRequests(state, dependencies, now, epoch)
    if dependencies.busy() then
        state.Pending, state.AcceptedUntil = nil, 0
        state.Status = "WORKING"
        return
    end
    if not dependencies.enabled() then
        if state.Pending and dependencies.pending() == state.Pending then dependencies.cancel() end
        state.Pending, state.AcceptedUntil = nil, 0
        state.Status = "OFF"
        return
    end
    if dependencies.joining() or now < state.AcceptedUntil then state.Status = "JOINING"; return end
    local pending = dependencies.pending()
    if pending then
        state.Status = state.Pending == pending and "REQUEST PENDING" or "WAITING FOR CURRENT REQUEST"
        if state.Pending == pending and now - state.SentAt >= CONFIG.GuardRequestTimeout then
            dependencies.cancel()
            finishGuardRequest(state, now, "REQUEST TIMED OUT")
            state.NextScan = now -- Try another train on the next control tick, without an extra delay.
        end
        return
    end
    if state.Pending then
        -- Leave a short margin for the native response listener to run.
        if now - state.SentAt < 1 then return end
        finishGuardRequest(state, now, "RETRY LATER")
    end
    if now < state.NextScan then state.Status = "SEARCH COOLDOWN"; return end
    state.NextScan = now + CONFIG.GuardRequestSearchInterval
    local best = chooseGuardRequestCandidate(dependencies.candidates(), state.Blocked, now, epoch)
    if not best then state.Status = "NO AVAILABLE TRAINS"; return end
    state.Pending, state.Target, state.SentAt = best.name, best, now
    state.Status = "REQUEST PENDING"
    local ok, sent = pcall(dependencies.send, best)
    if not ok or sent == false then
        finishGuardRequest(state, now, "REQUEST FAILED")
        return
    end
    state.NextScan = now + CONFIG.GuardRequestInterval
end

local function initGuardRequests(now)
    if GuardRequest.Map then
        if not GuardRequest.Connection or not GuardRequest.Connection.Connected then
            GuardRequest.Connection = GuardRequest.Remote.OnClientEvent:Connect(function(action, target)
                receiveGuardRequestResponse(GuardRequest, action, target, os.clock())
            end)
        end
        return true
    end
    if now < GuardRequest.NextInit then return false end
    GuardRequest.NextInit = now + 5
    local events = game:GetService("ReplicatedStorage"):FindFirstChild("Events")
    local remote = events and events:FindFirstChild("GuardReq")
    if not remote or not remote:IsA("RemoteEvent") then return false end
    local map = nil
    for _, connection in ipairs(getconnections(remote.OnClientEvent)) do
        if type(connection.Function) == "function" then
            for _, value in pairs(getupvalues(connection.Function)) do
                if type(value) == "table" and value.isPending and value.requestAccepted
                    and value.requestRejected and value.selectedTrain and type(value.setCooldown) == "function" then
                    map = value
                    break
                end
            end
        end
        if map then break end
    end
    if not map then return false end
    local trains = nil
    for _, value in pairs(getupvalues(map.setCooldown)) do
        if type(value) == "table" and value ~= map then
            for _, data in pairs(value) do
                if type(data) == "table" and data.canShow and data.reqSetting and data.unitString then
                    trains = value
                    break
                end
            end
        end
        if trains then break end
    end
    if not trains then return false end
    local storage = game:GetService("ReplicatedStorage")
    local config = storage:FindFirstChild("Config")
    local data = storage:FindFirstChild("Data")
    local catalogModule = (config and config:FindFirstChild("Trains")) or (data and data:FindFirstChild("Trains"))
    if not catalogModule then return false end
    GuardRequest.Catalog = require(catalogModule)
    GuardRequest.Map, GuardRequest.Trains, GuardRequest.Remote = map, trains, remote
    GuardRequest.Connection = remote.OnClientEvent:Connect(function(action, target)
        receiveGuardRequestResponse(GuardRequest, action, target, os.clock())
    end)
    return true
end

local function guardCandidateCars(data, model)
    local known = model and (model:GetAttribute("Cars") or model:GetAttribute("Coaches"))
    if type(known) == "number" and known > 0 then return known end
    local units = data.unitString
    if type(units) ~= "string" then return 0 end
    if GuardRequest.LengthCache[units] ~= nil then return GuardRequest.LengthCache[units] end
    local ok, parsed = pcall(function() return game:GetService("HttpService"):JSONDecode(units) end)
    local total = 0
    if ok and type(parsed) == "table" then
        for _, unit in ipairs(parsed) do
            if type(unit) ~= "string" then return 0 end
            local cars = guardUnitCars(unit, GuardRequest.Catalog)
            if not cars then return 0 end
            total = total + cars
        end
    end
    GuardRequest.LengthCache[units] = total
    return total
end

local function collectGuardRequestCandidates()
    local candidates = {}
    local visible = workspace:FindFirstChild("Trains")
    local hidden = game:GetService("ReplicatedStorage"):FindFirstChild("HiddenTrainStorage")
    for name, data in pairs(GuardRequest.Trains) do
        if type(data) == "table" and data.canShow and data.canShow:get() then
            local model = (visible and visible:FindFirstChild(name)) or (hidden and hidden:FindFirstChild(name))
            local guard = data.guard or (model and model:GetAttribute("Guard"))
            local line = data.line and data.line:get() or "Other"
            table.insert(candidates, {
                name = name, line = tostring(line), allowed = true,
                self = name == LocalPlayer.Name, guard = guard and guard ~= "" and guard or nil,
                cooldown = data.cooldown, cars = guardCandidateCars(data, model),
                length = (model and model:GetAttribute("Length")) or 0,
                looking = data.reqSetting == "Looking For Guard",
            })
        end
    end
    return candidates
end

local function runAutoGuardRequests(now)
    if isGuarding() or GameplayStates.isDriving then return end
    if not State.Enabled or not CONFIG.AutoGuardRequests or not CONFIG.AutoGuard then
        if GuardRequest.Map then
            stepGuardRequests(GuardRequest, {
                busy = function() return false end,
                enabled = function() return false end,
                pending = function() return GuardRequest.Map.isPending:get() end,
                cancel = function()
                    GuardRequest.Remote:FireServer("Cancel")
                    GuardRequest.Map.isPending:set(false)
                end,
            }, now, os.time())
        else GuardRequest.Status = "OFF" end
        return
    end
    if not initGuardRequests(now) then GuardRequest.Status = "WAITING FOR TRAIN LIST"; return end
    if not GuardRequest.Pending and not GuardRequest.Map.isPending:get()
        and not GuardJoinStatus.Pending and now >= GuardRequest.AcceptedUntil
        and now >= GuardRequest.NextScan and not prepareGuardRequestMenu(now) then
        GuardRequest.Status = "WAITING FOR GUARD MENU"
        return
    end
    stepGuardRequests(GuardRequest, {
        busy = function() return isGuarding() or GameplayStates.isDriving end,
        enabled = function() return State.Enabled and CONFIG.AutoGuardRequests and CONFIG.AutoGuard end,
        joining = function() return GuardJoinStatus.Pending end,
        pending = function() return GuardRequest.Map.isPending:get() end,
        candidates = collectGuardRequestCandidates,
        cancel = function()
            GuardRequest.Remote:FireServer("Cancel")
            GuardRequest.Map.isPending:set(false)
        end,
        send = function(best)
            -- Recheck native availability before the same operation as the menu button.
            if isGuarding() or GameplayStates.isDriving or GuardJoinStatus.Pending
                or GuardRequest.Map.isPending:get() then return false end
            local data = GuardRequest.Trains[best.name]
            if not data or not data.canShow:get() or data.guard then return false end
            GuardRequest.Map.requestRejected:set(false)
            GuardRequest.Map.requestAccepted:set(false)
            GuardRequest.Map.selectedTrain:set(best.name)
            GuardRequest.Map.isPending:set(best.name)
            GuardRequest.Remote:FireServer("Send", best.name)
            return true
        end,
    }, now, os.time())
end

local function updateGuardRequestHud()
    TitleLabel.Text = "BR AUTOPILOT / GUARD SEARCH"
    StatusLabel.RichText = false
    StatusLabel.Text = "AUTO GUARD / " .. GuardRequest.Status
    local target = GuardRequest.Target
    SpeedRow.Text = target and string.format("%s | %d cars", target.line, target.cars) or "Preference: long Semifast trains"
    StationRow.Text = "Train: " .. (target and target.name or "Searching")
    SignalRow.RichText = false
    SignalRow.Text = GuardRequest.Pending and "Waiting for driver response" or "Available trains only"
    TimeRow.Text = string.format("Next search: %.0fs", math.max(0, GuardRequest.NextScan - os.clock()))
    PhysicsRow.Text = "Automatic requests | P = ON/OFF"
end

GuardInstance.Destroying:Connect(function()
    if GuardRequest.Connection then GuardRequest.Connection:Disconnect() end
end)


task.spawn(function()
    while isRunning do
        task.wait(CONFIG.ControlInterval)
        if not isRunning then break end
        if isGuarding() then
            local ok, err = pcall(runGuardAuto, os.clock())
            if not ok then
                State.CurrentPhase = "GUARD / RETRYING"
                if os.clock() - (GuardAuto.LastError or 0) > 5 then
                    GuardAuto.LastError = os.clock()
                    warn("[BR Autopilot Guard] " .. tostring(err))
                end
            end
            updateGuardHud()
        elseif GameplayStates.isDriving then
            TitleLabel.Text = "BR AUTOPILOT (SIMPLIFIED)"
            if State.LastRole ~= "DRIVER" then
                State.StationPhase = "NONE"
                State.CurrentStopStation = ""
                State.CurrentStopIsTerminus = false
                resetBrakeFeedback(false)
            end
            if CONFIG.AutoAWS then pcall(function() Triggers:call("AWS") end) end
            local ctx = readRuntimeSnapshot()
            if CONFIG.AutoDeclineGuard then autoDeclineGuardRequests() end
            tryAutoSelectNextRoute()
            runAutopilot(ctx)
            updateHud(ctx)
        else
            GuardMenu.SyncedTrain = nil
            local ok, err = pcall(runAutoGuardRequests, os.clock())
            if not ok then
                GuardRequest.Status = "RETRYING TRAIN SEARCH"
                if os.clock() - (GuardRequest.LastError or 0) > 5 then
                    GuardRequest.LastError = os.clock()
                    warn("[BR Autopilot Requests] " .. tostring(err))
                end
            end
            updateGuardRequestHud()
            if GuardJoinStatus.Pending then
                StatusLabel.Text = string.format("GUARD / SPAWNING (%.0fs)",
                    math.max(0, GuardJoinStatus.Deadline - os.clock()))
            end
        end
        State.LastRole = isGuarding() and "GUARD" or (GameplayStates.isDriving and "DRIVER" or "NONE")
    end
end)

GuardInstance.Destroying:Connect(function()
    isRunning = false
    if keyConn then keyConn:Disconnect() end
    if antiAfkConn then antiAfkConn:Disconnect() end
    if ScreenGui then ScreenGui:Destroy() end
    if GameplayStates.isDriving and not isGuarding() then pcall(holdStopped) end
end)
print("[BR Autopilot v4.11] Ready: Simplified driver + automatic guard dispatch/ticketing. P = toggle.")
]==])
