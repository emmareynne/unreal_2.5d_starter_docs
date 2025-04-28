# Gameplay Ability System: Integration (Part 2)

This guide continues our coverage of integrating the Gameplay Ability System with other game systems for your 2.5D platformer.

## Input System Integration

### 1. Enhanced Input Integration

For projects using Enhanced Input, connect it with GAS:

```cpp
// PlatformerEnhancedInputComponent.h
#pragma once

#include "CoreMinimal.h"
#include "EnhancedInputComponent.h"
#include "GameplayTagContainer.h"
#include "PlatformerEnhancedInputComponent.generated.h"

class UAbilitySystemComponent;

/**
 * Enhanced input component with GAS integration.
 */
UCLASS()
class PLATFORMER_API UPlatformerEnhancedInputComponent : public UEnhancedInputComponent
{
    GENERATED_BODY()

public:
    // Bind an action to an ability input
    template<class UserClass>
    void BindAbilityActions(const UInputAction* InputAction, ETriggerEvent TriggerEvent, UserClass* Object,
        UAbilitySystemComponent* AbilitySystemComponent, int32 InputID);
};

// Template implementation
template<class UserClass>
void UPlatformerEnhancedInputComponent::BindAbilityActions(const UInputAction* InputAction, ETriggerEvent TriggerEvent, 
    UserClass* Object, UAbilitySystemComponent* AbilitySystemComponent, int32 InputID)
{
    // Pressed event
    BindAction(InputAction, TriggerEvent, Object, [AbilitySystemComponent, InputID](const FInputActionValue& Value) {
        AbilitySystemComponent->AbilityLocalInputPressed(InputID);
    });
    
    // Released event (only bind to Released and Completed)
    if (TriggerEvent == ETriggerEvent::Started || TriggerEvent == ETriggerEvent::Triggered)
    {
        BindAction(InputAction, ETriggerEvent::Completed, Object, [AbilitySystemComponent, InputID](const FInputActionValue& Value) {
            AbilitySystemComponent->AbilityLocalInputReleased(InputID);
        });
        
        BindAction(InputAction, ETriggerEvent::Canceled, Object, [AbilitySystemComponent, InputID](const FInputActionValue& Value) {
            AbilitySystemComponent->AbilityLocalInputReleased(InputID);
        });
    }
}
```

### 2. Input Component Setup

Set up the input component in your character class:

```cpp
// PlatformerCharacter.cpp
void APlatformerCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    Super::SetupPlayerInputComponent(PlayerInputComponent);
    
    // Cast to our enhanced input component
    UPlatformerEnhancedInputComponent* EnhancedInputComponent = Cast<UPlatformerEnhancedInputComponent>(PlayerInputComponent);
    if (!EnhancedInputComponent)
    {
        return;
    }
    
    // Get local player subsystem
    APlayerController* PC = Cast<APlayerController>(GetController());
    if (!PC)
    {
        return;
    }
    
    UEnhancedInputLocalPlayerSubsystem* Subsystem = ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(PC->GetLocalPlayer());
    if (!Subsystem)
    {
        return;
    }
    
    // Clear any existing mappings and add our mapping context
    Subsystem->ClearMappingContext();
    Subsystem->AddMappingContext(DefaultMappingContext, 0);
    
    // Bind ability inputs
    if (AbilitySystemComponent)
    {
        // Jump
        EnhancedInputComponent->BindAbilityActions(
            JumpAction, 
            ETriggerEvent::Started, 
            this, 
            AbilitySystemComponent, 
            static_cast<int32>(EPlatformerAbilityInputID::Jump)
        );
        
        // Dash
        EnhancedInputComponent->BindAbilityActions(
            DashAction, 
            ETriggerEvent::Started, 
            this, 
            AbilitySystemComponent, 
            static_cast<int32>(EPlatformerAbilityInputID::Dash)
        );
        
        // Attack
        EnhancedInputComponent->BindAbilityActions(
            AttackAction, 
            ETriggerEvent::Started, 
            this, 
            AbilitySystemComponent, 
            static_cast<int32>(EPlatformerAbilityInputID::Attack)
        );
        
        // Special ability
        EnhancedInputComponent->BindAbilityActions(
            SpecialAction, 
            ETriggerEvent::Started, 
            this, 
            AbilitySystemComponent, 
            static_cast<int32>(EPlatformerAbilityInputID::SpecialAbility)
        );
    }
    
    // Bind movement inputs (not GAS related)
    EnhancedInputComponent->BindAction(MoveAction, ETriggerEvent::Triggered, this, &APlatformerCharacter::Move);
    EnhancedInputComponent->BindAction(LookAction, ETriggerEvent::Triggered, this, &APlatformerCharacter::Look);
}
```

