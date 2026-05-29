local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local HttpService = game:GetService("HttpService")
local UserInputService = game:GetService("UserInputService")
local MarketplaceService = game:GetService("MarketplaceService")
local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")

if PlayerGui:FindFirstChild("ScriptBloxUI") then
    PlayerGui.ScriptBloxUI:Destroy()
end

local UITransparency = 0.3

local Colors = {
    Background = Color3.fromRGB(17, 17, 27),
    Secondary = Color3.fromRGB(24, 24, 37),
    Card = Color3.fromRGB(30, 30, 46),
    Primary = Color3.fromRGB(137, 180, 250),
    Green = Color3.fromRGB(166, 227, 161),
    Red = Color3.fromRGB(243, 139, 168),
    Yellow = Color3.fromRGB(249, 226, 175),
    Orange = Color3.fromRGB(250, 179, 135),
    Purple = Color3.fromRGB(203, 166, 247),
    Teal = Color3.fromRGB(148, 226, 213),
    Pink = Color3.fromRGB(245, 194, 231),
    Cyan = Color3.fromRGB(137, 220, 235),
    Text = Color3.fromRGB(205, 214, 244),
    SubText = Color3.fromRGB(147, 153, 178),
    Border = Color3.fromRGB(49, 50, 68),
    Dislike = Color3.fromRGB(150, 150, 150),
}

local TabData = {
    Free = { page = 1, hasMore = true, isLoading = false, totalLoaded = 0, totalApi = 0, query = nil, seen = {} },
    ThisGame = { page = 1, hasMore = true, isLoading = false, totalLoaded = 0, totalApi = 0, query = nil, seen = {} },
    Search = { page = 1, hasMore = true, isLoading = false, totalLoaded = 0, totalApi = 0, query = nil, seen = {} }
}
local CurrentTab = "Free"
local maxPerPage = 50 

local CurrentGameId = game.PlaceId
local CurrentGameName = "Unknown Game"
local currentScriptForView = ""
local currentUrlForView = ""

-- Lock state
local isUIDraggable = true

-- Safe String Splitter for infinite descriptions
local function SplitString(str, delim)
    local result = {}
    for match in (str..delim):gmatch("(.-)"..delim) do
        table.insert(result, match)
    end
    return result
end

local function FormatNumber(num)
    if not num or num == 0 then return "0" end
    if num >= 1000000 then return string.format("%.1fM", num / 1000000):gsub("%.0M", "M")
    elseif num >= 1000 then return string.format("%.1fK", num / 1000):gsub("%.0K", "K") end
    return tostring(num)
end

local function GetLikePercentage(likes, dislikes)
    local total = (likes or 0) + (dislikes or 0)
    if total == 0 then return 0 end 
    return math.floor(((likes or 0) / total) * 100)
end

local function FormatTimeAgo(dateString)
    if not dateString then return "?" end
    local ok, result = pcall(function()
        local y, mo, d, h, mi, s = dateString:match("(%d+)-(%d+)-(%d+)T(%d+):(%d+):(%d+)")
        if not y then return "?" end
        local t = os.time({year=tonumber(y), month=tonumber(mo), day=tonumber(d), hour=tonumber(h), min=tonumber(mi), sec=tonumber(s)})
        local diff = os.time() - t
        if diff < 60 then return "now"
        elseif diff < 3600 then return math.floor(diff/60).."m"
        elseif diff < 86400 then return math.floor(diff/3600).."h"
        elseif diff < 604800 then return math.floor(diff/86400).."d"
        else return math.floor(diff/604800).."w" end
    end)
    return ok and result or "?"
end

local function GetScriptUrl(data)
    if data.slug and data.slug ~= "" then return "https://scriptblox.com/script/" .. data.slug
    elseif data._id and data._id ~= "" then return "https://scriptblox.com/script/" .. data._id end
    return "https://scriptblox.com/"
end

local function GetCurrentGameInfo()
    local gameId = game.PlaceId
    local gameName = "Unknown Game"
    pcall(function()
        local info = MarketplaceService:GetProductInfo(gameId)
        if info and info.Name then gameName = info.Name end
    end)
    return gameId, gameName
end

local function HttpRequest(url)
    local ok, res = pcall(function()
        local r
        if syn and syn.request then r = syn.request({Url=url, Method="GET"})
        elseif http_request then r = http_request({Url=url, Method="GET"})
        elseif request then r = request({Url=url, Method="GET"})
        elseif http and http.request then r = http.request({Url=url, Method="GET"}) end
        if r then
            local body = r.Body or r.body
            if body then return HttpService:JSONDecode(body) end
        end
        return nil
    end)
    return ok and res or nil
end

local function GetImageAsset(url, id)
    if not url or url == "" then return nil end
    local gca = getcustomasset or getsynasset
    if not gca or not isfile or not writefile or not makefolder then return nil end
    
    local success, result = pcall(function()
        local folderName = ".ScriptBlox_Thumbnails"
        if not isfolder(folderName) then makefolder(folderName) end
        
        local safeId = tostring(id):gsub("[^%w%_]", "")
        if safeId == "" then return nil end
        
        local path = folderName .. "/" .. safeId .. ".txt"
        
        if isfile(path) then return gca(path) end
        
        local req = nil
        if syn and syn.request then req = syn.request
        elseif http_request then req = http_request
        elseif request then req = request
        elseif http and http.request then req = http.request end
        
        if req then
            local res = req({Url = url, Method = "GET"})
            if res and (res.StatusCode == 200 or res.Success) then
                writefile(path, res.Body or res.body)
                return gca(path)
            end
        end
        return nil
    end)
    return success and result or nil
end

local function CopyToClipboard(text)
    pcall(function()
        if setclipboard then setclipboard(text)
        elseif toclipboard then toclipboard(text)
        elseif Clipboard and Clipboard.set then Clipboard.set(text) end
    end)
end

-- GUI Setup
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "ScriptBloxUI"
ScreenGui.ResetOnSpawn = false
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
ScreenGui.DisplayOrder = 5
ScreenGui.Parent = PlayerGui

local MainFrame = Instance.new("Frame")
MainFrame.Name = "Main"
MainFrame.Size = UDim2.new(0, 520, 0, 420)
MainFrame.Position = UDim2.new(0.5, -260, 0.5, -210)
MainFrame.BackgroundColor3 = Colors.Background
MainFrame.BackgroundTransparency = UITransparency
MainFrame.BorderSizePixel = 0
MainFrame.ClipsDescendants = true
MainFrame.Parent = ScreenGui
MainFrame.Active = true 
Instance.new("UICorner", MainFrame).CornerRadius = UDim.new(0, 16)
Instance.new("UIStroke", MainFrame).Color = Colors.Border

-- Custom Toggle Button
local ToggleBtn = Instance.new("TextButton")
ToggleBtn.Name = "Toggle"
ToggleBtn.Size = UDim2.new(0, 60, 0, 60)
ToggleBtn.Position = UDim2.new(1, -80, 0.5, -30)
ToggleBtn.BackgroundColor3 = Colors.Primary
ToggleBtn.BackgroundTransparency = 0.75
ToggleBtn.Text = "" 
ToggleBtn.Visible = false
ToggleBtn.ZIndex = 100
ToggleBtn.Parent = ScreenGui
ToggleBtn.Active = true 
Instance.new("UICorner", ToggleBtn).CornerRadius = UDim.new(1, 0)
Instance.new("UIStroke", ToggleBtn).Color = Colors.Text

local ToggleIcon = Instance.new("ImageLabel")
ToggleIcon.Size = UDim2.new(0, 32, 0, 32)
ToggleIcon.Position = UDim2.new(0.5, -16, 0.5, -16)
ToggleIcon.BackgroundTransparency = 1
ToggleIcon.Image = "rbxassetid://93664761702799"
ToggleIcon.ScaleType = Enum.ScaleType.Fit
ToggleIcon.ZIndex = 101
ToggleIcon.Parent = ToggleBtn

-- FLAWLESS DRAG LOGIC (Now updated to support UI locking)
local function Dragify(frame, dragHandle)
    dragHandle = dragHandle or frame
    local dragging = false
    local dragInput, dragStart, startPos

    dragHandle.InputBegan:Connect(function(input)
        if frame == MainFrame and not isUIDraggable then return end -- Check Lock Status

        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragInput = input
            dragStart = input.Position
            startPos = frame.Position

            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                end
            end)
        end
    end)

    dragHandle.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            dragInput = input
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if input == dragInput and dragging then
            local delta = input.Position - dragStart
            frame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)
end

local Header = Instance.new("Frame")
Header.Size = UDim2.new(1, 0, 0, 45)
Header.BackgroundColor3 = Colors.Secondary
Header.BackgroundTransparency = UITransparency
Header.Parent = MainFrame
Header.Active = true
Instance.new("UICorner", Header).CornerRadius = UDim.new(0, 16)

