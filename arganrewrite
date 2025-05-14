--// Services
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local TextChatService = game:GetService("TextChatService")

local camera = workspace.CurrentCamera

--// UIM Global
local Rayfield = nil

-- Attempt to load Rayfield
local successRayfield, rayfieldInstance = pcall(function()
    return loadstring(game:HttpGet('https://sirius.menu/rayfield', true))()
end)

if not successRayfield or not rayfieldInstance then
    warn("CRITICAL: Rayfield UI Library failed to load. Error: " .. tostring(rayfieldInstance))
    local coreGui = game:GetService("CoreGui")
    local errScreen = Instance.new("ScreenGui", coreGui)
    errScreen.ResetOnSpawn = false
    errScreen.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    local errLabel = Instance.new("TextLabel", errScreen)
    errLabel.Size = UDim2.new(0.8, 0, 0.2, 0)
    errLabel.Position = UDim2.new(0.1, 0, 0.4, 0)
    errLabel.Text = "FATAL ERROR:\nArgan UI (Rayfield) failed to load.\nScript cannot continue.\nDetails: " .. tostring(rayfieldInstance)
    errLabel.TextColor3 = Color3.new(1,0.2,0.2)
    errLabel.BackgroundColor3 = Color3.new(0.1,0.1,0.1)
    errLabel.BorderColor3 = Color3.new(1,0.2,0.2)
    errLabel.BorderSizePixel = 2
    errLabel.Font = Enum.Font.SourceSansSemibold
    errLabel.TextWrapped = true
    errLabel.TextScaled = false
    errLabel.FontSize = Enum.FontSize.Size18
    return -- Stop script execution
end
Rayfield = rayfieldInstance
Rayfield:Notify({Title = "Argan", Content = "UI Library Loaded.", Duration = 2, Type = "info"})

--// Player and Character Globals (initialized after Rayfield for notifications)
local player = nil
local character = nil
local humanoid = nil
local mouse = nil

-- Forward declare functions that might be called by OnCharacterAdded or init
local setupJump
local ApplyNoClipState -- For NoClip logic on character changes

--// Global state variable for CamLock target
local camLockLockedTarget = nil
local toggleKey = Enum.KeyCode.X -- Keybind for CamLock actions

--// Spider Climb Variables
local climbSpeed = 25
local wallCheckDistance = 2
local keysDown = {}
local isClimbing = false
local currentWallNormal = Vector3.new(0, 0, 0)

--// Helper Functions
local function isEnemy(p_param) 
    if not player or not p_param then return false end
    if p_param == player then return false end
    if not player.Team or not p_param.Team then return true end
    return p_param.Team ~= player.Team
end

local function findNearestLivingEnemyHRP(maxDist)
    maxDist = maxDist or math.huge
    local nearestHRP = nil
    local minDist = maxDist
    if not player or not player.Character or not player.Character:FindFirstChild("HumanoidRootPart") then return nil end
    local playerPos = player.Character.HumanoidRootPart.Position

    for _, p_iter in ipairs(Players:GetPlayers()) do
        if p_iter ~= player and p_iter.Character and p_iter.Character:FindFirstChild("HumanoidRootPart") and isEnemy(p_iter) then
            local enemyHumanoid = p_iter.Character:FindFirstChildOfClass("Humanoid")
            if enemyHumanoid and enemyHumanoid.Health > 0 then
                local dist = (p_iter.Character.HumanoidRootPart.Position - playerPos).Magnitude
                if dist < minDist then
                    minDist = dist
                    nearestHRP = p_iter.Character.HumanoidRootPart
                end
            end
        end
    end
    return nearestHRP
end

--// Improved Spider Climb Functions
local function checkWallsAroundCharacter()
    if not character or not character:FindFirstChild("HumanoidRootPart") then return false, Vector3.new(0, 0, 0) end
    
    local hrp = character.HumanoidRootPart
    local rayDirections = {
        hrp.CFrame.LookVector,  -- Front
        -hrp.CFrame.LookVector, -- Back
        hrp.CFrame.RightVector,  -- Right
        -hrp.CFrame.RightVector, -- Left
        Vector3.new(0, 1, 0),    -- Up
        Vector3.new(0, -1, 0)    -- Down
    }
    
    local raycastParams = RaycastParams.new()
    raycastParams.FilterDescendantsInstances = {character}
    raycastParams.FilterType = Enum.RaycastFilterType.Blacklist
    
    for _, direction in ipairs(rayDirections) do
        local rayOrigin = hrp.Position
        local rayDirection = direction * wallCheckDistance
        local result = workspace:Raycast(rayOrigin, rayDirection, raycastParams)
        
        if result and result.Instance then
            return true, result.Normal
        end
    end
    
    return false, Vector3.new(0, 0, 0)
end

