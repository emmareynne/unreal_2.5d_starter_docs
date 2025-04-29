# Gameplay Ability System: Gameplay Cues

This guide covers implementing Gameplay Cues for your 2.5D platformer using Unreal Engine 5.5's Gameplay Ability System.

## What Are Gameplay Cues?

Gameplay Cues provide visual and audio feedback for gameplay events. They handle:

- Visual effects (particles, decals, meshes)
- Sound effects
- Camera shakes
- Animation triggers
- Environmental interactions

Unlike other parts of GAS, Gameplay Cues focus on cosmetic effects rather than gameplay mechanics.

## Core Components of Gameplay Cues

### 1. GameplayCueNotify Classes

Two main types of Gameplay Cue Notifies:

- **GameplayCueNotify_Static**: One-off effects with no persistence
- **GameplayCueNotify_Actor**: Spawned actors with state and duration

### 2. Execution Events

Gameplay Cues respond to these events:

- **OnActive**: Called when a cue is added with execution
- **WhileActive**: Called periodically for persistent cues
- **Removed**: Called when a cue is removed
- **Executed**: Called when a cue is executed (one-shot)

### 3. Tags and Routing

Cues use the `GameplayCue.` tag prefix to route effects to the proper handlers.

## Setting Up Basic Gameplay Cues

### 1. Create a Base Gameplay Cue Manager

First, create a custom Gameplay Cue Manager for your game:

```cpp
// PlatformerGameplayCueManager.h
#pragma once

#include "CoreMinimal.h"
#include "GameplayCueManager.h"
#include "PlatformerGameplayCueManager.generated.h"

/**
 * Custom Gameplay Cue Manager for the platformer.
 */
UCLASS()
class PLATFORMER_API UPlatformerGameplayCueManager : public UGameplayCueManager
{
    GENERATED_BODY()

public:
    UPlatformerGameplayCueManager();
    
    // Optional: Override methods for custom behavior
    // virtual bool ShouldAsyncLoadRuntimeObjectLibraries() const override;
};
```

```cpp
// PlatformerGameplayCueManager.cpp
#include "PlatformerGameplayCueManager.h"

UPlatformerGameplayCueManager::UPlatformerGameplayCueManager()
{
    // Configuration settings
}

/*
bool UPlatformerGameplayCueManager::ShouldAsyncLoadRuntimeObjectLibraries() const
{
    return true;
}
*/
```

### 2. Register Your Manager in GameEngine.ini

```ini
[/Script/GameplayAbilities.AbilitySystemGlobals]
GameplayCueManagerClassName="/Script/YourGame.PlatformerGameplayCueManager"
```

## Creating Static Gameplay Cues

### 1. Jump Effect Cue (One-Shot)

```cpp
// PlatformerJumpCue.h
#pragma once

#include "CoreMinimal.h"
#include "GameplayCueNotify_Static.h"
#include "PlatformerJumpCue.generated.h"

/**
 * Gameplay Cue for jump effects.
 */
UCLASS()
class PLATFORMER_API UPlatformerJumpCue : public UGameplayCueNotify_Static
{
    GENERATED_BODY()

public:
    UPlatformerJumpCue();
    
    // Override to handle the jump visual/audio effect
    virtual bool OnExecute_Implementation(AActor* MyTarget, const FGameplayCueParameters& Parameters) const override;

protected:
    // Jump particles
    UPROPERTY(EditDefaultsOnly, Category = "Effects")
    UParticleSystem* JumpParticles;
    
    // Jump sound
    UPROPERTY(EditDefaultsOnly, Category = "Effects")
    USoundBase* JumpSound;
};
```