local BottomDragBar = Instance.new("Frame", MainFrame)
BottomDragBar.Size = UDim2.new(1, 0, 0, 20)
BottomDragBar.Position = UDim2.new(0, 0, 1, -20)
BottomDragBar.BackgroundTransparency = 1
BottomDragBar.Active = true

local DragPill = Instance.new("Frame", BottomDragBar)
DragPill.Size = UDim2.new(0, 40, 0, 4)
DragPill.Position = UDim2.new(0.5, -20, 0.5, -2)
DragPill.BackgroundColor3 = Colors.SubText
DragPill.BackgroundTransparency = 0.5
Instance.new("UICorner", DragPill).CornerRadius = UDim.new(1, 0)

Dragify(MainFrame, Header)
Dragify(MainFrame, BottomDragBar)
Dragify(MainFrame, MainFrame)
Dragify(ToggleBtn, ToggleBtn)

local toggleClickStart
ToggleBtn.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then toggleClickStart = input.Position end
end)
ToggleBtn.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        if toggleClickStart and (input.Position - toggleClickStart).Magnitude < 10 then
            MainFrame.Visible = true; ToggleBtn.Visible = false
        end
    end
end)

-- CUSTOM IMAGE LOGO IN HEADER
local LogoIcon = Instance.new("ImageLabel")
LogoIcon.Size = UDim2.new(0, 22, 0, 22)
LogoIcon.Position = UDim2.new(0, 12, 0.5, -11)
LogoIcon.BackgroundTransparency = 1
LogoIcon.Image = "rbxassetid://93664761702799"
LogoIcon.ScaleType = Enum.ScaleType.Fit
LogoIcon.Parent = Header

local Logo = Instance.new("TextLabel")
Logo.Size = UDim2.new(0, 80, 0, 45)
Logo.Position = UDim2.new(0, 40, 0, 0)
Logo.BackgroundTransparency = 1
Logo.Text = "ScriptBlox"
Logo.TextColor3 = Colors.Text
Logo.TextSize = 14
Logo.Font = Enum.Font.GothamBold
Logo.TextXAlignment = Enum.TextXAlignment.Left
Logo.Parent = Header

local SearchFrame = Instance.new("Frame")
SearchFrame.Size = UDim2.new(0, 200, 0, 28)
SearchFrame.Position = UDim2.new(0, 125, 0.5, -14)
SearchFrame.BackgroundColor3 = Colors.Background
SearchFrame.BackgroundTransparency = UITransparency
SearchFrame.Parent = Header
Instance.new("UICorner", SearchFrame).CornerRadius = UDim.new(1, 0)

local SearchBox = Instance.new("TextBox")
SearchBox.Size = UDim2.new(1, -55, 1, 0)
SearchBox.Position = UDim2.new(0, 8, 0, 0)
SearchBox.BackgroundTransparency = 1
SearchBox.PlaceholderText = "Search scripts..."
SearchBox.TextColor3 = Colors.Text
SearchBox.TextSize = 11
SearchBox.Font = Enum.Font.Gotham
SearchBox.TextXAlignment = Enum.TextXAlignment.Left
SearchBox.ClearTextOnFocus = false
SearchBox.Text = ""
SearchBox.Parent = SearchFrame

local ClearBtn = Instance.new("TextButton")
ClearBtn.Size = UDim2.new(0, 24, 1, 0)
ClearBtn.Position = UDim2.new(1, -54, 0, 0)
ClearBtn.BackgroundColor3 = Colors.Card
ClearBtn.BackgroundTransparency = 0.5
ClearBtn.Text = "🗑️"
ClearBtn.TextColor3 = Colors.Text
ClearBtn.TextSize = 12
ClearBtn.Font = Enum.Font.GothamBold
ClearBtn.Parent = SearchFrame
Instance.new("UICorner", ClearBtn).CornerRadius = UDim.new(1, 0)

ClearBtn.MouseButton1Click:Connect(function() SearchBox.Text = "" end)

local SearchBtn = Instance.new("TextButton")
SearchBtn.Size = UDim2.new(0, 26, 1, 0)
SearchBtn.Position = UDim2.new(1, -26, 0, 0)
SearchBtn.BackgroundColor3 = Colors.Primary
SearchBtn.Text = "→"
SearchBtn.TextColor3 = Colors.Background
SearchBtn.TextSize = 14
SearchBtn.Font = Enum.Font.GothamBold
SearchBtn.Parent = SearchFrame
Instance.new("UICorner", SearchBtn).CornerRadius = UDim.new(1, 0)

-- QUIT CONFIRMATION MODAL
local ConfirmOverlay = Instance.new("Frame")
ConfirmOverlay.Size = UDim2.new(1, 0, 1, 0)
ConfirmOverlay.BackgroundColor3 = Color3.new(0,0,0)
ConfirmOverlay.BackgroundTransparency = 0.5
ConfirmOverlay.Visible = false
ConfirmOverlay.ZIndex = 80
ConfirmOverlay.Parent = MainFrame
ConfirmOverlay.Active = true

local ConfirmBox = Instance.new("Frame")
ConfirmBox.Size = UDim2.new(0, 280, 0, 130)
ConfirmBox.Position = UDim2.new(0.5, -140, 0.5, -65)
ConfirmBox.BackgroundColor3 = Colors.Card
ConfirmBox.ZIndex = 81
ConfirmBox.Parent = ConfirmOverlay
Instance.new("UICorner", ConfirmBox).CornerRadius = UDim.new(0, 12)
Instance.new("UIStroke", ConfirmBox).Color = Colors.Border

local ConfirmText = Instance.new("TextLabel")
ConfirmText.Size = UDim2.new(1, -20, 0, 60)
ConfirmText.Position = UDim2.new(0, 10, 0, 15)
ConfirmText.BackgroundTransparency = 1
ConfirmText.Text = "Are you sure about removing the UI?"
ConfirmText.TextColor3 = Colors.Text
ConfirmText.TextSize = 13
ConfirmText.Font = Enum.Font.GothamBold
ConfirmText.TextWrapped = true
ConfirmText.ZIndex = 82
ConfirmText.Parent = ConfirmBox

local ConfirmYes = Instance.new("TextButton")
ConfirmYes.Size = UDim2.new(0, 110, 0, 32)
ConfirmYes.Position = UDim2.new(0.5, -120, 1, -45)
ConfirmYes.BackgroundColor3 = Colors.Red
ConfirmYes.Text = "Yes"
ConfirmYes.TextColor3 = Colors.Text
ConfirmYes.Font = Enum.Font.GothamBold
ConfirmYes.ZIndex = 82
ConfirmYes.Parent = ConfirmBox
Instance.new("UICorner", ConfirmYes).CornerRadius = UDim.new(1, 0)

local ConfirmCancel = Instance.new("TextButton")
ConfirmCancel.Size = UDim2.new(0, 110, 0, 32)
ConfirmCancel.Position = UDim2.new(0.5, 10, 1, -45)
ConfirmCancel.BackgroundColor3 = Colors.Secondary
ConfirmCancel.Text = "Cancel"
ConfirmCancel.TextColor3 = Colors.Text
ConfirmCancel.Font = Enum.Font.GothamBold
ConfirmCancel.ZIndex = 82
ConfirmCancel.Parent = ConfirmBox
Instance.new("UICorner", ConfirmCancel).CornerRadius = UDim.new(1, 0)

ConfirmYes.MouseButton1Click:Connect(function() ScreenGui:Destroy() end)
ConfirmCancel.MouseButton1Click:Connect(function() ConfirmOverlay.Visible = false end)

-- EXECUTE CONFIRMATION MODAL
local ExecConfirmOverlay = Instance.new("Frame")
ExecConfirmOverlay.Size = UDim2.new(1, 0, 1, 0)
ExecConfirmOverlay.BackgroundColor3 = Color3.new(0,0,0)
ExecConfirmOverlay.BackgroundTransparency = 0.5
ExecConfirmOverlay.Visible = false
ExecConfirmOverlay.ZIndex = 90
ExecConfirmOverlay.Parent = MainFrame
ExecConfirmOverlay.Active = true

local ExecConfirmBox = Instance.new("Frame")
ExecConfirmBox.Size = UDim2.new(0, 280, 0, 140)
ExecConfirmBox.Position = UDim2.new(0.5, -140, 0.5, -70)
ExecConfirmBox.BackgroundColor3 = Colors.Card
ExecConfirmBox.ZIndex = 91
ExecConfirmBox.Parent = ExecConfirmOverlay
Instance.new("UICorner", ExecConfirmBox).CornerRadius = UDim.new(0, 12)
Instance.new("UIStroke", ExecConfirmBox).Color = Colors.Border

