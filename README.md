
local SGUI = game:GetService("StarterGui")
cloneref = cloneref or function(...) return ... end

local service = setmetatable({}, {
    __index = function(self, name)
        local s = game:GetService(name)
        rawset(self, name, cloneref(s))
        return rawget(self, name)
    end
})

if not game:IsLoaded() then
    game.Loaded:Wait()
end

local env = getgenv and getgenv() or _G
local SCRIPT_FLAG = '_AVATAR_CHANGER_LOADED_'

local SGUI = game:GetService("StarterGui")

if env[SCRIPT_FLAG] then
    return
end

cloneref = cloneref or function(...) return ... end

local service = setmetatable({}, {
    __index = function(self, name)
        rawset(self, name, cloneref(game:GetService(name)))
        return rawget(self, name)
    end
})

local Players = service.Players
local TweenService = service.TweenService
local HttpService = service.HttpService
local UserInputService = service.UserInputService
local RunService = service.RunService

local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild('PlayerGui')

local IS_MOBILE = UserInputService.TouchEnabled and not UserInputService.MouseEnabled

local Icons = {
    eye = 'rbxassetid://10723346959',
    minus = 'rbxassetid://10734896206',
    plus = 'rbxassetid://10734924532',
    search = 'rbxassetid://10734943674',
    play = 'rbxassetid://10734923549',
    dice = 'rbxassetid://10723343321',
    rotate = 'rbxassetid://10734940376',
    chevron_left = 'rbxassetid://10709791281',
    chevron_right = 'rbxassetid://10709791437',
    check = 'rbxassetid://10709790644',
    x = 'rbxassetid://10747384394',
    warning = 'rbxassetid://10709753149',
    info = 'rbxassetid://10723415903',
    heart = 'rbxassetid://10723406885',
    heart_off = 'rbxassetid://10723406662',
    home = 'rbxassetid://10723407389',
    users = 'rbxassetid://10747373426',
    user = 'rbxassetid://10747373176',
    settings = 'rbxassetid://10734950309',
    bookmark = 'rbxassetid://10709782154',
    refresh = 'rbxassetid://10734933222',
    stats = 'rbxassetid://10747372167',
    palette = 'rbxassetid://10734963400'
}

local Sounds = {
    ui_toggle = 'rbxassetid://124972635680154',
    reset = 'rbxassetid://123698506133442',
    dice = 'rbxassetid://500839268',
    tab_switch = 'rbxassetid://9065073444',
    button_click = 'rbxassetid://12948391899',
    favorite = 'rbxassetid://74914703480819',
    unfavorite = 'rbxassetid://92708987611847'
}

local State = {
    applying = false,
    auto = false,
    undo_history = {},
    redo_history = {},
    max_history = 10,
    applied_id = nil,
    minimized = false,
    visible = true,
    preview_open = false,
    client_preview_open = false,
    cache = {},
    randoms = {},
    conns = {},
    original_desc = nil,
    input_ready = false,
    random_cooldown = false,
    fab_toggle_cooldown = false,
    visibility_animating = false,
    favorites = {},
    favorites_index = {},
    current_tab = 'main',
    fav_name_cache = {},
    width_scale = 1,
    width_enabled = false,
    client_applying = false,
    player_original_descs = {},
    fav_search_text = '',
    fav_ui_initialized = false,
    zoom_level = 5.5,
    client_zoom_level = 5.5,
    avatar_details = {},
    height_scale = 1,
    height_enabled = false,
    depth_scale = 1,
    depth_enabled = false,
    head_scale = 1,
    head_enabled = false,
    music_playing = false,
    music_index = 1,
    music_sound = nil,
    music_loop = false,
    music_volume = 1.0,
    music_dynamic_volume = false,
    bass_boost_enabled = false,
    sound_effects_enabled = true,
    apply_counts = {},
    stats_apply_cooldown = false,
    rgb_mode = false,
    rgb_hue = 0,
    rgb_mode_type = 'cycle',
    rgb_saturation = 1,
    rgb_brightness = 1,
    rgb_speed = 1,
    rgb_time = 0,
    rgb_element_delays = {},
    rgb_intro_playing = false,
    minimize_cooldown = false,
    current_theme = 'purple',
    esp_enabled = false,
    esp_teamcheck = false,
    esp_highlights = true,
    esp_show_only_highlights = false,
    esp_show_teamname = true,
    esp_update_rate = 0.01,
    esp_last_update = 0,
    esp_priority_update_rate = 0.005,
    esp_low_priority_update_rate = 0.016,
    toggle_keybind = Enum.KeyCode.LeftControl,
    last_apply_time = 0,
    apply_cooldown = 0.25,
    notifications_enabled = true,
    music_loaded = false,
    presets_loaded = false
}

