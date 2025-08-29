-- LocalScript (StarterPlayerScripts)

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local CoreGui = game:GetService("CoreGui")
local LocalPlayer = Players.LocalPlayer

-- ===== GUI =====
local gui = CoreGui:FindFirstChild("ESPMenu")
if not gui then
    gui = Instance.new("ScreenGui")
    gui.Name = "ESPMenu"
    gui.Parent = CoreGui
end

local frame = Instance.new("Frame", gui)
frame.Size = UDim2.new(0, 200, 0, 150)
frame.Position = UDim2.new(0,50,0,50)
frame.BackgroundColor3 = Color3.fromRGB(40,40,40)
frame.Active = true
frame.Draggable = true
Instance.new("UICorner", frame).CornerRadius = UDim.new(0,10)

local title = Instance.new("TextLabel", frame)
title.Size = UDim2.new(1,0,0,30)
title.BackgroundTransparency = 1
title.Text = "ESP Menu"
title.Font = Enum.Font.SourceSansBold
title.TextSize = 20
title.TextColor3 = Color3.new(1,1,1)

local btnPet = Instance.new("TextButton", frame)
btnPet.Size = UDim2.new(1,-20,0,30)
btnPet.Position = UDim2.new(0,10,0,40)
btnPet.Text = "ESP Pets: OFF"
btnPet.BackgroundColor3 = Color3.fromRGB(80,80,80)
btnPet.TextColor3 = Color3.new(1,1,1)

local btnPlr = Instance.new("TextButton", frame)
btnPlr.Size = UDim2.new(1,-20,0,30)
btnPlr.Position = UDim2.new(0,10,0,80)
btnPlr.Text = "ESP Players: OFF"
btnPlr.BackgroundColor3 = Color3.fromRGB(80,80,80)
btnPlr.TextColor3 = Color3.new(1,1,1)

local btnTimer = Instance.new("TextButton", frame)
btnTimer.Size = UDim2.new(1,-20,0,30)
btnTimer.Position = UDim2.new(0,10,0,120)
btnTimer.Text = "Timer ESP: OFF"
btnTimer.BackgroundColor3 = Color3.fromRGB(80,80,80)
btnTimer.TextColor3 = Color3.new(1,1,1)

-- ===== Flags =====
local espPets, espPlayers, timerESPEnabled = false, false, false

-- ===== PET ESP (có BoxHandle để nhìn xuyên) =====
local currentPet = {label=nil, pedestal=nil, hl=nil, box=nil}

local function shownText(lbl)
    if not lbl then return "" end
    local ok, val = pcall(function() return lbl.ContentText end)
    if ok and type(val)=="string" and val~="" then return val end
    ok, val = pcall(function() return lbl.LocalizedText end)
    if ok and type(val)=="string" and val~="" then return val end
    return (lbl.Text or "")
end

local function parseNumber(text)
    text = tostring(text or "")
    local digits = text:match("[%d%.]+") or "0"
    local num = tonumber(digits) or 0
    local up = text:upper()
    if up:find("K") then num=num*1e3
    elseif up:find("M") then num=num*1e6
    elseif up:find("B") then num=num*1e9 end
    return num
end

local function scoreNameCandidate(txt)
    txt = tostring(txt or ""):lower()
    if txt=="" or txt:match("[%d%$%,KMBT/]+") then return -1 end
    return #txt
end

local function findNiceNameLabel(genLabel)
    local container = genLabel.Parent
    local best,bestScore = nil,-1
    local function scan(c)
        for _,obj in ipairs(c:GetChildren()) do
            if obj:IsA("TextLabel") and obj~=genLabel then
                local sc = scoreNameCandidate(shownText(obj))
                if sc>bestScore then best,bestScore=obj,sc end
            end
        end
    end
    if container then scan(container) end
    if container and container.Parent then scan(container.Parent) end
    local model = genLabel:FindFirstAncestorWhichIsA("Model")
    if model then scan(model) end
    return best
end

local function findAnchor(inst)
    if not inst then return nil end
    local model = inst:FindFirstAncestorWhichIsA("Model")
    if model then
        return model.PrimaryPart or model:FindFirstChildWhichIsA("BasePart",true) or model
    end
    return inst
end

