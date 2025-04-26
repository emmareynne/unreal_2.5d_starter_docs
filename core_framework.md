# Core Framework Implementation

## Prerequisites
- Completed steps in [project_setup.md](project_setup.md) and [version_control.md](version_control.md)
- Basic understanding of [C++ in Unreal Engine](https://docs.unrealengine.com/5.3/en-US/unreal-engine-cpp-programming-tutorials/)

## Step 1: Create Game Instance Class
1. In editor: Tools → New C++ Class
2. Parent Class: [GameInstance](https://docs.unrealengine.com/5.3/en-US/API/Runtime/Engine/Engine/UGameInstance/)
3. Name: "PlatformerGameInstance"
4. Add basic implementation:

```cpp
// PlatformerGameInstance.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/GameInstance.h"
#include "PlatformerGameInstance.generated.h"

UCLASS()
class YOURPROJECT_API UPlatformerGameInstance : public UGameInstance
{
    GENERATED_BODY()
    
public:
    // Persistent game data
    UPROPERTY(BlueprintReadWrite, Category="Game")
    int32 CollectedItems = 0;
    
    UPROPERTY(BlueprintReadWrite, Category="Game")
    FVector LastCheckpointLocation;
    
    // Simple save/load
    UFUNCTION(BlueprintCallable, Category="Save")
    void SaveCheckpoint(FVector Location);
    
    UFUNCTION(BlueprintCallable, Category="Save")
    FVector GetLastCheckpoint() const;
};
```

**SUCCESS CHECK:** Class compiles without errors

## Step 2: Create Game Mode Class
1. In editor: Tools → New C++ Class
2. Parent Class: [GameModeBase](https://docs.unrealengine.com/5.3/en-US/API/Runtime/Engine/GameFramework/AGameModeBase/)
3. Name: "PlatformerGameMode"
4. Add basic implementation:

```cpp
// PlatformerGameMode.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/GameModeBase.h"
#include "PlatformerGameMode.generated.h"

UCLASS()
class YOURPROJECT_API APlatformerGameMode : public AGameModeBase
{
    GENERATED_BODY()
    
public:
    APlatformerGameMode();
    
    // Set default classes
    virtual void InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage) override;
};
```

**SUCCESS CHECK:** Class compiles, can be selected as default GameMode

## Step 3: Create Interfaces
1. Create [Damageable Interface](https://docs.unrealengine.com/5.3/en-US/interfaces-in-unreal-engine/):

```cpp
// DamageableInterface.h
#pragma once

#include "CoreMinimal.h"
#include "UObject/Interface.h"
#include "DamageableInterface.generated.h"

UINTERFACE(MinimalAPI, BlueprintType)
class UDamageableInterface : public UInterface
{
    GENERATED_BODY()
};

class YOURPROJECT_API IDamageableInterface
{
    GENERATED_BODY()
    
public:
    // Pure virtual function for receiving damage
    UFUNCTION(BlueprintCallable, BlueprintNativeEvent)
    void ReceiveDamage(float Amount);
};
```

2. Create Interactable Interface (for collectibles, etc)

**SUCCESS CHECK:** Interfaces compile without errors

## Step 4: Create Debug Subsystem
1. In editor: Tools → New C++ Class
2. Parent Class: [WorldSubsystem](https://docs.unrealengine.com/5.3/en-US/API/Runtime/Engine/Subsystems/UWorldSubsystem/)
3. Name: "PlatformerDebugSubsystem"
4. Add implementation:

```cpp
// PlatformerDebugSubsystem.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/WorldSubsystem.h"
#include "PlatformerDebugSubsystem.generated.h"

UCLASS()
class YOURPROJECT_API UPlatformerDebugSubsystem : public UWorldSubsystem
{
    GENERATED_BODY()
    
public:
    // Debug state
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bShowCollisionDebug = false;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bShowMovementDebug = false;
    
    // Debug functions
    UFUNCTION(BlueprintCallable, Exec)
    void ToggleCollisionDebug();
    
    UFUNCTION(BlueprintCallable, Exec)
    void ToggleMovementDebug();
};
```

**SUCCESS CHECK:** Can access subsystem in Blueprint and toggle debug settings

## Step 5: Set Up Custom [Collision Channels](https://docs.unrealengine.com/5.3/en-US/collision-in-unreal-engine/)
1. Go to Edit → Project Settings
2. Under "Engine → Collision":
   - Add new channel: "Platformer_Player"
   - Add new channel: "Platformer_Enemy"
   - Add new channel: "Platformer_Collectible"
3. Configure default responses

**SUCCESS CHECK:** Custom collision channels available in object settings

## Step 6: Configure [Gameplay Ability System](https://docs.unrealengine.com/5.3/en-US/gameplay-ability-system-for-unreal-engine/)
1. Create GameplayAbilitySet asset
2. Create basic ability classes (move to combat system later)
3. Follow the [official GAS documentation](https://docs.unrealengine.com/5.3/en-US/gameplay-ability-system-for-unreal-engine/)

**SUCCESS CHECK:** GAS framework initialized without errors

## Step 7: Set Up [Enhanced Input System](https://docs.unrealengine.com/5.3/en-US/enhanced-input-in-unreal-engine/)
1. Create Input Mapping Context
2. Define Input Actions for:
   - Movement
   - Jump
   - Attack
3. Bind actions to functions in the character class

**SUCCESS CHECK:** Input system processes player inputs correctly

## Next Steps
Once your core framework is in place, proceed to implementing the character movement system in [movement_system.md](movement_system.md). 