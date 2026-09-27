--[[
	╔══════════════════════════════════════════════════════════════════╗
	║  AshUI  ·  Lunar GUI Library  ·  для инжектора Solara            ║
	║  9300+ строк · 20 GUI-модулей · 9 палитр · сохранение конфига     ║
	╚══════════════════════════════════════════════════════════════════╝

	Запуск:
	  loadstring(game:HttpGet("URL"))()
	  или вставь файл целиком в редактор Solara и нажми Execute.

	Защита от повторного запуска: getgenv().AshUI_Loaded / getgenv().AshUI
	Все опасные вызовы обёрнуты в pcall, весь код — чистый Luau (task.*).
]]

-- ==== ЗАЩИТА ОТ ПОВТОРНОГО ЗАПУСКА ====

local RAW_ENV = (type(getgenv) == "function" and getgenv()) or _G
if type(RAW_ENV) ~= "table" then
	RAW_ENV = _G
end

-- Проверяем, не загружен ли скрипт раньше
local PREV = RAW_ENV.AshUI
if type(PREV) == "table" and PREV.Loaded == true then
	-- Мягко предупреждаем и выходим, интерфейс не дублируется
	pcall(function()
		if type(PREV.Window) == "table" and PREV.CurrentWindow ~= nil then
			PREV.CurrentWindow:Open()
		end
	end)
	return
end

-- Общие настройки скрипта (можно менять до загрузки)
local DEFAULT_OPTIONS = {
	Name		= "AshUI",
	Version		= "1.4.0",
	Author		= "Ash",
	-- Имя папки конфигов (gethui-совместимо)
	Folder		= "AshUI",
	-- Имя файла конфига
	ConfigFile	= "ashui_config.dat",
	-- Клавиша скрытия/показа окна
	ToggleKey	= Enum.KeyCode.K,
	-- Скорость анимаций (секунды)
	Duration	= 0.3,
	-- Радиус скругления (px)
	Radius		= 12,
	-- Прозрачность фона окна (0..1)
	Opacity		= 0,
	-- Размер окна по умолчанию
	WindowSize	= Vector2.new(880, 560),
	-- Максимум одновременных уведомлений
	MaxToasts	= 5,
	-- Время жизни уведомления (секунды)
	ToastLife	= 5,
	-- Snap окна к краям экрана при отпускании
	Snap		= true,
	-- Отступ от края при snap
	SnapPad		= 24,
	-- Автосохранение конфига (секунды, 0 = выключено)
	AutoSave	= 45,
	-- Отладочный вывод в консоль
	Debug		= false,
}

-- Пользовательские настройки извне
local OPTIONS = {}
for k, v in pairs(DEFAULT_OPTIONS) do
	OPTIONS[k] = v
end
if type(RAW_ENV.AshUI_Options) == "table" then
	for k, v in pairs(RAW_ENV.AshUI_Options) do
		OPTIONS[k] = v
	end
end

-- Публичная таблица библиотеки
local AshUI = {
	Options	= OPTIONS,
	Loaded	= false,
	Version	= OPTIONS.Version,
	Name	= OPTIONS.Name,
	Modules	= {},
	Windows	= {},
	CurrentWindow = nil,
}

RAW_ENV.AshUI = AshUI
RAW_ENV.AshUI_Options = OPTIONS

-- Сервисы (всегда есть на клиенте, но берём через pcall для параноиков)
local Services = {}
Services.RunService		= (function() local ok, v = pcall(function() return RunService end) return ok and v or nil end)()
Services.UserInputService	= (function() local ok, v = pcall(function() return UserInputService end) return ok and v or nil end)()
Services.Players			= (function() local ok, v = pcall(function() return Players end) return ok and v or nil end)()
Services.Lighting		= (function() local ok, v = pcall(function() return Lighting end) return ok and v or nil end)()
Services.TextService	= (function() local ok, v = pcall(function() return TextService end) return ok and v or nil end)()
Services.TweenService	= (function() local ok, v = pcall(function() return TweenService end) return ok and v or nil end)()
Services.Workspace		= (function() local ok, v = pcall(function() return workspace end) return ok and v or nil end)()
Services.Workspace		= workspace
Services.game			= (function() local ok, v = pcall(function() return game end) return ok and v or nil end)()

-- Проверка «мы вообще в Roblox-клиенте»
if type(Instance) ~= "table" and type(Instance) ~= "function" then
	-- Не критично: скрипт не сможет создать GUI, но и не упадёт
end

-- ==== ПОБИТОВАЯ СОВМЕСТИМОСТЬ ====
-- Solara/старый Luau не поддерживают операторы ~, >>, <<, &, | (вводятся с Luau 0.542).
-- Используем библиотеку bit32 (Lua 5.2+ / Roblox Luau) или bit (LuaJIT).
local bit = (function()
	if type(bit32) == "table" and type(bit32.band) == "function" then
		return bit32
	end
	local ok, lib = pcall(function() return (bit or bit32) end)
	if ok and type(lib) == "table" and type(lib.band) == "function" then
		return lib
	end
	-- Последний шанс: пробуем глобальный bit32
	if type(_G.bit32) == "table" then
		return _G.bit32
	end
	return nil
end)()

-- Хелперы-обёртки (если битовой библиотеки нет, работаем через Lua-числа)
local band, bxor, rshift
if type(bit) == "table" then
	band  = bit.band
	bxor  = bit.bxor
	rshift = bit.rshift
else
	-- Фолбэк: чистая Lua (медленнее, но работает)
	local function _band(a, b)
		a, b = a % 0x100000000, b % 0x100000000
		local r, p = 0, 1
		for _ = 1, 32 do
			if a % 2 == 1 and b % 2 == 1 then r = r + p end
			a, b, p = math.floor(a / 2), math.floor(b / 2), p * 2
		end
		return r
	end
	local function _bxor(a, b)
		a, b = a % 0x100000000, b % 0x100000000
		local r, p = 0, 1
		for _ = 1, 32 do
			if (a % 2) ~= (b % 2) then r = r + p end
			a, b, p = math.floor(a / 2), math.floor(b / 2), p * 2
		end
		return r
	end
	local function _rshift(a, n)
		a = a % 0x100000000
		for _ = 1, n do a = math.floor(a / 2) end
		return a
	end
	band  = _band
	bxor  = _bxor
	rshift = _rshift
end

-- Быстрый доступ к консоли
-- ВАЖНО: ... нельзя использовать внутри вложенной функции без `...` в её сигнатуре,
-- поэтому сначала упаковываем аргументы в таблицу, и только потом зовём pcall.
local function log(...)
	if OPTIONS.Debug ~= true then
		return
	end
	local args = table.pack(...)
	pcall(function()
		print("[AshUI]", table.unpack(args, 1, args.n))
	end)
end
AshUI.log = log

-- Хелпер: получить глобальную функцию окружения (writefile, isfile, gethui...)
function AshUI.env(name)
	local ok, v = pcall(function()
		return RAW_ENV[name]
	end)
	if not ok or v == nil then
		ok, v = pcall(function()
			return _G[name]
		end)
	end
	if not ok then
		return nil
	end
	return v
end

-- Хелпер: папка для конфига (gethui-совместимая)
function AshUI.getFolder()
	local gethui = AshUI.env("gethui")
	if type(gethui) == "function" then
		local ok, hui = pcall(gethui)
		if ok and type(hui) == "table" and hui ~= nil then
			-- Обычно gethui возвращает userdata-объект с полем Name
			local okName, nm = pcall(function()
				return hui.Name
			end)
			if okName and type(nm) == "string" and #nm > 0 then
				return OPTIONS.Folder .. "/" .. nm
			end
		end
	end
	-- Fallback: папка исполнителя
	local identifyexecutor = AshUI.env("identifyexecutor")
	if type(identifyexecutor) == "function" then
		local okName, nm = pcall(identifyexecutor)
		if okName and type(nm) == "string" and #nm > 0 then
			return OPTIONS.Folder .. "/" .. nm:gsub("[^%w%._%-]", "")
		end
	end
	return OPTIONS.Folder
end

-- Типовая обёртка: pcall + безопасное значение по умолчанию
function AshUI.safe(fn, fallback, ...)
	local results = table.pack(pcall(fn, ...))
	if results[1] == true then
		return table.unpack(results, 2, results.n)
	end
	return fallback
end

-- Хелпер: глубокое копирование через сериализацию (совместимо с Luau)
function AshUI.duplicate(tbl)
	local t = {}
	if type(tbl) ~= "table" then
		return t
	end
	for k, v in pairs(tbl) do
		if type(v) == "table" then
			t[k] = AshUI.duplicate(v)
		elseif type(v) ~= "function" then
			t[k] = v
		end
	end
	return t
end


-- ==== Utils — числа, цвета, таблицы, строки, инстансы ====

local Utils = {}
AshUI.Modules.Utils = Utils
AshUI.Utils = Utils

-- --- числа ---

function Utils.round(value, step)
	if type(value) ~= "number" then
		return 0
	end
	local s = step or 0
	if s <= 0 then
		return math.floor(value + 0.5)
	end
	local inv = 1 / s
	return math.floor(value * inv + 0.5) / inv
end

function Utils.clamp(value, minv, maxv)
	if type(value) ~= "number" then
		return minv
	end
	if value < minv then
		return minv
	elseif value > maxv then
		return maxv
	end
	return value
end

function Utils.lerp(a, b, t)
	if type(a) ~= "number" or type(b) ~= "number" then
		return a
	end
	return a + (b - a) * t
end

function Utils.inverseLerp(a, b, v)
	if a == b then
		return 0
	end
	return (v - a) / (b - a)
end

function Utils.remap(v, inMin, inMax, outMin, outMax)
	return Utils.lerp(outMin, outMax, Utils.inverseLerp(inMin, inMax, v))
end

function Utils.sign(v)
	if v > 0 then
		return 1
	elseif v < 0 then
		return -1
	end
	return 0
end

function Utils.roundTo(v, mult)
	if mult == nil or mult == 0 then
		return v
	end
	return math.round(v / mult) * mult
end

function Utils.snap(v, step)
	if step == nil or step == 0 then
		return v
	end
	return math.floor(v / step + 0.5) * step
end

-- Кадронезависимое приближение (экспоненциальное сглаживание)
function Utils.damp(current, target, lambda, dt)
	if type(dt) ~= "number" or dt <= 0 then
		dt = 1 / 60
	end
	return Utils.lerp(current, target, 1 - math.exp(-lambda * dt))
end

-- Линейное приближение с максимальной скоростью
function Utils.approach(current, target, maxDelta)
	local diff = target - current
	if math.abs(diff) <= maxDelta then
		return target
	end
	return current + Utils.sign(diff) * maxDelta
end

function Utils.pingPong(t)
	t = t % 2
	if t > 1 then
		t = 2 - t
	end
	return t
end

function Utils.wave(t, speed, phase)
	return (math.sin((t or 0) * (speed or 1) * math.pi * 2 + (phase or 0)) + 1) * 0.5
end

function Utils.triangle(t, phase)
	return Utils.pingPong((t or 0) + (phase or 0))
end

function Utils.rand(minv, maxv)
	if minv == nil then
		minv, maxv = 0, 1
	end
	if maxv == nil then
		maxv = minv
		minv = 0
	end
	return minv + (maxv - minv) * math.random()
end

function Utils.randInt(minv, maxv)
	return math.floor(Utils.rand(minv, maxv + 1))
end

function Utils.pick(list)
	if type(list) ~= "table" then
		return nil
	end
	local n = #list
	if n == 0 then
		return nil
	end
	return list[math.random(1, n)]
end

function Utils.chance(probability)
	return math.random() < (probability or 0.5)
end

function Utils.isNear(a, b, epsilon)
	return math.abs(a - b) <= (epsilon or 0.001)
end

function Utils.formatNumber(v)
	if type(v) ~= "number" then
		return tostring(v)
	end
	if v == math.floor(v) and math.abs(v) < 1e15 then
		return string.format("%d", v)
	end
	local s = string.format("%.2f", v)
	s = s:gsub("0+$", "")
	s = s:gsub("%.$", "")
	return s
end

function Utils.formatTime(seconds)
	seconds = math.max(0, math.floor(seconds or 0))
	local h = math.floor(seconds / 3600)
	local m = math.floor(seconds / 60) % 60
	local s = seconds % 60
	if h > 0 then
		return string.format("%02d:%02d:%02d", h, m, s)
	elseif m > 0 then
		return string.format("%02d:%02d", m, s)
	end
	return string.format("%02d", s)
end

function Utils.padNumber(v, digits)
	v = tostring(v or 0)
	while #v < (digits or 2) do
		v = "0" .. v
	end
	return v
end

-- --- цвета ---

function Utils.rgb(r, g, b)
	return Color3.new(Utils.clamp(r or 0, 0, 1), Utils.clamp(g or 0, 0, 1), Utils.clamp(b or 0, 0, 1))
end

function Utils.toHex(color)
	if type(color) ~= "Color3" then
		return "FFFFFF"
	end
	local r = math.round(Utils.clamp(color.R, 0, 1) * 255)
	local g = math.round(Utils.clamp(color.G, 0, 1) * 255)
	local b = math.round(Utils.clamp(color.B, 0, 1) * 255)
	return string.format("%02X%02X%02X", r, g, b)
end

function Utils.fromHex(hex)
	if type(hex) ~= "string" then
		return Color3.new(1, 1, 1)
	end
	hex = hex:gsub("#", ""):gsub("%s", "")
	if #hex == 3 then
		hex = hex:sub(1, 1) .. hex:sub(1, 1) .. hex:sub(2, 2) .. hex:sub(2, 2) .. hex:sub(3, 3) .. hex:sub(3, 3)
	end
	if #hex < 6 then
		return Color3.new(1, 1, 1)
	end
	local r = tonumber(hex:sub(1, 2), 16) or 255
	local g = tonumber(hex:sub(3, 4), 16) or 255
	local b = tonumber(hex:sub(5, 6), 16) or 255
	return Color3.fromRGB(r, g, b)
end

function Utils.mix(a, b, t)
	if type(a) ~= "Color3" or type(b) ~= "Color3" then
		return a
	end
	return Color3.new(
		Utils.lerp(a.R, b.R, t),
		Utils.lerp(a.G, b.G, t),
		Utils.lerp(a.B, b.B, t)
	)
end

function Utils.lighten(color, amount)
	return Utils.mix(color, Color3.new(1, 1, 1), Utils.clamp(amount or 0.1, 0, 1))
end

function Utils.darken(color, amount)
	return Utils.mix(color, Color3.new(0, 0, 0), Utils.clamp(amount or 0.1, 0, 1))
end

function Utils.saturate(color, amount)
	local h, s, v = Utils.rgbToHsv(color)
	return Utils.hsv(h, Utils.clamp(s + (amount or 0.2), 0, 1), v)
end

function Utils.hsv(h, s, v)
	h = (h or 0) % 360
	if h < 0 then
		h = h + 360
	end
	s = Utils.clamp(s or 0, 0, 1)
	v = Utils.clamp(v or 0, 0, 1)
	local c = v * s
	local hp = h / 60
	local x = c * (1 - math.abs((hp % 2) - 1))
	local m = v - c
	local r, g, b = 0, 0, 0
	if hp < 1 then
		r, g, b = c, x, 0
	elseif hp < 2 then
		r, g, b = x, c, 0
	elseif hp < 3 then
		r, g, b = 0, c, x
	elseif hp < 4 then
		r, g, b = 0, x, c
	elseif hp < 5 then
		r, g, b = x, 0, c
	else
		r, g, b = c, 0, x
	end
	return Color3.new(r + m, g + m, b + m)
end

function Utils.rgbToHsv(color)
	if type(color) ~= "Color3" then
		return 0, 0, 0
	end
	local r, g, b = color.R, color.G, color.B
	local mx, mn = math.max(r, g, b), math.min(r, g, b)
	local d = mx - mn
	local h = 0
	if d > 0 then
		if mx == r then
			h = ((g - b) / d) % 6
		elseif mx == g then
			h = ((b - r) / d) + 2
		else
			h = ((r - g) / d) + 4
		end
		h = h * 60
	end
	local s = 0
	if mx > 0 then
		s = d / mx
	end
	return h, s, mx
end

function Utils.luminance(color)
	if type(color) ~= "Color3" then
		return 0
	end
	return 0.299 * color.R + 0.587 * color.G + 0.114 * color.B
end

-- Контрастный цвет текста для фона
function Utils.readable(bg, dark, light)
	dark = dark or Color3.new(0.05, 0.05, 0.07)
	light = light or Color3.new(1, 1, 1)
	if Utils.luminance(bg) > 0.55 then
		return dark
	end
	return light
end

-- Смещение цвета по HSL — для генерации палитр
function Utils.shift(color, dh, ds, dl)
	local h, s, l = Utils.rgbToHsv(color)
	local maxv = 1 - l
	local minv = 0.5 - (l - 0.5)
	l = Utils.clamp(l + (dl or 0), 0.02, 0.98)
	maxv = 1 - l
	s = Utils.clamp(s + (ds or 0), 0, 1)
	h = h + (dh or 0)
	return Utils.hsv(h, s, l)
end

function Utils.rainbow(t, saturation, value)
	return Utils.hsv((t or 0) * 60 % 360, saturation or 0.7, value or 0.95)
end

function Utils.randomColor(minL, maxL)
	local l = Utils.rand(minL or 0.3, maxL or 0.7)
	local h = Utils.rand(0, 360)
	return Utils.hsv(h, Utils.rand(0.35, 0.75), l)
end

-- --- таблицы ---

function Utils.clone(tbl)
	local t = {}
	if type(tbl) ~= "table" then
		return t
	end
	for k, v in pairs(tbl) do
		if type(v) == "table" then
			t[k] = Utils.clone(v)
		elseif type(v) ~= "function" then
			t[k] = v
		end
	end
	return t
end

function Utils.merge(base, extra)
	if type(base) ~= "table" then
		base = {}
	end
	if type(extra) ~= "table" then
		return base
	end
	for k, v in pairs(extra) do
		if type(v) == "table" and type(base[k]) == "table" then
			Utils.merge(base[k], v)
		elseif type(v) ~= "function" then
			base[k] = v
		end
	end
	return base
end

function Utils.count(tbl)
	local n = 0
	if type(tbl) ~= "table" then
		return 0
	end
	for _ in pairs(tbl) do
		n = n + 1
	end
	return n
end

function Utils.listCount(list)
	return #list
end

function Utils.indexOf(list, value)
	for i = 1, #list do
		if list[i] == value then
			return i
		end
	end
	return nil
end

function Utils.contains(list, value)
	return Utils.indexOf(list, value) ~= nil
end

function Utils.insertUnique(list, value)
	if not Utils.contains(list, value) then
		table.insert(list, value)
		return true
	end
	return false
end

function Utils.removeValue(list, value)
	local i = Utils.indexOf(list, value)
	if i then
		table.remove(list, i)
		return true
	end
	return false
end

function Utils.keys(tbl)
	local out = {}
	if type(tbl) ~= "table" then
		return out
	end
	for k in pairs(tbl) do
		table.insert(out, k)
	end
	table.sort(out, function(a, b)
		return tostring(a) < tostring(b)
	end)
	return out
end

function Utils.shuffle(list)
	for i = #list, 2, -1 do
		local j = math.random(1, i)
		list[i], list[j] = list[j], list[i]
	end
	return list
end

function Utils.empty(tbl)
	if type(tbl) ~= "table" then
		return
	end
	for k in pairs(tbl) do
		tbl[k] = nil
	end
end

-- Сортировка по весу (для аккордеонов и т.п.)
function Utils.sortByWeight(list)
	table.sort(list, function(a, b)
		return (a.Weight or 0) < (b.Weight or 0)
	end)
	return list
end

-- Поиск с поддержкой "вложенных" таблиц вида {Label=, Value=}
function Utils.optionLabel(opt, index)
	if type(opt) == "table" then
		return opt.Label or opt.Name or opt.Value or ("Item " .. tostring(index))
	end
	return tostring(opt)
end

function Utils.optionValue(opt, index)
	if type(opt) == "table" then
		return opt.Value ~= nil and opt.Value or opt.Label or opt.Name
	end
	return opt
end

-- Безопасная индексация (Lua: 1..n, без nil-дыр)
function Utils.at(list, i)
	if type(list) ~= "table" then
		return nil
	end
	return list[i]
end

-- --- строки ---

function Utils.truncate(text, maxLen)
	text = tostring(text or "")
	maxLen = maxLen or 28
	if #text <= maxLen then
		return text
	end
	return text:sub(1, math.max(1, maxLen - 1)) .. "…"
end

function Utils.titleCase(text)
	return (tostring(text or ""):gsub("(%a)([%w']*)", function(a, b)
		return a:upper() .. b:lower()
	end))
end

function Utils.split(text, sep)
	local out = {}
	if type(text) ~= "string" then
		return out
	end
	sep = sep or ","
	for part in text:gmatch("([^" .. sep .. "]+)") do
		table.insert(out, part)
	end
	return out
end

-- --- инстансы ---

-- Создание инстанса с безопасной установкой свойств
function Utils.new(className, props, parent)
	local ok, obj = pcall(function()
		local o = Instance.new(className)
		if type(props) == "table" then
			for k, v in pairs(props) do
				pcall(function()
					o[k] = v
				end)
			end
		end
		if parent ~= nil then
			o.Parent = parent
		end
		return o
	end)
	if not ok or obj == nil then
		return nil
	end
	return obj
end

function Utils.set(obj, props)
	if type(obj) ~= "table" and type(obj) ~= "Instance" then
		return false
	end
	if type(props) ~= "table" then
		return false
	end
	for k, v in pairs(props) do
		pcall(function()
			obj[k] = v
		end)
	end
	return true
end

function Utils.get(obj, prop)
	local ok, v = pcall(function()
		return obj[prop]
	end)
	if not ok then
		return nil
	end
	return v
end

function Utils.find(root, className, recursive)
	if root == nil then
		return nil
	end
	if recursive then
		return root:FindFirstChild(className, true)
	end
	return root:FindFirstChild(className)
end

function Utils.waitFor(root, name, timeout)
	if root == nil then
		return nil
	end
	local deadline = (os.clock() + (timeout or 5))
	while os.clock() < deadline do
		local found = pcall(function()
			return root:WaitForChild(name, 0.5)
		end)
		if found and found ~= nil then
			return found
		end
	end
	return nil
end

function Utils.clear(obj)
	if obj == nil then
		return
	end
	pcall(function()
		for _, child in ipairs(obj:GetChildren()) do
			child:Destroy()
		end
	end)
end

function Utils.destroy(obj)
	pcall(function()
		obj:Destroy()
	end)
end

-- Шрифт: пробуем Font.new, при неудаче — Enum.Font
local FONT_ENUM = {
	Gotham		= Enum.Font.Gotham,
	GothamBold	= Enum.Font.GothamBold,
	GothamMedium= Enum.Font.GothamMedium,
	GothamSemibold = Enum.Font.GothamSemibold,
	SourceSans = Enum.Font.SourceSans,
	Code		= Enum.Font.Code,
	Legacy		= Enum.Font.Legacy,
}

function Utils.fontEnum(name)
	return FONT_ENUM[name] or Enum.Font.Gotham
end

-- Создаёт объект Font (или возвращает Enum-фолбэк)
function Utils.makeFont(family, weightName, styleName)
	local ok, font = pcall(function()
		local weight = Enum.FontWeight[weightName or "Medium"]
		local style = Enum.FontStyle[styleName or "Regular"]
		return Font.new(family or "Gotham", weight or Enum.FontWeight.Medium, style or Enum.FontStyle.Regular)
	end)
	if ok and font ~= nil then
		return font
	end
	return Utils.fontEnum(family)
end

-- Установка шрифта с fallback
function Utils.setFont(guiObject, family, weightName, styleName)
	if guiObject == nil then
		return
	end
	local font = Utils.makeFont(family, weightName, styleName)
	pcall(function()
		guiObject.Font = font
	end)
	if guiObject.Font == nil or tostring(guiObject.Font) == "nil" then
		pcall(function()
			guiObject.Font = Utils.fontEnum(family)
		end)
	end
end

-- Скругление
function Utils.corner(parent, radius)
	return Utils.new("UICorner", {
		CornerRadius = UDim.new(0, radius or OPTIONS.Radius),
		Parent = parent,
	})
end

-- Обводка
function Utils.stroke(parent, color, transparency, thickness)
	return Utils.new("UIStroke", {
		Color = color or Color3.new(1, 1, 1),
		Transparency = transparency or 0.8,
		Thickness = thickness or 1,
		ApplyStrokeMode = Enum.ApplyStrokeMode.Border,
		Parent = parent,
	})
end

-- Градиент
function Utils.gradient(parent, c0, c1, rotation)
	return Utils.new("UIGradient", {
		Color = ColorSequence.new(c0 or Color3.new(1, 1, 1), c1 or Color3.new(0, 0, 0)),
		Rotation = rotation or 0,
		Parent = parent,
	})
end

function Utils.padding(parent, top, right, bottom, left)
	return Utils.new("UIPadding", {
		PaddingTop = UDim.new(0, top or 0),
		PaddingRight = UDim.new(0, right or 0),
		PaddingBottom = UDim.new(0, bottom or 0),
		PaddingLeft = UDim.new(0, left or 0),
		Parent = parent,
	})
end

function Utils.listLayout(parent, direction, gap, hAlign, vAlign, sortOrder)
	return Utils.new("UIListLayout", {
		FillDirection = direction or Enum.FillDirection.Vertical,
		Padding = UDim.new(0, gap or 8),
		HorizontalAlignment = hAlign or Enum.HorizontalAlignment.Left,
		VerticalAlignment = vAlign or Enum.VerticalAlignment.Top,
		SortOrder = sortOrder or Enum.SortOrder.LayoutOrder,
		Parent = parent,
	})
end

function Utils.sizeConstraint(parent, minH, maxH, minW, maxW)
	return Utils.new("UISizeConstraint", {
		MinSize = Vector2.new(minW or 0, minH or 0),
		MaxSize = Vector2.new(maxW or math.huge, maxH or math.huge),
		Parent = parent,
	})
end

function Utils.pixels(v)
	return UDim.new(0, v or 0)
end

function Utils.scale(v)
	return UDim.new(v or 0, 0)
end

-- Размер окна с учётом разрешения
function Utils.screenSize()
	local cam = Services.Workspace and Services.Workspace.CurrentCamera
	if cam then
		local v = cam.ViewportSize
		if type(v) == "Vector2" and v.X > 0 then
			return v
		end
	end
	return Vector2.new(1920, 1080)
end

-- Габариты GUI: абсолютный размер фрейма в пикселях
function Utils.absoluteSize(gui)
	if gui == nil then
		return Vector2.zero
	end
	local ok, v = pcall(function()
		return gui.AbsoluteSize
	end)
	if not ok or type(v) ~= "Vector2" then
		return Vector2.zero
	end
	return v
end

function Utils.absolutePos(gui)
	if gui == nil then
		return Vector2.zero
	end
	local ok, v = pcall(function()
		return gui.AbsolutePosition
	end)
	if not ok or type(v) ~= "Vector2" then
		return Vector2.zero
	end
	return v
end

-- Позиция курсора без GuiInset
function Utils.mousePos()
	local uis = Services.UserInputService
	if uis == nil then
		return Vector2.zero
	end
	local ok, p = pcall(function()
		return uis:GetMouseLocation()
	end)
	if not ok or type(p) ~= "Vector2" then
		return Vector2.zero
	end
	local inset = Vector2.zero
	if type(guiInset) == "table" and type(guiInset) == "UDim2" then
		inset = Vector2.new(guiInset.X.Offset, guiInset.Y.Offset)
	end
	return Vector2.new(p.X - inset.X, p.Y - inset.Y)
end

-- Прямоугольник содержит точку
function Utils.rectContains(pos, size, point)
	return point.X >= pos.X and point.X <= pos.X + size.X
		and point.Y >= pos.Y and point.Y <= pos.Y + size.Y
end

-- Плавное масштабирование окна под маленькие разрешения
function Utils.clampWindowSize(size, minSize)
	local screen = Utils.screenSize()
	local maxW = math.min(size.X, screen.X - 40)
	local maxH = math.min(size.Y, screen.Y - 40)
	maxW = math.max(maxW, (minSize or Vector2.new(420, 320)).X)
	maxH = math.max(maxH, (minSize or Vector2.new(420, 320)).Y)
	return Vector2.new(maxW, maxH)
end

-- --- сигналы (простое событие) ---

function Utils.signal()
	local handlers = {}
	local signal = {}

	function signal.Connect(fn)
		if type(fn) ~= "function" then
			return function() end
		end
		local id = tostring(fn)
		handlers[id] = fn
		return function()
			handlers[id] = nil
		end
	end

	function signal.Fire(...)
		local list = {}
		for _, fn in pairs(handlers) do
			table.insert(list, fn)
		end
		for _, fn in ipairs(list) do
			pcall(fn, ...)
		end
	end

	function signal.Destroy()
		Utils.empty(handlers)
	end

	return signal
end

-- Счётчик LayoutOrder (Roblox не разрешает хранить поля в инстансах, поэтому таблица)
local OrderCounters = setmetatable({}, {__mode = "k"})

-- Возвращает следующий LayoutOrder для контейнера (гарантирует стабильный порядок)
function Utils.nextOrder(parent)
	if parent == nil then
		return 0
	end
	local n = (OrderCounters[parent] or 0) + 1
	OrderCounters[parent] = n
	return n
end

-- Хелпер: отложенный вызов через task (без wait в рендер-циклах)
function Utils.defer(fn, ...)
	local args = table.pack(...)
	pcall(task.spawn, function()
		pcall(fn, table.unpack(args, 1, args.n))
	end)
end

function Utils.delay(seconds, fn, ...)
	local args = table.pack(...)
	pcall(task.delay, seconds or 0, function()
		pcall(fn, table.unpack(args, 1, args.n))
	end)
end

-- ==== CRC32 (чистая реализация, без bit32 — работает в любом Luau) ====

local CRC_TABLE = nil

local function buildCrcTable()
	local t = {}
	for i = 0, 255 do
		local c = i
		for _ = 1, 8 do
			if c % 2 == 1 then
				c = bxor(0xEDB88320, rshift(c, 1))
			else
				c = rshift(c, 1)
			end
			c = band(c, 0xFFFFFFFF)
		end
		t[i] = c
	end
	return t
end

-- CRC32 (poly 0xEDB88320) — используется для проверки целостности конфига
function Utils.crc32(str)
	CRC_TABLE = CRC_TABLE or buildCrcTable()
	str = tostring(str or "")
	local crc = 0xFFFFFFFF
	for i = 1, #str do
		local byte = string.byte(str, i) or 0
		crc = bxor(rshift(crc, 8), CRC_TABLE[band(crc, 0xFF) % 256])
		crc = band(crc, 0xFFFFFFFF)
	end
	crc = band(bxor(crc, 0xFFFFFFFF), 0xFFFFFFFF)
	return string.format("%08X", crc)
end

-- Быстрый djb2 — запасной хеш, если CRC-таблица недоступна
function Utils.djb2(str)
	str = tostring(str or "")
	local hash = 5381
	for i = 1, #str do
		hash = band((hash * 33) + string.byte(str, i), 0xFFFFFFFF)
	end
	return string.format("%08X", hash)
end
-- ==== КОНСТАНТЫ И ОБЩИЕ ССЫЛКИ ====

-- Текущая палитра (заполняется модулем Theme)
local Palette = {}
local PaletteFrom = {}
local PaletteTo = {}
local TransitionT = 1
local TransitionDuration = 0.4

-- Зарегистрированные привязки: {Key, Instance, Property}
local Bindings = {}

-- Затемнение фона
local FADE = 0


-- ==== Theme — палитры и плавные переходы ====

local Theme = {}
AshUI.Modules.Theme = Theme
AshUI.Theme = Theme

-- Список палитр (id -> палитра)
local Palettes = {}

-- Значения по умолчанию для любой палитры
local PALETTE_DEFAULTS = {
	Background		= Color3.fromRGB(14, 16, 22),	-- фон окна
	Surface			= Color3.fromRGB(26, 28, 36),	-- поверхности (сайдбар, панели)
	SurfaceAlt		= Color3.fromRGB(34, 37, 47),	-- приподнятые поверхности
	SurfaceDeep		= Color3.fromRGB(20, 22, 29),	-- поля ввода, «утопленные» зоны
	Card			= Color3.fromRGB(30, 32, 41),	-- карточки элементов
	CardHover		= Color3.fromRGB(40, 43, 55),	-- карточка под курсором
	Accent			= Color3.fromRGB(200, 205, 220),	-- основной акцент
	AccentText		= Color3.fromRGB(14, 16, 22),	-- текст на акценте
	Text			= Color3.fromRGB(226, 230, 240),	-- основной текст
	TextDim			= Color3.fromRGB(140, 146, 163),	-- приглушённый текст
	TextFaint		= Color3.fromRGB(92, 97, 112),	-- еле заметный текст
	Border			= Color3.fromRGB(48, 52, 66),	-- обводки
	BorderSoft		= Color3.fromRGB(36, 39, 50),	-- мягкие разделители
	Success			= Color3.fromRGB(96, 201, 145),
	Error			= Color3.fromRGB(224, 96, 108),
	Warning			= Color3.fromRGB(232, 190, 88),
	Info			= Color3.fromRGB(104, 162, 232),
	Track			= Color3.fromRGB(44, 47, 59),	-- дорожки слайдера/прогресса
	Overlay			= Color3.fromRGB(6, 7, 11),	-- затемнение за окном
	Shadow			= Color3.fromRGB(0, 0, 0),	-- тень
	Star			= Color3.fromRGB(226, 230, 240),	-- звёзды фона
	StarDim			= Color3.fromRGB(120, 128, 150),
	Moon			= Color3.fromRGB(240, 242, 248),
	MoonShadow		= Color3.fromRGB(150, 155, 172),
	Aurora			= Color3.fromRGB(120, 132, 190),
	AuroraAlt		= Color3.fromRGB(150, 120, 200),
}

-- Регистрация палитры
function Theme.Register(id, name, overrides)
	if type(id) ~= "string" or #id == 0 then
		return false
	end
	local palette = {}
	-- сначала дефолты, потом переопределения темы
	for k, v in pairs(PALETTE_DEFAULTS) do
		palette[k] = v
	end
	if type(overrides) == "table" then
	for k, v in pairs(overrides) do
		if type(v) == "Color3" then
			palette[k] = v
		end
	end
end
	Palettes[id] = {
		Id		= id,
		Name	= name or Utils.titleCase(id),
		Colors	= palette,
		BuiltIn	= false,
	}
	return true
end

-- ==== ВСТРОЕННЫЕ ПАЛИТРЫ ====

-- Lunar — холодный лунный свет (по умолчанию)
Theme.Register("Lunar", "Lunar", {
	Background		= Color3.fromRGB(14, 16, 22),
	Surface			= Color3.fromRGB(22, 24, 32),
	SurfaceAlt		= Color3.fromRGB(30, 33, 43),
	SurfaceDeep		= Color3.fromRGB(18, 20, 27),
	Card			= Color3.fromRGB(27, 30, 39),
	CardHover		= Color3.fromRGB(37, 41, 53),
	Accent			= Color3.fromRGB(200, 205, 220),
	AccentText		= Color3.fromRGB(14, 16, 22),
	Text			= Color3.fromRGB(226, 230, 240),
	TextDim			= Color3.fromRGB(140, 146, 163),
	TextFaint		= Color3.fromRGB(92, 97, 112),
	Border			= Color3.fromRGB(50, 54, 68),
	BorderSoft		= Color3.fromRGB(36, 39, 50),
	Track			= Color3.fromRGB(44, 48, 60),
	Aurora			= Color3.fromRGB(110, 124, 190),
	AuroraAlt		= Color3.fromRGB(160, 132, 205),
	Star			= Color3.fromRGB(232, 236, 246),
	StarDim			= Color3.fromRGB(120, 128, 150),
	Moon			= Color3.fromRGB(242, 244, 250),
	MoonShadow		= Color3.fromRGB(148, 154, 172),
})

-- Blood — тёмный багровый
Theme.Register("Blood", "Blood", {
	Background		= Color3.fromRGB(20, 11, 13),
	Surface			= Color3.fromRGB(31, 17, 20),
	SurfaceAlt		= Color3.fromRGB(43, 22, 26),
	SurfaceDeep		= Color3.fromRGB(25, 13, 16),
	Card			= Color3.fromRGB(40, 20, 24),
	CardHover		= Color3.fromRGB(56, 27, 32),
	Accent			= Color3.fromRGB(226, 92, 104),
	AccentText		= Color3.fromRGB(20, 8, 10),
	Text			= Color3.fromRGB(240, 224, 224),
	TextDim			= Color3.fromRGB(172, 132, 132),
	TextFaint		= Color3.fromRGB(122, 88, 90),
	Border			= Color3.fromRGB(72, 33, 40),
	BorderSoft		= Color3.fromRGB(52, 25, 30),
	Track			= Color3.fromRGB(62, 28, 34),
	Success			= Color3.fromRGB(120, 200, 140),
	Error			= Color3.fromRGB(240, 78, 90),
	Warning			= Color3.fromRGB(238, 180, 80),
	Info			= Color3.fromRGB(130, 150, 230),
	Aurora			= Color3.fromRGB(190, 60, 80),
	AuroraAlt		= Color3.fromRGB(120, 40, 90),
	Star			= Color3.fromRGB(250, 210, 210),
	StarDim			= Color3.fromRGB(170, 100, 110),
	Moon			= Color3.fromRGB(250, 220, 220),
	MoonShadow		= Color3.fromRGB(160, 110, 116),
})

