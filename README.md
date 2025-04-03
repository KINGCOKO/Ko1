--[[
    KillAura Script (Modified Version)
    - Removed the maximum reach limit of 40 from the UI input.
    - Added notification on successful reach change.
]]

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local StarterGui = game:GetService("StarterGui")
local plr = Players.LocalPlayer

local loop = false -- เริ่มต้นด้วยการปิด
local retry = true
_G.name = "sword" -- ชื่อเครื่องมือเริ่มต้น
_G.enemOnly = true -- โจมตีเฉพาะศัตรูเริ่มต้น
local reach = 10 -- ระยะเริ่มต้น

-- โหลด UI Library
local library = loadstring(game:HttpGet("https://raw.githubusercontent.com/bloodball/-back-ups-for-libs/main/turtle"))()
local Window = library:Window("KillAura (Modded)") -- เปลี่ยนชื่อหน้าต่างเล็กน้อย

-- ฟังก์ชันหาเครื่องมือ
function findTool(String)
    local strl = String:lower()
    if plr.Character then
        for _, v in pairs(plr.Character:GetChildren()) do
            if v:IsA("Tool") and v.Name:lower():match(strl) then
                return v
            end
        end
    end
    if plr.Backpack then
        for _, v in pairs(plr.Backpack:GetChildren()) do
            if v:IsA("Tool") and v.Name:lower():match(strl) then
                return v
            end
        end
    end
    return nil -- ไม่เจอเครื่องมือ
end

-- ฟังก์ชันเอาเครื่องมือ (Handle กรณีไม่เจอ)
function getTool()
    local tool = findTool(_G.name)
    if tool and tool.Parent ~= plr.Character then
         -- ถ้าเจอในเป้ ให้เอามาใส่ตัวละคร
         tool.Parent = plr.Character
         RunService.Heartbeat:Wait() -- รอเล็กน้อยให้แน่ใจว่า Equip แล้ว
    end
    return tool -- คืนค่า tool (อาจจะเป็น nil ถ้าหาไม่เจอ)
end

-- ฟังก์ชันแจ้งเตือน
local function notify(Title, Text, Duration)
    StarterGui:SetCore("SendNotification", {
        Title = Title or "Notification",
        Text = Text or "",
        Duration = Duration or 5,
    })
end

