--[[
    ═══════════════════════════════════════════════════════════════════════
    MENU MODERNO + SISTEMA DE LOCK-ON — Roblox Studio (Luau)
    ═══════════════════════════════════════════════════════════════════════
    LocalScript → StarterPlayer > StarterPlayerScripts
    Tecla F2 = abrir/fechar menu · Tecla Q = travar/soltar alvo
    Botão flutuante ◎ sempre visível no canto inferior direito
    ═══════════════════════════════════════════════════════════════════════
]]

-- ═══════════════════════════════════════════════════════════════════════
-- CONFIGURAÇÃO (edite aqui)
-- ═══════════════════════════════════════════════════════════════════════
local CONFIG = {
    CreatorName = "SeuNome",
    Version     = "1.1.0",
    Credits     = "Desenvolvido com Luau · Roblox Studio",
    Description = "Menu moderno com sistema de lock-on integrado. Personalize cores, veja dados do jogador e configure o lock-on em tempo real.",
    ToggleKey   = Enum.KeyCode.F2,
}

-- ═══════════════════════════════════════════════════════════════════════
-- SERVIÇOS
-- ═══════════════════════════════════════════════════════════════════════
local Players            = game:GetService("Players")
local RunService         = game:GetService("RunService")
local TweenService       = game:GetService("TweenService")
local UserInputService   = game:GetService("UserInputService")
local MarketplaceService = game:GetService("MarketplaceService")
local Workspace          = game:GetService("Workspace")

local LocalPlayer = Players.LocalPlayer
local PlayerGui   = LocalPlayer:WaitForChild("PlayerGui")

-- ═══════════════════════════════════════════════════════════════════════
-- TEMA
-- ═══════════════════════════════════════════════════════════════════════
local Theme = {
    Primary    = Color3.fromRGB(45, 125, 210),
    Background = Color3.fromRGB(28, 28, 34),
    Surface    = Color3.fromRGB(38, 38, 46),
    Button     = Color3.fromRGB(52, 52, 62),
    Border     = Color3.fromRGB(72, 72, 84),
    Text       = Color3.fromRGB(240, 240, 245),
    TextDim    = Color3.fromRGB(158, 158, 170),
}

local Settings = { AnimationsEnabled = true, ShadowEnabled = true }

-- ═══════════════════════════════════════════════════════════════════════
-- SISTEMA DE TEMA
-- ═══════════════════════════════════════════════════════════════════════
local themeBindings, themeCallbacks = {}, {}

local function bindTheme(instance, property, themeKey)
    themeBindings[instance] = themeBindings[instance] or {}
    themeBindings[instance][property] = themeKey
    instance[property] = Theme[themeKey]
end

local function onThemeChanged(fn) table.insert(themeCallbacks, fn) end

local function applyTheme()
    for inst, props in pairs(themeBindings) do
        if inst and inst.Parent then
            for prop, key in pairs(props) do
                pcall(function() inst[prop] = Theme[key] end)
            end
        else
            themeBindings[inst] = nil
        end
    end
    for _, fn in ipairs(themeCallbacks) do pcall(fn) end
end

-- ═══════════════════════════════════════════════════════════════════════
-- HELPERS
-- ═══════════════════════════════════════════════════════════════════════
local function new(className, props, children)
    local inst = Instance.new(className)
    local parent
    if props then
        for k, v in pairs(props) do
            if k == "Parent" then parent = v else inst[k] = v end
        end
    end
    if children then for _, c in ipairs(children) do c.Parent = inst end end
    if parent then inst.Parent = parent end
    return inst
end

local function tween(instance, props, duration, style, dir)
    if not Settings.AnimationsEnabled then duration = 0 end
    local info = TweenInfo.new(duration or 0.2, style or Enum.EasingStyle.Quad, dir or Enum.EasingDirection.Out)
    local t = TweenService:Create(instance, info, props)
    t:Play()
    return t
end

local function lighten(color, amount)
    return Color3.new(
        math.clamp(color.R + amount, 0, 1),
        math.clamp(color.G + amount, 0, 1),
        math.clamp(color.B + amount, 0, 1)
    )
end

local function formatDuration(seconds)
    seconds = math.max(0, math.floor(seconds))
    local h = math.floor(seconds / 3600)
    local m = math.floor((seconds % 3600) / 60)
    local s = seconds % 60
    return string.format("%02d:%02d:%02d", h, m, s)
end

local sessionStart = os.clock()
local entryTime    = DateTime.now()

-- ═══════════════════════════════════════════════════════════════════════
-- SCREENGUI
-- ═══════════════════════════════════════════════════════════════════════
local screenGui = new("ScreenGui", {
    Name = "ModernMenuGui",
    ResetOnSpawn = false,
    IgnoreGuiInset = true,
    ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
    Parent = PlayerGui,
})

local function getInitialSize()
    local vp = Workspace.CurrentCamera and Workspace.CurrentCamera.ViewportSize or Vector2.new(1280, 720)
    return UDim2.fromOffset(math.min(780, vp.X - 40), math.min(520, vp.Y - 40))
end

local DEFAULT_SIZE  = getInitialSize()
local TITLE_HEIGHT  = 44
local SIDEBAR_WIDTH = 180

local windowRoot = new("Frame", {
    Name = "WindowRoot",
    Size = DEFAULT_SIZE,
    Position = UDim2.new(0.5, 0, 0.5, 0),
    AnchorPoint = Vector2.new(0.5, 0.5),
    BackgroundTransparency = 1,
    Parent = screenGui,
})

local shadow = new("Frame", {
    Size = UDim2.fromScale(1, 1),
    Position = UDim2.fromOffset(0, 8),
    BackgroundColor3 = Color3.new(0, 0, 0),
    BackgroundTransparency = 0.55,
    BorderSizePixel = 0,
    ZIndex = 1,
    Parent = windowRoot,
})
new("UICorner", { CornerRadius = UDim.new(0, 12), Parent = shadow })

local mainFrame = new("Frame", {
    Size = UDim2.fromScale(1, 1),
    BackgroundColor3 = Theme.Background,
    BorderSizePixel = 0,
    ClipsDescendants = true,
    ZIndex = 2,
    Parent = windowRoot,
})
bindTheme(mainFrame, "BackgroundColor3", "Background")
new("UICorner", { CornerRadius = UDim.new(0, 12), Parent = mainFrame })
local mainStroke = new("UIStroke", { Color = Theme.Border, Thickness = 1, Parent = mainFrame })
bindTheme(mainStroke, "Color", "Border")

-- ═══════════════════════════════════════════════════════════════════════
-- BARRA DE TÍTULO
-- ═══════════════════════════════════════════════════════════════════════
local titleBar = new("Frame", {
    Size = UDim2.new(1, 0, 0, TITLE_HEIGHT),
    BackgroundColor3 = Theme.Primary,
    BorderSizePixel = 0,
    Parent = mainFrame,
})
bindTheme(titleBar, "BackgroundColor3", "Primary")

