-- GR33D Panel v7.7: anti-fling stationary guard + pre/post physics correction; AFK Streak, flight, and escape retained
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local CollectionService = game:GetService("CollectionService")
local Workspace = game:GetService("Workspace")

local player = Players.LocalPlayer
if not player then
    warn("[GR33D] Run this as a LocalScript, not a server Script.")
    return
end
local playerGui = player:WaitForChild("PlayerGui", 10)
if not playerGui then
    warn("[GR33D] PlayerGui was not available; panel was not started.")
    return
end

-- LocalScript utilities for games you control. Server physics/damage remains authoritative.
local Config = {
    ScanInterval = 0.06, -- ~16 scans/second; HP response is immediate via HealthChanged
    IndexBatchSize = 120,
    UseHazardNameHints = true, -- only moving, on-course projectiles; names alone never cause a dodge
    UseNPCNameHints = false,
    EscapeMinDuration = 1.7,
    EscapeCooldown = 1.35,
    ReturnSafeGrace = 0.65,
    EscapeHeight = 14,
    MaxEscapeDistance = 65,
    -- Emergency tier: react to small real HP drops, or strongly abnormal
    -- motion/grabbing of nearby hazard parts. Keep normal small dodges separate.
    EmergencyHeight = 100,
    EmergencyMinDuration = 3.5,
    EmergencyReturnQuiet = 2.6,
    EmergencyRetriggerCooldown = 0.45,
    EmergencyGuardRadius = 34,
    EmergencyProactiveRadiusCap = 45, -- ignore giant name-based danger radii
    EmergencyFastThreatSpeed = 65,
    EmergencySpinThreshold = 18,
    EmergencyDirectionChangeSpeed = 25,
    EmergencyDamageFraction = 0.001,
    EmergencyDamageAbsolute = 0.05, -- 0.05 HP or less is enough on default health
    EmergencyMinDamageThreshold = 0.025, -- filter numerical jitter
    DamageBurstWindow = 0.7,
    DamageBurstCount = 2,
    EmergencyHomeDangerHold = 1.4,
    ShieldDiameter = 8,
    ChaseRelocateCooldown = 0.28, -- do not spin around every scan frame
    ChaseGuardRadius = 27,
    ChaseQuietWindow = 1.1,
    EscapeHoldResponsiveness = 150,
    EscapeHoldMaxVelocity = 420,
    EscapeDriftCorrection = 13,
    ManualRearmDelay = 0.35,
    ManualTeleportLiftSteps = {0, 1.5, 3, 5, 8, 12, 18, 25},
    HazardMargin = 2.5,
    ProjectileSpeedThreshold = 9,
    PredictionSeconds = 1.0,
    FlingLinearThreshold = 72, -- ordinary movement still permitted up to Run Speed + margin
    FlingAngularThreshold = 20,
    FlingRecoveryCooldown = 0.05, -- rate-limit notices, not actual protection
    FlingImpulseDeltaThreshold = 26, -- suspicious single-step speed change (studs/s)
    FlingFrameDistance = 5.5, -- unusually large displacement in one simulation step
    FlingStationaryMaxSpeed = 4.5,
    FlingStationaryMaxVertical = 7,
    FlingGroundSnapshotAge = 0.55,
    FlingInputReleaseGrace = 0.30,
    SafePositionMaxAge = 3,
    FallSpeedLimit = 45,
    FlightSpeed = 50,
    FlightMaxForce = 100000,
    MinRunSpeed = 5,
    MaxRunSpeed = 200,
    DoubleTapWindow = 0.35,
    NPCStartDistance = 15,
    NPCStopDistance = 25,
    NPCLiftHeight = 15,
    NPCLiftCooldown = 1.5,
    IdleSettleTime = 0.55,
    IdleSettleRadius = 1.8, -- small rig/physics jitter does not reset the idle timer
    IdleMaxHorizontalSpeed = 4.5,
    IdleInputGrace = 0.28,
    IdleMovingPlatformSpeed = 1.25,
    AFKPulsingInterval = 18,
    AFKPulsingDuration = 0.16,
    AFKMovementPulse = 0.025,
    -- Broad 'Disaster'/'Volcano' labels do NOT imply an imminent impact.
    HazardTags = {"ProtectionHazard", "Hazard", "Disaster", "Nuke", "NuclearMissile",
        "Tsunami", "TsunamiWave", "Lightning", "C4Bomb", "Volcano", "Meteor",
        "Deadzone", "Killzone", "Explosion", "Bomb", "Trap", "Lava"},
    ConfirmedHazardTags = {"ProtectionHazard", "Nuke", "NuclearMissile", "TsunamiWave",
        "Lightning", "C4Bomb", "Meteor", "Deadzone", "Killzone", "Explosion",
        "Bomb", "Trap", "Lava"},
    -- These names identify candidate *parts*, not lethal proof or a damage radius.
    ProjectileHints = {"magma", "lava", "meteor", "meteorite", "missile", "nuclearmissile",
        "nuke", "nuclear", "rocket", "grenade", "bomb", "comet", "asteroid", "lightning",
        "fireball", "acid", "tsunamiwave", "warhead"},
    HostileTags = {"Hostile", "Enemy", "Monster", "NPC_Hostile", "Chaser", "Boss", "Zombie"},
    FriendlyTags = {"Friendly", "Neutral", "Passive", "NonHostile", "ProtectionIgnore"},
    HazardAttributes = {"Damage", "DamageAmount", "Hazard", "KillZone", "IsDisaster",
        "Dangerous", "Lethal", "Kill", "Deadly", "Explosive"},
}

-- All state exists before callbacks capture it.
local S = {
    alive = true, context = nil, setupToken = 0, connections = {}, characterConnections = {},
    watchers = {}, hazards = {}, motion = {}, npcs = {}, pending = {}, pendingIndex = 1,
    properties = {}, states = {}, flags = {}, switches = {}, ui = {}, cameraConnection = nil,
    controls = nil, runSpeed = 50, flying = false, hovering = false, mover = nil,
    lockTarget = nil, escape = nil, lastEscape = -math.huge, idleTarget = nil,
    idleSince = nil, idleSample = nil, idleBlockedUntil = -math.huge,
    afkLastPulse = os.clock(), afkPulseUntil = nil, afkPostPulseUntil = -math.huge,
    safeRoot = nil, safeTime = -math.huge, hoverY = nil,
    lastDrop = -math.huge, previousHealth = nil, healing = false, lastJump = -math.huge,
    lastJumpRequest = -math.huge, scanClock = 0, notice = "Client panel ready", noticeUntil = 0,
    savedLocation = (typeof(player:GetAttribute("GR33D_SavedLocation")) == "CFrame"
        and player:GetAttribute("GR33D_SavedLocation") or nil), entrySpawn = nil, joinSpawn = nil,
    lastDamageAt = -math.huge, damageEvents = {}, lastSevereThreatAt = -math.huge,
    lastEmergency = -math.huge, homeThreatAt = -math.huge,
    lastFlingRecovery = -math.huge,
    flingHoldTarget = nil, flingPauseUntil = -math.huge, flingInputUntil = -math.huge,
    flingSnapshot = nil, flingLastStableCF = nil, flingLastStableAt = -math.huge,
    flingRecovering = false,
    lockSuspended = false, escapeHold = nil, lastEscapeDriftCorrection = -math.huge,
    lastChaseRelocate = -math.huge, lastChaseThreatAt = -math.huge,
    impactEntries = {}, recentHazard = nil,
    minimized = false, dragging = nil, suppressMinimizeUntil = -math.huge,
}
local H = {}
for _, key in ipairs({"flight", "health", "fall", "noclip", "fling", "trap", "run", "lock", "monster", "afk"}) do
    S.flags[key] = false
end

function H.finite(number)
    return type(number) == "number" and number == number and math.abs(number) < math.huge
end

function H.finiteVector(vector)
    return H.finite(vector.X) and H.finite(vector.Y) and H.finite(vector.Z)
end

function H.connect(signal, callback, bucket)
    local connection = signal:Connect(callback)
    table.insert(bucket or S.connections, connection)
    return connection
end

function H.disconnectAll(bucket)
    for _, connection in ipairs(bucket) do connection:Disconnect() end
    for i = #bucket, 1, -1 do bucket[i] = nil end
end

function H.notice(message, duration)
    S.notice = message
    S.noticeUntil = os.clock() + (duration or 3)
end

function H.report(label, message)
    warn("[GR33D] " .. label .. ": " .. tostring(message))
    H.notice(label .. " failed; see Output", 5)
end

function H.ready()
    local ctx = S.context
    return S.alive and ctx ~= nil and player.Character == ctx.character
        and ctx.character:IsDescendantOf(Workspace) and ctx.root:IsDescendantOf(ctx.character)
        and ctx.humanoid:IsDescendantOf(ctx.character) and ctx.humanoid.Health > 0
end

-- Property leases allow noclip + hover to share CanCollide without bad restores.
function H.claim(instance, property, value, owner)
    local fields = S.properties[instance]
    if not fields then fields = {}; S.properties[instance] = fields end
    local entry = fields[property]
    if not entry then
        entry = {original = instance[property], owners = {}}
        fields[property] = entry
    end
    entry.owners[owner] = true
    if instance[property] ~= value then instance[property] = value end
end

function H.claimState(humanoid, state, enabled, owner)
    local fields = S.states[humanoid]
    if not fields then fields = {}; S.states[humanoid] = fields end
    local entry = fields[state]
    if not entry then
        entry = {original = humanoid:GetStateEnabled(state), owners = {}}
        fields[state] = entry
    end
    entry.owners[owner] = true
    if humanoid:GetStateEnabled(state) ~= enabled then humanoid:SetStateEnabled(state, enabled) end
end

function H.release(owner)
    for instance, fields in pairs(S.properties) do
        for property, entry in pairs(fields) do
            if entry.owners[owner] then
                entry.owners[owner] = nil
                if next(entry.owners) == nil then
                    local ok, message = pcall(function() instance[property] = entry.original end)
                    if not ok and instance.Parent then H.report("Restore " .. property, message) end
                    fields[property] = nil
                end
            end
        end
        if next(fields) == nil then S.properties[instance] = nil end
    end
    for humanoid, fields in pairs(S.states) do
        for state, entry in pairs(fields) do
            if entry.owners[owner] then
                entry.owners[owner] = nil
                if next(entry.owners) == nil then
                    local ok, message = pcall(function() humanoid:SetStateEnabled(state, entry.original) end)
                    if not ok and humanoid.Parent then H.report("Restore humanoid state", message) end
                    fields[state] = nil
                end
            end
        end
        if next(fields) == nil then S.states[humanoid] = nil end
    end
end

function H.releaseAll()
    local owners = {}
    for _, fields in pairs(S.properties) do
        for _, entry in pairs(fields) do
            for owner in pairs(entry.owners) do owners[owner] = true end
        end
    end
    for _, fields in pairs(S.states) do
        for _, entry in pairs(fields) do
            for owner in pairs(entry.owners) do owners[owner] = true end
        end
    end
    for owner in pairs(owners) do H.release(owner) end
end

-- Standalone LocalScript. This cannot override authoritative server collision/damage.

function H.zeroVelocity()
    if S.context then
        S.context.root.AssemblyLinearVelocity = Vector3.zero
        S.context.root.AssemblyAngularVelocity = Vector3.zero
    end
end

function H.moveRoot(target)
    if not H.ready() or not H.finiteVector(target.Position) then return false end
    local ctx = S.context
    -- PivotTo takes a MODEL pivot, not necessarily the HumanoidRootPart CFrame.
    local rootToPivot = ctx.root.CFrame:ToObjectSpace(ctx.character:GetPivot())
    ctx.character:PivotTo(target * rootToPivot)
    H.zeroVelocity()
    return true
end

function H.tokens(name)
    local tokens = {}
    local splitName = name:gsub("(%l)(%u)", "%1 %2"):lower()
    for word in splitName:gmatch("%w+") do tokens[word] = true end
    tokens[name:lower():gsub("[^%w]", "")] = true
    return tokens
end

function H.projectileNameHint(name)
    if not Config.UseHazardNameHints then return false end
    local tokens = H.tokens(name)
    for _, word in ipairs(Config.ProjectileHints) do
        if tokens[word] then return true end
    end
    return false
end

function H.hasTag(instance, tags)
    for _, tag in ipairs(tags) do
        if CollectionService:HasTag(instance, tag) then return true end
    end
    return false
end

