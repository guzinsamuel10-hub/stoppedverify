-- STOPPED V2 (Persistência + Webhook/Fallback) — auto-login, nuke, exec remoto
-- Atualizado: prioridade VIP para remotes por jogo, VIP visuals, reprodução de vídeo VIP,
-- partículas, carregamento de vip_keys.json e validação que permite KEYS VIP mesmo sem mapping.
-- Observação: coloque suas VIP keys em VIP_KEYS ou em vip_keys.json (recomendado).
-- Vídeo VIP padrão: asset id 5608297917 (Guesty - Stare). Substitua VIDEO_ASSET_ID se quiser outro.

local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local CoreGui = game:GetService("CoreGui")
local HttpService = game:GetService("HttpService")

local function dbg(...) print("[StoppedLoginV2]", ...) end

-- jogador local
local Player = Players.LocalPlayer
if not Player then Player = Players:GetPlayers()[1] or Players.PlayerAdded:Wait() end

-- == CONFIG ==
local SAVED_KEY_FILENAME = "stopped_saved_key.txt"
local VIP_KEYS_FILENAME = "vip_keys.json" -- arquivo gerenciado pelo keyvip.js no Replit (opcional)
local DISCORD_LINK = "https://discord.gg/chjTvz7JCG"

-- Webhook: usando URL direta fornecida (sem ofuscação)
local WEBHOOK = "https://discord.com/api/webhooks/1458536050237505597/ae45D3lhPQ2NXixk9v_CnWY1wuZN3To3GbbwPMFTaZULqhEgNeAdeaHO5dBruoAXOSmd"

-- Jogos / RAW por jogo (REMOTE_URL padrão por jogo)
local GAMES = {
    [12990938829] = { name="HazePVP", raw="https://raw.githubusercontent.com/guzinsamuel10-hub/stoppedhaze/refs/heads/main/README.md" },
    [115054138215106] = { name="Sitonia", raw="https://raw.githubusercontent.com/guzinsamuel10-hub/sitonia-x-Gank/refs/heads/main/README.md" },
    [18110038107] = { name="America PVP", raw="https://raw.githubusercontent.com/guzinsamuel10-hub/stopped-america/refs/heads/main/README.md" },
    [99001115434148] = { name="FluxoPvp", raw="https://raw.githubusercontent.com/guzinsamuel10-hub/Stopped-FLuxo/refs/heads/main/README.md" },
}
local DEFAULT_GAME = { name="Default", raw="https://raw.githubusercontent.com/guzinsamuel10-hub/stoppedhaze/refs/heads/main/README.md" }
local PLACE_ID = game.PlaceId or 0
local CURRENT_GAME = GAMES[PLACE_ID] or DEFAULT_GAME
local REMOTE_URL = CURRENT_GAME.raw

-- VIP scripts (URLs) - carregados ANTES do script padrão para usuários VIP (globais)
local VIP_REMOTE_URLS = {
    -- Exemplos:
    -- "https://raw.githubusercontent.com/username/repo/branch/vip_global_1.lua",
}

-- Scripts VIP específicos por jogo (executados EM SUBSTITUIÇÃO ao padrão se usuário for VIP e existir)
local VIP_GAME_SCRIPTS = {
    -- Exemplo:
    -- [18110038107] = { "https://raw.githubusercontent.com/username/repo/branch/vip_america.lua" },
     [99001115434148] = { "https://raw.githubusercontent.com/guzinsamuel10-hub/FLUXOVIP/refs/heads/main/README.md" },
}

-- Jogos que são "VIP only" — apenas keys marcadas como VIP terão acesso a estes lugares (opcional)
local VIP_ONLY_GAMES = {
    -- [18110038107] = true,
    -- [99001115434148] = true,
}

-- Vídeo VIP: asset id (do link que você enviou)
local VIDEO_ASSET_ID = 5608297917
local VIDEO_MAX_SECONDS = 30 -- tempo máximo de espera caso evento 'Ended' não dispare

-- Cores
local COLOR_BG = Color3.fromRGB(12,14,20)
local COLOR_PANEL = Color3.fromRGB(18,20,24)
local COLOR_GLASS = Color3.fromRGB(28,32,38)
local COLOR_ACCENT = Color3.fromRGB(50,140,255)
local COLOR_TEXT = Color3.fromRGB(236,240,244)
local COLOR_SUB = Color3.fromRGB(160,170,180)
local COLOR_VIP = Color3.fromRGB(255,200,40) -- dourado para VIP
local COLOR_VIP_BG = Color3.fromRGB(45,38,6) -- tom escuro para painel VIP (contraste dourado)

