# Level Design Implementation

## Prerequisites
- Completed steps in ui_system.md
- Working character with movement and combat
- Basic UI framework in place

## Step 1: Create Platform Base Classes

1. Create Static Platform Actor:
   - Tools → New C++ Class
   - Parent Class: Actor
   - Name: "PlatformBase"
   - Implement basic platform with StaticMeshComponent

```cpp
// PlatformBase.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "PlatformBase.generated.h"

UCLASS()
class YOURPROJECT_API APlatformBase : public AActor
{
    GENERATED_BODY()
    
public:
    APlatformBase();
    
    // Components
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category="Components")
    class UStaticMeshComponent* PlatformMesh;
    
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category="Components")
    class UBoxComponent* CollisionBox;
    
    // Properties
    UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category="Platform")
    bool bIsSolid = true;
    
protected:
    virtual void BeginPlay() override;
};
```

```cpp
// PlatformBase.cpp
#include "PlatformBase.h"
#include "Components/StaticMeshComponent.h"
#include "Components/BoxComponent.h"

APlatformBase::APlatformBase()
{
    PrimaryActorTick.bCanEverTick = false;
    
    // Create root collision box
    CollisionBox = CreateDefaultSubobject<UBoxComponent>(TEXT("CollisionBox"));
    CollisionBox->SetCollisionProfileName("BlockAllDynamic");
    RootComponent = CollisionBox;
    
    // Create static mesh component
    PlatformMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("PlatformMesh"));
    PlatformMesh->SetupAttachment(RootComponent);
    PlatformMesh->SetCollisionEnabled(ECollisionEnabled::NoCollision);
}

void APlatformBase::BeginPlay()
{
    Super::BeginPlay();
    
    // Configure collision based on properties
    if (bIsSolid)
    {
        CollisionBox->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
    }
    else
    {
        CollisionBox->SetCollisionEnabled(ECollisionEnabled::QueryOnly);
    }
}
```

2. Create Moving Platform Actor:
   - Tools → New C++ Class
   - Parent Class: PlatformBase
   - Name: "MovingPlatform"
   - Implement movement logic

```cpp
// MovingPlatform.h
#pragma once

#include "CoreMinimal.h"
#include "PlatformBase.h"
#include "MovingPlatform.generated.h"

UENUM(BlueprintType)
enum class EMovementPattern : uint8
{
    LinearBackAndForth,
    Loop,
    OneWay
};

UCLASS()
class YOURPROJECT_API AMovingPlatform : public APlatformBase
{
    GENERATED_BODY()
    
public:
    AMovingPlatform();
    
    virtual void Tick(float DeltaTime) override;
    
    // Movement properties
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Movement")
    EMovementPattern MovementPattern;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Movement")
    TArray<FVector> WayPoints;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Movement")
    float MoveSpeed = 100.0f;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Movement")
    float WaitTimeAtWaypoints = 0.0f;
    
protected:
    virtual void BeginPlay() override;
    
    // Movement logic
    void MoveToNextPoint(float DeltaTime);
    
    // Properties
    int32 CurrentWayPointIndex = 0;
    bool bIsReversing = false;
    bool bIsWaiting = false;
    float WaitTimer = 0.0f;
    FVector StartLocation;
};
```

