--[[
    Mitra Menu O melhor menu - Sistema de Key Temporária e Logs
    Sistema de autenticação com keys temporárias individuais
]]

local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local HttpService = game:GetService("HttpService")
local DataStoreService = game:GetService("DataStoreService")

-- Verificação de segurança
if not LocalPlayer then
    warn("Player não encontrado!")
    return
end

-- Aguardar o character carregar se necessário
if not LocalPlayer.Character then
    LocalPlayer.CharacterAdded:Wait()
end

-- Configuração dos webhooks Discord
local WEBHOOK_URL = "https://discord.com/api/webhooks/1383078429968302080/7-m1myy5yHREP6bju1uxgw17wQv979BdBtQhueAgsEZcIqIYArn4UqLfbayBSCcq8cUJ"
local KEY_REQUEST_WEBHOOK = "https://discord.com/api/webhooks/1384197487027556422/Y9Rlx15njkGCsxUJ4fzJqSRkL5Oe3UQ1Y5WQ3SRlZv57tQzLxpjTujAhgYAar_X4mc0f"

-- Sistema de armazenamento de keys (simulação de DataStore local)
local KeyStorage = {}

-- Função para gerar key aleatória
local function generateRandomKey()
    local characters = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789"
    local key = "MITRA-"
    
    for i = 1, 12 do
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
    
    return currentTime < keyExpireTime
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
        createdTime = os.time()
    }
    
    return newKey, expireTime
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
                    }
                },
                footer = {
                    text = "Mitra Menu V2.0 - Sistema de Keys",
                    icon_url = playerThumbnail
                },
                timestamp = os.date("!%Y-%m-%dT%H:%M:%SZ")
            }
            
            local data = {
                embeds = {embed}
            }
            
            local jsonData = HttpService:JSONEncode(data)
            
            -- Função de request
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
                requestFunction({
                    Url = KEY_REQUEST_WEBHOOK,
                    Method = "POST",
                    Headers = {
                        ["Content-Type"] = "application/json"
                    },
                    Body = jsonData
                })
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
                    text = "Mitra Menu V2.0 | TralhaDevScripting",
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
                requestFunction({
                    Url = WEBHOOK_URL,
                    Method = "POST",
                    Headers = {
                        ["Content-Type"] = "application/json"
                    },
                    Body = jsonData
                })
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

-- Carregar interface Rayfield
local success, Rayfield = pcall(function()
    return loadstring(game:HttpGet('https://sirius.menu/rayfield'))()
end)

if not success then
    warn("❌ Erro ao carregar Rayfield interface!")
    return
end

