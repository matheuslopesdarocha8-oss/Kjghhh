-- LocalScript: Botão de ServerHop arrastável (mobile + desktop)
-- Coloque este LocalScript em StarterPlayerScripts

local Players = game:GetService("Players")
local HttpService = game:GetService("HttpService")
local TeleportService = game:GetService("TeleportService")
local UserInputService = game:GetService("UserInputService")
local player = Players.LocalPlayer

local placeId = game.PlaceId
local currentJobId = tostring(game.JobId)
local DEBOUNCE = false

-- ---------- UI CREATION ----------
local playerGui = player:WaitForChild("PlayerGui")

local screenGui = Instance.new("ScreenGui")
screenGui.Name = "ServerHopGui"
screenGui.ResetOnSpawn = false
screenGui.Parent = playerGui

local container = Instance.new("Frame")
container.Name = "Container"
container.Size = UDim2.new(1, 0, 1, 0)
container.BackgroundTransparency = 1
container.Parent = screenGui

-- Status label (small feedback)
local status = Instance.new("TextLabel")
status.Name = "Status"
status.AnchorPoint = Vector2.new(0.5, 1)
status.Position = UDim2.new(0.9, 0, 0.9, -60)
status.Size = UDim2.new(0, 150, 0, 28)
status.BackgroundTransparency = 0.25
status.BackgroundColor3 = Color3.fromRGB(0,0,0)
status.TextColor3 = Color3.fromRGB(255,255,255)
status.Font = Enum.Font.SourceSansSemibold
status.TextSize = 14
status.Text = ""
status.Visible = false
status.Parent = container

-- The draggable button
local btn = Instance.new("TextButton")
btn.Name = "ServerHopButton"
btn.Size = UDim2.new(0, 88, 0, 88)
-- default bottom-right corner
btn.Position = UDim2.new(0.9, -44, 0.9, -44)
btn.AnchorPoint = Vector2.new(0.5, 0.5)
btn.BackgroundColor3 = Color3.fromRGB(66, 135, 245)
btn.AutoButtonColor = false
btn.Text = "HOP"
btn.TextColor3 = Color3.fromRGB(255,255,255)
btn.Font = Enum.Font.SourceSansBold
btn.TextSize = 20
btn.Parent = container

local uic = Instance.new("UICorner")
uic.CornerRadius = UDim.new(0, 999)
uic.Parent = btn

local inner = Instance.new("ImageLabel")
inner.Name = "Icon"
inner.Size = UDim2.new(0.55, 0, 0.55, 0)
inner.AnchorPoint = Vector2.new(0.5, 0.5)
inner.Position = UDim2.new(0.5, 0, 0.48, 0)
inner.BackgroundTransparency = 1
inner.Image = "" -- opcional: colocar ícone
inner.Parent = btn

-- ---------- HELPER: UI UTIL ----------
local function clamp(n, minV, maxV) return math.max(minV, math.min(maxV, n)) end
local function setStatus(text)
    if text == "" or not text then
        status.Visible = false
        status.Text = ""
    else
        status.Visible = true
        status.Text = text
    end
end

-- ---------- SERVER HOP LOGIC ----------
-- Busca até 100 amigos (primeira página)
local function getFriends(userId)
    local url = ("https://friends.roblox.com/v1/users/%d/friends"):format(userId)
    local ok, res = pcall(function() return HttpService:GetAsync(url) end)
    if not ok then
        warn("getFriends HTTP error:", res)
        return {}
    end
    local data = HttpService:JSONDecode(res)
    local ids = {}
    if data and data.data then
        for _, entry in ipairs(data.data) do
            if entry and entry.id then
                table.insert(ids, tostring(entry.id))
            end
        end
    end
    return ids
end

-- Post presence request for up to 100 ids per call
local function getPresenceForUsers(userIds)
    local map = {}
    if #userIds == 0 then return map end
    local url = "https://presence.roblox.com/v1/presence/users"
    local payload = HttpService:JSONEncode({ userIds = userIds })
    local ok, res = pcall(function() return HttpService:PostAsync(url, payload, Enum.HttpContentType.ApplicationJson) end)
    if not ok then
        warn("getPresenceForUsers HTTP error:", res)
        return map
    end
    local data = HttpService:JSONDecode(res)
    -- Common response has 'userPresences' array
    if data and data.userPresences then
        for _, p in ipairs(data.userPresences) do
            if p and p.userId then
                map[tostring(p.userId)] = { placeId = tostring(p.placeId or ""), gameId = tostring(p.gameId or "") }
            end
        end
        return map
    end
    -- Fallback: try array-style
    if type(data) == "table" then
        for _, p in ipairs(data) do
            if p and p.userId then
                map[tostring(p.userId)] = { placeId = tostring(p.placeId or ""), gameId = tostring(p.gameId or "") }
            end
        end
    end
    return map
end

