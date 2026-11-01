--[[
    DEAHT HUB | Admin UI
    Version: 1.0.0
    Displayed creator: Sr_Script

    Place this code inside a LocalScript (for example, StarterPlayerScripts).
    This is a UI-only version: admin buttons are visual demonstrations and do not
    execute commands, affect other players, or send chat messages.

    Discord: replace DISCORD_INVITE with your server invite.
]]

local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local StarterGui = game:GetService("StarterGui")

local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")

local VERSION = "1.0.0"
local OWNER_NAME = "Sr_Script"
local DISCORD_INVITE = "https://discord.gg/COLOQUE-SEU-CONVITE"

local C = {
    Background = Color3.fromRGB(7, 7, 9),
    Panel = Color3.fromRGB(12, 12, 15),
    Panel2 = Color3.fromRGB(18, 15, 18),
    Panel3 = Color3.fromRGB(25, 12, 16),
    Red = Color3.fromRGB(235, 25, 52),
    RedBright = Color3.fromRGB(255, 48, 70),
    RedDark = Color3.fromRGB(105, 12, 27),
    White = Color3.fromRGB(218, 70, 92),
    Muted = Color3.fromRGB(133, 39, 55),
    Line = Color3.fromRGB(91, 12, 30),
    Green = Color3.fromRGB(235, 25, 52),
}

local function create(className, props, parent)
    local object = Instance.new(className)
    for key, value in pairs(props or {}) do
        object[key] = value
    end
    object.Parent = parent
    return object
end

local function corner(parent, radius)
    return create("UICorner", {CornerRadius = UDim.new(0, radius or 8)}, parent)
end

local function stroke(parent, color, thickness, transparency)
    return create("UIStroke", {
        Color = color or C.Line,
        Thickness = thickness or 1,
        Transparency = transparency or 0,
        ApplyStrokeMode = Enum.ApplyStrokeMode.Border,
    }, parent)
end

-- Centralized tween control prevents overlapping light animations from fighting.
local activeTweens = setmetatable({}, {__mode = "k"})
local effectsPaused = false
local gui, panel

local function cancelPanelTweens()
    for object, animation in pairs(activeTweens) do
        if object and object.Parent and panel and (object == panel or object:IsDescendantOf(panel)) then
            pcall(function() animation:Cancel() end)
            activeTweens[object] = nil
        end
    end
end

local function tween(object, duration, properties, style, direction)
    if not object or not object.Parent then return nil end
    local info = TweenInfo.new(
        duration,
        style or Enum.EasingStyle.Quart,
        direction or Enum.EasingDirection.Out
    )

    -- While minimized, background effects stay paused instead of accumulating
    -- competing tweens that can flash or jump when the panel is restored.
    if effectsPaused and panel and (object == panel or object:IsDescendantOf(panel)) then
        return TweenService:Create(object, info, properties)
    end

    local previous = activeTweens[object]
    if previous then
        pcall(function() previous:Cancel() end)
    end

    local animation = TweenService:Create(object, info, properties)
    activeTweens[object] = animation
    animation.Completed:Connect(function()
        if activeTweens[object] == animation then
            activeTweens[object] = nil
        end
    end)
    animation:Play()
    return animation
end

local old = PlayerGui:FindFirstChild("DeahtHubUI")
if old then
    old:Destroy()
end

gui = create("ScreenGui", {
    Name = "DeahtHubUI",
    ResetOnSpawn = false,
    IgnoreGuiInset = true,
    ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
    DisplayOrder = 50,
}, PlayerGui)

-- Toast de notificacao
local toast = create("Frame", {
    Name = "Toast",
    AnchorPoint = Vector2.new(0.5, 0),
    Position = UDim2.new(0.5, 0, 0, -70),
    Size = UDim2.fromOffset(310, 48),
    BackgroundColor3 = C.Panel2,
    BackgroundTransparency = 0.05,
    BorderSizePixel = 0,
    ZIndex = 100,
}, gui)
corner(toast, 22)
stroke(toast, C.Red, 1, 0.15)

local toastText = create("TextLabel", {
    BackgroundTransparency = 1,
    Position = UDim2.fromOffset(12, 0),
    Size = UDim2.new(1, -24, 1, 0),
    Font = Enum.Font.GothamMedium,
    Text = "",
    TextColor3 = C.White,
    TextSize = 13,
    TextXAlignment = Enum.TextXAlignment.Center,
    ZIndex = 101,
}, toast)

