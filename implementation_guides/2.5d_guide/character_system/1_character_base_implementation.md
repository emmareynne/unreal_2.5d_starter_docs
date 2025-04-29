# Character Base Implementation for 2.5D Platformer

This guide covers implementing the base character class for your 2.5D platformer in Unreal Engine 5.5, using a component-based approach to create a flexible foundation for all gameplay mechanics.

## Overview

The character system serves as the primary player-controlled entity and consists of:
- A core character class inheriting from ACharacter
- Specialized components for different gameplay systems
- Integration with the Gameplay Ability System
- Custom movement constraints for 2.5D gameplay
- Animation system integration for responsive feedback

This modular design enables clean separation of gameplay mechanics while allowing components to communicate efficiently.

## Prerequisites

Before implementing the character:
- Completed [project setup](../project_setup.md)
- Enabled required plugins (Enhanced Input, GameplayAbilitySystem)
- Configured basic collision presets
- Set up the initial scene layout to test movement

## Implementation Steps

### Step 1: Create the Character Base Class

1. Create a new C++ class inheriting from ACharacter
2. Add component declarations
3. Initialize components in constructor
4. Configure input handling
5. Add animation montage references

```cpp
// PlatformerCharacter.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Character.h"
#include "AbilitySystemInterface.h"
#include "PlatformerCharacter.generated.h"

UCLASS()
class PLATFORMER_API APlatformerCharacter : public ACharacter, public IAbilitySystemInterface
{
    GENERATED_BODY()
    
public:
    // Constructor
    APlatformerCharacter(const FObjectInitializer& ObjectInitializer);
    
    // Override base functions
    virtual void BeginPlay() override;
    virtual void Tick(float DeltaTime) override;
    virtual void SetupPlayerInputComponent(class UInputComponent* PlayerInputComponent) override;
    virtual void PossessedBy(AController* NewController) override;
    
    // IAbilitySystemInterface
    virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;
    
protected:
    // Core components for platformer gameplay
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    class UPlatformerMovementComponent* PlatformerMovement;
    
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    class UPlatformerJumpComponent* JumpComponent;
    
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    class UPlatformerCombatComponent* CombatComponent;
    
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    class UPlatformerInteractionComponent* InteractionComponent;
    
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    class UAbilitySystemComponent* AbilitySystemComponent;
    
    // Timeline for position adjustment during animations
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Animation")
    class UTimelineComponent* PositionAdjustmentTimeline;
    
    // Animation montages
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Animation")
    class UAnimMontage* JumpMontage;
    
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Animation")
    class UAnimMontage* LedgeGrabMontage;
    
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Animation")
    class UAnimMontage* LandingMontage;
    
    // Enhanced Input actions
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Input")
    class UInputAction* MoveAction;
    
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Input")
    class UInputAction* JumpAction;
    
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Input")
    class UInputAction* AttackAction;
    
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Input")
    class UInputMappingContext* DefaultMappingContext;
    
    // Position adjustment functions
    UFUNCTION(BlueprintCallable, Category = "Animation")
    void StartPositionAdjustment(FVector TargetPosition, float Duration = 0.2f);
    
    UFUNCTION()
    void UpdatePositionAdjustment(float Alpha);
    
    UFUNCTION()
    void FinishPositionAdjustment();
    
private:
    // Input handlers
    void HandleMoveInput(const struct FInputActionValue& Value);
    void HandleJumpInput(const struct FInputActionValue& Value);
    void HandleJumpReleased(const struct FInputActionValue& Value);
    void HandleAttackInput(const struct FInputActionValue& Value);
    
    // Stored positions for timeline interpolation
    FVector StartPosition;
    FVector EndPosition;
    
    // Timeline curve
    UPROPERTY()
    class UCurveFloat* PositionAdjustmentCurve;
};
```

### Step 2: Implement Core Character Functions