```cpp
// PlatformerJumpCue.cpp
#include "PlatformerJumpCue.h"
#include "Kismet/GameplayStatics.h"
#include "Particles/ParticleSystemComponent.h"

UPlatformerJumpCue::UPlatformerJumpCue()
{
    // Tag that will trigger this cue
    GameplayCueTag = FGameplayTag::RequestGameplayTag(FName("GameplayCue.Ability.Jump"));
    
    // Load default assets
    static ConstructorHelpers::FObjectFinder<UParticleSystem> DefaultParticles(TEXT("/Game/Effects/Particles/P_JumpDust"));
    if (DefaultParticles.Succeeded())
    {
        JumpParticles = DefaultParticles.Object;
    }
    
    static ConstructorHelpers::FObjectFinder<USoundBase> DefaultSound(TEXT("/Game/Audio/SFX/S_Jump"));
    if (DefaultSound.Succeeded())
    {
        JumpSound = DefaultSound.Object;
    }
}

bool UPlatformerJumpCue::OnExecute_Implementation(AActor* MyTarget, const FGameplayCueParameters& Parameters) const
{
    // Only proceed if we have a valid target
    if (!MyTarget)
    {
        return false;
    }
    
    // Spawn jump particles at feet
    if (JumpParticles)
    {
        FVector ParticleLocation = MyTarget->GetActorLocation();
        ParticleLocation.Z -= MyTarget->GetSimpleCollisionHalfHeight(); // At character's feet
        
        UGameplayStatics::SpawnEmitterAtLocation(
            MyTarget->GetWorld(),
            JumpParticles,
            ParticleLocation,
            FRotator::ZeroRotator,
            true
        );
    }
    
    // Play jump sound
    if (JumpSound)
    {
        UGameplayStatics::PlaySoundAtLocation(
            MyTarget,
            JumpSound,
            MyTarget->GetActorLocation()
        );
    }
    
    return true;
}
```

## Creating Actor Gameplay Cues

### 1. Dash Effect Cue (Duration-Based)

```cpp
// PlatformerDashCue.h
#pragma once

#include "CoreMinimal.h"
#include "GameplayCueNotify_Actor.h"
#include "PlatformerDashCue.generated.h"

/**
 * Gameplay Cue for dash trail effects.
 */
UCLASS()
class PLATFORMER_API APlatformerDashCue : public AGameplayCueNotify_Actor
{
    GENERATED_BODY()

public:
    APlatformerDashCue();
    
    // Called when dash begins
    virtual bool OnActive_Implementation(AActor* MyTarget, const FGameplayCueParameters& Parameters) override;
    
    // Called each frame during dash
    virtual bool WhileActive_Implementation(AActor* MyTarget, const FGameplayCueParameters& Parameters) override;
    
    // Called when dash ends
    virtual bool OnRemove_Implementation(AActor* MyTarget, const FGameplayCueParameters& Parameters) override;

protected:
    // Trail particle system
    UPROPERTY(EditDefaultsOnly, Category = "Effects")
    UParticleSystemComponent* TrailParticles;
    
    // Dash sound
    UPROPERTY(EditDefaultsOnly, Category = "Effects")
    USoundBase* DashSound;
    
    // Optional camera shake
    UPROPERTY(EditDefaultsOnly, Category = "Effects")
    TSubclassOf<UCameraShakeBase> DashCameraShake;
    
    // Audio component for dash sound
    UPROPERTY()
    UAudioComponent* ActiveSoundComponent;
};
```