local ExecConfirmText = Instance.new("TextLabel")
ExecConfirmText.Size = UDim2.new(1, -20, 0, 65)
ExecConfirmText.Position = UDim2.new(0, 10, 0, 15)
ExecConfirmText.BackgroundTransparency = 1
ExecConfirmText.Text = "Do you want to execute this script?"
ExecConfirmText.TextColor3 = Colors.Text
ExecConfirmText.TextSize = 13
ExecConfirmText.Font = Enum.Font.ArialBold 
ExecConfirmText.TextWrapped = true
ExecConfirmText.ZIndex = 92
ExecConfirmText.Parent = ExecConfirmBox

local ExecConfirmYes = Instance.new("TextButton")
ExecConfirmYes.Size = UDim2.new(0, 110, 0, 32)
ExecConfirmYes.Position = UDim2.new(0.5, -120, 1, -45)
ExecConfirmYes.BackgroundColor3 = Colors.Green
ExecConfirmYes.Text = "Yes"
ExecConfirmYes.TextColor3 = Colors.Background
ExecConfirmYes.Font = Enum.Font.GothamBold
ExecConfirmYes.ZIndex = 92
ExecConfirmYes.Parent = ExecConfirmBox
Instance.new("UICorner", ExecConfirmYes).CornerRadius = UDim.new(1, 0)

local ExecConfirmCancel = Instance.new("TextButton")
ExecConfirmCancel.Size = UDim2.new(0, 110, 0, 32)
ExecConfirmCancel.Position = UDim2.new(0.5, 10, 1, -45)
ExecConfirmCancel.BackgroundColor3 = Colors.Secondary
ExecConfirmCancel.Text = "Cancel"
ExecConfirmCancel.TextColor3 = Colors.Text
ExecConfirmCancel.Font = Enum.Font.GothamBold
ExecConfirmCancel.ZIndex = 92
ExecConfirmCancel.Parent = ExecConfirmBox
Instance.new("UICorner", ExecConfirmCancel).CornerRadius = UDim.new(1, 0)

local CurrentExecCallback = nil

ExecConfirmYes.MouseButton1Click:Connect(function()
    ExecConfirmOverlay.Visible = false
    if CurrentExecCallback then
        CurrentExecCallback(true)
        CurrentExecCallback = nil
    end
end)

ExecConfirmCancel.MouseButton1Click:Connect(function()
    ExecConfirmOverlay.Visible = false
    if CurrentExecCallback then
        CurrentExecCallback(false)
        CurrentExecCallback = nil
    end
end)

local function PromptExecute(scriptTitle, callback)
    local limitedTitle = scriptTitle
    if #limitedTitle > 40 then limitedTitle = limitedTitle:sub(1,37).."..." end
    ExecConfirmText.Text = 'Do you want to execute:\n"'..limitedTitle..'"?'
    CurrentExecCallback = callback
    ExecConfirmOverlay.Visible = true
end

-- RIGHT HEADER BUTTONS: Close, Min, and Lock
local CloseBtn = Instance.new("TextButton")
CloseBtn.Size = UDim2.new(0, 24, 0, 24)
CloseBtn.Position = UDim2.new(1, -32, 0.5, -12)
CloseBtn.BackgroundColor3 = Colors.Red
CloseBtn.Text = "×"
CloseBtn.TextColor3 = Colors.Text
CloseBtn.TextSize = 16
CloseBtn.Font = Enum.Font.GothamBold
CloseBtn.Parent = Header
Instance.new("UICorner", CloseBtn).CornerRadius = UDim.new(1, 0)
CloseBtn.MouseButton1Click:Connect(function() ConfirmOverlay.Visible = true end)

local MinBtn = Instance.new("TextButton")
MinBtn.Size = UDim2.new(0, 24, 0, 24)
MinBtn.Position = UDim2.new(1, -60, 0.5, -12)
MinBtn.BackgroundColor3 = Colors.Yellow
MinBtn.Text = "-"
MinBtn.TextColor3 = Colors.Background
MinBtn.TextSize = 18
MinBtn.Font = Enum.Font.GothamBold
MinBtn.Parent = Header
Instance.new("UICorner", MinBtn).CornerRadius = UDim.new(1, 0)
MinBtn.MouseButton1Click:Connect(function() MainFrame.Visible = false; ToggleBtn.Visible = true end)

local LockBtn = Instance.new("TextButton")
LockBtn.Size = UDim2.new(0, 24, 0, 24)
LockBtn.Position = UDim2.new(1, -88, 0.5, -12)
LockBtn.BackgroundColor3 = Colors.Secondary
LockBtn.Text = "🔓"
LockBtn.TextColor3 = Colors.Text
LockBtn.TextSize = 14
LockBtn.Font = Enum.Font.Gotham
LockBtn.Parent = Header
Instance.new("UICorner", LockBtn).CornerRadius = UDim.new(1, 0)

LockBtn.MouseButton1Click:Connect(function() 
    isUIDraggable = not isUIDraggable
    LockBtn.Text = isUIDraggable and "🔓" or "🔒"
end)


local TabBar = Instance.new("Frame")
TabBar.Size = UDim2.new(1, 0, 0, 32)
TabBar.Position = UDim2.new(0, 0, 0, 45)
TabBar.BackgroundColor3 = Colors.Secondary
TabBar.BackgroundTransparency = UITransparency
TabBar.Parent = MainFrame

local TL = Instance.new("UIListLayout", TabBar)
TL.FillDirection = Enum.FillDirection.Horizontal
TL.Padding = UDim.new(0, 6)
TL.VerticalAlignment = Enum.VerticalAlignment.Center
Instance.new("UIPadding", TabBar).PaddingLeft = UDim.new(0, 8)

