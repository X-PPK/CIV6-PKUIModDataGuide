# PKUI本地数据框架使用指南
## 一. 文档信息
- **作者：皮皮凯(PiPiKai)**
- **时间：2025.1.20**
---
本文档旨在阐述《文明6》中皮凯UI框架mod下的“mod本地数据存储子框架”的使用方法。该框架允许每个mod拥有独立的持久化数据存储，这些数据能够跨游戏对局保存，存储于玩家的本地电脑中。以下提供的详细使用指南旨在帮助那些决定采用此框架的mod开发者们，能够轻松掌握并有效地运用这一工具。  
以下内容中提到的“框架”，均指该子框架部分。

## 二. 框架设计概述
### 1. 数据存储原理
- **集中存储**：框架采用集中式数据存储机制，所有mod数据均存储在一个“配置存档”中。游戏启动并进入主菜单时，框架会自动读取该“配置存档”中的数据。
- **数据同步**：读取的数据会被同步至相应的Lua脚本变量，确保游戏中的数据状态是最新的。同时，框架会生成LuaEvents和ExposedMembers等API，以便mod开发者能够便捷地在其他lua脚本中进行数据操作。
### 2. 设计缘由
- **避免冲突**：若每个mod各自使用独立的“配置存档”，则不仅会增加数据读取时间，还可能在加载时发生冲突，因为同一时间只能加载一个配置存档。
- **统一管理机制**：为了避免上述问题，框架采用统一的管理机制和存储来处理所有mod的数据，确保数据的一致性和系统的稳定性。
### 3. 统一管理优势
- **一致性保障**：通过框架统一管理，可以保持数据的一致性，避免因数据分散而导致的混乱。
- **规范化约束**：这种做法不仅是一种规范化的约束，也有助于mod开发者更有效地管理本地数据，减少不同mod之间的相互干扰。
### 4.额外补充
- **框架lua环境**:框架是跨越游戏前端环境和InGame环境的，而前端是没有GameLua环境的，因此框架的lua环境是UILua环境,同时mod开发者要注意数据在前端和InGame环境的不同处理程序
- **我的期望**：框架提供很多便利的API，旨在为mod开发者提供更广阔的创作空间和开发效率，同时也为玩家带来更加愉悦的游戏体验。
- **开放反馈**：我们欢迎任何建议和反馈，以不断优化框架，使其更好地服务于mod开发者和玩家社区。  
- **PS**：我会在本指南记中尽量使用通俗易懂的语言来讲解，因此在专业术语上可能会有所欠缺，我会尽可能并提供一些示例代码，供大家参考。  
会涉及到lua的setmetatable定义的Lua表，在Lua中，虽然从严格的编程语言理论角度来说，使用setmetatable定义的表不能被称为“类”，因为Lua采用的是基于原型的继承模型而非传统的类继承模型。  
然而，为了在Lua中实现类似面向对象编程中的“类”功能，setmetatable是主要的机制。  
因此我个人更习惯叫它'类'或者'伪类',在后续指南中我也会这样称呼它。

最后通过本指南的说明，我希望能够帮助mod开发者和玩家更好地理解框架的设计意图和使用规范，从而更加高效和安全地使用该框架。

## 三.框架结构概述
本框架被划分为两个主要子部分，以满足不同类型的数据存储需求：
### 1. Mod配置数据管理
- **功能定位**：这部分专注于管理mod的设置和配置数据，这些数据是mod开发者提供给玩家的，允许玩家根据个人偏好进行自定义。这些配置数据通常涉及游戏界面、控制选项、等可调整的参数。
- **API支持**：框架提供了全面的API支持，简化了配置数据的处理过程。mod开发者只需要提供设置的键（key）和默认值以及发生更改时使用的回调函数。而无需设定对应UI和数据更改/保存逻辑，这些框架会已经帮你处理好。
- **数据更改/保存逻辑**：当玩家通过UI更改设置时，框架自动处理数据的更新和保存，确保玩家自定义的设置能够即时生效并持久保存。
### 2. Mod非配置数据管理
- **功能定位**：这部分负责处理mod的非配置数据，即那些不直接暴露给玩家设置的数据。
- **API提供**：框架同样提供了设置默认值、更改数据和保存数据的API，但与配置数据不同的是，这些数据的具体操作需要mod开发者自行定义和实现。
- **自定义操作**：开发者可以根据mod的具体需求，定义数据的存储、更改等操作。例如，可以用于记录领袖的挑战进度、实现跨存档的能力解锁等，为开发者提供了极大的灵活性和创造空间。
通过这种结构设计，框架既能够满足玩家对配置数据自定义的需求，又能够为mod开发者提供处理复杂游戏状态的强大工具，从而共同提升游戏体验和扩展游戏的可玩性。


## 四.框架的使用
- 这里我的讲述应该会不是很规整，想到一点整理一点，后续或许会慢慢完善。
### 1. 框架引入
首先，确保您的模组（mod）向框架注册，游戏数据库中填写ModDataIds表，这是必要的，因为：
- **避免ID冲突**：因为如果在lua直接添加对应的这些参数，一旦不同mod使用相同ID会导致mod互相干扰问题，所以在SQLite数据库中实施ModId和DataId的唯一性约束，可以在数据插入阶段就进行初步的ID检查，从而成功避免潜在的冲突。

```Sql
-- Sql 演示
INSERT INTO ModDataIds (ModId, DataId, Version) VALUES
("7b7bda2b-f6e2-45d2-ba05-bbeb4bd9c46a", "PK_Test", 1);
```
- 当然你也可以选择对应的xml来填表
```Xml
<!-- Xml 演示 -->
<ModDataIds>
    <Row ModId="7b7bda2b-f6e2-45d2-ba05-bbeb4bd9c46a" DataId="PK_Test" Version="1" />
</ModDataIds>
```
> <details>
> <summary>ModDataIds表的字段说明</summary>
> 
> 列名 | 说明
> --- | ---
> ModId | 这是你的mod的唯一标识，即你在modinfo文件中使用的Mod id。
> DataId | 本地数据ID，用于在lua中操作数据。<br>**注意**：DataId应当只包含字母数字和下划线<br>因为框架会根据这个ID> 定义对应的LuaEvents，以便在lua中进行数据操作，具体使用方法将在后面介绍。
> Version | 代表的是mod数据的版本号，而非mod本身的版本。<br>当您的mod需要进行数据更新时，必须更新这个版本号。<br>框架会根据这个版本号来决定是否应该覆盖现有的数据。<br>在默认情况下，框架会继续使用之前存储的数据来填充Lua变量，只有在版本号发生变化时，才会进行数据覆盖。<br>Lua还提供了一套接口，用于在版本更新时根据旧数据执行必要的更新操作。关于如何使用这些接口的详细说明，将在后续章节中提供。
> </details>

