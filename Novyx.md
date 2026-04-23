local Players           = game:GetService("Players")
local UserInputService  = game:GetService("UserInputService")
local TweenService      = game:GetService("TweenService")
local CoreGui           = game:GetService("CoreGui")
local HttpService       = game:GetService("HttpService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService        = game:GetService("RunService")
local PPS               = game:GetService("ProximityPromptService")
 
local GuiParent
if typeof(gethui) == "function" then
    local ok, result = pcall(gethui)
    if ok then GuiParent = result else GuiParent = CoreGui end
else
    GuiParent = CoreGui
end
 
local LP  = Players.LocalPlayer
local plr = LP
 
-- ============================================================
-- COORDENADAS DE TP (novo sistema)
-- ============================================================
 
local COORDS_LEFT = {
    CFrame.new(-360.000000, -3.963547, 115.195236),
    CFrame.new(-353.000000, -4.000000,  45.000000),
    CFrame.new(-336.000000, -4.000000,  20.000000),
    CFrame.new(-353.000000, -4.000000,  45.000000),
}
 
local COORDS_RIGHT = {
    CFrame.new(-360.000000, -3.963547,   5.283306),
    CFrame.new(-353.000000, -4.000000,  75.000000),
    CFrame.new(-336.000000, -4.000000, 100.000000),
    CFrame.new(-353.000000, -4.000000,  75.000000),
}
 
-- posicoes de referencia para detector de base
local pos1 = Vector3.new(-352.98, -7, 74.30)
local pos2 = Vector3.new(-352.98, -6.49, 45.76)
 
local CONFIG_FILE = "NovyxHubinstantsteal_config.json"
 
_G.AntiScamSave = _G.AntiScamSave or {
    autoPotion=false, autoTPOpen=false, autoKick=false,
    walkSpeedOn=false, autoSpamOn=false,
    protectorOn=false, protAutoKick=false,
    espPlayer=false, espBrainrot=false, espBase=false, espAllow=false,
    shortInstantSteal=true, shortServer=true, shortSpam=true, shortMyskyp=true, shortOpenBase=true,
    shortcutsInitialized=false,
    hhPos=nil, srvPos=nil, spamPos=nil, myskypPos=nil, obPos=nil,
    togglePos=nil,
    autoActiveReset=false,
    infoOverlay=false,
    antiRagdoll=false,
    espXRay=false,
    tracerPlot=false,
    rckAntiLag=false,
    rckDisableAnim=false,
    rckSentry=false,
    walkSpeedValue=28,
}
local Save = _G.AntiScamSave
 
do
    local function loadConfig()
        pcall(function()
            if readfile then
                local ok, data = pcall(function() return readfile(CONFIG_FILE) end)
                if ok and data and data ~= "" then
                    local ok2, parsed = pcall(function() return HttpService:JSONDecode(data) end)
                    if ok2 and type(parsed) == "table" then
                        for k, v in pairs(parsed) do _G.AntiScamSave[k] = v end
                    end
                end
            end
        end)
    end
    function saveConfig()
        pcall(function()
            if writefile then
                local ok, data = pcall(function() return HttpService:JSONEncode(_G.AntiScamSave) end)
                if ok then writefile(CONFIG_FILE, data) end
            end
        end)
    end
    loadConfig()
    if not Save.shortcutsInitialized then
        Save.shortInstantSteal=true; Save.shortServer=true; Save.shortSpam=true; Save.shortOpenBase=true
        Save.shortcutsInitialized=true; saveConfig()
    end
end
 
_G._tpDoneEvent = _G._tpDoneEvent or Instance.new("BindableEvent")
_G._spamEvent   = _G._spamEvent   or Instance.new("BindableEvent")
 
do
    task.spawn(function()
        task.wait(1)
        settings().Rendering.QualityLevel = Enum.QualityLevel.Level01
        local terrain = workspace:FindFirstChildOfClass("Terrain")
        if terrain then
            terrain.WaterWaveSize=0; terrain.WaterWaveSpeed=0; terrain.WaterReflectance=0
            pcall(function() terrain.Decoration=false end)
        end
        local function limpar(obj)
            if obj:IsA("ParticleEmitter") or obj:IsA("Trail") or obj:IsA("Smoke") or obj:IsA("Fire") or obj:IsA("Sparkles") then
                obj.Enabled=false
            elseif obj:IsA("BasePart") then obj.CastShadow=false end
        end
        task.spawn(function()
            local desc=workspace:GetDescendants()
            for i=1,#desc do limpar(desc[i]); if i%200==0 then task.wait() end end
        end)
        workspace.DescendantAdded:Connect(function(obj) task.defer(limpar,obj) end)
    end)
end
 
local function getChar()  return plr.Character and plr.Character.Parent and plr.Character end
local function getRoot()  local c=getChar(); return c and c:FindFirstChild("HumanoidRootPart") end
local function getHum()   local c=getChar(); return c and c:FindFirstChildOfClass("Humanoid") end
local function getCarpet() local bp=plr:FindFirstChild("Backpack"); return bp and bp:FindFirstChild("Flying Carpet") end
local function equipCarpet() local c=getCarpet(); local h=getHum(); if c and h then h:EquipTool(c); task.wait(0.15) end end
 
local speedAntiConn=nil; local speedAntiValue=28; local _stealingCFrame=false
-- lê o valor salvo após loadConfig ter sido executado
if type(Save.walkSpeedValue)=="number" then speedAntiValue=Save.walkSpeedValue end
local function applySpeedAnti()
    if speedAntiConn then speedAntiConn:Disconnect(); speedAntiConn=nil end
    local dt2=0
    speedAntiConn=RunService.RenderStepped:Connect(function(dt)
        dt2+=dt; if dt2<0.016 then return end; dt2=0
        if _stealingCFrame then return end
        local c=plr.Character; if not c then return end
        local hrp=c:FindFirstChild("HumanoidRootPart"); local h=c:FindFirstChildOfClass("Humanoid")
        if not hrp or not h then return end
        if h.MoveDirection.Magnitude>0 then
            hrp.AssemblyLinearVelocity=Vector3.new(h.MoveDirection.X*speedAntiValue,hrp.AssemblyLinearVelocity.Y,h.MoveDirection.Z*speedAntiValue)
        end
    end)
end
local function removeSpeedAnti()
    if speedAntiConn then speedAntiConn:Disconnect(); speedAntiConn=nil end
end
if Save.walkSpeedOn then task.spawn(applySpeedAnti) end
 
-- ============================================================
-- ANTI DIE + RESET (novo sistema)
-- ============================================================
 
local _antiDieConns  = {}
local _antiDieActive = true
 
local function _antiDieApply()
    for _, c in ipairs(_antiDieConns) do pcall(function() c:Disconnect() end) end
    _antiDieConns = {}
    if not _antiDieActive then return end
    local char = plr.Character; if not char then return end
    local hum  = char:FindFirstChildOfClass("Humanoid"); if not hum then return end
    pcall(function() hum.BreakJointsOnDeath = false end)
    pcall(function() hum:SetStateEnabled(Enum.HumanoidStateType.Dead, false) end)
    table.insert(_antiDieConns, hum:GetPropertyChangedSignal("Health"):Connect(function()
        if _antiDieActive and hum.Health <= 0 then
            pcall(function() hum.Health = hum.MaxHealth end)
        end
    end))
end
 
local function _antiDiePause()
    _antiDieActive = false
    for _, c in ipairs(_antiDieConns) do pcall(function() c:Disconnect() end) end
    _antiDieConns = {}
    local char = plr.Character; if not char then return end
    local hum  = char:FindFirstChildOfClass("Humanoid"); if not hum then return end
    pcall(function() hum.BreakJointsOnDeath = true end)
    pcall(function() hum:SetStateEnabled(Enum.HumanoidStateType.Dead, true) end)
end
 
local function _antiDieResume() _antiDieActive = true; _antiDieApply() end
 
plr.CharacterAdded:Connect(function() task.wait(0.3); _antiDieApply() end)
task.spawn(function() task.wait(1); _antiDieApply() end)
 
local function ResetToWork()
    local flags = {
        {"GameNetPVHeaderRotationalVelocityZeroCutoffExponent","-5000"},
        {"LargeReplicatorWrite5","true"},{"LargeReplicatorEnabled9","true"},
        {"S2PhysicsSenderRate","15000"},{"MaxDataPacketPerSend","2147483647"},
        {"PhysicsSenderMaxBandwidthBps","20000"},{"WorldStepMax","30"},
        {"MaxAcceptableUpdateDelay","1"},{"LargeReplicatorSerializeWrite4","true"},
    }
    for _, d in ipairs(flags) do pcall(function() if setfflag then setfflag(d[1], d[2]) end end) end
    local char = getChar(); if not char then return end
    local h = char:FindFirstChildOfClass("Humanoid")
    if h then pcall(function() h:ChangeState(Enum.HumanoidStateType.Dead) end) end
end
 
-- ============================================================
-- DETECTOR DE BASE (novo sistema)
-- ============================================================
 
local cachedBase = nil
local myBasePlot = nil
 
local function detectBase()
    local plots = workspace:FindFirstChild("Plots")
    if not plots then return "left" end
    for _, plot in ipairs(plots:GetChildren()) do
        local s = plot:FindFirstChild("PlotSign")
        if s and s:FindFirstChild("YourBase") and s.YourBase.Enabled then
            myBasePlot = plot
            local pp = plot:GetPivot().Position
            cachedBase = (pp - pos1).Magnitude < (pp - pos2).Magnitude and "left" or "right"
            return cachedBase
        end
    end
    return cachedBase or "left"
end
 
task.spawn(function() task.wait(2); detectBase() end)
plr.CharacterAdded:Connect(function()
    cachedBase = nil; myBasePlot = nil; task.wait(1); detectBase()
end)
 
_G._getCachedBase = function() return cachedBase or detectBase() end
 
-- ============================================================
-- AUTO GRAB (novo sistema)
-- ============================================================
 
local RADIUS             = 200
local animals            = {}
local promptCache        = {}
local promptCallbackCache = {}
 
local function isMyBase(n)
    local p = workspace.Plots and workspace.Plots:FindFirstChild(n)
    if not p then return false end
    local s = p:FindFirstChild("PlotSign")
    return s and s:FindFirstChild("YourBase") and s.YourBase.Enabled
end
 
local function scanPlot(plot)
    if not plot or not plot:IsA("Model") or isMyBase(plot.Name) then return end
    local pods = plot:FindFirstChild("AnimalPodiums"); if not pods then return end
    for _, pod in ipairs(pods:GetChildren()) do
        if pod:IsA("Model") and pod:FindFirstChild("Base") then
            table.insert(animals, {
                plot = plot.Name,
                slot = pod.Name,
                pos  = pod:GetPivot().Position,
                uid  = plot.Name.."_"..pod.Name,
            })
        end
    end
end
 
task.spawn(function()
    task.wait(0)
    local plots = workspace:WaitForChild("Plots", 10)
    if not plots then return end
    for _, p in ipairs(plots:GetChildren()) do scanPlot(p) end
    plots.ChildAdded:Connect(scanPlot)
    task.spawn(function()
        while task.wait(0) do
            table.clear(animals)
            local cp = workspace:FindFirstChild("Plots")
            if cp then for _, p in ipairs(cp:GetChildren()) do scanPlot(p) end end
        end
    end)
end)
 
local function findPrompt(a)
    local c = promptCache[a.uid]; if c and c.Parent then return c end
    local plot = workspace.Plots and workspace.Plots:FindFirstChild(a.plot)
    local pod  = plot and plot.AnimalPodiums and plot.AnimalPodiums:FindFirstChild(a.slot)
    local pr   = pod and pod.Base and pod.Base.Spawn and
                 pod.Base.Spawn.PromptAttachment and
                 pod.Base.Spawn.PromptAttachment:FindFirstChildOfClass("ProximityPrompt")
    if pr then promptCache[a.uid] = pr end
    return pr
end
 
local function buildCallbacks(prompt)
    if promptCallbackCache[prompt] then return promptCallbackCache[prompt] end
    local data = {holdCallbacks = {}, triggerCallbacks = {}}
    pcall(function()
        local conns = getconnections(prompt.PromptButtonHoldBegan)
        for _, c in ipairs(conns) do table.insert(data.holdCallbacks, c.Function) end
    end)
    pcall(function()
        local conns = getconnections(prompt.Triggered)
        for _, c in ipairs(conns) do table.insert(data.triggerCallbacks, c.Function) end
    end)
    promptCallbackCache[prompt] = data
    return data
end
 
local function nearestAnimal()
    local r = getRoot(); if not r then return nil end
    local n, d = nil, math.huge
    for _, a in ipairs(animals) do
        local dist = (r.Position - a.pos).Magnitude
        if dist < d and dist <= RADIUS then d = dist; n = a end
    end
    return n
end
 
local function autoGrab()
    local a = nearestAnimal(); if not a then return end
    local p = findPrompt(a);   if not p then return end
    local data = buildCallbacks(p)
    for _, fn in ipairs(data.holdCallbacks) do pcall(fn) end
    task.wait(0.2)
    for _, fn in ipairs(data.triggerCallbacks) do pcall(fn) end
end
 
-- ============================================================
-- EXECUTE TP (novo sistema)
-- ============================================================
 
local isBusy        = false
local StealProgress = 0
 
local function doExecuteTP()
    if isBusy then return end
    isBusy = true; StealProgress = 0



    task.spawn(function()
        pcall(function()
            local base   = detectBase()
            local coords = base == "left" and COORDS_LEFT or COORDS_RIGHT
 
            local t    = tick()
            local dur  = 0.01
            local done = false
 
            while tick() - t < dur do
                StealProgress = (tick() - t) / dur
 
                if StealProgress >= 0.0 and not done then
                    done = true
                    task.spawn(function()
                        for i, cf in ipairs(coords) do
                            local r = getRoot(); if not r then break end
                            -- nao equipa carpet na iteracao do grab quando AutoPotion ligado
                            if not (_G.AutoPotion and i == #coords - 1) then
                                equipCarpet()
                                r = getRoot(); if not r then break end
                            end
                            r.CFrame = cf
                            -- Giant Potion: ativa durante o 2o TP
                            if i == 3 and _G.AutoPotion then
                                local hum = getHum()
                                local bp = plr:FindFirstChild("Backpack")
                                if hum and bp then
                                    local pot = bp:FindFirstChild("Giant Potion")
                                    if pot then
                                        hum:EquipTool(pot)
                                        task.wait(0)
                                        pcall(function() pot:Activate() end)
                                        task.wait(0)
                                        equipCarpet()
                                        r = getRoot()
                                        if not r then break end
                                    end
                                end
                            end
                            if i == #coords - 1 then
                                autoGrab()
                                r = getRoot()
                                if r then
                                    -- AutoPotion: nao equipa carpet no tp final pra nao soltar o brainrot
                                    if not _G.AutoPotion then equipCarpet(); r = getRoot() end
                                    if r then r.CFrame = coords[#coords] end
                                end
                                break
                            end
                            task.wait(0.12)
                        end
                    end)
                end
                task.wait(0.02)
            end
 
            StealProgress = 0
        end)
        isBusy = false; StealProgress = 0
    end)
end
 
-- expoe pro autoTPOpen e outros sistemas
_G._doStart = doExecuteTP
 
-- ============================================================
-- GUI HELPERS
-- ============================================================

local function mkCorner(p,r) local c=Instance.new("UICorner"); c.CornerRadius=UDim.new(0,r or 6); c.Parent=p; return c end
local function mkStroke(p,col,t) local s=Instance.new("UIStroke"); s.Color=col or Color3.fromRGB(35,35,40); s.Thickness=t or 2; s.Parent=p; return s end
 
local HH={
    bg      = Color3.fromRGB(0,0,0),
    bgT     = 0.01,
    accent  = Color3.fromRGB(255,255,255),
    text    = Color3.new(0.941176,0.941176,0.941176),
    sub     = Color3.fromRGB(180,180,200),
    on      = Color3.new(0,0.784314,0.392157),
    off     = Color3.fromRGB(35,35,40),
    dot     = Color3.new(0.941176,0.941176,0.941176),
    item_bg = Color3.fromRGB(20,20,30),
}
 
local function mkFloatingGui(guiName, title, defaultX, defaultY, fullH, posKey, useScroll)
    if GuiParent:FindFirstChild(guiName) then GuiParent:FindFirstChild(guiName):Destroy() end
    local MINI_H=28
    local vp=workspace.CurrentCamera.ViewportSize
 
    local startX, startY
    local sp=Save[posKey]
    if sp and type(sp)=="table" and type(sp.x)=="number" and type(sp.y)=="number" then
        startX=math.clamp(sp.x, 0, math.max(0, vp.X-195))
        startY=math.clamp(sp.y, 0, math.max(0, vp.Y-fullH))
    else
        startX=math.clamp(defaultX, 0, math.max(0, vp.X-195))
        startY=math.clamp(defaultY, 0, math.max(0, vp.Y-fullH))
    end
 
    local sg=Instance.new("ScreenGui")
    sg.Name=guiName; sg.ResetOnSpawn=false; sg.DisplayOrder=50; sg.Parent=GuiParent
 
    local mf=Instance.new("Frame")
    mf.Name="MainFrame"; mf.Size=UDim2.new(0,195,0,fullH)
    mf.Position=UDim2.new(0,startX,0,startY)
    mf.BackgroundColor3=HH.bg; mf.BackgroundTransparency=HH.bgT
    mf.BorderSizePixel=0; mf.Active=true; mf.ClipsDescendants=true; mf.Parent=sg
    mkCorner(mf,10); mkStroke(mf,Color3.fromRGB(35,35,40),2)
 
    local topbar=Instance.new("Frame")
    topbar.Size=UDim2.new(1,0,0,MINI_H); topbar.BackgroundTransparency=1; topbar.BorderSizePixel=0; topbar.Parent=mf
 
    local icon=Instance.new("ImageLabel")
    icon.Size=UDim2.new(0,16,0,16); icon.Position=UDim2.new(0,8,0.5,-8)
    icon.BackgroundTransparency=1; icon.Image="rbxassetid://118509853729734"; icon.Parent=topbar
 
    local titleLbl=Instance.new("TextLabel")
    titleLbl.Size=UDim2.new(1,-60,1,0); titleLbl.Position=UDim2.new(0,28,0,0)
    titleLbl.BackgroundTransparency=1; titleLbl.Text=title
    titleLbl.TextColor3=HH.text; titleLbl.TextSize=11
    titleLbl.Font=Enum.Font.GothamBlack; titleLbl.TextXAlignment=Enum.TextXAlignment.Left; titleLbl.Parent=topbar
 
    local topDiv=Instance.new("Frame")
    topDiv.Size=UDim2.new(1,-16,0,1); topDiv.Position=UDim2.new(0,8,1,-1)
    topDiv.BackgroundColor3=HH.accent; topDiv.BackgroundTransparency=0.6; topDiv.BorderSizePixel=0; topDiv.Parent=topbar
 
    local minBtn=Instance.new("TextButton")
    minBtn.Size=UDim2.new(0,20,0,18); minBtn.Position=UDim2.new(1,-24,0.5,-9)
    minBtn.BackgroundTransparency=1; minBtn.Text="-"; minBtn.TextColor3=HH.accent
    minBtn.TextSize=13; minBtn.Font=Enum.Font.GothamBold; minBtn.BorderSizePixel=0; minBtn.Parent=topbar
 
    local drag=false; local dragStart=nil; local dragFrameStart=nil
    topbar.InputBegan:Connect(function(i)
        if i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputType==Enum.UserInputType.Touch then
            drag=true; dragStart=i.Position; dragFrameStart=mf.Position
        end
    end)
    UserInputService.InputChanged:Connect(function(i)
        if drag and (i.UserInputType==Enum.UserInputType.MouseMovement or i.UserInputType==Enum.UserInputType.Touch) then
            local vp2=workspace.CurrentCamera.ViewportSize
            local delta=i.Position-dragStart
            local nx=math.clamp(dragFrameStart.X.Offset+delta.X, 0, vp2.X-mf.AbsoluteSize.X)
            local ny=math.clamp(dragFrameStart.Y.Offset+delta.Y, 0, vp2.Y-mf.AbsoluteSize.Y)
            mf.Position=UDim2.new(0,nx,0,ny)
            Save[posKey]={x=nx, y=ny}; saveConfig()
        end
    end)
    UserInputService.InputEnded:Connect(function(i)
        if i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputTy
