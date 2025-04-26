# Gameplay Systems for 2.5D Platformer

## Core Gameplay Systems

### 1. Health System

#### Health Component (C++)
```cpp
// PlatformerHealthComponent.h
#pragma once

#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "PlatformerHealthComponent.generated.h"

DECLARE_DYNAMIC_MULTICAST_DELEGATE_FourParams(FOnHealthChangedSignature, UPlatformerHealthComponent*, HealthComp, float, Health, float, HealthDelta, const class UDamageType*, DamageType);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnDeathSignature, AActor*, DeadActor);

UCLASS( ClassGroup=(Custom), meta=(BlueprintSpawnableComponent) )
class PLATFORMER_API UPlatformerHealthComponent : public UActorComponent
{
	GENERATED_BODY()

public:	
	UPlatformerHealthComponent();

	UPROPERTY(BlueprintAssignable, Category = "Events")
	FOnHealthChangedSignature OnHealthChanged;
	
	UPROPERTY(BlueprintAssignable, Category = "Events")
	FOnDeathSignature OnDeath;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Health")
	float MaxHealth;
	
	UPROPERTY(BlueprintReadOnly, Category = "Health")
	float CurrentHealth;
	
	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Health")
	bool bInvulnerable;
	
	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Health")
	float InvulnerabilityTime;
	
	UFUNCTION(BlueprintCallable, Category = "Health")
	void Heal(float HealAmount);
	
	UFUNCTION(BlueprintCallable, Category = "Health")
	void SetInvulnerable(bool bNewInvulnerable);

protected:
	virtual void BeginPlay() override;
	
	UFUNCTION()
	void HandleTakeAnyDamage(AActor* DamagedActor, float Damage, const class UDamageType* DamageType, class AController* InstigatedBy, AActor* DamageCauser);
	
	FTimerHandle InvulnerabilityTimer;
	bool bIsDead;

private:
	void EndInvulnerability();
};
```

```cpp
// PlatformerHealthComponent.cpp
#include "PlatformerHealthComponent.h"
#include "GameFramework/Actor.h"
#include "TimerManager.h"

UPlatformerHealthComponent::UPlatformerHealthComponent()
{
	MaxHealth = 100.0f;
	bInvulnerable = false;
	InvulnerabilityTime = 1.0f;
	bIsDead = false;
}

void UPlatformerHealthComponent::BeginPlay()
{
	Super::BeginPlay();
	CurrentHealth = MaxHealth;
	
	AActor* Owner = GetOwner();
	if (Owner)
	{
		Owner->OnTakeAnyDamage.AddDynamic(this, &UPlatformerHealthComponent::HandleTakeAnyDamage);
	}
}

void UPlatformerHealthComponent::HandleTakeAnyDamage(AActor* DamagedActor, float Damage, const UDamageType* DamageType, AController* InstigatedBy, AActor* DamageCauser)
{
	if (Damage <= 0.0f || bIsDead || bInvulnerable)
	{
		return;
	}
	
	CurrentHealth = FMath::Clamp(CurrentHealth - Damage, 0.0f, MaxHealth);
	
	OnHealthChanged.Broadcast(this, CurrentHealth, -Damage, DamageType);
	
	if (CurrentHealth <= 0.0f)
	{
		bIsDead = true;
		OnDeath.Broadcast(GetOwner());
	}
	else 
	{
		// Make temporarily invulnerable after taking damage
		SetInvulnerable(true);
		GetWorld()->GetTimerManager().SetTimer(InvulnerabilityTimer, this, &UPlatformerHealthComponent::EndInvulnerability, InvulnerabilityTime, false);
	}
}

void UPlatformerHealthComponent::Heal(float HealAmount)
{
	if (HealAmount <= 0.0f || CurrentHealth <= 0.0f || CurrentHealth >= MaxHealth)
	{
		return;
	}
	
	CurrentHealth = FMath::Clamp(CurrentHealth + HealAmount, 0.0f, MaxHealth);
	OnHealthChanged.Broadcast(this, CurrentHealth, HealAmount, nullptr);
}

void UPlatformerHealthComponent::SetInvulnerable(bool bNewInvulnerable)
{
	bInvulnerable = bNewInvulnerable;
}

void UPlatformerHealthComponent::EndInvulnerability()
{
	bInvulnerable = false;
}
```