### 2. 框架lua使用前言
- 框架的数据直接使用环境是在游戏的UI-lua环境中，因此需要使用UIlua来操作数据
- > 关于框架Mod数据保存:  
如果是属于'配置数据'（既使用几个"Add...ParamUI"API注册过的key的对应数据）可以放心，当玩家对这些数据进行更改时，框架会自动保存更改后的数据。  
如果是属于'非配置数据'（既没有使用几个"Add...ParamUI"API注册过的key的对应数据）那么是分情况的：  
  > - 如果是前端环境，需要在自己Mod所有数据都更改完成后，手动调用保存数据的API。  
  > - 而在InGame环境，每当保存游戏存档发生时，框架都会自动保存当前所有mod的数据（当然也会留有主动保存的API）。
  >
  > 由于数据存储设是保存在一个配置存档中，每次保存时都需要将所有mod数据写入这个配置文件。如果频繁地进行保存操作，很可能会造成性能的大量消耗，并干扰玩家的正常游戏体验。因此，减少数据的频繁保存是必要的，采取了这样的设计策略。  
- 关于如何在你的lua脚本中使用框架的数据呢？有多种方式
- 一些参数的说明：
  - modUUID：mod的唯一标识，既你的Mod的modinfo文件中的ModId
  - modKey：mod的本地数据存储key，是由modUUID生成的唯一对应的字符串

**第一种**：是直接使用框架构造的LuaEvents来与框架进行交互的
- 这部分API，可以直接被其他UI的lua使用，当然也包括你定义的'ModLocalDataManager_脚本'使用
- 至于'ModLocalDataManager_脚本'是什么接着往下看

**第二种**：是直接将你自己的lua脚本文件引入到框架的lua中，这样框架会自动加载并执行你的脚本
- 如果您具有高级Lua编程技能，也许可以直接通过查看我的lua源代码，使用这种方法你可以直接针对框架的进行操作，毕竟本框架本质是基于Lua表和setmetatable构建的伪类结构。
  - 如果你真这样做，请确保不会影响到框架的稳定性和正常运行，以及考虑到其他Mod的兼容性，避免造成不必要的影响或破坏其他Mod的功能。
> - 相关源代码：框架会先加载相关lua，包括你添加的'ModLocalDataManager_脚本'lua文件，在执行框架初始化
> ```lua
> -- 加载所有的框架相关lua文件
> include( "ModLocalDataManager_", true );
> -- 最后初始化MLDM
> MLDM:init()
> ```

- 首先你需要定义一个lua脚本文件，并且这个lua脚本的命名需要规范为是'ModLocalDataManager_'开头的lua文件
- 同时在modinfo文件中，你需要采用ImportFiles来导入这个lua脚本
> 对应演示
> - 例如我定义的lua脚本文件为：ModLocalDataManager_Test.lua
> - 命名符合'ModLocalDataManager_'开头
> - 然后在moinfo文件如下添加即可
> ```xml
> <!-- 省略moinfo其他代码 -->
>   <!-- 注意我这里是直接一个lua脚本根据FrontEnd/InGame环境不同对数据会有不同的处理/运行程序，你也可以分开写来个lua文件，分别在对应环境在modinfo进行加载 -->
>   <FrontEndActions>
>     <ImportFiles id="UI_Additions_ImportFiles">
>       <File>Common/UI/Additions/ModLocalDataManager_Test.lua</File>
>     </ImportFiles>
>   </FrontEndActions>
>   <InGameActions>
>     <ImportFiles id="UI_Additions_ImportFiles">
>       <File>Common/UI/Additions/ModLocalDataManager_Test.lua</File>
>     </ImportFiles>
>   </InGameActions>
> <!-- 省略moinfo其他代码 -->
> ```

- 这样框架会自动加载这个脚本
- 这种方式使用，你的脚本可以更加灵活的对自己的数据进行操作，同时通过构造你自己的LuaEvents来实现和自己其他UI的lua程序的交互
- 我强烈推荐这种方式
- **注意**：框架的'配置数据管理'功能的相关API很多不是使用LuaEvents，因此要用到这个方法，具体后续我会讲解到

### 3. 框架Lua的API
- 前面也说到，部分API不是LuaEvents，因此需要用到'UIlua直接使用'的方式来使用
- 那么我们分开来讲吧

#### 3.1 ModLocalDataManager_脚本API
- 是非LuaEvents/ExposedMembers等等可以跨UIlua调用的API
- 只能在你定义的'ModLocalDataManager_脚本'使用
- **注意**：为了避免干扰其他人mod的lua脚本，你需要使用local局部声明来避免变量冲突
- 框架的lua类的代码被封装在表'ModLocalDataManager', '你使用时不要直接用ModLocalDataManager',而是应当使用'MLDM'
> - 相关源代码(省略很多代码)：
> ```lua
> -- 官方的来个lua脚本
> include("InstanceManager");
> include("PopupDialog");
> -- 皮凯框架mod的lua脚本
> include("DeBugManager");
> include("ModIdToKey");
> DBM = DeBugManager:new(true)
> -- 省略多个InstanceManager全局参数...
> m_CoreModUUID = "e1f06d2d-68ab-4f8c-9b71-204099fc027d"
> m_CoreModHash = DB.MakeHash(m_CoreModUUID) -- 返回唯一对应Hash，一定程度减小数据大小
> m_CoreModHash = ModIdToKey(m_CoreModUUID) -- 避免发生撞key，单纯的数值还是有较大可能撞的
> m_PopupDialog = PopupDialog:new( "PKUI_ModData" );
> ModLocalDataManager={
>   -- 省略...
> }
> MLDM = ModLocalDataManager:new()
> -- 构造本框架mod数据
> CoreModObject = MLDM:GetModObject(m_CoreModUUID)
> -- 加载所有的框架相关lua文件
> include( "ModLocalDataManager_", true );
> -- 最后初始化MLDM
> MLDM:init()
> ```


下面开始具体接口介绍
- 开始前，请允许我进行一个总体概述。首先值得注意是在框架ModLocalDataManager表中，我构造众多函数。  
这些函数中，有很大一部分是框架内部使用的，因此，除非绝对必要，否则建议不要使用这些函数。  
然而，其中也有一部分函数是专为mod开发者设计的API，可以放心使用。  
- 'MLDM'就是框架的类，它是ModLocalDataManager的实例，是ModLocalDataManager_脚本的全局变量，可以直接使用。

接下来，我会依次介绍这些API的使用方法。


##### a. GetKey
- 对象是MLDM，需要一个隐式的 “self” 参数，所以请使用:
- 用于获得mod的数据存储key，这个key是框架内部用来存储数据的唯一标识，可以用来操作数据

> 操作演示：
> ```Lua
> -- 输入参数：modUUID
> -- 返回值：mod的本地数据存储key
> -- 示例：
> local modkey = MLDM:GetKey(modUUID)
> ```

> <details><summary>相关源码</summary>
> 
> 在框架ModLocalDataManager.lua的代码：
> ```Lua
>   -- 如果你在其他UIlua中，可以直接通过 include("ModIdToKey"); 使用ModIdToKey函数
>   GetKey = function(self, modID:string) return ModIdToKey(modID) end,
> ```
> 
> 在ModIdToKey.lua的代码：
> ```Lua
> function ModIdToKey(modID:string)
>   local modHash = DB.MakeHash(modID) -- 返回唯一对应Hash，一定程度减小数据大小
>   return "MLDM" ..(modHash < 0 and "N" or "") .. tostring(math.abs(modHash))
> end
> ```
> </details>


