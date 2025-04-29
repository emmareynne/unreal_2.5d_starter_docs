# Streamlined Debug System for 2.5D Platformer

This guide provides a lightweight, efficient approach to implementing a debug system for your 2.5D Platformer project in Unreal Engine 5.5 using Epic's official ImGui plugin.

## Overview

For a 2.5D platformer, we need a debug system that is:
- Lightweight with minimal performance impact
- Easy to access during gameplay
- Capable of visualizing hitboxes, movement paths, and other gameplay elements
- Configurable without requiring code recompilation

This implementation is split into multiple parts for better organization:
- [Main Setup & Component Implementation](./4_debug_system_main.md)
- [Gameplay Debug Helpers](./4_debug_system_gameplay.md)
- [Visual Debugging Tools](./4_debug_system_visual.md)

## Why ImGui?

ImGui (Immediate Mode GUI) is lightweight, bloat-free graphical user interface library that provides:
- Minimal overhead (significantly faster than Slate or UMG)
- Simple implementation (less code to maintain)
- Real-time editing of variables without recompiling
- Collapsible sections and intuitive organization
- Modern and widely used across game engines (valuable knowledge transfer)

## Prerequisites

- Unreal Engine 5.5 installed
- Platformer project created with the recommended plugins
- [GameInstance](./1_game_instance_implementation.md) implemented
- [GameMode](./2_game_mode_implementation.md) implemented
- [Interface Definitions](./3_interface_definitions.md) implemented

## Getting Started

Follow the guides in order:

1. Start with [Main Setup & Component Implementation](./4_debug_system_main.md) to set up the ImGui plugin and core debug component
2. Then implement [Gameplay Debug Helpers](./4_debug_system_gameplay.md) for gameplay testing features
3. Finally, add [Visual Debugging Tools](./4_debug_system_visual.md) for visualization features

## Next Steps

After completing your debug system implementation, you can move on to:

1. Implementing [Custom Collision Channels](./5_custom_collision_setup.md) for your 2.5D platformer
2. Integrating with the [Gameplay Ability System](./6_gameplay_ability_integration.md)
3. Configuring [Enhanced Input](./7_enhanced_input_configuration.md) for your platformer controls

## Resources

- [UnrealImGui GitHub Repository](https://github.com/segross/UnrealImGui)
- [ImGui Documentation](https://github.com/ocornut/imgui)
- [Unreal Engine Console Commands](https://docs.unrealengine.com/5.5/en-US/console-commands-in-unreal-engine/)
- [DrawDebugHelpers Documentation](https://docs.unrealengine.com/5.5/en-US/API/Runtime/Engine/DrawDebugHelpers/)

## Troubleshooting

**Problem**: ImGui window not appearing
**Solution**: Make sure the ImGui plugin is properly installed and enabled

**Problem**: Debug component not found
**Solution**: Verify that the component is properly attached to your GameMode

**Problem**: Debug window not responding to Alt+D shortcut
**Solution**: Check your input bindings and make sure the action is properly set up

**Problem**: Debug visualization not working
**Solution**: Ensure that the proper rendering flags are enabled 