local function MakeTab(txt, col, par)
    local b = Instance.new("TextButton")
    b.Size = UDim2.new(0, #txt * 6 + 20, 0, 22)
    b.BackgroundColor3 = col; b.Text = txt; b.TextColor3 = Colors.Background
    b.TextSize = 10; b.Font = Enum.Font.GothamBold; b.Parent = par
    Instance.new("UICorner", b).CornerRadius = UDim.new(1, 0)
    return b
end

local FreeTab = MakeTab("🌐 All", Colors.Green, TabBar)
local ThisGameTab = MakeTab("🎮 This Game", Colors.Cyan, TabBar)
local RefreshTab = MakeTab("🔄 Refresh", Colors.Orange, TabBar)

local StatusLabel = Instance.new("TextLabel")
StatusLabel.Size = UDim2.new(0, 180, 0, 22)
StatusLabel.BackgroundTransparency = 1
StatusLabel.Text = ""
StatusLabel.TextColor3 = Colors.SubText
StatusLabel.TextSize = 9
StatusLabel.Font = Enum.Font.Gotham
StatusLabel.TextXAlignment = Enum.TextXAlignment.Left
StatusLabel.Parent = TabBar

local GameInfoBar = Instance.new("Frame")
GameInfoBar.Size = UDim2.new(1, 0, 0, 20)
GameInfoBar.Position = UDim2.new(0, 0, 0, 77)
GameInfoBar.BackgroundColor3 = Colors.Cyan
GameInfoBar.BackgroundTransparency = 0.85
GameInfoBar.Parent = MainFrame

local GameInfoLabel = Instance.new("TextLabel")
GameInfoLabel.Size = UDim2.new(1, -16, 1, 0)
GameInfoLabel.Position = UDim2.new(0, 8, 0, 0)
GameInfoLabel.BackgroundTransparency = 1
GameInfoLabel.Text = "🎮 Loading..."
GameInfoLabel.TextColor3 = Colors.Cyan
GameInfoLabel.TextSize = 9
GameInfoLabel.Font = Enum.Font.GothamMedium
GameInfoLabel.TextXAlignment = Enum.TextXAlignment.Left
GameInfoLabel.TextTruncate = Enum.TextTruncate.AtEnd
GameInfoLabel.Parent = GameInfoBar

local CopyIdBtn = Instance.new("TextButton")
CopyIdBtn.Size = UDim2.new(0, 55, 0, 16)
CopyIdBtn.Position = UDim2.new(1, -115, 0, 2)
CopyIdBtn.BackgroundColor3 = Colors.Primary
CopyIdBtn.BackgroundTransparency = 0.3
CopyIdBtn.Text = "📋 ID"
CopyIdBtn.TextColor3 = Colors.Text
CopyIdBtn.TextSize = 8
CopyIdBtn.Font = Enum.Font.GothamBold
CopyIdBtn.Visible = false
CopyIdBtn.Parent = GameInfoBar
Instance.new("UICorner", CopyIdBtn).CornerRadius = UDim.new(1, 0)

local CopyNameBtn = Instance.new("TextButton")
CopyNameBtn.Size = UDim2.new(0, 55, 0, 16)
CopyNameBtn.Position = UDim2.new(1, -57, 0, 2)
CopyNameBtn.BackgroundColor3 = Colors.Purple
CopyNameBtn.BackgroundTransparency = 0.3
CopyNameBtn.Text = "📋 Name"
CopyNameBtn.TextColor3 = Colors.Text
CopyNameBtn.TextSize = 8
CopyNameBtn.Font = Enum.Font.GothamBold
CopyNameBtn.Visible = false
CopyNameBtn.Parent = GameInfoBar
Instance.new("UICorner", CopyNameBtn).CornerRadius = UDim.new(1, 0)

task.spawn(function()
    CurrentGameId, CurrentGameName = GetCurrentGameInfo()
end)

CopyIdBtn.MouseButton1Click:Connect(function() CopyToClipboard(tostring(CurrentGameId)); CopyIdBtn.Text = "✓ Done"; task.delay(1, function() CopyIdBtn.Text = "📋 ID" end) end)
CopyNameBtn.MouseButton1Click:Connect(function() CopyToClipboard(CurrentGameName); CopyNameBtn.Text = "✓ Done"; task.delay(1, function() CopyNameBtn.Text = "📋 Name" end) end)

local function SetCopyVisible(v)
    CopyIdBtn.Visible = v; CopyNameBtn.Visible = v
    GameInfoLabel.Size = v and UDim2.new(1, -120, 1, 0) or UDim2.new(1, -16, 1, 0)
end

local Containers = {}
for _, tab in ipairs({"Free", "ThisGame", "Search"}) do
    local sf = Instance.new("ScrollingFrame")
    sf.Name = "Container_" .. tab
    sf.Size = UDim2.new(1, -16, 1, -105)
    sf.Position = UDim2.new(0, 8, 0, 100)
    sf.BackgroundTransparency = 1
    sf.ScrollBarThickness = 4
    sf.ScrollBarImageColor3 = Colors.Primary
    sf.CanvasSize = UDim2.new(0, 0, 0, 0)
    sf.Parent = MainFrame
    sf.Visible = false

    local grid = Instance.new("UIGridLayout", sf)
    grid.CellSize = UDim2.new(0, 160, 0, 166)
    grid.CellPadding = UDim2.new(0, 6, 0, 6)
    grid.SortOrder = Enum.SortOrder.LayoutOrder
    Containers[tab] = sf
end

local LoadingFrame = Instance.new("Frame")
LoadingFrame.Size = UDim2.new(1, 0, 1, 0)
LoadingFrame.BackgroundColor3 = Colors.Background
LoadingFrame.BackgroundTransparency = 0.5
LoadingFrame.Visible = false
LoadingFrame.ZIndex = 50
LoadingFrame.Parent = MainFrame

local LoadingText = Instance.new("TextLabel")
LoadingText.Size = UDim2.new(1, 0, 1, 0)
LoadingText.BackgroundTransparency = 1
LoadingText.Text = "🔄 Loading..."
LoadingText.TextColor3 = Colors.Primary
LoadingText.TextSize = 16
LoadingText.Font = Enum.Font.GothamBold
LoadingText.ZIndex = 51
LoadingText.Parent = LoadingFrame

--=============================================
-- MODAL SCRIPT VIEWER: DESCRIPTION, CODE, COMMENTS
--=============================================
local ModalOverlay = Instance.new("Frame")
ModalOverlay.Size = UDim2.new(1, 0, 1, 0)
ModalOverlay.BackgroundColor3 = Color3.new(0, 0, 0)
ModalOverlay.BackgroundTransparency = 0.5
ModalOverlay.Visible = false
ModalOverlay.ZIndex = 60
ModalOverlay.Parent = MainFrame
ModalOverlay.Active = true -- Stops clicks from passing through

local Modal = Instance.new("Frame")
Modal.Size = UDim2.new(0, 480, 0, 380)
Modal.Position = UDim2.new(0.5, -240, 0.5, -190)
Modal.BackgroundColor3 = Colors.Card
Modal.BackgroundTransparency = UITransparency
Modal.ZIndex = 61
Modal.Parent = ModalOverlay
Instance.new("UICorner", Modal).CornerRadius = UDim.new(0, 16)
Instance.new("UIStroke", Modal).Color = Colors.Border

local MH = Instance.new("Frame")
MH.Size = UDim2.new(1, 0, 0, 35)
MH.BackgroundColor3 = Colors.Secondary
MH.BackgroundTransparency = UITransparency
MH.ZIndex = 62; MH.Parent = Modal
Instance.new("UICorner", MH).CornerRadius = UDim.new(0, 16)

local MT = Instance.new("TextLabel")
MT.Size = UDim2.new(1, -50, 1, 0)
MT.Position = UDim2.new(0, 10, 0, 0)
MT.BackgroundTransparency = 1
MT.Text = "📜 Script Viewer"
MT.TextColor3 = Colors.Text
MT.TextSize = 12
MT.Font = Enum.Font.ArialBold
MT.TextXAlignment = Enum.TextXAlignment.Left
MT.TextTruncate = Enum.TextTruncate.AtEnd
MT.ZIndex = 62; MT.Parent = MH

local MC = Instance.new("TextButton")
MC.Size = UDim2.new(0, 25, 0, 25)
MC.Position = UDim2.new(1, -30, 0.5, -12)
MC.BackgroundColor3 = Colors.Red
MC.Text = "×"; MC.TextColor3 = Colors.Text
MC.TextSize = 14; MC.Font = Enum.Font.GothamBold
MC.ZIndex = 63; MC.Parent = MH
Instance.new("UICorner", MC).CornerRadius = UDim.new(1, 0)
MC.MouseButton1Click:Connect(function() ModalOverlay.Visible = false end)

-- 3-WAY TAB BAR
local ModalTabBar = Instance.new("Frame")
ModalTabBar.Size = UDim2.new(1, 0, 0, 30)
ModalTabBar.Position = UDim2.new(0, 0, 0, 35)
ModalTabBar.BackgroundColor3 = Colors.Background
ModalTabBar.BackgroundTransparency = UITransparency
ModalTabBar.ZIndex = 62; ModalTabBar.Parent = Modal

local DescTabBtn = Instance.new("TextButton")
DescTabBtn.Size = UDim2.new(0.33, 0, 1, 0)
DescTabBtn.Position = UDim2.new(0, 0, 0, 0)
DescTabBtn.BackgroundTransparency = 1
DescTabBtn.Text = "📝 Description"
DescTabBtn.TextColor3 = Colors.Primary
DescTabBtn.Font = Enum.Font.GothamBold
DescTabBtn.TextSize = 11
DescTabBtn.ZIndex = 63; DescTabBtn.Parent = ModalTabBar

local CodeTabBtn = Instance.new("TextButton")
CodeTabBtn.Size = UDim2.new(0.34, 0, 1, 0)
CodeTabBtn.Position = UDim2.new(0.33, 0, 0, 0)
CodeTabBtn.BackgroundTransparency = 1
CodeTabBtn.Text = "📜 View Raw"
CodeTabBtn.TextColor3 = Colors.SubText
CodeTabBtn.Font = Enum.Font.GothamBold
CodeTabBtn.TextSize = 11
CodeTabBtn.ZIndex = 63; CodeTabBtn.Parent = ModalTabBar

local CommentsTabBtn = Instance.new("TextButton")
CommentsTabBtn.Size = UDim2.new(0.33, 0, 1, 0)
CommentsTabBtn.Position = UDim2.new(0.67, 0, 0, 0)
CommentsTabBtn.BackgroundTransparency = 1
CommentsTabBtn.Text = "💬 Comments (...)"
CommentsTabBtn.TextColor3 = Colors.SubText
CommentsTabBtn.Font = Enum.Font.GothamBold
CommentsTabBtn.TextSize = 11
CommentsTabBtn.ZIndex = 63; CommentsTabBtn.Parent = ModalTabBar

local MI = Instance.new("Frame")
MI.Size = UDim2.new(1, -16, 0, 20)
MI.Position = UDim2.new(0, 8, 0, 68)
MI.BackgroundTransparency = 1; MI.ZIndex = 62; MI.Parent = Modal

local GB = Instance.new("TextLabel")
GB.Size = UDim2.new(0, 120, 0, 16)
GB.BackgroundColor3 = Colors.Primary; GB.BackgroundTransparency = 0.7
GB.Text = "🎮 Game"; GB.TextColor3 = Colors.Primary
GB.TextSize = 9
GB.Font = Enum.Font.ArialBold
GB.TextTruncate = Enum.TextTruncate.AtEnd; GB.ZIndex = 62; GB.Parent = MI
Instance.new("UICorner", GB).CornerRadius = UDim.new(1, 0)

local SB = Instance.new("TextLabel")
SB.Size = UDim2.new(0, 55, 0, 16)
SB.Position = UDim2.new(0, 125, 0, 0)
SB.BackgroundColor3 = Colors.Green
SB.Text = "✓ Working"; SB.TextColor3 = Colors.Background
SB.TextSize = 7; SB.Font = Enum.Font.GothamBold; SB.ZIndex = 62; SB.Parent = MI
Instance.new("UICorner", SB).CornerRadius = UDim.new(1, 0)

local ML = Instance.new("TextLabel")
ML.Size = UDim2.new(0, 120, 0, 16)
ML.Position = UDim2.new(0, 185, 0, 0)
ML.BackgroundTransparency = 1
ML.Text = "❤️ 0 | 👎 0"; ML.TextColor3 = Colors.SubText
ML.TextSize = 9; ML.Font = Enum.Font.GothamMedium
ML.TextXAlignment = Enum.TextXAlignment.Left; ML.ZIndex = 62; ML.Parent = MI

-- SCROLLING FRAME: DESCRIPTION 
local DS = Instance.new("ScrollingFrame")
DS.Size = UDim2.new(1, -16, 1, -135)
DS.Position = UDim2.new(0, 8, 0, 95)
DS.BackgroundColor3 = Colors.Background
DS.BackgroundTransparency = UITransparency
DS.ScrollBarThickness = 4; DS.ScrollBarImageColor3 = Colors.Primary
DS.Visible = true 
DS.ZIndex = 62; DS.Parent = Modal
Instance.new("UICorner", DS).CornerRadius = UDim.new(0, 12)

local dsLayout = Instance.new("UIListLayout", DS)
dsLayout.SortOrder = Enum.SortOrder.LayoutOrder
dsLayout.Padding = UDim.new(0, 4) 
local dsPad = Instance.new("UIPadding", DS)
dsPad.PaddingTop = UDim.new(0, 8)
dsPad.PaddingLeft = UDim.new(0, 8)
dsPad.PaddingRight = UDim.new(0, 12)

dsLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
    DS.CanvasSize = UDim2.new(0, 0, 0, dsLayout.AbsoluteContentSize.Y + 20)
end)