local dragHandle = new("Frame", {
    Size = UDim2.new(1, -140, 1, 0),
    BackgroundTransparency = 1,
    Parent = titleBar,
})
new("TextLabel", {
    Size = UDim2.new(1, -20, 1, 0),
    Position = UDim2.fromOffset(16, 0),
    BackgroundTransparency = 1,
    Text = "Menu Principal",
    TextSize = 16,
    TextXAlignment = Enum.TextXAlignment.Left,
    Font = Enum.Font.GothamBold,
    TextColor3 = Theme.Text,
    Parent = dragHandle,
})

local controlsBar = new("Frame", {
    Size = UDim2.fromOffset(124, 32),
    Position = UDim2.new(1, -10, 0.5, 0),
    AnchorPoint = Vector2.new(1, 0.5),
    BackgroundTransparency = 1,
    Parent = titleBar,
})
new("UIListLayout", {
    FillDirection = Enum.FillDirection.Horizontal,
    HorizontalAlignment = Enum.HorizontalAlignment.Right,
    VerticalAlignment = Enum.VerticalAlignment.Center,
    Padding = UDim.new(0, 4),
    SortOrder = Enum.SortOrder.LayoutOrder,
    Parent = controlsBar,
})

local minimizeWindow, toggleMaximize, closeMenu, openMenu

local function createWindowButton(text, order, hoverColor, callback)
    local btn = new("TextButton", {
        Size = UDim2.fromOffset(32, 32),
        BackgroundColor3 = Theme.Primary,
        BackgroundTransparency = 0.35,
        Text = text,
        TextSize = 18,
        Font = Enum.Font.GothamBold,
        TextColor3 = Theme.Text,
        AutoButtonColor = false,
        LayoutOrder = order,
        Parent = controlsBar,
    })
    new("UICorner", { CornerRadius = UDim.new(0, 8), Parent = btn })
    btn.MouseEnter:Connect(function()
        tween(btn, { BackgroundTransparency = 0.05, BackgroundColor3 = hoverColor or lighten(Theme.Primary, 0.15) }, 0.12)
    end)
    btn.MouseLeave:Connect(function()
        tween(btn, { BackgroundTransparency = 0.35, BackgroundColor3 = Theme.Primary }, 0.12)
    end)
    btn.Activated:Connect(callback)
    onThemeChanged(function() btn.BackgroundColor3 = Theme.Primary end)
    return btn
end

createWindowButton("−", 1, nil, function() minimizeWindow() end)
createWindowButton("□", 2, nil, function() toggleMaximize() end)
createWindowButton("×", 3, Color3.fromRGB(220, 60, 60), function() closeMenu() end)

-- ═══════════════════════════════════════════════════════════════════════
-- DRAG (arrastar janela)
-- ═══════════════════════════════════════════════════════════════════════
local windowState = "normal"
local restoreSize = DEFAULT_SIZE
local restorePos  = UDim2.new(0.5, 0, 0.5, 0)
local dragging, dragStartPos, dragStartInput = false, nil, nil

dragHandle.InputBegan:Connect(function(input)
    if windowState == "maximized" then return end
    if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then
        local ap, as = windowRoot.AbsolutePosition, windowRoot.AbsoluteSize
        local cx, cy = ap.X + as.X / 2, ap.Y + as.Y / 2
        windowRoot.Position = UDim2.fromOffset(cx, cy)
        dragStartPos   = Vector2.new(cx, cy)
        dragStartInput = input.Position
        dragging = true
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if not dragging then return end
    if input.UserInputType == Enum.UserInputType.MouseMovement
        or input.UserInputType == Enum.UserInputType.Touch then
        local d = input.Position - dragStartInput
        windowRoot.Position = UDim2.fromOffset(dragStartPos.X + d.X, dragStartPos.Y + d.Y)
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then
        dragging = false
    end
end)

-- ═══════════════════════════════════════════════════════════════════════
-- CONTEÚDO (SIDEBAR + PÁGINAS)
-- ═══════════════════════════════════════════════════════════════════════
local contentRoot = new("Frame", {
    Size = UDim2.new(1, 0, 1, -TITLE_HEIGHT),
    Position = UDim2.new(0, 0, 0, TITLE_HEIGHT),
    BackgroundTransparency = 1,
    Parent = mainFrame,
})

local sidebar = new("Frame", {
    Size = UDim2.new(0, SIDEBAR_WIDTH, 1, 0),
    BackgroundColor3 = Theme.Surface,
    BorderSizePixel = 0,
    Parent = contentRoot,
})
bindTheme(sidebar, "BackgroundColor3", "Surface")
new("UIPadding", {
    PaddingLeft = UDim.new(0, 8), PaddingRight = UDim.new(0, 8),
    PaddingTop  = UDim.new(0, 8), PaddingBottom = UDim.new(0, 8),
    Parent = sidebar,
})
new("UIListLayout", {
    Padding = UDim.new(0, 4),
    SortOrder = Enum.SortOrder.LayoutOrder,
    Parent = sidebar,
})

local pagesArea = new("Frame", {
    Size = UDim2.new(1, -SIDEBAR_WIDTH, 1, 0),
    Position = UDim2.new(0, SIDEBAR_WIDTH, 0, 0),
    BackgroundTransparency = 1,
    Parent = contentRoot,
})

local pages, navEntries = {}, {}
local activePageId = nil

local function createPage(id, title, order)
    local page = new("ScrollingFrame", {
        Name = "Page_" .. id,
        Size = UDim2.fromScale(1, 1),
        BackgroundTransparency = 1,
        BorderSizePixel = 0,
        ScrollBarThickness = 6,
        ScrollBarImageColor3 = Theme.Border,
        CanvasSize = UDim2.new(0, 0, 0, 0),
        AutomaticCanvasSize = Enum.AutomaticSize.Y,
        Visible = false,
        LayoutOrder = order,
        Parent = pagesArea,
    })
    bindTheme(page, "ScrollBarImageColor3", "Border")
    new("UIPadding", {
        PaddingLeft = UDim.new(0, 16), PaddingRight = UDim.new(0, 16),
        PaddingTop  = UDim.new(0, 16), PaddingBottom = UDim.new(0, 16),
        Parent = page,
    })
    new("UIListLayout", {
        Padding = UDim.new(0, 10),
        SortOrder = Enum.SortOrder.LayoutOrder,
        Parent = page,
    })
    local lbl = new("TextLabel", {
        Size = UDim2.new(1, 0, 0, 32),
        BackgroundTransparency = 1,
        Text = title, TextSize = 22,
        Font = Enum.Font.GothamBold,
        TextColor3 = Theme.Text,
        TextXAlignment = Enum.TextXAlignment.Left,
        LayoutOrder = 0, Parent = page,
    })
    bindTheme(lbl, "TextColor3", "Text")
    pages[id] = page
    return page
end

