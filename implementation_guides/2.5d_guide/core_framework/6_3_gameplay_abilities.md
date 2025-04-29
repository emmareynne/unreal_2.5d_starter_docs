# Gameplay Ability System: Implementing Abilities

This guide covers implementing gameplay abilities for your 2.5D platformer using Unreal Engine 5.5's Gameplay Ability System.

## What Are Gameplay Abilities?

Gameplay Abilities represent discrete actions that characters can perform, such as jumping, dashing, attacking, or using special moves. Key features include:

- Self-contained logic for ability activation and execution
- Input binding system for triggering abilities
- Cost and cooldown management
- Integration with gameplay tags for requirements and effects
- Network replication for multiplayer games

## Setting Up Basic Platformer Abilities

For a 2.5D platformer, let's implement these core abilities:
1. Jump
2. Double Jump
3. Dash

### Common Ability Structure

All abilities should follow this general flow:
1. Player provides input
2. Ability checks if it can be activated
3. Ability commits resources (if applicable)
4. Ability executes its effects
5. Ability ends

## Implementing the Jump Ability

### 1. Create the Jump Ability Class

```cpp
// PlatformerJumpAbility.h
#pragma once

#include "CoreMinimal.h"
#include "PlatformerGameplayAbility.h"
#include "PlatformerJumpAbility.generated.h"

/**
 * Basic jump ability for platformer character.
 */
UCLASS()
class PLATFORMER_API UPlatformerJumpAbility : public UPlatformerGameplayAbility
{
    GENERATED_BODY()

public:
    UPlatformerJumpAbility();

    // Overrides from UGameplayAbility
    virtual bool CanActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayTagContainer* SourceTags, const FGameplayTagContainer* TargetTags, OUT FGameplayTagContainer* OptionalRelevantTags) const override;
    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEventData* TriggerEventData) override;
    virtual void EndAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, bool bReplicateEndAbility, bool bWasCancelled) override;

protected:
    // Jump force - can be adjusted in Blueprint derived classes
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Jump")
    float JumpForce = 600.0f;

    // Optional: Override in Blueprint
    UFUNCTION(BlueprintImplementableEvent, Category = "Jump")
    void OnJumpStarted();

    UFUNCTION(BlueprintImplementableEvent, Category = "Jump")
    void OnJumpCompleted();
};
```