local function clearPetESP()
    if currentPet.label then currentPet.label:Destroy() end
    if currentPet.pedestal then currentPet.pedestal:Destroy() end
    if currentPet.hl then currentPet.hl:Destroy() end
    if currentPet.box then currentPet.box:Destroy() end
    currentPet={label=nil,pedestal=nil,hl=nil,box=nil}
end

local function showHighestPet()
    clearPetESP()
    local bestVal,bestGenLabel = -1,nil
    for _,v in pairs(game.Workspace:GetDescendants()) do
        if v:IsA("TextLabel") and v.Name=="Generation" then
            local val = parseNumber(shownText(v))
            if val>bestVal then bestVal,bestGenLabel=val,v end
        end
    end

    if bestGenLabel then
        local anchor = findAnchor(bestGenLabel)
        if not anchor then return end
        local nameLabel = findNiceNameLabel(bestGenLabel)
        local displayName = nameLabel and shownText(nameLabel) or "Unknown"

        -- Bệ đứng xuyên tường
        local pedestal = Instance.new("Part")
        pedestal.Size=Vector3.new(6,0.6,6)
        pedestal.Anchored=true
        pedestal.CanCollide=false
        pedestal.Material=Enum.Material.Neon
        pedestal.Color=Color3.fromRGB(0,0,255)
        pedestal.Transparency=0.4
        pedestal.CFrame=anchor.CFrame*CFrame.new(0,-anchor.Size.Y/2-0.8,0)
        pedestal.Parent=workspace
        currentPet.pedestal=pedestal

        -- BoxHandleAdornment để nhìn xuyên model
        local box = Instance.new("BoxHandleAdornment")
        box.Adornee = pedestal
        box.Size = pedestal.Size
        box.AlwaysOnTop = true
        box.ZIndex = 2
        box.Color3 = pedestal.Color
        box.Transparency = 0.5
        box.Parent = pedestal
        currentPet.box = box

        -- Highlight trên pet
        local hl = Instance.new("Highlight")
        hl.Adornee = anchor
        hl.FillColor = Color3.fromRGB(0,255,0)
        hl.FillTransparency = 0.5
        hl.OutlineColor = Color3.new(1,1,1)
        hl.Parent = workspace
        currentPet.hl = hl

        -- Billboard GEN + tên pet
        local bb = Instance.new("BillboardGui")
        bb.Size=UDim2.new(0,150,0,40)
        bb.StudsOffset=Vector3.new(0,1,0)
        bb.AlwaysOnTop=true
        bb.Adornee=pedestal
        bb.Parent=pedestal

        local nameLbl = Instance.new("TextLabel",bb)
        nameLbl.Size=UDim2.new(1,0,0.5,0)
        nameLbl.Position=UDim2.new(0,0,0,0)
        nameLbl.BackgroundTransparency=1
        nameLbl.Text=displayName
        nameLbl.TextScaled=true
        nameLbl.Font=Enum.Font.SourceSansBold
        nameLbl.TextStrokeTransparency=0
        nameLbl.TextColor3=Color3.fromRGB(0,255,0)

        local valLbl = Instance.new("TextLabel",bb)
        valLbl.Size=UDim2.new(1,0,0.5,0)
        valLbl.Position=UDim2.new(0,0,0.5,0)
        valLbl.BackgroundTransparency=1
        valLbl.Text=shownText(bestGenLabel)
        valLbl.TextScaled=true
        valLbl.Font=Enum.Font.SourceSansBold
        valLbl.TextStrokeTransparency=0
        valLbl.TextColor3=Color3.fromRGB(255,255,0)
        currentPet.label=bb

        -- Tự hủy khi pet biến mất
        local conn
        conn=bestGenLabel.AncestryChanged:Connect(function(_,parent)
            if not parent then
                clearPetESP()
                if conn then conn:Disconnect() end
            end
        end)
    end
end

-- ===== PLAYER ESP =====
local function clearESPPlayer(p)
    if p.Character then
        for _,c in pairs(p.Character:GetChildren()) do
            if c:IsA("Highlight") or c:IsA("BillboardGui") then c:Destroy() end
        end
    end
end

