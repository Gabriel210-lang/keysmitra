--[[
    Mitra Menu V2.0 - Sistema Otimizado
    Whitelist + Keys Temporárias + Proteção Anti-Cheat
]]

local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local HttpService = game:GetService("HttpService")
local StarterGui = game:GetService("StarterGui")

-- CONFIGURAÇÕES
local WHITELIST = {
    "danizin1356",
    "Guilherm2584", 
    "gabrjvo",
    "unaruto245",
    "caveirar6six",
    "hlambats05"
}

local WEBHOOK_URL = "https://discord.com/api/webhooks/1386042354615587018/BObPujWmBplkndpbnm_EVEQ4mglXJtcEggocpZ7eURdi1LksOHlQE9bJprNbmHesF5l2"
local KEY_WEBHOOK = "https://discord.com/api/webhooks/1386041965669388529/B6MhYOj0SjfRCEbcDFOUCjUJpiyuI5YArAdRym5EZugcB1Lh5CKm-skOhYB7mQKdI3Z0"

-- VERIFICAÇÃO DE WHITELIST
local function isWhitelisted(username)
    for _, nick in pairs(WHITELIST) do
        if nick == username then return true end
    end
    return false
end

if not isWhitelisted(LocalPlayer.Name) then
    StarterGui:SetCore("SendNotification", {
        Title = "🚫 ACESSO NEGADO";
        Text = "Você não está autorizado!";
        Duration = 5;
    })
    warn("❌ Acesso negado: " .. LocalPlayer.Name)
    return
end

-- SISTEMA DE KEYS
local KeyStorage = {}
local STORAGE_FILE = "MitraKeys_" .. LocalPlayer.UserId .. ".dat"

local function saveData()
    if writefile then
        pcall(function()
            writefile(STORAGE_FILE, HttpService:JSONEncode(KeyStorage))
        end)
    end
end

local function loadData()
    if readfile and isfile and isfile(STORAGE_FILE) then
        pcall(function()
            local data = readfile(STORAGE_FILE)
            if data and data ~= "" then
                KeyStorage = HttpService:JSONDecode(data)
            end
        end)
    end
end