-- Ignore ordinary avatar limbs and accessories, but not genuinely hazardous
-- objects temporarily held by another player (e.g. an equipped nuclear tool).
function H.ignored(instance)
    if player.Character and instance:IsDescendantOf(player.Character) then return true end
    local nominated = H.projectileNameHint(instance.Name)
        or H.hasTag(instance, Config.ConfirmedHazardTags)
        or instance:GetAttribute("Lethal") == true
        or instance:GetAttribute("KillZone") == true
        or (H.finite(instance:GetAttribute("Damage")) and instance:GetAttribute("Damage") > 0)
    local current = instance
    while current and current ~= Workspace do
        if current:GetAttribute("ProtectionIgnore") == true
            or CollectionService:HasTag(current, "ProtectionIgnore") then return true end
        if current:IsA("Accessory") then return true end
        if H.projectileNameHint(current.Name)
            or H.hasTag(current, Config.ConfirmedHazardTags)
            or current:GetAttribute("Lethal") == true
            or current:GetAttribute("KillZone") == true
            or (H.finite(current:GetAttribute("Damage")) and current:GetAttribute("Damage") > 0) then
            nominated = true
        end
        if current:IsA("Tool") or (current:IsA("Model") and Players:GetPlayerFromCharacter(current)) then
            if not nominated then return true end
        end
        current = current.Parent
    end
    return false
end

function H.projectileHintInTree(instance)
    local current = instance
    while current and current ~= Workspace do
        if H.projectileNameHint(current.Name) then return true end
        current = current.Parent
    end
    return false
end

function H.grabEvidence(instance)
    local current = instance
    while current and current ~= Workspace do
        for _, attribute in ipairs({"IsGrabbed", "Grabbed", "IsHeld", "Held", "BeingCarried"}) do
            if current:GetAttribute(attribute) == true then return true end
        end
        for _, attribute in ipairs({"GrabbedBy", "HeldBy", "CarriedBy"}) do
            local value = current:GetAttribute(attribute)
            if value ~= nil and value ~= false and value ~= "" and value ~= 0 then return true end
        end
        if current:IsA("Tool") and current.Parent
            and current.Parent:IsA("Model")
            and Players:GetPlayerFromCharacter(current.Parent) then return true end
        current = current.Parent
    end
    return false
end

-- A danger label is only a *candidate*. Dodge requires a real contact / predicted impact.
function H.hazardInfo(instance)
    if not instance.Parent or not instance:IsDescendantOf(Workspace) or H.ignored(instance) then return nil end
    if not (instance:IsA("BasePart") or instance:IsA("Model") or instance:IsA("Explosion")) then return nil end

    local confirmed, explicitlyActive, inactive = instance:IsA("Explosion"), false, false
    local radius = 0
    local activeRadius
    local cur = instance
    while cur and cur ~= Workspace do
        for _, attr in ipairs({"Active", "IsActive", "Armed"}) do
            local value = cur:GetAttribute(attr)
            if value == false then inactive = true end
            if value == true then explicitlyActive = true end
        end
        local value = cur:GetAttribute("Damage") or cur:GetAttribute("DamageAmount")
        if H.finite(value) and value > 0 then confirmed = true end
        for _, attr in ipairs({"Lethal", "KillZone", "Kill", "Deadly"}) do
            if cur:GetAttribute(attr) == true then confirmed = true end
        end
        if H.hasTag(cur, Config.ConfirmedHazardTags) then confirmed = true end
        if activeRadius == nil then
            local requested = cur:GetAttribute("HazardRadius")
            if requested == nil then requested = cur:GetAttribute("DangerRadius") end
            if H.finite(requested) and requested >= 0 then
                activeRadius = math.clamp(requested, 0, 500)
            end
        end
        cur = cur.Parent
    end
    if inactive then return nil end

    local bounds, size, part
    if instance:IsA("BasePart") then
        bounds, size, part = instance.CFrame, instance.Size, instance
    elseif instance:IsA("Model") then
        part = instance.PrimaryPart or instance:FindFirstChildWhichIsA("BasePart", true)
        if not part then return nil end
        bounds, size = instance:GetBoundingBox()
    else
        bounds, size = CFrame.new(instance.Position), Vector3.zero
        radius = math.max(0, instance.BlastRadius)
    end
    if not H.finiteVector(bounds.Position) or not H.finiteVector(size) then return nil end

    -- Tweened projectiles may report zero physics velocity, so sample their motion too.
    local velocity = part and part.AssemblyLinearVelocity or Vector3.zero
    if not H.finiteVector(velocity) then velocity = Vector3.zero end
    local now = os.clock()
    local previous = S.motion[instance]
    local sampledVelocity = Vector3.zero
    local abruptReversal = false
    if previous and now - previous.time >= Config.ScanInterval * 0.75 then
        local elapsed = now - previous.time
        sampledVelocity = (bounds.Position - previous.position) / elapsed
        local before = previous.velocity
        -- A high-speed direction reversal is stronger evidence of violent
        -- handling than ordinary falling/accelerating volcanic projectiles.
        if before.Magnitude >= Config.EmergencyDirectionChangeSpeed
            and sampledVelocity.Magnitude >= Config.EmergencyDirectionChangeSpeed then
            abruptReversal = before.Unit:Dot(sampledVelocity.Unit) < -0.2
        end
        S.motion[instance] = {position = bounds.Position, time = now,
            velocity = sampledVelocity, reversedAt = abruptReversal and now or previous.reversedAt}
    elseif not previous then
        S.motion[instance] = {position = bounds.Position, time = now,
            velocity = Vector3.zero, reversedAt = -math.huge}
    else
        sampledVelocity = previous.velocity
    end
    local motion = S.motion[instance]
    abruptReversal = abruptReversal or (motion and motion.reversedAt
        and now - motion.reversedAt < 0.18) or false
    if H.finiteVector(sampledVelocity) and sampledVelocity.Magnitude > velocity.Magnitude then
        velocity = sampledVelocity
    end

    local moving = velocity.Magnitude >= Config.ProjectileSpeedThreshold
    local hinted = instance:IsA("BasePart") and H.projectileHintInTree(instance)
    -- An ordinary named / static decoration should NEVER trigger the escape system.
    if not confirmed and not (hinted and (moving or explicitlyActive or H.grabEvidence(instance))) then return nil end
    -- Tags on large *container models* are often scenery: only explicitly marked
    -- regions/models qualify, and movement is evaluated using their own bounding box.
    if instance:IsA("Model") and not (instance:GetAttribute("Lethal") == true
        or instance:GetAttribute("KillZone") == true
        or instance:GetAttribute("HazardRadius") ~= nil
        or H.hasTag(instance, Config.ConfirmedHazardTags)) then return nil end

    if activeRadius ~= nil then radius = activeRadius end
    local angular = part and part.AssemblyAngularVelocity or Vector3.zero
    if not H.finiteVector(angular) then angular = Vector3.zero end
    return {instance = instance, name = instance.Name, bounds = bounds, size = size,
        velocity = velocity, angularSpeed = angular.Magnitude,
        moving = moving, radius = radius, grabbed = H.grabEvidence(instance),
        erratic = abruptReversal,
        explosion = instance:IsA("Explosion"), explicit = confirmed}
end

function H.boxDistance(position, bounds, size)
    local offset = bounds:PointToObjectSpace(position)
    local half = size * 0.5
    return Vector3.new(math.max(0, math.abs(offset.X) - half.X),
        math.max(0, math.abs(offset.Y) - half.Y), math.max(0, math.abs(offset.Z) - half.Z)).Magnitude
end

function H.threatAt(info, position)
    local distance = H.boxDistance(position, info.bounds, info.size)
    local required = info.radius + Config.HazardMargin
    if info.explosion then
        return distance <= required, distance, 0
    end
    if info.moving then
        local displacement = position - info.bounds.Position
        local velocity = info.velocity
        local speed2 = velocity:Dot(velocity)
        local t = math.clamp(displacement:Dot(velocity) / speed2, 0, Config.PredictionSeconds)
        local predictedBounds = info.bounds + velocity * t
        local miss = H.boxDistance(position, predictedBounds, info.size)
        -- A projectile moving away from the player must not trigger at a distance.
        if (t > 0 or distance <= required) and miss <= required then
            return true, miss, t
        end
        return false, miss, t
    end
    -- Only explicitly confirmed static kill volumes are actionable. Nearby scenery
    -- (including an idle volcano model) is harmless to the movement system.
    if info.explicit and distance <= required then
        return true, distance, 0
    end
    return false, distance, 0
end

function H.hazardCandidate(instance)
    if H.ignored(instance) then return false end
    if instance:IsA("Explosion") then return true end
    if instance:IsA("BasePart") and H.projectileHintInTree(instance) then return true end
    if not (instance:IsA("BasePart") or instance:IsA("Model")) then return false end
    local cur = instance
    while cur and cur ~= Workspace do
        if H.hasTag(cur, Config.ConfirmedHazardTags) then return true end
        for _, attr in ipairs({"Damage", "DamageAmount", "Lethal", "KillZone", "Kill", "Deadly"}) do
            local value = cur:GetAttribute(attr)
            if value == true or (H.finite(value) and value > 0) then return true end
        end
        cur = cur.Parent
    end
    return false
end

function H.findHazard(position)
    local best, soonest, bestDistance = nil, math.huge, math.huge
    for instance in pairs(S.hazards) do
        local info = H.hazardInfo(instance)
        if info then
            local danger, distance, t = H.threatAt(info, position)
            if danger and (t < soonest or (t == soonest and distance < bestDistance)) then
                info.distance, info.impactTime = distance, t
                best, soonest, bestDistance = info, t, distance
            end
        end
    end
    return best
end

function H.refreshObject(instance)
    if not S.alive or not instance.Parent then return end
    S.hazards[instance] = H.hazardCandidate(instance) or nil
    if instance:IsA("Model") then
        S.npcs[instance] = instance:FindFirstChildOfClass("Humanoid") ~= nil or nil
    end
    -- A model can be inserted before its parts or humanoid exist.
    local ancestor = instance.Parent
    while ancestor and ancestor ~= Workspace do
        if ancestor:IsA("Model") then
            S.hazards[ancestor] = H.hazardCandidate(ancestor) or nil
            S.npcs[ancestor] = ancestor:FindFirstChildOfClass("Humanoid") ~= nil or nil
        end
        ancestor = ancestor.Parent
    end
end

function H.watchObject(instance)
    if not S.alive or not instance:IsDescendantOf(Workspace) then return end
    if instance:IsA("Humanoid") then H.refreshObject(instance); return end
    if not (instance:IsA("BasePart") or instance:IsA("Model") or instance:IsA("Explosion") or instance:IsA("Folder")) then return end
    if S.watchers[instance] then H.refreshObject(instance); return end
    local connections = {}
    S.watchers[instance] = connections
    H.connect(instance:GetPropertyChangedSignal("Name"), function() H.refreshObject(instance) end, connections)
    H.connect(instance.AttributeChanged, function(attribute)
        H.refreshObject(instance)
        if attribute == "ProtectionIgnore" or attribute == "Active" or attribute == "IsActive" or attribute == "Armed" then
            H.queueBranch(instance)
        end
    end, connections)
    H.refreshObject(instance)
end

function H.queueBranch(instance)
    if not S.alive or not instance:IsDescendantOf(Workspace) then return end
    -- Refresh existing watches as well as newly discovered children.
    for _, child in ipairs(instance:GetDescendants()) do table.insert(S.pending, child) end
end

function H.removeObject(instance)
    local connections = S.watchers[instance]
    if connections then H.disconnectAll(connections); S.watchers[instance] = nil end
    S.hazards[instance], S.motion[instance], S.npcs[instance] = nil, nil, nil
end

function H.clearSpace(target)
    if not H.ready() or not H.finiteVector(target.Position)
        or target.Position.Y <= Workspace.FallenPartsDestroyHeight + 8 then return false end
    local ctx = S.context
    -- Character:GetBoundingBox includes oversized pets, shields and cosmetics, which
    -- previously made many perfectly reasonable saved/spawn positions look obstructed.
    local clearanceHeight = math.max(3.8, ctx.humanoid.HipHeight + ctx.root.Size.Y + 1.0)
    local clearance = Vector3.new(math.max(2.6, ctx.root.Size.X + 0.5),
        clearanceHeight, math.max(2.6, ctx.root.Size.Z + 0.5))
    local params = OverlapParams.new()
    params.FilterType = Enum.RaycastFilterType.Exclude
    params.FilterDescendantsInstances = {ctx.character}
    params.MaxParts = 0
    for _, part in ipairs(Workspace:GetPartBoundsInBox(target, clearance * 0.92, params)) do
        if part.CanCollide then return false end
    end
    -- Terrain cannot be found by bounding-box overlap reliably on every terrain shape.
    local rayParams = RaycastParams.new()
    rayParams.FilterType = Enum.RaycastFilterType.Exclude
    rayParams.FilterDescendantsInstances = {ctx.character}
    rayParams.RespectCanCollide = true
    for _, axis in ipairs({target.RightVector * clearance.X * 0.45,
        target.UpVector * clearance.Y * 0.45,
        target.LookVector * clearance.Z * 0.45}) do
        local hit = Workspace:Raycast(target.Position - axis, axis * 2, rayParams)
        if hit and hit.Instance:IsA("Terrain") then return false end
    end
    return true
end

function H.clearEscapePath(target)
    local ctx = S.context
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Exclude
    params.FilterDescendantsInstances = {ctx.character}
    params.RespectCanCollide = true
    local displacement = target.Position - ctx.root.Position
    for _, height in ipairs({0, math.max(2, ctx.root.Size.Y * 0.5 + 1)}) do
        if Workspace:Raycast(ctx.root.Position + Vector3.new(0, height, 0), displacement, params) then return false end
    end
    return true