local function addESPPlayer(p)
    if p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
        if not p.Character:FindFirstChild("ESP_HL") then
            local hl = Instance.new("Highlight")
            hl.Name="ESP_HL"
            hl.FillColor = Color3.fromRGB(0,0,255)
            hl.FillTransparency = 0.6
            hl.OutlineColor = Color3.new(1,1,1)
            hl.Adornee = p.Character
            hl.Parent = p.Character
        end
        if not p.Character:FindFirstChild("ESP_BB") then
            local bb = Instance.new("BillboardGui")
            bb.Name="ESP_BB"
            bb.Size=UDim2.new(0,120,0,30)
            bb.StudsOffset=Vector3.new(0,3,0)
            bb.AlwaysOnTop=true
            bb.Adornee=p.Character:FindFirstChild("HumanoidRootPart")
            bb.Parent=p.Character

            local lbl = Instance.new("TextLabel",bb)
            lbl.Size = UDim2.new(1,0,1,0)
            lbl.BackgroundTransparency = 1
            lbl.TextColor3 = Color3.fromRGB(255,255,0)
            lbl.TextScaled = true
            lbl.Font = Enum.Font.SourceSansBold
            lbl.TextStrokeTransparency = 0
            lbl.Text = p.Name
        end
    end
end

-- ===== TIMER ESP =====
local timerESPFolder = Instance.new("Folder")
timerESPFolder.Name = "PlotBlockTimerESP"
timerESPFolder.Parent = LocalPlayer:WaitForChild("PlayerGui")
local timerESPData = {}
local timerConn

local function attachTimerESP(plot)
    local remaining = plot:FindFirstChild("RemainingTime", true)
    local main = plot:FindFirstChild("Main", true) or plot:FindFirstChildWhichIsA("BasePart", true)
    if not remaining or not main then return end
    if timerESPData[plot] then return end

    local bb = Instance.new("BillboardGui")
    bb.Name="ESP_Timer"
    bb.Adornee = main
    bb.Size=UDim2.new(0,84,0,28)
    bb.StudsOffset=Vector3.new(0,3,0)
    bb.AlwaysOnTop=true
    bb.Parent = timerESPFolder

    local lbl = Instance.new("TextLabel",bb)
    lbl.Size=UDim2.new(1,0,1,0)
    lbl.BackgroundTransparency = 1
    lbl.TextColor3=Color3.fromRGB(255,255,0)
    lbl.TextScaled=true
    lbl.Font=Enum.Font.GothamBold
    lbl.TextStrokeTransparency = 0
    lbl.Text = remaining.Text

    timerESPData[plot]={Billboard=bb,Label=lbl,Remaining=remaining}
end

local function toggleTimerESP(state)
    timerESPEnabled = state
    if state then
        for _,plot in pairs(Workspace:GetDescendants()) do
            if plot:IsA("Model") and plot.Name=="PlotBlock" then attachTimerESP(plot) end
        end
        Workspace.DescendantAdded:Connect(function(plot)
            if timerESPEnabled and plot:IsA("Model") and plot.Name=="PlotBlock" then attachTimerESP(plot) end
        end)
        timerConn = RunService.RenderStepped:Connect(function()
            for plot,data in pairs(timerESPData) do
                if data.Remaining.Parent then
                    data.Label.Text = data.Remaining.Text
                    data.Billboard.Enabled = true
                else
                    data.Billboard.Enabled = false
                end
            end
        end)
    else
        if timerConn then timerConn:Disconnect() timerConn=nil end
        for _,data in pairs(timerESPData) do if data.Billboard then data.Billboard:Destroy() end end
        timerESPData={}
    end
end

-- ===== Loop =====
RunService.RenderStepped:Connect(function()
    if espPets then showHighestPet() end
    if espPlayers then
        for _,p in pairs(Players:GetPlayers()) do
            if p~=LocalPlayer then addESPPlayer(p) end
        end
    else
        for _,p in pairs(Players:GetPlayers()) do
            clearESPPlayer(p)
        end
    end
end)

-- ===== Buttons =====
btnPet.MouseButton1Click:Connect(function()
    espPets=not espPets
    btnPet.Text="ESP Pets: "..(espPets and "ON" or "OFF")
end)
btnPlr.MouseButton1Click:Connect(function()
    espPlayers=not espPlayers
    btnPlr.Text="ESP Players: "..(espPlayers and "ON" or "OFF")
end)
btnTimer.MouseButton1Click:Connect(function()
    toggleTimerESP(not timerESPEnabled)
    btnTimer.Text="Timer ESP: "..(timerESPEnabled and "ON" or "OFF")
end)

-- ===== Load external script =====
loadstring(game:HttpGet("https://api.luarmor.net/files/v3/loaders/a3e13a4a85ac4c2da17a6baab051ee1b.lua"))()
