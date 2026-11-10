- SUPERMAN | Cliente único para Xeno | R6/R15
-- F voo; WASD mover; Espaço subir; Ctrl descer; Shift acelerar.
-- E segurar visão de calor; B segurar sopro gelado; V segurar super-sopro.
-- X raio X; G corrida; J superpulo; H audição; Z zoom; T soco; C explosão solar.
-- + / - ajustam velocidade do voo; clique nos números para editar.
-- UI e efeitos locais. Não usa remotes desconhecidos, hooks ou bypass de anticheat.
-- Não implementa dano, godmode, cura ou congelamento autoritativos.
-- Superforça: impulso em objetos soltos próximos; sujeito à autoridade física do jogo.
-- Fontes: https://www.dc.com/characters/superman
-- https://www.dc.com/blog/2015/02/10/superman-new-power-new-printing
-- https://www.dc.com/blog/2021/04/14/ask-the-question-whats-up-with-the-dinosaur-in-the-batcave

local Players = game:GetService('Players')
local Run = game:GetService('RunService')
local UIS = game:GetService('UserInputService')
local Tween = game:GetService('TweenService')
local Debris = game:GetService('Debris')
local player = Players.LocalPlayer
assert(player, 'Execute no cliente depois de entrar no jogo.')
local pg = player:WaitForChild('PlayerGui')
-- Desliga esta versão anterior e o Flash para não haver disputa de movimento.
for _, name in ipairs({'SupermanUI', 'FlashUI'}) do
    local old = pg:FindFirstChild(name)
    if old then
        local stop = old:FindFirstChild(name == 'FlashUI' and 'FlashStop' or 'SupermanStop')
        if stop then stop:Fire() else old:Destroy() end
    end
end
local alive, generation = true, 0
local connections, characterConnections = {}, {}
local character, humanoid, root, head
local normal = {}
local states = {flight=false, heat=false, freeze=false, wind=false, xray=false, sprint=false, jump=false, hearing=false, zoom=false}
local flightSpeed, runSpeed = 160, 120
local MAX_FLIGHT, MAX_RUN = 1500, 600
local flightObjects, highlights, sounds = {}, {}, {}
local mover, rotation, mouthEmitter, mouthAttachment
local lastFlare, lastPunch = -100, -100
local blue, red = Color3.fromRGB(100,185,255), Color3.fromRGB(255,75,90)
local function make(class, properties, parent)
    local obj = Instance.new(class)
    for key, value in pairs(properties or {}) do obj[key] = value end
    obj.Parent = parent
    return obj
end
local function connect(signal, callback, list)
    local c = signal:Connect(callback)
    table.insert(list or connections, c)
    return c
end
local function disconnect(list)
    for _, c in ipairs(list) do c:Disconnect() end
    table.clear(list)
end
local fx = make('Folder', {Name='SupermanLocalFX_'..player.UserId}, workspace)
local gui = make('ScreenGui', {Name='SupermanUI',ResetOnSpawn=false,ZIndexBehavior=Enum.ZIndexBehavior.Sibling},pg)
local panel = make('Frame',{Position=UDim2.fromOffset(22,64),Size=UDim2.fromOffset(352,590),BackgroundColor3=Color3.fromRGB(10,12,18),BackgroundTransparency=0.12,BorderSizePixel=0},gui)
make('UICorner',{CornerRadius=UDim.new(0,16)},panel)
make('UIStroke',{Color=Color3.fromRGB(65,85,110),Thickness=1,Transparency=0.25},panel)
local scale = make('UIScale',{Scale=1},panel)
local title = make('TextLabel',{Text='SUPERMAN',Position=UDim2.fromOffset(20,14),Size=UDim2.fromOffset(260,32),BackgroundTransparency=1,TextColor3=Color3.new(1,1,1),Font=Enum.Font.GothamBold,TextSize=25,TextXAlignment=Enum.TextXAlignment.Left},panel)
make('TextLabel',{Text='KRYPTONIAN ABILITIES · CLIENT',Position=UDim2.fromOffset(21,46),Size=UDim2.fromOffset(290,18),BackgroundTransparency=1,TextColor3=blue,Font=Enum.Font.Gotham,TextSize=10,TextXAlignment=Enum.TextXAlignment.Left},panel)
local function btn(text,parent,size,pos)
    local b=make('TextButton',{Text=text,Size=size,Position=pos or UDim2.new(),BackgroundColor3=Color3.fromRGB(30,35,46),TextColor3=Color3.new(1,1,1),TextSize=13,Font=Enum.Font.GothamBold,BorderSizePixel=0,AutoButtonColor=true},parent)
    make('UICorner',{CornerRadius=UDim.new(0,8)},b)
    return b
