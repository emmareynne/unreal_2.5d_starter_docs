# Core Framework Implementation Overview

This document outlines the implementation plan for the core framework of our 2.5D platformer in Unreal Engine 5.5, following best practices and leveraging the installed plugins.

## What is the Core Framework?

The core framework serves as the foundation for all other game systems. It provides:
- Persistence across level transitions (GameInstance)
- Game rules and mode-specific logic (GameMode)
- Standardized interfaces for common gameplay functions
- Debug tools for development
- Collision setup specific to 2.5D gameplay
- Input system configuration
- Core ability system setup

## Implementation Steps

### 1. Directory Structure Setup

We'll create the following organization for our core framework:

```
Source/Platformer/
  ├─ Core/
  │   ├─ GameInstance/
  │   ├─ GameMode/
  │   ├─ Debug/
  │   └─ Interfaces/
  ├─ Input/
  ├─ Abilities/
  └─ Characters/
```

### 2. Game Instance Implementation

The GameInstance will persist throughout the game's lifecycle and store:
- Player progression data
- Level transitions and loading screens
- Configuration settings
- Save/load functionality
- Persistent audio management

**Technical Approach:**
- Create a C++ GameInstance subclass with exposed Blueprint properties
- Implement static getter functions for easy access from anywhere
- Add data structures for persistent game state

### 3. Game Mode Setup

The GameMode will handle:
- Player spawning and respawn logic
- Level-specific rules and win conditions
- Default pawn and controller class assignment
- Session management for our 2.5D platformer

**Technical Approach:**
- Create a base GameMode with common functionality
- Implement level-specific GameMode variants as needed
- Set up player start positioning for 2.5D levels

### 4. Interface Definitions

We'll create these key interfaces:
- **Damageable**: For anything that can receive damage
- **Interactable**: For collectibles and interactive objects
- **MovementModifier**: For objects that modify character movement

**Technical Approach:**
- C++ interfaces with BlueprintNativeEvent functions
- Clear documentation on expected behavior
- Helper functions for common operations

### 5. Debug Subsystem

Create a robust debugging system that provides:
- Visual debugging toggles (hitboxes, paths, triggers)
- Console commands for testing
- Performance monitoring tools
- Cheats for development testing

**Technical Approach:**
- Implement as a WorldSubsystem for easy access
- Add console variables (CVars) for toggling debugging features
- Integrate with Enhanced Input for debug shortcuts

### 6. Custom Collision Setup

Configure collision specifically for 2.5D gameplay:
- Custom channels for player, enemies, projectiles, etc.
- Optimization for 2.5D collision checks
- Preset profiles for common object types

**Technical Approach:**
- Define collision channels in C++
- Create preset object types with appropriate responses
- Document collision matrix for reference

### 7. Gameplay Ability System Integration

Leverage GAS for:
- Character abilities (jump, dash, attack)
- Status effects
- Attribute management (health, energy)
- Predictable networking for multiplayer potential

**Technical Approach:**
- Create base ability sets and tags
- Implement attribute set for core character stats
- Set up gameplay effects for status changes
- Create ability tasks specific to 2.5D movement

### 8. Enhanced Input Configuration

Configure the Enhanced Input system for:
- Character movement in 2.5D space
- Context-sensitive actions
- Support for multiple input methods (keyboard, gamepad)
- UI navigation integration with CommonUI

**Technical Approach:**
- Create Input Action assets for all core gameplay actions
- Define Input Mapping Contexts for different game states
- Implement input modifiers for 2.5D constraints

## Technical Considerations

### Performance Optimization
- Minimize code in Tick functions
- Use spatially optimized queries for 2.5D space
- Configure appropriate culling distances for 2.5D perspective

### Cross-Platform Compatibility
- Abstract input handling for different devices
- Consider mobile performance requirements
- Design UI for various screen ratios

### Blueprint vs C++ Division
- Core systems and interfaces in C++
- Specific implementations and behaviors in Blueprint
- Follow the principle: "C++ defines what's possible, Blueprint defines what happens"

## Dependencies and Requirements

- Enhanced Input Plugin
- Gameplay Ability System Plugin
- CommonUI Plugin
- Basic understanding of Unreal C++ programming

## Next Steps After Implementation

After completing the core framework:
1. Implement the character movement system
2. Test core functionality with automated tests
3. Create a simple test level to validate the framework
4. Begin implementing game-specific systems

## Success Criteria

The core framework implementation is successful when:
- All components compile without errors
- GameInstance persists data across level transitions
- GameMode spawns characters in appropriate 2.5D constraints
- Debug tools allow visualization of key game elements
- Input system correctly processes 2.5D movement
- Ability system correctly handles basic character actions
- Collision system properly detects interactions in 2.5D space

## Development Timeline

Estimated implementation time: 5-10 hours depending on experience level.

Phase breakdown:
1. Basic class creation and directory structure (1-2 hours)
2. Game Instance and Game Mode implementation (1-2 hours)
3. Interface development (1 hour)
4. Debugging system (1 hour)
5. Collision setup (30 minutes)
6. GAS configuration (1-2 hours)
7. Input system setup (1 hour)
8. Testing and refinement (1 hour)