local function handleSpiderClimb()
    if not _G.SpiderEnabled or not character or not character:FindFirstChild("HumanoidRootPart") or not character:FindFirstChildOfClass("Humanoid") then 
        isClimbing = false
        return 
    end
    
    local hrp = character.HumanoidRootPart
    local hum = character:FindFirstChildOfClass("Humanoid")
    
    local wallFound, wallNormal = checkWallsAroundCharacter()
    currentWallNormal = wallNormal
    
    if wallFound then
        isClimbing = true
        hum:ChangeState(Enum.HumanoidStateType.Freefall)
        
        local climbVector = Vector3.new(0, 0, 0)
        local cameraCF = camera.CFrame
        local rightVector = cameraCF.RightVector
        local upVector = Vector3.new(0, 1, 0)
        
        -- Calculate movement direction relative to wall normal
        if keysDown[Enum.KeyCode.W] then
            climbVector = climbVector - currentWallNormal
        end
        if keysDown[Enum.KeyCode.S] then
            climbVector = climbVector + currentWallNormal
        end
        if keysDown[Enum.KeyCode.A] then
            climbVector = climbVector - rightVector
        end
        if keysDown[Enum.KeyCode.D] then
            climbVector = climbVector + rightVector
        end
        if keysDown[Enum.KeyCode.Space] then
            climbVector = climbVector + upVector
        end
        if keysDown[Enum.KeyCode.LeftShift] then
            climbVector = climbVector - upVector
        end
        
        -- Apply movement
        if climbVector.Magnitude > 0 then
            hrp.Velocity = climbVector.Unit * climbSpeed
        else
            -- Stick to wall when no movement keys are pressed
            hrp.Velocity = Vector3.new(0, 0, 0)
        end
    else
        isClimbing = false
    end
end

--// ESP Logic
local function removeESP(char_param)
    if char_param then
        local hl = char_param:FindFirstChild("ArganHighlight")
        if hl then hl:Destroy() end
    end
end

local function applyESP(char_param, enemyPlayer) 
    if not char_param or not enemyPlayer then return end
    local hl = char_param:FindFirstChild("ArganHighlight")
    local targetColor
    if enemyPlayer.Team and enemyPlayer.Team.TeamColor then
        targetColor = enemyPlayer.Team.TeamColor.Color
    else
        targetColor = Color3.new(1, 0, 0)
    end
    if not hl then
        hl = Instance.new("Highlight", char_param); hl.Name = "ArganHighlight"; hl.FillTransparency = 1; hl.OutlineTransparency = 0; hl.Adornee = char_param; hl.OutlineColor = targetColor
    else
        if hl.OutlineColor ~= targetColor then hl.OutlineColor = targetColor end
    end
end

local function updateESP()
    if not (_G and _G.ESPEnabled) then -- Check _G exists
        for _, p_iter in ipairs(Players:GetPlayers()) do
            if p_iter.Character then removeESP(p_iter.Character) end
        end
        return
    end

    if not player then return end
    for _, p_iter in ipairs(Players:GetPlayers()) do
        if p_iter.Character then
            if p_iter ~= player then
                if isEnemy(p_iter) then applyESP(p_iter.Character, p_iter) else removeESP(p_iter.Character) end
            else removeESP(p_iter.Character) end 
        end
    end
end

--// Fly Logic
local flySpeed = 18
local flyTimer_Active = false
local flyTimer_EndsAt = 0
local function Fly()
    if not (_G and _G.FlyEnabled) then return end
    if not player or not character or not humanoid then return end

    local hrp = character:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    
    local playerGui = player:FindFirstChild("PlayerGui")
    if not playerGui or not camera then return end

    local moveDirection = Vector3.new(0,0,0)
    local camCF = camera.CFrame
    local forwardXZ = (camCF.LookVector * Vector3.new(1,0,1)).Unit
    local rightXZ = (camCF.RightVector * Vector3.new(1,0,1)).Unit
    if UserInputService:IsKeyDown(Enum.KeyCode.W) then moveDirection = moveDirection + forwardXZ end
    if UserInputService:IsKeyDown(Enum.KeyCode.S) then moveDirection = moveDirection - forwardXZ end
    if UserInputService:IsKeyDown(Enum.KeyCode.A) then moveDirection = moveDirection - rightXZ end
    if UserInputService:IsKeyDown(Enum.KeyCode.D) then moveDirection = moveDirection + rightXZ end
    if UserInputService:IsKeyDown(Enum.KeyCode.Space) then moveDirection = moveDirection + Vector3.new(0,1,0) end
    if UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then moveDirection = moveDirection - Vector3.new(0,1,0) end
    
    local finalVelocity = moveDirection.Magnitude > 0.001 and moveDirection.Unit * flySpeed or Vector3.new(0,0,0)
    
    local bodyVelocity = hrp:FindFirstChild("ArganFlyVelocity")
    if not bodyVelocity then 
        bodyVelocity = Instance.new("BodyVelocity", hrp)
        bodyVelocity.Name = "ArganFlyVelocity"; bodyVelocity.MaxForce = Vector3.new(math.huge,math.huge,math.huge); bodyVelocity.P = 50000 
    end
    bodyVelocity.Velocity = finalVelocity

    local timerScreenGui = playerGui:FindFirstChild("FlyTimerUI_Argan")
    if not timerScreenGui then
        timerScreenGui = Instance.new("ScreenGui", playerGui); timerScreenGui.Name = "FlyTimerUI_Argan"; timerScreenGui.ResetOnSpawn = false; timerScreenGui.IgnoreGuiInset = true
        local frame = Instance.new("Frame", timerScreenGui); frame.Name = "TimerFrame"; frame.BackgroundColor3 = Color3.new(0,0,0); frame.BackgroundTransparency = 0.5; frame.BorderSizePixel = 0; frame.Size = UDim2.new(1,0,0.08,0); frame.Position = UDim2.new(0,0,0,0)
        local label = Instance.new("TextLabel", frame); label.Name = "CountdownLabel"; label.BackgroundTransparency = 1; label.Size = UDim2.new(1,0,1,0); label.TextColor3 = Color3.new(1,1,1); label.Font = Enum.Font.SourceSansBold; label.TextScaled = true; label.Text = ""
    end
    timerScreenGui.Enabled = true
    local frame = timerScreenGui:FindFirstChild("TimerFrame")
    local label = frame and frame:FindFirstChild("CountdownLabel")
    if not label then return end

    if humanoid:GetState() == Enum.HumanoidStateType.Physics then
        if not flyTimer_Active then flyTimer_Active = true; flyTimer_EndsAt = tick() + 2.3 end
    else 
        if flyTimer_Active then flyTimer_Active = false end
    end

    if flyTimer_Active then
        local remainingTime = flyTimer_EndsAt - tick()
        label.Text = string.format("%.1f", math.max(0, remainingTime))
        if remainingTime <= 0 then
            flyTimer_Active = false
            if hrp and hrp.Parent and hrp.Parent:FindFirstChildOfClass("Humanoid") then 
                local bv = hrp:FindFirstChild("ArganFlyVelocity"); if bv then bv:Destroy() end 
                hrp.Velocity = Vector3.new(0, -200, 0) 
            end
        end
    else label.Text = "" end