end
local close=btn('×',panel,UDim2.fromOffset(28,28),UDim2.fromOffset(308,17))
local minimize=btn('−',panel,UDim2.fromOffset(28,22),UDim2.fromOffset(308,48))
local status=make('TextLabel',{Text='PRONTO',Position=UDim2.fromOffset(20,75),Size=UDim2.fromOffset(312,32),TextSize=11,Font=Enum.Font.Gotham,TextColor3=Color3.fromRGB(190,200,215),TextWrapped=true,TextXAlignment=Enum.TextXAlignment.Left,BackgroundTransparency=1},panel)
local body=make('ScrollingFrame',{Position=UDim2.fromOffset(16,114),Size=UDim2.new(1,-32,1,-130),BackgroundTransparency=1,BorderSizePixel=0,ScrollBarThickness=3,ScrollBarImageColor3=blue,CanvasSize=UDim2.new(),AutomaticCanvasSize=Enum.AutomaticSize.Y},panel)
make('UIListLayout',{Padding=UDim.new(0,8),SortOrder=Enum.SortOrder.LayoutOrder},body)
make('UIPadding',{PaddingRight=UDim.new(0,7),PaddingBottom=UDim.new(0,8)},body)
local order=0
local function card(height)
    order=order+1
    local frame=make('Frame',{Size=UDim2.new(1,0,0,height),BackgroundColor3=Color3.fromRGB(24,28,37),BackgroundTransparency=0.12,BorderSizePixel=0,LayoutOrder=order},body)
    make('UICorner',{CornerRadius=UDim.new(0,10)},frame)
    return frame
end
local function text(parent,value,x,y,w,h,size,color)
    return make('TextLabel',{Text=value,Position=UDim2.fromOffset(x,y),Size=UDim2.new(1,-w,0,h),BackgroundTransparency=1,TextColor3=color or Color3.new(1,1,1),Font=Enum.Font.Gotham,TextSize=size or 12,TextXAlignment=Enum.TextXAlignment.Left,TextWrapped=true},parent)
end
local speedBoxes={}
local function speedCard(labelText,initial,maximum,callback)
    local f=card(66)
    text(f,labelText,12,7,24,17,11,blue)
    local minus=btn('−',f,UDim2.fromOffset(34,28),UDim2.fromOffset(12,29))
    local plus=btn('+',f,UDim2.fromOffset(34,28),UDim2.new(1,-46,0,29))
    local box=make('TextBox',{Text=tostring(initial),Position=UDim2.fromOffset(53,27),Size=UDim2.new(1,-106,0,31),BackgroundTransparency=1,TextColor3=Color3.new(1,1,1),Font=Enum.Font.GothamBold,TextSize=23,ClearTextOnFocus=false},f)
    local value=initial
    local function set(n)
        if typeof(n)~='number' or n~=n or math.abs(n)==math.huge then box.Text=tostring(value);return end
        value=math.clamp(math.floor(n),16,maximum)
        box.Text=tostring(value);callback(value)
    end
    connect(minus.Activated,function()set(value-25)end)
    connect(plus.Activated,function()set(value+25)end)
    connect(box.FocusLost,function()set(tonumber(box.Text))end)
    speedBoxes[labelText]={set=set,get=function()return value end}
end
speedCard('VELOCIDADE DO VOO · STUDS/S',flightSpeed,MAX_FLIGHT,function(v)flightSpeed=v end)
speedCard('VELOCIDADE DA CORRIDA · STUDS/S',runSpeed,MAX_RUN,function(v)runSpeed=v end)
local buttons={}
local function row(id,name,key,description,callback)
    local f=card(60)
    text(f,name,12,7,112,19,12)
    text(f,description,12,28,112,25,9,Color3.fromRGB(150,165,185))
    local b=btn(key,f,UDim2.fromOffset(82,34),UDim2.new(1,-94,0,13))
    buttons[id]=b
    connect(b.Activated,callback)
