# Combat System Implementation

## Prerequisites
- Completed steps in movement_system.md
- Working character with movement controls
- Basic understanding of GameplayAbilitySystem

## Step 1: Add Attack Input to Character

1. Update PlatformerCharacter.h to include attack input:

```cpp
// Add to existing input properties in PlatformerCharacter.h
UPROPERTY(EditDefaultsOnly, Category="Input")
class UInputAction* AttackAction;

// Add to protected section
void PerformAttack();
bool bCanAttack = true;
float AttackCooldown = 0.5f;
FTimerHandle AttackCooldownTimer;
```

2. Update the input binding in SetupPlayerInputComponent():

```cpp
// In PlatformerCharacter.cpp's SetupPlayerInputComponent()
EnhancedInputComponent->BindAction(AttackAction, ETriggerEvent::Triggered, this, &APlatformerCharacter::PerformAttack);
```

3. Implement the attack method:

```cpp
void APlatformerCharacter::PerformAttack()
{
    if (bCanAttack)
    {
        // Set cooldown
        bCanAttack = false;
        GetWorldTimerManager().SetTimer(AttackCooldownTimer, [this]() { bCanAttack = true; }, AttackCooldown, false);
        
        // Execute attack (will implement with GAS)
        if (AbilitySystemComponent)
        {
            AbilitySystemComponent->TryActivateAbilityByTag(FGameplayTagContainer(FGameplayTag::RequestGameplayTag(FName("Ability.Attack"))));
        }
    }
}
```

**SUCCESS CHECK:** Character responds to attack input with cooldown

## Step 2: Create Combat Component for 2D

1. Create new C++ class:
   - Tools → New C++ Class
   - Parent Class: ActorComponent
   - Name: "CombatComponent"

2. Implement the header:

```cpp
// CombatComponent.h
#pragma once

#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "CombatComponent.generated.h"

UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class YOURPROJECT_API UCombatComponent : public UActorComponent
{
    GENERATED_BODY()
    
public:
    UCombatComponent();
    
    // Basic attack function
    UFUNCTION(BlueprintCallable, Category="Combat")
    void PerformMeleeAttack();
    
    // For projectile shooting
    UFUNCTION(BlueprintCallable, Category="Combat")
    void FireProjectile();
    
    // Combat properties
    UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category="Combat")
    float AttackDamage = 20.f;
    
    UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category="Combat|Melee")
    float AttackWidth = 100.f;
    
    UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category="Combat|Melee")
    float AttackHeight = 50.f;
    
    UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category="Combat|Melee")
    float AttackForwardOffset = 50.f;
    
    UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category="Combat")
    float AttackCooldown = 0.5f;
    
    UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category="Combat|Projectile")
    TSubclassOf<class AProjectile> ProjectileClass;
    
    UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category="Combat|Projectile")
    float ProjectileSpeed = 1000.f;
    
    // References
    UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category="Combat")
    class UAnimMontage* AttackMontage;
    
    // Public helpers
    UFUNCTION(BlueprintCallable, Category="Combat")
    void ApplyDamageToTarget(AActor* Target, float DamageAmount);

protected:
    // Protected helpers
    TArray<AActor*> FindEnemiesInAttackRange();
    
    // State
    bool bCanAttack = true;
    FTimerHandle AttackCooldownTimer;
    
    // Owner reference
    UPROPERTY()
    class APlatformerCharacter* OwnerCharacter;
    
    // Called when the game starts
    virtual void BeginPlay() override;
};
```

3. Implement the basic functionality for 2D:

```cpp
// CombatComponent.cpp
#include "CombatComponent.h"
#include "PlatformerCharacter.h"
#include "Kismet/GameplayStatics.h"
#include "DrawDebugHelpers.h"
#include "Engine/World.h"

UCombatComponent::UCombatComponent()
{
    PrimaryComponentTick.bCanEverTick = false;
}

void UCombatComponent::BeginPlay()
{
    Super::BeginPlay();
    
    // Cache owner reference
    OwnerCharacter = Cast<APlatformerCharacter>(GetOwner());
}

void UCombatComponent::PerformMeleeAttack()
{
    if (!bCanAttack || !OwnerCharacter)
    {
        return;
    }
    
    // Set cooldown
    bCanAttack = false;
    GetWorld()->GetTimerManager().SetTimer(
        AttackCooldownTimer, 
        [this](){ bCanAttack = true; }, 
        AttackCooldown, 
        false);
    
    // Play attack animation if available
    if (AttackMontage)
    {
        OwnerCharacter->PlayAnimMontage(AttackMontage);
    }
    
    // Find enemies in range and apply damage
    TArray<AActor*> EnemiesInRange = FindEnemiesInAttackRange();
    for (AActor* Enemy : EnemiesInRange)
    {
        ApplyDamageToTarget(Enemy, AttackDamage);
    }
    
    // Debug visualization
    if (OwnerCharacter->bShowDebugTraces)
    {
        // Get facing direction (in 2D, will be either -1 or 1 along X axis)
        float FacingDirection = FMath::Sign(OwnerCharacter->GetActorForwardVector().X);
        
        // Draw a 2D box for the attack hitbox
        FVector BoxCenter = OwnerCharacter->GetActorLocation() + 
                          FVector(FacingDirection * AttackForwardOffset, 0.0f, 0.0f);
        FVector BoxExtent = FVector(AttackWidth * 0.5f, 10.0f, AttackHeight * 0.5f);
        
        DrawDebugBox(
            GetWorld(),
            BoxCenter,
            BoxExtent,
            FQuat::Identity,
            FColor::Red,
            false,
            1.0f
        );
    }
}

TArray<AActor*> UCombatComponent::FindEnemiesInAttackRange()
{
    TArray<AActor*> EnemiesInRange;
    if (!OwnerCharacter)
    {
        return EnemiesInRange;
    }
    
    // Get facing direction (in 2D, will be either -1 or 1 along X axis)
    float FacingDirection = FMath::Sign(OwnerCharacter->GetActorForwardVector().X);
    
    // Calculate attack box center (shifted forward in direction character is facing)
    FVector BoxCenter = OwnerCharacter->GetActorLocation() + 
                      FVector(FacingDirection * AttackForwardOffset, 0.0f, 0.0f);
    
    // Set up the box extent (half size in each direction)
    FVector BoxExtent = FVector(AttackWidth * 0.5f, 10.0f, AttackHeight * 0.5f);
    
    // Array to store overlapping actors
    TArray<FOverlapResult> Overlaps;
    FCollisionQueryParams QueryParams;
    QueryParams.AddIgnoredActor(OwnerCharacter); // Don't hit self
    
    // Box trace
    GetWorld()->OverlapMultiByObjectType(
        Overlaps,
        BoxCenter,
        FQuat::Identity,
        FCollisionObjectQueryParams::AllDynamicObjects,
        FCollisionShape::MakeBox(BoxExtent),
        QueryParams
    );
    
    // Filter overlaps to find enemies
    for (const FOverlapResult& Overlap : Overlaps)
    {
        if (Overlap.GetActor() && Overlap.GetActor()->Implements<UDamageableInterface>())
        {
            EnemiesInRange.Add(Overlap.GetActor());
        }
    }
    
    return EnemiesInRange;
}

void UCombatComponent::FireProjectile()
{
    if (!bCanAttack || !ProjectileClass || !OwnerCharacter)
    {
        return;
    }
    
    // Set cooldown
    bCanAttack = false;
    GetWorld()->GetTimerManager().SetTimer(
        AttackCooldownTimer, 
        [this](){ bCanAttack = true; }, 
        AttackCooldown, 
        false);
    
    // Get spawn location (slightly in front of character)
    float FacingDirection = FMath::Sign(OwnerCharacter->GetActorForwardVector().X);
    FVector SpawnLocation = OwnerCharacter->GetActorLocation() + 
                          FVector(FacingDirection * 50.0f, 0.0f, 0.0f);
    
    // For 2D, we only need rotation around Z axis (yaw)
    FRotator SpawnRotation = FRotator(0.0f, FacingDirection > 0 ? 0.0f : 180.0f, 0.0f);
    
    // Spawn the projectile
    FActorSpawnParameters SpawnParams;
    SpawnParams.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AdjustIfPossibleButAlwaysSpawn;
    SpawnParams.Owner = OwnerCharacter;
    
    AProjectile* Projectile = GetWorld()->SpawnActor<AProjectile>(
        ProjectileClass,
        SpawnLocation,
        SpawnRotation,
        SpawnParams
    );
    
    // Set projectile velocity (along X axis for 2D)
    if (Projectile)
    {
        UPrimitiveComponent* ProjectileComponent = Cast<UPrimitiveComponent>(Projectile->GetComponentByClass(UPrimitiveComponent::StaticClass()));
        if (ProjectileComponent)
        {
            ProjectileComponent->SetPhysicsLinearVelocity(FVector(FacingDirection * ProjectileSpeed, 0.0f, 0.0f));
        }
    }
}

void UCombatComponent::ApplyDamageToTarget(AActor* Target, float DamageAmount)
{
    if (!Target)
    {
        return;
    }
    
    // Apply gameplay damage
    UGameplayStatics::ApplyDamage(
        Target,
        DamageAmount,
        OwnerCharacter ? OwnerCharacter->GetController() : nullptr,
        OwnerCharacter,
        UDamageType::StaticClass()
    );
    
    // Also call the interface method if implemented
    IDamageableInterface* DamageableTarget = Cast<IDamageableInterface>(Target);
    if (DamageableTarget)
    {
        DamageableTarget->Execute_ReceiveDamage(Target, DamageAmount);
    }
}
```