end

function H.safeDestination(target)
    return H.clearSpace(target) and H.findHazard(target.Position) == nil
end

function H.landingAt(target)
    if not H.ready() then return nil end
    local ctx = S.context
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Exclude
    params.FilterDescendantsInstances = {ctx.character}
    params.RespectCanCollide = true
    local hit = Workspace:Raycast(target.Position + Vector3.new(0, 2, 0),
        Vector3.new(0, -(Config.MaxEscapeDistance + 50), 0), params)
    if not hit or hit.Normal.Y < 0.5 then return nil end
    local height = math.max(3, ctx.humanoid.HipHeight + ctx.root.Size.Y * 0.5 + 0.25)
    local result = CFrame.new(hit.Position + Vector3.new(0, height, 0)) * (target - target.Position)
    if H.safeDestination(result) then return result end
    return nil
end

-- This tier is separate from the small 12-24-stud predicted-impact dodge.
-- It is used for severe/burst damage or abnormally handled disaster objects.
function H.emergencyEscape(reason, pursuit)
    if not S.flags.lock or not H.ready() then return false end
    local now = os.clock()
    local current = S.context.root.CFrame
    local previous = S.escape
    local already = previous and previous.mode == "emergency"
    local home = previous and previous.home or S.lockTarget or current
    if already then
        previous.earliestReturn = math.max(previous.earliestReturn, now + Config.EmergencyMinDuration)
        previous.safeSince = nil
        if now - S.lastEmergency < Config.EmergencyRetriggerCooldown then return true end
        pursuit = pursuit or H.findSevereHazardNear(current.Position)
            or H.findHazard(current.Position)
        -- A new HP drop is enough to shift, even without a named hazard.
        if not pursuit and now - S.lastDamageAt > 0.28 then return true end
    end

    local orientation = current - current.Position
    local base = already and current.Position or home.Position + Vector3.new(0, Config.EmergencyHeight, 0)
    local offsets
    if already then
        local away = pursuit and (current.Position - pursuit.bounds.Position) or current.LookVector
        away = Vector3.new(away.X, 0, away.Z)
        if away.Magnitude < 0.1 then away = Vector3.new(1, 0, 0) end
        away = away.Unit
        local side = Vector3.new(-away.Z, 0, away.X)
        local phase = previous.chaseCount or 0
        if phase % 2 == 1 then side = -side end
        offsets = {
            away * 32 + side * 22 + Vector3.new(0, 13, 0),
            away * 42 - side * 22 + Vector3.new(0, 17, 0),
            side * 40 + Vector3.new(0, 22, 0),
            -side * 40 + Vector3.new(0, 22, 0),
            away * 52 + Vector3.new(0, 12, 0),
            Vector3.new(0, 27, 0) + side * 26,
        }
    else
        offsets = {Vector3.zero,
            Vector3.new(20, 0, 14), Vector3.new(-20, 0, -14),
            Vector3.new(0, 10, 28), Vector3.new(0, 10, -28),
            Vector3.new(28, 16, 0), Vector3.new(-28, 16, 0)}
    end
    for _, offset in ipairs(offsets) do
        local target = CFrame.new(base + offset) * orientation
        if H.safeDestination(target) and H.clearEscapePath(target) then
            local count = (previous and previous.chaseCount or 0) + (already and 1 or 0)
            if H.beginEscapeMove(target, "emergency", home, reason) then
                S.escape.chaseCount = count
                S.escape.earliestReturn = now + Config.EmergencyMinDuration
                S.lastEmergency, S.lastEscape = now, now
                S.lastChaseRelocate = now
                H.notice(already and "EMERGENCY: switched escape angle"
                    or "EMERGENCY: moved approximately 100 studs up", 3)
                return true
            end
        end
    end
    if already then
        previous.safeSince = nil
        H.notice("Chase detected; no clear alternative escape point", 3)
    else
        H.notice("Emergency move blocked by local geometry", 3)
    end
    return false
end

function H.isSevereThreat(info, position)
    if not info then return false end
    local distance = H.boxDistance(position, info.bounds, info.size)
    -- Large model name/tags do not imply an emergency hundreds of studs away.
    local reach = Config.EmergencyGuardRadius
        + math.min(info.radius, Config.EmergencyProactiveRadiusCap)
    if distance > reach then return false end
    return info.grabbed
        or (info.moving and info.velocity.Magnitude >= Config.EmergencyFastThreatSpeed)
        or (info.moving and info.angularSpeed >= Config.EmergencySpinThreshold)
        or (info.moving and info.erratic)
        or (os.clock() - S.lastDamageAt < 0.8 and info.explicit)
end

-- Fast preliminary screening does not require the threat to already be on a
-- collision course. Grabs, violent motion and sudden reversals can precede hits.
function H.findSevereHazardNear(position)
    local closest, bestDistance = nil, math.huge
    for instance in pairs(S.hazards) do
        local info = H.hazardInfo(instance)
        if info and H.isSevereThreat(info, position) then
            local distance = H.boxDistance(position, info.bounds, info.size)
            if distance < bestDistance then
                closest, bestDistance = info, distance
            end
        end
    end
    return closest
end

function H.nearHazard(position, radius)
    local closest, bestDistance = nil, math.huge
    for instance in pairs(S.hazards) do
        local info = H.hazardInfo(instance)
        if info then
            -- Do not allow static name-only scenery to pin the player in the sky.
            if info.explicit or info.moving or info.grabbed then
                local distance = H.boxDistance(position, info.bounds, info.size)
                if distance <= radius + info.radius and distance < bestDistance then
                    closest, bestDistance = info, distance
                end
            end
        end
    end
    return closest
end

function H.recordContact(info)
    if not info then return false end
    local now = os.clock()
    local key = info.instance
    local last = S.impactEntries[key]
    if not last or now - last.last > 3.5 then
        S.impactEntries[key] = {last = now, first = now, count = 1}
        return false
    end
    -- Count repeated encounters only when a new approach happens after a gap,
    -- not every scan frame (that would produce constant false emergencies).
    if now - last.last >= 0.55 then last.count = last.count + 1 end
    last.last = now
    return last.count >= 3
end

function H.escape(hazard, reason)
    if not hazard or not S.flags.lock or not H.ready()
        or (S.escape and S.escape.mode == "emergency")
        or os.clock() - S.lastEscape < Config.EscapeCooldown then return false end
    local ctx = S.context
    local home = S.escape and S.escape.home or S.lockTarget or ctx.root.CFrame
    local origin = ctx.root.CFrame
    local velocity = hazard.velocity
    local away = origin.Position - hazard.bounds.Position
    away = Vector3.new(away.X, 0, away.Z)
    if away.Magnitude < 0.1 and velocity.Magnitude > 0.1 then
        away = Vector3.new(-velocity.Z, 0, velocity.X)
    end
    away = away.Magnitude > 0.1 and away.Unit or Vector3.new(1, 0, 0)
    local side = Vector3.new(-away.Z, 0, away.X)
    local offsets = {side * 12, -side * 12, away * 14,
        side * 20, -side * 20, away * 24,
        Vector3.new(0, Config.EscapeHeight, 0),
        side * 14 + Vector3.new(0, 8, 0),
        -side * 14 + Vector3.new(0, 8, 0)}
    for _, offset in ipairs(offsets) do
        if offset.Magnitude <= Config.MaxEscapeDistance then
            local target = origin + offset
            if H.safeDestination(target) and H.clearEscapePath(target) then
                if H.beginEscapeMove(target, "dodge", home, reason) then
                    S.lastEscape = os.clock()
                    H.notice("Dodged " .. tostring(reason) .. "; lock temporarily released", 3)
                    return true
                end
            end
        end
    end
    H.notice("Danger detected; no unobstructed dodge path", 3)
    return false
end

function H.destroyShield()
    if S.ui.shield and S.ui.shield.Parent then S.ui.shield:Destroy() end
    S.ui.shield = nil
    if S.ui.forceField and S.ui.forceField.Parent then S.ui.forceField:Destroy() end
    S.ui.forceField = nil
end

function H.ensureShield()
    if not H.ready() or not S.flags.lock then return end
    local ctx = S.context
    if S.ui.shield and S.ui.shield.Parent == ctx.character then return end
    H.destroyShield()
    -- The sphere is welded to the locked root; it does not generate a local
    -- touch-triggered damage zone. It is part of the character, so all our
    -- clearance queries exclude it. CanQuery stays true because Roblox requires
    -- CanCollide=false before CanQuery can be disabled.
    local shield = Instance.new("Part")
    shield.Name = "GR33D_InvisibleCollisionShell"
    shield.Shape = Enum.PartType.Ball
    shield.Size = Vector3.new(Config.ShieldDiameter, Config.ShieldDiameter, Config.ShieldDiameter)
    shield.CFrame = ctx.root.CFrame
    shield.Transparency = 1
    shield.CanCollide = true
    shield.CanTouch = false
    shield.CanQuery = true
    shield.Massless = true
    shield.CastShadow = false
    shield.Anchored = false
    shield.Parent = ctx.character
    local weld = Instance.new("WeldConstraint")
    weld.Name = "GR33D_ShellWeld"
    weld.Part0, weld.Part1 = ctx.root, shield
    weld.Parent = shield
    S.ui.shield = shield
    local field = Instance.new("ForceField")
    field.Name, field.Visible = "GR33D_LocalField", false
    field.Parent = ctx.character
    S.ui.forceField = field
end

-- Temporarily release collision/anchor leases while preserving the ON toggle.
-- A local AlignPosition holds the actual unanchored rig at the escape position,
-- rather than continuing to pin body limbs to the original lock point.
function H.stopEscapeHold()
    local hold = S.escapeHold
    S.escapeHold = nil
    if hold then
        if hold.position then hold.position:Destroy() end
        if hold.orientation then hold.orientation:Destroy() end
        if hold.attachment then hold.attachment:Destroy() end
    end
end

function H.suspendLock()
    if not S.flags.lock then return end
    if not S.lockSuspended then
        H.release("lock")
        H.destroyShield() -- local solid shell would fight movement while teleporting
        S.lockSuspended = true
    end
end

function H.ensureEscapeHold(target)
    if not H.ready() then return false end
    local root = S.context.root
    local hold = S.escapeHold
    if not hold or not hold.attachment or hold.attachment.Parent ~= root then
        H.stopEscapeHold()
        local attachment, position, orientation
        local ok, message = pcall(function()
            attachment = Instance.new("Attachment")
            attachment.Name, attachment.Parent = "GR33D_EscapeAttachment", root
            position = Instance.new("AlignPosition")
            position.Name = "GR33D_EscapePosition"
            position.Mode = Enum.PositionAlignmentMode.OneAttachment
            position.Attachment0 = attachment
            position.ApplyAtCenterOfMass = true
            position.MaxForce = math.max(160000, root.AssemblyMass * (Workspace.Gravity * 18 + 1200))
            position.MaxVelocity = Config.EscapeHoldMaxVelocity
            position.Responsiveness = Config.EscapeHoldResponsiveness
            position.Position = target.Position
            position.Parent = root
            orientation = Instance.new("AlignOrientation")
            orientation.Name = "GR33D_EscapeOrientation"
            orientation.Mode = Enum.OrientationAlignmentMode.OneAttachment
            orientation.Attachment0 = attachment
            orientation.MaxTorque = math.max(120000, root.AssemblyMass * 150000)
            orientation.MaxAngularVelocity = 45
            orientation.Responsiveness = 85
            orientation.CFrame = target - target.Position
            orientation.Parent = root
        end)
        if not ok then
            if orientation then orientation:Destroy() end
            if position then position:Destroy() end
            if attachment then attachment:Destroy() end
            H.report("Escape stabilization", message)
            return false
        end
        hold = {attachment = attachment, position = position, orientation = orientation}
        S.escapeHold = hold
    end
    hold.position.Position = target.Position
    hold.orientation.CFrame = target - target.Position
    return true
end

function H.beginEscapeMove(target, mode, home, reason)
    if not S.flags.lock or not H.ready() then return false end
    local prior = S.escape
    local wasSuspended = S.lockSuspended
    local oldTarget = S.lockTarget
    H.suspendLock() -- unanchor BEFORE PivotTo; this is the key v7.4 fix
    if not H.moveRoot(target) then
        if not wasSuspended then
            S.lockSuspended = false
            H.applyLock()
        end
        return false
    end
    H.zeroRigVelocity()
    local ok, held = pcall(H.ensureEscapeHold, target)
    if not ok or held ~= true then
        H.notice("Escape hold unavailable; restoring previous lock", 5)
        H.stopEscapeHold()
        S.lockTarget = oldTarget
        if oldTarget then H.moveRoot(oldTarget) end
        if wasSuspended and oldTarget then
            local restored, result = pcall(H.ensureEscapeHold, oldTarget)
            if restored and result then
                S.lockSuspended = true
                return false
            end
        end
        -- If even the previous hold cannot be recovered, do not strand the
        -- character in a temporary unlocked state with no active stabilizer.
        S.escape, S.lockSuspended = nil, false
        H.applyLock()
        return false
    end
    local now = os.clock()
    S.lockTarget = target
    S.escape = {home = home, mode = mode, reason = reason,
        earliestReturn = now + (mode == "emergency" and Config.EmergencyMinDuration
            or mode == "manual" and Config.ManualRearmDelay or Config.EscapeMinDuration),
        safeSince = nil,
        chaseCount = prior and prior.chaseCount or 0,
        lastThreatAt = prior and prior.lastThreatAt or -math.huge}
    return true