```cpp
// PlatformerJumpAbility.cpp
#include "PlatformerJumpAbility.h"
#include "GameFramework/Character.h"
#include "GameFramework/CharacterMovementComponent.h"
#include "AbilitySystemComponent.h"

UPlatformerJumpAbility::UPlatformerJumpAbility()
{
    // Set up ability properties
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::LocalPredicted;
    
    // Set up tags
    FGameplayTagContainer AbilityTags;
    AbilityTags.AddTag(FGameplayTag::RequestGameplayTag(FName("Ability.Movement.Jump")));
    AbilityTags.AddTag(FGameplayTag::RequestGameplayTag(FName("State.InAir")));
    
    AbilityTags.AddTag(FGameplayTag::RequestGameplayTag(FName("Ability.Movement")));
    
    // Tags that block this ability
    ActivationBlockedTags.AddTag(FGameplayTag::RequestGameplayTag(FName("State.Dead")));
    ActivationBlockedTags.AddTag(FGameplayTag::RequestGameplayTag(FName("State.Stunned")));
    
    // Tags that cancel this ability
    CancelAbilitiesWithTag.AddTag(FGameplayTag::RequestGameplayTag(FName("Ability.Movement.Jump")));
}

bool UPlatformerJumpAbility::CanActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayTagContainer* SourceTags, const FGameplayTagContainer* TargetTags, OUT FGameplayTagContainer* OptionalRelevantTags) const
{
    if (!Super::CanActivateAbility(Handle, ActorInfo, SourceTags, TargetTags, OptionalRelevantTags))
    {
        return false;
    }

    const ACharacter* Character = Cast<ACharacter>(ActorInfo->AvatarActor.Get());
    if (!Character)
    {
        return false;
    }

    // Can only jump if on the ground
    if (!Character->GetCharacterMovement()->IsMovingOnGround())
    {
        return false;
    }

    return true;
}

void UPlatformerJumpAbility::ActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEventData* TriggerEventData)
{
    // Call parent first - will handle things like committing ability costs
    if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
        return;
    }

    // Get the character
    ACharacter* Character = Cast<ACharacter>(ActorInfo->AvatarActor.Get());
    if (!Character)
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
        return;
    }

    // Apply jump force
    Character->GetCharacterMovement()->Velocity.Z = JumpForce;
    Character->GetCharacterMovement()->SetMovementMode(MOVE_Falling);
    
    // Apply tags
    UAbilitySystemComponent* AbilitySystemComponent = ActorInfo->AbilitySystemComponent.Get();
    if (AbilitySystemComponent)
    {
        FGameplayTagContainer TagsToAdd;
        TagsToAdd.AddTag(FGameplayTag::RequestGameplayTag(FName("State.InAir")));
        AbilitySystemComponent->AddLooseGameplayTags(TagsToAdd);
    }

    // Notify Blueprint
    OnJumpStarted();

    // End the ability immediately since jumping is an instant action
    // The tags will be removed when the character lands
    EndAbility(Handle, ActorInfo, ActivationInfo, true, false);
}

void UPlatformerJumpAbility::EndAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, bool bReplicateEndAbility, bool bWasCancelled)
{
    // Notify Blueprint
    OnJumpCompleted();
    
    Super::EndAbility(Handle, ActorInfo, ActivationInfo, bReplicateEndAbility, bWasCancelled);
}
```

## Implementing the Double Jump Ability

```cpp
// PlatformerDoubleJumpAbility.h
#pragma once

#include "CoreMinimal.h"
#include "PlatformerGameplayAbility.h"
#include "PlatformerDoubleJumpAbility.generated.h"

/**
 * Double jump ability for platformer character.
 */
UCLASS()
class PLATFORMER_API UPlatformerDoubleJumpAbility : public UPlatformerGameplayAbility
{
    GENERATED_BODY()

public:
    UPlatformerDoubleJumpAbility();

    // Overrides from UGameplayAbility
    virtual bool CanActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayTagContainer* SourceTags, const FGameplayTagContainer* TargetTags, OUT FGameplayTagContainer* OptionalRelevantTags) const override;
    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEventData* TriggerEventData) override;
    virtual void EndAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, bool bReplicateEndAbility, bool bWasCancelled) override;

protected:
    // Double jump force - can be adjusted in Blueprint derived classes
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Jump")
    float DoubleJumpForce = 600.0f;

    // Optional: Double jump height multiplier compared to regular jump
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Jump")
    float HeightMultiplier = 0.8f;
    
    // Optional: Override in Blueprint
    UFUNCTION(BlueprintImplementableEvent, Category = "Jump")
    void OnDoubleJumpStarted();

    UFUNCTION(BlueprintImplementableEvent, Category = "Jump")
    void OnDoubleJumpCompleted();
};
```