local function createNavButton(id, icon, title, order)
    local btn = new("TextButton", {
        Size = UDim2.new(1, 0, 0, 36),
        BackgroundColor3 = Theme.Button,
        BorderSizePixel = 0, Text = "",
        AutoButtonColor = false,
        LayoutOrder = order,
        Parent = sidebar,
    })
    new("UICorner", { CornerRadius = UDim.new(0, 8), Parent = btn })
    new("TextLabel", {
        Size = UDim2.fromOffset(28, 28),
        Position = UDim2.new(0, 6, 0.5, 0),
        AnchorPoint = Vector2.new(0, 0.5),
        BackgroundTransparency = 1,
        Text = icon, TextSize = 16,
        Font = Enum.Font.GothamBold,
        TextColor3 = Theme.Text,
        Parent = btn,
    })
    new("TextLabel", {
        Size = UDim2.new(1, -46, 1, 0),
        Position = UDim2.new(0, 40, 0, 0),
        BackgroundTransparency = 1,
        Text = title, TextSize = 13,
        Font = Enum.Font.GothamMedium,
        TextColor3 = Theme.Text,
        TextXAlignment = Enum.TextXAlignment.Left,
        Parent = btn,
    })
    table.insert(navEntries, { id = id, btn = btn })
    return btn
end

local function refreshNavColors()
    for _, e in ipairs(navEntries) do
        e.btn.BackgroundColor3 = (e.id == activePageId) and Theme.Primary or Theme.Button
    end
end
onThemeChanged(refreshNavColors)

local function showPage(id)
    activePageId = id
    for pid, p in pairs(pages) do p.Visible = (pid == id) end
    refreshNavColors()
    local page = pages[id]
    if page then
        if Settings.AnimationsEnabled then
            page.GroupTransparency = 1
            TweenService:Create(page, TweenInfo.new(0.2), { GroupTransparency = 0 }):Play()
        else
            page.GroupTransparency = 0
        end
    end
end

local function addNav(id, icon, title, order)
    local btn = createNavButton(id, icon, title, order)
    btn.Activated:Connect(function() showPage(id) end)
    btn.MouseEnter:Connect(function()
        if activePageId ~= id then
            tween(btn, { BackgroundColor3 = lighten(Theme.Button, 0.08) }, 0.1)
        end
    end)
    btn.MouseLeave:Connect(function()
        if activePageId ~= id then
            tween(btn, { BackgroundColor3 = Theme.Button }, 0.1)
        end
    end)
end

-- ═══════════════════════════════════════════════════════════════════════
-- COMPONENTES REUTILIZÁVEIS
-- ═══════════════════════════════════════════════════════════════════════
local function createAutoSection(parent, order)
    local s = new("Frame", {
        Size = UDim2.new(1, 0, 0, 0),
        AutomaticSize = Enum.AutomaticSize.Y,
        BackgroundColor3 = Theme.Surface,
        BorderSizePixel = 0, LayoutOrder = order,
        Parent = parent,
    })
    bindTheme(s, "BackgroundColor3", "Surface")
    new("UICorner", { CornerRadius = UDim.new(0, 10), Parent = s })
    new("UIPadding", {
        PaddingLeft = UDim.new(0, 14), PaddingRight = UDim.new(0, 14),
        PaddingTop  = UDim.new(0, 12), PaddingBottom = UDim.new(0, 12),
        Parent = s,
    })
    new("UIListLayout", {
        Padding = UDim.new(0, 6),
        SortOrder = Enum.SortOrder.LayoutOrder,
        Parent = s,
    })
    return s
end

local function createSectionHeader(parent, text, order)
    local l = new("TextLabel", {
        Size = UDim2.new(1, 0, 0, 24),
        BackgroundTransparency = 1,
        Text = text, TextSize = 15,
        Font = Enum.Font.GothamBold,
        TextColor3 = Theme.Text,
        TextXAlignment = Enum.TextXAlignment.Left,
        LayoutOrder = order, Parent = parent,
    })
    bindTheme(l, "TextColor3", "Text")
    return l
end

local function createInfoRow(parent, label, order)
    local row = new("Frame", {
        Size = UDim2.new(1, 0, 0, 34),
        BackgroundColor3 = Theme.Button,
        BorderSizePixel = 0, LayoutOrder = order,
        Parent = parent,
    })
    bindTheme(row, "BackgroundColor3", "Button")
    new("UICorner", { CornerRadius = UDim.new(0, 6), Parent = row })
    local lbl = new("TextLabel", {
        Size = UDim2.new(0.45, 0, 1, 0),
        Position = UDim2.fromOffset(12, 0),
        BackgroundTransparency = 1,
        Text = label, TextSize = 13,
        Font = Enum.Font.GothamMedium,
        TextColor3 = Theme.TextDim,
        TextXAlignment = Enum.TextXAlignment.Left,
        Parent = row,
    })
    bindTheme(lbl, "TextColor3", "TextDim")
    local val = new("TextLabel", {
        Size = UDim2.new(0.55, -24, 1, 0),
        Position = UDim2.new(0.45, 0, 0, 0),
        BackgroundTransparency = 1,
        Text = "—", TextSize = 13,
        Font = Enum.Font.Gotham,
        TextColor3 = Theme.Text,
        TextXAlignment = Enum.TextXAlignment.Right,
        TextTruncate = Enum.TextTruncate.AtEnd,
        Parent = row,
    })
    bindTheme(val, "TextColor3", "Text")
    return val
end

local function createSlider(parent, label, order, callback)
    local value = 0
    local container = new("Frame", {
        Size = UDim2.new(1, 0, 0, 46),
        BackgroundTransparency = 1,
        LayoutOrder = order, Parent = parent,
    })
    local textLabel = new("TextLabel", {
        Size = UDim2.new(1, 0, 0, 16),
        BackgroundTransparency = 1,
        Text = label .. ": 0", TextSize = 12,
        Font = Enum.Font.GothamMedium,
        TextColor3 = Theme.Text,
        TextXAlignment = Enum.TextXAlignment.Left,
        Parent = container,
    })
    bindTheme(textLabel, "TextColor3", "Text")

    local track = new("Frame", {
        Size = UDim2.new(1, -16, 0, 10),
        Position = UDim2.new(0, 8, 0, 26),
        BackgroundColor3 = Theme.Border,
        BorderSizePixel = 0, Parent = container,
    })
    bindTheme(track, "BackgroundColor3", "Border")
    new("UICorner", { CornerRadius = UDim.new(1, 0), Parent = track })

    local fill = new("Frame", {
        Size = UDim2.new(0, 0, 1, 0),
        BackgroundColor3 = Theme.Primary,
        BorderSizePixel = 0, Parent = track,
    })
    bindTheme(fill, "BackgroundColor3", "Primary")
    new("UICorner", { CornerRadius = UDim.new(1, 0), Parent = fill })

    local knob = new("Frame", {
        Size = UDim2.fromOffset(16, 16),
        AnchorPoint = Vector2.new(0.5, 0.5),
        Position = UDim2.new(0, 0, 0.5, 0),
        BackgroundColor3 = Color3.new(1, 1, 1),
        BorderSizePixel = 0, ZIndex = 3, Parent = track,
    })
    new("UICorner", { CornerRadius = UDim.new(1, 0), Parent = knob })

    local overlay = new("Frame", {
        Size = UDim2.new(1, 16, 0, 24),
        Position = UDim2.new(0, -8, 0.5, 0),
        AnchorPoint = Vector2.new(0, 0.5),
        BackgroundTransparency = 1, ZIndex = 5, Parent = track,
    })

    local drag = false
    local function visual(v)
        local rel = math.clamp(v / 255, 0, 1)
        fill.Size = UDim2.new(rel, 0, 1, 0)
        knob.Position = UDim2.new(rel, 0, 0.5, 0)
        textLabel.Text = label .. ": " .. math.floor(v + 0.5)
    end
    local function set(v, fire)
        value = math.clamp(v, 0, 255)
        visual(value)
        if fire then callback(value) end
    end
    local function fromX(x)
        local rel = math.clamp((x - track.AbsolutePosition.X) / math.max(track.AbsoluteSize.X, 1), 0, 1)
        set(rel * 255, true)
    end

    overlay.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
            or input.UserInputType == Enum.UserInputType.Touch then
            drag = true
            fromX(input.Position.X)
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if drag and (input.UserInputType == Enum.UserInputType.MouseMovement
            or input.UserInputType == Enum.UserInputType.Touch) then
            fromX(input.Position.X)
        end
    end)
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
            or input.UserInputType == Enum.UserInputType.Touch then drag = false end
    end)

    visual(0)
    return { set = set, getValue = function() return math.floor(value + 0.5) end }