-- Criar interface de verificação de key
local KeyWindow = Rayfield:CreateWindow({
    Name = "🔐 Mitra Menu - Verificação de Key",
    LoadingTitle = "Sistema de Autenticação",
    LoadingSubtitle = "Por TralhaDevScripting",
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
local InfoSection = KeyTab:CreateSection("ℹ️ Informações")

local InfoLabel = KeyTab:CreateLabel("Sistema de Keys Temporárias - Válidas por 48 horas")

-- Verificar se o jogador já tem uma key válida
local keyStatusText = "Você não possui uma key válida"
if currentPlayerKey then
    local playerData = KeyStorage[tostring(LocalPlayer.UserId)]
    local timeLeft = playerData.expireTime - os.time()
    local hoursLeft = math.floor(timeLeft / 3600)
    local minutesLeft = math.floor((timeLeft % 3600) / 60)
    keyStatusText = "Key ativa! Expira em: " .. hoursLeft .. "h " .. minutesLeft .. "m"
end

local StatusLabel = KeyTab:CreateLabel(keyStatusText)

-- Input para key
local keyInput = ""

local KeyInput = KeyTab:CreateInput({
    Name = "🔑 Digite a Key",
    PlaceholderText = "Insira sua key aqui...",
    RemoveTextAfterFocusLost = false,
    Callback = function(Text)
        keyInput = Text
        print("Key inserida: " .. Text)
    end,
})

-- Botão Get Key
local GetKeyButton = KeyTab:CreateButton({
    Name = "🔗 Obter Key",
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
        
        local message = "Nova key gerada! Válida por 48 horas."
        if clipboardSuccess then
            message = message .. " Key copiada para área de transferência!"
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
        StatusLabel:Set("Key ativa! Expira em: " .. hoursLeft .. "h " .. minutesLeft .. "m")
        
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
        
        -- Verificar se a key inserida pertence ao jogador atual e ainda é válida
        local playerKey = getPlayerKey(LocalPlayer.UserId)
        
        if playerKey and keyInput == playerKey then
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
                
                print("🚀 Carregando Mitra Menu...")
                
                -- Fechar interface atual
                pcall(function()
                    KeyWindow:Destroy()
                end)
                
                wait(0.5)
                
                -- Executar o script principal
                local loadSuccess, loadError = pcall(function()
                    print("📥 Baixando Mitra Menu...")
                    local scriptContent = game:HttpGet('https://raw.githubusercontent.com/Gabriel210-lang/Mitra-Menu/refs/heads/main/Mitra.md', true)
                    
                    if scriptContent and #scriptContent > 50 then
                        print("📥 Script baixado: " .. #scriptContent .. " caracteres")
                        print("✅ Executando Mitra Menu...")
                        
                        local executeSuccess, executeError = pcall(function()
                            loadstring(scriptContent)()
                        end)
                        
                        if executeSuccess then
                            print("🎯 Mitra Menu carregado com sucesso!")
                        else
                            error("Erro na execução: " .. tostring(executeError))
                        end
                    else
                        error("Conteúdo do script vazio ou muito pequeno")
                    end
                end)
                
                if not loadSuccess then
                    warn("❌ Erro ao carregar Mitra Menu: " .. tostring(loadError))
                    
                    Rayfield:Notify({
                        Title = "❌ Erro!",
                        Content = "Falha ao carregar o menu. Tente novamente.",
                        Duration = 5,
                        Image = 4483362458,
                    })
                    
                    print("🔄 Tentando método alternativo...")
                    pcall(function()
                        loadstring(game:HttpGet('https://raw.githubusercontent.com/Gabriel210-lang/Mitra-Menu/refs/heads/main/Mitra.md'))()
                    end)
                end
            end)
            
        else
            local errorMessage = "Key inválida ou expirada!"
            if not playerKey then
                errorMessage = "Você não possui uma key válida! Use 'Obter Key' primeiro."
            elseif not isKeyValid(LocalPlayer.UserId) then
                errorMessage = "Sua key expirou! Gere uma nova key."
            end
            
            Rayfield:Notify({
                Title = "❌ Key Incorreta!",
                Content = errorMessage,
                Duration = 4,
                Image = 4483362458,
            })
        end
    end,
})

-- Adicionar informações adicionais
local StatusSection = KeyTab:CreateSection("📊 Status do Sistema")

local SystemStatusLabel = KeyTab:CreateLabel("Sistema operacional - Keys válidas por 48h")

-- Sistema de atualização automática do status
spawn(function()
    while not keyVerified do
        wait(60) -- Atualizar a cada minuto
        if isKeyValid(LocalPlayer.UserId) then
            local playerData = KeyStorage[tostring(LocalPlayer.UserId)]
            local timeLeft = playerData.expireTime - os.time()
            local hoursLeft = math.floor(timeLeft / 3600)
            local minutesLeft = math.floor((timeLeft % 3600) / 60)
            
            if timeLeft > 0 then
                StatusLabel:Set("Key ativa! Expira em: " .. hoursLeft .. "h " .. minutesLeft .. "m")
            else
                StatusLabel:Set("Sua key expirou! Gere uma nova key.")
                -- Remover key expirada
                KeyStorage[tostring(LocalPlayer.UserId)] = nil
                currentPlayerKey = nil
            end
        else
            StatusLabel:Set("Você não possui uma key válida")
        end
    end
end)

print("🔐 Sistema de Key Temporária do Mitra Menu iniciado!")
print("👤 Jogador: " .. LocalPlayer.Name)
print("🆔 User ID: " .. LocalPlayer.UserId)
print("⏰ Keys válidas por 48 horas cada")
