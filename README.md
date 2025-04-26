# 2.5D Platformer Implementation Guide

This repository contains a comprehensive guide to implementing a 2.5D platformer game in Unreal Engine 5.3+. The game features 2D gameplay (movement along a fixed plane) within a 3D environment, similar to games like Little Nightmares, Inside, or Crash Bandicoot. Each markdown file covers a specific aspect of development, with step-by-step instructions, code samples, and success criteria.

## Table of Contents

1. **[Project Setup](project_setup.md)**
   - Initial project creation
   - Required plugins
   - Perforce version control setup
   - Folder structure
   - Project settings

2. **[Version Control](version_control.md)**
   - Perforce setup for Unreal Engine
   - Stream strategy
   - Submit best practices
   - Unreal-specific considerations
   - Collaborative workflows
   - File locking and management

3. **[Core Framework](core_framework.md)**
   - Game instance
   - Game mode
   - Interface definitions
   - Debug systems
   - Collision setup
   - GameplayAbilitySystem configuration

4. **[Movement System](movement_system.md)**
   - Character base class
   - 2.5D movement constraints (2D gameplay in 3D world)
   - Camera tracking system
   - Wall jumping
   - Slope sliding
   - Enhanced movement controls
   - Blueprint extensions

5. **[Combat System](combat_system.md)**
   - Attack input
   - Combat component
   - Projectile system
   - GAS integration
   - Enemy health system
   - Blueprint extensions
   - Animation integration

6. **[UI System](ui_system.md)**
   - HUD class
   - CommonUI integration
   - Pause menu
   - Game over screen
   - Collectible counter
   - Game state connection
   - Input hints

7. **[Level Design](level_design.md)**
   - Platform classes
   - Collectible items
   - Hazards and obstacles
   - Level components
   - Test level building
   - Level streaming
   - Environmental polish

8. **[Animation System](animation_setup.md)**
   - Animation blueprint
   - Attack montages
   - Animation notify system
   - Root motion
   - Animation blending
   - Animation blueprint interface
   - Camera follow
   - Enemy animations

9. **[Gameplay Systems](gameplay_systems.md)**
   - Health system
   - Combat mechanics
   - Collectible implementation
   - Game state and progress tracking
   - User interface systems
   - Performance considerations

10. **[Testing Framework](TestingGuide.md)**
    - Blueprint functional tests
    - C++ unit tests
    - Test organization
    - Running tests locally and in CI
    - Troubleshooting test failures

## Usage Guide

1. Start with the project setup and work through each section sequentially
2. Each file includes SUCCESS CHECK criteria to verify completion
3. Code samples are provided in C++ with explanations
4. Blueprint steps are outlined for visual configuration

## Plugin Requirements

- EnhancedInput (required)
- CommonUI (strongly recommended)
- GameplayAbilitySystem (strongly recommended)

## Development Strategy

This guide follows a hybrid approach using both C++ and Blueprints:
- C++ for core systems, performance, and architecture
- Blueprints for iteration and visual logic
- Editor tools for visual design, tweaking, and asset integration

This creates clear separation of concerns:
- C++ defines what's POSSIBLE
- Blueprints define what HAPPENS
- Editor tools define how things LOOK and FEEL

## What is 2.5D?

Our approach to 2.5D involves:
- Character movement constrained to a fixed plane (generally along the X and Z axes)
- 3D environment and visuals that create depth and visual richness 
- Perspective camera that follows the character along the gameplay path
- Core gameplay mechanics (movement, jumping, combat) operating in essentially 2D space

This gives the game visual depth while maintaining the tight, predictable controls of a 2D platformer.

## First Development Goal

Your first development goal should be creating a character with smooth 2.5D movement in a test level. This foundation provides the core experience of your platformer and is essential before adding more complex systems.

## Learning Path

This guide assumes basic familiarity with Unreal Engine but explains concepts as they're introduced. The step-by-step approach is designed to build knowledge incrementally.

For best results, implement each section completely before moving to the next one, ensuring you meet all the success criteria before continuing.

## CI/CD Setup with GitHub Actions

This project uses GitHub Actions for continuous integration and deployment. The workflow automatically runs tests and builds the game when changes are pushed.

### Setting Up Self-Hosted Runner

To run the CI/CD pipeline, you need to set up a self-hosted GitHub Actions runner on your development machine:

1. In your GitHub repository, go to Settings > Actions > Runners
2. Click "New self-hosted runner"
3. Follow the instructions to download and configure the runner
4. Start the runner as a service

### Test Framework

The project uses Unreal's Automation Testing Framework. Tests are located in:
```
Content/Tests/
```

To create new tests:
1. In Unreal Editor, go to Window > Developer Tools > Session Frontend
2. Navigate to the Automation tab
3. Create new Blueprint Function Tests in Content/Tests/ directory

### Running Tests Locally

You can run tests locally with:
```bash
"C:/Program Files/Epic Games/UE_5.3/Engine/Binaries/Win64/UnrealEditor-Cmd.exe" "[YourGameName].uproject" -ExecCmds="Automation RunTests Platformer.Tests;Quit" -TestExit="Automation Test Queue Empty" -Unattended -NullRHI -Log
```

### CI/CD Pipeline Workflow

The GitHub Actions workflow consists of:
1. Running all automation tests on code changes
2. Building a packaged game version for Windows
3. Uploading the build as an artifact (available for 7 days)

This ensures that all code changes are tested before building, and provides a downloadable game build for each successful pipeline run.

## Implementation Status and Next Steps

### Current Status

✅ Project structure complete  
✅ Testing framework established  
✅ CI/CD pipeline configured  
✅ Core gameplay systems designed  
✅ Movement system documented  
✅ Level design guidelines created  
✅ Animation setup outlined  

### Recommended Next Steps

1. **Setup Your Project Environment**
   - Follow [project_setup.md](project_setup.md) to create your initial project
   - Configure Perforce for version control
   - Set up the self-hosted runner for CI/CD

2. **Implement Core Character Movement**
   - Start with the character movement system as your foundation
   - Create and test the 2.5D movement constraints
   - Set up camera follow behavior

3. **Build a Test Level**
   - Create a simple level with platforms to test movement
   - Implement collectibles for testing gameplay elements
   - Set up a basic enemy for combat testing

4. **Create Basic Tests**
   - Implement first tests following the [TestingGuide.md](TestingGuide.md)
   - Ensure CI pipeline runs and validates your code

5. **Iterate and Refine**
   - Use the feedback from testing to refine your implementations
   - Gradually add complexity by implementing more systems

### Development Tips

- **Start Small**: Begin with a minimal viable prototype focused on core movement
- **Test Often**: Use the testing framework to validate each system as you build
- **Commit Regularly**: Make small, incremental changes with descriptive commit messages
- **Track Progress**: Use the checklists in each implementation guide to monitor completion
- **Profiling Early**: Watch for performance issues even in early development to prevent harder fixes later 