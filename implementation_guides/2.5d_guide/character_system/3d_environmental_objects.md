# Environmental Object Types

This guide covers implementing common interactive environmental objects for your 2.5D platformer using the interaction system.

## Overview

Environmental objects enhance your platformer gameplay by providing:
- Interactive elements for puzzle solving
- Traversal mechanics for level design
- Collectibles and power-ups
- Narrative elements through interaction

Each object type will use our interaction interface and position adjustment system.

## Interactable Object Types

### 1. Basic Switch/Button

A simple interactable that triggers events when pressed.

```cpp
// PlatformerButton.h
#pragma once

#include "CoreMinimal.h"
#include "BaseInteractableActor.h"
#include "PlatformerButton.generated.h"

DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnButtonActivated);
DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnButtonDeactivated);

UCLASS()
class PLATFORMER_API APlatformerButton : public ABaseInteractableActor
{
    GENERATED_BODY()
    
public:
    APlatformerButton();
    
    virtual bool Interact_Implementation(AActor* Interactor) override;
    
    // Events
    UPROPERTY(BlueprintAssignable, Category = "Button")
    FOnButtonActivated OnButtonActivated;
    
    UPROPERTY(BlueprintAssignable, Category = "Button")
    FOnButtonDeactivated OnButtonDeactivated;
    
protected:
    // Visual component
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    class UStaticMeshComponent* ButtonMesh;
    
    // Whether button is currently pressed
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Button")
    bool bIsActivated = false;
    
    // Whether button stays activated when pressed
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Button")
    bool bToggleable = false;
    
    // Auto-reset timer (0 = no auto-reset)
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Button")
    float AutoResetTime = 0.0f;
    
    // Activation and deactivation handlers
    UFUNCTION(BlueprintCallable, Category = "Button")
    void Activate(AActor* Activator);
    
    UFUNCTION(BlueprintCallable, Category = "Button")
    void Deactivate();
    
private:
    // Handle auto-reset
    FTimerHandle ResetTimerHandle;
};
```

### 2. Moving Platform

A platform that moves between points when triggered.

```cpp
// MovingPlatform.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "MovingPlatform.generated.h"

UCLASS()
class PLATFORMER_API AMovingPlatform : public AActor
{
    GENERATED_BODY()
    
public:
    AMovingPlatform();
    
    virtual void BeginPlay() override;
    virtual void Tick(float DeltaTime) override;
    
    // Activate platform movement
    UFUNCTION(BlueprintCallable, Category = "Platform")
    void Activate();
    
    // Deactivate platform movement
    UFUNCTION(BlueprintCallable, Category = "Platform")
    void Deactivate();
    
    // Toggle activation
    UFUNCTION(BlueprintCallable, Category = "Platform")
    void Toggle();
    
protected:
    // Platform mesh
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    class UStaticMeshComponent* PlatformMesh;
    
    // Waypoints for platform movement
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Movement")
    TArray<FVector> Waypoints;
    
    // Movement speed
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Movement")
    float MovementSpeed = 200.0f;
    
    // Wait time at waypoints
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Movement")
    float WaitTime = 0.5f;
    
    // Whether platform moves automatically when game starts
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Movement")
    bool bAutoStart = false;
    
    // Whether platform is looping through waypoints
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Movement")
    bool bLooping = true;
    
private:
    // Whether platform is active
    bool bIsActive = false;
    
    // Current waypoint index
    int32 CurrentWaypointIndex = 0;
    
    // Original location (for relative waypoints)
    FVector OriginalLocation;
    
    // Whether platform is waiting at a waypoint
    bool bIsWaiting = false;
    
    // Timer for waiting at waypoints
    FTimerHandle WaitTimerHandle;
    
    // Move to next waypoint
    void MoveToNextWaypoint();
    
    // Called when wait time expires
    void OnWaitTimeEnded();
};
```

### 3. Collectible Item

Interactable collectible for the player to gather.