### 3. Input-Driven Abilities

Structure abilities to be triggered by input:

```cpp
// PlatformerGameplayAbility.h
// Add to your base ability class:

// Input ID for this ability
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Input")
EPlatformerAbilityInputID AbilityInputID = EPlatformerAbilityInputID::None;

// Determines if the ability activates on input press or release
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Input")
EAbilityActivationMode ActivationMode = EAbilityActivationMode::OnInputPressed;

// Whether ability triggers on input or not
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Input")
bool bActivateOnInput = true;
```

## AI Integration

### 1. AI Controller with GAS Support

Create an AI controller that can use abilities:

```cpp
// PlatformerAIController.h
#pragma once

#include "CoreMinimal.h"
#include "AIController.h"
#include "AbilitySystemInterface.h"
#include "PlatformerAIController.generated.h"

class UAbilitySystemComponent;
class UGameplayAbility;

/**
 * AI controller with GAS integration.
 */
UCLASS()
class PLATFORMER_API APlatformerAIController : public AAIController
{
    GENERATED_BODY()

public:
    APlatformerAIController();
    
    // Override possession to set up ability system
    virtual void OnPossess(APawn* InPawn) override;
    
    // Get ASC from possessed character
    UAbilitySystemComponent* GetAbilitySystemComponentFromPawn() const;
    
    // Try to activate an ability by tag
    UFUNCTION(BlueprintCallable, Category = "AI|Abilities")
    bool TryActivateAbilityByTag(const FGameplayTagContainer& GameplayTagContainer);
    
    // Try to activate an ability by class
    UFUNCTION(BlueprintCallable, Category = "AI|Abilities")
    bool TryActivateAbilityByClass(TSubclassOf<UGameplayAbility> AbilityClass);
};
```

```cpp
// PlatformerAIController.cpp
#include "PlatformerAIController.h"
#include "AbilitySystemInterface.h"
#include "AbilitySystemComponent.h"
#include "GameplayAbilities/Public/Abilities/GameplayAbility.h"

APlatformerAIController::APlatformerAIController()
{
    // Setup defaults
}

void APlatformerAIController::OnPossess(APawn* InPawn)
{
    Super::OnPossess(InPawn);
    
    // Set up behavior tree
    if (BehaviorTree)
    {
        RunBehaviorTree(BehaviorTree);
    }
}

UAbilitySystemComponent* APlatformerAIController::GetAbilitySystemComponentFromPawn() const
{
    if (!GetPawn())
    {
        return nullptr;
    }
    
    IAbilitySystemInterface* AbilitySystemInterface = Cast<IAbilitySystemInterface>(GetPawn());
    if (!AbilitySystemInterface)
    {
        return nullptr;
    }
    
    return AbilitySystemInterface->GetAbilitySystemComponent();
}

bool APlatformerAIController::TryActivateAbilityByTag(const FGameplayTagContainer& GameplayTagContainer)
{
    UAbilitySystemComponent* ASC = GetAbilitySystemComponentFromPawn();
    if (!ASC)
    {
        return false;
    }
    
    return ASC->TryActivateAbilitiesByTag(GameplayTagContainer);
}

bool APlatformerAIController::TryActivateAbilityByClass(TSubclassOf<UGameplayAbility> AbilityClass)
{
    UAbilitySystemComponent* ASC = GetAbilitySystemComponentFromPawn();
    if (!ASC)
    {
        return false;
    }
    
    return ASC->TryActivateAbilityByClass(AbilityClass);
}
```

