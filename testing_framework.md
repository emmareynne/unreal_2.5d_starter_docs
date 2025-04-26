# Testing Framework Implementation

## Prerequisites
- Completed steps in [project_setup.md](project_setup.md)
- Basic understanding of [Unreal Automation System](https://docs.unrealengine.com/5.3/en-US/automation-system-in-unreal-engine/)

## Step 1: Set Up Test Directories
1. Create dedicated test folders:
   ```
   Source/[YourGameName]/Tests/  # For C++ automation tests
   Content/Tests/                # For Blueprint Functional Tests
   Content/Tests/TestMaps/       # For test-specific levels
   ```

2. Add test-specific assets:
   - Create a basic test level with minimal geometry
   - Set up test-specific materials (red for fail states, green for success)
   - Consider adding a debug HUD for test feedback

**SUCCESS CHECK:** Folders created and visible in Content Browser

## Step 2: Implement Blueprint Functional Tests
1. Create a test level for gameplay systems:
   - Go to File → New Level → Empty Level
   - Save as "Content/Tests/TestMaps/BasicMovementTest"
   - Add basic platform geometry for movement testing
   - Add PlayerStart at appropriate location

2. Add [Functional Test Actor](https://docs.unrealengine.com/5.3/en-US/functional-testing-in-unreal-engine/):
   - Place in test level
   - In Details panel, set "Auto Start Test" to true
   - Set appropriate timeout (30-60 seconds)

3. Set up Movement Constraint Test:
   ```
   // In Functional Test Actor Event Graph
   
   // On Test Start
   Begin Test -> Spawn Character -> Get Character Reference
   
   // Check Constraints
   Get Character Position -> Store Initial Y Position
   Wait 2 seconds
   Get Character Position -> Get Current Y Position
   
   // Verify character hasn't moved on Y axis
   If (Current Y == Initial Y +/- tolerance) -> Pass Test
   Else -> Fail Test with message "Character moved out of 2.5D plane"
   ```

4. Set up Jump Test:
   ```
   // In Functional Test Actor Event Graph
   
   // On Test Start
   Begin Test -> Spawn Character -> Get Character Reference
   
   // Test Jump Height
   Get Character Z Position -> Store Ground Z
   Simulate Jump Input -> Wait For Max Height
   Get Character Z Position -> Store Jump Z
   
   // Verify jump meets expected height
   If (Jump Z - Ground Z >= Expected Height) -> Pass Test
   Else -> Fail Test with message "Jump height below expected value"
   ```

**SUCCESS CHECK:** Functional tests run successfully via Session Frontend

## Step 3: Implement C++ Automation Tests (Core Systems)
1. Create base test file:
   ```cpp
   // Source/[YourGameName]/Tests/PlatformerTests.h
   #pragma once
   
   #include "CoreMinimal.h"
   #include "Misc/AutomationTest.h"
   
   // Define test constants
   #define PLATFORMER_TEST_CATEGORY "Platformer"
   ```

2. Create Simple Automation Test:
   ```cpp
   // Source/[YourGameName]/Tests/MovementTests.cpp
   #include "PlatformerTests.h"
   #include "GameFramework/CharacterMovementComponent.h"
   #include "[YourGameName]/PlatformerCharacter.h"
   
   IMPLEMENT_SIMPLE_AUTOMATION_TEST(FCharacterPlaneConstraintTest, 
       PLATFORMER_TEST_CATEGORY ".Movement.PlaneConstraint",
       "Checks that character has correct plane constraint settings")
   
   bool FCharacterPlaneConstraintTest::RunTest(const FString& Parameters)
   {
       // Create test world
       UWorld* World = FAutomationEditorCommonUtils::CreateNewMap();
       if (!World)
       {
           AddError("Failed to create test world");
           return false;
       }
       
       // Spawn character
       FActorSpawnParameters SpawnParams;
       APlatformerCharacter* Character = World->SpawnActor<APlatformerCharacter>(APlatformerCharacter::StaticClass(), FVector::ZeroVector, FRotator::ZeroRotator, SpawnParams);
       if (!Character)
       {
           AddError("Failed to spawn test character");
           return false;
       }
       
       // Test plane constraint settings
       UCharacterMovementComponent* MovementComp = Character->GetCharacterMovement();
       bool bResult = true;
       
       // Check plane constraint is enabled
       if (!MovementComp->bConstrainToPlane)
       {
           AddError("Plane constraint is not enabled on character");
           bResult = false;
       }
       
       // Check constraint normal is set to Y axis
       FVector ExpectedNormal(0.0f, 1.0f, 0.0f);
       if (!MovementComp->PlaneConstraintNormal.Equals(ExpectedNormal, 0.01f))
       {
           AddError(FString::Printf(TEXT("Plane constraint normal is incorrect. Expected: %s, Actual: %s"), 
               *ExpectedNormal.ToString(), *MovementComp->PlaneConstraintNormal.ToString()));
           bResult = false;
       }
       
       // Clean up
       Character->Destroy();
       
       return bResult;
   }
   ```

3. Register module for testing (in YourGame.Build.cs):
   ```csharp
   PublicDependencyModuleNames.AddRange(new string[] { 
       "Core", 
       "CoreUObject", 
       "Engine", 
       "InputCore",
       "TestingFramework" // Add this for automation tests
   });
   ```

**SUCCESS CHECK:** C++ tests compile and appear in the Session Frontend

## Step 4: Set Up Continuous Integration (Optional)
1. Create a build script for automated testing:
   ```
   // BuildTestRunner.bat
   @echo off
   
   set UE_PATH=C:\Path\To\UE_5.3
   set PROJECT_PATH=C:\Path\To\YourProject\YourProject.uproject
   
   echo Running Automation Tests...
   %UE_PATH%\Engine\Binaries\Win64\UnrealEditor-Cmd.exe %PROJECT_PATH% -ExecCmds="Automation RunTests Platformer;Quit" -TestExit="Automation Test Queue Empty" -Unattended -NullRHI -Log
   
   echo Tests Complete!
   ```

2. Add to Perforce for team access:
   ```
   p4 add BuildTestRunner.bat
   p4 submit -d "Added test runner script"
   ```

**SUCCESS CHECK:** Tests can be run via command line

## Step 5: Create Test Documentation
1. Create a test manifest:
   - Document each test's purpose
   - Note any required setup
   - List expected results

2. Add to source control:
   ```
   p4 add Tests/TestManifest.md
   p4 submit -d "Added test documentation"
   ```

**SUCCESS CHECK:** Documentation created and accessible to team

## Common Test Types for 2.5D Platformers

### Movement Tests
- **Plane Constraint**: Verify character stays on 2.5D plane
- **Jump Height**: Confirm jumps reach expected height
- **Wall Jump**: Test wall detection and jump response
- **Slope Sliding**: Verify character slides at correct speed

### Camera Tests
- **Camera Tracking**: Ensure camera follows player correctly
- **Camera Boundaries**: Test camera stops at level boundaries
- **Look-Ahead**: Verify camera provides appropriate look-ahead based on movement

### Combat Tests (When Implemented)
- **Attack Range**: Verify attack hitboxes have correct dimensions
- **Damage Application**: Test damage is applied correctly
- **Projectile Movement**: Ensure projectiles move along 2.5D plane

### Collectible Tests
- **Pickup Detection**: Verify items can be collected
- **Inventory Updates**: Confirm collected items appear in inventory
- **Score Calculation**: Test score increases appropriately

## Next Steps
Once your basic test framework is in place, begin adding specific tests for each core system as it's implemented. Prioritize testing critical gameplay mechanics early.

## Resources
- [Unreal Automation System Documentation](https://docs.unrealengine.com/5.3/en-US/automation-system-in-unreal-engine/)
- [Functional Testing in Unreal](https://docs.unrealengine.com/5.3/en-US/functional-testing-in-unreal-engine/)
- [Automation Technical Guide](https://docs.unrealengine.com/5.3/en-US/automation-technical-guide/) 

## Lightweight CI/CD for Solo Developers and Small Teams

### Prerequisites
- Perforce Helix Core server set up and configured
- Unreal Engine 5.3+ project under Perforce version control
- Basic testing framework implemented as outlined above

### Step 1: Set Up Local Pre-Submit Test Scripts

1. **Create a pre-submit test script**:
   ```batch
   @echo off
   :: pre_submit_tests.bat
   
   set UE_PATH=C:\Path\To\UE_5.3
   set PROJECT_PATH=%~dp0[YourGameName].uproject
   
   echo Running critical tests before submission...
   
   :: Run only the most critical tests for quick validation
   %UE_PATH%\Engine\Binaries\Win64\UnrealEditor-Cmd.exe %PROJECT_PATH% -ExecCmds="Automation RunTests Platformer.Tests.Critical;Quit" -TestExit="Automation Test Queue Empty" -Unattended -NullRHI -Log
   
   if %ERRORLEVEL% NEQ 0 (
       echo ERROR: Critical tests failed! Fix issues before submitting.
       pause
       exit /b 1
   )
   
   echo All critical tests passed! Safe to submit.
   exit /b 0
   ```

2. **Create a simple test executor** in the Unreal Editor:
   - Create a new Editor Utility Widget (EUW) named "TestRunner"
   - Add buttons for running different test sets:
     - "Run Movement Tests"
     - "Run Camera Tests"
     - "Run All Tests"
   - Implement functionality to execute the appropriate tests

3. **Add your script to source control**:
   ```
   p4 add pre_submit_tests.bat
   p4 submit -d "Added pre-submit test script"
   ```

**SUCCESS CHECK:** Test script runs successfully and validates core functionality

### Step 2: Implement GitHub Actions for Basic CI (Optional)

If you're also using GitHub for issue tracking or PR reviews:

1. **Create a simple GitHub Action workflow**:
   ```yaml
   # .github/workflows/unreal-tests.yml
   name: Unreal Engine Tests
   
   on:
     push:
       branches: [ main, develop ]
     pull_request:
       branches: [ main, develop ]
   
   jobs:
     test:
       runs-on: self-hosted # Requires a self-hosted runner on your dev machine
       steps:
         - name: Sync from Perforce
           run: p4 sync //[YourGameName]/...
           
         - name: Run Tests
           run: |
             "C:/Program Files/Epic Games/UE_5.3/Engine/Binaries/Win64/UnrealEditor-Cmd.exe" "%P4_WORKSPACE%/[YourGameName].uproject" -ExecCmds="Automation RunTests Platformer.Tests;Quit" -TestExit="Automation Test Queue Empty" -Unattended -NullRHI -Log
   ```

**SUCCESS CHECK:** Basic CI workflow runs when code is pushed

### Step 3: Create Automated Nightly Builds with Windows Task Scheduler

1. **Create a build script**:
   ```batch
   @echo off
   :: nightly_build.bat
   
   set UE_PATH=C:\Path\To\UE_5.3
   set PROJECT_PATH=C:\Path\To\YourProject\[YourGameName].uproject
   set BUILD_DIR=C:\Builds\%date:~-4,4%%date:~-7,2%%date:~-10,2%
   
   echo Creating build directory...
   mkdir %BUILD_DIR%
   
   echo Syncing latest from Perforce...
   p4 sync //[YourGameName]/...
   
   echo Running all tests...
   %UE_PATH%\Engine\Binaries\Win64\UnrealEditor-Cmd.exe %PROJECT_PATH% -ExecCmds="Automation RunTests Platformer.Tests.All;Quit" -TestExit="Automation Test Queue Empty" -Unattended -NullRHI -Log -log="%BUILD_DIR%\TestResults.log"
   
   echo Packaging game...
   %UE_PATH%\Engine\Build\BatchFiles\RunUAT.bat BuildCookRun -project=%PROJECT_PATH% -noP4 -platform=Win64 -clientconfig=Development -cook -compressed -stage -pak -archive -archivedirectory=%BUILD_DIR%
   
   echo Build complete! Output located at %BUILD_DIR%
   ```

2. **Set up Windows Task Scheduler**:
   - Open Task Scheduler
   - Create Basic Task → Name it "Nightly Game Build"
   - Set trigger to daily at a time when you're not using your computer
   - Action: Start a program → Browse to your nightly_build.bat
   - Finish and save the task

**SUCCESS CHECK:** Automated builds run nightly without manual intervention

### Step 4: Implement Simple Build Notification System

1. **Modify your build script to send notifications**:
   ```batch
   :: At the end of your nightly_build.bat
   
   if %ERRORLEVEL% NEQ 0 (
       echo Build failed! Sending notification...
       powershell -Command "Send-MailMessage -From 'build@yourdomain.com' -To 'you@yourdomain.com' -Subject 'Nightly Build Failed!' -Body 'The nightly build failed. Check the logs at %BUILD_DIR%\TestResults.log for details.' -SmtpServer 'your.smtp.server'"
   ) else (
       echo Build succeeded! Sending notification...
       powershell -Command "Send-MailMessage -From 'build@yourdomain.com' -To 'you@yourdomain.com' -Subject 'Nightly Build Succeeded' -Body 'The nightly build completed successfully. New build available at %BUILD_DIR%.' -SmtpServer 'your.smtp.server'"
   )
   ```

2. **For team collaboration**, use Discord webhooks:
   ```batch
   :: Alternative notification using Discord webhook
   curl -H "Content-Type: application/json" -d "{\"content\": \"Nightly build completed with status: %ERRORLEVEL%\"}" https://discord.com/api/webhooks/your/webhook/url
   ```

**SUCCESS CHECK:** Team receives notifications about build status

### Step 5: Implement Visual Regression Testing for 2.5D Elements

Even for a small team, visual testing is critical for 2.5D games:

1. **Create a simple screenshot test**:
   ```cpp
   // Add to your test suite
   IMPLEMENT_SIMPLE_AUTOMATION_TEST(FScreenshotTest, 
       PLATFORMER_TEST_CATEGORY ".Visual.Camera2DAlignment",
       "Takes screenshots to verify camera alignment remains consistent")
   
   bool FScreenshotTest::RunTest(const FString& Parameters)
   {
       // Load test map with reference camera position
       AutomationOpenMap("/Game/Tests/TestMaps/CameraAlignmentTest");
       
       // Position character at test location
       APlayerController* PlayerController = GEngine->GetFirstLocalPlayerController(GWorld);
       if (!PlayerController)
       {
           AddError("Could not get player controller");
           return false;
       }
       
       // Move to test position
       PlayerController->SetControlRotation(FRotator(0, 90, 0));
       PlayerController->GetPawn()->SetActorLocation(FVector(0, 0, 100));
       
       // Take screenshot
       FString ScreenshotName = FString::Printf(TEXT("Camera2D_%s"), *FDateTime::Now().ToString());
       AutomationCommon::TakeGameScreenshot(GWorld, ScreenshotName, false);
       
       AddInfo(FString::Printf(TEXT("Screenshot saved as %s"), *ScreenshotName));
       return true;
   }
   ```

2. **Create a simple image comparison tool**:
   - Store "golden" reference images in your repo
   - Use a simple image comparison library to check for differences
   - Flag tests as failing if differences exceed threshold

**SUCCESS CHECK:** Visual regression tests catch unexpected camera or perspective changes

### Best Practices for Small Teams

1. **Focus on the most critical tests**:
   - 2D plane constraints (highest priority)
   - Camera behavior
   - Core jump mechanics
   - Basic performance benchmarks
   
2. **Automate what hurts**:
   - Identify what breaks most often and test it first
   - Prioritize tests that catch issues you've had to fix repeatedly
   
3. **Keep build times short**:
   - For pre-submit tests, aim for under 2 minutes
   - For nightly builds, under 30 minutes
   
4. **Use editor tooling extensively**:
   - Create Editor Utility Widgets for common test operations
   - Make test running as simple as clicking a button
   
5. **Version your test assets properly**:
   - Keep test maps and data in Perforce
   - Ensure all team members have access to the same test environment

### Resources for Lightweight Game CI/CD
- [Unreal Automation System Documentation](https://docs.unrealengine.com/5.3/en-US/automation-system-in-unreal-engine/)
- [Windows Task Scheduler Tutorial](https://learn.microsoft.com/en-us/windows/win32/taskschd/task-scheduler-start-page)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Discord Webhook Documentation](https://discord.com/developers/docs/resources/webhook) 