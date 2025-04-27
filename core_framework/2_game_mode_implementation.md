# Game Mode Implementation for 2.5D Platformer

This guide provides a detailed, step-by-step approach to implementing a custom GameMode for your Platformer project in Unreal Engine 5.5. The GameMode defines the rules of your game, handles player spawning and respawning, and manages level-specific game logic.

## What is a GameMode?

A GameMode defines the rules and core gameplay elements of your game. It determines:
- How players spawn into the world
- Win/loss conditions
- Scoring and game states
- Player start locations
- Default classes for players, HUDs, etc.

There are two main classes in Unreal Engine for implementing game modes:
- **AGameModeBase**: The core functionality for a game mode with minimal features
- **AGameMode**: Extends GameModeBase with additional functionality for match-based games

For a 2.5D platformer with levels, checkpoints, and potentially multiplayer features, **AGameMode** is recommended as it provides match state functionality that can be useful for managing level progression.

## Prerequisites

- Unreal Engine 5.5 installed
- Platformer project created with the following plugins:
  - Enhanced Input
  - Gameplay Abilities
- Basic understanding of C++ (though no prior Unreal experience is assumed)
- [PlatformerGameInstance](./1_game_instance_implementation.md) implemented

## Implementation Steps

### Step 1: Create the GameMode C++ Class

1. Open your Platformer project in Unreal Editor
2. From the main menu, select **Tools → New C++ Class**
3. In the dialog, click **Show All Classes**
4. In the search field, type "GameMode"
5. Select **GameMode** from the results
6. Click **Next**
7. Name your class `PlatformerGameMode`
8. Click **Create Class**

This will create a new C++ class that inherits from AGameMode in your project's Source directory.

### Step 2: Set Up Core Header File

Open the generated header file (`PlatformerGameMode.h`) and update it with the following code:

```cpp
// PlatformerGameMode.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/GameMode.h"
#include "PlatformerGameMode.generated.h"

// Forward declare classes we'll implement later
// This allows us to reference them without including their headers
class APlatformerCharacter;

/**
 * Game Mode for Platformer
 * Handles spawning, level rules, and game states
 */
UCLASS()
class PLATFORMER_API APlatformerGameMode : public AGameMode
{
    GENERATED_BODY()
    
public:
    // Constructor
    APlatformerGameMode();
    
    // Virtual functions from GameMode
    virtual void InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage) override;
    virtual void InitGameState() override;
    virtual void HandleStartingNewPlayer_Implementation(APlayerController* NewPlayer) override;
    virtual AActor* FindPlayerStart_Implementation(AController* Player, const FString& IncomingName) override;
    
    // Phase 1: Core functionality without character-specific logic
    
    // Respawn functionality (character-agnostic version)
    UFUNCTION(BlueprintCallable, Category="Game")
    void RespawnPlayer(APlayerController* PlayerController);
    
    // Phase 2: These functions will be fully implemented after character creation
    
    // Level completion tracking (will be implemented later)
    UFUNCTION(BlueprintCallable, Category="Game", meta=(DevelopmentOnly))
    void LevelCompleted(AActor* CompletingActor);
    
protected:
    // Time between death and respawn
    UPROPERTY(EditDefaultsOnly, Category="Game")
    float RespawnDelay = 2.0f;
    
    // Whether to use checkpoints for respawning
    UPROPERTY(EditDefaultsOnly, Category="Game")
    bool bUseCheckpoints = true;
    
    // Note: We don't need to declare DefaultPawnClass here because it's already
    // declared in AGameMode (our parent class). We'll set it in the constructor instead.
    
private:
    // Internal respawn handling
    void InternalRespawnPlayer(APlayerController* PlayerController);
};
```

### Step 3: Implement the GameMode Class

Now open the implementation file (`PlatformerGameMode.cpp`) and update it with the following code:

```cpp
// PlatformerGameMode.cpp
#include "PlatformerGameMode.h"
#include "GameFramework/PlayerStart.h"
#include "GameFramework/GameStateBase.h"
#include "Kismet/GameplayStatics.h"
#include "TimerManager.h"
#include "GameFramework/Pawn.h"

APlatformerGameMode::APlatformerGameMode()
{
    // Set default values
    bStartPlayersAsSpectators = false;
    bDelayedStart = false;
    
    // We're using the DefaultPawnClass inherited from AGameMode
    // Setting this here follows proper inheritance practice in Unreal Engine
    static ConstructorHelpers::FClassFinder<APawn> DefaultPawnClassFinder(TEXT("/Game/ThirdPerson/Blueprints/BP_ThirdPersonCharacter"));
    if (DefaultPawnClassFinder.Class != nullptr)
    {
        DefaultPawnClass = DefaultPawnClassFinder.Class;
    }
    else
    {
        // Fall back to engine default if not found
        UE_LOG(LogTemp, Warning, TEXT("Default pawn class not found, using engine default"));
    }
    
    // Log initialization
    UE_LOG(LogTemp, Log, TEXT("PlatformerGameMode Initialized"));
}

void APlatformerGameMode::InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage)
{
    Super::InitGame(MapName, Options, ErrorMessage);
    
    UE_LOG(LogTemp, Log, TEXT("PlatformerGameMode InitGame: %s"), *MapName);
}

void APlatformerGameMode::InitGameState()
{
    Super::InitGameState();
    
    UE_LOG(LogTemp, Log, TEXT("PlatformerGameMode InitGameState"));
}

void APlatformerGameMode::HandleStartingNewPlayer_Implementation(APlayerController* NewPlayer)
{
    // Call the parent implementation first
    Super::HandleStartingNewPlayer_Implementation(NewPlayer);
    
    UE_LOG(LogTemp, Log, TEXT("PlatformerGameMode HandleStartingNewPlayer"));
    
    // Custom logic for starting a new player will be added later
    // when abilities and input mappings are implemented
}

AActor* APlatformerGameMode::FindPlayerStart_Implementation(AController* Player, const FString& IncomingName)
{
    // Get the GameInstance to check if we have a saved checkpoint
    UGameInstance* GameInstance = GetGameInstance();
    if (GameInstance && bUseCheckpoints)
    {
        // The checkpoint functionality will be implemented after integrating
        // with PlatformerGameInstance in a future step
        
        // At that point, we'll uncommment and complete this code:
        /*
        #include "PlatformerGameInstance.h"
        UPlatformerGameInstance* PlatformerGI = Cast<UPlatformerGameInstance>(GameInstance);
        if (PlatformerGI)
        {
            FVector CheckpointLocation = PlatformerGI->GetLastCheckpoint();
            if (CheckpointLocation != FVector::ZeroVector)
            {
                // Will implement checkpoint spawning logic here
            }
        }
        */
    }
    
    // Fall back to default behavior
    return Super::FindPlayerStart_Implementation(Player, IncomingName);
}

void APlatformerGameMode::LevelCompleted(AActor* CompletingActor)
{
    // This is a placeholder implementation - will be expanded when character class is created
    UE_LOG(LogTemp, Log, TEXT("PlatformerGameMode LevelCompleted by %s"), 
        CompletingActor ? *CompletingActor->GetName() : TEXT("Unknown Actor"));
    
    // In the future implementation, we'll handle:
    // - Saving progress through GameInstance
    // - Showing UI for level completion
    // - Loading the next level
}

void APlatformerGameMode::RespawnPlayer(APlayerController* PlayerController)
{
    if (!PlayerController)
    {
        UE_LOG(LogTemp, Warning, TEXT("PlatformerGameMode RespawnPlayer: Invalid PlayerController"));
        return;
    }
    
    // Add delay before respawn
    if (RespawnDelay > 0.0f)
    {
        FTimerHandle RespawnTimerHandle;
        GetWorldTimerManager().SetTimer(
            RespawnTimerHandle, 
            FTimerDelegate::CreateUObject(this, &APlatformerGameMode::InternalRespawnPlayer, PlayerController),
            RespawnDelay, 
            false
        );
        
        UE_LOG(LogTemp, Log, TEXT("PlatformerGameMode: Respawning player in %f seconds"), RespawnDelay);
    }
    else
    {
        InternalRespawnPlayer(PlayerController);
    }
}

void APlatformerGameMode::InternalRespawnPlayer(APlayerController* PlayerController)
{
    if (!PlayerController)
    {
        return;
    }

    // Destroy the old pawn
    if (PlayerController->GetPawn())
    {
        PlayerController->GetPawn()->Destroy();
    }
    
    // Respawn the player
    RestartPlayer(PlayerController);
    
    UE_LOG(LogTemp, Log, TEXT("PlatformerGameMode: Player respawned"));
}
```

### Step 4: Set Up Project Settings

Now that you've created your custom GameMode class, you need to tell the engine to use it.

1. In the Unreal Editor, go to **Edit → Project Settings**
2. In the left panel, navigate to **Project → Maps & Modes**
3. Find the **Default Modes** section
4. Set **Default GameMode** to your `PlatformerGameMode`
5. Click **Save**

### Step 5: Create a Blueprint Extension of Your GameMode

While the C++ class provides the foundation, creating a Blueprint child class allows for easier tweaking of game rules without recompiling code.

1. In the Content Browser, navigate to a suitable location (e.g., `Content/Core/Modes`)
2. Right-click and select **Blueprint Class**
3. In the **Pick Parent Class** dialog, search for "PlatformerGameMode"
4. Select your C++ class and click **Select**
5. Name it `BP_PlatformerGameMode`
6. Open the Blueprint to customize and extend your GameMode

In the Blueprint editor, you can set default values for properties like:
- Respawn delay
- Whether to use checkpoints