```cpp
// PlatformerDoubleJumpAbility.cpp
#include "PlatformerDoubleJumpAbility.h"
#include "GameFramework/Character.h"
#include "GameFramework/CharacterMovementComponent.h"
#include "AbilitySystemComponent.h"

UPlatformerDoubleJumpAbility::UPlatformerDoubleJumpAbility()
{
    // Set up ability properties
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::LocalPredicted;
    
    // Set up tags
    FGameplayTagContainer AbilityTags;
    AbilityTags.AddTag(FGameplayTag::RequestGameplayTag(FName("Ability.Movement.DoubleJump")));
    AbilityTags.AddTag(FGameplayTag::RequestGameplayTag(FName("State.InAir")));
    AbilityTags.AddTag(FGameplayTag::RequestGameplayTag(FName("State.DoubleJumping")));
    
    // Tags that block this ability
    ActivationBlockedTags.AddTag(FGameplayTag::RequestGameplayTag(FName("State.Dead")));
    ActivationBlockedTags.AddTag(FGameplayTag::RequestGameplayTag(FName("State.Stunned")));
    ActivationBlockedTags.AddTag(FGameplayTag::RequestGameplayTag(FName("State.DoubleJumping")));
    
    // Tags that cancel this ability
    CancelAbilitiesWithTag.AddTag(FGameplayTag::RequestGameplayTag(FName("Ability.Movement.Jump")));
}

bool UPlatformerDoubleJumpAbility::CanActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayTagContainer* SourceTags, const FGameplayTagContainer* TargetTags, OUT FGameplayTagContainer* OptionalRelevantTags) const
{
    if (!Super::CanActivateAbility(Handle, ActorInfo, SourceTags, TargetTags, OptionalRelevantTags))
    {
        return false;
    }

    const ACharacter* Character = Cast<ACharacter>(ActorInfo->AvatarActor.Get());
    if (!Character)
    {
        return false;
    }

    // Must be in the air and not already used double jump
    if (Character->GetCharacterMovement()->IsMovingOnGround())
    {
        return false;
    }

    // Check if the double jump has already been used
    UAbilitySystemComponent* AbilitySystemComponent = ActorInfo->AbilitySystemComponent.Get();
    if (AbilitySystemComponent && AbilitySystemComponent->HasMatchingGameplayTag(FGameplayTag::RequestGameplayTag(FName("State.DoubleJumping"))))
    {
        return false;
    }

    return true;
}

void UPlatformerDoubleJumpAbility::ActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEventData* TriggerEventData)
{
    // Call parent first - will handle things like committing ability costs
    if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
        return;
    }

    // Get the character
    ACharacter* Character = Cast<ACharacter>(ActorInfo->AvatarActor.Get());
    if (!Character)
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
        return;
    }

    // Apply double jump force
    Character->GetCharacterMovement()->Velocity.Z = DoubleJumpForce;
    
    // Apply tags
    UAbilitySystemComponent* AbilitySystemComponent = ActorInfo->AbilitySystemComponent.Get();
    if (AbilitySystemComponent)
    {
        FGameplayTagContainer TagsToAdd;
        TagsToAdd.AddTag(FGameplayTag::RequestGameplayTag(FName("State.DoubleJumping")));
        AbilitySystemComponent->AddLooseGameplayTags(TagsToAdd);
    }

    // Notify Blueprint
    OnDoubleJumpStarted();

    // End the ability immediately since double jumping is an instant action
    EndAbility(Handle, ActorInfo, ActivationInfo, true, false);
}

void UPlatformerDoubleJumpAbility::EndAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, bool bReplicateEndAbility, bool bWasCancelled)
{
    // Notify Blueprint
    OnDoubleJumpCompleted();
    
    Super::EndAbility(Handle, ActorInfo, ActivationInfo, bReplicateEndAbility, bWasCancelled);
}
```

## Implementing the Dash Ability

```cpp
// PlatformerDashAbility.h
#pragma once

#include "CoreMinimal.h"
#include "PlatformerGameplayAbility.h"
#include "GameplayTagContainer.h"
#include "Abilities/Tasks/AbilityTask_MoveToLocation.h"
#include "PlatformerDashAbility.generated.h"

/**
 * Dash ability for platformer character.
 */
UCLASS()
class PLATFORMER_API UPlatformerDashAbility : public UPlatformerGameplayAbility
{
    GENERATED_BODY()

public:
    UPlatformerDashAbility();

    // Overrides from UGameplayAbility
    virtual bool CanActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayTagContainer* SourceTags, const FGameplayTagContainer* TargetTags, OUT FGameplayTagContainer* OptionalRelevantTags) const override;
    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEventData* TriggerEventData) override;
    virtual void EndAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, bool bReplicateEndAbility, bool bWasCancelled) override;

protected:
    // Dash distance
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Dash")
    float DashDistance = 500.0f;

    // Dash duration
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Dash")
    float DashDuration = 0.2f;

    // Dash cooldown
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Dash")
    float DashCooldown = 1.0f;

    // Optional: Override in Blueprint
    UFUNCTION(BlueprintImplementableEvent, Category = "Dash")
    void OnDashStarted();

    UFUNCTION(BlueprintImplementableEvent, Category = "Dash")
    void OnDashCompleted();

private:
    // Movement task
    UPROPERTY()
    UAbilityTask_MoveToLocation* DashTask;

    // Callback functions for ability task
    UFUNCTION()
    void OnDashFinished();

    UFUNCTION()
    void OnDashCancelled();
};
```

