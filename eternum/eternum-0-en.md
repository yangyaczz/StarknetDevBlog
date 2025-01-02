# 0x0 Eternum Contract Analysis

## Introduction to Eternum
[Eternum](https://eternum-docs.realms.world/overview/introduction) is a fully on-chain game where the entire game logic is deployed and executed on the blockchain, and it is completely open-source. It is a strategy game centered around resources. Players can explore an infinitely extending map, mine resources, establish trade relations, and form armies to conquer territories. The game also supports forming tribal alliances, allowing players to cooperate or engage in warfare. Ultimately, players contribute to superstructures to accumulate points and win the game.

Unlike previous pseudo blockchain games, Eternum truly implements on-chain game logic. Traditional blockchain games typically use NFTs as entry tokens, while the actual game logic runs off-chain, similar to traditional games. In Eternum, all game logic is executed on-chain, ensuring transparency and immutability.

## Dojo Development Framework
Eternum is developed using the [Dojo](https://book.dojoengine.org/what-is-dojo) framework, primarily adopting the ECS (Entity-Component-System) architecture pattern. This is a common architecture pattern in game development, dividing the game into entities, components, and systems. Entities are the basic units in the game, identified by IDs, and serve as containers for components without containing data or behavior themselves. Components are typically data structures representing the attributes of entities and can be freely added to entities. Systems handle the game logic, updating the changes and states of these components within entities.

This design offers several advantages: high modularity, allowing components to be freely combined, and systems to be independent, making it easy to extend new features. Additionally, the separation of data and behavior allows for easier modification of game logic, as everything is handled uniformly within systems, making maintenance more convenient.

## ECS Design Example
Below is an example of an ECS system design. Note that this is not the code for Eternum, but is used for analysis purposes.

Suppose the current requirement is to have two types of buildings: a resource building and a defense building.

```rust
// ❌ Poor Design: All attributes are in one struct
#[derive(PartialEq, Drop, Copy, Serde)]
#[dojo::model]
struct Building {
    id: ID,
    // Position attributes
    x: u32,
    y: u32,
    // Production attributes
    resource_type: u8,
    production_rate: u128,
    is_paused: bool,
    // Combat attributes
    defense: u32,
    health: u32,
    // Ownership attributes
    owner: ContractAddress,
}

// ✅ Good Design: Entity is just an ID identifier
#[derive(PartialEq, Drop, Copy, Serde)]
struct Building {
    entity_id: ID,
    category: BuildingCategory
}

// Position component
#[dojo::model]
struct Position {
    #[key]
    entity_id: ID,
    x: u32,
    y: u32
}

// Production component
#[dojo::model]
struct Production {
    #[key]
    entity_id: ID,
    resource_type: u8,
    production_rate: u128,
    is_paused: bool
}

// Combat component
#[dojo::model]
struct Combat {
    #[key]
    entity_id: ID,
    defense: u32,
    health: u32
}

// Ownership component
#[dojo::model]
struct Owner {
    #[key]
    entity_id: ID,
    address: ContractAddress
}
```

```rust
// Create a standard resource building
fn create_resource_building(ref world: WorldStorage) {
    // 1. Create the base entity
    let mut building = Building {
        entity_id: world.dispatcher.uuid(),
        category: BuildingCategory::Resource
    };
    
    // 2. Add different component combinations
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
    
    // 3. Save components
    world.write_model(@building);
    world.write_model(@position);
    world.write_model(@production);
    world.write_model(@owner);
}

// Create a defense building
fn create_defense_building(ref world: WorldStorage) {
    // 1. Create the base entity
    let mut building = Building {
        entity_id: world.dispatcher.uuid(),
        category: BuildingCategory::Defense
    };
    
    // 2. Add different component combinations
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
    
    // 3. Save components
    world.write_model(@building);
    world.write_model(@position);
    world.write_model(@combat);
    world.write_model(@owner);
}
```

The issues with the former design are evident:
1. **Inflexibility**
   - Every building must contain all attributes.
   - Even if a building does not need combat functionality, it must have combat attributes.
2. **Difficult to Maintain**
   - The struct is too large.
   - Adding new features requires modifying the entire struct, making code reuse difficult.

If future functionality expansion is needed, such as allowing buildings to upgrade, new components can be added, and the system can handle the new components.

```rust
// Easily add new components
#[dojo::model]
struct Upgrade {
    #[key]
    entity_id: ID,
    level: u8,
    upgrade_cost: u128
}

// The system only needs to handle relevant components
fn upgrade_building(ref world: WorldStorage, building_id: ID) {
    let mut upgrade = world.read_model(building_id);
    let mut resource = world.read_model(building_id);
    // Only process the necessary components
}
```

Components can also be reused by different entities.

```rust
struct Position {
    #[key]
    entity_id: ID,
    x: u32,
    y: u32
}

// Can be used by buildings
let building_pos = Position { entity_id: building.id, x: 0, y: 0 };
// Can be used by armies
let army_pos = Position { entity_id: army.id, x: 10, y: 10 };
```

### Eternum Contract Architecture
In `lib.cairo`, models define entities and components, systems are the systems, and the game's external call logic is all handled by calling functions here. `alias` defines the global entity ID data type as `u32`, `constants` define global immutable variables, and `utils` contains auxiliary methods, such as handling mathematical operations and random numbers.

```rust
pub mod alias;
pub mod constants;
pub mod models;
pub mod systems;
pub mod utils;
```

The following articles will analyze how to build a fully on-chain game like Eternum from several core game perspectives:

- Realm creation
- Buildings
- Armies
- Map
- Resource transfer
- Bank
- Superstructures and tribes
