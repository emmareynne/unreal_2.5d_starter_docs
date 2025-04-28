# Gameplay Ability System: Integration (Part 1)

This guide covers integrating the Gameplay Ability System with other game systems for your 2.5D platformer in Unreal Engine 5.5.

## Overview of GAS Integration

While previous guides have covered the core GAS components, this guide focuses on connecting those components with other systems:

- User Interface (HUD, menus, ability icons)
- Animation system
- Input handling
- AI and behavior trees
- Camera systems
- Level design elements

## UI Integration

### 1. Health and Status Bars

Create a widget to display health, energy, and other attributes:

```cpp
// PlatformerAttributeWidget.h
#pragma once

#include "CoreMinimal.h"
#include "Blueprint/UserWidget.h"
#include "GameplayEffectTypes.h"
#include "PlatformerAttributeWidget.generated.h"

class UProgressBar;
class UTextBlock;
class UPlatformerHealthAttributeSet;
class UAbilitySystemComponent;

/**
 * Widget for displaying character attributes in the UI.
 */
UCLASS()
class PLATFORMER_API UPlatformerAttributeWidget : public UUserWidget
{
    GENERATED_BODY()

public:
    // Initialize the widget with the character's ASC
    UFUNCTION(BlueprintCallable, Category = "Attributes")
    void InitFromCharacter(AActor* Character);

protected:
    // Override to set up attribute delegates
    virtual void NativeConstruct() override;
    
    // Override to remove delegates
    virtual void NativeDestruct() override;
    
    // Bind to attribute changes
    void BindAttributeChanges();
    
    // Health attribute changed
    UFUNCTION()
    void OnHealthChanged(const FOnAttributeChangeData& Data);
    
    // Max health attribute changed
    UFUNCTION()
    void OnMaxHealthChanged(const FOnAttributeChangeData& Data);
    
    // Update the health bar UI
    void UpdateHealthBar();
    
    // Health progress bar
    UPROPERTY(meta = (BindWidget))
    UProgressBar* HealthBar;
    
    // Health text
    UPROPERTY(meta = (BindWidget))
    UTextBlock* HealthText;
    
    // Reference to ASC
    UPROPERTY()
    UAbilitySystemComponent* AbilitySystemComponent;
    
    // Cached attribute set
    UPROPERTY()
    const UPlatformerHealthAttributeSet* HealthAttributes;
    
    // Delegate handles
    FDelegateHandle HealthChangedHandle;
    FDelegateHandle MaxHealthChangedHandle;
};
```

```cpp
// PlatformerAttributeWidget.cpp
#include "PlatformerAttributeWidget.h"
#include "Components/ProgressBar.h"
#include "Components/TextBlock.h"
#include "AbilitySystemComponent.h"
#include "AbilitySystemBlueprintLibrary.h"
#include "PlatformerHealthAttributeSet.h"

void UPlatformerAttributeWidget::NativeConstruct()
{
    Super::NativeConstruct();
    
    // Set up initial values
    if (AbilitySystemComponent)
    {
        BindAttributeChanges();
        UpdateHealthBar();
    }
}

void UPlatformerAttributeWidget::NativeDestruct()
{
    // Clean up delegates
    if (AbilitySystemComponent)
    {
        AbilitySystemComponent->GetGameplayAttributeValueChangeDelegate(
            UPlatformerHealthAttributeSet::GetHealthAttribute()).Remove(HealthChangedHandle);
            
        AbilitySystemComponent->GetGameplayAttributeValueChangeDelegate(
            UPlatformerHealthAttributeSet::GetMaxHealthAttribute()).Remove(MaxHealthChangedHandle);
    }
    
    Super::NativeDestruct();
}

void UPlatformerAttributeWidget::InitFromCharacter(AActor* Character)
{
    if (!Character)
    {
        return;
    }
    
    // Get ASC from character
    AbilitySystemComponent = UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent(Character);
    
    if (!AbilitySystemComponent)
    {
        return;
    }
    
    // Get attribute set
    HealthAttributes = AbilitySystemComponent->GetSet<UPlatformerHealthAttributeSet>();
    
    if (HealthAttributes)
    {
        // Initial update
        UpdateHealthBar();
        
        // Bind to attribute changes
        BindAttributeChanges();
    }
}

void UPlatformerAttributeWidget::BindAttributeChanges()
{
    if (!AbilitySystemComponent)
    {
        return;
    }
    
    // Bind to health change
    HealthChangedHandle = AbilitySystemComponent->GetGameplayAttributeValueChangeDelegate(
        UPlatformerHealthAttributeSet::GetHealthAttribute()).AddUObject(
            this, &UPlatformerAttributeWidget::OnHealthChanged);
            
    // Bind to max health change
    MaxHealthChangedHandle = AbilitySystemComponent->GetGameplayAttributeValueChangeDelegate(
        UPlatformerHealthAttributeSet::GetMaxHealthAttribute()).AddUObject(
            this, &UPlatformerAttributeWidget::OnMaxHealthChanged);
}

void UPlatformerAttributeWidget::OnHealthChanged(const FOnAttributeChangeData& Data)
{
    UpdateHealthBar();
}

void UPlatformerAttributeWidget::OnMaxHealthChanged(const FOnAttributeChangeData& Data)
{
    UpdateHealthBar();
}

void UPlatformerAttributeWidget::UpdateHealthBar()
{
    if (!HealthAttributes)
    {
        return;
    }
    
    // Get current health values
    float Health = HealthAttributes->GetHealth();
    float MaxHealth = HealthAttributes->GetMaxHealth();
    
    // Update progress bar
    if (HealthBar)
    {
        HealthBar->SetPercent(MaxHealth > 0 ? Health / MaxHealth : 0);
    }
    
    // Update text
    if (HealthText)
    {
        HealthText->SetText(FText::FromString(FString::Printf(TEXT("%d / %d"), 
            FMath::RoundToInt(Health), FMath::RoundToInt(MaxHealth))));
    }
}
```