-- Ocean — глубокая синь
Theme.Register("Ocean", "Ocean", {
	Background		= Color3.fromRGB(10, 16, 24),
	Surface			= Color3.fromRGB(16, 25, 36),
	SurfaceAlt		= Color3.fromRGB(22, 34, 48),
	SurfaceDeep		= Color3.fromRGB(13, 21, 30),
	Card			= Color3.fromRGB(21, 32, 45),
	CardHover		= Color3.fromRGB(29, 44, 61),
	Accent			= Color3.fromRGB(96, 190, 220),
	AccentText		= Color3.fromRGB(8, 18, 26),
	Text			= Color3.fromRGB(222, 238, 246),
	TextDim			= Color3.fromRGB(126, 160, 178),
	TextFaint		= Color3.fromRGB(84, 112, 128),
	Border			= Color3.fromRGB(36, 62, 82),
	BorderSoft		= Color3.fromRGB(26, 45, 60),
	Track			= Color3.fromRGB(30, 50, 68),
	Warning			= Color3.fromRGB(240, 198, 96),
	Aurora			= Color3.fromRGB(70, 150, 210),
	AuroraAlt		= Color3.fromRGB(60, 200, 190),
	Star			= Color3.fromRGB(214, 240, 250),
	StarDim			= Color3.fromRGB(96, 150, 175),
	Moon			= Color3.fromRGB(230, 248, 255),
	MoonShadow		= Color3.fromRGB(122, 168, 186),
})

-- Forest — приглушённая зелень
Theme.Register("Forest", "Forest", {
	Background		= Color3.fromRGB(12, 18, 15),
	Surface			= Color3.fromRGB(19, 28, 23),
	SurfaceAlt		= Color3.fromRGB(26, 38, 31),
	SurfaceDeep		= Color3.fromRGB(15, 23, 19),
	Card			= Color3.fromRGB(24, 35, 29),
	CardHover		= Color3.fromRGB(33, 48, 39),
	Accent			= Color3.fromRGB(126, 200, 140),
	AccentText		= Color3.fromRGB(10, 20, 14),
	Text			= Color3.fromRGB(222, 238, 224),
	TextDim			= Color3.fromRGB(132, 162, 138),
	TextFaint		= Color3.fromRGB(90, 114, 96),
	Border			= Color3.fromRGB(44, 66, 52),
	BorderSoft		= Color3.fromRGB(32, 48, 38),
	Track			= Color3.fromRGB(38, 56, 44),
	Warning			= Color3.fromRGB(232, 194, 96),
	Aurora			= Color3.fromRGB(90, 180, 120),
	AuroraAlt		= Color3.fromRGB(180, 190, 90),
	Star			= Color3.fromRGB(226, 244, 228),
	StarDim			= Color3.fromRGB(110, 150, 122),
	Moon			= Color3.fromRGB(236, 248, 234),
	MoonShadow		= Color3.fromRGB(128, 158, 134),
})

-- Abyss — почти чёрный с фиолетовым подтоном
Theme.Register("Abyss", "Abyss", {
	Background		= Color3.fromRGB(10, 8, 16),
	Surface			= Color3.fromRGB(18, 15, 27),
	SurfaceAlt		= Color3.fromRGB(26, 21, 38),
	SurfaceDeep		= Color3.fromRGB(14, 11, 21),
	Card			= Color3.fromRGB(24, 19, 35),
	CardHover		= Color3.fromRGB(34, 27, 49),
	Accent			= Color3.fromRGB(170, 132, 240),
	AccentText		= Color3.fromRGB(12, 8, 20),
	Text			= Color3.fromRGB(232, 226, 246),
	TextDim			= Color3.fromRGB(150, 140, 180),
	TextFaint		= Color3.fromRGB(104, 95, 130),
	Border			= Color3.fromRGB(50, 40, 74),
	BorderSoft		= Color3.fromRGB(36, 29, 54),
	Track			= Color3.fromRGB(42, 34, 62),
	Aurora			= Color3.fromRGB(130, 90, 220),
	AuroraAlt		= Color3.fromRGB(200, 90, 200),
	Star			= Color3.fromRGB(238, 230, 255),
	StarDim			= Color3.fromRGB(140, 126, 180),
	Moon			= Color3.fromRGB(244, 238, 255),
	MoonShadow		= Color3.fromRGB(150, 140, 176),
})

-- Neon — кислотный киберпанк
Theme.Register("Neon", "Neon", {
	Background		= Color3.fromRGB(10, 10, 14),
	Surface			= Color3.fromRGB(18, 18, 24),
	SurfaceAlt		= Color3.fromRGB(26, 26, 34),
	SurfaceDeep		= Color3.fromRGB(14, 14, 19),
	Card			= Color3.fromRGB(24, 24, 32),
	CardHover		= Color3.fromRGB(34, 34, 46),
	Accent			= Color3.fromRGB(0, 240, 200),
	AccentText		= Color3.fromRGB(4, 20, 18),
	Text			= Color3.fromRGB(236, 240, 246),
	TextDim			= Color3.fromRGB(138, 146, 160),
	TextFaint		= Color3.fromRGB(92, 98, 112),
	Border			= Color3.fromRGB(46, 50, 60),
	BorderSoft		= Color3.fromRGB(32, 35, 44),
	Track			= Color3.fromRGB(40, 44, 54),
	Info			= Color3.fromRGB(60, 180, 255),
	Warning			= Color3.fromRGB(255, 210, 70),
	Aurora			= Color3.fromRGB(0, 220, 190),
	AuroraAlt		= Color3.fromRGB(200, 60, 255),
	Star			= Color3.fromRGB(200, 255, 248),
	StarDim			= Color3.fromRGB(80, 200, 190),
	Moon			= Color3.fromRGB(220, 255, 250),
	MoonShadow		= Color3.fromRGB(90, 190, 185),
})

-- Midnight — тёмно-синий с приглушённым акцентом
Theme.Register("Midnight", "Midnight", {
	Background		= Color3.fromRGB(11, 14, 22),
	Surface			= Color3.fromRGB(18, 22, 33),
	SurfaceAlt		= Color3.fromRGB(25, 30, 44),
	SurfaceDeep		= Color3.fromRGB(15, 18, 28),
	Card			= Color3.fromRGB(23, 28, 41),
	CardHover		= Color3.fromRGB(32, 39, 56),
	Accent			= Color3.fromRGB(140, 160, 200),
	AccentText		= Color3.fromRGB(10, 12, 20),
	Text			= Color3.fromRGB(216, 224, 240),
	TextDim			= Color3.fromRGB(126, 138, 162),
	TextFaint		= Color3.fromRGB(84, 94, 116),
	Border			= Color3.fromRGB(42, 50, 68),
	BorderSoft		= Color3.fromRGB(30, 36, 50),
	Track			= Color3.fromRGB(36, 43, 58),
	Aurora			= Color3.fromRGB(90, 120, 190),
	AuroraAlt		= Color3.fromRGB(120, 100, 200),
	Star			= Color3.fromRGB(220, 230, 250),
	StarDim			= Color3.fromRGB(104, 118, 150),
	Moon			= Color3.fromRGB(230, 238, 255),
	MoonShadow		= Color3.fromRGB(126, 138, 168),
})

-- Sunset — тёплый закат
Theme.Register("Sunset", "Sunset", {
	Background		= Color3.fromRGB(20, 14, 18),
	Surface			= Color3.fromRGB(32, 22, 28),
	SurfaceAlt		= Color3.fromRGB(45, 30, 37),
	SurfaceDeep		= Color3.fromRGB(25, 17, 22),
	Card			= Color3.fromRGB(42, 28, 35),
	CardHover		= Color3.fromRGB(58, 38, 47),
	Accent			= Color3.fromRGB(246, 146, 92),
	AccentText		= Color3.fromRGB(28, 14, 10),
	Text			= Color3.fromRGB(246, 232, 226),
	TextDim			= Color3.fromRGB(178, 148, 142),
	TextFaint		= Color3.fromRGB(128, 102, 100),
	Border			= Color3.fromRGB(74, 48, 56),
	BorderSoft		= Color3.fromRGB(54, 36, 42),
	Track			= Color3.fromRGB(64, 42, 48),
	Info			= Color3.fromRGB(120, 170, 240),
	Aurora			= Color3.fromRGB(240, 120, 90),
	AuroraAlt		= Color3.fromRGB(240, 190, 90),
	Star			= Color3.fromRGB(255, 232, 208),
	StarDim			= Color3.fromRGB(190, 148, 120),
	Moon			= Color3.fromRGB(255, 226, 198),
	MoonShadow		= Color3.fromRGB(184, 142, 120),
})

-- Mono — полностью монохромная
Theme.Register("Mono", "Mono", {
	Background		= Color3.fromRGB(16, 16, 16),
	Surface			= Color3.fromRGB(24, 24, 24),
	SurfaceAlt		= Color3.fromRGB(32, 32, 32),
	SurfaceDeep		= Color3.fromRGB(20, 20, 20),
	Card			= Color3.fromRGB(30, 30, 30),
	CardHover		= Color3.fromRGB(42, 42, 42),
	Accent			= Color3.fromRGB(235, 235, 235),
	AccentText		= Color3.fromRGB(16, 16, 16),
	Text			= Color3.fromRGB(240, 240, 240),
	TextDim			= Color3.fromRGB(150, 150, 150),
	TextFaint		= Color3.fromRGB(100, 100, 100),
	Border			= Color3.fromRGB(56, 56, 56),
	BorderSoft		= Color3.fromRGB(40, 40, 40),
	Track			= Color3.fromRGB(46, 46, 46),
	Success			= Color3.fromRGB(190, 190, 190),
	Error			= Color3.fromRGB(140, 140, 140),
	Warning			= Color3.fromRGB(170, 170, 170),
	Info			= Color3.fromRGB(200, 200, 200),
	Aurora			= Color3.fromRGB(90, 90, 90),
	AuroraAlt		= Color3.fromRGB(70, 70, 70),
	Star			= Color3.fromRGB(240, 240, 240),
	StarDim			= Color3.fromRGB(120, 120, 120),
	Moon			= Color3.fromRGB(250, 250, 250),
	MoonShadow		= Color3.fromRGB(140, 140, 140),
})

-- Помечаем встроенные палитры
for _, id in ipairs({"Lunar", "Blood", "Ocean", "Forest", "Abyss", "Neon", "Midnight", "Sunset", "Mono"}) do
	if Palettes[id] then
		Palettes[id].BuiltIn = true
	end
end

-- Список id палитр
function Theme.List()
	local out = {}
	for id, p in pairs(Palettes) do
		table.insert(out, id)
	end
	table.sort(out)
	return out
end

function Theme.Get(id)
	return Palettes[id]
end

function Theme.Exists(id)
	return type(id) == "string" and Palettes[id] ~= nil
end

-- Текущая палитра
function Theme.Current()
	return Palettes[Palette.__id or "Lunar"] or Palettes.Lunar
end

function Theme.CurrentId()
	return Palette.__id or "Lunar"
end

-- Мгновенное применение палитры (без анимации)
function Theme.ApplyInstant(id)
	local p = Palettes[id]
	if p == nil then
		return false
	end
	for k, v in pairs(p.Colors) do
		Palette[k] = v
		PaletteFrom[k] = v
		PaletteTo[k] = v
	end
	Palette.__id = p.Id
	Palette.__name = p.Name
	TransitionT = 1
	-- обновляем все привязки
	for _, b in ipairs(Bindings) do
		local col = Palette[b.Key]
		if col ~= nil and b.Instance ~= nil then
			pcall(function()
				b.Instance[b.Property] = col
			end)
		end
	end
	return true
end

-- Плавный переход к палитре
function Theme.Set(id, instant)
	if not Theme.Exists(id) then
		return false
	end
	if instant == true then
		return Theme.ApplyInstant(id)
	end
	PaletteFrom = Utils.clone(Palette)
	local p = Palettes[id]
	PaletteTo = p.Colors
	Palette.__id = p.Id
	Palette.__name = p.Name
	TransitionT = 0
	return true
end

-- Перезафиксировать «стартовую» палитру (после ручной правки цветов)
function Theme.Recapture()
	PaletteFrom = Utils.clone(Palette)
end

-- Список всех привязок (для отладки)
function Theme.BindingCount()
	return #Bindings
end

-- Регистрация объекта, который должен менять цвет при смене темы
-- key: имя цвета в палитре ("Text", "Card", ...)
-- obj: GuiObject/UIStroke/UIGradient
-- prop: "BackgroundColor3" / "Color" / "TextColor3"
function Theme.Bind(key, obj, prop)
	if type(key) ~= "string" or obj == nil then
		return false
	end
	prop = prop or "BackgroundColor3"
	for i = 1, #Bindings do
		if Bindings[i].Instance == obj and Bindings[i].Property == prop then
			Bindings[i].Key = key
			return true
		end
	end
	table.insert(Bindings, {
		Key = key,
		Instance = obj,
		Property = prop,
	})
	local col = Palette[key]
	if col ~= nil then
		pcall(function()
			obj[prop] = col
		end)
	end
	return true
end

function Theme.Unbind(obj)
	for i = #Bindings, 1, -1 do
		if Bindings[i].Instance == obj then
			table.remove(Bindings, i)
		end
	end
end

-- Получить цвет по ключу (текущий интерполированный)
function Theme.Get_(key)
	if key == nil then
		return Color3.new(1, 1, 1)
	end
	local c = Palette[key]
	if c == nil then
		return Color3.new(1, 1, 1)
	end
	return c
end
Theme.C = Theme.Get_

-- Экспорт палитры в строку (для Config)
function Theme.Export(id)
	local p = Palettes[id] or Theme.Current()
	if p == nil then
		return {}
	end
	local out = {}
	for k, v in pairs(p.Colors) do
		out[k] = Utils.toHex(v)
	end
	return out
end

-- ==== ОБНОВЛЕНИЕ ПЕРЕХОДА (вызывается из Motion) ====

function Theme.Update(dt)
	if TransitionT >= 1 then
		return false
	end
	local duration = Utils.clamp(TransitionDuration, 0.05, 5)
	local rate = 1 / duration
	TransitionT = Utils.clamp(TransitionT + dt * rate, 0, 1)
	local e = Utils.clamp(TransitionT, 0, 1)
	for k, target in pairs(PaletteTo) do
		local from = PaletteFrom[k]
		if type(target) == "Color3" then
			if type(from) == "Color3" then
				Palette[k] = Utils.mix(from, target, e)
			else
				Palette[k] = target
			end
		end
	end
	for _, b in ipairs(Bindings) do
		local col = Palette[b.Key]
		if col ~= nil and b.Instance ~= nil then
			pcall(function()
				b.Instance[b.Property] = col
			end)
		end
	end
	if TransitionT >= 1 then
		PaletteFrom = Utils.clone(PaletteTo)
	end
	return true
end

function Theme.TransitionProgress()
	return TransitionT
end

-- Принудительно завершить переход (например, при скрытии окна)
function Theme.FinishTransition()
	if TransitionT >= 1 then
		return
	end
	for k, target in pairs(PaletteTo) do
		if type(target) == "Color3" then
			Palette[k] = target
		end
	end
	TransitionT = 1
	for _, b in ipairs(Bindings) do
		local col = Palette[b.Key]
		if col ~= nil and b.Instance ~= nil then
			pcall(function()
				b.Instance[b.Property] = col
			end)
		end
	end
end

-- Установка длительности перехода (в секундах)
function Theme.SetTransitionSpeed(seconds)
	TransitionDuration = Utils.clamp(seconds or 0.4, 0.05, 5)
end

-- Начальная палитра — Lunar
Theme.ApplyInstant("Lunar")
-- ==== Motion — анимации, твины, пружины, drag, эффекты ====

local Motion = {}
AshUI.Modules.Motion = Motion
AshUI.Motion = Motion

-- ==== EASING (математика кривых) ====

local Easings = {}
Motion.Easings = Easings

function Easings.linear(t)
	return t
end

function Easings.inQuad(t)
	return t * t
end

function Easings.outQuad(t)
	return t * (2 - t)
end

function Easings.inOutQuad(t)
	if t < 0.5 then
		return 2 * t * t
	end
	return -1 + (4 - 2 * t) * t
end

function Easings.inCubic(t)
	return t * t * t
end

function Easings.outCubic(t)
	local f = t - 1
	return f * f * f + 1
end

function Easings.inOutCubic(t)
	if t < 0.5 then
		return 4 * t * t * t
	end
	local f = -2 * t + 2
	return 0.5 * f * f * f + 1
end

function Easings.inQuart(t)
	return t * t * t * t
end

function Easings.outQuart(t)
	local f = t - 1
	return 1 - f * f * f * f
end

function Easings.inOutQuart(t)
	if t < 0.5 then
		return 8 * t * t * t * t
	end
	local f = t - 1
	return 1 - 8 * f * f * f * f
end

function Easings.inQuint(t)
	return t * t * t * t * t
end

function Easings.outQuint(t)
	local f = t - 1
	return 1 + f * f * f * f * f
end

function Easings.inExpo(t)
	if t <= 0 then
		return 0
	end
	return math.pow(2, 10 * (t - 1))
end

function Easings.outExpo(t)
	if t >= 1 then
		return 1
	end
	return 1 - math.pow(2, -10 * t)
end

function Easings.inOutExpo(t)
	if t <= 0 then
		return 0
	elseif t >= 1 then
		return 1
	end
	if t < 0.5 then
		return math.pow(2, 20 * t - 10) * 0.5
	end
	return 0.5 * (2 - math.pow(2, -20 * t + 10))
end

function Easings.inSine(t)
	return 1 - math.cos((t * math.pi) / 2)
end

function Easings.outSine(t)
	return math.sin((t * math.pi) / 2)
end

function Easings.inOutSine(t)
	return -(math.cos(math.pi * t) - 1) / 2
end

function Easings.inCirc(t)
	return 1 - math.sqrt(1 - t * t)
end

function Easings.outCirc(t)
	return math.sqrt(1 - (t - 1) * (t - 1))
end

function Easings.inOutCirc(t)
	if t < 0.5 then
		return (1 - math.sqrt(1 - (2 * t) * (2 * t))) * 0.5
	end
	return (math.sqrt(1 - (-2 * t + 2) * (-2 * t + 2)) + 1) * 0.5
end

function Easings.inBack(t)
	local c1 = 1.70158
	local c3 = c1 + 1
	return c3 * t * t * t - c1 * t * t
end

function Easings.outBack(t)
	local c1 = 1.70158
	local c3 = c1 + 1
	local f = t - 1
	return 1 + c3 * f * f * f + c1 * f * f
end

function Easings.inOutBack(t)
	local c1 = 1.70158
	local c2 = c1 * 1.525
	if t < 0.5 then
		return (math.pow(2 * t, 2) * ((c2 + 1) * 2 * t - c2)) * 0.5
	end
	return (math.pow(2 * t - 2, 2) * ((c2 + 1) * (t * 2 - 2) + c2) + 2) * 0.5
end

function Easings.outElastic(t)
	local c4 = (2 * math.pi) / 3
	if t <= 0 then
		return 0
	elseif t >= 1 then
		return 1
	end
	return math.pow(2, -10 * t) * math.sin((t * 10 - 0.75) * c4) + 1
end

function Easings.inElastic(t)
	local c4 = (2 * math.pi) / 3
	local c5 = (2 * math.pi) / 4.5
	if t <= 0 then
		return 0
	elseif t >= 1 then
		return 1
	end
	return -math.pow(2, 10 * t - 10) * math.sin((t * 10 - 10.75) * c5)
end

function Easings.outBounce(t)
	local n1 = 7.5625
	local d1 = 2.75
	if t < 1 / d1 then
		return n1 * t * t
	elseif t < 2 / d1 then
		t = t - 1.5 / d1
		return n1 * t * t + 0.75
	elseif t < 2.5 / d1 then
		t = t - 2.25 / d1
		return n1 * t * t + 0.9375
	end
	t = t - 2.625 / d1
	return n1 * t * t + 0.984375
end

function Easings.inBounce(t)
	return 1 - Easings.outBounce(1 - t)
end

function Easings.inOutBounce(t)
	if t < 0.5 then
		return (1 - Easings.outBounce(1 - 2 * t)) * 0.5
	end
	return (1 + Easings.outBounce(2 * t - 1)) * 0.5
end

-- Smoothstep и его вариации — мягкие кривые для UI
function Easings.smoothStep(t)
	return t * t * (3 - 2 * t)
end

function Easings.smootherStep(t)
	return t * t * t * (t * (t * 6 - 15) + 10)
end

-- Именованный доступ: "outBack" либо функция
function Motion.Ease(nameOrFn)
	if type(nameOrFn) == "function" then
		return nameOrFn
	end
	if type(nameOrFn) == "string" then
		return Easings[nameOrFn] or Easings.outQuad
	end
	return Easings.outQuad
end

-- ==== ГЛАВНЫЙ ЦИКЛ ОБНОВЛЕНИЙ ====

-- Все активные твины / пружины / драйверы
local ActiveTweens = {}
local ActiveSprings = {}
local Drivers = {}
local FrameListeners = {}
local Uptime = 0

-- Ключи конфига твина, которые не являются свойствами инстанса
local RESERVED_TWEEN_KEYS = {
	Instance		= true,
	Time			= true,
	Duration		= true,
	Easing			= true,
	Delay			= true,
	OnUpdate		= true,
	OnComplete		= true,
	Steps			= true,
}

-- Зарегистрировать слушателя кадра
function Motion.OnUpdate(fn, order)
	if type(fn) ~= "function" then
		return function() end
	end
	local entry = {
		Fn		= fn,
		Order	= order or 0,
	}
	table.insert(FrameListeners, entry)
	return function()
		for i = #FrameListeners, 1, -1 do
			if FrameListeners[i] == entry then
				table.remove(FrameListeners, i)
			end
		end
	end
end

function Motion.Update(dt)
	Uptime = Uptime + dt

	-- 1) Твины
	for i = #ActiveTweens, 1, -1 do
		local tw = ActiveTweens[i]
		local done = false
		if not tw.Cancelled then
			tw.Elapsed = tw.Elapsed + dt
			local raw = Utils.clamp(tw.Duration <= 0 and 1 or (tw.Elapsed / tw.Duration), 0, 1)
			local t = tw.Ease(raw)
			if type(tw.OnUpdate) == "function" then
				pcall(tw.OnUpdate, t, raw)
			end
			if raw >= 1 then
				done = true
			end
		else
			done = true
		end
		if done then
			if type(tw.OnComplete) == "function" and not tw.Cancelled then
				pcall(tw.OnComplete)
			end
			table.remove(ActiveTweens, i)
		end
	end

	-- 2) Пружины
	for i = #ActiveSprings, 1, -1 do
		local sp = ActiveSprings[i]
		if not sp.Cancelled then
			sp.Elapsed = sp.Elapsed + dt
			if type(sp.OnUpdate) == "function" then
				pcall(sp.OnUpdate, dt)
			end
			if sp.Elapsed >= sp.Duration then
				if type(sp.OnComplete) == "function" then
					pcall(sp.OnComplete)
				end
				table.remove(ActiveSprings, i)
			end
		else
			table.remove(ActiveSprings, i)
		end
	end

	-- 3) Произвольные драйверы (каждый кадр)
	for i = #Drivers, 1, -1 do
		local d = Drivers[i]
		if d.Cancelled then
			table.remove(Drivers, i)
		else
			pcall(d.Fn, dt, Uptime)
		end
	end

	-- 4) Слушатели кадра
	for i = 1, #FrameListeners do
		pcall(FrameListeners[i].Fn, dt, Uptime)
	end

	-- 5) Переход темы
	pcall(Theme.Update, dt)
end

function Motion.Uptime()
	return Uptime
end

function Motion.ActiveCount()
	return #ActiveTweens + #ActiveSprings + #Drivers
end

-- Подключение к RenderStepped (внутри НЕТ wait — только чистые вычисления)
local RenderConnection = nil
function Motion.Start()
	if RenderConnection ~= nil then
		return
	end
	if Services.RunService == nil then
		log("RunService недоступен — анимации будут статичными")
		return
	end
	local ok, c = pcall(function()
		return Services.RunService.RenderStepped:Connect(function(dt)
			Motion.Update(dt)
		end)
	end)
	if ok and c ~= nil then
		RenderConnection = c
	end
end

function Motion.Stop()
	if RenderConnection ~= nil then
		pcall(function()
			RenderConnection:Disconnect()
		end)
		RenderConnection = nil
	end
end

-- ==== ТВИНЫ ====

-- Создать твин. Свойства описываются прямо в конфиге:
-- Motion.Tween({Instance=btn, Time=0.2, Easing="outQuad", BackgroundTransparency=0})
function Motion.Tween(config)
	if type(config) ~= "table" then
		return {Cancel = function() end, IsAlive = function() return false end}
	end
	local inst = config.Instance
	local duration = config.Time or config.Duration or OPTIONS.Duration
	local ease = Motion.Ease(config.Easing or Easings.outQuad)
	local delay = config.Delay or 0

	local tw = {
		Instance	= inst,
		Duration	= Utils.clamp(duration, 0, 60),
		Ease		= ease,
		Elapsed		= -(delay or 0),
		Cancelled	= false,
	}

	-- Собираем список целевых свойств
	local steps = {}
	if type(config.Steps) == "table" then
		for i, step in ipairs(config.Steps) do
			if type(step) == "table" then
				table.insert(steps, {
					Property = step.Property or step[1],
					From	= step.From,
					To		= step.To ~= nil and step.To or step[2],
				})
			end
		end
	else
		for k, v in pairs(config) do
			if RESERVED_TWEEN_KEYS[k] ~= true and type(v) ~= "function" then
				table.insert(steps, { Property = k, To = v })
			end
		end
	end

	-- Запоминаем стартовые значения
	for _, step in ipairs(steps) do
		local prop = step.Property
		if prop ~= nil and inst ~= nil then
			local cur = Utils.get(inst, prop)
			if step.From ~= nil then
				cur = step.From
				pcall(function()
					inst[prop] = cur
				end)
			end
			step.StartValue = cur
			step.TargetValue = step.To
		end
	end

	tw.OnUpdate = function(t)
		if inst == nil then
			return
		end
		for _, step in ipairs(steps) do
			local prop = step.Property
			local from = step.StartValue
			local to = step.TargetValue
			if prop == nil or from == nil or to == nil then
				continue
			end
			local value
			if type(from) == "number" and type(to) == "number" then
				value = Utils.lerp(from, to, t)
			elseif type(from) == "Color3" and type(to) == "Color3" then
				value = Utils.mix(from, to, t)
			elseif type(from) == "UDim2" and type(to) == "UDim2" then
				value = UDim2.new(
					Utils.lerp(from.X.Scale, to.X.Scale, t),
					Utils.lerp(from.X.Offset, to.X.Offset, t),
					Utils.lerp(from.Y.Scale, to.Y.Scale, t),
					Utils.lerp(from.Y.Offset, to.Y.Offset, t)
				)
			elseif type(from) == "Vector2" and type(to) == "Vector2" then
				value = Vector2.new(Utils.lerp(from.X, to.X, t), Utils.lerp(from.Y, to.Y, t))
			elseif type(from) == "Vector3" and type(to) == "Vector3" then
				value = Vector3.new(
					Utils.lerp(from.X, to.X, t),
					Utils.lerp(from.Y, to.Y, t),
					Utils.lerp(from.Z, to.Z, t)
				)
			elseif type(from) == "NumberSequence" and type(to) == "NumberSequence" then
				-- Для UISizeConstraint и прочих: просто переключаем
				value = t >= 0.5 and to or from
			else
				value = t >= 0.5 and to or from
			end
			pcall(function()
				inst[prop] = value
			end)
		end
		if type(config.OnUpdate) == "function" then
			pcall(config.OnUpdate, t)
		end
	end

	tw.OnComplete = config.OnComplete

	table.insert(ActiveTweens, tw)
	return {
		Cancel = function()
			tw.Cancelled = true
		end,
		IsAlive = function()
			return not tw.Cancelled
		end,
		Raw = tw,
	}
end

-- Несколько твинов с задержкой (stagger-появление)
function Motion.Stagger(items, config)
	local out = {}
	if type(items) ~= "table" or type(config) ~= "table" then
		return out
	end
	local stepDelay = config.Step or 0.04
	local baseCfg = config.Config or {}
	for i, item in ipairs(items) do
		local cfg = Utils.clone(baseCfg)
		cfg.Instance = item
		cfg.Delay = (config.Delay or 0) + (i - 1) * stepDelay
		if type(config.Each) == "function" then
			local extra = config.Each(item, i)
			if type(extra) == "table" then
				Utils.merge(cfg, extra)
			end
		end
		out[i] = Motion.Tween(cfg)
	end
	return out
end

-- Цикл (повторяющаяся анимация)
function Motion.Loop(config)
	if type(config) ~= "table" then
		return {Cancel = function() end}
	end
	local handle = { Cancelled = false }
	local elapsed = 0
	local index = 0
	local list = config.List or {}

	local function driver(dt)
		if handle.Cancelled then
			return
		end
		elapsed = elapsed + dt
		local dur = config.Time or 1
		if dur <= 0 then
			dur = 1
		end
		if elapsed >= dur then
			elapsed = 0
			index = index + 1
			if #list > 0 and index > #list then
				index = 1
			end
		end
		if #list == 0 then
			return
		end
		local t = elapsed / dur
		local from = list[index] or list[1]
		local to = list[index + 1] or list[1]
		if type(config.OnUpdate) == "function" then
			pcall(config.OnUpdate, t, from, to, index)
		end
	end

	local entry = { Fn = driver, Cancelled = false }
	handle.Entry = entry
	table.insert(Drivers, entry)

	handle.Cancel = function()
		handle.Cancelled = true
		for i = #Drivers, 1, -1 do
			if Drivers[i] == entry then
				table.remove(Drivers, i)
			end
		end
	end
	return handle
end

-- ==== ПРУЖИНЫ ====

-- Пружина с затуханием
function Motion.Spring(config)
	if type(config) ~= "table" then
		return {Cancel = function() end}
	end
	local inst = config.Instance
	local property = config.Property or "Position"
	local to = config.To
	local from = config.From
	if from == nil and inst ~= nil then
		from = Utils.get(inst, property)
	end
	if type(from) == "UDim2" and type(to) == "UDim2" then
		from = Vector2.new(from.X.Offset, from.Y.Offset)
		to = Vector2.new(to.X.Offset, to.Y.Offset)
	end
	if type(from) ~= "number" then
		from = 0
	end
	if type(to) ~= "number" then
		to = 0
	end

	local stiffness = config.Stiffness or 180
	local damping = config.Damping or 18
	local mass = config.Mass or 1
	local epsilon = config.RestDelta or 0.4

	local current = from
	local velocity = 0

	local sp = {
		Elapsed		= 0,
		Duration	= config.MaxTime or 3,
		Cancelled	= false,
	}

	sp.OnUpdate = function(dt)
		if dt <= 0 then
			return
		end
		local force = -stiffness * (current - to) - damping * velocity
		local accel = force / mass
		velocity = velocity + accel * dt
		current = current + velocity * dt
		if inst ~= nil and type(property) == "string" then
			pcall(function()
				inst[property] = current
			end)
		end
		if type(config.OnUpdate) == "function" then
			pcall(config.OnUpdate, current, velocity)
		end
		if math.abs(current - to) < epsilon and math.abs(velocity) < epsilon then
			sp.Elapsed = sp.Duration
		end
	end

	sp.OnComplete = function()
		if inst ~= nil and config.Commit ~= false then
			pcall(function()
				if type(config.To) == "UDim2" or type(config.To) == "number" then
					inst[property] = config.To
				end
			end)
		end
		if type(config.OnComplete) == "function" then
			pcall(config.OnComplete)
		end
	end

	table.insert(ActiveSprings, sp)
	return {
		Cancel = function()
			sp.Cancelled = true
		end,
		IsAlive = function()
			return not sp.Cancelled
		end,
		Raw = sp,
	}
end
-- ==== ПЕРЕТАСКИВАНИЕ (drag) ====

-- Универсальный drag с инерцией и snap к краям экрана.
-- Используется для окна и для перемещаемых панелей.
function Motion.Drag(handle, options)
	options = options or {}
	local uis = Services.UserInputService
	if handle == nil or uis == nil then
		return
	end

	local host = options.Host or handle.Parent
	local draggable = options.Draggable or handle
	local useAnchor = options.UseAnchor ~= false
	local thresh = options.Threshold or 0

	local dragging = false
	local enabled = options.Enabled ~= false
	local startPos = Vector2.zero
	local startOffset = Vector2.zero
	local grabOffset = Vector2.zero
	local lastPos = Vector2.zero
	local lastTime = 0
	local velocity = Vector2.zero
	local inertiaHandle = nil
	local inertiaPos = Vector2.zero
	local released = false
	local moved = false
	local moveConn = nil
	local releaseConn = nil
	local dragInputConn = nil

	-- Текущая абсолютная позиция (UDim2 от левого верхнего угла)
	local function currentOffset()
		local p = Utils.get(draggable, "Position")
		if type(p) ~= "UDim2" then
			return Vector2.zero
		end
		return Vector2.new(p.X.Offset, p.Y.Offset)
	end

	local function applyOffset(off)
		if host == nil then
			return
		end
		local screen = Utils.screenSize()
		local size = Utils.absoluteSize(host)
		local anchor = useAnchor and 0.5 or 0
		local ax, ay = (host.AnchorPoint.X), (host.AnchorPoint.Y)
		-- Позиция в пикселях относительно центра/левого верхнего угла
		local x = host.Parent and host.Parent.AbsoluteSize.X or screen.X
		local y = host.Parent and host.Parent.AbsoluteSize.Y or screen.Y
		local maxX = x - size.X * ax - 8
		local maxY = y - size.Y * ay - 8
		local minX = 8 - size.X * ax
		local minY = 8 - size.Y * ay
		off = Vector2.new(
			Utils.clamp(off.X, math.min(minX, maxX), math.max(minX, maxX)),
			Utils.clamp(off.Y, math.min(minY, maxY), math.max(minY, maxY))
		)
		pcall(function()
			draggable.Position = UDim2.new(anchor, off.X, anchor, off.Y)
		end)
	end

	-- Остановка инерции (снимает драйвер с кадра)
	local function stopInertia()
		if inertiaHandle ~= nil and inertiaHandle.Entry ~= nil then
			for i = #Drivers, 1, -1 do
				if Drivers[i] == inertiaHandle.Entry then
					table.remove(Drivers, i)
				end
			end
		end
		inertiaHandle = nil
	end

	local function onInputBegan(input)
		if not enabled or dragging or released then
			return
		end
		local processed = false
		pcall(function()
			processed = uis:GetFocusedTextBox() ~= nil
		end)
		if processed then
			return
		end
		if input.UserInputType ~= Enum.UserInputType.MouseButton1
			and input.UserInputType ~= Enum.UserInputType.Touch then
			return
		end
		local mouse = Utils.mousePos()
		local size = Utils.absoluteSize(draggable)
		local pos = Utils.absolutePos(draggable)
		if thresh > 0 then
			if not Utils.rectContains(mouse - pos, size, Vector2.zero) then
				return
			end
		end
		dragging = true
		moved = false
		-- новая точка отсчёта для инерции
		inertiaPos = currentOffset()
		startPos = mouse
		startOffset = currentOffset()
		grabOffset = mouse - pos
		lastPos = mouse
		lastTime = os.clock()
		velocity = Vector2.zero
		stopInertia()
		pcall(function()
			draggable.Active = true
		end)
		if type(options.OnStart) == "function" then
			pcall(options.OnStart)
		end
		if type(options.Lift) == "function" then
			pcall(options.Lift)
		end
	end

	local function onInputChanged(input)
		if not dragging then
			return
		end
		if input.UserInputType ~= Enum.UserInputType.MouseMovement
			and input.UserInputType ~= Enum.UserInputType.Touch then
			return
		end
		local mouse = Utils.mousePos()
		local dt = math.max(os.clock() - lastTime, 0.001)
		velocity = (mouse - lastPos) / dt
		lastPos = mouse
		lastTime = os.clock()
		local delta = mouse - startPos
		if math.abs(delta.X) > 1 or math.abs(delta.Y) > 1 then
			moved = true
		end
		applyOffset(startOffset + delta)
	end

	local function onInputEnded()
		if not dragging then
			return
		end
		dragging = false
		pcall(function()
			draggable.Active = false
		end)
		if type(options.OnEnd) == "function" then
			pcall(options.OnEnd, moved)
		end
		-- Инерция: окно «долистывается» по инерции и тормозит трением
		stopInertia()
		local speed = velocity.Magnitude
		if options.Inertia == false or (options.Inertia == nil and moved == false) then
			speed = 0
		end
		if speed > 30 then
			local pos = currentOffset()
			local vx = Utils.clamp(velocity.X, -2600, 2600)
			local vy = Utils.clamp(velocity.Y, -2600, 2600)
			local friction = options.Friction or 4.2
			local entry
			inertiaHandle = { Entry = nil }
			entry = {
				Fn = function(dt)
					if inertiaHandle == nil or inertiaHandle.Entry ~= entry then
						return
					end
					-- интегрируем позицию, затем гасим скорость
					local step = Vector2.new(vx, vy) * dt
					pos = pos + step
					applyOffset(pos)
					vx = Utils.damp(vx, 0, friction, dt)
					vy = Utils.damp(vy, 0, friction, dt)
					if vx * vx + vy * vy < 36 then
						stopInertia()
					end
				end,
				Cancelled = false,
			}
			inertiaHandle.Entry = entry
			table.insert(Drivers, entry)
		end

		-- Snap к краям
		if options.Snap == true or (options.Snap == nil and OPTIONS.Snap) then
			local pad = options.SnapPad or OPTIONS.SnapPad
			local size = Utils.absoluteSize(host)
			local off = currentOffset()
			local parentSize = host.Parent and host.Parent.AbsoluteSize or Utils.screenSize()
			local ax, ay = host.AnchorPoint.X, host.AnchorPoint.Y
			local left = off.X - size.X * ax
			local top = off.Y - size.Y * ay
			local right = off.X + size.X * (1 - ax)
			local bottom = off.Y + size.Y * (1 - ay)
			local targetX, targetY = off.X, off.Y

			local closest, bestDist = nil, math.huge
			-- верхний край (к titlebar-высоте, если задан)
			if options.TopInset then
				local d = math.abs(top - (pad - options.TopInset))
				if d < bestDist then
					bestDist, closest = d, Vector2.new(off.X, pad - options.TopInset + size.Y * ay)
				end
			end
			local d = math.abs(left - pad)
			if d < bestDist then
				bestDist, closest = d, Vector2.new(pad + size.X * ax, off.Y)
			end
			d = math.abs(right - (parentSize.X - pad))
			if d < bestDist then
				bestDist, closest = d, Vector2.new(parentSize.X - pad - size.X * (1 - ax), off.Y)
			end
			d = math.abs(top - pad)
			if d < bestDist then
				bestDist, closest = d, Vector2.new(off.X, pad + size.Y * ay)
			end
			d = math.abs(bottom - (parentSize.Y - pad))
			if d < bestDist then
				bestDist, closest = d, Vector2.new(off.X, parentSize.Y - pad - size.Y * (1 - ay))
			end
			if closest ~= nil and bestDist <= (options.SnapRange or 60) then
				Motion.Tween({
					Instance = draggable,
					Time = options.SnapDuration or 0.22,
					Easing = Easings.outCubic,
					Position = UDim2.new(useAnchor and 0.5 or 0, closest.X, useAnchor and 0.5 or 0, closest.Y),
					OnComplete = function()
						if type(options.OnSnap) == "function" then
							pcall(options.OnSnap)
						end
					end,
				})
			end
		end
		-- moveConn / releaseConn / dragInputConn остаются подключёнными
		-- на всё время жизни окна — переподключать их не нужно.
	end

	dragInputConn = uis.InputBegan:Connect(onInputBegan)
	moveConn = uis.InputChanged:Connect(onInputChanged)
	releaseConn = uis.InputEnded:Connect(onInputEnded)

	handle.Dragging = function()
		return dragging
	end
	handle.Moved = function()
		return moved
	end
	handle.StopInertia = stopInertia
	handle.Enable = function(v)
		enabled = v ~= false
	end
	handle.Disable = function()
		enabled = false
		stopInertia()
	end