##### b. SetIgnoreModVersionUpdates
- 对象是MLDM，需要一个隐式的 “self” 参数，所以请使用:
- 作用是否开启 '忽略在ModDataIds定义的Version参数' 
  - 以便在每次启动游戏时，都会使用你lua脚本中定义的默认数据覆盖掉之前存储的数据
  - 避免之前存储数据干扰，毕竟mod开发中，数据经常需要变更，以便于开发者开发
  - 这样就不用每次更改数据结构还需要专门清除之前的数据

> 操作演示：
> ```Lua
> MLDM:SetIgnoreModVersionUpdates(modUUID, true)
> ```

> <details><summary>相关源码</summary>
> 
> ```Lua
>   SetIgnoreModVersionUpdates  = function(self, modID:string, open:boolean)
>     local key = self:GetKey(modID)
>     self.IgnoreModVersionUpdates[key] = open
>   end,
>   LoadConfigComplete = function(self)
>   -- 省略...
>               if not self.IgnoreModVersionUpdates[k] then
>               -- 省略...
>               else
>                 -- 在这里直接把lua脚本中定义的默认数据覆盖掉之前存储的数据
>                 oldModConfig[k] = newModConfig[k]
>               end
>   -- 省略...
>   end,
> ```
> </details>

##### c. SaveConfig
- 对象是MLDM，需要一个隐式的 “self” 参数，所以请使用:
- 不要被这里"Config"误导，其实这里的字面意思是"保存配置存档"
- 实际的功能是用于保存mod的数据，mod的数据会被框架一起存储到配置存档中，以便在下次启动游戏时，会自动读取之前存储的数据
- **注意**：它虽然有一个参数saveID(string),但这个参数是框架内部使用的，请不要使用它。  
其他具体注意细节在前面的"**2. 框架lua使用前言**"有说明。
> 操作演示：
> ```Lua
> -- 保存所有mod的数据到配置存档中（因为数据是存在一个配置存档所以每次都是所有mod数据一起保存）
> MLDM:SaveConfig()
> ```

> <details><summary>相关源码</summary>
> ```Lua
>   -- 注意： saveID是框架内部使用的，请Mod开发者注意不要使用它，保持saveID=nil即可
>   SaveConfig = function(self, saveID:string)
>     local function Save()
>       local gameFile = {
>        Name = saveID or self.ConfigFileName,
>         Type = SaveTypes.SINGLE_PLAYER,
>         FileType = SaveFileTypes.GAME_CONFIGURATION,
>       };
>       local EMMDC = ExposedMembers.ModDataCache
>       local Traversed = table.count(EMMDC) > 0 and EMMDC or self.CurrentlyModDatas
>       for k, v in pairs(Traversed) do
>         GameConfiguration.SetValue(k, v) --每个mod单独一个配置ID存储
>       end
>       if not saveID then
>         Events.SaveComplete.Add(function(eResult, eType, eOptions, eFileType )
>           if eFileType == SaveFileTypes.GAME_CONFIGURATION then
>             for k, _ in pairs(Traversed) do
>               GameConfiguration.SetValue(k, nil) -- 已保存到本地, 及时清除这里数据
>             end
>           end
>         end, 1)
>       end
>       UIManager:SetUICursor( 1 );
>       Network.SaveGame(gameFile);
>       UIManager:SetUICursor( 0 );
>     end
>     if (UI.IsInFrontEnd()) then
>       Save()
>     else
>       self:OpenMenu(4) -- 实际测试前端保存可以直接不打开对应的保存页面, 可以直接使用Network.SaveGame, 同时如果Name相同会自动覆盖同名的保存文件
>       Save()
>       self:CloseMenu(4)
>     end
>   end,
> ```
> </details>

##### d. GetModObject
- 对象是MLDM，需要一个隐式的 “self” 参数，所以请使用:
- 他会返回一个"类"对象(setmetatable定义的Lua表)，是为了更方便大家管理自己mod数据而设计。  
你可以使用这个对象来添加/定制属于你的Mod配置UI，或者进行非配置数据的操作。  
这个返回的对象，提供很多API，可以让你更方便的管理数据。   
- 最后为了方便介绍这个对象的API，我会使用“ModObject”来代指它

> 操作演示：
> ```Lua
> -- Lua 演示
> local modUUID = '7b7bda2b-f6e2-45d2-ba05-bbeb4bd9c46a' 
> -- 直接使用ModUUID，无需转为ModKey
> local TestModObject = MLDM:GetModObject(modUUID)
> -- 此时TestModObject就是这个mod的本地数据管理对象，你可以直接使用它提供的API来操作数据
> ```

