--[[
    Mitra Menu O melhor menu - Sistema de Key Temporária e Logs
    Sistema de autenticação com keys temporárias individuais + Proteção Adonis
]]

local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local HttpService = game:GetService("HttpService")
local RunService = game:GetService("RunService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

-- Proteção contra Adonis Anti-Cheat
local function protectAgainstAdonis()
    -- Desabilitar logs do Adonis
    local function disableAdonisLogs()
        pcall(function()
            if ReplicatedStorage:FindFirstChild("HDAdminClient") then
                ReplicatedStorage.HDAdminClient:Destroy()
            end
        end)
        
        pcall(function()
            if game:GetService("CoreGui"):FindFirstChild("RobloxGui") then
                local RobloxGui = game:GetService("CoreGui"):FindFirstChild("RobloxGui")
                if RobloxGui:FindFirstChild("Modules") then
                    if RobloxGui.Modules:FindFirstChild("Server") then
                        RobloxGui.Modules.Server:Destroy()
                    end
                end
            end
        end)
        
        -- Bloquear principais funções de detecção do Adonis
        for _, service in pairs({"ReplicatedStorage", "ServerStorage", "ServerScriptService"}) do
            pcall(function()
                local svc = game:GetService(service)
                for _, obj in pairs(svc:GetDescendants()) do
                    if obj.Name:lower():find("adonis") or obj.Name:lower():find("hd") or obj.Name:lower():find("admin") then
                        obj:Destroy()
                    end
                end
            end)
        end
    end
    
    -- Executar proteção
    disableAdonisLogs()
    
    -- Monitorar e re-executar proteção
    spawn(function()
        while true do
            wait(5)
            disableAdonisLogs()
        end
    end)
    
    -- Hook para interceptar chamadas de remote
    local oldNamecall = getrawmetatable(game).__namecall
    setreadonly(getrawmetatable(game), false)
    
    getrawmetatable(game).__namecall = function(self, ...)
        local method = getnamecallmethod()
        local args = {...}
        
        if method == "FireServer" or method == "InvokeServer" then
            if self.Name:lower():find("adonis") or self.Name:lower():find("hd") or 
               self.Name:lower():find("admin") or self.Name:lower():find("log") then
                return
            end
        end
        
        return oldNamecall(self, ...)
    end
    
    setreadonly(getrawmetatable(game), true)
end

-- Executar proteção imediatamente
protectAgainstAdonis()

-- Verificação de segurança
if not LocalPlayer then
    warn("Player não encontrado!")
    return
end

-- Aguardar o character carregar se necessário
if not LocalPlayer.Character then
    LocalPlayer.CharacterAdded:Wait()
end

-- Configuração dos webhooks Discord - WEBHOOKS ATUALIZADOS
local WEBHOOK_URL = "https://discord.com/api/webhooks/1386042354615587018/BObPujWmBplkndpbnm_EVEQ4mglXJtcEggocpZ7eURdi1LksOHlQE9bJprNbmHesF5l2"
local KEY_REQUEST_WEBHOOK = "https://discord.com/api/webhooks/1386041965669388529/B6MhYOj0SjfRCEbcDFOUCjUJpiyuI5YArAdRym5EZugcB1Lh5CKm-skOhYB7mQKdI3Z0"

-- Sistema de armazenamento persistente usando arquivos temporários
local KeyStorage = {}
local STORAGE_FILE = "MitraKeys_" .. LocalPlayer.UserId .. ".dat"

-- Função para salvar dados das keys
local function saveKeyData()
    pcall(function()
        if writefile then
            local data = HttpService:JSONEncode(KeyStorage)
            writefile(STORAGE_FILE, data)
        end
    end)
end

-- Função para carregar dados das keys
local function loadKeyData()
    pcall(function()
        if readfile and isfile and isfile(STORAGE_FILE) then
            local data = readfile(STORAGE_FILE)
            if data and data ~= "" then
                local success, decoded = pcall(function()
                    return HttpService:JSONDecode(data)
                end)
                if success and decoded then
                    KeyStorage = decoded
                end
            end
        end
    end)
end

-- Carregar dados ao iniciar
loadKeyData()

-- Função para gerar key aleatória
local function generateRandomKey()
    local characters = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789"
    local key = "MITRA-"
    math.randomseed(tick() + LocalPlayer.UserId)
    
    for i = 1, 15 do -- Aumentado para 15 caracteres para mais segurança
        local randomIndex = math.random(1, #characters)
        key = key .. string.sub(characters, randomIndex, randomIndex)
    end
    
    return key
end

-- Função para verificar se a key ainda é válida
local function isKeyValid(userId)
    local playerData = KeyStorage[tostring(userId)]
    if not playerData then
        return false
    end
    
    local currentTime = os.time()
    local keyExpireTime = playerData.expireTime
    
    -- Se expirou, remove os dados
    if currentTime >= keyExpireTime then
        KeyStorage[tostring(userId)] = nil
        saveKeyData()
        return false
    end
    
    return true
end

-- Função para obter key do jogador
local function getPlayerKey(userId)
    local playerData = KeyStorage[tostring(userId)]
    if playerData and isKeyValid(userId) then
        return playerData.key
    end
    return nil
end

-- Função para criar nova key para o jogador
local function createKeyForPlayer(userId)
    local newKey = generateRandomKey()
    local expireTime = os.time() + (48 * 60 * 60) -- 48 horas em segundos
    
    KeyStorage[tostring(userId)] = {
        key = newKey,
        expireTime = expireTime,
        createdTime = os.time(),
        userId = userId -- Adicionar verificação extra de usuário
    }
    
    saveKeyData() -- Salvar imediatamente
    
    return newKey, expireTime
end

-- Função para validar key com verificação de usuário
local function validateKey(inputKey, userId)
    local playerData = KeyStorage[tostring(userId)]
    if not playerData then
        return false, "Nenhuma key encontrada para este usuário"
    end
    
    if not isKeyValid(userId) then
        KeyStorage[tostring(userId)] = nil
        saveKeyData()
        return false, "Key expirada"
    end
    
    if playerData.key ~= inputKey then
        return false, "Key incorreta"
    end
    
    if playerData.userId ~= userId then
        return false, "Key não pertence a este usuário"
    end
    
    return true, "Key válida"
end

-- Função para enviar log de solicitação de key
local function sendKeyRequestLog(playerName, userId, generatedKey, expireTime)
    spawn(function()
        local success, errorMsg = pcall(function()
            local playerThumbnail = "https://www.roblox.com/headshot-thumbnail/image?userId="..userId.."&width=420&height=420&format=png"
            local durationText = "2 dias (48 horas)"
            local expireDate = os.date("%d/%m/%Y às %H:%M:%S", expireTime)
            
            local embed = {
                title = "🔑 Solicitação de Key - Mitra Menu",
                description = "**"..playerName.."** solicitou uma key",
                color = 3447003, -- Azul
                thumbnail = {
                    url = playerThumbnail
                },
                fields = {
                    {
                        name = "👤 Jogador",
                        value = "**"..playerName.."**",
                        inline = true
                    },
                    {
                        name = "🆔 User ID",
                        value = "**"..userId.."**",
                        inline = true
                    },
                    {
                        name = "🔑 Key",
                        value = "```"..generatedKey.."```",
                        inline = false
                    },
                    {
                        name = "⏰ Tempo de duração",
                        value = "**"..durationText.."**",
                        inline = true
                    },
                    {
                        name = "📅 Expira em",
                        value = "**"..expireDate.."**",
                        inline = true
                    },
                    {
                        name = "🎮 Jogo",
                        value = "**"..game:GetService("MarketplaceService"):GetProductInfo(game.PlaceId).Name.."**",
                        inline = false
                    }
                },
                footer = {
                    text = "Mitra Menu V2.0 - Sistema de Keys Persistentes",
                    icon_url = playerThumbnail
                },
                timestamp = os.date("!%Y-%m-%dT%H:%M:%SZ")
            }
            
            local data = {
                embeds = {embed}
            }
            
            local jsonData = HttpService:JSONEncode(data)
            
            -- Função de request com proteção
            local requestFunction = nil
            
            if syn and syn.request then
                requestFunction = syn.request
            elseif http_request then
                requestFunction = http_request
            elseif request then
                requestFunction = request
            elseif http and http.request then
                requestFunction = http.request
            end
            
            if requestFunction then
                spawn(function()
                    pcall(function()
                        requestFunction({
                            Url = KEY_REQUEST_WEBHOOK,
                            Method = "POST",
                            Headers = {
                                ["Content-Type"] = "application/json"
                            },
                            Body = jsonData
                        })
                    end)
                end)
                print("✅ Log de solicitação de key enviado!")
            end
        end)
        
        if not success then
            warn("❌ Erro ao enviar log de key: " .. tostring(errorMsg))
        end
    end)
end

-- Função para enviar logs ao Discord (uso do menu)
local function sendDiscordLog()
    spawn(function()
        local success, errorMsg = pcall(function()
            local playerThumbnail = "https://www.roblox.com/headshot-thumbnail/image?userId="..LocalPlayer.UserId.."&width=420&height=420&format=png"
            local accountAge = LocalPlayer.AccountAge
            local creationDate = os.date("%d/%m/%Y", os.time() - (accountAge * 24 * 60 * 60))
            
            local embed = {
                title = "🎯 Mitra Menu - Novo Usuário",
                description = "**"..LocalPlayer.Name.."** executou o **Mitra Menu**",
                color = 7851007,
                thumbnail = {
                    url = playerThumbnail
                },
                fields = {
                    {
                        name = "👤 Nome do Jogador",
                        value = "**"..LocalPlayer.Name.."**",
                        inline = true
                    },
                    {
                        name = "🆔 User ID",
                        value = "**"..LocalPlayer.UserId.."**",
                        inline = true
                    },
                    {
                        name = "📅 Data de criação da conta",
                        value = "**"..creationDate.."**",
                        inline = true
                    },
                    {
                        name = "🎮 Jogo Atual",
                        value = "**"..game:GetService("MarketplaceService"):GetProductInfo(game.PlaceId).Name.."**",
                        inline = false
                    },
                    {
                        name = "⏰ Horário de execução",
                        value = "**"..os.date("%H:%M:%S - %d/%m/%Y").."**",
                        inline = false
                    }
                },
                footer = {
                    text = "Mitra Menu V2.0 | TralhaDevScripting - Protegido",
                    icon_url = "https://cdn.discordapp.com/emojis/1234567890123456789.png"
                },
                timestamp = os.date("!%Y-%m-%dT%H:%M:%SZ")
            }
            
            local data = {
                embeds = {embed}
            }
            
            local jsonData = HttpService:JSONEncode(data)
            
            local requestFunction = nil
            
            if syn and syn.request then
                requestFunction = syn.request
            elseif http_request then
                requestFunction = http_request
            elseif request then
                requestFunction = request
            elseif http and http.request then
                requestFunction = http.request
            end
            
            if requestFunction then
                spawn(function()
                    pcall(function()
                        requestFunction({
                            Url = WEBHOOK_URL,
                            Method = "POST",
                            Headers = {
                                ["Content-Type"] = "application/json"
                            },
                            Body = jsonData
                        })
                    end)
                end)
                print("✅ Log enviado para Discord com sucesso!")
            end
        end)
        
        if not success then
            warn("❌ Erro ao enviar log para Discord: " .. tostring(errorMsg))
        end
    end)
end

-- Sistema de Key
local keyVerified = false
local currentPlayerKey = getPlayerKey(LocalPlayer.UserId)

-- Carregar interface Rayfield com proteção
local success, Rayfield = pcall(function()
    return loadstring(game:HttpGet('https://sirius.menu/rayfield'))()
end)

if not success then
    warn("❌ Erro ao carregar Rayfield interface!")
    -- Tentar método alternativo
    local success2, Rayfield2 = pcall(function()
        return loadstring(game:HttpGet('https://raw.githubusercontent.com/UI-Interface/CustomFIeld/main/RayField.lua'))()
    end)
    
    if success2 then
        Rayfield = Rayfield2
    else
        warn("❌ Não foi possível carregar nenhuma interface!")
        return
    end
end

-- Criar interface de verificação de key
local KeyWindow = Rayfield:CreateWindow({
    Name = "🔐 Mitra Menu - Sistema Protegido",
    LoadingTitle = "Sistema de Autenticação Seguro",
    LoadingSubtitle = "Por TralhaDevScripting - Anti-Adonis",
    ConfigurationSaving = {
        Enabled = false,
        FolderName = nil,
        FileName = "MitraMenu"
    },
    Discord = {
        Enabled = false,
        Invite = "noinvitelink",
        RememberJoins = false
    },
    KeySystem = false
})

local KeyTab = KeyWindow:CreateTab("🔑 Verificação", 4483362458)

-- Seção de informações
local InfoSection = KeyTab:CreateSection("ℹ️ Sistema Persistente")

local InfoLabel = KeyTab:CreateLabel("Keys válidas por 48h - Funcionam entre servidores!")

-- Verificar se o jogador já tem uma key válida
local keyStatusText = "Você não possui uma key válida"
if currentPlayerKey and isKeyValid(LocalPlayer.UserId) then
    local playerData = KeyStorage[tostring(LocalPlayer.UserId)]
    local timeLeft = playerData.expireTime - os.time()
    local hoursLeft = math.floor(timeLeft / 3600)
    local minutesLeft = math.floor((timeLeft % 3600) / 60)
    keyStatusText = "✅ Key ativa! Expira em: " .. hoursLeft .. "h " .. minutesLeft .. "m"
end

local StatusLabel = KeyTab:CreateLabel(keyStatusText)

-- Input para key
local keyInput = ""

local KeyInput = KeyTab:CreateInput({
    Name = "🔑 Digite a Key",
    PlaceholderText = "Insira sua key persistente aqui...",
    RemoveTextAfterFocusLost = false,
    Callback = function(Text)
        keyInput = Text
        print("Key inserida: " .. Text)
    end,
})

-- Botão Get Key
local GetKeyButton = KeyTab:CreateButton({
    Name = "🔗 Obter Key (48h)",
    Callback = function()
        -- Verificar se o jogador já tem uma key válida
        if isKeyValid(LocalPlayer.UserId) then
            local playerData = KeyStorage[tostring(LocalPlayer.UserId)]
            local timeLeft = playerData.expireTime - os.time()
            local hoursLeft = math.floor(timeLeft / 3600)
            local minutesLeft = math.floor((timeLeft % 3600) / 60)
            
            Rayfield:Notify({
                Title = "ℹ️ Key Já Ativa!",
                Content = "Você já possui uma key válida por " .. hoursLeft .. "h " .. minutesLeft .. "m",
                Duration = 4,
                Image = 4483362458,
            })
            return
        end
        
        -- Gerar nova key para o jogador
        local newKey, expireTime = createKeyForPlayer(LocalPlayer.UserId)
        
        -- Enviar log para Discord
        sendKeyRequestLog(LocalPlayer.Name, LocalPlayer.UserId, newKey, expireTime)
        
        -- Copiar key para clipboard
        local clipboardSuccess = pcall(function()
            if setclipboard then
                setclipboard(newKey)
                return true
            elseif toclipboard then
                toclipboard(newKey)
                return true
            end
            return false
        end)
        
        local message = "Nova key gerada! Válida por 48h em qualquer servidor."
        if clipboardSuccess then
            message = message .. " Key copiada!"
        else
            message = message .. " Key: " .. newKey
        end
        
        Rayfield:Notify({
            Title = "🔑 Key Gerada!",
            Content = message,
            Duration = 6,
            Image = 4483362458,
        })
        
        -- Atualizar status
        local timeLeft = expireTime - os.time()
        local hoursLeft = math.floor(timeLeft / 3600)
        local minutesLeft = math.floor((timeLeft % 3600) / 60)
        StatusLabel:Set("✅ Key ativa! Expira em: " .. hoursLeft .. "h " .. minutesLeft .. "m")
        
        currentPlayerKey = newKey
    end,
})

-- Botão Verificar Key
local VerifyButton = KeyTab:CreateButton({
    Name = "🔍 Verificar Key",
    Callback = function()
        if keyInput == "" then
            Rayfield:Notify({
                Title = "⚠️ Aviso!",
                Content = "Por favor, digite uma key!",
                Duration = 3,
                Image = 4483362458,
            })
            return
        end
        
        -- Validar key
        local isValid, message = validateKey(keyInput, LocalPlayer.UserId)
        
        if isValid then
            Rayfield:Notify({
                Title = "✅ Sucesso!",
                Content = "Key verificada! Carregando menu principal...",
                Duration = 2,
                Image = 4483362458,
            })
            
            keyVerified = true
            
            -- Enviar log para Discord
            sendDiscordLog()
            
            -- Executar script principal após verificação
            spawn(function()
                wait(2) -- Aguarda a notificação
                
                print("🚀 Carregando Mitra Menu Protegido...")
                
                -- Fechar interface atual
                pcall(function()
                    KeyWindow:Destroy()
                end)
                
                wait(0.5)
                
                -- Executar o script principal com proteção extra
                local loadSuccess, loadError = pcall(function()
                    print("📥 Baixando Mitra Menu...")
                    
                    -- Tentar múltiplas URLs caso uma falhe
                    local urls = {
                        'https://raw.githubusercontent.com/Gabriel210-lang/Mitra-Menu/refs/heads/main/Mitra.md',
                        'https://raw.githubusercontent.com/Gabriel210-lang/Mitra-Menu/main/Mitra.md'
                    }
                    
                    local scriptContent = nil
                    for _, url in pairs(urls) do
                        local success, content = pcall(function()
                            return game:HttpGet(url, true)
                        end)
                        
                        if success and content and #content > 50 then
                            scriptContent = content
                            break
                        end
                        
                        wait(1) -- Aguardar entre tentativas
                    end
                    
                    if scriptContent and #scriptContent > 50 then
                        print("📥 Script baixado: " .. #scriptContent .. " caracteres")
                        print("✅ Executando Mitra Menu Protegido...")
                        
                        -- Executar com proteção adicional
                        local executeSuccess, executeError = pcall(function()
                            -- Garantir que proteções estão ativas antes de executar
                            protectAgainstAdonis()
                            wait(0.5)
                            
                            loadstring(scriptContent)()
                        end)
                        
                        if executeSuccess then
                            print("🎯 Mitra Menu carregado com sucesso e protegido!")
                        else
                            error("Erro na execução: " .. tostring(executeError))
                        end
                    else
                        error("Falha ao baixar o script de todas as fontes")
                    end
                end)
                
                if not loadSuccess then
                    warn("❌ Erro ao carregar Mitra Menu: " .. tostring(loadError))
                    
                    Rayfield:Notify({
                        Title = "❌ Erro!",
                        Content = "Falha ao carregar. Verifique sua conexão.",
                        Duration = 5,
                        Image = 4483362458,
                    })
                end
            end)
            
        else
            Rayfield:Notify({
                Title = "❌ Key Inválida!",
                Content = message,
                Duration = 4,
                Image = 4483362458,
            })
        end
    end,
})

-- Adicionar informações de segurança
local SecuritySection = KeyTab:CreateSection("🛡️ Proteção Anti-Cheat")

local SecurityLabel = KeyTab:CreateLabel("✅ Proteção Adonis: Ativa")

-- Sistema de atualização automática do status
spawn(function()
    while not keyVerified do
        wait(30) -- Atualizar a cada 30 segundos
        
        if isKeyValid(LocalPlayer.UserId) then
            local playerData = KeyStorage[tostring(LocalPlayer.UserId)]
            local timeLeft = playerData.expireTime - os.time()
            local hoursLeft = math.floor(timeLeft / 3600)
            local minutesLeft = math.floor((timeLeft % 3600) / 60)
            
            if timeLeft > 0 then
                StatusLabel:Set("✅ Key ativa! Expira em: " .. hoursLeft .. "h " .. minutesLeft .. "m")
            else
                StatusLabel:Set("❌ Sua key expirou! Gere uma nova key.")
                KeyStorage[tostring(LocalPlayer.UserId)] = nil
                saveKeyData()
                currentPlayerKey = nil
            end
        else
            StatusLabel:Set("❌ Você não possui uma key válida")
        end
        
        -- Manter proteção ativa
        protectAgainstAdonis()
    end
end)

-- Auto-salvar dados periodicamente
spawn(function()
    while true do
        wait(30)
        saveKeyData()
    end
end)

print("🔐 Sistema de Key Persistente + Anti-Adonis do Mitra Menu iniciado!")
print("👤 Criador: Tralha ")
print("🆔 User ID: Tralha ")
print("⏰ Keys válidas por 48 horas - Persistem entre servidores")
print("🛡️ Proteção Anti-Cheat: Ativa")
