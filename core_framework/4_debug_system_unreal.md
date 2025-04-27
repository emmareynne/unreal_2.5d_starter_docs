# Debug System: Unreal Engine 5.5 Debug Tools

This guide covers Unreal Engine's built-in debug tools that complement our custom ImGui-based debug system.

## Built-in Unreal Console Commands

Unreal Engine has a rich set of console commands for debugging. Here are some of the most useful ones for 2.5D platformer development:

### Rendering Debug

```
stat fps               // Display framerate
stat unit              // Display frame time breakdown
stat unitgraph         // More detailed performance graph
stat game              // Game thread stats
stat gpu               // GPU stats
stat scenerendering    // Scene rendering stats
stat memory            // Memory usage stats
stat anim              // Animation stats
stat particles         // Particle system stats
r.setres 1920x1080     // Change resolution
r.ScreenPercentage 100 // Set resolution scale (50-200)

// Visualization modes
viewmode wireframe
viewmode unlit
viewmode lit
viewmode shadercomplexity
```

### Gameplay Debugging

```
showdebug gameplay     // Show gameplay debug info
showdebug ai           // Show AI debug info
showdebug physics      // Show physics debug info
showdebug collision    // Show collision debug info
pause                  // Pause the game
slomo 0.5              // Set time dilation to half speed
slomo 1                // Reset time dilation to normal
```

### Camera Commands

```
camera free            // Free camera mode
camera thirdperson     // Third-person camera mode
fov 90                 // Set field of view to 90 degrees
```

## Using the Output Log

The Output Log window in the Unreal Editor (Window → Developer Tools → Output Log) displays various messages, warnings, and errors. You can add your own messages using:

```cpp
UE_LOG(LogTemp, Display, TEXT("Normal message"));
UE_LOG(LogTemp, Warning, TEXT("Warning message"));
UE_LOG(LogTemp, Error, TEXT("Error message"));

// With variables
UE_LOG(LogTemp, Display, TEXT("Player position: %s"), *PlayerPosition.ToString());
```

## Creating Custom Log Categories

Define a custom log category in your game's header file:

```cpp
// In YourGame.h
DECLARE_LOG_CATEGORY_EXTERN(LogYourGame, Log, All);

// In YourGame.cpp
DEFINE_LOG_CATEGORY(LogYourGame);

// Usage
UE_LOG(LogYourGame, Display, TEXT("Game-specific log message"));
```

## Visual Logger

The Visual Logger is a powerful tool for debugging gameplay systems over time:

```cpp
// Setup in your GameMode or component
void AYourGameMode::BeginPlay()
{
    Super::BeginPlay();
    
    FVisualLogger::Get().SetIsRecording(true);
}

// Log events with visual representation
UE_VLOG(this, LogYourGame, Log, TEXT("Jump started at height: %f"), GetActorLocation().Z);
UE_VLOG_LOCATION(this, LogYourGame, Log, GetActorLocation(), 10.0f, FColor::Red, TEXT("Player jumped"));
UE_VLOG_SEGMENT(this, LogYourGame, Log, StartLocation, EndLocation, FColor::Green, TEXT("Movement Path"));
```

Open the Visual Logger in Unreal Editor via Window → Developer Tools → Visual Logger.

## Draw Debug Helpers

Use these functions for temporary visual debugging:

```cpp
// Draw a debug point
DrawDebugPoint(GetWorld(), Location, 10.0f, FColor::Red, false, 5.0f);

// Draw a debug line
DrawDebugLine(GetWorld(), Start, End, FColor::Green, false, 5.0f, 0, 2.0f);

// Draw a debug box
DrawDebugBox(GetWorld(), Center, Extent, FColor::Blue, false, 5.0f);

// Draw a debug sphere
DrawDebugSphere(GetWorld(), Center, Radius, 12, FColor::Yellow, false, 5.0f);

// Draw a debug capsule
DrawDebugCapsule(GetWorld(), Center, HalfHeight, Radius, Rotation, FColor::Cyan, false, 5.0f);

// Draw debug text
DrawDebugString(GetWorld(), Location, Text, nullptr, FColor::White, 5.0f);
```

