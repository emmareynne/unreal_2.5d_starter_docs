# Interaction Component Implementation

This guide covers implementing the core interaction component for your 2.5D platformer character, handling detection and processing of interactive elements in the environment.

## Component Overview

The `UPlatformerInteractionComponent` is responsible for:
- Detecting interactable objects within range
- Handling player input for interactions
- Processing interaction requests
- Providing feedback about available interactions
- Managing active interactions

## Implementation Steps

### Step 1: Create the Interaction Component Class

```cpp
// PlatformerInteractionComponent.h
#pragma once

#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "PlatformerInteractionComponent.generated.h"

class IInteractable;

/**
 * Handles interaction with environmental objects and elements
 */
UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class PLATFORMER_API UPlatformerInteractionComponent : public UActorComponent
{
    GENERATED_BODY()
    
public:    
    UPlatformerInteractionComponent();
    
    // Begin ActorComponent interface
    virtual void BeginPlay() override;
    virtual void TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction) override;
    // End ActorComponent interface
    
    // Attempt to interact with closest interactable object
    UFUNCTION(BlueprintCallable, Category = "Interaction")
    bool TryInteract();
    
    // Check if there's an interactable object in range
    UFUNCTION(BlueprintPure, Category = "Interaction")
    bool CanInteract() const;
    
    // Get the closest interactable object
    UFUNCTION(BlueprintPure, Category = "Interaction")
    AActor* GetClosestInteractable() const;
    
    // Returns the current interaction state
    UFUNCTION(BlueprintPure, Category = "Interaction")
    bool IsInteracting() const { return bIsInteracting; }
    
    // End current interaction
    UFUNCTION(BlueprintCallable, Category = "Interaction")
    void EndInteraction();
    
protected:
    // Reference to the character owner
    UPROPERTY()
    class ACharacter* CharacterOwner;
    
    // Distance at which interactions can occur
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Interaction")
    float InteractionRange = 150.0f;
    
    // Trace channel to use for interaction detection
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Interaction")
    TEnumAsByte<ECollisionChannel> InteractionChannel = ECC_GameTraceChannel1;
    
    // Debug interaction detection
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Debug")
    bool bDebugInteraction = false;
    
    // Interaction state
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Interaction")
    bool bIsInteracting = false;
    
    // Currently interacting actor
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Interaction")
    AActor* CurrentInteractable = nullptr;

private:
    // Find all interactable objects in range
    TArray<AActor*> FindInteractablesInRange() const;
    
    // Trace line forward from character for direct interaction detection
    bool LineTraceForInteraction(FHitResult& HitResult) const;
    
    // Sort interactables by distance
    void SortInteractablesByDistance(TArray<AActor*>& Interactables) const;
};
```

### Step 2: Implement Core Functionality

