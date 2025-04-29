# Interactable Interface Implementation

This guide covers implementing the interactable interface for your 2.5D platformer, which allows any object to be interacted with by the player character.

## Interface Overview

The `IInteractable` interface provides a common set of methods that all interactable objects must implement:
- Interact - Called when player initiates an interaction
- End Interact - Called when player ends an interaction
- Can Interact - Determines if interaction is possible
- Get Interaction Text - Provides context-specific UI text

This interface enables a uniform way of handling different types of interactions while allowing for object-specific behaviors.

## Implementation Steps

### Step 1: Create the Interface

```cpp
// IInteractable.h
#pragma once

#include "CoreMinimal.h"
#include "UObject/Interface.h"
#include "IInteractable.generated.h"

// This class does not need to be modified.
UINTERFACE(MinimalAPI, Blueprintable)
class UInteractable : public UInterface
{
    GENERATED_BODY()
};

/**
 * Interface for all objects that can be interacted with by the player
 */
class PLATFORMER_API IInteractable
{
    GENERATED_BODY()

public:
    // Called when player initiates interaction
    UFUNCTION(BlueprintCallable, BlueprintNativeEvent, Category = "Interaction")
    bool Interact(class AActor* Interactor);
    
    // Called when player ends interaction
    UFUNCTION(BlueprintCallable, BlueprintNativeEvent, Category = "Interaction")
    void EndInteract(class AActor* Interactor);
    
    // Whether interaction is currently available
    UFUNCTION(BlueprintCallable, BlueprintNativeEvent, Category = "Interaction")
    bool CanInteract(class AActor* Interactor) const;
    
    // Get text to display for interaction prompt
    UFUNCTION(BlueprintCallable, BlueprintNativeEvent, Category = "Interaction")
    FText GetInteractionText(class AActor* Interactor) const;
    
    // Get interaction icon (optional)
    UFUNCTION(BlueprintCallable, BlueprintNativeEvent, Category = "Interaction")
    class UTexture2D* GetInteractionIcon(class AActor* Interactor) const;
};
```

### Step 2: Create a Base Interactable Class

While not required, a base class can simplify implementation of common interactable behaviors:

```cpp
// BaseInteractableActor.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "IInteractable.h"
#include "BaseInteractableActor.generated.h"

UCLASS(Abstract)
class PLATFORMER_API ABaseInteractableActor : public AActor, public IInteractable
{
    GENERATED_BODY()
    
public:    
    ABaseInteractableActor();

    // IInteractable interface
    virtual bool Interact_Implementation(AActor* Interactor) override;
    virtual void EndInteract_Implementation(AActor* Interactor) override;
    virtual bool CanInteract_Implementation(AActor* Interactor) const override;
    virtual FText GetInteractionText_Implementation(AActor* Interactor) const override;
    virtual UTexture2D* GetInteractionIcon_Implementation(AActor* Interactor) const override;
    // End IInteractable interface
    
protected:
    virtual void BeginPlay() override;
    
    // Default interaction text
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Interaction")
    FText InteractionText;
    
    // Default interaction icon
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Interaction")
    UTexture2D* InteractionIcon;
    
    // Whether interaction is enabled
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Interaction")
    bool bInteractionEnabled = true;
    
    // Current interacting actor
    UPROPERTY(BlueprintReadOnly, Category = "Interaction")
    AActor* CurrentInteractor = nullptr;
    
    // Whether this object is currently being interacted with
    UPROPERTY(BlueprintReadOnly, Category = "Interaction")
    bool bIsBeingInteracted = false;
    
    // Called when interaction starts (for child classes to override)
    UFUNCTION(BlueprintImplementableEvent, Category = "Interaction")
    void OnInteractionStarted(AActor* Interactor);
    
    // Called when interaction ends (for child classes to override)
    UFUNCTION(BlueprintImplementableEvent, Category = "Interaction")
    void OnInteractionEnded(AActor* Interactor);
};
```

