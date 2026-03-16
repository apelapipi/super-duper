if getgenv().SA_Hub then return end
getgenv().SA_Hub = true

-- ================================================================
--  STRANDED AWAY HUB  v4  |  PratesDev
-- ================================================================
local Plrs  = game:GetService("Players")
local RS    = game:GetService("RunService")
local RepS  = game:GetService("ReplicatedStorage")
local VU    = game:GetService("VirtualUser")
local TpS   = game:GetService("TeleportService")
local HS    = game:GetService("HttpService")
local UIS   = game:GetService("UserInputService")
local CS    = game:GetService("CollectionService")
local Light = game:GetService("Lighting")

local LP       = Plrs.LocalPlayer
local Cam      = workspace.CurrentCamera
local Controls = require(LP:WaitForChild("PlayerScripts"):WaitForChild("PlayerModule")):GetControls()
local CoreGui  = gethui and gethui() or game:GetService("CoreGui")

local WindUI = loadstring(game:HttpGet("https://github.com/Footagesus/WindUI/releases/latest/download/main.lua"))()
local Window = WindUI:CreateWindow({
  Title="Stranded Away Hub v5", Icon="anchor", Author="PratesDev",
  Folder="StrandedAwayHub", Size=UDim2.fromOffset(640,540),
  Transparent=true, Theme="Purple", Resizable=true,
  OpenButton={Title="Open Hub", CornerRadius=UDim.new(1,0), StrokeThickness=3}
})

local TC   = Window:Tab({Title="Combat",  Icon="swords"})
local TI   = Window:Tab({Title="Items",   Icon="package"})
local TM   = Window:Tab({Title="Move",    Icon="move"})
local TV   = Window:Tab({Title="Visuals", Icon="eye"})
local TP   = Window:Tab({Title="Optim",   Icon="gauge"})
local TS2  = Window:Tab({Title="Server",  Icon="globe"})
local TST  = Window:Tab({Title="Settings",Icon="settings"})
local TEXP = Window:Tab({Title="Lab",     Icon="flask-conical"})

-- ================================================================
--  CONFIG
-- ================================================================
local Cfg = {
  KillAura=false, AuraDist=30, AuraInterval=0.15,
  CollectDropped=false, CarryDropped=false, CarryGrabbable=false,
  GrabInterval=0.5, GrabDist=0,
  AutoEat=false, EatInterval=10,
  AutoDrink=false, DrinkInterval=10,
  FarmDay=false,
  Fly=false, FlySpeed=50,
  Freecam=false, FCSpeed=2,
  Noclip=false, InfJ=false, AFK=false,
  WalkSpeed=16, JumpPower=50, JumpHeight=7.2,
  LockAim=false,
  EspN=false, EspB=false, EspL=false, EspDist=false,
  RndC=false, EspSkeleton=false, PlayerHealth=false,
  MobEsp=false, MobHealth=false, MobDist=false,
  ItemEspDropped=false, ItemEspGrabbable=false,
  Crosshair=false, StatsUI=false,
  SaveCfg=false,
}

local function SaveConfig()
  if not isfolder("StrandedAwayHub") then makefolder("StrandedAwayHub") end
  pcall(function()
    writefile("StrandedAwayHub/config.json", HS:JSONEncode({
      WalkSpeed=Cfg.WalkSpeed, JumpPower=Cfg.JumpPower,
      FlySpeed=Cfg.FlySpeed,   FCSpeed=Cfg.FCSpeed,
      AuraDist=Cfg.AuraDist,
    }))
  end)
end
pcall(function()
  if isfile("StrandedAwayHub/config.json") then
    for k,v in pairs(HS:JSONDecode(readfile("StrandedAwayHub/config.json"))) do Cfg[k]=v end
  end
end)

local function HttpReq(url)
  local req=(syn and syn.request) or request or http_request
  local ok,res=pcall(req,{Url=url,Method="GET"})
  if ok and res and res.Body then
    local j,d=pcall(function() return HS:JSONDecode(res.Body) end)
    if j then return d end
  end
end
local function FmtShort(n)
  n=n or 0
  if n>=1e9 then return string.format("%.1fB",n/1e9)
  elseif n>=1e6 then return string.format("%.1fM",n/1e6)
  elseif n>=1e3 then return string.format("%.1fK",n/1e3) end
  return tostring(math.floor(n))
end

-- ================================================================
--  GUI: CROSSHAIR  (IgnoreGuiInset=true → centro real da tela)
-- ================================================================
local CrossGui = Instance.new("ScreenGui")
CrossGui.Name="SA_Cross" CrossGui.IgnoreGuiInset=true
CrossGui.ResetOnSpawn=false CrossGui.Parent=CoreGui CrossGui.Enabled=false
local function MkCL(pos,sz)
  local f=Instance.new("Frame",CrossGui)
  f.BackgroundColor3=Color3.new(1,1,1) f.BackgroundTransparency=0.1
  f.BorderSizePixel=0 f.Position=pos f.Size=sz
end
MkCL(UDim2.new(0.5,0,0.5,-7), UDim2.new(0,1,0,14))
MkCL(UDim2.new(0.5,-7,0.5,0), UDim2.new(0,14,0,1))

-- ================================================================
--  GUI: STATS HUD
-- ================================================================
local StatsGui=Instance.new("ScreenGui")
StatsGui.Name="SA_Stats" StatsGui.IgnoreGuiInset=true
StatsGui.ResetOnSpawn=false StatsGui.Parent=CoreGui StatsGui.Enabled=false
local StatsTxt=Instance.new("TextLabel",StatsGui)
StatsTxt.Size=UDim2.fromOffset(450,22) StatsTxt.Position=UDim2.new(0,10,1,-32)
StatsTxt.BackgroundTransparency=1 StatsTxt.TextColor3=Color3.new(1,1,1)
StatsTxt.TextStrokeTransparency=0.3 StatsTxt.TextStrokeColor3=Color3.new(0,0,0)
StatsTxt.RichText=true StatsTxt.TextXAlignment=Enum.TextXAlignment.Left
StatsTxt.Font=Enum.Font.GothamBold StatsTxt.TextSize=13
local lastStatT=0 local cachedFPS=60
local function FpsClr(f)
  if f<20 then return "#FF2222" elseif f<30 then return "#FF6600"
  elseif f<40 then return "#FFB300" elseif f<50 then return "#CCDD00"
  else return "#33EE66" end
end

-- ================================================================
--  GUI: CONSOLE
-- ================================================================
local FCPart=Instance.new("Part") FCPart.Anchored=true FCPart.CanCollide=false FCPart.Transparency=1
local ConsoleGui=Instance.new("ScreenGui") ConsoleGui.Name="SA_Console" ConsoleGui.Parent=CoreGui
ConsoleGui.ZIndexBehavior=Enum.ZIndexBehavior.Sibling ConsoleGui.ResetOnSpawn=false ConsoleGui.Enabled=false
local CF2=Instance.new("Frame",ConsoleGui)
CF2.Size=UDim2.fromOffset(390,250) CF2.Position=UDim2.fromOffset(12,12)
CF2.BackgroundColor3=Color3.fromRGB(18,18,24) CF2.BackgroundTransparency=0.15
CF2.BorderSizePixel=0 CF2.Active=true CF2.ClipsDescendants=true
Instance.new("UICorner",CF2).CornerRadius=UDim.new(0,10)
local CFStroke=Instance.new("UIStroke",CF2) CFStroke.Color=Color3.fromRGB(60,50,80) CFStroke.Thickness=1
local CTB=Instance.new("Frame",CF2) CTB.Size=UDim2.new(1,0,0,30)
CTB.BackgroundColor3=Color3.fromRGB(28,26,38) CTB.BackgroundTransparency=0.05
CTB.BorderSizePixel=0 CTB.Active=true
Instance.new("UICorner",CTB).CornerRadius=UDim.new(0,10)
local CTBFix=Instance.new("Frame",CTB) CTBFix.Size=UDim2.new(1,0,0.5,0) CTBFix.Position=UDim2.new(0,0,0.5,0)
CTBFix.BackgroundColor3=Color3.fromRGB(28,26,38) CTBFix.BackgroundTransparency=0.05 CTBFix.BorderSizePixel=0
local function MkDot(x,c,h)
  local b=Instance.new("TextButton",CTB) b.Size=UDim2.fromOffset(11,11) b.Position=UDim2.new(0,x,0.5,-5.5)
  b.BackgroundColor3=c b.Text="" b.BorderSizePixel=0 b.AutoButtonColor=false
  Instance.new("UICorner",b).CornerRadius=UDim.new(1,0)
  b.MouseEnter:Connect(function() b.BackgroundColor3=h end)
  b.MouseLeave:Connect(function() b.BackgroundColor3=c end)
  return b
end
local DotX=MkDot(9, Color3.fromRGB(255,95,86),  Color3.fromRGB(200,40,30))
local DotM=MkDot(25,Color3.fromRGB(255,189,46), Color3.fromRGB(200,140,10))
local DotZ=MkDot(41,Color3.fromRGB(39,201,63),  Color3.fromRGB(20,150,35))
local CTBLbl=Instance.new("TextLabel",CTB)
CTBLbl.Size=UDim2.new(1,0,1,0) CTBLbl.BackgroundTransparency=1
CTBLbl.TextColor3=Color3.fromRGB(185,185,205) CTBLbl.Font=Enum.Font.GothamBold
CTBLbl.TextSize=12 CTBLbl.Text="SA Console  v4"
local CSep=Instance.new("Frame",CF2)
CSep.Size=UDim2.new(1,0,0,1) CSep.Position=UDim2.new(0,0,0,30)
CSep.BackgroundColor3=Color3.fromRGB(55,50,70) CSep.BorderSizePixel=0
local CScroll=Instance.new("ScrollingFrame",CF2)
CScroll.Position=UDim2.new(0,0,0,31) CScroll.Size=UDim2.new(1,0,1,-31)
CScroll.BackgroundTransparency=1 CScroll.BorderSizePixel=0
CScroll.ScrollBarThickness=3 CScroll.ScrollBarImageColor3=Color3.fromRGB(90,80,120)
CScroll.CanvasSize=UDim2.new(0,0,0,0) CScroll.AutomaticCanvasSize=Enum.AutomaticSize.Y
Instance.new("UIListLayout",CScroll).SortOrder=Enum.SortOrder.LayoutOrder
local CPad=Instance.new("UIPadding",CScroll)
CPad.PaddingLeft=UDim.new(0,7) CPad.PaddingRight=UDim.new(0,6)
CPad.PaddingTop=UDim.new(0,3) CPad.PaddingBottom=UDim.new(0,3)
local conMin=false
DotX.MouseButton1Click:Connect(function() ConsoleGui.Enabled=false end)
DotM.MouseButton1Click:Connect(function()
  conMin=not conMin CScroll.Visible=not conMin CSep.Visible=not conMin
  CF2.Size=conMin and UDim2.fromOffset(390,30) or UDim2.fromOffset(390,250)
end)
DotZ.MouseButton1Click:Connect(function()
  CF2.Size=CF2.Size.Y.Offset<420 and UDim2.fromOffset(390,420) or UDim2.fromOffset(390,250)
end)
local ConC={
  info=Color3.fromRGB(170,170,200), grab=Color3.fromRGB(75,215,100),
  zombie=Color3.fromRGB(255,95,75), warn=Color3.fromRGB(255,148,28),
  sys=Color3.fromRGB(140,95,225),   item=Color3.fromRGB(80,200,255),
  kill=Color3.fromRGB(255,105,45),  food=Color3.fromRGB(160,230,100),
  farm=Color3.fromRGB(255,200,60),
}
local conLine=0
local function ConLog(msg,kind)
  conLine=conLine+1
  local row=Instance.new("Frame",CScroll)
  row.LayoutOrder=conLine row.Size=UDim2.new(1,0,0,17)
  row.AutomaticSize=Enum.AutomaticSize.Y row.BorderSizePixel=0
  row.BackgroundColor3=conLine%2==0 and Color3.fromRGB(26,24,34) or Color3.fromRGB(18,18,24)
  row.BackgroundTransparency=conLine%2==0 and 0.35 or 1
  local t=Instance.new("TextLabel",row)
  t.Size=UDim2.new(1,0,0,17) t.BackgroundTransparency=1
  t.TextColor3=ConC[kind or "info"] or ConC.info
  t.Font=Enum.Font.Code t.TextSize=12
  t.TextXAlignment=Enum.TextXAlignment.Left
  t.TextWrapped=true t.AutomaticSize=Enum.AutomaticSize.Y t.RichText=true
  local ts=os.date("*t")
  t.Text=string.format('<font color="#50507A">[%02d:%02d:%02d]</font> %s',ts.hour,ts.min,ts.sec,msg)
  task.defer(function() CScroll.CanvasPosition=Vector2.new(0,math.huge) end)
  local n=0 for _,c in pairs(CScroll:GetChildren()) do if c:IsA("Frame") then n=n+1 end end
  if n>200 then for _,c in pairs(CScroll:GetChildren()) do if c:IsA("Frame") then c:Destroy() break end end end
