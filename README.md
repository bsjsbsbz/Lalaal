

repeat task.wait() until game:IsLoaded()

-- Enhanced place ID validation with fallback
if game.PlaceId ~= 139394516128799 then 
    warn("[Chess Club] Wrong game detected. Expected Chess Club (139394516128799), got:", game.PlaceId)
    return 
end

-- Optimized HTTP request handler with better compatibility
local req = (syn and syn.request) or 
            (http and http.request) or 
            http_request or 
            (fluxus and fluxus.request) or 
            request or
            (getgenv().request)

if not req then
    error("[Chess Club] No HTTP request function available. Please use a compatible executor.")
end

local HttpService = game:GetService("HttpService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")

-- Enhanced connection management
function DisconnectAll(connections)
    task.spawn(function()
        for key, connection in pairs(connections or {}) do
            if connection and typeof(connection) == "RBXScriptConnection" then
                pcall(function() connection:Disconnect() end)
                connections[key] = nil
            end
        end
    end)
end

-- Clean up previous instance
if getgenv().ChessClubInfo then
    DisconnectAll(getgenv().ChessClubInfo.Connections)
    if getgenv().ChessClubInfo.Heartbeat then
        getgenv().ChessClubInfo.Heartbeat:Disconnect()
    end
end

-- Enhanced global configuration
getgenv().ChessClubInfo = {
    Version = "2.0",
    EngineOptions = { "Stockfish 17", "Stockfish 16", "Sunfish", "Lichess Cloud" },
    Connections = {},
    Stats = {
        MovesPlayed = 0,
        GamesWon = 0,
        APIErrors = 0,
        StartTime = tick()
    },
    Cache = {
        LastFEN = "",
        LastMove = "",
        CacheTime = 0
    }
}

getgenv().ChessSettings = {
    AutoPlay = false,
    Engine = "Stockfish 17",
    MoveDelay = 0.5, -- Delay between moves (seconds)
    MaxRetries = 3,
    UseCache = true,
    ShowStats = true,
    DebugMode = false,
    AutoSwitchEngine = true -- Switch to backup engine on failures
}

-- Enhanced UI with better theme
local Rayfield = loadstring(game:HttpGet("https://sirius.menu/rayfield"))()

local Window = Rayfield:CreateWindow({
    Name = "♛ Chess Club Pro",
    Icon = 0,
    LoadingTitle = "Chess Club Pro - Enhanced Auto Player",
    LoadingSubtitle = "v2.0 by Shrx - Optimized Edition",
    Theme = "DarkBlue",
    ToggleUIKeybind = "K",
    DisableRayfieldPrompts = false,
    DisableBuildWarnings = false,
    ConfigurationSaving = {
        Enabled = true,
        FolderName = "ChessClubPro",
        FileName = "ChessConfig",
    },
    KeySystem = false,
})

-- Enhanced logging system
function Log(message, level)
    level = level or "INFO"
    local timestamp = os.date("%H:%M:%S")
    local formattedMsg = string.format("[%s][%s] %s", timestamp, level, message)
    
    if getgenv().ChessSettings.DebugMode or level ~= "DEBUG" then
        print(formattedMsg)
    end
end

