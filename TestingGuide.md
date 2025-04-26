# Unreal Engine Testing Guide for 2.5D Platformer

## Test Types

### 1. Blueprint Functional Tests
Blueprint Functional Tests allow you to create automated tests directly in the Unreal Editor. These are great for testing gameplay mechanics.

#### Creating a Basic Movement Test

1. In the Content Browser, navigate to **Content/Tests/**
2. Right-click and select **Blueprint Class**
3. Choose **Functional Test** as the parent class
4. Name it `BP_Test_PlayerMovement`
5. Open the new Blueprint and set up:
   - Set **Number of Participants** to 1
   - Add a **Player Start** component to position the player
   - In the Event Graph, implement the test logic:

```
Event Receive Start Test
|
+-- Spawn Player Character at PlayerStart location
|
+-- Wait 0.5 seconds
|
+-- Move Player Right (Simulate Input)
|
+-- Wait 1.0 seconds
|
+-- Get Player Location
|
+-- Assert: Player X position > starting X position
|
+-- If True → Finish Test (Success)
+-- If False → Finish Test (Failed)
```

### 2. C++ Unit Tests

For more complex systems, use C++ unit tests:

1. Create a new test module in your project:
   ```cpp
   // PlatformerTests.cpp
   #include "CoreMinimal.h"
   #include "Misc/AutomationTest.h"
   #include "PlatformerCharacter.h"
   #include "Engine/World.h"
   
   IMPLEMENT_SIMPLE_AUTOMATION_TEST(FPlatformerJumpTest, "Platformer.Tests.JumpTest", 
       EAutomationTestFlags::ApplicationContextMask | EAutomationTestFlags::ProductFilter)
   
   bool FPlatformerJumpTest::RunTest(const FString& Parameters)
   {
       // Create test world
       UWorld* World = UWorld::CreateWorld(EWorldType::Game, false);
       FWorldContext& WorldContext = GEngine->CreateNewWorldContext(EWorldType::Game);
       WorldContext.SetCurrentWorld(World);
       
       // Spawn character
       FActorSpawnParameters SpawnParams;
       APlatformerCharacter* Character = World->SpawnActor<APlatformerCharacter>(SpawnParams);
       
       // Initial height
       float InitialZ = Character->GetActorLocation().Z;
       
       // Simulate jump
       Character->Jump();
       
       // Tick world to simulate physics
       World->Tick(ELevelTick::LEVELTICK_All, 0.1f);
       
       // Get new height
       float NewZ = Character->GetActorLocation().Z;
       
       // Verify character moved upward
       bool bJumpWorked = NewZ > InitialZ;
       
       // Cleanup
       GEngine->DestroyWorldContext(World);
       World->DestroyWorld(false);
       
       return bJumpWorked;
   }
   ```

## Running Tests

### Local Testing

#### Blueprint Tests
1. Open Unreal Editor
2. Open the Session Frontend (Window → Developer Tools → Session Frontend)
3. Go to the Automation tab
4. Enable the "Platformer.Tests" group
5. Click "Start Tests"

#### Command Line Testing
```bash
"C:/Program Files/Epic Games/UE_5.3/Engine/Binaries/Win64/UnrealEditor-Cmd.exe" "PlatformerGame.uproject" -ExecCmds="Automation RunTests Platformer.Tests;Quit" -TestExit="Automation Test Queue Empty" -Unattended -NullRHI -Log
```

### CI/CD Testing
Tests will run automatically via GitHub Actions on your self-hosted runner when:
- You push to main or develop branch
- You create a Pull Request to these branches
- You manually trigger the workflow

## Test Organization

Structure your tests like this:
```
Content/
└── Tests/
    ├── Blueprint/
    │   ├── BP_Test_PlayerMovement
    │   ├── BP_Test_WallJump
    │   ├── BP_Test_GroundPound
    │   └── BP_Test_Collectibles
    ├── Levels/
    │   ├── TEST_BasicMovement
    │   └── TEST_CollisionDetection
    └── Data/
        └── TEST_EnemyData
```

## Test Coverage Goals

- **Core Movement:** Basic movement, jumping, wall jumping, ground pound
- **Camera System:** Camera follows player correctly, handles level boundaries
- **Collectibles:** Player can collect items, score updates
- **Combat:** Player attacks, enemy behavior, damage system
- **Level Mechanics:** Platforms, hazards, level transitions

## Troubleshooting Tests

If tests fail in CI but pass locally:
1. Check log output in GitHub Actions
2. Look for environment differences (engine version, plugins)
3. Add more debug output to failing tests
4. Try running with the `-log` parameter locally to compare logs

## Setting Up Custom Test Maps

For complex testing scenarios:
1. Create a new level named "TEST_[Feature]"
2. Add only the necessary components
3. Place Functional Test actors in strategic locations
4. Use level streaming for performance in large test suites 