```cpp
// MovingPlatform.cpp
#include "MovingPlatform.h"

AMovingPlatform::AMovingPlatform()
{
    PrimaryActorTick.bCanEverTick = true;
}

void AMovingPlatform::BeginPlay()
{
    Super::BeginPlay();
    
    // Store starting location if no waypoints
    StartLocation = GetActorLocation();
    
    // Add starting location as first point if not already present
    if (WayPoints.Num() == 0)
    {
        WayPoints.Add(StartLocation);
    }
}

void AMovingPlatform::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);
    
    // Skip if no waypoints
    if (WayPoints.Num() <= 1)
    {
        return;
    }
    
    // Handle waiting at waypoints
    if (bIsWaiting)
    {
        WaitTimer += DeltaTime;
        if (WaitTimer >= WaitTimeAtWaypoints)
        {
            bIsWaiting = false;
            WaitTimer = 0.0f;
        }
        else
        {
            return;
        }
    }
    
    // Move platform
    MoveToNextPoint(DeltaTime);
}

void AMovingPlatform::MoveToNextPoint(float DeltaTime)
{
    // Get current target
    if (CurrentWayPointIndex >= WayPoints.Num())
    {
        CurrentWayPointIndex = 0;
    }
    
    FVector TargetLocation = WayPoints[CurrentWayPointIndex];
    FVector CurrentLocation = GetActorLocation();
    
    // Calculate direction and distance
    FVector Direction = (TargetLocation - CurrentLocation).GetSafeNormal();
    float DistanceToTarget = FVector::Dist(CurrentLocation, TargetLocation);
    
    // Check if we've reached the target
    float MovementThisFrame = MoveSpeed * DeltaTime;
    if (MovementThisFrame >= DistanceToTarget)
    {
        // We've reached the target
        SetActorLocation(TargetLocation);
        
        // Start waiting if configured
        if (WaitTimeAtWaypoints > 0.0f)
        {
            bIsWaiting = true;
            return;
        }
        
        // Move to next waypoint based on movement pattern
        switch (MovementPattern)
        {
            case EMovementPattern::LinearBackAndForth:
                if (bIsReversing)
                {
                    CurrentWayPointIndex--;
                    if (CurrentWayPointIndex <= 0)
                    {
                        CurrentWayPointIndex = 0;
                        bIsReversing = false;
                    }
                }
                else
                {
                    CurrentWayPointIndex++;
                    if (CurrentWayPointIndex >= WayPoints.Num() - 1)
                    {
                        CurrentWayPointIndex = WayPoints.Num() - 1;
                        bIsReversing = true;
                    }
                }
                break;
                
            case EMovementPattern::Loop:
                CurrentWayPointIndex = (CurrentWayPointIndex + 1) % WayPoints.Num();
                break;
                
            case EMovementPattern::OneWay:
                CurrentWayPointIndex++;
                if (CurrentWayPointIndex >= WayPoints.Num())
                {
                    // Teleport back to first waypoint
                    CurrentWayPointIndex = 0;
                    SetActorLocation(WayPoints[0]);
                }
                break;
        }
    }
    else
    {
        // Move toward target
        FVector NewLocation = CurrentLocation + (Direction * MovementThisFrame);
        SetActorLocation(NewLocation);
    }
}
```

3. Create Destructible Platform:
   - Derive from PlatformBase
   - Add destruction functionality

**SUCCESS CHECK:** Platforms compile and can be placed in level

## Step 2: Create Collectible Items

1. Create Base Collectible Class:
   - Tools → New C++ Class
   - Parent Class: Actor
   - Name: "CollectibleBase"
   - Implement collection logic

```cpp
// CollectibleBase.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "CollectibleBase.generated.h"

UENUM(BlueprintType)
enum class ECollectibleType : uint8
{
    Coin,
    PowerUp,
    Health,
    ExtraLife
};

UCLASS()
class YOURPROJECT_API ACollectibleBase : public AActor
{
    GENERATED_BODY()
    
public:
    ACollectibleBase();
    
    // Components
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category="Components")
    class USphereComponent* CollectionSphere;
    
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category="Components")
    class UStaticMeshComponent* CollectibleMesh;
    
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category="Components")
    class URotatingMovementComponent* RotatingMovement;
    
    // Properties
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Collectible")
    ECollectibleType CollectibleType;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Collectible")
    int32 Value = 1;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Collectible")
    class USoundBase* CollectionSound;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Collectible")
    class UParticleSystem* CollectionEffect;
    
protected:
    virtual void BeginPlay() override;
    
    // Collection logic
    UFUNCTION()
    void OnOverlapBegin(UPrimitiveComponent* OverlappedComponent, 
                       AActor* OtherActor, UPrimitiveComponent* OtherComp, 
                       int32 OtherBodyIndex, bool bFromSweep, 
                       const FHitResult& SweepResult);
    
    // Collect functionality (can be overridden)
    UFUNCTION(BlueprintNativeEvent)
    void Collect(class APlatformerCharacter* Character);
    virtual void Collect_Implementation(class APlatformerCharacter* Character);
};
```