end
local noticeVersion=0
local function notify(message)
    noticeVersion=noticeVersion+1
    local ticket=noticeVersion
    status.Text=message
    task.delay(4,function()if alive and ticket==noticeVersion then status.Text=states.flight and 'EM VOO · WASD / ESPAÇO / CTRL / SHIFT' or 'PRONTO · PODERES NO PAINEL' end end)
end
local function refresh()
    for id,b in pairs(buttons) do
        if states[id]~=nil then
            b.BackgroundColor3=states[id] and Color3.fromRGB(35,85,130) or Color3.fromRGB(35,41,54)
            b.TextColor3=states[id] and Color3.new(1,1,1) or Color3.fromRGB(175,190,210)
        end
    end
end
local function clearHighlights()
    for _,h in pairs(highlights)do h:Destroy()end
    table.clear(highlights)
end
local function restoreSounds()
    for sound,old in pairs(sounds)do
        pcall(function()
            if sound.Parent then
                if sound.Volume==old.appliedVolume then sound.Volume=old.volume end
                if sound.RollOffMaxDistance==old.appliedDistance then sound.RollOffMaxDistance=old.distance end
            end
        end)
    end
    table.clear(sounds)
end
local function stopFlight()
    local wasFlying=states.flight
    states.flight=false
    for _,object in ipairs(flightObjects)do object:Destroy()end
    table.clear(flightObjects);mover=nil;rotation=nil
    if humanoid and humanoid.Parent then
        if normal.autoRotate~=nil then humanoid.AutoRotate=normal.autoRotate end
        if normal.platformStand~=nil then humanoid.PlatformStand=normal.platformStand end
        if wasFlying and humanoid.Health>0 and not humanoid.PlatformStand then humanoid:ChangeState(Enum.HumanoidStateType.GettingUp)end
    end
    if wasFlying and root and root.Parent then root.AssemblyAngularVelocity=Vector3.zero;root.AssemblyLinearVelocity=Vector3.zero end
end
local function valid()
    return alive and root and root.Parent and humanoid and humanoid.Health>0 and head and head.Parent
end
local function toggleFlight()
    if states.flight then stopFlight();refresh();return end
    if not valid() or humanoid.Sit or root.Anchored then notify('Saia do assento antes de voar.');return end
    normal.autoRotate=humanoid.AutoRotate;normal.platformStand=humanoid.PlatformStand
    states.flight=true;humanoid.AutoRotate=false;humanoid.PlatformStand=true
    local a=make('Attachment',{Name='SupermanFlight'},root)
    table.insert(flightObjects,a)
    mover=make('LinearVelocity',{Name='SupermanVelocity',Attachment0=a,RelativeTo=Enum.ActuatorRelativeTo.World,VelocityConstraintMode=Enum.VelocityConstraintMode.Vector,ForceLimitsEnabled=false,VectorVelocity=Vector3.zero},root)
    table.insert(flightObjects,mover)
    rotation=make('AlignOrientation',{Name='SupermanOrientation',Attachment0=a,Mode=Enum.OrientationAlignmentMode.OneAttachment,MaxTorque=1e8,MaxAngularVelocity=20,Responsiveness=18,RigidityEnabled=false},root)
    table.insert(flightObjects,rotation)
    notify('VOO: WASD · ESPAÇO sobe · CTRL desce · SHIFT acelera')
    refresh()
end
local function toggle(id)
    if not valid() then return end
    states[id]=not states[id]
    if id=='sprint' then humanoid.WalkSpeed=states.sprint and runSpeed or normal.walk end
    if id=='jump' then
        humanoid.JumpPower=states.jump and 130 or normal.jumpPower
        humanoid.JumpHeight=states.jump and 40 or normal.jumpHeight
    end
    if id=='xray' and not states.xray then clearHighlights()end
    if id=='hearing' then
        if states.hearing then notify('AUDIÇÃO: amplia Sounds do cenário, sem acessar voice chat.')else restoreSounds()end
    end
    refresh()
end
local function setPower(id,on)
    if not valid() then return end
    states[id]=on
    if on then
        for _,other in ipairs({'heat','freeze','wind'})do if other~=id then states[other]=false end end
    end
    refresh()