```cpp
// Collectible.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "IInteractable.h"
#include "Collectible.generated.h"

UCLASS()
class PLATFORMER_API ACollectible : public AActor, public IInteractable
{
    GENERATED_BODY()
    
public:
    ACollectible();
    
    // IInteractable interface
    virtual bool Interact_Implementation(AActor* Interactor) override;
    virtual bool CanInteract_Implementation(AActor* Interactor) const override;
    virtual FText GetInteractionText_Implementation(AActor* Interactor) const override;
    // End IInteractable interface
    
    // Overlap event handler
    UFUNCTION()
    void OnOverlapBegin(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, 
                        UPrimitiveComponent* OtherComp, int32 OtherBodyIndex,
                        bool bFromSweep, const FHitResult& SweepResult);
    
protected:
    // Visuals
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    class UStaticMeshComponent* CollectibleMesh;
    
    // Collision
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    class USphereComponent* CollectionSphere;
    
    // Particle effect on collection
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Effects")
    class UParticleSystem* CollectionEffect;
    
    // Sound on collection
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Effects")
    class USoundBase* CollectionSound;
    
    // Value of the collectible
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Collectible")
    int32 Value = 1;
    
    // Whether collectible can be collected with interact button
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Collectible")
    bool bRequiresInteraction = false;
    
    // Collect the item
    UFUNCTION(BlueprintCallable, Category = "Collectible")
    virtual void Collect(AActor* Collector);
    
    // Called when collected (for blueprint)
    UFUNCTION(BlueprintImplementableEvent, Category = "Collectible")
    void OnCollected(AActor* Collector);
};
```

### 4. Interactive Door

A door that can be opened and closed by the player.

```cpp
// InteractiveDoor.h
#pragma once

#include "CoreMinimal.h"
#include "BaseInteractableActor.h"
#include "InteractiveDoor.generated.h"

UCLASS()
class PLATFORMER_API AInteractiveDoor : public ABaseInteractableActor
{
    GENERATED_BODY()
    
public:
    AInteractiveDoor();
    
    virtual bool Interact_Implementation(AActor* Interactor) override;
    virtual void Tick(float DeltaTime) override;
    
    // Open the door
    UFUNCTION(BlueprintCallable, Category = "Door")
    void Open();
    
    // Close the door
    UFUNCTION(BlueprintCallable, Category = "Door")
    void Close();
    
    // Check if door is open
    UFUNCTION(BlueprintPure, Category = "Door")
    bool IsOpen() const { return bIsOpen; }
    
protected:
    // Door mesh
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    class UStaticMeshComponent* DoorMesh;
    
    // Door frame mesh
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    class UStaticMeshComponent* FrameMesh;
    
    // Is the door currently open?
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Door")
    bool bIsOpen = false;
    
    // Is the door currently moving?
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Door")
    bool bIsMoving = false;
    
    // Door opening style
    UENUM(BlueprintType)
    enum class EDoorOpenStyle : uint8
    {
        Rotate,
        Slide,
        Lift
    };
    
    // How the door opens
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Door")
    EDoorOpenStyle OpenStyle = EDoorOpenStyle::Rotate;
    
    // Open amount (degrees for rotation, units for slide/lift)
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Door")
    float OpenAmount = 90.0f;
    
    // Opening speed
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Door")
    float OpenSpeed = 2.0f;
    
    // Door opening sound
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Door")
    class USoundBase* OpenSound;
    
    // Door closing sound
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Door")
    class USoundBase* CloseSound;
    
    // Is door locked?
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Door")
    bool bIsLocked = false;
    
    // Required key (if locked)
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Door", meta = (EditCondition = "bIsLocked"))
    FName RequiredKeyName;
    
private:
    // Initial transform for door
    FVector InitialLocation;
    FRotator InitialRotation;
    
    // Current interpolation alpha
    float CurrentAlpha = 0.0f;
    
    // Target alpha (0 = closed, 1 = open)
    float TargetAlpha = 0.0f;
    
    // Update door position or rotation
    void UpdateDoorTransform();
    
    // Check if player has required key
    bool HasRequiredKey(AActor* Interactor);
};
```

### 5. Pressure Plate

A plate that activates when stepped on.