end

--// Kill Aura Logic
local killAuraRange = 15; local killAuraCooldown = 0.2; local lastKillAuraHitTime = 0
local function performSimulatedAttackOn(targetChar) end
local function processKillAura()
    if not (_G and _G.KillAuraEnabled) then return end
    if not player or not player.Character or not player.Character:FindFirstChild("HumanoidRootPart") then return end
    if tick() - lastKillAuraHitTime < killAuraCooldown then return end
    local targetHRP = findNearestLivingEnemyHRP(killAuraRange)
    if targetHRP and targetHRP.Parent then
        lastKillAuraHitTime = tick(); performSimulatedAttackOn(targetHRP.Parent)
    end
end

--// Player Character and Input Setup Definitions (before connecting to events)
setupJump = function(h_param)
    if not h_param then return end
    UserInputService.JumpRequest:Connect(function()
        if _G and _G.InfiniteJumpEnabled and h_param and h_param.Parent and h_param:GetState() ~= Enum.HumanoidStateType.Dead then
            h_param:ChangeState(Enum.HumanoidStateType.Jumping)
        end
    end)
end

ApplyNoClipState = function()
    if not character then return end
    local noClipActive = _G and _G.NoClipEnabled
    for _, part_descendant in ipairs(character:GetDescendants()) do
        if part_descendant:IsA("BasePart") then
            part_descendant.CanCollide = not noClipActive
        end
    end
end

--// Player Initialization Logic
local characterAddedConnection = nil
local playerAddedConnection = nil

local function OnCharacterAdded_Local(newChar) -- Renamed to avoid conflict if any
    character = newChar
    humanoid = character:WaitForChild("Humanoid")
    if humanoid then
        if typeof(setupJump) == "function" then pcall(setupJump, humanoid) end
        if _G and humanoid.Parent then -- Apply speed after humanoid is confirmed
             humanoid.WalkSpeed = (_G.SpeedLocked and 23 or 16)
        end
    end
    if typeof(ApplyNoClipState) == "function" then pcall(ApplyNoClipState) end
end

local function InitializePlayerAndCharacter()
    local localPlayerAttempt = Players.LocalPlayer
    if localPlayerAttempt then
        player = localPlayerAttempt
        mouse = player:GetMouse()

        if characterAddedConnection then characterAddedConnection:Disconnect(); characterAddedConnection = nil; end
        characterAddedConnection = player.CharacterAdded:Connect(OnCharacterAdded_Local)

        if player.Character then OnCharacterAdded_Local(player.Character) end
        
        if playerAddedConnection then playerAddedConnection:Disconnect(); playerAddedConnection = nil; end
        Rayfield:Notify({Title = "Argan", Content = "Player Ready.", Duration = 2, Type = "success"})
        return true
    end
    return false
end

if not InitializePlayerAndCharacter() then
    local waitingNotif = Rayfield:Notify({Title = "Argan", Content = "Waiting for Player...", Duration = math.huge, Image = "loader-2"})
    playerAddedConnection = Players:GetPropertyChangedSignal("LocalPlayer"):Connect(function()
        if Players.LocalPlayer and InitializePlayerAndCharacter() then
            if waitingNotif then waitingNotif:Destroy(); waitingNotif = nil; end
        end
    end)
    -- Backup loop
    while not player do
        RunService.Stepped:Wait()
        if Players.LocalPlayer and InitializePlayerAndCharacter() then
             if waitingNotif then waitingNotif:Destroy(); waitingNotif = nil; end
             break
        end
    end
    if not player and waitingNotif then -- Fallback if still no player
        waitingNotif:Destroy()
        Rayfield:Notify({Title = "Argan Error", Content = "Player object timeout.", Duration = 5, Type = "error"})
    end
end