### 2. Ability Cooldown Display

Create a widget to display ability cooldowns:

```cpp
// PlatformerAbilityCooldownWidget.h
#pragma once

#include "CoreMinimal.h"
#include "Blueprint/UserWidget.h"
#include "GameplayTagContainer.h"
#include "PlatformerAbilityCooldownWidget.generated.h"

class UImage;
class UProgressBar;
class UAbilitySystemComponent;

/**
 * Widget for displaying ability cooldowns.
 */
UCLASS()
class PLATFORMER_API UPlatformerAbilityCooldownWidget : public UUserWidget
{
    GENERATED_BODY()

public:
    // Initialize with ASC and ability tag
    UFUNCTION(BlueprintCallable, Category = "Ability")
    void InitializeWithAbility(UAbilitySystemComponent* InASC, FGameplayTagContainer InAbilityTags, FGameplayTagContainer InCooldownTags);

protected:
    // Override to set up timer
    virtual void NativeConstruct() override;
    
    // Override to clean up timer
    virtual void NativeDestruct() override;
    
    // Update cooldown display
    void UpdateCooldown();
    
    // Ability icon image
    UPROPERTY(meta = (BindWidget))
    UImage* AbilityIcon;
    
    // Cooldown overlay
    UPROPERTY(meta = (BindWidget))
    UProgressBar* CooldownProgress;
    
    // ASC reference
    UPROPERTY()
    UAbilitySystemComponent* AbilitySystemComponent;
    
    // Tags for this ability
    UPROPERTY()
    FGameplayTagContainer AbilityTags;
    
    // Tags for the cooldown
    UPROPERTY()
    FGameplayTagContainer CooldownTags;
    
    // Timer handle for updating cooldown
    FTimerHandle UpdateTimerHandle;
    
    // Is this ability currently on cooldown?
    bool bIsOnCooldown;
    
    // Current remaining cooldown time
    float RemainingCooldown;
    
    // Total cooldown duration
    float CooldownDuration;
};
```

