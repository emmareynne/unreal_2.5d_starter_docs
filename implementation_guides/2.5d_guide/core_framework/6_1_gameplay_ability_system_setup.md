# Gameplay Ability System: Project Setup

This guide walks you through the initial setup process for integrating the Gameplay Ability System into your 2.5D platformer project in Unreal Engine 5.5.

## Adding Required Plugins

First, ensure that your project has the necessary plugins enabled:

1. Open your project
2. Navigate to **Edit → Plugins**
3. Enable the following plugins (if not already enabled):
   - **Gameplay Abilities**: The core GAS plugin
   - **GameplayTags**: Required for the tag system
   - **ModularGameplay**: Helpful for ability-based gameplay

4. Restart the editor when prompted

## Configuring Build Dependencies

Next, update your project's build files to include the necessary modules:

1. Open your project's `.Build.cs` file located at `Source/YourProject/YourProject.Build.cs`
2. Add the required module dependencies:

```csharp
// In YourProject.Build.cs constructor
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
```

3. Save the file and close it
4. Right-click on your `.uproject` file and select **Generate Visual Studio project files**
5. Rebuild your project in your IDE (Visual Studio/Rider)

## Component Types for GAS Implementation

Implementing GAS requires adding specific components to your actors. Here's what to use for different actor types:

### 1. For Characters and Pawns

Characters should implement both the `IAbilitySystemInterface` and `IGameplayTagAssetInterface`.

**Base Components to Add:**
- **Character Base Class**: Derive from `ACharacter` 
- **Ability System Component**: Add `UAbilitySystemComponent` (or your custom derived version)
- **Attribute Sets**: Add attribute sets as components or owner UObjects

```cpp
// PlatformerCharacter.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Character.h"
#include "AbilitySystemInterface.h"
#include "GameplayTagAssetInterface.h"
#include "PlatformerCharacter.generated.h"

class UPlatformerAbilitySystemComponent;
class UPlatformerAttributeSet;

UCLASS()
class PLATFORMER_API APlatformerCharacter : public ACharacter, 
                                           public IAbilitySystemInterface, 
                                           public IGameplayTagAssetInterface
{
    GENERATED_BODY()

public:
    APlatformerCharacter(const FObjectInitializer& ObjectInitializer);

    // IAbilitySystemInterface
    virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;
    
    // IGameplayTagAssetInterface
    virtual void GetOwnedGameplayTags(FGameplayTagContainer& TagContainer) const override;

protected:
    // The ability system component
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Abilities")
    UPlatformerAbilitySystemComponent* AbilitySystemComponent;
    
    // The main attribute set
    UPROPERTY()
    UPlatformerAttributeSet* AttributeSet;
    
    virtual void BeginPlay() override;
    virtual void PossessedBy(AController* NewController) override;
    virtual void OnRep_PlayerState() override;
    
    // Initialize the ability system component
    virtual void InitializeAbilitySystem();
};
```

### 2. For Player States (Recommended for Multiplayer Games)

In multiplayer games, it's often better to add the AbilitySystemComponent to the PlayerState:

```cpp
// PlatformerPlayerState.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/PlayerState.h"
#include "AbilitySystemInterface.h"
#include "PlatformerPlayerState.generated.h"

class UPlatformerAbilitySystemComponent;
class UPlatformerAttributeSet;

UCLASS()
class PLATFORMER_API APlatformerPlayerState : public APlayerState, public IAbilitySystemInterface
{
    GENERATED_BODY()

public:
    APlatformerPlayerState();
    
    // IAbilitySystemInterface
    virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;
    
    // Get the attribute set
    UPlatformerAttributeSet* GetAttributeSet() const;

protected:
    // The ability system component
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Abilities")
    UPlatformerAbilitySystemComponent* AbilitySystemComponent;
    
    // The main attribute set
    UPROPERTY()
    UPlatformerAttributeSet* AttributeSet;
};
```

### 3. For Non-Character Actors (Items, Projectiles, etc.)

For actors that aren't characters but need ability functionality:

```cpp
// PlatformerGameplayItem.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "AbilitySystemInterface.h"
#include "PlatformerGameplayItem.generated.h"

class UPlatformerAbilitySystemComponent;

UCLASS()
class PLATFORMER_API APlatformerGameplayItem : public AActor, public IAbilitySystemInterface
{
    GENERATED_BODY()

public:
    APlatformerGameplayItem();
    
    // IAbilitySystemInterface
    virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;

protected:
    // The ability system component
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Abilities")
    UPlatformerAbilitySystemComponent* AbilitySystemComponent;
    
    virtual void BeginPlay() override;
};
```

## Where to Put the AbilitySystemComponent

For a 2.5D platformer, here are the recommended setups:

1. **Single Player Game:**
   - Add AbilitySystemComponent and AttributeSets directly to the Character class
   - Initialize in BeginPlay or PossessedBy

2. **Multiplayer Game:**
   - Add AbilitySystemComponent and AttributeSets to the PlayerState
   - Character class retrieves ASC from its PlayerState
   - AI characters keep components on themselves

3. **Mixed Approach:**
   - Player characters get ASC from PlayerState
   - AI and non-player characters have components directly added

## Setting Up Core GAS Classes

For a well-structured implementation, create these foundational classes:

### 1. Create a Custom Ability System Component

```cpp
// PlatformerAbilitySystemComponent.h
#pragma once

#include "CoreMinimal.h"
#include "AbilitySystemComponent.h"
#include "PlatformerAbilitySystemComponent.generated.h"

/**
 * Custom Ability System Component for 2.5D platformer.
 * Handles ability activation, effect application, and attribute management.
 */
UCLASS()
class PLATFORMER_API UPlatformerAbilitySystemComponent : public UAbilitySystemComponent
{
    GENERATED_BODY()

public:
    UPlatformerAbilitySystemComponent();

    // Optional: Add custom functions for your platformer abilities
    
    // Example: Helper function for movement abilities
    UFUNCTION(BlueprintCallable, Category = "Platformer|Abilities")
    bool TryActivateMovementAbility(TSubclassOf<UGameplayAbility> AbilityClass);
};
```

```cpp
// PlatformerAbilitySystemComponent.cpp
#include "PlatformerAbilitySystemComponent.h"

UPlatformerAbilitySystemComponent::UPlatformerAbilitySystemComponent()
{
    // Set default values if needed
}

bool UPlatformerAbilitySystemComponent::TryActivateMovementAbility(TSubclassOf<UGameplayAbility> AbilityClass)
{
    // Implementation for activating movement abilities
    // This would include custom logic for platformer-specific abilities
    return TryActivateAbilityByClass(AbilityClass);
}
```

### 2. Create a Base Ability Class

```cpp
// PlatformerGameplayAbility.h
#pragma once

#include "CoreMinimal.h"
#include "Abilities/GameplayAbility.h"
#include "PlatformerGameplayAbility.generated.h"

/**
 * Base class for all gameplay abilities in the platformer game.
 * Provides common functionality for 2.5D movement and actions.
 */
UCLASS()
class PLATFORMER_API UPlatformerGameplayAbility : public UGameplayAbility
{
    GENERATED_BODY()

public:
    UPlatformerGameplayAbility();

    // Override to add platformer-specific checks
    virtual bool CanActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayTagContainer* SourceTags = nullptr, const FGameplayTagContainer* TargetTags = nullptr, OUT FGameplayTagContainer* OptionalRelevantTags = nullptr) const override;

    // Input ID for ability activation
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Ability")
    int32 AbilityInputID = 0;
};
```

```cpp
// PlatformerGameplayAbility.cpp
#include "PlatformerGameplayAbility.h"

UPlatformerGameplayAbility::UPlatformerGameplayAbility()
{
    // Default ability properties
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::LocalPredicted;
}

bool UPlatformerGameplayAbility::CanActivateAbility(const FGameplayAbilitySpecHandle Handle, 
                                                  const FGameplayAbilityActorInfo* ActorInfo, 
                                                  const FGameplayTagContainer* SourceTags, 
                                                  const FGameplayTagContainer* TargetTags, 
                                                  OUT FGameplayTagContainer* OptionalRelevantTags) const
{
    // First check the parent class (important)
    if (!Super::CanActivateAbility(Handle, ActorInfo, SourceTags, TargetTags, OptionalRelevantTags))
    {
        return false;
    }

    // Add any platformer-specific activation requirements
    // For example, checking if character is on ground, in air, etc.
    
    return true;
}
```