--// Auto Toxic Logic
local function initializeAutoToxic()
    local localPlayer = Players.LocalPlayer
    if not localPlayer then
        localPlayer = Players.PlayerAdded:Wait()
    end

    local keywords = {
        "hack", "hacks", "hacker", "hacking", "hax", "h4x", "h4ck", "h4cker", "h4x0r", "haxor", "hackzor",
        "cheat", "cheats", "cheater", "cheating", "ch3at", "ch3ater", "cheata", "cheeze", "cheez", "cheezer",
        "exploit", "exploits", "exploiter", "exploiting", "expl0it", "expl0iter", "expolit", "expoliter",
        "script", "scripts", "scripter", "scripting", "skript", "skripter", "skripting",
        "bot", "bots", "botter", "botting", "b0t", "b0tter", "b0tting",
        "mod", "mods", "modder", "modding", "modded", "m0d", "m0dder",
        "aimbot", "aimb0t", "aim lock", "aimlock", "aim assist", "aimassist", "silent aim", "silentaim",
        "wallhack", "wallhacks", "wall hack", "wall hax", "wall esp", "see through walls", "xray", "x-ray",
        "speedhack", "speedhacks", "speed hack", "speed hax", "fast run", "fastrun", "speed boost",
        "noclip", "no clip", "no-clip", "clip through", "phase", "phasing", "ghost", "ghosting",
        "flyhack", "fly hack", "fly hax", "flying", "airwalk", "air walk", "hover", "hovering",
        "teleport", "teleporting", "teleport hack", "telehack", "tping", "tp hack", "blink", "blinking",
        "bhop", "bunny hop", "bunnyhop", "bhop script", "autobhop", "auto bhop", "bunny hop hack",
        "triggerbot", "trigger bot", "auto shoot", "autoshoot", "auto fire", "autofire", "trigger",
        "esp", "e.s.p", "extra sensory", "see players", "player glow", "highlight", "highlighter",
        "radar", "radar hack", "minimap hack", "map hack", "maphack", "minimap cheat", "worldhack",
        "spinbot", "spin bot", "spin hack", "anti-aim", "antiaim", "aa", "a.a", "desync", "de-sync",
        "hitbox", "hit box", "hitbox extend", "extended hitbox", "big hitbox", "hitbox hack",
        "lag switch", "lagswitch", "lagging", "intentional lag", "fake lag", "fake ping",
        "damage", "damage hack", "damage mod", "one hit", "onehit", "ohk", "instakill", "insta kill",
        "recoil", "recoil hack", "no recoil", "norecoil", "recoil control", "recoil script",
        "spread", "spread hack", "no spread", "nospread", "perfect accuracy", "accuracy hack",
        "rate", "rate hack", "fire rate", "firerate", "rapid fire", "rapidfire", "fast shoot",
        "ammo", "ammo hack", "infinite ammo", "inf ammo", "no reload", "noreload", "unlimited",
        "health", "health hack", "god mode", "godmode", "invincible", "infinite health", "inf health",
        "stamina", "stamina hack", "infinite stamina", "inf stamina", "no exhaust", "noexhaust",
        "cooldown", "cooldown hack", "no cooldown", "nocooldown", "ability hack", "skill hack",
        "smurf", "smurfs", "smurfing", "smurf account", "alt account", "alt", "alting",
        "boost", "boosting", "account boost", "rank boost", "elo boost", "booster", "boosted",
        "stream", "stream snipe", "streamsniping", "stream cheat", "stream hack", "streamer",
        "team", "teaming", "team hack", "team exploit", "cross team", "crossteam", "team up",
        "report", "reporting", "false report", "mass report", "report abuse", "report spam",
        "toxic", "toxicity", "being toxic", "toxic player", "toxic kid", "toxic noob",
        "trash", "garbage", "no skill", "skill issue", "bad", "awful", "terrible", "horrible", "worst",
        "unfair", "broken", "op", "overpowered", "imba", "nerf", "buff", "balance", "game balance",
        "devs", "developer", "developers", "mods", "admins", "admin", "staff", "game master", "gm",
        "pay", "pay to win", "p2w", "ptw", "whale", "whales", "spender", "spenders", "purchase",
        "match", "matchmaking", "mm", "elo hell", "ranked", "ranking", "rank", "ranks", "tier",
        "luck", "lucky", "unlucky", "rng", "random", "rigged", "fixed", "scripted", "predetermined",
        "pc", "master race", "console", "xbox", "playstation", "ps", "switch", "mobile", "phone",
        "controller", "mnk", "mouse", "keyboard", "input", "input hack", "input cheat", "macro",
        "ping", "high ping", "low ping", "ping abuse", "ping exploit", "lag", "lagging", "latency",
        "fps", "frame", "frames", "frame rate", "fps hack", "fps cheat", "fps boost", "fps unlock",
        "noob", "nub", "newb", "newbie", "beginner", "bad player", "trash player", "garbage player",
        "kid", "child", "baby", "infant", "toddler", "son", "daughter", "fatherless", "motherless",
        "dog", "dogwater", "bot", "ai", "robot", "automaton", "script kiddie", "skid", "skiddie",
        "loser", "failure", "disappointment", "embarrassment", "shame", "shameful", "pathetic",
        "cry", "crying", "tears", "salty", "salt", "mad", "anger", "angry", "rage", "raging",
        "quit", "quitting", "leave", "leaving", "afk", "away", "gone", "disconnected", "dc",
        "report", "reported", "ban", "banned", "suspend", "suspended", "kick", "kicked", "remove",
        "sus", "sussy", "suspect", "suspicious", "fishy", "dodgy", "shady", "sketchy", "weird",
        "glitch", "glitches", "glitched", "glitching", "bug", "bugs", "bugged", "bug abuse",
        "abuse", "abusing", "abuser", "exploitative", "unethical", "dishonest", "cheap", "lame",
        "tryhard", "try hard", "sweat", "sweaty", "competitive", "comp", "ranked", "grinding",
        "carry", "carried", "hard carry", "boosted", "elo inflated", "rank inflated", "fake rank",
        "kys", "kill yourself", "end yourself", "uninstall", "delete game", "quit game", "never play again",
        "disgusting", "disgrace", "disgraceful", "worthless", "useless", "brainless", "clueless",
        "retard", "retarded", "autistic", "disabled", "handicapped", "mental", "mentally ill",
        "cancer", "aids", "hiv", "covid", "plague", "disease", "infected", "contagious",
        "ugly", "fat", "obese", "skinny", "weak", "frail", "pathetic", "pitiful", "waste",
        "fatherless", "motherless", "orphan", "adopted", "bastard", "illegitimate", "mistake",
        "hacker" ,"tricheur" ,"cheater" ,"hacker" ,"tricheur" ,"cheater" ,"hacker" ,"tricheur" ,"cheater", -- French
        "hacker" ,"tramposo" ,"cheater" ,"hacker" ,"tramposo" ,"cheater" ,"hacker" ,"tramposo" ,"cheater", -- Spanish
        "hacker" ,"imbroglione" ,"cheater" ,"hacker" ,"imbroglione" ,"cheater" ,"hacker" ,"imbroglione" ,"cheater", -- Italian
        "hacker" ,"Schummler" ,"cheater" ,"hacker" ,"Schummler" ,"cheater" ,"hacker" ,"Schummler" ,"cheater", -- German
        "hacker" ,"жулик" ,"cheater" ,"hacker" ,"жулик" ,"cheater" ,"hacker" ,"жулик" ,"cheater", -- Russian
        "hacker" ,"cheater" ,"骗子" ,"hacker" ,"cheater" ,"骗子" ,"hacker" ,"cheater" ,"骗子", -- Chinese
        "hacker" ,"cheater" ,"詐欺師" ,"hacker" ,"cheater" ,"詐欺師" ,"hacker" ,"cheater" ,"詐欺師", -- Japanese
        "hacker" ,"cheater" ,"사기꾼" ,"hacker" ,"cheater" ,"사기꾼" ,"hacker" ,"cheater" ,"사기꾼", -- Korean
        "h4x0r", "h4ckz0r", "h4ck3r", "h4ck1ng", "h4ck5", "h4x1ng", "h4x5", "h4x3d",
        "ch33z", "ch33z3", "ch33z3r", "ch33z1ng", "ch33t", "ch33t3r", "ch33t1ng",
        "3xpl01t", "3xpl01t3r", "3xpl01t1ng", "3xp10it", "3xp10it3r", "3xp10it1ng",
        "5cr1pt", "5cr1pt3r", "5cr1pt1ng", "5kr1pt", "5kr1pt3r", "5kr1pt1ng",
        "b07", "b0773r", "b071ng", "b0t", "b0tt3r", "b0tt1ng",
        "m0d", "m0dd3r", "m0dd1ng", "m0d3d", "m0d1f13d", "m0d1fy",
        "haker", "hakcer", "haxer", "haxcer", "haxor", "haxxor", "haxxer", "haxx0r",
        "cheeter", "cheate", "cheta", "chetaer", "cheet", "cheetr", "cheetar", "cheetah",
        "expliot", "explot", "explot", "exploter", "exploiter", "exploer", "exploar",
        "scrip", "scirpt", "scirpter", "scirpting", "skript", "skirpt", "skirpter",
        "bott", "bote", "bottar", "botting", "boting", "boter", "botar",
        "modd", "moder", "modding", "moding", "moder", "modar", "modder"
    }

    local replyTemplates = {
        "Nah {playerName}, you just suck at the game.",
        "Seriously {playerName}? Get good.",
        "{playerName}, maybe try practicing instead of complaining?",
        "Sounds like a skill issue, {playerName}.",
        "Are we playing the same game, {playerName}? I don't see any hackers.",
        "{playerName}, that's a bold accusation. Any proof?",
        "Instead of blaming 'hacks', {playerName}, focus on your gameplay.",
        "Lol, {playerName}. Classic excuse.",
        "Not everyone better than you is a hacker, {playerName}.",
        "Keep crying '{playerName}', it won't make you better.",
        "Pretty sure that was just a good play, {playerName}.",
        "Maybe the 'hacker' is just {playerName} lagging?",
        "I think {playerName} needs a break from the game.",
        "{playerName}, did you check your internet connection first?",
        "Maybe they just have a better gaming chair, {playerName}?",
        "Or... hear me out {playerName}... they're just better than you.",
        "Accusations without proof, {playerName}? Tsk tsk.",
        "{playerName}, stop projecting your losses onto others.",
        "If everyone seems like a hacker, maybe the problem is you, {playerName}.",
        "Cope harder, {playerName}.",
        "Is 'hacker' the only word in your vocabulary, {playerName}?",
        "Wow, {playerName}, you died? Must be hacks, right?",
        "Don't worry {playerName}, we all have bad games. Doesn't mean hacks."
    }

    local function containsKeyword(messageText)
        local lowerMessage = string.lower(messageText)
        for _, keyword in ipairs(keywords) do
            if string.find(lowerMessage, string.lower(keyword)) then
                return true
            end
        end
        return false
    end

    local function sendReply(offendingPlayerName)
        if not _G.AutoToxicEnabled then return end
        
        local randomIndex = math.random(1, #replyTemplates)
        local chosenTemplate = replyTemplates[randomIndex]
        local replyMessage = string.gsub(chosenTemplate, "{playerName}", offendingPlayerName)

        local generalChannel = nil
        if TextChatService then
            local textChannels = TextChatService:WaitForChild("TextChannels", 5)
            if textChannels then
                generalChannel = textChannels:FindFirstChild("RBXGeneral")
            end
        end

        if generalChannel then
            pcall(function()
                generalChannel:SendAsync(replyMessage)
            end)
        end
    end

    if TextChatService and TextChatService.MessageReceived then
        TextChatService.MessageReceived:Connect(function(messageObject)
            if not _G.AutoToxicEnabled then return end
            if messageObject and messageObject.TextSource and messageObject.Text then
                local speakerUserId = messageObject.TextSource.UserId
                local speakerPlayer = Players:GetPlayerByUserId(speakerUserId)

                if speakerPlayer and speakerPlayer ~= localPlayer then
                    -- Add team check here
                    if not isEnemy(speakerPlayer) then return end -- Don't reply to teammates
                    
                    if containsKeyword(messageObject.Text) then
                        task.wait(math.random(1, 3))
                        sendReply(speakerPlayer.Name)
                    end
                end
            end
        end)
    else
        warn("TextChatService or TextChatService.MessageReceived not available. Falling back to Player.Chatted for detection.")

        local function handleChatted(player, messageText)
            if not _G.AutoToxicEnabled then return end
            if player ~= localPlayer then
                -- Add team check here
                if not isEnemy(player) then return end -- Don't reply to teammates
                
                if containsKeyword(messageText) then
                    task.wait(math.random(1, 3))
                    sendReply(player.Name)
                end
            end
        end

        Players.PlayerAdded:Connect(function(player)
            player.Chatted:Connect(function(messageText)
                handleChatted(player, messageText)
            end)
        end)

        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= localPlayer then
                player.Chatted:Connect(function(messageText)
                    handleChatted(player, messageText)
                end)
            end
        end
    end
end

-- Initialize the Auto Toxic system
pcall(initializeAutoToxic)

--// Input Handling for Spider Climb
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.UserInputType == Enum.UserInputType.Keyboard then
        keysDown[input.KeyCode] = true
    end

    -- Original CamLock input handling
    if input.KeyCode == toggleKey then
        if _G and _G.CameraLocked then -- Only proceed if CamLock is armed
            if camLockLockedTarget == nil then -- If not locked, try to lock
                camLockLockedTarget = findNearestLivingEnemyHRP()
                if camLockLockedTarget and camLockLockedTarget.Parent then
                    if Rayfield and Rayfield.Notify then
                        Rayfield:Notify({
                            Title = "CamLock",
                            Content = "Locked onto "..camLockLockedTarget.Parent.Name..". Press X to unlock.",
                            Duration = 3,
                            Type = "success"
                        })
                    end
                else
                    camLockLockedTarget = nil
                    if Rayfield and Rayfield.Notify then
                        Rayfield:Notify({
                            Title = "CamLock",
                            Content = "No target found to lock. Press X to try again.",
                            Duration = 3,
                            Type = "info"
                        })
                    end
                end
            else -- If already locked, unlock
                local oldTargetName = camLockLockedTarget.Parent and camLockLockedTarget.Parent.Name or "target"
                camLockLockedTarget = nil
                if Rayfield and Rayfield.Notify then
                    Rayfield:Notify({
                        Title = "CamLock",
                        Content = "Unlocked from "..oldTargetName..". Press X to lock new target.",
                        Duration = 3,
                        Type = "warning"
                    })
                end
            end
        else
            -- CamLock is not armed
            if Rayfield and Rayfield.Notify then
                Rayfield:Notify({
                    Title = "CamLock",
                    Content = "System is OFF. Enable CamLock via toggle to use.",
                    Duration = 3.5,
                    Type = "info"
                })
            end
        end
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.Keyboard then
        keysDown[input.KeyCode] = false
    end
end)