end

function H.rearmLockAt(target)
    if not H.ready() or not S.flags.lock then return false end
    H.stopEscapeHold()
    if not H.moveRoot(target) then
        if S.lockTarget then H.ensureEscapeHold(S.lockTarget) end
        return false
    end
    S.lockTarget, S.escape, S.lockSuspended = target, nil, false
    H.applyLock()
    return true
end

function H.updateEscapeHold()
    if not S.escape or not H.ready() then return end
    H.suspendLock()
    local target = S.lockTarget
    local ok, held = pcall(H.ensureEscapeHold, target)
    if not ok or held ~= true then
        H.notice("Escape stabilizer unavailable; check Output", 4)
        return
    end
    -- If another local controller displaced the rig, try one correction.
    -- Server-owned displacement cannot be overridden with a LocalScript.
    local now = os.clock()
    if (S.context.root.Position - target.Position).Magnitude > Config.EscapeDriftCorrection
        and now - (S.lastEscapeDriftCorrection or -math.huge) > 0.3 then
        S.lastEscapeDriftCorrection = now
        H.moveRoot(target)
        H.zeroRigVelocity()
    end
end

function H.stopLock(tryLanding)
    -- Turning OFF must cancel the temporary hold before dropping the player.
    local landing = nil
    if tryLanding and S.escape and H.ready() then
        landing = H.safeDestination(S.escape.home) and S.escape.home
            or H.landingAt(S.context.root.CFrame)
        if not landing then H.notice("Lock off; no safe landing found", 5) end
    end
    H.stopEscapeHold()
    if landing then H.moveRoot(landing) end
    S.lockSuspended = false
    S.lockTarget, S.escape = nil, nil
    H.destroyShield()
    H.release("lock")
end

local BODY_PART_NAMES = {
    HumanoidRootPart = true, Head = true, Torso = true,
    UpperTorso = true, LowerTorso = true,
    LeftUpperArm = true, LeftLowerArm = true, LeftHand = true,
    RightUpperArm = true, RightLowerArm = true, RightHand = true,
    LeftUpperLeg = true, LeftLowerLeg = true, LeftFoot = true,
    RightUpperLeg = true, RightLowerLeg = true, RightFoot = true,
    ["Left Arm"] = true, ["Right Arm"] = true,
    ["Left Leg"] = true, ["Right Leg"] = true,
}

local FLING_STATES = {
    Enum.HumanoidStateType.Ragdoll,
    Enum.HumanoidStateType.FallingDown,
    Enum.HumanoidStateType.Physics,
    Enum.HumanoidStateType.PlatformStanding,
}

function H.zeroRigVelocity()
    if not S.context then return end
    for _, part in ipairs(S.context.character:GetChildren()) do
        if part:IsA("BasePart") and BODY_PART_NAMES[part.Name] then
            pcall(function()
                part.AssemblyLinearVelocity = Vector3.zero
                part.AssemblyAngularVelocity = Vector3.zero
            end)
        end
    end
end

function H.recoverRagdoll()
    if not H.ready() or not (S.flags.fling or S.flags.lock) then return end
    if (S.flying or S.hovering) and not S.flags.lock then return end
    local humanoid = S.context.humanoid
    if S.flags.fling then
        for _, state in ipairs(FLING_STATES) do H.claimState(humanoid, state, false, "fling") end
    end
    if S.flags.lock then
        for _, state in ipairs(FLING_STATES) do H.claimState(humanoid, state, false, "lock") end
    end
    if humanoid.PlatformStand then humanoid.PlatformStand = false end
    if humanoid.Sit then humanoid.Sit = false end
    local state = humanoid:GetState()
    if state == Enum.HumanoidStateType.Ragdoll
        or state == Enum.HumanoidStateType.FallingDown
        or state == Enum.HumanoidStateType.Physics
        or state == Enum.HumanoidStateType.PlatformStanding then
        humanoid:ChangeState(Enum.HumanoidStateType.GettingUp)
    end
end

function H.applyLock()
    if not S.flags.lock or not H.ready() then return end
    -- During dodge/emergency/manual teleport, NEVER re-anchor the rig.
    if S.lockSuspended or S.escape then return end
    local ctx = S.context
    if not S.lockTarget then S.lockTarget = ctx.root.CFrame end
    S.lockSuspended = false
    -- Anchor R6/R15 body parts as in the reference; preserve the ORIGINAL property
    -- for each part so turning Lock OFF can restore it without damaging the rig.
    for _, part in ipairs(ctx.character:GetChildren()) do
        if part:IsA("BasePart") and BODY_PART_NAMES[part.Name] then
            H.claim(part, "Anchored", true, "lock")
            -- Prevent our own locked limbs from colliding with the shell.
            H.claim(part, "CanCollide", false, "lock")
            if not H.finiteVector(part.AssemblyLinearVelocity)
                or part.AssemblyLinearVelocity.Magnitude > 0 then
                part.AssemblyLinearVelocity = Vector3.zero
            end
            if not H.finiteVector(part.AssemblyAngularVelocity)
                or part.AssemblyAngularVelocity.Magnitude > 0 then
                part.AssemblyAngularVelocity = Vector3.zero
            end
        end
    end
    if (ctx.root.Position - S.lockTarget.Position).Magnitude > 0.1
        or ctx.root.CFrame.LookVector:Dot(S.lockTarget.LookVector) < 0.999 then
        H.moveRoot(S.lockTarget)
    end
    ctx.humanoid:Move(Vector3.zero, false)
    -- A collision sphere helps on the locally simulated side. Roblox's server
    -- must also create/authorize a shield to block server-owned objects.
    H.ensureShield()
end

function H.updateEscape()
    if not S.flags.lock or not H.ready() then return end
    local now = os.clock()
    local position = S.context.root.Position
    local home = S.escape and S.escape.home or S.lockTarget
    local severeAtHome = home and H.findSevereHazardNear(home.Position)
    if severeAtHome then
        S.lastSevereThreatAt, S.homeThreatAt = now, now
        if not S.escape or S.escape.mode == "dodge" then
            if H.emergencyEscape(severeAtHome.name .. " abnormal movement", severeAtHome) then
                return
            end
        end
    end
    if S.escape then H.updateEscapeHold() end
    if home and S.escape and S.escape.mode == "emergency"
        and H.nearHazard(home.Position, Config.EmergencyGuardRadius) then
        S.homeThreatAt = now
    end

    local threat = H.findHazard(position)
    local pursuing = S.escape and (H.findSevereHazardNear(position)
        or threat or H.nearHazard(position, Config.ChaseGuardRadius))
    if pursuing then
        S.lastChaseThreatAt = now
        S.escape.lastThreatAt = now
        S.escape.safeSince = nil
        if now - S.lastChaseRelocate >= Config.ChaseRelocateCooldown then
            if S.escape.mode == "emergency" then
                H.emergencyEscape("pursuing " .. pursuing.name, pursuing)
            elseif S.escape.mode == "dodge" and H.isSevereThreat(pursuing, position) then
                H.emergencyEscape("pursuing " .. pursuing.name, pursuing)
            elseif S.escape.mode == "dodge" then
                -- A continuing projectile near the new spot: evade diagonally,
                -- and escalate if no safe small-shift is available.
                local away = position - pursuing.bounds.Position
                away = Vector3.new(away.X, 0, away.Z)
                away = away.Magnitude > 0.1 and away.Unit or Vector3.new(1, 0, 0)
                local side = Vector3.new(-away.Z, 0, away.X)
                local target = S.context.root.CFrame + away * 18 + side * 15 + Vector3.new(0, 7, 0)
                if H.safeDestination(target) and H.clearEscapePath(target) then
                    H.beginEscapeMove(target, "dodge", S.escape.home, pursuing.name)
                    S.lastChaseRelocate = now
                else
                    H.emergencyEscape("repeated " .. pursuing.name, pursuing)
                end
            end
        end
    elseif threat then
        S.recentHazard = threat
    end

    if not S.escape and threat then
        S.recentHazard = threat
        if H.isSevereThreat(threat, position) or H.recordContact(threat) then
            S.lastSevereThreatAt = now
            H.emergencyEscape(threat.name .. " abnormal contact", threat)
        else
            H.escape(threat, threat.name)
        end
    end
    if not S.escape then return end
    if now < S.escape.earliestReturn then return end
    -- Cannot safely re-arm while a hazard is still following the airborne rig.
    if now - math.max(S.lastChaseThreatAt, S.escape.lastThreatAt or -math.huge)
        < Config.ChaseQuietWindow then
        S.escape.safeSince = nil
        return
    end
    local safeHome = H.safeDestination(S.escape.home)
    if S.escape.mode == "emergency" then
        if H.nearHazard(S.escape.home.Position, Config.EmergencyGuardRadius) then
            S.homeThreatAt = now
            safeHome = false
        end
        if now - S.homeThreatAt < Config.EmergencyHomeDangerHold
            or now - S.lastDamageAt < Config.EmergencyReturnQuiet then
            safeHome = false
        end
    end
    if not safeHome then
        S.escape.safeSince = nil
        return
    end
    S.escape.safeSince = S.escape.safeSince or now
    local grace = S.escape.mode == "emergency" and Config.EmergencyReturnQuiet
        or S.escape.mode == "manual" and Config.ManualRearmDelay
        or Config.ReturnSafeGrace
    if now - S.escape.safeSince < grace then return end
    local destination = S.escape.home
    if H.rearmLockAt(destination) then
        H.notice("Returned to safe position; lock restored", 3)
    end
end

function H.noclip(owner, enabled)
    if not H.ready() then return end
    if not enabled then H.release(owner); return end
    for part in pairs(S.context.parts) do
        if part:IsDescendantOf(S.context.character) then H.claim(part, "CanCollide", false, owner) end
    end
end

function H.stopMover()
    if S.mover then
        S.mover.velocity:Destroy()
        S.mover.attachment:Destroy()
        S.mover = nil
    end
    H.release("mover")
    S.flying, S.hovering, S.hoverY = false, false, nil
end

function H.startMover(kind)
    if not H.ready() or S.flags.lock then return false end
    H.release("idle")
    S.idleTarget, S.idleSince, S.idleSample = nil, nil, nil
    S.idleBlockedUntil = os.clock() + Config.IdleInputGrace
    if S.context.root.Anchored then return false end
    H.stopMover()
    local ctx = S.context
    local attachment = Instance.new("Attachment")
    attachment.Name = "GR33D_MovementAttachment"
    attachment.Parent = ctx.root
    local velocity = Instance.new("LinearVelocity")
    velocity.Name = "GR33D_MovementVelocity"
    velocity.Attachment0 = attachment
    velocity.RelativeTo = Enum.ActuatorRelativeTo.World
    velocity.VelocityConstraintMode = Enum.VelocityConstraintMode.Vector
    velocity.ForceLimitsEnabled = true
    velocity.MaxForce = kind == "flight" and Config.FlightMaxForce
        or math.max(10000, ctx.root.AssemblyMass * (Workspace.Gravity * 4 + 1000))
    velocity.VectorVelocity = Vector3.zero
    velocity.Parent = ctx.root
    S.mover = {attachment = attachment, velocity = velocity, kind = kind}
    -- Reference flight leaves the Humanoid's state and normal movement control intact.
    if kind == "hover" then
        H.claim(ctx.humanoid, "PlatformStand", true, "mover")
        H.claim(ctx.humanoid, "AutoRotate", false, "mover")
    else
        ctx.humanoid:ChangeState(Enum.HumanoidStateType.Freefall)
    end
    S.flying, S.hovering = kind == "flight", kind == "hover"
    if S.hovering then S.hoverY = ctx.root.Position.Y + Config.NPCLiftHeight; H.noclip("mover", true) end
    return true
end

function H.endHover(land)
    if not S.hovering then return end
    local target = land and H.ready() and H.landingAt(S.context.root.CFrame) or nil
    H.stopMover()
    if target then H.moveRoot(target) end
    S.lastDrop = os.clock()
end

