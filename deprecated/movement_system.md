# Character Movement System Implementation

## Prerequisites
- Completed steps in [core_framework.md](core_framework.md)
- Basic understanding of [Unreal character movement](https://docs.unrealengine.com/5.3/en-US/API/Runtime/Engine/GameFramework/UCharacterMovementComponent/)

## Step 1: Create Character Base Class
1. In editor: Tools → New C++ Class
2. Parent Class: [Character](https://docs.unrealengine.com/5.3/en-US/API/Runtime/Engine/GameFramework/ACharacter/)
3. Name: "PlatformerCharacter"
4. Add base implementation:

```cpp
// PlatformerCharacter.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Character.h"
#include "AbilitySystemInterface.h"
#include "PlatformerCharacter.generated.h"

UCLASS()
class YOURPROJECT_API APlatformerCharacter : public ACharacter, public IAbilitySystemInterface
{
    GENERATED_BODY()
    
public:
    APlatformerCharacter();
    
    // Called every frame
    virtual void Tick(float DeltaTime) override;
    
    // Called to bind functionality to input
    virtual void SetupPlayerInputComponent(class UInputComponent* PlayerInputComponent) override;
    
    // Input handling
    UPROPERTY(EditDefaultsOnly, Category="Input")
    class UInputMappingContext* DefaultMappingContext;
    
    UPROPERTY(EditDefaultsOnly, Category="Input")
    class UInputAction* MoveAction;
    
    UPROPERTY(EditDefaultsOnly, Category="Input")
    class UInputAction* JumpAction;
    
    // AbilitySystem component
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category="Abilities")
    class UAbilitySystemComponent* AbilitySystemComponent;
    
    // Interface implementation
    virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;
    
    // Movement constraints for 2.5D
    UPROPERTY(EditDefaultsOnly, Category="2.5D Settings")
    bool bConstrainToPlane = true;
    
    UPROPERTY(EditDefaultsOnly, Category="2.5D Settings")
    FVector PlaneConstraintNormal = FVector(0.0f, 1.0f, 0.0f); // Constrain Y-axis by default
    
    // Movement feel tuning
    UPROPERTY(EditDefaultsOnly, Category="Movement|Feel")
    float JumpApexBoost = 0.1f;
    
    UPROPERTY(EditDefaultsOnly, Category="Movement|Feel")
    float CoyoteTimeDuration = 0.1f;
    
    // Debug
    UPROPERTY(EditDefaultsOnly, Category="Debug")
    bool bShowDebugTraces = false;

protected:
    // Called when the game starts or when spawned
    virtual void BeginPlay() override;
    
    // Core movement setup
    void TuneCharacterMovement();
    
    // Input callbacks
    void Move(const struct FInputActionValue& Value);
    void StartJump();
    void StopJump();
    
    // Movement state tracking
    bool bWasJumping = false;
    float TimeSinceLeftGround = 0.0f;
    
    // Coyote time implementation
    void CheckCoyoteTime(float DeltaTime);
};
```

**SUCCESS CHECK:** Class compiles without errors

## Step 2: Implement [Character Movement](https://docs.unrealengine.com/5.3/en-US/movement-in-unreal-engine/) for 2.5D
Add to character .cpp file:

```cpp
// PlatformerCharacter.cpp

#include "PlatformerCharacter.h"
#include "GameFramework/CharacterMovementComponent.h"
#include "AbilitySystemComponent.h"
#include "EnhancedInputComponent.h"
#include "EnhancedInputSubsystems.h"

APlatformerCharacter::APlatformerCharacter()
{
    // Set this character to call Tick() every frame
    PrimaryActorTick.bCanEverTick = true;
    
    // Create ability system component
    AbilitySystemComponent = CreateDefaultSubobject<UAbilitySystemComponent>(TEXT("AbilitySystemComponent"));
}

void APlatformerCharacter::BeginPlay()
{
    Super::BeginPlay();
    
    // Apply tuned movement settings
    TuneCharacterMovement();
    
    // Set up input mapping context
    if (APlayerController* PlayerController = Cast<APlayerController>(GetController()))
    {
        if (UEnhancedInputLocalPlayerSubsystem* Subsystem = ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(PlayerController->GetLocalPlayer()))
        {
            Subsystem->ClearMappingContext(DefaultMappingContext);
            Subsystem->AddMappingContext(DefaultMappingContext, 0);
        }
    }
}

void APlatformerCharacter::TuneCharacterMovement()
{
    // Get movement component
    UCharacterMovementComponent* MovementComponent = GetCharacterMovement();
    
    // Tune for platformer feel
    MovementComponent->GravityScale = 1.5f;
    MovementComponent->JumpZVelocity = 1000.0f;
    MovementComponent->AirControl = 0.8f;
    MovementComponent->MaxAcceleration = 2048.0f;
    MovementComponent->BrakingDecelerationWalking = 2048.0f;
    MovementComponent->GroundFriction = 8.0f;
    
    // Constraint to 2.5D plane
    if (bConstrainToPlane)
    {
        MovementComponent->SetPlaneConstraintEnabled(true);
        MovementComponent->SetPlaneConstraintNormal(PlaneConstraintNormal);
    }
}

void APlatformerCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    Super::SetupPlayerInputComponent(PlayerInputComponent);
    
    // Set up action bindings
    if (UEnhancedInputComponent* EnhancedInputComponent = CastChecked<UEnhancedInputComponent>(PlayerInputComponent))
    {
        // Moving
        EnhancedInputComponent->BindAction(MoveAction, ETriggerEvent::Triggered, this, &APlatformerCharacter::Move);
        
        // Jumping
        EnhancedInputComponent->BindAction(JumpAction, ETriggerEvent::Triggered, this, &APlatformerCharacter::StartJump);
        EnhancedInputComponent->BindAction(JumpAction, ETriggerEvent::Completed, this, &APlatformerCharacter::StopJump);
    }
}

UAbilitySystemComponent* APlatformerCharacter::GetAbilitySystemComponent() const
{
    return AbilitySystemComponent;
}

void APlatformerCharacter::Move(const FInputActionValue& Value)
{
    // Input is a Vector2D
    FVector2D MovementVector = Value.Get<FVector2D>();
    
    if (Controller != nullptr)
    {
        // Find out which way is forward
        const FRotator Rotation = Controller->GetControlRotation();
        const FRotator YawRotation(0, Rotation.Yaw, 0);
        
        // Get forward direction
        const FVector ForwardDirection = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::X);
        
        // Add movement input (only along axes allowed by plane constraint)
        AddMovementInput(ForwardDirection, MovementVector.Y);
        AddMovementInput(FVector::RightVector, MovementVector.X);
    }
}

void APlatformerCharacter::StartJump()
{
    // Check if we can jump (either grounded or in coyote time)
    bool bIsGrounded = GetCharacterMovement()->IsMovingOnGround();
    bool bInCoyoteTime = TimeSinceLeftGround <= CoyoteTimeDuration;
    
    if (bIsGrounded || bInCoyoteTime)
    {
        Jump();
    }
}

void APlatformerCharacter::StopJump()
{
    StopJumping();
}

void APlatformerCharacter::CheckCoyoteTime(float DeltaTime)
{
    bool bIsGrounded = GetCharacterMovement()->IsMovingOnGround();
    
    if (!bIsGrounded)
    {
        TimeSinceLeftGround += DeltaTime;
    }
    else
    {
        TimeSinceLeftGround = 0.0f;
    }
}

void APlatformerCharacter::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);
    
    // Check for coyote time
    CheckCoyoteTime(DeltaTime);
    
    // Jump apex handling - reduce gravity at apex for better feel
    UCharacterMovementComponent* MovementComponent = GetCharacterMovement();
    if (!MovementComponent->IsMovingOnGround())
    {
        // Near apex of jump (low vertical velocity)
        if (FMath::Abs(GetVelocity().Z) < 100.0f)
        {
            MovementComponent->GravityScale = 1.0f - JumpApexBoost;
        }
        else
        {
            MovementComponent->GravityScale = 1.5f;
        }
    }
}
```

## Step 3: Create [Blueprint Derived Class](https://docs.unrealengine.com/5.3/en-US/creating-blueprints-from-c-plus-plus-classes-in-unreal-engine/)

1. In Content Browser: Right-click on C++ PlatformerCharacter class → Create Blueprint Class
2. Name: "BP_PlatformerCharacter"
3. Configure Blueprint:
   - Create Skeletal Mesh component
   - Set default animation blueprint
   - Set Enhanced Input Actions
   - Adjust movement values

**SUCCESS CHECK:** Blueprint character can be placed and moves along the constrained plane

## Step 4: Implement [2.5D Camera System](https://docs.unrealengine.com/5.3/en-US/using-cameras-in-unreal-engine/)

1. Create a new C++ Class:
   - Parent Class: SpringArmComponent
   - Name: "SideScrollCameraArm"

2. Implement the 2.5D camera constraints:

```cpp
// SideScrollCameraArm.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/SpringArmComponent.h"
#include "SideScrollCameraArm.generated.h"

UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class YOURPROJECT_API USideScrollCameraArm : public USpringArmComponent
{
    GENERATED_BODY()
    
public:
    USideScrollCameraArm();
    
    // Camera plane constraint (for 2.5D)
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Camera")
    bool bConstrainCameraToPath = true;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Camera")
    float CameraDistance = 400.0f;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Camera")
    float CameraHeight = 200.0f;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Camera")
    float LookAheadDistance = 100.0f;
    
    virtual void TickComponent(float DeltaTime, enum ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction) override;
    
protected:
    // Track player direction for look-ahead
    FVector LastPlayerDirection;
    
    // Apply look-ahead based on player movement
    void ApplyLookAhead();
    
    // Position camera along the gameplay path
    void PositionCamera();
};
```

3. Implement the camera follow logic:

```cpp
// SideScrollCameraArm.cpp
#include "SideScrollCameraArm.h"
#include "PlatformerCharacter.h"
#include "Kismet/KismetMathLibrary.h"

USideScrollCameraArm::USideScrollCameraArm()
{
    PrimaryComponentTick.bCanEverTick = true;
    
    // Set default spring arm properties
    TargetArmLength = CameraDistance;
    bEnableCameraLag = true;
    CameraLagSpeed = 5.0f;
    
    // Don't rotate arm with controller
    bUsePawnControlRotation = false;
    bInheritPitch = false;
    bInheritYaw = false;
    bInheritRoll = false;
}

void USideScrollCameraArm::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
{
    Super::TickComponent(DeltaTime, TickType, ThisTickFunction);
    
    if (bConstrainCameraToPath)
    {
        PositionCamera();
        ApplyLookAhead();
    }
}

void USideScrollCameraArm::PositionCamera()
{
    // Get owner character
    APlatformerCharacter* Character = Cast<APlatformerCharacter>(GetOwner());
    if (!Character)
    {
        return;
    }
    
    // Get the constraint plane normal from character
    FVector PlaneNormal = Character->PlaneConstraintNormal;
    
    // Calculate camera position perpendicular to constraint plane
    SetWorldRotation(UKismetMathLibrary::MakeRotFromX(PlaneNormal));
    
    // Set spring arm length
    TargetArmLength = CameraDistance;
    
    // Adjust camera height
    SetRelativeLocation(FVector(0.0f, 0.0f, CameraHeight));
}

void USideScrollCameraArm::ApplyLookAhead()
{
    // Get owner character
    APlatformerCharacter* Character = Cast<APlatformerCharacter>(GetOwner());
    if (!Character)
    {
        return;
    }
    
    // Get character velocity
    FVector Velocity = Character->GetVelocity();
    
    // Skip if not moving
    if (Velocity.SizeSquared() < 10.0f)
    {
        return;
    }
    
    // Project velocity to plane
    FVector PlaneNormal = Character->PlaneConstraintNormal;
    FVector PlaneDirection = Velocity - (PlaneNormal * FVector::DotProduct(Velocity, PlaneNormal));
    PlaneDirection.Normalize();
    
    // Calculate look-ahead offset
    FVector LookAheadOffset = PlaneDirection * LookAheadDistance;
    
    // Apply to socket offset (smoothed)
    FVector CurrentOffset = GetSocketOffset();
    FVector TargetOffset = FVector(LookAheadOffset.X, LookAheadOffset.Y, CurrentOffset.Z);
    FVector NewOffset = FMath::VInterpTo(CurrentOffset, TargetOffset, GetWorld()->GetDeltaSeconds(), 2.0f);
    
    // Update socket offset
    SetSocketOffset(NewOffset);
}
```

**SUCCESS CHECK:** Camera follows player with smooth side-scrolling effect

## Step 5: Implement [Wall Jumping](https://docs.unrealengine.com/5.3/en-US/API/Runtime/Engine/GameFramework/UCharacterMovementComponent/#bEnableWallJump)

Add to character header:

```cpp
// In PlatformerCharacter.h

// Wall jump properties
UPROPERTY(EditDefaultsOnly, Category="Movement|WallJump")
bool bEnableWallJump = true;

UPROPERTY(EditDefaultsOnly, Category="Movement|WallJump")
float WallJumpForce = 700.0f;

UPROPERTY(EditDefaultsOnly, Category="Movement|WallJump")
float WallJumpHeight = 600.0f;

UPROPERTY(EditDefaultsOnly, Category="Movement|WallJump")
float WallSlideSpeed = 200.0f;

// Wall detection
void DetectWalls();
bool CheckWall(FVector Direction, FHitResult& OutHit);
void ProcessWallJump();

// Wall state
bool bIsWallSliding = false;
FVector WallNormal;
```

**SUCCESS CHECK:** Character can wall jump when colliding with vertical surfaces

## Step 6: Implement [Slope Sliding](https://docs.unrealengine.com/5.3/en-US/API/Runtime/Engine/GameFramework/UCharacterMovementComponent/#GetWalkingDecelerationForSlidingOnSlope)

Add to character header:

```cpp
// In PlatformerCharacter.h

// Slope properties
UPROPERTY(EditDefaultsOnly, Category="Movement|Slopes")
float SlopeSlideSpeed = 600.0f;

UPROPERTY(EditDefaultsOnly, Category="Movement|Slopes")
float MinSlopeAngle = 30.0f;

// Slope detection
void CheckForSlope();
void ProcessSlope(float SlopeAngle, const FVector& SlopeDirection);

// Slope state
bool bIsOnSlope = false;
FVector SlopeNormal;
```

**SUCCESS CHECK:** Character slides down steep slopes at appropriate speed

## Step 7: Add [Enhanced Movement Controls](https://docs.unrealengine.com/5.3/en-US/enhanced-input-in-unreal-engine/)

Create Input Actions for:
1. Dash
2. Ground Pound
3. Ledge Grab
4. Wall Slide

**SUCCESS CHECK:** All movement actions can be bound to inputs and function correctly

## Next Steps
Once your movement system is implemented, proceed to adding the combat system in [combat_system.md](combat_system.md). 