local DescAuthor = Instance.new("TextLabel", DS)
DescAuthor.Name = "DescAuthor"
DescAuthor.Size = UDim2.new(1, 0, 0, 20)
DescAuthor.BackgroundTransparency = 1
DescAuthor.Text = "By: Loading..."
DescAuthor.TextColor3 = Colors.Primary
DescAuthor.TextSize = 12
DescAuthor.Font = Enum.Font.ArialBold
DescAuthor.TextXAlignment = Enum.TextXAlignment.Left

-- SCROLLING FRAME: CODE
local SS = Instance.new("ScrollingFrame")
SS.Size = UDim2.new(1, -16, 1, -135)
SS.Position = UDim2.new(0, 8, 0, 95)
SS.BackgroundColor3 = Colors.Background
SS.BackgroundTransparency = UITransparency
SS.ScrollBarThickness = 4; SS.ScrollBarImageColor3 = Colors.Primary
SS.Visible = false
SS.ZIndex = 62; SS.Parent = Modal
Instance.new("UICorner", SS).CornerRadius = UDim.new(0, 12)

local ssLayout = Instance.new("UIListLayout", SS)
ssLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
    SS.CanvasSize = UDim2.new(0, 0, 0, ssLayout.AbsoluteContentSize.Y + 20)
end)

local SC = Instance.new("TextLabel")
SC.Size = UDim2.new(1, -10, 0, 0)
SC.Position = UDim2.new(0, 5, 0, 5)
SC.BackgroundTransparency = 1; SC.Text = ""
SC.TextColor3 = Colors.Green; SC.TextSize = 10; SC.Font = Enum.Font.Code
SC.TextXAlignment = Enum.TextXAlignment.Left; SC.TextYAlignment = Enum.TextYAlignment.Top
SC.TextWrapped = true
SC.AutomaticSize = Enum.AutomaticSize.Y
SC.ZIndex = 62; SC.Parent = SS

-- SCROLLING FRAME: COMMENTS
local CS = Instance.new("ScrollingFrame")
CS.Size = UDim2.new(1, -16, 1, -100)
CS.Position = UDim2.new(0, 8, 0, 95)
CS.BackgroundColor3 = Colors.Background
CS.BackgroundTransparency = UITransparency
CS.ScrollBarThickness = 4; CS.ScrollBarImageColor3 = Colors.Primary
CS.CanvasSize = UDim2.new(0, 0, 0, 0)
CS.AutomaticCanvasSize = Enum.AutomaticSize.Y
CS.Visible = false
CS.ZIndex = 62; CS.Parent = Modal
Instance.new("UICorner", CS).CornerRadius = UDim.new(0, 12)

local MBF = Instance.new("Frame")
MBF.Size = UDim2.new(1, -16, 0, 35)
MBF.Position = UDim2.new(0, 8, 1, -45)
MBF.BackgroundTransparency = 1; MBF.ZIndex = 62; MBF.Parent = Modal

local CopyScriptBtn = Instance.new("TextButton")
CopyScriptBtn.Size = UDim2.new(0.48, 0, 1, 0)
CopyScriptBtn.Position = UDim2.new(0, 0, 0, 0)
CopyScriptBtn.BackgroundColor3 = Colors.Secondary
CopyScriptBtn.Text = "📋 Copy Script"
CopyScriptBtn.TextColor3 = Colors.Text
CopyScriptBtn.TextSize = 11
CopyScriptBtn.Font = Enum.Font.GothamBold
CopyScriptBtn.ZIndex = 63
CopyScriptBtn.Parent = MBF
Instance.new("UICorner", CopyScriptBtn).CornerRadius = UDim.new(1, 0)

local CopyUrlBtn = Instance.new("TextButton")
CopyUrlBtn.Size = UDim2.new(0.48, 0, 1, 0)
CopyUrlBtn.Position = UDim2.new(0.52, 0, 0, 0)
CopyUrlBtn.BackgroundColor3 = Colors.Pink
CopyUrlBtn.Text = "🔗 Copy URL"
CopyUrlBtn.TextColor3 = Colors.Background
CopyUrlBtn.TextSize = 11
CopyUrlBtn.Font = Enum.Font.GothamBold
CopyUrlBtn.ZIndex = 63
CopyUrlBtn.Parent = MBF
Instance.new("UICorner", CopyUrlBtn).CornerRadius = UDim.new(1, 0)

CopyScriptBtn.MouseButton1Click:Connect(function() CopyToClipboard(currentScriptForView); CopyScriptBtn.Text = "✓ Copied!"; task.delay(1.5, function() CopyScriptBtn.Text = "📋 Copy Script" end) end)
CopyUrlBtn.MouseButton1Click:Connect(function() if currentUrlForView and currentUrlForView ~= "" then CopyToClipboard(currentUrlForView); CopyUrlBtn.Text = "✓ URL Copied!"; task.delay(1.5, function() CopyUrlBtn.Text = "🔗 Copy URL" end) end end)

-- TAB SWITCHING LOGIC
local function SwitchModalTab(tabName)
    DescTabBtn.TextColor3 = (tabName == "Desc") and Colors.Primary or Colors.SubText
    CodeTabBtn.TextColor3 = (tabName == "Code") and Colors.Primary or Colors.SubText
    CommentsTabBtn.TextColor3 = (tabName == "Comments") and Colors.Primary or Colors.SubText

    DS.Visible = (tabName == "Desc")
    SS.Visible = (tabName == "Code")
    CS.Visible = (tabName == "Comments")
    
    MBF.Visible = (tabName ~= "Comments")
end

DescTabBtn.MouseButton1Click:Connect(function() SwitchModalTab("Desc") end)
CodeTabBtn.MouseButton1Click:Connect(function() SwitchModalTab("Code") end)
CommentsTabBtn.MouseButton1Click:Connect(function() SwitchModalTab("Comments") end)

