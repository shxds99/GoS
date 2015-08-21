local spellList = { LeeSin = _W, Katarina = _E, Jax = _Q }
local myHero = GetMyHero()
local spellSlot = spellList[ GetObjectName(myHero) ]

-- only 3 champion can do ward jump now
if not spellSlot then return end

require 'MapPositionGOS'

wardRange = 600
local debug = false
local jumpTarget
local wardLock
local mousePos
local wardpos
local maxPos 
local spellObj

-- copy from perfect ward
local wardItems = {        
        { id = 3340, spellName = "TrinketTotemLvl1", screenName = "?"},
        { id = 3350, spellName = "TrinketTotemLvl2", screenName = "?"},
        { id = 3361, spellName = "TrinketTotemLvl3", screenName = "?"},
        { id = 3362, spellName = "TrinketTotemLvl3B", screenName = "?"},
        -- { id = 3154, spellName = "wrigglelantern", screenName = "?"},
        -- { id = 3160, spellName = "FeralFlare", screenName = "?"},
        { id = 2045, spellName = "ItemGhostWard", screenName = "?"},
        { id = 2049, spellName = "ItemGhostWard", screenName = "?"},
        { id = 2050, spellName = "ItemMiniWard", screenName = "?"},        
        { id = 2044, spellName = "sightward", screenName = "Sight Ward"},
        { id = 2043, spellName = "VisionWard", screenName = "Vision Ward"}
}

-- modified from Inspired.lua
local function GetDistanceSqr(p1,p2)
    p2 = p2 or GetOrigin(myHero)
    local dx = p1.x - p2.x
    local dz = (p1.z or p1.y) - (p2.z or p2.y)
    return dx*dx + dz*dz
end

local function IsInDistance(r, p1, p2, fast)
		local fast = fast or false
		-- local fast = true
		if fast then
			-- faster check, but don't know why still fps drop...
			local p1y = p1.z or p1.y
			local p2y = p2.z or p2.y
			return (p1.x + r >= p2.x) and (p1.x - r <= p2.x) and (p1y + r >= p2y) and (p1y - r <= p2y)
		else
    	return GetDistanceSqr(p1, p2) < r*r
    end
end

-- modified from vayne.lua(Laiha)
local function calcMaxPos(pos)
	local origin = GetOrigin(myHero)
	local vectorx = pos.x-origin.x
	local vectory = pos.y-origin.y
	local vectorz = pos.z-origin.z
	local dist= math.sqrt(vectorx^2+vectory^2+vectorz^2)
	return {x = origin.x + wardRange * vectorx / dist ,y = origin.y + wardRange * vectory / dist, z = origin.z + wardRange * vectorz / dist}
end

local function validTarget( object )
	objType = GetObjectType(object)
	
	-- need check type == ward, but rito set everything is minion lol	
	if GetObjectName(myHero) == "LeeSin" then
		-- poor lee cannot jump to enemy
		return objType == Obj_AI_Hero or objType == Obj_AI_Minion and IsVisible(object) and GetTeam(object) == GetTeam(myHero)
	else		
		return objType == Obj_AI_Hero or objType == Obj_AI_Minion and IsVisible(object)
	end
end

local findWardSlot = function ()
	local slot = 0
	for i,wardItem in pairs(wardItems) do
		slot = GetItemSlot(myHero,wardItem.id)
		if slot > 0 and CanUseSpell(myHero, slot) == READY then return slot end
	end
end

local function putWard(pos0)	
	local slot = findWardSlot()

	local pos = pos0
	if not IsInDistance(wardRange, pos) then
		pos = calcMaxPos(pos)
	end
	
	-- don't put ward in wall, or it will jump fail like noob
	if MapPosition:inWall(pos) then return end

	if slot and slot > 0 then
		if debug then DrawText("slot : "..slot,20,0,50,0xffffff00) end
		CastSkillShot(slot,pos.x,pos.y,pos.z)
	end
end

local drawDebugInfo = function ()
	DrawCircle(mousePos.x,mousePos.y,mousePos.z,200,0,0,0xff00ff00);
	if wardpos then
		DrawCircle(wardpos.x,wardpos.y,wardpos.z,200,0,0,0xffffff00);
	end
	DrawCircle(maxPos.x,maxPos.y,maxPos.z,200,0,0,0xffff0000);
	DrawCircle( GetOrigin(myHero).x,GetOrigin(myHero).y,GetOrigin(myHero).z,GetCastRange(myHero,spellSlot),0,0,0xffffffff);

	if jumpTarget then
		DrawText("jumpTarget : "..GetObjectName(jumpTarget),20,0,100,0xffffff00)
	end

	if spellObj and spellObj.name then
		DrawText("spellObj : "..spellObj.name,20,0,200,0xffffff00)
	end

	if wardLock then
		DrawText("wardLock : "..wardLock,20,100,100,0xffffff00)
	end
end

local spellLock = nil

function wardJump( pos )
	if not spellLock and CanUseSpell(myHero, spellSlot) == READY and GetCastName(myHero, spellSlot) ~= "blindmonkwtwo" then
		if jumpTarget then
			CastTargetSpell(jumpTarget, spellSlot)
			if debug then PrintChat(GetCastName(myHero, spellSlot).." GetTickCount "..GetTickCount()) end
			spellLock = GetTickCount()
		elseif not wardLock then
			wardLock = GetTickCount()
			putWard(pos)
		end
	end
end

OnLoop(function(myHero)
	mousePos = GetMousePos()
	maxPos = calcMaxPos(mousePos)

	if debug then	drawDebugInfo()	end	
	
	-- check spell name not lee W2
	if not spellLock and wardLock and jumpTarget and CanUseSpell(myHero, spellSlot) == READY and GetCastName(myHero, spellSlot) ~= "blindmonkwtwo" then
		CastTargetSpell(jumpTarget, spellSlot)
		if debug then PrintChat(GetCastName(myHero, spellSlot).." wardLock "..wardLock) end
		spellLock = GetTickCount()
	end

	if KeyIsDown(string.byte("Z")) then
		wardJump(mousePos)
	end

	-- wardLock putWard every 500 ms
	if wardLock and (wardLock + 500) < GetTickCount()  then
		wardLock = nil
	end

	-- spellLock putWard every 500 ms
	if spellLock and (spellLock + 500) < GetTickCount()  then
		spellLock = nil
	end

	jumpTarget = nil
	spellObj = nil
	wardpos = nil
end)

OnObjectLoop(function(object,myHero)

	-- if jumpTarget or not mousePos then return end
	if not mousePos or not validTarget(object) then return end

	local pos = mousePos
	if not IsInDistance(wardRange, mousePos, GetOrigin(myHero)) then
		pos = maxPos
	end

  if wardpos and IsInDistance(200, GetOrigin(object), wardpos) then
  	jumpTarget = object
  end
  if IsInDistance(200, GetOrigin(object), pos) then
   	jumpTarget = object
  end
end)

OnProcessSpell(function(unit,spell)
	-- TODO check spell == ward instead just check not hero spell
	if unit and unit == myHero and spell and not spell.name:lower():find("katarina") and not spell.name:lower():find("leesin") and not spell.name:lower():find("jax") then
		spellObj = spell
		wardpos = spellObj.endPos
	end
end)

PrintChat("shxds99's WardJump - Loaded")
