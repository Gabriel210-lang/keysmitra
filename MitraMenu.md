--[[
    Mitra Menu V3.0 - Sistema Completo de Gerenciamento
    🔧 Funcionalidades Completas + Correção de Key Persistente
    ✅ CORRIGIDO: Callback e execução do script principal
]]

local Players, HttpService, LocalPlayer = game:GetService("Players"), game:GetService("HttpService"), game:GetService("Players").LocalPlayer

-- ========== CONFIGURAÇÕES ==========
local CONFIG = {
    BIN_ID = "685f65988561e97a502d2be2",
    API_KEY = "$2a$10$UyPUnhCf6itJaEjtMj.RwOYbAsTUZYZ1ms9UXpIiyRprOHxo327vO",
    WEBHOOK = "https://discord.com/api/webhooks/1390126976337055827/vCsHifASS7I5uRvM-sPj6rEHImaCP-xatqY0Mi7RLjX48vPYn6csg4d-8j5gzCKN7617",
    KEY_WEBHOOK = "https://discord.com/api/webhooks/1390126976337055827/vCsHifASS7I5uRvM-sPj6rEHImaCP-xatqY0Mi7RLjX48vPYn6csg4d-8j5gzCKN7617",
    MAIN_SCRIPT_URL = "https://raw.githubusercontent.com/Gabriel210-lang/Mitra-Menu/refs/heads/main/Mitra.md"
}

-- ========== DADOS GLOBAIS PERSISTENTES ==========
local DB = {
    users = {
        {username = "danizin1356", addedBy = "SYSTEM", addedTime = os.time(), expireTime = nil},
        {username = "hlambats05", addedBy = "SYSTEM", addedTime = os.time(), expireTime = nil},
        {username = "gabrjvo", addedBy = "SYSTEM", addedTime = os.time(), expireTime = nil},
        {username = "unaruto245", addedBy = "SYSTEM", addedTime = os.time(), expireTime = nil},
        {username = "caveirar6six", addedBy = "SYSTEM", addedTime = os.time(), expireTime = nil}
    },
    staff = {"gabrjvo"}, 
    admins = {"gabrjvo"}, 
    keys = {}, 
    lastUpdate = os.time()
}

-- Variáveis globais
local Window = nil
local scriptExecuted = false

-- ========== FUNÇÕES CORE ==========
local function getRequestFunc() 
    return syn and syn.request or http_request or request or (fluxus and fluxus.request)
end

local function makeRequest(method, data)
    local requestFunc = getRequestFunc()
    if not requestFunc then 
        warn("❌ Request não disponível") 
        return nil 
    end
    
    local success, response = pcall(function()
        return requestFunc({
            Url = "https://api.jsonbin.io/v3/b/" .. CONFIG.BIN_ID .. (method == "GET" and "/latest" or ""),
            Method = method,
            Headers = {
                ["Content-Type"] = "application/json", 
                ["X-Master-Key"] = CONFIG.API_KEY
            },
            Body = data and HttpService:JSONEncode(data) or nil
        })
    end)
    
    if success and response and response.StatusCode == 200 then
        return HttpService:JSONDecode(response.Body)
    end
    warn("❌ Request falhou:", response and response.StatusCode or "No response")
    return nil
end

