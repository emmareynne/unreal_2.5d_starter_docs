# Position Adjustment System

This guide covers implementing a position adjustment system that allows precise character positioning during interactions with objects in your 2.5D platformer.

## System Overview

The position adjustment system:
- Places the character at the correct location for interactions
- Smoothly interpolates between positions
- Allows for object-specific positioning
- Integrates with the animation system

## Implementation Approach

We'll use the Timeline component already added to the character class in our base implementation.

### Step 1: Update the Character Class

Make sure you have these functions in the PlatformerCharacter class:

```cpp
// Already implemented in character_base_implementation.md:

// Position adjustment functions
UFUNCTION(BlueprintCallable, Category = "Animation")
void StartPositionAdjustment(FVector TargetPosition, float Duration = 0.2f);

UFUNCTION()
void UpdatePositionAdjustment(float Alpha);

UFUNCTION()
void FinishPositionAdjustment();
```

### Step 2: Create a Position Adjustment Component

For more complex objects, create a component that defines interaction positions:

```cpp
// InteractionPositionComponent.h
#pragma once

#include "CoreMinimal.h"
#include "Components/SceneComponent.h"
#include "InteractionPositionComponent.generated.h"

/**
 * Component that defines a position for character interaction
 */
UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class PLATFORMER_API UInteractionPositionComponent : public USceneComponent
{
    GENERATED_BODY()
    
public:    
    UInteractionPositionComponent();
    
    // Position the character for interaction
    UFUNCTION(BlueprintCallable, Category = "Interaction")
    bool PositionCharacter(class ACharacter* Character, float BlendTime = 0.2f);
    
    // Check if character is close enough to use this position
    UFUNCTION(BlueprintPure, Category = "Interaction")
    bool IsCharacterInRange(class ACharacter* Character, float MaxDistance = 200.0f) const;
    
protected:
    // Whether to rotate character to match this component's rotation
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Interaction")
    bool bMatchRotation = true;
    
    // Whether to snap character to ground after positioning
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Interaction")
    bool bSnapToGround = true;
    
    // Debug visualization
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Debug")
    bool bDebugDraw = false;
    
    // Visualize the position
    virtual void TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction) override;
};
```

```cpp
// InteractionPositionComponent.cpp
#include "InteractionPositionComponent.h"
#include "GameFramework/Character.h"
#include "DrawDebugHelpers.h"
#include "PlatformerCharacter.h"

UInteractionPositionComponent::UInteractionPositionComponent()
{
    PrimaryComponentTick.bCanEverTick = true;
}

bool UInteractionPositionComponent::PositionCharacter(ACharacter* Character, float BlendTime)
{
    if (!Character)
    {
        return false;
    }
    
    // Get target position
    FVector TargetPosition = GetComponentLocation();
    
    // Adjust for ground if needed
    if (bSnapToGround)
    {
        FHitResult HitResult;
        FVector Start = TargetPosition;
        FVector End = Start - FVector(0, 0, 100.0f);
        
        if (GetWorld()->LineTraceSingleByChannel(HitResult, Start, End, ECC_Visibility))
        {
            TargetPosition = HitResult.Location;
            
            // Adjust for character height
            TargetPosition.Z += Character->GetCapsuleComponent()->GetScaledCapsuleHalfHeight();
        }
    }
    
    // Set rotation if needed
    if (bMatchRotation)
    {
        Character->SetActorRotation(GetComponentRotation());
    }
    
    // Use character's position adjustment system
    APlatformerCharacter* PlatformerChar = Cast<APlatformerCharacter>(Character);
    if (PlatformerChar)
    {
        PlatformerChar->StartPositionAdjustment(TargetPosition, BlendTime);
        return true;
    }
    
    // Fallback for non-platformer characters
    Character->SetActorLocation(TargetPosition);
    return true;
}

bool UInteractionPositionComponent::IsCharacterInRange(ACharacter* Character, float MaxDistance) const
{
    if (!Character)
    {
        return false;
    }
    
    float Distance = FVector::Dist(Character->GetActorLocation(), GetComponentLocation());
    return Distance <= MaxDistance;
}

void UInteractionPositionComponent::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
{
    Super::TickComponent(DeltaTime, TickType, ThisTickFunction);
    
    if (bDebugDraw)
    {
        // Draw position marker
        DrawDebugSphere(GetWorld(), GetComponentLocation(), 20.0f, 8, FColor::Yellow, false, -1.0f, 0, 2.0f);
        
        // Draw forward direction
        DrawDebugLine(
            GetWorld(),
            GetComponentLocation(),
            GetComponentLocation() + GetForwardVector() * 50.0f,
            FColor::Blue,
            false,
            -1.0f,
            0,
            2.0f
        );
    }
}
```

### Step 3: Integrate with Interactable Objects

Update BaseInteractableActor to use the position component:

