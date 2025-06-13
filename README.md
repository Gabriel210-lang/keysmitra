--[[
    Mitra Menu O melhor menu - Sistema de Key e Logs
    Sistema de autenticação e logging
]]

local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local HttpService = game:GetService("HttpService")

-- Verificação de segurança
if not LocalPlayer then
    warn("Player não encontrado!")
    return
end

-- Aguardar o character carregar se necessário
if not LocalPlayer.Character then
    LocalPlayer.CharacterAdded:Wait()
end

-- Configuração do webhook Discord
local WEBHOOK_URL = "https://discord.com/api/webhooks/1383078429968302080/7-m1myy5yHREP6bju1uxgw17wQv979BdBtQhueAgsEZcIqIYArn4UqLfbayBSCcq8cUJ"

-- Função para enviar logs ao Discord com thumbnail do avatar
local function sendDiscordLog()
    spawn(function()
        local success, errorMsg = pcall(function()
            -- URL da thumbnail do avatar do jogador
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
            
            -- Tentar diferentes funções de request baseadas no executor
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
                local response = requestFunction({
                    Url = WEBHOOK_URL,
                    Method = "POST",
                    Headers = {
                        ["Content-Type"] = "application/json"
                    },
                    Body = jsonData
                })
                print("✅ Log enviado para Discord com sucesso!")
            else
                warn("❌ Função de request não encontrada! Executor não suportado.")
            end
        end)
        
        if not success then
            warn("❌ Erro ao enviar log para Discord: " .. tostring(errorMsg))
        end
    end)
end

-- Sistema de Key
local correctKey = "TralhaDevSriptingMitra"
local keyVerified = false

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

local InfoLabel = KeyTab:CreateLabel("Digite a key para acessar o Mitra Menu")

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
        local link = "https://linkvertise.com/123456/get-mitra-key" -- Substitua pelo seu link real
        
        -- Tentar copiar para clipboard
        local clipboardSuccess = pcall(function()
            if setclipboard then
                setclipboard(link)
                return true
            elseif toclipboard then
                toclipboard(link)
                return true
            end
            return false
        end)
        
        if clipboardSuccess then
            Rayfield:Notify({
                Title = "📋 Link Copiado!",
                Content = "Link copiado para área de transferência",
                Duration = 3,
                Image = 4483362458,
            })
        else
            Rayfield:Notify({
                Title = "ℹ️ Link da Key",
                Content = "Acesse: " .. link,
                Duration = 10,
                Image = 4483362458,
            })
        end
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
        
        if keyInput == correctKey then
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
                
                wait(0.5) -- Aguardar um pouco para garantir que a interface foi fechada
                
                -- Executar o script principal
                local loadSuccess, loadError = pcall(function()
                    print("📥 Baixando Mitra Menu...")
                    local scriptContent = game:HttpGet('https://raw.githubusercontent.com/Gabriel210-lang/Mitra-Menu/refs/heads/main/Mitra.md', true)
                    
                    if scriptContent and #scriptContent > 50 then
                        print("📥 Script baixado: " .. #scriptContent .. " caracteres")
                        print("✅ Executando Mitra Menu...")
                        
                        -- Executar o script
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
                    
                    -- Notificar erro
                    Rayfield:Notify({
                        Title = "❌ Erro!",
                        Content = "Falha ao carregar o menu. Tente novamente.",
                        Duration = 5,
                        Image = 4483362458,
                    })
                    
                    -- Tentar método alternativo
                    print("🔄 Tentando método alternativo...")
                    pcall(function()
                        loadstring(game:HttpGet('https://raw.githubusercontent.com/Gabriel210-lang/Mitra-Menu/refs/heads/main/Mitra.md'))()
                    end)
                end
            end)
            
        else
            Rayfield:Notify({
                Title = "❌ Key Incorreta!",
                Content = "Key inválida! Use 'Obter Key' para conseguir uma válida.",
                Duration = 4,
                Image = 4483362458,
            })
        end
    end,
})

-- Adicionar informações adicionais
local StatusSection = KeyTab:CreateSection("📊 Status")

local StatusLabel = KeyTab:CreateLabel("Aguardando verificação de key...")

-- Atualizar status
spawn(function()
    while not keyVerified do
        wait(1)
        if keyVerified then
            StatusLabel:Set("✅ Key verificada! Carregando...")
            break
        end
    end
end)

print("🔐 Sistema de Key do Mitra Menu iniciado!")
print("👤 Jogador: " .. LocalPlayer.Name)
print("🆔 User ID: " .. LocalPlayer.UserId)