end

-- ==== HOVER / НАВЕДЕНИЕ ====

-- Универсальный hover-эффект: смена прозрачностей/цветов с плавным твином.
-- Поддерживает вложенный список потомков, если передан Options.Chain.
function Motion.Hover(guiObject, config)
	if guiObject == nil or type(config) ~= "table" then
		return {}
	end
	local from = config.From or {}
	local to = config.To or {}
	local duration = config.Time or 0.14
	local easing = config.Easing or Easings.outQuad
	local hovered = false
	local tween = nil
	local handlers = {}
	local chain = config.Chain
	local defaultFrom = {}
	local defaultTo = {}

	for k, v in pairs(from) do
		defaultFrom[k] = v
	end
	for k, v in pairs(to) do
		defaultTo[k] = v
	end

	local function targets()
		if chain ~= nil and config.Self then
			return chain
		end
		if chain ~= nil then
			return { chain }
		end
		return { guiObject }
	end

	local function applyState(isHovered)
		for _, target in ipairs(targets()) do
			if target ~= nil then
				for k, v in pairs(defaultFrom) do
					Motion.Tween({ Instance = target, Time = duration, Easing = easing, [k] = v })
				end
				for k, v in pairs(defaultTo) do
					Motion.Tween({ Instance = target, Time = duration, Easing = easing, [k] = v })
				end
			end
		end
		if type(config.OnHover) == "function" then
			pcall(config.OnHover, isHovered)
		end
	end

	local function bind(target)
		if target == nil or target.MouseEnter == nil then
			return
		end
		local enter = target.MouseEnter:Connect(function()
			if config.Disabled == true then
				return
			end
			hovered = true
			if tween then pcall(tween.Cancel) end
			tween = nil
			if config.Scale then
				Motion.Tween({
					Instance = target,
					Time = duration,
					Easing = easing,
					Size = config.Scale,
					Rotation = config.Rotation or 0,
				})
			end
			applyState(true)
		end)
		local leave = target.MouseLeave:Connect(function()
			hovered = false
			if tween then pcall(tween.Cancel) end
			tween = nil
			if config.Scale then
				Motion.Tween({
					Instance = target,
					Time = duration,
					Easing = easing,
					Size = config.ScaleFrom or config.BaseSize,
					Rotation = 0,
				})
			end
			applyState(false)
		end)
		if config.PressScale then
			target.MouseButton1Down:Connect(function()
				Motion.Tween({
					Instance = target,
					Time = 0.08,
					Easing = Easings.outQuad,
					Size = config.PressScale,
				})
			end)
			local function release()
				Motion.Tween({
					Instance = target,
					Time = 0.14,
					Easing = Easings.outBack,
					Size = hovered and config.Scale or (config.ScaleFrom or config.BaseSize),
				})
			end
			target.MouseButton1Up:Connect(release)
			target.MouseButton1Leave:Connect(release)
		end
		table.insert(handlers, enter)
		table.insert(handlers, leave)
	end

	if config.Chain ~= nil and config.All then
		for _, child in ipairs(targets()) do
			bind(child)
		end
	else
		bind(guiObject)
	end

	return {
		IsHovered = function()
			return hovered
		end,
		Destroy = function()
			for _, h in ipairs(handlers) do
				pcall(function()
					h:Disconnect()
				end)
			end
		end,
	}
end

-- ==== ЭФФЕКТЫ ====

-- Пульсация (плавно меняет прозрачность/цвет туда-обратно)
function Motion.Pulse(guiObject, options)
	options = options or {}
	if guiObject == nil then
		return {Cancel = function() end}
	end
	local prop = options.Property or "ImageTransparency"
	local a = options.From or 0.2
	local b = options.To or 0.8
	local duration = options.Time or 1.4
	local ease = options.Easing or Easings.inOutSine
	local cancelled = false
	local t = 0
	local direction = 1

	Motion.OnUpdate(function(dt)
		if cancelled then
			return
		end
		t = t + dt * direction
		if t >= duration then
			t = duration
			direction = -1
		elseif t <= 0 then
			t = 0
			direction = 1
		end
		local e = ease(t / duration)
		local v = Utils.lerp(a, b, e)
		if type(v) == "Color3" and type(a) == "Color3" then
			v = Utils.mix(a, b, e)
		end
		pcall(function()
			guiObject[prop] = v
		end)
	end)

	return {
		Cancel = function()
			cancelled = true
		end,
	}
end

-- Дрожание (короткая серия смещений)
function Motion.Shake(guiObject, options)
	options = options or {}
	if guiObject == nil then
		return
	end
	local strength = options.Strength or 8
	local duration = options.Time or 0.35
	local axis = options.Axis or "X"
	local basePos = Utils.get(guiObject, options.Property or "Position")
	local start = 0
	local freq = options.Frequency or 34
	local elapsed = 0

	Motion.OnUpdate(function(dt)
		elapsed = elapsed + dt
		if elapsed > duration then
			if type(basePos) == "UDim2" then
				pcall(function()
					guiObject[options.Property or "Position"] = basePos
				end)
			end
			return
		end
		local dampen = 1 - (elapsed / duration)
		local off = math.sin(elapsed * freq) * strength * dampen
		if type(basePos) == "UDim2" then
			local dx, dy = 0, 0
			if axis == "X" then
				dx = off
			elseif axis == "Y" then
				dy = off
			else
				dx, dy = off, off
			end
			pcall(function()
				guiObject[options.Property or "Position"] = UDim2.new(
					basePos.X.Scale, basePos.X.Offset + dx,
					basePos.Y.Scale, basePos.Y.Offset + dy
				)
			end)
		end
	end)
end

-- Кэш стартовых свойств для Reveal (Roblox не разрешает дописывать поля в инстансы)
local RevealCache = setmetatable({}, {__mode = "k"})

-- Появление (fade + сдвиг) — базовый «reveal»
function Motion.Reveal(guiObject, options)
	options = options or {}
	if guiObject == nil then
		return
	end
	local props = RevealCache[guiObject]
	if props == nil then
		props = {}
		if guiObject:IsA("TextLabel") or guiObject:IsA("TextButton") then
			props = { TextTransparency = 0 }
		end
		if guiObject:IsA("Frame") or guiObject:IsA("ScrollingFrame") then
			props.BackgroundTransparency = 0
		end
		RevealCache[guiObject] = props
	end

	-- Стартовое состояние
	for k in pairs(props) do
		pcall(function()
			guiObject[k] = 1
		end)
	end
	local dx = options.Dx or 0
	local dy = options.Dy or -8
	local basePos = Utils.get(guiObject, "Position")
	local hiddenPos = nil
	if type(basePos) == "UDim2" then
		hiddenPos = UDim2.new(basePos.X.Scale, basePos.X.Offset + dx, basePos.Y.Scale, basePos.Y.Offset + dy)
	end

	local tweenCfg = {
		Instance = guiObject,
		Time = options.Time or 0.24,
		Easing = options.Easing or Easings.outCubic,
		Delay = options.Delay or 0,
	}
	for k, v in pairs(props) do
		tweenCfg[k] = v
	end
	if hiddenPos ~= nil then
		tweenCfg.Position = basePos
	end
	local handle = Motion.Tween(tweenCfg)
	if hiddenPos ~= nil then
		pcall(function()
			guiObject.Position = hiddenPos
		end)
	end
	return handle
end

-- Скрытие элемента (обратное reveal)
function Motion.Hide(guiObject, options)
	options = options or {}
	if guiObject == nil then
		return
	end
	local props = RevealCache[guiObject] or { BackgroundTransparency = 0 }
	local tweenCfg = {
		Instance = guiObject,
		Time = options.Time or 0.16,
		Easing = options.Easing or Easings.inQuad,
	}
	for k in pairs(props) do
		tweenCfg[k] = 1
	end
	return Motion.Tween(tweenCfg)
end

-- Появление списка элементов с задержкой (stagger)
function Motion.RevealList(items, options)
	options = options or {}
	local step = options.Step or 0.035
	local baseDelay = options.Delay or 0
	local out = {}
	if type(items) ~= "table" then
		return out
	end
	for i, item in ipairs(items) do
		local cfg = Utils.clone(options.Config or {})
		cfg.Time = cfg.Time or options.Time or 0.26
		cfg.Easing = cfg.Easing or Easings.outCubic
		cfg.Delay = baseDelay + (i - 1) * step
		cfg.Dx = cfg.Dx or options.Dx or 0
		cfg.Dy = cfg.Dy or options.Dy or -10
		out[i] = Motion.Reveal(item, cfg)
	end
	return out
end

-- Плавное переключение двух объектов (появление/исчезновение)
function Motion.SwapIn(newObj, oldObj, options)
	options = options or {}
	if newObj ~= nil then
		pcall(function()
			newObj.Visible = true
		end)
		Motion.Reveal(newObj, {
			Time = options.Time or 0.22,
			Delay = options.Delay or 0,
			Dx = options.Dx or 18,
		})
	end
	if oldObj ~= nil then
		Motion.Tween({
			Instance = oldObj,
			Time = (options.Time or 0.22) * 0.6,
			Easing = Easings.inQuad,
			BackgroundTransparency = 1,
			OnComplete = function()
				pcall(function()
					oldObj.Visible = false
				end)
			end,
		})
		if oldObj:IsA("TextLabel") or oldObj:IsA("TextButton") then
			Motion.Tween({
				Instance = oldObj,
				Time = (options.Time or 0.22) * 0.6,
				TextTransparency = 1,
			})
		end
	end
end

-- Быстрое «мигание» цвета (например, при сохранении)
function Motion.Flash(guiObject, color, duration)
	if guiObject == nil then
		return
	end
	local property = "BackgroundColor3"
	local old = Utils.get(guiObject, property)
	Motion.Tween({
		Instance = guiObject,
		Time = (duration or 0.5) * 0.35,
		Easing = Easings.outQuad,
		[property] = color or Theme.C("Success"),
		OnComplete = function()
			Motion.Tween({
				Instance = guiObject,
				Time = (duration or 0.5) * 0.65,
				Easing = Easings.inOutQuad,
				[property] = old,
			})
		end,
	})
end

-- Счётчик числа (анимированное изменение Text)
function Motion.CountTo(guiObject, from, to, options)
	options = options or {}
	if guiObject == nil then
		return
	end
	local fmt = options.Format or Utils.formatNumber
	local suffix = options.Suffix or ""
	local decimals = options.Decimals
	Motion.Tween({
		Instance = nil,
		Time = options.Time or 0.6,
		Easing = options.Easing or Easings.outCubic,
		OnUpdate = function(t)
			local v = Utils.lerp(from, to, t)
			local text
			if decimals then
				text = string.format("%." .. decimals .. "f%s", v, suffix)
			else
				text = fmt(v) .. suffix
			end
			pcall(function()
				guiObject.Text = text
			end)
		end,
		OnComplete = function()
			local text = fmt(to) .. suffix
			pcall(function()
				guiObject.Text = text
			end)
		end,
	})
end
-- ==== Anim — декоративные слои: луна, звёзды, аврора, shimmer ====

local Anim = {}
AshUI.Modules.Anim = Anim
AshUI.Anim = Anim

-- Счётчики звезёзд
local StarCounter = 0

-- Создаёт слой со скругением и клиппингом
function Anim.clipFrame(parent, radius)
	local f = Utils.new("Frame", {
		BackgroundTransparency = 1,
		BorderSizePixel = 0,
		ClipsDescendants = true,
		Size = UDim2.fromScale(1, 1),
		Parent = parent,
	})
	if radius then
		Utils.corner(f, radius)
	end
	return f
end

-- ==== ФОН СО ЗВЁЗДАМИ ====

-- Создаёт поле со звёздами + луной + авророй. Возвращает контроллер.
function Anim.CreateSpaceField(parent, options)
	options = options or {}
	local width = options.Width or 300
	local height = options.Height or 160
	local starCount = options.Stars or 46

	local field = Anim.clipFrame(parent, nil)
	field.Name = "SpaceField"
	field.Size = UDim2.fromOffset(width, height)
	field.Visible = options.Visible ~= false

	-- Аврора — два больших размытых пятна
	local auroraA = Utils.new("Frame", {
		BackgroundColor3 = Theme.C("Aurora"),
		BackgroundTransparency = 0.86,
		BorderSizePixel = 0,
		Size = UDim2.fromOffset(width * 1.1, height * 0.9),
		Position = UDim2.fromOffset(-width * 0.15, -height * 0.2),
		Rotation = 18,
		Parent = field,
	})
	Utils.corner(auroraA, 120)
	local auroraGradA = Utils.gradient(auroraA, Theme.C("Aurora"), Theme.C("AuroraAlt"), 30)
	auroraGradA.Transparency = NumberSequence.new({
		NumberSequenceKeypoint.new(0, 1),
		NumberSequenceKeypoint.new(0.5, 0.55),
		NumberSequenceKeypoint.new(1, 1),
	})

	local auroraB = Utils.new("Frame", {
		BackgroundColor3 = Theme.C("AuroraAlt"),
		BackgroundTransparency = 0.9,
		BorderSizePixel = 0,
		Size = UDim2.fromOffset(width * 0.8, height * 0.8),
		Position = UDim2.fromOffset(width * 0.45, height * 0.3),
		Rotation = -22,
		Parent = field,
	})
	Utils.corner(auroraB, 110)
	local auroraGradB = Utils.gradient(auroraB, Theme.C("AuroraAlt"), Theme.C("Aurora"), 210)
	auroraGradB.Transparency = NumberSequence.new({
		NumberSequenceKeypoint.new(0, 1),
		NumberSequenceKeypoint.new(0.5, 0.6),
		NumberSequenceKeypoint.new(1, 1),
	})

	-- Свечение за луной
	local glow = Utils.new("Frame", {
		BackgroundColor3 = Theme.C("Moon"),
		BackgroundTransparency = 0.92,
		BorderSizePixel = 0,
		Size = UDim2.fromOffset(120, 120),
		Position = UDim2.fromOffset(-34, -30),
		Parent = field,
	})
	Utils.corner(glow, 60)
	local glowGrad = Utils.gradient(glow, Color3.new(1, 1, 1), Color3.new(1, 1, 1), 45)
	glowGrad.Transparency = NumberSequence.new({
		NumberSequenceKeypoint.new(0, 0.6),
		NumberSequenceKeypoint.new(1, 1),
	})

	-- ==== ЛУНА ====
	local moonRoot = Utils.new("Frame", {
		BackgroundTransparency = 1,
		Size = UDim2.fromOffset(74, 74),
		Position = UDim2.fromOffset(width - 66, -24),
		Rotation = -8,
		Parent = field,
	})

	-- Внешнее мягкое свечение
	local moonGlow = Utils.new("ImageLabel", {
		BackgroundTransparency = 1,
		Size = UDim2.fromScale(2.1, 2.1),
		Position = UDim2.fromOffset(-28, -28),
		Image = "rbxassetid://6029598422",
		ImageColor3 = Theme.C("Moon"),
		ImageTransparency = 0.72,
		ZIndex = 1,
		Parent = moonRoot,
	})
	Utils.corner(moonGlow, 999)

	-- Тело луны
	local moon = Utils.new("Frame", {
		BackgroundColor3 = Theme.C("Moon"),
		BackgroundTransparency = 0.02,
		BorderSizePixel = 0,
		Size = UDim2.fromOffset(56, 56),
		Position = UDim2.fromOffset(9, 9),
		ZIndex = 2,
		Parent = moonRoot,
	})
	Utils.corner(moon, 999)
	local moonGrad = Utils.gradient(moon, Theme.C("Moon"), Theme.C("MoonShadow"), 35)
	moonGrad.Transparency = NumberSequence.new({
		NumberSequenceKeypoint.new(0, 0.05),
		NumberSequenceKeypoint.new(1, 0.45),
	})

	-- Кратеры (маленькие тёмные кружки)
	local craters = {}
	local craterData = {
		{0.28, 0.30, 7.5},
		{0.62, 0.24, 5.0},
		{0.40, 0.56, 9.0},
		{0.72, 0.58, 6.2},
		{0.22, 0.70, 4.4},
		{0.58, 0.78, 7.0},
		{0.84, 0.36, 3.6},
		{0.12, 0.46, 3.2},
	}
	for _, c in ipairs(craterData) do
		local crater = Utils.new("Frame", {
			BackgroundColor3 = Theme.C("MoonShadow"),
			BackgroundTransparency = 0.62,
			BorderSizePixel = 0,
			Size = UDim2.fromOffset(c[3], c[3]),
			Position = UDim2.fromScale(c[1], c[2]),
			ZIndex = 3,
			Parent = moon,
		})
		Utils.corner(crater, 999)
		-- Подсветка края кратера
		local rim = Utils.new("Frame", {
			BackgroundColor3 = Theme.C("Moon"),
			BackgroundTransparency = 0.72,
			BorderSizePixel = 0,
			Size = UDim2.fromScale(1, 1),
			Position = UDim2.fromOffset(-1.2, -1.2),
			ZIndex = 2,
			Parent = crater,
		})
		Utils.corner(rim, 999)
		table.insert(craters, crater)
	end

	-- Терминатор — тёмная сторона
	local shadow = Utils.new("Frame", {
		BackgroundColor3 = Theme.C("MoonShadow"),
		BackgroundTransparency = 0.5,
		BorderSizePixel = 0,
		Size = UDim2.fromOffset(50, 56),
		Position = UDim2.fromOffset(30, 4),
		ZIndex = 4,
		Parent = moon,
	})
	Utils.corner(shadow, 999)
	local shadowGrad = Utils.gradient(shadow, Color3.new(0, 0, 0), Color3.new(1, 1, 1), 0)
	shadowGrad.Color = ColorSequence.new(Theme.C("MoonShadow"), Theme.C("Moon"))
	shadowGrad.Transparency = NumberSequence.new({
		NumberSequenceKeypoint.new(0, 0.15),
		NumberSequenceKeypoint.new(0.75, 0.85),
		NumberSequenceKeypoint.new(1, 1),
	})

	-- ==== ЗВЁЗДЫ ====
	local stars = {}
	for i = 1, starCount do
		StarCounter = StarCounter + 1
		local size = Utils.rand(1.4, 3.4)
		local x = Utils.rand(0, width - 8)
		local y = Utils.rand(2, height - 10)
		local star = Utils.new("Frame", {
			BackgroundColor3 = (i % 3 == 0) and Theme.C("StarDim") or Theme.C("Star"),
			BackgroundTransparency = Utils.rand(0.25, 0.6),
			BorderSizePixel = 0,
			Size = UDim2.fromOffset(size, size),
			Position = UDim2.fromOffset(x, y),
			Rotation = Utils.rand(0, 360),
			ZIndex = 2,
			Parent = field,
		})
		Utils.corner(star, 999)
		-- Свечение для крупных звёзд
		local starGlow = nil
		if size > 2.6 then
			starGlow = Utils.new("ImageLabel", {
				BackgroundTransparency = 1,
				Size = UDim2.fromOffset(size * 6, size * 6),
				Position = UDim2.fromOffset(-size * 2.5, -size * 2.5),
				Image = "rbxassetid://6029598422",
				ImageColor3 = Theme.C("Star"),
				ImageTransparency = 0.86,
				ZIndex = 1,
				Parent = star,
			})
			Utils.corner(starGlow, 999)
		end
		-- Состояние звезды держим в side-таблице (инстансы не принимают кастомных полей)
		table.insert(stars, {
			Node		= star,
			Glow		= starGlow,
			Phase		= Utils.rand(0, math.pi * 2),
			Speed		= Utils.rand(0.25, 1.1),
			BaseT		= star.BackgroundTransparency,
			DriftX		= Utils.rand(-0.6, 0.6),
			DriftY		= Utils.rand(-0.9, 0.35),
			BaseX		= x,
			BaseY		= y,
		})
	end

	-- Пыльный слой (мелкие точки)
	local dust = {}
	for _ = 1, math.floor(starCount * 0.6) do
		local px, py = Utils.rand(0, width), Utils.rand(0, height)
		local d = Utils.new("Frame", {
			BackgroundColor3 = Theme.C("StarDim"),
			BackgroundTransparency = Utils.rand(0.75, 0.95),
			BorderSizePixel = 0,
			Size = UDim2.fromOffset(1, 1),
			Position = UDim2.fromOffset(px, py),
			Parent = field,
		})
		Utils.corner(d, 999)
		table.insert(dust, {
			Node	= d,
			BaseX	= px,
			BaseY	= py,
			DriftX	= Utils.rand(-0.35, 0.35),
			DriftY	= Utils.rand(-0.5, 0.1),
		})
	end

	-- ==== АНИМАЦИЯ ДЕКОРА ====
	local time = 0
	local moonPhase = 0
	local controller = {
		Field		= field,
		Moon		= moon,
		MoonRoot	= moonRoot,
		Stars		= stars,
		Alive		= true,
	}

	controller.Update = function(dt)
		if not controller.Alive then
			return
		end
		time = time + dt
		-- Мерцание звёзд
		for _, s in ipairs(stars) do
			local twinkle = Utils.wave(time, s.Speed, s.Phase)
			pcall(function()
				s.Node.BackgroundTransparency = Utils.clamp(s.BaseT + (0.45 - twinkle) * 0.5, 0, 1)
			end)
			-- Медленный дрейф
			s.BaseX = s.BaseX + s.DriftX * dt
			s.BaseY = s.BaseY + s.DriftY * dt
			if s.BaseX < -4 then s.BaseX = width + 4 end
			if s.BaseX > width + 4 then s.BaseX = -4 end
			if s.BaseY < -4 then s.BaseY = height + 4 end
			if s.BaseY > height + 4 then s.BaseY = -4 end
			pcall(function()
				s.Node.Position = UDim2.fromOffset(s.BaseX, s.BaseY)
			end)
			if s.Glow ~= nil then
				pcall(function()
					s.Glow.ImageTransparency = 0.7 + twinkle * 0.25
				end)
			end
		end
		-- Пыль
		for _, d in ipairs(dust) do
			d.BaseX = d.BaseX + d.DriftX * dt
			d.BaseY = d.BaseY + d.DriftY * dt
			if d.BaseX < 0 then d.BaseX = width end
			if d.BaseX > width then d.BaseX = 0 end
			if d.BaseY < 0 then d.BaseY = height end
			if d.BaseY > height then d.BaseY = 0 end
			pcall(function()
				d.Node.Position = UDim2.fromOffset(d.BaseX, d.BaseY)
			end)
		end
		-- Дыхание луны
		local pulse = Utils.wave(time, 0.22, 0)
		pcall(function()
			moonGlow.ImageTransparency = 0.6 + pulse * 0.28
		end)
		-- Медленное вращение луны
		moonPhase = time * 4
		pcall(function()
			moonRoot.Rotation = -8 + math.sin(moonPhase * 0.12) * 3
			moonRoot.Position = UDim2.new(1, -66 + math.sin(moonPhase * 0.08) * 4, 0, -24 + math.cos(moonPhase * 0.1) * 3)
		end)
		-- Дрейф авроры
		pcall(function()
			auroraA.Position = UDim2.fromOffset(-width * 0.15 + math.sin(time * 0.15) * 22, -height * 0.2 + math.cos(time * 0.12) * 14)
			auroraB.Position = UDim2.fromOffset(width * 0.45 + math.cos(time * 0.11) * 26, height * 0.3 + math.sin(time * 0.14) * 12)
		end)
	end

	controller.Theme = function()
		-- Перекрашиваем декор под текущую палитру
		Theme.Bind("Aurora", auroraA, "BackgroundColor3")
		Theme.Bind("AuroraAlt", auroraB, "BackgroundColor3")
		Theme.Bind("Moon", moon, "BackgroundColor3")
		Theme.Bind("Moon", moonGlow, "ImageColor3")
		Theme.Bind("Moon", glow, "BackgroundColor3")
		Theme.Bind("MoonShadow", shadow, "BackgroundColor3")
		for i, s in ipairs(stars) do
			Theme.Bind((i % 3 == 0) and "StarDim" or "Star", s.Node, "BackgroundColor3")
			if s.Glow ~= nil then
				Theme.Bind("Star", s.Glow, "ImageColor3")
			end
		end
	end

	controller.Destroy = function()
		controller.Alive = false
		Utils.destroy(field)
	end

	Motion.OnUpdate(function(dt)
		controller.Update(dt)
	end, 5)

	return controller
end

-- ==== SHIMMER (блик, пробегающий по кнопке) ====

function Anim.Shimmer(guiObject, options)
	options = options or {}
	if guiObject == nil then
		return {Cancel = function() end}
	end
	local sh = Utils.new("Frame", {
		BackgroundColor3 = Color3.new(1, 1, 1),
		BackgroundTransparency = 0.92,
		BorderSizePixel = 0,
		Size = UDim2.new(0.4, 0, 1, 0),
		Position = UDim2.new(-0.6, 0, 0, 0),
		Rotation = 18,
		ZIndex = (guiObject.ZIndex or 1) + 1,
		ClipsDescendants = true,
		Parent = guiObject,
	})
	Utils.corner(sh, OPTIONS.Radius)
	local grad = Utils.gradient(sh, Color3.new(1, 1, 1), Color3.new(1, 1, 1), 90)
	grad.Transparency = NumberSequence.new({
		NumberSequenceKeypoint.new(0, 1),
		NumberSequenceKeypoint.new(0.5, 0.55),
		NumberSequenceKeypoint.new(1, 1),
	})

	local handle = {
		Cancelled = false,
		Progress = 0,
	}
	local interval = options.Interval or 4.5
	local timer = 0

	Motion.OnUpdate(function(dt)
		if handle.Cancelled then
			return
		end
		timer = timer + dt
		if timer < interval then
			return
		end
		handle.Progress = handle.Progress + dt * (options.Speed or 1.1)
		if handle.Progress >= 1.6 then
			handle.Progress = 0
			timer = 0
		end
		local p = Utils.clamp(handle.Progress, 0, 1)
		pcall(function()
			sh.Position = UDim2.new(-0.6 + p * 1.7, 0, 0, 0)
			sh.BackgroundTransparency = 0.92 - math.sin(p * math.pi) * 0.45
		end)
	end, 6)

	return {
		Cancel = function()
			handle.Cancelled = true
			Utils.destroy(sh)
		end,
		Frame = sh,
	}
end

-- ==== МЕРЦАНИЕ/ПУЛЬС КАРТОЧЕК ====

function Anim.PulseFrame(guiObject, options)
	options = options or {}
	local baseColor = options.Color or Theme.C("Accent")
	local toColor = options.To or Utils.lighten(baseColor, 0.25)
	local time = 0
	local cancelled = false
	local duration = options.Time or 2.2
	Motion.OnUpdate(function(dt)
		if cancelled then
			return
		end
		time = time + dt
		local t = Utils.wave(time, 1 / duration, 0)
		local col = Utils.mix(baseColor, toColor, t)
		pcall(function()
			guiObject.BackgroundColor3 = col
		end)
	end, 4)
	return {
		Cancel = function()
			cancelled = true
		end,
	}
end

-- ==== СВЕЧЕНИЕ (UIStroke-подобное) ====

-- Создаёт «неоновое» свечение вокруг элемента
function Anim.Glow(parent, color, thickness, transparency)
	local stroke = Utils.stroke(parent, color or Theme.C("Accent"), transparency or 0.4, thickness or 2)
	Utils.set(stroke, {
		LineJoinMode = Enum.LineJoinMode.Round,
	})
	return stroke
end

-- ==== СОГНУТАЯ (изогнутая) полоса-разделитель ====

function Anim.CurvedLine(parent, color, thickness, width)
	local holder = Utils.new("Frame", {
		BackgroundTransparency = 1,
		Size = UDim2.new(1, 0, 0, (thickness or 2) + 4),
		Parent = parent,
	})
	-- width — доля ширины родителя (0..1); если > 1, трактуется как пиксели
	local lineSize
	if type(width) == "number" and width <= 1 then
		lineSize = UDim2.new(width, 0, 0, thickness or 2)
	else
		lineSize = UDim2.new(0, type(width) == "number" and width or 0, 0, thickness or 2)
	end
	local line = Utils.new("Frame", {
		BackgroundColor3 = color or Theme.C("BorderSoft"),
		BackgroundTransparency = 0,
		BorderSizePixel = 0,
		Size = lineSize,
		Position = UDim2.new(0.5, 0, 0.5, 0),
		AnchorPoint = Vector2.new(0.5, 0.5),
		Parent = holder,
	})
	Utils.corner(line, 999)
	return holder, line
end

-- ==== Registry — реестр флагов и элементов ====

local Registry = {}
AshUI.Modules.Registry = Registry
AshUI.Registry = Registry

-- Хранилище флагов: Registry.Flags[Имя] = flagObject
Registry.Flags = {}
-- Хранилище элементов (нужно для Apply/Save/Load)
Registry.Elements = {}
-- Подписчики на изменение флага
local FlagListeners = {}
-- Позиция в Config (порядок сохранения)
local FlagOrder = {}

-- Типы значений флага
local FLAG_TYPES = {
	Boolean	= "boolean",
	Number	= "number",
	String	= "string",
	Color	= "Color3",
	Key	= "string",
	Table	= "table",
}

-- Создать флаг
function Registry.Create(flagName, defaultValue, meta)
	if type(flagName) ~= "string" or #flagName == 0 then
		return nil
	end
	meta = meta or {}
	local existing = Registry.Flags[flagName]
	if existing ~= nil then
		-- Обновляем метаданные, значение оставляем (или сбрасываем)
		existing.Meta = Utils.merge(existing.Meta or {}, meta)
		existing.Type = FLAG_TYPES[type(defaultValue)] or existing.Type
		existing.DefaultValue = defaultValue
		if meta.Reset then
			existing:Set(defaultValue, true)
		end
		return existing
	end

	local flag = {
		Name		= flagName,
		Value		= defaultValue,
		DefaultValue	= defaultValue,
		Type		= FLAG_TYPES[type(defaultValue)] or "string",
		Meta		= meta,
		Changed		= 0,
		Hidden		= meta.Hidden == true,
	}

	function flag:Get()
		return self.Value
	end

	function flag:GetMeta(key)
		return self.Meta[key]
	end

	-- Установить значение с проверкой типа и оповещением подписчиков
	function flag:Set(value, silent)
		-- Декодируем сериализованную форму {__t="Color3", v="#RRGGBB"} из конфига
		if type(value) == "table" and type(value.__t) == "string" then
			if value.__t == "Color3" then
				value = Utils.fromHex(value.v)
			elseif value.__t == "Table" then
				local arr = {}
				for i, item in ipairs(value.v or {}) do
					arr[i] = tostring(item)
				end
				value = arr
			else
				value = value.v
			end
		end
		local changed = self.Value ~= value
		-- Нормализация
		if self.Type == "number" and type(value) == "number" then
			if self.Meta.Step then
				value = Utils.round(value, self.Meta.Step)
			end
			if self.Meta.Min then
				value = math.max(value, self.Meta.Min)
			end
			if self.Meta.Max then
				value = math.min(value, self.Meta.Max)
			end
		elseif self.Type == "Color3" and type(value) == "Color3" then
			-- без изменений
		elseif self.Type == "boolean" then
			value = value == true
		elseif self.Type == "string" and type(value) == "number" then
			value = tostring(value)
		end
		self.Value = value
		if changed then
			self.Changed = self.Changed + 1
			if not silent then
				Registry.NotifyChanged(self.Name, value, self)
			end
		end
		return self
	end

	function flag:Toggle()
		if type(self.Value) == "boolean" then
			self:Set(not self.Value)
		end
		return self.Value
	end

	function flag:Reset(silent)
		return self:Set(self.DefaultValue, silent)
	end

	table.insert(FlagOrder, flagName)
	Registry.Flags[flagName] = flag
	return flag
end

-- Создать флаг из GUI-элемента (общий хелпер)
function Registry.FromElement(element, options)
	options = options or {}
	local name = options.Flag or options.Name or element.Name
	if type(name) ~= "string" then
		return nil
	end
	local meta = Utils.clone(options)
	meta.Flag = name
	return Registry.Create(name, options.Default, meta)
end

function Registry.Get(flagName)
	local f = Registry.Flags[flagName]
	if f == nil then
		return nil
	end
	return f.Value
end

function Registry.Exists(flagName)
	return Registry.Flags[flagName] ~= nil
end

function Registry.Set(flagName, value, silent)
	local f = Registry.Flags[flagName]
	if f == nil then
		f = Registry.Create(flagName, value, { Auto = true })
		if f == nil then
			return false
		end
	end
	f:Set(value, silent)
	return true
end

function Registry.Toggle(flagName)
	local f = Registry.Flags[flagName]
	if f == nil then
		return false
	end
	f:Toggle()
	return f.Value
end

function Registry.Names()
	return Utils.clone(FlagOrder)
end

function Registry.Count()
	return Utils.count(Registry.Flags)
end

-- Подписка на изменение флага
function Registry.OnChanged(flagName, callback)
	if type(flagName) ~= "string" or type(callback) ~= "function" then
		return function() end
	end
	if FlagListeners[flagName] == nil then
		FlagListeners[flagName] = {}
	end
	local entry = callback
	table.insert(FlagListeners[flagName], entry)
	return function()
		local list = FlagListeners[flagName]
		if list == nil then
			return
		end
		for i = #list, 1, -1 do
			if list[i] == entry then
				table.remove(list, i)
			end
		end
	end
end

function Registry.NotifyChanged(flagName, value, flag)
	local list = FlagListeners[flagName]
	if list == nil then
		return
	end
	for _, fn in ipairs(Utils.clone(list)) do
		pcall(fn, value, flag)
	end
end

-- Массовая установка (используется при Load)
function Registry.Apply(tbl, silent)
	if type(tbl) ~= "table" then
		return 0
	end
	local applied = 0
	for name, value in pairs(tbl) do
		local flag = Registry.Flags[name]
		if flag ~= nil then
			flag:Set(value, silent)
			applied = applied + 1
		end
	end
	return applied
end

-- Снимок всех флагов (для сохранения)
function Registry.Snapshot()
	local out = {}
	for i, name in ipairs(FlagOrder) do
		local flag = Registry.Flags[name]
		if flag ~= nil and not flag.Meta.Hidden then
			local v = flag.Value
			if type(v) == "Color3" then
				out[name] = { __t = "Color3", v = Utils.toHex(v) }
			elseif type(v) == "table" then
				local items = {}
				for k, item in ipairs(v) do
					items[k] = tostring(item)
				end
				out[name] = { __t = "Table", v = items }
			else
				out[name] = v
			end
		end
	end
	return out
end

-- Дефолты (для Reset)
function Registry.Defaults()
	local out = {}
	for i, name in ipairs(FlagOrder) do
		local flag = Registry.Flags[name]
		if flag ~= nil and not flag.Meta.Hidden then
			local v = flag.DefaultValue
			if type(v) == "Color3" then
				out[name] = { __t = "Color3", v = Utils.toHex(v) }
			elseif type(v) == "table" then
				local items = {}
				for k, item in ipairs(v) do
					items[k] = tostring(item)
				end
				out[name] = { __t = "Table", v = items }
			else
				out[name] = v
			end
		end
	end
	return out
end

-- Сброс всех флагов к значениям по умолчанию
function Registry.ResetAll(silent)
	local n = 0
	for _, name in ipairs(FlagOrder) do
		local flag = Registry.Flags[name]
		if flag ~= nil then
			flag:Set(flag.DefaultValue, silent)
			n = n + 1
		end
	end
	return n
end

-- Реестр элементов (поиск/обновление)
function Registry.RegisterElement(element, meta)
	if type(element) ~= "table" then
		return
	end
	meta = meta or {}
	element.Meta = element.Meta or meta
	table.insert(Registry.Elements, element)
end

function Registry.ElementsOfType(kind)
	local out = {}
	for _, el in ipairs(Registry.Elements) do
		if el.Meta and el.Meta.Kind == kind then
			table.insert(out, el)
		end
	end
	return out
end

function Registry.FindElements(pattern)
	local out = {}
	for _, el in ipairs(Registry.Elements) do
		if el.Flag and el.Flag:find(pattern) then
			table.insert(out, el)
		end
	end
	return out