end
local drg,drgS,drgP=false
CTB.InputBegan:Connect(function(i)
  if i.UserInputType==Enum.UserInputType.MouseButton1 then drg=true drgS=i.Position drgP=CF2.Position end
end)
UIS.InputChanged:Connect(function(i)
  if drg and i.UserInputType==Enum.UserInputType.MouseMovement then
    local d=i.Position-drgS local vp=Cam.ViewportSize
    CF2.Position=UDim2.fromOffset(
      math.clamp(drgP.X.Offset+d.X,0,vp.X-390),
      math.clamp(drgP.Y.Offset+d.Y,0,vp.Y-30))
  end
end)
UIS.InputEnded:Connect(function(i)
  if i.UserInputType==Enum.UserInputType.MouseButton1 then drg=false end
end)

-- ================================================================
--  HELPERS
-- ================================================================
local function GetZombies()
  local list={}
  local ok,folder=pcall(function() return workspace.World.Zombies end)
  if ok and folder then
    for _,m in ipairs(folder:GetChildren()) do
      if m:FindFirstChildOfClass("Humanoid") then table.insert(list,m) end
    end
  end
  if #list==0 then
    for _,m in ipairs(CS:GetTagged("Monster")) do table.insert(list,m) end
  end
  return list
end

local function GetItemPart(item)
  if item:IsA("BasePart") then return item end
  if item.PrimaryPart then return item.PrimaryPart end
  for _,c in ipairs(item:GetDescendants()) do
    if c:IsA("BasePart") then return c end
  end
  return nil
end

local function ReadVal(inst,name)
  local child=inst:FindFirstChild(name)
  if child and child:IsA("ValueBase") then return child.Value end
  local ok,v=pcall(function() return inst[name] end)
  if ok and v~=nil then return v end
  return nil
end

-- ---- Remotes ----
local function GetWeaponRemote()
  local ok,r=pcall(function() return RepS.Weapon_System.Events.WeaponEvent.ReplicateBulletHit end)
  return ok and r or nil
end
local function GetGrabRemote()
  local ok,r=pcall(function() return RepS.Game_System.Events.ItemGrab.OnGrabRequest end)
  return ok and r or nil
end
local function GetCarryRemote()
  local ok,r=pcall(function() return RepS.Game_System.Events.ItemCarry.OnCarryRequest end)
  return ok and r or nil
end
local function GetWeapon()
  local char=LP.Character if not char then return nil end
  for _,t in ipairs(char:GetChildren()) do if t:IsA("Tool") then return t end end
  return nil
end

-- ---- Consume: tenta 3 formas diferentes ----

-- ---- Coleta item para inventário (ItemGrab) ----
local function CollectItem(item)
  local r=GetGrabRemote() if not r then return false end
  local ok=pcall(function() r:FireServer(item) end)
  return ok
end

-- ---- Puxa item até o player (ItemCarry x2 = grab + release) ----
local function CarryItem(item)
  local r=GetCarryRemote() if not r then return false end
  local ok=pcall(function() r:FireServer(item) end)
  if ok then task.wait(0.12) pcall(function() r:FireServer(item) end) end
  return ok
end

-- ---- Listas de itens do mundo ----
local function GetDroppedItems()
  local list,names={},{}
  local ok,folder=pcall(function() return workspace.World.DroppedItems end)
  if ok and folder then
    for _,item in ipairs(folder:GetChildren()) do
      local iName=tostring(ReadVal(item,"ItemName") or item.Name)
      local iAmt=ReadVal(item,"Amount")
      table.insert(list,{inst=item,name=iName,amt=iAmt,src="dropped"})
      local label=iName..(iAmt and " x"..tostring(math.floor(tonumber(iAmt) or 0)) or "")
      table.insert(names,label)
    end
  end
  return list,names
end

local function GetGrabbableItems()
  local list,names={},{}
  local ok,folder=pcall(function() return workspace.World.GrabbableItems end)
  if ok and folder then
    for _,item in ipairs(folder:GetChildren()) do
      local iName=tostring(ReadVal(item,"ItemName") or item.Name)
      table.insert(list,{inst=item,name=iName,src="grabbable"})
      table.insert(names,iName)
    end
  end
  return list,names
end

-- ================================================================
--  TAB: COMBAT
-- ================================================================
local auraMode="Perto"
local selectedZombies={}
local mobEspCleanTimer=0  -- cleanup timer para ESP mortos

TC:Section({Title="⚔ Kill Aura"})
local auraModeLabel=TC:Paragraph({Title="Modo: Perto",Desc="Ataca zombies automaticamente"})
TC:Dropdown({Title="Modo de Ataque",Values={"Perto","Todos","Custom"},Default="Perto",Multi=false,
  Callback=function(v) auraMode=v auraModeLabel:SetTitle("Modo: "..v) end
})
local zombieDrop
local function RefreshZombieList()
  local seen,names={},{}
  for _,m in ipairs(GetZombies()) do
    if not seen[m.Name] then seen[m.Name]=true table.insert(names,m.Name) end
  end
  table.sort(names) return names