local SoundManager = {}
SoundManager.pool = {}
SoundManager.pool_size = 10
SoundManager.active_sounds = {}
SoundManager.last_play_time = {}

function SoundManager.init()
    for i = 1, SoundManager.pool_size do
        local sound = Instance.new('Sound')
        sound.Volume = 0.3
        sound.Parent = PlayerGui
        table.insert(SoundManager.pool, sound)
    end
end

function SoundManager.get_sound()
    for i, sound in ipairs(SoundManager.pool) do
        if not sound.IsPlaying then
            return sound
        end
    end
    
    local sound = Instance.new('Sound')
    sound.Volume = 0.3
    sound.Parent = PlayerGui
    table.insert(SoundManager.pool, sound)
    return sound
end

function SoundManager.play(sound_id, volume)
    if not State.sound_effects_enabled then return end
    
    local current_time = tick()
    local last_time = SoundManager.last_play_time[sound_id] or 0
    
    SoundManager.last_play_time[sound_id] = current_time
    
    local sound = SoundManager.get_sound()
    sound.SoundId = sound_id
    sound.Volume = volume or 0.3
    
    sound:Play()
    
    SoundManager.active_sounds[sound] = true
    
    local connection
    connection = sound.Ended:Connect(function()
    SoundManager.active_sounds[sound] = nil
    connection:Disconnect()
    end)
    
    return sound
end

function SoundManager.play_click()
    return SoundManager.play(Sounds.button_click, 0.35)
end

function SoundManager.play_toggle()
    return SoundManager.play(Sounds.ui_toggle, 0.2)
end

function SoundManager.play_tab_switch()
    return SoundManager.play(Sounds.tab_switch, 0.5)
end

function SoundManager.play_favorite()
    return SoundManager.play(Sounds.favorite, 0.2)
end

function SoundManager.play_unfavorite()
    return SoundManager.play(Sounds.unfavorite, 0.15)
end

function SoundManager.play_dice()
    return SoundManager.play(Sounds.dice, 0.2)
end

function SoundManager.play_reset()
    return SoundManager.play(Sounds.reset, 0.2)
end

function SoundManager.cleanup()
    for _, sound in ipairs(SoundManager.pool) do
        sound:Stop()
        sound:Destroy()
    end
    SoundManager.pool = {}
    SoundManager.active_sounds = {}
end

SoundManager.init()

local Maid = {}
Maid.__index = Maid

function Maid.new()
    return setmetatable({
        _tasks = {}
    }, Maid)
end

function Maid:Add(task, cleanup_method)
    if not task then return end
    
    table.insert(self._tasks, {
        task = task,
        method = cleanup_method
    })
    
    return task
end

function Maid:AddConnection(connection)
    return self:Add(connection, 'Disconnect')
end

function Maid:AddInstance(instance)
    return self:Add(instance, 'Destroy')
end

function Maid:Cleanup()
    for _, task_data in ipairs(self._tasks) do
        local task = task_data.task
        local method = task_data.method
        
        if task then
            if method and type(task[method]) == 'function' then
                pcall(function()
                    task[method](task)
                end)
            elseif type(task) == 'function' then
                pcall(task)
            end
        end
    end
    
    self._tasks = {}
end

function Maid:Destroy()
    self:Cleanup()
end