end

-- Применение значения флага к элементу интерфейса
-- (визуальная синхронизация при Load)
function Registry.RefreshElements()
	for _, el in ipairs(Registry.Elements) do
		if el.Flag and Registry.Flags[el.Flag] ~= nil then
			local value = Registry.Flags[el.Flag].Value
			if type(el.SetValue) == "function" then
				pcall(el.SetValue, el, value, true)
			end
		end
	end
end
-- ==== Config — сериализация, CRC32, сохранение/загрузка файла ====

local Config = {}
AshUI.Modules.Config = Config
AshUI.Config = Config

-- ==== JSON (собственный сериализатор — не зависит от HttpService) ====

local ESCAPES = {
	['"']  = '\\"',
	['\\'] = '\\\\',
	['\b'] = '\\b',
	['\f'] = '\\f',
	['\n'] = '\\n',
	['\r'] = '\\r',
	['\t'] = '\\t',
}

local function escapeString(str)
	return (str:gsub('[%c"\\]', function(c)
		return ESCAPES[c] or string.format("\\u%04x", string.byte(c))
	end))
end

local function isArray(tbl)
	local n = 0
	for k in pairs(tbl) do
		if type(k) ~= "number" then
			return false
		end
		n = n + 1
	end
	return n == #tbl
end

local function encodeValue(value, indent, level)
	indent = indent or ""
	level = level or 0
	local t = type(value)

	if value == nil then
		return "null"
	elseif t == "boolean" then
		return tostring(value)
	elseif t == "number" then
		if value ~= value or value == math.huge or value == -math.huge then
			return "null"
		end
		if value == math.floor(value) and math.abs(value) < 1e15 then
			return string.format("%d", value)
		end
		return string.format("%.10g", value)
	elseif t == "string" then
		return '"' .. escapeString(value) .. '"'
	elseif t == "table" then
		if getmetatable(value) ~= nil then
			-- Userdata/Color3 и прочее — в JSON не пишем
			return "null"
		end
		local nl = "\n" .. indent:rep(level + 1)
		local closeNl = "\n" .. indent:rep(level)
		if isArray(value) then
			if #value == 0 then
				return "[]"
			end
			local parts = {}
			for _, v in ipairs(value) do
				table.insert(parts, nl .. encodeValue(v, indent, level + 1))
			end
			return "[" .. table.concat(parts, ",") .. closeNl .. "]"
		end
		local keys = {}
		for k in pairs(value) do
			if type(k) == "string" or type(k) == "number" then
				table.insert(keys, k)
			end
		end
		if #keys == 0 then
			return "{}"
		end
		table.sort(keys, function(a, b)
			return tostring(a) < tostring(b)
		end)
		local parts = {}
		for _, k in ipairs(keys) do
			table.insert(parts, nl .. '"' .. escapeString(tostring(k)) .. '": ' .. encodeValue(value[k], indent, level + 1))
		end
		return "{" .. table.concat(parts, ",") .. closeNl .. "}"
	end
	return "null"
end

-- Кодирование объекта в JSON-строку
function Config.encode(value)
	local ok, str = pcall(encodeValue, value, "\t", 0)
	if not ok or type(str) ~= "string" then
		return "{}"
	end
	return str
end

-- Разбор JSON-строки (рекурсивный спуск)
function Config.decode(str)
	if type(str) ~= "string" or #str == 0 then
		return nil, "empty input"
	end
	local pos = 1
	local len = #str

	local function error_(msg)
		error(string.format("JSON error at %d: %s", pos, msg), 0)
	end

	local skipWhitespace = function()
		while pos <= len do
			local c = str:sub(pos, pos)
			if c == " " or c == "\t" or c == "\n" or c == "\r" then
				pos = pos + 1
			else
				break
			end
		end
	end

	local parseValue

	local function parseString()
		pos = pos + 1 -- пропускаем открывающую кавычку
		local buf = {}
		while pos <= len do
			local c = str:sub(pos, pos)
			if c == '"' then
				pos = pos + 1
				return table.concat(buf)
			elseif c == "\\" then
				local nxt = str:sub(pos + 1, pos + 1)
				if nxt == "n" then
					table.insert(buf, "\n")
				elseif nxt == "t" then
					table.insert(buf, "\t")
				elseif nxt == "r" then
					table.insert(buf, "\r")
				elseif nxt == "b" then
					table.insert(buf, "\b")
				elseif nxt == "f" then
					table.insert(buf, "\f")
				elseif nxt == "u" then
					local hex = str:sub(pos + 2, pos + 5)
					local code = tonumber(hex, 16) or 63
					table.insert(buf, utf8 and utf8.char and utf8.char(code) or "?")
					pos = pos + 4
				else
					table.insert(buf, nxt)
				end
				pos = pos + 2
			else
				table.insert(buf, c)
				pos = pos + 1
			end
		end
		error_("unterminated string")
	end

	local function parseNumber()
		local s, e = str:find("^-?%d+%.?%d*[eE]?[-+]?%d*", pos)
		if not s then
			error_("bad number")
		end
		local numStr = str:sub(s, e)
		pos = e + 1
		local num = tonumber(numStr)
		if num == nil then
			return 0
		end
		return num
	end

	local function parseArray()
		local arr = {}
		pos = pos + 1
		skipWhitespace()
		if str:sub(pos, pos) == "]" then
			pos = pos + 1
			return arr
		end
		while true do
			skipWhitespace()
			table.insert(arr, parseValue())
			skipWhitespace()
			local c = str:sub(pos, pos)
			if c == "," then
				pos = pos + 1
			elseif c == "]" then
				pos = pos + 1
				return arr
			else
				error_("expected , or ]")
			end
		end
	end

	local function parseObject()
		local obj = {}
		pos = pos + 1
		skipWhitespace()
		if str:sub(pos, pos) == "}" then
			pos = pos + 1
			return obj
		end
		while true do
			skipWhitespace()
			if str:sub(pos, pos) ~= '"' then
				error_("expected key")
			end
			local key = parseString()
			skipWhitespace()
			if str:sub(pos, pos) ~= ":" then
				error_("expected :")
			end
			pos = pos + 1
			skipWhitespace()
			obj[key] = parseValue()
			skipWhitespace()
			local c = str:sub(pos, pos)
			if c == "," then
				pos = pos + 1
			elseif c == "}" then
				pos = pos + 1
				return obj
			else
				error_("expected , or }")
			end
		end
	end

	parseValue = function()
		skipWhitespace()
		local c = str:sub(pos, pos)
		if c == "" then
			error_("unexpected end")
		elseif c == "{" then
			return parseObject()
		elseif c == "[" then
			return parseArray()
		elseif c == '"' then
			return parseString()
		elseif c == "t" and str:sub(pos, pos + 3) == "true" then
			pos = pos + 4
			return true
		elseif c == "f" and str:sub(pos, pos + 4) == "false" then
			pos = pos + 5
			return false
		elseif c == "n" and str:sub(pos, pos + 3) == "null" then
			pos = pos + 4
			return nil
		end
		return parseNumber()
	end

	local ok, result = pcall(function()
		local v = parseValue()
		skipWhitespace()
		return v
	end)
	if not ok then
		return nil, tostring(result)
	end
	if type(result) ~= "table" then
		return nil, "root is not an object"
	end
	return result
end

-- ==== ПУТЬ И ФАЙЛ ====

Config.LastError = nil
Config.LastSaveTime = 0
Config.LastPath = nil

function Config.getPath()
	if Config.LastPath then
		return Config.LastPath
	end
	return AshUI.getFolder() .. "/" .. OPTIONS.ConfigFile
end

-- Готова ли файловая система
function Config.fsAvailable()
	local wf = AshUI.env("writefile")
	local rf = AshUI.env("readfile")
	return type(wf) == "function" and type(rf) == "function"
end

-- Создаём папку (вложенные уровни)
function Config.ensureFolder(path)
	local mk = AshUI.env("makefolder")
	if type(mk) ~= "function" then
		return false
	end
	-- Разбиваем путь и создаём по уровням
	local acc = nil
	for part in tostring(path):gmatch("[^/]+") do
		acc = acc and (acc .. "/" .. part) or part
		pcall(mk, acc)
	end
	return true
end

-- Запись в файл (с фолбэком на file.store)
function Config.writeFile(path, content)
	local wf = AshUI.env("writefile")
	if type(wf) == "function" then
		local ok, err = pcall(wf, path, content)
		if ok then
			return true
		end
		Config.LastError = tostring(err)
	end
	-- Фолбэк: раздел file.writeable
	local okFile = false
	pcall(function()
		if type(file) == "table" and type(file.store) == "function" then
			file.store(path, content)
			okFile = true
		end
	end)
	if okFile then
		return true
	end
	return false
end

function Config.readFile(path)
	local rf = AshUI.env("readfile")
	if type(rf) == "function" then
		local ok, data = pcall(rf, path)
		if ok and type(data) == "string" then
			return data
		end
	end
	pcall(function()
		if type(file) == "table" and type(file.read) == "function" then
			local data = file.read(path)
			if type(data) == "string" then
				return data
			end
		end
	end)
	return nil
end

function Config.fileExists(path)
	local isf = AshUI.env("isfile")
	if type(isf) == "function" then
		local ok, res = pcall(isf, path)
		if ok and res == true then
			return true
		end
	end
	pcall(function()
		if type(file) == "table" and type(file.exists) == "function" then
			return file.exists(path) == true
		end
	end)
	return false
end

function Config.deleteFile(path)
	local df = AshUI.env("delfile")
	if type(df) == "function" then
		pcall(df, path)
	end
end

-- ==== СБОРКА / РАЗБОР ПАКЕТА КОНФИГА ====

-- Собрать данные конфига (флаги + настройки окна + тема)
function Config.collect()
	local data = {
		Meta = {
			Version	= OPTIONS.Version,
			Library	= OPTIONS.Name,
			Saved	= os.time(),
			CRC	= nil,
		},
		Flags	= Registry.Snapshot(),
		UI		= {
			Theme	= Theme.CurrentId(),
			Radius	= OPTIONS.Radius,
			Duration	= OPTIONS.Duration,
			Snap	= OPTIONS.Snap,
			ToggleKey = tostring(OPTIONS.ToggleKey.Name or "K"),
		},
	}
	return data
end

-- Применить данные конфига
function Config.apply(data, options)
	options = options or {}
	local applied = 0
	if type(data) ~= "table" then
		return 0
	end
	-- UI-настройки
	local ui = data.UI
	if type(ui) == "table" then
		if type(ui.Theme) == "string" and Theme.Exists(ui.Theme) then
			Theme.Set(ui.Theme, options.InstantTheme == true)
		end
		if type(ui.Radius) == "number" then
			OPTIONS.Radius = ui.Radius
		end
		if type(ui.Duration) == "number" then
			OPTIONS.Duration = ui.Duration
		end
		if type(ui.Snap) == "boolean" then
			OPTIONS.Snap = ui.Snap
		end
		if type(ui.ToggleKey) == "string" then
			local key = Enum.KeyCode[ui.ToggleKey]
			if key ~= nil then
				OPTIONS.ToggleKey = key
			end
		end
	end
	-- Флаги
	if type(data.Flags) == "table" then
		applied = Registry.Apply(data.Flags, true)
	end
	Registry.RefreshElements()
	return applied
end

-- CRC-обёртка: "PAYLOAD\n--CRC:XXXXXXXX"
function Config.pack(data)
	local payload = Config.encode(data)
	local crc = Utils.crc32(payload)
	return payload .. "\n--CRC:" .. crc, crc
end

-- Проверка целостности и разбор пакета
function Config.unpack(raw)
	if type(raw) ~= "string" then
		return nil, "нет данных"
	end
	local payload, crc = raw:match("^(.-)\n%-%-CRC:(%x+)%s*$")
	if payload == nil then
		-- Формат без CRC (старый конфиг) — принимаем как есть
		local data, err = Config.decode(raw)
		if data == nil then
			return nil, "битый JSON: " .. tostring(err)
		end
		return data, "без CRC"
	end
	local actual = Utils.crc32(payload)
	if actual ~= crc then
		return nil, string.format("CRC mismatch: %s != %s", actual, crc)
	end
	local data, err = Config.decode(payload)
	if data == nil then
		return nil, "битый JSON: " .. tostring(err)
	end
	return data, "ok"
end

-- ==== ПУБЛИЧНЫЕ ОПЕРАЦИИ ====

function Config.Save(silent)
	local data = Config.collect()
	local packed, crc = Config.pack(data)
	local path = Config.getPath()
	Config.ensureFolder(path:match("^(.*)/[^/]*$") or AshUI.getFolder())
	local ok = Config.writeFile(path, packed)
	if ok then
		Config.LastSaveTime = os.time()
		Config.LastError = nil
		if not silent then
			log("Конфиг сохранён:", path, "CRC", crc)
		end
		return true, crc
	end
	Config.LastError = "нет доступа на запись"
	if not silent then
		log("Не удалось сохранить конфиг:", Config.LastError)
	end
	return false
end

function Config.Load(options)
	options = options or {}
	local path = options.Path or Config.getPath()
	if not Config.fileExists(path) then
		return false, "файл не найден"
	end
	local raw = Config.readFile(path)
	if raw == nil then
		return false, "не удалось прочитать файл"
	end
	local data, status = Config.unpack(raw)
	if data == nil then
		Config.LastError = status
		return false, status
	end
	local applied = Config.apply(data, options)
	log("Конфиг загружен:", path, "статус", status, "флагов", applied)
	return true, applied, status
end

function Config.Reset()
	Registry.ResetAll(false)
	Theme.Set("Lunar")
	OPTIONS.Radius = DEFAULT_OPTIONS.Radius
	OPTIONS.Duration = DEFAULT_OPTIONS.Duration
	OPTIONS.Snap = DEFAULT_OPTIONS.Snap
	Registry.RefreshElements()
	return true
end

function Config.Delete()
	Config.deleteFile(Config.getPath())
	return true
end

-- Экспорт конфига в буфер обмена
function Config.Copy()
	local packed = Config.pack(Config.collect())
	local setclip = AshUI.env("setclipboard")
	if type(setclip) == "function" then
		local ok = pcall(setclip, packed)
		if ok then
			return true
		end
	end
	return false
end

-- Импорт конфига из буфера обмена
function Config.Paste()
	local getclip = AshUI.env("getclipboard")
	if type(getclip) ~= "function" then
		return false, "getclipboard недоступен"
	end
	local ok, raw = pcall(getclip)
	if not ok or type(raw) ~= "string" then
		return false, "пустой буфер"
	end
	local data, status = Config.unpack(raw)
	if data == nil then
		return false, status
	end
	local applied = Config.apply(data)
	return true, applied
end

-- Автосохранение по таймеру
function Config.StartAutoSave()
	if type(OPTIONS.AutoSave) ~= "number" or OPTIONS.AutoSave <= 0 then
		return
	end
	pcall(task.spawn, function()
		while AshUI.Loaded and Registry.Count() > 0 do
			task.wait(OPTIONS.AutoSave)
			if not AshUI.Loaded then
				break
			end
			-- Сохраняем только если что-то менялось
			pcall(Config.Save, true)
		end
	end)
end
-- ==== Notify — всплывающие уведомления (тосты) ====

local Notify = {}
AshUI.Modules.Notify = Notify
AshUI.Notify = Notify

-- Контейнер для тостов (создаётся лениво, привязан к ScreenGui)
Notify.Container = nil
Notify.Gui = nil
-- Можно отключить все уведомления (тумблер в Settings)
Notify.Enabled = true
-- Очередь активных тостов
Notify.Active = {}
-- Типы → цвета
local TOAST_STYLES = {
	Success	= {ColorKey = "Success", Icon = "✓", Label = "Success"},
	Error	= {ColorKey = "Error",   Icon = "✕", Label = "Error"},
	Warning	= {ColorKey = "Warning", Icon = "!",  Label = "Warning"},
	Info	= {ColorKey = "Info",    Icon = "i", Label = "Info"},
}

local ICON_SYMBOLS = {
	Success = "✓",
	Error   = "✕",
	Warning = "!",
	Info    = "i",
}

-- Получить/создать корневой ScreenGui и контейнер тостов
function Notify.Setup(screenGui)
	Notify.Gui = screenGui
	if screenGui == nil then
		return nil
	end
	if Notify.Container ~= nil and Notify.Container.Parent == screenGui then
		return Notify.Container
	end
	local holder = Utils.new("Frame", {
		Name = "NotifyContainer",
		BackgroundTransparency = 1,
		Size = UDim2.fromOffset(320, 400),
		Position = UDim2.new(1, -336, 0, 20),
		Parent = screenGui,
	})
	Utils.listLayout(holder, Enum.FillDirection.Vertical, 10, Enum.HorizontalAlignment.Right, Enum.VerticalAlignment.Top)
	-- Отступы
	Notify.Container = holder
	return holder
end

-- Сдвинуть контейнер в правый верхний угол (только при изменении разрешения)
local LastScreenSize = Vector2.zero
function Notify.Reposition()
	if Notify.Container == nil or Notify.Container.Parent == nil then
		return
	end
	local screen = Utils.screenSize()
	if screen == LastScreenSize then
		return
	end
	LastScreenSize = screen
	pcall(function()
		Notify.Container.Size = UDim2.fromOffset(310, math.max(screen.Y - 40, 200))
		Notify.Container.Position = UDim2.new(1, -326, 0, 20)
	end)
end

-- Создать тост
local function buildToast(style, title, message, options)
	options = options or {}
	local colors = TOAST_STYLES[style] or TOAST_STYLES.Info
	local accent = Theme.C(colors.ColorKey)
	local card = Utils.new("Frame", {
		Name = "Toast",
		BackgroundColor3 = Theme.C("Card"),
		BackgroundTransparency = 1,
		BorderSizePixel = 0,
		Size = UDim2.new(1, 0, 0, 78),
		Parent = Notify.Container,
	})
	Utils.corner(card, 12)
	Utils.padding(card, 12, 12, 12, 14)
	Theme.Bind("Card", card, "BackgroundColor3")

	-- Цветная полоска слева
	local bar = Utils.new("Frame", {
		BackgroundColor3 = accent,
		BorderSizePixel = 0,
		Size = UDim2.fromOffset(3, 44),
		Position = UDim2.new(0, 0, 0, 12),
		Parent = card,
	})
	Utils.corner(bar, 999)
	-- Мягкое свечение полоски
	local barGlow = Utils.new("ImageLabel", {
		BackgroundTransparency = 1,
		Size = UDim2.fromOffset(14, 54),
		Position = UDim2.new(0, -3, 0, 6),
		Image = "rbxassetid://6029598422",
		ImageColor3 = accent,
		ImageTransparency = 0.55,
		ZIndex = card.ZIndex,
		Parent = bar,
	})
	Utils.corner(barGlow, 999)

	-- Иконка
	local icon = Utils.new("TextLabel", {
		Text = ICON_SYMBOLS[colors.ColorKey] or "i",
		TextColor3 = accent,
		TextSize = 16,
		TextXAlignment = Enum.TextXAlignment.Center,
		BackgroundTransparency = 1,
		Size = UDim2.fromOffset(24, 24),
		Position = UDim2.new(0, 12, 0, 12),
		Parent = card,
	})
	Utils.corner(icon, 999)
	icon.BackgroundColor3 = accent
	icon.BackgroundTransparency = 0.82
	Utils.setFont(icon, "GothamBold")

	-- Заголовок
	local titleLabel = Utils.new("TextLabel", {
		Text = tostring(title or ""),
		TextColor3 = Theme.C("Text"),
		TextSize = 13,
		TextXAlignment = Enum.TextXAlignment.Left,
		TextTruncate = Enum.TextTruncate.AtEnd,
		BackgroundTransparency = 1,
		Size = UDim2.new(1, -52, 0, 18),
		Position = UDim2.new(0, 44, 0, 11),
		Parent = card,
	})
	Utils.setFont(titleLabel, "Gotham", "Bold")
	Theme.Bind("Text", titleLabel, "TextColor3")

	-- Сообщение
	local bodyLabel = Utils.new("TextLabel", {
		Text = tostring(message or ""),
		TextColor3 = Theme.C("TextDim"),
		TextSize = 12,
		TextWrapped = true,
		TextXAlignment = Enum.TextXAlignment.Left,
		TextYAlignment = Enum.TextYAlignment.Top,
		BackgroundTransparency = 1,
		Size = UDim2.new(1, -52, 0, 30),
		Position = UDim2.new(0, 44, 0, 30),
		Parent = card,
	})
	Utils.setFont(bodyLabel, "Gotham", "Regular")
	Theme.Bind("TextDim", bodyLabel, "TextColor3")

	-- Кнопка закрытия
	local close = Utils.new("TextButton", {
		Text = "✕",
		TextColor3 = Theme.C("TextFaint"),
		TextSize = 12,
		BackgroundTransparency = 1,
		Size = UDim2.fromOffset(20, 20),
		Position = UDim2.new(1, -8, 0, 6),
		AnchorPoint = Vector2.new(1, 0),
		Parent = card,
	})
	Utils.corner(close, 999)
	Utils.setFont(close, "Gotham", "Bold")
	Theme.Bind("TextFaint", close, "TextColor3")
	Motion.Hover(close, {
		Time = 0.12,
		From = {BackgroundTransparency = 1},
		To = {BackgroundTransparency = 0.85, TextColor3 = Theme.C("Text")},
		OnHover = function() end,
	})

	-- Прогресс-бар до закрытия
	local track = Utils.new("Frame", {
		BackgroundColor3 = Theme.C("Track"),
		BackgroundTransparency = 0.5,
		BorderSizePixel = 0,
		Size = UDim2.new(1, -28, 0, 2),
		Position = UDim2.new(0, 14, 1, -6),
		Parent = card,
	})
	Utils.corner(track, 999)
	local fill = Utils.new("Frame", {
		BackgroundColor3 = accent,
		BorderSizePixel = 0,
		Size = UDim2.fromScale(1, 1),
		Parent = track,
	})
	Utils.corner(fill, 999)

	return {
		Card		= card,
		Icon		= icon,
		Title		= titleLabel,
		Body		= bodyLabel,
		Close		= close,
		Fill		= fill,
		Style		= style,
		ColorKey	= colors.ColorKey,
		Time		= 0,
		Life		= options.Life or OPTIONS.ToastLife,
		Closing		= false,
	}
end

-- Закрыть тост с анимацией
function Notify.closeToast(toast, instant)
	if toast == nil or toast.Closing then
		return
	end
	toast.Closing = true
	if instant == true or toast.Card == nil then
		Utils.destroy(toast.Card)
		for i = #Notify.Active, 1, -1 do
			if Notify.Active[i] == toast then
				table.remove(Notify.Active, i)
			end
		end
		return
	end
	-- Ускоряем анимацию
	Motion.Tween({
		Instance = toast.Card,
		Time = 0.18,
		Easing = Easings.inQuad,
		BackgroundTransparency = 1,
		Size = UDim2.new(1, 0, 0, toast.Card.Size.Y.Offset - 6),
		OnComplete = function()
			Utils.destroy(toast.Card)
			for i = #Notify.Active, 1, -1 do
				if Notify.Active[i] == toast then
					table.remove(Notify.Active, i)
				end
			end
		end,
	})
	Motion.Tween({
		Instance = toast.Card,
		Time = 0.18,
		Easing = Easings.inQuad,
		Position = UDim2.new(1, 24, toast.Card.Position.Y.Scale, toast.Card.Position.Y.Offset),
	})
	for _, child in ipairs(toast.Card:GetChildren()) do
		if child:IsA("GuiObject") then
			Motion.Tween({ Instance = child, Time = 0.14, BackgroundTransparency = 1 })
			if child:IsA("TextLabel") or child:IsA("TextButton") then
				Motion.Tween({ Instance = child, Time = 0.14, TextTransparency = 1 })
			end
		end
	end
end

-- Публичный API: показать уведомление
-- Notify.Create({Title="...", Message="...", Style="Success", Life=4, OnClick=fn})
function Notify.Create(options)
	options = options or {}
	if Notify.Enabled == false then
		return nil
	end
	if Notify.Container == nil or Notify.Container.Parent == nil then
		if AshUI.CurrentGui ~= nil then
			Notify.Setup(AshUI.CurrentGui)
		end
	end
	if Notify.Container == nil then
		log("Нет контейнера тостов — уведомление пропущено")
		return nil
	end

	local style = options.Style or "Info"
	if TOAST_STYLES[style] == nil then
		style = "Info"
	end
	local title = options.Title or TOAST_STYLES[style].Label
	local message = options.Message or ""

	-- Ограничение очереди
	if #Notify.Active >= OPTIONS.MaxToasts then
		local oldest = Notify.Active[1]
		Notify.closeToast(oldest, true)
	end

	local toast = buildToast(style, title, message, options)
	-- кнопка закрытия
	toast.Close.MouseButton1Click:Connect(function()
		Notify.closeToast(toast)
	end)
	if options.OnClick then
		toast.Card.InputBegan:Connect(function(input)
			if input.UserInputType == Enum.UserInputType.MouseButton1 then
				pcall(options.OnClick)
				Notify.closeToast(toast)
			end
		end)
	end
	table.insert(Notify.Active, toast)

	-- Анимация появления
	Motion.Tween({
		Instance = toast.Card,
		Time = 0.26,
		Easing = Easings.outBack,
		BackgroundTransparency = 0,
		Size = UDim2.new(1, 0, 0, toast.Size),
		OnComplete = function()
			-- добавить «добивание» высоты после появления
		end,
	})
	-- Начальная высота 0
	pcall(function()
		toast.Card.Size = UDim2.new(1, 0, 0, 0)
	end)
	Motion.Tween({
		Instance = toast.Card,
		Time = 0.26,
		Easing = Easings.outBack,
		Size = UDim2.new(1, 0, 0, toast.Size),
	})
	Motion.Tween({
		Instance = toast.Card,
		Time = 0.3,
		Easing = Easings.outCubic,
		Position = UDim2.new(1, 0, toast.Card.Position.Y.Scale, toast.Card.Position.Y.Offset),
	})
	-- Иконка слегка «всплывает»
	Motion.Tween({
		Instance = toast.Icon,
		Time = 0.4,
		Easing = Easings.outElastic,
		BackgroundTransparency = 0.82,
		Size = UDim2.fromOffset(24, 24),
	})
	-- Полоса вырастает
	Motion.Tween({
		Instance = toast.Card,
		Time = 0.1,
		OnUpdate = function() end,
	})

	return toast
end

-- Короткие хелперы
function Notify.Success(title, message, life)
	return Notify.Create({Title = title, Message = message, Style = "Success", Life = life})
end

function Notify.Error(title, message, life)
	return Notify.Create({Title = title, Message = message, Style = "Error", Life = life})
end

function Notify.Warning(title, message, life)
	return Notify.Create({Title = title, Message = message, Style = "Warning", Life = life})
end

function Notify.Info(title, message, life)
	return Notify.Create({Title = title, Message = message, Style = "Info", Life = life})
end

-- Тик таймеров: обновляем прогресс-бар и убиваем истёкшие тосты
function Notify.Tick(dt)
	if #Notify.Active == 0 then
		return
	end
	local alive = {}
	for _, toast in ipairs(Notify.Active) do
		if toast.Closing or toast.Card == nil then
			continue
		end
		toast.Time = toast.Time + dt
		local life = math.max(toast.Life or OPTIONS.ToastLife, 0.1)
		local left = life - toast.Time
		if left <= 0 then
			Notify.closeToast(toast)
		else
			local p = Utils.clamp(left / life, 0, 1)
			pcall(function()
				toast.Fill.Size = UDim2.fromScale(p, 1)
			end)
			table.insert(alive, toast)
		end
	end
	Notify.Active = alive
end

-- Очистить все уведомления
function Notify.ClearAll()
	for _, toast in ipairs(Notify.Active) do
		Notify.closeToast(toast, true)
	end
	Notify.Active = {}
end
-- ==== Elements — базовые элементы интерфейса ====

local Elements = {}
AshUI.Modules.Elements = Elements
AshUI.Elements = Elements

-- Таблица debounce для кнопок (инстансы не принимают кастомных полей)
local ButtonDebounce = setmetatable({}, {__mode = "k"})

-- Режимы кнопок
local BUTTON_STYLES = {
	Default = {
		UseAccent	= false,
		UseDanger	= false,
	},
	Accent = {
		UseAccent	= true,
		UseDanger	= false,
	},
	Danger = {
		UseAccent	= false,
		UseDanger	= true,
	},
	Success = {
		UseAccent	= false,
		UseDanger	= false,
		Custom		= "Success",
	},
}

-- Общий конструктор элемента: карточка-контейнер с подписью
local function makeCard(parent, name, height, options)
	options = options or {}
	local card = Utils.new("Frame", {
		Name = name or "Element",
		BackgroundColor3 = options.Background or Theme.C("Card"),
		BackgroundTransparency = options.Transparency or 0,
		BorderSizePixel = 0,
		Size = UDim2.new(1, 0, 0, height or 40),
		Parent = parent,
	})
	if card ~= nil then
		card.LayoutOrder = options.LayoutOrder or Utils.nextOrder(parent)
		Utils.corner(card, OPTIONS.Radius - 2)
		Theme.Bind("Card", card, "BackgroundColor3")
	end
	return card
end
Elements.makeCard = makeCard

-- Подпись элемента (слева) + значение (справа)
local function makeLabel(card, text, size, color, extra)
	local l = Utils.new("TextLabel", {
		Text = tostring(text or ""),
		TextColor3 = color or Theme.C("Text"),
		TextSize = size or 13,
		TextXAlignment = Enum.TextXAlignment.Left,
		TextYAlignment = Enum.TextYAlignment.Center,
		BackgroundTransparency = 1,
		Size = UDim2.new(1, -(extra or 0), 1, 0),
		Parent = card,
	})
	Utils.setFont(l, "Gotham", "Medium")
	return l
end

-- Привязка элемента к флагу + синхронизация
local function attachFlag(element, options)
	if options.Flag == nil then
		return
	end
	local flag = Registry.Create(options.Flag, options.Default, {
		Min = options.Min,
		Max = options.Max,
		Step = options.Step,
		Options = options.Options,
		Save = options.Save,
	})
	element.Flag = options.Flag
	Registry.RegisterElement(element, {Kind = options.Kind or "Element"})
	-- Синхронизация при внешнем изменении (Load/Reset)
	Registry.OnChanged(options.Flag, function(value)
		if type(element.SetValue) == "function" and not element.SilentSet then
			pcall(element.SetValue, element, value, true)
		end
	end)
	if flag ~= nil and type(element.SetValue) == "function" then
		pcall(element.SetValue, element, flag.Value, true)
	end
end

-- Подсветка карточки при наведении
local function cardHover(card, radius)
	return Motion.Hover(card, {
		Time = 0.14,
		From = {BackgroundTransparency = 0},
		To = {BackgroundTransparency = 0.15},
	})
end
Elements.cardHover = cardHover

--==== ЭЛЕМЕНТ: CreateLabel ====

function Elements.CreateLabel(tab, text, options)
	options = options or {}
	local parent = tab.Scroll or tab.Content or tab
	local card = makeCard(parent, "Label", options.Height or 34, {
		Transparency = options.Background == nil and 1 or 0,
	})
	local label = makeLabel(card, text, options.Size or 13, options.Color or Theme.C(options.ColorKey or "Text"), -8)
	if options.Align then
		label.TextXAlignment = options.Align
	end
	if options.Wrap then
		label.TextWrapped = true
		label.TextYAlignment = Enum.TextYAlignment.Top
		label.Size = UDim2.new(1, -8, 1, 0)
	end

	local element = {
		Frame		= card,
		Label		= label,
		Meta		= {Kind = "Label"},
	}
	function element:SetValue(v, silent)
		label.Text = tostring(v)
	end
	function element:GetValue()
		return label.Text
	end
	element.Set = element.SetValue
	attachFlag(element, options)
	return element
end

-- ==== ЭЛЕМЕНТ: CreateButton ====

function Elements.CreateButton(tab, text, callback, options)
	options = options or {}
	local parent = tab.Scroll or tab.Content or tab
	local styleName = options.Style or "Default"
	local style = BUTTON_STYLES[styleName] or BUTTON_STYLES.Default

	local h = options.Height or 38
	local card = makeCard(parent, "Button", h)
	local normalColor = style.Custom and Theme.C(style.Custom)
		or (style.UseAccent and Theme.C("Accent")
		or (style.UseDanger and Theme.C("Error") or Theme.C("SurfaceAlt")))
	local hoverColor = style.Custom and Utils.lighten(Theme.C(style.Custom), 0.12)
		or (style.UseAccent and Utils.lighten(Theme.C("Accent"), 0.12)
		or (style.UseDanger and Utils.lighten(Theme.C("Error"), 0.12)
		or Theme.C("CardHover")))
	local textColor = (style.UseAccent or style.Custom) and Theme.C("AccentText") or Theme.C("Text")

	card.BackgroundColor3 = normalColor
	if style.UseAccent then
		Theme.Bind("Accent", card, "BackgroundColor3")
	elseif style.UseDanger then
		Theme.Bind("Error", card, "BackgroundColor3")
	elseif style.Custom then
		Theme.Bind(style.Custom, card, "BackgroundColor3")
	else
		Theme.Bind("SurfaceAlt", card, "BackgroundColor3")
	end

	-- Эффект блика
	if options.Shimmer ~= false then
		Anim.Shimmer(card, {Interval = options.ShimmerInterval or 5})
	end

	-- Эффект нажатия (масштаб)
	local press = Utils.new("TextButton", {
		Text = "",
		BackgroundTransparency = 1,
		Size = UDim2.fromScale(1, 1),
		AutoButtonColor = false,
		Parent = card,
	})
	Utils.corner(press, OPTIONS.Radius - 2)

	local label = makeLabel(card, text, options.Size or 13, textColor, 0)
	label.TextXAlignment = Enum.TextXAlignment.Center
	label.ZIndex = card.ZIndex + 1
	if style.UseAccent or style.Custom then
		Theme.Bind("AccentText", label, "TextColor3")
	else
		Theme.Bind("Text", label, "TextColor3")
	end
	label.ZIndex = press.ZIndex + 1

	-- Hover
	cardHover(card)
	-- Свечение при наведении
	local glowStroke = Utils.stroke(card, style.UseDanger and Theme.C("Error") or Theme.C("Accent"), 1, 1.5)
	Theme.Bind(style.UseDanger and "Error" or "Accent", glowStroke, "Color")

	local function setHover(hovered)
		Motion.Tween({
			Instance = card,
			Time = 0.16,
			Easing = Easings.outQuad,
			BackgroundColor3 = hovered and hoverColor or normalColor,
			OnComplete = function()
				-- возвращаем привязку темы после твина
			end,
		})
		Motion.Tween({
			Instance = glowStroke,
			Time = 0.16,
			Easing = Easings.outQuad,
			Transparency = hovered and 0.25 or 1,
		})
		Motion.Tween({
			Instance = label,
			Time = 0.16,
			TextColor3 = hovered and Utils.lighten(textColor, 0.2) or textColor,
		})
	end

	press.MouseEnter:Connect(function() setHover(true) end)
	press.MouseLeave:Connect(function() setHover(false) end)

	local isDown = false
	press.MouseButton1Down:Connect(function()
		isDown = true
		Motion.Tween({ Instance = card, Time = 0.08, Easing = Easings.outQuad, BackgroundTransparency = 0.16 })
	end)
	local function release()
		if not isDown then
			return
		end
		isDown = false
		Motion.Tween({ Instance = card, Time = 0.2, Easing = Easings.outCubic, BackgroundTransparency = 0 })
	end
	press.MouseButton1Up:Connect(release)
	press.MouseButton1Leave:Connect(release)

	press.MouseButton1Click:Connect(function()
		if options.Debounce then
			-- отсекаем повторные клики (состояние храним в side-таблице)
			if ButtonDebounce[press] and os.clock() - ButtonDebounce[press] < options.Debounce then
				return
			end
			ButtonDebounce[press] = os.clock()
		end
		if type(callback) == "function" then
			local ok, err = pcall(callback)
			if not ok then
				log("Ошибка в callback кнопки:", err)
			end
		end
		if options.Click == true then
			-- визуальный отклик
		end
		-- Волна расходящихся кругов
		if options.Ripple ~= false then
			local size = Utils.absoluteSize(card)
			local ripple = Utils.new("Frame", {
				BackgroundColor3 = Color3.new(1, 1, 1),
				BackgroundTransparency = 0.82,
				BorderSizePixel = 0,
				Size = UDim2.fromOffset(size.Y, size.Y),
				Position = UDim2.fromOffset(Utils.mousePos().X - Utils.absolutePos(card).X - size.Y / 2, Utils.mousePos().Y - Utils.absolutePos(card).Y - size.Y / 2),
				ZIndex = card.ZIndex + 2,
				Parent = card,
			})
			Utils.corner(ripple, 999)
			Motion.Tween({
				Instance = ripple,
				Time = 0.45,
				Easing = Easings.outCubic,
				Size = UDim2.fromOffset(size.X * 1.4, size.Y * 1.4),
				BackgroundTransparency = 1,
				OnComplete = function()
					Utils.destroy(ripple)
				end,
			})
		end
	end)

	local element = {
		Frame		= card,
		Press		= press,
		Label		= label,
		Meta		= {Kind = "Button", Style = styleName},
		Callback	= callback,
	}
	function element:SetText(v)
		label.Text = tostring(v)
	end
	function element:SetCallback(fn)
		element.Callback = fn
	end
	function element:Press2() end
	function element:SetValue(v, silent) end
	Registry.RegisterElement(element, {Kind = "Button"})
	return element
end

-- ==== ЭЛЕМЕНТ: CreateToggle ====