```cpp
// PlatformerDashAbility.cpp
#include "PlatformerDashAbility.h"
#include "GameFramework/Character.h"
#include "GameFramework/CharacterMovementComponent.h"
#include "AbilitySystemComponent.h"
#include "Abilities/Tasks/AbilityTask_MoveToLocation.h"

UPlatformerDashAbility::UPlatformerDashAbility()
{
    // Set up ability properties
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::LocalPredicted;
    
    // Set up tags
    FGameplayTagContainer AbilityTags;
    AbilityTags.AddTag(FGameplayTag::RequestGameplayTag(FName("Ability.Movement.Dash")));
    AbilityTags.AddTag(FGameplayTag::RequestGameplayTag(FName("State.Dashing")));
    
    // Tags that block this ability
    ActivationBlockedTags.AddTag(FGameplayTag::RequestGameplayTag(FName("State.Dead")));
    ActivationBlockedTags.AddTag(FGameplayTag::RequestGameplayTag(FName("State.Stunned")));
    ActivationBlockedTags.AddTag(FGameplayTag::RequestGameplayTag(FName("State.Dashing")));
    
    // Tags that cancel this ability
    CancelAbilitiesWithTag.AddTag(FGameplayTag::RequestGameplayTag(FName("Ability.Movement.Jump")));
    CancelAbilitiesWithTag.AddTag(FGameplayTag::RequestGameplayTag(FName("Ability.Movement.DoubleJump")));
}

bool UPlatformerDashAbility::CanActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayTagContainer* SourceTags, const FGameplayTagContainer* TargetTags, OUT FGameplayTagContainer* OptionalRelevantTags) const
{
    if (!Super::CanActivateAbility(Handle, ActorInfo, SourceTags, TargetTags, OptionalRelevantTags))
    {
        return false;
    }

    const ACharacter* Character = Cast<ACharacter>(ActorInfo->AvatarActor.Get());
    if (!Character)
    {
        return false;
    }

    // Check cooldown
    UAbilitySystemComponent* AbilitySystemComponent = ActorInfo->AbilitySystemComponent.Get();
    if (AbilitySystemComponent && AbilitySystemComponent->HasMatchingGameplayTag(FGameplayTag::RequestGameplayTag(FName("Cooldown.Ability.Movement.Dash"))))
    {
        return false;
    }

    return true;
}

void UPlatformerDashAbility::ActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEventData* TriggerEventData)
{
    // Call parent first - will handle things like committing ability costs
    if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
        return;
    }

    // Get the character
    ACharacter* Character = Cast<ACharacter>(ActorInfo->AvatarActor.Get());
    if (!Character)
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
        return;
    }

    // Calculate dash direction based on character's current orientation
    FVector ForwardVector = Character->GetActorForwardVector();
    FVector DashLocation = Character->GetActorLocation() + (ForwardVector * DashDistance);
    
    // Create the dash task
    DashTask = UAbilityTask_MoveToLocation::MoveToLocation(
        this,
        TEXT("DashTask"),
        DashLocation,
        DashDuration, 
        nullptr,  // Optional curve
        false     // Save previous movement
    );
    
    if (DashTask)
    {
        DashTask->OnTargetLocationReached.AddDynamic(this, &UPlatformerDashAbility::OnDashFinished);
        DashTask->OnCancelled.AddDynamic(this, &UPlatformerDashAbility::OnDashCancelled);
        DashTask->ReadyForActivation();
        
        // Apply tags
        UAbilitySystemComponent* AbilitySystemComponent = ActorInfo->AbilitySystemComponent.Get();
        if (AbilitySystemComponent)
        {
            FGameplayTagContainer TagsToAdd;
            TagsToAdd.AddTag(FGameplayTag::RequestGameplayTag(FName("State.Dashing")));
            AbilitySystemComponent->AddLooseGameplayTags(TagsToAdd);
            
            // Apply cooldown tag (will be removed after cooldown duration)
            FGameplayTagContainer CooldownTags;
            CooldownTags.AddTag(FGameplayTag::RequestGameplayTag(FName("Cooldown.Ability.Movement.Dash")));
            AbilitySystemComponent->AddLooseGameplayTags(CooldownTags);
            
            // Start a timer to remove the cooldown tag
            FTimerHandle CooldownTimerHandle;
            FTimerDelegate CooldownTimerDelegate;
            CooldownTimerDelegate.BindLambda([AbilitySystemComponent]() {
                FGameplayTagContainer CooldownTags;
                CooldownTags.AddTag(FGameplayTag::RequestGameplayTag(FName("Cooldown.Ability.Movement.Dash")));
                AbilitySystemComponent->RemoveLooseGameplayTags(CooldownTags);
            });
            
            Character->GetWorldTimerManager().SetTimer(
                CooldownTimerHandle,
                CooldownTimerDelegate,
                DashCooldown,
                false
            );
        }
        
        // Modify character movement during dash
        UCharacterMovementComponent* MovementComponent = Character->GetCharacterMovement();
        if (MovementComponent)
        {
            // Store current gravity scale to restore later
            MovementComponent->GravityScale = 0.0f; // Disable gravity during dash
        }
        
        // Notify Blueprint
        OnDashStarted();
    }
    else
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
    }
}

void UPlatformerDashAbility::EndAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, bool bReplicateEndAbility, bool bWasCancelled)
{
    // Cancel the dash task if it's active
    if (DashTask)
    {
        DashTask->EndTask();
        DashTask = nullptr;
    }
    
    // Restore character movement properties
    ACharacter* Character = Cast<ACharacter>(ActorInfo->AvatarActor.Get());
    if (Character)
    {
        UCharacterMovementComponent* MovementComponent = Character->GetCharacterMovement();
        if (MovementComponent)
        {
            MovementComponent->GravityScale = 1.0f; // Restore default gravity
        }
    }
    
    // Remove the dashing tag
    UAbilitySystemComponent* AbilitySystemComponent = ActorInfo->AbilitySystemComponent.Get();
    if (AbilitySystemComponent)
    {
        FGameplayTagContainer TagsToRemove;
        TagsToRemove.AddTag(FGameplayTag::RequestGameplayTag(FName("State.Dashing")));
        AbilitySystemComponent->RemoveLooseGameplayTags(TagsToRemove);
    }

    // Notify Blueprint
    OnDashCompleted();
    
    Super::EndAbility(Handle, ActorInfo, ActivationInfo, bReplicateEndAbility, bWasCancelled);
}

void UPlatformerDashAbility::OnDashFinished()
{
    // Dash completed successfully
    EndAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, true, false);
}

void UPlatformerDashAbility::OnDashCancelled()
{
    // Dash was cancelled
    EndAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, true, true);
}
```