--// RunService Connections
RunService.Stepped:Connect(function()
    if _G and _G.ESPEnabled then pcall(updateESP) end
    if _G and _G.FlyEnabled then pcall(Fly) end
    if _G and _G.KillAuraEnabled then pcall(processKillAura) end
end)

RunService.RenderStepped:Connect(function()
    -- Original CamLock functionality
    if not (_G and _G.CameraLocked) or not camLockLockedTarget then
        -- Continue with other RenderStepped logic
    else
        -- Validate the currently locked target
        if camera and camLockLockedTarget.Parent and camLockLockedTarget:IsDescendantOf(workspace) then
            local targetCharacterModel = camLockLockedTarget.Parent
            local targetHumanoid = targetCharacterModel:FindFirstChildOfClass("Humanoid")

            if targetHumanoid and targetHumanoid.Health > 0 then
                -- Target is valid and alive, CFrame the camera
                camera.CFrame = CFrame.new(camera.CFrame.Position, camLockLockedTarget.Position)
            else
                -- Target is dead or humanoid missing
                if Rayfield and Rayfield.Notify then
                    Rayfield:Notify({
                        Title = "CamLock", 
                        Content = "Locked target lost. Press X to lock new target.", 
                        Duration = 3, 
                        Type = "warning"
                    })
                end
                camLockLockedTarget = nil -- Clear the target; system remains armed
            end
        else
            -- Target is no longer in workspace
            if Rayfield and Rayfield.Notify then
                Rayfield:Notify({
                    Title = "CamLock", 
                    Content = "Locked target lost. Press X to lock new target.", 
                    Duration = 3, 
                    Type = "warning"
                })
            end
            camLockLockedTarget = nil -- Clear the target; system remains armed
        end
    end
    
    -- Spider Climb functionality
    if _G and _G.SpiderEnabled then
        pcall(handleSpiderClimb)
    end
end)