### 2. Behavior Tree Tasks for Abilities

Create custom behavior tree tasks to use abilities:

```cpp
// BTTask_UseAbility.h
#pragma once

#include "CoreMinimal.h"
#include "BehaviorTree/BTTaskNode.h"
#include "GameplayTagContainer.h"
#include "BTTask_UseAbility.generated.h"

/**
 * Task to use abilities from behavior tree.
 */
UCLASS()
class PLATFORMER_API UBTTask_UseAbility : public UBTTaskNode
{
    GENERATED_BODY()

public:
    UBTTask_UseAbility();
    
    // Override ExecuteTask
    virtual EBTNodeResult::Type ExecuteTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory) override;
    
protected:
    // The gameplay tag of the ability to use
    UPROPERTY(EditAnywhere, Category = "Ability")
    FGameplayTagContainer AbilityTagToUse;
    
    // Whether to wait for the ability to end
    UPROPERTY(EditAnywhere, Category = "Ability")
    bool bWaitForAbilityEnd;
};
```

```cpp
// BTTask_UseAbility.cpp
#include "BTTask_UseAbility.h"
#include "PlatformerAIController.h"
#include "BehaviorTree/BlackboardComponent.h"
#include "AIController.h"

UBTTask_UseAbility::UBTTask_UseAbility()
{
    NodeName = "Use Ability";
    bWaitForAbilityEnd = false;
}

EBTNodeResult::Type UBTTask_UseAbility::ExecuteTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory)
{
    APlatformerAIController* AIController = Cast<APlatformerAIController>(OwnerComp.GetAIOwner());
    if (!AIController)
    {
        return EBTNodeResult::Failed;
    }
    
    bool bSuccess = AIController->TryActivateAbilityByTag(AbilityTagToUse);
    return bSuccess ? EBTNodeResult::Succeeded : EBTNodeResult::Failed;
}
```

## Camera System Integration

### 1. Camera Shake Triggers from Abilities

Use gameplay cues to trigger camera effects:

```cpp
// PlatformerCameraManager.h
#pragma once

#include "CoreMinimal.h"
#include "Camera/PlayerCameraManager.h"
#include "GameplayTagContainer.h"
#include "PlatformerCameraManager.generated.h"

class UCameraShakeBase;

/**
 * Camera manager with GAS integration.
 */
UCLASS()
class PLATFORMER_API APlatformerCameraManager : public APlayerCameraManager
{
    GENERATED_BODY()

public:
    APlatformerCameraManager();
    
    // Handle gameplay cue events
    UFUNCTION()
    void OnGameplayCueReceived(FGameplayTag GameplayCueTag, EGameplayCueEvent::Type EventType, const FGameplayCueParameters& Parameters);
    
    // Register with the ability system
    void RegisterWithAbilitySystem(UAbilitySystemComponent* AbilitySystemComponent);
    
    // Unregister from the ability system
    void UnregisterFromAbilitySystem(UAbilitySystemComponent* AbilitySystemComponent);

protected:
    // Map of gameplay cue tags to camera shakes
    UPROPERTY(EditDefaultsOnly, Category = "Camera Shakes")
    TMap<FGameplayTag, TSubclassOf<UCameraShakeBase>> CameraShakeMap;
};
```