**SUCCESS CHECK:** Combat component can be added to character and performs 2D melee attacks

## Step 3: Create Projectile Class for 2D

1. Create new C++ class:
   - Tools → New C++ Class
   - Parent Class: Actor
   - Name: "Projectile"

2. Implement the header:

```cpp
// Projectile.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "Projectile.generated.h"

UCLASS()
class YOURPROJECT_API AProjectile : public AActor
{
    GENERATED_BODY()
    
public:
    AProjectile();
    
    // Called every frame
    virtual void Tick(float DeltaTime) override;
    
    // Damage amount this projectile applies
    UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category="Projectile")
    float Damage = 10.0f;
    
    // Lifetime before auto-destruction
    UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category="Projectile")
    float LifeSpan = 5.0f;
    
    // Visual component
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category="Components")
    class UStaticMeshComponent* MeshComponent;
    
    // Collision component
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category="Components")
    class USphereComponent* CollisionComponent;
    
    // Particle effect for impact
    UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category="Effects")
    class UParticleSystem* ImpactEffect;
    
    // Sound effect for impact
    UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category="Effects")
    class USoundBase* ImpactSound;

protected:
    // Called when the game starts or when spawned
    virtual void BeginPlay() override;
    
    // Collision handling
    UFUNCTION()
    void OnProjectileHit(UPrimitiveComponent* HitComponent, AActor* OtherActor, 
                        UPrimitiveComponent* OtherComp, FVector NormalImpulse, 
                        const FHitResult& Hit);

private:
    // For 2D constraint
    void ConstrainTo2DPlane();
};
```

3. Implement the 2D projectile:

```cpp
// Projectile.cpp
#include "Projectile.h"
#include "Components/SphereComponent.h"
#include "Components/StaticMeshComponent.h"
#include "GameFramework/ProjectileMovementComponent.h"
#include "Kismet/GameplayStatics.h"
#include "Particles/ParticleSystem.h"

AProjectile::AProjectile()
{
    PrimaryActorTick.bCanEverTick = true;
    
    // Set up collision component
    CollisionComponent = CreateDefaultSubobject<USphereComponent>(TEXT("CollisionComponent"));
    CollisionComponent->InitSphereRadius(15.0f);
    CollisionComponent->SetCollisionProfileName("Projectile");
    CollisionComponent->OnComponentHit.AddDynamic(this, &AProjectile::OnProjectileHit);
    RootComponent = CollisionComponent;
    
    // Set up mesh component
    MeshComponent = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("MeshComponent"));
    MeshComponent->SetupAttachment(RootComponent);
    MeshComponent->SetCollisionProfileName("NoCollision");
    
    // Configure for 2D
    SetActorRotation(FRotator(0.0f, 0.0f, 0.0f)); // Align with 2D plane
    
    // Auto-destroy after lifetime
    InitialLifeSpan = LifeSpan;
}

void AProjectile::BeginPlay()
{
    Super::BeginPlay();
}

void AProjectile::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);
    
    // Ensure projectile stays on 2D plane
    ConstrainTo2DPlane();
}

void AProjectile::ConstrainTo2DPlane()
{
    // Force Y position to 0 to stay on 2D plane
    FVector CurrentLocation = GetActorLocation();
    if (CurrentLocation.Y != 0.0f)
    {
        SetActorLocation(FVector(CurrentLocation.X, 0.0f, CurrentLocation.Z));
    }
    
    // Also constrain rotation to face along X axis only
    SetActorRotation(FRotator(0.0f, GetActorForwardVector().X >= 0 ? 0.0f : 180.0f, 0.0f));
}

void AProjectile::OnProjectileHit(UPrimitiveComponent* HitComponent, AActor* OtherActor, 
                                 UPrimitiveComponent* OtherComp, FVector NormalImpulse, 
                                 const FHitResult& Hit)
{
    // Don't hit owner
    if (OtherActor && OtherActor != this && OtherActor != GetOwner())
    {
        // Apply damage
        UGameplayStatics::ApplyDamage(
            OtherActor,
            Damage,
            GetInstigatorController(),
            this,
            UDamageType::StaticClass()
        );
        
        // Spawn impact effects
        if (ImpactEffect)
        {
            UGameplayStatics::SpawnEmitterAtLocation(
                GetWorld(),
                ImpactEffect,
                Hit.Location,
                FRotator::ZeroRotator
            );
        }
        
        // Play impact sound
        if (ImpactSound)
        {
            UGameplayStatics::PlaySoundAtLocation(
                this,
                ImpactSound,
                Hit.Location
            );
        }
        
        // Destroy the projectile
        Destroy();
    }
}
```