RunService.Heartbeat:Connect(function()
    if humanoid and humanoid.Parent and _G then -- Check _G exists
        humanoid.WalkSpeed = (_G.SpeedLocked and 23 or 16)
    end
end)

--// GUI Construction
local Window = Rayfield:CreateWindow({
    Name = "Argan • Rewrite", Icon = 0, LoadingTitle = "Argan • Rewrite", LoadingSubtitle = "JustaRandomGuy, AGFX, voltzvoid",
    Theme = "Amethyst", DisableRayfieldPrompts = false, DisableBuildWarnings = false,
    ConfigurationSaving = { Enabled = true, FolderName = nil, FileName = "Argan" },
    Discord = { Enabled = true, Invite = "JEAY7M8gnW", RememberJoins = true },
    KeySystem = true, KeySettings = { Title = "Argan • Rewrite", Subtitle = "You need a key! ", Note = "Join the discord. discord.gg/JEAY7M8gnW", FileName = "ArganKEY", SaveKey = true, GrabKeyFromSite = false, Key = {"v3ishere"} }
})
Rayfield:Notify({ Title = "Argan • Rewrite", Content = "Interface Loaded.", Duration = 3.2, Image = "bell-ring", Type = "success" })

local CombatTab = Window:CreateTab("Combat", "swords")
local MISCTab = Window:CreateTab("MISC", "dices")
local CreditsTab = Window:CreateTab("Credits", "creative-commons")
local ThemesTab = Window:CreateTab("Themes", "palette")