local toastQueue = {}
local toastBusy = false
local toastIcon = create("TextLabel", {
    BackgroundTransparency = 1, Position = UDim2.fromOffset(12, 0),
    Size = UDim2.fromOffset(28, 48), Font = Enum.Font.GothamBlack,
    Text = "+", TextColor3 = C.RedBright, TextSize = 19, ZIndex = 102,
}, toast)
local toastLine = create("Frame", {
    Position = UDim2.new(0, 0, 1, -2), Size = UDim2.new(0, 0, 0, 2),
    BackgroundColor3 = C.RedBright, BorderSizePixel = 0, ZIndex = 102,
}, toast)
local function notify(message, kind)
    table.insert(toastQueue, {message = tostring(message), kind = kind or "info"})
    if toastBusy then return end
    toastBusy = true
    task.spawn(function()
        while #toastQueue > 0 and gui.Parent do
            local item = table.remove(toastQueue, 1)
            toastText.Text = item.message
            toastIcon.Text = item.kind == "success" and "+" or (item.kind == "warning" and "!" or "+")
            toastIcon.TextColor3 = C.RedBright
            tween(toast, 0.32, {Position = UDim2.new(0.5, 0, 0, 22)}, Enum.EasingStyle.Back)
            toastLine.Size = UDim2.new(0, 0, 0, 2)
            tween(toastLine, 2.35, {Size = UDim2.new(1, 0, 0, 2)}, Enum.EasingStyle.Linear)
            task.wait(2.45)
            tween(toast, 0.28, {Position = UDim2.new(0.5, 0, 0, -70)})
            task.wait(0.32)
        end
        toastBusy = false
    end)
end

-- Tela de carregamento
local loading = create("Frame", {
    Name = "Loading",
    Size = UDim2.fromScale(1, 1),
    BackgroundColor3 = Color3.fromRGB(3, 3, 5),
    BorderSizePixel = 0,
    ZIndex = 20,
}, gui)

-- Premium loading atmosphere: red rain, controlled lightning, dripping trails and corner webs.
local loadingFX = create("Frame", {
    Name = "LoadingEffects", Size = UDim2.fromScale(1, 1),
    BackgroundTransparency = 1, ClipsDescendants = true, ZIndex = 20,
}, loading)

-- Red bat silhouettes orbit inward; no floating letters or random symbols.
task.spawn(function()
    while loading.Parent and loading.Visible do
        for i = 1, 18 do
            local angle = math.rad(i * 47)
            local radius = 0.43 - (i % 6) * 0.012
            local bat = create("Frame", {
                BackgroundTransparency = 1, AnchorPoint = Vector2.new(0.5, 0.5),
                Position = UDim2.fromScale(0.5 + math.cos(angle) * radius, 0.5 + math.sin(angle) * radius),
                Size = UDim2.fromOffset(26, 12), ZIndex = 20,
            }, loadingFX)
            local body = create("Frame", {AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
                Size = UDim2.fromOffset(4, 9), BackgroundColor3 = C.RedBright, BorderSizePixel = 0, ZIndex = 21}, bat)
            corner(body, 4)
            local leftWing = create("Frame", {AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(0.5, -1, 0.5, 0),
                Size = UDim2.fromOffset(12, 5), Rotation = -18, BackgroundColor3 = C.RedDark, BorderSizePixel = 0, ZIndex = 20}, bat)
            local rightWing = create("Frame", {AnchorPoint = Vector2.new(0, 0.5), Position = UDim2.new(0.5, 1, 0.5, 0),
                Size = UDim2.fromOffset(12, 5), Rotation = 18, BackgroundColor3 = C.RedDark, BorderSizePixel = 0, ZIndex = 20}, bat)
            local target = UDim2.fromScale(0.5 + math.cos(angle + 1.55) * 0.035, 0.5 + math.sin(angle + 1.55) * 0.035)
            tween(bat, 2.4 + (i % 4) * 0.16, {Position = target, Rotation = bat.Rotation + 260}, Enum.EasingStyle.Quad, Enum.EasingDirection.In)
            tween(leftWing, 0.32, {Rotation = 22}, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut)
            tween(rightWing, 0.32, {Rotation = -22}, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut)
            task.delay(2.9, function() if bat and bat.Parent then bat:Destroy() end end)
        end
        task.wait(1.25)
    end
end)