end

local function createToggle(parent, label, initial, order, callback)
    local row = new("Frame", {
        Size = UDim2.new(1, 0, 0, 40),
        BackgroundColor3 = Theme.Button,
        BorderSizePixel = 0, LayoutOrder = order,
        Parent = parent,
    })
    bindTheme(row, "BackgroundColor3", "Button")
    new("UICorner", { CornerRadius = UDim.new(0, 8), Parent = row })

    local lbl = new("TextLabel", {
        Size = UDim2.new(1, -80, 1, 0),
        Position = UDim2.fromOffset(12, 0),
        BackgroundTransparency = 1,
        Text = label, TextSize = 13,
        Font = Enum.Font.GothamMedium,
        TextColor3 = Theme.Text,
        TextXAlignment = Enum.TextXAlignment.Left,
        Parent = row,
    })
    bindTheme(lbl, "TextColor3", "Text")

    local track = new("Frame", {
        Size = UDim2.fromOffset(44, 24),
        Position = UDim2.new(1, -56, 0.5, 0),
        AnchorPoint = Vector2.new(0, 0.5),
        BackgroundColor3 = initial and Theme.Primary or Theme.Border,
        BorderSizePixel = 0, Parent = row,
    })
    new("UICorner", { CornerRadius = UDim.new(1, 0), Parent = track })

    local knob = new("Frame", {
        Size = UDim2.fromOffset(18, 18),
        Position = initial and UDim2.new(1, -21, 0.5, 0) or UDim2.new(0, 3, 0.5, 0),
        AnchorPoint = Vector2.new(0, 0.5),
        BackgroundColor3 = Color3.new(1, 1, 1),
        BorderSizePixel = 0, Parent = track,
    })
    new("UICorner", { CornerRadius = UDim.new(1, 0), Parent = knob })

    local state = initial
    local function setState(v)
        state = v
        tween(track, { BackgroundColor3 = state and Theme.Primary or Theme.Border }, 0.15)
        tween(knob, { Position = state and UDim2.new(1, -21, 0.5, 0) or UDim2.new(0, 3, 0.5, 0) }, 0.15)
        callback(state)
    end

    local click = new("TextButton", {
        Size = UDim2.new(1, 0, 1, 0),
        BackgroundTransparency = 1,
        Text = "", AutoButtonColor = false,
        Parent = row,
    })
    click.Activated:Connect(function() setState(not state) end)
    onThemeChanged(function()
        track.BackgroundColor3 = state and Theme.Primary or Theme.Border
    end)
    return { set = setState, get = function() return state end }
end

-- ═══════════════════════════════════════════════════════════════════════
-- CONTROLES DE JANELA
-- ═══════════════════════════════════════════════════════════════════════
local function saveRestore()
    if windowState == "normal" then
        restoreSize = windowRoot.Size
        restorePos  = windowRoot.Position
    end
end

minimizeWindow = function()
    if windowState == "minimized" then
        tween(windowRoot, { Size = restoreSize }, 0.3, Enum.EasingStyle.Quart)
        contentRoot.Visible = true
        shadow.Visible = Settings.ShadowEnabled
        windowState = "normal"
    else
        saveRestore()
        local w = windowRoot.AbsoluteSize.X
        tween(windowRoot, { Size = UDim2.new(0, w, 0, TITLE_HEIGHT) }, 0.3, Enum.EasingStyle.Quart)
        contentRoot.Visible = false
        shadow.Visible = false
        windowState = "minimized"
    end
end

toggleMaximize = function()
    if windowState == "maximized" then
        tween(windowRoot, { Size = restoreSize, Position = restorePos }, 0.3, Enum.EasingStyle.Quart)
        shadow.Visible = Settings.ShadowEnabled
        windowState = "normal"
    else
        saveRestore()
        shadow.Visible = false
        tween(windowRoot, {
            Size = UDim2.new(1, -20, 1, -20),
            Position = UDim2.new(0.5, 0, 0.5, 0),
        }, 0.3, Enum.EasingStyle.Quart)
        windowState = "maximized"
    end
end

local toggleButton = new("TextButton", {
    Size = UDim2.fromOffset(48, 48),
    Position = UDim2.new(0, 16, 0, 16),
    BackgroundColor3 = Theme.Primary,
    Text = "≡", TextSize = 24,
    Font = Enum.Font.GothamBold,
    TextColor3 = Theme.Text,
    Visible = false, AutoButtonColor = false,
    Parent = screenGui,
})
new("UICorner", { CornerRadius = UDim.new(1, 0), Parent = toggleButton })
onThemeChanged(function() toggleButton.BackgroundColor3 = Theme.Primary end)

closeMenu = function()
    if Settings.AnimationsEnabled then
        tween(windowRoot, { GroupTransparency = 1 }, 0.2)
        task.delay(0.22, function()
            windowRoot.Visible = false
            toggleButton.Visible = true
        end)
    else
        windowRoot.Visible = false
        toggleButton.Visible = true
    end
end

openMenu = function()
    toggleButton.Visible = false
    windowRoot.Visible = true
    windowRoot.GroupTransparency = 1
    tween(windowRoot, { GroupTransparency = 0 }, 0.25)
end

toggleButton.Activated:Connect(function()
    if windowRoot.Visible then closeMenu() else openMenu() end
end)

-- ═══════════════════════════════════════════════════════════════════════
-- SISTEMA DE LOCK-ON
-- ═══════════════════════════════════════════════════════════════════════
local LockOn = {
    enabled   = false,
    target    = nil,
    highlight = nil,
    settings  = {
        MaxRange      = 150,
        CameraFollow  = true,
        AutoAcquire   = false,
        ShowHighlight = true,
        Keybind       = Enum.KeyCode.Q,
    },
    scanAccum = 0,
    scanInterval = 0.4,
}