CreditsTab:CreateSection("Credits :D")
CreditsTab:CreateLabel("Argan • Rewrite Was made fully by both JustaRandomGuy and AGFX. UI was made by JustaRandomGuy and functions made by AGFX.", "shield")
CreditsTab:CreateLabel(" We are NOT responsible for any bans that may happen to you. When executing this script you should know that what you are doing can and will ban you.", "shield")

ThemesTab:CreateSection("Themes")
local themes = {"Default", "AmberGlow", "Amethyst", "Bloom", "DarkBlue", "Green", "Light", "Ocean", "Serenity"}
for _, themeName in ipairs(themes) do
    ThemesTab:CreateButton({ Name = themeName == "Green" and "light-Green" or themeName, Callback = function() Window.ModifyTheme(themeName) end })
end

CombatTab:CreateSection("Main Stuff")
local ESPtoggle = CombatTab:CreateToggle({ Name = "ESP", CurrentValue = false, Flag = "ESPEnabledFlag", 
    Callback = function(Value) _G.ESPEnabled = Value; 
        Rayfield:Notify({Title = "ESP", Content = (_G.ESPEnabled and "ENABLED" or "DISABLED"), Duration = 2, Image = (_G.ESPEnabled and "check-circle" or "close-circle"), Type = (_G.ESPEnabled and "success" or "warning")});
        pcall(updateESP);
    end 
})
local infJtoggle = CombatTab:CreateToggle({ Name = "Infinite Jump", CurrentValue = false, Flag = "InfiniteJumpEnabledFlag", 
    Callback = function(Value) _G.InfiniteJumpEnabled = Value;
        Rayfield:Notify({Title = "Infinite Jump", Content = (_G.InfiniteJumpEnabled and "ENABLED" or "DISABLED"), Duration = 2, Image = (_G.InfiniteJumpEnabled and "check-circle" or "close-circle"), Type = (_G.InfiniteJumpEnabled and "success" or "warning")});
    end 
})
local camLockToggle = CombatTab:CreateToggle({ Name = "Cam Lock", CurrentValue = false, Flag = "CameraLockedFlag",
    Callback = function(Value) _G.CameraLocked = Value;
        if _G.CameraLocked then 
            camLockLockedTarget = nil 
            Rayfield:Notify({
                Title = "CamLock", 
                Content = "ARMED. Press X to lock onto nearest target.", 
                Duration = 3.5, 
                Image = "target", 
                Type = "info"
            })
        else 
            camLockLockedTarget = nil 
            Rayfield:Notify({
                Title = "CamLock", 
                Content = "DISABLED.", 
                Duration = 3, 
                Image = "close-circle", 
                Type = "default"
            })
        end
    end,
})
local KillAuraToggle = CombatTab:CreateToggle({Name = "Kill Aura", CurrentValue = false, Flag = "KillAuraEnabledFlag", 
    Callback = function(Value) _G.KillAuraEnabled = Value; if _G.KillAuraEnabled then lastKillAuraHitTime = 0 end;
        Rayfield:Notify({Title = "Kill Aura", Content = (_G.KillAuraEnabled and "ENABLED" or "DISABLED"), Duration = 2, Image = (_G.KillAuraEnabled and "check-circle" or "close-circle"), Type = (_G.KillAuraEnabled and "success" or "warning")});
    end 
})