```cpp
// PlatformerDashCue.cpp
#include "PlatformerDashCue.h"
#include "Particles/ParticleSystemComponent.h"
#include "Components/AudioComponent.h"
#include "Kismet/GameplayStatics.h"
#include "Camera/CameraShakeBase.h"
#include "GameFramework/Character.h"

APlatformerDashCue::APlatformerDashCue()
{
    // Set up the actor components
    TrailParticles = CreateDefaultSubobject<UParticleSystemComponent>(TEXT("TrailParticles"));
    TrailParticles->SetupAttachment(RootComponent);
    TrailParticles->bAutoActivate = false;
    
    // Tag that will trigger this cue
    GameplayCueTag = FGameplayTag::RequestGameplayTag(FName("GameplayCue.Ability.Dash"));
    
    // Auto destroy when finished
    bAutoDestroyOnRemove = true;
    
    // Load default assets
    static ConstructorHelpers::FObjectFinder<UParticleSystem> DefaultTrail(TEXT("/Game/Effects/Particles/P_DashTrail"));
    if (DefaultTrail.Succeeded())
    {
        TrailParticles->SetTemplate(DefaultTrail.Object);
    }
    
    static ConstructorHelpers::FObjectFinder<USoundBase> DefaultSound(TEXT("/Game/Audio/SFX/S_Dash"));
    if (DefaultSound.Succeeded())
    {
        DashSound = DefaultSound.Object;
    }
}

bool APlatformerDashCue::OnActive_Implementation(AActor* MyTarget, const FGameplayCueParameters& Parameters)
{
    if (!MyTarget)
    {
        return false;
    }
    
    // Attach to target
    AttachToComponent(
        MyTarget->GetRootComponent(),
        FAttachmentTransformRules::SnapToTargetNotIncludingScale
    );
    
    // Start trail effect
    if (TrailParticles)
    {
        TrailParticles->Activate(true);
    }
    
    // Play dash sound
    if (DashSound)
    {
        ActiveSoundComponent = UGameplayStatics::SpawnSoundAttached(
            DashSound,
            MyTarget->GetRootComponent(),
            NAME_None,
            FVector::ZeroVector,
            EAttachLocation::SnapToTarget,
            true
        );
    }
    
    // Apply camera shake
    if (DashCameraShake)
    {
        APlayerController* PC = MyTarget->GetWorld()->GetFirstPlayerController();
        if (PC)
        {
            PC->ClientStartCameraShake(DashCameraShake);
        }
    }
    
    return true;
}

bool APlatformerDashCue::WhileActive_Implementation(AActor* MyTarget, const FGameplayCueParameters& Parameters)
{
    if (!MyTarget)
    {
        return false;
    }
    
    // Update effects while dashing
    // For example, you could change the trail intensity based on speed
    
    return true;
}

bool APlatformerDashCue::OnRemove_Implementation(AActor* MyTarget, const FGameplayCueParameters& Parameters)
{
    // Stop trail effect
    if (TrailParticles)
    {
        TrailParticles->Deactivate();
    }
    
    // Stop sound
    if (ActiveSoundComponent)
    {
        ActiveSoundComponent->FadeOut(0.1f, 0.0f);
    }
    
    return true;
}
```

## Triggering Gameplay Cues

### 1. From Within Abilities

```cpp
// Inside your ability's ActivateAbility method
void UPlatformerJumpAbility::ActivateAbility(...)
{
    // Existing jump code...
    
    // Trigger jump cue
    FGameplayCueParameters CueParams;
    CueParams.Location = Character->GetActorLocation();
    CueParams.Normal = FVector::UpVector;
    CueParams.Instigator = Character;
    CueParams.EffectCauser = Character;
    
    UAbilitySystemComponent* AbilitySystemComponent = GetAbilitySystemComponentFromActorInfo();
    if (AbilitySystemComponent)
    {
        AbilitySystemComponent->ExecuteGameplayCue(FGameplayTag::RequestGameplayTag("GameplayCue.Ability.Jump"), CueParams);
    }
}
```

### 2. From Gameplay Effects

```cpp
// In your GameplayEffect constructor
UPlatformerDashEffect::UPlatformerDashEffect()
{
    // Set up effect duration
    DurationPolicy = EGameplayEffectDurationType::HasDuration;
    DurationMagnitude = FScalableFloat(0.5f); // 0.5 seconds
    
    // Link gameplay cue to this effect
    GameplayCues.Add(FGameplayCueTag(FGameplayTag::RequestGameplayTag("GameplayCue.Ability.Dash")));
}
```

### 3. Manually from Game Code

```cpp
// In your character class
void APlatformerCharacter::TriggerHitReactionCue(const FVector& ImpactLocation, const FVector& ImpactNormal)
{
    if (AbilitySystemComponent)
    {
        FGameplayCueParameters CueParams;
        CueParams.Location = ImpactLocation;
        CueParams.Normal = ImpactNormal;
        CueParams.Instigator = this;
        CueParams.EffectCauser = this;
        CueParams.EffectContext = AbilitySystemComponent->MakeEffectContext();
        
        AbilitySystemComponent->ExecuteGameplayCue(FGameplayTag::RequestGameplayTag("GameplayCue.Character.HitReaction"), CueParams);
    }
}
```

## Creating Gameplay Cues in Blueprints

For designers, Blueprint Gameplay Cues are often more practical:

1. Create a Blueprint derived from `GameplayCueNotify_Static` or `GameplayCueNotify_Actor`
2. Set the Gameplay Cue Tag in the Blueprint
3. Override the appropriate event functions (OnExecute, OnActive, WhileActive, OnRemove)
4. Add visual and audio effects in the Blueprint event graph

### Example Blueprint Implementations

#### 1. Static Cue Blueprint (Double Jump)