> <details><summary>相关源码</summary>
> 
> - 都是直接在表 methods 定义的函数，所以不需要隐式的self参数，所以请使用.
> 
> ```Lua
>   GetModObject = function(self, modID:string)
>     -- 创建一个MOD的对象, 用于操作特定MOD的数据和配置UI（配置UI可以没有, 一切由modder决定, 存的数据我只负责存储和读取）
>     local key = self:GetKey(modID)
>     self:EnsureModData(modID)
>     self.ModConfigUI[key] = self.ModConfigUI[key] or { mainPanel={}, subpanel = {}, isHide = false, }
>     local function CheckPanelAndStoreDefaultConfig(subpanelKey:number, iKey)
>       -- 保存默认配置
>       if iKey then
>         self.DefaultModConfigs[key] = self.DefaultModConfigs[key] or {}
>         self.DefaultModConfigs[key][iKey] = self.CurrentlyModDatas[key][iKey] or nil
>       end
>       -- 确认控件插入的面板
>       if subpanelKey then
>         self.ModConfigUI[key].subpanel[subpanelKey] = self.ModConfigUI[key].subpanel[subpanelKey] or {}
>         return self.ModConfigUI[key].subpanel[subpanelKey]
>       end
>       return self.ModConfigUI[key].mainPanel
>     end
> 
>     local internalObj = {}
>     -- 定义内部对象的方法
>     local methods = {
>       ID = modID,
>       -- mod-数据
>       GetParam = function(iKey)
>         if type(iKey) == "string" or type(iKey) == "number" then
>           return self.CurrentlyModDatas[key][iKey]
>         end
>         print('Argument must be a string or number iKey.')
>       end,
>       GetParams = function()
>         return self.CurrentlyModDatas[key]
>       end,
>       SetParam = function(value1, value2)
>           -- 确保 value1 是一个字符串或数字
>         if type(value1) ~= "string" and type(value1) ~= "number" then
>           print("The first argument must be a string or number key.")
>         end
> 
>         self.CurrentlyModDatas[key][value1] = value2
>       end,
>       SetParams = function(t:table)
>         -- 批量设置参数模式：当 value1 是一个表时
>         for k, v in pairs(t) do
>           self.CurrentlyModDatas[key][k] = v
>         end
>       end,
>       -- 弃用，直接在sql注册对应的version和DataId
>       -- SetVersion = function(version:number) self.ModVersions[Key] = version end,
> 
>       SetIgnoreModVersionUpdates  = function(open:boolean) self.IgnoreModVersionUpdates[key] = open end,
>       -------------------------------------------------------------------------------------
>       -- mod-Config-UI 每个需要配置的mod都需要定义自己的配置UI, 这里提供一些基础的UI控件, modder可以根据自己的需求来增加自己的UI控件
>       GetUIControl = function(type:string, jKey)
>         if type == 'ModContainer' then return self.ModConfigUIControls[key][type] end
>         return self.ModConfigUIControls[key][type][jKey]
>       end,
> 
>       SetControlTitle = function(name:string, describe:string) self.ModConfigUI[key].mainPanel[0] = {Name = name, Describe = describe} end,
> 
>       AddSubpanelUI = function(name:string, describe:string, subpanelKey:number)
>         subpanelKey = subpanelKey or #self.ModConfigUI[key].subpanel+1 -- 如果没有指定subpanelKey, 则默认使用最后一个键值加1
>         self.ModConfigUI[key].subpanel[subpanelKey] = self.ModConfigUI[key].subpanel[subpanelKey] or {} -- 避免无对subpanelKey的面板
>         table.insert(self.ModConfigUI[key].mainPanel, {Name = name, Describe = describe, Type = "Subpanel", SubpanelKey = subpanelKey})
>       end,
> 
>       AddFirstLevelTitle = function(name:string, describe:string, isX:boolean, subpanelKey:number, sizeX:number)
>         table.insert(CheckPanelAndStoreDefaultConfig(subpanelKey), {Name = name, Describe = describe, Type = "FirstLevelTitle", SizeX = sizeX, IsX = isX})
>       end,
> 
>       AddSecondaryTitle = function(name:string, describe:string, subpanelKey:number)
>         table.insert(CheckPanelAndStoreDefaultConfig(subpanelKey), {Name = name, Describe = describe, Type = "SecondaryTitle"})
>       end,
> 
>       AddBooleanParamUI = function(name:string, describe:string, iKey, editableInGame:boolean, saveCallback:ifunction, subpanelKey:number, uiUpdateCallback:ifunction, sizeX:number)
>         table.insert(CheckPanelAndStoreDefaultConfig(subpanelKey, iKey), {Name = name, Describe = describe, Type = "Boolean", Key = iKey, SaveCallback = saveCallback, UIUpdateCallback = uiUpdateCallback, EditableInGame = editableInGame, SizeX = sizeX or 400})
>       end,
> 
>       AddCustomStringParamUI = function(name:string, describe:string, iKey, editableInGame:boolean, saveCallback:ifunction, subpanelKey:number, uiUpdateCallback:ifunction, sizeX:number)
>         table.insert(CheckPanelAndStoreDefaultConfig(subpanelKey, iKey), {Name = name, Describe = describe, Type = "CustomString", Key = iKey, SaveCallback = saveCallback, UIUpdateCallback = uiUpdateCallback, EditableInGame = editableInGame, SizeX = sizeX or 400})
>       end,
> 
>       AddPullDownParamUI = function(name:string, describe:string, iKey, range:table, editableInGame:boolean, saveCallback:ifunction, subpanelKey:number, uiUpdateCallback:ifunction, sizeX:number)
>         table.insert(CheckPanelAndStoreDefaultConfig(subpanelKey, iKey), {Name = name, Describe = describe, Type = "PullDown", Range = range, Key = iKey, SaveCallback = saveCallback, UIUpdateCallback = uiUpdateCallback, EditableInGame = editableInGame, SizeX = sizeX or 400})
>       end,
> 
>       AddStrSliderParamUI = function(name:string, describe:string, iKey, range:table, isX:boolean, editableInGame:boolean, saveCallback:ifunction, subpanelKey:number, uiUpdateCallback:ifunction, displayTextSizeX:number, sliderSizeX:number)
>         table.insert(CheckPanelAndStoreDefaultConfig(subpanelKey, iKey), {Name = name, Describe = describe, Type = "StrSlider", Range = range, IsX = isX, Key = iKey, SaveCallback = saveCallback, UIUpdateCallback = uiUpdateCallback, EditableInGame = editableInGame, DisplayTextSizeX = displayTextSizeX or 60, SliderSizeX = sliderSizeX or 335})
>       end,
> 
>       AddNumSliderParamUI = function(name:string, describe:string, iKey, minValue:number, maxValue:number, range:table, editableInGame:boolean, saveCallback:ifunction, subpanelKey:number, uiUpdateCallback:ifunction, displayTextSizeX:number, sliderSizeX:number)
>         table.insert(CheckPanelAndStoreDefaultConfig(subpanelKey, iKey), {Name = name, Describe = describe, Type = "NumSlider", Range = range, MinValue = minValue or 0, MaxValue = maxValue or 10, Key = iKey, SaveCallback = saveCallback, UIUpdateCallback = uiUpdateCallback, EditableInGame = editableInGame, DisplayTextSizeX = displayTextSizeX or 32, SliderSizeX = sliderSizeX or 363})
>       end,
>       --[[ 弃用!!!
>       AddKeyBindingTitleUI = function(title1:string, title2:string, subpanelKey:number, titleSizeX:number)
>         table.insert(CheckPanelAndStoreDefaultConfig(subpanelKey), {Title1 = title1, Title2 = title2, Type = "KeyBindingTitle", TitleSizeX = titleSizeX or 210})
>       end,
> 
>       AddKeyBindingActionUI = function(name:string, describe:string, iKey, editableInGame:boolean, saveCallback:ifunction, subpanelKey:number, uiUpdateCallback:ifunction, buttonSizeX:number)
>         table.insert(CheckPanelAndStoreDefaultConfig(subpanelKey), {Name = name, Describe = describe, Type = "KeyBindingAction", Key = iKey, SaveCallback = saveCallback, UIUpdateCallback = uiUpdateCallback, EditableInGame = editableInGame, ButtonSizeX = buttonSizeX or 210})
>       end,
>       ]]
>     }
>     -- 设置嵌套伪类, 便于使用, 对象直接可以访问 getParams 和 setParams 方法
>     setmetatable(internalObj, { __index = methods })
>     return internalObj
>   end,
> ```
> </details>

