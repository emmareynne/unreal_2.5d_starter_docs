# Camera Setup for 2.5D Platformer

This document guides you through setting up a camera system specifically designed for 2.5D platformer gameplay in Unreal Engine 5.5.

## Overview

A well-configured camera is crucial for 2.5D platformer gameplay. Our approach uses a blend of side-scrolling perspective with depth perception to create a visually appealing experience while maintaining clear gameplay visibility.

## Creating a Game Mode Blueprint with Custom Camera Manager

Here's how to set up your camera manager properly:

1. Create a Blueprint Class Based on Your C++ Game Mode:
   - In Unreal Editor, go to the Content Browser
   - Right-click in an empty area → Create Basic Asset → Blueprint Class
   - In the "Pick Parent Class" window, search for "PlatformerGameMode" and select it
   - Name your Blueprint something like "BP_PlatformerGameMode"

2. Set the Camera Manager in Your Game Mode Blueprint:
   - Double-click your new BP_PlatformerGameMode to open it
   - In the Class Defaults panel (right side), scroll down to find the Classes section
   - Look for the property called "Player Camera Manager Class"
   - Click the dropdown arrow and select your BP_PlayerCameraManager asset

3. Set Your Blueprint as the Default Game Mode:
   - Option A: Project-wide Default
     - Go to Edit → Project Settings
     - In the left panel, find Maps & Modes
     - Under Default Modes, find "Default GameMode"
     - Set it to your new BP_PlatformerGameMode
   - Option B: Level-specific Default
     - Open your level
     - Go to Window → World Settings
     - Find "GameMode Override" at the top
     - Set it to your new BP_PlatformerGameMode
     
## Creating the Side-Scrolling Camera Component

Next, we'll create a specialized Camera Component to attach to our character:

1. Create a new Blueprint class:
   - Right-click in the Content Browser and select **Blueprint Class**
   - In the **All Classes** tab, search for and select `CameraComponent`
   - Name it `BP_SideScrollCamera`

2. Open the Blueprint and set up basic properties:
   - In the Class Defaults:
     - Projection Mode: Perspective
     - Field of View: 60
     - Camera Location: X=0, Y=400, Z=100
     - Camera Rotation: Pitch=-10, Yaw=0, Roll=0
     - Enable Camera Lag: True
     - Camera Lag Speed: 5.0
     - Camera Rotation Lag Speed: 10.0