-- ฟังก์ชันหลัก KillAura
function KillAura()
    while loop and RunService.Heartbeat:Wait() do
        local playerCharacter = plr.Character
        local playerRoot = playerCharacter and playerCharacter:FindFirstChild("HumanoidRootPart")
        local playerHumanoid = playerCharacter and playerCharacter:FindFirstChildOfClass("Humanoid")

        if not playerCharacter or not playerRoot or not playerHumanoid or playerHumanoid.Health <= 0 then
            -- ถ้าผู้เล่นไม่มีตัวละคร หรือตายแล้ว ให้หยุดชั่วคราว
            task.wait(1)
            continue
        end

        local currentTool = getTool() -- หาเครื่องมือ

        for _, targetPlr in pairs(Players:GetPlayers()) do
            if targetPlr ~= plr then -- ไม่ใช่ตัวเอง
                local targetCharacter = targetPlr.Character
                local targetRoot = targetCharacter and targetCharacter:FindFirstChild("HumanoidRootPart")
                local targetHumanoid = targetCharacter and targetCharacter:FindFirstChildOfClass("Humanoid")

                if targetCharacter and targetRoot and targetHumanoid and targetHumanoid.Health > 0 then -- เป้าหมายมีชีวิต
                    if targetCharacter:FindFirstChildOfClass("ForceField") == nil then -- ไม่มี ForceField
                        if (_G.enemOnly and targetPlr.TeamColor ~= plr.TeamColor) or (not _G.enemOnly) then -- เช็คทีมถ้าเปิดโหมด Enemies Only

                            local distance = (targetRoot.Position - playerRoot.Position).magnitude
                            if distance <= reach then -- อยู่ในระยะ
                                pcall(function()
                                    -- วาร์ปไปใกล้ๆ เป้าหมาย
                                    playerRoot.CFrame = targetRoot.CFrame * CFrame.new(-1.6, 0, 1.8)
                                    task.wait() -- รอเล็กน้อยหลังวาร์ป

                                    if currentTool then -- ถ้ามีเครื่องมือ
                                        -- ตรวจสอบว่า Equip ถูกต้องไหม
                                        if playerCharacter:FindFirstChildOfClass("Tool") ~= currentTool then
                                             playerHumanoid:UnequipTools()
                                             task.wait()
                                             currentTool.Parent = playerCharacter -- ลอง Equip ใหม่
                                             task.wait()
                                        end

                                        -- โจมตี (วนซ้ำตามเดิม แต่ใส่ delay เล็กน้อย)
                                        for i = 1, 5 do -- ลดจำนวนรอบลง อาจจะพอ
                                             if targetHumanoid.Health <= 0 then break end -- ถ้าเป้าหมายตายแล้ว หยุดโจมตี
                                             currentTool:Activate()
                                             task.wait(0.05) -- ใส่ดีเลย์เล็กน้อยระหว่าง Activate
                                        end

                                        -- เช็คว่าเป้าหมายตายไหมหลังโจมตี
                                        if targetHumanoid.Health <= 0 then
                                            if retry then
                                                task.wait(1) -- รอ 1 วิ ก่อนหาเป้าหมายใหม่ในรอบถัดไป
                                            else
                                                loop = false -- หยุดถ้าไม่ให้ Retry
                                            end
                                            break -- ออกจากลูปโจมตีเป้าหมายนี้
                                        end
                                    else
                                         -- notify("Error", "Tool '".._G.name.."' not found.")
                                         -- อาจจะหยุด loop หรือ แจ้งเตือนถ้าหาเครื่องมือไม่เจอ
                                         -- loop = false
                                    end
                                end)
                                break -- โจมตีแค่คนเดียวต่อรอบ Heartbeat เพื่อลด Lag/โหลด
                            end
                        end
                    end
                end
            end
        end -- end for loop players
    end -- end while loop
    -- notify("KillAura", "Stopped.") -- แจ้งเตือนเมื่อหยุดทำงาน
end

-- UI Buttons
Window:Button("On", function()
    if not loop then
        loop = true
        retry = true
        -- notify("KillAura", "Started!")
        task.spawn(KillAura) -- รันใน task.spawn เพื่อไม่ให้ UI ค้าง
    end
end)

Window:Button("Off", function()
    if loop then
        loop = false
        retry = false
        -- notify("KillAura", "Stopping...")
    end
end)

-- UI Input Box (แก้ไขแล้ว)
Window:Box("Reach ("..reach..")", function(text, focuslost)
   if focuslost then
       local num = tonumber(text)
       if not num then
           reach = 10
           notify("Invalid Input", "Please enter a number. Reach reset to 10.")
       elseif num <= 10 then
           reach = 10
           notify("Minimum Reach", "The minimum reach is 10. Current reach: "..reach)
       -- ลบส่วนที่จำกัดค่าสูงสุด 40 ออกไปแล้ว
       else -- ถ้าเป็นตัวเลขมากกว่า 10
           reach = num
           notify("Reach Changed!", "Reach is now set to "..reach)
       end
       -- อัปเดต Label ของ Input Box (ถ้า UI library รองรับ)
       -- อาจจะต้องหาทางอัปเดต Text ของ Box ให้แสดงค่า reach ปัจจุบัน
   end
end)

-- UI Dropdown
Window:Dropdown("Mode", {"enemies only", "others"}, function(o)
    if o == "enemies only" then
        _G.enemOnly = true
        notify("Mode", "Set to Enemies Only")
    elseif o == "others" then
        _G.enemOnly = false
        notify("Mode", "Set to Attack Others")
    end
end)

-- UI Input Box สำหรับชื่อ Tool
Window:Box("Tool Name (".._G.name..")", function(text, focuslost)
    if focuslost then
        if text and text ~= "" then
             _G.name = text
             notify("Tool Name", "Set to: ".._G.name)
        else
             notify("Tool Name", "Name cannot be empty. Kept: ".._G.name)
        end
        -- อาจจะเพิ่มการอัปเดต Label ของ Box ด้วย
    end
end)

notify("KillAura (Modded)", "Script Loaded. Set Reach/Tool and press On.")