> <details><summary>相关API</summary>
> 这里也有 SetIgnoreModVersionUpdates
> 但和上面不同，这里是直接ModData设定该mod数据是否"忽略版本更新"，而上面还需要提供modUUID参数
> 
> 
> ###### **基础API**
> 
>  .或: | 名字 | 参数 | 返回 |说明
> --- | --- | --- | --- | --- 
> . | SetIgnoreModVersionUpdates | open(bool) | - | 作用是否开启 '忽略在ModDataIds定义的Version参数' 以便在每次启动游戏时，都会使用你lua脚本中定义的默认数据覆盖掉之前存储的数据<br>避免之前存储数据干扰，毕竟mod开发中，数据经常需要变更，以便于开发者开发
> . | ~~SetVersion~~ | ~~version(number)~~ | - | ~~设置当前mod的版本号，框架会根据这个版本号来决定是否应该覆盖现有的数据。~~<br>弃用，请使用 SetIgnoreModVersionUpdates/在sql中更新Version字段
> . | SetParam | key(string/number)<br>param(string/number/table) | - | 初始注册参数，也是更改参数的API
> . | SetParams | params(table) | - | 批量初始注册/更改参数的API，会自动遍历输入的params表执行SetParam
> . | GetParam | key(string/number) | param(string/number/table/nil) | 获取参数值，注意返回nil表示参数不存在
> . | GetParams | - | params(table/nil) | 返回一个table是这个mod所有参数，注意返回nil表示参数不存在
> 
> ###### **Mod配置数据UI-API**
> (以下API用于创建和管理Mod的配置界面)  
> **注意**：在使用几个Add~ParamUI时，不要忘记先执行前面的SetParam/SetParams，提前注册好对应的key和默认值
> 
>  .或: | 名字 | 参数 | 返回 |说明
> --- | --- | --- | --- | --- 
> . | GetUIControl | type(string)<br>jKey(string/number/nil) | Control(object) | 获取UI控件，type是控件类型， jKey是控件的唯一标识，返回值是控件对象，具体情况见注释【1】
> . | SetControlTitle | name(string)<br>describe(string) | - | 设定这个mod配置的标题和标题的提示，可以不设定，默认使用mod的名字和描述
> . | AddSubpanelUI | name(string)<br>describe(string)<br>subpanelKey(number) | - | 添加一个可以折叠的子面板，这里name和Describe是设定这个按钮上的名字和提示，可以不设定，默认使用“LOC_OPTIONS_SHOW_ADVANCED_GRAPHICS”(既“显示高级设置”)<br>subpanelKey是这个子面板的唯一标识，用于后续添加控件时指定添加到哪个子面板
> . | AddFirstLevelTitle | name(string)<br>describe(string)<br>isX(boolean)<br>subpanelKey(number)<br>sizeX(number) | - | 添加一级标题，用于分割不同配置项<br>sizeX设定这个标题宽度<br>isX是否是用这个控件的变体（我准备两种一级标题控件）
> . | AddSecondaryTitle | name(string)<br>describe(string)<br>subpanelKey(number) | - | 添加二级标题<br>用于分割不同配置项
> . | AddBooleanParamUI | name(string)<br>describe(string)<br>iKey<br>editableInGame(boolean)<br>saveCallback(ifunction)<br>subpanelKey(number)<br>uiUpdateCallback(ifunction)<br>sizeX(number) | - | 添加一个开关选项<br>sizeX设定这个开关选项宽度
> . | AddCustomStringParamUI | name(string)<br>describe(string)<br>iKey<br>editableInGame(boolean)<br>saveCallback(ifunction)<br>subpanelKey(number)<br>uiUpdateCallback(ifunction)<br>sizeX(number) | - | 添加一个字符串选项<br>sizeX设定这个字符串选项宽度
> . | AddPullDownParamUI | name(string)<br>describe(string)<br>iKey<br>range(table)<br>editableInGame(boolean)<br>saveCallback(ifunction)<br>subpanelKey(number)<br>uiUpdateCallback(ifunction)<br>sizeX(number) | - | 添加一个下拉菜单选项<br>range是下拉菜单选项的值域，具体情况见注释【2】<br>sizeX设定这个下拉菜单选项宽度
> . | AddStrSliderParamUI | name(string)<br>describe(string)<br>iKey<br>range(table)<br>isX(boolean)<br>editableInGame(boolean)<br>saveCallback(ifunction)<br>subpanelKey(number)<br>uiUpdateCallback(ifunction)<br>displayTextSizeX(number)<br>sliderSizeX(number) | - | 添加一个文本滑块选项，根据滑动的不同值显示不同文本<br>range是下拉菜单选项的值域，具体情况见注释【2】<br>displayTextSizeX和sliderSizeX分别决定显示数值和滑块的宽度，推荐displayTextSizeX和sliderSizeX相加等于395 (由400-5:Padding 得到)
> . | AddNumSliderParamUI | name(string)<br>describe(string)<br>iKey<br>minValue(number)<br>maxValue(number)<br>range(table)<br>editableInGame(boolean)<br>saveCallback(ifunction)<br>subpanelKey(number)<br>uiUpdateCallback(ifunction)<br>displayTextSizeX(number)<br>sliderSizeX(number) | - | 添加一个数值滑块选项，是直接显示滑快的数值<br>range是下拉菜单选项的值域，具体情况见注释【3】，与上面来个下拉菜单和文本滑块的range不同<br>minValue/maxValue是滑块选项最小/大值的数值，默认为0/1<br>displayTextSizeX和sliderSizeX分别决定显示数值和滑块的宽度，推荐displayTextSizeX和sliderSizeX相加等于395 (由400-5:Padding 得到)
> 
> 
> 其他通用参数说明：
> - **Name(string)**：UI控件显示的名字
> - **Describe(string)**：UI控件鼠标悬停时显示的提示内容
> - **SubpanelKey(number)**：只在添加控件到子面板时，才需要设定这个参数。(否则为nil会正常添加控件到主面板)
> - **EditableInGame(bool)**： 默认为true
>   - true  表示在FrontEnd前端（主菜单）和ingame对局内的环境都可以修改。
>   - false 表示只能在游戏的FrontEnd前端（主菜单）环境修改。会同步更新存档内这个值
>   - 额外： 如果需要只能在游戏FrontEnd前端（主菜单）环境修改。但不影响已有的存档是固定的值，请使用游戏官方的Parameters表相关的
> - **SaveCallback(ifunction)**：为配置保存后的回调函数 (它会在玩家确认保存配置后调用, 并传入两个参数：修改后的参数和之前的参数)
> - **UIUpdateCallback(ifunction)**：为配置修改后的立即更新UI的回调函数 (它会在玩家调整配置时立即调用, 例如实现配置A为某个值时，配置B被禁用修改)
> - **iKey(string/number)**：这个是UI控件对应的mod数据参数的建(key)，既对应前面'SetParam'的'key(string/number)'参数
最后注意是按添加顺序来排列对应UI, 所以注意添加顺序    
> 
> 注释【1】GetUIControl的type参数为控件类型, 支持的类型有：ModContainer, Subpanel, FirstLevelTitle, SecondaryTitle, Boolean, CustomString, PullDown, StrSlider, NumSlider
> - type为 ModContainer 时, jKey为nil, 此时返回的是整个mod的配置面板控件
> - type为FirstLevelTitle, SecondaryTitle时, jKey为你添加这些的标题的顺序，此时返回的是对应的标题控件
> - type为Subpanel时, jKey为你添加的子面板的subpanelKey，此时返回的是对应的子面板控件
> - type为Boolean, CustomString, PullDown, StrSlider, NumSlider时, jKey为你添加的控件时的iKey， 此时返回的是对应的控件对象
> 
> 注释【2】范围参数range：是一个数组table, 该数组的每个元素是一个子表, 包含显示文本，内部存储值，描述文本三个元素（描述文本是鼠标悬停时显示的提示内容）,当然如果不需要描述文本，可以不填
> - 例如：range = {  { "显示文本1", "内部存储值1" ，"描述文本1" }, { "显示文本2", "内部存储值2", "描述文本2" }, { "显示文本3", "内部存储值3"}, }
> 
> 注释【3】数值滑块的 范围参数range 有所区别：也是一个数组table，但直接将数值作为内部存储值，和显示文本，因此只需要提供"描述文本", 当然如果不需要描述文本，可以不填
> - 例如：range = {  { "描述文本1" }, { "描述文本2" }, }
> 
> </details>