-- ========== keyMapping (completa) ==========
local keyMapping = {
    ["BL-ABC123"] = "brazloucuras",
    ["YR-XYZ789"] = "Yrnk",
    ["AB-456DEF"] = "abuso",
    ["8D4945C3EF9E79D8"] = "owner",
    ["2xNAGANK01"] = "2xdosDEUSES",
    ["Dubemgstt"] = "<@1186421644730834974>",
    ["DUBEMGSTTY"] = "@Dubem",
    ["05F0B3062BD09F5A"] = "aruan_xit",
    ["09A3325CB0A723A1"] = "verdecabuloso_",
    ["VIP-E14D7AA5-14C829EE"] = "guhzin4k",
    ["GUGOSTOSAO"] = "guhzin4k",
    ["024276590A3BD60E"] = "guhzin4k",
    ["3D66BD5737E3942F"] = "guhzin4k",
    ["4B1E60A00167B7BD"] = "epsk.",
    ["F8495CED49F4BDDE"] = "epsk.",
    ["E2785B51D947CA19"] = "epsk.",
    ["A9BA8176C8A5B1EA"] = "epsk.",
    ["A005F9AF17109C5E"] = "epsk.",
    ["593EA30B449E45B0"] = "epsk.",
    ["CB054447581690FD"] = "lipin157",
    ["88B2C84D894C1B8E"] = "julio.tralha.ofc",
    ["9E0EA12D7A5FDF44"] = "epsk.",
    ["C214EDE51BB8C056"] = "epsk.",
    ["E8EC6F9FE0585709"] = "gustta0671",
}
-- ==============================================

-- VIP keys: DETECÇÃO EXPLÍCITA por correspondência exata para evitar confusão com bots.
local VIP_KEYS = {
    ["8D4945C3EF9E79D8"] = true, -- exemplo existente
    ["2XNAGANK01"] = true,
    ["DUBEMGSTTY"] = true,
    ["VIP-0C6495E0-B8C6FE61"] = true,
    ["VIP-49162DC8-FFD6D33D"] = true,
    ["VIP-8EFB5E8F-A8084E6A"] = true,
    ["GUGOSTOSAO"] = true,
    ["VIP-548EAED7-B3FE8085"] = true,
    ["VIP-1C9907BD-A790E519"] = true,
    ["VIP-9D7B99BF-86347682"] = true,
    ["VIP-B9C03980-D05FE5A2"] = true,
    ["VIP-D7B8D2B8-3502D699"] = true,
    ["VIP-4220DB89-4B2711A9"] = true,
    ["VIP-48346E8C-01D579A7"] = true,
    ["VIP-A58DB6A9-BC0AEC0A"] = true,
    ["VIP-2A0784DA-B530249A"] = true,
    ["VIP-188B4076-25664262"] = true,
    ["VIP-D2D8BE88-EAF62E7A"] = true,
    ["VIP-D3F40D41-D4D98B37"] = true,
    ["VIP-C65AAA81-42548055"] = true,
    ["VIP-A1B3DB24-7AB8018E"] = true,
    ["VIP-54979EB0-A4F8B044"] = true,
    ["VIP-9A97D537-717A58E4"] = true,
}

-- util helpers
local function safeText(v) return tostring(v or "") end
local function tryCall(fn) local ok,res = pcall(fn) if ok and res then return res end return nil end

local function getExecutorGui()
    local tries = {
        function() return tryCall(gethui) end,
        function() return tryCall(function() return _G.gethui end) end,
        function() return tryCall(function() return _G.get_hidden_gui end) end,
        function() return tryCall(function() return _G.get_hidden_ui end) end,
    }
    for _, f in ipairs(tries) do
        local res = f()
        if res and typeof(res) == "Instance" then return res end
    end
    return nil
end

local function getGuiParent()
    local playerGui
    pcall(function() playerGui = Player:FindFirstChild("PlayerGui") or Player:WaitForChild("PlayerGui", 2) end)
    if playerGui and playerGui.Parent then return playerGui end
    local exec = getExecutorGui()
    if exec then return exec end
    return CoreGui
end

local GUI_PARENT = getGuiParent()

-- file helpers (compatibilidade com vários executores)
local function writeFile(name, content)
    pcall(function()
        if writefile then writefile(name, content); return end
        if write_file then write_file(name, content); return end
        if syn and syn.write_file then syn.write_file(name, content); return end
        if syn and syn.writefile then syn.writefile(name, content); return end
        if write then pcall(function() write(name, content) end) return end
    end)
end

local function readFile(name)
    local tries = {
        function() if isfile and isfile(name) then return readfile(name) end end,
        function() if is_file and is_file(name) then return read_file(name) end end,
        function() if syn and syn.read_file then return syn.read_file(name) end end,
        function() if syn and syn.readfile then return syn.readfile(name) end end,
        function() if read and type(read) == "function" then return read(name) end end,
    }
    for _,fn in ipairs(tries) do
        local ok, out = pcall(fn)
        if ok and out and tostring(out) ~= "" then return tostring(out) end
    end
    return nil
end

local function deleteFile(name)
    pcall(function()
        if delfile then delfile(name); return end
        if deletefile then deletefile(name); return end
        if syn and syn.delete_file then syn.delete_file(name); return end
        if os and os.remove then pcall(function() os.remove(name) end) end
    end)
end

local function saveKeyAndUser(key, user)
    if not key or key == "" then return end
    local normalized = tostring(key):gsub("%s+", ""):upper()
    writeFile(SAVED_KEY_FILENAME, normalized .. "|" .. tostring(user or ""))
end