**SUCCESS CHECK:** Projectiles travel in 2D plane and interact with targets

## Step 4: Integrate Combat with GameplayAbilitySystem (GAS)

1. Create an attack ability class:
   - Tools → New C++ Class
   - Parent Class: GameplayAbility
   - Name: "GA_MeleeAttack"

2. Implement the ability:

```cpp
// GA_MeleeAttack.h
#pragma once

#include "CoreMinimal.h"
#include "Abilities/GameplayAbility.h"
#include "GA_MeleeAttack.generated.h"

UCLASS()
class YOURPROJECT_API UGA_MeleeAttack : public UGameplayAbility
{
    GENERATED_BODY()
    
public:
    UGA_MeleeAttack();
    
    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle, 
                                const FGameplayAbilityActorInfo* ActorInfo, 
                                const FGameplayAbilityActivationInfo ActivationInfo,
                                const FGameplayEventData* TriggerEventData) override;
    
protected:
    // Called when animation montage completes or is interrupted
    UFUNCTION()
    void OnCompleted();
    
    UFUNCTION()
    void OnCancelled();
    
    UFUNCTION()
    void OnBlendOut();
    
    UFUNCTION()
    void OnInterrupted();
    
    // Called when animation notifies trigger damage phase
    UFUNCTION()
    void ApplyDamage();
    
    // Handle to the active montage
    FGameplayAbilitySpecHandle MontageHandle;
};
```

3. Create a projectile ability:
   - Tools → New C++ Class
   - Parent Class: GameplayAbility
   - Name: "GA_FireProjectile"

**SUCCESS CHECK:** GAS abilities trigger correct combat functionality

## Step 5: Create Blueprint Extensions

1. Create Blueprint based on your combat component
2. Create Blueprint based on projectile class
3. Set up VFX, meshes, and sounds

**SUCCESS CHECK:** Combat feels satisfying with visual and audio feedback

## Step 6: Create Enemy Character with Health System

1. Create base enemy class:
   - Tools → New C++ Class
   - Parent Class: Character
   - Name: "EnemyCharacter"

2. Implement health system and damage response:

```cpp
// Implement DamageableInterface in EnemyCharacter.h
UCLASS()
class YOURPROJECT_API AEnemyCharacter : public ACharacter, public IDamageableInterface
{
    GENERATED_BODY()
    
public:
    AEnemyCharacter();
    
    // Health properties
    UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category="Health")
    float MaxHealth = 100.0f;
    
    UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category="Health")
    float CurrentHealth = 100.0f;
    
    // DamageableInterface implementation
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable)
    void ReceiveDamage(float Amount);
    virtual void ReceiveDamage_Implementation(float Amount);
    
    // Death handling
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable)
    void Die();
    virtual void Die_Implementation();
    
    // Is enemy dead?
    UPROPERTY(BlueprintReadOnly, Category="Health")
    bool bIsDead = false;
    
    // Enemy 2D movement behavior  
    UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category="AI")
    float MovementSpeed = 100.0f;
    
    UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category="AI")
    bool bPatrol = true;
    
    // Direction enemy is facing (-1 for left, 1 for right)
    UPROPERTY(BlueprintReadOnly, Category="AI")
    float CurrentDirection = 1.0f;
    
protected:
    virtual void BeginPlay() override;
    virtual void Tick(float DeltaTime) override;
    
    // Patrolling behavior
    void UpdateMovement(float DeltaTime);
    
    // Check for walls to turn around
    void CheckForObstacles();
};
```

**SUCCESS CHECK:** Enemies respond to damage and have working health system

## Step 7: Integrate Combat Animations

1. Set up animation blueprint for player combat
2. Create attack montages
3. Add animation notifies for:
   - Damage application
   - Projectile spawning
   - Effects/sounds

**SUCCESS CHECK:** Animations match combat actions with proper timing

## Next Steps
Once your combat system is implemented, proceed to setting up the UI system in [ui_system.md](ui_system.md). 