![Blueprint Static Cue](./images/blueprint_static_cue.png)

#### 2. Actor Cue Blueprint (Power-Up Effect)

![Blueprint Actor Cue](./images/blueprint_actor_cue.png)

## Managing Gameplay Cue Assets

### 1. Asynchronous Loading

To prevent hitches, set up asynchronous loading:

```cpp
// In your custom GameplayCueManager
bool UPlatformerGameplayCueManager::ShouldAsyncLoadRuntimeObjectLibraries() const
{
    return true;
}
```

### 2. Asset Loading and Preloading

To preload commonly used Gameplay Cues:

```cpp
// In your game mode or subsystem initialization
void APlatformerGameMode::PreloadGameplayCues()
{
    if (UAbilitySystemGlobals::Get().GetGameplayCueManager())
    {
        const TArray<FString> CuesToPreload = {
            "GameplayCue.Ability.Jump",
            "GameplayCue.Ability.DoubleJump",
            "GameplayCue.Ability.Dash"
        };
        
        for (const FString& CueString : CuesToPreload)
        {
            UAbilitySystemGlobals::Get().GetGameplayCueManager()->LoadGameplayCueNotify(FGameplayTag::RequestGameplayTag(*CueString));
        }
    }
}
```

## Advanced Gameplay Cue Techniques

### 1. Passing Additional Data via Parameters

```cpp
// When executing a cue
FGameplayCueParameters CueParams;
CueParams.Location = ImpactLocation;
CueParams.Normal = ImpactNormal;
CueParams.Instigator = this;
CueParams.EffectCauser = this;

// Add custom data
FGameplayEffectContextHandle EffectContext = AbilitySystemComponent->MakeEffectContext();
FPlatformerGameplayEffectContext* PlatformerContext = StaticCast<FPlatformerGameplayEffectContext*>(EffectContext.Get());
if (PlatformerContext)
{
    PlatformerContext->SetDamageType(DamageType);
    PlatformerContext->SetImpactForce(ImpactForce);
}
CueParams.EffectContext = EffectContext;

// Execute the cue
AbilitySystemComponent->ExecuteGameplayCue(FGameplayTag::RequestGameplayTag("GameplayCue.Character.HitReaction"), CueParams);
```

### 2. Multiplayer Considerations

For networked gameplay, consider these rules:

- **Client-Predicted**: Visual effects predicted on the client
- **Server-Only**: Effects only executed on the server
- **Client-Only**: Effects only executed on the client who received the cue

```cpp
// In your ability implementation
void UPlatformerDashAbility::ActivateAbility(...)
{
    // Normal activation code...
    
    // Option 1: Execute locally (client-predicted)
    AbilitySystemComponent->ExecuteGameplayCue(FGameplayTag::RequestGameplayTag("GameplayCue.Ability.Dash"), CueParams);
    
    // Option 2: Execute with more control over replication
    EGameplayCueEvent::Type EventType = EGameplayCueEvent::OnActive;
    AbilitySystemComponent->AddGameplayCue(FGameplayTag::RequestGameplayTag("GameplayCue.Ability.Dash"), CueParams);
    
    // Later remove it
    AbilitySystemComponent->RemoveGameplayCue(FGameplayTag::RequestGameplayTag("GameplayCue.Ability.Dash"));
}
```

### 3. Creating Layered Effects

Combine multiple cues for more complex effects:

```cpp
// Dash ability with multiple cues
void UPlatformerDashAbility::ActivateAbility(...)
{
    // Trigger trail effect
    AbilitySystemComponent->ExecuteGameplayCue(FGameplayTag::RequestGameplayTag("GameplayCue.Ability.Dash.Trail"), CueParams);
    
    // Trigger screen effect for player
    AbilitySystemComponent->ExecuteGameplayCue(FGameplayTag::RequestGameplayTag("GameplayCue.Ability.Dash.Screen"), CueParams);
    
    // Add persistent dash state
    AbilitySystemComponent->AddGameplayCue(FGameplayTag::RequestGameplayTag("GameplayCue.State.Dashing"), CueParams);
}

void UPlatformerDashAbility::EndAbility(...)
{
    // Remove persistent dash state
    AbilitySystemComponent->RemoveGameplayCue(FGameplayTag::RequestGameplayTag("GameplayCue.State.Dashing"));
    
    // Add dash end effect
    AbilitySystemComponent->ExecuteGameplayCue(FGameplayTag::RequestGameplayTag("GameplayCue.Ability.Dash.End"), CueParams);
}
```

