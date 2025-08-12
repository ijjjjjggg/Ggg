
-- Lead Hub (Auto Block & ESP Script)
-- By TripleX's Hub

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")
local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")

local lp = Players.LocalPlayer
local PlayerGui = lp:WaitForChild("PlayerGui")
local Humanoid

local Rayfield = loadstring(game:HttpGet("https://sirius.menu/rayfield"))()
local Window = Rayfield:CreateWindow({
    Name = "Lead Hub",
    LoadingTitle = "Auto Block Script",
    LoadingSubtitle = "by TripleX's Hub",
    ConfigurationSaving = {
        Enabled = true,
        FolderName = "LeadHub",
        FileName = "Settings"
    },
    Discord = {Enabled = false},
    KeySystem = false
})

-- Tabs
local AutoBlockTab = Window:CreateTab("Auto Block", 4483362458)
local VisualsTab = Window:CreateTab("Visuals", 4483362458)
local MiscTab = Window:CreateTab("Misc", 4483362458)

-- Auto Block Variables
local autoBlockOn = false
local autoBlockAudioOn = false
local strictRangeOn = false
local facingCheckEnabled = true
local looseFacing = true
local detectionRange = 18

-- Auto Block Sounds & Animations IDs
local autoBlockTriggerSounds = {
    ["102228729296384"] = true,
    ["140242176732868"] = true,
    ["112809109188560"] = true,
    ["136323728355613"] = true,
    ["115026634746636"] = true,
    ["84116622032112"] = true,
    ["108907358619313"] = true,
    ["127793641088496"] = true,
    ["86174610237192"] = true,
    ["95079963655241"] = true,
    ["101199185291628"] = true,
    ["119942598489800"] = true,
    ["84307400688050"] = true,
}

local autoBlockTriggerAnims = {
    "126830014841198", "126355327951215", "121086746534252", "18885909645",
    "98456918873918", "105458270463374", "83829782357897", "125403313786645",
    "118298475669935", "82113744478546", "70371667919898", "99135633258223",
    "97167027849946", "109230267448394", "139835501033932", "126896426760253",
    "109667959938617", "126681776859538", "129976080405072", "121293883585738",
    "81639435858902", "137314737492715",
    "92173139187970"
}

-- Helper Functions for Auto Block
local function fireRemoteBlock()
    local args = {"UseActorAbility", "Block"}
    ReplicatedStorage:WaitForChild("Modules"):WaitForChild("Network"):WaitForChild("RemoteEvent"):FireServer(unpack(args))
end

local function isFacing(localRoot, targetRoot)
    if not facingCheckEnabled then return true end
    local dir = (localRoot.Position - targetRoot.Position).Unit
    local dot = targetRoot.CFrame.LookVector:Dot(dir)
    return looseFacing and dot > -0.3 or dot > 0
end

-- Robust Sound Auto Block (simplified)
local soundHooks = {}
local soundBlockedUntil = {}

local function extractNumericSoundId(sound)
    if not sound or not sound.SoundId then return nil end
    local sid = tostring(sound.SoundId)
    local num = sid:match("%d+")
    return num
end

local function getSoundWorldPosition(sound)
    if not sound then return nil end
    if sound.Parent and sound.Parent:IsA("BasePart") then
        return sound.Parent.Position, sound.Parent
    end
    if sound.Parent and sound.Parent:IsA("Attachment") and sound.Parent.Parent and sound.Parent.Parent:IsA("BasePart") then
        return sound.Parent.Parent.Position, sound.Parent.Parent
    end
    return nil, nil
end