MISCTab:CreateSection("Others")
local Speedtoggle = MISCTab:CreateToggle({ Name = "KeepSprint", CurrentValue = false, Flag = "SpeedLockedFlag", 
    Callback = function(Value) _G.SpeedLocked = Value; 
        if humanoid and humanoid.Parent then humanoid.WalkSpeed = (_G.SpeedLocked and 23 or 16) end;
        Rayfield:Notify({Title = "KeepSprint", Content = (_G.SpeedLocked and "ENABLED (Speed: 23)" or "DISABLED (Speed: 16)"), Duration = 2, Image = (_G.SpeedLocked and "check-circle" or "close-circle"), Type = (_G.SpeedLocked and "success" or "warning")});
    end 
})
local NoClipToggle = MISCTab:CreateToggle({ Name = "NoClip", CurrentValue = false, Flag = "NoClipEnabledFlag",
    Callback = function(Value) _G.NoClipEnabled = Value; 
        if typeof(ApplyNoClipState) == "function" then pcall(ApplyNoClipState) end;
        Rayfield:Notify({Title = "NoClip", Content = (_G.NoClipEnabled and "ENABLED" or "DISABLED"), Duration = 2, Image = (_G.NoClipEnabled and "check-circle" or "close-circle"), Type = (_G.NoClipEnabled and "success" or "warning")});
    end,
})
local FlyToggle = MISCTab:CreateToggle({ Name = "Fly", CurrentValue = false, Flag = "FlyEnabledFlag",
    Callback = function(Value) _G.FlyEnabled = Value; flyTimer_Active = false;
        Rayfield:Notify({Title = "Fly", Content = (_G.FlyEnabled and "ENABLED" or "DISABLED"), Duration = 2, Image = (_G.FlyEnabled and "check-circle" or "close-circle"), Type = (_G.FlyEnabled and "success" or "warning")});
        if not _G.FlyEnabled then
            if character then local hrp = character:FindFirstChild("HumanoidRootPart"); if hrp then local bv = hrp:FindFirstChild("ArganFlyVelocity"); if bv then bv:Destroy() end end end
            if player then local playerGui = player:FindFirstChild("PlayerGui"); if playerGui then local timerGui = playerGui:FindFirstChild("FlyTimerUI_Argan"); if timerGui then timerGui:Destroy() end end end
        end
    end,
})
local AutoToxicToggle = MISCTab:CreateToggle({
    Name = "Auto Toxic",
    CurrentValue = false,
    Flag = "AutoToxicEnabledFlag",
    Callback = function(Value)
        _G.AutoToxicEnabled = Value
        Rayfield:Notify({
            Title = "Auto Toxic", 
            Content = (_G.AutoToxicEnabled and "ENABLED" or "DISABLED"), 
            Duration = 2, 
            Image = (_G.AutoToxicEnabled and "check-circle" or "close-circle"), 
            Type = (_G.AutoToxicEnabled and "success" or "warning")
        })
    end
})
local SpiderToggle = MISCTab:CreateToggle({
    Name = "Spider Climb",
    CurrentValue = false,
    Flag = "SpiderEnabledFlag",
    Callback = function(Value)
        _G.SpiderEnabled = Value
        Rayfield:Notify({
            Title = "Spider Climb", 
            Content = (_G.SpiderEnabled and "ENABLED" or "DISABLED"), 
            Duration = 2, 
            Image = (_G.SpiderEnabled and "check-circle" or "close-circle"), 
            Type = (_G.SpiderEnabled and "success" or "warning")
        })
    end
})

-- Initialize global states (script logic variables) from toggle default values
_G.ESPEnabled = ESPtoggle.CurrentValue
_G.InfiniteJumpEnabled = infJtoggle.CurrentValue
_G.CameraLocked = camLockToggle.CurrentValue 
_G.KillAuraEnabled = KillAuraToggle.CurrentValue
_G.NoClipEnabled = NoClipToggle.CurrentValue
_G.SpeedLocked = Speedtoggle.CurrentValue
_G.FlyEnabled = FlyToggle.CurrentValue
_G.AutoToxicEnabled = AutoToxicToggle.CurrentValue
_G.SpiderEnabled = SpiderToggle.CurrentValue

-- Final initial state applications after _G vars are set
task.wait(0.1) -- Brief yield to ensure everything is settled
if typeof(ApplyNoClipState) == "function" then pcall(ApplyNoClipState) end
if typeof(updateESP) == "function" then pcall(updateESP) end
if humanoid and humanoid.Parent and _G then humanoid.WalkSpeed = (_G.SpeedLocked and 23 or 16) end
if not (_G and _G.FlyEnabled) then -- Ensure Fly cleanup if starting disabled
    if character then local hrp = character:FindFirstChild("HumanoidRootPart"); if hrp then local bv = hrp:FindFirstChild("ArganFlyVelocity"); if bv then bv:Destroy() end end end
    if player then local playerGui = player:FindFirstChild("PlayerGui"); if playerGui then local timerGui = playerGui:FindFirstChild("FlyTimerUI_Argan"); if timerGui then timerGui:Destroy() end end end
end

Rayfield:Notify({Title = "Argan", Content = "Fully Initialized!", Duration = 3, Type = "success"})