local Music = {
    {name = 'Merry Christmas', id = 'rbxassetid://1838667168'},
    {name = 'TacoBot-3000', id = 'rbxassetid://9245552700'},
    {name = 'Montagem Blue Shirt', id = 'rbxassetid://122203857226173'},
    {name = 'CHAOS', id = 'rbxassetid://1843497734'},
    {name = 'SAVAGE GHOST SLOWED', id = 'rbxassetid://72811197789142'},
    {name = 'MONTAGEM AURA', id = 'rbxassetid://117759015860393'},
    {name = 'GOOD GIRL FUNK', id = 'rbxassetid://123723674864058'},
    {name = 'FUNK FESTA', id = 'rbxassetid://103409297553965'},
    {name = 'I Still Think of You', id = 'rbxassetid://86766967120839'},
    {name = 'Shattered Reflections', id = 'rbxassetid://127812909272456'},
    {name = 'Kakusei Erwachen', id = 'rbxassetid://124853612881772'},
    {name = 'Turkish Art', id = 'rbxassetid://1842150151'},
    {name = 'Scary Beats For You', id = 'rbxassetid://88456225545062'},
    {name = 'The Lonely Guy', id = 'rbxassetid://114213622974713'},
    {name = 'TOMA FUNK PHONK', id = 'rbxassetid://129098116998483'},
    {name = 'Raining Tacos', id = 'rbxassetid://142376088'},
    {name = 'Happy Song', id = 'rbxassetid://1843404009'},
    {name = 'The Cult', id = 'rbxassetid://98183757946208'},
    {name = 'Windows To The Sun', id = 'rbxassetid://119384335997294'},
   {name = 'Heavenly Song', id = 'rbxassetid://9125603749'},
    {name = 'SIGMA BOY', id = 'rbxassetid://109420952262721'},
    {name = 'Evangelion Sword Outro', id = 'rbxassetid://102468604705752'},
    {name = 'Christmas Anime Opening', id = 'rbxassetid://112881604345924'},
    {name = 'Kieta Hoshi', id = 'rbxassetid://88704298134223'},
    {name = 'Rise to the Horizon', id = 'rbxassetid://72573266268313'},
    {name = 'Fading Memories', id = 'rbxassetid://87065827699369'},
    {name = 'Backrooms', id = 'rbxassetid://120817494107898'},
    {name = 'Kimi no Kodou', id = 'rbxassetid://127296569646417'},
    {name = 'Moeru Kodou', id = 'rbxassetid://138614247392811'},
    {name = 'Kienai Koe', id = 'rbxassetid://90588286632687'},
    {name = 'Saigo no Yakusoku', id = 'rbxassetid://87102415343081'},
    {name = 'Menta Ma', id = 'rbxassetid://98337901681441'},
    {name = 'Megavalanio Remix', id = 'rbxassetid://101598365847186'},
    {name = 'Tsuko G - Deja Vu', id = 'rbxassetid://16831106636'},
    {name = 'Jumpstyle', id = 'rbxassetid://1839246711'},
    {name = 'Total Confusion', id = 'rbxassetid://103419239604004'},
    {name = 'Run Away', id = 'rbxassetid://128118999630439'},
    {name = 'I Have No Enemies', id = 'rbxassetid://109550779401749'},
    {name = 'Hold On (Speed Up)', id = 'rbxassetid://71045969776776'},
    {name = 'United No Borders', id = 'rbxassetid://134841562378374'},
    {name = 'Dear Lana', id = 'rbxassetid://119589412825080'},
    {name = 'Woo Woo Woo Woo', id = 'rbxassetid://77139878722989'},
    {name = 'Fur Elise', id = 'rbxassetid://1836283178'},
    {name = 'Clair De Lune', id = 'rbxassetid://1838457617'},
    {name = 'Beauty PHONK', id = 'rbxassetid://115249562236391'},
    {name = 'Slava Bass Boosted', id = 'rbxassetid://117237954599243'},
    {name = 'BRAZIL FUNK', id = 'rbxassetid://133498554139200'},
    {name = 'CREPPY FUNK', id = 'rbxassetid://110170687361544'},
    {name = 'ORAY BEY FUNK', id = 'rbxassetid://135286582331883'},
    {name = 'Marginal Phonk', id = 'rbxassetid://95177104398382'},
    {name = 'Recognized', id = 'rbxassetid://86255275876700'},
    {name = 'Montagem', id = 'rbxassetid://88913864169288'},
    {name = 'Nocturne', id = 'rbxassetid://129108903964685'},
    {name = 'Hide & Seek', id = 'rbxassetid://131571387429103'},
    {name = 'キラメキ∞ループ', id = 'rbxassetid://108849060438649'},
    {name = 'Banana Bashin', id = 'rbxassetid://118231802185865'}
}

