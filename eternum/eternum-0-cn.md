# 0x0 Eternum合约解析

## Eternum 介绍
[Eternum](https://eternum-docs.realms.world/overview/introduction) 是一个全链游戏，游戏的运行逻辑完全部署并运行在链上且完全开源。它是以资源为核心的战略游戏。玩家可以在无限延伸的地图上探索、开采资源、建立商贸关系，也可以组建军队争夺领土。游戏还支持组建部落联盟，玩家间可以互相合作，也可以敌对战争，最后通过贡献超级建筑累计得分来赢得最终的游戏。

与之前的伪链游不同，Eternum 真正实现了游戏逻辑的链上运行。传统的链游通常只是将 NFT 作为进入游戏的凭证，而游戏的实际逻辑仍然在链下运行，与传统游戏无异。而在 Eternum 中，所有的游戏逻辑都在链上执行，确保了游戏的透明性和不可篡改性。

## Dojo 开发框架
Eternum 使用 [Dojo](https://book.dojoengine.org/what-is-dojo) 框架进行开发，主要采用 ECS（实体-组件-系统）架构模式。这是一种游戏开发常用的架构模式，主要将游戏分为实体（Entity）、组件（Component）和系统（System）。实体是游戏中的基本单位，通过 ID 来标识，它不包含数据或行为，作为组件的容器存在。组件通常是一组数据结构，是实体的属性集合，可以被自由地添加到实体上。系统则处理游戏的逻辑，更新实体中这些组件的变化和状态。

这种设计有很多好处：模块化高，组件可以自由组合，系统相互独立，扩展新功能很方便。其次，数据和行为分离，实体自由添加修改组件，更容易修改游戏的逻辑，因为都在系统中统一处理，维护起来更方便。

## ECS 设计示例
以下是一个 ECS 系统的设计示例，注意这不是 Eternum 的代码，仅作为分析使用。

假设目前需求是有两类建筑，一个是资源建筑一个是防御建筑，

```rust
// ❌ 不好的设计：所有属性都在一个结构体中
#[derive(PartialEq, Drop, Copy, Serde)]
#[dojo::model]
struct Building {
    id: ID,
    // 位置属性
    x: u32,
    y: u32,
    // 生产属性
    resource_type: u8,
    production_rate: u128,
    is_paused: bool,
    // 战斗属性
    defense: u32,
    health: u32,
    // 所有权属性
    owner: ContractAddress,


}

// ✅ 好的设计：实体只是一个ID标识
#[derive(PartialEq, Drop, Copy, Serde)]
struct Building {
    entity_id: ID,
    category: BuildingCategory
}

// 位置组件
#[dojo::model]
struct Position {
    #[key]
    entity_id: ID,
    x: u32,
    y: u32
}

// 生产组件
#[dojo::model]
struct Production {
    #[key]
    entity_id: ID,
    resource_type: u8,
    production_rate: u128,
    is_paused: bool
}

// 战斗组件
#[dojo::model]
struct Combat {
    #[key]
    entity_id: ID,
    defense: u32,
    health: u32
}

// 所有权组件
#[dojo::model]
struct Owner {
    #[key]
    entity_id: ID,
    address: ContractAddress
}
```

```rust
// 创建一个普通资源建筑
fn create_resource_building(ref world: WorldStorage) {
    // 1. 创建基础实体
    let mut building = Building {
        entity_id: world.dispatcher.uuid(),
        category: BuildingCategory::Resource
    };
    
    // 2. 添加不同的组件组合
    let mut position = Position { entity_id: building.entity_id, x: 10, y: 20 };
    let mut production = Production { 
        entity_id: building.entity_id,
        resource_type: ResourceTypes::WOOD,
        production_rate: 100
    };
    let mut owner = Owner {
        entity_id: building.entity_id,
        address: get_caller_address()
    };
    
    // 3. 保存组件
    world.write_model(@building);
    world.write_model(@position);
    world.write_model(@production);
    world.write_model(@owner);
}

// 创建一个防御建筑
fn create_defense_building(ref world: WorldStorage) {
    // 1. 创建基础实体
    let mut building = Building {
        entity_id: world.dispatcher.uuid(),
        category: BuildingCategory::Defense
    };
    
    // 2. 添加不同的组件组合
    let mut position = Position { entity_id: building.entity_id, x: 15, y: 25 };
    let mut combat = Combat {
        entity_id: building.entity_id,
        defense: 1000,
        health: 5000
    };
    let mut owner = Owner {
        entity_id: building.entity_id,
        address: get_caller_address()
    };
    
    // 3. 保存组件
    world.write_model(@building);
    world.write_model(@position);
    world.write_model(@combat);
    world.write_model(@owner);
}
```

通过以上对比可以看出前者设计的问题：
1. 不灵活
    - 每个建筑都必须包含所有属性
    - 即使建筑不需要战斗功能也必须有战斗属性
2. 难以维护
    - 结构体过于庞大
    - 添加新功能需要修改整个结构体，代码复用困难

假如后续需要功能扩展，可以让建筑物进行升级。那么只需添加新组件，系统处理新组件即可。
```rust
// 轻松添加新组件
#[dojo::model]
struct Upgrade {
    #[key]
    entity_id: ID,
    level: u8,
    upgrade_cost: u128
}

// 系统只需要关心相关组件
fn upgrade_building(ref world: WorldStorage, building_id: ID) {
    let mut upgrade = world.read_model(building_id);
    let mut resource = world.read_model(building_id);
    // 只处理需要的组件
}

```

同时组件可以被不同实体复用。
```rust
struct Position {
    #[key]
    entity_id: ID,
    x: u32,
    y: u32
}

// 建筑可以用
let building_pos = Position { entity_id: building.id, x: 0, y: 0 };
// 军队可以用
let army_pos = Position { entity_id: army.id, x: 10, y: 10 };

```

### Eternum合约架构
在`lib.cairo`中，models 中定义实体和组件，systems 中是系统，游戏的外部调用逻辑都是调用这里的函数。alias 定义了全局的实体 ID 的数据类型是 u32，constants 定义全局不变的变量，utils 里是一些辅助功能的方法，比如处理数学运算、随机数等。

 ```rust
pub mod alias;
pub mod constants;
pub mod models;
pub mod systems;
pub mod utils;
 ```

接下来的文章会从以下几个游戏核心角度做解析如何搭建一个像 Eternum 这样的全链游戏。

- realm的创建
- 建筑物
- 军队
- 地图
- 资源转移
- 银行
- 超级建筑与部落