function Elements.CreateToggle(tab, text, default, callback, options)
	options = options or {}
	local parent = tab.Scroll or tab.Content or tab
	default = default == true

	local card = makeCard(parent, "Toggle", 40)
	local label = makeLabel(card, text, 13, Theme.C("Text"), -56)
	Theme.Bind("Text", label, "TextColor3")

	-- Дорожка
	local track = Utils.new("Frame", {
		BackgroundColor3 = Theme.C("Track"),
		BorderSizePixel = 0,
		Size = UDim2.fromOffset(42, 22),
		Position = UDim2.new(1, -52, 0.5, -11),
		Parent = card,
	})
	Utils.corner(track, 999)
	Theme.Bind("Track", track, "BackgroundColor3")

	-- Ползунок
	local knob = Utils.new("Frame", {
		BackgroundColor3 = Theme.C("TextDim"),
		BorderSizePixel = 0,
		Size = UDim2.fromOffset(16, 16),
		Position = UDim2.fromOffset(3, 3),
		Parent = track,
	})
	Utils.corner(knob, 999)
	Theme.Bind("TextDim", knob, "BackgroundColor3")
	local knobGlow = Utils.new("ImageLabel", {
		BackgroundTransparency = 1,
		Size = UDim2.fromOffset(26, 26),
		Position = UDim2.fromOffset(-5, -5),
		Image = "rbxassetid://6029598422",
		ImageColor3 = Theme.C("TextDim"),
		ImageTransparency = 0.7,
		Parent = knob,
	})
	Utils.corner(knobGlow, 999)

	-- Клик-зона
	local hit = Utils.new("TextButton", {
		Text = "",
		BackgroundTransparency = 1,
		Size = UDim2.fromScale(1, 1),
		AutoButtonColor = false,
		Parent = card,
	})
	Utils.corner(hit, OPTIONS.Radius - 2)

	local state = {
		Enabled	= default,
		Animating = false,
	}

	local function paint(enabled, instant)
		local bgOn = Theme.C(options.ColorKey or "Accent")
		local targetBg = enabled and bgOn or Theme.C("Track")
		local knobX = enabled and 42 - 16 - 3 or 3
		local knobCol = enabled and (options.ColorKey and Theme.C(options.ColorKey) or Theme.C("Accent")) or Theme.C("TextDim")
		local dur = instant and 0.001 or (options.Duration or 0.18)
		Motion.Tween({
			Instance = track,
			Time = dur,
			Easing = Easings.outQuad,
			BackgroundColor3 = targetBg,
		})
		Motion.Tween({
			Instance = knob,
			Time = dur,
			Easing = Easings.outBack,
			Position = UDim2.fromOffset(knobX, 3),
			BackgroundColor3 = knobCol,
		})
		Motion.Tween({
			Instance = knobGlow,
			Time = dur * 1.4,
			ImageTransparency = enabled and 0.45 or 0.75,
			ImageColor3 = knobCol,
		})
		if enabled then
			-- Свечение дорожки
			Motion.Tween({
				Instance = track,
				Time = dur * 1.5,
				BackgroundTransparency = enabled and 0.1 or 0,
			})
		else
			Motion.Tween({ Instance = track, Time = dur, BackgroundTransparency = 0 })
		end
		if not instant then
			pcall(Anim.PulseFrame, knob, {Color = knobCol, To = Utils.lighten(knobCol, 0.4), Time = 0.6})
		end
	end

	hit.MouseButton1Click:Connect(function()
		state.Enabled = not state.Enabled
		paint(state.Enabled)
		if options.Flag then
			Registry.Set(options.Flag, state.Enabled)
		end
		if type(callback) == "function" then
			local ok, err = pcall(callback, state.Enabled)
			if not ok then
				log("Ошибка в callback тумблера:", err)
			end
		end
	end)

	-- Hover на всей карточке
	cardHover(card)

	local element = {
		Frame		= card,
		Track		= track,
		Knob		= knob,
		Label		= label,
		Meta		= {Kind = "Toggle"},
	}
	function element:GetValue()
		return state.Enabled
	end
	function element:SetValue(v, silent)
		state.SilentSet = silent == true
		state.Enabled = v == true
		paint(state.Enabled, silent == true)
		state.SilentSet = nil
		if not silent and type(callback) == "function" then
			pcall(callback, state.Enabled)
		end
	end
	function element:SetText(v)
		label.Text = tostring(v)
	end
	element.Toggle = function()
		state.Enabled = not state.Enabled
		paint(state.Enabled)
		if options.Flag then
			Registry.Set(options.Flag, state.Enabled)
		end
		return state.Enabled
	end
	-- применить начальное
	paint(default, true)
	attachFlag(element, {Flag = options.Flag, Default = default, Kind = "Toggle"})
	return element
end

-- ==== ЭЛЕМЕНТ: CreateSlider ====

function Elements.CreateSlider(tab, text, minv, maxv, default, suffix, callback, options)
	options = options or {}
	local parent = tab.Scroll or tab.Content or tab
	minv = minv or 0
	maxv = maxv or 100
	default = Utils.clamp(default or minv, minv, maxv)
	suffix = suffix or ""
	local decimals = options.Decimals or 0

	local card = makeCard(parent, "Slider", 58)
	local label = makeLabel(card, text, 13, Theme.C("Text"), -70)
	Theme.Bind("Text", label, "TextColor3")

	local valueLabel = Utils.new("TextLabel", {
		Text = "",
		TextColor3 = Theme.C("Accent"),
		TextSize = 13,
		BackgroundTransparency = 1,
		Size = UDim2.fromOffset(64, 20),
		Position = UDim2.new(1, -8, 0, 4),
		AnchorPoint = Vector2.new(1, 0),
		TextXAlignment = Enum.TextXAlignment.Right,
		Parent = card,
	})
	Utils.setFont(valueLabel, "Gotham", "Bold")
	Theme.Bind("Accent", valueLabel, "TextColor3")

	-- Дорожка
	local trackY = 38
	local track = Utils.new("Frame", {
		BackgroundColor3 = Theme.C("Track"),
		BorderSizePixel = 0,
		Size = UDim2.new(1, -16, 0, 6),
		Position = UDim2.new(0, 8, 0, trackY),
		Parent = card,
	})
	Utils.corner(track, 999)
	Theme.Bind("Track", track, "BackgroundColor3")

	-- Заполненная часть
	local fill = Utils.new("Frame", {
		BackgroundColor3 = Theme.C("Accent"),
		BorderSizePixel = 0,
		Size = UDim2.fromScale(0.5, 1),
		Parent = track,
	})
	Utils.corner(fill, 999)
	Theme.Bind(options.ColorKey or "Accent", fill, "BackgroundColor3")

	-- Ползунок
	local knob = Utils.new("Frame", {
		BackgroundColor3 = Theme.C("Accent"),
		BorderSizePixel = 0,
		Size = UDim2.fromOffset(16, 16),
		Position = UDim2.fromScale(0.5, 0.5),
		AnchorPoint = Vector2.new(0.5, 0.5),
		Parent = track,
	})
	Utils.corner(knob, 999)
	Theme.Bind(options.ColorKey or "Accent", knob, "BackgroundColor3")
	local knobGlow = Utils.new("ImageLabel", {
		BackgroundTransparency = 1,
		Size = UDim2.fromOffset(34, 34),
		Position = UDim2.fromOffset(-9, -9),
		Image = "rbxassetid://6029598422",
		ImageColor3 = Theme.C("Accent"),
		ImageTransparency = 0.6,
		Parent = knob,
	})
	Utils.corner(knobGlow, 999)

	-- Зона захвата (расширенная)
	local hit = Utils.new("TextButton", {
		Text = "",
		BackgroundTransparency = 1,
		Size = UDim2.new(1, 0, 1, -trackY + 14),
		Position = UDim2.new(0, 0, 0, trackY - 8),
		AutoButtonColor = false,
		Parent = card,
	})

	local dragging = false
	local current = default
	local targetValue = default

	local function format(v)
		if decimals > 0 then
			return string.format("%." .. decimals .. "f%s", v, suffix)
		end
		return Utils.formatNumber(v) .. suffix
	end

	local function applyVisual(animated)
		local p = Utils.inverseLerp(minv, maxv, current)
		p = Utils.clamp(p, 0, 1)
		if animated then
			Motion.Tween({
				Instance = fill,
				Time = 0.14,
				Easing = Easings.outQuad,
				Size = UDim2.fromScale(p, 1),
			})
			Motion.Tween({
				Instance = knob,
				Time = 0.14,
				Easing = Easings.outBack,
				Position = UDim2.fromScale(p, 0.5),
			})
		else
			pcall(function()
				fill.Size = UDim2.fromScale(p, 1)
				knob.Position = UDim2.fromScale(p, 0.5)
			end)
		end
		Motion.Tween({
			Instance = valueLabel,
			Time = 0.1,
			TextTransparency = 0.35,
			OnComplete = function()
				Motion.Tween({ Instance = valueLabel, Time = 0.2, TextTransparency = 0 })
			end,
		})
		valueLabel.Text = format(current)
	end

	local function setFromX(x)
		local pos = Utils.absolutePos(track)
		local size = Utils.absoluteSize(track)
		if size.X <= 0 then
			return
		end
		local p = Utils.clamp((x - pos.X) / size.X, 0, 1)
		local v = Utils.remap(p, 0, 1, minv, maxv)
		if options.Step then
			v = Utils.round(v, options.Step)
		end
		v = Utils.clamp(v, minv, maxv)
		if v ~= current then
			current = v
			applyVisual(true)
			if options.Flag then
				Registry.Set(options.Flag, current)
			end
			if type(callback) == "function" then
				local ok, err = pcall(callback, current)
				if not ok then
					log("Ошибка в callback слайдера:", err)
				end
			end
		end
	end

	hit.MouseButton1Down:Connect(function()
		dragging = true
		setFromX(Utils.mousePos().X)
		Motion.Tween({ Instance = knob, Time = 0.15, Size = UDim2.fromOffset(19, 19) })
		Motion.Tween({ Instance = knobGlow, Time = 0.15, ImageTransparency = 0.35 })
	end)
	hit.MouseButton1Up:Connect(function()
		dragging = false
		Motion.Tween({ Instance = knob, Time = 0.2, Easing = Easings.outBack, Size = UDim2.fromOffset(16, 16) })
		Motion.Tween({ Instance = knob, Time = 0.2, Easing = Easings.outBack, Position = UDim2.fromScale(Utils.inverseLerp(minv, maxv, current), 0.5) })
		Motion.Tween({ Instance = knobGlow, Time = 0.3, ImageTransparency = 0.6 })
	end)
	hit.MouseButton1Leave:Connect(function()
		if dragging then
			dragging = false
		end
	end)

	-- Колесо мыши над слайдером (шаг всегда работает, протяжка — только при зажатой кнопке)
	hit.InputChanged:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseMovement then
			if dragging then
				setFromX(Utils.mousePos().X)
			end
		elseif input.UserInputType == Enum.UserInputType.MouseWheel then
			local stepSize = (maxv - minv) / 40
			current = Utils.clamp(current - input.Position.Z * stepSize, minv, maxv)
			if options.Step then
				current = Utils.round(current, options.Step)
			end
			applyVisual(true)
			if options.Flag then
				Registry.Set(options.Flag, current)
			end
			if type(callback) == "function" then
				pcall(callback, current)
			end
		end
	end)

	local element = {
		Frame		= card,
		Track		= track,
		Fill		= fill,
		Knob		= knob,
		Label		= label,
		ValueLabel	= valueLabel,
		Min			= minv,
		Max			= maxv,
		Suffix		= suffix,
		Meta		= {Kind = "Slider"},
	}
	function element:GetValue()
		return current
	end
	function element:SetValue(v, silent)
		v = Utils.clamp(tonumber(v) or minv, minv, maxv)
		if options.Step then
			v = Utils.round(v, options.Step)
		end
		targetValue = v
		if silent then
			current = v
			applyVisual(false)
		else
			current = v
			applyVisual(true)
			if type(callback) == "function" then
				pcall(callback, v)
			end
		end
	end
	function element:SetText(v)
		label.Text = tostring(v)
	end
	function element:SetMinMax(newMin, newMax)
		minv, maxv = newMin, newMax
		element:SetValue(current, true)
	end
	current = targetValue
	applyVisual(false)
	attachFlag(element, {Flag = options.Flag, Default = default, Min = minv, Max = maxv, Step = options.Step, Kind = "Slider"})
	return element
end
-- ==== ЭЛЕМЕНТ: CreateProgress ====

function Elements.CreateProgress(tab, text, default, options)
	options = options or {}
	local parent = tab.Scroll or tab.Content or tab
	default = Utils.clamp(default or 0, 0, 1)

	local height = options.Height or 56
	local card = makeCard(parent, "Progress", height)
	local label = makeLabel(card, text, 13, Theme.C("Text"), -60)
	Theme.Bind("Text", label, "TextColor3")

	local percentLabel = Utils.new("TextLabel", {
		Text = "0%",
		TextColor3 = Theme.C("Accent"),
		TextSize = 12,
		BackgroundTransparency = 1,
		Size = UDim2.fromOffset(50, 18),
		Position = UDim2.new(1, -6, 0, 3),
		AnchorPoint = Vector2.new(1, 0),
		TextXAlignment = Enum.TextXAlignment.Right,
		Parent = card,
	})
	Utils.setFont(percentLabel, "Gotham", "Bold")
	Theme.Bind(options.ColorKey or "Accent", percentLabel, "TextColor3")

	local trackY = 30
	local track = Utils.new("Frame", {
		BackgroundColor3 = Theme.C("Track"),
		BorderSizePixel = 0,
		Size = UDim2.new(1, -16, 0, options.BarHeight or 8),
		Position = UDim2.new(0, 8, 0, trackY),
		Parent = card,
	})
	Utils.corner(track, 999)
	Theme.Bind("Track", track, "BackgroundColor3")

	local fill = Utils.new("Frame", {
		BackgroundColor3 = Theme.C(options.ColorKey or "Accent"),
		BorderSizePixel = 0,
		Size = UDim2.fromScale(default, 1),
		Parent = track,
	})
	Utils.corner(fill, 999)
	Theme.Bind(options.ColorKey or "Accent", fill, "BackgroundColor3")
	-- Градиент поверх заливки для «живого» вида
	if options.Gradient ~= false then
		local grad = Utils.gradient(fill, Utils.lighten(Theme.C(options.ColorKey or "Accent"), 0.2), Theme.C(options.ColorKey or "Accent"), 0)
		grad.Transparency = NumberSequence.new({
			NumberSequenceKeypoint.new(0, 0.35),
			NumberSequenceKeypoint.new(0.5, 0.0),
			NumberSequenceKeypoint.new(1, 0.3),
		})
	end
	-- Свечение на конце полосы
	local head = Utils.new("ImageLabel", {
		BackgroundTransparency = 1,
		Size = UDim2.fromOffset(28, 28),
		Position = UDim2.new(1, -14, 0.5, -14),
		Image = "rbxassetid://6029598422",
		ImageColor3 = Theme.C(options.ColorKey or "Accent"),
		ImageTransparency = 0.5,
		Parent = fill,
	})
	Utils.corner(head, 999)

	-- Блик, пробегающий по заполненной части (если “живой” режим)
	local shine = Utils.new("Frame", {
		BackgroundColor3 = Color3.new(1, 1, 1),
		BackgroundTransparency = 0.86,
		BorderSizePixel = 0,
		Size = UDim2.new(0.35, 0, 1, 0),
		Parent = fill,
	})
	Utils.corner(shine, 999)

	local progress = default
	local shown = default
	local paused = false

	Motion.OnUpdate(function(dt)
		-- Блик
		if not paused and progress > 0.02 then
			local w = track.AbsoluteSize.X
			local t = (Motion.Uptime() * 0.35) % 1.4
			if t < 1 then
				pcall(function()
					shine.Position = UDim2.fromOffset((t - 0.3) * w, 0)
					shine.BackgroundTransparency = 0.86
				end)
			else
				pcall(function()
					shine.Position = UDim2.fromOffset(-w, 0)
					shine.BackgroundTransparency = 1
				end)
			end
		end
		-- Плавное «догоняющее» значение
		if not Utils.isNear(shown, progress, 0.0005) then
			shown = Utils.damp(shown, progress, 12, dt)
			pcall(function()
				fill.Size = UDim2.fromScale(Utils.clamp(shown, 0, 1), 1)
				percentLabel.Text = tostring(math.floor(shown * 100 + 0.5)) .. "%"
			end)
		end
	end, 6)

	local element = {
		Frame		= card,
		Track		= track,
		Fill		= fill,
		Label		= label,
		PercentLabel	= percentLabel,
		Meta		= {Kind = "Progress"},
	}
	function element:GetValue()
		return progress
	end
	function element:SetValue(v, silent)
		v = Utils.clamp(tonumber(v) or 0, 0, 1)
		progress = v
		if silent then
			shown = v
			pcall(function()
				fill.Size = UDim2.fromScale(v, 1)
				percentLabel.Text = tostring(math.floor(v * 100 + 0.5)) .. "%"
			end)
		end
	end
	-- Плавная установка (с анимацией)
	function element:Set(v, duration)
		v = Utils.clamp(tonumber(v) or 0, 0, 1)
		element:GetValue()
		progress = v
		Motion.Tween({
			Instance = nil,
			Time = duration or 0.5,
			Easing = Easings.outCubic,
			OnUpdate = function(t)
				shown = Utils.lerp(element._from or 0, v, t)
			end,
		})
	end
	function element:SetText(v)
		label.Text = tostring(v)
	end
	function element:Pause()
		paused = true
	end
	function element:Resume()
		paused = false
	end
	attachFlag(element, {Flag = options.Flag, Default = default, Min = 0, Max = 1, Kind = "Progress"})
	pcall(function()
		percentLabel.Text = tostring(math.floor(default * 100 + 0.5)) .. "%"
		fill.Size = UDim2.fromScale(default, 1)
	end)
	return element
end

-- ==== ЭЛЕМЕНТ: CreateParagraph ====

function Elements.CreateParagraph(tab, text, options)
	options = options or {}
	local parent = tab.Scroll or tab.Content or tab
	local card = makeCard(parent, "Paragraph", options.Height or 70, {
		Background = options.Background or Theme.C("Card"),
	})
	local label = Utils.new("TextLabel", {
		Text = tostring(text or ""),
		TextColor3 = Theme.C(options.ColorKey or "TextDim"),
		TextSize = options.Size or 12,
		TextWrapped = true,
		TextXAlignment = options.Align or Enum.TextXAlignment.Left,
		TextYAlignment = Enum.TextYAlignment.Top,
		BackgroundTransparency = 1,
		Size = UDim2.new(1, -24, 1, -20),
		Position = UDim2.fromOffset(12, 10),
		RichText = true,
		Parent = card,
	})
	Utils.setFont(label, options.Font or "Gotham", options.Weight or "Regular")
	Theme.Bind(options.ColorKey or "TextDim", label, "TextColor3")
	cardHover(card)

	local element = {
		Frame		= card,
		Label		= label,
		Meta		= {Kind = "Paragraph"},
	}
	function element:SetValue(v)
		label.Text = tostring(v)
	end
	function element:SetText(v)
		label.Text = tostring(v)
	end
	function element:GetValue()
		return label.Text
	end
	attachFlag(element, {Flag = options.Flag, Default = text, Kind = "Paragraph"})
	return element
end

-- ==== ЭЛЕМЕНТ: CreateSpacer ====

function Elements.CreateSpacer(tab, height, options)
	options = options or {}
	local parent = tab.Scroll or tab.Content or tab
	if options.Divider == true then
		local holder = Anim.CurvedLine(parent, Theme.C("BorderSoft"), 1, 1)
		holder.LayoutOrder = (options.LayoutOrder or 9999)
		return {
			Frame	= holder,
			Meta	= {Kind = "Spacer", Divider = true},
			SetValue = function() end,
			GetValue = function() return height or 8 end,
		}
	end
	local h = height or 8
	local spacer = Utils.new("Frame", {
		Name = "Spacer",
		BackgroundTransparency = 1,
		Size = UDim2.new(1, 0, 0, h),
		Parent = parent,
	})
	return {
		Frame		= spacer,
		Meta		= {Kind = "Spacer"},
		SetValue	= function() end,
		GetValue	= function() return h end,
	}
end

-- ==== ЭЛЕМЕНТ: CreateStepper ====

function Elements.CreateStepper(tab, text, default, minv, maxv, callback, options)
	options = options or {}
	local parent = tab.Scroll or tab.Content or tab
	default = default or 0
	minv = minv or 0
	maxv = maxv or 100

	local card = makeCard(parent, "Stepper", 40)
	local label = makeLabel(card, text, 13, Theme.C("Text"), -110)
	Theme.Bind("Text", label, "TextColor3")

	local valueLabel = Utils.new("TextLabel", {
		Text = tostring(default),
		TextColor3 = Theme.C("Accent"),
		TextSize = 13,
		BackgroundTransparency = 1,
		Size = UDim2.fromOffset(48, 24),
		Position = UDim2.new(1, -84, 0.5, -12),
		TextXAlignment = Enum.TextXAlignment.Center,
		Parent = card,
	})
	Utils.setFont(valueLabel, "Gotham", "Bold")
	Theme.Bind(options.ColorKey or "Accent", valueLabel, "TextColor3")

	local function makeStepBtn(glyph, posX, flip)
		local b = Utils.new("TextButton", {
			Text = glyph,
			TextColor3 = Theme.C("Text"),
			TextSize = 15,
			BackgroundColor3 = Theme.C("SurfaceAlt"),
			BackgroundTransparency = 0,
			BorderSizePixel = 0,
			Size = UDim2.fromOffset(24, 24),
			Position = UDim2.new(1, posX, 0.5, -12),
			AutoButtonColor = false,
			Parent = card,
		})
		Utils.corner(b, 8)
		Utils.setFont(b, "Gotham", "Bold")
		Theme.Bind("SurfaceAlt", b, "BackgroundColor3")
		Theme.Bind("Text", b, "TextColor3")
		b.MouseEnter:Connect(function()
			Motion.Tween({ Instance = b, Time = 0.12, BackgroundColor3 = Theme.C("CardHover"), BackgroundTransparency = 0 })
		end)
		b.MouseLeave:Connect(function()
			Motion.Tween({ Instance = b, Time = 0.16, BackgroundColor3 = Theme.C("SurfaceAlt") })
		end)
		b.MouseButton1Down:Connect(function()
			Motion.Tween({ Instance = b, Time = 0.08, BackgroundTransparency = 0.25 })
		end)
		b.MouseButton1Up:Connect(function()
			Motion.Tween({ Instance = b, Time = 0.16, BackgroundTransparency = 0 })
		end)
		if flip then
			b.Rotation = 0
		end
		return b
	end

	local minus = makeStepBtn("−", -54, false)
	local plus = makeStepBtn("+", -10, false)

	local value = default
	local step = options.Step or 1

	local function commit(v, silent)
		v = Utils.clamp(v, minv, maxv)
		if step then
			v = Utils.round(v, step)
		end
		local changed = v ~= value
		value = v
		valueLabel.Text = options.Format and options.Format(v) or Utils.formatNumber(v)
		-- анимация «пульса» числа
		Motion.Tween({
			Instance = valueLabel,
			Time = 0.12,
			TextTransparency = 0.5,
			OnComplete = function()
				Motion.Tween({ Instance = valueLabel, Time = 0.2, TextTransparency = 0 })
			end,
		})
		if changed and not silent then
			if options.Flag then
				Registry.Set(options.Flag, value)
			end
			if type(callback) == "function" then
				local ok, err = pcall(callback, value)
				if not ok then
					log("Ошибка в callback счётчика:", err)
				end
			end
		end
	end

	minus.MouseButton1Click:Connect(function()
		commit(value - (options.Increment or step))
	end)
	plus.MouseButton1Click:Connect(function()
		commit(value + (options.Increment or step))
	end)

	cardHover(card)

	local element = {
		Frame		= card,
		Label		= label,
		ValueLabel	= valueLabel,
		Minus		= minus,
		Plus		= plus,
		Min			= minv,
		Max			= maxv,
		Meta		= {Kind = "Stepper"},
	}
	function element:GetValue()
		return value
	end
	function element:SetValue(v, silent)
		commit(tonumber(v) or 0, silent)
	end
	function element:SetText(v)
		label.Text = tostring(v)
	end
	function element:Step(dir)
		commit(value + dir * (options.Increment or step))
	end
	attachFlag(element, {Flag = options.Flag, Default = default, Min = minv, Max = maxv, Step = step, Kind = "Stepper"})
	valueLabel.Text = options.Format and options.Format(default) or Utils.formatNumber(default)
	return element
end

-- ==== ЭЛЕМЕНТ: CreateTextbox ====

function Elements.CreateTextbox(tab, text, placeholder, default, callback, options)
	options = options or {}
	local parent = tab.Scroll or tab.Content or tab
	default = default or ""

	local card = makeCard(parent, "Textbox", 62)
	local label = Utils.new("TextLabel", {
		Text = tostring(text or ""),
		TextColor3 = Theme.C("Text"),
		TextSize = 13,
		BackgroundTransparency = 1,
		Size = UDim2.new(1, -16, 0, 18),
		Position = UDim2.fromOffset(8, 5),
		TextXAlignment = Enum.TextXAlignment.Left,
		Parent = card,
	})
	Utils.setFont(label, "Gotham", "Medium")
	Theme.Bind("Text", label, "TextColor3")

	local box = Utils.new("TextBox", {
		Text = tostring(default),
		PlaceholderText = tostring(placeholder or ""),
		TextColor3 = Theme.C("Text"),
		PlaceholderColor3 = Theme.C("TextFaint"),
		TextSize = 12,
		BackgroundColor3 = Theme.C("SurfaceDeep"),
		BackgroundTransparency = 0,
		BorderSizePixel = 0,
		ClearTextOnFocus = false,
		MultiLine = options.MultiLine == true,
		TextXAlignment = options.Align or Enum.TextXAlignment.Left,
		Size = UDim2.new(1, -16, 0, 30),
		Position = UDim2.fromOffset(8, 25),
		Parent = card,
	})
	Utils.corner(box, 9)
	Utils.padding(box, 4, 10, 4, 10)
	Utils.setFont(box, options.Font or "Gotham", "Regular")
	Theme.Bind("SurfaceDeep", box, "BackgroundColor3")
	Theme.Bind("Text", box, "TextColor3")
	Theme.Bind("TextFaint", box, "PlaceholderColor3")

	local boxStroke = Utils.stroke(box, Theme.C("Accent"), 1, 1.4)
	Theme.Bind("Accent", boxStroke, "Color")

	local focusStroke = Utils.stroke(box, Theme.C("Border"), 0.6, 1)
	Theme.Bind("Border", focusStroke, "Color")

	local value = tostring(default)

	local function commit(text2, silent)
		value = tostring(text2 or "")
		if not silent and type(callback) == "function" then
			local ok, err = pcall(callback, value)
			if not ok then
				log("Ошибка в callback текстового поля:", err)
			end
		end
		if options.Flag and not silent then
			Registry.Set(options.Flag, value)
		end
	end

	box.FocusLost:Connect(function(enterPressed)
		pcall(function()
			box.ReleaseFocus()
		end)
		commit(box.Text)
		if options.Enter == true and enterPressed and type(callback) == "function" then
			pcall(callback, value, true)
		end
		Motion.Tween({ Instance = boxStroke, Time = 0.2, Transparency = 1 })
		Motion.Tween({ Instance = focusStroke, Time = 0.2, Transparency = 0.6 })
		Motion.Tween({ Instance = box, Time = 0.2, BackgroundColor3 = Theme.C("SurfaceDeep") })
	end)
	box.Focused:Connect(function()
		Motion.Tween({ Instance = boxStroke, Time = 0.18, Easing = Easings.outQuad, Transparency = 0.2 })
		Motion.Tween({ Instance = focusStroke, Time = 0.18, Transparency = 1 })
		Motion.Tween({ Instance = box, Time = 0.18, Easing = Easings.outQuad, BackgroundColor3 = Theme.C("SurfaceAlt") })
		-- свечение фокуса
		Motion.Tween({ Instance = box, Time = 0.25, BackgroundTransparency = 0 })
	end)
	if options.Lossless ~= true then
		box:GetPropertyChangedSignal("Text"):Connect(function()
			commit(box.Text)
		end)
	end

	cardHover(card)

	local element = {
		Frame		= card,
		Box			= box,
		Label		= label,
		Meta		= {Kind = "Textbox"},
	}
	function element:GetValue()
		return box.Text
	end
	function element:SetValue(v, silent)
		v = tostring(v or "")
		pcall(function()
			box.Text = v
		end)
		commit(v, silent)
	end
	function element:SetPlaceholder(v)
		box.PlaceholderText = tostring(v)
	end
	function element:SetText(v)
		label.Text = tostring(v)
	end
	function element:Clear()
		pcall(function()
			box.Text = ""
		end)
		commit("")
	end
	function element:Focus()
		pcall(function()
			box:CaptureFocus()
		end)
	end
	attachFlag(element, {Flag = options.Flag, Default = default, Kind = "Textbox"})
	return element
end

-- ==== ЭЛЕМЕНТ: CreateKeybind ====

function Elements.CreateKeybind(tab, text, default, callback, options)
	options = options or {}
	local parent = tab.Scroll or tab.Content or tab
	local card = makeCard(parent, "Keybind", 40)
	local label = makeLabel(card, text, 13, Theme.C("Text"), -110)
	Theme.Bind("Text", label, "TextColor3")

	local keyLabel = Utils.new("TextButton", {
		Text = default and tostring(default.Name) or "—",
		TextColor3 = Theme.C("Accent"),
		TextSize = 12,
		BackgroundColor3 = Theme.C("SurfaceDeep"),
		BackgroundTransparency = 0,
		BorderSizePixel = 0,
		Size = UDim2.fromOffset(96, 26),
		Position = UDim2.new(1, -8, 0.5, -13),
		AnchorPoint = Vector2.new(1, 0),
		AutoButtonColor = false,
		Parent = card,
	})
	Utils.corner(keyLabel, 9)
	Utils.setFont(keyLabel, "Gotham", "Bold")
	Theme.Bind("Accent", keyLabel, "TextColor3")
	Theme.Bind("SurfaceDeep", keyLabel, "BackgroundColor3")

	local binding = default
	local listening = false
	local uis = Services.UserInputService

	local function prettyName(key)
		if type(key) ~= "userdata" and type(key) ~= "table" then
			return tostring(key or "—")
		end
		return key.Name
	end

	local function setBinding(key, silent)
		binding = key
		keyLabel.Text = prettyName(key)
		if not silent then
			if options.Flag then
				Registry.Set(options.Flag, prettyName(key))
			end
			if type(callback) == "function" then
				local ok, err = pcall(callback, key)
				if not ok then
					log("Ошибка в callback keybind:", err)
				end
			end
		end
	end

	-- Соединение слушателя ввода (объявлено до функции-обработчика)
	local listenConn = nil

	local function inputListener(input)
		if not listening then
			return
		end
		if input.UserInputType == Enum.UserInputType.Keyboard then
			local valid = true
			-- служебные клавиши игнорируем
			for _, k in ipairs({"LeftControl", "RightControl", "LeftShift", "RightShift", "LeftAlt", "RightAlt"}) do
				if input.KeyCode == Enum.KeyCode[k] then
					valid = false
				end
			end
			if input.KeyCode == Enum.KeyCode.Unknown then
				valid = false
			end
			if valid then
				setBinding(input.KeyCode)
				pcall(function()
					keyLabel:ReleaseFocus()
				end)
			end
		elseif input.UserInputType == Enum.UserInputType.MouseButton1
			or input.UserInputType == Enum.UserInputType.MouseButton2
			or input.UserInputType == Enum.UserInputType.MouseButton3 then
			setBinding(input.UserInputType)
			pcall(function()
				keyLabel:ReleaseFocus()
			end)
		end
		listening = false
		-- Отписываем слушатель ввода (не кнопку!)
		if listenConn ~= nil then
			pcall(function()
				listenConn:Disconnect()
			end)
			listenConn = nil
		end
		Motion.Tween({ Instance = keyLabel, Time = 0.2, BackgroundColor3 = Theme.C("SurfaceDeep") })
		Motion.Tween({ Instance = keyLabel, Time = 0.2, TextColor3 = Theme.C("Accent") })
		keyLabel.Text = prettyName(binding)
	end

	local function startListening()
		if listening or uis == nil then
			return
		end
		listening = true
		keyLabel.Text = "..."
		keyLabel.TextColor3 = Theme.C("Warning")
		Motion.Tween({ Instance = keyLabel, Time = 0.2, BackgroundColor3 = Theme.C("Warning"), TextTransparency = 0.1 })
		-- мерцание, пока слушаем
		Motion.Pulse(keyLabel, {Property = "BackgroundTransparency", From = 0.05, To = 0.35, Time = 0.6})
		listenConn = uis.InputBegan:Connect(inputListener)
	end

	keyLabel.MouseButton1Click:Connect(startListening)
	keyLabel.MouseEnter:Connect(function()
		if not listening then
			Motion.Tween({ Instance = keyLabel, Time = 0.14, BackgroundColor3 = Theme.C("CardHover") })
		end
	end)
	keyLabel.MouseLeave:Connect(function()
		if not listening then
			Motion.Tween({ Instance = keyLabel, Time = 0.18, BackgroundColor3 = Theme.C("SurfaceDeep") })
		end
	end)
	cardHover(card)

	local element = {
		Frame		= card,
		KeyLabel	= keyLabel,
		Label		= label,
		Meta		= {Kind = "Keybind"},
	}
	function element:GetValue()
		return prettyName(binding)
	end
	function element:GetKey()
		return binding
	end
	function element:SetValue(v, silent)
		-- Принимаем строку (из конфига) или KeyCode
		if type(v) == "string" then
			local k = Enum.KeyCode[v]
			setBinding(k, silent)
		elseif type(v) == "userdata" or type(v) == "table" then
			setBinding(v, silent)
		end
	end
	function element:SetText(v)
		label.Text = tostring(v)
	end
	function element:Listen()
		startListening()
	end
	attachFlag(element, {Flag = options.Flag, Default = prettyName(default), Kind = "Keybind"})
	return element
end

-- ==== ЭЛЕМЕНТ: CreateDropdown ====