local Themes = {
    purple = {
        accent = Color3.fromRGB(99, 102, 241),
        accent_glow = Color3.fromRGB(129, 132, 255)
    },
    blue = {
        accent = Color3.fromRGB(59, 130, 246),
        accent_glow = Color3.fromRGB(96, 165, 250)
    },
    red = {
        accent = Color3.fromRGB(239, 68, 68),
        accent_glow = Color3.fromRGB(248, 113, 113)
    },
    green = {
        accent = Color3.fromRGB(34, 197, 94),
        accent_glow = Color3.fromRGB(74, 222, 128)
    },
    pink = {
        accent = Color3.fromRGB(236, 72, 153),
        accent_glow = Color3.fromRGB(244, 114, 182)
    },
    cyan = {
        accent = Color3.fromRGB(6, 182, 212),
        accent_glow = Color3.fromRGB(34, 211, 238)
    }
}

local C = {
    base = Color3.fromRGB(12, 12, 16),
    panel = Color3.fromRGB(18, 18, 24),
    card = Color3.fromRGB(26, 26, 34),
    elevated = Color3.fromRGB(36, 36, 46),
    hover = Color3.fromRGB(46, 46, 58),
    border = Color3.fromRGB(55, 55, 70),
    text = Color3.fromRGB(255, 255, 255),
    subtext = Color3.fromRGB(180, 180, 195),
    muted = Color3.fromRGB(100, 100, 120),
    accent = Color3.fromRGB(99, 102, 241),
    accent_glow = Color3.fromRGB(129, 132, 255),
    success = Color3.fromRGB(34, 197, 94),
    error = Color3.fromRGB(239, 68, 68),
    warning = Color3.fromRGB(245, 158, 11),
    star = Color3.fromRGB(255, 200, 50)
}

local function apply_theme(theme_name)
    local theme = Themes[theme_name]
    if theme then
        C.accent = theme.accent
        C.accent_glow = theme.accent_glow
    end
end

local TRANSPARENCY = 0.22

local Presets = {
   289438135, 1707711223, 188732, 2298753899, 9119588309, 5254879171, 8595350470, 6007609888, 124751865, 5019714978, 5007631110, 9088628683, 7223875998, 2474943274, 3104949425, 3335871296, 203030608, 2596305840, 201124389, 1981724228, 3731169417, 205419201, 7422492329, 406436524, 1803380, 9406742928, 1359861204, 3012958642, 2260118449, 188829949, 2261820401, 8094705681, 9894023718, 6077615334, 2281971469, 1946404863, 660132420, 1125262365, 3018607207, 144018186, 3577671250, 2017401176, 3473976672, 9122248242, 1667867130, 9294642379, 5366504429, 8264800124, 283156132, 1630540916, 4416918097, 344091683, 6538096, 7623744992, 1099702304, 1199088309, 1369842558, 3624257547, 145740081, 215710487, 2255861564, 7330109199, 524749295, 272574783, 4100936320, 4863227235, 1132340350, 5210946332, 3331434198, 2618555079, 4201687597, 147198435, 704071723, 465771760, 254829155, 8069027498, 2646550793, 366768658, 2885260147
}

local ContentLoader = {}
ContentLoader.loaded_music = {}
ContentLoader.loaded_presets = {}