local function generateKey()
    local chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789"
    local key = "MITRA-"
    math.randomseed(tick())
    
    for i = 1, 12 do
        local idx = math.random(1, #chars)
        key = key .. chars:sub(idx, idx)
    end
    return key
end

local function isKeyValid(userId)
    local data = KeyStorage[tostring(userId)]
    if not data then return false end
    
    if os.time() >= data.expireTime then
        KeyStorage[tostring(userId)] = nil
        saveData()
        return false
    end
    return true
end

local function createKey(userId)
    local key = generateKey()
    local expireTime = os.time() + (48 * 3600) -- 48 horas
    
    KeyStorage[tostring(userId)] = {
        key = key,
        expireTime = expireTime,
        userId = userId
    }
    saveData()
    return key, expireTime
end

local function validateKey(inputKey, userId)
    local data = KeyStorage[tostring(userId)]
    if not data then return false, "Key não encontrada" end
    if not isKeyValid(userId) then return false, "Key expirada" end
    if data.key ~= inputKey then return false, "Key incorreta" end
    return true, "Key válida"
end

-- PROTEÇÃO ANTI-CHEAT
local function protectScript()
    pcall(function()
        -- Remover componentes de detecção
        for _, service in pairs({"ReplicatedStorage", "ServerStorage"}) do
            local svc = game:GetService(service)
            for _, obj in pairs(svc:GetDescendants()) do
                if obj.Name:lower():find("adonis") or obj.Name:lower():find("admin") then
                    obj:Destroy()
                end
            end
        end
        
        -- Hook de proteção
        if getrawmetatable then
            local oldNamecall = getrawmetatable(game).__namecall
            setreadonly(getrawmetatable(game), false)
            
            getrawmetatable(game).__namecall = function(self, ...)
                local method = getnamecallmethod()
                if (method == "FireServer" or method == "InvokeServer") and 
                   (self.Name:lower():find("admin") or self.Name:lower():find("log")) then
                    return
                end
                return oldNamecall(self, ...)
            end
            
            setreadonly(getrawmetatable(game), true)
        end
    end)
end

-- SISTEMA DE LOGS
local function sendLog(webhook, title, description, color)
    spawn(function()
        pcall(function()
            local embed = {
                title = title,
                description = description,
                color = color or 3447003,
                thumbnail = {
                    url = "https://www.roblox.com/headshot-thumbnail/image?userId="..LocalPlayer.UserId.."&width=420&height=420&format=png"
                },
                fields = {
                    {name = "👤 Jogador", value = LocalPlayer.Name, inline = true},
                    {name = "🆔 ID", value = tostring(LocalPlayer.UserId), inline = true},
                    {name = "🎮 Jogo", value = game:GetService("MarketplaceService"):GetProductInfo(game.PlaceId).Name, inline = false}
                },
                timestamp = os.date("!%Y-%m-%dT%H:%M:%SZ")
            }
            
            local data = {embeds = {embed}}
            local jsonData = HttpService:JSONEncode(data)
            
            local requestFunc = syn and syn.request or http_request or request
            if requestFunc then
                requestFunc({
                    Url = webhook,
                    Method = "POST",
                    Headers = {["Content-Type"] = "application/json"},
                    Body = jsonData
                })
            end
        end)
    end)
end

-- INICIALIZAÇÃO
loadData()
protectScript()

-- INTERFACE
local success, Rayfield = pcall(function()
    return loadstring(game:HttpGet('https://sirius.menu/rayfield'))()
end)

if not success then
    warn("❌ Erro ao carregar interface!")
    return
end

local Window = Rayfield:CreateWindow({
    Name = "🔐 Mitra Menu - Sistema Protegido",
    LoadingTitle = "Carregando...",
    LoadingSubtitle = "Sistema de Autenticação",
    KeySystem = false
})

local Tab = Window:CreateTab("🔑 Autenticação", 4483362458)
local Section = Tab:CreateSection("Sistema de Keys (48h)")

-- Status da key
local currentKey = KeyStorage[tostring(LocalPlayer.UserId)]
local statusText = "❌ Nenhuma key ativa"

if currentKey and isKeyValid(LocalPlayer.UserId) then
    local timeLeft = currentKey.expireTime - os.time()
    local hours = math.floor(timeLeft / 3600)
    local minutes = math.floor((timeLeft % 3600) / 60)
    statusText = "✅ Key válida por " .. hours .. "h " .. minutes .. "m"
end

local StatusLabel = Tab:CreateLabel(statusText)
local keyInput = ""

-- Input de key
Tab:CreateInput({
    Name = "🔑 Digite sua Key",
    PlaceholderText = "Insira a key aqui...",
    RemoveTextAfterFocusLost = false,
    Callback = function(text)
        keyInput = text
    end,
})

-- Botão obter key
Tab:CreateButton({
    Name = "🔗 Obter Key (48h)",
    Callback = function()
        if isKeyValid(LocalPlayer.UserId) then
            Rayfield:Notify({
                Title = "ℹ️ Key Ativa",
                Content = "Você já possui uma key válida!",
                Duration = 3,
            })
            return
        end
        
        local newKey, expireTime = createKey(LocalPlayer.UserId)
        
        -- Copiar para clipboard
        pcall(function()
            if setclipboard then
                setclipboard(newKey)
            elseif toclipboard then
                toclipboard(newKey)
            end
        end)
        
        -- Enviar log
        sendLog(KEY_WEBHOOK, "🔑 Nova Key Gerada", 
                "Key: `" .. newKey .. "`\nExpira em: " .. os.date("%d/%m/%Y %H:%M", expireTime))
        
        Rayfield:Notify({
            Title = "🔑 Key Gerada!",
            Content = "Key copiada! Válida por 48h",
            Duration = 4,
        })
        
        -- Atualizar status
        local timeLeft = expireTime - os.time()
        local hours = math.floor(timeLeft / 3600)
        local minutes = math.floor((timeLeft % 3600) / 60)
        StatusLabel:Set("✅ Key válida por " .. hours .. "h " .. minutes .. "m")
    end,
})

-- Botão verificar key
Tab:CreateButton({
    Name = "🔍 Verificar Key",
    Callback = function()
        if keyInput == "" then
            Rayfield:Notify({
                Title = "⚠️ Aviso",
                Content = "Digite uma key primeiro!",
                Duration = 3,
            })
            return
        end
        
        local isValid, message = validateKey(keyInput, LocalPlayer.UserId)
        
        if isValid then
            Rayfield:Notify({
                Title = "✅ Sucesso!",
                Content = "Key verificada! Carregando menu...",
                Duration = 2,
            })
            
            -- Enviar log de acesso
            sendLog(WEBHOOK_URL, "🎯 Acesso Autorizado", 
                    "Usuário **" .. LocalPlayer.Name .. "** acessou o menu", 7851007)
            
            -- Carregar menu principal
            spawn(function()
                wait(2)
                pcall(function() Window:Destroy() end)
                wait(0.5)
                
                local urls = {
                    'https://raw.githubusercontent.com/Gabriel210-lang/Mitra-Menu/refs/heads/main/Mitra.md',
                    'https://raw.githubusercontent.com/Gabriel210-lang/Mitra-Menu/main/Mitra.md'
                }
                
                for _, url in pairs(urls) do
                    local success, content = pcall(function()
                        return game:HttpGet(url, true)
                    end)
                    
                    if success and content and #content > 100 then
                        print("✅ Carregando Mitra Menu...")
                        pcall(function()
                            protectScript() -- Reforçar proteção
                            loadstring(content)()
                        end)
                        return
                    end
                end
                
                warn("❌ Erro ao carregar o menu principal")
            end)
        else
            Rayfield:Notify({
                Title = "❌ Key Inválida",
                Content = message,
                Duration = 3,
            })
        end
    end,
})

-- Seção de informações
local InfoSection = Tab:CreateSection("ℹ️ Informações")
Tab:CreateLabel("✅ Usuário autorizado: " .. LocalPlayer.Name)
Tab:CreateLabel("🛡️ Proteção anti-cheat ativa")
Tab:CreateLabel("📱 Keys válidas por 48h em qualquer servidor")

print("🔐 Mitra Menu V2.0 iniciado!")
print("✅ Usuário autorizado: " .. LocalPlayer.Name)