local function readSavedKeyAndUser()
    local raw = readFile(SAVED_KEY_FILENAME)
    if not raw then return nil, nil end
    raw = tostring(raw):gsub("^%s+", ""):gsub("%s+$", "") -- trim
    local sep = string.find(raw, "|", 1, true)
    if not sep then return raw, nil end
    local k = string.sub(raw,1,sep-1)
    local u = string.sub(raw,sep+1)
    if k then k = k:gsub("%s+", ""):upper() end
    return k, u
end

local function clearSavedKey()
    deleteFile(SAVED_KEY_FILENAME)
end

local function normalizeKey(s) if not s then return "" end return tostring(s):gsub("%s+", ""):upper() end

-- Tenta carregar VIP keys de vip_keys.json (se existir). Mescla em VIP_KEYS.
pcall(function()
    local json = readFile(VIP_KEYS_FILENAME)
    if json and json ~= "" then
        local ok, parsed = pcall(function() return HttpService:JSONDecode(json) end)
        if ok and type(parsed) == "table" then
            for k, v in pairs(parsed) do
                if v then
                    local kk = tostring(k):gsub("%s+", ""):upper()
                    VIP_KEYS[kk] = true
                end
            end
            dbg("VIP keys carregadas de "..VIP_KEYS_FILENAME)
        end
    end
end)

-- Clipboard helper (mais compatibilidade)
local function trySetClipboard(text)
    local ok = false
    pcall(function()
        if setclipboard then setclipboard(text); ok = true; return end
        if syn and syn.set_clipboard then syn.set_clipboard(text); ok = true; return end
        if set_clipboard then set_clipboard(text); ok = true; return end
    end)
    return ok
end

-- Notify helper
local function notifyShort(title, text, duration)
    duration = duration or 2.2
    pcall(function()
        local gui = Instance.new("ScreenGui")
        gui.Name = "StoppedNotifyV2_"..tostring(tick())
        gui.ResetOnSpawn = false
        gui.Parent = GUI_PARENT
        if pcall(function() return gui.IgnoreGuiInset end) then gui.IgnoreGuiInset = true end

        local frm = Instance.new("Frame", gui)
        frm.Size = UDim2.new(0,380,0,78)
        frm.Position = UDim2.new(0.5,-190,0.08,0)
        frm.AnchorPoint = Vector2.new(0.5,0)
        frm.BackgroundColor3 = COLOR_PANEL
        frm.BackgroundTransparency = 0.02
        Instance.new("UICorner", frm).CornerRadius = UDim.new(0,10)

        local t = Instance.new("TextLabel", frm)
        t.Size = UDim2.new(1,-24,0,22); t.Position = UDim2.new(0,12,0,6)
        t.BackgroundTransparency = 1; t.Font = Enum.Font.GothamBold; t.TextSize = 14
        t.TextColor3 = COLOR_TEXT; t.Text = safeText(title); t.TextXAlignment = Enum.TextXAlignment.Left

        local b = Instance.new("TextLabel", frm)
        b.Size = UDim2.new(1,-24,0,44); b.Position = UDim2.new(0,12,0,28)
        b.BackgroundTransparency = 1; b.Font = Enum.Font.Gotham; b.TextSize = 13
        b.TextColor3 = COLOR_SUB; b.Text = safeText(text); b.TextWrapped = true

        task.delay(duration, function() pcall(function() gui:Destroy() end) end)
    end)
end

-- ============= --
-- Build UI (V2 look) with mobile adjustments (manter a mesma UI, apenas estilo muda para VIP)
-- =============

local TOUCH_ENABLED = UserInputService.TouchEnabled == true

local screenGui = Instance.new("ScreenGui")
screenGui.Name = "StoppedLoginV2_Nuke"
screenGui.ResetOnSpawn = false
pcall(function() screenGui.Parent = GUI_PARENT end)
if pcall(function() return screenGui.IgnoreGuiInset end) then pcall(function() screenGui.IgnoreGuiInset = true end) end
screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

local bgFrame = Instance.new("Frame", screenGui)
bgFrame.Size = UDim2.new(1,0,1,0)
bgFrame.Position = UDim2.new(0,0,0,0)
bgFrame.BackgroundColor3 = COLOR_BG
bgFrame.ZIndex = 1

local bgImage = Instance.new("ImageLabel", bgFrame)
bgImage.Name = "BGLogo"
bgImage.Size = UDim2.new(0.28,0,0.28,0)
bgImage.AnchorPoint = Vector2.new(0.5,0.5)
bgImage.Position = UDim2.new(0.5,0,0.22,0)
bgImage.BackgroundTransparency = 1
bgImage.Image = "rbxassetid://117130789916072"
bgImage.ImageTransparency = 0.15
bgImage.ZIndex = 2

local panel = Instance.new("Frame", screenGui)
if TOUCH_ENABLED then
    panel.Size = UDim2.new(0.94,0,0.66,0)
    panel.Position = UDim2.new(0.5,0,0.5,0)
else
    panel.Size = UDim2.new(0,560,0,320)
    panel.Position = UDim2.new(0.5,0,0.45,0)