function H.hostileNPC(model)
    if not model.Parent or H.ignored(model) then return false end
    local humanoid = model:FindFirstChildOfClass("Humanoid")
    if not humanoid or humanoid.Health <= 0 then return false end
    if H.hasTag(model, Config.FriendlyTags) or model:GetAttribute("Friendly") == true then return false end
    if H.hasTag(model, Config.HostileTags) then return true end
    for _, attribute in ipairs({"Hostile", "IsHostile", "IsEnemy", "Attacking", "Aggro"}) do
        if model:GetAttribute(attribute) == true then return true end
    end
    local hostileStates = {attack = true, attacking = true, chase = true, chasing = true,
        aggro = true, aggressive = true, hostile = true, combat = true, hunting = true}
    for _, attribute in ipairs({"AIState", "State"}) do
        local value = model:GetAttribute(attribute)
        if type(value) == "string" and hostileStates[value:lower()] then return true end
    end
    local ctx = S.context
    if ctx then
        for _, value in ipairs(model:GetDescendants()) do
            if value:IsA("ObjectValue") then
                local name = value.Name:lower()
                if name:find("target", 1, true) or name:find("enemy", 1, true) then
                    if value.Value == player or value.Value == ctx.character
                        or value.Value == ctx.humanoid or value.Value == ctx.root then return true end
                end
            end
        end
    end
    if Config.UseNPCNameHints then
        local tokens = H.tokens(model.Name)
        for _, name in ipairs({"enemy", "monster", "zombie", "killer", "hunter", "chaser", "boss"}) do
            if tokens[name] then return true end
        end
    end
    return false
end

function H.updateNPCs()
    if not S.flags.monster or not H.ready() or S.flags.lock or S.flying then return end
    local position = S.context.root.Position
    local closest3D, closestXZ = math.huge, math.huge
    for model in pairs(S.npcs) do
        if H.hostileNPC(model) then
            local root = model:FindFirstChild("HumanoidRootPart") or model:FindFirstChildWhichIsA("BasePart", true)
            if root and root:IsA("BasePart") then
                local offset = position - root.Position
                closest3D = math.min(closest3D, offset.Magnitude)
                if math.abs(offset.Y) < 60 then
                    closestXZ = math.min(closestXZ, Vector3.new(offset.X, 0, offset.Z).Magnitude)
                end
            end
        end
    end
    if not S.hovering then
        if closest3D <= Config.NPCStartDistance and os.clock() - S.lastDrop >= Config.NPCLiftCooldown then
            local target = S.context.root.CFrame + Vector3.new(0, Config.NPCLiftHeight, 0)
            local params = RaycastParams.new()
            params.FilterType = Enum.RaycastFilterType.Exclude
            params.FilterDescendantsInstances = {S.context.character}
            params.RespectCanCollide = true
            local ceiling = Workspace:Raycast(position, Vector3.new(0, Config.NPCLiftHeight + 4, 0), params)
            if not ceiling and H.safeDestination(target) then H.startMover("hover") end
        end
    elseif closestXZ >= Config.NPCStopDistance then
        H.endHover(true)
    end
end

function H.inputMoveVector()
    if not S.controls then return nil end
    local ok, vector = pcall(function() return S.controls:GetMoveVector() end)
    if ok and typeof(vector) == "Vector3" and H.finiteVector(vector) then return vector end
    return nil
end

function H.updateMover()
    if not S.mover or not H.ready() then return end
    local ctx = S.context
    if ctx.root.Anchored then H.stopMover(); H.notice("Movement stopped: root is anchored", 3); return end
    local direction = ctx.humanoid.MoveDirection
    if S.flying then
        -- Same steering as the reference v6.1: Camera X/Z movement is normalized;
        -- looking up/down permits climbing/descending without separate keys.
        if ctx.humanoid.FloorMaterial ~= Enum.Material.Air then
            H.stopMover()
            return
        end
        local camera = Workspace.CurrentCamera
        local input = H.inputMoveVector()
        if camera and input then
            direction = camera.CFrame.RightVector * input.X - camera.CFrame.LookVector * input.Z
        end
        direction = direction.Magnitude > 0.001 and direction.Unit * Config.FlightSpeed or Vector3.zero
    else
        H.claim(ctx.humanoid, "PlatformStand", true, "mover")
        H.noclip("mover", true)
        local speed = S.flags.run and S.runSpeed or ctx.humanoid.WalkSpeed
        direction = Vector3.new(direction.X * speed,
            math.clamp((S.hoverY - ctx.root.Position.Y) * 8, -20, 35), direction.Z * speed)
    end
    S.mover.velocity.VectorVelocity = direction
end

-- Shield against locally simulated flings without freezing ordinary movement.
-- While truly idle on stable ground, anchor only the root assembly (not every
-- limb: anchoring all limbs splits physical assemblies and breaks animations).
function H.releaseFlingHold(grace)
    if S.flingHoldTarget then
        H.release("fling_hold")
        S.flingHoldTarget = nil
    end
    if grace and grace > 0 then
        S.flingPauseUntil = math.max(S.flingPauseUntil, os.clock() + grace)
    end
end

function H.flingMovementInput(ctx)
    local now = os.clock()
    local input = H.inputMoveVector()
    local synthetic = S.afkPulseUntil ~= nil or now < S.afkPostPulseUntil
    return now < S.flingInputUntil or (input and input.Magnitude > 0.06)
        or (not synthetic and ctx.humanoid.MoveDirection.Magnitude > 0.06)
        or ctx.humanoid.Jump == true
end

function H.flingCanOperate(ctx)
    -- Important: never fight Lock On's temporary escape, voluntary flight,
    -- hover, teleporting, or another feature's movement constraint.
    return S.flags.fling and not S.flags.lock and not S.escape
        and not S.mover and not S.flying and not S.hovering
        and ctx.character:GetAttribute("ProtectionAllowHighVelocity") ~= true
end

function H.flingRestore(reason, fallback)
    local ctx = S.context
    if not ctx or S.flingRecovering then return end
    S.flingRecovering = true
    -- Velocity neutralization happens on every detected impact, not just once
    -- per cooldown. The cooldown only limits notification spam.
    H.zeroRigVelocity()
    H.recoverRagdoll()
    local now = os.clock()
    local target = fallback
    if not target and S.flingLastStableCF
        and now - S.flingLastStableAt <= Config.FlingGroundSnapshotAge then
        target = S.flingLastStableCF
    end
    if target and H.finiteVector(target.Position)
        and (ctx.root.Position - target.Position).Magnitude < 38
        and H.safeDestination(target) then
        H.moveRoot(target)
    end
    S.flingSnapshot = nil
    if now - S.lastFlingRecovery >= 1 then
        S.lastFlingRecovery = now
        H.notice("Anti-Fling: " .. tostring(reason) .. " corrected locally", 2)
    end
    S.flingRecovering = false
end

function H.updateFling(phase, deltaTime)
    if not H.ready() or not (S.flags.fling or S.flags.lock) then
        H.releaseFlingHold()
        return
    end
    local ctx = S.context
    local humanoid, root = ctx.humanoid, ctx.root
    -- Capture the forced state BEFORE recovery changes it to GettingUp;
    -- otherwise the high-priority position correction can be skipped.
    local stateBefore = humanoid:GetState()
    local wasForced = stateBefore == Enum.HumanoidStateType.Ragdoll
        or stateBefore == Enum.HumanoidStateType.Physics
        or stateBefore == Enum.HumanoidStateType.FallingDown
        or stateBefore == Enum.HumanoidStateType.PlatformStanding
        or humanoid.PlatformStand or humanoid.Sit
    H.recoverRagdoll()
    if not H.flingCanOperate(ctx) then
        H.releaseFlingHold()
        S.flingSnapshot = nil
        return
    end

    local now = os.clock()
    local state = humanoid:GetState()
    local forcedState = wasForced or state == Enum.HumanoidStateType.Ragdoll
        or state == Enum.HumanoidStateType.Physics
        or state == Enum.HumanoidStateType.FallingDown
        or state == Enum.HumanoidStateType.PlatformStanding
        or humanoid.PlatformStand or humanoid.Sit
    local hasInput = H.flingMovementInput(ctx)
    local grounded, movingPlatform = H.idleGroundInfo(ctx)
    local canHold = now >= S.flingPauseUntil and not hasInput
        and not forcedState and grounded and not movingPlatform
        and not S.flags.noclip and not S.flying and not S.hovering
        and state ~= Enum.HumanoidStateType.Jumping
        and state ~= Enum.HumanoidStateType.Climbing
        and state ~= Enum.HumanoidStateType.Swimming
        and state ~= Enum.HumanoidStateType.Seated
        and not (S.flags.afk and S.idleTarget) -- AFK Streak owns that hold

    if S.flingHoldTarget then
        if not canHold then
            H.releaseFlingHold(hasInput and Config.FlingInputReleaseGrace or nil)
        else
            -- Keep the same exact place even during light repeated contact.
            H.claim(root, "Anchored", true, "fling_hold")
            if (root.Position - S.flingHoldTarget.Position).Magnitude > 0.08 then
                H.moveRoot(S.flingHoldTarget)
            end
            H.zeroRigVelocity()
            S.flingSnapshot = nil
            return
        end
    end

    local linear, angular = root.AssemblyLinearVelocity, root.AssemblyAngularVelocity
    local finite = H.finiteVector(linear) and H.finiteVector(angular)
    local limit = math.max(Config.FlingLinearThreshold,
        S.flags.run and S.runSpeed + 30 or 0,
        humanoid.WalkSpeed + 30)
    local bad = not finite
    if finite then
        if linear.Magnitude > limit or angular.Magnitude > Config.FlingAngularThreshold then
            bad = true
        end
    end
    -- Check parts as well as the root: constraints can transfer a fling through
    -- the limbs while the root still reports an ordinary velocity.
    if not bad then
        for _, part in ipairs(ctx.character:GetChildren()) do
            if part:IsA("BasePart") and BODY_PART_NAMES[part.Name] then
                local v, w = part.AssemblyLinearVelocity, part.AssemblyAngularVelocity
                if not H.finiteVector(v) or not H.finiteVector(w)
                    or v.Magnitude > limit or w.Magnitude > Config.FlingAngularThreshold then
                    bad = true
                    break
                end
            end
        end
    end
    local previous = S.flingSnapshot
    if not bad and phase == "post" and previous and finite then
        local dt = math.clamp(deltaTime or 1 / 60, 1 / 240, 0.12)
        local velocityJump = (linear - previous.velocity).Magnitude
        local positionJump = (root.Position - previous.cframe.Position).Magnitude
        local expectedMovement = (S.flags.run and S.runSpeed or humanoid.WalkSpeed) * dt
        -- Ignore large acceleration if controlled movement is already ongoing,
        -- but never ignore extreme spins / impulses in avatar body parts.
        if not hasInput and not humanoid.Jump and velocityJump > Config.FlingImpulseDeltaThreshold then
            bad = true
        elseif positionJump > math.max(Config.FlingFrameDistance, expectedMovement + 3.5)
            and not hasInput then
            bad = true
        end
    end
    if bad or forcedState then
        H.flingRestore(forcedState and "ragdoll" or "unexpected impulse",
            previous and previous.cframe or nil)
        return
    end

    if phase == "pre" then
        -- The most recent pre-physics pose is our safest non-lagging recovery
        -- point. Do not copy an already violent velocity into the snapshot.
        S.flingSnapshot = {cframe = root.CFrame, velocity = linear}
        if grounded and not movingPlatform then
            S.flingLastStableCF = root.CFrame
            S.flingLastStableAt = now
        end
        -- Idle hold engages immediately once ground and low movement are
        -- verified. Releasing it happens before the next physics step.
        if canHold and not root.Anchored and finite then
            local flat = Vector3.new(linear.X, 0, linear.Z).Magnitude
            if flat <= Config.FlingStationaryMaxSpeed
                and math.abs(linear.Y) <= Config.FlingStationaryMaxVertical then
                S.flingHoldTarget = root.CFrame
                H.claim(root, "Anchored", true, "fling_hold")
                H.zeroRigVelocity()
                S.flingSnapshot = nil
            end
        end
    elseif phase == "post" then
        S.flingSnapshot = nil
    end
end

-- AFK Streak combines stable position hold with the original 18-second local
-- movement pulse. The single AFK switch owns both behaviors. Never anchor
-- when moving, jumping, flying, locked, teleporting, or on a moving platform.
function H.releaseIdle(grace, keepTarget)
    H.release("idle")
    if not keepTarget then
        S.idleTarget, S.idleSince, S.idleSample = nil, nil, nil
    end
    if grace then
        S.idleBlockedUntil = os.clock() + Config.IdleInputGrace
    end
end

function H.idleGroundInfo(ctx)
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Exclude
    params.FilterDescendantsInstances = {ctx.character}
    params.RespectCanCollide = true
    local height = math.max(3.5, ctx.humanoid.HipHeight + ctx.root.Size.Y * 0.5 + 1.5)
    local hit = Workspace:Raycast(ctx.root.Position, Vector3.new(0, -height, 0), params)
    if not hit or hit.Normal.Y < 0.45 then return false, false end
    if hit.Instance:IsA("BasePart") then
        local velocity = hit.Instance.AssemblyLinearVelocity
        if not H.finiteVector(velocity) then return false, false end
        return true, velocity.Magnitude > Config.IdleMovingPlatformSpeed
    end
    return true, false -- Terrain is stationary.
end