```cpp
// PlatformerCameraManager.cpp
#include "PlatformerCameraManager.h"
#include "AbilitySystemComponent.h"
#include "AbilitySystemGlobals.h"

APlatformerCameraManager::APlatformerCameraManager()
{
    // Set up default camera shakes
    static ConstructorHelpers::FClassFinder<UCameraShakeBase> DashShake(TEXT("/Game/Effects/Camera/CS_Dash"));
    static ConstructorHelpers::FClassFinder<UCameraShakeBase> HitShake(TEXT("/Game/Effects/Camera/CS_Hit"));
    static ConstructorHelpers::FClassFinder<UCameraShakeBase> LandingShake(TEXT("/Game/Effects/Camera/CS_Landing"));
    
    if (DashShake.Succeeded())
    {
        CameraShakeMap.Add(FGameplayTag::RequestGameplayTag("GameplayCue.Ability.Dash"), DashShake.Class);
    }
    
    if (HitShake.Succeeded())
    {
        CameraShakeMap.Add(FGameplayTag::RequestGameplayTag("GameplayCue.Character.Hit"), HitShake.Class);
    }
    
    if (LandingShake.Succeeded())
    {
        CameraShakeMap.Add(FGameplayTag::RequestGameplayTag("GameplayCue.Character.Land"), LandingShake.Class);
    }
}

void APlatformerCameraManager::RegisterWithAbilitySystem(UAbilitySystemComponent* AbilitySystemComponent)
{
    if (!AbilitySystemComponent)
    {
        return;
    }
    
    AbilitySystemComponent->GenericGameplayCueCallbacks.AddUObject(this, &APlatformerCameraManager::OnGameplayCueReceived);
}

void APlatformerCameraManager::UnregisterFromAbilitySystem(UAbilitySystemComponent* AbilitySystemComponent)
{
    if (!AbilitySystemComponent)
    {
        return;
    }
    
    AbilitySystemComponent->GenericGameplayCueCallbacks.RemoveAll(this);
}

void APlatformerCameraManager::OnGameplayCueReceived(FGameplayTag GameplayCueTag, EGameplayTagEvent::Type EventType, const FGameplayCueParameters& Parameters)
{
    if (EventType != EGameplayCueEvent::OnActive && EventType != EGameplayCueEvent::Executed)
    {
        return;
    }
    
    // Try to find a camera shake for this gameplay cue
    TSubclassOf<UCameraShakeBase>* CameraShakeClass = CameraShakeMap.Find(GameplayCueTag);
    if (CameraShakeClass && *CameraShakeClass)
    {
        // Scale the shake based on the magnitude
        float Scale = 1.0f;
        if (Parameters.Magnitude > 0.0f)
        {
            Scale = Parameters.Magnitude;
        }
        
        // Play the camera shake
        PlayerCameraManager->StartCameraShake(*CameraShakeClass, Scale);
    }
}
```

## Level Design Integration

### 1. Interactive Objects with GAS

Create level objects that can grant abilities or trigger effects:

```cpp
// PlatformerPowerUp.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "GameplayEffectTypes.h"
#include "PlatformerPowerUp.generated.h"

class UGameplayEffect;
class UGameplayAbility;
class USphereComponent;

/**
 * Power-up actor that can grant abilities or apply effects.
 */
UCLASS()
class PLATFORMER_API APlatformerPowerUp : public AActor
{
    GENERATED_BODY()

public:
    APlatformerPowerUp();
    
    // Called when a character overlaps with this power-up
    UFUNCTION()
    void OnOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, 
        int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult);
        
protected:
    // The collision component
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    USphereComponent* CollisionComponent;
    
    // Mesh component
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    UStaticMeshComponent* MeshComponent;
    
    // Effect to apply when collected
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "PowerUp")
    TSubclassOf<UGameplayEffect> GameplayEffectClass;
    
    // Ability to grant when collected
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "PowerUp")
    TSubclassOf<UGameplayAbility> GameplayAbilityClass;
    
    // Level of the effect/ability
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "PowerUp")
    float PowerUpLevel = 1.0f;
    
    // Apply the power-up to the character
    void ApplyPowerUpToCharacter(AActor* Character);
    
    // Gameplay cue to trigger on pickup
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "PowerUp")
    FGameplayTag PickupGameplayCueTag;
    
    // Whether this power-up should be destroyed after use
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "PowerUp")
    bool bDestroyOnUse = true;
    
    // How long before the power-up respawns if not destroyed
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "PowerUp", meta = (EditCondition = "!bDestroyOnUse"))
    float RespawnTime = 10.0f;
    
    // Handle respawning
    void Respawn();
    
    // Whether the power-up is active
    bool bIsActive = true;
    
    // Timer handle for respawn
    FTimerHandle RespawnTimerHandle;
};
```