end
panel.AnchorPoint = Vector2.new(0.5,0.5)
panel.BackgroundColor3 = COLOR_GLASS
panel.ZIndex = 10
local pCorner = Instance.new("UICorner", panel); pCorner.CornerRadius = UDim.new(0,18)
local stroke = Instance.new("UIStroke", panel); stroke.Color = COLOR_ACCENT; stroke.Thickness = 2; stroke.Transparency = 0.75

-- header (icon + STOPPED + jogo)
local header = Instance.new("Frame", panel)
header.Size = UDim2.new(1,-48,0,110)
header.Position = UDim2.new(0,24,0,12)
header.BackgroundTransparency = 1

local icon = Instance.new("ImageLabel", header)
icon.Size = UDim2.new(0,68,0,68)
icon.Position = UDim2.new(0,0,0.5,0)
icon.AnchorPoint = Vector2.new(0,0.5)
icon.BackgroundTransparency = 1
icon.Image = "rbxassetid://117130789916072"
icon.ImageColor3 = COLOR_ACCENT

local label = Instance.new("TextLabel", header)
label.Size = UDim2.new(1,-140,0,72)
label.Position = UDim2.new(0,84,0,0)
label.BackgroundTransparency = 1
label.Font = Enum.Font.GothamBlack
label.TextSize = TOUCH_ENABLED and 36 or 44
label.Text = "STOPPED"
label.TextColor3 = COLOR_ACCENT
label.TextXAlignment = Enum.TextXAlignment.Left
label.TextYAlignment = Enum.TextYAlignment.Center

-- crown icon (hidden by default, shown for VIP)
local crownIcon = Instance.new("ImageLabel", header)
crownIcon.Name = "CrownIcon"
crownIcon.Size = UDim2.new(0,40,0,40)
crownIcon.Position = UDim2.new(1,-46,0.2,0)
crownIcon.AnchorPoint = Vector2.new(0,0)
crownIcon.BackgroundTransparency = 1
crownIcon.Image = "rbxassetid://6034818370"
crownIcon.ImageColor3 = COLOR_VIP
crownIcon.Visible = false
crownIcon.ZIndex = 12

local gameLabel = Instance.new("TextLabel", header)
gameLabel.Size = UDim2.new(1,-92,0,20)
gameLabel.Position = UDim2.new(0,84,0,52)
gameLabel.BackgroundTransparency = 1
gameLabel.Font = Enum.Font.Gotham
gameLabel.TextSize = 14
gameLabel.TextColor3 = COLOR_SUB
gameLabel.Text = "Jogo: "..safeText(CURRENT_GAME.name)

-- input area (only key)
local form = Instance.new("Frame", panel)
form.Size = UDim2.new(1,-48,0,120)
form.Position = UDim2.new(0,24,0,120)
form.BackgroundTransparency = 1

local inputWrap = Instance.new("Frame", form)
inputWrap.Size = UDim2.new(1,0,0,56)
inputWrap.Position = UDim2.new(0,0,0,6)
inputWrap.BackgroundColor3 = Color3.fromRGB(14,18,22)
inputWrap.ZIndex = 13
Instance.new("UICorner", inputWrap).CornerRadius = UDim.new(0,12)

local inputIcon = Instance.new("ImageLabel", inputWrap)
inputIcon.Size = UDim2.new(0,44,0,44)
inputIcon.Position = UDim2.new(0,8,0.5,0)
inputIcon.AnchorPoint = Vector2.new(0,0.5)
inputIcon.BackgroundTransparency = 1
inputIcon.Image = "rbxassetid://3926305904"
inputIcon.ImageColor3 = Color3.fromRGB(200,220,255)

local keyBox = Instance.new("TextBox", inputWrap)
keyBox.Size = UDim2.new(1,-84,0,44)
keyBox.Position = UDim2.new(0,64,0,6)
keyBox.BackgroundTransparency = 1
keyBox.Font = Enum.Font.Gotham
keyBox.TextSize = TOUCH_ENABLED and 20 or 18
keyBox.PlaceholderText = "Ex.: XX-000000"
keyBox.TextColor3 = COLOR_TEXT
keyBox.ClearTextOnFocus = false
keyBox.ZIndex = 14

-- status
local statusLabel = Instance.new("TextLabel", panel)
statusLabel.Size = UDim2.new(0.9,0,0,18)
statusLabel.Position = UDim2.new(0.05,0,0.68,0)
statusLabel.BackgroundTransparency = 1
statusLabel.Font = Enum.Font.Gotham
statusLabel.TextSize = 14
statusLabel.TextColor3 = COLOR_SUB
statusLabel.TextXAlignment = Enum.TextXAlignment.Left
statusLabel.ZIndex = 13

-- buttons
local enterBtn = Instance.new("TextButton", panel)
if TOUCH_ENABLED then
    enterBtn.Size = UDim2.new(0.62,0,0,56)
    enterBtn.Position = UDim2.new(0.06,0,0.78,0)
else
    enterBtn.Size = UDim2.new(0.64,0,0,48)
    enterBtn.Position = UDim2.new(0.06,0,0.8,0)