end
zombieDrop=TC:Dropdown({Title="Zombies (Custom)",Values=RefreshZombieList(),Default={},Multi=true,
  Callback=function(vals)
    selectedZombies={} for _,n in pairs(vals) do selectedZombies[n]=true end
  end
})
TC:Button({Title="Atualizar Lista",Callback=function()
  local n=RefreshZombieList() zombieDrop:Refresh(n,{})
  WindUI:Notify({Title="Lista",Content=#n.." zombies",Duration=3})
end})
TC:Slider({Title="Distância Máx (m)",Step=5,Value={Min=5,Max=300,Default=30},
  Callback=function(v) Cfg.AuraDist=v end})
TC:Slider({Title="Intervalo (ms)",Step=50,Value={Min=50,Max=1000,Default=150},
  Callback=function(v) Cfg.AuraInterval=v/1000 end})

-- Carrega dados da arma (Hammer) no servidor antes de atacar
local function LoadWeaponData(weapon)
  pcall(function()
    RepS.Weapon_System.Events.WeaponData.LoadData:FireServer(weapon)
  end)
end

TC:Toggle({Title="Kill Aura",Desc="Ataca zombies automaticamente",Default=false,
  Callback=function(v)
    Cfg.KillAura=v
    if v then
      task.spawn(function()
        while Cfg.KillAura do
          local char=LP.Character
          local root=char and char:FindFirstChild("HumanoidRootPart")
          local weapon=GetWeapon()
          local remote=GetWeaponRemote()
          if root and weapon and remote then
            -- carrega dados da arma (necessário para o servidor aceitar os hits)
            LoadWeaponData(weapon)
            task.wait(0.05)
            for _,mob in ipairs(GetZombies()) do
              if not Cfg.KillAura then break end
              local torso=mob:FindFirstChild("Torso") or mob:FindFirstChild("UpperTorso") or mob:FindFirstChild("HumanoidRootPart")
              local hum=mob:FindFirstChildOfClass("Humanoid")
              if torso and hum and hum.Health>0 then
                local dist=(root.Position-torso.Position).Magnitude
                local inRange=(auraMode~="Perto") or dist<=Cfg.AuraDist
                local isTarget=(auraMode=="Todos")
                  or (auraMode=="Perto" and inRange)
                  or (auraMode=="Custom" and selectedZombies[mob.Name] and inRange)
                if isTarget then
                  local pos=torso.Position local dir=(pos-root.Position).Unit
                  pcall(function()
                    remote:FireServer(weapon,{Normal=dir,Direction=dir,Instance=torso,Position=pos})
                  end)
                  ConLog("⚔ Hit "..mob.Name.." ["..math.floor(dist).."m]","kill")
                end
              end
            end
          elseif not weapon then ConLog("Equipe uma arma!","warn") end
          task.wait(Cfg.AuraInterval)
        end
        ConLog("Kill Aura OFF","warn")
      end)
    end
  end
})

TC:Button({Title="Matar Todos (1x)",Callback=function()
  local char=LP.Character local root=char and char:FindFirstChild("HumanoidRootPart")
  local weapon=GetWeapon() local remote=GetWeaponRemote()
  if not root or not weapon or not remote then
    WindUI:Notify({Title="Erro",Content="Equipe uma arma!",Duration=4}) return
  end
  LoadWeaponData(weapon) task.wait(0.05)
  local n=0
  for _,mob in ipairs(GetZombies()) do
    local torso=mob:FindFirstChild("Torso") or mob:FindFirstChild("UpperTorso") or mob:FindFirstChild("HumanoidRootPart")
    local hum=mob:FindFirstChildOfClass("Humanoid")
    if torso and hum and hum.Health>0 then
      local pos=torso.Position local dir=(pos-root.Position).Unit
      pcall(function() remote:FireServer(weapon,{Normal=dir,Direction=dir,Instance=torso,Position=pos}) end)
      n=n+1
    end
  end
  WindUI:Notify({Title="Kill All",Content=n.." zombie(s)",Duration=3})
  ConLog("Kill All: "..n,"kill")
end})

-- ---- Seção: Start Game ----
TC:Section({Title="🏕 Start Game"})
TC:Paragraph({Title="Executar apenas 1x",Desc="Coloca a fogueira (Campfire) na posição do player"})
local startGameUsed=false
TC:Button({Title="🔥 Colocar Fogueira (1x)",Callback=function()
  if startGameUsed then
    WindUI:Notify({Title="Start Game",Content="Já executado! Só pode usar 1x.",Duration=4}) return
  end
  local hrp=LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
  if not hrp then
    WindUI:Notify({Title="Erro",Content="Personagem não encontrado",Duration=3}) return
  end
  -- posição do player + pequeno offset para não spawnar dentro do chão
  local pos=hrp.Position+Vector3.new(3,0,0)
  local cf=CFrame.new(pos)
  local campfire=RepS:FindFirstChild("Buildings") and RepS.Buildings:FindFirstChild("Campfire")
  if not campfire then
    WindUI:Notify({Title="Erro",Content="RepS.Buildings.Campfire não encontrado",Duration=4})
    ConLog("Campfire não encontrado em RepS.Buildings","warn") return
  end
  local buildRemote
  local ok2=pcall(function()
    buildRemote=RepS.Game_System.Events.ItemCarry.OnBuildRequest
  end)
  if not ok2 or not buildRemote then
    WindUI:Notify({Title="Erro",Content="OnBuildRequest não encontrado",Duration=4}) return
  end
  pcall(function() buildRemote:FireServer(campfire, cf) end)
  startGameUsed=true
  WindUI:Notify({Title="🔥 Fogueira!",Content="Campfire colocada perto de você",Duration=5})
  ConLog("🔥 Campfire colocada na posição do player","sys")
end})

-- ================================================================
--  TAB: ITEMS
-- ================================================================

-- ---- Seção: Collect (inventário) ----
TI:Section({Title="Collect → Inventário  (ItemGrab.OnGrabRequest)"})
TI:Toggle({Title="Auto Collect Dropped",Desc="Coleta DroppedItems para o inventário",Default=false,
  Callback=function(v)
    Cfg.CollectDropped=v
    if v then
      task.spawn(function()
        while Cfg.CollectDropped do
          local root=LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
          local items=GetDroppedItems()
          if root then
            for _,it in ipairs(items) do
              if not Cfg.CollectDropped then break end
              local part=GetItemPart(it.inst)
              if part then
                local dist=(root.Position-part.Position).Magnitude
                if Cfg.GrabDist==0 or dist<=Cfg.GrabDist then
                  if CollectItem(it.inst) then
                    ConLog("Collected: "..it.name..(it.amt and " x"..tostring(math.floor(tonumber(it.amt) or 0)) or ""),"grab")
                    task.wait(0.08)
                  end
                end
              end
            end
          end
          task.wait(Cfg.GrabInterval)
        end
      end)
    end
  end
})
TI:Button({Title="Collect Todos Dropped Agora",Callback=function()
  local items=GetDroppedItems() local n=0
  for _,it in ipairs(items) do
    if CollectItem(it.inst) then
      n=n+1 ConLog("Collected: "..it.name,"grab") task.wait(0.08)
    end
  end
  WindUI:Notify({Title="Collect",Content=n.." itens",Duration=3})
end})

-- ---- Seção: Carry (puxa até você) ----
TI:Section({Title="Carry → Puxar até Mim  (ItemCarry.OnCarryRequest x2)"})
TI:Toggle({Title="Auto Carry Dropped",Desc="Puxa DroppedItems até o player",Default=false,
  Callback=function(v)
    Cfg.CarryDropped=v
    if v then
      task.spawn(function()
        while Cfg.CarryDropped do
          local root=LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
          local items=GetDroppedItems()
          if root then
            for _,it in ipairs(items) do
              if not Cfg.CarryDropped then break end
              local part=GetItemPart(it.inst)
              if part then
                local dist=(root.Position-part.Position).Magnitude
                if Cfg.GrabDist==0 or dist<=Cfg.GrabDist then
                  if CarryItem(it.inst) then
                    ConLog("Carried: "..it.name,"grab") task.wait(0.12)
                  end
                end
              end
            end
          end
          task.wait(Cfg.GrabInterval)
        end
      end)
    end
  end
})
TI:Toggle({Title="Auto Carry Grabbable",Desc="Puxa GrabbableItems até o player",Default=false,
  Callback=function(v)
    Cfg.CarryGrabbable=v
    if v then
      task.spawn(function()
        while Cfg.CarryGrabbable do
          local root=LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
          local items=GetGrabbableItems()
          if root then
            for _,it in ipairs(items) do
              if not Cfg.CarryGrabbable then break end
              local part=GetItemPart(it.inst)
              if part then
                local dist=(root.Position-part.Position).Magnitude
                if Cfg.GrabDist==0 or dist<=Cfg.GrabDist then
                  if CarryItem(part) then
                    ConLog("Carried Grabbable: "..it.name,"grab") task.wait(0.12)
                  end
                end
              end
            end
          end
          task.wait(Cfg.GrabInterval)
        end
      end)
    end
  end
})
TI:Button({Title="Carry Todos Dropped Agora",Callback=function()
  local items=GetDroppedItems() local n=0
  for _,it in ipairs(items) do
    if CarryItem(it.inst) then n=n+1 ConLog("Carried: "..it.name,"grab") task.wait(0.12) end
  end
  WindUI:Notify({Title="Carry",Content=n.." itens puxados",Duration=3})
end})
TI:Button({Title="Carry Todos Grabbable Agora",Callback=function()
  local items=GetGrabbableItems() local n=0
  for _,it in ipairs(items) do
    local part=GetItemPart(it.inst)
    if part and CarryItem(part) then n=n+1 ConLog("Carried Grabbable: "..it.name,"grab") task.wait(0.12) end
  end
  WindUI:Notify({Title="Carry",Content=n.." itens",Duration=3})
end})

-- ---- Seção: Lista Seletiva de Itens ----
TI:Section({Title="Coletar Item Específico"})
local selItemList={} local selItemDrop
local function BuildItemList()
  local all,names={},{}
  local di=GetDroppedItems()
  local gi=GetGrabbableItems()
  for _,it in ipairs(di) do table.insert(all,it) table.insert(names,"[D] "..it.name) end
  for _,it in ipairs(gi) do table.insert(all,it) table.insert(names,"[G] "..it.name) end
  selItemList=all return names
end
selItemDrop=TI:Dropdown({Title="Itens disponíveis",Values=BuildItemList(),Default={},Multi=true,
  Callback=function(v) end  -- só para seleção, ação está no botão
})
TI:Button({Title="Atualizar Lista de Itens",Callback=function()
  local n=BuildItemList() selItemDrop:Refresh(n,{})
  WindUI:Notify({Title="Itens",Content=#n.." itens encontrados",Duration=3})
end})
TI:Button({Title="Collect Itens Selecionados",Callback=function()
  -- re-busca o estado atual do dropdown (trabalha com todos pois não tem getter)
  -- estratégia: coleta todos que estiverem na lista atual
  local n=0
  for _,it in ipairs(selItemList) do
    if CollectItem(it.inst) then n=n+1 ConLog("Sel-Collect: "..it.name,"grab") task.wait(0.08) end
  end
  WindUI:Notify({Title="Sel Collect",Content=n.." itens coletados",Duration=3})
end})

-- ================================================================
--  SISTEMA DE COMIDA/BEBIDA
--  Baseado no script real do jogo (Consumable LocalScript):
--    1. Coleta o item do DroppedItems pro Backpack via ItemGrab
--    2. Equipa a tool (hum:EquipTool)
--    3. Chama tool:Activate() — inicia o hold
--    4. O jogo aguarda ~5s (v_u_18 >= 5) então chama:
--         HealthScript.Consume:FireServer("Hunger"/"Thirst", 20, 40, tool)
--       O parâmetro [4] é o PRÓPRIO ITEM (tool), não RepS.Food/Water
--    5. Após consumo o item vai pro RepS via Debris, então desequipamos se ainda aqui
-- ================================================================

local invFood  = 0
local invWater = 0
local foodLbl, waterLbl

-- Emojis para tipos de item no ESP
local ITEM_EMOJI = {
  food  = "🍖", water = "💧",
  stone = "🪨", metal = "⚙️",
  plank = "🪵", rope  = "🪢",
}
local function ItemEmoji(name)
  local n=name:lower()
  for k,e in pairs(ITEM_EMOJI) do if n:find(k) then return e.." " end end
  return ""
end

local function SyncInvCounter()
  local bp=LP:FindFirstChild("Backpack")
  local f,w=0,0
  if bp then
    for _,t in ipairs(bp:GetChildren()) do
      if t:IsA("Tool") then
        local n=t.Name:lower()
        if n:find("food") then f=f+1 elseif n:find("water") then w=w+1 end
      end
    end
  end
  local char=LP.Character
  if char then
    for _,t in ipairs(char:GetChildren()) do
      if t:IsA("Tool") then
        local n=t.Name:lower()
        if n:find("food") then f=f+1 elseif n:find("water") then w=w+1 end
      end
    end
  end
  invFood=f invWater=w
  if foodLbl  then pcall(function() foodLbl:SetTitle( "🍖 Food  no inventário: "..invFood)  end) end
  if waterLbl then pcall(function() waterLbl:SetTitle("💧 Water no inventário: "..invWater) end) end
end

-- Auto-sincroniza contador quando o Backpack muda
task.spawn(function()
  local bp=LP:WaitForChild("Backpack")
  bp.ChildAdded:Connect(function() task.wait(0.1) SyncInvCounter() end)
  bp.ChildRemoved:Connect(function() task.wait(0.1) SyncInvCounter() end)
  -- também quando personagem muda
  LP.CharacterAdded:Connect(function(char)
    task.wait(1)
    SyncInvCounter()
    char.ChildAdded:Connect(function(c) if c:IsA("Tool") then task.wait(0.1) SyncInvCounter() end end)
    char.ChildRemoved:Connect(function(c) if c:IsA("Tool") then task.wait(0.1) SyncInvCounter() end end)
  end)
end)

-- Equipa a tool e usa segurando por ~5.2s (tempo real do jogo = 5s de hold)
-- O próprio script do item faz FireServer quando chega em 5s
-- Nós apenas garantimos que a tool fique equipada durante esse tempo
local function EquipAndUse(toolName)
  local char=LP.Character
  local bp=LP:FindFirstChild("Backpack")
  if not char or not bp then return false end

  local tool=nil
  for _,t in ipairs(bp:GetChildren()) do
    if t:IsA("Tool") and t.Name:lower():find(toolName:lower()) then tool=t break end
  end
  if not tool then
    ConLog("🔍 '"..toolName.."' não no Backpack","warn") return false
  end

  local hum=char:FindFirstChildOfClass("Humanoid")
  if not hum then return false end

  -- Equipa
  pcall(function() hum:EquipTool(tool) end)
  task.wait(0.3)

  -- Ativa (dispara o coroutine do Consumable que aguarda 5s internamente)
  pcall(function() tool:Activate() end)

  -- Aguarda até 12s: sai cedo se o item sumiu do char (foi consumido pelo jogo)
  -- O jogo move o item para RepS via Debris após 5s de hold
  local t0=tick()
  while tick()-t0 < 12 do
    if not tool or not tool.Parent or tool.Parent~=char then break end
    task.wait(0.1)
  end

  -- Se ainda no char, desequipa
  if tool and tool.Parent and tool.Parent==char then
    pcall(function() tool.Parent=bp end)
  end
  task.wait(0.1)
  return true
end

-- Coleta todos itens de um keyword do DroppedItems
local function CollectConsumable(keyword)
  local ok2,folder=pcall(function() return workspace.World.DroppedItems end)
  if not ok2 or not folder then return 0 end
  local n=0
  for _,item in ipairs(folder:GetChildren()) do
    local iName=tostring(ReadVal(item,"ItemName") or ""):lower()
    if iName:find(keyword) then
      if CollectItem(item) then n=n+1 task.wait(0.12) end
    end
  end
  return n
end

-- ---- Seção: Auto Comer & Beber ----
TI:Section({Title="🍽 Survival — Auto Comer & Beber"})
TI:Paragraph({Title="Como funciona",Desc="Coleta item → equipa → Activate → aguarda 5s (hold real do jogo)"})

foodLbl  = TI:Paragraph({Title="🍖 Food  no inventário: 0", Desc="Atualizado automaticamente"})
waterLbl = TI:Paragraph({Title="💧 Water no inventário: 0",Desc="Atualizado automaticamente"})

TI:Toggle({Title="🍖 Auto Comer (Food)",Desc="Coleta Food e usa automaticamente",Default=false,
  Callback=function(v)
    Cfg.AutoEat=v
    if v then
      task.spawn(function()
        while Cfg.AutoEat do
          local n=CollectConsumable("food")
          if n>0 then ConLog("🍖 Coletou "..n.." Food","grab") task.wait(0.3) end
          SyncInvCounter()
          if invFood>0 then
            ConLog("🍖 Comendo Food...","food")
            EquipAndUse("food")
            SyncInvCounter()
            ConLog("🍖 Comeu! Restam: "..invFood,"food")
          else
            ConLog("🍖 Sem Food no inventário","warn")
          end
          task.wait(Cfg.EatInterval)
        end
      end)
    end
  end
})

TI:Toggle({Title="💧 Auto Beber (Water)",Desc="Coleta Water e usa automaticamente",Default=false,
  Callback=function(v)
    Cfg.AutoDrink=v
    if v then
      task.spawn(function()
        while Cfg.AutoDrink do
          local n=CollectConsumable("water")
          if n>0 then ConLog("💧 Coletou "..n.." Water","grab") task.wait(0.3) end
          SyncInvCounter()
          if invWater>0 then
            ConLog("💧 Bebendo Water...","food")
            EquipAndUse("water")
            SyncInvCounter()
            ConLog("💧 Bebeu! Restam: "..invWater,"food")
          else
            ConLog("💧 Sem Water no inventário","warn")
          end
          task.wait(Cfg.DrinkInterval)
        end
      end)
    end
  end
})

TI:Dropdown({Title="⏱ Intervalo Comer",Values={"1min","2min","5min"},Default="1min",Multi=false,
  Callback=function(v)
    if v=="1min" then Cfg.EatInterval=60 elseif v=="2min" then Cfg.EatInterval=120 else Cfg.EatInterval=300 end
  end
})
TI:Dropdown({Title="⏱ Intervalo Beber",Values={"1min","2min","5min"},Default="1min",Multi=false,
  Callback=function(v)
    if v=="1min" then Cfg.DrinkInterval=60 elseif v=="2min" then Cfg.DrinkInterval=120 else Cfg.DrinkInterval=300 end
  end
})

TI:Button({Title="🍖 Comer Agora (1x)",Callback=function()
  task.spawn(function()
    if invFood<=0 then
      local n=CollectConsumable("food")
      if n>0 then ConLog("🍖 Coletou "..n.." Food","grab") task.wait(0.3) end
      SyncInvCounter()
    end
    if invFood<=0 then
      WindUI:Notify({Title="Erro",Content="Sem Food no inventário ou no mundo",Duration=3}) return
    end
    ConLog("🍖 Comendo Food...","food")
    EquipAndUse("food")
    SyncInvCounter()
    WindUI:Notify({Title="✅ Comeu",Content="Restam: "..invFood.." Food",Duration=3})
    ConLog("🍖 Manual Eat OK — Restam: "..invFood,"food")
  end)
end})

TI:Button({Title="💧 Beber Agora (1x)",Callback=function()
  task.spawn(function()
    if invWater<=0 then
      local n=CollectConsumable("water")
      if n>0 then ConLog("💧 Coletou "..n.." Water","grab") task.wait(0.3) end
      SyncInvCounter()
    end
    if invWater<=0 then
      WindUI:Notify({Title="Erro",Content="Sem Water no inventário ou no mundo",Duration=3}) return
    end
    ConLog("💧 Bebendo Water...","food")
    EquipAndUse("water")
    SyncInvCounter()
    WindUI:Notify({Title="✅ Bebeu",Content="Restam: "..invWater.." Water",Duration=3})
    ConLog("💧 Manual Drink OK — Restam: "..invWater,"food")
  end)
end})

TI:Button({Title="🔄 Sync Contador Agora",Callback=function()
  SyncInvCounter()
  WindUI:Notify({Title="Sync",Content="Food: "..invFood.."  Water: "..invWater,Duration=3})
end})

-- ---- Seção: Farm Day ----
TI:Section({Title="Farm Day — Sobrevivência Automática"})
TI:Paragraph({Title="O que faz",Desc="Coleta Food & Water → Come & Bebe → Mata Zombies → repete"})
TI:Toggle({Title="Farm Day Mode",Desc="Loop completo de sobrevivência + combat",Default=false,
  Callback=function(v)
    Cfg.FarmDay=v
    if v then
      task.spawn(function()
        ConLog("=== FARM DAY INICIADO ===","farm")
        while Cfg.FarmDay do
          -- 1) Coleta Food & Water
          local nf=CollectConsumable("food")
          local nw=CollectConsumable("water")
          if nf>0 then invFood=invFood+nf foodLbl:SetTitle("🍖 Food  no inventário: "..invFood) end
          if nw>0 then invWater=invWater+nw waterLbl:SetTitle("💧 Water no inventário: "..invWater) end
          if (nf+nw)>0 then ConLog("[FarmDay] Coletou "..nf.." Food + "..nw.." Water","farm") task.wait(0.3) end

          -- 2) Come (se tiver Food)
          if invFood>0 then
            if EquipAndUse("food") then
              invFood=math.max(0,invFood-1)
              foodLbl:SetTitle("🍖 Food  no inventário: "..invFood)
              ConLog("[FarmDay] Comeu Food","farm")
            end
            task.wait(0.3)
          end

          -- 3) Bebe (se tiver Water)
          if invWater>0 then
            if EquipAndUse("water") then
              invWater=math.max(0,invWater-1)
              waterLbl:SetTitle("💧 Water no inventário: "..invWater)
              ConLog("[FarmDay] Bebeu Water","farm")
            end
            task.wait(0.3)
          end

          -- 4) Mata zombies (usa arma equipada atualmente)
          local char=LP.Character
          local root=char and char:FindFirstChild("HumanoidRootPart")
          local weapon=GetWeapon()
          local remote=GetWeaponRemote()
          if root and weapon and remote then
            LoadWeaponData(weapon) task.wait(0.05)
            local killed=0
            for _,mob in ipairs(GetZombies()) do
              if not Cfg.FarmDay then break end
              local torso=mob:FindFirstChild("Torso") or mob:FindFirstChild("UpperTorso") or mob:FindFirstChild("HumanoidRootPart")
              local hum=mob:FindFirstChildOfClass("Humanoid")
              if torso and hum and hum.Health>0 then
                local pos=torso.Position local dir=(pos-root.Position).Unit
                pcall(function()
                  remote:FireServer(weapon,{Normal=dir,Direction=dir,Instance=torso,Position=pos})
                end)
                killed=killed+1
              end
            end
            if killed>0 then ConLog("⚔ [FarmDay] Matou "..killed.." zombie(s)","farm") end
          else
            ConLog("⚔ [FarmDay] Sem arma — equipe uma arma","warn")
          end

          task.wait(5)
        end
        ConLog("=== FARM DAY PARADO ===","farm")
      end)
    end
  end
})