```cpp
// PlatformerPowerUp.cpp
#include "PlatformerPowerUp.h"
#include "Components/SphereComponent.h"
#include "AbilitySystemInterface.h"
#include "AbilitySystemComponent.h"
#include "AbilitySystemBlueprintLibrary.h"

APlatformerPowerUp::APlatformerPowerUp()
{
    // Set up components
    CollisionComponent = CreateDefaultSubobject<USphereComponent>(TEXT("CollisionComponent"));
    RootComponent = CollisionComponent;
    
    MeshComponent = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("MeshComponent"));
    MeshComponent->SetupAttachment(RootComponent);
    
    // Set up collision
    CollisionComponent->SetCollisionProfileName("Trigger");
    CollisionComponent->OnComponentBeginOverlap.AddDynamic(this, &APlatformerPowerUp::OnOverlap);
}

void APlatformerPowerUp::OnOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, 
                                  UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, 
                                  bool bFromSweep, const FHitResult& SweepResult)
{
    if (!bIsActive || !OtherActor)
    {
        return;
    }
    
    // Check if actor implements ability system interface
    IAbilitySystemInterface* AbilitySystemInterface = Cast<IAbilitySystemInterface>(OtherActor);
    if (!AbilitySystemInterface)
    {
        return;
    }
    
    // Apply power-up
    ApplyPowerUpToCharacter(OtherActor);
    
    // Handle pickup
    if (bDestroyOnUse)
    {
        Destroy();
    }
    else
    {
        // Deactivate and start respawn timer
        bIsActive = false;
        MeshComponent->SetVisibility(false);
        CollisionComponent->SetCollisionEnabled(ECollisionEnabled::NoCollision);
        GetWorldTimerManager().SetTimer(RespawnTimerHandle, this, &APlatformerPowerUp::Respawn, RespawnTime, false);
    }
}

void APlatformerPowerUp::ApplyPowerUpToCharacter(AActor* Character)
{
    UAbilitySystemComponent* AbilitySystemComponent = 
        UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent(Character);
    
    if (!AbilitySystemComponent)
    {
        return;
    }
    
    // Apply effect if specified
    if (GameplayEffectClass)
    {
        FGameplayEffectContextHandle EffectContext = AbilitySystemComponent->MakeEffectContext();
        EffectContext.AddSourceObject(this);
        
        FGameplayEffectSpecHandle SpecHandle = AbilitySystemComponent->MakeOutgoingSpec(
            GameplayEffectClass, PowerUpLevel, EffectContext);
            
        if (SpecHandle.IsValid())
        {
            AbilitySystemComponent->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get());
        }
    }
    
    // Grant ability if specified
    if (GameplayAbilityClass)
    {
        // Generate a new ability handle (we don't care about the returned handle)
        AbilitySystemComponent->GiveAbility(
            FGameplayAbilitySpec(GameplayAbilityClass, PowerUpLevel, INDEX_NONE, this));
    }
    
    // Trigger gameplay cue
    if (PickupGameplayCueTag.IsValid())
    {
        FGameplayCueParameters CueParams;
        CueParams.Location = GetActorLocation();
        CueParams.Normal = FVector::UpVector;
        CueParams.Instigator = Character;
        CueParams.EffectCauser = this;
        CueParams.SourceObject = this;
        CueParams.Magnitude = PowerUpLevel;
        
        AbilitySystemComponent->ExecuteGameplayCue(PickupGameplayCueTag, CueParams);
    }
}

void APlatformerPowerUp::Respawn()
{
    bIsActive = true;
    MeshComponent->SetVisibility(true);
    CollisionComponent->SetCollisionEnabled(ECollisionEnabled::QueryOnly);
}
```