```cpp
// Add to BaseInteractableActor.h
#include "InteractionPositionComponent.h"

// In protected section:
// Interaction position component
UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
UInteractionPositionComponent* InteractionPosition;

// In constructor:
InteractionPosition = CreateDefaultSubobject<UInteractionPositionComponent>(TEXT("InteractionPosition"));
InteractionPosition->SetupAttachment(RootComponent);

// Add to Interact_Implementation:
// Position character if needed
if (InteractionPosition && Interactor->IsA(ACharacter::StaticClass()))
{
    InteractionPosition->PositionCharacter(Cast<ACharacter>(Interactor));
}
```

### Step 4: Animation Integration

For animation-driven positioning, use a custom Animation Notify:

```cpp
// AnimNotify_PositionAdjust.h
UCLASS()
class PLATFORMER_API UAnimNotify_PositionAdjust : public UAnimNotify
{
    GENERATED_BODY()
    
public:
    // The relative offset to apply
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Position")
    FVector PositionOffset;
    
    // Duration of adjustment
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Position")
    float AdjustmentDuration = 0.2f;
    
    // Whether to apply in world space
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Position")
    bool bWorldSpace = false;
    
    virtual void Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation) override;
};
```

```cpp
// AnimNotify_PositionAdjust.cpp
#include "AnimNotify_PositionAdjust.h"
#include "PlatformerCharacter.h"

void UAnimNotify_PositionAdjust::Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation)
{
    if (!MeshComp || !MeshComp->GetOwner())
    {
        return;
    }
    
    APlatformerCharacter* Character = Cast<APlatformerCharacter>(MeshComp->GetOwner());
    if (!Character)
    {
        return;
    }
    
    // Calculate target position
    FVector TargetPosition;
    if (bWorldSpace)
    {
        TargetPosition = PositionOffset;
    }
    else
    {
        // Apply offset relative to character
        TargetPosition = Character->GetActorLocation() + 
            Character->GetActorRotation().RotateVector(PositionOffset);
    }
    
    // Start position adjustment
    Character->StartPositionAdjustment(TargetPosition, AdjustmentDuration);
}
```

## Example: Creating a Ledge Grab

Implementing a ledge grab mechanic using the position adjustment system:

```cpp
// In PlatformerCharacter.h
// Ledge grab detection
UFUNCTION(BlueprintCallable, Category = "Movement")
bool DetectLedge(FVector& OutLedgePosition, FVector& OutWallNormal);

// Ledge grab functionality
UFUNCTION(BlueprintCallable, Category = "Movement")
bool GrabLedge();

// In PlatformerCharacter.cpp
bool APlatformerCharacter::DetectLedge(FVector& OutLedgePosition, FVector& OutWallNormal)
{
    if (!GetCharacterMovement() || !GetCharacterMovement()->IsFalling())
    {
        return false;
    }
    
    // Forward trace to detect wall
    FHitResult WallHit;
    FVector Start = GetActorLocation();
    FVector End = Start + GetActorForwardVector() * 50.0f;
    
    FCollisionQueryParams QueryParams;
    QueryParams.AddIgnoredActor(this);
    
    // Find wall
    if (GetWorld()->LineTraceSingleByChannel(WallHit, Start, End, ECC_Visibility, QueryParams))
    {
        OutWallNormal = WallHit.Normal;
        
        // Trace up to find ledge
        FVector LedgeTraceStart = WallHit.Location + FVector(0, 0, 50.0f);
        FVector LedgeTraceEnd = LedgeTraceStart + WallHit.Normal * -50.0f;
        
        FHitResult LedgeHit;
        if (!GetWorld()->LineTraceSingleByChannel(LedgeHit, LedgeTraceStart, LedgeTraceEnd, ECC_Visibility, QueryParams))
        {
            // If no hit above, we found a ledge
            OutLedgePosition = FVector(WallHit.Location.X, WallHit.Location.Y, LedgeTraceStart.Z);
            return true;
        }
    }
    
    return false;
}

bool APlatformerCharacter::GrabLedge()
{
    FVector LedgePosition, WallNormal;
    
    if (!DetectLedge(LedgePosition, WallNormal))
    {
        return false;
    }
    
    // Calculate grab position (slightly away from ledge)
    FVector GrabPosition = LedgePosition + WallNormal * -30.0f;
    
    // Set rotation to face away from wall
    FRotator NewRotation = (-WallNormal).Rotation();
    SetActorRotation(NewRotation);
    
    // Stop movement
    GetCharacterMovement()->StopMovementImmediately();
    
    // Adjust position
    StartPositionAdjustment(GrabPosition, 0.2f);
    
    // Play ledge grab animation
    if (LedgeGrabMontage)
    {
        PlayAnimMontage(LedgeGrabMontage);
    }
    
    return true;
}
```

## Testing Position Adjustment

Test your position adjustment system with these steps:

1. Add an InteractionPositionComponent to an interactable object
2. Position and rotate the component to define the interaction pose
3. Enable debug visualization to see the position in-editor
4. Test interaction to verify smooth positioning
5. Try different blend times and distances

## Next Steps

After implementing the position adjustment system:

1. Create [Environmental Object Types](3d_environmental_objects.md) that use the system
2. Implement more advanced interaction animations
3. Add additional adjustment parameters like character rotation
4. Fine-tune the interpolation curve for more natural movement 