```cpp
// PlatformerCharacter.cpp
#include "PlatformerCharacter.h"
#include "PlatformerMovementComponent.h"
#include "PlatformerJumpComponent.h"
#include "PlatformerCombatComponent.h"
#include "PlatformerInteractionComponent.h"
#include "AbilitySystemComponent.h"
#include "Components/TimelineComponent.h"
#include "Components/CapsuleComponent.h"
#include "EnhancedInputComponent.h"
#include "EnhancedInputSubsystems.h"
#include "InputActionValue.h"
#include "GameFramework/Controller.h"

APlatformerCharacter::APlatformerCharacter(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer.SetDefaultSubobjectClass<UPlatformerMovementComponent>(ACharacter::CharacterMovementComponentName))
{
    // Set this character to call Tick() every frame
    PrimaryActorTick.bCanEverTick = true;
    
    // Get the movement component (already created by constructor through SetDefaultSubobjectClass)
    PlatformerMovement = Cast<UPlatformerMovementComponent>(GetCharacterMovement());
    
    // Create jump component
    JumpComponent = CreateDefaultSubobject<UPlatformerJumpComponent>(TEXT("JumpComponent"));
    
    // Create combat component
    CombatComponent = CreateDefaultSubobject<UPlatformerCombatComponent>(TEXT("CombatComponent"));
    
    // Create interaction component
    InteractionComponent = CreateDefaultSubobject<UPlatformerInteractionComponent>(TEXT("InteractionComponent"));
    
    // Create ability system component
    AbilitySystemComponent = CreateDefaultSubobject<UAbilitySystemComponent>(TEXT("AbilitySystemComponent"));
    
    // Create position adjustment timeline component
    PositionAdjustmentTimeline = CreateDefaultSubobject<UTimelineComponent>(TEXT("PositionAdjustmentTimeline"));
}
```

### Step 3: Configure Enhanced Input System

The character will use Enhanced Input for more responsive controls:

```cpp
void APlatformerCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    // Cast to Enhanced Input Component
    UEnhancedInputComponent* EnhancedInputComponent = Cast<UEnhancedInputComponent>(PlayerInputComponent);
    if (!EnhancedInputComponent)
    {
        return;
    }
    
    // Get player controller
    APlayerController* PC = Cast<APlayerController>(GetController());
    if (!PC)
    {
        return;
    }
    
    // Get local player subsystem
    UEnhancedInputLocalPlayerSubsystem* Subsystem = ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(PC->GetLocalPlayer());
    if (!Subsystem)
    {
        return;
    }
    
    // Clear any existing mappings and add our mapping context
    Subsystem->ClearMappingContext();
    Subsystem->AddMappingContext(DefaultMappingContext, 0);
    
    // Bind actions
    EnhancedInputComponent->BindAction(MoveAction, ETriggerEvent::Triggered, this, &APlatformerCharacter::HandleMoveInput);
    EnhancedInputComponent->BindAction(JumpAction, ETriggerEvent::Started, this, &APlatformerCharacter::HandleJumpInput);
    EnhancedInputComponent->BindAction(JumpAction, ETriggerEvent::Completed, this, &APlatformerCharacter::HandleJumpReleased);
    EnhancedInputComponent->BindAction(AttackAction, ETriggerEvent::Started, this, &APlatformerCharacter::HandleAttackInput);
}
```

### Step 4: Implement Position Adjustment Timeline

For smooth animation-driven position adjustments:

```cpp
void APlatformerCharacter::BeginPlay()
{
    Super::BeginPlay();
    
    // Set up position adjustment timeline
    if (PositionAdjustmentTimeline)
    {
        FOnTimelineFloat ProgressUpdate;
        ProgressUpdate.BindUFunction(this, FName("UpdatePositionAdjustment"));
        
        FOnTimelineEvent FinishedEvent;
        FinishedEvent.BindUFunction(this, FName("FinishPositionAdjustment"));
        
        // Create curve if needed
        if (!PositionAdjustmentCurve)
        {
            PositionAdjustmentCurve = NewObject<UCurveFloat>(this, UCurveFloat::StaticClass());
            PositionAdjustmentCurve->FloatCurve.AddKey(0.0f, 0.0f);
            PositionAdjustmentCurve->FloatCurve.AddKey(1.0f, 1.0f);
        }
        
        PositionAdjustmentTimeline->AddInterpFloat(PositionAdjustmentCurve, ProgressUpdate);
        PositionAdjustmentTimeline->SetTimelineFinishedFunc(FinishedEvent);
    }
}

void APlatformerCharacter::StartPositionAdjustment(FVector TargetPosition, float Duration)
{
    // Store start and end positions
    StartPosition = GetActorLocation();
    EndPosition = TargetPosition;
    
    // Set timeline duration and play
    PositionAdjustmentTimeline->SetPlayRate(1.0f / FMath::Max(Duration, 0.001f));
    PositionAdjustmentTimeline->PlayFromStart();
}

void APlatformerCharacter::UpdatePositionAdjustment(float Alpha)
{
    // Interpolate position
    FVector NewPosition = FMath::Lerp(StartPosition, EndPosition, Alpha);
    SetActorLocation(NewPosition);
}

void APlatformerCharacter::FinishPositionAdjustment()
{
    // Ensure final position is exact
    SetActorLocation(EndPosition);
}
```

### Step 5: Create Input Handler Functions

