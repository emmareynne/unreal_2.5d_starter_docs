# 2.5D Platformer Project Setup

## Step 1: Create New Project
1. Launch [Unreal Engine 5.3+](https://www.unrealengine.com/download)
2. Click "New Project"
3. Select "Games" tab
4. Choose "[Third Person template](https://docs.unrealengine.com/5.3/en-US/creating-a-new-project-in-unreal-engine/)" (C++ version, NOT Blueprint)
5. Set project name: "[YourGameName]"
6. Set location for your project
7. Click "Create"

**SUCCESS CHECK:** Engine launches with template character in third-person view

## Step 2: Enable Required Plugins
1. Go to Edit → Plugins
2. Enable the following (check if already enabled):
   - **[Enhanced Input](https://docs.unrealengine.com/5.3/en-US/enhanced-input-in-unreal-engine/)** (Input category)
   - **[CommonUI](https://docs.unrealengine.com/5.3/en-US/common-ui-plugin-for-unreal-engine/)** (User Interface category)
   - **[Gameplay Abilities](https://docs.unrealengine.com/5.3/en-US/gameplay-ability-system-for-unreal-engine/)** (Gameplay category)
   - **[Paper2D](https://docs.unrealengine.com/5.3/en-US/paper-2d-in-unreal-engine/)** (2D category) - Optional, useful for some 2.5D elements

**SUCCESS CHECK:** All plugins enabled without errors, project recompiles successfully

## Step 3: Create Basic Folder Structure
1. In Content Browser, right-click → New Folder
2. Create these folders according to [best practices](https://docs.unrealengine.com/5.3/en-US/unreal-engine-directory-structure-reference/):
   - Blueprints
   - Characters
   - Levels
   - UI
   - Animations
   - FX
   - Materials
   - Environment

**SUCCESS CHECK:** Folder structure visible in Content Browser

## Step 4: Set Up Git LFS Version Control
See VCS doc. you should use Git lfs and project borealis.

## Step 5: Configure Project Settings for 2.5D
1. Go to Edit → Project Settings
2. Under "Project → Maps & Modes":
   - Set Default GameMode to a new custom GameMode (we'll create this)
   - Set Default Maps for Editor/Game
3. Under "Engine → Input":
   - Verify [Enhanced Input System](https://docs.unrealengine.com/5.3/en-US/enhanced-input-in-unreal-engine/) is set as default
4. Under "Engine → Rendering":
   - Consider adjusting the default post-processing settings to enhance the 2.5D visuals
5. Under "Engine → Physics":
   - Set default gravity to match your desired platforming feel (default is -980, you might want to adjust)

**SUCCESS CHECK:** Project settings saved without errors

## Step 6: Configure Camera for 2.5D View
1. Create a basic test level
2. Add a floor plane and some test platforms
3. Place your character in the level
4. Create a new Blueprint based on your `SideScrollCameraArm` component (we'll implement this later)
5. Add the camera component to your character blueprint:
   - Place a spring arm component on your character (or use `SideScrollCameraArm`)
   - Add a camera component as a child of the spring arm
   - Configure the spring arm length to get appropriate distance from the character
   - Position the camera to look perpendicular to your 2.5D plane

**SUCCESS CHECK:** Camera provides a proper side-scrolling view of the 2.5D environment

## Step 7: Test Basic Movement Constraints
1. Open your character blueprint
2. Add a Plane Constraint component if not already present
3. Set the constraint normal to (0,1,0) to lock Y-axis movement
4. Test the character can only move along the X and Z axes

**SUCCESS CHECK:** Character movement is constrained to the 2.5D plane

## Step 8: Set Up Testing Framework
1. Create a "Tests" folder in your project structure:
   - Create "Source/[YourGameName]/Tests" for C++ tests
   - Create "Content/Tests" for Blueprint Functional Tests
   
2. Set up Blueprint Functional Tests:
   - Create a test level in "Content/Tests/TestMaps"
   - Add a "[Functional Test Actor](https://docs.unrealengine.com/5.3/en-US/functional-testing-in-unreal-engine/)" to test basic movement
   - Configure it to validate character constraints

3. Set up C++ Automation Tests (optional for core systems):
   - Create base test files in the "Tests" directory
   - Set up simple tests for your core game systems
   - These will run in the [Session Frontend](https://docs.unrealengine.com/5.3/en-US/automation-technical-guide/)

**SUCCESS CHECK:** Functional Test Actor placed in test level, basic test runs successfully

## Next Steps
Once complete, move on to setting up a more comprehensive version control workflow as outlined in `version_control.md`.

## Resources
- [Git LFS Documentation](https://git-lfs.github.com/)
- [Project Borealis GitHub](https://github.com/ProjectBorealis/PBCharacterMovement)
- [Unreal Engine Git Integration](https://docs.unrealengine.com/5.3/en-US/using-git-source-control-with-unreal-engine/)
- [Unreal Engine Project Structure Best Practices](https://docs.unrealengine.com/5.3/en-US/unreal-engine-directory-structure-reference/) 