-- FETCH FULL SCRIPT DETAILS 
local function FetchScriptDetails(slug)
    DescAuthor.Text = "By: Loading..."
    
    for _, child in pairs(DS:GetChildren()) do
        if child:IsA("TextLabel") and child.Name == "DescLine" then
            child:Destroy()
        end
    end

    local loadingLbl = Instance.new("TextLabel", DS)
    loadingLbl.Name = "DescLine"
    loadingLbl.Size = UDim2.new(1, 0, 0, 20)
    loadingLbl.BackgroundTransparency = 1
    loadingLbl.Text = "Loading full description..."
    loadingLbl.TextColor3 = Colors.Text
    loadingLbl.TextSize = 11
    loadingLbl.Font = Enum.Font.Arial
    loadingLbl.TextXAlignment = Enum.TextXAlignment.Left

    task.spawn(function()
        local url = "https://scriptblox.com/api/script/" .. tostring(slug)
        local res = HttpRequest(url)
        
        if loadingLbl then loadingLbl:Destroy() end

        if res and res.script then
            local scriptData = res.script
            local authorName = "Unknown"
            
            if scriptData.owner and scriptData.owner.username then
                authorName = scriptData.owner.username
            elseif scriptData.user and scriptData.user.username then
                authorName = scriptData.user.username
            end
            
            DescAuthor.Text = "By: " .. authorName

            local rawDesc = scriptData.features or scriptData.description or "No description provided for this script."
            rawDesc = rawDesc:gsub("\r", "")
            
            local lines = SplitString(rawDesc, "\n")
            for _, lineText in ipairs(lines) do
                local lineLbl = Instance.new("TextLabel", DS)
                lineLbl.Name = "DescLine"
                lineLbl.Size = UDim2.new(1, 0, 0, lineText == "" and 10 or 0)
                lineLbl.AutomaticSize = lineText == "" and Enum.AutomaticSize.None or Enum.AutomaticSize.Y
                lineLbl.BackgroundTransparency = 1
                lineLbl.Text = lineText
                lineLbl.TextColor3 = Colors.Text
                lineLbl.TextSize = 11 
                lineLbl.Font = Enum.Font.Arial
                lineLbl.TextXAlignment = Enum.TextXAlignment.Left
                lineLbl.TextYAlignment = Enum.TextYAlignment.Top
                lineLbl.TextWrapped = true
            end
        else
            DescAuthor.Text = "By: Unknown"
            local errLbl = Instance.new("TextLabel", DS)
            errLbl.Name = "DescLine"
            errLbl.Size = UDim2.new(1, 0, 0, 0)
            errLbl.AutomaticSize = Enum.AutomaticSize.Y
            errLbl.BackgroundTransparency = 1
            errLbl.Text = "Failed to load description."
            errLbl.TextColor3 = Colors.Red
            errLbl.TextSize = 11
            errLbl.Font = Enum.Font.Arial
            errLbl.TextXAlignment = Enum.TextXAlignment.Left
            errLbl.TextWrapped = true
        end
    end)
end

-- COMMENTS RENDERING ENGINE
local function CreateCommentUI(comment, parent, isReply)
    local userObj = comment.commentBy or comment.replyBy or comment.user or comment.author or comment.by or {}
    local uName = userObj.username or userObj.name or "Unknown"
    local pPic = userObj.profilePicture or userObj.avatar or ""
    local txt = comment.text or comment.body or ""
    local tAgo = FormatTimeAgo(comment.createdAt)

    local function countVotes(val)
        if type(val) == "number" then return val end
        if type(val) == "table" then return #val end
        return 0
    end
    local cLikes = comment.likeCount or countVotes(comment.likes) or countVotes(comment.upvotes) or 0
    local cDislikes = comment.dislikeCount or countVotes(comment.dislikes) or countVotes(comment.downvotes) or 0

    txt = txt:gsub("<", "&lt;"):gsub(">", "&gt;")
    txt = txt:gsub("(@[%w_]+)", '<font color="#89b4fa">%1</font>')

    local cFrame = Instance.new("Frame", parent)
    cFrame.Size = UDim2.new(1, 0, 0, 0)
    cFrame.BackgroundTransparency = 1
    cFrame.AutomaticSize = Enum.AutomaticSize.Y
    
    local padding = Instance.new("UIPadding", cFrame)
    padding.PaddingLeft = UDim.new(0, isReply and 45 or 5)

    local avatar = Instance.new("ImageLabel", cFrame)
    avatar.Size = UDim2.new(0, 30, 0, 30)
    avatar.Position = UDim2.new(0, 0, 0, 0)
    avatar.BackgroundTransparency = 1
    avatar.Image = "rbxassetid://10023616654" 
    avatar.ScaleType = Enum.ScaleType.Crop
    Instance.new("UICorner", avatar).CornerRadius = UDim.new(1, 0)
    
    task.spawn(function()
        if pPic ~= "" then
            local fullUrl = pPic:match("^http") and pPic or "https://scriptblox.com" .. (pPic:sub(1,1) == "/" and "" or "/") .. pPic
            local img = GetImageAsset(fullUrl, uName)
            if img then pcall(function() avatar.Image = img end) end
        end
    end)

    local contentBox = Instance.new("Frame", cFrame)
    contentBox.Size = UDim2.new(1, -40, 0, 0)
    contentBox.Position = UDim2.new(0, 40, 0, 0)
    contentBox.BackgroundTransparency = 1
    contentBox.AutomaticSize = Enum.AutomaticSize.Y
    
    local cLayout = Instance.new("UIListLayout", contentBox)
    cLayout.SortOrder = Enum.SortOrder.LayoutOrder
    cLayout.Padding = UDim.new(0, 4)

    local topRow = Instance.new("Frame", contentBox)
    topRow.Size = UDim2.new(1, 0, 0, 16)
    topRow.BackgroundTransparency = 1
    
    local hLayout = Instance.new("UIListLayout", topRow)
    hLayout.FillDirection = Enum.FillDirection.Horizontal
    hLayout.VerticalAlignment = Enum.VerticalAlignment.Center
    hLayout.Padding = UDim.new(0, 6)

    local userLbl = Instance.new("TextLabel", topRow)
    userLbl.AutomaticSize = Enum.AutomaticSize.XY
    userLbl.BackgroundTransparency = 1
    userLbl.Text = uName
    userLbl.TextColor3 = Colors.Text
    userLbl.Font = Enum.Font.ArialBold 
    userLbl.TextSize = 11

    local timeLbl = Instance.new("TextLabel", topRow)
    timeLbl.AutomaticSize = Enum.AutomaticSize.XY
    timeLbl.BackgroundTransparency = 1
    timeLbl.Text = tAgo
    timeLbl.TextColor3 = Colors.SubText
    timeLbl.Font = Enum.Font.Arial
    timeLbl.TextSize = 10

    local bodyLbl = Instance.new("TextLabel", contentBox)
    bodyLbl.Size = UDim2.new(1, 0, 0, 0)
    bodyLbl.AutomaticSize = Enum.AutomaticSize.Y
    bodyLbl.BackgroundTransparency = 1
    bodyLbl.TextWrapped = true
    bodyLbl.RichText = true
    bodyLbl.TextColor3 = Colors.Text
    bodyLbl.Font = Enum.Font.Arial
    bodyLbl.TextSize = 11
    bodyLbl.TextXAlignment = Enum.TextXAlignment.Left
    bodyLbl.Text = txt

    local actionRow = Instance.new("Frame", contentBox)
    actionRow.Size = UDim2.new(1, 0, 0, 16)
    actionRow.BackgroundTransparency = 1
    
    local actLayout = Instance.new("UIListLayout", actionRow)
    actLayout.FillDirection = Enum.FillDirection.Horizontal
    actLayout.Padding = UDim.new(0, 15)

    local function createAct(txtIcon)
        local l = Instance.new("TextLabel", actionRow)
        l.AutomaticSize = Enum.AutomaticSize.XY
        l.BackgroundTransparency = 1
        l.Text = txtIcon
        l.TextColor3 = Colors.SubText
        l.Font = Enum.Font.Arial 
        l.TextSize = 10
    end
    
    createAct("👍 " .. FormatNumber(cLikes))
    createAct("👎 " .. FormatNumber(cDislikes))
    createAct("Reply")
end