3. Add a Spring Arm Component:
   - In the Components panel, add a **Spring Arm Component**
   - Make it the root component
   - Set Length to 0 (we'll control distance through offset)
   - Enable `Inherit Pitch`, `Inherit Yaw`, and `Inherit Roll`
   - Set Target Arm Length to 400
   - Set Socket Offset to (0, 0, 100)
   - Enable Camera Lag
   - Enable Camera Rotation Lag

4. Attach the Camera Component to the Spring Arm:
   - Drag the Camera Component onto the Spring Arm in the Components panel
   - Set the Camera's relative location to (0, 0, 0)

## Implementing the Camera System in C++

For more precise control, let's implement our camera system in C++:

1. Create a new C++ class:
   - In the editor, go to **File → New C++ Class**
   - Select `CameraComponent` as the parent class
   - Name it `PlatformerCameraComponent`

2. Implement the header file (`PlatformerCameraComponent.h`):

```cpp
#pragma once

#include "CoreMinimal.h"
#include "Camera/CameraComponent.h"
#include "PlatformerCameraComponent.generated.h"

/**
 * Specialized camera component for 2.5D platformer games
 * Maintains a side-view with slight depth perception
 */
UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class YOURPROJECT_API UPlatformerCameraComponent : public UCameraComponent
{
    GENERATED_BODY()
    
public:
    // Sets default values
    UPlatformerCameraComponent();
    
    // Called every frame
    virtual void TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction) override;
    
    // Configure the target offset
    UFUNCTION(BlueprintCallable, Category = "Camera")
    void SetTargetOffset(FVector NewOffset);
    
protected:
    // Called when the game starts
    virtual void BeginPlay() override;
    
    // Process camera movement
    void UpdateCameraPosition(float DeltaTime);
    
    // Target offset from character
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Camera")
    FVector TargetOffset = FVector(0.0f, 400.0f, 100.0f);
    
    // How quickly the camera follows the character
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Camera")
    float FollowSpeed = 5.0f;
    
    // How quickly camera rotates to face target direction
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Camera")
    float RotationSpeed = 10.0f;
    
    // Fixed camera angles for 2.5D view
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Camera")
    float CameraPitch = -10.0f;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Camera")
    float CameraYaw = 0.0f;
    
    // Current target to follow (typically the owning actor)
    UPROPERTY(Transient)
    AActor* TargetActor;
};
```

3. Implement the source file (`PlatformerCameraComponent.cpp`):

```cpp
#include "PlatformerCameraComponent.h"
#include "Kismet/KismetMathLibrary.h"

UPlatformerCameraComponent::UPlatformerCameraComponent()
{
    // Enable tick to update camera every frame
    PrimaryComponentTick.bCanEverTick = true;
    
    // Set default FOV
    FieldOfView = 60.0f;
}

void UPlatformerCameraComponent::BeginPlay()
{
    Super::BeginPlay();
    
    // Store the owning actor as target
    TargetActor = GetOwner();
}

void UPlatformerCameraComponent::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
{
    Super::TickComponent(DeltaTime, TickType, ThisTickFunction);

    // Update camera position and rotation
    UpdateCameraPosition(DeltaTime);
}

void UPlatformerCameraComponent::SetTargetOffset(FVector NewOffset)
{
    TargetOffset = NewOffset;
}

void UPlatformerCameraComponent::UpdateCameraPosition(float DeltaTime)
{
    if (!TargetActor)
        return;
    
    // Get the target position (character's location)
    FVector TargetLocation = TargetActor->GetActorLocation();
    
    // Calculate desired camera position with offset
    FVector DesiredLocation = TargetLocation + TargetOffset;
    
    // Get current camera location
    FVector CurrentLocation = GetComponentLocation();
    
    // Interpolate to the new position for smooth camera movement
    FVector NewLocation = FMath::VInterpTo(
        CurrentLocation,
        DesiredLocation,
        DeltaTime,
        FollowSpeed
    );
    
    // Set the new camera location
    SetWorldLocation(NewLocation);
    
    // Create a fixed rotation for the 2.5D view
    FRotator DesiredRotation = FRotator(CameraPitch, CameraYaw, 0.0f);
    
    // Get current camera rotation
    FRotator CurrentRotation = GetComponentRotation();
    
    // Interpolate to the new rotation for smooth camera movement
    FRotator NewRotation = FMath::RInterpTo(
        CurrentRotation,
        DesiredRotation,
        DeltaTime,
        RotationSpeed
    );
    
    // Set the new camera rotation
    SetWorldRotation(NewRotation);
}
```

## Integrating with the Player Character

Now let's integrate our camera component with the character:

1. In your character class header (`PlatformerCharacter.h`), add:

```cpp
// Add to your existing character class
// Camera component for 2.5D view
UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Camera")
class UPlatformerCameraComponent* CameraComponent;
```

2. In your character class constructor, initialize the camera:

```cpp
// Inside your character constructor
// Create camera component
CameraComponent = CreateDefaultSubobject<UPlatformerCameraComponent>("CameraComponent");
CameraComponent->SetupAttachment(GetRootComponent());
```

## Setting Camera Constraints

To ensure the camera maintains proper framing, let's add camera constraints:

1. Update the Player Game Mode or Level Blueprint to use our custom Camera Manager:
   - Open your GameMode class
   - Set `PlayerCameraManagerClass` to `BP_PlatformerCameraManager`

2. Add camera boundary constraints to prevent the camera from showing empty areas:
   - Open `PlatformerCameraComponent.h` and add:

```cpp
// Add to the protected section
// Camera boundary constraints
UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Camera Constraints")
bool bEnableBoundaryConstraints = true;

UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Camera Constraints", meta = (EditCondition = "bEnableBoundaryConstraints"))
float MinXBoundary = -2000.0f;

UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Camera Constraints", meta = (EditCondition = "bEnableBoundaryConstraints"))
float MaxXBoundary = 2000.0f;

UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Camera Constraints", meta = (EditCondition = "bEnableBoundaryConstraints"))
float MinYBoundary = -2000.0f;

UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Camera Constraints", meta = (EditCondition = "bEnableBoundaryConstraints"))
float MaxYBoundary = 2000.0f;

UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Camera Constraints", meta = (EditCondition = "bEnableBoundaryConstraints"))
float MinZBoundary = 0.0f;

UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Camera Constraints", meta = (EditCondition = "bEnableBoundaryConstraints"))
float MaxZBoundary = 2000.0f;
```

3. Modify the `UpdateCameraPosition` method in `PlatformerCameraComponent.cpp`:

```cpp
void UPlatformerCameraComponent::UpdateCameraPosition(float DeltaTime)
{
    // ... existing code ...
    
    // Apply boundary constraints if enabled
    if (bEnableBoundaryConstraints)
    {
        // Clamp X position
        NewLocation.X = FMath::Clamp(NewLocation.X, MinXBoundary, MaxXBoundary);
        
        // Clamp Y position
        NewLocation.Y = FMath::Clamp(NewLocation.Y, MinYBoundary, MaxYBoundary);
        
        // Clamp Z position
        NewLocation.Z = FMath::Clamp(NewLocation.Z, MinZBoundary, MaxZBoundary);
    }
    
    // Set the new camera location
    SetWorldLocation(NewLocation);
    
    // ... rest of existing code ...
}
```

## Adding Camera Collision Avoidance

To ensure the camera doesn't clip through walls:

1. Update the Spring Arm component settings in the Blueprint:
   - Enable `Do Collision Test`
   - Set `Probe Channel` to `Camera` or `Visibility`
   - Set `Probe Size` to 12.0

2. For the C++ implementation, update the character class constructor:

```cpp
// Inside your character constructor, if using a SpringArm for the camera
USpringArmComponent* SpringArm = CreateDefaultSubobject<USpringArmComponent>("CameraSpringArm");
SpringArm->SetupAttachment(GetRootComponent());
SpringArm->TargetArmLength = 400.0f;
SpringArm->SocketOffset = FVector(0.0f, 0.0f, 100.0f);
SpringArm->bEnableCameraLag = true;
SpringArm->CameraLagSpeed = 5.0f;
SpringArm->bDoCollisionTest = true;
SpringArm->ProbeChannel = ECC_Camera;
SpringArm->ProbeSize = 12.0f;

// Attach camera to spring arm
CameraComponent->SetupAttachment(SpringArm);
```

## Adding Camera Shake for Feedback

Add subtle camera shake for gameplay feedback:

1. Create a new Blueprint class:
   - Base class: `CameraShakeBase`
   - Name: `BP_PlatformerCameraShake_Jump`

2. Configure the camera shake:
   - Set OscillationDuration to 0.25
   - Set RotOscillation to small values (0.5-1.0)
   - Set FOVOscillation to a small value (0.5-1.0)

3. Add a method to trigger camera shake in your character class:

```cpp
// In PlatformerCharacter.h
UFUNCTION(BlueprintCallable, Category = "Camera")
void PlayJumpCameraShake();

// In PlatformerCharacter.cpp
void APlatformerCharacter::PlayJumpCameraShake()
{
    if (APlayerController* PC = Cast<APlayerController>(GetController()))
    {
        PC->ClientStartCameraShake(JumpCameraShakeClass);
    }
}
```

## Testing the Camera

Test your camera setup to ensure it works correctly:

1. Place your character in the test level
2. Enter Play mode and check for:
   - Proper side-scrolling perspective
   - Smooth camera following
   - Correct constraints to prevent seeing outside the playable area
   - Camera collision avoidance around walls
   - Appropriate field of view and depth perception

3. Fine-tune camera parameters:
   - Adjust the FOV for better or worse depth perception
   - Modify camera lag speed for smoother or more responsive following
   - Tweak the camera height and distance for better visibility

## Advanced Feature: Looking Ahead

Implement an advanced "look ahead" feature for better visibility:

```cpp
// Add to PlatformerCameraComponent.h
UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Camera")
bool bEnableLookAhead = true;

UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Camera", meta = (EditCondition = "bEnableLookAhead"))
float LookAheadDistance = 200.0f;

UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Camera", meta = (EditCondition = "bEnableLookAhead"))
float LookAheadSpeed = 3.0f;

// Current movement direction for look-ahead
UPROPERTY(Transient)
FVector MovementDirection;

// Add to PlatformerCameraComponent.cpp
void UPlatformerCameraComponent::UpdateCameraPosition(float DeltaTime)
{
    // ... existing code ...
    
    // Apply look-ahead if enabled
    if (bEnableLookAhead && TargetActor)
    {
        // Get character velocity
        FVector Velocity = FVector::ZeroVector;
        if (TargetActor->IsA(ACharacter::StaticClass()))
        {
            ACharacter* Character = Cast<ACharacter>(TargetActor);
            Velocity = Character->GetVelocity();
            Velocity.Z = 0; // Ignore vertical velocity for look-ahead
        }
        
        // Only apply look-ahead if moving
        if (!Velocity.IsNearlyZero())
        {
            // Normalize direction and keep just the X component for 2.5D
            MovementDirection = Velocity.GetSafeNormal();
            
            // Calculate look-ahead offset
            FVector LookAheadOffset = MovementDirection * LookAheadDistance;
            
            // Apply to desired location with interpolation
            DesiredLocation += LookAheadOffset;
        }
    }
    
    // ... rest of existing code ...
}
```

## Next Steps

Now that you have set up the camera system for your 2.5D platformer, you can proceed to [Enhanced Input Setup](./4_enhanced_input_setup.md) to implement the player controls. 