local function attemptBlockForSound(sound)
    if not autoBlockAudioOn then return end
    if not sound or not sound:IsA("Sound") then return end
    if not sound.IsPlaying then return end

    local id = extractNumericSoundId(sound)
    if not id or not autoBlockTriggerSounds[id] then return end

    local myRoot = lp.Character and lp.Character:FindFirstChild("HumanoidRootPart")
    if not myRoot then return end

    if soundBlockedUntil[sound] and tick() < soundBlockedUntil[sound] then return end

    local soundPos, soundPart = getSoundWorldPosition(sound)
    local shouldBlock = false
    local facingOK = true

    if soundPos then
        local dist = (myRoot.Position - soundPos).Magnitude
        if dist <= detectionRange then
            shouldBlock = true
            if facingCheckEnabled and soundPart and soundPart:IsA("BasePart") then
                facingOK = isFacing(myRoot, soundPart)
            end
        end
    else
        shouldBlock = true
        facingOK = true
    end

    if shouldBlock and facingOK then
        fireRemoteBlock()
        soundBlockedUntil[sound] = tick() + 1.2
    end
end

local function hookSound(sound)
    if not sound or not sound:IsA("Sound") then return end
    if soundHooks[sound] then return end

    local playedConn = sound.Played:Connect(function()
        pcall(attemptBlockForSound, sound)
    end)

    local propConn = sound:GetPropertyChangedSignal("IsPlaying"):Connect(function()
        if sound.IsPlaying then
            pcall(attemptBlockForSound, sound)
        end
    end)

    local destroyConn
    destroyConn = sound.Destroying:Connect(function()
        if playedConn and playedConn.Connected then playedConn:Disconnect() end
        if propConn and propConn.Connected then propConn:Disconnect() end
        if destroyConn and destroyConn.Connected then destroyConn:Disconnect() end
        soundHooks[sound] = nil
        soundBlockedUntil[sound] = nil
    end)

    soundHooks[sound] = {playedConn, propConn, destroyConn}

    if sound.IsPlaying then
        task.spawn(function() pcall(attemptBlockForSound, sound) end)
    end
end

-- Hook existing and new sounds
for _, desc in ipairs(game:GetDescendants()) do
    if desc:IsA("Sound") then
        pcall(hookSound, desc)
    end
end
game.DescendantAdded:Connect(function(desc)
    if desc:IsA("Sound") then
        pcall(hookSound, desc)
    end
end)

-- Auto Block Toggle UI
AutoBlockTab:CreateToggle({
    Name = "Auto Block (Animation)",
    CurrentValue = false,
    Callback = function(Value) autoBlockOn = Value end
})

AutoBlockTab:CreateToggle({
    Name = "Auto Block (Audio)",
    CurrentValue = false,
    Callback = function(Value) autoBlockAudioOn = Value end
})

AutoBlockTab:CreateToggle({
    Name = "Strict Range",
    CurrentValue = false,
    Callback = function(Value) strictRangeOn = Value end
})

AutoBlockTab:CreateToggle({
    Name = "Enable Facing Check",
    CurrentValue = true,
    Callback = function(Value) facingCheckEnabled = Value end
})

AutoBlockTab:CreateDropdown({
    Name = "Facing Check Mode",
    Options = {"Loose", "Strict"},
    CurrentOption = "Loose",
    Callback = function(Option)
        looseFacing = Option == "Loose"
    end
})

AutoBlockTab:CreateInput({
    Name = "Detection Range",
    PlaceholderText = "18",
    RemoveTextAfterFocusLost = false,
    Callback = function(Text)
        local num = tonumber(Text)
        if num then detectionRange = num end
    end
})

-- Auto Block loop
RunService.RenderStepped:Connect(function()
    if not autoBlockOn then return end
    local myChar = lp.Character
    if not myChar then return end
    local myRoot = myChar:FindFirstChild("HumanoidRootPart")
    local myHum = myChar:FindFirstChildOfClass("Humanoid")
    if not myRoot or not myHum then return end

    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= lp and plr.Character then
            local hrp = plr.Character:FindFirstChild("HumanoidRootPart")
            local hum = plr.Character:FindFirstChildOfClass("Humanoid")
            local animator = hum and hum:FindFirstChildOfClass("Animator")
            local animTracks = animator and animator:GetPlayingAnimationTracks()

            if hrp and (hrp.Position - myRoot.Position).Magnitude <= detectionRange then
                for _, track in ipairs(animTracks or {}) do
                    local id = tostring(track.Animation.AnimationId):match("%d+")
                    if table.find(autoBlockTriggerAnims, id) then
                        if (not strictRangeOn or (hrp.Position - myRoot.Position).Magnitude <= detectionRange) and isFacing(myRoot, hrp) then
                            fireRemoteBlock()
                        end
                    end
                end
            end
        end
    end
end)