function H.updateAFK()
    if not S.flags.afk or not H.ready() then
        H.releaseIdle(false)
        S.afkPulseUntil = nil
        return
    end

    local ctx = S.context
    local now = os.clock()
    local humanoid = ctx.humanoid
    local state = humanoid:GetState()
    local input = H.inputMoveVector()
    -- During and briefly after our own pulse, MoveDirection reflects the
    -- synthetic Move() call, not necessarily actual player input.
    local synthetic = S.afkPulseUntil ~= nil or now < S.afkPostPulseUntil
    local playerMoving = (input and input.Magnitude > 0.06)
        or (not synthetic and humanoid.MoveDirection.Magnitude > 0.06)
    local grounded, movingPlatform = H.idleGroundInfo(ctx)
    local unavailable = playerMoving or humanoid.Jump or humanoid.Sit
        or humanoid.PlatformStand or state == Enum.HumanoidStateType.Jumping
        or state == Enum.HumanoidStateType.Ragdoll
        or state == Enum.HumanoidStateType.FallingDown
        or state == Enum.HumanoidStateType.PlatformStanding
        or (state == Enum.HumanoidStateType.Freefall and not grounded)
        or not grounded or movingPlatform
        or S.flags.lock or S.escape ~= nil or S.mover ~= nil
        or S.flying or S.hovering

    if unavailable then
        if S.afkPulseUntil then humanoid:Move(Vector3.zero, false) end
        S.afkPulseUntil = nil
        S.afkPostPulseUntil = -math.huge
        S.afkLastPulse = now
        H.releaseIdle(playerMoving or humanoid.Jump)
        return
    end

    -- Keep the exact held CFrame during the brief activity pulse. Previously
    -- resetting the target here caused intermittent drift and failed re-arming.
    if S.afkPulseUntil then
        if now >= S.afkPulseUntil then
            humanoid:Move(Vector3.zero, false)
            S.afkPulseUntil = nil
            S.afkPostPulseUntil = now + Config.IdleInputGrace
            H.releaseIdle(true, true)
        else
            humanoid:Move(Vector3.new(0, 0, Config.AFKMovementPulse), false)
        end
        return
    end

    if S.idleTarget and now - S.afkLastPulse >= Config.AFKPulsingInterval then
        H.releaseIdle(false, true) -- unanchor temporarily, keep original position
        S.afkLastPulse = now
        S.afkPulseUntil = now + Config.AFKPulsingDuration
        humanoid:Move(Vector3.new(0, 0, Config.AFKMovementPulse), false)
        return
    end

    if now < S.idleBlockedUntil then
        H.release("idle") -- cooldown must not erase the pre-pulse destination
        return
    end

    if not S.idleTarget then
        local velocity = ctx.root.AssemblyLinearVelocity
        if not H.finiteVector(velocity) then H.releaseIdle(false); return end
        local planarSpeed = Vector3.new(velocity.X, 0, velocity.Z).Magnitude
        if planarSpeed > Config.IdleMaxHorizontalSpeed then
            S.idleSample, S.idleSince = nil, nil
            return
        end
        local position = ctx.root.Position
        if not S.idleSample or (position - S.idleSample).Magnitude > Config.IdleSettleRadius then
            S.idleSample, S.idleSince = position, now
            return
        end
        if now - (S.idleSince or now) < Config.IdleSettleTime then return end
        S.idleTarget = ctx.root.CFrame
        S.afkLastPulse = now -- start 18 seconds after successfully becoming idle
    end

    -- Anchor only while genuinely idle. Do not keep resetting the target if
    -- minor physics jitter or the AFK pulse has displaced the player slightly.
    H.claim(ctx.root, "Anchored", true, "idle")
    if (ctx.root.Position - S.idleTarget.Position).Magnitude > 0.35 then
        H.moveRoot(S.idleTarget)
    end
    H.zeroVelocity()
end

function H.applyCollision()
    if S.flags.noclip then H.noclip("noclip", true) end
end

function H.teleport(target, label)
    if not H.ready() or typeof(target) ~= "CFrame" or not H.finiteVector(target.Position) then
        H.notice("Teleport failed: character or destination unavailable", 4)
        return false
    end
    local chosen
    for _, lift in ipairs(Config.ManualTeleportLiftSteps) do
        local candidate = target + Vector3.new(0, lift, 0)
        if H.safeDestination(candidate) then chosen = candidate; break end
    end
    if not chosen then
        H.notice("Teleport blocked: destination obstructed or unsafe", 5)
        return false
    end
    H.stopMover()
    H.releaseIdle(true)
    H.releaseFlingHold(0.9)
    S.flingSnapshot = nil
    S.afkPulseUntil = nil
    S.afkPostPulseUntil = -math.huge
    S.afkLastPulse = os.clock()
    local oldLock, oldEscape = S.lockTarget, S.escape
    if S.flags.lock then
        if not H.beginEscapeMove(chosen, "manual", chosen, label) then
            S.lockTarget, S.escape = oldLock, oldEscape
            H.notice("Teleport move failed", 4)
            return false
        end
        S.damageEvents, S.impactEntries = {}, {}
    elseif not H.moveRoot(chosen) then
        H.notice("Teleport move failed", 4)
        return false
    end
    S.safeRoot, S.safeTime = chosen, os.clock()
    H.notice("TP to " .. tostring(label or "destination") .. " requested", 3)
    local ctx = S.context
    task.delay(0.75, function()
        if S.alive and ctx == S.context and ctx.root.Parent
            and (ctx.root.Position - chosen.Position).Magnitude > 16 then
            H.notice("Server may be correcting client-only teleport", 5)
        end
    end)
    return true
end

local function eligibleSpawn(inst)
    return inst and inst:IsA("SpawnLocation") and inst.Enabled
        and inst:IsDescendantOf(Workspace)
        and (inst.Neutral or inst.TeamColor == player.TeamColor)
end

function H.findJoinSpawn()
    -- RespawnLocation is an explicit server-selected spawn where available.
    if eligibleSpawn(player.RespawnLocation) then return player.RespawnLocation end
    if eligibleSpawn(S.joinSpawn) then return S.joinSpawn end
    -- For maps without RespawnLocation, use the spawn nearest to the position
    -- first observed when this panel was started, not the player's current spot.
    local origin = S.entrySpawn and S.entrySpawn.Position
        or (H.ready() and S.context.root.Position)
    if not origin then return nil end
    local closest, bestDistance
    for _, inst in ipairs(Workspace:GetDescendants()) do
        if eligibleSpawn(inst) then
            local distance = (inst.Position - origin).Magnitude
            if not bestDistance or distance < bestDistance then
                closest, bestDistance = inst, distance
            end
        end
    end
    return closest
end

function H.spawnRootTarget(spawn)
    if not H.ready() or not eligibleSpawn(spawn) then return nil end
    local root = S.context.root
    local humanoid = S.context.humanoid
    local height = spawn.Size.Y * 0.5 + humanoid.HipHeight + root.Size.Y * 0.5 + 0.8
    return CFrame.new(spawn.Position + spawn.CFrame.UpVector * height)
        * (root.CFrame - root.Position)
end

function H.updateSwitches()
    for key, switch in pairs(S.switches) do
        local enabled = S.flags[key]
        switch.button.BackgroundColor3 = enabled and Color3.fromRGB(42, 176, 117) or Color3.fromRGB(48, 50, 61)
        switch.knob.Position = enabled and UDim2.new(1, -18, 0.5, -7) or UDim2.new(0, 4, 0.5, -7)
    end
end

function H.setToggle(key, enabled)
    if not S.alive or S.flags[key] == nil then return false end
    enabled = enabled == true
    if S.flags[key] == enabled then return true end
    if enabled and not H.ready() then H.notice("Wait for a living character", 4); return false end
    if enabled then
        -- AFK Streak is independent of flight and Lock On; it pauses its hold
        -- during either feature and resumes when normal idle movement returns.
        local conflicts = {lock = {"flight", "monster"}, flight = {"lock", "monster"},
            monster = {"lock", "flight"}}
        for _, other in ipairs(conflicts[key] or {}) do H.setToggle(other, false) end
    end
    S.flags[key] = enabled
    local ok, message = pcall(function()
        if key == "lock" then
            if enabled then
                H.releaseIdle(true)
                H.stopEscapeHold()
                S.lockSuspended = false
                S.lockTarget, S.escape, S.lastEscape = S.context.root.CFrame, nil, -math.huge
                S.lastChaseRelocate, S.lastChaseThreatAt = -math.huge, -math.huge
                H.applyLock()
            else H.stopLock(true) end
        elseif key == "flight" then
            if not enabled then H.stopMover() end
            if enabled then H.notice("Flight armed: double-tap jump to fly", 4) end
        elseif key == "monster" then
            if not enabled then H.endHover(true) end
        elseif key == "noclip" then
            if enabled then H.noclip("noclip", true) else H.release("noclip") end
        elseif key == "fling" then
            if enabled then
                S.flingSnapshot = nil
                S.flingLastStableCF = S.context.root.CFrame
                S.flingLastStableAt = os.clock()
                S.flingPauseUntil = os.clock() + 0.1
                H.recoverRagdoll()
            else
                H.releaseFlingHold()
                H.release("fling")
                S.flingSnapshot = nil
            end
        elseif key == "afk" then
            if S.afkPulseUntil and H.ready() then
                S.context.humanoid:Move(Vector3.zero, false)
            end
            H.releaseIdle(true)
            S.afkLastPulse, S.afkPulseUntil = os.clock(), nil
            S.afkPostPulseUntil = -math.huge
            if enabled then H.notice("AFK Streak on: hold when idle + periodic pulse", 4) end
        elseif key == "health" and enabled then
            H.notice("Health Recovery is local; server damage may still kill", 5)
        elseif key == "fall" and enabled then
            H.notice("Descent limited; game-defined fall damage may still apply", 4)
        end
    end)
    if not ok then
        S.flags[key] = false
        -- Undo partial activation, not just its visual state.
        if key == "lock" then H.stopLock(false)
        elseif key == "noclip" then H.release("noclip")
        elseif key == "flight" or key == "monster" then H.stopMover()
        elseif key == "fling" then H.releaseFlingHold(); H.release("fling")
        elseif key == "afk" then
            H.releaseIdle(false)
            S.afkPulseUntil = nil
        end
        H.report("Toggle " .. key, message)
    end
    H.updateSwitches()
    return ok
end

function H.cleanupCharacter()
    H.disconnectAll(S.characterConnections)
    H.stopLock(false)
    H.stopMover()
    H.releaseFlingHold()
    H.releaseAll()
    S.context, S.idleTarget, S.idleSince, S.idleSample = nil, nil, nil, nil
    S.idleBlockedUntil = -math.huge
    S.afkLastPulse, S.afkPulseUntil, S.afkPostPulseUntil = os.clock(), nil, -math.huge
    S.safeRoot, S.previousHealth = nil, nil
    S.safeTime, S.healing, S.lastDrop, S.lastJump = -math.huge, false, -math.huge, -math.huge
    S.lastJumpRequest = -math.huge
    S.lastDamageAt, S.lastSevereThreatAt = -math.huge, -math.huge
    S.homeThreatAt, S.lastEmergency = -math.huge, -math.huge
    S.damageEvents, S.impactEntries, S.recentHazard = {}, {}, nil
    S.lastFlingRecovery = -math.huge
    S.flingHoldTarget, S.flingSnapshot, S.flingLastStableCF = nil, nil, nil
    S.flingLastStableAt, S.flingPauseUntil, S.flingInputUntil = -math.huge, -math.huge, -math.huge
    S.flingRecovering = false
    S.lockSuspended, S.escapeHold = false, nil
    S.lastChaseRelocate, S.lastChaseThreatAt = -math.huge, -math.huge
end