```cpp
void APlatformerCharacter::HandleMoveInput(const FInputActionValue& Value)
{
    // Get the movement vector
    FVector2D MovementVector = Value.Get<FVector2D>();
    
    if (Controller != nullptr)
    {
        // Get controller rotation
        const FRotator Rotation = Controller->GetControlRotation();
        const FRotator YawRotation(0, Rotation.Yaw, 0);
        
        // Get forward and right vectors
        const FVector ForwardDirection = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::X);
        const FVector RightDirection = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::Y);
        
        // Add movement input
        AddMovementInput(ForwardDirection, MovementVector.Y);
        AddMovementInput(RightDirection, MovementVector.X);
    }
}

void APlatformerCharacter::HandleJumpInput(const FInputActionValue& Value)
{
    if (JumpComponent)
    {
        JumpComponent->StartJump();
    }
    else
    {
        // Fall back to default jump if component not available
        Jump();
    }
}

void APlatformerCharacter::HandleJumpReleased(const FInputActionValue& Value)
{
    if (JumpComponent)
    {
        JumpComponent->StopJump();
    }
    else
    {
        // Fall back to default stop jump if component not available
        StopJumping();
    }
}

void APlatformerCharacter::HandleAttackInput(const FInputActionValue& Value)
{
    if (CombatComponent)
    {
        CombatComponent->StartAttack();
    }
}
```

## Component Implementation Guidelines

Each specialized component should handle a specific gameplay aspect:

### 1. Movement Component (UPlatformerMovementComponent)

Extend `UCharacterMovementComponent` to constrain movement to a 2.5D plane:

```cpp
// PlatformerMovementComponent.h
UCLASS()
class PLATFORMER_API UPlatformerMovementComponent : public UCharacterMovementComponent
{
    GENERATED_BODY()
    
public:
    UPlatformerMovementComponent();
    
    // Override movement functions
    virtual void PhysWalking(float deltaTime, int32 Iterations) override;
    virtual void PhysFlying(float deltaTime, int32 Iterations) override;
    virtual void PhysFalling(float deltaTime, int32 Iterations) override;
    
    // 2.5D specific functions
    void ConstrainLocationToPlane(FVector& Location);
    
protected:
    // Which axis to constrain (Y for side-scrolling)
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "2.5D Movement")
    TEnumAsByte<EAxis::Type> ConstraintAxis = EAxis::Y;
    
    // Depth position on the constraint axis
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "2.5D Movement")
    float DepthPosition = 0.0f;
};
```

### 2. Jump Component (UPlatformerJumpComponent)

Handle all jump-related mechanics:

```cpp
// PlatformerJumpComponent.h
UCLASS()
class PLATFORMER_API UPlatformerJumpComponent : public UActorComponent
{
    GENERATED_BODY()
    
public:
    UPlatformerJumpComponent();
    
    UFUNCTION(BlueprintCallable, Category = "Jump")
    void StartJump();
    
    UFUNCTION(BlueprintCallable, Category = "Jump")
    void StopJump();
    
    UFUNCTION(BlueprintCallable, Category = "Jump")
    bool CanDoubleJump() const;
    
    UFUNCTION(BlueprintCallable, Category = "Jump")
    void ResetJumpState();
    
protected:
    // Owner reference cached on BeginPlay
    UPROPERTY()
    ACharacter* CharacterOwner;
    
    // Number of jumps performed
    UPROPERTY(BlueprintReadOnly, Category = "Jump")
    int32 JumpCount;
    
    // Maximum number of jumps allowed
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Jump")
    int32 MaxJumpCount = 2;
    
    // Jump force override
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Jump")
    float JumpZVelocity = 700.0f;
    
    virtual void BeginPlay() override;
    virtual void TickComponent(float DeltaTime, enum ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction) override;
};
```

### 3. Combat Component (UPlatformerCombatComponent)

Handle attack mechanics:

```cpp
// PlatformerCombatComponent.h
UCLASS()
class PLATFORMER_API UPlatformerCombatComponent : public UActorComponent
{
    GENERATED_BODY()
    
public:
    UPlatformerCombatComponent();
    
    UFUNCTION(BlueprintCallable, Category = "Combat")
    void StartAttack();
    
    UFUNCTION(BlueprintCallable, Category = "Combat")
    void EndAttack();
    
    UFUNCTION(BlueprintCallable, Category = "Combat")
    bool IsAttacking() const;
    
protected:
    // Owner reference cached on BeginPlay
    UPROPERTY()
    ACharacter* CharacterOwner;
    
    // Attack animation montages
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Combat")
    TArray<UAnimMontage*> AttackMontages;
    
    // Current combo index
    UPROPERTY(BlueprintReadOnly, Category = "Combat")
    int32 ComboIndex = 0;
    
    // Whether an attack is in progress
    UPROPERTY(BlueprintReadOnly, Category = "Combat")
    bool bIsAttacking = false;
    
    // Time window for combo continuation
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Combat")
    float ComboWindowTime = 0.8f;
    
    virtual void BeginPlay() override;
    
    // Apply damage to targets
    UFUNCTION()
    void ApplyDamage();
};
```

