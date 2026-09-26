-- ============================================================
-- Ash UI Library v1.0
-- Чёрно-белая минималистичная GUI-библиотека
-- Совместимо с Real / Xeno инжектором
-- Скачивается через loadstring(game:HttpGet(...))
-- ============================================================

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local CoreGui = game:GetService("CoreGui")
local LocalPlayer = Players.LocalPlayer

-- ============================================================
-- ПАЛИТРА (чёрно-белая)
-- ============================================================
local Palette = {
    Background = Color3.fromRGB(10, 10, 10),
    Surface = Color3.fromRGB(18, 18, 18),
    SurfaceLight = Color3.fromRGB(26, 26, 26),
    SurfaceHover = Color3.fromRGB(34, 34, 34),
    Border = Color3.fromRGB(45, 45, 45),
    BorderLight = Color3.fromRGB(70, 70, 70),
    Text = Color3.fromRGB(255, 255, 255),
    TextDim = Color3.fromRGB(160, 160, 160),
    TextDisabled = Color3.fromRGB(90, 90, 90),
    Accent = Color3.fromRGB(255, 255, 255),
}

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

local function Tween(inst, props, dur, style)
    local info = TweenInfo.new(
        dur or 0.2,
        style or Enum.EasingStyle.Quad,
        Enum.EasingDirection.Out
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
Ash.Theme = Palette

-- ============================================================
-- СОЗДАНИЕ ОКНА
-- ============================================================
function Ash:CreateWindow(config)
    config = config or {}
    local title = config.Title or "Ash"
    local size = config.Size or UDim2.new(0, 480, 0, 380)
    local theme = Palette
    local guiParent = GetGuiParent()

    -- Основной ScreenGui
    local ScreenGui = Create("ScreenGui", {
        Name = "Ash_UI_" .. tostring(math.random(100000, 999999)),
        ResetOnSpawn = false,
        IgnoreGuiInset = true,
        ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
        DisplayOrder = 999999,
        Parent = guiParent
    })

    -- ========================================
    -- ГЛАВНОЕ ОКНО
    -- ========================================
    local Main = Create("Frame", {
        Name = "Main",
        Size = size,
        Position = UDim2.new(0.5, -size.X.Offset / 2, 0.5, -size.Y.Offset / 2),
        BackgroundColor3 = theme.Background,
        BorderSizePixel = 0,
        ClipsDescendants = true,
        Parent = ScreenGui
    })
    Create("UICorner", {CornerRadius = UDim.new(0, 10), Parent = Main})
    Create("UIStroke", {Color = theme.Border, Thickness = 1, Transparency = 0.3, Parent = Main})

    -- Тень
    local Shadow = Create("Frame", {
        Size = UDim2.new(1, 4, 1, 4),
        Position = UDim2.new(0, -2, 0, -2),
        BackgroundColor3 = Color3.new(0, 0, 0),
        BackgroundTransparency = 0.5,
        BorderSizePixel = 0,
        ZIndex = -1,
        Parent = Main
    })
    Create("UICorner", {CornerRadius = UDim.new(0, 10), Parent = Shadow})

    -- ========================================
    -- TITLE BAR
    -- ========================================
    local TitleBar = Create("Frame", {
        Size = UDim2.new(1, 0, 0, 40),
        BackgroundColor3 = theme.Surface,
        BorderSizePixel = 0,
        Parent = Main
    })
    Create("UICorner", {CornerRadius = UDim.new(0, 10), Parent = TitleBar})
    Create("Frame", {
        Size = UDim2.new(1, 0, 0, 12),
        Position = UDim2.new(0, 0, 1, -12),
        BackgroundColor3 = theme.Surface,
        BorderSizePixel = 0,
        Parent = TitleBar
    })
    Create("Frame", {
        Size = UDim2.new(1, 0, 0, 1),
        Position = UDim2.new(0, 0, 1, -1),
        BackgroundColor3 = theme.Border,
        BorderSizePixel = 0,
        Parent = TitleBar
    })

    -- Иконка "A"
    local IconFrame = Create("Frame", {
        Size = UDim2.new(0, 22, 0, 22),
        Position = UDim2.new(0, 12, 0.5, -11),
        BackgroundColor3 = theme.Accent,
        BorderSizePixel = 0,
        Parent = TitleBar
    })
    Create("UICorner", {CornerRadius = UDim.new(0, 6), Parent = IconFrame})
    Create("TextLabel", {
        Size = UDim2.new(1, 0, 1, 0),
        BackgroundTransparency = 1,
        Text = "A",
        TextColor3 = theme.Background,
        TextSize = 14,
        Font = Enum.Font.GothamBold,
        Parent = IconFrame
    })

    -- Заголовок
    Create("TextLabel", {
        Size = UDim2.new(1, -100, 1, 0),
        Position = UDim2.new(0, 42, 0, 0),
        BackgroundTransparency = 1,
        Text = title,
        TextColor3 = theme.Text,
        TextSize = 13,
        Font = Enum.Font.GothamBold,
        TextXAlignment = Enum.TextXAlignment.Left,
        Parent = TitleBar
    })

    -- Версия
    if config.Version then
        Create("TextLabel", {
            Size = UDim2.new(0, 50, 1, 0),
            Position = UDim2.new(1, -120, 0, 0),
            BackgroundTransparency = 1,
            Text = "v" .. config.Version,
            TextColor3 = theme.TextDisabled,
            TextSize = 10,
            Font = Enum.Font.Gotham,
            TextXAlignment = Enum.TextXAlignment.Right,
            Parent = TitleBar
        })
    end

    -- Кнопки в TitleBar
    local buttonSize = 24

    local MinimizeBtn = Create("TextButton", {
        Size = UDim2.new(0, buttonSize, 0, buttonSize),
        Position = UDim2.new(1, -buttonSize * 2 - 10, 0.5, -buttonSize / 2),
        BackgroundColor3 = theme.SurfaceLight,
        BorderSizePixel = 0,
        Text = "",
        AutoButtonColor = false,
        Parent = TitleBar
    })
    Create("UICorner", {CornerRadius = UDim.new(0, 6), Parent = MinimizeBtn})
    Create("Frame", {
        Size = UDim2.new(0, 10, 0, 2),
        Position = UDim2.new(0.5, -5, 0.5, -1),
        BackgroundColor3 = theme.TextDim,
        BorderSizePixel = 0,
        Parent = MinimizeBtn
    })

    local CloseBtn = Create("TextButton", {
        Size = UDim2.new(0, buttonSize, 0, buttonSize),
        Position = UDim2.new(1, -buttonSize - 10, 0.5, -buttonSize / 2),
        BackgroundColor3 = theme.SurfaceLight,
        BorderSizePixel = 0,
        Text = "",
        AutoButtonColor = false,
        Parent = TitleBar
    })
    Create("UICorner", {CornerRadius = UDim.new(0, 6), Parent = CloseBtn})
    local cross1 = Create("Frame", {
        Size = UDim2.new(0, 9, 0, 2),
        Position = UDim2.new(0.5, -4.5, 0.5, -1),
        BackgroundColor3 = theme.TextDim,
        BorderSizePixel = 0,
        Rotation = 45,
        Parent = CloseBtn
    })
    local cross2 = Create("Frame", {
        Size = UDim2.new(0, 9, 0, 2),
        Position = UDim2.new(0.5, -4.5, 0.5, -1),
        BackgroundColor3 = theme.TextDim,
        BorderSizePixel = 0,
        Rotation = -45,
        Parent = CloseBtn
    })

    MinimizeBtn.MouseEnter:Connect(function()
        Tween(MinimizeBtn, {BackgroundColor3 = theme.SurfaceHover}, 0.15)
    end)
    MinimizeBtn.MouseLeave:Connect(function()
        Tween(MinimizeBtn, {BackgroundColor3 = theme.SurfaceLight}, 0.15)
    end)
    CloseBtn.MouseEnter:Connect(function()
        Tween(CloseBtn, {BackgroundColor3 = theme.Text}, 0.15)
        Tween(cross1, {BackgroundColor3 = theme.Background}, 0.15)
        Tween(cross2, {BackgroundColor3 = theme.Background}, 0.15)
    end)
    CloseBtn.MouseLeave:Connect(function()
        Tween(CloseBtn, {BackgroundColor3 = theme.SurfaceLight}, 0.15)
        Tween(cross1, {BackgroundColor3 = theme.TextDim}, 0.15)
        Tween(cross2, {BackgroundColor3 = theme.TextDim}, 0.15)
    end)

    -- Drag
    local dragging, dragStart, startPos = false, nil, nil
    TitleBar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
           or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = Main.Position
        end
    end)
    TitleBar.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement
                          or input.UserInputType == Enum.UserInputType.Touch) then
            local delta = input.Position - dragStart
            Main.Position = UDim2.new(
                startPos.X.Scale, startPos.X.Offset + delta.X,
                startPos.Y.Scale, startPos.Y.Offset + delta.Y
            )
        end
    end)
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
           or input.UserInputType == Enum.UserInputType.Touch then
            dragging = false
        end
    end)

    -- ========================================
    -- КОНТЕНТ
    -- ========================================
    local Container = Create("Frame", {
        Size = UDim2.new(1, 0, 1, -40),
        Position = UDim2.new(0, 0, 0, 40),
        BackgroundTransparency = 1,
        Parent = Main
    })

    local Sidebar = Create("Frame", {
        Size = UDim2.new(0, 130, 1, 0),
        BackgroundColor3 = theme.Surface,
        BorderSizePixel = 0,
        Parent = Container
    })
    Create("Frame", {
        Size = UDim2.new(0, 1, 1, 0),
        Position = UDim2.new(1, -1, 0, 0),
        BackgroundColor3 = theme.Border,
        BorderSizePixel = 0,
        Parent = Sidebar
    })

    local TabContainer = Create("ScrollingFrame", {
        Size = UDim2.new(1, 0, 1, -60),
        Position = UDim2.new(0, 0, 0, 8),
        BackgroundTransparency = 1,
        BorderSizePixel = 0,
        ScrollBarThickness = 0,
        CanvasSize = UDim2.new(0, 0, 0, 0),
        AutomaticCanvasSize = Enum.AutomaticSize.Y,
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

    -- Инфо игрока
    local UserInfo = Create("Frame", {
        Size = UDim2.new(1, -12, 0, 44),
        Position = UDim2.new(0, 6, 1, -50),
        BackgroundColor3 = theme.SurfaceLight,
        BorderSizePixel = 0,
        Parent = Sidebar
    })
    Create("UICorner", {CornerRadius = UDim.new(0, 6), Parent = UserInfo})
    Create("TextLabel", {
        Size = UDim2.new(1, -12, 0, 16),
        Position = UDim2.new(0, 8, 0, 6),
        BackgroundTransparency = 1,
        Text = LocalPlayer.Name,
        TextColor3 = theme.Text,
        TextSize = 11,
        Font = Enum.Font.GothamBold,
        TextXAlignment = Enum.TextXAlignment.Left,
        TextTruncate = Enum.TextTruncate.AtEnd,
        Parent = UserInfo
    })
    Create("TextLabel", {
        Size = UDim2.new(1, -12, 0, 12),
        Position = UDim2.new(0, 8, 0, 22),
        BackgroundTransparency = 1,
        Text = "Ash UI",
        TextColor3 = theme.TextDisabled,
        TextSize = 9,
        Font = Enum.Font.Gotham,
        TextXAlignment = Enum.TextXAlignment.Left,
        Parent = UserInfo
    })

    -- Область страниц
    local PageContainer = Create("Frame", {
        Size = UDim2.new(1, -130, 1, 0),
        Position = UDim2.new(0, 130, 0, 0),
        BackgroundTransparency = 1,
        Parent = Container
    })

    -- ========================================
    -- ОБЪЕКТ ОКНА
    -- ========================================
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
        OrigPosition = Main.Position,
        Config = config
    }
    setmetatable(Window, {__index = Ash})

    -- ========================================
    -- МЕТОД: Создать таб
    -- ========================================
    function Window:CreateTab(tabConfig)
        tabConfig = tabConfig or {}
        local tabName = tabConfig.Name or "Tab"

        local TabBtn = Create("TextButton", {
            Size = UDim2.new(1, 0, 0, 30),
            BackgroundColor3 = theme.Background,
            BackgroundTransparency = 1,
            BorderSizePixel = 0,
            Text = "",
            AutoButtonColor = false,
            Parent = self.TabContainer
        })
        Create("UICorner", {CornerRadius = UDim.new(0, 6), Parent = TabBtn})

        local Indicator = Create("Frame", {
            Size = UDim2.new(0, 3, 0, 14),
            Position = UDim2.new(0, 0, 0.5, -7),
            BackgroundColor3 = theme.Accent,
            BorderSizePixel = 0,
            Visible = false,
            Parent = TabBtn
        })
        Create("UICorner", {CornerRadius = UDim.new(0, 2), Parent = Indicator})

        local TabLabel = Create("TextLabel", {
            Size = UDim2.new(1, -12, 1, 0),
            Position = UDim2.new(0, 12, 0, 0),
            BackgroundTransparency = 1,
            Text = tabName,
            TextColor3 = theme.TextDim,
            TextSize = 11,
            Font = Enum.Font.GothamMedium,
            TextXAlignment = Enum.TextXAlignment.Left,
            Parent = TabBtn
        })

        local Page = Create("ScrollingFrame", {
            Size = UDim2.new(1, -16, 1, -16),
            Position = UDim2.new(0, 8, 0, 8),
            BackgroundTransparency = 1,
            BorderSizePixel = 0,
            ScrollBarThickness = 3,
            ScrollBarImageColor3 = theme.TextDim,
            ScrollBarImageTransparency = 0.5,
            CanvasSize = UDim2.new(0, 0, 0, 0),
            AutomaticCanvasSize = Enum.AutomaticSize.Y,
            Visible = false,
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
        -- Секция
        -- ========================================
        function tab:CreateSection(sectionName)
            local Sec = Create("Frame", {
                Size = UDim2.new(1, 0, 0, 20),
                BackgroundTransparency = 1,
                Parent = self.Page
            })
            Create("TextLabel", {
                Size = UDim2.new(1, 0, 1, 0),
                BackgroundTransparency = 1,
                Text = string.upper(sectionName),
                TextColor3 = theme.TextDisabled,
                TextSize = 9,
                Font = Enum.Font.GothamBold,
                TextXAlignment = Enum.TextXAlignment.Left,
                Parent = Sec
            })
            Create("Frame", {
                Size = UDim2.new(1, -80, 0, 1),
                Position = UDim2.new(0, 80, 0.5, 0),
                BackgroundColor3 = theme.Border,
                BorderSizePixel = 0,
                Parent = Sec
            })
            return Sec
        end

        -- ========================================
        -- Кнопка
        -- ========================================
        function tab:CreateButton(cfg)
            cfg = cfg or {}
            local Btn = Create("TextButton", {
                Size = UDim2.new(1, 0, 0, 32),
                BackgroundColor3 = theme.SurfaceLight,
                BorderSizePixel = 0,
                Text = "",
                AutoButtonColor = false,
                Parent = self.Page
            })
            Create("UICorner", {CornerRadius = UDim.new(0, 6), Parent = Btn})
            Create("UIStroke", {Color = theme.Border, Thickness = 1, Parent = Btn})
            Create("TextLabel", {
                Size = UDim2.new(1, -16, 1, 0),
                Position = UDim2.new(0, 10, 0, 0),
                BackgroundTransparency = 1,
                Text = cfg.Name or "Button",
                TextColor3 = theme.Text,
                TextSize = 11,
                Font = Enum.Font.GothamMedium,
                TextXAlignment = Enum.TextXAlignment.Left,
                Parent = Btn
            })
            Btn.MouseEnter:Connect(function()
                Tween(Btn, {BackgroundColor3 = theme.SurfaceHover}, 0.15)
            end)
            Btn.MouseLeave:Connect(function()
                Tween(Btn, {BackgroundColor3 = theme.SurfaceLight}, 0.15)
            end)
            Btn.MouseButton1Click:Connect(function()
                pcall(cfg.Callback or function() end)
            end)
            return Btn
        end

        -- ========================================
        -- Тумблер
        -- ========================================
        function tab:CreateToggle(cfg)
            cfg = cfg or {}
            local default = cfg.Default or false
            local callback = cfg.Callback or function() end
            local flag = cfg.Flag

            local Frame = Create("Frame", {
                Size = UDim2.new(1, 0, 0, 32),
                BackgroundColor3 = theme.SurfaceLight,
                BorderSizePixel = 0,
                Parent = self.Page
            })
            Create("UICorner", {CornerRadius = UDim.new(0, 6), Parent = Frame})
            Create("UIStroke", {Color = theme.Border, Thickness = 1, Parent = Frame})
            Create("TextLabel", {
                Size = UDim2.new(1, -56, 1, 0),
                Position = UDim2.new(0, 10, 0, 0),
                BackgroundTransparency = 1,
                Text = cfg.Name or "Toggle",
                TextColor3 = theme.Text,
                TextSize = 11,
                Font = Enum.Font.GothamMedium,
                TextXAlignment = Enum.TextXAlignment.Left,
                Parent = Frame
            })

            local Switch = Create("TextButton", {
                Size = UDim2.new(0, 34, 0, 18),
                Position = UDim2.new(1, -44, 0.5, -9),
                BackgroundColor3 = default and theme.Accent or theme.BorderLight,
                BorderSizePixel = 0,
                Text = "",
                AutoButtonColor = false,
                Parent = Frame
            })
            Create("UICorner", {CornerRadius = UDim.new(1, 0), Parent = Switch})
            local Knob = Create("Frame", {
                Size = UDim2.new(0, 14, 0, 14),
                Position = default and UDim2.new(0, 18, 0.5, -7) or UDim2.new(0, 2, 0.5, -7),
                BackgroundColor3 = default and theme.Background or theme.TextDim,
                BorderSizePixel = 0,
                Parent = Switch
            })
            Create("UICorner", {CornerRadius = UDim.new(1, 0), Parent = Knob})

            local state = default
            local obj = {State = state, Flag = flag}

            function obj:Set(value, fireCallback)
                state = value
                obj.State = value
                if value then
                    Tween(Switch, {BackgroundColor3 = theme.Accent}, 0.2)
                    Tween(Knob, {Position = UDim2.new(0, 18, 0.5, -7), BackgroundColor3 = theme.Background}, 0.2)
                else
                    Tween(Switch, {BackgroundColor3 = theme.BorderLight}, 0.2)
                    Tween(Knob, {Position = UDim2.new(0, 2, 0.5, -7), BackgroundColor3 = theme.TextDim}, 0.2)
                end
                if flag then Ash.Flags[flag] = value end
                if fireCallback ~= false then pcall(callback, value) end
            end

            function obj:Get() return state end

            Switch.MouseButton1Click:Connect(function()
                obj:Set(not state)
            end)

            obj:Set(default, false)
            return obj
        end

        -- ========================================
        -- Слайдер
        -- ========================================
        function tab:CreateSlider(cfg)
            cfg = cfg or {}
            local min = cfg.Min or 0
            local max = cfg.Max or 100
            local default = cfg.Default or min
            local callback = cfg.Callback or function() end
            local flag = cfg.Flag
            local suffix = cfg.Suffix or ""

            local Frame = Create("Frame", {
                Size = UDim2.new(1, 0, 0, 44),
                BackgroundColor3 = theme.SurfaceLight,
                BorderSizePixel = 0,
                Parent = self.Page
            })
            Create("UICorner", {CornerRadius = UDim.new(0, 6), Parent = Frame})
            Create("UIStroke", {Color = theme.Border, Thickness = 1, Parent = Frame})
            Create("TextLabel", {
                Size = UDim2.new(1, -70, 0, 16),
                Position = UDim2.new(0, 10, 0, 4),
                BackgroundTransparency = 1,
                Text = cfg.Name or "Slider",
                TextColor3 = theme.Text,
                TextSize = 11,
                Font = Enum.Font.GothamMedium,
                TextXAlignment = Enum.TextXAlignment.Left,
                Parent = Frame
            })
            local ValueLabel = Create("TextLabel", {
                Size = UDim2.new(0, 60, 0, 16),
                Position = UDim2.new(1, -70, 0, 4),
                BackgroundTransparency = 1,
                Text = tostring(default) .. suffix,
                TextColor3 = theme.Text,
                TextSize = 11,
                Font = Enum.Font.GothamBold,
                TextXAlignment = Enum.TextXAlignment.Right,
                Parent = Frame
            })

            local Track = Create("Frame", {
                Size = UDim2.new(1, -20, 0, 4),
                Position = UDim2.new(0, 10, 0, 28),
                BackgroundColor3 = theme.BorderLight,
                BorderSizePixel = 0,
                Parent = Frame
            })
            Create("UICorner", {CornerRadius = UDim.new(1, 0), Parent = Track})
            local Fill = Create("Frame", {
                Size = UDim2.new((default - min) / (max - min), 0, 1, 0),
                BackgroundColor3 = theme.Accent,
                BorderSizePixel = 0,
                Parent = Track
            })
            Create("UICorner", {CornerRadius = UDim.new(1, 0), Parent = Fill})
            local Thumb = Create("Frame", {
                Size = UDim2.new(0, 12, 0, 12),
                Position = UDim2.new((default - min) / (max - min), -6, 0.5, -6),
                BackgroundColor3 = theme.Text,
                BorderSizePixel = 0,
                Parent = Track
            })
            Create("UICorner", {CornerRadius = UDim.new(1, 0), Parent = Thumb})

            local value = default
            local obj = {Value = value, Flag = flag}

            function obj:Set(newVal, fireCallback)
                newVal = math.clamp(newVal, min, max)
                value = newVal
                obj.Value = newVal
                local alpha = (newVal - min) / (max - min)
                Tween(Fill, {Size = UDim2.new(alpha, 0, 1, 0)}, 0.1)
                Tween(Thumb, {Position = UDim2.new(alpha, -6, 0.5, -6)}, 0.1)
                ValueLabel.Text = tostring(math.floor(newVal * 100) / 100) .. suffix
                if flag then Ash.Flags[flag] = newVal end
                if fireCallback ~= false then pcall(callback, newVal) end
            end

            function obj:Get() return value end

            local dragging = false
            local function updateFromInput(input)
                local pos = input.Position.X - Track.AbsolutePosition.X
                local alpha = math.clamp(pos / Track.AbsoluteSize.X, 0, 1)
                local newVal = min + alpha * (max - min)
                if max - min > 20 then newVal = math.floor(newVal) end
                obj:Set(newVal)
            end

            Track.InputBegan:Connect(function(input)
                if input.UserInputType == Enum.UserInputType.MouseButton1
                   or input.UserInputType == Enum.UserInputType.Touch then
                    dragging = true
                    updateFromInput(input)
                end
            end)
            UserInputService.InputChanged:Connect(function(input)
                if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement
                                  or input.UserInputType == Enum.UserInputType.Touch) then
                    updateFromInput(input)
                end
            end)
            UserInputService.InputEnded:Connect(function(input)
                if input.UserInputType == Enum.UserInputType.MouseButton1
                   or input.UserInputType == Enum.UserInputType.Touch then
                    dragging = false
                end
            end)

            obj:Set(default, false)
            return obj
        end

        -- ========================================
        -- Текстовое поле
        -- ========================================
        function tab:CreateTextBox(cfg)
            cfg = cfg or {}
            local default = cfg.Default or ""
            local placeholder = cfg.Placeholder or "Введите текст..."
            local callback = cfg.Callback or function() end
            local flag = cfg.Flag

            local Frame = Create("Frame", {
                Size = UDim2.new(1, 0, 0, 32),
                BackgroundColor3 = theme.SurfaceLight,
                BorderSizePixel = 0,
                Parent = self.Page
            })
            Create("UICorner", {CornerRadius = UDim.new(0, 6), Parent = Frame})
            Create("UIStroke", {Color = theme.Border, Thickness = 1, Parent = Frame})
            Create("TextLabel", {
                Size = UDim2.new(0, 90, 1, 0),
                Position = UDim2.new(0, 10, 0, 0),
                BackgroundTransparency = 1,
                Text = cfg.Name or "TextBox",
                TextColor3 = theme.Text,
                TextSize = 11,
                Font = Enum.Font.GothamMedium,
                TextXAlignment = Enum.TextXAlignment.Left,
                Parent = Frame
            })

            local Input = Create("TextBox", {
                Size = UDim2.new(0, 140, 0, 22),
                Position = UDim2.new(1, -150, 0.5, -11),
                BackgroundColor3 = theme.Background,
                BorderSizePixel = 0,
                Text = default,
                PlaceholderText = placeholder,
                PlaceholderColor3 = theme.TextDisabled,
                TextColor3 = theme.Text,
                TextSize = 10,
                Font = Enum.Font.Gotham,
                ClearTextOnFocus = false,
                Parent = Frame
            })
            Create("UICorner", {CornerRadius = UDim.new(0, 4), Parent = Input})
            Create("UIStroke", {Color = theme.Border, Thickness = 1, Parent = Input})
            Create("UIPadding", {PaddingLeft = UDim.new(0, 6), PaddingRight = UDim.new(0, 6), Parent = Input})

            local obj = {Value = default, Flag = flag}

            function obj:Set(val, fireCallback)
                Input.Text = val
                obj.Value = val
                if flag then Ash.Flags[flag] = val end
                if fireCallback ~= false then pcall(callback, val) end
            end

            function obj:Get() return Input.Text end

            Input.FocusLost:Connect(function()
                obj.Value = Input.Text
                if flag then Ash.Flags[flag] = Input.Text end
                pcall(callback, Input.Text)
            end)

            return obj
        end

        -- ========================================
        -- Dropdown
        -- ========================================
        function tab:CreateDropdown(cfg)
            cfg = cfg or {}
            local options = cfg.Options or {}
            local default = cfg.Default or options[1] or "—"
            local callback = cfg.Callback or function() end
            local flag = cfg.Flag

            local Frame = Create("Frame", {
                Size = UDim2.new(1, 0, 0, 32),
                BackgroundColor3 = theme.SurfaceLight,
                BorderSizePixel = 0,
                ClipsDescendants = true,
                Parent = self.Page
            })
            Create("UICorner", {CornerRadius = UDim.new(0, 6), Parent = Frame})
            Create("UIStroke", {Color = theme.Border, Thickness = 1, Parent = Frame})

            local Header = Create("TextButton", {
                Size = UDim2.new(1, 0, 0, 32),
                BackgroundTransparency = 1,
                Text = "",
                AutoButtonColor = false,
                Parent = Frame
            })
            Create("TextLabel", {
                Size = UDim2.new(1, -100, 1, 0),
                Position = UDim2.new(0, 10, 0, 0),
                BackgroundTransparency = 1,
                Text = cfg.Name or "Dropdown",
                TextColor3 = theme.Text,
                TextSize = 11,
                Font = Enum.Font.GothamMedium,
                TextXAlignment = Enum.TextXAlignment.Left,
                Parent = Header
            })
            local ValueLabel = Create("TextLabel", {
                Size = UDim2.new(0, 60, 1, 0),
                Position = UDim2.new(1, -80, 0, 0),
                BackgroundTransparency = 1,
                Text = tostring(default),
                TextColor3 = theme.TextDim,
                TextSize = 10,
                Font = Enum.Font.GothamBold,
                TextXAlignment = Enum.TextXAlignment.Right,
                Parent = Header
            })
            Create("TextLabel", {
                Size = UDim2.new(0, 14, 1, 0),
                Position = UDim2.new(1, -20, 0, 0),
                BackgroundTransparency = 1,
                Text = "▼",
                TextColor3 = theme.TextDisabled,
                TextSize = 8,
                Font = Enum.Font.GothamBold,
                Parent = Header
            })

            local expanded = false
            local obj = {Value = default, Flag = flag}

            local function toggleExpand()
                expanded = not expanded
                if expanded then
                    Tween(Frame, {Size = UDim2.new(1, 0, 0, 32 + #options * 26)}, 0.2)
                else
                    Tween(Frame, {Size = UDim2.new(1, 0, 0, 32)}, 0.2)
                end
            end

            Header.MouseButton1Click:Connect(toggleExpand)

            for i, opt in ipairs(options) do
                local OptBtn = Create("TextButton", {
                    Size = UDim2.new(1, -12, 0, 22),
                    Position = UDim2.new(0, 6, 0, 30 + (i - 1) * 26),
                    BackgroundColor3 = theme.Surface,
                    BorderSizePixel = 0,
                    Text = tostring(opt),
                    TextColor3 = theme.Text,
                    TextSize = 10,
                    Font = Enum.Font.Gotham,
                    AutoButtonColor = false,
                    Parent = Frame
                })
                Create("UICorner", {CornerRadius = UDim.new(0, 4), Parent = OptBtn})
                OptBtn.MouseEnter:Connect(function()
                    Tween(OptBtn, {BackgroundColor3 = theme.SurfaceHover}, 0.1)
                end)
                OptBtn.MouseLeave:Connect(function()
                    Tween(OptBtn, {BackgroundColor3 = theme.Surface}, 0.1)
                end)
                OptBtn.MouseButton1Click:Connect(function()
                    obj.Value = opt
                    ValueLabel.Text = tostring(opt)
                    toggleExpand()
                    if flag then Ash.Flags[flag] = opt end
                    pcall(callback, opt)
                end)
            end

            function obj:Set(val, fireCallback)
                obj.Value = val
                ValueLabel.Text = tostring(val)
                if flag then Ash.Flags[flag] = val end
                if fireCallback ~= false then pcall(callback, val) end
            end

            return obj
        end

        -- ========================================
        -- Keybind
        -- ========================================
        function tab:CreateKeybind(cfg)
            cfg = cfg or {}
            local default = cfg.Default or Enum.KeyCode.F
            local callback = cfg.Callback or function() end
            local flag = cfg.Flag

            local Frame = Create("Frame", {
                Size = UDim2.new(1, 0, 0, 32),
                BackgroundColor3 = theme.SurfaceLight,
                BorderSizePixel = 0,
                Parent = self.Page
            })
            Create("UICorner", {CornerRadius = UDim.new(0, 6), Parent = Frame})
            Create("UIStroke", {Color = theme.Border, Thickness = 1, Parent = Frame})
            Create("TextLabel", {
                Size = UDim2.new(1, -110, 1, 0),
                Position = UDim2.new(0, 10, 0, 0),
                BackgroundTransparency = 1,
                Text = cfg.Name or "Keybind",
                TextColor3 = theme.Text,
                TextSize = 11,
                Font = Enum.Font.GothamMedium,
                TextXAlignment = Enum.TextXAlignment.Left,
                Parent = Frame
            })
            local KeyBtn = Create("TextButton", {
                Size = UDim2.new(0, 100, 0, 22),
                Position = UDim2.new(1, -110, 0.5, -11),
                BackgroundColor3 = theme.Background,
                BorderSizePixel = 0,
                Text = default.Name,
                TextColor3 = theme.Text,
                TextSize = 10,
                Font = Enum.Font.GothamBold,
                AutoButtonColor = false,
                Parent = Frame
            })
            Create("UICorner", {CornerRadius = UDim.new(0, 4), Parent = KeyBtn})
            Create("UIStroke", {Color = theme.Border, Thickness = 1, Parent = KeyBtn})

            local currentKey = default
            local listening = false
            local obj = {Key = currentKey, Flag = flag}

            function obj:Set(newKey)
                currentKey = newKey
                obj.Key = newKey
                KeyBtn.Text = newKey.Name
                if flag then Ash.Flags[flag] = newKey end
            end

            KeyBtn.MouseButton1Click:Connect(function()
                if listening then return end
                listening = true
                KeyBtn.Text = "..."
                local conn
                conn = UserInputService.InputBegan:Connect(function(input, gpe)
                    if gpe then return end
                    if input.UserInputType == Enum.UserInputType.Keyboard then
                        obj:Set(input.KeyCode)
                        listening = false
                        if conn then conn:Disconnect() end
                    end
                end)
            end)

            UserInputService.InputBegan:Connect(function(input, gpe)
                if gpe then return end
                if input.KeyCode == currentKey then
                    pcall(callback, currentKey)
                end
            end)

            return obj
        end

        return tab
    end

    -- ========================================
    -- МЕТОД: Выбрать таб
    -- ========================================
    function Window:SelectTab(tab)
        for _, t in ipairs(self.Tabs) do
            t.Page.Visible = false
            t.Indicator.Visible = false
            Tween(t.Button, {BackgroundTransparency = 1}, 0.15)
            Tween(t.Label, {TextColor3 = theme.TextDim}, 0.15)
        end
        tab.Page.Visible = true
        tab.Indicator.Visible = true
        Tween(tab.Button, {BackgroundTransparency = 0.9}, 0.15)
        Tween(tab.Label, {TextColor3 = theme.Text}, 0.15)
        self.ActiveTab = tab
    end

    -- ========================================
    -- КНОПКА "As" (когда окно скрыто)
    -- ========================================
    local AsButton = Create("TextButton", {
        Name = "AsButton",
        Size = UDim2.new(0, 44, 0, 44),
        Position = UDim2.new(0, 20, 0.5, -22),
        BackgroundColor3 = theme.Background,
        BorderSizePixel = 0,
        Text = "",
        AutoButtonColor = false,
        Visible = false,
        Parent = ScreenGui
    })
    Create("UICorner", {CornerRadius = UDim.new(0, 10), Parent = AsButton})
    Create("UIStroke", {Color = theme.Border, Thickness = 1.5, Transparency = 0.2, Parent = AsButton})
    Create("TextLabel", {
        Size = UDim2.new(1, 0, 1, 0),
        BackgroundTransparency = 1,
        Text = "As",
        TextColor3 = theme.Text,
        TextSize = 15,
        Font = Enum.Font.GothamBold,
        Parent = AsButton
    })

    AsButton.MouseEnter:Connect(function()
        Tween(AsButton, {BackgroundColor3 = theme.SurfaceLight}, 0.15)
    end)
    AsButton.MouseLeave:Connect(function()
        Tween(AsButton, {BackgroundColor3 = theme.Background}, 0.15)
    end)

    -- ========================================
    -- МЕТОД: Скрыть окно
    -- ========================================
    function Window:Hide()
        if self.Hidden then return end
        self.Hidden = true
        self.OrigPosition = Main.Position

        Tween(Main, {
            Position = UDim2.new(0.5, -size.X.Offset / 2, 1.5, 0),
            BackgroundTransparency = 1
        }, 0.3)

        task.delay(0.3, function()
            Main.Visible = false
            AsButton.Visible = true
        end)
    end

    -- ========================================
    -- МЕТОД: Показать окно
    -- ========================================
    function Window:Show()
        if not self.Hidden then return end
        self.Hidden = false

        AsButton.Visible = false
        Main.Visible = true

        Tween(Main, {
            Position = UDim2.new(0.5, -size.X.Offset / 2, 0.5, -size.Y.Offset / 2),
            BackgroundTransparency = 0
        }, 0.3)
    end

    -- ========================================
    -- МЕТОД: Переключить
    -- ========================================
    function Window:Toggle()
        if self.Hidden then
            self:Show()
        else
            self:Hide()
        end
    end

    -- Клик по AsButton
    AsButton.MouseButton1Click:Connect(function()
        Window:Show()
    end)

    -- Скрытие на K
    UserInputService.InputBegan:Connect(function(input, gpe)
        if gpe then return end
        if input.KeyCode == Enum.KeyCode.K then
            Window:Toggle()
        end
    end)

    -- Кнопка свернуть
    MinimizeBtn.MouseButton1Click:Connect(function()
        Window:Hide()
    end)

    -- Кнопка закрыть
    CloseBtn.MouseButton1Click:Connect(function()
        Tween(Main, {
            Size = UDim2.new(0, 0, 0, 0),
            Position = UDim2.new(0.5, 0, 0.5, 0),
            BackgroundTransparency = 1
        }, 0.2)
        task.delay(0.25, function()
            ScreenGui:Destroy()
        end)
    end)

    table.insert(Ash.Windows, Window)
    return Window
end

-- ============================================================
-- УВЕДОМЛЕНИЯ
-- ============================================================
function Ash:Notify(title, text, notifType, duration)
    duration = duration or 3
    local theme = Palette
    local guiParent = GetGuiParent()

    local NotifGui = Create("ScreenGui", {
        Name = "Ash_Notif_" .. tostring(math.random(100000, 999999)),
        ResetOnSpawn = false,
        IgnoreGuiInset = true,
        DisplayOrder = 1000000,
        Parent = guiParent
    })

    local Container = Create("Frame", {
        Size = UDim2.new(0, 280, 1, 0),
        Position = UDim2.new(1, -300, 0, 0),
        BackgroundTransparency = 1,
        Parent = NotifGui
    })
    Create("UIListLayout", {
        Padding = UDim.new(0, 8),
        VerticalAlignment = Enum.VerticalAlignment.Bottom,
        SortOrder = Enum.SortOrder.LayoutOrder,
        Parent = Container
    })
    Create("UIPadding", {
        PaddingBottom = UDim.new(0, 20),
        Parent = Container
    })

    local Notif = Create("Frame", {
        Size = UDim2.new(1, 0, 0, 60),
        BackgroundColor3 = theme.Surface,
        BackgroundTransparency = 1,
        BorderSizePixel = 0,
        Parent = Container
    })
    Create("UICorner", {CornerRadius = UDim.new(0, 8), Parent = Notif})
    local Stroke = Create("UIStroke", {
        Color = theme.Accent,
        Thickness = 1,
        Transparency = 1,
        Parent = Notif
    })

    local Bar = Create("Frame", {
        Size = UDim2.new(0, 3, 1, -20),
        Position = UDim2.new(0, 0, 0, 10),
        BackgroundColor3 = theme.Accent,
        BorderSizePixel = 0,
        Parent = Notif
    })
    Create("UICorner", {CornerRadius = UDim.new(0, 2), Parent = Bar})

    local TitleLabel = Create("TextLabel", {
        Size = UDim2.new(1, -20, 0, 16),
        Position = UDim2.new(0, 14, 0, 8),
        BackgroundTransparency = 1,
        Text = title,
        TextColor3 = theme.Text,
        TextSize = 12,
        Font = Enum.Font.GothamBold,
        TextXAlignment = Enum.TextXAlignment.Left,
        TextTransparency = 1,
        Parent = Notif
    })
    local TextLabel = Create("TextLabel", {
        Size = UDim2.new(1, -20, 0, 28),
        Position = UDim2.new(0, 14, 0, 26),
        BackgroundTransparency = 1,
        Text = text,
        TextColor3 = theme.TextDim,
        TextSize = 10,
        Font = Enum.Font.Gotham,
        TextXAlignment = Enum.TextXAlignment.Left,
        TextYAlignment = Enum.TextYAlignment.Top,
        TextWrapped = true,
        TextTransparency = 1,
        Parent = Notif
    })

    Tween(Notif, {BackgroundTransparency = 0.05}, 0.25)
    Tween(Stroke, {Transparency = 0.4}, 0.25)
    Tween(TitleLabel, {TextTransparency = 0}, 0.25)
    Tween(TextLabel, {TextTransparency = 0.15}, 0.25)

    task.delay(duration, function()
        Tween(Notif, {BackgroundTransparency = 1}, 0.25)
        Tween(Stroke, {Transparency = 1}, 0.25)
        Tween(TitleLabel, {TextTransparency = 1}, 0.25)
        Tween(TextLabel, {TextTransparency = 1}, 0.25)
        task.wait(0.3)
        if NotifGui then NotifGui:Destroy() end
    end)
end

-- ============================================================
-- ГЛОБАЛЬНАЯ УСТАНОВКА
-- ============================================================
_G.AshLibrary = Ash

print("[Ash UI] Библиотека загружена")

return Ash
