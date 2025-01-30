# PKUI本地数据框架使用指南
## 一. 文档信息
- **作者：皮皮凯(PiPiKai)**
- **时间：2025.1.20**
- **版本：v1.2**
- [Githup链接](https://github.com/X-PPK/CIV6-PKUIModDataGuide)
- [Gitee链接](https://gitee.com/XPPK/pk-civ6-ModLocalDataFrameworkGuide)
---
本文档旨在阐述《文明6》中皮凯UI框架mod下的“mod本地数据存储子框架”的使用方法。该框架允许每个mod拥有独立的持久化数据存储，这些数据能够跨游戏对局保存，存储于玩家的本地电脑中。以下提供的详细使用指南旨在帮助那些决定采用此框架的mod开发者们，能够轻松掌握并有效地运用这一工具。
以下内容中提到的“框架”，均指该子框架部分。
> 感谢以下人员对本框架的帮助 : (根据拼音首字母/字母排序)
> - 感谢'[atts.leo](https://steamcommunity.com/profiles/76561199589258333)' 非常有耐心陪伴我测试联机，研究游戏联机相关的API。如果没有他的帮助，我恐怕难以解决框架的数据同步问题。
> - 感谢'[号码菌Synora](https://steamcommunity.com/profiles/76561198147378701)' 帮我解答了一些技术难题，减少了我研究联机同步时验证官方API的时间，让我能更高效地推进项目进度。
> - 感谢'[UzukiShimamura卯月](https://steamcommunity.com/profiles/76561198402598762)' 在我实现这个框架过程中提供了许多具有建设性的建议和帮助，让我规避了许多未曾考虑到的问题，大大提高了项目的稳定性。
> - 感谢'[夏凉凉凉](https://steamcommunity.com/profiles/76561199052584728)' 告知ExposedMembers可以实现前端数据与InGame数据之间的交互，在此之前我只知道它可以实现InGame的Game环境和UI环境的lua交互。得益于他的帮助，我现在可以更加便捷地实现数据从前端传递到InGame，避免了采用更为复杂的方法。
---

## 二. 框架设计概述
本部分将介绍框架的设计原因和优势，帮助理解其核心功能和目的。
### 1. 数据存储/读取方式
- **集中存储**：框架采用集中式数据存储机制，所有mod数据均存储在一个“配置存档”中。游戏启动并进入主菜单时，框架会自动读取该“配置存档”中的数据。
- **数据读取**：读取的数据会被同步至相应的Lua脚本变量，确保让使用框架的mod获取正确的数据。同时，框架会生成LuaEvents和ExposedMembers等API，以便mod开发者能够便捷地在其他lua脚本中进行数据操作。
### 2. 设计框架管理缘由
- **避免冲突**：若每个mod各自使用独立的“配置存档”，则不仅会增加数据读取时间，还可能在加载时发生冲突，因为同一时间只能加载/保存一个配置存档。
- **减小存储占用**：由于“配置存档”包含大量非必要的其他数据，若每个mod单独存储，将浪费存储空间。集中存储可以有效利用空间。
- **统一管理机制**：为了避免上述问题，框架采用统一的管理机制和存储来处理所有mod的数据，确保数据的一致性和系统的稳定性。
### 3. 统一管理优势
- **一致性保障**：通过框架统一管理，可以保持数据的一致性，避免因数据分散而导致的混乱。
- **更可靠稳定性**：统一管理有助于维持数据稳定性，减少意外干扰，避免不同mod间的数据处理引发冲突和BUG。
- **规范化约束**：这种做法不仅是一种规范化的约束，也有助于mod开发者更有效地管理本地数据，减少不同mod之间的相互干扰。
### 4.额外补充
- **框架lua环境**:框架是跨越游戏前端环境和InGame环境的，而前端是没有GameLua环境的，因此框架的lua环境是UILua环境，同时mod开发者要注意数据在前端和InGame环境的不同处理程序
- **我的期望**：框架提供很多便利的API，旨在扩展mod开发者的创作空间和提升开发效率，同时为玩家带来更佳体验。
- **开放反馈**：我欢迎任何建议和反馈，以不断优化框架，使其更好地服务于mod开发者和玩家社区。  
- **PS**：我会在本指南记中尽量使用通俗易懂的语言来讲解，因此在专业术语上可能会有所欠缺，我会尽可能并提供一些示例代码，供大家参考。  
在Lua中，使用setmetatable设置的表虽非传统“类”，但为实现面向对象编程，常被视为“类”或“伪类”，后续指南中将沿用此称呼。

通过本指南，我们希望帮助mod开发者和玩家深入理解框架的设计意图和使用规范，从而高效、安全地利用该框架。

---

## 三.框架结构概述
本框架被划分为两个主要子部分，以满足不同类型的数据存储需求：
### 1. Mod配置数据管理
- **功能定位**：这部分专注于管理mod的设置和配置数据，这些数据是mod开发者提供给玩家的，允许玩家根据个人偏好进行自定义。这些配置数据通常涉及游戏界面、控制选项、等可调整的参数。
- **API支持**：框架提供了全面的API支持，简化了配置数据的处理流程。mod开发者只需要提供设置的键（key）和默认值以及发生更改时使用的回调函数。而无需编写对应UI和数据更改/保存逻辑，这些框架会已经帮你处理好。
- **数据更改/保存逻辑**：当玩家通过UI更改设置时，框架自动处理数据的更新和保存，确保玩家自定义的设置能够即时生效并持久保存。
### 2. Mod非配置数据管理
- **功能定位**：这部分负责处理mod的非配置数据，即那些不直接暴露给玩家设置的数据。
- **API提供**：框架同样提供了设置默认值、更改数据和保存数据的API，但与配置数据不同的是，这些数据的具体操作需要mod开发者自行定义和实现。
- **自定义操作**：开发者可以根据mod的具体需求，定义数据的存储、更改等操作。例如，可以用于记录领袖的挑战进度、实现跨存档的能力解锁等，为开发者提供了极大的灵活性和创造空间。
通过这种结构设计，框架既能够满足玩家对配置数据自定义的需求，又能够为mod开发者提供处理复杂游戏状态的强大工具，从而共同提升游戏体验和扩展游戏的可玩性。
### 3. 框架中Mod数据的结构
- **数据结构**：框架本质是Lua表，CurrentlyModDatas子表负责存储当前所有mod的数据。每个使用框架的mod在CurrentlyModDatas中都有一个{_localData={}, _snycData={}}结构的子表，用于存储数据，并且这个子表拥有元表属性。
- **元表属性**：该表的元表属性允许以表为对象添加参数，自动根据键值分配到_localData或_snycData中。例如，当设置CurrentlyModDatas[modkey][key] = value时，若键以"_"开头，则自动分配到_snycData；否则分配到_localData。具体细节后续会详细讲解
- **_snycData**：存储需要在联机游戏中同步的数据。框架在联机游戏中自动同步这些数据，具体操作将在后续联机指南中详细讲解。
- **_localData**：存储无需同步的数据。框架不会自动同步这些数据，但如有需要，开发者可以通过框架API手动进行同步，

---

## 四.框架的使用
本部分将介绍如何使用框架，内容可能会逐步完善，请根据以下步骤操作。  

框架的数据操作是在游戏的UI-lua环境中进行的，因此需要使用UI Lua来操作数据。  
### 1. 框架引入
首先，确保您的模组（mod）向框架注册。需要在游戏数据库中填写ModDataIds表，这是至关重要的，原因如下：
- **避免ID冲突**：因为直接在Lua中添加参数可能导致不同mod使用相同ID，从而引起干扰。通过在SQLite数据库中实施ModId和DataId的唯一性约束，可以在数据插入阶段进行初步的ID检查，有效避免潜在的冲突。

以下是在数据库中插入ModDataIds的SQL和XML示例：

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
> <details><summary>ModDataIds表的字段说明</summary>
> 
> 列名 | 说明
> --- | ---
> ModId | 您的mod的唯一标识，即您在modinfo文件中使用的Mod id。
> DataId | 本地数据ID，用于在Lua中操作数据。<br>注意：DataId应当只包含字母数字和下划线，因为框架会根据这个ID定义对应的LuaEvents，以便在Lua中进行数据操作，具体使用方法将在后面介绍。
> Version | 代表的是mod数据的版本号，而非mod本身的版本。<br>当您的mod需要进行数据更新时，必须更新这个版本号。<br>框架会根据这个版本号来决定是否应该覆盖现有的数据。<br>在默认情况下，框架会继续使用之前存储的数据来填充Lua变量，只有在版本号发生变化时，才会进行数据覆盖。<br>Lua还提供了一套接口，用于在版本更新时根据旧数据执行必要的更新操作。关于如何使用这些接口的详细说明，将在后续章节中提供。
> </details>

> <details><summary>相关源码（以对DataId进行约束）</summary>
> 
> ```sql
> CREATE TABLE ModDataIds (
>     ModId TEXT PRIMARY KEY, -- ModId 是主键，也有UNIQUE 约束，不允许重复
>     DataId TEXT NOT NULL UNIQUE, -- UNIQUE 约束确保 DataId 字段的值在整个表中是唯一的
>     Version INTEGER NOT NULL DEFAULT 1
>     -- EditEnvironment INTEGER NOT NULL DEFAULT -1 CHECK (EditEnvironment IN (-1, 0, 1)) -- 弃用，每个mod配置参数在lua设置环境
> );
>```
> </details>
---

### 2. Mod数据表及键值规范
- 框架会为每个使用框架的mod 提供一个预设的数据表结构{_localData={}, _snycData={}}, 以下简称该结构为“Mod数据表”。
- Mod数据表存储于框架的 CurrentlyModDatas 子表中，键值是由 ModUUID 生成唯一的字符串标识 ModKey。
- 可以通过框架直接访问Mod数据：MLDM.CurrentlyModDatas[modKey]。
> **命名规范要点**：mod开发者在使用Mod数据表进行键值赋值时，必须遵循以下命名规则： (原因参考下面给与mod数据表的元表属性) 
> - 键值前缀规范:  
>   - 若键值以“_”开头，则自动分配到_snycData，即同步数据, 框架在联机游戏中自动同步这些数据。  
>   - 若键值不以“_”开头，自动分配到_localData，即非同步数据, 框架不会自动同步这些数据，但如有需要，开发者可以通过框架API手动进行同步。 
> - 禁止使用的键值：   
>   - 请勿使用“_localData”或“_syncData”作为键值，除非您的意图是直接修改Mod数据表的 _localData 和 _syncData 子表本身。  
> - 保持Mod数据表结构：  
>   - 如果您确实需要替换 _localData 或 _syncData 子表，请确保不改变它们的数据类型。维持正确的数据类型是确保元表属性正常运作的关键。
>   - 不要使用rawset添加赋值,它会忽略元表的__newindex元方法，直接赋值到Mod数据表，会有潜在键值相同导致数据更改/获取错误。
>  
> 遵守以上规范将有助于确保Mod数据表的功能性和稳定性，避免潜在的运行时错误。

Mod数据表具备元表属性，实现了 _localData 和 _syncData 的隐性表结构。以下为具体实现示例：
> <details><summary>Mod数据表的元表-应用例子</summary>
> 
> ```lua
> local modData = MLDM.CurrentlyModDatas[modKey]
> 
> -- 以下是对modData的操作
> modData['key1'] = 'value1' -- 会自动分配到_localData，此时modData = {_localData={key1 = 'value1'}, _snycData={}}
> print(modData['key1']) -- 直接输出 'value1'
> modData['_key1'] = 'value2' -- 会自动分配到_snycData，此时modData = {_localData={key1 = 'value1'}, _snycData={_key1 = 'value2'}}
> print(modData['_key1']) -- 直接输出 'value2'
> -- 当你想要遍历自己modData时，不要直接使用 pairs
> for k, v in pairs(modData) do
>     print(k, v)
> end
> -- 输出：_localData table: 0x12345678
> --       _snycData table: 0x87654321
> -- 应当如下遍历
> for k,v in modData() do
>     print(k, v)
> end
> -- 输出：key1 value1
> --       _key1 value2
> 
> modData['key2'] = 'value3' -- 此时有3个数据
> 
> -- 想要获取当前modData的长度时，不要使用 # 或table.count，而是使用modData(true)函数
> print(#modData) -- 输出 0 因为modData不是数组表，#返回0
> print(table.count(modData)) -- 输出 2 同上,是_localData和_snycData这两个元素的数量
> print(table.count(modData(true))) -- 输出 3 会获得_localData子表和_snycData子表的总数量
>
> modData['_snycData'] = {} -- 直接修改_snycData子表，清空原有同步表数据，此时modData = {_localData={key1 = 'value1'}, _snycData={}}
>
> -- 下面是错误的示范1：
> -- 不要这样作 因为_syncData和_localData是框架预留的关键字，它的在mod数据表中的类型应当是表
> modData['_localData'] = 1 -- 此时modData = {_localData=1, _snycData={}} -- _localData错误的类型，应当是表
> -- 此时在给mod数据表进行赋值，会在调用元表的__newindex元方法触发报错，因为_localData的类型已经不是表，无法添加数据
> modData['key3'] = {1,2,3}
>
> -- 下面是错误的示范2：
> modData['_localData'] = {key1 = 'value1'} -- 先恢复正确的mod数据表，此时modData = {_localData={key1 = 'value1'}, _snycData={}}
> -- 不要使用rawset来赋值，因为这样会忽略元表的__newindex元方法，直接赋值到Mod数据表，会有潜在键值相同导致数据更改错误
> rawset(modData, 'key1', 'value0') -- 此时 modData = {_localData={key1 = 'value1'}, _snycData={}, key1 = value0}
> print(modData['key1']) -- 输出 'value0' ,键值查询也不会调用__index元方法
> ```
> </details>

此设计旨在简化数据管理，同时兼顾联机模式下的同步数据快速管理以及单机模式下的数据统一管理。
> <details><summary>元表源码</summary>
> 
> - 我在PK_MetaTableUtility.lua设置两种元表如果有需要可以去include使用
> - 这也是一个很好的学习lua元表的例子(给lua萌新的建议)
> - 一种数据管理方案，直接根据键值不同对数据进行分配，同时又能兼顾整体数据（好吧跑题了，哈哈）
> 
> ```lua
>    -- 省略代码...
>    local function iterator(tbl1, tbl2) -- 通过__index和__newindex已经约束两个表不会有相同键值存在，因此这里无需检测键值对相同情况
>     -- 初始化两个表的键迭代器
>     local next1, next2 = next, next
>     local k, v, state = nil, nil, 1 -- 状态: 1 时 syncData, 2 时 localData
> 
>     return function()
>       if state == 1 then
>         k, v = next1(tbl1, k)
>         if k ~= nil then return k, v end
>         state = 2
>       end
>       k, v = next2(tbl2, k)
>       return k, v
>      end
>    end
>    -- 省略代码...
>     ModDataMetatable = { -- 构建mod数据表的元表
>       -- 当访问一个表中不存在的键时, 应当到_syncData和_localData检查数据
>       __index = function(t, k) return k:sub(1, 1) == "_" and t._syncData[k] or t._localData[k] end,
>       -- 当表中的一个不存在的键赋值时，自动根据是否以_开头来判断是否是同步数据，如果是则添加到_syncData中,不是加入到_localData中
>       __newindex = function(t, k, v)
>         if k:sub(1, 1) == "_" then
>           t._syncData[k] = v
>         else 
>           t._localData[k] = v
>         end
>       end,
>       -- 文明6的lua应该是lua5.1改编的版本，不支持__pairs和__len这两个元方法
>       --__pairs = function(t) return iterator(t._syncData, t._localData) end, -- 返回合并的迭代器
>       --__len = function(t) return table.count(t._syncData) + table.count(t._localData) end, -- 返回总长度
>       -- 这里使用__call来实现__pairs和__len两个元方法功能
>       __call = function(t, isLen:boolean) return isLen and (table.count(t._syncData) + table.count(t._localData)) or iterator(t._syncData, t._localData) end, -- 返回合并的迭代器// 返回总长度
>     },
>     -- 省略代码...
> ```
> </details>

### 3. MOD版本变更时对玩家既有数据的处理策略
在MOD开发过程中，数据默认值及数据结构可能会因开发迭代而发生变化。同时，也可能存在MOD版本回退的情况。以下为框架针对此类情况的处理方法。

#### 3.1 框架处理机制
- 关于MOD数据的版本设置，在之前的“1. 框架引入”部分已有提及，具体是在SQLite数据库中的ModDataIds表中的Version字段进行定义。 

当MOD数据版本发生更迭（包括升级和降级）时，框架默认采用新数据覆盖旧数据。然而，框架也为MOD开发者预留了自定义处理机制，以便在版本更迭时，能够提供更为灵活的处理手段。

#### 3.2 自定义处理机制
- 自定义处理机制主要通过MOD开发者提供的回调函数来实现。回调函数的具体规范如下：
> ```lua
> -- 注意升级的回调函数和降级的回调函数结构上是一样的，这里不分开说明
> function CustomModDataVersionHandler(oldVersion, newVersion, oldData, newData)
>   -- 处理Mod数据的版本更迭
>   -- oldVersion: 旧版本号
>   -- newVersion: 新版本号
>   -- oldData: 存储的旧数据
>   -- newData: 新版本的默认数据
>
>   -- 你的处理逻辑,省略...
> 
>   -- 框架会接收两个参数返回值：
>   --   newNewData : 处理后的新数据
>   --   isRequiredModToSets : 是文本变量(LOC_...)，用于是否提醒玩家mod配置发生更改需要重新设置mod的配置（仅用于Mod配置数据管理）
>   return newNewData, isRequiredModToSets
> end
> -- 注册Mod数据版本更迭的回调函数，用于处理版本更迭的回调函数(可以分为两个函数，也可以合并为一个函数，当然也可以不注册回调函数)
> MLDM.ModUpdateHashrs[modKey] = CustomModDataVersionHandler -- 注册Mod数据版本升级的回调函数，用于处理版本升级的回调函数
> MLDM.ModRollbackHashs[modKey] = CustomModDataVersionHandler -- 注册Mod数据版本回退降级的回调函数，用于处理版本回退降级的回调函数
> ```

> <details><summary>部分相关源码</summary>
>
> ```lua
>           if newModConfig ~= {} then
>             for k, v in pairs(newModConfig) do
>               newModVersions[k] = newModVersions[k] or 0
>               if not self.IgnoreModVersionUpdates[k] then
>                 if oldModConfig[k] then -- 对比保存的数据, 确认是否需要更新
>                   if newModVersions[k] > oldModVersions[k] then
>                     if self.ModUpdateHashrs[k] then -- 如果该mod存在modder自定义的数据调整方案, 则按这个方案调整
>                       -- 需要返回修改后的数据和是否需要提醒玩家重新配置数据(如果是需要调配的mod)
>                       oldModConfig[k], self.RequiredModToSets[k]  = self.ModUpdateHashrs[k](oldModVersions[k], newModVersions[k], oldModConfig[k], newModConfig[k])
>                     else
>                       -- -- 默认新的版本数据覆盖旧的数据, 既CurrentlyModDatas[k]当前版本既默认值，所以无需调整，只用标记
>                       self.RequiredModToSets[k] = 'LOC_PK_CIV6_MLDM_LEFT_BUTTION_NEW_TT_0' -- 标记以便提醒玩家需要重新调整该mod的配置(如果是需要调配的mod)
>                     
>                       oldModConfig[k] = v -- 反转数据, 新的mod数据覆盖旧的mod数据
>                     end
>                   elseif newModVersions[k] < oldModVersions[k] then
>                     if self.ModRollbackHashs[k] then -- 如果该mod存在modder自定义的回退版本方案, 则按这个方案调整
>                       -- 需要返回修改后的数据和是否需要提醒玩家重新配置数据(如果是需要调配的mod)
>                       oldModConfig[k], self.RequiredModToSets[k] = self.ModRollbackHashs[k](oldModVersions[k], newModVersions[k], oldModConfig[k], newModConfig[k]) -- 需要返回修改后的数据和是否需要提醒玩家重新配置数据
>                     else
>                       -- -- 默认恢复为旧版本默认数据既CurrentlyModDatas[k]当前版本既默认值，所以无需调整，只用标记
>                       self.RequiredModToSets[k] = 'LOC_PK_CIV6_MLDM_LEFT_BUTTION_NEW_TT_1' -- 标记以便提醒玩家需要重新调整该mod的配置(如果是需要调配的mod)
>                       oldModConfig[k] = v -- 反转数据, 新的mod数据覆盖旧的mod数据
>                     end
>                   else
>                     -- 不需要更新的mod数据则保持原有的数据
>                     -- self.CurrentlyModDatas[k] = oldModConfig[k] -- 反转这里已经是oldModConfig[k]
>                   end
>                 else
>                   self.RequiredModToSets[k] = 'LOC_PK_CIV6_MLDM_LEFT_BUTTION_NEW_TT_2' -- 提醒玩家这个mod是新增加的，建议根据需要进行自定义配置(如果是需要调配的mod)
>                   table.insert(ModHashRecord, k) -- 添加新的ModHash到记录表中
>                   oldModConfig[k] = v -- 添加新的mod数据
>                 end
>               else
>                 self.RequiredModToSets[k] = 'LOC_PK_CIV6_MLDM_LEFT_BUTTION_NEW_TT_3' -- 提醒modder这个mod还处于调试的mod状态
>                 oldModConfig[k] = v -- 调控模式使用保持最新的mod默认数据，不使用之前存储的数据
>               end
>               -- 最后更新最新mod版本
>               oldModVersions[k] = newModVersions[k] -- 添加新的mod版本
>             end
>           end
> 
>           self.CurrentlyModDatas = oldModConfig
>           self.ModVersions = oldModVersions
>           
>           -- 存储最新的ModHashRecord和ModVersions
>           self.CurrentlyModDatas[m_CoreModKey].ModHashRecord = ModHashRecord
>           self.CurrentlyModDatas[m_CoreModKey].ModVersions = oldModVersions -- 保存最新的版本
> ```
> </details>

### 4. 关于框架Mod数据保存
由于所有mod数据都存储在一个配置存档中，频繁的保存操作可能会影响性能和游戏体验，因此建议减少不必要的频繁保存。
#### 4.1 框架的保存机制
- **配置数据**：对于通过"Add...ParamUI" API注册的配置数据，框架会自动保存玩家所做的更改。  
- **非配置数据**：非配置数据的保存则需要根据环境不同而有所不同。
  - **前端环境**：在所有数据更改完成后，需要手动调用保存数据的API。
  - **游戏内(InGame)环境**：框架会在保存游戏存档时自动保存所有mod数据，同时也提供了主动保存的API。

#### 4.2 数据保存原理和存储文件
- **具体原理概述**：使用GameConfiguration.SetValue来以‘配置参数’的形式存储在配置存档，并将配置存档保存到玩家本地。在下次启动游戏时加载配置存档后读取出这些数据(GameConfiguration.GetValue)
- **存储文件**：默认保存文件名为‘PKUI_ModData.Civ6Cfg’,一般位于C:\Users\[Your username]\Documents\My Games\Sid Meier's Civilization VI\Saves\Single

如何更改默认命名：(Mod的MyModDataFileNameX.lua文件有讲内容如下)
```lua
-- === Mod数据存储文件名 === --
-- 若您希望自定义Mod数据存储的配置存档名称，请修改以下变量DefaultModDataFileName的值。
-- 需要注意的是，修改后请将文件名后面的大写字母“X”去掉。
-- 例如，将文件名从“MyModDataFileNameX”更改为“MyModDataFileName”。

-- === Mod Data Storage Filename === --
-- If you wish to customize the name of the configuration archive for mod data storage, please modify the value of the variable DefaultModDataFileName below.
-- Note that after making the change, you should remove the uppercase letter "X" from the end of the filename.
-- For example, change the filename from “MyModDataFileNameX” to “MyModDataFileName”.
DefaultModDataFileName = "PKUI_ModData"

-- 重要提示：
-- 1. 更改文件名后，该文件将不会随Mod更新而更新。
-- 2. 如果您取消订阅该Mod或卸载文明6，此文件不会被自动删除，需要您手动操作。
-- 3. 若您只是暂时不玩文明6而卸载游戏，建议保留此文件，以免忘记Mod数据存档的文件名称。
-- 4. 如果您计划长期不玩文明6，建议备份此文件，以记录您的Mod数据配置存档名（.Civ6Cfg）。
-- 5. 同时，请确保备份对应的Mod数据配置存档文件（.Civ6Cfg），以免数据丢失。
-- 6. 一般Mod数据配置存档文件（.Civ6Cfg）位于C:\Users\[Your username]\Documents\My Games\Sid Meier's Civilization VI\Saves\Single

-- Important Notes:
-- 1. After changing the filename, this file will not be updated with mod updates.
-- 2. If you unsubscribe from this mod or uninstall Civilization VI, this file will not be deleted automatically and will require manual action from you.
-- 3. If you are only uninstalling Civilization VI temporarily and not planning to play, it is recommended to keep this file to avoid forgetting the filename of the mod data archive.
-- 4. If you are planning to take an extended break from playing Civilization VI, it is advised to back up this file to keep a record of your mod data configuration archive name (.Civ6Cfg).
-- 5. Also, please make sure to back up the corresponding mod data configuration archive file (.Civ6Cfg) to prevent data loss.
-- 6. Generally, the mod data configuration archive file (.Civ6Cfg) is located at C:\Users[Your username]\Documents\My Games\Sid Meier's Civilization VI\Saves\Single

```



### 5. 关于网络多人联机数据同步问题
**数据影响评估**：Mod开发者应评估自己Mod数据使用中，是否会影响游戏联机同步。例如，仅用于玩家个性化自我展示的UI界面的配置数据不会影响同步，而Game环境Lua中不同玩家使用的数据则可能导致同步问题。

**官方API利用**：对于可能引起同步问题的数据，我们推荐使用官方API `UI.RequestPlayerOperation` 来确保数据的一致性。此API能够处理大多数同步需求。
- 实际也可以用于实现UI数据同步，只是需要先同步到Game环境在传到UI环境，但在前端联机房间中时不存在Game环境

**本框架提供的API同步**：是直接UI环境进行同步，不需要Game环境参与。因此支持在前端联机房间中直接同步数据。
- 例如对于需要在前端联机房间UI界面直接展现玩家个性化称号。

在深入探讨本框架的同步机制之前，我们先简要介绍官方联机模式下的数据同步流程，以便mod开发者根据自身需求判断是否采用本框架的同步方案。
#### 4.1 官方的联机
以下是对官方联机模式下数据同步机制的理解，如有不准确之处，欢迎指正。
- 要了解概念：游戏配置‘GameConfiguration’，地图配置‘MapConfiguration’，玩家配置‘PlayerConfiguration’，UI.RequestPlayerOperation等等。

**UI环境同步**：
- **初始配置数据生成**：
- 若玩家为房主，系统将根据其设置生成相应的配置数据（包括游戏配置和地图配置）。
- 玩家如果不是房主时，在进入房间时，会自动同步当前房间内的配置数据（包括游戏配置和地图配置,已经房间内当前玩家的玩家配置数据）。
- **房间内数据同步规则**：
  - 只有房主有权修改游戏配置数据（包括游戏配置和地图配置），并可将更改同步给其他玩家。（非房主我测试lua更改本地配置成功但无法成功广播，并且这样做后有可能会数据不同步无法正常联机）
  - 各个人类玩家的玩家配置数据，仅允许该玩家自行修改，并可将修改同步给其他玩家。（本框架同步机制即基于此原理。）

**Game环境同步**：
- 官方并未提供直接的Game环境同步机制，而是通过UI环境的官方API‘UI.RequestPlayerOperation’调用Game环境函数实现同步。
- 例如，使用‘UI.RequestPlayerOperation’可实现单个玩家点击按钮调用Game环境函数，并将结果同步给其他玩家。
- 虽然理论上‘UI.RequestPlayerOperation’可用于数据同步，但存在以下局限性：
  - Game环境为InGame环境的子环境，无法在前端联机房间阶段使用，仅能在游戏内使用。
  - Game环境数据传递至UI环境时，存在一定延迟。
> <details><summary>UI.RequestPlayerOperation和Game数据快速传递UI使用示范</summary>
>
> - 关于这个官方接口的使用示范
>
> 首先是在UI环境使用，可以调用对应的Game环境，用GameEvents绑定的函数
> ```lua
> function UICallMyGameEventsName(playerID)
>    --先创建一个table，这个表包含调用的Game环境的GameEvents名以及参数
>    local kParameters:table = {};
>    kParameters.OnStart = 'MyGameEventsName' -- 这里是GameEvents的名字，需要设定固定key值'OnStart'
>    -- 参数key和value可以随意设置,我这里演示三个需要传递给Game环境的参数
>    kParameters.Value1 = 'Hello' -- 这里是你需要传递的参数1
>    kParameters.Value2 = 'World' -- 这里是你需要传递的参数2
>    kParameters.Value3 = '!'
>    -- kParameters.Value4 = 'xxx'
>    -- 省略...
>    -- kParameters.ValueN = 'nnnn'
>    -- 然后使用UI.RequestPlayerOperation来调用GameEvents.MyGameEventsName绑定的函数
>    -- playerID是玩家id参数, PlayerOperations.EXECUTE_SCRIPT 是文明6lua全局常量
>    UI.RequestPlayerOperation(playerID, PlayerOperations.EXECUTE_SCRIPT, kParameters)
>    
>    -- 在扩展一个简单的Game传递数据给UI的例子
>    local delayedExecution
>    delayedExecution = function()
>      local isSuccess = Players[playerID]:GetProperty('MyGameEventsNameRanSuccessfully')
>      if isSuccess
>        if isSuccess == true then
>          Controls.TextLabel.SetText("Success to call GameEvents.MyGameEventsName")
>        else
>          Controls.TextLabel.SetText("Failed to call GameEvents.MyGameEventsName")
>        end
>        -- 成功UI环境获得Game环境SetProperty的数据，需要及时Remove，否则delayedExecution还会继续1秒执行60次
>        Events.GameCoreEventPublishComplete.Remove(delayedExecution)
>      end
>    end
>    -- 实际测试GameCoreEventPublishComplete是一个1秒执行60次的Event，可以用来做计时器
>    -- 关于文明6lua计时器我有专门的研究总结：
>      -- https://gitee.com/XPPK/pk-civ6-LuaTimer
>      -- https://github.com/X-PPK/Civilization-6Lua-Timer
>      -- 同时我在PKUI框架中还有另一个相关的计时器lua脚本'UITimerManager.lua'比上面更加高级，可以满足更复杂的需求。
>    -- 实际测试在Game环境SetProperty后，GameCoreEventPublishComplete事件需运行第二次，UI环境GetProperty可以成功获得数据。
>    -- 也就是从Game环境SetProperty到UI环境GetProperty的延迟时间是小于1/60秒的，这种延迟是不影响玩家体验的
>    Events.GameCoreEventPublishComplete.Add(delayedExecution)
>  end
> ```
> 对应的Game环境lua代码，需要定义对应的GameEvents和函数
> ```lua
> -- 定义函数
> function MyGameFunction(playerID, kParameters)
>   PlayerConfigurations[playerID]:SetValue('MyGameEventsNameRanSuccessfully', (kParameters.Value1 == 'Hello' and kParameters.Value2 == 'World' and kParameters.Value3 == '!'))
> end
> -- 注册GameEvents,并绑定函数
> GameEvents.MyGameEventsName.Add(MyGameFunction)
> ```
> </details>

#### 4.2 框架的同步机制
本框架在设计初期未充分考虑同步问题，因此相关设计可能不尽完善。
- **同步数据表**：每个mod数据表均包含一个_snycData子表，用于存储同步数据。框架将自动同步该子表数据至其他玩家游戏。
- **个性化同步实现**：若mod开发者需实现个性化同步，可不必使用_snycData子表，而是自行编写同步逻辑（框架提供UI环境同步API）。框架仅负责同步_snycData子表数据。
- **游戏开始前同步**：框架将在游戏开始前（前端联机房间阶段）自动同步_snycData子表数据。
- **数据获取差异**：
  - _localData数据：按正常框架方式获取。
  - _snycData数据：分为前端和InGame环境两种情况：
    - 在前端，非联机房间时_snycData数据获取方式与_localData相同，联机房间则可以通过玩家ID获取各个人类玩家的_snycData数据。
    - 在InGame环境，需通过玩家ID获取，我这样设计目的是使得单机和联机mod开发者只需要设置一种获取方式，不需要考虑游戏是单机还是在联机。
    - 注意：在InGame环境，本地框架的_snycData子表仅包含本地玩家的同步数据，非所有联机玩家的同步数据。
**如何通过玩家ID获取_snycData子表数据**：
- 使用场景：在前端环境的联机房间中 /在InGame环境
- 需通过游戏中玩家ID获取数据，示例代码如下：
```lua
local isMultiplayerRoom = MLDM:IsMultiplayerRoom() -- 框架提供的API，判断当前是否是联机房间/联机的游戏中
local playerModSyncDataStr = PlayerConfigurations[playerId]:GetValue("MLDM_ModSyncData_" .. modDataId)
local playerModSyncData = deserialize(playerModSyncDataStr) -- deserialize函数来自mod的CIV6_Serialize.lua，如果是ModLocalDataManager_脚本，则可以直接使用，无需在include("CIV6_Serialize")
```
> <details><summary>部分相关源码</summary>
>
> ```lua  GetLocalPlayerID = function(self)
>   local iLocalPlayerID:number = -1;
>   if Network.IsInGameStartedState() then
>     iLocalPlayerID = Game.GetLocalPlayer();
>   else
>     iLocalPlayerID = Network.GetLocalPlayerID();
>   end
>   return iLocalPlayerID;
> end,
> IsMultiplayerRoom = function(self)
>   if (UI.IsInFrontEnd()) then
>     return self.MultiplayerPlayerRecord and true or false
>   else
>     return GameConfiguration.IsNetworkMultiplayer()
>   end
> end,
> -- 同步指定Mod数据到其他玩家
> SyncModDataToOtherPlayers = function(self, dataId, modSyncUpdateData:table)
>   local localPlayerID = self:GetLocalPlayerID()
>   if localPlayerID == -1 then return end
>   if type(dataId) == 'table' then -- 此时更新多个mod的同步数据
>     if table.count(dataId) == 0 then return end -- 无实际的更新数据，不进行同步
>     if not modSyncUpdateData then
>       modSyncUpdateData = {}
>       for i, idataId in pairs(dataId) do
>         modSyncUpdateData[i] = self.CurrentlyModDatas[self.ModDataIds[idataId]]._syncData
>       end
>     end
>     dataId = serialize(dataId)
>   elseif type(dataId) == 'string' then -- 此时更新一个mod的同步数据
>     modSyncUpdateData = modSyncUpdateData or self.CurrentlyModDatas[self.ModDataIds[dataId]]._syncData
>   else
>     print("MLDM:SyncModDataToOtherPlayers dataId参数类型错误")
>     return
>   end
>   local PlayerConfig = PlayerConfigurations[localPlayerID]
>   local serializedModSyncUpdateDataStr = serialize(modSyncUpdateData)
>   -- 采用打包方式统一管理，一起同步
>   PlayerConfig:SetValue("MLDM_ModSyncDataUpdateId", dataId)
>   PlayerConfig:SetValue("MLDM_ModSyncDataUpdate", serializedModSyncUpdateDataStr)
>   print('成功同步数据SyncModDataToOtherPlayers')
>   -- 通知其他玩家同步数据
>   Network.BroadcastPlayerInfo(localPlayerID)
>   -- 修复bug
>   -- 同时本地玩家也需要更新同步数据，因为MLDM_ModSyncDataUpdateId，和MLDM_ModSyncDataUpdate是打包的
>   -- 否则modder还需要额外区分玩家是否是本地玩家采用不同的获取snycData的方式
>   -- 因此本地玩家也应当解包MLDM_ModSyncDataUpdateId，和MLDM_ModSyncDataUpdate的数据
>   -- 这种设计也是为了兼顾不同的同步情况，例如一个mod又更改了个别数据，那么只用同步这部分数据，通过MLDM_ModSyncDataUpdate缓存过度，数据传到其他端，其他端根据这个更改
>   self:ReceiveModDataSyncFromOtherPlayers(localPlayerID)
> end,
> -- 接收其他玩家的Mod数据同步请求
> ReceiveModDataSyncFromOtherPlayers = function(self, playerId:number)
>   local PlayerConfig = PlayerConfigurations[playerId]
>   local modSyncDataUpdateId = PlayerConfig:GetValue("MLDM_ModSyncDataUpdateId")
>   if modSyncDataUpdateId then
>     local updateSingleMod = self.ModDataIds[modSyncDataUpdateId]
>     local needUpdateDatas = deserialize(PlayerConfig:GetValue("MLDM_ModSyncDataUpdate"))
>     needUpdateDatas = updateSingleMod and {needUpdateDatas} or needUpdateDatas
>     local DataIds = updateSingleMod and {modSyncDataUpdateId} or deserialize(modSyncDataUpdateId)
>     
>     for i, dataId in ipairs(DataIds) do
>       local oldModSyncDataStr = PlayerConfig:GetValue("MLDM_ModSyncData_" .. dataId)
>       local newModSyncData = oldModSyncDataStr and deserialize(oldModSyncDataStr) or {}
>       local modUpdateData = needUpdateDatas[i]
>       
>       -- 根据传送过来mod数据更新表，一个一个键值更改对应数据，而不是直接替换整个表，以减小网络传输量
>       -- 例如一个mod的当前同步数据是{a=1,b=2,c=3}，新的mod的同步数据是{a=4,b=2,c=3}，那么只需要更新a=4，那么只需要发送{a=4}，而不用发整个表
>       for k, v in pairs(modUpdateData) do
>         newModSyncData[k] = v
>       end
>       -- 每个mod单独一个存储ID
>       PlayerConfig:SetValue("MLDM_ModSyncData_" .. dataId, serialize(newModSyncData))
>       LuaEvents['MLDM_ModSyncDataSync_' ..dataId](playerId,newModSyncData) -- 通知对应mod的同步数据发生更新事件
>     end
>     -- 已经完成更新，及时清除
>     PlayerConfig:SetValue("MLDM_ModSyncDataUpdateId", nil)
>     PlayerConfig:SetValue("MLDM_ModSyncDataUpdate", nil)
>   end
> end,
> -- 清除玩家的同步数据
> ClearSyncDataByPlayeId = function(self, PlayeId)
>   -- 数据是玩家独有的，玩家离开房间自动清除
>   -- 当玩家在联机房间内离开或者被踢需要清除玩家的同步数据
>   local PlayerConfig = PlayerConfigurations[PlayeId]
>   for dataId, modkey in pairs(self.ModDataIds) do
>     local moduuid = self.ModHash[modkey]
>     if Modding.IsModEnabled(moduuid) then
>       if table.count(self.CurrentlyModDatas[modkey]._syncData) > 0 then
>         PlayerConfig:SetValue("MLDM_ModSyncData_" .. dataId, nil) -- 清除玩家的同步数据
>         LuaEvents['MLDM_ModSyncDataClear_' ..dataId](PlayeId,{}) -- 通知对应mod的同步数据发生清除，请给予默认数据/或其他操作
>       end
>     end
>   end
> end,
> init = function(self)
>   -- 省略其他代码...
>   -- 当处于联机游戏时(包括前端联机房间到处于联机游戏的Ingame环境)，需要监听玩家信息变化，避免错过数据同步请求
>   if (UI.IsInFrontEnd()) then
>     -- 监听玩家信息变化，如果玩家信息变化，则同步对应玩家数据
>     local function OnPlayerInfoChanged(PlayerID)
>       local islocalPlayer = PlayerID == self:GetLocalPlayerID()
>       local PlayerConfig = PlayerConfigurations[PlayerID];
>       
>       local old = self.MultiplayerPlayerRecord[PlayerID]
>       local oldIsHuman = old and old.isHuman
>       local newIsHuman = PlayerConfig:GetSlotStatus() == SlotStatus.SS_TAKEN
>       local new = {isHuman = newIsHuman}
>       if newIsHuman then
>         local newNetworkId = PlayerConfig:GetNetworkIdentifer()
>         local newNickName = PlayerConfig:GetNickName()
>         new.networkId = newNetworkId
>         new.nickName = newNickName
>         local isNewPlayerJoining = not old -- 此时在MultiplayerPlayerRecord未被记录，应当是刚加入/创键房间的玩家ID
>         local isAIChangedToHuman = not oldIsHuman -- AI玩家槽位被人类玩家占用/切换
>         local isDiffHumanPlayer  = oldIsHuman and ( old.networkId ~= newNetworkId or old.nickName ~= newNickName) -- 人类玩家之间发生切换位置 (检测人类玩家ID和昵称是否发生变化)
>         -- 此时新的人类玩家加入/AI玩家变人类/人类玩家之间位置切换，如果是我们自己，需要广播我们的Mod数据，以便其他人同步我们的数据
>         if islocalPlayer then
>         -- 同步自己的数据，以保证同步
>         if isNewPlayerJoining or isAIChangedToHuman or isDiffHumanPlayer then SyncMyModData(self.CurrentlyModDatas) end
>         else
>         -- 检测是不是其他玩家的同步mod数据请求，如果是，则同步对应玩家数据
>         self:ReceiveModDataSyncFromOtherPlayers(PlayerID)
>         end
>         LuaEvents.MLDM_MultiplayerRoomHumanPlayerPositionChange(PlayerID) -- 玩家位置变化事件
>       else
>         -- 此时发生人类玩家变AI，需要清除对应ID数据,可能是该玩家离开游戏/被踢，亦或者玩家和AI位置切换，AI被换到人类玩家之前的位置
>         if oldIsHuman then
>         self:ClearSyncDataByPlayeId(PlayerID)
>         LuaEvents.MLDM_MultiplayerRoomHumanPlayerPositionChange(PlayerID) -- 玩家位置变化事件
>         end
>       end
>       self.MultiplayerPlayerRecord[PlayerID] = new
>     end
>     -- 前端联机房间中当玩家信息变化需要及时检测是否是数据同步请求
>     Events.MultiplayerJoinRoomComplete.Add(function()
>       --print("MLDM:MultiplayerJoinRoomComplete联机房间创建/加入完成", GameConfiguration.IsNetworkMultiplayer())
>       if GameConfiguration.IsNetworkMultiplayer() then
>         self.MultiplayerPlayerRecord = {}
>         print("开始监听玩家信息变化,以进行数据同步")
>         Events.PlayerInfoChanged.Add(OnPlayerInfoChanged)
>       end
>     end)
>     LuaEvents.Multiplayer_ExitShell.Add(function()
>       if self.MultiplayerPlayerRecord then
>         --print('退出联机房间，停止监听玩家信息变化')
>         self.MultiplayerPlayerRecord = nil -- 及时清除记录,也意味着不处于联机房间
>         Events.PlayerInfoChanged.Remove(OnPlayerInfoChanged)
>       end
>     end)
>   else
>     -- 游戏对局中如果人类玩家离场不清除数据，以便后续人类玩家重新联线。
>     -- 只需要正常处理有可能的同步请求即可
>     local function OnPlayerInfoChanged(PlayerID)
>       if PlayerID ~= self:GetLocalPlayerID() then self:ReceiveModDataSyncFromOtherPlayers(PlayerID) end
>     end
>     -- Events.LoadComplete
>     Events.LoadGameViewStateDone.Add(function(eResult, eType, eOptions, eFileType)
>       --if eFileType ~= SaveFileTypes.GAME_STATE then return end
>       if GameConfiguration.IsNetworkMultiplayer() then
>         Events.PlayerInfoChanged.Add(OnPlayerInfoChanged)
>         -- 联机模式已经在前端联机房间中处理SyncMyModData，这里不再处理
>       else
>         -- 正常将同步数据存入玩家,和联机保持一样，这样直接按联机模式中数据的获取方式来使用这部分数据
>         -- 目的是避免还需要要额外构造代码来区分单机模式和联机模式采取不同的数据获取，从而减小mod开发者的工作量
>         SyncMyModData(self.CurrentlyModDatas)
>       end
>     end)
>   end
> end,
> ```
> </details>


### 6. 如何在Lua脚本中使用框架数据
有多种方式可以在Lua脚本中使用框架数据，以下是一些关键参数的说明：
  - modUUID：mod的唯一标识，既你的Mod的modinfo文件中的ModId
  - modKey：mod的本地数据存储key，是由modUUID生成的唯一对应的字符串

**第一种**：是直接使用框架构造的LuaEvents等可以跨UI调用的方法来与框架进行交互的
- 这些API可以被其他UI Lua脚本使用，包括你定义的'ModLocalDataManager_'脚本。

**第二种**：将Lua脚本文件引入框架，让框架会自动加载并执行你的脚本
- 这种方法你可以直接针对框架的进行操作，毕竟本框架本质是基于Lua表和setmetatable构建的
  - 如果你真这样做，请确保不会影响到框架的稳定性和正常运行，以及考虑到其他Mod的兼容性，避免造成不必要的影响或破坏其他Mod的功能。
> - 相关源代码：框架会先加载相关lua，包括你添加的'ModLocalDataManager_脚本'lua文件，在执行框架初始化
> ```lua
> -- 加载所有的框架相关lua文件
> include( "ModLocalDataManager_", true );
> -- 最后初始化MLDM
> MLDM:init()
> ```

- 首先你需要定义一个lua脚本文件，并且这个lua脚本的**命名需要规范为是'ModLocalDataManager_'开头的lua文件**
- 同时在modinfo文件中，你需要采**用ImportFiles来导入这个lua脚本**
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

这样，框架会自动加载你的脚本，允许你更灵活地操作数据，并通过构造自己的LuaEvents与其他UI Lua脚本交互。
- **注意**：框架的'配置数据管理'功能的相关API很多不是使用LuaEvents，因此要用到这个方法，具体后续我会讲解到

---

## 五.框架Lua的API
> - **重点**：
- 前面也说到，部分API不是LuaEvents，因此需要将你自己的lua脚本文件引入到框架的lua中
- 那么我们分开来讲吧

### 1. ModLocalDataManager_脚本API
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

#### 1.1 GetKey
- 对象是MLDM，需要一个隐式的 “self” 参数，所以请使用:
- 用于获得mod数据在框架内存储使用的key，这个ModKey是框架内部用来存储数据的唯一标识，可以用来操作数据

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


#### 1.2 SetIgnoreModVersionUpdates
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

#### 1.3 SaveConfig
- 对象是MLDM，需要一个隐式的 “self” 参数，所以请使用:
- 不要被这里"Config"误导，其实这里的意思是"保存配置存档"
- 实际的功能是用于保存mod的数据，mod的数据会被框架一起存储到配置存档中，以便在下次启动游戏时，会自动读取之前存储的数据
- **注意**：它虽然有一个参数saveID(string),但这个参数是框架内部使用的，请不要使用它。  
其他具体注意细节在前面的"**2. 框架lua使用前言**"有说明。
> 操作演示：
> ```Lua
> -- 保存所有mod的数据到配置存档中（因为数据是存在一个配置存档所以每次都是所有mod数据一起保存）
> MLDM:SaveConfig()
> ```

> <details><summary>相关源码</summary>
> 
> ```Lua
>   -- 注意： saveID是框架内部使用的，请Mod开发者注意不要使用它，保持saveID=nil即可
>   SaveConfig = function(self, saveID:string)
>     -- 有待测试保存需要消耗的时间或许后续可以改进，避免多个mod同时调用时，重复保存最新的数据，应当利用计时器，对多余的保存请求进行合并
>     
>     print("ModLocalDataManager SaveModData")
>     local function Save()
>       local gameFile = {
>         Name = saveID or self.ConfigFileName,
>         Type = SaveTypes.SINGLE_PLAYER,
>         FileType = SaveFileTypes.GAME_CONFIGURATION,
>       };
>       if not saveID then
>         --GameConfiguration.SetValue(saveID or self.ConfigFilxeName, self.CurrentlyModDatas) -- 存入当前配置
>         local tempKeys = {}
>         for k, v in pairs(self.CurrentlyModDatas) do
>           -- 存储策略上，我更偏向于保守，因为我不知道文明6的具体存储机制
>           -- 虽然GameConfiguration.SetValue这里支持直接存储表，但在存储中很可能会有问题：字符串的键值变为哈希值，导致无法获得原有的字符串键值的表
>           -- 目前猜测可能和文明6的表存储的序列化有关，不清楚文明6存储表采用那种序列化存储/读取时采用那种反序列化方式
>           -- 总之我在完成框架中，初始版本是正常可以获取对应表，但在后续开发中遇到这个问题，导致字符串键值丢失，无法正常读取之前存储的数据
>           -- 现在的解决方案是，这里将表序列化后转为字符串，然后在读取时反序列化，以确保获得原表
>           -- 之所以不直接存储一个长字符串是我担心官方这个存储接口在存储字符串时，对字符串有长度限制
> 
>           -- 发现直接存储字符串不会被压缩可以直接在存档文件中明码显示出来
>           -- 也就是存档没有对字符串有压缩，那么这里增加压缩
>           -- 如果一个mod数据过大拆分存储，避免数据过大导致存储失败(压缩后的字符串每3000个字符就在GameConfiguration存储一次)
>           local compressedTab = PKSerializeAndCompressTab({_localData = v._localData, _syncData = v._syncData}, 1, 3000)
>           local strNum = #compressedTab
>           if strNum > 1 then
>             GameConfiguration.SetValue(k, strNum)
>             tempKeys[k] = strNum
>             for i, compressedStr in ipairs(compressedTab) do
>               GameConfiguration.SetValue(k .. "_" .. i, compressedStr)
>             end
>           else
>             GameConfiguration.SetValue(k, compressedTab) --每个mod单独一个配置ID存储
>           end
>         end
>         self:DataCache() -- 更新缓存数据
>         if (UI.IsInGame()) then
>           Events.SaveComplete.Add(function(eResult, eType, eOptions, eFileType )
>             if eFileType == SaveFileTypes.GAME_CONFIGURATION then
>               -- 已保存到本地, 及时当前游戏中这些数据
>               for k, _ in pairs(self.CurrentlyModDatas) do
>                  GameConfiguration.SetValue(k, nil)
>                  if tempKeys[k] then
>                    for i = 1, tempKeys[k] do
>                      GameConfiguration.SetValue(k .. "_" .. i, nil)
>                    end
>                  end
>               end
>             end
>           end, 1)
>         end
>       end
>       UIManager:SetUICursor( 1 ); -- 我忘记这个是干什么的了，算了保留，不影响运行
>       Network.SaveGame(gameFile);
>       UIManager:SetUICursor( 0 );
>     end
>     Save()
>   end,
> ```
> </details>

#### 1.4 IsMultiplayerRoom
- 对象是MLDM，需要一个隐式的 “self” 参数，所以请使用:
- 用于判断当前是否是联机房间/联机的游戏中
- 返回值：boolean，true表示当前是联机房间/联机的游戏中，false表示当前是不是联机房间/单机游戏
> 操作演示：
> ```Lua
> -- 如果是在前端这是判断是否是联机房间，如果是在游戏中则判断是否是联机游戏
> ；local isMultiplayerRoom = MLDM:IsMultiplayerRoom()
> ```

> <details><summary>相关源码</summary>
> 
> ```Lua
>   IsMultiplayerRoom = function(self)
>     if (UI.IsInFrontEnd()) then
>       return self.MultiplayerPlayerRecord and true or false
>     else
>       return GameConfiguration.IsNetworkMultiplayer()
>     end
>   end,
> ```
> </details>

#### 1.5 GetLocalPlayerID
- 对象是MLDM，需要一个隐式的 “self” 参数，所以请使用:
- 用于获得当前本地玩家的ID
- 返回值：number，当前本地玩家的ID
> 操作演示：
> ```Lua
> -- 如果是在前端的联机房间中，这里是玩家当前槽位的ID（注意在联机房间玩家可以切换玩家槽位会更换ID），如果是在游戏中则是当前本机的玩家的ID
> ；local localPlayerID = MLDM:GetLocalPlayerID()
> ```

> <details><summary>相关源码</summary>
> 
> ```Lua
>   GetLocalPlayerID = function(self)
>     local iLocalPlayerID:number = -1;
>     if Network.IsInGameStartedState() then
>       iLocalPlayerID = Game.GetLocalPlayer();
>     else
>       iLocalPlayerID = Network.GetLocalPlayerID();
>     end
>     return iLocalPlayerID;
>   end,
> ```
> </details>

#### 1.6 SyncModDataToOtherPlayers
- 对象是MLDM，需要一个隐式的 “self” 参数，所以请使用:
- 用于同步指定Mod数据到其他玩家
- 参数1：dataId：string/table，同步数据的ID，可以是字符串，也可以是table。如果是字符串，则同步一个modDataID对应mod的数据；如果是table，需要为数组表，数组中包含多个modDataID，同步多个对应mod的数据。
- 参数2：modSyncUpdateData：table，是需要更新的数据，如果不提供，则使用当前mod数据中的_syncData同步数据，会更新整个mod的_syncData。
  - 如果只需要更新部分参数，那么这里务必要提供该参数，需要包含要更改的键值对即可，不更改的键值对表中无需添加。
  - 例如只更新 modDataId='myModDataId'对应mod的_syncData.myParam1，则提供的参数为 MLDM:SyncModDataToOtherPlayers('myModDataId', { myParam1 = newValue })

> <details><summary>操作演示：</summary>
> 
> ```Lua
> -- 假设'myModDataId'对应mod的_syncData = { _myParam1 = 10, _myParam2 = 'hello' }
> local function GetlocalPlayerModSyncData()
>   local localPlayerID = MLDM:GetLocalPlayerID()
>   local localPlayerConfig = PlayerConfigurations[localPlayerID]
>   local playerModSyncDataStr = localPlayerConfig:GetValue("MLDM_ModSyncData_myModDataId")
>   local localPlayerModSyncData = deserialize(playerModSyncDataStr)
>   return localPlayerModSyncData
> end
> -- 同步指定Mod的_syncData数据到其他玩家
> MLDM:SyncModDataToOtherPlayers('myModDataId') -- 相相当于 MLDM:SyncModDataToOtherPlayers('myModDataId', { _myParam1 = 10, _myParam2 = 'hello' })
> -- 数据同步后
> local localPlayerModSyncData1 = GetlocalPlayerModSyncData() -- { _myParam1 = 10, _myParam2 = 'hello' }
>
>
> -- 假如要更新覆盖同步指定Mod的_syncData.myParam1数据到其他玩家
> MLDM:SyncModDataToOtherPlayers('myModDataId', { myParam1 = 20 })
> -- 数据同步后
> local localPlayerModSyncData2 = GetlocalPlayerModSyncData() -- { _myParam1 = 20, _myParam2 = 'hello' }
>
>
> -- 当然也支持mod开发者同步自定义的同步数据
> MLDM:SyncModDataToOtherPlayers('myModDataId', { CustomizeSynchronizedDataKEY1 = {1,2,3} })
> -- 数据同步后
> local localPlayerModSyncData3 = GetlocalPlayerModSyncData() -- { _myParam1 = 20, _myParam2 = 'hello', CustomizeSynchronizedDataKEY1 = {1,2,3} }
> 
> -- 同步多个Mod的_syncData数据到其他玩家，例如：
> MLDM:SyncModDataToOtherPlayers({ 'Mod1DataId', 'Mod2DataId', 'Mod3DataId' })
> MLDM:SyncModDataToOtherPlayers({ 'Mod1DataId', 'Mod2DataId', 'Mod3DataId' }, { {_mod1key=1},{mod2key=2},{mod3key=3} })
> ```
> </details>

> <details><summary>相关源码</summary>
> 
> ```Lua
> SyncModDataToOtherPlayers = function(self, dataId, modSyncUpdateData:table)
>   local localPlayerID = self:GetLocalPlayerID()
>   if localPlayerID == -1 then return end
>   if type(dataId) == 'table' then -- 此时更新多个mod的同步数据
>     if table.count(dataId) == 0 then return end -- 无实际的更新数据，不进行同步
>     if not modSyncUpdateData then
>     modSyncUpdateData = {}
>     for i, idataId in pairs(dataId) do
>       modSyncUpdateData[i] = self.CurrentlyModDatas[self.ModDataIds[idataId]]._syncData
>     end
>     end
>     dataId = serialize(dataId)
>   elseif type(dataId) == 'string' then -- 此时更新一个mod的同步数据
>     modSyncUpdateData = modSyncUpdateData or self.CurrentlyModDatas[self.ModDataIds[dataId]]._syncData
>   else
>     print("MLDM:SyncModDataToOtherPlayers dataId参数类型错误")
>     return
>   end
>   local PlayerConfig = PlayerConfigurations[localPlayerID]
>   local serializedModSyncUpdateDataStr = serialize(modSyncUpdateData)
>   -- 采用打包方式统一管理，一起同步
>   PlayerConfig:SetValue("MLDM_ModSyncDataUpdateId", dataId)
>   PlayerConfig:SetValue("MLDM_ModSyncDataUpdate", serializedModSyncUpdateDataStr)
>   print('成功同步数据SyncModDataToOtherPlayers')
>   -- 通知其他玩家同步数据
>   Network.BroadcastPlayerInfo(localPlayerID)
>   -- 修复bug
>   -- 同时本地玩家也需要更新同步数据，因为MLDM_ModSyncDataUpdateId，和MLDM_ModSyncDataUpdate是打包的
>   -- 否则modder还需要额外区分玩家是否是本地玩家采用不同的获取snycData的方式
>   -- 因此本地玩家也应当解包MLDM_ModSyncDataUpdateId，和MLDM_ModSyncDataUpdate的数据
>   -- 这种设计也是为了兼顾不同的同步情况，例如一个mod又更改了个别数据，那么只用同步这部分数据，通过MLDM_ModSyncDataUpdate缓存过度，数据传到其他端，其他端根据这个更改
>   self:ReceiveModDataSyncFromOtherPlayers(localPlayerID)
> end,
> ```
> </details>

#### 1.7 GetModObject
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
> - 都是直接在methods表直接定义的子函数，所以不需要隐式的self参数，所以请使用.
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
>       SetParam = function(iKey, value)
>           -- 确保 value1 是一个字符串或数字
>         if type(iKey) ~= "string" and type(iKey) ~= "number" then
>           print("The first argument must be a string or number key.")
>         end
>         -- 禁止使用_localData 和 _syncData 为key,除非是替换整个_localData或_syncData的数据
>         if key == '_localData' or key == '_syncData' or type(value) ~= 'table' then
>           print("Can't set param for _localData or _syncData.", iKey, value)
>           return
>         end
>         self.CurrentlyModDatas[key][iKey] = value
>       end,
>       SetParams = function(t:table)
>         -- 批量设置参数模式：当 value1 是一个表时
>         for k, v in pairs(t) do
>           if k == '_localData' or k == '_syncData' or type(value) ~= 'table' then
>             print("Can't set params for _localData or _syncData.", k, v)
>           else
>             self.CurrentlyModDatas[key][k] = v
>           end
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
>     -- 设置元表, 便于使用, 对象直接可以访问 getParams 和 setParams 方法
>     setmetatable(internalObj, { __index = methods })
>     return internalObj
>   end,
> ```
> </details>

> <details><summary>“ModObject”相关API</summary>
>
> 这里也有 SetIgnoreModVersionUpdates
> 但和上面不同，这里是直接ModData设定该mod数据是否"忽略版本更新"，而上面还需要提供modUUID参数
> 
> 
> ##### **基础API**
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
> ##### **Mod配置数据UI-API**
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
 
> <details><summary>“ModObject”相关API演示</summary>
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
> > <details><summary>AddSubpanelUI不设置的效果</summary>
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


### 2. 跨UIlua脚本API
如果熟悉文明6的lua，我想说到跨UI-lua脚本的交互应该立马想到LuaEvents和ExposedMembes。  
是的我们需要使用他们来完成跨UI-lua脚本的交互，同时还有一个独特的官方API也会被使用，请让我一点一点讲述  
这里我们没有使用ExposedMembes，而是使用了LuaEvents。  
因为ExposedMembes，更适合单机本地游戏，不适用于联机游戏。因为你无法获取联机游戏中对方的ExposedMembers。  

- **注意**: 是在UI环境定义的的LuaEvents，你在Game环境无法使用这里的LuaEvents
- 不知道你是否还记得前面ModDataIds的DataId，很快要用到它

> <details><summary>部分相关源码</summary>
> 
>```Lua
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
> - 两种使用方法，一中是mod开发者主动触发调用的LuaEvents(下面我称为'事件触发')，另一种是框架触发调用的LuaEvents，mod开发者绑定需要被调用的函数(下面我称为'事件监听')
> - LuaEvents无法直接返回数据，所以这里参数是有调用触发放给监听方的，别搞混
> 
> 方法 | 使用方法 | 参数 | 说明
> --- | --- | --- | --- 
> MLDM_ConfigUIStart | 事件监听 | - | 通知其他mod脚本配置UI开始刷新，每次打开配置UI时，都会调用这个事件
> MLDM_ConfigUIComplete | 事件监听 | - | 通知其他mod脚本配置UI刷新完成，每次打开配置UI时，都会根据配置数据刷新UI，在刷新完成后，会调用这个事件
> MLDM_AlmostComplete | 事件监听 | - | 通知其他mod脚本，框架几乎初始化完成，但加载Mod数据的屏幕还未关闭，可以正常获得各自Mod的数据了<br> 只在前端环境有触发
> MLDM_InitComplete | 事件监听 | - | 通知其他mod脚本，框架初始化完成，可以正常获得各自Mod的数据了
> MLDM_Saving | 事件触发 | - | 用于让框架对当前数据进行保存。<br>因为数据的存储是保存在配置存档中的，为了避免频繁的保存，所以每次发生数据修改不会自动立马保存，而需要Mod开发者在更改完成多个数据后，手动调用这个事件来保存数据。<br>以减小保存频率，提高性能。<br>具体注意细节在前面的"**四.框架的使用**"的"3. 关于框架Mod数据保存"有说明。
> MLDM_SetIMVU | 事件触发 | modUUID(string)<br>isIgnore(boolean) | 既SetIgnoreModVersionUpdates的缩写，用于设置是否忽略Mod版本更新<br>具体参考前面关于SetIgnoreModVersionUpdates的其他API说明
> GetModDataByDataId | 事件触发 | DataId(string)<br>Key(string/number/nil) | 向框架发出获取指定Mod数据的请求，有两种模式：<br>1. 同时传入DataId和Key，获取指定DataId对应Mod的Key的数据<br>2. 不传入Key，获取整个Mod数据<br>注意LuaEvents无法直接返回数据，所以需要配合另一个LuaEvents[DataId]来获取数据<br>具体情况见注释【1】
> [DataId]/[DataId_Key] | 事件监听 | Value(any) | 配合上面的GetModDataByDataId来实现跨UIlua脚本的数据传递，获取指定DataId对应Mod的数据，并返回给调用者<br>具体情况见注释【1】
> SetModDataByModId | 事件触发 | modUUID(string)<br>Value1(string/number/table)<br>Value2(any) | 设置指定Mod数据，有两种模式：<br>1. Value1作为key(string/number)，同时传入modUUID、Key和Value2，设置指定Mod的Key的数据为Value2<br>2. Value1类型是表(table)，传入modUUID和Value1，会遍历Value1的key-value对，设置指定Mod的key的数据为value<br>最后完成所有数据更改，如果需要立即保存数据别忘记调用保存的API
> MLDM_MultiplayerRoomHumanPlayerPositionChange | 事件监听 | playerId(number) | 通知其他mod脚本，在游戏前端的多人联机房间中，某个玩家的位置发生变化
> ['MLDM_ModSyncDataSync_' ..dataId] | 事件监听 | playerId(number)<br>ModSyncData(table) | 通知ModDataId对应的mod脚本，在游戏前端的多玩家联机房间中，特定人类玩家ID的该mod的同步数据已完成同步。<br> modder需要在自己的脚本中定义自己的处理同步数据的函数，，并在自己的脚本中绑定监听事件，例如设置该ID对应的数据。
> ['MLDM_ModSyncDataClear_' ..dataId] | 事件监听 | playerId(number) | 通知ModDataId对应的mod脚本，在游戏前端的多玩家联机房间中，特定人类玩家ID的该mod的同步数据已清除。<br> modder需要在自己的脚本中定义自己的处理同步数据的函数，并在自己的脚本中绑定监听事件，例如将该ID对应的数据恢复默认/置空等操作。
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

### 3. 联机数据的获取API
- 已经在前面的'四.框架的使用'的'4. 关于网络多人联机数据同步问题'中有说明，这里不再赘述

---
**最后，由于本框架的设计可能存在不完善之处，我们诚挚欢迎社区成员提供反馈和建议，以共同改进和优化机制。**