-- Botão flutuante ◎ (fora do menu)
local lockButton = new("TextButton", {
    Name = "LockOnButton",
    Size = UDim2.fromOffset(64, 64),
    AnchorPoint = Vector2.new(1, 1),
    Position = UDim2.new(1, -24, 1, -24),
    BackgroundColor3 = Theme.Primary,
    BackgroundTransparency = 0.15,
    Text = "◎", TextSize = 30,
    Font = Enum.Font.GothamBold,
    TextColor3 = Theme.Text,
    AutoButtonColor = false,
    Parent = screenGui,
})
new("UICorner", { CornerRadius = UDim.new(1, 0), Parent = lockButton })
local lockBtnStroke = new("UIStroke", { Color = Theme.Text, Thickness = 2, Transparency = 0.5, Parent = lockButton })
onThemeChanged(function()
    if not LockOn.enabled then lockButton.BackgroundColor3 = Theme.Primary end
end)

-- Nome do alvo acima do botão
local lockLabel = new("TextLabel", {
    Size = UDim2.new(0, 200, 0, 22),
    Position = UDim2.new(1, -56, 1, -96),
    AnchorPoint = Vector2.new(1, 0),
    BackgroundTransparency = 1,
    Text = "", TextSize = 13,
    Font = Enum.Font.GothamBold,
    TextColor3 = Theme.Text,
    Visible = false,
    Parent = screenGui,
})
new("UIStroke", { Color = Color3.new(0, 0, 0), Thickness = 2, Transparency = 0.3, Parent = lockLabel })

-- Encontra alvos próximos
local function findTargets()
    local list = {}
    local char = LocalPlayer.Character
    if not char then return list end
    local myRoot = char:FindFirstChild("HumanoidRootPart") or char:FindFirstChild("Head")
    if not myRoot then return list end

    local function tryAdd(model, hum, root, name)
        if not hum or not root or hum.Health <= 0 then return end
        local dist = (root.Position - myRoot.Position).Magnitude
        if dist <= LockOn.settings.MaxRange then
            table.insert(list, { part = root, char = model, hum = hum, name = name, dist = dist })
        end
    end

    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and p.Character then
            tryAdd(p.Character,
                p.Character:FindFirstChildOfClass("Humanoid"),
                p.Character:FindFirstChild("HumanoidRootPart"),
                p.DisplayName)
        end
    end

    for _, m in ipairs(Workspace:GetChildren()) do
        if m:IsA("Model") and not Players:GetPlayerFromCharacter(m) then
            local hum = m:FindFirstChildOfClass("Humanoid")
            local root = m:FindFirstChild("HumanoidRootPart")
            if hum and root then tryAdd(m, hum, root, m.Name) end
        end
    end

    table.sort(list, function(a, b) return a.dist < b.dist end)
    return list
end

local function clearHighlight()
    if LockOn.highlight then LockOn.highlight:Destroy(); LockOn.highlight = nil end
end

local function applyHighlight(char)
    clearHighlight()
    if not LockOn.settings.ShowHighlight or not char then return end
    local h = Instance.new("Highlight")
    h.Name = "LockOnHighlight"
    h.FillColor = Color3.fromRGB(255, 60, 60)
    h.FillTransparency = 0.75
    h.OutlineColor = Color3.fromRGB(255, 60, 60)
    h.OutlineTransparency = 0
    h.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    h.Parent = char
    LockOn.highlight = h
end

local function updateLockVisual()
    if LockOn.enabled and LockOn.target then
        lockButton.BackgroundColor3 = Color3.fromRGB(220, 50, 50)
        lockButton.BackgroundTransparency = 0
        lockButton.Text = "✕"
        lockLabel.Text = "🔒 " .. LockOn.target.name
        lockLabel.Visible = true
        lockBtnStroke.Transparency = 0.1
    else
        lockButton.BackgroundColor3 = Theme.Primary
        lockButton.BackgroundTransparency = 0.15
        lockButton.Text = "◎"
        lockLabel.Text = ""
        lockLabel.Visible = false
        lockBtnStroke.Transparency = 0.5
    end
end
onThemeChanged(updateLockVisual)

local function lockTo(target)
    if not target then return end
    LockOn.target  = target
    LockOn.enabled = true
    applyHighlight(target.char)
    updateLockVisual()
end

local function releaseLock()
    LockOn.enabled = false
    LockOn.target  = nil
    clearHighlight()
    updateLockVisual()
end

local function cycleTarget()
    local targets = findTargets()
    if #targets == 0 then
        local t = lockButton.Text
        lockButton.Text = "—"
        task.delay(0.4, function()
            if not LockOn.enabled then lockButton.Text = t end
        end)
        return
    end
    local nextIdx = 1
    if LockOn.target then
        for i, t in ipairs(targets) do
            if t.part == LockOn.target.part then
                nextIdx = (i % #targets) + 1
                break
            end
        end
    end
    lockTo(targets[nextIdx])
end

lockButton.Activated:Connect(function()
    if LockOn.enabled then releaseLock() else cycleTarget() end
end)

-- Input global (F2 + Q)
UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if input.KeyCode == CONFIG.ToggleKey then
        if windowRoot.Visible then closeMenu() else openMenu() end
        return
    end
    if input.KeyCode == LockOn.settings.Keybind then
        if LockOn.enabled then releaseLock() else cycleTarget() end
    end
end)

LocalPlayer.CharacterAdded:Connect(function() releaseLock() end)

-- Loop de scan + auto-acquire
RunService.Heartbeat:Connect(function(dt)
    LockOn.scanAccum = LockOn.scanAccum + dt
    if LockOn.scanAccum < LockOn.scanInterval then return end
    LockOn.scanAccum = 0

    if LockOn.enabled and LockOn.target then
        local t = LockOn.target
        local myChar = LocalPlayer.Character
        local myRoot = myChar and (myChar:FindFirstChild("HumanoidRootPart") or myChar:FindFirstChild("Head"))
        local valid = t.char and t.char.Parent and t.part and t.part.Parent
            and t.hum and t.hum.Parent and t.hum.Health > 0
        if valid and myRoot then
            local dist = (t.part.Position - myRoot.Position).Magnitude
            if dist > LockOn.settings.MaxRange * 1.2 then valid = false end
        end
        if not valid then
            local list = findTargets()
            if #list > 0 then lockTo(list[1]) else releaseLock() end
        end
    end

    if LockOn.settings.AutoAcquire and not LockOn.enabled then
        local list = findTargets()
        if #list > 0 then lockTo(list[1]) end
    end
end)

-- Câmera suave seguindo alvo
RunService.RenderStepped:Connect(function(dt)
    if not LockOn.enabled or not LockOn.target then return end
    if not LockOn.settings.CameraFollow then return end
    local target = LockOn.target
    if not target.part or not target.part.Parent then return end
    local camera = Workspace.CurrentCamera
    if not camera then return end
    local desired = CFrame.lookAt(camera.CFrame.Position, target.part.Position)
    camera.CFrame = camera.CFrame:Lerp(desired, math.min(dt * 10, 1))
end)