function Elements.CreateDropdown(tab, text, options, default, callback, opts)
	options = options or {}
	opts = opts or {}
	local parent = tab.Scroll or tab.Content or tab
	local optionList = options.Options or opts.Options or {}

	local card = makeCard(parent, "Dropdown", 40)
	local label = makeLabel(card, text, 13, Theme.C("Text"), -110)
	Theme.Bind("Text", label, "TextColor3")

	local selected = nil
	local selectedIndex = nil

	local valueLabel = Utils.new("TextButton", {
		Text = default and tostring(default) or "Выберите...",
		TextColor3 = default and Theme.C("Text") or Theme.C("TextFaint"),
		TextSize = 12,
		BackgroundColor3 = Theme.C("SurfaceDeep"),
		BackgroundTransparency = 0,
		BorderSizePixel = 0,
		Size = UDim2.fromOffset(120, 26),
		Position = UDim2.new(1, -8, 0.5, -13),
		AnchorPoint = Vector2.new(1, 0),
		AutoButtonColor = false,
		TextXAlignment = Enum.TextXAlignment.Left,
		TextTruncate = Enum.TextTruncate.AtEnd,
		Parent = card,
	})
	Utils.corner(valueLabel, 9)
	Utils.padding(valueLabel, 0, 22, 0, 10)
	Utils.setFont(valueLabel, "Gotham", "SemiBold")
	Theme.Bind("SurfaceDeep", valueLabel, "BackgroundColor3")
	Theme.Bind("TextFaint", valueLabel, "TextColor3")

	-- Стрелочка
	local arrow = Utils.new("TextLabel", {
		Text = "▾",
		TextColor3 = Theme.C("TextDim"),
		TextSize = 12,
		BackgroundTransparency = 1,
		Size = UDim2.fromOffset(16, 16),
		Position = UDim2.new(1, -4, 0.5, -8),
		AnchorPoint = Vector2.new(1, 0),
		Parent = valueLabel,
	})
	Utils.setFont(arrow, "Gotham", "Bold")
	Theme.Bind("TextDim", arrow, "TextColor3")

	-- Выпадающий список
	local dropdown = Utils.new("Frame", {
		Name = "DropdownList",
		BackgroundColor3 = Theme.C("SurfaceAlt"),
		BorderSizePixel = 0,
		Size = UDim2.new(1, -16, 0, 0),
		Position = UDim2.fromOffset(8, 42),
		Visible = false,
		ClipsDescendants = true,
		ZIndex = card.ZIndex + 2,
		Parent = card,
	})
	Utils.corner(dropdown, OPTIONS.Radius - 2)
	Utils.stroke(dropdown, Theme.C("Border"), 0.5, 1)
	Theme.Bind("SurfaceAlt", dropdown, "BackgroundColor3")
	local ddList = Utils.listLayout(dropdown, Enum.FillDirection.Vertical, 2, Enum.HorizontalAlignment.Left, Enum.VerticalAlignment.Top)
	Utils.padding(dropdown, 4, 4, 4, 4)
	dropdown.AutomaticSize = Enum.AutomaticSize.Y

	-- Построить строки
	local rows = {}
	local element = nil
	local rowConns = {}
	local function disconnectRows()
		for i = #rowConns, 1, -1 do
			pcall(function()
				rowConns[i]:Disconnect()
			end)
			table.remove(rowConns, i)
		end
	end
	local function rebuildOptions()
		disconnectRows()
		Utils.clear(dropdown)
		Utils.clear(dropdown)
		rows = {}
		ddList = Utils.listLayout(dropdown, Enum.FillDirection.Vertical, 2)
		Utils.padding(dropdown, 4, 4, 4, 4)
		for i, opt in ipairs(optionList) do
			local optLabel = Utils.optionLabel(opt, i)
			local optValue = Utils.optionValue(opt, i)
			local row = Utils.new("TextButton", {
				Text = "  " .. tostring(optLabel),
				TextColor3 = Theme.C("Text"),
				TextSize = 12,
				TextXAlignment = Enum.TextXAlignment.Left,
				BackgroundColor3 = Theme.C("SurfaceAlt"),
				BackgroundTransparency = 1,
				BorderSizePixel = 0,
				Size = UDim2.new(1, -8, 0, 26),
				AutoButtonColor = false,
				ZIndex = dropdown.ZIndex + 1,
				Parent = dropdown,
			})
			Utils.corner(row, 8)
			Utils.setFont(row, "Gotham", "Medium")
			Theme.Bind("Text", row, "TextColor3")
			-- галочка выбора
			local check = Utils.new("TextLabel", {
				Text = "✓",
				TextColor3 = Theme.C("Accent"),
				TextSize = 11,
				BackgroundTransparency = 1,
				Size = UDim2.fromOffset(14, 14),
				Position = UDim2.new(0, 6, 0.5, -7),
				Visible = false,
				Parent = row,
			})
			Utils.setFont(check, "Gotham", "Bold")
			Theme.Bind("Accent", check, "TextColor3")
			rows[i] = {Row = row, Check = check, Value = optValue, Label = optLabel}

			table.insert(rowConns, row.MouseEnter:Connect(function()
				Motion.Tween({ Instance = row, Time = 0.12, BackgroundTransparency = 0, BackgroundColor3 = Theme.C("CardHover") })
			end))
			table.insert(rowConns, row.MouseLeave:Connect(function()
				Motion.Tween({ Instance = row, Time = 0.16, BackgroundTransparency = 1, BackgroundColor3 = Theme.C("SurfaceAlt") })
			end))
			table.insert(rowConns, row.MouseButton1Click:Connect(function()
				element:SetValue(optValue)
				if type(callback) == "function" then
					local ok, err = pcall(callback, optValue, i)
					if not ok then
						log("Ошибка в callback dropdown:", err)
					end
				end
				element:Close()
			end))
		end
		-- подсветить выбранное
		if selectedIndex ~= nil and rows[selectedIndex] ~= nil then
			rows[selectedIndex].Check.Visible = true
		end
	end

	local isOpen = false
	element = {
		Frame		= card,
		ValueLabel	= valueLabel,
		Dropdown	= dropdown,
		Rows		= rows,
		Meta		= {Kind = "Dropdown"},
	}

	function element:IsOpen()
		return isOpen
	end

	function element:Open()
		if isOpen or #optionList == 0 then
			return
		end
		isOpen = true
		pcall(function()
			dropdown.Visible = true
			dropdown.Size = UDim2.new(1, -16, 0, 0)
			dropdown.BackgroundTransparency = 1
		end)
		-- целевая высота
		local targetH = #optionList * 28 + 8
		Motion.Tween({
			Instance = dropdown,
			Time = 0.2,
			Easing = Easings.outCubic,
			Size = UDim2.new(1, -16, 0, targetH),
			BackgroundTransparency = 0,
		})
		-- строки с задержкой
		for i, r in ipairs(rows) do
			Motion.Reveal(r.Row, {Time = 0.2, Delay = i * 0.03, Dy = -6})
		end
		arrow.Rotation = 180
		Motion.Tween({ Instance = arrow, Time = 0.2, Easing = Easings.outBack, Rotation = 180 })
		-- увеличиваем карточку, чтобы список не обрезался
		pcall(function()
			dropdown.ZIndex = card.ZIndex + 5
		end)
		Motion.Tween({ Instance = card, Time = 0.2, Size = UDim2.new(1, 0, 0, 40 + targetH) })
	end

	function element:Close()
		if not isOpen then
			return
		end
		isOpen = false
		Motion.Tween({
			Instance = dropdown,
			Time = 0.15,
			Easing = Easings.inQuad,
			Size = UDim2.new(1, -16, 0, 0),
			BackgroundTransparency = 1,
			OnComplete = function()
				pcall(function()
					dropdown.Visible = false
				end)
			end,
		})
		Motion.Tween({ Instance = card, Time = 0.18, Size = UDim2.new(1, 0, 0, 40) })
		Motion.Tween({ Instance = arrow, Time = 0.2, Easing = Easings.outBack, Rotation = 0 })
	end

	valueLabel.MouseButton1Click:Connect(function()
		if isOpen then
			element:Close()
		else
			element:Open()
		end
	end)
	valueLabel.MouseEnter:Connect(function()
		Motion.Tween({ Instance = valueLabel, Time = 0.14, BackgroundColor3 = Theme.C("CardHover") })
	end)
	valueLabel.MouseLeave:Connect(function()
		Motion.Tween({ Instance = valueLabel, Time = 0.18, BackgroundColor3 = Theme.C("SurfaceDeep") })
	end)
	cardHover(card)

	function element:GetValue()
		return selected
	end
	function element:SetValue(v, silent)
		local idx = nil
		for i, opt in ipairs(optionList) do
			if Utils.optionValue(opt, i) == v then
				idx = i
				break
			end
		end
		selectedIndex = idx
		if idx ~= nil then
			selected = Utils.optionValue(optionList[idx], idx)
			valueLabel.Text = tostring(Utils.optionLabel(optionList[idx], idx))
			valueLabel.TextColor3 = Theme.C("Text")
		else
			selected = v
			valueLabel.Text = tostring(v or "Выберите...")
			valueLabel.TextColor3 = Theme.C("Text")
		end
		for i, r in ipairs(rows) do
			if r ~= nil then
				r.Check.Visible = (i == idx)
			end
		end
		if not silent and options.Flag then
			Registry.Set(options.Flag, selected)
		end
	end
	function element:SetOptions(newOptions)
		optionList = newOptions or {}
		selectedIndex = nil
		rebuildOptions()
	end
	function element:SetText(v)
		label.Text = tostring(v)
	end
	attachFlag(element, {Flag = options.Flag, Default = default, Kind = "Dropdown"})
	rebuildOptions()
	-- установить начальное значение
	if default ~= nil then
		element:SetValue(default, true)
	end
	return element
end

-- ==== ЭЛЕМЕНТ: CreateSection ====

function Elements.CreateSection(tab, text, options)
	options = options or {}
	local parent = tab.Scroll or tab.Content or tab

	local holder = Utils.new("Frame", {
		Name = "Section",
		BackgroundTransparency = 1,
		Size = UDim2.new(1, 0, 0, 34),
		Parent = parent,
	})

	-- Заголовок
	local titleLabel = Utils.new("TextLabel", {
		Text = tostring(text or ""),
		TextColor3 = Theme.C("TextDim"),
		TextSize = 11,
		TextXAlignment = Enum.TextXAlignment.Left,
		BackgroundTransparency = 1,
		Size = UDim2.new(1, 0, 0, 18),
		Position = UDim2.fromOffset(4, 0),
		Parent = holder,
	})
	Utils.setFont(titleLabel, "Gotham", "Bold")
	Theme.Bind("TextDim", titleLabel, "TextColor3")

	-- Линия после заголовка
	local line = Utils.new("Frame", {
		BackgroundColor3 = Theme.C("BorderSoft"),
		BorderSizePixel = 0,
		Size = UDim2.new(1, -8, 0, 1),
		Position = UDim2.new(1, 0, 0, 13),
		AnchorPoint = Vector2.new(1, 0),
		Parent = holder,
	})
	Utils.corner(line, 999)
	Theme.Bind("BorderSoft", line, "BackgroundColor3")

	-- Контейнер для содержимого раздела
	local body = Utils.new("Frame", {
		Name = "SectionBody",
		BackgroundTransparency = 1,
		Size = UDim2.new(1, 0, 0, 0),
		Position = UDim2.fromOffset(0, 26),
		Parent = holder,
	})
	local bodyLayout = Utils.listLayout(body, Enum.FillDirection.Vertical, options.Gap or 8)
	local bodySize = Utils.sizeConstraint(body, 0, 10000)

	-- Обновление высоты body по AutomaticSize
	body.AutomaticSize = Enum.AutomaticSize.Y

	-- Скрываем line если раздел пустой
	if options.NoLine then
		line.Visible = false
	end

	local section = {
		Frame		= holder,
		Title		= titleLabel,
		Body		= body,
		Line		= line,
		Meta		= {Kind = "Section"},
	}
	function section:GetBody()
		return body
	end
	function section:SetTitle(v)
		titleLabel.Text = tostring(v)
	end
	function section:SetValue() end
	-- Reveal
	if options.Reveal ~= false then
		Motion.Reveal(titleLabel, {Time = 0.2})
	end
	return section
end
-- ==== ElementsAdvanced — продвинутые элементы ====

local Advanced = {}
AshUI.Modules.ElementsAdvanced = Advanced
AshUI.ElementsAdvanced = Advanced

-- Ссылка на базовые хелперы
local function makeCard(parent, name, height, options)
	return Elements.makeCard(parent, name, height, options)
end
local function cardHover(card)
	return Elements.cardHover(card)
end

-- Привязка флага для цвета (объявлена до использования)
local function attachColorFlag(element, flagName, default)
	if flagName == nil or element == nil then
		return
	end
	local flag = Registry.Create(flagName, default, {Kind = "ColorPicker"})
	element.Flag = flagName
	Registry.RegisterElement(element, {Kind = "ColorPicker"})
	Registry.OnChanged(flagName, function(value)
		if type(value) == "string" then
			value = Utils.fromHex(value)
		end
		if type(value) == "Color3" then
			element:SetValue(value, true)
		end
	end)
end

-- ==== ЭЛЕМЕНТ: CreateColorPicker ====

function Advanced.CreateColorPicker(tab, text, default, callback, options)
	options = options or {}
	local parent = tab.Scroll or tab.Content or tab
	default = default or Color3.new(1, 1, 1)

	local card = Utils.new("Frame", {
		Name = "ColorPicker",
		BackgroundColor3 = Theme.C("Card"),
		BorderSizePixel = 0,
		Size = UDim2.new(1, 0, 0, 132),
		Parent = parent,
	})
	Utils.corner(card, OPTIONS.Radius - 2)
	Theme.Bind("Card", card, "BackgroundColor3")

	local color = default
	local hue, sat, val = Utils.rgbToHsv(color)

	-- Заголовок + swatch
	local titleLabel = Utils.new("TextLabel", {
		Text = tostring(text or "Цвет"),
		TextColor3 = Theme.C("Text"),
		TextSize = 13,
		BackgroundTransparency = 1,
		Size = UDim2.new(1, -70, 0, 18),
		Position = UDim2.fromOffset(10, 6),
		TextXAlignment = Enum.TextXAlignment.Left,
		Parent = card,
	})
	Utils.setFont(titleLabel, "Gotham", "Medium")
	Theme.Bind("Text", titleLabel, "TextColor3")

	local hexLabel = Utils.new("TextLabel", {
		Text = "#" .. Utils.toHex(color),
		TextColor3 = Theme.C("TextFaint"),
		TextSize = 10,
		BackgroundTransparency = 1,
		Size = UDim2.fromOffset(72, 18),
		Position = UDim2.new(1, -10, 0, 6),
		AnchorPoint = Vector2.new(1, 0),
		TextXAlignment = Enum.TextXAlignment.Right,
		Parent = card,
	})
	Utils.setFont(hexLabel, "Code")
	Theme.Bind("TextFaint", hexLabel, "TextColor3")

	-- Палитра (SV-плоскость)
	local paletteTop = 26
	local paletteH = 58
	local palette = Utils.new("Frame", {
		BackgroundColor3 = Color3.new(1, 1, 1),
		BorderSizePixel = 0,
		Size = UDim2.new(1, -20, 0, paletteH),
		Position = UDim2.fromOffset(10, paletteTop),
		ClipsDescendants = true,
		Parent = card,
	})
	Utils.corner(palette, 9)
	-- Горизонтальный градиент: белый → цвет тона
	local whiteGrad = Utils.gradient(palette, Color3.new(1, 1, 1), Utils.hsv(hue, 1, 1), 0)
	-- Вертикальный градиент: прозрачный → чёрный
	local blackGrad = Utils.new("UIGradient", {
		Color = ColorSequence.new(Color3.new(0, 0, 0), Color3.new(0, 0, 0)),
		Rotation = 90,
		Transparency = NumberSequence.new({
			NumberSequenceKeypoint.new(0, 0),
			NumberSequenceKeypoint.new(1, 1),
		}),
		Parent = palette,
	})
	local paletteStroke = Utils.stroke(palette, Theme.C("Border"), 0.4, 1)
	Theme.Bind("Border", paletteStroke, "Color")

	-- Курсор выбора цвета
	local selector = Utils.new("Frame", {
		BackgroundTransparency = 1,
		Size = UDim2.fromOffset(14, 14),
		AnchorPoint = Vector2.new(0.5, 0.5),
		Position = UDim2.fromScale(0.5, 0.5),
		Parent = palette,
	})
	Utils.corner(selector, 999)
	Utils.stroke(selector, Color3.new(1, 1, 1), 0.1, 2)
	local selectorInner = Utils.new("Frame", {
		BackgroundColor3 = color,
		BorderSizePixel = 0,
		Size = UDim2.fromScale(1, 1),
		Parent = selector,
	})
	Utils.corner(selectorInner, 999)

	-- Полоса тона (Hue)
	local hueTop = paletteTop + paletteH + 10
	local hueBar = Utils.new("Frame", {
		BackgroundColor3 = Color3.new(1, 1, 1),
		BorderSizePixel = 0,
		Size = UDim2.new(1, -20, 0, 12),
		Position = UDim2.fromOffset(10, hueTop),
		ClipsDescendants = true,
		Parent = card,
	})
	Utils.corner(hueBar, 999)
	local hueGrad = Utils.new("UIGradient", {
		Color = ColorSequence.new({
			ColorSequenceKeypoint.new(0, Color3.fromHSV(0, 1, 1)),
			ColorSequenceKeypoint.new(1 / 6, Color3.fromHSV(1 / 6, 1, 1)),
			ColorSequenceKeypoint.new(2 / 6, Color3.fromHSV(2 / 6, 1, 1)),
			ColorSequenceKeypoint.new(3 / 6, Color3.fromHSV(3 / 6, 1, 1)),
			ColorSequenceKeypoint.new(4 / 6, Color3.fromHSV(4 / 6, 1, 1)),
			ColorSequenceKeypoint.new(5 / 6, Color3.fromHSV(5 / 6, 1, 1)),
			ColorSequenceKeypoint.new(1, Color3.fromHSV(1, 1, 1)),
		}),
		Parent = hueBar,
	})
	local hueSelector = Utils.new("Frame", {
		BackgroundColor3 = Color3.new(1, 1, 1),
		BorderSizePixel = 0,
		Size = UDim2.fromOffset(10, 16),
		AnchorPoint = Vector2.new(0.5, 0.5),
		Position = UDim2.fromScale(hue / 360, 0.5),
		Parent = hueBar,
	})
	Utils.corner(hueSelector, 999)
	Utils.stroke(hueSelector, Color3.new(0, 0, 0), 0.3, 1.5)

	-- Ряд пресетов + кнопка сброса
	local presetY = hueTop + 20
	local presets = {}
	local presetColors = {
		Color3.fromRGB(226, 232, 244), Color3.fromRGB(120, 170, 240), Color3.fromRGB(96, 200, 145),
		Color3.fromRGB(232, 190, 88), Color3.fromRGB(226, 96, 108), Color3.fromRGB(170, 132, 240),
		Color3.fromRGB(90, 210, 200), Color3.fromRGB(246, 146, 92),
	}
	for i, c in ipairs(presetColors) do
		local sw = Utils.new("TextButton", {
			BackgroundColor3 = c,
			BorderSizePixel = 0,
			Size = UDim2.fromOffset(16, 16),
			Position = UDim2.fromOffset(10 + (i - 1) * 20, presetY),
			AutoButtonColor = false,
			Parent = card,
		})
		Utils.corner(sw, 999)
		Utils.stroke(sw, Color3.new(0, 0, 0), 0.4, 1)
		presets[i] = sw
		sw.MouseButton1Click:Connect(function()
			element:SetValue(c)
			if type(callback) == "function" then
				pcall(callback, c)
			end
		end)
		sw.MouseEnter:Connect(function()
			Motion.Tween({ Instance = sw, Time = 0.12, Size = UDim2.fromOffset(19, 19) })
		end)
		sw.MouseLeave:Connect(function()
			Motion.Tween({ Instance = sw, Time = 0.16, Easing = Easings.outBack, Size = UDim2.fromOffset(16, 16) })
		end)
	end

	-- Кнопка «копировать hex»
	local copyBtn = Utils.new("TextButton", {
		Text = "COPY",
		TextColor3 = Theme.C("TextDim"),
		TextSize = 9,
		BackgroundColor3 = Theme.C("SurfaceAlt"),
		BorderSizePixel = 0,
		Size = UDim2.fromOffset(40, 16),
		Position = UDim2.new(1, -10, 0, presetY),
		AnchorPoint = Vector2.new(1, 0),
		AutoButtonColor = false,
		Parent = card,
	})
	Utils.corner(copyBtn, 6)
	Utils.setFont(copyBtn, "Gotham", "Bold")
	Theme.Bind("SurfaceAlt", copyBtn, "BackgroundColor3")
	copyBtn.MouseButton1Click:Connect(function()
		local setclip = AshUI.env("setclipboard")
		if type(setclip) == "function" then
			pcall(setclip, Utils.toHex(color))
			Notify.Success("Скопировано", "#" .. Utils.toHex(color), 2.5)
		else
			Notify.Warning("Нет clipboard", "setclipboard недоступен", 3)
		end
	end)

	-- Логика выбора цвета
	local function updateFromPointer(target, hueBarMode)
		local mouse = Utils.mousePos()
		local pos = Utils.absolutePos(target)
		local size = Utils.absoluteSize(target)
		if size.X <= 0 then
			return
		end
		local p = Utils.clamp((mouse.X - pos.X) / size.X, 0, 1)
		if hueBarMode then
			hue = p * 360
			sat, val = sat or 1, val or 1
			-- сохраняем насыщенность/яркость
			local nv = Utils.hsv(hue, sat, val)
			whiteGrad.Color = ColorSequence.new(Color3.new(1, 1, 1), Utils.hsv(hue, 1, 1))
			pcall(function()
				hueSelector.Position = UDim2.fromScale(p, 0.5)
			end)
			color = nv
		else
			sat = p
			local p2 = Utils.clamp((mouse.Y - pos.Y) / size.Y, 0, 1)
			val = 1 - p2
			color = Utils.hsv(hue, sat, val)
			pcall(function()
				selector.Position = UDim2.fromScale(sat, 1 - val)
			end)
		end
		pcall(function()
			selectorInner.BackgroundColor3 = color
		end)
		hexLabel.Text = "#" .. Utils.toHex(color)
	end

	local function makeInteractive(target, isHue)
		target.Active = true
		target.InputBegan:Connect(function(input)
			if input.UserInputType == Enum.UserInputType.MouseButton1
				or input.UserInputType == Enum.UserInputType.Touch then
				updateFromPointer(target, isHue)
			end
		end)
		target.InputChanged:Connect(function(input)
			if input.UserInputType == Enum.UserInputType.MouseMovement then
				updateFromPointer(target, isHue)
			end
		end)
		target.InputEnded:Connect(function(input)
			if input.UserInputType == Enum.UserInputType.MouseButton1
				or input.UserInputType == Enum.UserInputType.Touch then
				updateFromPointer(target, isHue)
				if type(callback) == "function" then
					pcall(callback, color)
				end
				if options.Flag then
					Registry.Set(options.Flag, color)
				end
			end
		end)
	end
	makeInteractive(palette, false)
	makeInteractive(hueBar, true)

	local element
	element = {
		Frame		= card,
		Palette		= palette,
		HueBar		= hueBar,
		Selector	= selector,
		Meta		= {Kind = "ColorPicker"},
	}
	function element:GetValue()
		return color
	end
	function element:GetHex()
		return Utils.toHex(color)
	end
	function element:SetValue(v, silent)
		if type(v) == "string" then
			v = Utils.fromHex(v)
		end
		if type(v) ~= "Color3" then
			return
		end
		color = v
		hue, sat, val = Utils.rgbToHsv(v)
		whiteGrad.Color = ColorSequence.new(Color3.new(1, 1, 1), Utils.hsv(hue, 1, 1))
		pcall(function()
			selector.Position = UDim2.fromScale(sat, 1 - val)
			selectorInner.BackgroundColor3 = v
			hueSelector.Position = UDim2.fromScale(hue / 360, 0.5)
			hexLabel.Text = "#" .. Utils.toHex(v)
		end)
		if not silent and options.Flag then
			Registry.Set(options.Flag, v)
		end
		if not silent and type(callback) == "function" then
			pcall(callback, v)
		end
	end
	function element:SetText(v)
		titleLabel.Text = tostring(v)
	end
	element:SetValue(default, true)
	attachColorFlag(element, options.Flag, default)
	return element
end

-- ==== ЭЛЕМЕНТ: CreateListbox ====

function Advanced.CreateListbox(tab, text, options, default, callback, opts)
	options = options or {}
	opts = opts or {}
	local parent = tab.Scroll or tab.Content or tab
	local items = options.Items or opts.Items or {}
	local multiselect = options.Multiselect == true

	local card = Utils.new("Frame", {
		Name = "Listbox",
		BackgroundColor3 = Theme.C("Card"),
		BorderSizePixel = 0,
		Size = UDim2.new(1, 0, 0, 120),
		Parent = parent,
	})
	Utils.corner(card, OPTIONS.Radius - 2)
	Theme.Bind("Card", card, "BackgroundColor3")
	card.AutomaticSize = Enum.AutomaticSize.Y

	-- Заголовок
	local titleRow = Utils.new("Frame", {
		BackgroundTransparency = 1,
		Size = UDim2.new(1, -16, 0, 26),
		Position = UDim2.fromOffset(8, 4),
		Parent = card,
	})
	local titleLabel = Utils.new("TextLabel", {
		Text = tostring(text or "Список"),
		TextColor3 = Theme.C("Text"),
		TextSize = 13,
		BackgroundTransparency = 1,
		Size = UDim2.new(1, -70, 1, 0),
		Position = UDim2.fromOffset(4, 0),
		TextXAlignment = Enum.TextXAlignment.Left,
		Parent = titleRow,
	})
	Utils.setFont(titleLabel, "Gotham", "Medium")
	Theme.Bind("Text", titleLabel, "TextColor3")

	local counter = Utils.new("TextLabel", {
		Text = "0",
		TextColor3 = Theme.C("TextFaint"),
		TextSize = 11,
		BackgroundTransparency = 1,
		Size = UDim2.fromOffset(40, 20),
		Position = UDim2.new(1, -4, 0, 2),
		AnchorPoint = Vector2.new(1, 0),
		TextXAlignment = Enum.TextXAlignment.Right,
		Parent = titleRow,
	})
	Utils.setFont(counter, "Gotham", "SemiBold")
	Theme.Bind("TextFaint", counter, "TextColor3")

	-- Кнопки «все / нет»
	local allBtn = Utils.new("TextButton", {
		Text = "All",
		TextColor3 = Theme.C("Accent"),
		TextSize = 9,
		BackgroundColor3 = Theme.C("SurfaceAlt"),
		BorderSizePixel = 0,
		Size = UDim2.fromOffset(24, 18),
		Position = UDim2.new(1, -4, 1, -2),
		AnchorPoint = Vector2.new(1, 1),
		AutoButtonColor = false,
		Parent = titleRow,
	})
	Utils.corner(allBtn, 5)
	Utils.setFont(allBtn, "Gotham", "Bold")
	Theme.Bind("SurfaceAlt", allBtn, "BackgroundColor3")

	-- Скролл со списком
	local scroll = Utils.new("ScrollingFrame", {
		BackgroundColor3 = Theme.C("SurfaceDeep"),
		BackgroundTransparency = 0,
		BorderSizePixel = 0,
		CanvasSize = UDim2.new(0, 0, 0, 0),
		ScrollBarThickness = 3,
		ScrollBarImageColor3 = Theme.C("Track"),
		Size = UDim2.new(1, -16, 0, 80),
		Position = UDim2.fromOffset(8, 34),
		AutomaticCanvasSize = Enum.AutomaticSize.Y,
		ScrollingDirection = Enum.ScrollingDirection.Y,
		Parent = card,
	})
	Utils.corner(scroll, 9)
	Utils.padding(scroll, 4, 4, 4, 4)
	Theme.Bind("SurfaceDeep", scroll, "BackgroundColor3")
	Theme.Bind("Track", scroll, "ScrollBarImageColor3")
	Utils.sizeConstraint(scroll, 40, 220, 0, 0)
	Utils.listLayout(scroll, Enum.FillDirection.Vertical, 3)

	-- Состояние выбора
	local selected = {}
	if type(default) == "table" then
		for _, v in ipairs(default) do
			selected[v] = true
		end
	elseif default ~= nil then
		selected[default] = true
	end

	local rows = {}
	local function selectedList()
		local out = {}
		for _, item in ipairs(items) do
			local v = Utils.optionValue(item, 1)
			if selected[v] then
				table.insert(out, v)
			end
		end
		return out
	end

	local function updateCounter()
		local n = 0
		for _ in pairs(selected) do
			n = n + 1
		end
		counter.Text = tostring(n) .. "/" .. #items
		if options.Flag then
			Registry.Set(options.Flag, selectedList())
		end
	end

	local function rebuild()
		Utils.clear(scroll)
		Utils.listLayout(scroll, Enum.FillDirection.Vertical, 3)
		Utils.padding(scroll, 4, 4, 4, 4)
		rows = {}
		for i, item in ipairs(items) do
			local v = Utils.optionValue(item, i)
			local lbl = Utils.optionLabel(item, i)
			local row = Utils.new("TextButton", {
				Text = "",
				BackgroundColor3 = Theme.C("SurfaceAlt"),
				BackgroundTransparency = selected[v] and 0 or 0.5,
				BorderSizePixel = 0,
				Size = UDim2.new(1, -4, 0, 24),
				AutoButtonColor = false,
				Parent = scroll,
			})
			Utils.corner(row, 7)
			Theme.Bind("SurfaceAlt", row, "BackgroundColor3")

			-- Чекбокс
			local box = Utils.new("Frame", {
				BackgroundColor3 = selected[v] and Theme.C("Accent") or Theme.C("Track"),
				BorderSizePixel = 0,
				Size = UDim2.fromOffset(14, 14),
				Position = UDim2.fromOffset(6, 5),
				Parent = row,
			})
			Utils.corner(box, 4)
			local tick = Utils.new("TextLabel", {
				Text = "✓",
				TextColor3 = Theme.C("AccentText"),
				TextSize = 9,
				BackgroundTransparency = 1,
				Size = UDim2.fromScale(1, 1),
				Parent = box,
			})
			Utils.setFont(tick, "Gotham", "Bold")
			Theme.Bind("AccentText", tick, "TextColor3")
			tick.Visible = selected[v] == true

			local lblNode = Utils.new("TextLabel", {
				Text = tostring(lbl),
				TextColor3 = Theme.C("Text"),
				TextSize = 11,
				TextXAlignment = Enum.TextXAlignment.Left,
				TextTruncate = Enum.TextTruncate.AtEnd,
				BackgroundTransparency = 1,
				Size = UDim2.new(1, -30, 1, 0),
				Position = UDim2.fromOffset(28, 0),
				Parent = row,
			})
			Utils.setFont(lblNode, "Gotham", "Regular")
			Theme.Bind("Text", lblNode, "TextColor3")

			rows[i] = {Row = row, Box = box, Tick = tick, Value = v, Label = lblNode}

			row.MouseEnter:Connect(function()
				Motion.Tween({ Instance = row, Time = 0.12, BackgroundTransparency = 0 })
			end)
			row.MouseLeave:Connect(function()
				Motion.Tween({ Instance = row, Time = 0.16, BackgroundTransparency = selected[v] and 0 or 0.5 })
			end)
			row.MouseButton1Click:Connect(function()
				if multiselect then
					selected[v] = not selected[v]
				else
					-- одиночный выбор: снимаем остальные
					selected = {}
					selected[v] = true
				end
				-- обновить визуал всех строк
				for j, r in ipairs(rows) do
					if r ~= nil then
						local isOn = selected[r.Value] == true
						r.Tick.Visible = isOn
						pcall(function()
							r.Box.BackgroundColor3 = isOn and Theme.C("Accent") or Theme.C("Track")
							r.Row.BackgroundTransparency = isOn and 0 or 0.5
						end)
					end
				end
				updateCounter()
				if type(callback) == "function" then
					local ok, err = pcall(callback, selectedList())
					if not ok then
						log("Ошибка в callback listbox:", err)
					end
				end
			end)
		end
		updateCounter()
	end

	allBtn.MouseButton1Click:Connect(function()
		if multiselect then
			for _, item in ipairs(items) do
				selected[Utils.optionValue(item, 1)] = true
			end
		else
			selected = {}
		end
		rebuild()
		if type(callback) == "function" then
			pcall(callback, selectedList())
		end
	end)
	allBtn.MouseButton1Up:Connect(function()
		if multiselect then
			selected = {}
			rebuild()
			if type(callback) == "function" then
				pcall(callback, selectedList())
			end
		end
	end)

	local element = {
		Frame		= card,
		Scroll		= scroll,
		Rows		= rows,
		Meta		= {Kind = "Listbox"},
	}
	function element:GetValue()
		return selectedList()
	end
	function element:SetValue(v, silent)
		selected = {}
		if type(v) == "table" then
			for _, item in ipairs(v) do
				selected[item] = true
			end
		elseif v ~= nil then
			selected[v] = true
		end
		rebuild()
		if not silent and type(callback) == "function" then
			pcall(callback, selectedList())
		end
	end
	function element:SetItems(newItems)
		items = newItems or {}
		selected = {}
		rebuild()
	end
	function element:SelectAll()
		for _, item in ipairs(items) do
			selected[Utils.optionValue(item, 1)] = true
		end
		rebuild()
	end
	function element:DeselectAll()
		selected = {}
		rebuild()
	end
	function element:SetText(v)
		titleLabel.Text = tostring(v)
	end
	-- регистрация флага (таблица)
	if options.Flag then
		Registry.Create(options.Flag, selectedList(), {Kind = "Listbox"})
		element.Flag = options.Flag
		Registry.RegisterElement(element, {Kind = "Listbox"})
		Registry.OnChanged(options.Flag, function(value)
			if type(value) == "table" then
				element:SetValue(value, true)
			end
		end)
	end
	rebuild()
	return element
end

-- ==== ЭЛЕМЕНТ: CreateAccordion ====

function Advanced.CreateAccordion(tab, text, options, defaultOpen)
	options = options or {}
	local parent = tab.Scroll or tab.Content or tab
	local items = options.Items or {}

	local card = Utils.new("Frame", {
		Name = "Accordion",
		BackgroundColor3 = Theme.C("Card"),
		BorderSizePixel = 0,
		Size = UDim2.new(1, 0, 0, 42),
		Parent = parent,
	})
	Utils.corner(card, OPTIONS.Radius - 2)
	Theme.Bind("Card", card, "BackgroundColor3")
	card.AutomaticSize = Enum.AutomaticSize.Y

	-- Шапка
	local header = Utils.new("TextButton", {
		Text = "",
		BackgroundTransparency = 1,
		Size = UDim2.new(1, 0, 0, 42),
		AutoButtonColor = false,
		Parent = card,
	})
	Utils.corner(header, OPTIONS.Radius - 2)

	local titleLabel = Utils.new("TextLabel", {
		Text = tostring(text or "Аккордеон"),
		TextColor3 = Theme.C("Text"),
		TextSize = 13,
		BackgroundTransparency = 1,
		Size = UDim2.new(1, -50, 0, 42),
		Position = UDim2.fromOffset(12, 0),
		TextXAlignment = Enum.TextXAlignment.Left,
		Parent = header,
	})
	Utils.setFont(titleLabel, "Gotham", "Bold")
	Theme.Bind("Text", titleLabel, "TextColor3")

	local arrow = Utils.new("TextLabel", {
		Text = "▾",
		TextColor3 = Theme.C("TextDim"),
		TextSize = 14,
		BackgroundTransparency = 1,
		Size = UDim2.fromOffset(20, 42),
		Position = UDim2.new(1, -14, 0, 0),
		AnchorPoint = Vector2.new(1, 0),
		Parent = header,
	})
	Utils.setFont(arrow, "Gotham", "Bold")
	Theme.Bind("TextDim", arrow, "TextColor3")

	-- Тело
	local body = Utils.new("Frame", {
		BackgroundTransparency = 1,
		Size = UDim2.new(1, -16, 0, 0),
		Position = UDim2.fromOffset(8, 44),
		ClipsDescendants = true,
		Visible = defaultOpen == true,
		Parent = card,
	})
	Utils.listLayout(body, Enum.FillDirection.Vertical, 6)
	Utils.padding(body, 0, 4, 6, 4)
	body.AutomaticSize = Enum.AutomaticSize.Y

	-- Полоса-разделитель
	local divider = Utils.new("Frame", {
		BackgroundColor3 = Theme.C("BorderSoft"),
		BorderSizePixel = 0,
		Size = UDim2.new(1, -24, 0, 1),
		Position = UDim2.fromOffset(12, 42),
		Parent = card,
	})
	Utils.corner(divider, 999)
	Theme.Bind("BorderSoft", divider, "BackgroundColor3")

	-- Наполнение содержимым
	for _, item in ipairs(items) do
		local row = Utils.new("TextLabel", {
			Text = tostring(item.Text or item.Label or item),
			TextColor3 = Theme.C("TextDim"),
			TextSize = 12,
			TextWrapped = true,
			TextXAlignment = Enum.TextXAlignment.Left,
			TextYAlignment = Enum.TextYAlignment.Top,
			BackgroundTransparency = 1,
			Size = UDim2.new(1, -4, 0, 0),
			AutomaticSize = Enum.AutomaticSize.Y,
			Parent = body,
		})
		Utils.setFont(row, "Gotham", "Regular")
		Theme.Bind("TextDim", row, "TextColor3")
	end

	local isOpen = defaultOpen == true
	local bodyHeight = 0
	pcall(function()
		bodyHeight = body.AbsoluteContentSize.Y
	end)

	local element = {
		Frame		= card,
		Header		= header,
		Body		= body,
		Meta		= {Kind = "Accordion"},
	}

	function element:IsOpen()
		return isOpen
	end

	function element:Toggle()
		if isOpen then
			element:Close()
		else
			element:Open()
		end
		return isOpen
	end

	function element:Open()
		if isOpen then
			return
		end
		isOpen = true
		pcall(function()
			body.Visible = true
		end)
		local targetH = math.max(body.AbsoluteContentSize.Y, 20) + 8
		Motion.Tween({
			Instance = card,
			Time = 0.26,
			Easing = Easings.outCubic,
			Size = UDim2.new(1, 0, 0, 44 + targetH),
		})
		Motion.Tween({
			Instance = body,
			Time = 0.26,
			Easing = Easings.outCubic,
			Size = UDim2.new(1, -16, 0, targetH),
		})
		Motion.Tween({ Instance = arrow, Time = 0.26, Easing = Easings.outBack, Rotation = 180 })
		-- появление строк
		local children = {}
		for _, child in ipairs(body:GetChildren()) do
			if child:IsA("GuiObject") then
				table.insert(children, child)
			end
		end
		Motion.RevealList(children, {Step = 0.04, Time = 0.22, Dy = -6})
	end

	function element:Close()
		if not isOpen then
			return
		end
		isOpen = false
		Motion.Tween({
			Instance = body,
			Time = 0.2,
			Easing = Easings.inCubic,
			Size = UDim2.new(1, -16, 0, 0),
			OnComplete = function()
				if not isOpen then
					pcall(function()
						body.Visible = false
					end)
				end
			end,
		})
		Motion.Tween({
			Instance = card,
			Time = 0.24,
			Easing = Easings.outCubic,
			Size = UDim2.new(1, 0, 0, 44),
		})
		Motion.Tween({ Instance = arrow, Time = 0.24, Easing = Easings.outBack, Rotation = 0 })
	end

	header.MouseButton1Click:Connect(function()
		element:Toggle()
	end)
	header.MouseEnter:Connect(function()
		Motion.Tween({ Instance = titleLabel, Time = 0.14, TextColor3 = Theme.C("Accent") })
	end)
	header.MouseLeave:Connect(function()
		Motion.Tween({ Instance = titleLabel, Time = 0.18, TextColor3 = Theme.C("Text") })
	end)

	-- Начальное состояние
	if isOpen then
		pcall(function()
			local h = math.max(body.AbsoluteContentSize.Y, 20) + 8
			card.Size = UDim2.new(1, 0, 0, 44 + h)
			body.Size = UDim2.new(1, -16, 0, h)
		end)
		arrow.Rotation = 180
	else
		pcall(function()
			body.Size = UDim2.new(1, -16, 0, 0)
		end)
	end

	function element:SetValue() end
	return element
end

-- ==== ЭЛЕМЕНТ: CreateTooltip ====