-- Atmospheric red rain falling across the loading screen.
for i = 1, 54 do
    local rain = create("Frame", {
        Position = UDim2.new((i * 37 % 100) / 100, 0, -0.2 - (i % 7) * 0.08, 0),
        Size = UDim2.fromOffset(1 + (i % 2), 18 + (i % 5) * 8),
        Rotation = 8, BackgroundColor3 = (i % 4 == 0) and C.RedBright or C.RedDark,
        BackgroundTransparency = 0.35 + (i % 4) * 0.12, BorderSizePixel = 0, ZIndex = 20,
    }, loadingFX)
    task.spawn(function()
        while rain.Parent and loading.Visible do
            rain.Position = UDim2.new(rain.Position.X.Scale, 0, -0.25, 0)
            tween(rain, 1.4 + (i % 8) * 0.16, {Position = UDim2.new(rain.Position.X.Scale, 0, 1.15, 0), BackgroundTransparency = 0.82}, Enum.EasingStyle.Linear)
            task.wait(1.45 + (i % 8) * 0.16)
        end
    end)
end

-- Slow crimson drips from the top edge.
for i = 1, 13 do
    local drip = create("Frame", {
        Position = UDim2.new((i * 71 % 96 + 2) / 100, 0, -0.08, 0),
        Size = UDim2.fromOffset(2 + (i % 3), 20 + (i % 5) * 9),
        BackgroundColor3 = C.RedBright, BackgroundTransparency = 0.18 + (i % 3) * 0.16,
        BorderSizePixel = 0, ZIndex = 21,
    }, loadingFX)
    corner(drip, 5)
    task.spawn(function()
        while drip.Parent and loading.Visible do
            drip.Position = UDim2.new(drip.Position.X.Scale, 0, -0.12, 0)
            tween(drip, 2.8 + (i % 5) * 0.42, {Position = UDim2.new(drip.Position.X.Scale, 0, 1.12, 0), BackgroundTransparency = 0.72}, Enum.EasingStyle.Sine, Enum.EasingDirection.In)
            task.wait(3.0 + (i % 5) * 0.42)
        end
    end)
end

-- Corner spiderwebs built from thin crimson strands.
local function addLoadingWeb(anchor, sx, sy)
    local web = create("Frame", {AnchorPoint = anchor, Position = UDim2.new(sx, 0, sy, 0),
        Size = UDim2.fromOffset(150, 150), BackgroundTransparency = 1, ZIndex = 22}, loadingFX)
    local originX = anchor.X == 0 and 0 or 1
    local originY = anchor.Y == 0 and 0 or 1
    for ray = 1, 7 do
        local angle = math.rad((ray - 1) * 15 + 8)
        local line = create("Frame", {AnchorPoint = Vector2.new(0, 0.5),
            Position = UDim2.new(originX, 0, originY, 0), Size = UDim2.fromOffset(112, 1),
            Rotation = (anchor.X == 0 and 1 or -1) * (anchor.Y == 0 and 1 or -1) * math.deg(angle),
            BackgroundColor3 = C.RedDark, BackgroundTransparency = 0.28,
            BorderSizePixel = 0, ZIndex = 22}, web)
    end
    for ring = 1, 4 do
        local arc = create("Frame", {AnchorPoint = Vector2.new(originX, originY),
            Position = UDim2.new(originX, 0, originY, 0), Size = UDim2.fromOffset(24 + ring * 25, 24 + ring * 25),
            BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 22}, web)
        corner(arc, 100)
        stroke(arc, C.RedDark, 1, 0.2 + ring * 0.08)
    end
    return web
end
addLoadingWeb(Vector2.new(0, 0), 0, 0)
addLoadingWeb(Vector2.new(1, 0), 1, 0)
addLoadingWeb(Vector2.new(0, 1), 0, 1)
addLoadingWeb(Vector2.new(1, 1), 1, 1)

-- Occasional red lightning flashes, subtle and brief.
local lightningFlash = create("Frame", {Size = UDim2.fromScale(1, 1), BackgroundColor3 = C.RedBright,
    BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 23}, loadingFX)
task.spawn(function()
    while loading.Parent and loading.Visible do
        task.wait(math.random(22, 42) / 10)
        if not loading.Visible then break end
        tween(lightningFlash, 0.07, {BackgroundTransparency = 0.91}, Enum.EasingStyle.Linear)
        task.wait(0.08)
        tween(lightningFlash, 0.18, {BackgroundTransparency = 1}, Enum.EasingStyle.Linear)
        task.wait(0.11)
        tween(lightningFlash, 0.05, {BackgroundTransparency = 0.94}, Enum.EasingStyle.Linear)
        task.wait(0.06)
        tween(lightningFlash, 0.22, {BackgroundTransparency = 1}, Enum.EasingStyle.Linear)
    end
end)