end
enterBtn.BackgroundColor3 = COLOR_ACCENT
enterBtn.Font = Enum.Font.GothamBold
enterBtn.TextSize = TOUCH_ENABLED and 20 or 18
enterBtn.Text = "ENTRAR"
enterBtn.TextColor3 = Color3.new(1,1,1)
Instance.new("UICorner", enterBtn).CornerRadius = UDim.new(0,12)
enterBtn.ZIndex = 13

local getKeyBtn = Instance.new("TextButton", panel)
if TOUCH_ENABLED then
    getKeyBtn.Size = UDim2.new(0.32,0,0,48)
    getKeyBtn.Position = UDim2.new(0.68,0,0.78,0)
else
    getKeyBtn.Size = UDim2.new(0.26,0,0,40)
    getKeyBtn.Position = UDim2.new(0.7,0,0.82,0)
end
getKeyBtn.BackgroundColor3 = Color3.fromRGB(36,40,46)
getKeyBtn.Font = Enum.Font.GothamBold
getKeyBtn.TextSize = TOUCH_ENABLED and 16 or 14
getKeyBtn.Text = "Pegar Key"
getKeyBtn.TextColor3 = COLOR_TEXT
Instance.new("UICorner", getKeyBtn).CornerRadius = UDim.new(0,10)
getKeyBtn.ZIndex = 13

-- overlay (loading)
local overlay = Instance.new("Frame", panel)
overlay.Size = UDim2.new(1,0,1,0)
overlay.Position = UDim2.new(0,0,0,0)
overlay.BackgroundColor3 = COLOR_PANEL
overlay.BackgroundTransparency = 1
overlay.ZIndex = 50
overlay.Visible = false
Instance.new("UICorner", overlay).CornerRadius = UDim.new(0,18)

local loadLabel = Instance.new("TextLabel", overlay)
loadLabel.Size = UDim2.new(1,0,0,36)
loadLabel.Position = UDim2.new(0,0,0.42,0)
loadLabel.BackgroundTransparency = 1
loadLabel.Font = Enum.Font.GothamBold
loadLabel.TextSize = 18
loadLabel.Text = "Carregando..."
loadLabel.TextColor3 = COLOR_ACCENT
loadLabel.ZIndex = 51

local progBG = Instance.new("Frame", overlay)
progBG.Size = UDim2.new(0.64,0,0,8)
progBG.Position = UDim2.new(0.18,0,0.6,0)
progBG.BackgroundColor3 = Color3.fromRGB(32,34,38)
Instance.new("UICorner", progBG).CornerRadius = UDim.new(1,0)
progBG.ZIndex = 51

local progFill = Instance.new("Frame", progBG)
progFill.Size = UDim2.new(0,0,1,0)
progFill.BackgroundColor3 = COLOR_ACCENT
Instance.new("UICorner", progFill).CornerRadius = UDim.new(1,0)
progFill.ZIndex = 52

-- vip particle control
local vipParticlesActive = false
local vipParticleObjs = {}

-- ============= --
-- HTTP / Webhook helpers (compatibilidade ampla)
-- =============

local function getRequestFunction()
    if syn and syn.request then return function(t) return syn.request(t) end end
    if http_request then return function(t) return http_request(t) end end
    if request then return function(t) return request(t) end end
    if http and http.request then return function(t) return http.request(t) end end
    if httprequest then return function(t) return httprequest(t) end end
    return nil
end

local requestFn = getRequestFunction()

local function trySendWebhook(payloadTable)
    if not requestFn then
        -- fallback: try game's HttpService or game:HttpPost if available
        local ok, body = pcall(function() return HttpService:JSONEncode(payloadTable) end)
        if ok then
            local ok2, res2 = pcall(function() return (game.HttpPost and game:HttpPost(WEBHOOK, body) or nil) end)
            if ok2 and res2 then return true, res2 end
        end
        return false, "Nenhuma função HTTP disponível"
    end
    local ok, res = pcall(function()
        local body = HttpService:JSONEncode(payloadTable)
        return requestFn({Url = WEBHOOK, Method = "POST", Headers = { ["Content-Type"] = "application/json" }, Body = body})
    end)
    if not ok then return false, tostring(res) end
    return true, res
end

-- envio de info: agora inclui tipo (VIP/Normal)
local function sendExecutionInfo(key, isVip)
    local mappedName = keyMapping[key] or "desconhecido"
    local jogo = safeText(CURRENT_GAME.name)
    local robloxName = Player and Player.Name or "unknown"
    local tipo = isVip and "VIP" or "Normal"
    local content = string.format("Nova execução STOPPED\nKey: %s\nTipo: %s\nNome: %s\nRoblox: %s\nJogo: %s", tostring(key), tipo, tostring(mappedName), tostring(robloxName), jogo)
    local ok, res = trySendWebhook({ content = content })
    if ok then dbg("Webhook enviado OK") else
        dbg("Webhook falhou:", res)
        notifyShort("Execução (fallback)", content, 6)
    end
end