-- Показывает всплывающую подсказку рядом с элементом
function Advanced.CreateTooltip(trigger, text, options)
	options = options or {}
	if trigger == nil then
		return nil
	end
	if options.ScreenGui == nil then
		options.ScreenGui = AshUI.CurrentGui
	end
	local holder = Utils.new("Frame", {
		BackgroundColor3 = Theme.C("SurfaceAlt"),
		BorderSizePixel = 0,
		Size = UDim2.fromOffset(140, 30),
		Visible = false,
		ZIndex = 300,
		Parent = options.ScreenGui,
	})
	Utils.corner(holder, 8)
	Utils.padding(holder, 6, 10, 6, 10)
	Utils.sizeConstraint(holder, 0, 200, 40, 260)
	local holderStroke = Utils.stroke(holder, Theme.C("Border"), 0.4, 1)
	Theme.Bind("SurfaceAlt", holder, "BackgroundColor3")
	Theme.Bind("Border", holderStroke, "Color")

	local label = Utils.new("TextLabel", {
		Text = tostring(text or ""),
		TextColor3 = Theme.C("Text"),
		TextSize = 12,
		TextWrapped = true,
		TextXAlignment = Enum.TextXAlignment.Left,
		TextYAlignment = Enum.TextYAlignment.Center,
		BackgroundTransparency = 1,
		Size = UDim2.fromScale(1, 0),
		Position = UDim2.fromScale(0, 0),
		AutomaticSize = Enum.AutomaticSize.Y,
		Parent = holder,
	})
	Utils.setFont(label, "Gotham", "Medium")
	Theme.Bind("Text", label, "TextColor3")

	-- Треугольник-указатель
	local tip = Utils.new("Frame", {
		BackgroundColor3 = Theme.C("SurfaceAlt"),
		BorderSizePixel = 0,
		Size = UDim2.fromOffset(10, 10),
		Rotation = 45,
		Visible = false,
		ZIndex = 299,
		Parent = options.ScreenGui,
	})
	Utils.corner(tip, 3)
	Theme.Bind("SurfaceAlt", tip, "BackgroundColor3")

	local showing = false
	local anim = nil

	local function hide()
		if not showing then
			return
		end
		showing = false
		Motion.Tween({
			Instance = holder,
			Time = 0.14,
			Easing = Easings.inQuad,
			BackgroundTransparency = 1,
			OnComplete = function()
				pcall(function()
					holder.Visible = false
				end)
			end,
		})
		Motion.Tween({ Instance = tip, Time = 0.14, BackgroundTransparency = 1, OnComplete = function()
			pcall(function() tip.Visible = false end)
		end })
	end

	local function show()
		if showing then
			return
		end
		showing = true
		local pos = Utils.absolutePos(trigger)
		local size = Utils.absoluteSize(trigger)
		local mouse = Utils.mousePos()
		-- позиция: над элементом, если хватает места
		local spaceAbove = pos.Y
		local holderSize = Vector2.new(160, 30)
		pcall(function()
			holderSize = holder.AbsoluteSize
		end)
		local x = pos.X + size.X / 2
		local y
		local side = "top"
		if spaceAbove > holderSize.Y + 30 then
			y = pos.Y - holderSize.Y - 10
			side = "top"
		else
			y = pos.Y + size.Y + 10
			side = "bottom"
		end
		x = Utils.clamp(x - holderSize.X / 2, 8, Utils.screenSize().X - holderSize.X - 8)
		pcall(function()
			holder.Visible = true
			holder.Position = UDim2.fromOffset(x, y)
			holder.BackgroundTransparency = 1
			tip.Visible = true
			tip.Position = UDim2.fromOffset(pos.X + size.X / 2 - 5, side == "top" and (y + holderSize.Y) or (y - 10))
			tip.BackgroundTransparency = 1
		end)
		Motion.Tween({ Instance = holder, Time = 0.16, Easing = Easings.outQuad, BackgroundTransparency = 0 })
		Motion.Tween({ Instance = tip, Time = 0.16, Easing = Easings.outQuad, BackgroundTransparency = 0 })
		Motion.Tween({ Instance = holder, Time = 0.18, Easing = Easings.outBack, Position = UDim2.fromOffset(x, y - 3) })
	end

	if trigger.MouseEnter then
		trigger.MouseEnter:Connect(function()
			-- задержка перед показом
			pcall(task.delay, options.Delay or 0.25, show)
		end)
		trigger.MouseLeave:Connect(hide)
	end

	local api = {
		SetText = function(_, v)
			label.Text = tostring(v)
		end,
		Show = show,
		Hide = hide,
		Holder = holder,
	}
	return api
end

-- ==== ЭЛЕМЕНТ: CreateCamera ====