```cpp
// BaseInteractableActor.cpp
#include "BaseInteractableActor.h"

ABaseInteractableActor::ABaseInteractableActor()
{
    // Set default values
    PrimaryActorTick.bCanEverTick = true;
    
    // Set default interaction text
    InteractionText = FText::FromString("Interact");
}

void ABaseInteractableActor::BeginPlay()
{
    Super::BeginPlay();
}

bool ABaseInteractableActor::Interact_Implementation(AActor* Interactor)
{
    if (!bInteractionEnabled || bIsBeingInteracted)
    {
        return false;
    }
    
    bIsBeingInteracted = true;
    CurrentInteractor = Interactor;
    
    // Notify blueprint
    OnInteractionStarted(Interactor);
    
    return true;
}

void ABaseInteractableActor::EndInteract_Implementation(AActor* Interactor)
{
    if (Interactor != CurrentInteractor)
    {
        return;
    }
    
    bIsBeingInteracted = false;
    
    // Notify blueprint
    OnInteractionEnded(Interactor);
    
    CurrentInteractor = nullptr;
}

bool ABaseInteractableActor::CanInteract_Implementation(AActor* Interactor) const
{
    return bInteractionEnabled && !bIsBeingInteracted;
}

FText ABaseInteractableActor::GetInteractionText_Implementation(AActor* Interactor) const
{
    return InteractionText;
}

UTexture2D* ABaseInteractableActor::GetInteractionIcon_Implementation(AActor* Interactor) const
{
    return InteractionIcon;
}
```

### Step 3: Update Interaction Component to Use Interface

Ensure the Interaction Component is properly using the interface:

```cpp
// Update PlatformerInteractionComponent.cpp
#include "IInteractable.h"

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
    if (ClosestInteractable->GetClass()->ImplementsInterface(UInteractable::StaticClass()))
    {
        IInteractable* Interactable = Cast<IInteractable>(ClosestInteractable);
        
        // Check if we can interact
        if (Interactable->CanInteract(CharacterOwner))
        {
            // Start the interaction
            bIsInteracting = Interactable->Interact(CharacterOwner);
            if (bIsInteracting)
            {
                CurrentInteractable = ClosestInteractable;
            }
            return bIsInteracting;
        }
    }

    return false;
}
```

### Step 4: Create Blueprint Support

Make the interface usable from Blueprints:

1. Create a Blueprint Function Library for interaction helpers:

```cpp
// InteractionFunctionLibrary.h
#pragma once

#include "CoreMinimal.h"
#include "Kismet/BlueprintFunctionLibrary.h"
#include "InteractionFunctionLibrary.generated.h"

/**
 * Blueprint function library for interaction-related utilities
 */
UCLASS()
class PLATFORMER_API UInteractionFunctionLibrary : public UBlueprintFunctionLibrary
{
    GENERATED_BODY()
    
public:
    // Check if an actor implements the interactable interface
    UFUNCTION(BlueprintPure, Category = "Interaction")
    static bool IsInteractable(AActor* Actor);
    
    // Get interaction text for an actor
    UFUNCTION(BlueprintPure, Category = "Interaction")
    static FText GetInteractionText(AActor* Actor, AActor* Interactor);
    
    // Get interaction icon for an actor
    UFUNCTION(BlueprintPure, Category = "Interaction")
    static UTexture2D* GetInteractionIcon(AActor* Actor, AActor* Interactor);
};
```

```cpp
// InteractionFunctionLibrary.cpp
#include "InteractionFunctionLibrary.h"
#include "IInteractable.h"

bool UInteractionFunctionLibrary::IsInteractable(AActor* Actor)
{
    if (!Actor)
    {
        return false;
    }
    
    return Actor->GetClass()->ImplementsInterface(UInteractable::StaticClass());
}

FText UInteractionFunctionLibrary::GetInteractionText(AActor* Actor, AActor* Interactor)
{
    if (!Actor || !Interactor)
    {
        return FText::GetEmpty();
    }
    
    if (Actor->GetClass()->ImplementsInterface(UInteractable::StaticClass()))
    {
        IInteractable* Interactable = Cast<IInteractable>(Actor);
        return Interactable->GetInteractionText(Interactor);
    }
    
    return FText::GetEmpty();
}

UTexture2D* UInteractionFunctionLibrary::GetInteractionIcon(AActor* Actor, AActor* Interactor)
{
    if (!Actor || !Interactor)
    {
        return nullptr;
    }
    
    if (Actor->GetClass()->ImplementsInterface(UInteractable::StaticClass()))
    {
        IInteractable* Interactable = Cast<IInteractable>(Actor);
        return Interactable->GetInteractionIcon(Interactor);
    }
    
    return nullptr;
}
```