### 2. Level Triggers Using GAS

Create trigger volumes that apply effects or trigger abilities:

```cpp
// PlatformerAbilityTriggerVolume.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/TriggerVolume.h"
#include "GameplayTagContainer.h"
#include "PlatformerAbilityTriggerVolume.generated.h"

class UGameplayEffect;

/**
 * Trigger volume that activates abilities or applies effects.
 */
UCLASS()
class PLATFORMER_API APlatformerAbilityTriggerVolume : public ATriggerVolume
{
    GENERATED_BODY()

public:
    APlatformerAbilityTriggerVolume();
    
    // Begin play override
    virtual void BeginPlay() override;
    
    // Actor begin overlap
    UFUNCTION()
    void OnActorBeginOverlap(AActor* OverlappedActor, AActor* OtherActor);
    
    // Actor end overlap
    UFUNCTION()
    void OnActorEndOverlap(AActor* OverlappedActor, AActor* OtherActor);
    
protected:
    // Tags required for interaction
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Trigger")
    FGameplayTagContainer RequiredTags;
    
    // Tags blocked from interaction
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Trigger")
    FGameplayTagContainer BlockedTags;
    
    // Gameplay event to send
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Trigger")
    FGameplayTag EventTag;
    
    // Gameplay effect to apply
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Trigger")
    TSubclassOf<UGameplayEffect> EffectToApply;
    
    // Gameplay cue to trigger
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Trigger")
    FGameplayTag GameplayCueTag;
    
    // Whether effects apply only once or continuously
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Trigger")
    bool bApplyOnce = true;
    
    // Event magnitude
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Trigger")
    float EventMagnitude = 1.0f;
    
    // Check if an actor meets tag requirements
    bool DoesActorMeetTagRequirements(AActor* Actor) const;
    
    // Apply effects to actor
    void ApplyEffectsToActor(AActor* Actor);
};
```

