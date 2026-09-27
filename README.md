-- ============================================================
-- Ash UI Library v2.0 — Lunar Edition (Smooth)
-- Плавная лунная GUI-библиотека с настройками
-- ============================================================

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local CoreGui = game:GetService("CoreGui")
local LocalPlayer = Players.LocalPlayer

-- ============================================================
-- ПАЛИТРЫ (можно переключать в настройках)
-- ============================================================
local Themes = {
    Lunar = {
        Background   = Color3.fromRGB(14, 16, 22),
        Surface      = Color3.fromRGB(26, 28, 36),
        SurfaceLight = Color3.fromRGB(66, 66, 66),
        SurfaceHover = Color3.fromRGB(90, 90, 90),
        Border       = Color3.fromRGB(110, 110, 110),
        BorderLight  = Color3.fromRGB(140, 140, 140),
        Text         = Color3.fromRGB(232, 234, 240),
        TextDim      = Color3.fromRGB(150, 154, 164),
        TextDisabled = Color3.fromRGB(96, 98, 108),
        Accent       = Color3.fromRGB(200, 205, 220),
        MoonGlow     = Color3.fromRGB(180, 190, 220),
    },
    Blood = {
        Background   = Color3.fromRGB(18, 8, 8),
        Surface      = Color3.fromRGB(30, 14, 14),
        SurfaceLight = Color3.fromRGB(66, 30, 30),
        SurfaceHover = Color3.fromRGB(90, 40, 40),
        Border       = Color3.fromRGB(120, 50, 50),
        BorderLight  = Color3.fromRGB(160, 70, 70),
        Text         = Color3.fromRGB(245, 220, 220),
        TextDim      = Color3.fromRGB(180, 140, 140),
        TextDisabled = Color3.fromRGB(110, 80, 80),
        Accent       = Color3.fromRGB(255, 80, 80),
        MoonGlow     = Color3.fromRGB(255, 60, 60),
    },
    Ocean = {
        Background   = Color3.fromRGB(8, 14, 20),
        Surface      = Color3.fromRGB(14, 24, 34),
        SurfaceLight = Color3.fromRGB(30, 60, 80),
        SurfaceHover = Color3.fromRGB(45, 85, 110),
        Border       = Color3.fromRGB(70, 120, 150),
        BorderLight  = Color3.fromRGB(100, 150, 180),
        Text         = Color3.fromRGB(220, 235, 245),
        TextDim      = Color3.fromRGB(140, 170, 190),
        TextDisabled = Color3.fromRGB(80, 110, 130),
        Accent       = Color3.fromRGB(80, 180, 255),
        MoonGlow     = Color3.fromRGB(60, 160, 255),
    },
    Forest = {
        Background   = Color3.fromRGB(10, 18, 12),
        Surface      = Color3.fromRGB(18, 28, 20),
        SurfaceLight = Color3.fromRGB(40, 60, 44),
        SurfaceHover = Color3.fromRGB(60, 85, 65),
        Border       = Color3.fromRGB(80, 110, 85),
        BorderLight  = Color3.fromRGB(110, 145, 115),
        Text         = Color3.fromRGB(225, 240, 225),
        TextDim      = Color3.fromRGB(150, 180, 150),
        TextDisabled = Color3.fromRGB(90, 115, 90),
        Accent       = Color3.fromRGB(120, 220, 130),
        MoonGlow     = Color3.fromRGB(100, 200, 110),
    },
}

-- ============================================================
-- ГЛОБАЛЬНЫЕ НАСТРОЙКИ ВСЕХ ОКОН
-- ============================================================
local GlobalSettings = {
    ThemeName = "Lunar",
    Transparency = 0.0,       -- прозрачность окна (0 = непрозрачно, 0.5 = полупрозрачно)
    OpenSpeed = 0.35,         -- скорость открытия/закрытия (сек)
    DragSmooth = 0.15,        -- плавность перетаскивания (0 = мгновенно, 0.3 = с инерцией)
    HideKey = Enum.KeyCode.K,
}