## Example Implementation: Interactable Button

Here's an example of a simple interactable button:

```cpp
// InteractableButton.h
#pragma once

#include "CoreMinimal.h"
#include "BaseInteractableActor.h"
#include "InteractableButton.generated.h"

DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnButtonPressed);

UCLASS()
class PLATFORMER_API AInteractableButton : public ABaseInteractableActor
{
    GENERATED_BODY()
    
public:
    AInteractableButton();
    
    // Override interact to implement button-specific behavior
    virtual bool Interact_Implementation(AActor* Interactor) override;
    
    // Event fired when button is pressed
    UPROPERTY(BlueprintAssignable, Category = "Button")
    FOnButtonPressed OnButtonPressed;
    
protected:
    // Visual component
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    class UStaticMeshComponent* ButtonMesh;
    
    // Sound played when pressed
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Button")
    class USoundBase* PressSound;
    
    // Whether button can be pressed multiple times
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Button")
    bool bCanPressMultipleTimes = true;
    
    // Whether the button is currently pressed
    UPROPERTY(BlueprintReadOnly, Category = "Button")
    bool bIsPressed = false;
    
    // Reset button after delay
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Button")
    float ResetDelay = 1.0f;
    
    // Handle button press
    UFUNCTION(BlueprintCallable, Category = "Button")
    void PressButton(AActor* Presser);
    
    // Handle button reset
    UFUNCTION()
    void ResetButton();
};
```

```cpp
// InteractableButton.cpp
#include "InteractableButton.h"
#include "Components/StaticMeshComponent.h"
#include "Kismet/GameplayStatics.h"
#include "TimerManager.h"

AInteractableButton::AInteractableButton()
{
    // Create button mesh
    ButtonMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("ButtonMesh"));
    SetRootComponent(ButtonMesh);
    
    // Set default values
    InteractionText = FText::FromString("Press Button");
}

bool AInteractableButton::Interact_Implementation(AActor* Interactor)
{
    if (!bInteractionEnabled || (bIsPressed && !bCanPressMultipleTimes))
    {
        return false;
    }
    
    // Press the button
    PressButton(Interactor);
    
    // Call parent implementation to handle state
    return Super::Interact_Implementation(Interactor);
}

void AInteractableButton::PressButton(AActor* Presser)
{
    bIsPressed = true;
    
    // Play sound if set
    if (PressSound)
    {
        UGameplayStatics::PlaySoundAtLocation(this, PressSound, GetActorLocation());
    }
    
    // Broadcast event
    OnButtonPressed.Broadcast();
    
    // Visual feedback (can be expanded in blueprint)
    ButtonMesh->SetRelativeLocation(FVector(0.0f, 0.0f, -5.0f));
    
    // Set timer to reset button if allowed
    if (bCanPressMultipleTimes && ResetDelay > 0.0f)
    {
        FTimerHandle ResetTimerHandle;
        GetWorldTimerManager().SetTimer(ResetTimerHandle, this, &AInteractableButton::ResetButton, ResetDelay, false);
    }
}

void AInteractableButton::ResetButton()
{
    bIsPressed = false;
    
    // Visual feedback
    ButtonMesh->SetRelativeLocation(FVector(0.0f, 0.0f, 0.0f));
}
```

## Testing the Interface

Test your interactable interface with these steps:

1. Create a Blueprint class inheriting from `ABaseInteractableActor`
2. Override key interface functions as needed
3. Place it in a test level
4. Test interaction using the previously implemented interaction component
5. Verify interaction text and icons display correctly
6. Check edge cases like interrupted interactions

## Next Steps

After implementing the interactable interface:

1. Create common interactable object types (levers, collectibles, etc.)
2. Implement [Position Adjustment System](3c_position_adjustment.md) for precise interactions
3. Add visual feedback for interactable objects
4. Design UI elements to show interaction prompts 