```cpp
// PlatformerAbilityCooldownWidget.cpp
#include "PlatformerAbilityCooldownWidget.h"
#include "Components/Image.h"
#include "Components/ProgressBar.h"
#include "AbilitySystemComponent.h"
#include "GameplayEffectTypes.h"

void UPlatformerAbilityCooldownWidget::NativeConstruct()
{
    Super::NativeConstruct();
    
    // Start update timer
    GetWorld()->GetTimerManager().SetTimer(
        UpdateTimerHandle,
        this,
        &UPlatformerAbilityCooldownWidget::UpdateCooldown,
        0.1f, // Update 10 times per second
        true
    );
    
    // Initial update
    UpdateCooldown();
}

void UPlatformerAbilityCooldownWidget::NativeDestruct()
{
    // Clear timer
    if (UpdateTimerHandle.IsValid())
    {
        GetWorld()->GetTimerManager().ClearTimer(UpdateTimerHandle);
    }
    
    Super::NativeDestruct();
}

void UPlatformerAbilityCooldownWidget::InitializeWithAbility(UAbilitySystemComponent* InASC, 
                                                           FGameplayTagContainer InAbilityTags, 
                                                           FGameplayTagContainer InCooldownTags)
{
    AbilitySystemComponent = InASC;
    AbilityTags = InAbilityTags;
    CooldownTags = InCooldownTags;
    
    UpdateCooldown();
}

void UPlatformerAbilityCooldownWidget::UpdateCooldown()
{
    if (!AbilitySystemComponent)
    {
        return;
    }
    
    // Check if ability is on cooldown
    FGameplayEffectQuery CooldownQuery;
    CooldownQuery.OwningTagQuery = FGameplayTagQuery::MakeQuery_MatchAnyTags(CooldownTags);
    
    TArray<FActiveGameplayEffectHandle> ActiveCooldowns = 
        AbilitySystemComponent->GetActiveEffects(CooldownQuery);
    
    bIsOnCooldown = ActiveCooldowns.Num() > 0;
    
    if (bIsOnCooldown && ActiveCooldowns.Num() > 0)
    {
        // Get cooldown info
        FActiveGameplayEffect* Cooldown = AbilitySystemComponent->GetActiveGameplayEffect(ActiveCooldowns[0]);
        if (Cooldown)
        {
            // Calculate time remaining and duration
            RemainingCooldown = Cooldown->GetTimeRemaining(AbilitySystemComponent->GetWorld()->GetTimeSeconds());
            CooldownDuration = Cooldown->GetDuration();
            
            // Update progress bar
            if (CooldownProgress)
            {
                float CooldownPercent = 1.0f - (RemainingCooldown / CooldownDuration);
                CooldownProgress->SetPercent(CooldownPercent);
                CooldownProgress->SetVisibility(ESlateVisibility::Visible);
            }
        }
    }
    else
    {
        // Not on cooldown
        if (CooldownProgress)
        {
            CooldownProgress->SetVisibility(ESlateVisibility::Hidden);
        }
    }
}
```

### 3. Ability UI Manager

Create a manager to handle all ability-related UI:

```cpp
// PlatformerAbilityUIManager.h
#pragma once

#include "CoreMinimal.h"
#include "Blueprint/UserWidget.h"
#include "GameplayTagContainer.h"
#include "PlatformerAbilityUIManager.generated.h"

class UPlatformerAttributeWidget;
class UPlatformerAbilityCooldownWidget;
class UAbilitySystemComponent;
class UHorizontalBox;

/**
 * Manager widget for all ability UI components.
 */
UCLASS()
class PLATFORMER_API UPlatformerAbilityUIManager : public UUserWidget
{
    GENERATED_BODY()

public:
    // Initialize from character
    UFUNCTION(BlueprintCallable, Category = "UI")
    void InitializeFromCharacter(AActor* Character);
    
    // Add a cooldown widget for an ability
    UFUNCTION(BlueprintCallable, Category = "UI")
    void AddAbilityCooldown(FGameplayTagContainer AbilityTags, FGameplayTagContainer CooldownTags);

protected:
    // Attribute widget reference
    UPROPERTY(meta = (BindWidget))
    UPlatformerAttributeWidget* AttributeWidget;
    
    // Container for cooldown widgets
    UPROPERTY(meta = (BindWidget))
    UHorizontalBox* CooldownContainer;
    
    // ASC reference
    UPROPERTY()
    UAbilitySystemComponent* AbilitySystemComponent;
    
    // Cooldown widget class
    UPROPERTY(EditDefaultsOnly, Category = "UI")
    TSubclassOf<UPlatformerAbilityCooldownWidget> CooldownWidgetClass;
};
```

```cpp
// PlatformerAbilityUIManager.cpp
#include "PlatformerAbilityUIManager.h"
#include "PlatformerAttributeWidget.h"
#include "PlatformerAbilityCooldownWidget.h"
#include "Components/HorizontalBox.h"
#include "AbilitySystemBlueprintLibrary.h"

void UPlatformerAbilityUIManager::InitializeFromCharacter(AActor* Character)
{
    if (!Character)
    {
        return;
    }
    
    // Get ASC
    AbilitySystemComponent = UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent(Character);
    
    if (!AbilitySystemComponent)
    {
        return;
    }
    
    // Initialize attribute widget
    if (AttributeWidget)
    {
        AttributeWidget->InitFromCharacter(Character);
    }
}

void UPlatformerAbilityUIManager::AddAbilityCooldown(FGameplayTagContainer AbilityTags, FGameplayTagContainer CooldownTags)
{
    if (!AbilitySystemComponent || !CooldownWidgetClass || !CooldownContainer)
    {
        return;
    }
    
    // Create new cooldown widget
    UPlatformerAbilityCooldownWidget* NewCooldownWidget = 
        CreateWidget<UPlatformerAbilityCooldownWidget>(this, CooldownWidgetClass);
        
    if (NewCooldownWidget)
    {
        // Initialize and add to container
        NewCooldownWidget->InitializeWithAbility(AbilitySystemComponent, AbilityTags, CooldownTags);
        CooldownContainer->AddChild(NewCooldownWidget);
    }
}
```