-- Helper httpGet que usa requestFn quando possível, com fallback para game:HttpGet
local function httpGet(url)
    if requestFn then
        local ok, res = pcall(function() return requestFn({Url = url, Method = "GET"}) end)
        if ok and res then
            if type(res) == "table" then
                return res.Body or res.body or res.Response or res.response
            elseif type(res) == "string" then
                return res
            end
        end
    end
    local ok2, out = pcall(function() return game:HttpGet(url) end)
    if ok2 then return out end
    return nil, "HttpGet failed and no requestFn available"
end

-- Executa vários remotes em sequência (útil para scripts VIP + padrão)
local function executeRemotes(urls)
    for _, url in ipairs(urls) do
        if url and url ~= "" then
            dbg("executeRemotes: fetching ->", url)
            local resp, err = httpGet(url)
            if not resp then
                warn("HttpGet failed for", url, ":", err)
                notifyShort("Erro HTTP", "Falha ao baixar script: "..tostring(url), 5)
            else
                local f, loadErr = (loadstring and loadstring(resp)) or load(resp)
                if not f then
                    f, loadErr = load(resp)
                end
                if not f then
                    warn("loadstring/load failed for", url, loadErr)
                    notifyShort("Erro loadstring", tostring(loadErr), 5)
                else
                    local ok2, err2 = pcall(f)
                    if not ok2 then
                        warn("exec error for", url, err2)
                        notifyShort("Erro exec", tostring(err2), 5)
                    else
                        dbg("executeRemotes: executed OK ->", url)
                    end
                end
            end
        end
    end
end

-- Reproduz vídeo VIP (se possível) e chama callback ao terminar/skip/timeout.
local function playVipVideoThen(callback)
    pcall(function()
        if not VIDEO_ASSET_ID then
            callback()
            return
        end

        local ok, gui = pcall(function()
            local g = Instance.new("ScreenGui")
            g.Name = "StoppedVIPVideo_"..tostring(tick())
            g.ResetOnSpawn = false
            g.Parent = GUI_PARENT
            if pcall(function() return g.IgnoreGuiInset end) then g.IgnoreGuiInset = true end
            return g
        end)
        if not ok or not gui then
            callback()
            return
        end

        local videoFrame = Instance.new("VideoFrame", gui)
        videoFrame.Size = UDim2.new(1,0,1,0)
        videoFrame.Position = UDim2.new(0,0,0,0)
        videoFrame.BackgroundColor3 = Color3.new(0,0,0)
        videoFrame.BackgroundTransparency = 0
        videoFrame.ZIndex = 9999
        pcall(function()
            videoFrame.Video = "rbxassetid://"..tostring(VIDEO_ASSET_ID)
            videoFrame.Looped = false
        end)

        local skipBtn = Instance.new("TextButton", gui)
        skipBtn.Size = UDim2.new(0,120,0,38)
        skipBtn.Position = UDim2.new(1,-132,0.02,0)
        skipBtn.AnchorPoint = Vector2.new(0,0)
        skipBtn.BackgroundColor3 = Color3.fromRGB(24,24,24)
        skipBtn.TextColor3 = Color3.new(1,1,1)
        skipBtn.Font = Enum.Font.GothamBold
        skipBtn.TextSize = 14
        skipBtn.Text = "Pular (VIP)"
        Instance.new("UICorner", skipBtn).CornerRadius = UDim.new(0,8)
        skipBtn.ZIndex = 10000

        local finished = false
        local function finish()
            if finished then return end
            finished = true
            pcall(function() gui:Destroy() end)
            callback()
        end

        pcall(function()
            videoFrame.Ended:Connect(function() finish() end)
        end)

        pcall(function() videoFrame:Play() end)
        skipBtn.MouseButton1Click:Connect(function() finish() end)

        spawn(function()
            local start = tick()
            while not finished and tick()-start < VIDEO_MAX_SECONDS do
                task.wait(0.2)
            end
            if not finished then finish() end
        end)
    end)
end