> <details><summary>为什么没有添加的配置数据是热键类型的API</summary>
> 
> - 如果看的仔细你也许注意道在上面"相关源码"中，有一段AddKeyBinding的代码被注释。  
> 想了想还是解释为什么注释弃用，因为官方本身已经有添加自定义热键的功能了。  
> 如果感兴趣你首先可以看看官方相关代码，涉及官方数据库表：InputCategories,InputActions,InputActionDefaultGestures  
> 你需要填写这几个表，以便在游戏的官方热键设置这里有对应的UI生成。
> 然后需要构造对应的lua代码，需要在Events.InputActionTriggered事件中添加你的回调函数，这样即可添加能长久保存的热键的功能。
>  
> ```Lua
> -- 参考lua代码：
>   function OnInputActionTriggered( actionId )
>     -- XXX是在InputActions, InputActionDefaultGestures表定义的
>     if ( actionId == Input.GetActionId("XXX") ) then
>       Callback()
>     end
>   end
>   Events.InputActionTriggered.Add( OnInputActionTriggered ); -- 绑定事件驱动
> ```
> </details>
 
> <details><summary>相关API演示</summary>
> 
> ###### **基础API**
> ```Lua
> -- 直接是当前modUUID的值
> print(TestModObject.ID == modUUID) -- true
> 
> -- SetIgnoreModVersionUpdatesd(true) 的作用是忽略在ModDataIds定义的Version参数，每次启动游戏时，都会使用这个mod的默认数据来填充Lua变量，以便于开发者开发
> --TestModObject.SetIgnoreModVersionUpdates(true)
> 
> -- 注册一个参数，建（key）为 Default_Leader 值为 TEST_LEADER
> -- SetParam除了是初始注册参数，也是更改参数的API，
> TestModObject.SetParam("Default_key", "TEST_Value")
> -- GetParam可以获取参数值，他接受一个参数，参数是建（key）
> TestModObject.SetParam("Default_key2", "TEST_Value2")
> 
> -- 批量设置参数模式：此时第一个参数应该是表(table)类型。用于一次性设置多个参数
> local temp1 = {[Default_key3] = "TEST_Value3", [Default_key4] = "TEST_Value4"}
> TestModObject.SetParams(temp1)
> 
> -- GetParam可以获取参数值，他接受一个参数，参数是建（key）
> print(TestModObject.GetParam("Default_Leader")) -- TEST_LEADER
> print(TestModObject.GetParam("Default_key2")) -- TEST_Value2
> print(TestModObject.GetParam("Default_key3")) -- TEST_Value3
> print(TestModObject.GetParam("Default_key4")) -- TEST_Value4
> 
> -- 批量获取参数模式：此时返回值是一个表(table)类型。用于一次性获取mod的所有数据
> local temp2 = TestModObject.GetParams()
> print(temp2.Default_key) -- TEST_LEADER
> print(temp2.Default_key2) -- TEST_Value2
> print(temp2.Default_key3) -- TEST_Value3
> print(temp2.Default_key4) -- TEST_Value4
> ```
> ###### **Mod配置数据UI-API**
> - SetControlTitle，AddSubpanelUI，AddFirstLevelTitle，AddSecondaryTitle 这几个API是用来设置mod配置界面的标题和子面板的，可以不设定，使用默认值名字和描述
> - 必须至少有一个Add~ParamUI(既一个实际有效的配置项)才能正常生成配置界面，否则是无效的
> 
> **SetControlTitle**
> > ```Lua
> > TestModObject.SetControlTitle("测试Mod标题", "测试mod描述")
> > ```
> > ![对应效果](图片\1.png "mod配置界面的标题和鼠标悬停的提示都更改为这里设定的内容") 
> > <details>
> > <summary>SetControlTitle不设置的效果</summary>
> > 
> > ```Lua
> > -- TestModObject.SetControlTitle("测试Mod标题", "测试mod描述")
> > ```
> > ![默认效果](图片\2.png "mod配置界面的标题和鼠标悬停的提示的默认内容") 
> > </details>
> &nbsp;  
> **AddSubpanelUI**
> - 折叠子面板也可以直接在Add~ParamUI输入新的subpanelKey来生成新的子面板，无需使用AddSubpanelUI
> - 但这样生成的子面板的名字和描述都是默认的
> - 注意如果一个子面板没有通过任何Add~ParamUI添加配置项，则不会成功生成
> > ```Lua
> > -- 这里的subpanelKey是1，可以为nil，会自动设定为'当前子面板总数+1'
> > TestModObject.AddSubpanelUI("测试子面板1设置", "测试子面板1描述", 1)
> > ```
> > ![对应效果](图片\3.png "添加了一个子面板，并设定了名字和提示") 
> > <details>
> > <summary>AddSubpanelUI不设置的效果</summary>
> > 
> > ```Lua
> > -- TestModObject.AddSubpanelUI("高级设置", "显示高级设置", 1)
> > ```
> > ![默认效果](图片\4.png "子面板默认标题和鼠标悬停的提示的默认内容") 
> > </details>
> &nbsp;  
> **AddFirstLevelTitle**
> - 这里的isX参数是用来设定是否用这个控件的变体的，默认为false，即用普通的一级标题控件
> - 这里的sizeX参数是用来设定这个标题的宽度的，默认为400
> > ```Lua
> > TestModObject.AddFirstLevelTitle("测试一级标题1", "测试一级标题1描述", false, nil, 300)
> > TestModObject.AddFirstLevelTitle("测试一级标题2", "测试一级标题2描述", true)
> > TestModObject.AddFirstLevelTitle("测试折叠子面板1一级标题1", "测试子面板1一级标题1描述", false, 1, 300)
> > TestModObject.AddFirstLevelTitle("测试折叠子面板1一级标题2", "测试子面板1一级标题2描述", true, 1)
> > TestModObject.AddFirstLevelTitle("测试折叠子面板2一级标题1", "测试子面板2一级标题1描述", false, 2, 300)
> > TestModObject.AddFirstLevelTitle("测试折叠子面板2一级标题2", "测试子面板2一级标题2描述", true, 2)
> > ```
> > ![对应效果](图片\5.png "添加了6个一级标题，演示不同参数效果") 
> 
> **AddSecondaryTitle**
> - 这里的subpanelKey参数是用来指定添加到哪个子面板的，如果不指定，则会添加到主面板
> - 这里的sizeX参数是用来设定这个标题的宽度的，默认为400
> > ```Lua
> > TestModObject.AddSecondaryTitle("测试二级标题1", "测试二级标题1描述", nil, 300)
> > TestModObject.AddSecondaryTitle("测试二级标题2", "测试二级标题2描述", 1, 300)
> > TestModObject.AddSecondaryTitle("测试折叠子面板1二级标题1", "测试子面板1二级标题1描述", 1, 300)
> > TestModObject.AddSecondaryTitle("测试折叠子面板1二级标题2", "测试子面板1二级标题2描述", 1)
> > TestModObject.AddSecondaryTitle("测试折叠子面板2二级标题1", "测试子面板2二级标题1描述", 2, 300)
> > TestModObject.AddSecondaryTitle("测试折叠子面板2二级标题2", "测试子面板2二级标题2描述", 2)
> > ```
> > ![对应效果](图片\6.png "添加了6个二级标题，演示不同参数效果") 
> 
> **Add~ParamUI**
> > ```Lua
> > -- 添加一个开关选项
> > TestModObject.AddBooleanParamUI("测试布尔参数1", "测试布尔参数描述1", "KEY_BOOL_PARAM1", true, nil, nil, nil, 300)
> > TestModObject.AddCustomStringParamUI("测试自定义字符串参数1", "测试自定义字符串参数描述1", "KEY_CUSTOM_STRING_PARAM1")
> > TestModObject.AddPullDownParamUI("测试下拉框参数1", "测试下下拉框参数描述1", "KEY_PULLDOWN_PARAM1", {{"选项1", 1, "选项1描述"}, {"选项2", 2, "选项2描述"}, {"选项3", 3}})
> > TestModObject.AddStrSliderParamUI("测试显示字符滑块参数1", "测试字符串滑块参数描述1", "KEY_STR_SLIDER_PARAM1", {{"选项1", 1}, {"选项2", 2}, {"选项3", 3}})
> > TestModObject.AddStrSliderParamUI("测试显示字符滑块参数改1", "测试描字符滑块参数描述改1述1", "KEY_STR_SLIDER_PARAM2", {{"选项1", 1}, {"选项2", 2}, {"选项3", 3}}, true)
> > TestModObject.AddNumSliderParamUI("测试显示数值滑块参数1", "测试数值滑块参数描述改1", "KEY_NUM_SLIDER_PARAM1", 0, 100, 1, 10, true)
> > TestModObject.AddNumSliderParamUI("测试显示数值滑块参数2", "测试数值滑块参数描述2", "KEY_NUM_SLIDER_PARAM2", 0, 100, 1, 10)
> > ```
> > ![对应效果](图片\7.png "添加了6个配置项，演示不同参数效果") 
> > </details>
> 
> </details>