### 3. Create GameplayTags Configuration

Define your gameplay tags structure in the `DefaultGameplayTags.ini` file:

1. Navigate to your project's `Config` folder
2. Create or open `DefaultGameplayTags.ini`
3. Add a basic tag structure for your platformer:

```ini
[/Script/GameplayTags.GameplayTagsSettings]
ImportTagsFromConfig=True
WarnOnInvalidTags=True
ClearInvalidTags=False
AllowEditorTagUnloading=True
AllowGameTagUnloading=False
FastReplication=False
InvalidTagCharacters="\"\',"
NumBitsForContainerSize=6
NetIndexFirstBitSegment=16

; Platformer ability tags
+GameplayTagList=(Tag="Ability.Movement",DevComment="Base tag for all movement abilities")
+GameplayTagList=(Tag="Ability.Movement.Jump",DevComment="Basic jump ability")
+GameplayTagList=(Tag="Ability.Movement.DoubleJump",DevComment="Double jump ability")
+GameplayTagList=(Tag="Ability.Movement.Dash",DevComment="Dash ability")
+GameplayTagList=(Tag="Ability.Movement.WallJump",DevComment="Wall jump ability")

; Character state tags
+GameplayTagList=(Tag="State.Grounded",DevComment="Character is on the ground")
+GameplayTagList=(Tag="State.InAir",DevComment="Character is in the air")
+GameplayTagList=(Tag="State.OnWall",DevComment="Character is on a wall")
+GameplayTagList=(Tag="State.Dashing",DevComment="Character is dashing")
+GameplayTagList=(Tag="State.Attacking",DevComment="Character is attacking")
+GameplayTagList=(Tag="State.Stunned",DevComment="Character is stunned")

; Input tags
+GameplayTagList=(Tag="Input.Jump",DevComment="Jump input")
+GameplayTagList=(Tag="Input.Attack",DevComment="Attack input")
+GameplayTagList=(Tag="Input.Dash",DevComment="Dash input")
+GameplayTagList=(Tag="Input.Special",DevComment="Special ability input")
```

## Initializing the System

To initialize the system properly, you'll need to set up some global defaults. Create or modify your game module's initialization code:

```cpp
// YourProject.cpp
#include "YourProject.h"
#include "Modules/ModuleManager.h"
#include "AbilitySystemGlobals.h"

class FYourProjectModule : public FDefaultGameModuleImpl
{
    virtual void StartupModule() override
    {
        Super::StartupModule();
    }

    virtual void PostLoadCallback() override
    {
        // Initialize AbilitySystemGlobals
        UAbilitySystemGlobals::Get().InitGlobalData();
    }
};

IMPLEMENT_PRIMARY_GAME_MODULE(FYourProjectModule, YourProject, "YourProject");
```

## Next Steps

Now that your project is configured for the Gameplay Ability System, you can proceed to implementing:

1. [Ability System Components](./6_2_ability_system_components.md) - Integrating the ability system with your characters
2. [Attribute Sets](./6_3_attribute_sets.md) - Setting up character attributes like health, energy, etc.

## Common Issues

- **Compile Errors**: If you encounter compile errors about missing headers, double-check your module dependencies.
- **Linker Errors**: Make sure you've properly rebuilt the project after adding new modules.
- **Plugin Load Failures**: Verify plugin compatibility with your Unreal Engine version.
- **Tag Errors**: Ensure your gameplay tag hierarchy is valid and doesn't contain invalid characters.

## Best Practices

- Keep your tag hierarchy clean and organized
- Use a consistent naming convention for abilities and effects
- Create base classes that encapsulate common functionality
- Document tag usage and ability behaviors
- Start with minimal implementation and expand incrementally 