## Binding Abilities to Input

To connect your abilities to player input, you need to:
1. Define input actions
2. Bind abilities to these inputs

### Step 1: Define Input IDs

```cpp
// PlatformerAbilityInputID.h
#pragma once

#include "CoreMinimal.h"

// Define input IDs for abilities
UENUM(BlueprintType)
enum class EPlatformerAbilityInputID : uint8
{
    None            UMETA(DisplayName = "None"),
    Confirm         UMETA(DisplayName = "Confirm"),
    Cancel          UMETA(DisplayName = "Cancel"),
    Jump            UMETA(DisplayName = "Jump"),
    Dash            UMETA(DisplayName = "Dash"),
    Attack          UMETA(DisplayName = "Attack"),
    SpecialAbility  UMETA(DisplayName = "Special Ability")
};
```

### Step 2: Bind Abilities to Input in Your Character Class

```cpp
// PlatformerCharacter.h
#include "AbilitySystemInterface.h"
#include "PlatformerAbilityInputID.h"
#include "GameplayAbilitySpec.h"

UCLASS()
class PLATFORMER_API APlatformerCharacter : public ACharacter, public IAbilitySystemInterface
{
    GENERATED_BODY()

public:
    // Constructor and standard overrides...
    
    // Implement the IAbilitySystemInterface
    virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;
    
    // Called when player binds input
    virtual void SetupPlayerInputComponent(class UInputComponent* PlayerInputComponent) override;
    
    // Grant initial abilities
    virtual void PossessedBy(AController* NewController) override;
    
protected:
    // Ability System Component
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Abilities")
    class UPlatformerAbilitySystemComponent* AbilitySystemComponent;
    
    // Ability sets to grant on possession
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Abilities")
    TArray<TSubclassOf<class UGameplayAbility>> DefaultAbilities;
    
    // Handle input for abilities
    void HandleAbilityInput(EPlatformerAbilityInputID AbilityInputID, bool bPressed);
    
    // Map of granted ability specs
    TMap<EPlatformerAbilityInputID, FGameplayAbilitySpecHandle> AbilityHandles;
};
```