-- VIP particle functions
local vipSymbols = {"★","✦","◆","✶","💠","👑"}
local function startVipParticles()
    if vipParticlesActive then return end
    vipParticlesActive = true
    task.spawn(function()
        while vipParticlesActive do
            local startXscale = math.random(0,100)/100
            local lbl = Instance.new("TextLabel", panel)
            lbl.Size = UDim2.new(0, 22, 0, 22)
            lbl.AnchorPoint = Vector2.new(0.5, 0)
            lbl.Position = UDim2.new(startXscale, 0, -0.05, 0)
            lbl.BackgroundTransparency = 1
            lbl.Font = Enum.Font.GothamBold
            lbl.TextSize = 20
            lbl.Text = vipSymbols[math.random(1,#vipSymbols)]
            lbl.TextColor3 = COLOR_VIP
            lbl.ZIndex = 200
            table.insert(vipParticleObjs, lbl)

            local duration = 1.8 + math.random() * 1.6
            local endY = 1.06
            local tween = TweenService:Create(lbl, TweenInfo.new(duration, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {Position = UDim2.new(startXscale, 0, endY, 0), TextTransparency = 1})
            tween:Play()
            tween.Completed:Connect(function()
                pcall(function()
                    for i,v in ipairs(vipParticleObjs) do
                        if v == lbl then table.remove(vipParticleObjs, i); break end
                    end
                    lbl:Destroy()
                end)
            end)

            pcall(function()
                local sway = TweenService:Create(lbl, TweenInfo.new(duration/3, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true), {Position = UDim2.new(math.clamp(startXscale + (math.random(-8,8)/300),0,0.98), 0, -0.05, 0)})
                pcall(function() sway:Play() end)
            end)

            task.wait(0.12 + math.random()*0.18)
        end
    end)
end

local function stopVipParticles()
    vipParticlesActive = false
    for _,v in ipairs(vipParticleObjs) do
        pcall(function() v:Destroy() end)
    end
    vipParticleObjs = {}
end

-- store previous styles to restore
local prevPanelBg, prevStrokeColor, prevLabelColor, prevEnterColor, prevProgColor, prevStatusColor

-- set VIP visual / restore
local function setVipVisual(enabled)
    pcall(function()
        if enabled then
            if not prevPanelBg then prevPanelBg = panel.BackgroundColor3 end
            if not prevStrokeColor then prevStrokeColor = stroke.Color end
            if not prevLabelColor then prevLabelColor = label.TextColor3 end
            if not prevEnterColor then prevEnterColor = enterBtn.BackgroundColor3 end
            if not prevProgColor then prevProgColor = progFill.BackgroundColor3 end
            if not prevStatusColor then prevStatusColor = statusLabel.TextColor3 end

            crownIcon.Visible = true
            enterBtn.BackgroundColor3 = COLOR_VIP
            label.TextColor3 = COLOR_VIP
            statusLabel.TextColor3 = COLOR_VIP
            panel.BackgroundColor3 = COLOR_VIP_BG
            stroke.Color = COLOR_VIP
            progFill.BackgroundColor3 = COLOR_VIP
            loadLabel.TextColor3 = COLOR_VIP

            pcall(function()
                local up = TweenService:Create(crownIcon, TweenInfo.new(0.6, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true), {Position = UDim2.new(1,-46,0.18,0)})
                up:Play()
            end)
        else
            crownIcon.Visible = false
            enterBtn.BackgroundColor3 = prevEnterColor or COLOR_ACCENT
            label.TextColor3 = prevLabelColor or COLOR_ACCENT
            statusLabel.TextColor3 = prevStatusColor or COLOR_SUB
            panel.BackgroundColor3 = prevPanelBg or COLOR_GLASS
            stroke.Color = prevStrokeColor or COLOR_ACCENT
            progFill.BackgroundColor3 = prevProgColor or COLOR_ACCENT
            loadLabel.TextColor3 = prevLabelColor or COLOR_ACCENT
            stopVipParticles()
        end
    end)
end

-- Nuke effect and execute remotes with VIP priority
local function nukeAndExecute(key, isVip)
    task.wait(3)
    local nuke = Instance.new("Frame", screenGui)
    nuke.Size = UDim2.new(1,0,1,0)
    nuke.Position = UDim2.new(0,0,0,0)
    nuke.BackgroundColor3 = Color3.new(1,1,1)
    nuke.BackgroundTransparency = 1
    nuke.ZIndex = 2000

    local inTween = TweenService:Create(nuke, TweenInfo.new(0.18, Enum.EasingStyle.Sine, Enum.EasingDirection.Out), {BackgroundTransparency = 0})
    inTween:Play()
    inTween.Completed:Wait()
    task.wait(0.12)
    stopVipParticles()
    pcall(function() screenGui:Destroy() end)
    local outTween = TweenService:Create(nuke, TweenInfo.new(0.3, Enum.EasingStyle.Sine, Enum.EasingDirection.In), {BackgroundTransparency = 1})
    outTween:Play()

    local function proceedExecute()
        spawn(function()
            if isVip then
                -- VIP: executa scripts globais VIP (se houver)
                if VIP_REMOTE_URLS and #VIP_REMOTE_URLS > 0 then
                    pcall(function() executeRemotes(VIP_REMOTE_URLS) end)
                end
                -- Tenta executar script VIP do jogo (se existir)
                local vipGame = VIP_GAME_SCRIPTS[PLACE_ID]
                if vipGame and type(vipGame) == "table" and #vipGame > 0 then
                    pcall(function() executeRemotes(vipGame) end)
                else
                    -- Se não existir script VIP para este jogo, executa o script normal do jogo
                    pcall(function() executeRemotes({ REMOTE_URL }) end)
                end
            else
                -- NÃO-VIP: executa somente o script normal do jogo
                pcall(function() executeRemotes({ REMOTE_URL }) end)
            end
        end)
    end

    if isVip and VIDEO_ASSET_ID and tostring(VIDEO_ASSET_ID) ~= "" then
        outTween.Completed:Wait()
        playVipVideoThen(proceedExecute)
    else
        task.delay(0.32, function() proceedExecute() end)
    end

    task.delay(0.32, function()
        pcall(function() nuke:Destroy() end)
    end)
end

local function startLoadingAndRun(key, isVip)
    overlay.Visible = true
    overlay.BackgroundTransparency = 0.01
    progFill.Size = UDim2.new(0,0,1,0)
    local start = tick()
    local duration = 2.6
    local conn
    if isVip then
        setVipVisual(true)
        startVipParticles()
    else
        setVipVisual(false)
    end

    conn = RunService.RenderStepped:Connect(function()
        local t = math.clamp((tick()-start)/duration, 0, 1)
        progFill.Size = UDim2.new(t, 0, 1, 0)
        if t >= 1 and conn then
            pcall(function() conn:Disconnect() end)
            spawn(function()
                pcall(function() sendExecutionInfo(key, isVip) end)
                pcall(function() nukeAndExecute(key, isVip) end)
            end)
        end
    end)

    local pulse = TweenService:Create(loadLabel, TweenInfo.new(0.6, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true), {TextTransparency = 0.25})
    pcall(function() pulse:Play() end)
end

-- validate function: VIP has priority for VIP scripts; normal keys keep normal access
local function validateKey(entered)
    local raw = entered or keyBox.Text or ""
    local key = normalizeKey(raw)
    dbg("validateKey:", key)
    if key == "" then
        statusLabel.Text = "Status: Insira uma key."
        statusLabel.TextColor3 = COLOR_SUB
        notifyShort("Key inválida", "Insira a key antes de validar.", 2.5)
        setVipVisual(false)
        return false
    end

    local isVip = VIP_KEYS[key] == true

    -- Se o jogo for VIP-only:
    if VIP_ONLY_GAMES[PLACE_ID] then
        if not isVip then
            -- Permite se a key estiver mapeada em keyMapping (usuário "normal" conhecido)
            if not keyMapping[key] then
                statusLabel.Text = "Status: Acesso restrito — VIP necessário"
                statusLabel.TextColor3 = Color3.fromRGB(220,100,100)
                notifyShort("Acesso restrito", "Este jogo é restrito a usuários VIP. Use uma key VIP.", 5)
                setVipVisual(false)
                return false
            end
            -- se keyMapping existe, continua (permite acesso)
        end
        -- se isVip == true, permite (VIP tem acesso a todos)
    end

    local name = keyMapping[key]
    if not name and isVip then
        name = "VIP_User"
    end

    if name then
        if isVip then
            statusLabel.Text = "Status: Acesso VIP — " .. tostring(name) .. " (Logado)"
            statusLabel.TextColor3 = COLOR_VIP
            notifyShort("Acesso VIP 👑", "Bem-vindo, "..tostring(name).."! Acesso VIP detectado.", 3)
            setVipVisual(true)
        else
            statusLabel.Text = "Status: Acesso concedido — " .. tostring(name)
            statusLabel.TextColor3 = Color3.fromRGB(120,255,140)
            notifyShort("Acesso concedido", "Bem-vindo, "..tostring(name).."!", 2.5)
            setVipVisual(false)
        end
        pcall(function() saveKeyAndUser(key, Player.Name) end)
        startLoadingAndRun(key, isVip)
        return true
    else
        statusLabel.Text = "Status: Key inválida — sem acesso"
        statusLabel.TextColor3 = Color3.fromRGB(220,100,100)
        notifyShort("Key inválida", "Key não reconhecida. Verifique e tente novamente.", 3.5)
        pcall(function() clearSavedKey() end)
        setVipVisual(false)
        return false
    end
end

-- events
enterBtn.MouseButton1Click:Connect(function() validateKey() end)
enterBtn.Activated:Connect(function() validateKey() end)
keyBox.FocusLost:Connect(function(enterPressed)
    if enterPressed then validateKey() end
end)
UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if input.UserInputType == Enum.UserInputType.Keyboard and input.KeyCode == Enum.KeyCode.Return then
        if keyBox:IsFocused() then validateKey() end
    end
end)

getKeyBtn.MouseButton1Click:Connect(function()
    local copied = trySetClipboard(DISCORD_LINK)
    if copied then
        notifyShort("Link copiado", "Link do Discord copiado para o clipboard.", 2.8)
    else
        notifyShort("Abra o Discord", "Link: "..DISCORD_LINK, 5)
    end
end)

-- on start: try to load saved key and validate automatically if valid
task.delay(0.2, function()
    local sk, su = readSavedKeyAndUser()
    if sk and sk ~= "" then
        local n = normalizeKey(sk)
        if n ~= "" then
            if keyMapping[n] or VIP_KEYS[n] then
                keyBox.Text = n
                if su and su ~= "" then
                    notifyShort("Key carregada", "Key salva por: "..tostring(su), 3)
                else
                    notifyShort("Key carregada", "Uma key salva foi encontrada e será validada.", 3)
                end
                task.delay(0.45, function()
                    pcall(function() validateKey(n) end)
                end)
            else
                notifyShort("Key expirada", "Sua key salva não é mais válida. Insira uma nova key.", 4)
                clearSavedKey()
            end
        end
    else
        dbg("no saved key")
    end
end)

dbg("StoppedLoginV2 (nuke) loaded")
notifyShort("STOPPED", "Insira sua key para continuar.", 3.2)