-- Управление FOV и другими параметрами камеры
function Advanced.CreateCamera(tab, text, options, defaultFov, callback)
	options = options or {}
	local parent = tab.Scroll or tab.Content or tab
	defaultFov = defaultFov or 70

	local camera = Services.Workspace and Services.Workspace.CurrentCamera

	local card = Utils.new("Frame", {
		Name = "Camera",
		BackgroundColor3 = Theme.C("Card"),
		BorderSizePixel = 0,
		Size = UDim2.new(1, 0, 0, 96),
		Parent = parent,
	})
	Utils.corner(card, OPTIONS.Radius - 2)
	Theme.Bind("Card", card, "BackgroundColor3")

	local titleLabel = Utils.new("TextLabel", {
		Text = tostring(text or "Камера"),
		TextColor3 = Theme.C("Text"),
		TextSize = 13,
		BackgroundTransparency = 1,
		Size = UDim2.new(1, -20, 0, 18),
		Position = UDim2.fromOffset(10, 6),
		TextXAlignment = Enum.TextXAlignment.Left,
		Parent = card,
	})
	Utils.setFont(titleLabel, "Gotham", "Medium")
	Theme.Bind("Text", titleLabel, "TextColor3")

	-- Ползунок FOV
	local fovValue = defaultFov
	local fovSlider = Elements.CreateSlider({
		Scroll = parent,
	}, "FOV", 30, 110, fovValue, "°", function(v)
		fovValue = v
		applyFov(v)
		if type(callback) == "function" then
			pcall(callback, v)
		end
	end, {NoFlag = true})
	-- Переносим ползунок внутрь карточки
	pcall(function()
		fovSlider.Frame.Parent = card
		fovSlider.Frame.Position = UDim2.fromOffset(0, 26)
		fovSlider.Frame.Size = UDim2.new(1, 0, 0, 58)
	end)

	local function applyFov(v)
		if camera == nil then
			return
		end
		pcall(function()
			camera.FieldOfView = Utils.clamp(v, 1, 170)
		end)
	end

	-- Блокировка камеры
	local state = {
		Locked		= false,
		OriginalType = nil,
		OriginalPos	= nil,
	}
	local function setCameraLock(locked)
		if camera == nil then
			return
		end
		if locked then
			state.Locked = true
			pcall(function()
				state.OriginalType = camera.CameraType
				state.OriginalPos = camera.CFrame
				camera.CameraType = Enum.CameraType.Scriptable
			end)
		else
			state.Locked = false
			pcall(function()
				camera.CameraType = state.OriginalType or Enum.CameraType.Custom
			end)
			if state.OriginalPos then
				pcall(function()
					camera.CFrame = state.OriginalPos
				end)
			end
		end
	end

	-- Быстрые кнопки
	local row = Utils.new("Frame", {
		BackgroundTransparency = 1,
		Size = UDim2.new(1, -20, 0, 26),
		Position = UDim2.fromOffset(10, 64),
		Parent = card,
	})
	Utils.listLayout(row, Enum.FillDirection.Horizontal, 6)
	local function quickBtn(label2, fn, key)
		local b = Utils.new("TextButton", {
			Text = tostring(label2),
			TextColor3 = Theme.C("Text"),
			TextSize = 11,
			BackgroundColor3 = Theme.C("SurfaceAlt"),
			BorderSizePixel = 0,
			Size = UDim2.fromOffset(74, 24),
			AutoButtonColor = false,
			Parent = row,
		})
		Utils.corner(b, 8)
		Utils.setFont(b, "Gotham", "SemiBold")
		Theme.Bind("SurfaceAlt", b, "BackgroundColor3")
		Theme.Bind("Text", b, "TextColor3")
		b.MouseEnter:Connect(function()
			Motion.Tween({ Instance = b, Time = 0.12, BackgroundColor3 = Theme.C("CardHover") })
		end)
		b.MouseLeave:Connect(function()
			Motion.Tween({ Instance = b, Time = 0.16, BackgroundColor3 = Theme.C("SurfaceAlt") })
		end)
		b.MouseButton1Click:Connect(function()
			pcall(fn)
		end)
		if key then
			pcall(function()
				Advanced.CreateTooltip(b, key)
			end)
		end
		return b
	end
	quickBtn("FOV 70", function()
		fovSlider:SetValue(70)
		applyFov(70)
	end, "Сбросить поле зрения")
	quickBtn("FOV 100", function()
		fovSlider:SetValue(100)
		applyFov(100)
	end, "Широкий угол")
	quickBtn("+1 / -1", function()
		local v = Utils.clamp(fovValue + 1, 30, 110)
		fovSlider:SetValue(v)
		applyFov(v)
	end, "Тонкая настройка")
	quickBtn("Lock", function()
		setCameraLock(not state.Locked)
		Notify.Info("Камера", state.Locked and "Заблокирована" or "Разблокирована", 2.5)
	end, "Заблокировать/разблокировать камеру")
	quickBtn("Shake", function()
		-- тряска камеры через TweenService
		if camera == nil or Services.TweenService == nil then
			return
		end
		local start = camera.CFrame
		pcall(Services.TweenService:Create(camera, TweenInfo.new(0.35, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {
			CFrame = start * CFrame.new(0, 0.4, 0),
		}):Play())
		pcall(task.delay, 0.36, function()
			pcall(Services.TweenService:Create(camera, TweenInfo.new(0.35, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {
				CFrame = start,
			}):Play())
		end)
	end, "Тряска камеры")

	local element = {
		Frame		= card,
		Slider		= fovSlider,
		Meta		= {Kind = "Camera"},
	}
	function element:GetValue()
		return fovValue
	end
	function element:SetValue(v, silent)
		fovValue = Utils.clamp(tonumber(v) or 70, 30, 110)
		pcall(function() fovSlider:SetValue(fovValue, true) end)
		applyFov(fovValue)
		if not silent and type(callback) == "function" then
			pcall(callback, fovValue)
		end
	end
	function element:SetText(v)
		titleLabel.Text = tostring(v)
	end
	function element:ResetFov()
		element:SetValue(70)
	end
	function element:SetLock(v)
		setCameraLock(v == true)
	end
	return element
end
-- ==== Window — окно, сайдбар, вкладки, хоткей ====

local Window = {}
AshUI.Modules.Window = Window
AshUI.Window = Window

-- Глобальный ScreenGui для библиотеки
local function getOrCreateGui(name)
	-- Если уже есть наш ScreenGui — используем его
	if AshUI.CurrentGui ~= nil and AshUI.CurrentGui.Parent ~= nil then
		return AshUI.CurrentGui
	end

	local function resolvePlayerGui()
		if type(Services.Players) ~= "table" or Services.Players.LocalPlayer == nil then
			return nil
		end
		local ok, result = pcall(function()
			return Services.Players.LocalPlayer:FindFirstChildOfClass("PlayerGui")
				or Services.Players.LocalPlayer:WaitForChild("PlayerGui", 1)
		end)
		if ok and result ~= nil then
			return result
		end
		return nil
	end

	local function resolveCoreGui()
		if type(game) ~= "table" then
			return nil
		end
		local ok, core = pcall(function()
			return game:GetService("CoreGui")
		end)
		if ok and core ~= nil then
			return core
		end
		return nil
	end

	local parent = resolvePlayerGui()
	if parent == nil then
		local gethui = AshUI.env("gethui")
		if type(gethui) == "function" then
			local ok, hui = pcall(gethui)
			if ok and hui ~= nil then
				parent = hui
			end
		end
	end
	if parent == nil then
		parent = resolveCoreGui()
	end
	if parent == nil then
		return nil
	end

	local gui = Utils.new("ScreenGui", {
		Name = name or "AshUI",
		ResetOnSpawn = false,
		IgnoreGuiInset = false,
		ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
		DisplayOrder = 500,
		Parent = parent,
		Enabled = true,
	})
	if gui == nil and type(Instance) == "table" and type(Instance.new) == "function" then
		gui = Instance.new("ScreenGui")
		gui.Name = name or "AshUI"
		gui.ResetOnSpawn = false
		gui.IgnoreGuiInset = false
		gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
		gui.DisplayOrder = 500
		gui.Enabled = true
		gui.Parent = parent
	end
	if gui ~= nil then
		AshUI.CurrentGui = gui
		AshUI.log("ScreenGui создан:", gui:GetFullName())
	end
	return gui
end

-- ==== CreateWindow ====

function Window.Create(options)
	options = options or {}
	local gui = getOrCreateGui(options.GuiName or "AshUI")
	if gui == nil then
		log("Не удалось создать ScreenGui — интерфейс не будет отрисован")
		return nil
	end

	local title = options.Title or options.Name or OPTIONS.Name
	local size = Utils.clampWindowSize(options.Size or OPTIONS.WindowSize, Vector2.new(430, 340))
	local screen = Utils.screenSize()
	-- Стартовая позиция — по центру
	local startPos = options.Position or Vector2.new(
		(screen.X - size.X) / 2,
		(screen.Y - size.Y) / 2
	)

	-- ==== Затемнение ===
	local fade = Utils.new("Frame", {
		Name = "Fade",
		BackgroundColor3 = Theme.C("Overlay"),
		BackgroundTransparency = 1,
		BorderSizePixel = 0,
		Size = UDim2.fromScale(1, 1),
		ZIndex = 100,
		Visible = false,
		Parent = gui,
	})
	Theme.Bind("Overlay", fade, "BackgroundColor3")
	FADE = fade

	-- ==== Контейнер окна (для анимации масштаба) ===
	local wrapper = Utils.new("Frame", {
		Name = "WindowWrapper",
		BackgroundTransparency = 1,
		Size = UDim2.fromScale(1, 1),
		ZIndex = 200,
		Parent = gui,
	})

	-- ==== Тень и свечение ===
	-- Holder повторяет позицию/размер окна (обновляется каждый кадр)
	local shadowHolder = Utils.new("Frame", {
		BackgroundTransparency = 1,
		Size = UDim2.fromScale(1, 1),
		ZIndex = 201,
		ClipsDescendants = false,
		Parent = wrapper,
	})
	local glow = Utils.new("ImageLabel", {
		BackgroundTransparency = 1,
		Size = UDim2.fromScale(1.18, 1.3),
		Position = UDim2.fromScale(-0.09, -0.15),
		Image = "rbxassetid://6029598422",
		ImageColor3 = Theme.C("Accent"),
		ImageTransparency = 0.92,
		ZIndex = 201,
		Parent = shadowHolder,
	})
	Utils.corner(glow, 60)
	local shadow = Utils.new("ImageLabel", {
		BackgroundTransparency = 1,
		Size = UDim2.fromScale(1.12, 1.22),
		Position = UDim2.fromScale(-0.06, -0.11),
		Image = "rbxassetid://6029598422",
		ImageColor3 = Color3.new(0, 0, 0),
		ImageTransparency = 0.35,
		ZIndex = 202,
		Parent = shadowHolder,
	})
	Utils.corner(shadow, 60)

	-- ==== Основной фрейм ===
	local frame = Utils.new("Frame", {
		Name = "MainWindow",
		BackgroundColor3 = Theme.C("Background"),
		BackgroundTransparency = 0,
		BorderSizePixel = 0,
		Size = UDim2.fromOffset(size.X, size.Y),
		Position = UDim2.fromOffset(startPos.X, startPos.Y),
		ClipsDescendants = true,
		ZIndex = 203,
		Parent = wrapper,
	})
	Utils.corner(frame, OPTIONS.Radius)
	Theme.Bind("Background", frame, "BackgroundColor3")

	-- UIScale — анимация открытия/закрытия без конфликта с drag
	local uiScale = Utils.new("UIScale", {
		Scale = 1,
		Parent = frame,
	})

	-- Синхронизация тени с окном (каждый кадр, без wait)
	-- Ссылаемся только на frame, объявленный выше (не на win — он создан позже)
	Motion.OnUpdate(function()
		if frame == nil or frame.Parent == nil then
			return
		end
		local p = Utils.get(frame, "Position")
		local s = Utils.get(frame, "Size")
		if type(p) ~= "UDim2" or type(s) ~= "UDim2" then
			return
		end
		pcall(function()
			shadowHolder.Position = p
			shadowHolder.Size = s
		end)
	end, 0)

	local stroke = Utils.stroke(frame, Theme.C("Border"), 0.55, 1)
	Theme.Bind("Border", stroke, "Color")

	-- ==== Заголовок ===
	local titleBar = Utils.new("Frame", {
		Name = "TitleBar",
		BackgroundColor3 = Theme.C("Surface"),
		BorderSizePixel = 0,
		Size = UDim2.new(1, 0, 0, 46),
		ZIndex = 210,
		Parent = frame,
	})
	Theme.Bind("Surface", titleBar, "BackgroundColor3")
	-- Нижняя граница заголовка
	local titleLine = Utils.new("Frame", {
		BackgroundColor3 = Theme.C("BorderSoft"),
		BorderSizePixel = 0,
		Size = UDim2.new(1, 0, 0, 1),
		Position = UDim2.new(0, 0, 1, 0),
		ZIndex = 210,
		Parent = titleBar,
	})
	Utils.corner(titleLine, 999)
	Theme.Bind("BorderSoft", titleLine, "BackgroundColor3")

	-- Иконка-луна в заголовке
	local iconHolder = Utils.new("Frame", {
		BackgroundTransparency = 1,
		Size = UDim2.fromOffset(30, 30),
		Position = UDim2.fromOffset(14, 8),
		ZIndex = 212,
		Parent = titleBar,
	})
	local iconGlow = Utils.new("ImageLabel", {
		BackgroundTransparency = 1,
		Size = UDim2.fromScale(2, 2),
		Image = "rbxassetid://6029598422",
		ImageColor3 = Theme.C("Accent"),
		ImageTransparency = 0.55,
		ZIndex = 211,
		Parent = iconHolder,
	})
	Utils.corner(iconGlow, 999)
	local iconMoon = Utils.new("Frame", {
		BackgroundColor3 = Theme.C("Accent"),
		BorderSizePixel = 0,
		Size = UDim2.fromOffset(20, 20),
		Position = UDim2.fromOffset(5, 5),
		ZIndex = 212,
		Parent = iconHolder,
	})
	Utils.corner(iconMoon, 999)
	Theme.Bind("Accent", iconMoon, "BackgroundColor3")
	Theme.Bind("Accent", iconGlow, "ImageColor3")
	-- Кратеры на иконке
	for _, c in ipairs({{0.3, 0.32, 5}, {0.6, 0.62, 3.6}, {0.44, 0.78, 4.2}}) do
		local crater = Utils.new("Frame", {
			BackgroundColor3 = Theme.C("Background"),
			BackgroundTransparency = 0.45,
			BorderSizePixel = 0,
			Size = UDim2.fromOffset(c[3], c[3]),
			Position = UDim2.fromScale(c[1], c[2]),
			ZIndex = 213,
			Parent = iconMoon,
		})
		Utils.corner(crater, 999)
	end
	-- Анимация «дыхания» иконки
	Motion.Pulse(iconGlow, {Property = "ImageTransparency", From = 0.4, To = 0.75, Time = 2.2})

	-- Заголовок / подзаголовок
	local titleText = Utils.new("TextLabel", {
		Text = tostring(title),
		TextColor3 = Theme.C("Text"),
		TextSize = 15,
		TextXAlignment = Enum.TextXAlignment.Left,
		BackgroundTransparency = 1,
		Size = UDim2.new(1, -220, 0, 18),
		Position = UDim2.fromOffset(52, 6),
		ZIndex = 212,
		Parent = titleBar,
	})
	Utils.setFont(titleText, "Gotham", "Bold")
	Theme.Bind("Text", titleText, "TextColor3")

	local subtitleText = Utils.new("TextLabel", {
		Text = tostring(options.Subtitle or ("v" .. OPTIONS.Version)),
		TextColor3 = Theme.C("TextFaint"),
		TextSize = 10,
		TextXAlignment = Enum.TextXAlignment.Left,
		BackgroundTransparency = 1,
		Size = UDim2.new(1, -220, 0, 14),
		Position = UDim2.fromOffset(52, 24),
		ZIndex = 212,
		Parent = titleBar,
	})
	Utils.setFont(subtitleText, "Gotham", "Regular")
	Theme.Bind("TextFaint", subtitleText, "TextColor3")

	-- Кнопки справа
	local function titleButton(glyph, tooltip)
		local b = Utils.new("TextButton", {
			Text = glyph,
			TextColor3 = Theme.C("TextDim"),
			TextSize = 13,
			BackgroundTransparency = 1,
			BorderSizePixel = 0,
			Size = UDim2.fromOffset(28, 28),
			Position = UDim2.new(1, -34, 0, 9),
			AnchorPoint = Vector2.new(1, 0),
			AutoButtonColor = false,
			ZIndex = 212,
			Parent = titleBar,
		})
		Utils.corner(b, 8)
		Utils.setFont(b, "Gotham", "Bold")
		Theme.Bind("TextDim", b, "TextColor3")
		b.MouseEnter:Connect(function()
			Motion.Tween({ Instance = b, Time = 0.14, BackgroundTransparency = 0.82, TextColor3 = Theme.C("Text") })
		end)
		b.MouseLeave:Connect(function()
			Motion.Tween({ Instance = b, Time = 0.18, BackgroundTransparency = 1, TextColor3 = Theme.C("TextDim") })
		end)
		b.MouseButton1Down:Connect(function()
			Motion.Tween({ Instance = b, Time = 0.08, BackgroundTransparency = 0.6 })
		end)
		b.MouseButton1Up:Connect(function()
			Motion.Tween({ Instance = b, Time = 0.16, BackgroundTransparency = 0.82 })
		end)
		if tooltip then
			pcall(function()
				Advanced.CreateTooltip(b, tooltip)
			end)
		end
		return b
	end

	local closeBtn = titleButton("✕", "Закрыть окно (скрыть)")
	local themeBtn = titleButton("☾", "Сменить тему (следующая)")
	local pinBtn = titleButton("○", "Snap к краям экрана")

	-- ==== Тело окна ===
	local body = Utils.new("Frame", {
		Name = "Body",
		BackgroundTransparency = 1,
		Size = UDim2.new(1, 0, 1, -46),
		Position = UDim2.new(0, 0, 0, 46),
		ZIndex = 205,
		Parent = frame,
	})

	-- ==== Сайдбар ===
	local sidebar = Utils.new("Frame", {
		Name = "Sidebar",
		BackgroundColor3 = Theme.C("Surface"),
		BorderSizePixel = 0,
		Size = UDim2.new(0, 196, 1, 0),
		ZIndex = 206,
		Parent = body,
	})
	Theme.Bind("Surface", sidebar, "BackgroundColor3")
	-- Правая граница сайдбара
	local sideLine = Utils.new("Frame", {
		BackgroundColor3 = Theme.C("BorderSoft"),
		BorderSizePixel = 0,
		Size = UDim2.new(0, 1, 1, 0),
		Position = UDim2.new(1, 0, 0, 0),
		ZIndex = 207,
		Parent = sidebar,
	})
	Theme.Bind("BorderSoft", sideLine, "BackgroundColor3")

	-- Заголовок сайдбара
	local sideHead = Utils.new("TextLabel", {
		Text = tostring(options.SidebarTitle or "NAVIGATION").upper(),
		TextColor3 = Theme.C("TextFaint"),
		TextSize = 9,
		TextXAlignment = Enum.TextXAlignment.Left,
		BackgroundTransparency = 1,
		Size = UDim2.new(1, -24, 0, 16),
		Position = UDim2.fromOffset(16, 14),
		ZIndex = 208,
		Parent = sidebar,
	})
	Utils.setFont(sideHead, "Gotham", "Bold")
	Theme.Bind("TextFaint", sideHead, "TextColor3")

	-- Список вкладок
	local tabList = Utils.new("ScrollingFrame", {
		Name = "TabList",
		BackgroundTransparency = 1,
		BorderSizePixel = 0,
		ScrollBarThickness = 0,
		Size = UDim2.new(1, -12, 1, -122),
		Position = UDim2.fromOffset(6, 36),
		CanvasSize = UDim2.new(0, 0, 0, 0),
		AutomaticCanvasSize = Enum.AutomaticSize.Y,
		ScrollingDirection = Enum.ScrollingDirection.Y,
		ZIndex = 207,
		Parent = sidebar,
	})
	Utils.listLayout(tabList, Enum.FillDirection.Vertical, 4)
	Utils.padding(tabList, 0, 6, 0, 6)

	-- Индикатор активной вкладки
	local indicator = Utils.new("Frame", {
		Name = "Indicator",
		BackgroundColor3 = Theme.C("Accent"),
		BorderSizePixel = 0,
		Size = UDim2.fromOffset(3, 20),
		Position = UDim2.fromOffset(6, 44),
		AnchorPoint = Vector2.new(0, 0.5),
		ZIndex = 209,
		Parent = sidebar,
	})
	Utils.corner(indicator, 999)
	Theme.Bind("Accent", indicator, "BackgroundColor3")
	local indicatorGlow = Utils.new("ImageLabel", {
		BackgroundTransparency = 1,
		Size = UDim2.fromOffset(20, 34),
		Position = UDim2.fromOffset(-4, -17),
		Image = "rbxassetid://6029598422",
		ImageColor3 = Theme.C("Accent"),
		ImageTransparency = 0.55,
		ZIndex = 208,
		Parent = indicator,
	})
	Utils.corner(indicatorGlow, 999)
	Theme.Bind("Accent", indicatorGlow, "ImageColor3")

	-- ==== Карточка пользователя внизу сайдбара ===
	local userCard = Utils.new("Frame", {
		Name = "UserCard",
		BackgroundColor3 = Theme.C("SurfaceAlt"),
		BorderSizePixel = 0,
		Size = UDim2.new(1, -24, 0, 62),
		Position = UDim2.new(0, 12, 1, -74),
		ZIndex = 208,
		Parent = sidebar,
	})
	Utils.corner(userCard, 12)
	Theme.Bind("SurfaceAlt", userCard, "BackgroundColor3")
	cardHover(userCard)

	-- Аватар
	local avatarImg = Utils.new("Frame", {
		BackgroundColor3 = Theme.C("Accent"),
		BorderSizePixel = 0,
		Size = UDim2.fromOffset(34, 34),
		Position = UDim2.fromOffset(10, 14),
		ZIndex = 210,
		Parent = userCard,
	})
	Utils.corner(avatarImg, 999)
	Theme.Bind("Accent", avatarImg, "BackgroundColor3")
	-- Попытка загрузить аватар игрока
	pcall(function()
		local player = Services.Players and Services.Players.LocalPlayer
		if player and player.UserId then
			local thumb = Services.Players:GetUserThumbnailAsync(player.UserId, Enum.ThumbnailType.HeadShot, Enum.ThumbnailSize.Size48x48)
			if thumb and thumb.Thumbnail then
				avatarImg:Remove()
				local img = Utils.new("ImageLabel", {
					Image = thumb.Thumbnail,
					BackgroundTransparency = 1,
					Size = UDim2.fromOffset(34, 34),
					Position = UDim2.fromOffset(10, 14),
					ZIndex = 210,
					Parent = userCard,
				})
				Utils.corner(img, 999)
				avatarImg = img
			end
		end
	end)
	-- Первая буква ника, если аватара нет
	local initial = Utils.new("TextLabel", {
		Text = "?",
		TextColor3 = Theme.C("AccentText"),
		TextSize = 14,
		BackgroundTransparency = 1,
		Size = UDim2.fromScale(1, 1),
		ZIndex = 211,
		Parent = avatarImg,
	})
	Utils.setFont(initial, "GothamBold")
	Theme.Bind("AccentText", initial, "TextColor3")
	pcall(function()
		local player = Services.Players and Services.Players.LocalPlayer
		if player then
			initial.Text = string.upper(string.sub(player.DisplayName or player.Name, 1, 1))
		end
	end)

	local userName = Utils.new("TextLabel", {
		Text = "Guest",
		TextColor3 = Theme.C("Text"),
		TextSize = 12,
		TextXAlignment = Enum.TextXAlignment.Left,
		TextTruncate = Enum.TextTruncate.AtEnd,
		BackgroundTransparency = 1,
		Size = UDim2.new(1, -60, 0, 16),
		Position = UDim2.fromOffset(50, 14),
		ZIndex = 210,
		Parent = userCard,
	})
	Utils.setFont(userName, "Gotham", "Bold")
	Theme.Bind("Text", userName, "TextColor3")

	local userRole = Utils.new("TextLabel", {
		Text = "Player",
		TextColor3 = Theme.C("TextFaint"),
		TextSize = 10,
		TextXAlignment = Enum.TextXAlignment.Left,
		TextTruncate = Enum.TextTruncate.AtEnd,
		BackgroundTransparency = 1,
		Size = UDim2.new(1, -60, 0, 14),
		Position = UDim2.fromOffset(50, 32),
		ZIndex = 210,
		Parent = userCard,
	})
	Utils.setFont(userRole, "Gotham", "Regular")
	Theme.Bind("TextFaint", userRole, "TextColor3")

	-- Роль в зависимости от UserType / Team
	pcall(function()
		local player = Services.Players and Services.Players.LocalPlayer
		if player then
			userName.Text = player.DisplayName or player.Name
			local role = "Player"
			if player.UserType == Enum.UserType.Administrator then
				role = "Administrator"
			elseif player.UserType == Enum.UserType.Premium then
				role = "Premium"
			elseif player.Team then
				role = player.Team.Name
			end
			userRole.Text = role
		end
	end)

	-- Кнопка на карточке — скопировать UserID
	local copyName = Utils.new("TextButton", {
		Text = "ID",
		TextColor3 = Theme.C("Accent"),
		TextSize = 9,
		BackgroundTransparency = 1,
		BorderSizePixel = 0,
		Size = UDim2.fromOffset(22, 18),
		Position = UDim2.new(1, -6, 0, 6),
		AnchorPoint = Vector2.new(1, 0),
		AutoButtonColor = false,
		ZIndex = 211,
		Parent = userCard,
	})
	Utils.corner(copyName, 5)
	Utils.setFont(copyName, "Gotham", "Bold")
	Theme.Bind("Accent", copyName, "TextColor3")
	copyName.MouseButton1Click:Connect(function()
		local setclip = AshUI.env("setclipboard")
		pcall(function()
			local player = Services.Players and Services.Players.LocalPlayer
			local id = player and tostring(player.UserId) or "0"
			if type(setclip) == "function" then
				setclip(id)
				Notify.Success("UserID скопирован", id, 2.5)
			end
		end)
	end)

	-- ==== Контент вкладок ===
	local content = Utils.new("Frame", {
		Name = "Content",
		BackgroundColor3 = Theme.C("Background"),
		BorderSizePixel = 0,
		Size = UDim2.new(1, -196, 1, 0),
		Position = UDim2.fromOffset(196, 0),
		ClipsDescendants = true,
		ZIndex = 205,
		Parent = body,
	})
	Theme.Bind("Background", content, "BackgroundColor3")

	-- Декоративный фон контента (звёзды + луна)
	local spaceField = Anim.CreateSpaceField(content, {
		Width = math.max(math.min(size.X - 196, 760), 260),
		Height = math.max(math.min(size.Y - 46, 520), 200),
		Stars = 52,
	})
	-- Звёзды не должны перехватывать клики
	pcall(function()
		spaceField.Field.Active = true
		for _, child in ipairs(spaceField.Field:GetDescendants()) do
			if child:IsA("GuiObject") then
				child.Active = true
			end
		end
	end)
	-- Плавное затухание декора
	local fieldGrad = Utils.new("UIGradient", {
		Color = ColorSequence.new(Theme.C("Background"), Theme.C("Background")),
		Rotation = 90,
		Transparency = NumberSequence.new({
			NumberSequenceKeypoint.new(0, 0.45),
			NumberSequenceKeypoint.new(0.35, 0.72),
			NumberSequenceKeypoint.new(1, 1),
		}),
		Parent = spaceField.Field,
	})

	-- Контейнер, в который попадают страницы вкладок
	local pagesHolder = Utils.new("Frame", {
		Name = "Pages",
		BackgroundTransparency = 1,
		Size = UDim2.fromScale(1, 1),
		ClipsDescendants = true,
		ZIndex = 220,
		Parent = content,
	})

	-- ==== Состояние окна ====
	local win = {
		Name			= title,
		Frame			= frame,
		Wrapper		= wrapper,
		Gui			= gui,
		Fade			= fade,
		TitleBar		= titleBar,
		TitleText		= titleText,
		Subtitle		= subtitleText,
		Sidebar		= sidebar,
		Content		= content,
		Pages			= pagesHolder,
		SpaceField		= spaceField,
		Indicator		= indicator,
		CloseButton		= closeBtn,
		ThemeButton		= themeBtn,
		PinButton		= pinBtn,
		Stroke			= stroke,
		Glow			= glow,
		Shadow			= shadow,
		Tabs			= {},
		CurrentTab		= nil,
		Open			= false,
		Size			= size,
		Visible			= false,
		Meta			= {Kind = "Window"},
	}
	AshUI.CurrentWindow = win

	-- Ресайз (изменение размера окна)
	function win:SetSize(newSize)
		local sz = Utils.clampWindowSize(newSize, Vector2.new(380, 300))
		win.Size = sz
		Motion.Tween({
			Instance = frame,
			Time = 0.22,
			Easing = Easings.outCubic,
			Size = UDim2.fromOffset(sz.X, sz.Y),
		})
		Motion.Tween({
			Instance = spaceField.Field,
			Time = 0.22,
			Size = UDim2.fromOffset(math.max(math.min(sz.X - 196, 760), 260), math.max(math.min(sz.Y - 46, 520), 200)),
		})
	end

	-- Заголовок окна
	function win:SetTitle(newTitle)
		titleText.Text = tostring(newTitle)
		win.Name = tostring(newTitle)
	end

	-- Подзаголовок
	function win:SetSubtitle(newSub)
		subtitleText.Text = tostring(newSub)
	end

	-- ==== Переключение вкладок ====

	local function moveIndicator(tabButton, instant)
		local targetY = tabButton.Position.Y.Offset + tabButton.Size.Y.Offset / 2
		if instant then
			pcall(function()
				indicator.Position = UDim2.new(0, 6, 0, targetY)
			end)
		else
			Motion.Tween({
				Instance = indicator,
				Time = 0.26,
				Easing = Easings.outCubic,
				Position = UDim2.new(0, 6, 0, targetY),
			})
			Motion.Tween({
				Instance = indicatorGlow,
				Time = 0.26,
				ImageTransparency = 0.3,
			})
		end
	end

	function win:SelectTab(tabName, instant)
		local target = nil
		for _, t in ipairs(win.Tabs) do
			if t.Name == tabName or t.TabName == tabName then
				target = t
				break
			end
		end
		if target == nil then
			-- может быть объектом
			if type(tabName) == "table" and tabName.Scroll ~= nil then
				target = tabName
			else
				return false
			end
		end
		if target == win.CurrentTab then
			return true
		end
		local previous = win.CurrentTab
		win.CurrentTab = target
		target.Active = true

		-- состояние кнопок сайдбара
		for _, t in ipairs(win.Tabs) do
			local isActive = (t == target)
			t.Active = isActive
			if isActive then
				Motion.Tween({
					Instance = t.Label,
					Time = 0.2,
					Easing = Easings.outQuad,
					TextColor3 = Theme.C("Text"),
				})
				Motion.Tween({
					Instance = t.Icon,
					Time = 0.2,
					Easing = Easings.outQuad,
					TextColor3 = Theme.C("Accent"),
				})
				Motion.Tween({
					Instance = t.Background,
					Time = 0.2,
					Easing = Easings.outQuad,
					BackgroundTransparency = 0,
					BackgroundColor3 = Theme.C("SurfaceAlt"),
				})
			else
				Motion.Tween({
					Instance = t.Label,
					Time = 0.2,
					Easing = Easings.outQuad,
					TextColor3 = Theme.C("TextDim"),
				})
				Motion.Tween({
					Instance = t.Icon,
					Time = 0.2,
					Easing = Easings.outQuad,
					TextColor3 = Theme.C("TextFaint"),
				})
				Motion.Tween({
					Instance = t.Background,
					Time = 0.2,
					Easing = Easings.outQuad,
					BackgroundTransparency = 1,
					BackgroundColor3 = Theme.C("SurfaceAlt"),
				})
			end
		end

		-- скрыть предыдущую
		if previous ~= nil and previous.Scroll ~= nil then
			local oldPage = previous.Scroll
			Motion.Tween({
				Instance = oldPage,
				Time = 0.14,
				Easing = Easings.inQuad,
				BackgroundTransparency = 1,
				OnComplete = function()
					if win.CurrentTab ~= previous then
						pcall(function()
							oldPage.Visible = false
						end)
					end
				end,
			})
			-- текстовые элементы
			for _, child in ipairs(oldPage:GetDescendants()) do
				if child:IsA("TextLabel") or child:IsA("TextButton") then
					Motion.Tween({ Instance = child, Time = 0.12, TextTransparency = 1 })
				end
			end
		end

		-- показать новую
		local newPage = target.Scroll
		pcall(function()
			newPage.Visible = true
			newPage.BackgroundTransparency = 1
		end)
		local items = {}
		for _, child in ipairs(newPage:GetChildren()) do
			if child:IsA("GuiObject") and child.Name ~= "__spacer" then
				table.insert(items, child)
			end
		end
		Motion.RevealList(items, {
			Step = 0.045,
			Time = 0.3,
			Delay = instant and 0 or 0.03,
			Dx = 22,
			Dy = 0,
		})
		-- движение страницы (сдвиг)
		Motion.Tween({
			Instance = newPage,
			Time = 0.24,
			Easing = Easings.outCubic,
			Position = UDim2.fromOffset(0, 0),
		})

		moveIndicator(target.Button, instant)
		if type(target.OnSelect) == "function" then
			pcall(target.OnSelect)
		end
		return true
	end

	-- Возврат на предыдущую/следующую вкладку
	function win:NextTab()
		if #win.Tabs == 0 then
			return
		end
		local idx = 1
		for i, t in ipairs(win.Tabs) do
			if t == win.CurrentTab then
				idx = i
				break
			end
		end
		local nextIdx = (idx % #win.Tabs) + 1
		win:SelectTab(win.Tabs[nextIdx].Name)
	end

	function win:PrevTab()
		if #win.Tabs == 0 then
			return
		end
		local idx = 1
		for i, t in ipairs(win.Tabs) do
			if t == win.CurrentTab then
				idx = i
				break
			end
		end
		local prevIdx = ((idx - 2) % #win.Tabs) + 1
		win:SelectTab(win.Tabs[prevIdx].Name)
	end

	-- ==== Открытие / закрытие ====
	function win:Open(instant)
		if win.Visible == true then
			return
		end
		win.Visible = true
		pcall(function()
			win.Wrapper.Visible = true
			win.Fade.Visible = true
		end)

		-- Затемнение
		Motion.Tween({
			Instance = win.Fade,
			Time = instant and 0.001 or 0.22,
			Easing = Easings.outQuad,
			BackgroundTransparency = OPTIONS.Opacity,
		})
		-- Клик по затемнению — закрыть (подключаем один раз)
		if not win.__fadeBound then
			win.__fadeBound = true
			win.Fade.InputBegan:Connect(function()
				pcall(function()
					win:Close()
				end)
			end)
		end

		-- Анимация открытия: масштаб через UIScale + прозрачность (outBack, 0.3с)
		win.Open = true
		pcall(function()
			win.Wrapper.Visible = true
			uiScale.Scale = 0.88
		end)
		local startT = instant and 0.001 or OPTIONS.Duration
		Motion.Tween({
			Instance = nil,
			Time = startT,
			Easing = Easings.outBack,
			OnUpdate = function(t)
				pcall(function()
					uiScale.Scale = Utils.lerp(0.88, 1, t)
					win.Frame.BackgroundTransparency = Utils.lerp(0.55, 0, t)
					win.Shadow.ImageTransparency = Utils.lerp(0.85, 0.35, t)
					win.Glow.ImageTransparency = Utils.lerp(1, 0.9, t)
					stroke.Transparency = Utils.lerp(1, 0.55, t)
				end)
			end,
			OnComplete = function()
				pcall(function()
					uiScale.Scale = 1
					win.Frame.BackgroundTransparency = 0
					win.Shadow.ImageTransparency = 0.35
					win.Glow.ImageTransparency = 0.92
					stroke.Transparency = 0.55
				end)
			end,
		})
		-- первая вкладка
		if win.CurrentTab == nil and #win.Tabs > 0 then
			win:SelectTab(win.Tabs[1].Name, instant)
		end
		-- Уведомление о готовности
		pcall(task.delay, 0.35, function()
			if win.Visible then
				Notify.Info(title .. " загружен", "Нажми K, чтобы скрыть окно", 4)
			end
		end)
	end

	function win:Close()
		if win.Visible == false then
			return
		end
		win.Visible = false
		win.Open = false
		-- Анимация закрытия: масштаб в 0 + прозрачность (inQuad, 0.25с)
		Motion.Tween({
			Instance = nil,
			Time = 0.25,
			Easing = Easings.inQuad,
			OnUpdate = function(t)
				pcall(function()
					uiScale.Scale = Utils.lerp(1, 0, t)
					win.Frame.BackgroundTransparency = Utils.clamp(t * 1.2, 0, 1)
					win.Shadow.ImageTransparency = Utils.lerp(0.35, 1, t)
					win.Glow.ImageTransparency = 1
					stroke.Transparency = Utils.lerp(0.55, 1, t)
				end)
			end,
			OnComplete = function()
				pcall(function()
					win.Wrapper.Visible = false
					win.Fade.Visible = false
					uiScale.Scale = 1
					win.Frame.BackgroundTransparency = 0
					win.Shadow.ImageTransparency = 0.35
					win.Glow.ImageTransparency = 0.92
					stroke.Transparency = 0.55
				end)
			end,
		})
		Motion.Tween({
			Instance = win.Fade,
			Time = 0.22,
			Easing = Easings.inQuad,
			BackgroundTransparency = 1,
		})
	end

	function win:Toggle()
		if win.Visible then
			win:Close()
		else
			win:Open()
		end
	end

	-- ==== Создание вкладки ====
	function win:CreateTab(tabName, tabIcon)
		local tab = {}
		tab.Name = tabName
		tab.TabName = tabName
		tab.Icon = tabIcon or "◆"
		tab.Active = false
		tab.Meta = {Kind = "Tab"}

		-- Кнопка в сайдбаре
		local button = Utils.new("TextButton", {
			Name = "Tab_" .. tostring(tabName),
			Text = "",
			BackgroundColor3 = Theme.C("SurfaceAlt"),
			BackgroundTransparency = 1,
			BorderSizePixel = 0,
			Size = UDim2.new(1, -8, 0, 34),
			AutoButtonColor = false,
			ZIndex = 208,
			Parent = tabList,
		})
		Utils.corner(button, 10)
		Theme.Bind("SurfaceAlt", button, "BackgroundColor3")
		tab.Button = button
		tab.Background = button

		local icon = Utils.new("TextLabel", {
			Text = tostring(tabIcon or "◆"),
			TextColor3 = Theme.C("TextFaint"),
			TextSize = 12,
			BackgroundTransparency = 1,
			Size = UDim2.fromOffset(20, 34),
			Position = UDim2.fromOffset(8, 0),
			ZIndex = 209,
			Parent = button,
		})
		Utils.setFont(icon, "Gotham", "Bold")
		Theme.Bind("TextFaint", icon, "TextColor3")
		tab.IconNode = icon

		local label = Utils.new("TextLabel", {
			Text = tostring(tabName),
			TextColor3 = Theme.C("TextDim"),
			TextSize = 12,
			TextXAlignment = Enum.TextXAlignment.Left,
			BackgroundTransparency = 1,
			Size = UDim2.new(1, -34, 0, 34),
			Position = UDim2.fromOffset(30, 0),
			ZIndex = 209,
			Parent = button,
		})
		Utils.setFont(label, "Gotham", "Medium")
		Theme.Bind("TextDim", label, "TextColor3")
		tab.Label = label

		button.MouseEnter:Connect(function()
			if not tab.Active then
				Motion.Tween({ Instance = button, Time = 0.16, BackgroundTransparency = 0.55 })
				Motion.Tween({ Instance = label, Time = 0.16, TextColor3 = Theme.C("Text") })
			end
		end)
		button.MouseLeave:Connect(function()
			if not tab.Active then
				Motion.Tween({ Instance = button, Time = 0.2, BackgroundTransparency = 1 })
				Motion.Tween({ Instance = label, Time = 0.2, TextColor3 = Theme.C("TextDim") })
			end
		end)
		button.MouseButton1Click:Connect(function()
			win:SelectTab(tabName)
		end)

		-- Страница контента
		local page = Utils.new("ScrollingFrame", {
			Name = "Page_" .. tostring(tabName),
			BackgroundTransparency = 1,
			BorderSizePixel = 0,
			Size = UDim2.fromScale(1, 1),
			Position = UDim2.fromOffset(0, 0),
			CanvasSize = UDim2.new(0, 0, 0, 0),
			AutomaticCanvasSize = Enum.AutomaticSize.Y,
			ScrollBarThickness = 4,
			ScrollBarImageColor3 = Theme.C("Track"),
			ScrollingDirection = Enum.ScrollingDirection.Y,
			Visible = false,
			ClipsDescendants = true,
			ZIndex = 215,
			Parent = pagesHolder,
		})
		Theme.Bind("Track", page, "ScrollBarImageColor3")
		Utils.padding(page, 18, 20, 24, 20)
		Utils.listLayout(page, Enum.FillDirection.Vertical, 10)
		tab.Scroll = page
		tab.Page = page
		tab.Content = page

		-- Заголовок вкладки
		local headerRow = Utils.new("Frame", {
			BackgroundTransparency = 1,
			Size = UDim2.new(1, 0, 0, 42),
			ZIndex = 216,
			Parent = page,
		})
		local headerTitle = Utils.new("TextLabel", {
			Text = tostring(tabName),
			TextColor3 = Theme.C("Text"),
			TextSize = 22,
			BackgroundTransparency = 1,
			Size = UDim2.new(1, -120, 1, 0),
			Position = UDim2.fromOffset(2, 0),
			TextXAlignment = Enum.TextXAlignment.Left,
			ZIndex = 217,
			Parent = headerRow,
		})
		Utils.setFont(headerTitle, "Gotham", "Bold")
		Theme.Bind("Text", headerTitle, "TextColor3")
		tab.HeaderTitle = headerTitle

		local headerSub = Utils.new("TextLabel", {
			Text = tostring(options.SubtitleFor and options.SubtitleFor(tabName) or "Модуль AshUI"),
			TextColor3 = Theme.C("TextFaint"),
			TextSize = 11,
			BackgroundTransparency = 1,
			Size = UDim2.new(1, -120, 0, 14),
			Position = UDim2.fromOffset(4, 26),
			TextXAlignment = Enum.TextXAlignment.Left,
			ZIndex = 217,
			Parent = headerRow,
		})
		Utils.setFont(headerSub, "Gotham", "Regular")
		Theme.Bind("TextFaint", headerSub, "TextColor3")
		tab.HeaderSub = headerSub

		-- Линия под заголовком
		local headerLine = Utils.new("Frame", {
			BackgroundColor3 = Theme.C("BorderSoft"),
			BorderSizePixel = 0,
			Size = UDim2.new(1, 0, 0, 1),
			Position = UDim2.new(0, 0, 1, -2),
			ZIndex = 216,
			Parent = headerRow,
		})
		Utils.corner(headerLine, 999)
		Theme.Bind("BorderSoft", headerLine, "BackgroundColor3")

		tab.Header = headerRow
		win.Tabs[#win.Tabs + 1] = tab
		Registry.RegisterElement(tab, {Kind = "Tab"})

		-- Если это первая вкладка — делаем активной
		if win.CurrentTab == nil then
			win.CurrentTab = tab
			tab.Active = true
			pcall(function()
				page.Visible = true
			end)
			moveIndicator(button, true)
		end
		return tab
	end

	-- ==== Прямой доступ к элементам через win:Create* ====

	-- Все элементы хранятся в win.Elements
	win.Elements = {}

	local function wrap(fn)
		return function(self, ...)
			-- вызов как метод окна: self = win
			local tab = self.CurrentTab
			if tab == nil then
				tab = self:CreateTab("Home")
			end
			local el = fn(tab, ...)
			if el ~= nil then
				table.insert(win.Elements, el)
			end
			return el
		end
	end

	win.CreateLabel = wrap(Elements.CreateLabel)
	win.CreateButton = wrap(Elements.CreateButton)
	win.CreateToggle = wrap(Elements.CreateToggle)
	win.CreateSlider = wrap(Elements.CreateSlider)
	win.CreateProgress = wrap(Elements.CreateProgress)
	win.CreateParagraph = wrap(Elements.CreateParagraph)
	win.CreateStepper = wrap(Elements.CreateStepper)
	win.CreateTextbox = wrap(Elements.CreateTextbox)
	win.CreateKeybind = wrap(Elements.CreateKeybind)
	win.CreateDropdown = wrap(Elements.CreateDropdown)
	win.CreateSection = wrap(Elements.CreateSection)
	win.CreateColorPicker = wrap(Advanced.CreateColorPicker)
	win.CreateListbox = wrap(Advanced.CreateListbox)
	win.CreateAccordion = wrap(Advanced.CreateAccordion)
	win.CreateTooltip = wrap(Advanced.CreateTooltip)
	win.CreateCamera = wrap(Advanced.CreateCamera)

	function win:CreateSpacer(h, opts)
		local tab = self.CurrentTab
		if tab == nil then
			tab = self:CreateTab("Home")
		end
		local sp = Elements.CreateSpacer(tab, h, opts)
		table.insert(win.Elements, sp)
		return sp
	end

	-- ==== Drag ====
	local dragging = false
	Motion.Drag(titleBar, {
		Host = frame,
		Draggable = frame,
		UseAnchor = true,
		Snap = true,
		SnapPad = OPTIONS.SnapPad,
		TopInset = nil,
		OnStart = function()
			-- поднимаем над остальными
			Motion.Tween({ Instance = glow, Time = 0.2, ImageTransparency = 0.8 })
		end,
		OnEnd = function(moved)
			Motion.Tween({ Instance = glow, Time = 0.35, ImageTransparency = 0.92 })
		end,
	})

	-- ==== Хоткей (K) ====
	local uis = Services.UserInputService
	local keyConn = nil
	local function setupHotkey()
		if uis == nil then
			return
		end
		if keyConn then
			pcall(function()
				keyConn:Disconnect()
			end)
		end
		keyConn = uis.InputBegan:Connect(function(input, gpe)
			if gpe or not input.UserInputType == nil then
				return
			end
			if input.UserInputType ~= Enum.UserInputType.Keyboard then
				return
			end
			-- Не перехватываем ввод в текстовых полях
			local focused = false
			pcall(function()
				focused = uis:GetFocusedTextBox() ~= nil
			end)
			if focused then
				return
			end
			if input.KeyCode == OPTIONS.ToggleKey then
				pcall(function()
					win:Toggle()
				end)
			elseif input.KeyCode == Enum.KeyCode.Tab and uis:IsKeyDown(Enum.KeyCode.LeftControl) then
				pcall(function()
					win:NextTab()
				end)
			end
		end)
	end
	setupHotkey()

	-- Смена хоткея из конфига
	function win:SetToggleKey(keyCode)
		OPTIONS.ToggleKey = keyCode
		setupHotkey()
	end

	-- ==== Кнопки заголовка ====
	closeBtn.MouseButton1Click:Connect(function()
		win:Close()
	end)
	themeBtn.MouseButton1Click:Connect(function()
		local list = Theme.List()
		local current = Theme.CurrentId()
		local idx = Utils.indexOf(list, current) or 1
		local nextTheme = list[(idx % #list) + 1]
		Theme.Set(nextTheme)
		Notify.Info("Тема", nextTheme .. " активирована", 2.5)
		-- обновляем подпись иконки
		pcall(function()
			themeBtn.Text = nextTheme:sub(1, 1)
		end)
	end)
	pinBtn.MouseButton1Click:Connect(function()
		OPTIONS.Snap = not OPTIONS.Snap
		pinBtn.Text = OPTIONS.Snap and "●" or "○"
		Motion.Tween({ Instance = pinBtn, Time = 0.2, TextTransparency = 0.4, OnComplete = function()
			Motion.Tween({ Instance = pinBtn, Time = 0.2, TextTransparency = 0 })
		end })
		Notify.Info("Snap", OPTIONS.Snap and "Прилипание включено" or "Прилипание выключено", 2.5)
	end)

	-- Смена размера окна (перетаскивание за нижний правый угол)
	local resizeHandle = Utils.new("Frame", {
		BackgroundColor3 = Theme.C("Border"),
		BackgroundTransparency = 0.4,
		BorderSizePixel = 0,
		Size = UDim2.fromOffset(14, 14),
		Position = UDim2.new(1, -7, 1, -7),
		AnchorPoint = Vector2.new(1, 1),
		ZIndex = 230,
		Visible = false,
		Parent = frame,
	})
	Utils.corner(resizeHandle, 999)
	Theme.Bind("Border", resizeHandle, "BackgroundColor3")
	resizeHandle.MouseEnter:Connect(function()
		resizeHandle.Visible = true
		Motion.Tween({ Instance = resizeHandle, Time = 0.2, BackgroundTransparency = 0 })
	end)
	frame.InputBegan:Connect(function(input)
		-- показываем маркер при наведении на угол
		if input.UserInputType == Enum.UserInputType.MouseMovement then
			local pos = Utils.absolutePos(frame)
			local size = Utils.absoluteSize(frame)
			local mouse = Utils.mousePos()
			if mouse.X > pos.X + size.X - 22 and mouse.Y > pos.Y + size.Y - 22 then
				resizeHandle.Visible = true
			else
				resizeHandle.Visible = false
			end
		end
	end)

	-- Логика ресайза
	local resizing = false
	local resizeStart = nil
	local resizeInputBegan = nil
	local resizeInputChanged = nil
	local resizeInputEnded = nil
	if uis ~= nil then
		resizeInputBegan = uis.InputBegan:Connect(function(input)
			if input.UserInputType == Enum.UserInputType.MouseButton1 then
				local pos = Utils.absolutePos(frame)
				local size = Utils.absoluteSize(frame)
				local mouse = Utils.mousePos()
				if mouse.X > pos.X + size.X - 22 and mouse.Y > pos.Y + size.Y - 22 then
					resizing = true
					resizeStart = {Size = size, Mouse = mouse}
				end
			end
		end)
		resizeInputChanged = uis.InputChanged:Connect(function(input)
			if resizing and input.UserInputType == Enum.UserInputType.MouseMovement then
				local delta = Utils.mousePos() - resizeStart.Mouse
				local newSize = Vector2.new(
					Utils.clamp(resizeStart.Size.X + delta.X, 380, Utils.screenSize().X - 20),
					Utils.clamp(resizeStart.Size.Y + delta.Y, 300, Utils.screenSize().Y - 20)
				)
				win.Size = newSize
				pcall(function()
					frame.Size = UDim2.fromOffset(newSize.X, newSize.Y)
					spaceField.Field.Size = UDim2.fromOffset(
						math.max(math.min(newSize.X - 196, 760), 260),
						math.max(math.min(newSize.Y - 46, 520), 200)
					)
				end)
			end
		end)
		resizeInputEnded = uis.InputEnded:Connect(function(input)
			if resizing and input.UserInputType == Enum.UserInputType.MouseButton1 then
				resizing = false
				Notify.Info("Размер окна", math.floor(win.Size.X) .. " x " .. math.floor(win.Size.Y), 2)
			end
		end)
	end

	-- ==== Прочее ====
	function win:Destroy()
		pcall(function()
			gui:Destroy()
		end)
	end

	function win:GetSize()
		return win.Size
	end

	function win:GetPosition()
		return Utils.get(frame, "Position")
	end

	function win:SetPosition(pos)
		pcall(function()
			frame.Position = pos
		end)
	end

	-- Регистрация
	table.insert(AshUI.Windows, win)
	Registry.RegisterElement(win, {Kind = "Window"})

	-- Стартовая позиция
	pcall(function()
		frame.AnchorPoint = Vector2.new(0, 0)
		frame.Position = UDim2.fromOffset(startPos.X, startPos.Y)
	end)
	return win
end
-- ==== Load — точка входа и демонстрация ====

-- Запуск цикла анимаций
Motion.Start()

-- Планировщик тиков (уведомления + ресайз) — без wait внутри RenderStepped
Motion.OnUpdate(function(dt)
	-- тики тостов
	Notify.Tick(dt)
	-- обновление позиции контейнера тостов при ресайзе экрана
	Notify.Reposition()
end, 1)

-- Раннее уведомление о том, что интерфейс инициализируется
local function bootstrap()
	-- Загружаем сохранённый конфиг (если есть)
	local ok, info = Config.Load({InstantTheme = true})
	if ok then
		Notify.Success("AshUI", "Конфиг загружен (CRC OK) · флагов: " .. tostring(info), 4.5)
	else
		Notify.Info("AshUI " .. OPTIONS.Version, "Создаю интерфейс…", 3.5)
	end
end

-- ==== ДЕМО: ОКНО С 3 ОСНОВНЫМИ + SETTINGS ====

-- Функция для «живого» демо-графика
local demoTimer = 0
local demoProgress = 0
local demoSlider = nil
local demoToggle = nil
local demoFpsLabel = nil
local frameCount = 0
local frameTimer = 0
local currentFps = 60

local win = Window.Create({
	Title		= "AshUI · Lunar",
	Subtitle	= "v" .. OPTIONS.Version .. " · Solara edition",
	SidebarTitle = "Navigation",
	Size		= Vector2.new(900, 580),
	Position	= nil,
})

Notify.Setup(AshUI.CurrentGui)

if win ~= nil then
---------------------------------------------------------------
-- ВКЛАДКА 1: Dashboard — базовые элементы
---------------------------------------------------------------
	local dash = win:CreateTab("Dashboard", "◆")

	win:CreateSection("Основные элементы")
	local greet = win:CreateLabel("Привет! Это демонстрация GUI-библиотеки AshUI.", {Size = 14, ColorKey = "Text"})
	win:CreateParagraph(
		"В этой вкладке собраны базовые элементы: метки, кнопки, тумблеры, слайдеры и прогресс-бар. Все флаги сохраняются в конфиг автоматически.",
		{Size = 12, Height = 64}
	)

	win:CreateSpacer(6)

	local btnRow = win:CreateSection("Кнопки")
	win:CreateButton("Показать уведомление", function()
		Notify.Success("Успех", "Всё работает как часы", 3)
	end, {Style = "Accent", Flag = "Demo_ButtonAccent"})
	win:CreateButton("Обычная кнопка", function()
		Notify.Info("Кнопка", "Нажата обычная кнопка", 2.5)
	end, {Style = "Default"})
	win:CreateButton("Опасное действие", function()
		Notify.Warning("Подтверждение", "Это опасная кнопка — но ничего не произойдёт", 3.5)
	end, {Style = "Danger", Flag = "Demo_ButtonDanger"})

	win:CreateSpacer(6, {Divider = true})

	local toggleSection = win:CreateSection("Тумблеры")
	demoToggle = win:CreateToggle("Показывать декорации (звёзды/луна)", true, function(v)
		if win.SpaceField ~= nil then
			win.SpaceField.Field.Visible = v
		end
		Notify.Info("Переключатель", v and "Декор включён" or "Декор выключен", 2.5)
	end, {Flag = "Demo_ShowDecor"})
	win:CreateToggle("Плавные анимации", true, function(v)
		Theme.SetTransitionSpeed(v and 0.4 or 0.05)
		OPTIONS.Duration = v and 0.3 or 0.08
	end, {Flag = "Demo_SmoothAnim"})
	win:CreateToggle("Отладочный вывод", false, function(v)
		OPTIONS.Debug = v
	end, {Flag = "Demo_Debug"})
	win:CreateToggle("Звуковые эффекты", false, function(v)
		Notify.Info("Звук", v and "Включён (заглушка)" or "Выключен", 2)
	end)

	win:CreateSpacer(6, {Divider = true})

	local sliderSection = win:CreateSection("Слайдеры")
	demoSlider = win:CreateSlider("Громкость интерфейса", 0, 100, 65, "%", function(v)
		-- Демонстрация: меняем прозрачность декора
		if win.SpaceField ~= nil then
			Motion.Tween({
				Instance = win.SpaceField.Field,
				Time = 0.2,
				BackgroundTransparency = Utils.remap(v, 0, 100, 0.3, 1),
			})
		end
	end, {Flag = "Demo_Volume", Step = 1, ColorKey = "Accent"})
	win:CreateSlider("Размытие фона (0..1)", 0, 1, 0.35, "", function(v)
		-- Плавно меняем затемнение за окном
		Motion.Tween({
			Instance = win.Fade,
			Time = 0.25,
			BackgroundTransparency = v,
		})
	end, {Flag = "Demo_Blur", Decimals = 2, ColorKey = "Info"})
	win:CreateSlider("Скорость анимаций", 0.05, 1, 0.3, "s", function(v)
		Theme.SetTransitionSpeed(v)
		OPTIONS.Duration = v
	end, {Flag = "Demo_AnimSpeed", Decimals = 2, ColorKey = "Warning"})

	win:CreateSpacer(6, {Divider = true})
	local progressSection = win:CreateSection("Прогресс")
	local progress = win:CreateProgress("Загрузка модулей", 0.0, {Flag = "Demo_Progress", ColorKey = "Accent"})
	local progress2 = win:CreateProgress("Использование памяти (имитация)", 0.42, {ColorKey = "Info"})

	-- «Живой» прогресс + FPS
	Motion.OnUpdate(function(dt)
		frameCount = frameCount + 1
		frameTimer = frameTimer + dt
		if frameTimer >= 0.5 then
			currentFps = frameCount / frameTimer
			frameCount = 0
			frameTimer = 0
			if demoFpsLabel ~= nil then
				pcall(function()
					demoFpsLabel.Text = string.format("FPS: %d  ·  элементов: %d", math.floor(currentFps), #Registry.Elements)
				end)
			end
		end
		demoTimer = demoTimer + dt
		if demoTimer >= 2.5 then
			demoTimer = 0
			-- плавное «дыхание» прогрессов
			progress:Set(Utils.clamp(progress:GetValue() + Utils.rand(0.05, 0.2), 0, 1), 1.2)
			progress2:Set(Utils.clamp(0.35 + math.sin(os.clock() * 0.6) * 0.25, 0, 1), 1.5)
		end
	end, 3)

	---------------------------------------------------------------
	-- ВКЛАДКА 2: Widgets — продвинутые элементы
	---------------------------------------------------------------
	local widgets = win:CreateTab("Widgets", "◈")

	win:CreateSection("Ввод данных")
	win:CreateTextbox("Имя персонажа", "Введите имя...", "Hero", function(v)
		Notify.Info("Textbox", "Значение: " .. v, 2)
	end, {Flag = "Demo_Name", Enter = true})
	win:CreateTextbox("Заметка (многострочная)", "Заметка...", "—", function(v)
		-- просто сохраняем в флаг
	end, {Flag = "Demo_Note", MultiLine = true, Height = 78})

	win:CreateSpacer(6, {Divider = true})

	local ddSection = win:CreateSection("Выбор")
	win:CreateDropdown("Тема оформления", {Options = Theme.List()}, "Lunar", function(v)
		Theme.Set(v)
		Notify.Info("Тема", "Переключено на " .. v, 2.5)
	end, {Flag = "Demo_Theme"})
	win:CreateDropdown("Качество", {Options = {"Низкое", "Среднее", "Высокое", "Ультра"}}, "Высокое", function(v)
		Notify.Info("Качество", v, 2)
	end, {Flag = "Demo_Quality"})
	win:CreateStepper("Количество эффектов", 5, 0, 50, function(v)
		Notify.Info("Stepper", "Эффектов: " .. v, 2)
	end, {Flag = "Demo_Count", Increment = 1})
	win:CreateKeybind("Быстрое переключение окна", Enum.KeyCode.K, function(key)
		Notify.Info("Keybind", "Назначено: " .. tostring(key and key.Name or key), 2.5)
	end, {Flag = "Demo_QuickKey"})

	win:CreateSpacer(6, {Divider = true})

	local listSection = win:CreateSection("Мультивыбор")
	win:CreateListbox("Включённые модули", {
		Items = {
			{Label = "Звёзды", Value = "Stars"},
			{Label = "Аврора", Value = "Aurora"},
			{Label = "Луна", Value = "Moon"},
			{Label = "Свечение", Value = "Glow"},
			{Label = "Shimmer", Value = "Shimmer"},
			{Label = "Snap", Value = "Snap"},
		},
		Multiselect = true,
	}, {"Stars", "Aurora", "Moon"}, function(sel)
		-- применяем выбор к декорациям
		local function on(name)
			return Utils.contains(sel, name)
		end
		pcall(function()
			win.SpaceField.MoonRoot.Visible = on("Moon")
		end)
	end, {Flag = "Demo_Modules"})

	win:CreateSpacer(6, {Divider = true})

	local colorSection = win:CreateSection("Цвета")
	win:CreateColorPicker("Акцент интерфейса", Color3.fromRGB(200, 205, 220), function(c)
		-- Меняем акцент на лету (через временную палитру)
		Theme.Register("Custom", "Custom", {
			Accent = c,
			AccentText = Utils.readable(c),
		})
		Theme.Set("Custom")
	end, {Flag = "Demo_AccentColor"})
	win:CreateParagraph("Цвет применяется ко всем интерактивным элементам: кнопкам, тумблерам, ползункам и индикатору вкладки.", {Height = 52, Size = 11})

	---------------------------------------------------------------
	-- ВКЛАДКА 3: Anim — анимации и камера
	---------------------------------------------------------------
	local animTab = win:CreateTab("Anim", "✦")

	win:CreateSection("Анимации")
	win:CreateButton("Открыть / Закрыть (анимация)", function()
		win:Toggle()
	end, {Style = "Accent"})
	win:CreateButton("Shake окна", function()
		Motion.Shake(win.Frame, {Strength = 12, Time = 0.4})
		Notify.Warning("Shake", "Тряска окна!", 2)
	end)
	win:CreateButton("Pulse фона", function()
		Anim.PulseFrame(win.Glow, {Color = Theme.C("Accent"), To = Theme.C("Info"), Time = 1.5})
	end)
	win:CreateButton("Спать звёзд", function()
		-- мерцание: ускоряем/замедляем анимации
		Theme.SetTransitionSpeed(0.15)
		Notify.Info("Звёзды", "Ускорил мерцание на 2 сек", 2)
		task.delay(2, function()
			Theme.SetTransitionSpeed(0.4)
		end)
	end)
	win:CreateSpacer(4)

	win:CreateButton("Сбросить размер окна", function()
		win:SetSize(Vector2.new(900, 580))
		Notify.Info("Окно", "Размер сброшен", 2)
	end)

	win:CreateSpacer(6, {Divider = true})

	-- Аккордеон с примерами
	win:CreateAccordion("Как это работает?", {
		Items = {
			{Text = "• Создание окна: Window.Create({Title=..., Size=...})"},
			{Text = "• Вкладки: win:CreateTab('Имя', 'иконка')"},
			{Text = "• Элементы: win:CreateToggle(...), win:CreateSlider(...)"},
			{Text = "• Флаги: {Flag='Имя'} — значение сохраняется в конфиг"},
			{Text = "• Темы: Theme.Set('Abyss') — плавная интерполяция цветов"},
		},
	}, true)

	win:CreateSpacer(6, {Divider = true})

	local camSection = win:CreateSection("Камера")
	win:CreateCamera("Управление камерой", {NoFlag = true}, 70)
	win:CreateParagraph("Модуль камеры меняет FOV, блокирует камеру и добавляет эффект тряски. Значение FOV восстанавливается при закрытии окна.", {Height = 46, Size = 11})

	---------------------------------------------------------------
	-- ВКЛАДКА 4: Settings — темы, конфиг
	---------------------------------------------------------------
	local settings = win:CreateTab("Settings", "⚙")

	win:CreateSection("Тема")
	local themeDropdown = win:CreateDropdown("Выберите тему", {Options = Theme.List()}, Theme.CurrentId(), function(v)
		Theme.Set(v)
		Registry.Set("Settings_Theme", v)
		Notify.Info("Тема", "Применена: " .. v, 2.5)
	end, {Flag = "Settings_Theme"})

	win:CreateSpacer(4)
	win:CreateButton("Следующая тема →", function()
		local list = Theme.List()
		local idx = Utils.indexOf(list, Theme.CurrentId()) or 1
		local nextTheme = list[(idx % #list) + 1]
		Theme.Set(nextTheme)
		themeDropdown:SetValue(nextTheme)
		Registry.Set("Settings_Theme", nextTheme)
	end, {Style = "Default"})

	win:CreateSpacer(4)
	-- Быстрый выбор популярных тем
	local themeRow = win:CreateSection("Быстрый выбор")
	local themes = {"Lunar", "Blood", "Ocean", "Forest", "Abyss", "Neon", "Midnight", "Sunset", "Mono"}
	for _, tName in ipairs(themes) do
		local btn = win:CreateButton(tName, function()
			Theme.Set(tName)
			themeDropdown:SetValue(tName)
			Registry.Set("Settings_Theme", tName)
			Notify.Success("Тема", tName, 2)
		end, {Style = "Default", Height = 30, Shimmer = false})
	end

	win:CreateSpacer(8, {Divider = true})

	local configSection = win:CreateSection("Конфигурация")
	local statusLabel = win:CreateLabel("Файл: " .. Config.getPath(), {Size = 11, ColorKey = "TextFaint"})

	win:CreateButton("Сохранить конфиг", function()
		local ok, crc = Config.Save()
		if ok then
			Notify.Success("Сохранено", "CRC: " .. tostring(crc), 3.5)
		else
			Notify.Error("Ошибка сохранения", "Проверьте права на запись", 4)
		end
		Motion.Flash(statusLabel, Theme.C("Success"), 0.5)
	end, {Style = "Accent"})

	win:CreateButton("Загрузить конфиг", function()
		local ok, info2 = Config.Load({InstantTheme = true})
		if ok then
			Notify.Success("Загружено", "Флагов восстановлено: " .. tostring(info2), 3.5)
			Registry.RefreshElements()
			Motion.Flash(statusLabel, Theme.C("Info"), 0.5)
		else
			Notify.Error("Ошибка загрузки", tostring(info2), 4)
		end
	end, {Style = "Default"})

	win:CreateButton("Сбросить всё", function()
		Config.Reset()
		themeDropdown:SetValue("Lunar")
		Registry.RefreshElements()
		Notify.Warning("Сброшено", "Все значения вернулись к заводским", 3.5)
	end, {Style = "Danger"})

	win:CreateSpacer(4)
	win:CreateButton("Скопировать в буфер", function()
		if Config.Copy() then
			Notify.Success("Буфер", "Конфиг скопирован (JSON+CRC)", 3)
		else
			Notify.Error("Буфер", "setclipboard недоступен", 3)
		end
	end, {Style = "Default", Shimmer = false})

	win:CreateButton("Удалить файл конфига", function()
		Config.Delete()
		Notify.Info("Удалено", "Файл конфига удалён", 3)
	end, {Style = "Danger", Shimmer = false})

	win:CreateSpacer(6, {Divider = true})

	local miscSection = win:CreateSection("Прочее")
	win:CreateToggle("Автосохранение (каждые 45с)", true, function(v)
		OPTIONS.AutoSave = v and 45 or 0
		Notify.Info("Автосохранение", v and "Включено" or "Выключено", 2.5)
	end, {Flag = "Settings_AutoSave"})
	win:CreateToggle("Показывать FPS", true, function(v)
		Registry.Set("Settings_ShowFps", v)
	end, {Flag = "Settings_ShowFps"})
	win:CreateToggle("Уведомления", true, function(v)
		Notify.Enabled = v
		Notify.ClearAll()
		if v then
			Notify.Success("Уведомления", "Включены", 2)
		end
	end, {Flag = "Settings_Notifications"})
	win:CreateToggle("Скрывать курсор в Roblox", false, function(v)
		-- в эксплоитах это обычно недоступно, поэтому просто уведомляем
		Notify.Info("Курсор", v and "Запрошен" or "Возвращён системой Roblox", 2.5)
	end, {Flag = "Settings_Cursor"})
	win:CreateSlider("Задержка уведомлений (с)", 1, 10, 5, "s", function(v)
		OPTIONS.ToastLife = v
		Notify.Info("Тосты", "Время жизни: " .. v .. "с", 2.5)
	end, {Flag = "Settings_ToastLife"})

	win:CreateSpacer(4)
	win:CreateSlider("Прозрачность фона", 0, 0.8, 0, "", function(v)
		OPTIONS.Opacity = v
		if win.Visible then
			Motion.Tween({ Instance = win.Fade, Time = 0.2, BackgroundTransparency = v })
		end
	end, {Flag = "Settings_Opacity", Decimals = 2, ColorKey = "Info"})

	win:CreateSpacer(4)
	win:CreateButton("Все уведомления (тест)", function()
		Notify.Success("Success", "Зелёный — всё прошло успешно", 3)
		task.delay(0.3, function() Notify.Error("Error", "Красный — что-то пошло не так", 3) end)
		task.delay(0.6, function() Notify.Warning("Warning", "Жёлтый — проверь настройки", 3) end)
		task.delay(0.9, function() Notify.Info("Info", "Синий — просто информация", 3) end)
	end, {Style = "Default"})

	win:CreateSpacer(6, {Divider = true})

	local aboutSection = win:CreateSection("О библиотеке")
	win:CreateParagraph(
		string.format(
			"AshUI v%s\nМодулей GUI: %d\nПалитр: %d\nФлагов: %d\nЭкран: %dx%d\nFPS: %d",
			OPTIONS.Version, 20, #Theme.List(), Registry.Count(),
			math.floor(Utils.screenSize().X), math.floor(Utils.screenSize().Y), math.floor(currentFps)
		),
		{Height = 96, Size = 11, Font = "Code", ColorKey = "TextDim"}
	)
	demoFpsLabel = win:CreateLabel("FPS: —  ·  элементов: 0", {Size = 11, ColorKey = "Accent"})

	---------------------------------------------------------------
	-- ФИНАЛЬНАЯ ИНИЦИАЛИЗАЦИЯ
	---------------------------------------------------------------
	-- Открываем окно после короткой задержки (плавный старт)
	pcall(task.delay, 0.25, function()
		win:Open()
		bootstrap()
		pcall(task.delay, 0.6, function()
			Notify.Info("Демо", "Клавиша K — скрыть/показать окно", 4)
		end)
	end)

	-- Автосохранение
	Config.StartAutoSave()

	-- При выходе из игры — сохраняем
	if Services.game ~= nil then
		pcall(function()
			Services.game:GetPropertyChangedSignal("IsClosing"):Connect(function()
				if game.IsClosing then
					Config.Save(true)
				end
			end)
		end)
	end
end

-- ==== ХОТКЕЙ ГЛОБАЛЬНО (на случай если окно скрыто) ====

AshUI.ToggleWindow = function()
	if AshUI.CurrentWindow ~= nil then
		pcall(function()
			AshUI.CurrentWindow:Toggle()
		end)
	end
end

-- Глобальные хоткеи как отдельные обработчики (надёжнее, чем в Window)
do
	local uis = Services.UserInputService
	if uis ~= nil then
		pcall(function()
			uis.InputBegan:Connect(function(input, gpe)
				if gpe or input.UserInputType ~= Enum.UserInputType.Keyboard then
					return
				end
				local focused = false
				pcall(function()
					focused = uis:GetFocusedTextBox() ~= nil
				end)
				if focused then
					return
				end
				if input.KeyCode == OPTIONS.ToggleKey then
					AshUI.ToggleWindow()
				end
			end)
		end)
	end
end

-- ==== ФИНАЛ ====

AshUI.Loaded = true
AshUI.Modules.Window.Create = Window.Create
AshUI.Modules.Window.CreateWindow = Window.Create

-- Список всех модулей
AshUI.ModuleList = {
	"Utils", "Theme", "Motion", "Anim", "Registry", "Config",
	"Notify", "Elements", "ElementsAdvanced", "Window", "Load",
}

-- Доступ к элементам через AshUI.Elements
AshUI.Elements = Elements
AshUI.ElementsAdvanced = Advanced

-- Финальное сообщение в консоль
pcall(function()
	print(string.format("[AshUI] v%s загружен · %d палитр · %d модулей · флагов: %d",
		OPTIONS.Version, #Theme.List(), #AshUI.ModuleList, Registry.Count()))
end)

return AshUI