```cpp
// PlatformerCharacter.cpp
void APlatformerCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    Super::SetupPlayerInputComponent(PlayerInputComponent);
    
    // Set up ability inputs
    PlayerInputComponent->BindAction("Jump", IE_Pressed, this, &APlatformerCharacter::HandleJumpInput);
    PlayerInputComponent->BindAction("Dash", IE_Pressed, this, &APlatformerCharacter::HandleDashInput);
    // Add more ability bindings as needed
}

void APlatformerCharacter::HandleJumpInput()
{
    HandleAbilityInput(EPlatformerAbilityInputID::Jump, true);
}

void APlatformerCharacter::HandleDashInput()
{
    HandleAbilityInput(EPlatformerAbilityInputID::Dash, true);
}

void APlatformerCharacter::HandleAbilityInput(EPlatformerAbilityInputID AbilityInputID, bool bPressed)
{
    if (AbilitySystemComponent)
    {
        if (bPressed)
        {
            AbilitySystemComponent->AbilityLocalInputPressed(static_cast<int32>(AbilityInputID));
        }
        else
        {
            AbilitySystemComponent->AbilityLocalInputReleased(static_cast<int32>(AbilityInputID));
        }
    }
}

void APlatformerCharacter::PossessedBy(AController* NewController)
{
    Super::PossessedBy(NewController);
    
    // Initialize the ASC and grant abilities
    if (AbilitySystemComponent)
    {
        AbilitySystemComponent->InitAbilityActorInfo(this, this);
        
        // Grant default abilities
        for (TSubclassOf<UGameplayAbility>& DefaultAbility : DefaultAbilities)
        {
            if (DefaultAbility)
            {
                UPlatformerGameplayAbility* AbilityCDO = DefaultAbility->GetDefaultObject<UPlatformerGameplayAbility>();
                
                // Use the ability's Input ID to map it for input binding
                EPlatformerAbilityInputID AbilityInputID = static_cast<EPlatformerAbilityInputID>(AbilityCDO->AbilityInputID);
                
                // Grant the ability to the character
                FGameplayAbilitySpecHandle AbilityHandle = AbilitySystemComponent->GiveAbility(
                    FGameplayAbilitySpec(DefaultAbility, 1, static_cast<int32>(AbilityInputID), this));
                
                // Store the handle for later reference
                AbilityHandles.Add(AbilityInputID, AbilityHandle);
            }
        }
    }
}
```

