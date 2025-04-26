# Animation System Implementation

## Prerequisites
- Completed steps in level_design.md
- Working character with movement and combat
- Basic level layout in place

## Step 1: Set Up Character Animation Blueprint

1. Create Animation Blueprint:
   - Import character skeletal mesh and animations
   - In Content Browser, right-click → Animation → Animation Blueprint
   - Set Target Skeleton to your character's skeleton
   - Name it "ABP_PlatformerCharacter"

2. Configure Event Graph:
   - Implement animation blueprint event graph logic:

```
// Event Graph Basic Setup
Event Blueprint Update Animation
|
+-- Cast to PlatformerCharacter (store as variable)
    |
    +-- Is Valid?
        |
        +-- Get Movement Component
        |   |
        |   +-- Get Velocity → Calculate Speed
        |   |
        |   +-- Is Moving On Ground?
        |
        +-- Set 'Speed' variable
        |
        +-- Set 'IsInAir' variable
        |
        +-- Set 'IsAccelerating' variable (based on current acceleration)
        |
        +-- Set 'Direction' variable (use atan2 of X and Y velocity)
```

3. Create State Machine:
   - Create a new state machine named "Locomotion"
   - Set up the following states:
     - Idle
     - Run
     - Jump
     - Fall
     - Attack
     - TakeDamage
     - Death

4. Configure Transitions:
   - Idle to Run: Speed > 10
   - Run to Idle: Speed < 10
   - Idle/Run to Jump: IsInAir == true && Velocity.Z > 0
   - Jump to Fall: Velocity.Z < 0
   - Fall to Idle/Run: IsInAir == false
   - Any to Attack: Attack trigger (set by code)
   - Any to TakeDamage: Damage trigger (set by code)
   - Any to Death: Death trigger (set by code)

**SUCCESS CHECK:** Animation Blueprint plays correct animations for different states

## Step 2: Create Attack Animation Montages

1. Create Base Attack Montage:
   - Import attack animation
   - In Content Browser, right-click animation → Create → Animation Montage
   - Name it "AM_MeleeAttack"
   - Set up sections for combo attacks if needed

2. Configure Animation Notifies:
   - Add "AttackStart" notify at start of attack swing
   - Add "ApplyDamage" notify at impact point
   - Add "AttackEnd" notify at end of swing

3. Create Ranged Attack Montage:
   - Create "AM_RangedAttack" for projectile attacks
   - Add "FireProjectile" notify at firing point

4. Link to Character Class:
   - Add montage references in character class:

```cpp
// In PlatformerCharacter.h
UPROPERTY(EditDefaultsOnly, Category="Animation")
class UAnimMontage* MeleeAttackMontage;

UPROPERTY(EditDefaultsOnly, Category="Animation")
class UAnimMontage* RangedAttackMontage;

// Animation functions
void PlayAttackMontage();
void PlayRangedAttackMontage();
```

```cpp
// In PlatformerCharacter.cpp
void APlatformerCharacter::PlayAttackMontage()
{
    if (MeleeAttackMontage)
    {
        PlayAnimMontage(MeleeAttackMontage);
    }
}

void APlatformerCharacter::PlayRangedAttackMontage()
{
    if (RangedAttackMontage)
    {
        PlayAnimMontage(RangedAttackMontage);
    }
}
```

**SUCCESS CHECK:** Attack montages play correctly and notify events fire

## Step 3: Create Animation Notify System

1. Create Notify Class for Attack:
   - Tools → New C++ Class
   - Parent Class: AnimNotify
   - Name: "AnimNotify_ApplyDamage"

```cpp
// AnimNotify_ApplyDamage.h
#pragma once

#include "CoreMinimal.h"
#include "Animation/AnimNotifies/AnimNotify.h"
#include "AnimNotify_ApplyDamage.generated.h"

UCLASS()
class YOURPROJECT_API UAnimNotify_ApplyDamage : public UAnimNotify
{
    GENERATED_BODY()
    
public:
    virtual void Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation) override;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Attack")
    float DamageRadius = 100.0f;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Attack")
    FName SocketName = "WeaponTip";
};
```

