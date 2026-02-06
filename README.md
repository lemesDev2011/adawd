-- LocalScript (colocar em StarterPlayerScripts)
-- Ao pressionar F: toggle de um Highlight vermelho (ESP) em todos os jogadores (visível apenas localmente)

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")

local HIGHLIGHT_NAME = "LocalESPRed" -- nome do Highlight para facilitar remoção
local localPlayer = Players.LocalPlayer

local enabled = false
local charConns = {}        -- [player] = CharacterAdded connection
local playerAddedConn      -- conexão para Players.PlayerAdded
local playerRemovingConn   -- conexão para Players.PlayerRemoving

-- Cria e aplica Highlight no personagem (se ainda não existir)
local function applyHighlightToCharacter(character)
	if not character or not character:IsA("Model") then return end
	if character:FindFirstChild(HIGHLIGHT_NAME) then return end

	local highlight = Instance.new("Highlight")
	highlight.Name = HIGHLIGHT_NAME
	highlight.Adornee = character
	highlight.FillColor = Color3.fromRGB(255, 0, 0)
	highlight.OutlineColor = Color3.fromRGB(255, 0, 0)
	highlight.FillTransparency = 0.25
	highlight.OutlineTransparency = 0
	highlight.Parent = character
end

-- Remove Highlight do personagem (se existir)
local function removeHighlightFromCharacter(character)
	if not character or not character:IsA("Model") then return end
	local h = character:FindFirstChild(HIGHLIGHT_NAME)
	if h and h:IsA("Highlight") then
		h:Destroy()
	end
end

-- Aplica highlight em todos os personagens atuais
local function applyHighlightsToAll()
	for _, plr in ipairs(Players:GetPlayers()) do
		local char = plr.Character
		if char then
			applyHighlightToCharacter(char)
		end
	end
end

-- Remove highlight de todos os personagens
local function removeHighlightsFromAll()
	for _, plr in ipairs(Players:GetPlayers()) do
		local char = plr.Character
		if char then
			removeHighlightFromCharacter(char)
		end
	end
end

-- Conecta CharacterAdded de um jogador (para reaplicar no respawn)
local function connectCharacterAddedForPlayer(plr)
	if not plr then return end
	-- desconecta conexão anterior se houver
	if charConns[plr] then
		pcall(function() charConns[plr]:Disconnect() end)
	end
	charConns[plr] = plr.CharacterAdded:Connect(function(character)
		-- pequeno delay opcional para garantir que partes existam
		wait(0.05)
		if enabled then
			applyHighlightToCharacter(character)
		end
	end)
end

-- Conecta listeners para todos os jogadores (usado quando ativar)
local function connectAllPlayerListeners()
	-- limpa conexões antigas só por segurança
	for plr, conn in pairs(charConns) do
		if conn then pcall(function() conn:Disconnect() end) end
	end
	table.clear(charConns)

	for _, plr in ipairs(Players:GetPlayers()) do
		-- aplica agora se já tiver personagem
		if plr.Character then
			applyHighlightToCharacter(plr.Character)
		end
		connectCharacterAddedForPlayer(plr)
	end

	-- novo jogador entrou
	playerAddedConn = Players.PlayerAdded:Connect(function(newPlr)
		if enabled and newPlr.Character then
			applyHighlightToCharacter(newPlr.Character)
		end
		connectCharacterAddedForPlayer(newPlr)
	end)

	-- jogador saiu -> desconecta sua conexão e garante remoção
	playerRemovingConn = Players.PlayerRemoving:Connect(function(leftPlr)
		if charConns[leftPlr] then
			pcall(function() charConns[leftPlr]:Disconnect() end)
			charConns[leftPlr] = nil
		end
		-- highlight será destruído quando Character for removido; só por segurança:
		if leftPlr.Character then
			removeHighlightFromCharacter(leftPlr.Character)
		end
	end)
end

-- Desconecta todos os listeners que criamos
local function disconnectAllPlayerListeners()
	for plr, conn in pairs(charConns) do
		if conn then pcall(function() conn:Disconnect() end) end
	end
	table.clear(charConns)

	if playerAddedConn then
		pcall(function() playerAddedConn:Disconnect() end)
		playerAddedConn = nil
	end
	if playerRemovingConn then
		pcall(function() playerRemovingConn:Disconnect() end)
		playerRemovingConn = nil
	end
end

-- Alterna o ESP
local function toggleESP()
	enabled = not enabled
	if enabled then
		applyHighlightsToAll()
		connectAllPlayerListeners()
	else
		disconnectAllPlayerListeners()
		removeHighlightsFromAll()
	end
end

-- Ouve tecla F (ignora se o jogo processou o input ou se o jogador está digitando)
UserInputService.InputBegan:Connect(function(input, gameProcessed)
	if gameProcessed then return end
	if UserInputService:GetFocusedTextBox() then return end
	if input.UserInputType == Enum.UserInputType.Keyboard and input.KeyCode == Enum.KeyCode.F then
		toggleESP()
	end
end)

-- Limpeza ao fechar (opcional)
local function onClose()
	disconnectAllPlayerListeners()
	removeHighlightsFromAll()
end
game:BindToClose(onClose)