-- ---- Configurações ----
TI:Section({Title="Configurações de Grab"})
TI:Slider({Title="Distância Máx (0=Infinita)",Step=5,Value={Min=0,Max=250,Default=0},
  Callback=function(v) Cfg.GrabDist=v end})
TI:Slider({Title="Intervalo Loop (ms)",Step=50,Value={Min=50,Max=2000,Default=500},
  Callback=function(v) Cfg.GrabInterval=v/1000 end})

-- ================================================================
--  TAB: MOVE
-- ================================================================
LP.CharacterAdded:Connect(function(char)
  local hum=char:WaitForChild("Humanoid") task.wait(0.5)
  hum.WalkSpeed=Cfg.WalkSpeed hum.JumpPower=Cfg.JumpPower hum.JumpHeight=Cfg.JumpHeight
end)
local flyUD=0
UIS.InputBegan:Connect(function(i,gp)
  if gp then return end
  if Cfg.Fly then
    if i.KeyCode==Enum.KeyCode.Space then flyUD=1
    elseif i.KeyCode==Enum.KeyCode.LeftShift then flyUD=-1 end
  end
end)
UIS.InputEnded:Connect(function(i)
  if i.KeyCode==Enum.KeyCode.Space or i.KeyCode==Enum.KeyCode.LeftShift then flyUD=0 end
end)
UIS.JumpRequest:Connect(function()
  if Cfg.InfJ and LP.Character and LP.Character:FindFirstChild("Humanoid") then
    LP.Character.Humanoid:ChangeState(Enum.HumanoidStateType.Jumping)
  end
end)
LP.Idled:Connect(function()
  if Cfg.AFK then VU:Button2Down(Vector2.zero,Cam.CFrame) task.wait(1) VU:Button2Up(Vector2.zero,Cam.CFrame) end
end)