### 4. Interaction Component (UPlatformerInteractionComponent)

Handle environment interaction:

```cpp
// PlatformerInteractionComponent.h
UCLASS()
class PLATFORMER_API UPlatformerInteractionComponent : public UActorComponent
{
    GENERATED_BODY()
    
public:
    UPlatformerInteractionComponent();
    
    UFUNCTION(BlueprintCallable, Category = "Interaction")
    void Interact();
    
    UFUNCTION(BlueprintCallable, Category = "Interaction")
    bool CanInteract() const;
    
protected:
    // Owner reference cached on BeginPlay
    UPROPERTY()
    ACharacter* CharacterOwner;
    
    // Interaction range
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Interaction")
    float InteractionRange = 100.0f;
    
    // Interaction channel
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Interaction")
    TEnumAsByte<ECollisionChannel> InteractionChannel = ECC_GameTraceChannel1;
    
    virtual void BeginPlay() override;
    virtual void TickComponent(float DeltaTime, enum ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction) override;
    
    // Find interactable objects
    TArray<AActor*> FindInteractableObjects() const;
};
```

## Integration with Gameplay Ability System

For ability-driven gameplay, implement the `IAbilitySystemInterface` and integrate with GAS:

```cpp
// PlatformerCharacter.cpp - Add to PossessedBy function
void APlatformerCharacter::PossessedBy(AController* NewController)
{
    Super::PossessedBy(NewController);
    
    // Initialize ability system
    if (AbilitySystemComponent)
    {
        AbilitySystemComponent->InitAbilityActorInfo(this, this);
        
        // Grant default abilities (implemented in child class)
        GrantDefaultAbilities();
    }
}

// Implement GetAbilitySystemComponent as required by IAbilitySystemInterface
UAbilitySystemComponent* APlatformerCharacter::GetAbilitySystemComponent() const
{
    return AbilitySystemComponent;
}
```

## Blueprint Extensions

Create a Blueprint child class to easily configure and extend the character:

1. In the Unreal Editor, right-click in the Content Browser
2. Select **Blueprint Class**
3. Choose your PlatformerCharacter C++ class
4. Name it BP_PlatformerCharacter
5. Configure:
   - Skeletal mesh and animation blueprint
   - Input actions and mapping context
   - Component properties
   - Animation montages
   - Default abilities

## Technical Considerations

### 2.5D Movement Constraints

For true 2.5D gameplay, you need to constrain movement to a plane:

```cpp
// In PlatformerMovementComponent.cpp
void UPlatformerMovementComponent::PhysWalking(float deltaTime, int32 Iterations)
{
    Super::PhysWalking(deltaTime, Iterations);
    
    // Constrain location after physics
    FVector NewLocation = UpdatedComponent->GetComponentLocation();
    ConstrainLocationToPlane(NewLocation);
    UpdatedComponent->SetWorldLocation(NewLocation);
}

void UPlatformerMovementComponent::ConstrainLocationToPlane(FVector& Location)
{
    // Force the constrained axis to our fixed depth
    if (ConstraintAxis == EAxis::X)
        Location.X = DepthPosition;
    else if (ConstraintAxis == EAxis::Y)
        Location.Y = DepthPosition;
    else if (ConstraintAxis == EAxis::Z)
        Location.Z = DepthPosition;
}
```

### Collision Considerations

For 2.5D gameplay with 3D visuals:
- Configure the collision capsule width to match character visuals
- Use simpler collision for fast iteration
- Consider using overlap events for interactive elements
- Configure trace channels for different interaction types

## Testing Your Character

Create a simple test level to validate core functionality:

1. Add platforms at different heights
2. Include obstacles for collision testing
3. Add slope platforms for testing ground movement
4. Configure the camera for proper 2.5D view
5. Test jump height and distances
6. Validate 2.5D constraints are working properly

## Success Criteria

Your character implementation is complete when:
- Character only moves along the intended 2.5D plane
- Basic movement feels responsive and smooth
- Jump mechanics work with the desired arc and height
- Animation triggering works properly
- Components can be extended independently
- Position adjustment works during animations
- Enhanced Input controls are responsive and intuitive

## Next Steps

After completing the base character:
1. Implement specialized movement abilities
2. Create combat animations and mechanics
3. Add environmental interaction functionality
4. Integrate with UI for player feedback
5. Implement checkpoint and respawn systems

Proceed to [Character Animation Setup](2_character_animation_setup.md) after completing the base implementation. 