local loadingGlow = create("Frame", {
    AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.fromScale(0.5, 0.47),
    Size = UDim2.fromOffset(500, 260),
    BackgroundColor3 = C.RedDark,
    BackgroundTransparency = 0.88,
    BorderSizePixel = 0,
    ZIndex = 20,
}, loading)
corner(loadingGlow, 100)

local loadingTitle = create("TextLabel", {
    AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.fromScale(0.5, 0.405),
    Size = UDim2.fromOffset(560, 76),
    BackgroundTransparency = 1,
    Font = Enum.Font.GothamBlack,
    Text = "DEAHT",
    TextColor3 = C.RedBright,
    TextSize = 58,
    TextXAlignment = Enum.TextXAlignment.Center,
    ZIndex = 21,
}, loading)
local titleGradient = create("UIGradient", {
    Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, Color3.fromRGB(255, 95, 110)),
        ColorSequenceKeypoint.new(0.5, C.RedBright),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(160, 0, 28)),
    }),
    Rotation = 0,
}, loadingTitle)
local loadingHubWord = create("TextLabel", {
    AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.515),
    Size = UDim2.fromOffset(300, 28), BackgroundTransparency = 1,
    Font = Enum.Font.GothamBlack, Text = "H U B", TextColor3 = Color3.fromRGB(155, 18, 43),
    TextSize = 17, ZIndex = 22,
}, loading)
local loadingSeal = create("Frame", {
    AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.46),
    Size = UDim2.fromOffset(420, 176), BackgroundTransparency = 1, ZIndex = 20,
}, loading)
corner(loadingSeal, 34)
local sealStroke = stroke(loadingSeal, Color3.fromRGB(112, 9, 31), 1.4, 0.25)
local sealInner = create("Frame", {
    AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
    Size = UDim2.new(1, -18, 1, -18), BackgroundTransparency = 1, ZIndex = 20,
}, loadingSeal)
corner(sealInner, 28)
stroke(sealInner, Color3.fromRGB(55, 7, 18), 1, 0.12)
local loadingReflection = create("Frame", {
    Position = UDim2.new(-0.3, 0, 0, 0), Size = UDim2.new(0.18, 0, 1, 0),
    BackgroundColor3 = Color3.fromRGB(190, 20, 52), BackgroundTransparency = 0.78,
    BorderSizePixel = 0, ZIndex = 23,
}, loadingSeal)
local reflectionGradient = create("UIGradient", {
    Rotation = 0,
    Transparency = NumberSequence.new({NumberSequenceKeypoint.new(0, 1), NumberSequenceKeypoint.new(0.5, 0.18), NumberSequenceKeypoint.new(1, 1)}),
}, loadingReflection)
for i = 1, 18 do
    local dot = create("Frame", {
        AnchorPoint = Vector2.new(0.5, 0.5),
        Position = UDim2.fromScale(((i * 37) % 94 + 3) / 100, ((i * 23) % 86 + 7) / 100),
        Size = UDim2.fromOffset(2 + (i % 3), 2 + (i % 3)),
        BackgroundColor3 = (i % 2 == 0) and C.RedBright or C.RedDark,
        BackgroundTransparency = 0.22 + (i % 4) * 0.13, BorderSizePixel = 0, ZIndex = 21,
    }, loadingFX)
    corner(dot, 8)
    task.spawn(function()
        while dot.Parent and loading.Visible do
            tween(dot, 1.4 + (i % 3) * 0.3, {BackgroundTransparency = 0.78, Size = UDim2.fromOffset(5, 5)}, Enum.EasingStyle.Sine)
            task.wait(1.4 + (i % 3) * 0.3)
            if dot.Parent then tween(dot, 1.2, {BackgroundTransparency = 0.2, Size = UDim2.fromOffset(2, 2)}, Enum.EasingStyle.Sine) end
            task.wait(1.2)
        end
    end)
end
task.spawn(function()
    while loading.Parent and loading.Visible do
        tween(loadingReflection, 2.8, {Position = UDim2.new(1.15, 0, 0, 0)}, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut)
        task.wait(0.3)
        if loadingReflection.Parent then loadingReflection.Position = UDim2.new(-0.3, 0, 0, 0) end
        task.wait(1.1)
    end
end)
task.spawn(function()
    while loading.Parent and loading.Visible do
        tween(titleGradient, 2.2, {Rotation = titleGradient.Rotation + 360}, Enum.EasingStyle.Linear)
        task.wait(2.2)
    end
end)

local loadingSubtitle = create("TextLabel", {
    AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.fromScale(0.5, 0.555),
    Size = UDim2.fromOffset(300, 22),
    BackgroundTransparency = 1,
    Font = Enum.Font.Gotham,
    Text = "H U B   /   INITIALIZING INTERFACE",
    TextColor3 = C.Muted,
    TextSize = 10,
    ZIndex = 21,
}, loading)