local function loadDB()
    print("🔄 Carregando dados...")
    local response = makeRequest("GET")
    if response and response.record then
        DB = response.record
        -- Garantir estrutura das keys
        if not DB.keys then DB.keys = {} end
        print("✅ Dados carregados:", #DB.users, "usuários,", #(DB.keys or {}), "keys")
        return true
    end
    print("⚠️ Usando dados padrão")
    return saveDB()
end

local function saveDB()
    print("💾 Salvando dados...") 
    DB.lastUpdate = os.time()
    local response = makeRequest("PUT", DB)
    if response then 
        print("✅ Dados salvos!") 
        return true 
    end
    warn("❌ Erro ao salvar") 
    return false
end

-- ========== FUNÇÕES DE TEMPO ==========
local function parseTimeString(timeStr)
    if not timeStr or timeStr == "" then return nil end
    local amount, unit = tonumber(timeStr:match("%d+")), timeStr:match("%a+"):lower()
    if not amount then return nil end
    local multipliers = {
        ["m"] = 60, ["min"] = 60, 
        ["h"] = 3600, ["hour"] = 3600, 
        ["d"] = 86400, ["day"] = 86400, 
        ["w"] = 604800, ["week"] = 604800
    }
    local multiplier = multipliers[unit] or multipliers[unit:sub(1, 1)]
    return multiplier and (amount * multiplier) or nil
end

local function formatTimeLeft(expireTime)
    if not expireTime then return "Permanente" end
    local timeLeft = expireTime - os.time()
    if timeLeft <= 0 then return "Expirado" end
    local days = math.floor(timeLeft / 86400)
    local hours = math.floor((timeLeft % 86400) / 3600)
    local minutes = math.floor((timeLeft % 3600) / 60)
    if days > 0 then return days .. "d " .. hours .. "h"
    elseif hours > 0 then return hours .. "h " .. minutes .. "m"
    else return minutes .. "m" end
end

-- ========== VERIFICAÇÕES ==========
local function cleanExpiredUsers()
    local cleaned = false
    for i = #DB.users, 1, -1 do
        local user = DB.users[i]
        if user.expireTime and os.time() >= user.expireTime then
            table.remove(DB.users, i) 
            cleaned = true
            print("🗑️ Removido usuário expirado:", user.username)
        end
    end
    -- Limpar keys expiradas também
    for userId, keyData in pairs(DB.keys or {}) do
        if keyData.expireTime and os.time() >= keyData.expireTime then
            DB.keys[userId] = nil 
            cleaned = true
            print("🗑️ Key expirada removida:", userId)
        end
    end
    if cleaned then saveDB() end
end

local function isWhitelisted(username)
    cleanExpiredUsers()
    for _, user in pairs(DB.users) do
        if user.username and user.username:lower() == username:lower() then 
            return true 
        end
    end
    return false
end

local function isStaff(username)
    for _, staff in pairs(DB.staff) do
        if staff:lower() == username:lower() then 
            return true 
        end
    end
    return false
end

local function isAdmin(username)
    for _, admin in pairs(DB.admins) do
        if admin:lower() == username:lower() then 
            return true 
        end
    end
    return false
end

-- ========== GERENCIAMENTO ==========
local function addUser(username, addedBy, timeString)
    if not username or username == "" then return false, "Nome inválido" end
    if isWhitelisted(username) then return false, "Já está na whitelist" end
    
    local expireTime = nil
    if timeString and timeString ~= "" then
        local seconds = parseTimeString(timeString)
        if seconds then 
            expireTime = os.time() + seconds
        else 
            return false, "Formato de tempo inválido (use: 10m, 5h, 2d, 1w)" 
        end
    end
    
    table.insert(DB.users, {
        username = username, 
        addedBy = addedBy or "SYSTEM", 
        addedTime = os.time(), 
        expireTime = expireTime
    })
    
    local success = saveDB()
    local timeText = expireTime and (" por " .. formatTimeLeft(expireTime)) or " permanentemente"
    return success, success and ("Adicionado" .. timeText) or "Erro ao salvar"
end

local function removeUser(username)
    if not username or username == "" then return false, "Nome inválido" end
    for i, user in pairs(DB.users) do
        if user.username and user.username:lower() == username:lower() then
            table.remove(DB.users, i)
            return saveDB(), saveDB() and "Removido da whitelist!" or "Erro ao salvar"
        end
    end
    return false, "Usuário não encontrado"
end

local function addStaff(username, addedBy)
    if not username or username == "" then return false, "Nome inválido" end
    if isStaff(username) then return false, "Já é staff" end
    if not isWhitelisted(username) then addUser(username, addedBy) end
    table.insert(DB.staff, username)
    return saveDB(), saveDB() and "Promovido a Staff!" or "Erro ao salvar"
end

local function removeStaff(username)
    if not username or username == "" then return false, "Nome inválido" end
    if username:lower() == "gabrjvo" then return false, "Não pode remover o admin principal" end
    for i, staff in pairs(DB.staff) do
        if staff:lower() == username:lower() then
            table.remove(DB.staff, i)
            return saveDB(), saveDB() and "Removido do Staff!" or "Erro ao salvar"
        end
    end
    return false, "Staff não encontrado"
end

local function addAdmin(username, addedBy)
    if not username or username == "" then return false, "Nome inválido" end
    if isAdmin(username) then return false, "Já é admin" end
    if not isWhitelisted(username) then addUser(username, addedBy) end
    if not isStaff(username) then table.insert(DB.staff, username) end
    table.insert(DB.admins, username)
    return saveDB(), saveDB() and "Promovido a Admin!" or "Erro ao salvar"
end

local function removeAdmin(username)
    if not username or username == "" then return false, "Nome inválido" end
    if username:lower() == "gabrjvo" then return false, "Não pode remover o admin principal" end
    for i, admin in pairs(DB.admins) do
        if admin:lower() == username:lower() then
            table.remove(DB.admins, i)
            return saveDB(), saveDB() and "Removido dos Admins!" or "Erro ao salvar"
        end
    end
    return false, "Admin não encontrado"
end

-- ========== SISTEMA DE KEYS PERSISTENTE ==========
local function generateKey()
    local chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789"
    local key = "MITRA-"
    for i = 1, 12 do 
        key = key .. chars:sub(math.random(1, #chars), math.random(1, #chars)) 
    end
    return key
end

local function createKey(userId)
    if not DB.keys then DB.keys = {} end
    local key = generateKey()
    local expireTime = os.time() + (48 * 3600)
    
    DB.keys[tostring(userId)] = {
        key = key, 
        expireTime = expireTime, 
        createdTime = os.time()
    }
    saveDB() -- Salvar imediatamente para persistir
    return key, expireTime
end

local function isKeyValid(userId)
    if not DB.keys then return false end
    local data = DB.keys[tostring(userId)]
    return data and os.time() < data.expireTime
end

local function validateKey(inputKey, userId)
    if not DB.keys then return false, "Sistema de keys não inicializado" end
    local data = DB.keys[tostring(userId)]
    if not data then return false, "Key não encontrada para este usuário" end
    if not isKeyValid(userId) then return false, "Key expirada" end
    if data.key ~= inputKey then return false, "Key incorreta" end
    return true, "Key válida"
end

-- ========== LOGS E NOTIFICAÇÕES ==========
local function sendLog(title, description)
    spawn(function()
        pcall(function()
            local requestFunc = getRequestFunc()
            if not requestFunc then return end
            requestFunc({
                Url = CONFIG.WEBHOOK, 
                Method = "POST",
                Headers = {["Content-Type"] = "application/json"},
                Body = HttpService:JSONEncode({
                    embeds = {{
                        title = title, 
                        description = description, 
                        color = 3447003,
                        fields = {
                            {name = "👤 Jogador", value = LocalPlayer.Name, inline = true},
                            {name = "⏰ Horário", value = os.date("%d/%m/%Y %H:%M"), inline = true}
                        }
                    }}
                })
            })
        end)
    end)
end

local function notify(title, text)
    pcall(function() 
        game.StarterGui:SetCore("SendNotification", {
            Title = title, 
            Text = text, 
            Duration = 3
        }) 
    end)
end

-- ========== PROTEÇÃO ANTI-CHEAT ==========
local function antiCheat()
    spawn(function()
        pcall(function()
            for _, svc in pairs({"ReplicatedStorage", "ServerStorage"}) do
                local service = game:FindService(svc)
                if service then
                    for _, obj in pairs(service:GetDescendants()) do
                        if obj.Name:lower():find("admin") or obj.Name:lower():find("anti") then 
                            obj:Destroy() 
                        end
                    end
                end
            end
            if getrawmetatable then
                local mt = getrawmetatable(game)
                local oldNamecall = mt.__namecall
                setreadonly(mt, false)
                mt.__namecall = function(self, ...)
                    local method = getnamecallmethod()
                    if method == "FireServer" and self.Name:lower():find("admin") then 
                        return 
                    end
                    return oldNamecall(self, ...)
                end
                setreadonly(mt, true)
            end
        end)
    end)
end

-- ========== EXECUÇÃO DO SCRIPT PRINCIPAL - CORRIGIDO ==========
local function executeMainScript()
    if scriptExecuted then 
        print("⚠️ Script já executado!")
        return 
    end
    
    scriptExecuted = true
    
    spawn(function()
        local success, result = pcall(function()
            notify("🔄 Carregando", "Baixando script principal...")
            print("🔄 Executando script principal...")
            
            -- Executar o script diretamente
            loadstring(game:HttpGet("https://raw.githubusercontent.com/Gabriel210-lang/Mitra-Menu/refs/heads/main/Mitra.md"))()
            
            return true
        end)
        
        if success then
            print("✅ Script principal executado com sucesso!")
            notify("✅ Sucesso", "Mitra Menu carregado!")
            sendLog("🚀 Script Executado", LocalPlayer.Name .. " executou o menu principal com sucesso")
        else
            print("❌ Erro ao executar script principal:", result)
            notify("❌ Erro", "Falha na execução do script")
            scriptExecuted = false -- Resetar para permitir nova tentativa
        end
    end)
end

-- ========== INTERFACE ==========
local function loadRayfield()
    local urls = {
        "https://sirius.menu/rayfield", 
        "https://raw.githubusercontent.com/UI-Interface/CustomFIeld/main/RayField.lua"
    }
    
    for _, url in pairs(urls) do
        local success, content = pcall(game.HttpGet, game, url)
        if success and content and #content > 1000 then
            local loadSuccess, lib = pcall(loadstring(content))
            if loadSuccess and lib then 
                return lib 
            end
        end
    end
    return nil
end

local function showList(listType)
    local message = ""
    if listType == "users" then
        message = "📋 **WHITELIST (" .. #DB.users .. " usuários)**\n\n"
        for i, user in pairs(DB.users) do
            message = message .. i .. ". **" .. user.username .. "** (" .. formatTimeLeft(user.expireTime) .. ")\n"
        end
    elseif listType == "staff" then
        message = "👮 **STAFF (" .. #DB.staff .. " membros)**\n\n"
        for i, staff in pairs(DB.staff) do 
            message = message .. i .. ". **" .. staff .. "**\n" 
        end
    elseif listType == "admins" then
        message = "👑 **ADMINS (" .. #DB.admins .. " membros)**\n\n"
        for i, admin in pairs(DB.admins) do 
            message = message .. i .. ". **" .. admin .. "**\n" 
        end
    end
    sendLog("📋 Lista " .. listType:upper(), message) 
    notify("📋 Lista Enviada", "Verifique o Discord!")
end

local function createInterface()
    local Rayfield = loadRayfield()
    if not Rayfield then 
        notify("❌ ERRO", "Falha ao carregar interface!") 
        return false 
    end
    
    Window = Rayfield:CreateWindow({
        Name = "🔐 Mitra Menu V3.0 - Sistema Completo",
        LoadingTitle = "Conectando...", 
        LoadingSubtitle = "Sistema Global Ativo", 
        KeySystem = false
    })
    
    local AuthTab = Window:CreateTab("🔑 Autenticação", 4483362458)
    local keyInput = ""
    
    -- Status dinâmico
    local statusText = isKeyValid(LocalPlayer.UserId) and "✅ Key ativa" or "❌ Key necessária"
    AuthTab:CreateLabel(statusText)
    
    AuthTab:CreateInput({
        Name = "🔑 Digite sua Key", 
        CurrentValue = "", 
        PlaceholderText = "MITRA-XXXXXXXXXXXX",
        RemoveTextAfterFocusLost = false, 
        Flag = "KeyInput",
        Callback = function(text) 
            keyInput = text 
        end,
    })
    
    AuthTab:CreateButton({
        Name = "🔗 Obter Key (48h)",
        Callback = function()
            if isKeyValid(LocalPlayer.UserId) then 
                notify("ℹ️ Info", "Você já tem uma key!") 
                return 
            end
            local newKey = createKey(LocalPlayer.UserId)
            sendLog("🔑 Nova Key Gerada", "**Usuário:** " .. LocalPlayer.Name .. "\n**Key:** `" .. newKey .. "`\n**Válida por:** 48 horas")
            notify("📨 Enviado!", "Key enviada para Discord!")
        end,
    })
    
    AuthTab:CreateButton({
        Name = "✅ Verificar Key",
        Callback = function()
            if keyInput == "" then 
                notify("⚠️ Aviso", "Digite uma key!") 
                return 
            end
            
            -- Recarregar dados antes de validar para garantir sincronização
            loadDB()
            local valid, msg = validateKey(keyInput, LocalPlayer.UserId)
            
            if valid then
                notify("✅ Sucesso!", "Key válida! Executando script...")
                sendLog("🎯 Acesso Autorizado", "**Usuário:** " .. LocalPlayer.Name .. "\n**Key:** `" .. keyInput .. "`\n**Status:** Verificado com sucesso")
                
                -- Executar script principal sem destruir a interface
                wait(1)
                executeMainScript()
            else
                notify("❌ Erro", msg)
                print("Debug - Key digitada:", keyInput)
                print("Debug - User ID:", LocalPlayer.UserId)
                if DB.keys then
                    for userId, keyData in pairs(DB.keys) do
                        print("Debug - Key no DB:", userId, keyData.key)
                    end
                end
            end
        end,
    })
    
    -- ========== PAINEL STAFF ==========
    if isStaff(LocalPlayer.Name) then
        local StaffTab = Window:CreateTab("👮 Staff Panel", 4483362458)
        local userInput, removeInput, timeInput = "", "", ""
        
        StaffTab:CreateSection("👤 Gerenciar Whitelist")
        
        StaffTab:CreateInput({
            Name = "➕ Nome do usuário", 
            CurrentValue = "", 
            PlaceholderText = "Digite o nome...", 
            RemoveTextAfterFocusLost = false, 
            Callback = function(text) 
                userInput = text 
            end
        })
        
        StaffTab:CreateInput({
            Name = "⏰ Tempo (opcional)", 
            CurrentValue = "", 
            PlaceholderText = "Ex: 10d, 5h, 30m (vazio = permanente)", 
            RemoveTextAfterFocusLost = false, 
            Callback = function(text) 
                timeInput = text 
            end
        })
        
        StaffTab:CreateButton({
            Name = "✅ Adicionar à Whitelist", 
            Callback = function()
                local success, msg = addUser(userInput, LocalPlayer.Name, timeInput)
                notify(success and "✅ Sucesso" or "❌ Erro", msg)
                if success then
                    local timeText = timeInput ~= "" and (" - Tempo: " .. timeInput) or " - Permanente"
                    sendLog("👤 Usuário Adicionado", userInput .. " por " .. LocalPlayer.Name .. timeText)
                end
            end
        })
        
        StaffTab:CreateInput({
            Name = "🗑️ Remover usuário", 
            CurrentValue = "", 
            PlaceholderText = "Nome para remover...", 
            RemoveTextAfterFocusLost = false, 
            Callback = function(text) 
                removeInput = text 
            end
        })
        
        StaffTab:CreateButton({
            Name = "❌ Remover da Whitelist", 
            Callback = function()
                local success, msg = removeUser(removeInput)
                notify(success and "✅ Sucesso" or "❌ Erro", msg)
                if success then 
                    sendLog("🗑️ Usuário Removido", removeInput .. " por " .. LocalPlayer.Name) 
                end
            end
        })
        
        StaffTab:CreateSection("📋 Visualizar Listas")
        
        StaffTab:CreateButton({
            Name = "📋 Ver Whitelist", 
            Callback = function() 
                showList("users") 
            end
        })
        
        StaffTab:CreateButton({
            Name = "👮 Ver Staff", 
            Callback = function() 
                showList("staff") 
            end
        })
        
        StaffTab:CreateButton({
            Name = "🔄 Sincronizar Dados", 
            Callback = function()
                local success = loadDB() 
                notify(success and "✅ Sincronizado" or "❌ Erro", "")
            end
        })
    end
    
    -- ========== PAINEL ADMIN ==========
    if isAdmin(LocalPlayer.Name) then
        local AdminTab = Window:CreateTab("👑 Admin Panel", 4483362458)
        local staffInput = ""
        
        AdminTab:CreateSection("👮 Gerenciar Staff")
        
        AdminTab:CreateInput({
            Name = "👮 Nome do usuário", 
            CurrentValue = "", 
            PlaceholderText = "Digite o nome...", 
            RemoveTextAfterFocusLost = false, 
            Callback = function(text) 
                staffInput = text 
            end
        })
        
        AdminTab:CreateButton({
            Name = "⭐ Promover a Staff", 
            Callback = function()
                local success, msg = addStaff(staffInput, LocalPlayer.Name)
                notify(success and "✅ Sucesso" or "❌ Erro", msg)
                if success then 
                    sendLog("⭐ Staff Promovido", staffInput .. " por " .. LocalPlayer.Name) 
                end
            end
        })
        
        AdminTab:CreateButton({
            Name = "🔻 Remover Staff", 
            Callback = function()
                local success, msg = removeStaff(staffInput)
                notify(success and "✅ Sucesso" or "❌ Erro", msg)
                if success then 
                    sendLog("🔻 Staff Removido", staffInput .. " por " .. LocalPlayer.Name) 
                end
            end
        })
        
        AdminTab:CreateSection("👑 Gerenciar Admins")
        
        AdminTab:CreateButton({
            Name = "👑 Promover a Admin", 
            Callback = function()
                local success, msg = addAdmin(staffInput, LocalPlayer.Name)
                notify(success and "✅ Sucesso" or "❌ Erro", msg)
                if success then 
                    sendLog("👑 Admin Promovido", staffInput .. " por " .. LocalPlayer.Name) 
                end
            end
        })
        
        AdminTab:CreateButton({
            Name = "👑 Remover Admin", 
            Callback = function()
                local success, msg = removeAdmin(staffInput)
                notify(success and "✅ Sucesso" or "❌ Erro", msg)
                if success then 
                    sendLog("👑 Admin Removido", staffInput .. " por " .. LocalPlayer.Name) 
                end
            end
        })
        
        AdminTab:CreateSection("📋 Listas Administrativas")
        
        AdminTab:CreateButton({
            Name = "👑 Ver Admins", 
            Callback = function() 
                showList("admins") 
            end
        })
        
        AdminTab:CreateButton({
            Name = "🧹 Limpar Expirados", 
            Callback = function() 
                cleanExpiredUsers() 
                notify("🧹 Limpeza", "Usuários expirados removidos!") 
            end
        })
    end
    
    -- Info
    AuthTab:CreateSection("ℹ️ Informações do Sistema")
    AuthTab:CreateLabel("👤 Usuário: " .. LocalPlayer.Name)
    AuthTab:CreateLabel("📊 Total WL: " .. #DB.users)
    AuthTab:CreateLabel("👮 Total Staff: " .. #DB.staff)
    AuthTab:CreateLabel("👑 Total Admins: " .. #DB.admins)
    
    if isStaff(LocalPlayer.Name) then 
        AuthTab:CreateLabel("🔰 Seu Nível: Staff") 
    end
    if isAdmin(LocalPlayer.Name) then 
        AuthTab:CreateLabel("👑 Seu Nível: Admin") 
    end
    
    return true
end

-- ========== EXECUÇÃO PRINCIPAL ==========
local function main()
    print("🚀 Iniciando Mitra Menu V3.0...")
    antiCheat()
    
    if not loadDB() then 
        notify("❌ ERRO", "Falha na conexão com o banco de dados!") 
        return 
    end
    
    if not isWhitelisted(LocalPlayer.Name) then 
        notify("🚫 NEGADO", "Você não está autorizado!") 
        return 
    end
    
    if not createInterface() then 
        notify("❌ ERRO", "Falha ao criar interface!")
        return 
    end
    
    print("✅ Sistema iniciado com sucesso!") 
    sendLog("🌐 Conexão", LocalPlayer.Name .. " conectou-se ao sistema")
end

-- Executar
spawn(main)
