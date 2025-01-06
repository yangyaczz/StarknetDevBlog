# Eternum从入门到放弃

## 0x00 Eternum 介绍
[Eternum](https://eternum-docs.realms.world/overview/introduction) 是一个全链游戏，游戏的运行逻辑完全部署并运行在链上且完全开源。它是以资源为核心的战略游戏。玩家可以在无限延伸的地图上探索、开采资源、建立商贸关系，也可以组建军队争夺领土。游戏还支持组建部落联盟，玩家间可以互相合作，也可以敌对战争，最后通过贡献超级建筑累计得分来赢得最终的游戏。

与之前的伪链游不同，Eternum 真正实现了游戏逻辑的链上运行。传统的链游通常只是将 NFT 作为进入游戏的凭证，而游戏的实际逻辑仍然在链下运行，与传统游戏无异。而在 Eternum 中，所有的游戏逻辑都在链上执行，确保了游戏的透明性和不可篡改性。

## 0x01 Dojo 开发框架
Eternum 使用 [Dojo](https://book.dojoengine.org/what-is-dojo) 框架进行开发，主要采用 ECS（实体-组件-系统）架构模式。这是一种游戏开发常用的架构模式，主要将游戏分为实体（Entity）、组件（Component）和系统（System）。实体是游戏中的基本单位，通过 ID 来标识，它不包含数据或行为，作为组件的容器存在。组件通常是一组数据结构，是实体的属性集合，可以被自由地添加到实体上。系统则处理游戏的逻辑，更新实体中这些组件的变化和状态。

这种设计有很多好处：模块化高，组件可以自由组合，系统相互独立，扩展新功能很方便。其次，数据和行为分离，实体自由添加修改组件，更容易修改游戏的逻辑，因为都在系统中统一处理，维护起来更方便。

## 0x02 ECS 设计示例
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

## 0x03 Eternum合约架构
在`lib.cairo`中，models 中定义实体和组件，systems 中是系统，游戏的外部调用逻辑都是调用这里的函数。alias 定义了全局的实体 ID 的数据类型是 u32，constants 定义全局不变的变量，utils 里是一些辅助功能的方法，比如处理数学运算、随机数等。

 ```rust
pub mod alias;
pub mod constants;
pub mod models;
pub mod systems;
pub mod utils;
 ```

## 0x04 一些核心System
接下来的文章会从以下几个游戏核心系统做解析 Eternum 的运行逻辑。

<div style="font-size: 20px">

- [realm](#realm)
- [building](#building) 
- [combat](#combat)
- [map](#map)
- [resource](#resource)
- [bank](#bank)
- [hyperstructure & guild](#hyperstructure--guild)

</div>

### realm

在这个system中的逻辑主要3个, `create`, `upgrade_level`, `quest_claim`。

`create` 是进入游戏的第一步，用户需要持有赛季通行证的nft，靠它来激活并进入游戏。这里会创建实体并添加一系列的组件，还会涉及到和其他系统合约的交互。同时在realm的组件内还有nft数据的编码解码。

`upgrade_level`是将领地升级。

`quest_claim`是新用户第一次玩在前端做完新手任务后领的奖励。（直接调用合约领取也可以）

[system code link](https://github.com/BibliothecaDAO/eternum/blob/next/contracts/src/systems/realm/contracts.cairo)


```rust

    #[abi(embed_v0)]
    impl RealmSystemsImpl of super::IRealmSystems<ContractState> {
        // 创建一个新的领地
        // owner: 领地的所有者
        // realm_id: 领地的id, (领地的id也是赛季通行证的id)
        // frontend: 接收nft中物资的地址
        fn create(ref self: ContractState, owner: ContractAddress, realm_id: ID, frontend: ContractAddress) -> ID {
            // 确保赛季仍在进行，此处调用读取season的组件的内部函数进行检查
            let mut world: WorldStorage = self.world(DEFAULT_NS());
            SeasonImpl::assert_has_started(world);
            SeasonImpl::assert_season_is_not_over(world);

            // 读取赛季通行证nft的地址，并将它从所有者中进行转移核销
            let season: SeasonAddressesConfig = world.read_model(WORLD_CONFIG_ID);
            InternalRealmLogicImpl::collect_season_pass(season.season_pass_address, realm_id);

            // 从赛季通行证中获取领地元数据
            let (realm_name, regions, cities, harbors, rivers, wonder, order, resources) =
                InternalRealmLogicImpl::retrieve_metadata_from_season_pass(
                season.season_pass_address, realm_id
            );

            // 获取最新的坐标，然后创建领地
            let mut coord: Coord = InternalRealmLogicImpl::get_new_location(ref world);
            let (entity_id, realm_produced_resources_packed) = InternalRealmLogicImpl::create_realm(
                ref world, owner, realm_id, resources, order, 0, wonder, coord
            );

            // 从赛季通行证中收集领主并桥接到领地
            let lords_amount_attached: u256 = InternalRealmLogicImpl::collect_lords_from_season_pass(
                season.season_pass_address, realm_id
            );

            // 如果该领地有lords，则桥接到领地
            if lords_amount_attached.is_non_zero() {
                InternalRealmLogicImpl::bridge_lords_into_realm(
                    ref world, season.lords_address, entity_id, lords_amount_attached, frontend
                );
            }

            // 发出领地设置事件
            let address_name: AddressName = world.read_model(owner);
            world
                .emit_event(
                    @SettleRealmData {
                        id: world.dispatcher.uuid(),
                        event_id: EventType::SettleRealm,
                        entity_id,
                        owner_address: owner,
                        owner_name: address_name.name,
                        realm_name: realm_name,
                        produced_resources: realm_produced_resources_packed,
                        cities,
                        harbors,
                        rivers,
                        regions,
                        wonder,
                        order,
                        x: coord.x,
                        y: coord.y,
                        timestamp: starknet::get_block_timestamp(),
                    }
                );

            entity_id.into()
        }


        fn upgrade_level(ref self: ContractState, realm_id: ID) {
            // 确保调用者拥有该领地
            let mut world: WorldStorage = self.world(DEFAULT_NS());
            let entity_owner: EntityOwner = world.read_model(realm_id);
            entity_owner.assert_caller_owner(world);

            // 确保该实体是领地
            let mut realm: Realm = world.read_model(realm_id);
            realm.assert_is_set();

            // 确保领地不是已经达到最大等级
            let max_level = realm.max_level(world);
            assert(realm.level < max_level, 'realm is already at max level');

            // 支付升级到下一级的费用
            let next_level = realm.level + 1;
            let realm_level_config: RealmLevelConfig = world.read_model(next_level);
            let required_resources_id = realm_level_config.required_resources_id;
            let required_resource_count = realm_level_config.required_resource_count;
            let mut index = 0;
            loop {
                if index == required_resource_count {
                    break;
                }

                // 获取所需资源
                let mut required_resource: DetachedResource = world.read_model((required_resources_id, index));

                // 从领地中销毁所需资源
                let mut realm_resource = ResourceImpl::get(ref world, (realm_id, required_resource.resource_type));
                realm_resource.burn(required_resource.resource_amount);
                realm_resource.save(ref world);
                index += 1;
            };

            // 设置新等级
            realm.level = next_level;
            world.write_model(@realm);

            // [成就系统] 升级到最大等级
            if realm.level == max_level {
                let player_id: felt252 = starknet::get_caller_address().into();
                let task_id: felt252 = Task::Maximalist.identifier();
                let store = StoreTrait::new(world);
                store.progress(player_id, task_id, count: 1, time: starknet::get_block_timestamp(),);
            }
        }

        fn quest_claim(ref self: ContractState, quest_id: ID, entity_id: ID) {
            // 确保赛季仍在进行
            let mut world: WorldStorage = self.world(DEFAULT_NS());
            SeasonImpl::assert_season_is_not_over(world);

            // 确保实体是领地
            let realm: Realm = world.read_model(entity_id);
            realm.assert_is_set();

            // 确保任务未完成
            let mut quest: Quest = world.read_model((entity_id, quest_id));
            assert(!quest.completed, 'quest already completed');

            // 确保调用者是领地的最开始初始化的人（如果某个领地没完成任务，然后被其他人占领了，占领者不是settler，不能继续领取剩下的新手任务奖励了）
            assert(realm.settler_address == starknet::get_caller_address(), 'Caller not settler');

            // 确保任务有奖励
            let quest_config: QuestConfig = world.read_model(WORLD_CONFIG_ID);
            let quest_reward_config: QuestRewardConfig = world.read_model(quest_id);
            assert(quest_reward_config.detached_resource_count > 0, 'quest has no rewards');

            let mut index = 0;
            loop {
                if index == quest_reward_config.detached_resource_count {
                    break;
                }

                // 获取奖励资源
                let mut detached_resource: DetachedResource = world
                    .read_model((quest_reward_config.detached_resource_id, index));
                let reward_resource_type = detached_resource.resource_type;
                let mut reward_resource_amount = detached_resource.resource_amount;

                let mut quest_bonus: QuestBonus = world.read_model((entity_id, reward_resource_type));

                // 根据任务生产乘数缩放奖励资源数量
                // 如果奖励资源用于生产领地中的另一种资源，则只有在任务奖励未被领取且奖励资源不是食物时才会缩放。
                if !quest_bonus.claimed && !ResourceFoodImpl::is_food(reward_resource_type) {
                    let reward_resource_production_config: ProductionConfig = world.read_model(reward_resource_type);
                    let mut jndex = 0;
                    loop {
                        if jndex == reward_resource_production_config.output_count {
                            break;
                        }

                        let output_resource_type: ProductionOutput = world.read_model((reward_resource_type, jndex));
                        if realm.produces_resource(output_resource_type.output_resource_type) {
                            // scale reward resource amount by quest production multiplier
                            reward_resource_amount *= quest_config.production_material_multiplier.into();
                            // set quest bonus as claimed
                            quest_bonus.claimed = true;
                            world.write_model(@quest_bonus);

                            break;
                        }

                        jndex += 1;
                    }
                }

                if realm.has_wonder {
                    reward_resource_amount *= WONDER_QUEST_REWARD_BOOST.into();
                }

                let mut realm_resource = ResourceImpl::get(ref world, (entity_id.into(), reward_resource_type));
                realm_resource.add(reward_resource_amount);
                realm_resource.save(ref world);

                index += 1;
            };

            quest.completed = true;
            world.write_model(@quest);

            // 完成所有任务 成就
            let next_quest_id: ID = (quest_id + 1).into();
            let next_quest_reward_config: QuestRewardConfig = world.read_model(next_quest_id);
            // 如果下一个任务没有奖励，则玩家已经完成了所有任务
            if (next_quest_reward_config.detached_resource_count == 0) {
                let player_id: felt252 = starknet::get_caller_address().into();
                let task_id: felt252 = Task::Squire.identifier();
                let store = StoreTrait::new(world);
                store.progress(player_id, task_id, count: 1, time: starknet::get_block_timestamp(),);
            };
        }
    }

    // 内部函数部分
    #[generate_trait]
    impl InternalRealmLogicImpl of InternalRealmLogicTrait {
        // 创建领地
        fn create_realm(
            ref world: WorldStorage,
            owner: ContractAddress,
            realm_id: ID,
            resources: Array<u8>,
            order: u8,
            level: u8,
            wonder: u8,
            coord: Coord
        ) -> (ID, u128) {

            // 获取领地是否拥有奇迹
            let has_wonder = RealmReferenceImpl::wonder_mapping(wonder.into()) != "None";
            // 打包领地生产的资源类型
            let realm_produced_resources_packed = RealmResourcesImpl::pack_resource_types(resources.span());
            // 创建领地实体ID
            let entity_id = world.dispatcher.uuid();
            // 获取当前时间戳
            let now = starknet::get_block_timestamp();
            // 设置领地所有者  （实体的拥有者是一个地址）
            world.write_model(@Owner { entity_id: entity_id.into(), address: owner });
            // 设置领地所有者 （实体的拥有者是另一个实体）
            world.write_model(@EntityOwner { entity_id: entity_id.into(), entity_owner_id: entity_id.into() });
            // 设置地图中这块地的类型
            world
                .write_model(
                    @Structure { entity_id: entity_id.into(), category: StructureCategory::Realm, created_at: now, }
                );
            // 设置地图中这块地有超构数量
            world.write_model(@StructureCount { coord, count: 1 });
            // 设置领地容量类别
            world
                .write_model(
                    @CapacityCategory { entity_id: entity_id.into(), category: CapacityConfigCategory::Structure }
                );
            // 设置领地
            world
                .write_model(
                    @Realm {
                        entity_id: entity_id.into(),
                        realm_id,
                        produced_resources: realm_produced_resources_packed,
                        order,
                        level,
                        has_wonder,
                        settler_address: owner,
                    }
                );
            
            // 设置领地坐标
            world.write_model(@Position { entity_id: entity_id.into(), x: coord.x, y: coord.y, });

            // 探索领地所在的区域
            let mut tile: Tile = world.read_model((coord.x, coord.y));
            if tile.explored_at.is_zero() {
                // 探索领地所在的区域
                InternalMapSystemsImpl::explore(ref world, entity_id.into(), coord, array![(1, 0)].span());
            }

            (entity_id, realm_produced_resources_packed)
        }

        // 收集赛季通行证
        fn collect_season_pass(season_pass_address: ContractAddress, realm_id: ID) {
            let caller = starknet::get_caller_address();
            let this = starknet::get_contract_address();
            let season_pass = ISeasonPassDispatcher { contract_address: season_pass_address };

            // 将赛季通行证从调用者转移到当前合约
            season_pass.transfer_from(caller, this, realm_id.into());
        }

        fn collect_lords_from_season_pass(season_pass_address: ContractAddress, realm_id: ID) -> u256 {
            // 实例化赛季通行证
            let season_pass = ISeasonPassDispatcher { contract_address: season_pass_address };
            // 获取领主数量
            let token_lords_balance: u256 = season_pass.lords_balance(realm_id.into());
            // 将lords转出来
            season_pass.detach_lords(realm_id.into(), token_lords_balance);
            // 确保lords数量为0（被转移出来了）
            assert!(season_pass.lords_balance(realm_id.into()).is_zero(), "lords amount attached to realm should be 0");

            token_lords_balance
        }


        fn bridge_lords_into_realm(
            ref world: WorldStorage,
            lords_address: ContractAddress,
            realm_entity_id: ID,
            amount: u256,
            frontend: ContractAddress
        ) {
            // 获取桥接系统地址
            let (bridge_systems_address, _namespace_hash) =
                match world.dispatcher.resource(selector_from_tag!("s0_eternum-resource_bridge_systems")) {
                dojo::world::Resource::Contract((
                    contract_address, namespace_hash
                )) => (contract_address, namespace_hash),
                _ => (Zeroable::zero(), Zeroable::zero())
            };

            // 批准桥接系统花费lords
            IERC20Dispatcher { contract_address: lords_address }.approve(bridge_systems_address, amount);

            // 桥接lords
            IResourceBridgeSystemsDispatcher { contract_address: bridge_systems_address }
                .deposit_initial(lords_address, realm_entity_id, amount, frontend);
        }


        // 从赛季通行证中获取领地元数据
        fn retrieve_metadata_from_season_pass(
            season_pass_address: ContractAddress, realm_id: ID
        ) -> (felt252, u8, u8, u8, u8, u8, u8, Array<u8>) {
            // 实例化赛季通行证
            let season_pass = ISeasonPassDispatcher { contract_address: season_pass_address };
            // 读取赛季通行证编码元数据
            let (name_and_attrs, _urla, _urlb) = season_pass.get_encoded_metadata(realm_id.try_into().unwrap());
            // 用组件中的逻辑解码领地元数据
            RealmNameAndAttrsDecodingImpl::decode(name_and_attrs)
        }

        // 获取新的领地坐标
        fn get_new_location(ref world: WorldStorage) -> Coord {
            // 确保坐标未被其他结构占用
            let mut found_coords = false;
            let mut coord: Coord = Coord { x: 0, y: 0 };
            // 获取领地配置
            let mut settlement_config: SettlementConfig = world.read_model(WORLD_CONFIG_ID);
            while (!found_coords) {
                // 获取下一个领地坐标
                coord = settlement_config.get_next_settlement_coord();
                // 全局变量的组件计数
                let mut structure_count: StructureCount = world.read_model(coord);
                if structure_count.is_none() {
                    found_coords = true;
                    structure_count.count = 1;
                    world.write_model(@structure_count);
                }

                world.write_model(@settlement_config);
            };

            return coord;
        }
    }
}

```

models中的realm组件， 主要定义数据结构，以及会内部调用处理的一些方法， 主要涉及映射相关。 省略了它如何处理资源编码解码映射部分。

[model code link](https://github.com/BibliothecaDAO/eternum/blob/next/contracts/src/models/realm.cairo)

```rust
#[derive(IntrospectPacked, Copy, Drop, Serde)]
#[dojo::model]
pub struct Realm {
    #[key]
    entity_id: ID, // 实体ID
    realm_id: ID, // 领地ID
    produced_resources: u128, // 领地生产的资源类型
    order: u8, // 队形的id
    level: u8, // 领地等级
    has_wonder: bool, // 领地是否拥有奇迹
    settler_address: ContractAddress, // 领取quest的地址
}


#[generate_trait]
impl RealmImpl of RealmTrait {
    // 获取领地最大等级
    fn max_level(self: Realm, world: WorldStorage) -> u8 {
        let realm_max_level_config: RealmMaxLevelConfig = world.read_model(WORLD_CONFIG_ID);
        realm_max_level_config.max_level
    }

    // 确保领地实体ID已经被初始化
    fn assert_is_set(self: Realm) {
        assert(self.realm_id != 0, 'Entity is not a realm');
    }
}
```

同时还涉及和其他系统合约的互相调用，比如会调用资源系统合约，合约和合约之间的调用多要注意权限验证。

[system code link](https://github.com/BibliothecaDAO/eternum/blob/next/contracts/src/systems/resources/contracts/resource_bridge_systems.cairo)
```rust

    #[abi(embed_v0)]
    impl ResourceBridgeImpl of super::IResourceBridgeSystems<ContractState> {
        fn deposit_initial(
            ref self: ContractState,
            token: ContractAddress,
            recipient_realm_id: ID,
            amount: u256,
            client_fee_recipient: ContractAddress
        ) {
            let mut world: WorldStorage = self.world(DEFAULT_NS());
            SeasonBridgeConfigImpl::assert_bridge_is_open(world);

            // 确保调用者是领地系统 （权限检查，系统合约和系统合约之间的调用需要验证caller地址）
            let caller = get_caller_address();
            let (realm_systems_address, _namespace_hash) =
                match world.dispatcher.resource(selector_from_tag!("s0_eternum-realm_systems")) {
                dojo::world::Resource::Contract((
                    contract_address, namespace_hash
                )) => (contract_address, namespace_hash),
                _ => (Zeroable::zero(), Zeroable::zero())
            };
            assert!(caller == realm_systems_address, "only realm systems can call this system");

            // 确保领地存在，且是领地
            let recipient_structure: Structure = world.read_model(recipient_realm_id);
            recipient_structure.assert_is_structure();
            assert!(recipient_structure.category == StructureCategory::Realm, "recipient structure is not a realm");

            // 确保桥接系统未暂停
            InternalBridgeImpl::assert_deposit_not_paused(world);

            // 确保token在桥接白名单中
            let resource_bridge_token_whitelist: ResourceBridgeWhitelistConfig = world.read_model(token);
            InternalBridgeImpl::assert_resource_whitelisted(world, resource_bridge_token_whitelist);

            let this = get_contract_address();
            assert!(
                ERC20ABIDispatcher { contract_address: token }.transfer_from(realm_systems_address, this, amount),
                "Bridge: transfer failed"
            );

            // 发送非银行费用
            let non_bank_fees = InternalBridgeImpl::send_non_bank_fees(
                ref world, token, client_fee_recipient, amount, TxType::Deposit
            );

            let token_amount_less_non_bank_fees = amount - non_bank_fees;
            let resource_amount_less_non_bank_fees = InternalBridgeImpl::token_amount_to_resource_amount(
                token, token_amount_less_non_bank_fees
            );

            // 将资源转移到领地
            let resource = array![(resource_bridge_token_whitelist.resource_type, resource_amount_less_non_bank_fees)]
                .span();
            InternalResourceSystemsImpl::transfer(ref world, 0, recipient_realm_id, resource, 0, false, false);
        }
    }
```

### building

### combat

### map 

### resource

### bank

### hyperstructure & guild