local loadingBarBack = create("Frame", {
    AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.fromScale(0.5, 0.655),
    Size = UDim2.fromOffset(310, 7),
    BackgroundColor3 = Color3.fromRGB(35, 25, 29),
    BorderSizePixel = 0,
    ZIndex = 21,
}, loading)
corner(loadingBarBack, 8)

local loadingBar = create("Frame", {
    Size = UDim2.fromScale(0, 1),
    BackgroundColor3 = C.RedBright,
    BorderSizePixel = 0,
    ZIndex = 22,
}, loadingBarBack)
corner(loadingBar, 8)

local loadingPercent = create("TextLabel", {
    AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.fromScale(0.5, 0.705),
    Size = UDim2.fromOffset(100, 24),
    BackgroundTransparency = 1,
    Font = Enum.Font.GothamBold,
    Text = "0%",
    TextColor3 = C.RedBright,
    TextSize = 14,
    ZIndex = 21,
}, loading)

local loadingFooter = create("TextLabel", {
    AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.fromScale(0.5, 0.94),
    Size = UDim2.fromOffset(300, 20),
    BackgroundTransparency = 1,
    Font = Enum.Font.Gotham,
    Text = "BY SR_SCRIPT  /  VERSION " .. VERSION,
    TextColor3 = Color3.fromRGB(105, 24, 42),
    TextSize = 9,
    ZIndex = 21,
}, loading)

-- Painel principal: horizontal, retangular e responsivo
panel = create("Frame", {
    Name = "MainPanel",
    AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.fromScale(0.5, 0.53),
    Size = UDim2.new(0.88, 0, 0.82, 0),
    BackgroundColor3 = C.Background,
    BackgroundTransparency = 0.02,
    BorderSizePixel = 0,
    Visible = false,
    ClipsDescendants = true,
    ZIndex = 2,
}, gui)
corner(panel, 38)
local panelStroke = stroke(panel, C.Red, 2.2, 0.04)
local panelWatermark = create("Frame", {
    Name = "CrimsonWordmarkTexture", Position = UDim2.new(0, 9, 0, 80),
    Size = UDim2.new(0, 132, 1, -92), BackgroundColor3 = Color3.fromRGB(8, 3, 7),
    BackgroundTransparency = 0.18, BorderSizePixel = 0, ZIndex = 4,
}, panel)
corner(panelWatermark, 24)
stroke(panelWatermark, Color3.fromRGB(92, 8, 28), 1.2, 0.24)
local watermarkTitle = create("TextLabel", {
    AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.40),
    Size = UDim2.new(1, -10, 0, 30), BackgroundTransparency = 1,
    Font = Enum.Font.GothamBlack, Text = "DEAHT", TextColor3 = Color3.fromRGB(125, 13, 39),
    TextSize = 20, TextXAlignment = Enum.TextXAlignment.Center, ZIndex = 5,
}, panelWatermark)
local watermarkHub = create("TextLabel", {
    AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.48),
    Size = UDim2.new(1, -10, 0, 18), BackgroundTransparency = 1,
    Font = Enum.Font.GothamBold, Text = "H U B", TextColor3 = Color3.fromRGB(190, 20, 52),
    TextSize = 10, TextXAlignment = Enum.TextXAlignment.Center, ZIndex = 5,
}, panelWatermark)
-- Fine crimson corner webs on the panel watermark for a restrained premium finish.
local function addPanelWeb(parent, anchor, xScale, yScale)
    local web = create("Frame", {AnchorPoint = anchor, Position = UDim2.new(xScale, 0, yScale, 0),
        Size = UDim2.fromOffset(58, 58), BackgroundTransparency = 1, ZIndex = 5}, parent)
    local ox, oy = anchor.X, anchor.Y
    for ray = 1, 5 do
        local line = create("Frame", {AnchorPoint = Vector2.new(0, 0.5), Position = UDim2.new(ox, 0, oy, 0),
            Size = UDim2.fromOffset(48, 1), Rotation = (ray - 3) * 19 * (ox == 0 and 1 or -1) * (oy == 0 and 1 or -1),
            BackgroundColor3 = C.RedDark, BackgroundTransparency = 0.28, BorderSizePixel = 0, ZIndex = 5}, web)
    end
    for ring = 1, 3 do
        local arc = create("Frame", {AnchorPoint = anchor, Position = UDim2.new(ox, 0, oy, 0),
            Size = U