#### 3.2 跨UIlua脚本API
- 如果熟悉文明6的lua，我想说到跨UI-lua脚本的交互应该立马想到LuaEvents和ExposedMembes。  
是的我们需要使用他们来完成跨UI-lua脚本的交互，同时还有一个独特的官方API也会被使用，请让我一点一点讲述

##### a. LuaEvents
- **注意**: 是在UI环境定义的的LuaEvents，你在Game环境无法使用这里的LuaEvents
- 不知道你是否还记得前面ModDataIds的DataId，很快要用到它

> <details><summary>部分相关源码</summary>
> ```Lua
> CreateModConfigUIs = function(self)
>   -- 完成初始化
>   LuaEvents.MLDM_ConfigUIStart()
>   -- 省略其他代码...
>   LuaEvents.MLDM_ConfigUIComplete()
> end,
> init = function(self)
>   -- 省略其他代码...
>   LuaEvents.MLDM_Saving.Add(function() --saveID)
>     self:SaveConfig() --saveID)
>   end)
>   LuaEvents.MLDM_SetIMVU.Add(function(modUUID, isIgnore)
>     self:SetIgnoreModVersionUpdates(modUUID, isIgnore)
>   end)
>   LuaEvents.GetModDataByDataId.Add(function(DataId, Key)
>     -- 验证DataId是否只包含字母数字和下划线
>     local function validateDataId(dataId)
>       return dataId:match("^%a[%w_]*$")
>     end
>     
>     -- 如果DataId包含无效字符，则打印错误消息并返回
>     if not validateDataId(DataId) then
>       print("DataId contains invalid characters. Only alphanumeric and underscore are allowed.")
>       return
>     end
>
>     -- 检查Key是否为nil或字符串类型
>     if Key ~= nil and type(Key) ~= "string" and type(Key) ~= "number" then
>       print("Key must be a string, number or nil.")
>       return
>     end
>     -- print("发生数据获取", DataId, Key)
>     -- 根据Key是否存在来获取整个mod数据或对应key的数据
>     local value = Key and self.CurrentlyModDatas[self.ModDataIds[DataId] ][Key] or self.CurrentlyModDatas[self.ModDataIds[DataId] ]
>     
>     -- 如果Key存在，则构造新的DataId以包含Key
>     DataId = Key and DataId .. "_" .. Key or DataId
>
>     -- 触发对应DataId的LuaEvents事件，并传递获取到的数据
>     LuaEvents[DataId](value)
>   end)
> 
>   LuaEvents.SetModDataByModId.Add(function(modID:string, value1, value2)
>     -- 两种修改模式
>     -- 当value1是一个表, 则遍历value1表, 将每个键值对添加到当前modID的参数数组表中
>     -- 当value1是一个键值对, 则将value1作为键, value2作为值, 添加到当前modID的参数数组表中
>     local key = self:GetKey(modID)
>     if type(value1) == "table" then
>       for k, v in pairs(value1) do
>         self.CurrentlyModDatas[key][k] = v
>       end
>     else
>       self.CurrentlyModDatas[key][value1] = value2
>     end
>   end)
>   -- 省略其他代码...
> end,
> ```
> </details>

> <details><summary>相关API</summary>
>
> 方法 | 参数 | 说明
> --- | --- | --- 
> MLDM_ConfigUIStart | - | 通知其他mod脚本配置UI开始刷新，每次打开配置UI时，都会调用这个事件
> MLDM_ConfigUIComplete | - | 通知其他mod脚本配置UI刷新完成，每次打开配置UI时，都会根据配置数据刷新UI，在刷新完成后，会调用这个事件
> MLDM_AlmostComplete | - | 通知其他mod脚本，框架几乎初始化完成，但加载Mod数据的屏幕还未关闭，可以正常获得各自Mod的数据了
> MLDM_InitComplete | - | 通知其他mod脚本，框架初始化完成，可以正常获得各自Mod的数据了
> MLDM_Saving | - | 用于让框架对当前数据进行保存。<br>因为数据的存储是保存在配置存档中的，为了避免频繁的保存，所以每次发生数据修改不会自动立马保存，而需要Mod开发者在更改完成多个数据后，手动调用这个事件来保存数据。<br>以减小保存频率，提高性能。<br>具体注意细节在前面的"**2. 框架lua使用前言**"有说明。
> MLDM_SetIMVU | modUUID(string)<br>isIgnore(boolean) | 既SetIgnoreModVersionUpdates的缩写，用于设置是否忽略Mod版本更新<br>具体参考前面关于SetIgnoreModVersionUpdates的其他API说明
> GetModDataByDataId | DataId(string)<br>Key(string/number/nil) | 向框架发出获取指定Mod数据的请求，有两种模式：<br>1. 同时传入DataId和Key，获取指定DataId对应Mod的Key的数据<br>2. 不传入Key，获取整个Mod数据<br>注意LuaEvents无法直接返回数据，所以需要配合另一个LuaEvents[DataId]来获取数据<br>具体情况见注释【1】
> [DataId]/[DataId_Key] | Value(any) | 配合上面的GetModDataByDataId来实现跨UIlua脚本的数据传递，获取指定DataId对应Mod的数据，并返回给调用者<br>具体情况见注释【1】
> SetModDataByModId | modUUID(string)<br>Value1(string/number/table)<br>Value2(any) | 设置指定Mod数据，有两种模式：<br>1. Value1作为key(string/number)，同时传入modUUID、Key和Value2，设置指定Mod的Key的数据为Value2<br>2. Value1类型是表(table)，传入modUUID和Value1，会遍历Value1的key-value对，设置指定Mod的key的数据为value<br>最后完成所有数据更改，如果需要立即保存数据别忘记调用保存的API
>
> 注释【1】：演示如何通过DataId获取对应的Mod数据  
> 假设有一个mod的DataId为"MyModID"，modder需要定义了自己的处理数据的方法，在自己的脚本中可以这样定义自己的处理数据的
> ```Lua
> -- 定义自己的处理数据的方法
> -- 在自己的脚本中可以这样定义自己的处理数据的方法
> local MyModData = {}
> LuaEvents.MyModID.Add(function(value)
>   MyModData = value
>   -- 省略 modder定义的处理数据的方法，value是modder定义的mod数据
>   print("成功获取Mod数据")
> end)
> LuaEvents.MyModID_KEY1.Add(function(value)
>   MyModData["KEY1"] = value
>   -- 省略 modder定义的处理数据的方法，value是modder定义的mod数据
>   print("成功获取Mod数据中建为'KEY1'对应数据")
> end)
> -- 然后调用此LuaEvents向框架发送获取Mod数据的请求
> -- 后面框架会调用LuaEvents.MyModData以返回mod数据
> LuaEvents.GetModDataByDataId("MyModData")
> LuaEvents.GetModDataByDataId("MyModData", "KEY1")
> ```
> </details>