```cpp
// PlatformerAbilityTriggerVolume.cpp
#include "PlatformerAbilityTriggerVolume.h"
#include "AbilitySystemInterface.h"
#include "AbilitySystemComponent.h"
#include "AbilitySystemBlueprintLibrary.h"

APlatformerAbilityTriggerVolume::APlatformerAbilityTriggerVolume()
{
    // Set up defaults
    PrimaryActorTick.bCanEverTick = false;
}

void APlatformerAbilityTriggerVolume::BeginPlay()
{
    Super::BeginPlay();
    
    // Bind overlap events
    OnActorBeginOverlap.AddDynamic(this, &APlatformerAbilityTriggerVolume::OnActorBeginOverlap);
    OnActorEndOverlap.AddDynamic(this, &APlatformerAbilityTriggerVolume::OnActorEndOverlap);
}

void APlatformerAbilityTriggerVolume::OnActorBeginOverlap(AActor* OverlappedActor, AActor* OtherActor)
{
    if (!OtherActor || !DoesActorMeetTagRequirements(OtherActor))
    {
        return;
    }
    
    // Apply effects when actor enters
    ApplyEffectsToActor(OtherActor);
    
    // If continuous, start applying effects regularly
    if (!bApplyOnce)
    {
        // You could set up a timer here for continuous effects
    }
}

void APlatformerAbilityTriggerVolume::OnActorEndOverlap(AActor* OverlappedActor, AActor* OtherActor)
{
    if (!OtherActor)
    {
        return;
    }
    
    // If continuous effect, stop the timer
    if (!bApplyOnce)
    {
        // Stop any timers if continuous mode
    }
}

bool APlatformerAbilityTriggerVolume::DoesActorMeetTagRequirements(AActor* Actor) const
{
    UAbilitySystemComponent* AbilitySystemComponent = 
        UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent(Actor);
        
    if (!AbilitySystemComponent)
    {
        return false;
    }
    
    // Check required tags
    if (!RequiredTags.IsEmpty())
    {
        if (!AbilitySystemComponent->HasAllMatchingGameplayTags(RequiredTags))
        {
            return false;
        }
    }
    
    // Check blocked tags
    if (!BlockedTags.IsEmpty())
    {
        if (AbilitySystemComponent->HasAnyMatchingGameplayTags(BlockedTags))
        {
            return false;
        }
    }
    
    return true;
}

void APlatformerAbilityTriggerVolume::ApplyEffectsToActor(AActor* Actor)
{
    UAbilitySystemComponent* AbilitySystemComponent = 
        UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent(Actor);
        
    if (!AbilitySystemComponent)
    {
        return;
    }
    
    // Trigger gameplay event
    if (EventTag.IsValid())
    {
        FGameplayEventData EventData;
        EventData.EventTag = EventTag;
        EventData.Instigator = this;
        EventData.Target = Actor;
        EventData.EventMagnitude = EventMagnitude;
        
        AbilitySystemComponent->HandleGameplayEvent(EventTag, &EventData);
    }
    
    // Apply gameplay effect
    if (EffectToApply)
    {
        FGameplayEffectContextHandle EffectContext = AbilitySystemComponent->MakeEffectContext();
        EffectContext.AddSourceObject(this);
        
        FGameplayEffectSpecHandle SpecHandle = AbilitySystemComponent->MakeOutgoingSpec(
            EffectToApply, 1.0f, EffectContext);
            
        if (SpecHandle.IsValid())
        {
            AbilitySystemComponent->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get());
        }
    }
    
    // Trigger gameplay cue
    if (GameplayCueTag.IsValid())
    {
        FGameplayCueParameters CueParams;
        CueParams.Location = GetActorLocation();
        CueParams.Normal = FVector::UpVector;
        CueParams.Instigator = Actor;
        CueParams.EffectCauser = this;
        CueParams.SourceObject = this;
        CueParams.Magnitude = EventMagnitude;
        
        AbilitySystemComponent->ExecuteGameplayCue(GameplayCueTag, CueParams);
    }
}
```

## Next Steps

After integrating GAS with your game's core systems, consider implementing:

1. [Character Progression Systems](./6_7_character_progression.md) - Implement experience, leveling, and ability unlocking
2. [Networking and Multiplayer](./6_8_gas_networking.md) - Configure GAS for multiplayer gameplay

## Common Issues and Solutions

- **Input Binding Issues**: Check that ability input IDs match between the ability and input binding
- **Animation Sync Problems**: Use gameplay tags to synchronize animation states with ability states
- **Camera Effects Not Triggering**: Verify gameplay cue tags match between abilities and camera system
- **AI Not Using Abilities**: Ensure AI controllers have proper permission to activate abilities
- **Level Interactions Not Working**: Confirm that trigger volumes and interactive objects are properly set up with GAS integration

## Best Practices

- **Use Gameplay Tags Consistently**: Maintain a clear tag hierarchy across all systems
- **Create Reusable Components**: Design UI widgets and interaction components to be reusable
- **Separate Visuals from Logic**: Keep gameplay logic in abilities and effects, visual feedback in cues
- **Blueprint-Friendly Design**: Design C++ classes with Blueprint extensibility in mind
- **Profile Performance**: GAS integration can be performance-intensive; profile your game regularly
- **Documentation**: Maintain clear documentation on your tag hierarchy and integration points
``` 
</rewritten_file>