## Animation Integration

### 1. Animation Blueprint Integration

Create an animation instance class that responds to GAS:

```cpp
// PlatformerAnimInstance.h
#pragma once

#include "CoreMinimal.h"
#include "Animation/AnimInstance.h"
#include "GameplayTagContainer.h"
#include "AbilitySystemInterface.h"
#include "PlatformerAnimInstance.generated.h"

class UAbilitySystemComponent;

/**
 * Animation instance for platformer characters.
 */
UCLASS()
class PLATFORMER_API UPlatformerAnimInstance : public UAnimInstance
{
    GENERATED_BODY()

public:
    UPlatformerAnimInstance();
    
    // Native initialization override
    virtual void NativeInitializeAnimation() override;
    
    // Native update override
    virtual void NativeUpdateAnimation(float DeltaSeconds) override;
    
    // Handle gameplay tag changes
    UFUNCTION()
    void OnGameplayTagChanged(const FGameplayTag Tag, int32 NewCount);

protected:
    // Check character tags and update animation variables
    void UpdateFromGameplayTags();
    
    // Movement state values
    UPROPERTY(BlueprintReadOnly, Category = "Movement")
    bool bIsInAir;
    
    UPROPERTY(BlueprintReadOnly, Category = "Movement")
    bool bIsDashing;
    
    UPROPERTY(BlueprintReadOnly, Category = "Movement")
    bool bIsDoubleJumping;
    
    UPROPERTY(BlueprintReadOnly, Category = "Movement")
    float Speed;
    
    UPROPERTY(BlueprintReadOnly, Category = "Movement")
    float Direction;
    
    // Reference to character's ASC
    UPROPERTY()
    UAbilitySystemComponent* AbilitySystemComponent;
    
    // Cache owning character
    UPROPERTY()
    ACharacter* OwningCharacter;
    
    // Array of delegate handles for tag callbacks
    TArray<FDelegateHandle> TagDelegateHandles;
};
```

```cpp
// PlatformerAnimInstance.cpp
#include "PlatformerAnimInstance.h"
#include "AbilitySystemComponent.h"
#include "AbilitySystemBlueprintLibrary.h"
#include "GameFramework/Character.h"
#include "GameFramework/CharacterMovementComponent.h"

UPlatformerAnimInstance::UPlatformerAnimInstance()
{
    bIsInAir = false;
    bIsDashing = false;
    bIsDoubleJumping = false;
    Speed = 0.0f;
    Direction = 0.0f;
}

void UPlatformerAnimInstance::NativeInitializeAnimation()
{
    Super::NativeInitializeAnimation();
    
    // Get owner
    OwningCharacter = Cast<ACharacter>(GetOwningActor());
    
    if (OwningCharacter)
    {
        // Get ASC
        AbilitySystemComponent = UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent(OwningCharacter);
        
        if (AbilitySystemComponent)
        {
            // Register for tag callbacks
            FGameplayTagContainer TrackedTags;
            TrackedTags.AddTag(FGameplayTag::RequestGameplayTag("State.InAir"));
            TrackedTags.AddTag(FGameplayTag::RequestGameplayTag("State.Dashing"));
            TrackedTags.AddTag(FGameplayTag::RequestGameplayTag("State.DoubleJumping"));
            
            for (const FGameplayTag& Tag : TrackedTags)
            {
                FDelegateHandle Handle = AbilitySystemComponent->RegisterGameplayTagEvent(
                    Tag, EGameplayTagEventType::NewOrRemoved).AddUObject(
                        this, &UPlatformerAnimInstance::OnGameplayTagChanged);
                
                TagDelegateHandles.Add(Handle);
            }
            
            // Initialize values from current tags
            UpdateFromGameplayTags();
        }
    }
}

void UPlatformerAnimInstance::NativeUpdateAnimation(float DeltaSeconds)
{
    Super::NativeUpdateAnimation(DeltaSeconds);
    
    if (!OwningCharacter)
    {
        return;
    }
    
    // Update movement values
    UCharacterMovementComponent* MovementComponent = OwningCharacter->GetCharacterMovement();
    if (MovementComponent)
    {
        // Get speed
        Speed = MovementComponent->Velocity.Size2D();
        
        // Get direction (-180 to 180 degrees)
        Direction = CalculateDirection(MovementComponent->Velocity, OwningCharacter->GetActorRotation());
        
        // Update air state from movement component as backup
        if (!AbilitySystemComponent)
        {
            bIsInAir = MovementComponent->IsFalling();
        }
    }
}

void UPlatformerAnimInstance::OnGameplayTagChanged(const FGameplayTag Tag, int32 NewCount)
{
    UpdateFromGameplayTags();
}

void UPlatformerAnimInstance::UpdateFromGameplayTags()
{
    if (!AbilitySystemComponent)
    {
        return;
    }
    
    // Update state variables based on gameplay tags
    bIsInAir = AbilitySystemComponent->HasMatchingGameplayTag(FGameplayTag::RequestGameplayTag("State.InAir"));
    bIsDashing = AbilitySystemComponent->HasMatchingGameplayTag(FGameplayTag::RequestGameplayTag("State.Dashing"));
    bIsDoubleJumping = AbilitySystemComponent->HasMatchingGameplayTag(FGameplayTag::RequestGameplayTag("State.DoubleJumping"));
}
```