-- ===== ESP (Players & Items) =====

local espEnabled = false
local itemEspEnabled = false

local highlights = {}
local itemHighlights = {}

local function createHighlight(parent, color)
    local hl = Instance.new("Highlight")
    hl.Adornee = parent
    hl.FillColor = color
    hl.FillTransparency = 0.7
    hl.OutlineColor = color
    hl.OutlineTransparency = 0
    hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    hl.Parent = workspace
    return hl
end

-- Clear highlights (players)
local function clearHighlights()
    for _, hl in pairs(highlights) do
        if hl and hl.Parent then
            hl:Destroy()
        end
    end
    highlights = {}
end

-- Clear item highlights
local function clearItemHighlights()
    for _, hl in pairs(itemHighlights) do
        if hl and hl.Parent then
            hl:Destroy()
        end
    end
    itemHighlights = {}
end

-- Player ESP loop (optimized)
local function updatePlayerESP()
    if not espEnabled then
        clearHighlights()
        return
    end

    local myTeam = lp.Team
    clearHighlights()

    for _, player in pairs(Players:GetPlayers()) do
        if player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
            local hrp = player.Character.HumanoidRootPart
            local color = Color3.fromRGB(255, 0, 0) -- default Red enemy

            if myTeam and player.Team == myTeam then
                color = Color3.fromRGB(0, 135, 255) -- Blue for teammates
            end

            local hl = createHighlight(player.Character, color)
            highlights[player] = hl
        end
    end
end

-- Item ESP loop (optimized)
local function updateItemESP()
    if not itemEspEnabled then
        clearItemHighlights()
        return
    end

    clearItemHighlights()

    -- Search workspace for Medkit & Cola (case insensitive)
    for _, obj in pairs(workspace:GetDescendants()) do
        if obj:IsA("BasePart") then
            local nameLower = obj.Name:lower()
            if nameLower:find("medkit") or nameLower:find("cola") then
                -- Avoid duplicates if already highlighted
                if not itemHighlights[obj] then
                    local hl = createHighlight(obj, Color3.fromRGB(255, 255, 0)) -- Yellow highlight
                    itemHighlights[obj] = hl
                end
            end
        end
    end
end

-- ESP auto-refresh loop
task.spawn(function()
    while true do
        task.wait(2) -- Refresh every 2 seconds for optimization
        updatePlayerESP()
        updateItemESP()
    end
end)

-- Visuals Tab UI
VisualsTab:CreateToggle({
    Name = "Player ESP (Highlights)",
    CurrentValue = false,
    Callback = function(Value)
        espEnabled = Value
        if not Value then
            clearHighlights()
        end
    end
})

VisualsTab:CreateToggle({
    Name = "Item ESP (Medkit & Cola)",
    CurrentValue = false,
    Callback = function(Value)
        itemEspEnabled = Value
        if not Value then
            clearItemHighlights()
        end
    end
})

-- Misc Tab - Example Buttons
MiscTab:CreateButton({
    Name = "Run Infinite Yield",
    Callback = function()
        loadstring(game:HttpGet("https://raw.githubusercontent.com/EdgeIY/infiniteyield/master/source"))()
    end,
})

MiscTab:CreateParagraph({
    Title = "Tip",
    Content = 'Run Infinite Yield and type "antifling" to help with punch fling.'
})

-- Finished message
print("✅ Lead Hub loaded! Enjoy.")

--[[
Drop this whole script in your executor (Delta recommended) and run.
Use the toggles under 'Auto Block' and 'Visuals' tabs.
]]