```cpp
// AnimNotify_ApplyDamage.cpp
#include "AnimNotify_ApplyDamage.h"
#include "PlatformerCharacter.h"
#include "CombatComponent.h"
#include "Kismet/KismetSystemLibrary.h"

void UAnimNotify_ApplyDamage::Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation)
{
    if (!MeshComp || !MeshComp->GetOwner())
    {
        return;
    }
    
    // Get character
    APlatformerCharacter* Character = Cast<APlatformerCharacter>(MeshComp->GetOwner());
    if (!Character)
    {
        return;
    }
    
    // Get combat component
    UCombatComponent* CombatComp = Character->FindComponentByClass<UCombatComponent>();
    if (!CombatComp)
    {
        return;
    }
    
    // Get socket location for weapon hit
    FVector SocketLocation = MeshComp->GetSocketLocation(SocketName);
    
    // Perform sphere trace for enemies
    TArray<AActor*> ActorsToIgnore;
    ActorsToIgnore.Add(Character);
    
    TArray<FHitResult> HitResults;
    UKismetSystemLibrary::SphereTraceMulti(
        Character->GetWorld(),
        SocketLocation,
        SocketLocation + Character->GetActorForwardVector() * 10.0f, // Small offset to force direction
        DamageRadius,
        ETraceTypeQuery::TraceTypeQuery1, // "Visibility" trace channel
        false,
        ActorsToIgnore,
        EDrawDebugTrace::None,
        HitResults,
        true
    );
    
    // Apply damage to all hit actors
    for (const FHitResult& Hit : HitResults)
    {
        if (Hit.GetActor() && Hit.GetActor() != Character)
        {
            CombatComp->ApplyDamageToTarget(Hit.GetActor(), CombatComp->AttackDamage);
        }
    }
}
```

2. Create Notify Class for Projectile:
   - Tools → New C++ Class
   - Parent Class: AnimNotify
   - Name: "AnimNotify_FireProjectile"

```cpp
// AnimNotify_FireProjectile.h
#pragma once

#include "CoreMinimal.h"
#include "Animation/AnimNotifies/AnimNotify.h"
#include "AnimNotify_FireProjectile.generated.h"

UCLASS()
class YOURPROJECT_API UAnimNotify_FireProjectile : public UAnimNotify
{
    GENERATED_BODY()
    
public:
    virtual void Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation) override;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Projectile")
    FName SpawnSocketName = "WeaponTip";
};
```

```cpp
// AnimNotify_FireProjectile.cpp
#include "AnimNotify_FireProjectile.h"
#include "PlatformerCharacter.h"
#include "CombatComponent.h"

void UAnimNotify_FireProjectile::Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation)
{
    if (!MeshComp || !MeshComp->GetOwner())
    {
        return;
    }
    
    // Get character
    APlatformerCharacter* Character = Cast<APlatformerCharacter>(MeshComp->GetOwner());
    if (!Character)
    {
        return;
    }
    
    // Get combat component
    UCombatComponent* CombatComp = Character->FindComponentByClass<UCombatComponent>();
    if (!CombatComp)
    {
        return;
    }
    
    // Fire projectile
    CombatComp->FireProjectile();
}
```

3. Create Footstep Notify:
   - Create similar notify for footstep sounds
   - Connect to sound/particle system for feet

**SUCCESS CHECK:** Animation notifies trigger appropriate functionality

## Step 4: Set Up Root Motion

1. Configure Root Motion in Animations:
   - Enable root motion in movement animations
   - Set up root motion curves if needed

2. Configure Character Movement Component:
   - Update character setup:

```cpp
// In PlatformerCharacter constructor
GetCharacterMovement()->bAllowPhysicsRotationDuringAnimRootMotion = false;
GetCharacterMovement()->bOrientRotationToMovement = true;
```

3. Test Root Motion Movement:
   - Ensure character moves correctly with root motion
   - Adjust movement parameters if needed

**SUCCESS CHECK:** Character moves smoothly with root motion

## Step 5: Set Up Animation Blending

1. Create Blend Spaces:
   - Create "BS_GroundLocomotion" blend space
   - Set up with Idle/Walk/Run based on speed
   - Configure blend parameters (speed, direction)

2. Create Aim Offset:
   - Create aim offset for firing direction
   - Connect to character look direction

3. Configure Blending in State Machine:
   - Use blend spaces in animation state machine
   - Set up appropriate blend times between states

**SUCCESS CHECK:** Character smoothly transitions between animations

## Step 6: Set Up Animation Blueprint Interface

1. Create Animation Blueprint Interface:
   - Tools → New C++ Class
   - Parent Class: UAnimInstance
   - Name: "PlatformerAnimInstance"

```cpp
// PlatformerAnimInstance.h
#pragma once

#include "CoreMinimal.h"
#include "Animation/AnimInstance.h"
#include "PlatformerAnimInstance.generated.h"

UCLASS()
class YOURPROJECT_API UPlatformerAnimInstance : public UAnimInstance
{
    GENERATED_BODY()
    
public:
    UPlatformerAnimInstance();
    
    virtual void NativeInitializeAnimation() override;
    virtual void NativeUpdateAnimation(float DeltaSeconds) override;
    
    // Character reference
    UPROPERTY(BlueprintReadOnly, Category="Character")
    class APlatformerCharacter* OwningCharacter;
    
    // Animation properties
    UPROPERTY(BlueprintReadOnly, Category="Movement")
    float Speed;
    
    UPROPERTY(BlueprintReadOnly, Category="Movement")
    bool bIsInAir;
    
    UPROPERTY(BlueprintReadOnly, Category="Movement")
    bool bIsAccelerating;
    
    UPROPERTY(BlueprintReadOnly, Category="Movement")
    float Direction;
    
    UPROPERTY(BlueprintReadOnly, Category="State")
    bool bIsAttacking;
    
    UPROPERTY(BlueprintReadOnly, Category="State")
    float Health;
};
```