-- ═══════════════════════════════════════════════════════════════════════
-- PÁGINA: INÍCIO
-- ═══════════════════════════════════════════════════════════════════════
local pageInicio = createPage("inicio", "Início", 1)
local welcomeSection = createAutoSection(pageInicio, 1)
local welcomeTitle = new("TextLabel", {
    Size = UDim2.new(1, 0, 0, 24),
    BackgroundTransparency = 1,
    Text = "Bem-vindo!",
    TextSize = 18, Font = Enum.Font.GothamBold,
    TextColor3 = Theme.Text,
    TextXAlignment = Enum.TextXAlignment.Left,
    LayoutOrder = 1, Parent = welcomeSection,
})
bindTheme(welcomeTitle, "TextColor3", "Text")
local welcomeText = new("TextLabel", {
    Size = UDim2.new(1, 0, 0, 0),
    AutomaticSize = Enum.AutomaticSize.Y,
    BackgroundTransparency = 1,
    Text = "Sistema completo com lock-on. Use o botão ◎ no canto inferior direito (ou tecla Q) para travar no inimigo mais próximo. Clique novamente para ciclar alvos e ✕ para soltar.",
    TextSize = 14, Font = Enum.Font.Gotham,
    TextColor3 = Theme.TextDim,
    TextXAlignment = Enum.TextXAlignment.Left,
    TextYAlignment = Enum.TextYAlignment.Top,
    TextWrapped = true,
    LayoutOrder = 2, Parent = welcomeSection,
})
bindTheme(welcomeText, "TextColor3", "TextDim")

local tipsSection = createAutoSection(pageInicio, 2)
createSectionHeader(tipsSection, "Dicas rápidas", 1)
local tipsText = new("TextLabel", {
    Size = UDim2.new(1, 0, 0, 0),
    AutomaticSize = Enum.AutomaticSize.Y,
    BackgroundTransparency = 1,
    Text = "• Arraste a barra de título para mover o menu.\n• Use − / □ / × para minimizar, maximizar e fechar.\n• F2 abre/fecha o menu · Q trava/solta alvo.\n• Botão ◎ (canto inferior direito) sempre visível.\n• Personalize as cores na página Aparência.",
    TextSize = 13, Font = Enum.Font.Gotham,
    TextColor3 = Theme.TextDim,
    TextXAlignment = Enum.TextXAlignment.Left,
    TextYAlignment = Enum.TextYAlignment.Top,
    TextWrapped = true,
    LayoutOrder = 2, Parent = tipsSection,
})
bindTheme(tipsText, "TextColor3", "TextDim")

-- ═══════════════════════════════════════════════════════════════════════
-- PÁGINA: JOGADOR
-- ═══════════════════════════════════════════════════════════════════════
local pageJogador = createPage("jogador", "Jogador", 2)
local jogSection = createAutoSection(pageJogador, 1)
createSectionHeader(jogSection, "Informações do Jogador", 1)
local playerLabels = {
    name        = createInfoRow(jogSection, "Nome",        2),
    displayName = createInfoRow(jogSection, "DisplayName", 3),
    userId      = createInfoRow(jogSection, "UserId",      4),
    state       = createInfoRow(jogSection, "Estado",      5),
}

-- ═══════════════════════════════════════════════════════════════════════
-- PÁGINA: SERVIDOR
-- ═══════════════════════════════════════════════════════════════════════
local pageServidor = createPage("servidor", "Servidor", 3)
local srvSection = createAutoSection(pageServidor, 1)
createSectionHeader(srvSection, "Informações do Servidor", 1)
local serverLabels = {
    experience  = createInfoRow(srvSection, "Experiência", 2),
    placeId     = createInfoRow(srvSection, "PlaceId",     3),
    jobId       = createInfoRow(srvSection, "JobId",       4),
    playerCount = createInfoRow(srvSection, "Jogadores",   5),
}

-- ═══════════════════════════════════════════════════════════════════════
-- PÁGINA: INFORMAÇÕES
-- ═══════════════════════════════════════════════════════════════════════
local pageInfo = createPage("info", "Informações", 4)
local infoSection = createAutoSection(pageInfo, 1)
createSectionHeader(infoSection, "Sessão", 1)
local sessionLabels = {
    entryTime   = createInfoRow(infoSection, "Entrada",         2),
    sessionTime = createInfoRow(infoSection, "Tempo de sessão", 3),
    currentTime = createInfoRow(infoSection, "Horário atual",   4),
}
local posSection = createAutoSection(pageInfo, 2)
createSectionHeader(posSection, "Personagem", 1)
local posLabels = {
    position = createInfoRow(posSection, "Posição (X, Y, Z)", 2),
    state    = createInfoRow(posSection, "Estado",            3),
}

-- ═══════════════════════════════════════════════════════════════════════
-- PÁGINA: APARÊNCIA
-- ═══════════════════════════════════════════════════════════════════════
local pageAparencia = createPage("aparencia", "Aparência", 5)
local presetsSection = createAutoSection(pageAparencia, 1)
createSectionHeader(presetsSection, "Paletas predefinidas", 1)

local presetRow = new("Frame", {
    Size = UDim2.new(1, 0, 0, 36),
    BackgroundTransparency = 1,
    LayoutOrder = 2, Parent = presetsSection,
})
new("UIListLayout", {
    FillDirection = Enum.FillDirection.Horizontal,
    Padding = UDim.new(0, 8),
    SortOrder = Enum.SortOrder.LayoutOrder,
    VerticalAlignment = Enum.VerticalAlignment.Center,
    Parent = presetRow,
})

local presetPalettes = {
    { colors = { Primary = Color3.fromRGB(45,125,210),  Background = Color3.fromRGB(28,28,34),  Surface = Color3.fromRGB(38,38,46),  Button = Color3.fromRGB(52,52,62),  Border = Color3.fromRGB(72,72,84)   } },
    { colors = { Primary = Color3.fromRGB(200,50,50),   Background = Color3.fromRGB(34,22,22),  Surface = Color3.fromRGB(46,32,32),  Button = Color3.fromRGB(62,44,44),  Border = Color3.fromRGB(84,60,60)   } },
    { colors = { Primary = Color3.fromRGB(140,70,200),  Background = Color3.fromRGB(28,22,38),  Surface = Color3.fromRGB(38,32,50),  Button = Color3.fromRGB(54,46,68),  Border = Color3.fromRGB(78,66,94)   } },
    { colors = { Primary = Color3.fromRGB(50,170,100),  Background = Color3.fromRGB(20,30,24),  Surface = Color3.fromRGB(30,42,34),  Button = Color3.fromRGB(42,58,48),  Border = Color3.fromRGB(60,82,68)   } },
    { colors = { Primary = Color3.fromRGB(20,20,20),    Background = Color3.fromRGB(14,14,14),  Surface = Color3.fromRGB(22,22,22),  Button = Color3.fromRGB(32,32,32),  Border = Color3.fromRGB(48,48,48)   } },
    { colors = { Primary = Color3.fromRGB(200,200,210), Background = Color3.fromRGB(240,240,245), Surface = Color3.fromRGB(230,230,235), Button = Color3.fromRGB(215,215,222), Border = Color3.fromRGB(180,180,190) } },
}