TM:Section({Title="Speed & Jump"})
TM:Slider({Title="WalkSpeed",Step=1,Value={Min=16,Max=500,Default=16},Callback=function(v)
  Cfg.WalkSpeed=v
  if LP.Character and LP.Character:FindFirstChild("Humanoid") then LP.Character.Humanoid.WalkSpeed=v end
end})
TM:Slider({Title="JumpPower",Step=1,Value={Min=7,Max=500,Default=50},Callback=function(v)
  Cfg.JumpPower=v
  if LP.Character and LP.Character:FindFirstChild("Humanoid") then LP.Character.Humanoid.JumpPower=v end
end})
TM:Toggle({Title="Infinite Jump",Default=false,Callback=function(v) Cfg.InfJ=v end})
TM:Button({Title="Reset Speed & Jump",Callback=function()
  Cfg.WalkSpeed=16 Cfg.JumpPower=50
  if LP.Character and LP.Character:FindFirstChild("Humanoid") then
    LP.Character.Humanoid.WalkSpeed=16 LP.Character.Humanoid.JumpPower=50
  end
end})

TM:Section({Title="Flight"})
-- Fly e Freecam são mutuamente exclusivos para evitar bug de câmera andando sozinha
TM:Toggle({Title="Camera Fly",Desc="Space=subir  Shift=descer  (desliga Freecam)",Default=false,
  Callback=function(v)
    Cfg.Fly=v
    if v then
      -- desliga freecam se estiver ativo
      if Cfg.Freecam then
        Cfg.Freecam=false
        FCPart.Parent=nil
        local c=LP.Character
        if c then
          local h=c:FindFirstChild("HumanoidRootPart") local hm=c:FindFirstChild("Humanoid")
          if h then h.Anchored=false end
          if hm then Cam.CameraSubject=hm end
        end
      end
    else
      local c=LP.Character
      if c then
        local h=c:FindFirstChild("HumanoidRootPart") local hm=c:FindFirstChild("Humanoid")
        if h then
          if h:FindFirstChild("FlyBody") then h.FlyBody:Destroy() end
          if h:FindFirstChild("FlyGyro") then h.FlyGyro:Destroy() end
        end
        if hm then hm.PlatformStand=false end
      end
    end
  end
})
TM:Slider({Title="Fly Speed",Step=5,Value={Min=10,Max=500,Default=50},Callback=function(v) Cfg.FlySpeed=v end})

TM:Section({Title="Camera"})
TM:Toggle({Title="Freecam",Desc="Desacopla câmera do personagem  (desliga Fly)",Default=false,
  Callback=function(v)
    Cfg.Freecam=v
    if v then
      -- desliga fly se estiver ativo
      if Cfg.Fly then
        Cfg.Fly=false
        local c=LP.Character
        if c then
          local h=c:FindFirstChild("HumanoidRootPart") local hm=c:FindFirstChild("Humanoid")
          if h then
            if h:FindFirstChild("FlyBody") then h.FlyBody:Destroy() end
            if h:FindFirstChild("FlyGyro") then h.FlyGyro:Destroy() end
          end
          if hm then hm.PlatformStand=false end
        end
      end
      if LP.Character and LP.Character:FindFirstChild("HumanoidRootPart") then
        LP.Character.HumanoidRootPart.Anchored=true
      end
      FCPart.Position=Cam.CFrame.Position FCPart.Parent=workspace
      Cam.CameraSubject=FCPart
    else
      FCPart.Parent=nil
      local c=LP.Character
      if c then
        local h=c:FindFirstChild("HumanoidRootPart") local hm=c:FindFirstChild("Humanoid")
        if h then h.Anchored=false end
        if hm then Cam.CameraSubject=hm end
      end
    end
  end
})
TM:Slider({Title="Freecam Speed",Step=1,Value={Min=1,Max=50,Default=2},Callback=function(v) Cfg.FCSpeed=v end})
TM:Slider({Title="FOV",Step=1,Value={Min=40,Max=120,Default=70},Callback=function(v) Cam.FieldOfView=v end})

TM:Section({Title="Teleporte"})
TM:Button({Title="Teleportar para Spawn",Callback=function()
  if LP.Character and LP.Character:FindFirstChild("HumanoidRootPart") then
    local sp=workspace:FindFirstChild("SpawnLocation")
    if sp then LP.Character.HumanoidRootPart.CFrame=sp.CFrame+Vector3.new(0,5,0)
    else LP.Character.HumanoidRootPart.CFrame=CFrame.new(0,10,0) end
  end
end})

-- Teleporte para player
local tpPlayerDrop
local function GetPlayerNames()
  local names={}
  for _,p in ipairs(Plrs:GetPlayers()) do
    if p~=LP then table.insert(names,p.Name) end
  end
  return names
end
local tpTargetPlayer=nil
tpPlayerDrop=TM:Dropdown({Title="Teleportar para Player",Values=GetPlayerNames(),Default={},Multi=false,
  Callback=function(v) tpTargetPlayer=v end
})
TM:Button({Title="Ir para Player",Callback=function()
  if not tpTargetPlayer or tpTargetPlayer=="" then
    WindUI:Notify({Title="Erro",Content="Selecione um player",Duration=3}) return
  end
  local target=Plrs:FindFirstChild(tpTargetPlayer)
  if not target or not target.Character or not target.Character:FindFirstChild("HumanoidRootPart") then
    WindUI:Notify({Title="Erro",Content="Player não encontrado ou sem corpo",Duration=3}) return
  end
  if LP.Character and LP.Character:FindFirstChild("HumanoidRootPart") then
    LP.Character.HumanoidRootPart.CFrame=target.Character.HumanoidRootPart.CFrame+Vector3.new(2,0,0)
    WindUI:Notify({Title="TP",Content="Teleportado para "..tpTargetPlayer,Duration=3})
    ConLog("TP → "..tpTargetPlayer,"info")
  end
end})
TM:Button({Title="Atualizar Lista de Players",Callback=function()
  local n=GetPlayerNames() tpPlayerDrop:Refresh(n,{})
end})

TM:Section({Title="Misc"})
TM:Toggle({Title="Noclip",Default=false,Callback=function(v) Cfg.Noclip=v end})
TM:Toggle({Title="Anti AFK",Default=false,Callback=function(v) Cfg.AFK=v end})
TM:Button({Title="Force Respawn",Callback=function() if LP.Character then LP.Character:BreakJoints() end end})