```cpp
// PlatformerAnimInstance.cpp
#include "PlatformerAnimInstance.h"
#include "PlatformerCharacter.h"
#include "GameFramework/CharacterMovementComponent.h"
#include "Kismet/KismetMathLibrary.h"

UPlatformerAnimInstance::UPlatformerAnimInstance()
{
    Speed = 0.0f;
    bIsInAir = false;
    bIsAccelerating = false;
    Direction = 0.0f;
    bIsAttacking = false;
    Health = 100.0f;
}

void UPlatformerAnimInstance::NativeInitializeAnimation()
{
    Super::NativeInitializeAnimation();
    
    // Get the owning character
    OwningCharacter = Cast<APlatformerCharacter>(TryGetPawnOwner());
}

void UPlatformerAnimInstance::NativeUpdateAnimation(float DeltaSeconds)
{
    Super::NativeUpdateAnimation(DeltaSeconds);
    
    if (!OwningCharacter)
    {
        // Try to get the character again if not already set
        OwningCharacter = Cast<APlatformerCharacter>(TryGetPawnOwner());
        if (!OwningCharacter)
        {
            return;
        }
    }
    
    // Get movement component
    UCharacterMovementComponent* MovementComp = OwningCharacter->GetCharacterMovement();
    if (!MovementComp)
    {
        return;
    }
    
    // Update movement properties
    FVector Velocity = OwningCharacter->GetVelocity();
    Speed = Velocity.Size2D(); // Ignore Z component for speed
    
    bIsInAir = MovementComp->IsFalling();
    bIsAccelerating = MovementComp->GetCurrentAcceleration().Size() > 0.0f;
    
    // Calculate movement direction
    if (Speed > 0.0f)
    {
        FRotator MovementRotation = UKismetMathLibrary::MakeRotFromX(Velocity);
        FRotator ActorRotation = OwningCharacter->GetActorRotation();
        
        Direction = UKismetMathLibrary::NormalizedDeltaRotator(MovementRotation, ActorRotation).Yaw;
    }
    else
    {
        Direction = 0.0f;
    }
    
    // Get other state information
    // Example: bIsAttacking = OwningCharacter->IsAttacking();
    // Example: Health = OwningCharacter->GetCurrentHealth();
}
```

2. Connect Animation Instance to Character:
   - Set the animation instance class in character blueprint

**SUCCESS CHECK:** Animation blueprint correctly updates based on character state

## Step 7: Implement Smooth Camera Follow

1. Create Camera Logic:
   - Update camera behavior for smooth following
   - Implement camera lag for better feel

```cpp
// In PlatformerCharacter.h
UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category="Camera")
class USpringArmComponent* CameraBoom;

UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category="Camera")
class UCameraComponent* FollowCamera;

UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category="Camera")
float CameraLagSpeed = 3.0f;

UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category="Camera")
float CameraRotationLagSpeed = 10.0f;
```

```cpp
// In PlatformerCharacter constructor
// Create camera boom
CameraBoom = CreateDefaultSubobject<USpringArmComponent>(TEXT("CameraBoom"));
CameraBoom->SetupAttachment(RootComponent);
CameraBoom->TargetArmLength = 400.0f;
CameraBoom->bUsePawnControlRotation = true; // Rotate arm based on controller
CameraBoom->bEnableCameraLag = true;
CameraBoom->CameraLagSpeed = CameraLagSpeed;
CameraBoom->CameraRotationLagSpeed = CameraRotationLagSpeed;

// Create follow camera
FollowCamera = CreateDefaultSubobject<UCameraComponent>(TEXT("FollowCamera"));
FollowCamera->SetupAttachment(CameraBoom, USpringArmComponent::SocketName);
FollowCamera->bUsePawnControlRotation = false; // Camera doesn't rotate relative to arm
```

2. Implement Camera Controls:
   - Create camera zoom functionality
   - Implement camera shake for impacts

**SUCCESS CHECK:** Camera smoothly follows character with appropriate feel

## Step 8: Create Enemy Animations

1. Set Up Enemy Animation Blueprint:
   - Create animation blueprint for enemies
   - Set up appropriate state machine

2. Create Enemy Attack Animations:
   - Create attack montages
   - Set up animation notifies

3. Configure Enemy AI Animation:
   - Connect animation to AI behavior
   - Set up transition rules

**SUCCESS CHECK:** Enemies animate correctly based on their behavior

## Next Steps
With your animation system in place, your 2.5D platformer has all the fundamental systems needed for development. You can now focus on adding more content, polish, and game-specific features. 