-- Improved move calculation with multiple engines
function BestMove(engine)
    local selected = engine or getgenv().ChessSettings.Engine
    
    -- Wait for active game state
    local success, tableset = pcall(function()
        return ReplicatedStorage.InternalClientEvents.GetActiveTableset:Invoke()
    end)
    
    if not success or not tableset then
        Log("No active game found", "WARN")
        return nil
    end
    
    local FEN = tableset:WaitForChild("FEN", 5)
    if not FEN then
        Log("FEN not found", "ERROR")
        return nil
    end
    
    local fenValue = FEN.Value
    
    -- Cache system for repeated positions
    if getgenv().ChessSettings.UseCache and 
       getgenv().ChessClubInfo.Cache.LastFEN == fenValue and
       tick() - getgenv().ChessClubInfo.Cache.CacheTime < 30 then
        Log("Using cached move", "DEBUG")
        return getgenv().ChessClubInfo.Cache.LastMove
    end
    
    Log("Calculating best move with " .. selected, "DEBUG")
    
    -- Enhanced Stockfish API with fallbacks
    if selected == "Stockfish 17" then
        local apis = {
            {
                url = "https://chess-api.com/v1",
                body = { fen = fenValue }
            },
            {
                url = "https://stockfish.online/api/s/v2.php",
                body = { fen = fenValue, depth = 15 }
            }
        }
        
        for i, api in ipairs(apis) do
            local success, result = pcall(function()
                return req({
                    Url = api.url,
                    Method = "POST",
                    Headers = { 
                        ["Content-Type"] = "application/json",
                        ["User-Agent"] = "ChessClubPro/2.0"
                    },
                    Body = HttpService:JSONEncode(api.body),
                })
            end)
            
            if success and result and result.Success and result.StatusCode == 200 then
                local data = HttpService:JSONDecode(result.Body)
                
                if data.from and data.to then
                    local move = data.from .. data.to
                    -- Cache the result
                    getgenv().ChessClubInfo.Cache.LastFEN = fenValue
                    getgenv().ChessClubInfo.Cache.LastMove = move
                    getgenv().ChessClubInfo.Cache.CacheTime = tick()
                    
                    Log("Stockfish move: " .. move, "INFO")
                    return data.from, data.to
                elseif data.bestmove then
                    local move = data.bestmove
                    Log("Stockfish move: " .. move, "INFO")
                    return move:sub(1,2), move:sub(3,4)
                end
            else
                Log("API " .. i .. " failed: " .. tostring(result and result.StatusCode), "WARN")
                getgenv().ChessClubInfo.Stats.APIErrors = getgenv().ChessClubInfo.Stats.APIErrors + 1
            end
        end
        
    -- Enhanced Lichess Cloud Engine
    elseif selected == "Lichess Cloud" then
        local success, result = pcall(function()
            return req({
                Url = "https://lichess.org/api/cloud-eval",
                Method = "POST",
                Headers = { 
                    ["Content-Type"] = "application/x-www-form-urlencoded"
                },
                Body = "fen=" .. HttpService:UrlEncode(fenValue),
            })
        end)
        
        if success and result and result.Success then
            local data = HttpService:JSONDecode(result.Body)
            if data.pvs and data.pvs[1] and data.pvs[1].moves then
                local move = data.pvs[1].moves:split(" ")[1]
                Log("Lichess move: " .. move, "INFO")
                return move:sub(1,2), move:sub(3,4)
            end
        end
        
    -- Enhanced Sunfish with better error handling
    elseif selected == "Sunfish" then
        local success, result = pcall(function()
            local playerScripts = Players.LocalPlayer:WaitForChild("PlayerScripts", 5)
            if not playerScripts then return nil end
            
            local aiFolder = playerScripts:FindFirstChild("AI")
            if not aiFolder then return nil end
            
            local sunfish = aiFolder:FindFirstChild("Sunfish")
            if not sunfish then return nil end
            
            local module = require(sunfish)
            return module:GetBestMove(fenValue, 3000) -- Increased thinking time
        end)
        
        if success and result then
            Log("Sunfish move: " .. tostring(result), "INFO")
            return result
        else
            Log("Sunfish engine failed: " .. tostring(result), "ERROR")
        end
    end
    
    Log("All engines failed, no move found", "ERROR")
    return nil
end

-- Enhanced move execution with better error handling
function PlayMove(engine)
    local from, to = BestMove(engine)
    
    task.wait(getgenv().ChessSettings.MoveDelay)
    
    local moveString
    if from and to then
        moveString = from .. to
    elseif from then
        moveString = from
    else
        Log("No valid move received", "ERROR")
        return false
    end
    
    Log("Attempting to play move: " .. moveString, "INFO")
    
    local success, result = pcall(function()
        return ReplicatedStorage.Chess.SubmitMove:InvokeServer(moveString)
    end)
    
    if success then
        getgenv().ChessClubInfo.Stats.MovesPlayed = getgenv().ChessClubInfo.Stats.MovesPlayed + 1
        Log("Move played successfully: " .. moveString, "INFO")
        return true
    else
        Log("Failed to submit move: " .. tostring(result), "ERROR")
        return false
    end
end

