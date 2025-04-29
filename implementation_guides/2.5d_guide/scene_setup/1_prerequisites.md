# Prerequisites for Scene Setup

This document outlines the prerequisites for setting up the initial scene for our 2.5D platformer in Unreal Engine 5.5, including required plugins and installation instructions.

## Required Plugins

Before beginning the scene setup, ensure you have the following plugins installed and enabled in your project:

### Core Plugins

1. **Enhanced Input**
   - **Purpose:** Modern input system with contextual mapping and advanced processing
   - **Status:** Built-in with UE5.5, needs to be enabled
   - **Required:** Yes, essential for responsive platformer controls

2. **Gameplay Ability System (GAS)**
   - **Purpose:** Framework for implementing abilities, effects, and attributes
   - **Status:** Built-in with UE5.5, needs to be enabled
   - **Required:** Yes, foundation for character abilities and progression

3. **CommonUI**
   - **Purpose:** Standardized UI framework with better input handling
   - **Status:** Built-in with UE5.5, needs to be enabled
   - **Required:** Yes, for consistent UI implementation

4. **Niagara**
   - **Purpose:** Advanced VFX system for gameplay feedback
   - **Status:** Built-in with UE5.5, needs to be enabled
   - **Required:** Yes, for ability effects and environmental feedback

## Animation Approach

Instead of using the beta-status Motion Warping plugin, we'll implement a more stable animation solution:

### Animation Montages with Custom Offsets

- **Core Technique:** Combine standard Animation Montages with programmatic position adjustments
- **Implementation:** Use Animation Blueprints with timeline-based position corrections
- **Benefits:** Production-ready technology, stable across UE versions, good performance
- **Structure:** Character animations will use montages with custom code for environmental alignment

This approach provides excellent animation quality while avoiding beta-status plugins.

## Architecture Approach

For this project, we'll use a **Component-Based Architecture** rather than ModularGameplay:

- **Core Philosophy:** Break down functionality into focused, reusable components
- **Implementation:** Extend UE's Actor Component system for specialized behaviors
- **Benefits:** Better modularity, easier testing, cleaner dependency management
- **Structure:** Character will consist of specialized components for movement, combat, etc.

This approach provides excellent composition flexibility while avoiding beta-status plugins.

## Installation Process

### Step 1: Enable Built-in Plugins

1. Open your Unreal Engine 5.5 project
2. Go to **Edit → Plugins** from the main menu
3. Enable each of the required plugins:

   a. **Enhanced Input**
      - Navigate to the **Input** category
      - Find "Enhanced Input" and check the **Enabled** checkbox

   b. **Gameplay Abilities**
      - Navigate to the **Gameplay** category
      - Find "Gameplay Abilities" and check the **Enabled** checkbox

   c. **CommonUI**
      - Navigate to the **UI** category
      - Find "CommonUI" and check the **Enabled** checkbox
      
   d. **Niagara**
      - Navigate to the **FX** category
      - Find "Niagara" and check the **Enabled** checkbox

4. Click **Restart Now** when prompted to restart the editor