```cpp
// PressurePlate.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "PressurePlate.generated.h"

DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnPlateActivated);
DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnPlateDeactivated);

UCLASS()
class PLATFORMER_API APressurePlate : public AActor
{
    GENERATED_BODY()
    
public:
    APressurePlate();
    
    virtual void BeginPlay() override;
    
    // Events
    UPROPERTY(BlueprintAssignable, Category = "Pressure Plate")
    FOnPlateActivated OnPlateActivated;
    
    UPROPERTY(BlueprintAssignable, Category = "Pressure Plate")
    FOnPlateDeactivated OnPlateDeactivated;
    
protected:
    // Plate mesh
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    class UStaticMeshComponent* PlateMesh;
    
    // Trigger volume
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    class UBoxComponent* TriggerVolume;
    
    // Base mesh
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    class UStaticMeshComponent* BaseMesh;
    
    // Activation sound
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Pressure Plate")
    class USoundBase* ActivationSound;
    
    // Deactivation sound
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Pressure Plate")
    class USoundBase* DeactivationSound;
    
    // How much the plate moves when pressed
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Pressure Plate")
    float PressDepth = 5.0f;
    
    // Weight required to activate (0 = any weight)
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Pressure Plate")
    float RequiredWeight = 0.0f;
    
    // Whether plate stays activated when stepped off
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Pressure Plate")
    bool bStaysActivated = false;
    
    // Begin overlap event handler
    UFUNCTION()
    void OnOverlapBegin(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, 
                        UPrimitiveComponent* OtherComp, int32 OtherBodyIndex,
                        bool bFromSweep, const FHitResult& SweepResult);
    
    // End overlap event handler
    UFUNCTION()
    void OnOverlapEnd(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, 
                      UPrimitiveComponent* OtherComp, int32 OtherBodyIndex);
    
    // Count overlapping actors
    UFUNCTION()
    float CalculateTotalWeight() const;
    
    // Activate the plate
    UFUNCTION(BlueprintCallable, Category = "Pressure Plate")
    void Activate();
    
    // Deactivate the plate
    UFUNCTION(BlueprintCallable, Category = "Pressure Plate")
    void Deactivate();
    
private:
    // Is the plate currently activated?
    bool bIsActivated = false;
    
    // Initial plate location
    FVector InitialPlateLocation;
    
    // Overlapping actors
    TArray<AActor*> OverlappingActors;
};
```

## Creating Blueprint Variants

For each of these C++ classes, you can create Blueprint variants with specific configurations:

1. Create a Blueprint class inheriting from the C++ base class
2. Set up visual components (meshes, materials, effects)
3. Configure behavior parameters
4. Add custom logic in blueprint event graphs
5. Create variants for different visual and gameplay needs

## Integration with Level Design

When placing interactive objects:

1. **Connect Objects**: Use event dispatchers to link interactive objects
   ```
   // Example connection in level blueprint:
   PressurePlate.OnPlateActivated.AddDynamic(MovingPlatform, &AMovingPlatform::Activate);
   PressurePlate.OnPlateDeactivated.AddDynamic(MovingPlatform, &AMovingPlatform::Deactivate);
   ```

2. **Puzzle Design**: Create logical sequences of interactions
   ```
   // Example of sequence in level blueprint:
   Button1.OnButtonActivated.AddDynamic(this, &ALevelBlueprint::CheckAllButtonsPressed);
   Button2.OnButtonActivated.AddDynamic(this, &ALevelBlueprint::CheckAllButtonsPressed);
   Button3.OnButtonActivated.AddDynamic(this, &ALevelBlueprint::CheckAllButtonsPressed);
   
   // Function that checks if all buttons are pressed and opens the door
   void ALevelBlueprint::CheckAllButtonsPressed()
   {
       if (Button1->IsActivated() && Button2->IsActivated() && Button3->IsActivated())
       {
           Door->Open();
       }
   }
   ```

3. **Feedback**: Add visual and audio cues for interaction
   - Particle effects
   - Sound effects
   - Material changes
   - Camera effects

## Testing Interactions

Create a test level to verify:

1. All interaction types work correctly
2. Connections between objects function as expected
3. Player positioning during interactions is natural
4. Visual and audio feedback is clear and helpful
5. Edge cases (interrupted interactions, rapid activation) handle correctly

## Next Steps

After implementing environmental objects:
1. Design interaction-based puzzles
2. Create more specialized interactive objects 
3. Implement a save system to track object states
4. Add tutorial elements to teach interaction mechanics 