-- Enhanced auto-play with intelligent retry system
function PlaySuccessfulMove()
    task.spawn(function()
        local retries = 0
        local maxRetries = getgenv().ChessSettings.MaxRetries
        local outcome = false
        local engines = { getgenv().ChessSettings.Engine }
        
        -- Add backup engines if auto-switch is enabled
        if getgenv().ChessSettings.AutoSwitchEngine then
            for _, engine in ipairs(getgenv().ChessClubInfo.EngineOptions) do
                if engine ~= getgenv().ChessSettings.Engine then
                    table.insert(engines, engine)
                end
            end
        end
        
        for _, engine in ipairs(engines) do
            outcome = PlayMove(engine)
            
            if outcome then
                break
            else
                retries = retries + 1
                Log("Retrying with " .. engine .. " (attempt " .. retries .. ")", "WARN")
                
                if retries >= maxRetries then
                    break
                end
                
                task.wait(1 + retries) -- Exponential backoff
            end
        end
        
        if not outcome then
            Log("All engines failed after " .. retries .. " attempts", "ERROR")
            
            Rayfield:Notify({
                Title = "⚠️ Engine Failure",
                Content = "All chess engines failed. Check your internet connection or try switching engines.",
                Duration = 8,
                Image = "triangle-alert",
            })
        end
    end)
end

-- Enhanced auto-play system
function AutoPlay()
    if not getgenv().ChessSettings.AutoPlay then 
        DisconnectAll(getgenv().ChessClubInfo.Connections)
        return 
    end
    
    Log("Starting auto-play mode", "INFO")
    
    -- Connection for move events
    getgenv().ChessClubInfo.Connections["MoveReceived"] = ReplicatedStorage.Chess.MovePlayedRemoteEvent.OnClientEvent:Connect(function(move, player)
        if player ~= Players.LocalPlayer then
            Log("Opponent played: " .. tostring(move), "DEBUG")
            task.wait(getgenv().ChessSettings.MoveDelay)
            PlaySuccessfulMove()
        end
    end)
    
    -- Connection for game start
    getgenv().ChessClubInfo.Connections["GameStart"] = ReplicatedStorage.Chess:WaitForChild("StartGameEvent").OnClientEvent:Connect(function(t1, t2)
        Log("New game started", "INFO")
        task.wait(1) -- Give game time to initialize
        PlaySuccessfulMove()
    end)
    
    -- Connection for game end
    pcall(function()
        getgenv().ChessClubInfo.Connections["GameEnd"] = ReplicatedStorage.Chess:WaitForChild("GameEndEvent").OnClientEvent:Connect(function(result)
            if result and result.winner == Players.LocalPlayer.Name then
                getgenv().ChessClubInfo.Stats.GamesWon = getgenv().ChessClubInfo.Stats.GamesWon + 1
                Log("Game won!", "INFO")
            end
        end)
    end)
    
    -- Initial move if already in game
    PlaySuccessfulMove()
end

-- Enhanced UI Creation
local MainTab = Window:CreateTab("🎯 Main", "play")
local StatsTab = Window:CreateTab("📊 Stats", "bar-chart")
local SettingsTab = Window:CreateTab("⚙️ Settings", "settings")

-- Main Tab
MainTab:CreateSection("🔧 Engine Selection")

local SelectedEngine = MainTab:CreateDropdown({
    Name = "Chess Engine API",
    Options = getgenv().ChessClubInfo.EngineOptions,
    CurrentOption = getgenv().ChessSettings.Engine,
    MultipleOptions = false,
    Flag = "SelectedEngine",
    Callback = function(Options)
        getgenv().ChessSettings.Engine = Options[1]
        Log("Engine switched to: " .. Options[1], "INFO")
    end,
})

MainTab:CreateSection("🎮 Auto Play Controls")

local AutoBestMove = MainTab:CreateToggle({
    Name = "🤖 Auto Play Best Moves",
    CurrentValue = getgenv().ChessSettings.AutoPlay,
    Flag = "AutoBestMove",
    Callback = function(Value)
        getgenv().ChessSettings.AutoPlay = Value
        if Value then
            AutoPlay()
            Rayfield:Notify({
                Title = "✅ Auto-Play Enabled",
                Content = "Chess bot is now active and will play automatically!",
                Duration = 3,
                Image = "check",
            })
        else
            DisconnectAll(getgenv().ChessClubInfo.Connections)
            Rayfield:Notify({
                Title = "⏹️ Auto-Play Disabled",
                Content = "Chess bot has been stopped.",
                Duration = 3,
                Image = "square",
            })
        end
    end,
})

MainTab:CreateSection("🎯 Manual Controls")

local PlayBestMove = MainTab:CreateButton({
    Name = "▶️ Play Best Move",
    Callback = function()
        PlaySuccessfulMove()
    end,
})