local function FetchAndRenderComments(scriptId)
    for _, c in pairs(CS:GetChildren()) do
        if c:IsA("Frame") or c:IsA("TextLabel") then c:Destroy() end
    end

    local layout = CS:FindFirstChildOfClass("UIListLayout")
    if not layout then
        layout = Instance.new("UIListLayout", CS)
        layout.SortOrder = Enum.SortOrder.LayoutOrder
        layout.Padding = UDim.new(0, 15)
    end

    local loadingLbl = Instance.new("TextLabel", CS)
    loadingLbl.Size = UDim2.new(1, 0, 0, 30)
    loadingLbl.BackgroundTransparency = 1
    loadingLbl.Text = "🔄 Fetching All Comments..."
    loadingLbl.TextColor3 = Colors.Primary
    loadingLbl.Font = Enum.Font.ArialBold
    loadingLbl.TextSize = 11

    task.spawn(function()
        local allComments = {}
        local fetchPage = 1
        local maxPages = 1
        local totalCommentsCount = 0

        -- Fetches every page of comments sequentially 
        repeat
            local url = "https://scriptblox.com/api/comment/" .. tostring(scriptId) .. "?page="..tostring(fetchPage).."&max=100"
            local res = HttpRequest(url)
            
            local fetchedComments = {}
            if res then
                local dataSource = res.result or res.data or res
                if type(dataSource.comments) == "table" then
                    fetchedComments = dataSource.comments
                end
                maxPages = dataSource.totalPages or 1
                totalCommentsCount = dataSource.total or totalCommentsCount
            end

            for _, c in ipairs(fetchedComments) do
                table.insert(allComments, c)
            end

            if #fetchedComments == 0 then break end
            
            fetchPage = fetchPage + 1
            if fetchPage > 50 then break end -- hard limit to prevent infinite stall
        until fetchPage > maxPages
        
        if loadingLbl then loadingLbl:Destroy() end

        -- Set the text based on the full amount of comments found!
        CommentsTabBtn.Text = string.format("💬 Comments (%d)", totalCommentsCount > 0 and totalCommentsCount or #allComments)

        if #allComments == 0 then
            local noneLbl = Instance.new("TextLabel", CS)
            noneLbl.Size = UDim2.new(1, 0, 0, 30)
            noneLbl.BackgroundTransparency = 1
            noneLbl.Text = "No comments yet."
            noneLbl.TextColor3 = Colors.SubText
            noneLbl.Font = Enum.Font.Arial
            noneLbl.TextSize = 11
        else
            for _, c in ipairs(allComments) do
                CreateCommentUI(c, CS, false)
                if c.replies and type(c.replies) == "table" and #c.replies > 0 then
                    for _, r in ipairs(c.replies) do
                        CreateCommentUI(r, CS, true)
                    end
                end
            end
        end
    end)
end

--=============================================
-- MAIN APP LOGIC
--=============================================
local function UpdateUIState()
    local state = TabData[CurrentTab]
    for k, v in pairs(Containers) do v.Visible = (k == CurrentTab) end
    
    if CurrentTab == "ThisGame" then
        SetCopyVisible(true)
        GameInfoLabel.Text = "🎮 " .. CurrentGameName .. " (ID: " .. CurrentGameId .. ")"
    elseif CurrentTab == "Search" then
        SetCopyVisible(false)
        GameInfoLabel.Text = "🔍 Search Results"
    else
        SetCopyVisible(false)
        GameInfoLabel.Text = "🌐 All Scripts" 
    end

    if state.totalLoaded > 0 then
        StatusLabel.Text = string.format("📊 %d scripts", state.totalLoaded)
    elseif not state.isLoading then
        StatusLabel.Text = "❌ No scripts found"
    else
        StatusLabel.Text = "🔄 Loading..."
    end
end

local function CreateCard(data, index, tabName)
    local state = TabData[tabName]
    local id = data._id or data.slug or tostring(index)..tostring(tick())
    if state.seen[id] then return false end
    state.seen[id] = true
    state.totalLoaded = state.totalLoaded + 1

    local hasKey = data.key or data.keySystem or data.hasKey or false
    local likes = data.likes or data.likeCount or 0
    local dislikes = data.dislikes or data.dislikeCount or 0
    local views = data.views or 0

    local gid, gname = nil, "Universal"
    if data.game then
        gid = data.game.gameId or data.game.placeId or data.game.id
        gname = data.game.name or "Unknown"
    end

    local ut = FormatTimeAgo(data.createdAt or data.updatedAt)
    local isCur = gid and tonumber(gid) == tonumber(CurrentGameId)
    
    local imgUrl = nil
    if data.image and data.image ~= "" then
        imgUrl = data.image:match("^http") and data.image or ("https://scriptblox.com" .. (data.image:sub(1,1) == "/" and "" or "/") .. data.image)
    elseif data.game and data.game.imageUrl and data.game.imageUrl ~= "" then
        imgUrl = data.game.imageUrl:match("^http") and data.game.imageUrl or ("https://scriptblox.com" .. (data.game.imageUrl:sub(1,1) == "/" and "" or "/") .. data.game.imageUrl)
    end

    local card = Instance.new("Frame")
    card.Name = "C"..state.totalLoaded
    card.BackgroundColor3 = Colors.Card
    card.BackgroundTransparency = UITransparency
    card.LayoutOrder = state.totalLoaded
    card.Parent = Containers[tabName]
    Instance.new("UICorner", card).CornerRadius = UDim.new(0, 12)
    local cs = Instance.new("UIStroke", card)
    cs.Color = isCur and Colors.Cyan or Colors.Border
    cs.Transparency = UITransparency; cs.Thickness = isCur and 2 or 1

    local ia = Instance.new("Frame")
    ia.Size = UDim2.new(1, 0, 0, 75)
    ia.BackgroundColor3 = Colors.Secondary
    ia.ClipsDescendants = true; ia.Parent = card
    Instance.new("UICorner", ia).CornerRadius = UDim.new(0, 12)

    local gi = Instance.new("ImageLabel")
    gi.Size = UDim2.new(1, 0, 1, 0); gi.BackgroundTransparency = 1
    gi.ScaleType = Enum.ScaleType.Crop; gi.Parent = ia

    local fb = Instance.new("TextLabel")
    fb.Size = UDim2.new(1, 0, 1, 0); fb.BackgroundTransparency = 1
    fb.Text = "🎮"; fb.TextSize = 22; fb.Parent = ia
    
    task.spawn(function()
        local loadedImg = false
        if imgUrl then
            local asset = GetImageAsset(imgUrl, id)
            if asset then pcall(function() gi.Image = asset; fb.Visible = false; loadedImg = true end) end
        end
        if not loadedImg and gid then
            pcall(function() gi.Image = "rbxthumb://type=GameIcon&id="..tostring(gid).."&w=150&h=150"; task.wait(0.2); if gi.Image ~= "" then fb.Visible = false end end)
        end
    end)

    local st = Instance.new("TextLabel")
    st.Size = UDim2.new(0, 42, 0, 12); st.Position = UDim2.new(0, 3, 0, 3)
    st.BackgroundColor3 = data.isPatched and Colors.Red or Colors.Green; st.BackgroundTransparency = 0.15
    st.Text = data.isPatched and "Patched" or "Working"; st.TextColor3 = Colors.Text
    st.TextSize = 7; st.Font = Enum.Font.GothamBold; st.ZIndex = 2; st.Parent = ia
    Instance.new("UICorner", st).CornerRadius = UDim.new(1, 0)

    local kb = Instance.new("TextLabel")
    kb.Size = UDim2.new(0, 35, 0, 12); kb.Position = UDim2.new(1, -38, 0, 3)
    kb.BackgroundColor3 = hasKey and Colors.Orange or Colors.Purple; kb.BackgroundTransparency = 0.15
    kb.Text = hasKey and "🔑Key" or "🔓Free"; kb.TextColor3 = Colors.Text
    kb.TextSize = 6; kb.Font = Enum.Font.GothamBold; kb.ZIndex = 2; kb.Parent = ia
    Instance.new("UICorner", kb).CornerRadius = UDim.new(1, 0)

    if isCur and tabName == "ThisGame" then
        local tb = Instance.new("TextLabel")
        tb.Size = UDim2.new(0, 50, 0, 12); tb.Position = UDim2.new(0, 3, 1, -15)
        tb.BackgroundColor3 = Colors.Cyan; tb.BackgroundTransparency = 0.15
        tb.Text = "✓ Match"; tb.TextColor3 = Colors.Background
        tb.TextSize = 7; tb.Font = Enum.Font.GothamBold; tb.ZIndex = 2; tb.Parent = ia
        Instance.new("UICorner", tb).CornerRadius = UDim.new(1, 0)
    end

    local tl = Instance.new("TextLabel")
    tl.Size = UDim2.new(1, -6, 0, 14); tl.Position = UDim2.new(0, 3, 0, 78)
    tl.BackgroundTransparency = 1; tl.Text = data.title or "Unknown"
    tl.TextColor3 = Colors.Text; tl.TextSize = 10
    tl.Font = Enum.Font.ArialBold
    tl.TextXAlignment = Enum.TextXAlignment.Left; tl.TextTruncate = Enum.TextTruncate.AtEnd; tl.Parent = card

    local gl = Instance.new("TextLabel")
    gl.Size = UDim2.new(1, -6, 0, 11); gl.Position = UDim2.new(0, 3, 0, 92)
    gl.BackgroundTransparency = 1; gl.Text = "🎮 "..gname
    gl.TextColor3 = isCur and Colors.Cyan or Colors.SubText
    gl.TextSize = 8
    gl.Font = Enum.Font.Arial
    gl.TextXAlignment = Enum.TextXAlignment.Left; gl.TextTruncate = Enum.TextTruncate.AtEnd; gl.Parent = card

    local sr = Instance.new("Frame")
    sr.Size = UDim2.new(1, -6, 0, 14); sr.Position = UDim2.new(0, 3, 0, 104)
    sr.BackgroundTransparency = 1; sr.Parent = card

    local lt = Instance.new("TextLabel")
    lt.Size = UDim2.new(1, 0, 1, 0); lt.BackgroundTransparency = 1
    lt.Text = string.format("❤️%s  👎%s  👁%s", FormatNumber(likes), FormatNumber(dislikes), FormatNumber(views))
    lt.TextColor3 = Colors.SubText; lt.TextSize = 8
    lt.Font = Enum.Font.Arial
    lt.TextXAlignment = Enum.TextXAlignment.Left; lt.Parent = sr

    local tml = Instance.new("TextLabel")
    tml.Size = UDim2.new(1, -6, 0, 12); tml.Position = UDim2.new(0, 3, 0, 122)
    tml.BackgroundTransparency = 1; tml.Text = "🕐 "..ut
    tml.TextColor3 = Colors.Teal; tml.TextSize = 8
    tml.Font = Enum.Font.Arial
    tml.TextXAlignment = Enum.TextXAlignment.Left; tml.Parent = card

    local vb = Instance.new("TextButton")
    vb.Size = UDim2.new(0.48, -2, 0, 20); vb.Position = UDim2.new(0, 3, 0, 141)
    vb.BackgroundColor3 = Colors.Secondary; vb.Text = "👁 View"; vb.TextColor3 = Colors.Text
    vb.TextSize = 9; vb.Font = Enum.Font.GothamMedium; vb.Parent = card
    Instance.new("UICorner", vb).CornerRadius = UDim.new(1, 0)

    local rb = Instance.new("TextButton")
    rb.Size = UDim2.new(0.48, -2, 0, 20); rb.Position = UDim2.new(0.5, 1, 0, 141)
    rb.BackgroundColor3 = hasKey and Colors.Orange or Colors.Primary
    rb.Text = hasKey and "🔑 Execute" or "▶ Execute"; rb.TextColor3 = Colors.Background
    rb.TextSize = 9; rb.Font = Enum.Font.GothamBold; rb.Parent = card
    Instance.new("UICorner", rb).CornerRadius = UDim.new(1, 0)

    card.MouseEnter:Connect(function() TweenService:Create(cs, TweenInfo.new(0.15), {Color = Colors.Primary}):Play() end)
    card.MouseLeave:Connect(function() TweenService:Create(cs, TweenInfo.new(0.15), {Color = isCur and Colors.Cyan or Colors.Border}):Play() end)

    vb.MouseButton1Click:Connect(function()
        currentScriptForView = data.script or "-- No script provided by API"
        currentUrlForView = GetScriptUrl(data)
        
        MT.Text = "📜 "..(data.title or "Script")
        GB.Text = "🎮 "..gname
        SB.Text = data.isPatched and "✕ Patched" or "✓ Working"
        SB.BackgroundColor3 = data.isPatched and Colors.Red or Colors.Green
        
        ML.Text = string.format("❤️ %s | 👎 %s", FormatNumber(likes), FormatNumber(dislikes))
        SC.Text = currentScriptForView

        CommentsTabBtn.Text = "💬 Comments (...)"
        SwitchModalTab("Desc")
        ModalOverlay.Visible = true

        if data.slug then FetchScriptDetails(data.slug) end
        if data._id then FetchAndRenderComments(data._id) end
    end)

    rb.MouseButton1Click:Connect(function()
        local s = data.script or ""
        if s ~= "" then
            PromptExecute(data.title or "Unknown Script", function(confirmed)
                if confirmed then
                    rb.Text = "⏳..."
                    task.spawn(function()
                        local ok = pcall(function() loadstring(s)() end)
                        rb.Text = ok and "✓ Executed" or "✕ Error"
                        task.delay(1, function() rb.Text = hasKey and "🔑 Execute" or "▶ Execute" end)
                    end)
                end
            end)
        end
    end)

    return true
end

local function LoadPage(tabName)
    local state = TabData[tabName]
    if state.isLoading or not state.hasMore then return end
    
    state.isLoading = true
    if CurrentTab == tabName then LoadingFrame.Visible = true end

    task.spawn(function()
        local url
        if state.query and state.query ~= "" then
            url = string.format("https://scriptblox.com/api/script/search?q=%s&page=%d&max=%d", HttpService:UrlEncode(state.query), state.page, maxPerPage)
        else
            url = string.format("https://scriptblox.com/api/script/fetch?page=%d&max=%d", state.page, maxPerPage)
        end

        local result = HttpRequest(url)
        if CurrentTab == tabName then LoadingFrame.Visible = false end

        if result and result.result then
            local scripts = result.result.scripts or {}
            local totalPages = result.result.totalPages or 1
            state.totalApi = result.result.total or state.totalApi

            if #scripts > 0 then
                for _, d in ipairs(scripts) do CreateCard(d, state.totalLoaded + 1, tabName) end
                state.page = state.page + 1
                state.hasMore = state.page <= totalPages
            else
                state.hasMore = false
            end
        else state.hasMore = false end

        state.isLoading = false
        if CurrentTab == tabName then UpdateUIState(); task.defer(function()
            if Containers[tabName] then
                local grid = Containers[tabName]:FindFirstChildOfClass("UIGridLayout")
                if grid then Containers[tabName].CanvasSize = UDim2.new(0, 0, 0, grid.AbsoluteContentSize.Y + 20) end
            end
        end) end
    end)
end

local function ClearContent(tabName)
    if Containers[tabName] then 
        for _, c in pairs(Containers[tabName]:GetChildren()) do 
            if c:IsA("Frame") then c:Destroy() end 
        end 
        Containers[tabName].CanvasPosition = Vector2.new(0, 0)
    end
end

local function TriggerSearch(queryText)
    CurrentTab = "Search"
    TabData.Search = { page = 1, hasMore = true, isLoading = false, totalLoaded = 0, totalApi = 0, query = queryText, seen = {} }
    ClearContent("Search"); UpdateUIState(); LoadPage("Search")
end

local function SwitchToFree()
    CurrentTab = "Free"; UpdateUIState()
    if TabData.Free.totalLoaded == 0 then LoadPage("Free") end
end

local function SwitchToThisGame()
    CurrentTab = "ThisGame"; UpdateUIState()
    if TabData.ThisGame.totalLoaded == 0 then
        local cleanName = CurrentGameName:gsub("%[.-%]", ""):gsub("%(.-%)", ""):gsub("【.-】", ""):match("^%s*(.-)%s*$")
        if not cleanName or cleanName == "" then cleanName = CurrentGameName end
        TabData.ThisGame.query = cleanName; LoadPage("ThisGame")
    end
end

local function RefreshCurrentTab()
    local tab = CurrentTab; local q = TabData[tab].query
    ClearContent(tab)
    TabData[tab] = { page = 1, hasMore = true, isLoading = false, totalLoaded = 0, totalApi = 0, query = q, seen = {} }
    UpdateUIState(); LoadPage(tab)
end

task.spawn(function()
    while ScreenGui.Parent do
        task.wait(0.25)
        if not MainFrame.Visible or ModalOverlay.Visible or ConfirmOverlay.Visible or ExecConfirmOverlay.Visible then continue end
        local activeContainer = Containers[CurrentTab]
        local state = TabData[CurrentTab]
        if not activeContainer or state.isLoading or not state.hasMore then continue end
        local scrollPos = activeContainer.CanvasPosition.Y
        local canvasH = activeContainer.CanvasSize.Y.Offset
        local frameH = activeContainer.AbsoluteSize.Y
        if canvasH <= frameH + 50 or (canvasH - scrollPos - frameH < 400) then LoadPage(CurrentTab) end
    end
end)

for _, sf in pairs(Containers) do
    sf:GetPropertyChangedSignal("CanvasPosition"):Connect(function()
        if sf.Visible then
            local state = TabData[CurrentTab]
            if state.isLoading or not state.hasMore then return end
            if sf.CanvasSize.Y.Offset - sf.CanvasPosition.Y - sf.AbsoluteSize.Y < 300 then
                task.defer(function() LoadPage(CurrentTab) end)
            end
        end
    end)
end

SearchBtn.MouseButton1Click:Connect(function() local t = SearchBox.Text; if t and t ~= "" then TriggerSearch(t) end end)
SearchBox.FocusLost:Connect(function(e) if e then local t = SearchBox.Text; if t and t ~= "" then TriggerSearch(t) end end end)
FreeTab.MouseButton1Click:Connect(function() SwitchToFree() end)
ThisGameTab.MouseButton1Click:Connect(function() SwitchToThisGame() end)
RefreshTab.MouseButton1Click:Connect(function() RefreshCurrentTab() end)

task.spawn(function() SwitchToFree() end)