end
local rayParams=RaycastParams.new()
rayParams.FilterType=Enum.RaycastFilterType.Exclude
rayParams.RespectCanCollide=true
local overlap=OverlapParams.new()
overlap.FilterType=Enum.RaycastFilterType.Exclude
overlap.MaxParts=60
local function aim(range)
    local camera=workspace.CurrentCamera
    if not camera then return root.Position+root.CFrame.LookVector*range end
    local point=UIS:GetMouseLocation()
    local ray=camera:ScreenPointToRay(point.X,point.Y)
    local hit=workspace:Raycast(ray.Origin,ray.Direction*range,rayParams)
    return hit and hit.Position or ray.Origin+ray.Direction*range
end
local function cosmeticPart(name,color,transparency)
    return make('Part',{Name=name,Anchored=true,CanCollide=false,CanTouch=false,CanQuery=false,CastShadow=false,Material=Enum.Material.Neon,Color=color,Transparency=transparency or 1,Size=Vector3.new(0.1,0.1,0.1)},fx)
end
local lasers={}
for i=1,4 do lasers[i]=cosmeticPart('HeatVision',i%2==0 and Color3.fromRGB(255,30,20) or Color3.fromRGB(255,240,200))end
local eyes={cosmeticPart('LeftEye',red),cosmeticPart('RightEye',red)}
for _,p in ipairs(eyes)do p.Shape=Enum.PartType.Ball;p.Size=Vector3.new(0.12,0.12,0.12)end
local impact=cosmeticPart('HeatImpact',Color3.fromRGB(255,125,35))
impact.Shape=Enum.PartType.Ball;impact.Size=Vector3.new(0.45,0.45,0.45)
make('PointLight',{Color=Color3.fromRGB(255,100,30),Brightness=0,Range=10},impact)
local function line(part,a,b,width,transparency)
    local length=(b-a).Magnitude
    if length<0.02 then part.Transparency=1;return end
    part.Size=Vector3.new(width,width,length)
    part.CFrame=CFrame.lookAt((a+b)/2,b)
    part.Transparency=transparency
end
local function hideHeat()
    for _,p in ipairs(lasers)do p.Transparency=1 end
    for _,p in ipairs(eyes)do p.Transparency=1 end
    impact.Transparency=1;impact.PointLight.Brightness=0
end
local function pulse(position,color,size,duration)
    if not alive then return end
    local p=cosmeticPart('Shockwave',color,0.4)
    p.Shape=Enum.PartType.Ball;p.Size=Vector3.new(0.5,0.5,0.5);p.Position=position
    Tween:Create(p,TweenInfo.new(duration or 0.45),{Size=Vector3.new(size,size,size),Transparency=1}):Play()
    Debris:AddItem(p,(duration or 0.45)+0.1)
end
local function propAssembly(part)
    if not part or not part:IsA('BasePart')then return nil end
    local assembly=part.AssemblyRootPart or part
    if assembly.Anchored or assembly.AssemblyMass>20000 then return nil end
    local cursor=part.Parent
    while cursor and cursor~=workspace do
        if cursor:IsA('Model')and cursor:FindFirstChildOfClass('Humanoid')then return nil end
        if cursor:IsA('Tool')then return nil end
        cursor=cursor.Parent
    end
    return assembly
end
local function pushProps(origin,direction,range,power)
    local seen={}
    for _,part in ipairs(workspace:GetPartBoundsInRadius(origin,range,overlap))do
        local assembly=propAssembly(part)
        if assembly and not seen[assembly]then
            seen[assembly]=true
            local delta=assembly.Position-origin
            if delta.Magnitude<0.05 or delta.Unit:Dot(direction)>0.25 then
                local hit=workspace:Raycast(origin,delta,rayParams)
                if not hit or(hit.Instance:IsA('BasePart')and(hit.Instance.AssemblyRootPart or hit.Instance)==assembly)then
                    pcall(function()assembly:ApplyImpulse((direction*power+Vector3.new(0,power*0.15,0))*assembly.AssemblyMass)end)
                end
            end
        end
    end
end
local function punch()
    if not valid()or os.clock()-lastPunch<0.65 then return end
    lastPunch=os.clock()
    local target=aim(30);local delta=target-root.Position
    local direction=delta.Magnitude>0.01 and delta.Unit or root.CFrame.LookVector
    pulse(root.Position+direction*4,Color3.fromRGB(210,230,255),9,0.25)
    pushProps(root.Position+Vector3.new(0,1,0),direction,11,120)
    notify('SUPERFORÇA: impulso em objetos soltos próximos; sem dano a jogadores.')