local updateColorEditorSliders

for i, preset in ipairs(presetPalettes) do
    local btn = new("TextButton", {
        Size = UDim2.fromOffset(36, 36),
        BackgroundColor3 = preset.colors.Primary,
        Text = "", AutoButtonColor = false,
        LayoutOrder = i, Parent = presetRow,
    })
    new("UICorner", { CornerRadius = UDim.new(1, 0), Parent = btn })
    new("UIStroke", { Color = Theme.Border, Thickness = 2, Parent = btn })
    btn.Activated:Connect(function()
        for k, v in pairs(preset.colors) do Theme[k] = v end
        applyTheme()
        if updateColorEditorSliders then updateColorEditorSliders() end
    end)
end

local editorSection = createAutoSection(pageAparencia, 2)
createSectionHeader(editorSection, "Cor personalizada", 1)

local targetRow = new("Frame", {
    Size = UDim2.new(1, 0, 0, 34),
    BackgroundTransparency = 1,
    LayoutOrder = 2, Parent = editorSection,
})
new("UIListLayout", {
    FillDirection = Enum.FillDirection.Horizontal,
    Padding = UDim.new(0, 6),
    SortOrder = Enum.SortOrder.LayoutOrder,
    Parent = targetRow,
})

local colorTargets = {
    { key = "Primary",    name = "Destaque" },
    { key = "Background", name = "Fundo"    },
    { key = "Button",     name = "Botões"   },
    { key = "Border",     name = "Bordas"   },
}
local selectedTarget = "Primary"
local targetButtons   = {}

local function updateTargetButtons()
    for key, btn in pairs(targetButtons) do
        btn.BackgroundColor3 = (key == selectedTarget) and Theme.Primary or Theme.Button
    end
end
onThemeChanged(updateTargetButtons)

for i, t in ipairs(colorTargets) do
    local btn = new("TextButton", {
        Size = UDim2.fromOffset(86, 30),
        BackgroundColor3 = Theme.Button,
        Text = t.name, TextSize = 12,
        Font = Enum.Font.GothamBold,
        TextColor3 = Theme.Text,
        AutoButtonColor = false,
        LayoutOrder = i, Parent = targetRow,
    })
    new("UICorner", { CornerRadius = UDim.new(0, 6), Parent = btn })
    bindTheme(btn, "TextColor3", "Text")
    targetButtons[t.key] = btn
    btn.Activated:Connect(function()
        selectedTarget = t.key
        updateTargetButtons()
        if updateColorEditorSliders then updateColorEditorSliders() end
    end)
end

local previewRow = new("Frame", {
    Size = UDim2.new(1, 0, 0, 28),
    BackgroundTransparency = 1,
    LayoutOrder = 3, Parent = editorSection,
})
local previewLabel = new("TextLabel", {
    Size = UDim2.new(1, -80, 1, 0),
    BackgroundTransparency = 1,
    Text = "Prévia:", TextSize = 13,
    Font = Enum.Font.GothamMedium,
    TextColor3 = Theme.TextDim,
    TextXAlignment = Enum.TextXAlignment.Left,
    Parent = previewRow,
})
bindTheme(previewLabel, "TextColor3", "TextDim")
local previewSwatch = new("Frame", {
    Size = UDim2.fromOffset(64, 22),
    Position = UDim2.new(1, -64, 0.5, 0),
    AnchorPoint = Vector2.new(0, 0.5),
    BackgroundColor3 = Theme[selectedTarget],
    BorderSizePixel = 0, Parent = previewRow,
})
new("UICorner", { CornerRadius = UDim.new(0, 5), Parent = previewSwatch })
new("UIStroke", { Color = Theme.Border, Thickness = 1, Parent = previewSwatch })

local colorSliders = {}
local function applyCustomColor()
    local c = Color3.fromRGB(colorSliders.R.getValue(), colorSliders.G.getValue(), colorSliders.B.getValue())
    Theme[selectedTarget] = c
    previewSwatch.BackgroundColor3 = c
    applyTheme()
end
colorSliders.R = createSlider(editorSection, "R", 4, applyCustomColor)
colorSliders.G = createSlider(editorSection, "G", 5, applyCustomColor)
colorSliders.B = createSlider(editorSection, "B", 6, applyCustomColor)

updateColorEditorSliders = function()
    local c = Theme[selectedTarget]
    colorSliders.R.set(math.floor(c.R * 255 + 0.5), false)
    colorSliders.G.set(math.floor(c.G * 255 + 0.5), false)
    colorSliders.B.set(math.floor(c.B * 255 + 0.5), false)
    previewSwatch.BackgroundColor3 = c
end
updateColorEditorSliders()

-- ═══════════════════════════════════════════════════════════════════════
-- PÁGINA: LOCK-ON
-- ═══════════════════════════════════════════════════════════════════════
local pageLock = createPage("lock", "Lock-On", 6)

local lockInfoSection = createAutoSection(pageLock, 1)
createSectionHeader(lockInfoSection, "Status", 1)
local lockStatusLabel = createInfoRow(lockInfoSection, "Status", 2)
local lockTargetLabel = createInfoRow(lockInfoSection, "Alvo atual", 3)
local lockCountLabel  = createInfoRow(lockInfoSection, "Alvos no alcance", 4)

local lockCfgSection = createAutoSection(pageLock, 2)
createSectionHeader(lockCfgSection, "Configurações", 1)

local rangeSlider = createSlider(lockCfgSection, "Alcance (studs)", 2, function(v)
    LockOn.settings.MaxRange = math.floor(20 + (v / 255) * 280)
end)
rangeSlider.set(116, true)

createToggle(lockCfgSection, "Câmera seguir alvo", LockOn.settings.CameraFollow, 3, function(v)
    LockOn.settings.CameraFollow = v
end)

createToggle(lockCfgSection, "Trava automática (mais próximo)", LockOn.settings.AutoAcquire, 4, function(v)
    LockOn.settings.AutoAcquire = v
end)

createToggle(lockCfgSection, "Mostrar contorno no alvo", LockOn.settings.ShowHighlight, 5, function(v)
    LockOn.settings.ShowHighlight = v
    if LockOn.target then applyHighlight(LockOn.target.char) end
end)

-- ═══════════════════════════════════════════════════════════════════════
-- PÁGINA: CRIADOR
-- ═══════════════════════════════════════════════════════════════════════
local pageCriador = createPage("criador", "Criador", 7)
local creatorSection = createAutoSection(pageCriador, 1)
createSectionHeader(creatorSection, "Sobre o Criador", 1)
local creatorLabels = {
    name    = createInfoRow(creatorSection, "Criador",  2),
    version = createInfoRow(creatorSection, "Versão",   3),
    credits = createInfoRow(creatorSection, "Créditos", 4),
}
creatorLabels.name.Text    = CONFIG.CreatorName
creatorLabels.version.Text = "v" .. CONFIG.Version
creatorLabels.credits.Text = CONFIG.Credits