### 2. Combat System

#### Damage Application
```cpp
// Apply damage to enemies (Example from PlatformerCharacter.cpp)
bool APlatformerCharacter::AttackEnemy()
{
	// Check if we are in attack animation
	if (!bIsAttacking)
	{
		return false;
	}
	
	// Get attack hit box location/extent
	FVector AttackCenter = GetActorLocation() + (GetActorForwardVector() * AttackRange);
	FVector BoxExtent(30.0f, 60.0f, 50.0f);
	
	// Find overlapping actors
	TArray<FOverlapResult> Overlaps;
	FCollisionQueryParams QueryParams;
	QueryParams.AddIgnoredActor(this);
	
	GetWorld()->OverlapMultiByObjectType(
		Overlaps,
		AttackCenter,
		FQuat::Identity,
		FCollisionObjectQueryParams(ECC_Pawn),
		FCollisionShape::MakeBox(BoxExtent),
		QueryParams
	);
	
	bool bHitSomething = false;
	
	// Apply damage to all overlapping enemies
	for (auto& Overlap : Overlaps)
	{
		if (Overlap.GetActor()->Implements<UEnemyInterface>())
		{
			float DamageAmount = AttackDamage;
			FDamageEvent DamageEvent;
			Overlap.GetActor()->TakeDamage(DamageAmount, DamageEvent, GetController(), this);
			bHitSomething = true;
			
			// Visual/audio feedback
			if (HitEnemySound)
			{
				UGameplayStatics::PlaySoundAtLocation(this, HitEnemySound, Overlap.GetActor()->GetActorLocation());
			}
			
			if (HitEnemyParticle)
			{
				UGameplayStatics::SpawnEmitterAtLocation(GetWorld(), HitEnemyParticle, Overlap.GetActor()->GetActorLocation());
			}
		}
	}
	
	return bHitSomething;
}
```

#### Simple Enemy AI
```cpp
// PatrollingEnemy.h (Simplified)
UCLASS()
class PLATFORMER_API APatrollingEnemy : public ACharacter, public IEnemyInterface
{
	GENERATED_BODY()
	
public:
	APatrollingEnemy();
	
	virtual void Tick(float DeltaTime) override;
	
	// IEnemyInterface
	virtual float TakeDamage(float DamageAmount, FDamageEvent const& DamageEvent, AController* EventInstigator, AActor* DamageCauser) override;
	
protected:
	virtual void BeginPlay() override;
	
	UPROPERTY(EditAnywhere, Category = "Patrol")
	TArray<AActor*> PatrolPoints;
	
	UPROPERTY(EditAnywhere, Category = "Patrol")
	float PatrolSpeed = 200.0f;
	
	UPROPERTY(EditAnywhere, Category = "Patrol")
	float WaitTime = 1.0f;
	
	UPROPERTY(EditAnywhere, Category = "Attack")
	float DetectionRange = 500.0f;
	
	UPROPERTY(EditAnywhere, Category = "Attack")
	float AttackRange = 100.0f;
	
	UPROPERTY(EditAnywhere, Category = "Attack")
	float AttackDamage = 20.0f;
	
	UPROPERTY(EditAnywhere, Category = "Attack")
	float AttackCooldown = 2.0f;
	
private:
	int CurrentPatrolPoint = 0;
	bool bIsWaiting = false;
	FTimerHandle WaitTimerHandle;
	FTimerHandle AttackTimerHandle;
	bool bCanAttack = true;
	
	UPlatformerHealthComponent* HealthComp;
	
	void MoveToNextPatrolPoint();
	void WaitAtPatrolPoint();
	void ResumePatrol();
	void LookForPlayer();
	void ChasePlayer(APlatformerCharacter* Player);
	void AttackPlayer(APlatformerCharacter* Player);
	void ResetAttackCooldown();
	
	void OnHealthChanged(UPlatformerHealthComponent* HealthComponent, float Health, float HealthDelta, const UDamageType* DamageType);
	void OnDeath(AActor* DeadActor);
};
```

### 3. Collectible System