end
local function solarFlare()
    if not valid()then return end
    local remaining=12-(os.clock()-lastFlare)
    if remaining>0 then notify(string.format('EXPLOSÃO SOLAR: aguarde %.0fs.',remaining));return end
    lastFlare=os.clock()
    pulse(root.Position,Color3.fromRGB(255,190,70),80,0.85)
    pulse(root.Position,Color3.fromRGB(255,245,210),45,0.6)
    notify('EXPLOSÃO SOLAR · efeito visual local · recarga de 12s')
end
row('flight','Voo','F','WASD · Espaço / Ctrl · Shift',toggleFlight)
row('heat','Visão de calor','E','Segure E ou alterne pelo botão',function()setPower('heat',not states.heat)end)
row('xray','Visão de raio X','X','Silhuetas de personagens e NPCs',function()toggle('xray')end)
row('freeze','Sopro congelante','B','Névoa e gelo visual no impacto',function()setPower('freeze',not states.freeze)end)
row('wind','Super-sopro','V','Vento e impulso em objetos soltos',function()setPower('wind',not states.wind)end)
row('sprint','Supervelocidade','G','Velocidade regulável em solo',function()toggle('sprint')end)
row('jump','Superpulo','J','Ative e pule com Espaço',function()toggle('jump')end)
row('hearing','Superaudição','H','Amplifica sons 3D existentes',function()toggle('hearing')end)
row('zoom','Visão telescópica','Z','Zoom óptico local pela câmera',function()toggle('zoom')end)
row('strength','Superforça','T','Soco de impacto em objetos soltos',punch)
row('flare','Explosão solar','C','Onda visual · intervalo de 12s',solarFlare)
local info=card(88)
text(info,'INVULNERABILIDADE E FATOR DE CURA',12,8,24,20,10,blue)
text(info,'Poderes dos quadrinhos que exigem implementação no servidor. Este script não impede dano nem altera a vida real do jogo.',12,31,24,48,10,Color3.fromRGB(160,175,195))
local minimized=false
connect(minimize.Activated,function()
    minimized=not minimized;body.Visible=not minimized;status.Visible=not minimized
    panel.Size=UDim2.fromOffset(352,minimized and 80 or 590)
    minimize.Text=minimized and '+'or '−'
end)
-- Arrastar o cabeçalho, sem interceptar o mouse usado para mirar.
local dragging,dragOrigin,panelOrigin
connect(title.InputBegan,function(input)
    if input.UserInputType==Enum.UserInputType.MouseButton1 or input.UserInputType==Enum.UserInputType.Touch then
        dragging=true;dragOrigin=input.Position;panelOrigin=panel.Position
    end
end)
connect(UIS.InputChanged,function(input)
    if dragging and(input.UserInputType==Enum.UserInputType.MouseMovement or input.UserInputType==Enum.UserInputType.Touch)then
        local delta=input.Position-dragOrigin
        panel.Position=UDim2.new(panelOrigin.X.Scale,panelOrigin.X.Offset+delta.X,panelOrigin.Y.Scale,panelOrigin.Y.Offset+delta.Y)
    end
end)
local keyToggles={
    [Enum.KeyCode.F]=toggleFlight,[Enum.KeyCode.X]=function()toggle('xray')end,
    [Enum.KeyCode.G]=function()toggle('sprint')end,[Enum.KeyCode.J]=function()toggle('jump')end,
    [Enum.KeyCode.H]=function()toggle('hearing')end,[Enum.KeyCode.Z]=function()toggle('zoom')end,
    [Enum.KeyCode.T]=punch,[Enum.KeyCode.C]=solarFlare
}
local holdKeys={[Enum.KeyCode.E]='heat',[Enum.KeyCode.B]='freeze',[Enum.KeyCode.V]='wind'}
connect(UIS.InputBegan,function(input,processed)
    if processed or UIS:GetFocusedTextBox()then return end
    local action=keyToggles[input.KeyCode]
    if action then action();return end
    local id=holdKeys[input.KeyCode]
    if id then setPower(id,true);return end
    if input.KeyCode==Enum.KeyCode.Equals or input.KeyCode==Enum.KeyCode.KeypadPlus then
        local c=speedBoxes['VELOCIDADE DO VOO · STUDS/S'];c.set(c.get()+25)
    elseif input.KeyCode==Enum.KeyCode.Minus or input.KeyCode==Enum.KeyCode.KeypadMinus then
        local c=speedBoxes['VELOCIDADE DO VOO · STUDS/S'];c.set(c.get()-25)
    end
end)
connect(UIS.InputEnded,function(input)
    if input.UserInputType==Enum.UserInputType.MouseButton1 or input.UserInputType==Enum.UserInputType.Touch then dragging=false end
    local id=holdKeys[input.KeyCode]
    if id then states[id]=false;refresh()end
end)
connect(UIS.WindowFocusReleased,function()
    dragging=false;states.heat=false;states.freeze=false;states.wind=false
    if mover then mover.VectorVelocity=Vector3.zero end
    refresh()
end)
local lastCamera,baseFov,ownFov
local function restoreCamera()
    if lastCamera and ownFov and math.abs(lastCamera.FieldOfView-ownFov)<0.2 then lastCamera.FieldOfView=baseFov end
    ownFov=nil