For persistent debug drawing that stays on screen, set the bPersistentLines parameter to true:

```cpp
DrawDebugLine(GetWorld(), Start, End, FColor::Green, true, -1.0f, 0, 2.0f);
```

## Debug Camera Control

Add this to your PlayerController to enable a debug camera:

```cpp
// In your PlayerController.h
UFUNCTION(Exec)
void ToggleDebugCamera();

// In your PlayerController.cpp
void AYourPlayerController::ToggleDebugCamera()
{
    static bool bDebugCameraActive = false;
    bDebugCameraActive = !bDebugCameraActive;
    
    if (bDebugCameraActive)
    {
        // Store current view target
        OriginalViewTarget = GetViewTarget();
        
        // Create debug camera if it doesn't exist
        if (!DebugCameraActor)
        {
            FActorSpawnParameters SpawnParams;
            SpawnParams.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;
            DebugCameraActor = GetWorld()->SpawnActor<ACameraActor>(GetViewTarget()->GetActorLocation(), GetViewTarget()->GetActorRotation(), SpawnParams);
        }
        else
        {
            // Position the existing debug camera at the current view target
            DebugCameraActor->SetActorLocation(GetViewTarget()->GetActorLocation());
            DebugCameraActor->SetActorRotation(GetViewTarget()->GetActorRotation());
        }
        
        // Switch to debug camera
        SetViewTarget(DebugCameraActor);
        
        // Enable flying controls
        SetInputMode(FInputModeGameOnly());
        bShowMouseCursor = false;
        
        // Allow camera movement input
        DebugCameraMovementEnabled = true;
    }
    else
    {
        // Switch back to original view target
        if (OriginalViewTarget)
        {
            SetViewTarget(OriginalViewTarget);
        }
        
        // Disable flying controls
        DebugCameraMovementEnabled = false;
    }
}
```

## Blueprint Debugging

When working with Blueprints, use these techniques:

1. **Print String nodes**: Add Print String nodes to display values during gameplay
2. **Breakpoints**: Set breakpoints on Blueprint nodes to pause execution
3. **Watch Values**: In the Blueprint debugger, watch specific variables
4. **Blueprint Debug Mode**: When in Play mode, select the Blueprint actor and click "Debug Filter" in the Details panel

## Memory Profiling

For performance optimization:

1. Open the Memory Profiler: Window → Developer Tools → Memory Profiling
2. Capture a memory snapshot during gameplay
3. Analyze object allocations and memory usage

## Crash Reporter

If your game crashes, Unreal Engine generates crash reports in:
`[ProjectFolder]/Saved/Crashes/`

These reports include call stacks and other diagnostic information.

## Integrating with ImGui Debug System

Our custom ImGui debug system can work alongside these Unreal tools:

```cpp
// Add console command execution to the ImGui debug panel
void UPlatformerDebugComponent::ExecuteConsoleCommand(const FString& Command)
{
    UWorld* World = GetWorld();
    if (World && World->GetFirstPlayerController())
    {
        World->GetFirstPlayerController()->ConsoleCommand(Command);
    }
}

// In the ImGui render function
if (ImGui::Button("Show FPS"))
{
    ExecuteConsoleCommand(TEXT("stat fps"));
}

if (ImGui::Button("Toggle Wireframe"))
{
    ExecuteConsoleCommand(TEXT("viewmode wireframe"));
}
```

## Advanced Debugging Features

For even more advanced debugging, Unreal Engine provides:

1. **Gameplay Debugger**: Press Apostrophe (') during gameplay
2. **Network Profiler**: Window → Developer Tools → Network Profiler
3. **Hierarchical Profiler**: Window → Developer Tools → Session Frontend → Profiler
4. **Animation Debug Tools**: Window → Developer Tools → Animation Insights
5. **Physics Debug Tools**: Run with '-physx' command line argument

## Debug-Only Code

Use preprocessor directives to include debug-only code:

```cpp
#if !UE_BUILD_SHIPPING
    // This code will only compile in non-shipping builds
    DrawDebugHelpers();
    EnableCheats();
#endif
```

With these tools, you'll be able to efficiently debug and optimize your 2.5D platformer! 