![Plugin Menu](https://docs.unrealengine.com/5.3/Images/designing-visuals/rendering-optimization/ray-tracing/ray-tracing-resources/EnableRayTracingPlugin.webp)

### Step 2: Configure Build Files for GAS

The Gameplay Ability System requires additional module dependencies in your project's build files:

1. Navigate to your project's Source directory
2. Open `YourProjectName.Build.cs` in a code editor
3. Add the required module dependencies:

```csharp
public class YourProjectName : ModuleRules
{
    public YourProjectName(ReadOnlyTargetRules Target) : base(Target)
    {
        PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;
        
        PublicDependencyModuleNames.AddRange(new string[] { 
            "Core", 
            "CoreUObject", 
            "Engine", 
            "InputCore",
            // Add these GAS-related modules
            "GameplayAbilities",
            "GameplayTags",
            "GameplayTasks"
        });
        
        // ... rest of your build file
    }
}
```

4. Save the file
5. Right-click on your `.uproject` file and select **Generate Visual Studio project files**
6. Rebuild your project in your IDE (Visual Studio/Rider)

### Step 3: Animation System Setup

For our Animation Montage approach with custom positioning:

1. Create a base Animation Blueprint for your character
2. Set up state machine with primary character states (Idle, Move, Jump, Fall)
3. Create Animation Montages for special actions:
   - Jump montage with phases for launch, air, and landing
   - Ledge grab montage for edge interactions
   - Attack montages for combat actions
4. Add custom events in Animation Blueprint to trigger positional adjustments
5. Create a Timeline component in your character for smooth position blending

This setup provides all the functionality of Motion Warping using production-ready technology.

### Step 4: Verify Plugin Status

After enabling all required plugins and restarting the editor:

1. Create a new C++ class to verify access to plugin functionality
2. Check that you can include headers from the enabled plugins:

```cpp
#include "AbilitySystemComponent.h"
#include "EnhancedInputComponent.h"
#include "CommonUI/Public/CommonActivatableWidget.h"
#include "NiagaraComponent.h"
```

3. If you get compiler errors related to missing headers, double-check:
   - Plugin enabled status
   - Build.cs dependencies
   - Whether your project was properly rebuilt

## Planning Your Component Architecture

For our 2.5D platformer, consider these specialized components:

1. **PlatformerMovementComponent**
   - Handles ground movement, friction, acceleration
   - Manages character states (running, walking, crouching)

2. **PlatformerJumpComponent**
   - Manages jump height, count, and recovery
   - Handles wall jumps, double jumps, and other aerial abilities
   - Includes position adjustment logic for precise landings

3. **PlatformerCombatComponent**
   - Processes attack inputs and chains
   - Manages hitboxes, damage, and combat states

4. **PlatformerInteractionComponent**
   - Handles item pickups and environment interaction
   - Manages interactive objects detection
   - Controls ledge grab detection and position alignment

5. **PlatformerAnimationComponent**
   - Coordinates state-based animation
   - Handles animation events and feedback
   - Manages montage playback and position adjustments

This structure keeps your code modular while avoiding beta-status plugins.

## Troubleshooting Common Issues

### Plugin Won't Enable

**Issue:** Plugin appears in the list but can't be enabled (checkbox is disabled).

**Solution:** Some plugins have dependencies on other plugins or specific project types.
- Ensure your project is a C++ project (Blueprint-only projects have limitations)
- Check for required dependency plugins
- Verify your engine version supports the plugin

**Documentation:** [Plugin Dependencies](https://docs.unrealengine.com/5.3/en-US/plugins-in-unreal-engine/)

### Missing GAS Classes After Enabling

**Issue:** GAS plugin is enabled but classes like `UAbilitySystemComponent` aren't found.

**Solution:** Ensure proper module dependencies and header includes:
- Verify `GameplayAbilities`, `GameplayTags`, and `GameplayTasks` are in your build.cs
- Make sure to include the correct headers
- Regenerate project files and rebuild

**Documentation:** [GAS Setup Guide](https://docs.unrealengine.com/5.3/en-US/gameplay-ability-system-for-unreal-engine/)

### Enhanced Input Not Working

**Issue:** Enhanced Input plugin is enabled but input doesn't work.

**Solution:** Enhanced Input requires specific setup beyond enabling the plugin:
- Confirm your project settings use the Enhanced Input system
- Create Input Mapping Contexts and Input Actions
- Set up your character to use Enhanced Input

**Documentation:** [Enhanced Input Overview](https://docs.unrealengine.com/5.3/en-US/enhanced-input-in-unreal-engine/)

## Additional Resources

### Official Documentation

- [Gameplay Ability System Documentation](https://docs.unrealengine.com/5.3/en-US/gameplay-ability-system-for-unreal-engine/)
- [Enhanced Input System Documentation](https://docs.unrealengine.com/5.3/en-US/enhanced-input-in-unreal-engine/)
- [CommonUI Documentation](https://docs.unrealengine.com/5.3/en-US/common-ui-for-unreal-engine-projects/)
- [Niagara Documentation](https://docs.unrealengine.com/5.3/en-US/niagara-effects-for-unreal-engine/)
- [Animation Montage Documentation](https://docs.unrealengine.com/5.3/en-US/animation-montage-in-unreal-engine/)

### Community Resources

- [GAS Companion Plugin](https://github.com/tranek/GASDocumentation) (Optional but helpful)
- [GAS Documentation Project](https://github.com/tranek/GASDocumentation)
- [Enhanced Input Community Guide](https://dev.epicgames.com/community/learning/tutorials/l7Dz/unreal-engine-enhanced-input-subsystem)

## Next Steps

Once you have all prerequisites installed and configured:

1. Proceed to [Initial Scene Setup Overview](./0_initial_scene_overview.md)
2. Create your first test level using the guide
3. Begin implementing the core gameplay systems 