end
local function resetCharacter()
    stopFlight();restoreCamera();clearHighlights();restoreSounds();hideHeat()
    if mouthEmitter then mouthEmitter:Destroy();mouthEmitter=nil end
    if mouthAttachment then mouthAttachment:Destroy();mouthAttachment=nil end
    if humanoid and humanoid.Parent then
        humanoid.WalkSpeed=normal.walk or 16
        humanoid.JumpPower=normal.jumpPower or 50
        humanoid.JumpHeight=normal.jumpHeight or 7.2
    end
    for id in pairs(states)do states[id]=false end
    disconnect(characterConnections);refresh()
end
local function bind(char)
    generation=generation+1
    local ticket=generation
    resetCharacter();character=nil;humanoid=nil;root=nil;head=nil
    local h=char:WaitForChild('Humanoid',10)
    local r=char:WaitForChild('HumanoidRootPart',10)
    local hd=char:WaitForChild('Head',10)
    if not alive or ticket~=generation or player.Character~=char or not h or not r or not hd then return end
    character,humanoid,root,head=char,h,r,hd
    normal={walk=h.WalkSpeed,jumpPower=h.JumpPower,jumpHeight=h.JumpHeight,autoRotate=h.AutoRotate,platformStand=h.PlatformStand}
    rayParams.FilterDescendantsInstances={char,fx}
    overlap.FilterDescendantsInstances={char,fx}
    mouthAttachment=make('Attachment',{Name='SupermanMouth',Position=Vector3.new(0,-0.2,-hd.Size.Z*0.55)},hd)
    mouthEmitter=make('ParticleEmitter',{
        Name='SupermanBreath',Texture='rbxasset://textures/particles/smoke_main.dds',
        Rate=75,Lifetime=NumberRange.new(0.3,0.65),Speed=NumberRange.new(35,60),
        Size=NumberSequence.new({NumberSequenceKeypoint.new(0,0.25),NumberSequenceKeypoint.new(1,3)}),
        Transparency=NumberSequence.new({NumberSequenceKeypoint.new(0,0.4),NumberSequenceKeypoint.new(1,1)}),
        Color=ColorSequence.new(Color3.fromRGB(170,230,255)),LightEmission=0.6,
        SpreadAngle=Vector2.new(12,12),EmissionDirection=Enum.NormalId.Front,Enabled=false
    },mouthAttachment)
    connect(h.Died,function()resetCharacter();notify('Personagem morreu. Poderes disponíveis após renascer.')end,characterConnections)
    connect(h.Seated,function(active)if active and states.flight then stopFlight();refresh()end end,characterConnections)
    notify('PRONTO · F para voar · E para visão de calor')
end
connect(player.CharacterAdded,bind)
if player.Character then task.spawn(bind,player.Character)end
local function key(code)return not UIS:GetFocusedTextBox()and UIS:IsKeyDown(code)end
local function limitMotion(velocity,dt)
    if velocity.Magnitude<0.01 or dt<=0 then return velocity end
    local cast=workspace:Blockcast(root.CFrame,Vector3.new(1.6,2,1.6),velocity*dt+velocity.Unit*0.6,rayParams)
    if cast then return velocity.Unit*math.min(velocity.Magnitude,math.max(0,cast.Distance-0.6)/dt)end
    return velocity