```cpp
// CollectibleBase.cpp
#include "CollectibleBase.h"
#include "Components/SphereComponent.h"
#include "Components/StaticMeshComponent.h"
#include "GameFramework/RotatingMovementComponent.h"
#include "PlatformerCharacter.h"
#include "Kismet/GameplayStatics.h"

ACollectibleBase::ACollectibleBase()
{
    PrimaryActorTick.bCanEverTick = false;
    
    // Create collision sphere
    CollectionSphere = CreateDefaultSubobject<USphereComponent>(TEXT("CollectionSphere"));
    CollectionSphere->SetSphereRadius(50.0f);
    CollectionSphere->SetCollisionProfileName("Collectible");
    CollectionSphere->OnComponentBeginOverlap.AddDynamic(this, &ACollectibleBase::OnOverlapBegin);
    RootComponent = CollectionSphere;
    
    // Create mesh
    CollectibleMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("CollectibleMesh"));
    CollectibleMesh->SetupAttachment(RootComponent);
    CollectibleMesh->SetCollisionEnabled(ECollisionEnabled::NoCollision);
    
    // Add rotating movement
    RotatingMovement = CreateDefaultSubobject<URotatingMovementComponent>(TEXT("RotatingMovement"));
    RotatingMovement->RotationRate = FRotator(0, 180, 0);
}

void ACollectibleBase::BeginPlay()
{
    Super::BeginPlay();
}

void ACollectibleBase::OnOverlapBegin(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, 
                                    UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, 
                                    bool bFromSweep, const FHitResult& SweepResult)
{
    // Check if it's the player
    APlatformerCharacter* Character = Cast<APlatformerCharacter>(OtherActor);
    if (Character)
    {
        // Collect the item
        Collect(Character);
    }
}

void ACollectibleBase::Collect_Implementation(APlatformerCharacter* Character)
{
    // Play collection effects
    if (CollectionSound)
    {
        UGameplayStatics::PlaySoundAtLocation(this, CollectionSound, GetActorLocation());
    }
    
    if (CollectionEffect)
    {
        UGameplayStatics::SpawnEmitterAtLocation(GetWorld(), CollectionEffect, GetActorLocation());
    }
    
    // Apply effect based on collectible type
    switch (CollectibleType)
    {
        case ECollectibleType::Coin:
            // Add to coin count/score
            // Character->AddCoins(Value);
            break;
            
        case ECollectibleType::PowerUp:
            // Grant power-up
            // Character->ActivatePowerUp();
            break;
            
        case ECollectibleType::Health:
            // Restore health
            // Character->RestoreHealth(Value);
            break;
            
        case ECollectibleType::ExtraLife:
            // Add extra life
            // Character->AddExtraLife();
            break;
    }
    
    // Destroy the collectible
    Destroy();
}
```

2. Create Blueprint Subclasses:
   - Create BP_Coin from CollectibleBase
   - Create BP_PowerUp from CollectibleBase
   - Create BP_HealthPickup from CollectibleBase

**SUCCESS CHECK:** Collectibles can be placed and collected

## Step 3: Create Hazards and Obstacles

1. Create Base Hazard Class:
   - Tools → New C++ Class
   - Parent Class: Actor
   - Name: "HazardBase"
   - Implement damage logic

```cpp
// HazardBase.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "HazardBase.generated.h"

UCLASS()
class YOURPROJECT_API AHazardBase : public AActor
{
    GENERATED_BODY()
    
public:
    AHazardBase();
    
    // Components
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category="Components")
    class UBoxComponent* DamageBox;
    
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category="Components")
    class UStaticMeshComponent* HazardMesh;
    
    // Properties
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Hazard")
    float DamageAmount = 10.0f;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Hazard")
    bool bIsLethal = false;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Hazard")
    float KnockbackForce = 500.0f;
    
protected:
    virtual void BeginPlay() override;
    
    // Collision handlers
    UFUNCTION()
    void OnOverlapBegin(UPrimitiveComponent* OverlappedComponent, 
                       AActor* OtherActor, UPrimitiveComponent* OtherComp, 
                       int32 OtherBodyIndex, bool bFromSweep, 
                       const FHitResult& SweepResult);
    
    // Apply damage functionality (can be overridden)
    UFUNCTION(BlueprintNativeEvent)
    void ApplyDamage(class APlatformerCharacter* Character);
    virtual void ApplyDamage_Implementation(class APlatformerCharacter* Character);
};
```