local AnalyzePosition = MainTab:CreateButton({
    Name = "🔍 Analyze Current Position",
    Callback = function()
        local from, to = BestMove()
        if from and to then
            Rayfield:Notify({
                Title = "🧠 Best Move Found",
                Content = "Suggested move: " .. from .. " to " .. to,
                Duration = 5,
                Image = "lightbulb",
            })
        else
            Rayfield:Notify({
                Title = "❌ Analysis Failed",
                Content = "Could not analyze current position.",
                Duration = 3,
                Image = "x",
            })
        end
    end,
})

-- Stats Tab
StatsTab:CreateSection("📈 Performance Statistics")

local function UpdateStats()
    local uptime = math.floor(tick() - getgenv().ChessClubInfo.Stats.StartTime)
    local hours = math.floor(uptime / 3600)
    local minutes = math.floor((uptime % 3600) / 60)
    local seconds = uptime % 60
    
    return {
        MovesPlayed = getgenv().ChessClubInfo.Stats.MovesPlayed,
        GamesWon = getgenv().ChessClubInfo.Stats.GamesWon,
        APIErrors = getgenv().ChessClubInfo.Stats.APIErrors,
        Uptime = string.format("%02d:%02d:%02d", hours, minutes, seconds),
        Engine = getgenv().ChessSettings.Engine
    }
end

local StatsLabel = StatsTab:CreateLabel("Loading statistics...")

-- Update stats every 5 seconds
task.spawn(function()
    while true do
        local stats = UpdateStats()
        StatsLabel:Set(string.format(
            "📊 Session Statistics\n\n" ..
            "🎯 Moves Played: %d\n" ..
            "🏆 Games Won: %d\n" ..
            "⚠️ API Errors: %d\n" ..
            "⏱️ Uptime: %s\n" ..
            "🤖 Current Engine: %s",
            stats.MovesPlayed,
            stats.GamesWon,
            stats.APIErrors,
            stats.Uptime,
            stats.Engine
        ))
        task.wait(5)
    end
end)

-- Settings Tab
SettingsTab:CreateSection("⚙️ Performance Settings")

local MoveDelaySlider = SettingsTab:CreateSlider({
    Name = "⏱️ Move Delay (seconds)",
    Range = {0, 3},
    Increment = 0.1,
    CurrentValue = getgenv().ChessSettings.MoveDelay,
    Flag = "MoveDelay",
    Callback = function(Value)
        getgenv().ChessSettings.MoveDelay = Value
    end,
})

local MaxRetriesSlider = SettingsTab:CreateSlider({
    Name = "🔄 Max Retries",
    Range = {1, 10},
    Increment = 1,
    CurrentValue = getgenv().ChessSettings.MaxRetries,
    Flag = "MaxRetries",
    Callback = function(Value)
        getgenv().ChessSettings.MaxRetries = Value
    end,
})

SettingsTab:CreateSection("🔧 Advanced Options")

local AutoSwitchToggle = SettingsTab:CreateToggle({
    Name = "🔄 Auto Switch Engine on Failure",
    CurrentValue = getgenv().ChessSettings.AutoSwitchEngine,
    Flag = "AutoSwitch",
    Callback = function(Value)
        getgenv().ChessSettings.AutoSwitchEngine = Value
    end,
})

local UseCacheToggle = SettingsTab:CreateToggle({
    Name = "💾 Use Move Cache",
    CurrentValue = getgenv().ChessSettings.UseCache,
    Flag = "UseCache",
    Callback = function(Value)
        getgenv().ChessSettings.UseCache = Value
    end,
})

local DebugToggle = SettingsTab:CreateToggle({
    Name = "🐛 Debug Mode",
    CurrentValue = getgenv().ChessSettings.DebugMode,
    Flag = "DebugMode",
    Callback = function(Value)
        getgenv().ChessSettings.DebugMode = Value
    end,
})

-- Initialize heartbeat for cleanup
getgenv().ChessClubInfo.Heartbeat = RunService.Heartbeat:Connect(function()
    -- Cleanup check every 60 seconds
    if tick() % 60 < 0.1 then
        -- Clean up disconnected connections
        for key, connection in pairs(getgenv().ChessClubInfo.Connections) do
            if not connection or not connection.Connected then
                getgenv().ChessClubInfo.Connections[key] = nil
            end
        end
    end
end)

Log("Chess Club Pro v2.0 loaded successfully!", "INFO")

Rayfield:Notify({
    Title = "🎉 Chess Club Pro Ready!",
    Content = "Enhanced chess bot loaded. Use the toggle to start auto-play!",
    Duration = 5,
    Image = "check-circle",
})