function H.beginCharacter(character)
    S.setupToken = S.setupToken + 1
    local token = S.setupToken
    H.cleanupCharacter()
    task.spawn(function()
        local root = character:WaitForChild("HumanoidRootPart", 10)
        local humanoid = character:WaitForChild("Humanoid", 10)
        if not S.alive or token ~= S.setupToken or player.Character ~= character then return end
        if not root or not root:IsA("BasePart") or not humanoid or not humanoid:IsA("Humanoid") then
            H.notice("Character requires a Humanoid and HumanoidRootPart", 6)
            return
        end
        -- CharacterAdded may fire before Roblox parents the model to Workspace.
        local deadline = os.clock() + 10
        while not character:IsDescendantOf(Workspace) do
            if not S.alive or token ~= S.setupToken or player.Character ~= character or os.clock() >= deadline then return end
            task.wait(0.05)
        end
        local ctx = {character = character, root = root, humanoid = humanoid, parts = {}}
        S.context = ctx
        if not S.entrySpawn then
            S.entrySpawn = root.CFrame
            S.joinSpawn = H.findJoinSpawn()
        end
        for _, instance in ipairs(character:GetDescendants()) do
            if instance:IsA("BasePart") then ctx.parts[instance] = true end
        end
        S.previousHealth = humanoid.Health
        H.connect(character.DescendantAdded, function(instance)
            if S.context == ctx and instance:IsA("BasePart") then ctx.parts[instance] = true end
        end, S.characterConnections)
        H.connect(character.DescendantRemoving, function(instance)
            ctx.parts[instance] = nil
            if instance == root or instance == humanoid then
                if S.context == ctx then H.cleanupCharacter(); H.notice("Character components removed", 4) end
            end
        end, S.characterConnections)
        -- Reference v6.1 used property listeners as well as state-disable calls.
        -- Apply on each character (R6 or R15); cleaned on respawn.
        H.connect(humanoid:GetPropertyChangedSignal("PlatformStand"), function()
            if S.context == ctx and (S.flags.fling or S.flags.lock)
                and not S.hovering and not S.flying and humanoid.PlatformStand then
                H.recoverRagdoll()
            end
        end, S.characterConnections)
        H.connect(humanoid:GetPropertyChangedSignal("Sit"), function()
            if S.context == ctx and (S.flags.fling or S.flags.lock)
                and not S.hovering and not S.flying and humanoid.Sit then
                H.recoverRagdoll()
            end
        end, S.characterConnections)
        H.connect(humanoid.StateChanged, function(_, newState)
            if S.context ~= ctx or not (S.flags.fling or S.flags.lock) then return end
            if newState == Enum.HumanoidStateType.Ragdoll
                or newState == Enum.HumanoidStateType.FallingDown
                or newState == Enum.HumanoidStateType.Physics
                or newState == Enum.HumanoidStateType.PlatformStanding then
                H.recoverRagdoll()
            end
        end, S.characterConnections)
        H.recoverRagdoll()
        H.connect(humanoid.HealthChanged, function(health)
            if S.context ~= ctx then return end
            local previous = S.previousHealth or health
            S.previousHealth = health
            if health <= 0 then
                H.cleanupCharacter()
                H.notice("Waiting for respawn", 5)
                return
            end
            if health < previous and not S.healing then
                local now = os.clock()
                local loss = previous - health
                -- The event fires as soon as Roblox reports the HP change; no
                -- periodic scan or nearby-hazard match is required to respond.
                S.lastDamageAt = now
                for i = #S.damageEvents, 1, -1 do
                    if now - S.damageEvents[i] > Config.DamageBurstWindow then
                        table.remove(S.damageEvents, i)
                    end
                end
                table.insert(S.damageEvents, now)
                if S.flags.lock and H.ready() then
                    local threshold = math.max(Config.EmergencyMinDamageThreshold,
                        math.min(Config.EmergencyDamageAbsolute,
                            humanoid.MaxHealth * Config.EmergencyDamageFraction))
                    if loss + 1e-6 >= threshold
                        or (loss >= 0.005 and #S.damageEvents >= Config.DamageBurstCount) then
                        H.emergencyEscape(string.format("HP drop %.2f", loss))
                    end
                end
                if S.flags.health and H.ready() and health < humanoid.MaxHealth then
                    S.healing = true
                    local ok, message = pcall(function() humanoid.Health = humanoid.MaxHealth end)
                    S.healing = false
                    if not ok then H.report("Local health recovery", message) end
                end
            end
        end, S.characterConnections)
        H.connect(humanoid.Died, function()
            if S.context == ctx then H.cleanupCharacter(); H.notice("Waiting for respawn", 5) end
        end, S.characterConnections)
        if S.flags.lock and H.ready() then H.applyLock() end
        if S.flags.noclip and H.ready() then H.noclip("noclip", true) end
        H.notice("Character ready", 2)
    end)
end

function H.shutdown(destroyGui)
    if not S.alive then return end
    S.setupToken = S.setupToken + 1
    -- A living airborne character gets the same safe landing attempt as Lock OFF.
    if S.flags.lock then H.stopLock(true) end
    if S.hovering then H.endHover(true) end
    S.alive = false
    H.cleanupCharacter()
    H.disconnectAll(S.connections)
    if S.cameraConnection then S.cameraConnection:Disconnect(); S.cameraConnection = nil end
    for instance in pairs(S.watchers) do H.removeObject(instance) end
    S.hazards, S.motion, S.npcs, S.pending = {}, {}, {}, {}
    if destroyGui and S.ui.gui then S.ui.gui:Destroy() end
end

-- A new v7 instance asks its predecessor to restore everything before removal.
local previousGui = playerGui:FindFirstChild("DevPanelGui")
if previousGui then
    local shutdown = previousGui:FindFirstChild("GR33D_Shutdown")
    if shutdown and shutdown:IsA("BindableEvent") then
        shutdown:Fire()
    else
        warn("[GR33D] An older panel was found. Restart Play to remove its old loops/hooks.")
    end
    previousGui:Destroy()
end

-- Compact responsive panel. No deprecated Draggable property or manual canvas height.
function H.make(className, properties, parent)
    local instance = Instance.new(className)
    for property, value in pairs(properties or {}) do instance[property] = value end
    instance.Parent = parent
    return instance
end

function H.corner(instance, radius)
    H.make("UICorner", {CornerRadius = UDim.new(0, radius)}, instance)
end

function H.viewport()
    local camera = Workspace.CurrentCamera
    return camera and camera.ViewportSize or Vector2.new(800, 600)
end

function H.layout()
    local viewport = H.viewport()
    local width = math.min(270, math.max(170, viewport.X - 16))
    local height = math.min(535, math.max(150, viewport.Y - 24))
    local panel = S.ui.panel
    local size = S.minimized and Vector2.new(52, 52) or Vector2.new(width, height)
    panel.Size = UDim2.fromOffset(size.X, size.Y)
    local position = panel.Position
    panel.Position = UDim2.fromOffset(math.clamp(position.X.Offset, 4, math.max(4, viewport.X - size.X - 4)),
        math.clamp(position.Y.Offset, 4, math.max(4, viewport.Y - size.Y - 4)))
    S.ui.scroll.Visible, S.ui.status.Visible, S.ui.title.Visible = not S.minimized, not S.minimized, not S.minimized
    S.ui.close.Visible = not S.minimized
    S.ui.minimize.Size = S.minimized and UDim2.fromScale(1, 1) or UDim2.fromOffset(28, 28)
    S.ui.minimize.Position = S.minimized and UDim2.fromOffset(0, 0) or UDim2.new(1, -66, 0, 7)
    S.ui.minimize.Text = S.minimized and "+" or "-"
end

function H.row(label)
    S.ui.nextOrder = (S.ui.nextOrder or 0) + 1
    local row = H.make("Frame", {Size = UDim2.new(1, 0, 0, 37), BackgroundTransparency = 1,
        LayoutOrder = S.ui.nextOrder}, S.ui.scroll)
    H.make("TextLabel", {Size = UDim2.new(1, -54, 1, 0), BackgroundTransparency = 1, Text = label,
        TextColor3 = Color3.fromRGB(211, 216, 229), TextSize = 12, TextWrapped = true,
        Font = Enum.Font.GothamMedium, TextXAlignment = Enum.TextXAlignment.Left}, row)
    return row
end

function H.toggleRow(key, label)
    local row = H.row(label)
    local button = H.make("TextButton", {Name = key .. "Toggle", Size = UDim2.fromOffset(43, 23),
        Position = UDim2.new(1, -43, 0.5, -11), Text = "", AutoButtonColor = false,
        BackgroundColor3 = Color3.fromRGB(48, 50, 61)}, row)
    H.corner(button, 12)
    local knob = H.make("Frame", {Size = UDim2.fromOffset(14, 14), Position = UDim2.new(0, 4, 0.5, -7),
        BackgroundColor3 = Color3.fromRGB(240, 243, 250)}, button)
    H.corner(knob, 8)
    S.switches[key] = {button = button, knob = knob}
    H.connect(button.Activated, function() H.setToggle(key, not S.flags[key]) end)
end

function H.button(label, callback, color)
    S.ui.nextOrder = (S.ui.nextOrder or 0) + 1
    local button = H.make("TextButton", {Size = UDim2.new(1, 0, 0, 33), Text = label, LayoutOrder = S.ui.nextOrder,
        TextColor3 = Color3.fromRGB(240, 243, 250), TextSize = 12, Font = Enum.Font.GothamBold,
        BackgroundColor3 = color or Color3.fromRGB(44, 95, 143)}, S.ui.scroll)
    H.corner(button, 7)
    H.connect(button.Activated, function()
        local ok, message = pcall(callback)
        if not ok then H.report(label, message) end
    end)
    return button
end

S.ui.gui = H.make("ScreenGui", {Name = "DevPanelGui", ResetOnSpawn = false, IgnoreGuiInset = true,
    ZIndexBehavior = Enum.ZIndexBehavior.Sibling, DisplayOrder = 1000}, playerGui)
S.ui.panel = H.make("Frame", {Name = "MainFrame", Active = true, BackgroundColor3 = Color3.fromRGB(19, 22, 30),
    BorderSizePixel = 0, Position = UDim2.fromOffset(math.max(4, H.viewport().X - 282), 20)}, S.ui.gui)
H.corner(S.ui.panel, 12)
H.make("UIStroke", {Thickness = 1, Color = Color3.fromRGB(61, 72, 95)}, S.ui.panel)
S.ui.title = H.make("TextLabel", {Active = true, Size = UDim2.new(1, -82, 0, 43), Position = UDim2.fromOffset(12, 0),
    BackgroundTransparency = 1, Text = "GR33D  v7.6", TextColor3 = Color3.fromRGB(234, 240, 250),
    TextSize = 14, Font = Enum.Font.GothamBold, TextXAlignment = Enum.TextXAlignment.Left}, S.ui.panel)
S.ui.minimize = H.make("TextButton", {Text = "-", BackgroundColor3 = Color3.fromRGB(37, 43, 57),
    TextColor3 = Color3.fromRGB(234, 240, 250), TextSize = 19, Font = Enum.Font.GothamBold}, S.ui.panel)
H.corner(S.ui.minimize, 8)
S.ui.close = H.make("TextButton", {Size = UDim2.fromOffset(28, 28), Position = UDim2.new(1, -33, 0, 7),
    Text = "x", BackgroundColor3 = Color3.fromRGB(98, 45, 55), TextColor3 = Color3.fromRGB(255, 239, 239),
    TextSize = 15, Font = Enum.Font.GothamBold}, S.ui.panel)
H.corner(S.ui.close, 8)
S.ui.scroll = H.make("ScrollingFrame", {Size = UDim2.new(1, -22, 1, -91), Position = UDim2.fromOffset(11, 45),
    BackgroundTransparency = 1, BorderSizePixel = 0, ScrollBarThickness = 4, CanvasSize = UDim2.fromOffset(0, 0),
    AutomaticCanvasSize = Enum.AutomaticSize.Y, ScrollBarImageColor3 = Color3.fromRGB(95, 117, 151)}, S.ui.panel)
H.make("UIPadding", {PaddingRight = UDim.new(0, 7), PaddingBottom = UDim.new(0, 8)}, S.ui.scroll)
H.make("UIListLayout", {Padding = UDim.new(0, 4), SortOrder = Enum.SortOrder.LayoutOrder}, S.ui.scroll)
S.ui.status = H.make("TextLabel", {Size = UDim2.new(1, -24, 0, 35), Position = UDim2.new(0, 12, 1, -40),
    BackgroundTransparency = 1, Text = "Client protection tools", TextColor3 = Color3.fromRGB(133, 157, 190),
    TextSize = 10, TextWrapped = true, Font = Enum.Font.Gotham, TextXAlignment = Enum.TextXAlignment.Left}, S.ui.panel)

H.toggleRow("lock", "Anti Exploiter / Position Lock")
H.toggleRow("fling", "Anti-Fling / Ragdoll Recovery")
H.toggleRow("monster", "Hostile NPC Evasion")
H.toggleRow("trap", "Anti Trap (seat / stun recovery)")
H.toggleRow("fall", "Fall Speed Limit")
H.toggleRow("health", "Health Recovery (local)")
H.toggleRow("noclip", "Noclip")
H.toggleRow("flight", "Flight (double-tap jump)")
H.toggleRow("run", "Run Speed Override")

local speedRow = H.row("Run Speed (5-200)")
local speedInput = H.make("TextBox", {Name = "RunSpeedInput", Size = UDim2.fromOffset(58, 27),
    Position = UDim2.new(1, -58, 0.5, -13), Text = tostring(S.runSpeed), ClearTextOnFocus = false,
    TextColor3 = Color3.fromRGB(240, 243, 250), BackgroundColor3 = Color3.fromRGB(37, 43, 57),
    TextSize = 12, Font = Enum.Font.GothamBold}, speedRow)
H.corner(speedInput, 6)
H.connect(speedInput.FocusLost, function()
    local number = tonumber(speedInput.Text)
    if H.finite(number) then S.runSpeed = math.clamp(number, Config.MinRunSpeed, Config.MaxRunSpeed)
    else H.notice("Enter a finite numeric run speed", 3) end
    speedInput.Text = tostring(S.runSpeed)
end)
H.toggleRow("afk", "AFK Streak")

H.button("TP to Entry Spawn", function()
    if not H.ready() then H.notice("Wait for a living character", 3); return end
    local spawn = H.findJoinSpawn()
    -- A verified spawn pad takes priority. An initial recorded CFrame is used
    -- only if the map has no eligible SpawnLocation at all.
    local target = spawn and H.spawnRootTarget(spawn) or S.entrySpawn
    if target then H.teleport(target, "entry spawn")
    else H.notice("Join spawn unknown; run the panel at join or use a SpawnLocation", 5) end
end)
H.button("Save Current Map Location", function()
    if not H.ready() then H.notice("Wait for a living character", 3); return end
    S.savedLocation = S.context.root.CFrame
    player:SetAttribute("GR33D_SavedLocation", S.savedLocation)
    H.notice("Saved location at your current position", 3)
end, Color3.fromRGB(106, 65, 147))
H.button("TP to Saved Map Location", function()
    local destination = S.savedLocation or player:GetAttribute("GR33D_SavedLocation")
    if typeof(destination) == "CFrame" then
        S.savedLocation = destination
        H.teleport(destination, "saved map location")
    else H.notice("Save a map location first", 3) end
end, Color3.fromRGB(43, 121, 91))

H.button("Disable All / Restore Character", function()
    for _, key in ipairs({"lock", "monster", "flight", "noclip", "fling", "trap", "run", "afk", "health", "fall"}) do
        H.setToggle(key, false)
    end
    H.releaseAll()
    H.notice("All panel features disabled", 3)
end, Color3.fromRGB(113, 64, 62))

local shutdownEvent = H.make("BindableEvent", {Name = "GR33D_Shutdown"}, S.ui.gui)
H.connect(shutdownEvent.Event, function() H.shutdown(true) end)
H.connect(S.ui.gui.Destroying, function() H.shutdown(false) end)
if typeof(script) == "Instance" then H.connect(script.Destroying, function() H.shutdown(true) end) end
H.connect(S.ui.close.Activated, function() H.shutdown(true) end)
H.connect(S.ui.minimize.Activated, function()
    -- A dragged minimized icon must not also expand on release.
    if (S.dragging and S.dragging.moved) or os.clock() < S.suppressMinimizeUntil then return end
    S.minimized = not S.minimized
    H.layout()
end)

function H.beginDrag(input)
    if S.dragging then return end
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        S.dragging = {input = input, start = input.Position, position = S.ui.panel.Position, moved = false}
    end
end
H.connect(S.ui.title.InputBegan, H.beginDrag)
H.connect(S.ui.minimize.InputBegan, function(input) if S.minimized then H.beginDrag(input) end end)
H.connect(UserInputService.InputChanged, function(input)
    local drag = S.dragging
    if not drag then return end
    local matching = input == drag.input or (drag.input.UserInputType == Enum.UserInputType.MouseButton1
        and input.UserInputType == Enum.UserInputType.MouseMovement)
    if not matching then return end
    local delta = input.Position - drag.start
    if delta.Magnitude > 4 then drag.moved = true end
    S.ui.panel.Position = UDim2.fromOffset(drag.position.X.Offset + delta.X, drag.position.Y.Offset + delta.Y)
    H.layout()
end)
H.connect(UserInputService.InputEnded, function(input)
    if S.dragging and input == S.dragging.input then
        local ended = S.dragging
        if ended.moved then S.suppressMinimizeUntil = os.clock() + 0.2 end
        -- Activated can fire after InputEnded; retain the drag flag through that event.
        task.defer(function() if S.dragging == ended then S.dragging = nil end end)
    end
end)

function H.bindCamera()
    if S.cameraConnection then S.cameraConnection:Disconnect(); S.cameraConnection = nil end
    local camera = Workspace.CurrentCamera
    if camera then S.cameraConnection = camera:GetPropertyChangedSignal("ViewportSize"):Connect(H.layout) end
    H.layout()
end
H.connect(Workspace:GetPropertyChangedSignal("CurrentCamera"), H.bindCamera)
H.bindCamera()

task.spawn(function()
    local scripts = player:WaitForChild("PlayerScripts", 5)
    local module = scripts and scripts:WaitForChild("PlayerModule", 5)
    if not module or not S.alive then return end
    local ok, result = pcall(function() return require(module):GetControls() end)
    if ok then S.controls = result end
end)

H.connect(UserInputService.InputBegan, function(input, gameProcessed)
    if gameProcessed or not S.flags.fling or UserInputService:GetFocusedTextBox() then return end
    if input.UserInputType ~= Enum.UserInputType.Keyboard then return end
    local key = input.KeyCode
    if key == Enum.KeyCode.W or key == Enum.KeyCode.A or key == Enum.KeyCode.S
        or key == Enum.KeyCode.D or key == Enum.KeyCode.Up or key == Enum.KeyCode.Down
        or key == Enum.KeyCode.Left or key == Enum.KeyCode.Right
        or key == Enum.KeyCode.Space then
        S.flingInputUntil = os.clock() + Config.FlingInputReleaseGrace
        H.releaseFlingHold(Config.FlingInputReleaseGrace)
    end
end)

H.connect(UserInputService.InputBegan, function(input, gameProcessed)
    if gameProcessed or not S.flags.afk or UserInputService:GetFocusedTextBox() then return end
    if input.UserInputType ~= Enum.UserInputType.Keyboard then return end
    local key = input.KeyCode
    if key == Enum.KeyCode.W or key == Enum.KeyCode.A or key == Enum.KeyCode.S
        or key == Enum.KeyCode.D or key == Enum.KeyCode.Up or key == Enum.KeyCode.Down
        or key == Enum.KeyCode.Left or key == Enum.KeyCode.Right
        or key == Enum.KeyCode.Space then
        if S.afkPulseUntil and H.ready() then
            S.context.humanoid:Move(Vector3.zero, false)
        end
        H.releaseIdle(true)
        S.afkLastPulse, S.afkPulseUntil = os.clock(), nil
        S.afkPostPulseUntil = -math.huge
    end
end)

H.connect(UserInputService.JumpRequest, function()
    -- A jump always releases the idle lease, even when flight is not armed.
    H.releaseIdle(true)
    if S.flags.fling then
        S.flingInputUntil = os.clock() + Config.FlingInputReleaseGrace
        H.releaseFlingHold(Config.FlingInputReleaseGrace)
    end
    if not S.flags.flight or not H.ready() or S.flags.lock or UserInputService:GetFocusedTextBox() then return end
    local now = os.clock()
    if now - S.lastJumpRequest < 0.12 then return end
    S.lastJumpRequest = now
    if now - S.lastJump <= Config.DoubleTapWindow then
        if S.flying then H.stopMover()
        else
            local state = S.context.humanoid:GetState()
            if state == Enum.HumanoidStateType.Jumping or state == Enum.HumanoidStateType.Freefall
                or S.context.humanoid.FloorMaterial == Enum.Material.Air then H.startMover("flight") end
        end
        S.lastJump = -math.huge
    else S.lastJump = now end
end)

H.connect(Workspace.DescendantAdded, function(instance)
    if S.alive then table.insert(S.pending, instance) end
end)
H.connect(Workspace.DescendantRemoving, H.removeObject)
for _, tags in ipairs({Config.HazardTags, Config.HostileTags, Config.FriendlyTags, {"ProtectionIgnore"}}) do
    for _, tag in ipairs(tags) do
        H.connect(CollectionService:GetInstanceAddedSignal(tag), function(instance)
            if instance:IsDescendantOf(Workspace) then H.watchObject(instance); H.refreshObject(instance); H.queueBranch(instance) end
        end)
        H.connect(CollectionService:GetInstanceRemovedSignal(tag), function(instance) H.refreshObject(instance); H.queueBranch(instance) end)
    end
end
for _, instance in ipairs(Workspace:GetDescendants()) do table.insert(S.pending, instance) end

H.connect(player.CharacterAdded, H.beginCharacter)
H.connect(player.CharacterRemoving, function(character)
    if S.context and S.context.character == character then
        S.setupToken = S.setupToken + 1
        H.cleanupCharacter()
    end
end)
if player.Character then H.beginCharacter(player.Character) end

function H.physicsTick()
    if not H.ready() then return end
    local ctx = S.context
    H.applyCollision()
    H.updateFling("pre")
    H.updateAFK()
    if S.flags.lock then
        if S.escape then H.updateEscapeHold() else H.applyLock() end
        return
    end
    if S.mover then H.updateMover(); return end
    if ctx.root.Anchored then return end
    if S.flags.trap then
        if ctx.humanoid.Sit then ctx.humanoid.Sit = false end
        if ctx.humanoid.PlatformStand then ctx.humanoid.PlatformStand = false end
        local state = ctx.humanoid:GetState()
        if state == Enum.HumanoidStateType.Ragdoll or state == Enum.HumanoidStateType.FallingDown
            or state == Enum.HumanoidStateType.PlatformStanding then
            ctx.humanoid:ChangeState(Enum.HumanoidStateType.GettingUp)
        end
    end
    if S.flags.run and not ctx.humanoid.Sit then
        local direction = ctx.humanoid.MoveDirection
        if direction.Magnitude > 0.01 then
            local velocity = ctx.root.AssemblyLinearVelocity
            ctx.root.AssemblyLinearVelocity = Vector3.new(direction.X * S.runSpeed, velocity.Y, direction.Z * S.runSpeed)
        end
    end
    if S.flags.fall and ctx.humanoid.FloorMaterial == Enum.Material.Air then
        local velocity = ctx.root.AssemblyLinearVelocity
        if velocity.Y < -Config.FallSpeedLimit then
            ctx.root.AssemblyLinearVelocity = Vector3.new(velocity.X, -Config.FallSpeedLimit, velocity.Z)
        end
    end
end

H.connect(RunService.PreSimulation, function()
    local ok, message = pcall(H.physicsTick)
    if not ok then
        -- A broken physics callback must not repeatedly leave the rig anchored.
        for key in pairs(S.flags) do S.flags[key] = false end
        H.stopLock(false); H.stopMover(); H.releaseAll(); H.updateSwitches()
        H.report("Physics update; features disabled", message)
    end
end)

H.connect(RunService.PostSimulation, function(dt)
    if S.flags.fling and H.ready() then
        local ok, message = pcall(H.updateFling, "post", dt)
        if not ok then H.report("Post-physics anti-fling", message) end
    end
end)

H.connect(RunService.Heartbeat, function(deltaTime)
    if not S.alive then return end
    -- Initial indexing is spread across frames instead of blocking on a large map.
    local processed = 0
    while S.pendingIndex <= #S.pending and processed < Config.IndexBatchSize do
        local instance = S.pending[S.pendingIndex]
        S.pendingIndex, processed = S.pendingIndex + 1, processed + 1
        H.watchObject(instance)
    end
    if S.pendingIndex > #S.pending then S.pending, S.pendingIndex = {}, 1 end
    S.scanClock = S.scanClock + deltaTime
    if S.scanClock < Config.ScanInterval then return end
    S.scanClock = 0
    if H.ready() then
        local ok, message = pcall(function()
            H.updateEscape()
            H.updateNPCs()
            local ctx = S.context
            local velocity = ctx.root.AssemblyLinearVelocity
            if not S.flags.lock and not S.mover and not ctx.root.Anchored
                and ctx.humanoid.FloorMaterial ~= Enum.Material.Air and not ctx.humanoid.Sit
                and H.finiteVector(velocity) and velocity.Magnitude < Config.FlingLinearThreshold * 0.5
                and ctx.root.Position.Y > Workspace.FallenPartsDestroyHeight + 10 then
                S.safeRoot, S.safeTime = ctx.root.CFrame, os.clock()
            end
        end)
        if not ok then H.report("Protection scan", message) end
    end
    if S.ui.status and S.ui.status.Parent then
        if os.clock() < S.noticeUntil then S.ui.status.Text = S.notice
        elseif not H.ready() then S.ui.status.Text = "Waiting for a living character"
        elseif S.pendingIndex <= #S.pending then S.ui.status.Text = "Indexing hazards and NPCs..."
        elseif S.escape and S.escape.mode == "emergency" then
            S.ui.status.Text = "Emergency evasion: lock released; tracking pursuit"
        elseif S.escape and S.escape.mode == "manual" then
            S.ui.status.Text = "Teleporting: lock temporarily released"
        elseif S.escape then S.ui.status.Text = "Dodging: lock released; waiting for safety"
        elseif S.flags.lock then S.ui.status.Text = "Position locked - client protection"
        elseif S.flying then S.ui.status.Text = "Flying - double-tap jump to stop"
        elseif S.hovering then S.ui.status.Text = "Evading a hostile NPC"
        elseif S.flags.afk and S.afkPulseUntil then S.ui.status.Text = "AFK Streak: activity pulse"
        elseif S.flags.afk and S.idleTarget then S.ui.status.Text = "AFK Streak: holding idle position"
        elseif S.flags.afk then S.ui.status.Text = "AFK Streak: waiting for stable ground"
        else S.ui.status.Text = "Client protection tools ready" end
    end
end)

H.updateSwitches()
print("[GR33D v7.7] Anti-fling: immediate idle guard + pre/post impact recovery; anti-ragdoll, AFK Streak, flight, and escape preserved.")