local descSection = createAutoSection(pageCriador, 2)
createSectionHeader(descSection, "Descrição", 1)
local descLabel = new("TextLabel", {
    Size = UDim2.new(1, 0, 0, 0),
    AutomaticSize = Enum.AutomaticSize.Y,
    BackgroundTransparency = 1,
    Text = CONFIG.Description,
    TextSize = 13, Font = Enum.Font.Gotham,
    TextColor3 = Theme.TextDim,
    TextXAlignment = Enum.TextXAlignment.Left,
    TextYAlignment = Enum.TextYAlignment.Top,
    TextWrapped = true,
    LayoutOrder = 2, Parent = descSection,
})
bindTheme(descLabel, "TextColor3", "TextDim")

-- ═══════════════════════════════════════════════════════════════════════
-- PÁGINA: CONFIGURAÇÕES
-- ═══════════════════════════════════════════════════════════════════════
local pageConfig = createPage("config", "Configurações", 8)
local cfgSection = createAutoSection(pageConfig, 1)
createSectionHeader(cfgSection, "Interface", 1)
createToggle(cfgSection, "Animações", Settings.AnimationsEnabled, 2, function(v)
    Settings.AnimationsEnabled = v
end)
createToggle(cfgSection, "Sombra da janela", Settings.ShadowEnabled, 3, function(v)
    Settings.ShadowEnabled = v
    if windowState ~= "maximized" and windowState ~= "minimized" then
        shadow.Visible = v
    end
end)

-- ═══════════════════════════════════════════════════════════════════════
-- NAVEGAÇÃO
-- ═══════════════════════════════════════════════════════════════════════
addNav("inicio",    "⌂", "Início",        1)
addNav("jogador",   "☺", "Jogador",       2)
addNav("servidor",  "▣", "Servidor",      3)
addNav("info",      "⏱", "Informações",   4)
addNav("aparencia", "✦", "Aparência",     5)
addNav("lock",      "◎", "Lock-On",       6)
addNav("criador",   "★", "Criador",       7)
addNav("config",    "⚙", "Configurações", 8)

showPage("inicio")

-- ═══════════════════════════════════════════════════════════════════════
-- ATUALIZAÇÃO DE DADOS
-- ═══════════════════════════════════════════════════════════════════════
local experienceName = game.Name
task.spawn(function()
    local ok, info = pcall(function() return MarketplaceService:GetProductInfo(game.PlaceId) end)
    if ok and info and info.Name then
        experienceName = info.Name
        serverLabels.experience.Text = experienceName
    end
end)

local function getCharacterInfo()
    local char = LocalPlayer.Character
    if not char then return nil, "Sem personagem" end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    local pos = hrp and hrp.Position or nil
    local hum = char:FindFirstChildOfClass("Humanoid")
    local state = "—"
    if hum then
        local ok, s = pcall(function() return hum:GetState() end)
        if ok and s then state = tostring(s):gsub("Enum.HumanoidStateType.", "") end
    end
    return pos, state
end

playerLabels.name.Text        = LocalPlayer.Name
playerLabels.displayName.Text = LocalPlayer.DisplayName
playerLabels.userId.Text      = tostring(LocalPlayer.UserId)
serverLabels.experience.Text  = experienceName
serverLabels.placeId.Text     = tostring(game.PlaceId)
serverLabels.jobId.Text       = game.JobId ~= "" and game.JobId or "(Studio)"

local function updateAllInfo()
    playerLabels.name.Text        = LocalPlayer.Name
    playerLabels.displayName.Text = LocalPlayer.DisplayName
    playerLabels.userId.Text      = tostring(LocalPlayer.UserId)

    local pos, charState = getCharacterInfo()
    playerLabels.state.Text = charState
    posLabels.state.Text    = charState
    posLabels.position.Text = pos
        and string.format("%.1f, %.1f, %.1f", pos.X, pos.Y, pos.Z)
        or "—"

    serverLabels.experience.Text  = experienceName
    serverLabels.placeId.Text     = tostring(game.PlaceId)
    serverLabels.jobId.Text       = game.JobId ~= "" and game.JobId or "(Studio)"
    serverLabels.playerCount.Text = tostring(#Players:GetPlayers())

    sessionLabels.entryTime.Text   = entryTime:FormatLocalTime("HH:mm:ss", "en-us") .. " (local)"
    sessionLabels.sessionTime.Text = formatDuration(os.clock() - sessionStart)
    local now = DateTime.now()
    sessionLabels.currentTime.Text = string.format("%s (Local) · %s (UTC)",
        now:FormatLocalTime("HH:mm:ss", "en-us"),
        now:FormatUniversalTime("HH:mm:ss", "en-us"))

    lockStatusLabel.Text = LockOn.enabled and "ATIVO" or "Desligado"
    lockStatusLabel.TextColor3 = LockOn.enabled
        and Color3.fromRGB(255, 80, 80) or Theme.Text
    lockTargetLabel.Text = LockOn.target and LockOn.target.name or "—"
    lockCountLabel.Text = tostring(#findTargets())
end

updateAllInfo()

local accum = 0
RunService.Heartbeat:Connect(function(dt)
    if not windowRoot.Visible then return end
    accum = accum + dt
    if accum < 0.25 then return end
    accum = 0
    pcall(updateAllInfo)
end)

-- ═══════════════════════════════════════════════════════════════════════
-- RESPONSIVIDADE
-- ═══════════════════════════════════════════════════════════════════════
local function clampWindowToViewport()
    if not Workspace.CurrentCamera then return end
    local vp = Workspace.CurrentCamera.ViewportSize
    local maxW = math.max(380, vp.X - 40)
    local maxH = math.max(260, vp.Y - 40)
    local sz = windowRoot.AbsoluteSize
    if sz.X > maxW or sz.Y > maxH then
        windowRoot.Size = UDim2.fromOffset(math.min(sz.X, maxW), math.min(sz.Y, maxH))
    end
end
if Workspace.CurrentCamera then
    Workspace.CurrentCamera:GetPropertyChangedSignal("ViewportSize"):Connect(clampWindowToViewport)
end

-- ═══════════════════════════════════════════════════════════════════════
-- ANIMAÇÃO DE ABERTURA
-- ═══════════════════════════════════════════════════════════════════════
do
    local ts = windowRoot.Size
    if Settings.AnimationsEnabled then
        windowRoot.Size = UDim2.new(
            ts.X.Scale * 0.85, ts.X.Offset * 0.85,
            ts.Y.Scale * 0.85, ts.Y.Offset * 0.85
        )
        windowRoot.GroupTransparency = 1
        TweenService:Create(windowRoot,
            TweenInfo.new(0.35, Enum.EasingStyle.Quart, Enum.EasingDirection.Out),
            { Size = ts, GroupTransparency = 0 }
        ):Play()
    end
end