```cpp
// PlatformerInteractionComponent.cpp
#include "PlatformerInteractionComponent.h"
#include "GameFramework/Character.h"
#include "DrawDebugHelpers.h"
#include "IInteractable.h"

UPlatformerInteractionComponent::UPlatformerInteractionComponent()
{
    PrimaryComponentTick.bCanEverTick = true;
}

void UPlatformerInteractionComponent::BeginPlay()
{
    Super::BeginPlay();

    // Cache owner reference
    CharacterOwner = Cast<ACharacter>(GetOwner());
}

void UPlatformerInteractionComponent::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
{
    Super::TickComponent(DeltaTime, TickType, ThisTickFunction);

    // Debug visualization
    if (bDebugInteraction)
    {
        TArray<AActor*> InteractableObjects = FindInteractablesInRange();
        for (AActor* Object : InteractableObjects)
        {
            DrawDebugSphere(GetWorld(), Object->GetActorLocation(), 50.0f, 8, FColor::Green, false, -1.0f, 0, 2.0f);
        }
        
        // Draw interaction range
        DrawDebugSphere(GetWorld(), CharacterOwner->GetActorLocation(), InteractionRange, 16, FColor::Blue, false, -1.0f, 0, 1.0f);
    }
}

bool UPlatformerInteractionComponent::TryInteract()
{
    if (bIsInteracting || !CharacterOwner)
    {
        return false;
    }

    // Get the closest interactable
    AActor* ClosestInteractable = GetClosestInteractable();
    if (!ClosestInteractable)
    {
        return false;
    }

    // Interface check
    IInteractable* Interactable = Cast<IInteractable>(ClosestInteractable);
    if (!Interactable)
    {
        return false;
    }

    // Start the interaction
    bIsInteracting = Interactable->Interact(CharacterOwner);
    if (bIsInteracting)
    {
        CurrentInteractable = ClosestInteractable;
    }

    return bIsInteracting;
}

bool UPlatformerInteractionComponent::CanInteract() const
{
    return !bIsInteracting && GetClosestInteractable() != nullptr;
}

AActor* UPlatformerInteractionComponent::GetClosestInteractable() const
{
    TArray<AActor*> InteractableObjects = FindInteractablesInRange();
    
    if (InteractableObjects.Num() == 0)
    {
        return nullptr;
    }
    
    // Sort by distance
    SortInteractablesByDistance(InteractableObjects);
    
    return InteractableObjects[0];
}

void UPlatformerInteractionComponent::EndInteraction()
{
    if (!bIsInteracting || !CurrentInteractable)
    {
        return;
    }

    // Call end interaction on the interactable
    IInteractable* Interactable = Cast<IInteractable>(CurrentInteractable);
    if (Interactable)
    {
        Interactable->EndInteract(CharacterOwner);
    }

    // Reset state
    bIsInteracting = false;
    CurrentInteractable = nullptr;
}

TArray<AActor*> UPlatformerInteractionComponent::FindInteractablesInRange() const
{
    TArray<AActor*> InteractableObjects;
    
    if (!CharacterOwner)
    {
        return InteractableObjects;
    }

    // First check line trace for direct interaction
    FHitResult HitResult;
    if (LineTraceForInteraction(HitResult))
    {
        AActor* HitActor = HitResult.GetActor();
        if (HitActor && HitActor->GetClass()->ImplementsInterface(UInteractable::StaticClass()))
        {
            InteractableObjects.Add(HitActor);
            return InteractableObjects;
        }
    }
    
    // Then check for overlapping actors within range
    TArray<AActor*> OverlappingActors;
    CharacterOwner->GetOverlappingActors(OverlappingActors, AActor::StaticClass());
    
    // Filter for interactable objects within range
    for (AActor* Actor : OverlappingActors)
    {
        if (Actor && Actor->GetClass()->ImplementsInterface(UInteractable::StaticClass()))
        {
            float Distance = FVector::Distance(CharacterOwner->GetActorLocation(), Actor->GetActorLocation());
            if (Distance <= InteractionRange)
            {
                InteractableObjects.Add(Actor);
            }
        }
    }
    
    return InteractableObjects;
}

bool UPlatformerInteractionComponent::LineTraceForInteraction(FHitResult& HitResult) const
{
    if (!CharacterOwner)
    {
        return false;
    }

    // Calculate trace direction and end point
    FVector StartLocation = CharacterOwner->GetActorLocation();
    FVector ForwardVector = CharacterOwner->GetActorForwardVector();
    FVector EndLocation = StartLocation + (ForwardVector * InteractionRange);

    // Setup trace parameters
    FCollisionQueryParams QueryParams;
    QueryParams.AddIgnoredActor(CharacterOwner);
    
    // Perform trace
    bool bHit = GetWorld()->LineTraceSingleByChannel(
        HitResult,
        StartLocation,
        EndLocation,
        InteractionChannel,
        QueryParams
    );
    
    if (bDebugInteraction)
    {
        DrawDebugLine(GetWorld(), StartLocation, EndLocation, bHit ? FColor::Green : FColor::Red, false, -1.0f, 0, 2.0f);
    }
    
    return bHit;
}

void UPlatformerInteractionComponent::SortInteractablesByDistance(TArray<AActor*>& Interactables) const
{
    if (!CharacterOwner || Interactables.Num() <= 1)
    {
        return;
    }

    FVector OwnerLocation = CharacterOwner->GetActorLocation();
    
    Interactables.Sort([OwnerLocation](const AActor& A, const AActor& B) {
        float DistanceA = FVector::DistSquared(A.GetActorLocation(), OwnerLocation);
        float DistanceB = FVector::DistSquared(B.GetActorLocation(), OwnerLocation);
        return DistanceA < DistanceB;
    });
}
```

### Step 3: Integrate with Character Class

Now update your `PlatformerCharacter` class to handle interaction input and provide UI feedback:

```cpp
// Add to PlatformerCharacter.h
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Input")
class UInputAction* InteractAction;

// Input handling method
void HandleInteractInput(const struct FInputActionValue& Value);

// Add to PlatformerCharacter.cpp in SetupPlayerInputComponent
if (EnhancedInputComponent)
{
    // ... existing bindings ...
    
    // Add interaction binding
    EnhancedInputComponent->BindAction(InteractAction, ETriggerEvent::Started, this, &APlatformerCharacter::HandleInteractInput);
}

// Implement the handler
void APlatformerCharacter::HandleInteractInput(const FInputActionValue& Value)
{
    if (InteractionComponent)
    {
        if (InteractionComponent->IsInteracting())
        {
            InteractionComponent->EndInteraction();
        }
        else
        {
            InteractionComponent->TryInteract();
        }
    }
}
```

## Testing the Interaction Component

To test the interaction component:

1. Create a test level with some basic interactable objects
2. Add debugging visualization to verify detection range
3. Test interaction with different objects
4. Verify proper handling of interaction states
5. Test edge cases like interrupted interactions

## Next Steps

After implementing the interaction component:

1. Proceed to [Interactable Interface](3b_interactable_interface.md) implementation
2. Create the base interactable object types
3. Implement specific environmental interactions
4. Add feedback through UI elements 