## Using Ability Tasks

For more complex abilities, you may need to use Ability Tasks. These are specialized asynchronous tasks that abilities can use to perform complex operations over time.

### Example: WaitMovementModeChange Task

```cpp
// Move this to a separate file for reuse
UCLASS()
class PLATFORMER_API UAbilityTask_WaitMovementModeChange : public UAbilityTask
{
    GENERATED_BODY()

public:
    DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnMovementModeChanged, EMovementMode, PrevMovementMode, EMovementMode, NewMovementMode);
    
    // Delegate called when movement mode changes
    UPROPERTY(BlueprintAssignable)
    FOnMovementModeChanged OnMovementModeChanged;
    
    // Factory method to create the task
    UFUNCTION(BlueprintCallable, Category = "Ability|Tasks", meta = (HidePin = "OwningAbility", DefaultToSelf = "OwningAbility", BlueprintInternalUseOnly = "TRUE"))
    static UAbilityTask_WaitMovementModeChange* WaitForMovementModeChange(UGameplayAbility* OwningAbility, EMovementMode TargetMode);
    
    // Activation
    virtual void Activate() override;
    
    // Cleanup
    virtual void OnDestroy(bool AbilityEnded) override;
    
protected:
    // Target movement mode to wait for
    EMovementMode MovementModeToWaitFor;
    
    // Callback when movement mode changes
    UFUNCTION()
    void OnMovementModeChangedCallback(ACharacter* Character, EMovementMode PrevMovementMode, uint8 PreviousCustomMode);
    
    // Cached character pointer
    UPROPERTY()
    ACharacter* CharacterOwner;
};
```

## Blueprint Integration

Exposing your abilities to Blueprints allows designers to customize them without C++ changes.

### 1. Make Base Classes Blueprint Extendable

```cpp
// In your base ability class
UCLASS(Blueprintable)
class PLATFORMER_API UPlatformerGameplayAbility : public UGameplayAbility
{
    // ...
};
```

### 2. Expose Key Properties to Blueprints

```cpp
// Properties that designers might want to change
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Dash")
float DashDistance = 500.0f;

// Blueprint-implementable events for customization
UFUNCTION(BlueprintImplementableEvent, Category = "Dash")
void OnDashStarted();
```

### 3. Create Blueprint Event Interfaces

```cpp
// Provide events for important moments in the ability lifecycle
UFUNCTION(BlueprintImplementableEvent, Category = "Ability")
void OnAbilityActivated();

UFUNCTION(BlueprintImplementableEvent, Category = "Ability")
void OnAbilityEnded();
```

## Testing Your Abilities

To test your abilities:

1. Create a simple level with platforms
2. Spawn your character with the abilities granted
3. Log ability activation and state changes
4. Use visual feedback (particles, animations) to highlight ability states

```cpp
// Add to your abilities for debugging
void UPlatformerJumpAbility::ActivateAbility(...)
{
    // Existing code...
    
    // Debug logging
    UE_LOG(LogTemp, Log, TEXT("Jump Ability Activated!"));
}
```

## Next Steps

With basic movement abilities implemented, you can now move on to:

1. [Gameplay Effects](./6_4_gameplay_effects.md) - Create effects for abilities, like damage or speed boosts
2. [Gameplay Cues](./6_5_gameplay_cues.md) - Add visual and audio feedback for abilities

## Common Issues and Solutions

- **Abilities Not Activating**: Check input binding and ability activation conditions
- **Ability Ends Immediately**: Make sure it's properly maintaining its active state
- **Network Desyncs**: Verify that the NetExecutionPolicy is appropriate for the ability
- **Tags Not Applied/Removed**: Debug tag application in Activate/EndAbility

## Best Practices

- Keep abilities modular and focused on a single action
- Use tags to manage ability states and requirements
- Leverage Blueprint extensibility for design iteration
- Test abilities in various scenarios (on ground, in air, during other abilities)
- Use proper Ability Tasks for complex, time-based abilities
</rewritten_file> 