-- Percorre servidores públicos e retorna instância com <=1 jog e sem amigos
local function getServerToJoin(friendServerIdsSet)
    local baseUrl = ("https://games.roblox.com/v1/games/%d/servers/Public?sortOrder=Asc&limit=100"):format(placeId)
    local nextUrl = baseUrl
    while nextUrl do
        local ok, body = pcall(function() return HttpService:GetAsync(nextUrl) end)
        if not ok then
            return nil, "Erro HTTP ao listar servidores: "..tostring(body)
        end
        local data = HttpService:JSONDecode(body)
        if data and data.data then
            for _, server in ipairs(data.data) do
                if server and server.id then
                    local sid = tostring(server.id)
                    local playing = tonumber(server.playing) or 0
                    if sid ~= currentJobId and playing <= 1 then
                        if not (friendServerIdsSet and friendServerIdsSet[sid]) then
                            return sid
                        end
                    end
                end
            end
        end
        if data and data.nextPageCursor then
            local cursor = data.nextPageCursor
            nextUrl = ("https://games.roblox.com/v1/games/%d/servers/Public?limit=100&cursor=%s"):format(placeId, HttpService:UrlEncode(cursor))
        else
            nextUrl = nil
        end
        wait(0.12) -- gentil com a API
    end
    return nil, "Nenhum servidor disponível (0/1) sem amigos"
end

local function serverHopAvoidFriends()
    if DEBOUNCE then return end
    DEBOUNCE = true
    setStatus("Buscando amigos...")
    local friendIds = getFriends(player.UserId) -- até 100 por chamada
    setStatus("Verificando presença...")
    local friendPresenceMap = {}
    if #friendIds > 0 then
        -- chunk 100 (amigos até 100 normalmente)
        local i = 1
        while i <= #friendIds do
            local chunk = {}
            for j = i, math.min(i + 99, #friendIds) do
                table.insert(chunk, tonumber(friendIds[j]))
            end
            local part = getPresenceForUsers(chunk)
            for k,v in pairs(part) do friendPresenceMap[k] = v end
            i = i + 100
            wait(0.08)
        end
    end

    -- Constrói set de gameIds onde amigos estão no mesmo place
    local friendServerIdsSet = {}
    for uid, pres in pairs(friendPresenceMap) do
        if pres and pres.placeId and tostring(pres.placeId) == tostring(placeId) and pres.gameId and pres.gameId ~= "" then
            friendServerIdsSet[tostring(pres.gameId)] = true
        end
    end

    setStatus("Procurando servidor...")
    local serverId, err = getServerToJoin(friendServerIdsSet)
    if not serverId then
        warn("ServerHop:", err)
        setStatus("Nenhum servidor encontrado")
        wait(2)
        setStatus("")
        DEBOUNCE = false
        return
    end

    setStatus("Teleportando...")
    local ok, teleErr = pcall(function()
        TeleportService:TeleportToPlaceInstance(placeId, serverId, player)
    end)
    if not ok then
        warn("Teleport falhou:", teleErr)
        setStatus("Teleport falhou")
        wait(2)
        setStatus("")
    end

    DEBOUNCE = false
end

-- ---------- DRAG & TAP HANDLING (mobile + desktop) ----------
local dragging = false
local dragInput = nil
local dragStart = nil
local startPos = nil
local longPressThreshold = 0.18 -- segundos até considerar "segurando" (ajustável)

-- Usa InputChanged global para atualizar posição durante o drag
UserInputService.InputChanged:Connect(function(input)
    if not dragInput or input ~= dragInput then return end
    if not dragging then return end
    local delta = input.Position - dragStart
    local newX = startPos.X.Offset + delta.X
    local newY = startPos.Y.Offset + delta.Y
    local cam = workspace.CurrentCamera
    if not cam then return end
    local vw, vh = cam.ViewportSize.X, cam.ViewportSize.Y
    newX = clamp(newX, 0, vw - btn.AbsoluteSize.X)
    newY = clamp(newY, 0, vh - btn.AbsoluteSize.Y)
    btn.Position = UDim2.new(0, newX, 0, newY)
end)

btn.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragInput = input
        dragStart = input.Position
        startPos = btn.Position
        local startTick = tick()
        -- start a short timer to enable dragging if still holding
        delay(longPressThreshold, function()
            if dragInput == input and not dragging then
                -- activate dragging mode
                dragging = true
                setStatus("Arrastando")
                btn.BackgroundColor3 = Color3.fromRGB(50,120,230)
            end
        end)
        -- Listen for end of this input
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                -- ended touch/click
                if dragging then
                    -- finish dragging, just clear
                    dragging = false
                    dragInput = nil
                    setStatus("")
                    btn.BackgroundColor3 = Color3.fromRGB(66, 135, 245)
                else
                    -- it was a short tap: trigger server hop
                    -- small animation feedback
                    spawn(function()
                        btn.BackgroundColor3 = Color3.fromRGB(40,100,200)
                        setStatus("Procurando servidor...")
                        serverHopAvoidFriends()
                        wait(0.15)
                        btn.BackgroundColor3 = Color3.fromRGB(66, 135, 245)
                        setStatus("")
                    end)
                    dragInput = nil
                end
            end
        end)
    end
end)

-- Also handle if the input is interrupted (sudden disconnect)
UserInputService.InputEnded:Connect(function(input)
    if input == dragInput and dragging then
        dragging = false
        dragInput = nil
        setStatus("")
        btn.BackgroundColor3 = Color3.fromRGB(66, 135, 245)
    end
end)

-- Optional: botão de reset de posição com duplo-tap
local lastTap = 0
btn.MouseButton1Click:Connect(function()
    local now = tick()
    if now - lastTap < 0.35 then
        -- double-tap: reset to bottom-right
        btn.Position = UDim2.new(0.9, -44, 0.9, -44)
        setStatus("Posição resetada")
        delay(1, function() setStatus("") end)
    end
    lastTap = now
end)

-- ---------- END ----------
setStatus("")