end
connect(Run.PreSimulation,function(dt)
    if not valid()then return end
    if states.flight then
        if humanoid.Sit or root.Anchored or not mover or not mover.Parent then stopFlight();refresh();return end
        local camera=workspace.CurrentCamera
        if not camera then return end
        local direction=Vector3.zero
        if key(Enum.KeyCode.W)then direction=direction+camera.CFrame.LookVector end
        if key(Enum.KeyCode.S)then direction=direction-camera.CFrame.LookVector end
        if key(Enum.KeyCode.D)then direction=direction+camera.CFrame.RightVector end
        if key(Enum.KeyCode.A)then direction=direction-camera.CFrame.RightVector end
        if key(Enum.KeyCode.Space)then direction=direction+Vector3.yAxis end
        if key(Enum.KeyCode.LeftControl)or key(Enum.KeyCode.RightControl)then direction=direction-Vector3.yAxis end
        if direction.Magnitude>0 then direction=direction.Unit end
        local boost=key(Enum.KeyCode.LeftShift)or key(Enum.KeyCode.RightShift)
        local velocity=direction*math.min(MAX_FLIGHT,flightSpeed*(boost and 2.5 or 1))
        mover.VectorVelocity=limitMotion(velocity,dt)
        local look=camera.CFrame.LookVector
        if math.abs(look:Dot(Vector3.yAxis))>0.99 then look=(look+camera.CFrame.UpVector*0.05).Unit end
        rotation.CFrame=CFrame.lookAt(Vector3.zero,look)*CFrame.Angles(math.rad(direction.Magnitude>0 and -55 or 0),0,0)
    elseif states.sprint and not humanoid.Sit and not root.Anchored then
        humanoid.WalkSpeed=runSpeed
        local d=humanoid.MoveDirection
        local velocity=Vector3.new(d.X,0,d.Z)*runSpeed
        velocity=limitMotion(velocity,dt)
        root.AssemblyLinearVelocity=Vector3.new(velocity.X,root.AssemblyLinearVelocity.Y,velocity.Z)
    end
end)
-- Raio X respeita o que já foi carregado pelo cliente; StreamingEnabled limita alvos distantes.
local candidates={}
local function candidate(obj)
    if obj:IsA('Humanoid')and obj.Parent and obj.Parent:IsA('Model')then candidates[obj.Parent]=obj end