-- ================================================================
--  TAB: VISUALS
-- ================================================================
local SK_R15={
  -- coluna
  {"Head","UpperTorso"},{"UpperTorso","LowerTorso"},
  -- braço esquerdo (3 segmentos: ombro→cotovelo, cotovelo→antebraço, antebraço→mão)
  {"UpperTorso","LeftUpperArm"},{"LeftUpperArm","LeftLowerArm"},{"LeftLowerArm","LeftHand"},
  -- braço direito
  {"UpperTorso","RightUpperArm"},{"RightUpperArm","RightLowerArm"},{"RightLowerArm","RightHand"},
  -- perna esquerda (3 segmentos: quadril→joelho, joelho→canela, canela→pé)
  {"LowerTorso","LeftUpperLeg"},{"LeftUpperLeg","LeftLowerLeg"},{"LeftLowerLeg","LeftFoot"},
  -- perna direita
  {"LowerTorso","RightUpperLeg"},{"RightUpperLeg","RightLowerLeg"},{"RightLowerLeg","RightFoot"},
}
local SK_R6={
  {"Head","Torso"},{"Torso","HumanoidRootPart"},
  -- braço esq: 2 segmentos simulados usando posição média
  {"Torso","Left Arm"},{"Left Arm","Left Arm"},   -- placeholder duplicado, resolvido no render
  -- braço dir
  {"Torso","Right Arm"},{"Right Arm","Right Arm"},
  -- perna esq: 2 segmentos
  {"Torso","Left Leg"},{"Left Leg","Left Leg"},
  -- perna dir
  {"Torso","Right Leg"},{"Right Leg","Right Leg"},
}
local SK_MAX=math.max(#SK_R15,#SK_R6)

local EspObjs={}
local function InitEsp(p)
  local lines={}
  for i=1,SK_MAX do
    local l=Drawing.new("Line") l.Thickness=1.5 l.Visible=false
    table.insert(lines,l)
  end
  local e={
    N=Drawing.new("Text"), B=Drawing.new("Square"), L=Drawing.new("Line"),
    D=Drawing.new("Text"), H=Drawing.new("Text"),
    C=Color3.fromHSV(math.random(),1,1), SK=lines,
  }
  e.N.Size=15 e.N.Center=true e.N.Outline=true
  e.B.Thickness=1.2 e.B.Filled=false e.L.Thickness=1.2
  e.D.Size=12 e.D.Center=true e.D.Outline=true e.D.Color=Color3.new(1,1,0)
  e.H.Size=12 e.H.Center=true e.H.Outline=true e.H.Color=Color3.fromRGB(255,100,100)
  EspObjs[p]=e
end
local function ClearEsp(esp)
  esp.N.Visible=false esp.B.Visible=false esp.L.Visible=false
  esp.D.Visible=false esp.H.Visible=false
  for _,l in ipairs(esp.SK) do l.Visible=false end
end
for _,p in ipairs(Plrs:GetPlayers()) do if p~=LP then InitEsp(p) end end
Plrs.PlayerAdded:Connect(InitEsp)
Plrs.PlayerRemoving:Connect(function(p)
  if EspObjs[p] then
    for k,v in pairs(EspObjs[p]) do
      if k=="SK" then for _,l in ipairs(v) do l:Remove() end
      elseif k~="C" then v:Remove() end
    end
    EspObjs[p]=nil
  end
end)

-- Zombie ESP — limpeza robusta: rastreia mobs conhecidos
local MobEspObjs={}  -- mob instance → {N,H,MD}
local function CleanDeadMobEsp()
  for mob,e in pairs(MobEspObjs) do
    local hum=mob:FindFirstChildOfClass("Humanoid")
    local dead=(not mob.Parent) or (not hum) or (hum.Health<=0)
    if dead then
      pcall(function() e.N:Remove() e.H:Remove() e.MD:Remove() end)
      MobEspObjs[mob]=nil
    end
  end
end

local ItemEspObjs={}

TV:Section({Title="Player ESP"})
TV:Toggle({Title="ESP Names",   Default=false,Callback=function(v) Cfg.EspN=v end})
TV:Toggle({Title="ESP Boxes",   Default=false,Callback=function(v) Cfg.EspB=v end})
TV:Toggle({Title="ESP Lines",   Default=false,Callback=function(v) Cfg.EspL=v end})
TV:Toggle({Title="ESP Distance",Default=false,Callback=function(v) Cfg.EspDist=v end})
TV:Toggle({Title="Player Health",Default=false,Callback=function(v) Cfg.PlayerHealth=v end})
TV:Toggle({Title="Unique Colors",Default=false,Callback=function(v) Cfg.RndC=v end})
TV:Toggle({Title="Skeleton ESP",Desc="Bonequinho R6 e R15",Default=false,Callback=function(v)
  Cfg.EspSkeleton=v
  if not v then for _,esp in pairs(EspObjs) do for _,l in ipairs(esp.SK) do l.Visible=false end end end
end})

TV:Section({Title="Zombies  (World.Zombies)"})
TV:Toggle({Title="Zombie ESP",     Default=false,Callback=function(v) Cfg.MobEsp=v
  if not v then for _,e in pairs(MobEspObjs) do e.N.Visible=false e.H.Visible=false e.MD.Visible=false end end
end})
TV:Toggle({Title="Zombie Health",  Default=false,Callback=function(v) Cfg.MobHealth=v
  if not v then for _,e in pairs(MobEspObjs) do e.H.Visible=false end end
end})
TV:Toggle({Title="Zombie Distance",Default=false,Callback=function(v) Cfg.MobDist=v
  if not v then for _,e in pairs(MobEspObjs) do e.MD.Visible=false end end
end})
TV:Button({Title="Limpar ESP Mortos",Callback=function()
  CleanDeadMobEsp()
  WindUI:Notify({Title="ESP",Content="ESP de mortos limpo",Duration=3})
end})

TV:Section({Title="Items ESP"})
TV:Toggle({Title="Dropped Items",Desc="World.DroppedItems — nome, qtd, dist",Default=false,
  Callback=function(v)
    Cfg.ItemEspDropped=v
    if not v then for _,e in pairs(ItemEspObjs) do if e.src=="dropped" then e.N.Visible=false e.A.Visible=false e.D2.Visible=false end end end
  end
})
TV:Toggle({Title="Grabbable Items",Desc="World.GrabbableItems — nome, qtd, dist",Default=false,
  Callback=function(v)
    Cfg.ItemEspGrabbable=v
    if not v then for _,e in pairs(ItemEspObjs) do if e.src=="grabbable" then e.N.Visible=false e.A.Visible=false e.D2.Visible=false end end end
  end
})

TV:Section({Title="HUD"})
TV:Toggle({Title="Crosshair",Default=false,Callback=function(v) Cfg.Crosshair=v CrossGui.Enabled=v end})
TV:Toggle({Title="On-Screen Stats",Default=false,Callback=function(v) Cfg.StatsUI=v StatsGui.Enabled=v end})
TV:Toggle({Title="Fullbright",Default=false,Callback=function(v)
  Light.Brightness=v and 2 or 1
  Light.Ambient=v and Color3.new(1,1,1) or Color3.new(0,0,0)
  Light.OutdoorAmbient=v and Color3.new(1,1,1) or Color3.fromRGB(127,127,127)
end})
TV:Toggle({Title="No Fog",Default=false,Callback=function(v) Light.FogEnd=v and 1e6 or 1000 end})

-- ================================================================
--  TAB: OPTIM
-- ================================================================
local origPD={} local hidFX={} local hidMesh={} local hidGui={} local origL={} local origT={} local disScr={}
local function LOD_L(e) if e then origL={shadow=Light.GlobalShadows,tech=Light.Technology,bright=Light.Brightness,amb=Light.Ambient,fog=Light.FogEnd,oamb=Light.OutdoorAmbient} Light.GlobalShadows=false Light.Technology=Enum.Technology.Compatibility Light.Brightness=1 Light.Ambient=Color3.new(0.5,0.5,0.5) Light.OutdoorAmbient=Color3.new(0.5,0.5,0.5) Light.FogEnd=1e6 for _,fx in pairs(Light:GetChildren()) do if fx:IsA("PostEffect") then fx.Enabled=false end end else if origL.shadow~=nil then Light.GlobalShadows=origL.shadow end if origL.tech then Light.Technology=origL.tech end if origL.bright then Light.Brightness=origL.bright end if origL.amb then Light.Ambient=origL.amb end if origL.oamb then Light.OutdoorAmbient=origL.oamb end if origL.fog then Light.FogEnd=origL.fog end for _,fx in pairs(Light:GetChildren()) do if fx:IsA("PostEffect") then fx.Enabled=true end end end end
local function LOD_T(e) if e then origT={w=workspace.Terrain.WaterWaveSize,d=workspace.Terrain.Decoration,ws=workspace.Terrain.WaterWaveSpeed} workspace.Terrain.WaterWaveSize=0 workspace.Terrain.WaterWaveSpeed=0 workspace.Terrain.Decoration=false else if origT.w then workspace.Terrain.WaterWaveSize=origT.w end if origT.ws then workspace.Terrain.WaterWaveSpeed=origT.ws end workspace.Terrain.Decoration=origT.d~=nil and origT.d or true end end
local function LOD_P(e) for _,p in pairs(workspace:GetDescendants()) do if p:IsA("BasePart") and not (LP.Character and p:IsDescendantOf(LP.Character)) then if e then if not origPD[p] then origPD[p]={c=p.CastShadow} end p.CastShadow=false else if origPD[p] then p.CastShadow=origPD[p].c origPD[p]=nil end end end end end
local function LOD_FX(e) if e then for _,fx in pairs(workspace:GetDescendants()) do if fx:IsA("ParticleEmitter") or fx:IsA("Beam") or fx:IsA("Trail") or fx:IsA("Fire") or fx:IsA("Smoke") or fx:IsA("Sparkles") then if fx.Enabled~=false then hidFX[fx]=true fx.Enabled=false end end end else for fx in pairs(hidFX) do pcall(function() fx.Enabled=true end) end hidFX={} end end
local function LOD_M(e) if e then for _,o in pairs(workspace:GetDescendants()) do if (o:IsA("SpecialMesh") or o:IsA("SurfaceAppearance") or o:IsA("Texture") or o:IsA("Decal")) and o.Parent and not hidMesh[o] then hidMesh[o]=o.Parent o.Parent=nil end end else for o,par in pairs(hidMesh) do pcall(function() o.Parent=par end) end hidMesh={} end end
local function LOD_G(e) if e then for _,g in pairs(LP.PlayerGui:GetChildren()) do if g:IsA("ScreenGui") and g.Enabled and g.Name~="SA_Stats" and g.Name~="SA_Cross" and g.Name~="SA_Console" then hidGui[g]=true g.Enabled=false end end else for g in pairs(hidGui) do pcall(function() g.Enabled=true end) end hidGui={} end end
local function LOD_S(e) if e then for _,s in pairs(LP.PlayerGui:GetDescendants()) do if s:IsA("LocalScript") and not s.Disabled then disScr[s]=true s.Disabled=true end end else for s in pairs(disScr) do pcall(function() s.Disabled=false end) end disScr={} end end
local function LOD_Sky(e) for _,s in pairs(Light:GetChildren()) do if s:IsA("Sky") then s.Parent=e and nil or Light end end end
local function LOD_Atm(e) for _,a in pairs(Light:GetChildren()) do if a:IsA("Atmosphere") then a.Density=e and 0 or 0.395 a.Haze=e and 0 or 0 end end end
TP:Section({Title="Performance"})
TP:Toggle({Title="No Shadows & PostFX",Default=false,Callback=function(v) LOD_L(v) end})
TP:Toggle({Title="Simplify Terrain",Default=false,Callback=function(v) LOD_T(v) end})
TP:Toggle({Title="No Part Shadows",Default=false,Callback=function(v) LOD_P(v) end})
TP:Toggle({Title="Remove Particles/VFX",Default=false,Callback=function(v) LOD_FX(v) end})
TP:Toggle({Title="Low Poly Mode",Default=false,Callback=function(v) LOD_M(v) end})
TP:Toggle({Title="Hide Game GUIs",Default=false,Callback=function(v) LOD_G(v) end})
TP:Toggle({Title="Disable GUI Scripts",Default=false,Callback=function(v) LOD_S(v) end})
TP:Toggle({Title="Remove Skybox",Default=false,Callback=function(v) LOD_Sky(v) end})
TP:Toggle({Title="Clear Atmosphere",Default=false,Callback=function(v) LOD_Atm(v) end})
TP:Toggle({Title="Flat Material",Default=false,Callback=function(v) for _,p in pairs(workspace:GetDescendants()) do if p:IsA("BasePart") and not (LP.Character and p:IsDescendantOf(LP.Character)) then if v then origPD[p]=origPD[p] or {} origPD[p].mat=origPD[p].mat or p.Material p.Material=Enum.Material.SmoothPlastic else if origPD[p] and origPD[p].mat then p.Material=origPD[p].mat end end end end end})
TP:Slider({Title="Render Distance",Step=100,Value={Min=50,Max=3000,Default=1000},Callback=function(v) Cam.MaxAxisFieldOfView=v end})
TP:Section({Title="Presets"})
TP:Button({Title="MAX Performance",Callback=function() LOD_L(true) LOD_T(true) LOD_P(true) LOD_FX(true) LOD_Sky(true) LOD_Atm(true) WindUI:Notify({Title="Optim",Content="Max performance!",Duration=4}) end})
TP:Button({Title="Restore All",Callback=function() LOD_L(false) LOD_T(false) LOD_P(false) LOD_FX(false) LOD_M(false) LOD_G(false) LOD_S(false) LOD_Sky(false) LOD_Atm(false) WindUI:Notify({Title="Optim",Content="Restaurado!",Duration=4}) end})

-- ================================================================
--  TAB: SERVER
-- ================================================================
TS2:Section({Title="Server"})
TS2:Button({Title="Copy Server ID",Callback=function() setclipboard(game.JobId) WindUI:Notify({Title="Copiado",Content=game.JobId,Duration=4}) end})
TS2:Button({Title="Show Server Info",Callback=function()
  local info="JobId: "..game.JobId:sub(1,18).."..  Plrs: "..#Plrs:GetPlayers().."/"..Plrs.MaxPlayers
  WindUI:Notify({Title="Server Info",Content=info,Duration=6}) ConLog(info,"info")
end})
TS2:Button({Title="Rejoin Same Server",Callback=function() TpS:TeleportToPlaceInstance(game.PlaceId,game.JobId,LP) end})
TS2:Button({Title="Server Hop - Emptiest",Callback=function() pcall(function() local d=HttpReq("https://games.roblox.com/v1/games/"..game.PlaceId.."/servers/Public?sortOrder=Asc&limit=100") if not d then return end local low={id="",p=999} for _,s in pairs(d.data) do if s.playing<low.p and s.id~=game.JobId then low={id=s.id,p=s.playing} end end if low.id~="" then TpS:TeleportToPlaceInstance(game.PlaceId,low.id,LP) end end) end})
TS2:Button({Title="Server Hop - Fullest", Callback=function() pcall(function() local d=HttpReq("https://games.roblox.com/v1/games/"..game.PlaceId.."/servers/Public?sortOrder=Desc&limit=100") if not d then return end local high={id="",p=0} for _,s in pairs(d.data) do if s.playing>high.p and s.id~=game.JobId then high={id=s.id,p=s.playing} end end if high.id~="" then TpS:TeleportToPlaceInstance(game.PlaceId,high.id,LP) end end) end})
TS2:Button({Title="Server Hop - Random",  Callback=function() pcall(function() local d=HttpReq("https://games.roblox.com/v1/games/"..game.PlaceId.."/servers/Public?sortOrder=Desc&limit=100") if not d then return end local v={} for _,s in pairs(d.data) do if s.playing<s.maxPlayers and s.id~=game.JobId then table.insert(v,s.id) end end if #v>0 then TpS:TeleportToPlaceInstance(game.PlaceId,v[math.random(1,#v)],LP) end end) end})

-- ================================================================
--  TAB: SETTINGS
-- ================================================================
TST:Section({Title="Interface"})
TST:Toggle({Title="Show Console",Default=false,Callback=function(v) ConsoleGui.Enabled=v end})
TST:Section({Title="Mira Fixa (Lock Aim)"})
TST:Toggle({Title="Lock Aim",Desc="Força HRP a olhar para onde a câmera aponta",Default=false,
  Callback=function(v) Cfg.LockAim=v end
})
TST:Section({Title="Config"})
TST:Toggle({Title="Auto Save Config",Default=false,Callback=function(v) Cfg.SaveCfg=v end})
TST:Button({Title="Save Now",Callback=function() SaveConfig() WindUI:Notify({Title="Salvo",Content="config.json salvo",Duration=4}) end})
TST:Button({Title="Reset Config",Callback=function() pcall(function() delfile("StrandedAwayHub/config.json") end) WindUI:Notify({Title="Reset",Content="Reinjecte para aplicar",Duration=4}) end})

-- ================================================================
--  TAB: LAB (Experimental)
-- ================================================================
TEXP:Section({Title="⚗ Laboratório Experimental"})
TEXP:Paragraph({Title="Aviso",Desc="Funcionalidades em teste. Podem ser instáveis ou não funcionar."})

-- Exp 1: Teleporte para posição de item selecionado
TEXP:Section({Title="[EXP] Teleporte até Item"})
local expItemList={} local expItemDrop
local function BuildExpItemList()
  expItemList={}
  local names={}
  local di=GetDroppedItems()
  local gi=GetGrabbableItems()
  for _,it in ipairs(di) do table.insert(expItemList,it) table.insert(names,"[D] "..it.name) end
  for _,it in ipairs(gi) do table.insert(expItemList,it) table.insert(names,"[G] "..it.name) end
  return names
end
expItemDrop=TEXP:Dropdown({Title="Selecionar Item",Values=BuildExpItemList(),Default={},Multi=false,
  Callback=function(v) end
})
TEXP:Button({Title="Atualizar",Callback=function()
  local n=BuildExpItemList() expItemDrop:Refresh(n,{})
  WindUI:Notify({Title="Lab",Content=#n.." itens",Duration=2})
end})
local expSelItem=nil
expItemDrop=TEXP:Dropdown({Title="Item para TP",Values=BuildExpItemList(),Default={},Multi=false,
  Callback=function(v)
    for _,it in ipairs(expItemList) do
      local label=(it.src=="dropped" and "[D] " or "[G] ")..it.name
      if label==v then expSelItem=it break end
    end
  end
})
TEXP:Button({Title="TP até Item Selecionado",Callback=function()
  if not expSelItem then WindUI:Notify({Title="Lab",Content="Selecione um item",Duration=3}) return end
  local part=GetItemPart(expSelItem.inst)
  if not part then WindUI:Notify({Title="Lab",Content="Part não encontrada",Duration=3}) return end
  if LP.Character and LP.Character:FindFirstChild("HumanoidRootPart") then
    LP.Character.HumanoidRootPart.CFrame=CFrame.new(part.Position+Vector3.new(0,3,0))
    WindUI:Notify({Title="Lab TP",Content="Teleportado para "..expSelItem.name,Duration=3})
    ConLog("ExpTP → "..expSelItem.name,"item")
  end
end})

-- Exp 2: Múltiplos métodos de Carry
TEXP:Section({Title="[EXP] Carry Multi-Método"})
TEXP:Paragraph({Title="Testa diferentes formas de puxar itens",Desc="Útil para descobrir qual método funciona"})
TEXP:Button({Title="Método A: CarryRemote x1",Callback=function()
  local items=GetDroppedItems() local n=0
  local r=GetCarryRemote()
  if not r then WindUI:Notify({Title="Lab",Content="CarryRemote não encontrado",Duration=3}) return end
  for _,it in ipairs(items) do
    pcall(function() r:FireServer(it.inst) end) n=n+1 task.wait(0.1)
  end
  WindUI:Notify({Title="Lab A",Content=n.." itens (x1)",Duration=3})
end})
TEXP:Button({Title="Método B: CarryRemote x2 (grab+drop)",Callback=function()
  local items=GetDroppedItems() local n=0
  for _,it in ipairs(items) do
    if CarryItem(it.inst) then n=n+1 end task.wait(0.15)
  end
  WindUI:Notify({Title="Lab B",Content=n.." itens (x2)",Duration=3})
end})
TEXP:Button({Title="Método C: GrabRemote (inventário)",Callback=function()
  local items=GetDroppedItems() local n=0
  for _,it in ipairs(items) do
    if CollectItem(it.inst) then n=n+1 end task.wait(0.08)
  end
  WindUI:Notify({Title="Lab C",Content=n.." itens (grab)",Duration=3})
end})
TEXP:Button({Title="Método D: Carry pela BasePart",Callback=function()
  local items=GetDroppedItems() local n=0
  local r=GetCarryRemote()
  if not r then return end
  for _,it in ipairs(items) do
    local part=GetItemPart(it.inst)
    if part then pcall(function() r:FireServer(part) end) n=n+1 task.wait(0.1) end
  end
  WindUI:Notify({Title="Lab D",Content=n.." itens (part)",Duration=3})
end})

-- Exp 3: Consume multi-método (Tool-based)
TEXP:Section({Title="[EXP] Consume Multi-Método (Tool)"})
TEXP:Button({Title="Método A: EquipTool + Activate (Food)",Callback=function()
  local ok=EquipAndUse("food")
  if ok then
    invFood=math.max(0,invFood-1)
    foodLbl:SetTitle("🍖 Food  no inventário: "..invFood)
    ConLog("ExpConsume A: Food Tool OK","food")
  else ConLog("ExpConsume A: falhou","warn") end
end})
TEXP:Button({Title="Método B: EquipTool + Activate (Water)",Callback=function()
  local ok=EquipAndUse("water")
  if ok then
    invWater=math.max(0,invWater-1)
    waterLbl:SetTitle("💧 Water no inventário: "..invWater)
    ConLog("ExpConsume B: Water Tool OK","food")
  else ConLog("ExpConsume B: falhou","warn") end
end})
TEXP:Button({Title="Método C: Listar Tools no Backpack",Callback=function()
  local bp=LP:FindFirstChild("Backpack")
  ConLog("== Backpack Tools ==","sys")
  if bp then
    for _,t in ipairs(bp:GetChildren()) do
      ConLog("  "..t.ClassName.." '"..t.Name.."'","item")
    end
  else ConLog("  Backpack não encontrado","warn") end
  local char=LP.Character
  ConLog("== Character Tools (equipadas) ==","sys")
  if char then
    for _,t in ipairs(char:GetChildren()) do
      if t:IsA("Tool") then ConLog("  "..t.Name,"item") end
    end
  end
  ConLog("== Fim ==","sys")
  ConsoleGui.Enabled=true
end})
TEXP:Button({Title="Listar TUDO no Character",Callback=function()
  local char=LP.Character if not char then return end
  ConLog("=== Character descendants ===","sys")
  for _,v in ipairs(char:GetDescendants()) do
    if v:IsA("RemoteEvent") or v:IsA("RemoteFunction") or v:IsA("BindableEvent") then
      ConLog("  ["..v.ClassName.."] "..v:GetFullName(),"item")
    end
  end
  ConLog("=== Fim ===","sys")
  ConsoleGui.Enabled=true
end})

-- Exp 4: Noclip avançado
TEXP:Section({Title="[EXP] Misc"})
TEXP:Button({Title="Resetar todos ESPs de Zombies",Callback=function()
  for mob,e in pairs(MobEspObjs) do
    pcall(function() e.N:Remove() e.H:Remove() e.MD:Remove() end)
  end
  MobEspObjs={}
  WindUI:Notify({Title="Lab",Content="ESP zombies resetado",Duration=3})
end})
TEXP:Button({Title="Resetar todos ESPs de Itens",Callback=function()
  for _,e in pairs(ItemEspObjs) do
    pcall(function() e.N:Remove() e.A:Remove() e.D2:Remove() end)
  end
  ItemEspObjs={}
  WindUI:Notify({Title="Lab",Content="ESP itens resetado",Duration=3})
end})
TEXP:Button({Title="Listar RemoteEvents em RepS.Game_System",Callback=function()
  local ok,gs=pcall(function() return RepS.Game_System end)
  if not ok or not gs then ConLog("Game_System não encontrado","warn") return end
  ConLog("=== RepS.Game_System RemoteEvents ===","sys")
  for _,v in ipairs(gs:GetDescendants()) do
    if v:IsA("RemoteEvent") or v:IsA("RemoteFunction") then
      ConLog("  "..v:GetFullName(),"item")
    end
  end
  ConLog("=== Fim ===","sys")
  ConsoleGui.Enabled=true
end})

-- ================================================================
--  ITEM ESP HELPERS
-- ================================================================
local function NewItemEntry(src)
  local nT=Drawing.new("Text") nT.Size=14 nT.Center=true nT.Outline=true
  local aT=Drawing.new("Text") aT.Size=13 aT.Center=true aT.Outline=true aT.Color=Color3.fromRGB(255,220,80)
  local dT=Drawing.new("Text") dT.Size=11 dT.Center=true dT.Outline=true dT.Color=Color3.fromRGB(160,255,160)
  return {N=nT,A=aT,D2=dT,src=src}
end

local function DrawItemFolder(folder,src,nameColor,myRoot)
  if not folder then return end
  for _,item in ipairs(folder:GetChildren()) do
    if not item.Parent then
      if ItemEspObjs[item] then
        pcall(function() ItemEspObjs[item].N:Remove() ItemEspObjs[item].A:Remove() ItemEspObjs[item].D2:Remove() end)
        ItemEspObjs[item]=nil
      end
    else
      local part=GetItemPart(item)
      if part then
        if not ItemEspObjs[item] then ItemEspObjs[item]=NewItemEntry(src) end
        local e=ItemEspObjs[item]
        if e.src~=src then
          pcall(function() e.N:Remove() e.A:Remove() e.D2:Remove() end)
          ItemEspObjs[item]=NewItemEntry(src) e=ItemEspObjs[item]
        end
        local p2d,vis=Cam:WorldToViewportPoint(part.Position+Vector3.new(0,1.5,0))
        local dist=myRoot and math.floor((myRoot.Position-part.Position).Magnitude) or 0
        local iName=tostring(ReadVal(item,"ItemName") or item.Name)
        local iAmt=ReadVal(item,"Amount")
        local emoji=ItemEmoji(iName)
        e.N.Color=nameColor
        e.N.Visible=vis
        e.A.Visible=vis and iAmt~=nil
        e.D2.Visible=vis
        if vis then
          e.N.Position=Vector2.new(p2d.X,p2d.Y-16) e.N.Text=emoji..iName
          if iAmt~=nil then
            e.A.Position=Vector2.new(p2d.X,p2d.Y)
            e.A.Text="x"..tostring(math.floor(tonumber(iAmt) or 0))
          end
          e.D2.Position=Vector2.new(p2d.X,p2d.Y+14) e.D2.Text=dist.."m"
        end
      end
    end
  end
end

-- ================================================================
--  RENDER LOOP  — throttled
--  • Física (Fly/Freecam/Noclip/LockAim): todo frame  — precisa ser fluido
--  • ESP + Stats: a cada 0.05s  (~20 fps) — não precisa de 60fps
--  • Item ESP: a cada 0.1s  (~10 fps) — itens ficam parados
--  • Zombie ESP cleanup: a cada 3s
-- ================================================================
local espCleanAccum = 0
local espAccum      = 0
local itemEspAccum  = 0
local ESP_RATE      = 0.033  -- ~30 fps (was 0.05/20fps — smoother movement)
local ITEM_RATE     = 0.10   -- 10 fps — items are static, fine

RS.RenderStepped:Connect(function(dt)
  espCleanAccum = espCleanAccum + dt
  espAccum      = espAccum      + dt
  itemEspAccum  = itemEspAccum  + dt

  -- ── FÍSICA: todo frame ──────────────────────────────────────
  if Cfg.Fly and not Cfg.Freecam and LP.Character then
    local hrp=LP.Character:FindFirstChild("HumanoidRootPart")
    local hum=LP.Character:FindFirstChild("Humanoid")
    if hrp and hum then
      hum.PlatformStand=true
      if not hrp:FindFirstChild("FlyBody") then
        local bv=Instance.new("BodyVelocity") bv.Name="FlyBody" bv.MaxForce=Vector3.new(9e9,9e9,9e9) bv.Parent=hrp
      end
      if not hrp:FindFirstChild("FlyGyro") then
        local bg=Instance.new("BodyGyro") bg.Name="FlyGyro" bg.MaxTorque=Vector3.new(9e9,9e9,9e9) bg.P=9e4 bg.Parent=hrp
      end
      local mv=Controls:GetMoveVector()
      hrp.FlyGyro.CFrame=Cam.CFrame
      hrp.FlyBody.Velocity=(Cam.CFrame.LookVector*-mv.Z+Cam.CFrame.RightVector*mv.X+Vector3.new(0,flyUD,0))*Cfg.FlySpeed
    end
  end

  if Cfg.Freecam and not Cfg.Fly and FCPart.Parent then
    local mv=Controls:GetMoveVector()
    FCPart.Position=FCPart.Position
      +Cam.CFrame.LookVector*-mv.Z*Cfg.FCSpeed
      +Cam.CFrame.RightVector*mv.X*Cfg.FCSpeed
  end

  if Cfg.Noclip and LP.Character then
    for _,bp in pairs(LP.Character:GetDescendants()) do
      if bp:IsA("BasePart") then bp.CanCollide=false end
    end
  end

  if Cfg.LockAim and LP.Character then
    local hrp=LP.Character:FindFirstChild("HumanoidRootPart")
    if hrp then
      local flat=Vector3.new(Cam.CFrame.LookVector.X,0,Cam.CFrame.LookVector.Z)
      if flat.Magnitude>0.01 then
        hrp.CFrame=CFrame.new(hrp.Position,hrp.Position+flat)
      end
    end
  end

  -- ── ESP + STATS: throttled a 20fps ──────────────────────────
  if espAccum < ESP_RATE then return end
  espAccum = 0

  local myRoot=LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")

  -- Stats HUD
  if Cfg.StatsUI then
    local now=tick()
    if now-lastStatT>=0.3 then cachedFPS=math.floor(1/dt) lastStatT=now end
    local zc=0 for _,_ in ipairs(GetZombies()) do zc=zc+1 end
    StatsTxt.Text=string.format(
      'Plrs:<font color="#9999FF">%d</font>  FPS:<font color="%s">%d</font>  Ping:<font color="#FFCC44">%dms</font>  Zombies:<font color="#FF8888">%d</font>',
      #Plrs:GetPlayers(),FpsClr(cachedFPS),cachedFPS,math.floor(LP:GetNetworkPing()*1000),zc)
  end

  -- Cleanup periódico ESP mortos
  if espCleanAccum>=3 then
    espCleanAccum=0
    CleanDeadMobEsp()
  end

  -- ===== PLAYER ESP + SKELETON =====
  for p,esp in pairs(EspObjs) do
    local char=p.Character
    local active=char
      and char:FindFirstChild("HumanoidRootPart")
      and char:FindFirstChild("Humanoid")
      and char.Humanoid.Health>0
    if active then
      local hrp=char.HumanoidRootPart
      local head=char:FindFirstChild("Head") or hrp
      local hum=char.Humanoid
      local top=head.Position+Vector3.new(0,0.6,0)
      local bot=hrp.Position-Vector3.new(0,2.8,0)
      local t2d,tV=Cam:WorldToViewportPoint(top)
      local b2d,_ =Cam:WorldToViewportPoint(bot)
      local clr=Cfg.RndC and esp.C or Color3.new(1,1,1)
      local dist=myRoot and math.floor((myRoot.Position-hrp.Position).Magnitude) or 0
      if tV then
        local h=math.abs(t2d.Y-b2d.Y) local w=h/2
        esp.N.Visible=Cfg.EspN  esp.N.Position=Vector2.new(t2d.X,t2d.Y-18) esp.N.Text=p.Name esp.N.Color=clr
        esp.B.Visible=Cfg.EspB  esp.B.Size=Vector2.new(w,h) esp.B.Position=Vector2.new(t2d.X-w/2,t2d.Y) esp.B.Color=clr
        esp.L.Visible=Cfg.EspL  esp.L.From=Vector2.new(Cam.ViewportSize.X/2,Cam.ViewportSize.Y) esp.L.To=Vector2.new(b2d.X,b2d.Y) esp.L.Color=clr
        esp.D.Visible=Cfg.EspDist esp.D.Position=Vector2.new(t2d.X,t2d.Y-32) esp.D.Text=dist.."m"
        esp.H.Visible=Cfg.PlayerHealth
        if Cfg.PlayerHealth then
          local rat=math.clamp(hum.Health/math.max(hum.MaxHealth,1),0,1)
          esp.H.Color=Color3.fromRGB(math.floor(255*(1-rat)),math.floor(255*rat),30)
          esp.H.Position=Vector2.new(t2d.X,b2d.Y+3) esp.H.Text=math.floor(rat*100).."%"
        end
        if Cfg.EspSkeleton then
          local isR15=char:FindFirstChild("UpperTorso")~=nil
          local pairs2=isR15 and SK_R15 or SK_R6
          for i=1,SK_MAX do
            local sk=esp.SK[i] local pair=pairs2[i]
            if pair then
              local pA=char:FindFirstChild(pair[1]) local pB=char:FindFirstChild(pair[2])
              if pA and pB and pA:IsA("BasePart") and pB:IsA("BasePart") then
                local a2d,aV=Cam:WorldToViewportPoint(pA.Position)
                local b2ds,bV=Cam:WorldToViewportPoint(pB.Position)
                if aV or bV then
                  sk.Visible=true sk.Color=clr
                  sk.From=Vector2.new(a2d.X,a2d.Y) sk.To=Vector2.new(b2ds.X,b2ds.Y)
                else sk.Visible=false end
              else sk.Visible=false end
            else sk.Visible=false end
          end
        else for _,l in ipairs(esp.SK) do l.Visible=false end end
      else ClearEsp(esp) end
    else ClearEsp(esp) end
  end

  -- ===== ZOMBIE ESP =====
  for _,mob in ipairs(GetZombies()) do
    local mr=mob:FindFirstChild("HumanoidRootPart") or mob:FindFirstChild("Torso")
    local hum=mob:FindFirstChildOfClass("Humanoid")
    local alive=mr and hum and hum.Health>0
    if alive and (Cfg.MobEsp or Cfg.MobHealth or Cfg.MobDist) then
      if not MobEspObjs[mob] then
        local n=Drawing.new("Text") n.Size=14 n.Center=true n.Outline=true n.Color=Color3.fromRGB(255,95,75)
        local h=Drawing.new("Text") h.Size=12 h.Center=true h.Outline=true h.Color=Color3.fromRGB(255,195,45)
        local d=Drawing.new("Text") d.Size=11 d.Center=true d.Outline=true d.Color=Color3.fromRGB(160,160,255)
        MobEspObjs[mob]={N=n,H=h,MD=d}
      end
      local p2d,vis=Cam:WorldToViewportPoint(mr.Position+Vector3.new(0,3,0))
      MobEspObjs[mob].N.Visible =Cfg.MobEsp    and vis or false
      MobEspObjs[mob].H.Visible =Cfg.MobHealth  and vis or false
      MobEspObjs[mob].MD.Visible=Cfg.MobDist    and vis or false
      if vis then
        MobEspObjs[mob].N.Position =Vector2.new(p2d.X,p2d.Y)    MobEspObjs[mob].N.Text=mob.Name
        MobEspObjs[mob].H.Position =Vector2.new(p2d.X,p2d.Y+16) MobEspObjs[mob].H.Text=FmtShort(hum.Health).."/"..FmtShort(hum.MaxHealth)
        if myRoot then
          MobEspObjs[mob].MD.Position=Vector2.new(p2d.X,p2d.Y-16)
          MobEspObjs[mob].MD.Text=math.floor((myRoot.Position-mr.Position).Magnitude).."m"
        end
      end
    end
  end

  -- ── ITEM ESP: throttled a 10fps ──────────────────────────────
  if itemEspAccum < ITEM_RATE then return end
  itemEspAccum = 0

  for inst,e in pairs(ItemEspObjs) do
    if not inst.Parent then
      pcall(function() e.N:Remove() e.A:Remove() e.D2:Remove() end)
      ItemEspObjs[inst]=nil
    end
  end
  if Cfg.ItemEspDropped then
    local ok2,folder=pcall(function() return workspace.World.DroppedItems end)
    if ok2 and folder then DrawItemFolder(folder,"dropped",Color3.fromRGB(80,220,255),myRoot) end
  end
  if Cfg.ItemEspGrabbable then
    local ok2,folder=pcall(function() return workspace.World.GrabbableItems end)
    if ok2 and folder then DrawItemFolder(folder,"grabbable",Color3.fromRGB(255,200,80),myRoot) end
  end
  if not Cfg.ItemEspDropped and not Cfg.ItemEspGrabbable then
    for _,e in pairs(ItemEspObjs) do
      pcall(function() e.N.Visible=false e.A.Visible=false e.D2.Visible=false end)
    end
  end

end)

task.delay(1.5,function()
  WindUI:Notify({Title="Stranded Away Hub",Content="Bem-vindo, "..LP.DisplayName.."!  v5",Duration=5})
  ConLog("Hub carregado — "..LP.DisplayName.."  v5","sys")
  ConLog("Dica: Use Lab > Listar Tools no Backpack para verificar os itens","info")
  -- sync inicial
  task.wait(2) SyncInvCounter()
end)