## Responding to Gameplay Cues in Other Systems

### 1. Animation System Integration

```cpp
// In your animation instance
UFUNCTION()
void UPlatformerAnimInstance::HandleGameplayCue(FGameplayTag GameplayCueTag, EGameplayCueEvent::Type EventType)
{
    // Check for specific cue tags
    if (GameplayCueTag == FGameplayTag::RequestGameplayTag("GameplayCue.Ability.Dash"))
    {
        if (EventType == EGameplayCueEvent::OnActive)
        {
            bIsDashing = true;
        }
        else if (EventType == EGameplayCueEvent::Removed)
        {
            bIsDashing = false;
        }
    }
    else if (GameplayCueTag == FGameplayTag::RequestGameplayTag("GameplayCue.Character.HitReaction"))
    {
        if (EventType == EGameplayCueEvent::Executed)
        {
            PlayHitReactionMontage();
        }
    }
}
```

### 2. UI Feedback

```cpp
// In your player HUD class
UFUNCTION()
void APlatformerHUD::HandleGameplayCue(FGameplayTag GameplayCueTag, EGameplayCueEvent::Type EventType, const FGameplayCueParameters& Parameters)
{
    // Show UI effects based on gameplay cues
    if (GameplayCueTag == FGameplayTag::RequestGameplayTag("GameplayCue.Player.LowHealth"))
    {
        if (EventType == EGameplayCueEvent::OnActive)
        {
            ShowLowHealthWarning();
        }
        else if (EventType == EGameplayCueEvent::Removed)
        {
            HideLowHealthWarning();
        }
    }
    else if (GameplayCueTag == FGameplayTag::RequestGameplayTag("GameplayCue.Ability.Dash"))
    {
        if (EventType == EGameplayCueEvent::OnActive)
        {
            ShowDashFeedback();
        }
    }
}
```

## Testing Gameplay Cues

To test your Gameplay Cues:

1. Create a test map with your character
2. Add debug commands to trigger cues manually
3. Visualize gameplay tags to confirm they're being applied
4. Set up a test sequence that triggers multiple cues

```cpp
// Debug command for testing cues
UFUNCTION(Exec)
void APlatformerCharacter::TestGameplayCue(FString CueTag)
{
    if (AbilitySystemComponent)
    {
        FGameplayCueParameters CueParams;
        CueParams.Location = GetActorLocation();
        CueParams.Normal = FVector::UpVector;
        CueParams.Instigator = this;
        CueParams.EffectCauser = this;
        
        AbilitySystemComponent->ExecuteGameplayCue(FGameplayTag::RequestGameplayTag(*CueTag), CueParams);
    }
}
```

## Common Issues and Solutions

- **Cue Not Firing**: Verify tag names match exactly between execution and notification
- **Cue Firing Multiple Times**: Check for duplicate execution calls
- **Particles Not Showing**: Verify world context and spawn location
- **Persisting Too Long**: Make sure cues are properly removed in ability end/cancel

## Best Practices

- **Tag Hierarchy**: Use a clear tag hierarchy (e.g., `GameplayCue.Category.Specific`)
- **Performance**: Keep effects optimized and use LODs for particles
- **Modularity**: Design cues to work with multiple abilities
- **VFX Scaling**: Scale effect intensity based on ability power/context
- **Audio Mixing**: Use sound mix snapshots for better audio integration
- **Configuration**: Create data assets to configure cue behaviors without code changes

## Next Steps

With Gameplay Cues configured, you can now move on to:

1. [Integrating with Other Systems](./6_6_gas_integration.md) - Connect GAS with UI, animation, etc.
2. [Character Progression](./6_7_character_progression.md) - Implement leveling, unlocking abilities, etc.

## Resources

- [Unreal Engine Gameplay Cue Documentation](https://docs.unrealengine.com/5.5/en-US/gameplay-cues-for-unreal-engine/)
- [Community GAS Documentation on Cues](https://github.com/tranek/GASDocumentation/blob/master/GameplayCues.md)
- [VFX Best Practices in UE5](https://dev.epicgames.com/documentation/en-us/unreal-engine/vfx-in-unreal-engine) 