local function GetTheme()
    return Themes[GlobalSettings.ThemeName] or Themes.Lunar
end

-- ============================================================
-- УТИЛИТЫ
-- ============================================================
local function Create(class, props)
    local inst = Instance.new(class)
    if props then
        for k, v in pairs(props) do
            if k ~= "Parent" then inst[k] = v end
        end
        if props.Parent then inst.Parent = props.Parent end
    end
    return inst
end

local function Tween(inst, props, dur, style, dir)
    local info = TweenInfo.new(
        dur or 0.2,
        style or Enum.EasingStyle.Quad,
        dir or Enum.EasingDirection.Out
    )
    local t = TweenService:Create(inst, info, props)
    t:Play()
    return t
end

local function GetGuiParent()
    if gethui then
        local ok, hui = pcall(gethui)
        if ok and hui then return hui end
    end
    return LocalPlayer:WaitForChild("PlayerGui")
end

-- ============================================================
-- БИБЛИОТЕКА
-- ============================================================
local Ash = {}
Ash.__index = Ash
Ash.Windows = {}
Ash.Flags = {}
Ash.Themes = Themes
Ash.GlobalSettings = GlobalSettings
Ash.Theme = Themes.Lunar

function Ash:SetTheme(name)
    if Themes[name] then
        GlobalSettings.ThemeName = name
        Ash.Theme = Themes[name]
        -- Применяем ко всем окнам
        for _, win in ipairs(Ash.Windows) do
            if win.ApplyTheme then
                pcall(win.ApplyTheme, win)
            end
        end
    end
end

-- ============================================================
-- ЛУНА
-- ============================================================
local function BuildMoon(parent, size)
    local Glow = Create("Frame", {
        Size = UDim2.new(0, size * 2.4, 0, size * 2.4),
        Position = UDim2.new(0.5, 0, 0.5, 0),
        AnchorPoint = Vector2.new(0.5, 0.5),
        BackgroundColor3 = GetTheme().MoonGlow,
        BackgroundTransparency = 0.88,
        BorderSizePixel = 0,
        ZIndex = 0,
        Parent = parent
    })
    Create("UICorner", {CornerRadius = UDim.new(1, 0), Parent = Glow})

    local Glow2 = Create("Frame", {
        Size = UDim2.new(0, size * 1.6, 0, size * 1.6),
        Position = UDim2.new(0.5, 0, 0.5, 0),
        AnchorPoint = Vector2.new(0.5, 0.5),
        BackgroundColor3 = GetTheme().MoonGlow,
        BackgroundTransparency = 0.75,
        BorderSizePixel = 0,
        ZIndex = 0,
        Parent = parent
    })
    Create("UICorner", {CornerRadius = UDim.new(1, 0), Parent = Glow2})

    local MoonBody = Create("Frame", {
        Size = UDim2.new(0, size, 0, size),
        Position = UDim2.new(0.5, 0, 0.5, 0),
        AnchorPoint = Vector2.new(0.5, 0.5),
        BackgroundColor3 = GetTheme().Text,
        BackgroundTransparency = 0.15,
        BorderSizePixel = 0,
        ZIndex = 1,
        Parent = parent
    })
    Create("UICorner", {CornerRadius = UDim.new(1, 0), Parent = MoonBody})

    for _, c in ipairs({
        {pos = UDim2.new(0.28, 0, 0.32, 0), size = 0.22},
        {pos = UDim2.new(0.62, 0, 0.25, 0), size = 0.14},
        {pos = UDim2.new(0.55, 0, 0.62, 0), size = 0.20},
        {pos = UDim2.new(0.30, 0, 0.68, 0), size = 0.12},
    }) do
        local crater = Create("Frame", {
            Size = UDim2.new(c.size, 0, c.size, 0),
            Position = c.pos,
            BackgroundColor3 = GetTheme().SurfaceLight,
            BackgroundTransparency = 0.35,
            BorderSizePixel = 0,
            ZIndex = 2,
            Parent = MoonBody
        })
        Create("UICorner", {CornerRadius = UDim.new(1, 0), Parent = crater})
    end

    return MoonBody, Glow, Glow2