```cpp
// HazardBase.cpp
#include "HazardBase.h"
#include "Components/BoxComponent.h"
#include "Components/StaticMeshComponent.h"
#include "PlatformerCharacter.h"
#include "GameFramework/CharacterMovementComponent.h"

AHazardBase::AHazardBase()
{
    PrimaryActorTick.bCanEverTick = false;
    
    // Create damage box
    DamageBox = CreateDefaultSubobject<UBoxComponent>(TEXT("DamageBox"));
    DamageBox->SetCollisionProfileName("Trigger");
    DamageBox->OnComponentBeginOverlap.AddDynamic(this, &AHazardBase::OnOverlapBegin);
    RootComponent = DamageBox;
    
    // Create mesh
    HazardMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("HazardMesh"));
    HazardMesh->SetupAttachment(RootComponent);
    HazardMesh->SetCollisionEnabled(ECollisionEnabled::NoCollision);
}

void AHazardBase::BeginPlay()
{
    Super::BeginPlay();
}

void AHazardBase::OnOverlapBegin(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, 
                               UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, 
                               bool bFromSweep, const FHitResult& SweepResult)
{
    // Check if it's the player
    APlatformerCharacter* Character = Cast<APlatformerCharacter>(OtherActor);
    if (Character)
    {
        // Apply damage
        ApplyDamage(Character);
    }
}

void AHazardBase::ApplyDamage_Implementation(APlatformerCharacter* Character)
{
    if (!Character)
    {
        return;
    }
    
    // If lethal, kill immediately
    if (bIsLethal)
    {
        // Character->Die();
        return;
    }
    
    // Apply damage
    // Character->TakeDamage(DamageAmount);
    
    // Apply knockback
    FVector KnockbackDirection = Character->GetActorLocation() - GetActorLocation();
    KnockbackDirection.Z = 0.5f; // Add some upward force
    KnockbackDirection.Normalize();
    
    // Apply impulse
    UCharacterMovementComponent* MovementComp = Character->GetCharacterMovement();
    if (MovementComp)
    {
        MovementComp->AddImpulse(KnockbackDirection * KnockbackForce, true);
    }
}
```

2. Create Blueprint Subclasses:
   - Create BP_Spikes from HazardBase
   - Create BP_FireTrap from HazardBase
   - Create BP_KillVolume from HazardBase (set bIsLethal = true)

**SUCCESS CHECK:** Hazards damage the player and provide knockback

## Step 4: Create Level Components

1. Create Checkpoint System:
   - Create Checkpoint class
   - Implement save location functionality
   - Connect to player respawn logic

2. Create Level End Trigger:
   - Create LevelEnd class
   - Implement level completion
   - Connect to UI and progression

3. Create Camera Triggers:
   - Create CameraTrigger class
   - Implement camera position/zoom changes
   - Connect to player camera system

**SUCCESS CHECK:** Level components function properly

## Step 5: Build Test Level

1. Create a Basic Level Layout:
   - Create new level
   - Add floor and walls
   - Place player start point

2. Add Platforms:
   - Place static platforms
   - Configure moving platforms
   - Create platform sequences

3. Add Hazards and Collectibles:
   - Place spikes and other hazards
   - Add coins throughout the level
   - Place powerups at strategic locations

4. Add Checkpoints and Level End:
   - Place checkpoint at mid-point
   - Add level end trigger

**SUCCESS CHECK:** Level is playable from start to finish

## Step 6: Configure Level Streaming

1. Create Level Streaming Setup:
   - Create main persistent level
   - Configure streaming sublevels
   - Set up loading triggers

2. Create Level Transition System:
   - Implement level loading logic
   - Create transition screens
   - Handle player position in new level

**SUCCESS CHECK:** Levels load correctly and transitions are smooth

## Step 7: Add Environment Polish

1. Add Background Elements:
   - Create parallax background system
   - Add decorative elements
   - Implement background animations

2. Add Environmental Effects:
   - Add particle systems (dust, leaves, etc.)
   - Implement ambient sound effects
   - Create lighting effects

3. Add Visual Feedback:
   - Create footstep effects
   - Add impact effects for landing
   - Implement environmental reactions

**SUCCESS CHECK:** Level has visual polish and atmosphere

## Step 8: Test and Iterate

1. Complete Playthrough Testing:
   - Test entire level from start to finish
   - Verify collectibles and hazards work
   - Check checkpoint and respawn functionality

2. Adjust Difficulty:
   - Tweak platform spacing
   - Adjust enemy placement
   - Refine hazard timing

3. Performance Optimization:
   - Check for framerate issues
   - Optimize level streaming
   - Reduce particle effect complexity if needed

**SUCCESS CHECK:** Level plays smoothly with appropriate challenge

## Next Steps
With your level design in place, proceed to animation setup in `animation_setup.md`. 