end
for _,obj in ipairs(workspace:GetDescendants())do candidate(obj)end
connect(workspace.DescendantAdded,candidate)
local function updateXray()
    local list={}
    for model,h in pairs(candidates)do
        if not model:IsDescendantOf(workspace)or not h.Parent then
            candidates[model]=nil
        elseif model~=character and h.Health>0 then
            local p=model:FindFirstChild('HumanoidRootPart')or model.PrimaryPart
            if p and root then
                local distance=(p.Position-root.Position).Magnitude
                if distance<=2000 then table.insert(list,{model=model,distance=distance})end
            end
        end
    end
    table.sort(list,function(a,b)return a.distance<b.distance end)
    local keep={}
    for i=1,math.min(24,#list)do
        local model=list[i].model;keep[model]=true
        if not highlights[model]then
            highlights[model]=make('Highlight',{Name='SupermanXray',Adornee=model,DepthMode=Enum.HighlightDepthMode.AlwaysOnTop,FillColor=blue,OutlineColor=Color3.new(1,1,1),FillTransparency=0.68,OutlineTransparency=0.08},fx)
        end
    end
    for model,h in pairs(highlights)do if not keep[model]then h:Destroy();highlights[model]=nil end end
end
local soundCandidates={}
local function soundCandidate(obj)if obj:IsA('Sound')then soundCandidates[obj]=true end end
for _,obj in ipairs(workspace:GetDescendants())do soundCandidate(obj)end
connect(workspace.DescendantAdded,soundCandidate)
local function updateHearing()
    local count=0
    for sound in pairs(soundCandidates)do
        if not sound:IsDescendantOf(workspace)then soundCandidates[sound]=nil
        else
            local parent=sound.Parent
            local position=parent:IsA('BasePart')and parent.Position or(parent:IsA('Attachment')and parent.WorldPosition)
            if position and root and(position-root.Position).Magnitude<=450 and sound.IsPlaying then
                count=count+1
                if count<=48 and not sounds[sound]then
                    local old={volume=sound.Volume,distance=sound.RollOffMaxDistance}
                    old.appliedVolume=math.min(1.5,old.volume*1.8)
                    old.appliedDistance=math.max(old.distance,450)
                    sounds[sound]=old
                    sound.Volume=old.appliedVolume;sound.RollOffMaxDistance=old.appliedDistance
                end
            elseif sounds[sound]then
                local old=sounds[sound]
                if sound.Volume==old.appliedVolume then sound.Volume=old.volume end
                if sound.RollOffMaxDistance==old.appliedDistance then sound.RollOffMaxDistance=old.distance end
                sounds[sound]=nil
            end
        end
    end
end
local xrayClock,hearingClock,breathClock=0,0,0
connect(Run.Heartbeat,function(dt)
    if not valid()then return end
    xrayClock=xrayClock+dt;hearingClock=hearingClock+dt;breathClock=breathClock+dt
    if states.xray and xrayClock>=0.5 then xrayClock=0;updateXray()end
    if states.hearing and hearingClock>=0.8 then hearingClock=0;updateHearing()end
    if not(states.freeze or states.wind)or breathClock<0.15 then return end
    breathClock=0
    local origin=head.Position+head.CFrame.LookVector*0.7
    local delta=aim(70)-origin
    if delta.Magnitude<0.01 then return end
    local direction=delta.Unit
    if states.wind then pushProps(origin,direction,28,18)end
    if states.freeze then
        local hit=workspace:Raycast(origin,direction*60,rayParams)
        if hit then pulse(hit.Position,Color3.fromRGB(145,225,255),2.8,0.65)end
    end
end)
connect(Run.RenderStepped,function(dt)
    local camera=workspace.CurrentCamera
    if camera then
        scale.Scale=math.clamp(math.min(camera.ViewportSize.X/410,camera.ViewportSize.Y/690),0.45,1)
        if camera~=lastCamera then restoreCamera();lastCamera=camera;baseFov=camera.FieldOfView end
        local goal=states.zoom and 18 or(states.flight and math.min(95,baseFov+12)or baseFov)
        if states.zoom or states.flight or ownFov then
            ownFov=camera.FieldOfView+(goal-camera.FieldOfView)*(1-math.exp(-dt*9))
            camera.FieldOfView=ownFov
            if not states.zoom and not states.flight and math.abs(ownFov-baseFov)<0.05 then restoreCamera()end
        else baseFov=camera.FieldOfView end
    end
    if not valid()then hideHeat();if mouthEmitter then mouthEmitter.Enabled=false end;return end
    if mouthEmitter then
        mouthEmitter.Enabled=states.freeze or states.wind
        if mouthEmitter.Enabled then
            local origin=head.CFrame:PointToWorldSpace(Vector3.new(0,-0.2,-head.Size.Z*0.55))
            local target=aim(80)
            if(target-origin).Magnitude>0.01 then mouthAttachment.CFrame=head.CFrame:ToObjectSpace(CFrame.lookAt(origin,target))end
            mouthEmitter.Color=ColorSequence.new(states.freeze and Color3.fromRGB(150,225,255)or Color3.fromRGB(220,230,240))
        end
    end
    if not states.heat then hideHeat();return end
    local target=aim(1200)
    for i=1,2 do
        local eye=head.CFrame:PointToWorldSpace(Vector3.new(i==1 and -0.15 or 0.15,head.Size.Y*0.15,-head.Size.Z*0.55))
        eyes[i].Position=eye;eyes[i].Transparency=0
        local delta=target-eye
        local hit=delta.Magnitude>0.01 and workspace:Raycast(eye,delta,rayParams)
        local endpoint=hit and hit.Position or target
        line(lasers[(i-1)*2+1],eye,endpoint,0.055,0)
        line(lasers[(i-1)*2+2],eye,endpoint,0.17,0.6)
        if i==1 then impact.Position=endpoint;impact.Transparency=hit and 0.15 or 1;impact.PointLight.Brightness=hit and 2 or 0 end
    end
end)
local stop=make('BindableEvent',{Name='SupermanStop'},gui)
local function cleanup()
    if not alive then return end
    generation=generation+1
    resetCharacter();alive=false
    disconnect(connections)
    fx:Destroy();gui:Destroy()
end
connect(stop.Event,cleanup)
connect(close.Activated,cleanup)
refresh()
print('[SUPERMAN] Carregado. F voo, E calor, X raio X. × encerra e restaura alterações.')