##### b. ExposedMembers
> - 这里首先我想讲述一下我理解的ExposedMembers:  
首先字面意思是"公开的成员"  
而再实际的lua应用中，所有的lua环境都能访问使用它，包括前端环境，InGame环境，以及InGame的Game环境和UI环境。  
因此我们认为ExposedMembers是一种游戏内所有LUA环境都能访问使用的全局的变量，是一个表。是一个非常好的跨LUA环境的交互方式。

> - 因此ExposedMembers可以存储包括字符串，数字，表，函数等各种类型的数据，并且可以被所有LUA环境访问。  
但他也有缺陷，更适合单机本地游戏，不适用于联机游戏。因为你无法获取联机游戏中对方的ExposedMembers数据。  
导致容易造成数据不同步，使得联机断开。  
为此我后续会单独讲解框架在联机游戏中的注意项。  

> - 这里也要感谢'夏凉凉凉'告知ExposedMembers可以实现前端数据与InGame数据之间的交互，在此之前我只知道他可以实现InGame的Game环境和UI环境的lua交互。

> - 那么接下来我将详细介绍框架中ExposedMembers的使用方法。

当框架数据加载完成(既触发LuaEvents.MLDM_InitComplete)后，会将数据给与到ExposedMembers.ModDataCache中
你可以通过ExposedMembers.ModDataCache[Modkey]来获取自己mod数据。

##### c. GameConfiguration.GetValue
GameConfiguration是游戏官方的API，框架的核心就是加载/保存配置存档，再通过GameConfiguration.SetValue和GameConfiguration.GetValue来实现数据的存储和读取。
- 是的这个接口你也可以用于UI环境自己存储数据，但注意你使用的存储键值一定要保证唯一性，特别是不要覆盖到游戏本身用到的键值。以免导致游戏不正常

> **注意使用场景**：  
> 总的来说这个**仅仅推荐在联机中使用**
> - 在前端环境,因为GameConfiguration涉及游戏配置，当发生创键新的游戏/退回主菜单等等情况时，游戏都会自动清空GameConfiguration的数据，所以，请不要在前端使用GameConfiguration.GetValue来获取你的mod数据。
> - 而在InGame环境，且是单机的情况，理论上如果框架将数据在游戏开始时，将mod数据存入游戏存档的GameConfiguration，那么其他mod也能获取到，但我认为这是没有必要的操作，因为完全可以通过LuaEvents/ExposedMembers来实现数据的获取。没必要在使用GameConfiguration.GetValue来获取数据，因为直接通过GameConfiguration，会极大增加玩家每个游戏存档的体积
> - 最后在InGame环境的联机游戏中时，部分mod数据很可能是影响游戏的关键数据，所以，我在框架设计了一种机制来实现数据的同步，是在联机存档开始前在联机房间中统一使用GameConfiguration.SetValue来同步数据，在联机房间中，其他mod也能获取到数据，并且在游戏结束后统一使用GameConfiguration.SetValue来同步数据。
> - 具体的联机游戏中数据同步机制，请参考后面的联机游戏中注意事项部分。
#### 3.3 联机游戏中注意事项
- 如果你确定你的的数据不会影响游戏联机同步，其实完全不用看这个。  
例如是数据的用途是玩家个性化的UI界面使用的相关配置数据，可以放心使用。  
一般导致不同步的数据是再GameLua中不同玩家使用不同数据产生不同结果的情况，会导致不同步。

首先，由于我的时间精力有限，我无法充足测试所有可能的联机游戏情况，因此，这里我只能尽量根据自己经验来进行设计。  
所以这方面我非常需要大家的帮助，以及非常欢迎大家能给予我更好的建议。

> - 对于联机情况我的设计想法：  
为了要避免避免造成可能的数据不同步，以及玩家的数据被其他玩家数据污染。
因此，在联机的InGame环境不会进行保存数据的操作，既几个保存数据的API都不会工作。
而是采用房主的数据为标准
- 联机游戏有多种情况，又分房主和非房主情况，因此需要考虑多种情况，我们一点点来
##### a. 房主玩家未启框架mod，但非房主玩家启用了框架mod
如果你的mod是以框架为前置的，那么不用管这个了，你的mod和框架一起不工作了  

如果你的mod在框架不存在的情况依然可以工作，既你计划采用默认数据，那么或许你可以直接在游戏内使用ExposedMembers缓存的数据作为默认值


## 



##### MLDM_ConfigUI两个相关事件zf
- 是框架的配置UI相关的来个LuaEvent
- 如果你需要个性化自己的配置UI，我想你很可能需要用到这两个事件
- 例如你自己构建配置UI，插入到框架的配置UI中


#### 2.跨UIlua使用
- 在实际的mod中，我们经常会涉及多个lua文件，需要跨UIlua调用，例如可能需要在其他UI中显示一些mod配置数据，或者在其他UI中进行一些操作，那么框架提供了一些API来实现跨UIlua调用
- 不知道你是否还记得前面ModDataIds的DataId，现在要用到它

当你在其他UIlua脚本中需要获取自己mod数据时，可以调用如下API：
LuaEvents.MLDM_GetModDataIds(DataId, key)













#### 其他相通部分
##### 你需要在modinfo文件中添加框架的依赖
```Xml
```

#### Mod配置数据管理
- 那么先来演示一下如何使用框架进行mod配置数据的管理部分
- 需要现在注册默认参数
```Lua
-- Lua 演示

```


<details>
<summary></summary>

</details>












### 部分一：mod配置数据部分
当一些mod数据作为这个mod面向玩家的设置时，推荐使用这个，框架为此提供便利的UI添加和数据更改存储，modder只需要框架这里注册对应的默认参数和对应的UI控件需求

1.  第一步
添加本地
2.  