#### Base Collectible Class
```cpp
// Collectible.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "Collectible.generated.h"

UCLASS(Abstract)
class PLATFORMER_API ACollectible : public AActor
{
	GENERATED_BODY()
	
public:	
	ACollectible();

protected:
	virtual void BeginPlay() override;

	UPROPERTY(VisibleAnywhere, Category = "Components")
	UStaticMeshComponent* MeshComponent;
	
	UPROPERTY(VisibleAnywhere, Category = "Components")
	class USphereComponent* CollectionSphere;
	
	UPROPERTY(EditAnywhere, Category = "Effects")
	USoundBase* PickupSound;
	
	UPROPERTY(EditAnywhere, Category = "Effects")
	UParticleSystem* PickupEffect;
	
	UPROPERTY(EditAnywhere, Category = "Movement")
	bool bRotates = true;
	
	UPROPERTY(EditAnywhere, Category = "Movement", meta = (EditCondition = "bRotates"))
	float RotationRate = 100.0f;
	
	UPROPERTY(EditAnywhere, Category = "Movement")
	bool bBobs = true;
	
	UPROPERTY(EditAnywhere, Category = "Movement", meta = (EditCondition = "bBobs"))
	float BobSpeed = 2.0f;
	
	UPROPERTY(EditAnywhere, Category = "Movement", meta = (EditCondition = "bBobs"))
	float BobHeight = 10.0f;
	
	UFUNCTION()
	virtual void OnOverlapBegin(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult);
	
	// Override in child classes
	UFUNCTION(BlueprintNativeEvent)
	void OnCollected(class APlatformerCharacter* Character);
	virtual void OnCollected_Implementation(class APlatformerCharacter* Character);

public:	
	virtual void Tick(float DeltaTime) override;

private:
	FVector InitialLocation;
	float RunningTime;
};
```

#### Coin Collectible Implementation
```cpp
// CoinCollectible.h
#pragma once

#include "CoreMinimal.h"
#include "Collectible.h"
#include "CoinCollectible.generated.h"

UCLASS()
class PLATFORMER_API ACoinCollectible : public ACollectible
{
	GENERATED_BODY()
	
public:
	ACoinCollectible();
	
protected:
	UPROPERTY(EditAnywhere, Category = "Coin")
	int32 CoinValue = 1;
	
	virtual void OnCollected_Implementation(class APlatformerCharacter* Character) override;
};
```

```cpp
// CoinCollectible.cpp
#include "CoinCollectible.h"
#include "PlatformerCharacter.h"
#include "PlatformerGameMode.h"

ACoinCollectible::ACoinCollectible()
{
	// Set up default mesh
	static ConstructorHelpers::FObjectFinder<UStaticMesh> CoinMeshObj(TEXT("/Game/Platformer/Meshes/SM_Coin"));
	if (CoinMeshObj.Succeeded())
	{
		MeshComponent->SetStaticMesh(CoinMeshObj.Object);
		MeshComponent->SetRelativeScale3D(FVector(0.5f, 0.5f, 0.05f));
	}
	
	// Set up default material
	static ConstructorHelpers::FObjectFinder<UMaterialInterface> CoinMaterialObj(TEXT("/Game/Platformer/Materials/M_Coin"));
	if (CoinMaterialObj.Succeeded())
	{
		MeshComponent->SetMaterial(0, CoinMaterialObj.Object);
	}
}

void ACoinCollectible::OnCollected_Implementation(APlatformerCharacter* Character)
{
	Super::OnCollected_Implementation(Character);
	
	// Add coins to player/game state
	if (Character)
	{
		APlatformerGameMode* GameMode = Cast<APlatformerGameMode>(GetWorld()->GetAuthGameMode());
		if (GameMode)
		{
			GameMode->AddCoins(CoinValue);
		}
	}
}
```

### 4. Game State and Progress Tracking