end

-- ============================================================
-- СОЗДАНИЕ ОКНА
-- ============================================================
function Ash:CreateWindow(config)
    config = config or {}
    local title = config.Title or "Ash"
    local size = config.Size or UDim2.new(0, 560, 0, 440)
    local guiParent = GetGuiParent()

    local ScreenGui = Create("ScreenGui", {
        Name = "Ash_UI_" .. tostring(math.random(100000, 999999)),
        ResetOnSpawn = false,
        IgnoreGuiInset = true,
        ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
        DisplayOrder = 999999,
        Parent = guiParent
    })

    local Main = Create("Frame", {
        Name = "Main",
        Size = size,
        Position = UDim2.new(0.5, -size.X.Offset / 2, 0.5, -size.Y.Offset / 2),
        BackgroundColor3 = GetTheme().Background,
        BackgroundTransparency = 1,
        BorderSizePixel = 0,
        ClipsDescendants = true,
        Parent = ScreenGui
    })
    Create("UICorner", {CornerRadius = UDim.new(0, 14), Parent = Main})
    local MainStroke = Create("UIStroke", {
        Color = GetTheme().Border,
        Thickness = 1,
        Transparency = 0.4,
        Parent = Main
    })

    -- Свечение
    local OuterGlow = Create("Frame", {
        Size = UDim2.new(1, 24, 1, 24),
        Position = UDim2.new(0, -12, 0, -12),
        BackgroundColor3 = GetTheme().MoonGlow,
        BackgroundTransparency = 1,
        BorderSizePixel = 0,
        ZIndex = -1,
        Parent = Main
    })
    Create("UICorner", {CornerRadius = UDim.new(0, 20), Parent = OuterGlow})

    -- Тень
    local Shadow = Create("Frame", {
        Size = UDim2.new(1, 6, 1, 6),
        Position = UDim2.new(0, -3, 0, -3),
        BackgroundColor3 = Color3.new(0, 0, 0),
        BackgroundTransparency = 0.55,
        BorderSizePixel = 0,
        ZIndex = -2,
        Parent = Main
    })
    Create("UICorner", {CornerRadius = UDim.new(0, 14), Parent = Shadow})

    -- Луна внутри
    local MoonLayer = Create("Frame", {
        Size = UDim2.new(1, 0, 1, 0),
        BackgroundTransparency = 1,
        ClipsDescendants = true,
        ZIndex = 0,
        Parent = Main
    })
    BuildMoon(MoonLayer, 240)

    -- TITLE BAR
    local TitleBar = Create("Frame", {
        Size = UDim2.new(1, 0, 0, 42),
        BackgroundColor3 = GetTheme().Surface,
        BackgroundTransparency = 0.15,
        BorderSizePixel = 0,
        ZIndex = 3,
        Parent = Main
    })
    Create("UICorner", {CornerRadius = UDim.new(0, 14), Parent = TitleBar})
    Create("Frame", {
        Size = UDim2.new(1, 0, 0, 12),
        Position = UDim2.new(0, 0, 1, -12),
        BackgroundColor3 = GetTheme().Surface,
        BackgroundTransparency = 0.15,
        BorderSizePixel = 0,
        ZIndex = 3,
        Parent = TitleBar
    })
    local TitleBorder = Create("Frame", {
        Size = UDim2.new(1, 0, 0, 1),
        Position = UDim2.new(0, 0, 1, -1),
        BackgroundColor3 = GetTheme().Border,
        BackgroundTransparency = 0.5,
        BorderSizePixel = 0,
        ZIndex = 3,
        Parent = TitleBar
    })

    -- Иконка-луна
    local IconFrame = Create("Frame", {
        Size = UDim2.new(0, 24, 0, 24),
        Position = UDim2.new(0, 12, 0.5, -12),
        BackgroundColor3 = GetTheme().Accent,
        BorderSizePixel = 0,
        ZIndex = 4,
        Parent = TitleBar
    })
    Create("UICorner", {CornerRadius = UDim.new(1, 0), Parent = IconFrame})
    local IconCrater = Create("Frame", {
        Size = UDim2.new(0, 6, 0, 6),
        Position = UDim2.new(0.25, 0, 0.3, 0),
        BackgroundColor3 = GetTheme().SurfaceLight,
        BorderSizePixel = 0,
        ZIndex = 5,
        Parent = IconFrame
    })
    Create("UICorner", {CornerRadius = UDim.new(1, 0), Parent = IconCrater})

    local TitleLabel = Create("TextLabel", {
        Size = UDim2.new(1, -110, 1, 0),
        Position = UDim2.new(0, 46, 0, 0),
        BackgroundTransparency = 1,
        Text = title,
        TextColor3 = GetTheme().Text,
        TextSize = 13,
        Font = Enum.Font.GothamBold,
        TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 4,
        Parent = TitleBar
    })

    if config.Version then
        local VerLabel = Create("TextLabel", {
            Size = UDim2.new(0, 50, 1, 0),
            Position = UDim2.new(1, -120, 0, 0),
            BackgroundTransparency = 1,
            Text = "v" .. config.Version,
            TextColor3 = GetTheme().TextDisabled,
            TextSize = 10,
            Font = Enum.Font.Gotham,
            TextXAlignment = Enum.TextXAlignment.Right,
            ZIndex = 4,
            Parent = TitleBar
        })
    end

    local buttonSize = 26
    local MinimizeBtn = Create("TextButton", {
        Size = UDim2.new(0, buttonSize, 0, buttonSize),
        Position = UDim2.new(1, -buttonSize * 2 - 12, 0.5, -buttonSize / 2),
        BackgroundColor3 = GetTheme().SurfaceLight,
        BorderSizePixel = 0,
        Text = "",
        AutoButtonColor = false,
        ZIndex = 4,
        Parent = TitleBar
    })
    Create("UICorner", {CornerRadius = UDim.new(0, 6), Parent = MinimizeBtn})
    Create("Frame", {
        Size = UDim2.new(0, 10, 0, 2),
        Position = UDim2.new(0.5, -5, 0.5, -1),
        BackgroundColor3 = GetTheme().TextDim,
        BorderSizePixel = 0,
        ZIndex = 5,
        Parent = MinimizeBtn
    })

    local CloseBtn = Create("TextButton", {
        Size = UDim2.new(0, buttonSize, 0, buttonSize),
        Position = UDim2.new(1, -buttonSize - 12, 0.5, -buttonSize / 2),
        BackgroundColor3 = GetTheme().SurfaceLight,
        BorderSizePixel = 0,
        Text = "",
        AutoButtonColor = false,
        ZIndex = 4,
        Parent = TitleBar
    })
    Create("UICorner", {CornerRadius = UDim.new(0, 6), Parent = CloseBtn})
    local cross1 = Create("Frame", {
        Size = UDim2.new(0, 9, 0, 2),
        Position = UDim2.new(0.5, -4.5, 0.5, -1),
        BackgroundColor3 = GetTheme().TextDim,
        BorderSizePixel = 0,
        Rotation = 45,
        ZIndex = 5,
        Parent = CloseBtn
    })
    local cross2 = Create("Frame", {
        Size = UDim2.new(0, 9, 0, 2),
        Position = UDim2.new(0.5, -4.5, 0.5, -1),
        BackgroundColor3 = GetTheme().TextDim,
        BorderSizePixel = 0,
        Rotation = -45,
        ZIndex = 5,
        Parent = CloseBtn
    })

    MinimizeBtn.MouseEnter:Connect(function()
        Tween(MinimizeBtn, {BackgroundColor3 = GetTheme().SurfaceHover}, 0.15)
    end)
    MinimizeBtn.MouseLeave:Connect(function()
        Tween(MinimizeBtn, {BackgroundColor3 = GetTheme().SurfaceLight}, 0.15)
    end)
    CloseBtn.MouseEnter:Connect(function()
        Tween(CloseBtn, {BackgroundColor3 = GetTheme().Accent}, 0.15)
        Tween(cross1, {BackgroundColor3 = GetTheme().Background}, 0.15)
        Tween(cross2, {BackgroundColor3 = GetTheme().Background}, 0.15)
    end)
    CloseBtn.MouseLeave:Connect(function()
        Tween(CloseBtn, {BackgroundColor3 = GetTheme().SurfaceLight}, 0.15)
        Tween(cross1, {BackgroundColor3 = GetTheme().TextDim}, 0.15)
        Tween(cross2, {BackgroundColor3 = GetTheme().TextDim}, 0.15)
    end)

    -- ========================================
    -- ПЛАВНОЕ ПЕРЕТАСКИВАНИЕ С ИНЕРЦИЕЙ
    -- ========================================
    local dragging, dragStart, startPos = false, nil, nil
    local lastMoveTime, lastDelta = 0, Vector2.new(0, 0)

    local function smoothDragLoop()
        task.spawn(function()
            while dragging do
                task.wait()
            end
            -- Инерция после отпускания
            if GlobalSettings.DragSmooth > 0 and lastDelta.Magnitude > 1 then
                local vel = lastDelta
                local steps = 10
                for i = steps, 1, -1 do
                    local alpha = i / steps
                    Main.Position = UDim2.new(
                        Main.Position.X.Scale,
                        Main.Position.X.Offset + vel.X * alpha * 0.5,
                        Main.Position.Y.Scale,
                        Main.Position.Y.Offset + vel.Y * alpha * 0.5
                    )
                    task.wait(GlobalSettings.DragSmooth / steps)
                end
            end
        end)
    end

    TitleBar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
           or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = Main.Position
            lastMoveTime = tick()
            lastDelta = Vector2.new(0, 0)
        end
    end)
    TitleBar.InputChanged:Connect(function(input)
        if not dragging then return end
        if input.UserInputType == Enum.UserInputType.MouseMovement
           or input.UserInputType == Enum.UserInputType.Touch then
            local delta = input.Position - dragStart
            local targetPos = UDim2.new(
                startPos.X.Scale, startPos.X.Offset + delta.X,
                startPos.Y.Scale, startPos.Y.Offset + delta.Y
            )
            -- Плавное движение через Tween
            Tween(Main, {Position = targetPos}, GlobalSettings.DragSmooth, Enum.EasingStyle.Quad)
            lastDelta = input.Position - (dragStart + delta)
            lastMoveTime = tick()
        end
    end)
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
           or input.UserInputType == Enum.UserInputType.Touch then
            dragging = false
            smoothDragLoop()
        end
    end)

    -- ========================================
    -- КОНТЕНТ
    -- ========================================
    local Container = Create("Frame", {
        Size = UDim2.new(1, 0, 1, -42),
        Position = UDim2.new(0, 0, 0, 42),
        BackgroundTransparency = 1,
        ZIndex = 2,
        Parent = Main
    })

    local Sidebar = Create("Frame", {
        Size = UDim2.new(0, 150, 1, 0),
        BackgroundColor3 = GetTheme().Surface,
        BackgroundTransparency = 0.2,
        BorderSizePixel = 0,
        ZIndex = 2,
        Parent = Container
    })
    Create("Frame", {
        Size = UDim2.new(0, 1, 1, 0),
        Position = UDim2.new(1, -1, 0, 0),
        BackgroundColor3 = GetTheme().Border,
        BackgroundTransparency = 0.5,
        BorderSizePixel = 0,
        ZIndex = 3,
        Parent = Sidebar
    })

    local TabContainer = Create("ScrollingFrame", {
        Size = UDim2.new(1, 0, 1, -64),
        Position = UDim2.new(0, 0, 0, 8),
        BackgroundTransparency = 1,
        BorderSizePixel = 0,
        ScrollBarThickness = 0,
        CanvasSize = UDim2.new(0, 0, 0, 0),
        AutomaticCanvasSize = Enum.AutomaticSize.Y,
        ZIndex = 3,
        Parent = Sidebar
    })
    Create("UIListLayout", {
        Padding = UDim.new(0, 4),
        SortOrder = Enum.SortOrder.LayoutOrder,
        Parent = TabContainer
    })
    Create("UIPadding", {
        PaddingLeft = UDim.new(0, 6),
        PaddingRight = UDim.new(0, 6),
        Parent = TabContainer
    })

    local UserInfo = Create("Frame", {
        Size = UDim2.new(1, -12, 0, 46),
        Position = UDim2.new(0, 6, 1, -52),
        BackgroundColor3 = GetTheme().SurfaceLight,
        BackgroundTransparency = 0.3,
        BorderSizePixel = 0,
        ZIndex = 3,
        Parent = Sidebar
    })
    Create("UICorner", {CornerRadius = UDim.new(0, 8), Parent = UserInfo})
    local UnameLabel = Create("TextLabel", {
        Size = UDim2.new(1, -12, 0, 16),
        Position = UDim2.new(0, 8, 0, 6),
        BackgroundTransparency = 1,
        Text = LocalPlayer.Name,
        TextColor3 = GetTheme().Text,
        TextSize = 11,
        Font = Enum.Font.GothamBold,
        TextXAlignment = Enum.TextXAlignment.Left,
        TextTruncate = Enum.TextTruncate.AtEnd,
        ZIndex = 4,
        Parent = UserInfo
    })
    local UiLabel = Create("TextLabel", {
        Size = UDim2.new(1, -12, 0, 12),
        Position = UDim2.new(0, 8, 0, 24),
        BackgroundTransparency = 1,
        Text = "Lunar UI v2",
        TextColor3 = GetTheme().TextDisabled,
        TextSize = 9,
        Font = Enum.Font.Gotham,
        TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 4,
        Parent = UserInfo
    })

    local PageContainer = Create("Frame", {
        Size = UDim2.new(1, -150, 1, 0),
        Position = UDim2.new(0, 150, 0, 0),
        BackgroundTransparency = 1,
        ZIndex = 2,
        Parent = Container
    })

    -- Объект окна
    local Window = {
        Gui = ScreenGui,
        Main = Main,
        TitleBar = TitleBar,
        Sidebar = Sidebar,
        TabContainer = TabContainer,
        PageContainer = PageContainer,
        Tabs = {},
        ActiveTab = nil,
        Hidden = false,
        OrigSize = size,
        Config = config,
        _MainStroke = MainStroke,
        _OuterGlow = OuterGlow,
        _TitleBorder = TitleBorder,
        _IconFrame = IconFrame,
        _IconCrater = IconCrater,
        _TitleLabel = TitleLabel,
        _UnameLabel = UnameLabel,
        _UiLabel = UiLabel,
        _MinimizeBtn = MinimizeBtn,
        _CloseBtn = CloseBtn,
        _Sidebar = Sidebar,
    }
    setmetatable(Window, {__index = Ash})

    -- ========================================
    -- ПРИМЕНЕНИЕ ТЕМЫ
    -- ========================================
    function Window:ApplyTheme()
        local t = GetTheme()
        Main.BackgroundColor3 = t.Background
        TitleBar.BackgroundColor3 = t.Surface
        TitleBorder.BackgroundColor3 = t.Border
        IconFrame.BackgroundColor3 = t.Accent
        IconCrater.BackgroundColor3 = t.SurfaceLight
        TitleLabel.TextColor3 = t.Text
        UnameLabel.TextColor3 = t.Text
        UiLabel.TextColor3 = t.TextDisabled
        MinimizeBtn.BackgroundColor3 = t.SurfaceLight
        CloseBtn.BackgroundColor3 = t.SurfaceLight
        Sidebar.BackgroundColor3 = t.Surface
        UserInfo.BackgroundColor3 = t.SurfaceLight
        MainStroke.Color = t.Border
        OuterGlow.BackgroundColor3 = t.MoonGlow
    end

    -- ========================================
    -- МЕТОД: Создать таб
    -- ========================================
    function Window:CreateTab(tabConfig)
        tabConfig = tabConfig or {}
        local tabName = tabConfig.Name or "Tab"
        local t = GetTheme()

        local TabBtn = Create("TextButton", {
            Size = UDim2.new(1, 0, 0, 32),
            BackgroundColor3 = t.Background,
            BackgroundTransparency = 1,
            BorderSizePixel = 0,
            Text = "",
            AutoButtonColor = false,
            ZIndex = 4,
            Parent = self.TabContainer
        })
        Create("UICorner", {CornerRadius = UDim.new(0, 6), Parent = TabBtn})

        local Indicator = Create("Frame", {
            Size = UDim2.new(0, 3, 0, 16),
            Position = UDim2.new(0, 0, 0.5, -8),
            BackgroundColor3 = t.Accent,
            BorderSizePixel = 0,
            Visible = false,
            ZIndex = 5,
            Parent = TabBtn
        })
        Create("UICorner", {CornerRadius = UDim.new(0, 2), Parent = Indicator})

        local TabLabel = Create("TextLabel", {
            Size = UDim2.new(1, -14, 1, 0),
            Position = UDim2.new(0, 14, 0, 0),
            BackgroundTransparency = 1,
            Text = tabName,
            TextColor3 = t.TextDim,
            TextSize = 11,
            Font = Enum.Font.GothamMedium,
            TextXAlignment = Enum.TextXAlignment.Left,
            ZIndex = 5,
            Parent = TabBtn
        })

        local Page = Create("ScrollingFrame", {
            Size = UDim2.new(1, -16, 1, -16),
            Position = UDim2.new(0, 8, 0, 8),
            BackgroundTransparency = 1,
            BorderSizePixel = 0,
            ScrollBarThickness = 3,
            ScrollBarImageColor3 = t.TextDim,
            ScrollBarImageTransparency = 0.5,
            CanvasSize = UDim2.new(0, 0, 0, 0),
            AutomaticCanvasSize = Enum.AutomaticSize.Y,
            Visible = false,
            ZIndex = 3,
            Parent = self.PageContainer
        })
        Create("UIListLayout", {
            Padding = UDim.new(0, 6),
            SortOrder = Enum.SortOrder.LayoutOrder,
            Parent = Page
        })
        Create("UIPadding", {
            PaddingBottom = UDim.new(0, 16),
            PaddingRight = UDim.new(0, 6),
            Parent = Page
        })

        local tab = {
            Name = tabName,
            Button = TabBtn,
            Page = Page,
            Indicator = Indicator,
            Label = TabLabel
        }

        TabBtn.MouseButton1Click:Connect(function()
            self:SelectTab(tab)
        end)
        TabBtn.MouseEnter:Connect(function()
            if self.ActiveTab ~= tab then
                Tween(TabBtn, {BackgroundTransparency = 0.85}, 0.15)
            end
        end)
        TabBtn.MouseLeave:Connect(function()
            if self.ActiveTab ~= tab then
                Tween(TabBtn, {BackgroundTransparency = 1}, 0.15)
            end
        end)

        table.insert(self.Tabs, tab)
        if #self.Tabs == 1 then
            self:SelectTab(tab)
        end

        -- ========================================
        -- ПЛАВНАЯ СМЕНА СТРАНИЦ
        -- ========================================
        local function AnimatePageOpen(page)
            page.Visible = true
            page.Position = UDim2.new(0, 8 + 20, 0, 8)
            page.BackgroundTransparency = 1
            Tween(page, {Position = UDim2.new(0, 8, 0, 8)}, 0.25, Enum.EasingStyle.Quart)
        end

        -- Section
        function tab:CreateSection(sectionName)
            local tt = GetTheme()
            local Sec = Create("Frame", {
                Size = UDim2.new(1, 0, 0, 22),
                BackgroundTransparency = 1,
                ZIndex = 4,
                Parent = self.Page
            })
            Create("TextLabel", {
                Size = UDim2.new(1, 0, 1, 0),
                BackgroundTransparency = 1,
                Text = string.upper(sectionName),
                TextColor3 = tt.TextDisabled,
                TextSize = 9,
                Font = Enum.Font.GothamBold,
                TextXAlignment = Enum.TextXAlignment.Left,
                ZIndex = 5,
                Parent = Sec
            })
            Create("Frame", {
                Size = UDim2.new(1, -90, 0, 1),
                Position = UDim2.new(0, 90, 0.5, 0),
                BackgroundColor3 = tt.Border,
                BackgroundTransparency = 0.4,
                BorderSizePixel = 0,
                ZIndex = 5,
                Parent = Sec
            })
            return Sec
        end

        -- Button
        function tab:CreateButton(cfg)
            cfg = cfg or {}
            local tt = GetTheme()
            local Btn = Create("TextButton", {
                Size = UDim2.new(1, 0, 0, 34),
                BackgroundColor3 = tt.SurfaceLight,
                BackgroundTransparency = 0.15,
                BorderSizePixel = 0,
                Text = "",
                AutoButtonColor = false,
                ZIndex = 4,
                Parent = self.Page
            })
            Create("UICorner", {CornerRadius = UDim.new(0, 6), Parent = Btn})
            Create("UIStroke", {Color = tt.Border, Thickness = 1, Transparency = 0.5, Parent = Btn})
            local L = Create("TextLabel", {
                Size = UDim2.new(1, -16, 1, 0),
                Position = UDim2.new(0, 10, 0, 0),
                BackgroundTransparency = 1,
                Text = cfg.Name or "Button",
                TextColor3 = tt.Text,
                TextSize = 11,
                Font = Enum.Font.GothamMedium,
                TextXAlignment = Enum.TextXAlignment.Left,
                ZIndex = 5,
                Parent = Btn
            })
            Btn.MouseEnter:Connect(function()
                Tween(Btn, {BackgroundColor3 = tt.SurfaceHover, BackgroundTransparency = 0.05}, 0.15)
            end)
            Btn.MouseLeave:Connect(function()
                Tween(Btn, {BackgroundColor3 = tt.SurfaceLight, BackgroundTransparency = 0.15}, 0.15)
            end)
            Btn.MouseButton1Click:Connect(function()
                pcall(cfg.Callback or function() end)
            end)
            return Btn
        end

        -- Toggle
        function tab:CreateToggle(cfg)
            cfg = cfg or {}
            local tt = GetTheme()
            local default = cfg.Default or false
            local callback = cfg.Callback or function() end
            local flag = cfg.Flag

            local Frame = Create("Frame", {
                Size = UDim2.new(1, 0, 0, 34),
                BackgroundColor3 = tt.SurfaceLight,
                BackgroundTransparency = 0.15,
                BorderSizePixel = 0,
                ZIndex = 4,
                Parent = self.Page
            })
            Create("UICorner", {CornerRadius = UDim.new(0, 6), Parent = Frame})
            Create("UIStroke", {Color = tt.Border, Thickness = 1, Transparency = 0.5, Parent = Frame})
            Create("TextLabel", {
                Size = UDim2.new(1, -60, 1, 0),
                Position = UDim2.new(0, 10, 0, 0),