### Step 6: Testing Your GameMode

To test your GameMode without depending on other components:

1. Create a basic level with some platforms and a PlayerStart actor
2. Set the GameMode override in the World Settings to BP_PlatformerGameMode
3. Play the level in the editor - you should spawn with the default pawn
4. Try respawning the player using the console command:
   ```
   RestartPlayer 0
   ```
   This should trigger your respawn functionality

### Step 7: Preparing for Gameplay Ability System Integration

For a complete 2.5D platformer, you'll likely want to use the Gameplay Ability System for character abilities. For now, let's update your project's Build.cs file to include the required modules for later:

```csharp
// Platformer.Build.cs
PublicDependencyModuleNames.AddRange(new string[] { 
    "Core", 
    "CoreUObject", 
    "Engine", 
    "InputCore",
    "GameplayAbilities",
    "GameplayTags",
    "GameplayTasks"
});
```

The actual integration will be completed after we've created the character class.

## GameMode Match States in Detail

One of the advantages of using `AGameMode` over `AGameModeBase` is access to match states. Here's how you can leverage them in your 2.5D platformer:

1. **EnteringMap** - Initial state when a level is loaded
   - Use for initialization and setup
   - Show loading screens

2. **WaitingToStart** - Waiting for players or conditions before gameplay begins
   - Display level introduction
   - For multiplayer: wait for all players to join

3. **InProgress** - Main gameplay occurs
   - Enable player controls
   - Start enemy spawning or movement
   - Activate hazards and gameplay elements

4. **WaitingPostMatch** - Level completed, showing results
   - Display score or time
   - Show level completion animations
   - Prepare for next level

5. **LeavingMap** - Transitioning between levels
   - Save progress
   - Clean up resources

You can override `OnMatchStateSet` to add custom behavior when the match state changes:

```cpp
// Add to PlatformerGameMode.h
virtual void OnMatchStateSet() override;

// Add to PlatformerGameMode.cpp
void APlatformerGameMode::OnMatchStateSet()
{
    Super::OnMatchStateSet();
    
    // Log the current match state
    UE_LOG(LogTemp, Log, TEXT("PlatformerGameMode: Match state changed to %s"), *GetMatchState().ToString());
    
    // Handle different match states
    if (GetMatchState() == MatchState::InProgress)
    {
        // Start gameplay elements
        // For example: Start enemy spawning, activate hazards, etc.
    }
    else if (GetMatchState() == MatchState::WaitingPostMatch)
    {
        // Level completed logic
        // For example: Save progress, prepare to load next level
    }
}
```

## Best Practices

1. **State-Based Design**: Use match states to clearly define when certain gameplay elements should be active.

2. **Single Responsibility**: Keep your GameMode focused on game rules and player management. Use GameState for shared game data and PlayerState for player-specific data.

3. **Blueprint Extension**: Create your core functionality in C++ for performance, but extend with Blueprints for level-specific variations.

4. **Error Handling**: Always check for null pointers and invalid states, especially when working with player controllers or pawns.

5. **Documentation**: Document your GameMode's functionality and requirements to make it easier for team members to understand.

6. **Performance Considerations**: Be mindful of operations in frequently called functions like Tick().

7. **Multiplayer Awareness**: Even for a single-player game, design with potential multiplayer expansion in mind.

## Success Criteria

Your GameMode implementation is successful when:

- ✅ Players spawn correctly in the level
- ✅ Checkpoints work properly for respawning
- ✅ Character abilities are granted appropriately
- ✅ Level-specific rules function as expected
- ✅ Match state transitions are handled smoothly

## Next Steps

After completing your GameMode implementation, move on to:

1. Implementing your [Character Movement System](./movement_system.md) with 2.5D constraints
2. Setting up your [Combat System](./combat_system.md) using Gameplay Abilities
3. Creating the [UI System](./ui_system.md) for player feedback

## Troubleshooting

**Problem**: Players spawn at the wrong location
**Solution**: Check your FindPlayerStart implementation and ensure PlayerStart actors are placed correctly in the level

**Problem**: Abilities aren't granted to the player
**Solution**: Verify that your character implements the AbilitySystemInterface and the ability classes are set correctly

**Problem**: Level-specific rules are not applied
**Solution**: Ensure you've set the correct GameMode override in the World Settings for each level

**Problem**: Match states aren't transitioning as expected
**Solution**: Check your match state logic and use logs to debug state transitions

## Resources

- [Unreal Engine GameMode Documentation](https://docs.unrealengine.com/5.5/en-US/API/Runtime/Engine/GameFramework/AGameMode/)
- [Gameplay Ability System Documentation](https://docs.unrealengine.com/5.5/en-US/gameplay-ability-system-for-unreal-engine/)
- [Enhanced Input System Guide](https://docs.unrealengine.com/5.5/en-US/enhanced-input-in-unreal-engine/) 