```cpp
// PlatformerGameMode.h (Simplified)
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/GameModeBase.h"
#include "PlatformerGameMode.generated.h"

UCLASS()
class PLATFORMER_API APlatformerGameMode : public AGameModeBase
{
	GENERATED_BODY()
	
public:
	APlatformerGameMode();
	
	UFUNCTION(BlueprintCallable, Category = "Game")
	void AddCoins(int32 Amount);
	
	UFUNCTION(BlueprintCallable, Category = "Game")
	int32 GetCoins() const;
	
	UFUNCTION(BlueprintCallable, Category = "Game")
	void CompleteLevel(bool bSuccess);
	
	UFUNCTION(BlueprintCallable, Category = "Game")
	void RestartLevel();
	
	UFUNCTION(BlueprintCallable, Category = "Game")
	void SaveGame();
	
	UFUNCTION(BlueprintCallable, Category = "Game")
	void LoadGame();
	
protected:
	UPROPERTY(BlueprintReadOnly, Category = "Game")
	int32 CoinsCollected;
	
	UPROPERTY(BlueprintReadOnly, Category = "Game")
	int32 CurrentLevel;
	
	UPROPERTY(BlueprintReadOnly, Category = "Game")
	TArray<bool> LevelsCompleted;
	
	UPROPERTY(EditDefaultsOnly, Category = "Game")
	TArray<FName> LevelNames;
	
	virtual void BeginPlay() override;
};
```

## User Interface Systems

### 1. HUD Implementation

```cpp
// PlatformerHUD.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/HUD.h"
#include "PlatformerHUD.generated.h"

UCLASS()
class PLATFORMER_API APlatformerHUD : public AHUD
{
	GENERATED_BODY()
	
public:
	APlatformerHUD();
	
	virtual void DrawHUD() override;
	
	UFUNCTION(BlueprintCallable, Category = "HUD")
	void UpdateHealth(float NewHealth, float MaxHealth);
	
	UFUNCTION(BlueprintCallable, Category = "HUD")
	void UpdateCoins(int32 NewCoins);
	
	UFUNCTION(BlueprintImplementableEvent, Category = "HUD")
	void ShowGameOverScreen(bool bSuccess);
	
	UFUNCTION(BlueprintImplementableEvent, Category = "HUD")
	void ShowPauseMenu();
	
	UFUNCTION(BlueprintImplementableEvent, Category = "HUD")
	void HidePauseMenu();
	
protected:
	UPROPERTY(EditDefaultsOnly, Category = "HUD")
	UTexture2D* HealthBarTexture;
	
	UPROPERTY(EditDefaultsOnly, Category = "HUD")
	UTexture2D* HealthBarBGTexture;
	
	UPROPERTY(EditDefaultsOnly, Category = "HUD")
	UTexture2D* CoinIconTexture;
	
	UPROPERTY(EditDefaultsOnly, Category = "UI Classes")
	TSubclassOf<class UUserWidget> GameOverWidgetClass;
	
	UPROPERTY(EditDefaultsOnly, Category = "UI Classes")
	TSubclassOf<class UUserWidget> PauseMenuWidgetClass;
	
private:
	float CurrentHealth;
	float MaxHealth;
	int32 CurrentCoins;
	
	UPROPERTY()
	class UUserWidget* CurrentPauseMenu;
	
	UPROPERTY()
	class UUserWidget* CurrentGameOverScreen;
};
```

## Implementation Checklist

- [ ] **Health System**
  - [ ] Create the HealthComponent class
  - [ ] Add health pickup items
  - [ ] Visual feedback for damage (flash character, particles)
  - [ ] Test health system in isolation

- [ ] **Combat System**
  - [ ] Implement player attack mechanics
  - [ ] Create enemy AI with patrolling and attacking
  - [ ] Add enemy variety with different behaviors
  - [ ] Test combat with various enemy types

- [ ] **Collectible System**
  - [ ] Base collectible class with common functionality
  - [ ] Coin collectibles for score
  - [ ] Special power-up collectibles
  - [ ] Save/load collectible state

- [ ] **Game State**
  - [ ] Track player progress across levels
  - [ ] Save/load game functionality
  - [ ] Level completion criteria
  - [ ] Game over conditions

- [ ] **User Interface**
  - [ ] Health bar display
  - [ ] Coin counter
  - [ ] Pause menu
  - [ ] Game over screen
  - [ ] Level select menu

## Testing Guidelines

Test each gameplay system individually before integration:

1. Create a test map with only the required components
2. Use the Blueprint Functional Tests framework (see TestingGuide.md)
3. Write specific tests for:
   - Health damage and recovery
   - Combat hit detection
   - Collectible behavior
   - Game state persistence

## Performance Considerations

- Pool frequently spawned objects like projectiles and effects
- Use LODs for complex meshes
- Profile gameplay systems regularly to identify bottlenecks
- Consider network replication costs for multiplayer potential 