function ContentLoader.load_music_progressive(callback, progress_callback)
    if State.music_loaded then 
        if callback then callback() end
        return 
    end
    
    local batch_size = 5
    local delay_between_batches = 0.2
    
    task.spawn(function()
        for i = 1, #Music, batch_size do
            local batch_end = math.min(i + batch_size - 1, #Music)
            
            for j = i, batch_end do
                table.insert(ContentLoader.loaded_music, Music[j])
            end
            
            if progress_callback then
                local progress = math.floor((#ContentLoader.loaded_music / #Music) * 100)
                progress_callback(progress)
            end
            
            if i + batch_size <= #Music then
                task.wait(delay_between_batches)
            end
        end
        
        State.music_loaded = true
        if callback then callback() end
    end)
end

function ContentLoader.load_presets_progressive(callback, progress_callback)
    if State.presets_loaded then 
        if callback then callback() end
        return 
    end
    
    local batch_size = 10
    local delay_between_batches = 0.15
    
    task.spawn(function()
        for i = 1, #Presets, batch_size do
            local batch_end = math.min(i + batch_size - 1, #Presets)
            
            for j = i, batch_end do
                table.insert(ContentLoader.loaded_presets, Presets[j])
            end
            
            if progress_callback then
                local progress = math.floor((#ContentLoader.loaded_presets / #Presets) * 100)
                progress_callback(progress)
            end
            
            if i + batch_size <= #Presets then
                task.wait(delay_between_batches)
            end
        end
        
        State.presets_loaded = true
        if callback then callback() end
    end)
end

function ContentLoader.get_music()
    return State.music_loaded and Music or ContentLoader.loaded_music
end

function ContentLoader.get_presets()
    return State.presets_loaded and Presets or ContentLoader.loaded_presets
end

local StateProtection = {}
StateProtection.__index = StateProtection

function StateProtection.new(state_table)
    local self = setmetatable({}, StateProtection)
    self._state = state_table
    self._locked_keys = {
        favorites = true,
        favorites_index = true,
        cache = true,
        conns = true
    }
    return self
end

function StateProtection:is_locked(key)
    return self._locked_keys[key] == true
end

function StateProtection:get(key)
    return self._state[key]
end

function StateProtection:set(key, value)
    if self:is_locked(key) then
        warn('[StateProtection] Attempted To Modify Protected Key:', key)
        return false
    end
    self._state[key] = value
    return true
end

function StateProtection:add_favorite(user_id)
    if not user_id then return false end
    if not self._state.favorites_index[user_id] then
        local fav_data = {id = user_id, name = tostring(user_id), time = os.time()}
        table.insert(self._state.favorites, fav_data)
        self._state.favorites_index[user_id] = true
        return true
    end
    return false
end

function StateProtection:remove_favorite(user_id)
    if not self._state.favorites_index[user_id] then
        return false
    end
    
    for i, fav in ipairs(self._state.favorites) do
        if fav.id == user_id then
            table.remove(self._state.favorites, i)
            self._state.favorites_index[user_id] = nil
            return true
        end
    end
    return false
end

function StateProtection:is_favorite(user_id)
    return self._state.favorites_index[user_id] == true
end

function StateProtection:get_favorites()
    local copy = {}
    for i, v in ipairs(self._state.favorites) do
        copy[i] = v
    end
    return copy
end

local function get_char_and_hum(player_or_char)
    local char = player_or_char
    if typeof(player_or_char) == "Instance" and player_or_char:IsA("Player") then
        char = player_or_char.Character
    end
    
    if not char or not char:FindFirstChild('HumanoidRootPart') then
        return nil, nil
    end
    
    local hum = char:FindFirstChildOfClass('Humanoid')
    return char, hum
end

local function apply_description(humanoid, description)
    pcall(function()
        if humanoid.ApplyDescriptionClientServer then
            humanoid:ApplyDescriptionClientServer(description)
        else
            humanoid:ApplyDescription(description)
        end
    end)
end

local function is_click_input(input)
    return input.UserInputType == Enum.UserInputType.MouseButton1 or 
           input.UserInputType == Enum.UserInputType.Touch
end

local function find_player_by_name(playerName)
    local lowerName = playerName:lower()
    for _, p in ipairs(Players:GetPlayers()) do
        if p.Name:lower() == lowerName or p.DisplayName:lower() == lowerName then
            return p
        end
    end
    return nil
end

local function collect_tools(char)
    local tools = {}
    
    for _, c in ipairs(char:GetChildren()) do
        if c:IsA('Tool') then
            table.insert(tools, c)
        end
    end
    
    local backpack = char.Parent and char.Parent:FindFirstChild('Backpack')
    if backpack then
        for _, t in ipairs(backpack:GetChildren()) do
            if t:IsA('Tool') then
                table.insert(tools, t)
            end
        end
    end
    
    return tools, backpack
end

local function restore_tools(tools, backpack, char)
    for _, t in ipairs(tools) do
        if t and t.Parent == nil then
            if backpack then
                t.Parent = backpack
            else
                t.Parent = char
            end
        end
    end
end

local function client_apply_guard(operation_func)
    if State.client_applying then
        return
    end
    State.client_applying = true
    
    task.spawn(function()
        operation_func()
        State.client_applying = false
    end)
end

local function apply_history_state(history, target_history, is_undo)
    if #history == 0 then
        Notify.show(is_undo and 'Nothing To Undo' or 'Nothing To Redo', 'warning')
        return
    end
    
    if State.applying then
        return
    end
    
    local target_id = table.remove(history)
    
    if State.applied_id then
        table.insert(target_history, State.applied_id)
        if #target_history > State.max_history then
            table.remove(target_history, 1)
        end
    end
    
    Av.apply(target_id, nil, true, true)
    
    if E.update_undo_redo_ui then
        E.update_undo_redo_ui()
    end
    
    if E.input then
        E.input.Text = tostring(target_id)
    end
    
    save_data()
    play_sound(Sounds.reset, 0.1)
end

local Janitor = {}
Janitor.__index = Janitor

function Janitor.new()
    local self = setmetatable({}, Janitor)
    self._objects = {}
    return self
end

function Janitor:Add(object, cleanup_method)
    if not object then return end
    
    local index = #self._objects + 1
    self._objects[index] = {
        object = object,
        method = cleanup_method or (typeof(object) == 'RBXScriptConnection' and 'Disconnect' or 'Destroy')
    }
    
    return object
end

function Janitor:AddMultiple(objects, cleanup_method)
    for _, obj in ipairs(objects) do
        self:Add(obj, cleanup_method)
    end
end

function Janitor:LinkToJanitor(other_janitor)
    return self:Add(other_janitor, 'Cleanup')
end

function Janitor:Remove(object)
    for i, data in ipairs(self._objects) do
        if data.object == object then
            self:_cleanup_object(data)
            table.remove(self._objects, i)
            return true
        end
    end
    return false
end

function Janitor:_cleanup_object(data)
    if not data or not data.object then return end
    
    local success, err = pcall(function()
        if type(data.object[data.method]) == 'function' then
            data.object[data.method](data.object)
        end
    end)
    
    if not success and err then
        warn('[Janitor] Cleanup Error:', err)
    end
end

function Janitor:Cleanup()
    for i = #self._objects, 1, -1 do
        self:_cleanup_object(self._objects[i])
        self._objects[i] = nil
    end
end

function Janitor:Destroy()
    self:Cleanup()
end

function Janitor:GetCount()
    return #self._objects
end

local ScriptJanitor = Janitor.new()

local ESPSystem = {}
ESPSystem.Enabled = false
ESPSystem.TeamCheck = false
ESPSystem.ESPs = {}
ESPSystem.Config = {
    Chams = {
        Enabled = true,
        FillColor = Color3.fromRGB(255, 0, 0),
        FillTransparency = 0.5,
        OutlineColor = Color3.fromRGB(255, 255, 255),
        OutlineTransparency = 0
    }
}

ESPSystem.WeaponBlacklist = {
    ["Lunch"] = true,
    ["Lunchbox"] = true,
    ["Lunch Box"] = true,
    ["Combat"] = true,
    ["Assist"] = true,
    ["Camera"] = true,
    ["Binoculars"] = true,
    ["Flashlight"] = true,
    ["Phone"] = true,
    ["Radio"] = true,
    ["Tablet"] = true,
    ["Map"] = true,
    ["Compass"] = true,
    ["Watch"] = true,
    ["Medkit"] = true,
    ["Bandage"] = true,
    ["Pills"] = true,
    ["Syringe"] = true,
    ["Food"] = true,
    ["Drink"] = true,
    ["Water"] = true,
    ["Soda"] = true,
    ["Energy Drink"] = true,
    ["Snack"] = true,
    ["Candy"] = true,
    ["Chips"] = true,
    ["Burger"] = true,
    ["Pizza"] = true,
    ["Sandwich"] = true,
    ["Apple"] = true,
    ["Banana"] = true,
    ["Orange"] = true,
    ["Bread"] = true,
    ["Key"] = true,
    ["Keycard"] = true,
    ["ID Card"] = true,
    ["Badge"] = true,
    ["Pass"] = true,
    ["Ticket"] = true,
    ["Document"] = true,
    ["Folder"] = true,
    ["Notepad"] = true,
    ["Notebook"] = true,
    ["Book"] = true,
    ["Screwdriver"] = true,
    ["Toolbox"] = true,
    ["Pickaxe"] = true,
    ["Hammer"] = true,
    ["Dinner"] = true,
    ["Shovel"] = true,
    ["Hatchet"] = true,
    ["Paint"] = true,
    ["Spray"] = true,
    ["Rope"] = true,
    ["Net"] = true,
    ["Trap"] = true,
    ["Lockpick"] = true,
    ["Handcuffs"] = true,
    ["Zipties"] = true,
    ["Stun Gun"] = true,
    ["Pepper Spray"] = true,
    ["Baton"] = true,
    ["Crude Knife"] = true,
    ["Glowstick"] = true,
    ["VIP"] = true,
    ["Popcorn"] = true,
    ["Nightstick"] = true
}

local function is_weapon_blacklisted(weapon_name)
    if not weapon_name then return true end
    
    local lower_name = weapon_name:lower()
    
    for blacklisted, _ in pairs(ESPSystem.WeaponBlacklist) do
        if lower_name:find(blacklisted:lower()) then
            return true
        end
    end
    
    return false
end

local function create_esp_for_player(player)
    if player == LocalPlayer then return end
    
    local esp_data = {
        Player = player,
        Highlight = nil,
        Drawings = {},
        Connections = {}
    }
    
    local function create_drawings()
        esp_data.Drawings = {
            Box = Drawing.new("Square"),
            NameText = Drawing.new("Text"),
            DistanceText = Drawing.new("Text"),
            TeamText = Drawing.new("Text"),
            HealthText = Drawing.new("Text"),
            WeaponText = Drawing.new("Text"),
            HealthBar = Drawing.new("Line"),
            HealthBarOutline = Drawing.new("Line")
        }
        
        esp_data.Drawings.Box.Color = Color3.fromRGB(255, 255, 255)
        esp_data.Drawings.Box.Thickness = 1.5
        esp_data.Drawings.Box.Filled = false
        esp_data.Drawings.Box.Visible = false
        esp_data.Drawings.Box.ZIndex = 2
        
        esp_data.Drawings.NameText.Color = Color3.fromRGB(255, 255, 255)
        esp_data.Drawings.NameText.Size = 11.5
        esp_data.Drawings.NameText.Center = false
        esp_data.Drawings.NameText.Outline = true
        esp_data.Drawings.NameText.Visible = false
        esp_data.Drawings.NameText.Text = player.Name
        esp_data.Drawings.NameText.ZIndex = 3
        
        esp_data.Drawings.DistanceText.Color = Color3.fromRGB(150, 150, 150)
        esp_data.Drawings.DistanceText.Size = 10
        esp_data.Drawings.DistanceText.Center = false
        esp_data.Drawings.DistanceText.Outline = true
        esp_data.Drawings.DistanceText.Visible = false
        esp_data.Drawings.DistanceText.Text = ""
        esp_data.Drawings.DistanceText.ZIndex = 3
        
        esp_data.Drawings.TeamText.Color = Color3.fromRGB(0, 255, 255)
        esp_data.Drawings.TeamText.Size = 12
        esp_data.Drawings.TeamText.Center = true
        esp_data.Drawings.TeamText.Outline = true
        esp_data.Drawings.TeamText.Visible = false
        esp_data.Drawings.TeamText.ZIndex = 3
        
        esp_data.Drawings.HealthText.Color = Color3.fromRGB(255, 255, 255)
        esp_data.Drawings.HealthText.Size = 11
        esp_data.Drawings.HealthText.Center = false
        esp_data.Drawings.HealthText.Outline = true
        esp_data.Drawings.HealthText.Visible = false
        esp_data.Drawings.HealthText.Text = ""
        esp_data.Drawings.HealthText.ZIndex = 3
        
        esp_data.Drawings.WeaponText.Color = Color3.fromRGB(255, 255, 255)
        esp_data.Drawings.WeaponText.Size = 12
        esp_data.Drawings.WeaponText.Center = true
        esp_data.Drawings.WeaponText.Outline = true
        esp_data.Drawings.WeaponText.Visible = false
        esp_data.Drawings.WeaponText.Text = ""
        esp_data.Drawings.WeaponText.ZIndex = 3
        
        esp_data.Drawings.HealthBar.Color = Color3.fromRGB(0, 255, 0)
        esp_data.Drawings.HealthBar.Thickness = 25
        esp_data.Drawings.HealthBar.Visible = false
        esp_data.Drawings.HealthBar.ZIndex = 2
        
        esp_data.Drawings.HealthBarOutline.Color = Color3.fromRGB(0, 0, 0)
        esp_data.Drawings.HealthBarOutline.Thickness = 35
        esp_data.Drawings.HealthBarOutline.Visible = false
        esp_data.Drawings.HealthBarOutline.ZIndex = 1
    end
    
    local function update_esp()
        if not player.Character then
            for _, drawing in pairs(esp_data.Drawings) do
                drawing.Visible = false
            end
            return
        end
        
        local character = player.Character
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        local root = character:FindFirstChild("HumanoidRootPart")
        
        if not humanoid or not root or humanoid.Health <= 0 then
            for _, drawing in pairs(esp_data.Drawings) do
                drawing.Visible = false
            end
            if esp_data.Highlight then
                esp_data.Highlight.Enabled = false
            end
            return
        end
        
        if not esp_data.Highlight and State.esp_highlights then
            local highlight = Instance.new("Highlight")
            highlight.Name = "ESP_Highlight"
            highlight.Adornee = character
            
            local teamColor = Color3.fromRGB(255, 255, 255)
            if player.Team then
                if player.Team.TeamColor then
                    teamColor = player.Team.TeamColor.Color
                elseif player.TeamColor then
                    teamColor = player.TeamColor.Color
                end
            end
            
            highlight.FillColor = teamColor
            highlight.FillTransparency = 0.5
            highlight.OutlineColor = teamColor
            highlight.OutlineTransparency = 0
            highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
            highlight.Parent = character
            esp_data.Highlight = highlight
        end
        
        if esp_data.Highlight then
            if State.esp_highlights then
                local teamColor = Color3.fromRGB(255, 255, 255)
                if player.Team then
                    if player.Team.TeamColor then
                        teamColor = player.Team.TeamColor.Color
                    elseif player.TeamColor then
                        teamColor = player.TeamColor.Color
                    end
                end
                esp_data.Highlight.FillColor = teamColor
                esp_data.Highlight.OutlineColor = teamColor
                esp_data.Highlight.Enabled = true
            else
                esp_data.Highlight.Enabled = false
            end
        end
        
        if ESPSystem.TeamCheck and player.Team == LocalPlayer.Team then
            if esp_data.Highlight then
                esp_data.Highlight.Enabled = false
            end
            for _, drawing in pairs(esp_data.Drawings) do
                drawing.Visible = false
            end
            return
        else
            if esp_data.Highlight then
                esp_data.Highlight.Enabled = true
            end
        end
        
        local camera = workspace.CurrentCamera
        local rootPos, onScreen = camera:WorldToViewportPoint(root.Position)
        
        if not onScreen then
            for _, drawing in pairs(esp_data.Drawings) do
                drawing.Visible = false
            end
            return
        end
        
        local head = character:FindFirstChild("Head")
        if not head then
            for _, drawing in pairs(esp_data.Drawings) do
                drawing.Visible = false
            end
            return
        end
        
        local headPos = camera:WorldToViewportPoint(head.Position + Vector3.new(0, 0.5, 0))
        local legPos = camera:WorldToViewportPoint(root.Position - Vector3.new(0, 3, 0))
        
        local height = math.abs(headPos.Y - legPos.Y)
        local width = height / 1.5
        
        esp_data.Drawings.Box.Size = Vector2.new(width, height)
        esp_data.Drawings.Box.Position = Vector2.new(rootPos.X - width / 2, headPos.Y)
        esp_data.Drawings.Box.Visible = not State.esp_show_only_highlights
        
        esp_data.Drawings.NameText.Text = player.Name
        esp_data.Drawings.NameText.Visible = not State.esp_show_only_highlights
        
        local my_char = LocalPlayer.Character
        local my_root = my_char and my_char:FindFirstChild("HumanoidRootPart")
        if my_root then
            local distance = math.floor((my_root.Position - root.Position).Magnitude)
            
            local nameWidth = #player.Name * 4.25
      