### 2. Animation Notify Integration

Create animation notifies that can trigger abilities or gameplay cues:

```cpp
// PlatformerAbilityAnimNotify.h
#pragma once

#include "CoreMinimal.h"
#include "Animation/AnimNotifies/AnimNotify.h"
#include "GameplayTagContainer.h"
#include "PlatformerAbilityAnimNotify.generated.h"

/**
 * Animation notify for triggering abilities or gameplay cues.
 */
UCLASS()
class PLATFORMER_API UPlatformerAbilityAnimNotify : public UAnimNotify
{
    GENERATED_BODY()

public:
    // Override notify function
    virtual void Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation) override;
    
protected:
    // Tag to send as a gameplay event
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "GameplayEvent")
    FGameplayTag EventTag;
    
    // Optional gameplay cue to trigger
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "GameplayEvent")
    FGameplayTag GameplayCueTag;
    
    // Whether to activate an ability
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "GameplayEvent")
    bool bActivateAbility;
    
    // Whether to trigger a gameplay cue
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "GameplayEvent")
    bool bTriggerGameplayCue;
    
    // Optional event data
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "GameplayEvent")
    float Magnitude;
};
```

```cpp
// PlatformerAbilityAnimNotify.cpp
#include "PlatformerAbilityAnimNotify.h"
#include "AbilitySystemBlueprintLibrary.h"
#include "AbilitySystemComponent.h"
#include "GameFramework/Character.h"

void UPlatformerAbilityAnimNotify::Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation)
{
    Super::Notify(MeshComp, Animation);
    
    if (!MeshComp || !EventTag.IsValid())
    {
        return;
    }
    
    // Get owning actor
    AActor* Owner = MeshComp->GetOwner();
    if (!Owner)
    {
        return;
    }
    
    // Get ability system component
    UAbilitySystemComponent* AbilitySystemComponent = 
        UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent(Owner);
        
    if (!AbilitySystemComponent)
    {
        return;
    }
    
    // Send event data
    FGameplayEventData EventData;
    EventData.EventTag = EventTag;
    EventData.Instigator = Owner;
    EventData.Target = Owner;
    EventData.EventMagnitude = Magnitude;
    
    // Option 1: Activate ability via event
    if (bActivateAbility)
    {
        AbilitySystemComponent->HandleGameplayEvent(EventTag, &EventData);
    }
    
    // Option 2: Trigger gameplay cue
    if (bTriggerGameplayCue && GameplayCueTag.IsValid())
    {
        FGameplayCueParameters CueParams;
        CueParams.Location = Owner->GetActorLocation();
        CueParams.Normal = FVector::UpVector;
        CueParams.Instigator = Owner;
        CueParams.EffectCauser = Owner;
        CueParams.SourceObject = Owner;
        CueParams.Magnitude = Magnitude;
        
        AbilitySystemComponent->ExecuteGameplayCue(GameplayCueTag, CueParams);
    }
}
``` 