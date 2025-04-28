# Gameplay Ability System: Attribute Sets

This guide covers the implementation of Attribute Sets for your 2.5D platformer using Unreal Engine 5.5's Gameplay Ability System. Attributes are the numerical values that define your character's capabilities and stats.

## What Are Attribute Sets?

Attribute Sets are collections of `FGameplayAttributeData` properties that represent various character statistics (health, stamina, movement speed, etc.). They provide:

- Structured organization of related attributes
- Built-in replication for multiplayer games
- Callback functions for responding to attribute changes
- Methods for clamping and validating values

For a 2.5D platformer, you'll typically need attributes for:
- Health and damage handling
- Movement capabilities (jump height, dash distance)
- Special abilities (cooldowns, resource costs)
- Character progression (experience, levels, skill points)

## Creating Attribute Sets

### Basic Attribute Set Structure

For a 2.5D platformer, we'll create two attribute sets:
1. **Health attributes** - Handles health, damage, and shields
2. **Movement attributes** - Handles movement speed, jump height, etc.

### 1. Health Attribute Set

```cpp
// PlatformerHealthAttributeSet.h
#pragma once

#include "CoreMinimal.h"
#include "AttributeSet.h"
#include "AbilitySystemComponent.h"
#include "PlatformerHealthAttributeSet.generated.h"

// Uses macros from AttributeSet.h
#define ATTRIBUTE_ACCESSORS(ClassName, PropertyName) \
    GAMEPLAYATTRIBUTE_PROPERTY_GETTER(ClassName, PropertyName) \
    GAMEPLAYATTRIBUTE_VALUE_GETTER(PropertyName) \
    GAMEPLAYATTRIBUTE_VALUE_SETTER(PropertyName) \
    GAMEPLAYATTRIBUTE_VALUE_INITTER(PropertyName)

/**
 * Attribute set for health and damage-related attributes in the platformer game.
 */
UCLASS()
class PLATFORMER_API UPlatformerHealthAttributeSet : public UAttributeSet
{
    GENERATED_BODY()

public:
    UPlatformerHealthAttributeSet();

    // Attribute replication
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
    
    // Attribute changes
    virtual void PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue) override;
    virtual void PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data) override;

    // Health attributes
    UPROPERTY(BlueprintReadOnly, Category = "Health", ReplicatedUsing = OnRep_Health)
    FGameplayAttributeData Health;
    ATTRIBUTE_ACCESSORS(UPlatformerHealthAttributeSet, Health)

    UPROPERTY(BlueprintReadOnly, Category = "Health", ReplicatedUsing = OnRep_MaxHealth)
    FGameplayAttributeData MaxHealth;
    ATTRIBUTE_ACCESSORS(UPlatformerHealthAttributeSet, MaxHealth)

    // Meta attribute for damage (not replicated, used for calculations)
    UPROPERTY(BlueprintReadOnly, Category = "Damage")
    FGameplayAttributeData Damage;
    ATTRIBUTE_ACCESSORS(UPlatformerHealthAttributeSet, Damage)

protected:
    // Replication callbacks
    UFUNCTION()
    virtual void OnRep_Health(const FGameplayAttributeData& OldHealth);

    UFUNCTION()
    virtual void OnRep_MaxHealth(const FGameplayAttributeData& OldMaxHealth);

    // Helper function for attribute changes
    void AdjustAttributeForMaxChange(FGameplayAttributeData& AffectedAttribute, const FGameplayAttributeData& MaxAttribute, float NewMaxValue, const FGameplayAttribute& AffectedAttributeProperty);
};
```

```cpp
// PlatformerHealthAttributeSet.cpp
#include "PlatformerHealthAttributeSet.h"
#include "Net/UnrealNetwork.h"
#include "GameplayEffectExtension.h"
#include "GameplayEffectTypes.h"

UPlatformerHealthAttributeSet::UPlatformerHealthAttributeSet()
{
    // Initialize default values
    InitHealth(100.0f);
    InitMaxHealth(100.0f);
    InitDamage(0.0f);
}

void UPlatformerHealthAttributeSet::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME_CONDITION_NOTIFY(UPlatformerHealthAttributeSet, Health, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UPlatformerHealthAttributeSet, MaxHealth, COND_None, REPNOTIFY_Always);
}

void UPlatformerHealthAttributeSet::PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue)
{
    Super::PreAttributeChange(Attribute, NewValue);

    // Clamp health to max health
    if (Attribute == GetHealthAttribute())
    {
        NewValue = FMath::Clamp(NewValue, 0.0f, GetMaxHealth());
    }
    else if (Attribute == GetMaxHealthAttribute())
    {
        // When MaxHealth changes, adjust current Health to maintain the same percentage
        AdjustAttributeForMaxChange(Health, MaxHealth, NewValue, GetHealthAttribute());
    }
}

void UPlatformerHealthAttributeSet::PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data)
{
    Super::PostGameplayEffectExecute(Data);

    // Handle damage application
    if (Data.EvaluatedData.Attribute == GetDamageAttribute())
    {
        // Get the damage value
        const float LocalDamage = GetDamage();
        
        // Clear the damage value
        SetDamage(0.f);

        if (LocalDamage > 0.f)
        {
            // Apply the damage to health
            const float NewHealth = GetHealth() - LocalDamage;
            SetHealth(FMath::Clamp(NewHealth, 0.f, GetMaxHealth()));

            // Check for death
            if (GetHealth() <= 0.f)
            {
                // Handle death (you can add game-specific logic here)
                // For example, add a tag or trigger an event
            }
        }
    }
}

void UPlatformerHealthAttributeSet::OnRep_Health(const FGameplayAttributeData& OldHealth)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UPlatformerHealthAttributeSet, Health, OldHealth);
}

void UPlatformerHealthAttributeSet::OnRep_MaxHealth(const FGameplayAttributeData& OldMaxHealth)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UPlatformerHealthAttributeSet, MaxHealth, OldMaxHealth);
}

void UPlatformerHealthAttributeSet::AdjustAttributeForMaxChange(FGameplayAttributeData& AffectedAttribute, const FGameplayAttributeData& MaxAttribute, float NewMaxValue, const FGameplayAttribute& AffectedAttributeProperty)
{
    UAbilitySystemComponent* AbilityComp = GetOwningAbilitySystemComponent();
    const float CurrentMaxValue = MaxAttribute.GetCurrentValue();
    
    if (!FMath::IsNearlyEqual(CurrentMaxValue, NewMaxValue) && AbilityComp)
    {
        // Maintain the same percentage of max value
        const float CurrentValue = AffectedAttribute.GetCurrentValue();
        const float Percentage = (CurrentMaxValue > 0.f) ? (CurrentValue / CurrentMaxValue) : 0.f;
        const float NewValue = Percentage * NewMaxValue;
        
        AbilityComp->ApplyModToAttributeUnsafe(AffectedAttributeProperty, EGameplayModOp::Override, NewValue);
    }
}
```

### 2. Movement Attribute Set

```cpp
// PlatformerMovementAttributeSet.h
#pragma once

#include "CoreMinimal.h"
#include "AttributeSet.h"
#include "AbilitySystemComponent.h"
#include "PlatformerMovementAttributeSet.generated.h"

// Uses macros from AttributeSet.h
#define ATTRIBUTE_ACCESSORS(ClassName, PropertyName) \
    GAMEPLAYATTRIBUTE_PROPERTY_GETTER(ClassName, PropertyName) \
    GAMEPLAYATTRIBUTE_VALUE_GETTER(PropertyName) \
    GAMEPLAYATTRIBUTE_VALUE_SETTER(PropertyName) \
    GAMEPLAYATTRIBUTE_VALUE_INITTER(PropertyName)

/**
 * Attribute set for movement-related attributes in the platformer game.
 */
UCLASS()
class PLATFORMER_API UPlatformerMovementAttributeSet : public UAttributeSet
{
    GENERATED_BODY()

public:
    UPlatformerMovementAttributeSet();

    // Attribute replication
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
    
    // Attribute changes
    virtual void PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue) override;

    // Movement speed
    UPROPERTY(BlueprintReadOnly, Category = "Movement", ReplicatedUsing = OnRep_MoveSpeed)
    FGameplayAttributeData MoveSpeed;
    ATTRIBUTE_ACCESSORS(UPlatformerMovementAttributeSet, MoveSpeed)

    // Jump height
    UPROPERTY(BlueprintReadOnly, Category = "Movement", ReplicatedUsing = OnRep_JumpHeight)
    FGameplayAttributeData JumpHeight;
    ATTRIBUTE_ACCESSORS(UPlatformerMovementAttributeSet, JumpHeight)

    // Air control
    UPROPERTY(BlueprintReadOnly, Category = "Movement", ReplicatedUsing = OnRep_AirControl)
    FGameplayAttributeData AirControl;
    ATTRIBUTE_ACCESSORS(UPlatformerMovementAttributeSet, AirControl)

    // Dash strength
    UPROPERTY(BlueprintReadOnly, Category = "Movement", ReplicatedUsing = OnRep_DashStrength)
    FGameplayAttributeData DashStrength;
    ATTRIBUTE_ACCESSORS(UPlatformerMovementAttributeSet, DashStrength)

    // Dash cooldown
    UPROPERTY(BlueprintReadOnly, Category = "Movement", ReplicatedUsing = OnRep_DashCooldown)
    FGameplayAttributeData DashCooldown;
    ATTRIBUTE_ACCESSORS(UPlatformerMovementAttributeSet, DashCooldown)

protected:
    // Replication callbacks
    UFUNCTION()
    virtual void OnRep_MoveSpeed(const FGameplayAttributeData& OldMoveSpeed);

    UFUNCTION()
    virtual void OnRep_JumpHeight(const FGameplayAttributeData& OldJumpHeight);

    UFUNCTION()
    virtual void OnRep_AirControl(const FGameplayAttributeData& OldAirControl);

    UFUNCTION()
    virtual void OnRep_DashStrength(const FGameplayAttributeData& OldDashStrength);

    UFUNCTION()
    virtual void OnRep_DashCooldown(const FGameplayAttributeData& OldDashCooldown);
};
```

```cpp
// PlatformerMovementAttributeSet.cpp
#include "PlatformerMovementAttributeSet.h"
#include "Net/UnrealNetwork.h"

UPlatformerMovementAttributeSet::UPlatformerMovementAttributeSet()
{
    // Initialize movement attributes with default values
    InitMoveSpeed(600.0f);        // Base movement speed
    InitJumpHeight(600.0f);       // Base jump height
    InitAirControl(0.2f);         // Base air control (0-1)
    InitDashStrength(1000.0f);    // Base dash strength
    InitDashCooldown(1.0f);       // Base dash cooldown in seconds
}

void UPlatformerMovementAttributeSet::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME_CONDITION_NOTIFY(UPlatformerMovementAttributeSet, MoveSpeed, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UPlatformerMovementAttributeSet, JumpHeight, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UPlatformerMovementAttributeSet, AirControl, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UPlatformerMovementAttributeSet, DashStrength, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UPlatformerMovementAttributeSet, DashCooldown, COND_None, REPNOTIFY_Always);
}

void UPlatformerMovementAttributeSet::PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue)
{
    Super::PreAttributeChange(Attribute, NewValue);

    // Clamp movement values to reasonable ranges
    if (Attribute == GetMoveSpeedAttribute())
    {
        // Prevent negative movement speed
        NewValue = FMath::Max(0.0f, NewValue);
    }
    else if (Attribute == GetJumpHeightAttribute())
    {
        // Prevent negative jump height
        NewValue = FMath::Max(0.0f, NewValue);
    }
    else if (Attribute == GetAirControlAttribute())
    {
        // Clamp air control between 0 and 1
        NewValue = FMath::Clamp(NewValue, 0.0f, 1.0f);
    }
    else if (Attribute == GetDashCooldownAttribute())
    {
        // Prevent negative cooldowns
        NewValue = FMath::Max(0.0f, NewValue);
    }
}

void UPlatformerMovementAttributeSet::OnRep_MoveSpeed(const FGameplayAttributeData& OldMoveSpeed)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UPlatformerMovementAttributeSet, MoveSpeed, OldMoveSpeed);
}

void UPlatformerMovementAttributeSet::OnRep_JumpHeight(const FGameplayAttributeData& OldJumpHeight)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UPlatformerMovementAttributeSet, JumpHeight, OldJumpHeight);
}

void UPlatformerMovementAttributeSet::OnRep_AirControl(const FGameplayAttributeData& OldAirControl)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UPlatformerMovementAttributeSet, AirControl, OldAirControl);
}

void UPlatformerMovementAttributeSet::OnRep_DashStrength(const FGameplayAttributeData& OldDashStrength)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UPlatformerMovementAttributeSet, DashStrength, OldDashStrength);
}

void UPlatformerMovementAttributeSet::OnRep_DashCooldown(const FGameplayAttributeData& OldDashCooldown)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UPlatformerMovementAttributeSet, DashCooldown, OldDashCooldown);
}
```

## Important Attribute Set Callbacks

Attribute Sets provide several callback functions to handle attribute changes:

### 1. PreAttributeChange

Called before any attribute changes. Use this to clamp values or make adjustments:

```cpp
void PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue);
```

### 2. PostGameplayEffectExecute

Called after a Gameplay Effect has executed and modified an attribute:

```cpp
void PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data);
```

### 3. OnRep_* Callbacks

Called when an attribute is replicated to a client:

```cpp
void OnRep_Health(const FGameplayAttributeData& OldHealth);
```

## Initializing Attributes

There are two primary ways to initialize attributes:

### 1. Using Gameplay Effects (Recommended)

Create a Gameplay Effect that sets initial values:

```cpp
// PlatformerCharacterInitEffect.h
#pragma once

#include "CoreMinimal.h"
#include "GameplayEffect.h"
#include "PlatformerCharacterInitEffect.generated.h"

/**
 * Gameplay Effect for initializing character attributes.
 */
UCLASS()
class PLATFORMER_API UPlatformerCharacterInitEffect : public UGameplayEffect
{
    GENERATED_BODY()

public:
    UPlatformerCharacterInitEffect();
};
```

```cpp
// PlatformerCharacterInitEffect.cpp
#include "PlatformerCharacterInitEffect.h"
#include "PlatformerHealthAttributeSet.h"
#include "PlatformerMovementAttributeSet.h"

UPlatformerCharacterInitEffect::UPlatformerCharacterInitEffect()
{
    DurationPolicy = EGameplayEffectDurationType::Instant;

    // Health attribute modifiers
    {
        FGameplayModifierInfo ModifierInfo;
        ModifierInfo.Attribute = UPlatformerHealthAttributeSet::GetHealthAttribute();
        ModifierInfo.ModifierOp = EGameplayModOp::Override;
        ModifierInfo.ModifierMagnitude = FScalableFloat(100.0f);
        Modifiers.Add(ModifierInfo);
    }

    {
        FGameplayModifierInfo ModifierInfo;
        ModifierInfo.Attribute = UPlatformerHealthAttributeSet::GetMaxHealthAttribute();
        ModifierInfo.ModifierOp = EGameplayModOp::Override;
        ModifierInfo.ModifierMagnitude = FScalableFloat(100.0f);
        Modifiers.Add(ModifierInfo);
    }
    
    // Movement attribute modifiers
    {
        FGameplayModifierInfo ModifierInfo;
        ModifierInfo.Attribute = UPlatformerMovementAttributeSet::GetMoveSpeedAttribute();
        ModifierInfo.ModifierOp = EGameplayModOp::Override;
        ModifierInfo.ModifierMagnitude = FScalableFloat(600.0f);
        Modifiers.Add(ModifierInfo);
    }

    {
        FGameplayModifierInfo ModifierInfo;
        ModifierInfo.Attribute = UPlatformerMovementAttributeSet::GetJumpHeightAttribute();
        ModifierInfo.ModifierOp = EGameplayModOp::Override;
        ModifierInfo.ModifierMagnitude = FScalableFloat(600.0f);
        Modifiers.Add(ModifierInfo);
    }
    
    // Add other attribute initializations as needed
}
```

Apply the effect in your character or player state's initialization:

```cpp
// In your character or player state class
void APlatformerCharacter::InitializeAttributes()
{
    if (AbilitySystemComponent && DefaultAttributeEffect)
    {
        FGameplayEffectContextHandle EffectContext = AbilitySystemComponent->MakeEffectContext();
        EffectContext.AddSourceObject(this);

        FGameplayEffectSpecHandle SpecHandle = AbilitySystemComponent->MakeOutgoingSpec(
            DefaultAttributeEffect, 1, EffectContext);

        if (SpecHandle.IsValid())
        {
            AbilitySystemComponent->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get());
        }
    }
}
```

### 2. Using DataTables (Alternative)

You can also use DataTables for more designer-friendly configuration:

1. Create a DataTable with the `FAttributeMetaData` row structure
2. Initialize attributes with the DataTable:

```cpp
// In your character or player state initialization
if (AbilitySystemComponent && AttributeDataTable)
{
    const UAttributeSet* AttributeSet = AbilitySystemComponent->InitStats(
        UPlatformerHealthAttributeSet::StaticClass(), AttributeDataTable);
}
```

## Adding Attributes to Characters

To add attribute sets to your character, follow these steps:

1. Create the attribute set instance
2. Add it to the AbilitySystemComponent

```cpp
// In your character class constructor
APlatformerCharacter::APlatformerCharacter()
{
    // Create ability system component
    AbilitySystemComponent = CreateDefaultSubobject<UPlatformerAbilitySystemComponent>("AbilitySystemComponent");
    AbilitySystemComponent->SetIsReplicated(true);
    
    // Create attribute sets
    HealthAttributeSet = CreateDefaultSubobject<UPlatformerHealthAttributeSet>("HealthAttributeSet");
    MovementAttributeSet = CreateDefaultSubobject<UPlatformerMovementAttributeSet>("MovementAttributeSet");
}

// For player-controlled characters (if using a PlayerState to hold the ASC)
void APlatformerPlayerState::BeginPlay()
{
    Super::BeginPlay();
    
    // Initialize ability system
    if (AbilitySystemComponent)
    {
        // Create attribute sets
        HealthAttributeSet = NewObject<UPlatformerHealthAttributeSet>(this);
        MovementAttributeSet = NewObject<UPlatformerMovementAttributeSet>(this);

        // Add attribute sets to ASC
        AbilitySystemComponent->AddAttributeSetSubobject(HealthAttributeSet);
        AbilitySystemComponent->AddAttributeSetSubobject(MovementAttributeSet);
        
        // Initialize with default values
        InitializeAttributes();
    }
}
```

## Using Attributes in Gameplay Code

Access attribute values in your gameplay code:

```cpp
// Getting attribute values
float APlatformerCharacter::GetHealth() const
{
    if (HealthAttributeSet)
    {
        return HealthAttributeSet->GetHealth();
    }
    return 0.0f;
}

float APlatformerCharacter::GetMaxHealth() const
{
    if (HealthAttributeSet)
    {
        return HealthAttributeSet->GetMaxHealth();
    }
    return 0.0f;
}

float APlatformerCharacter::GetMoveSpeed() const
{
    if (MovementAttributeSet)
    {
        return MovementAttributeSet->GetMoveSpeed();
    }
    return 0.0f;
}

// Applying the move speed to character movement
void APlatformerCharacter::UpdateMovementProperties()
{
    if (GetCharacterMovement() && MovementAttributeSet)
    {
        GetCharacterMovement()->MaxWalkSpeed = MovementAttributeSet->GetMoveSpeed();
        GetCharacterMovement()->JumpZVelocity = MovementAttributeSet->GetJumpHeight();
        GetCharacterMovement()->AirControl = MovementAttributeSet->GetAirControl();
    }
}
```

## Listening for Attribute Changes

To respond to attribute changes:

```cpp
// In your character class header
FDelegateHandle HealthChangedDelegateHandle;
FDelegateHandle MaxHealthChangedDelegateHandle;

// In your character class implementation
void APlatformerCharacter::SetupAttributeChangeCallbacks()
{
    if (AbilitySystemComponent)
    {
        // Health change callback
        HealthChangedDelegateHandle = AbilitySystemComponent->GetGameplayAttributeValueChangeDelegate(
            UPlatformerHealthAttributeSet::GetHealthAttribute()).AddUObject(
                this, &APlatformerCharacter::OnHealthChanged);
        
        // Max health change callback
        MaxHealthChangedDelegateHandle = AbilitySystemComponent->GetGameplayAttributeValueChangeDelegate(
            UPlatformerHealthAttributeSet::GetMaxHealthAttribute()).AddUObject(
                this, &APlatformerCharacter::OnMaxHealthChanged);
    }
}

void APlatformerCharacter::OnHealthChanged(const FOnAttributeChangeData& Data)
{
    float OldValue = Data.OldValue;
    float NewValue = Data.NewValue;
    
    // Update UI or trigger gameplay events
    UpdateHealthUI();
    
    // Check for damage or healing
    if (NewValue < OldValue)
    {
        PlayDamageEffects();
    }
    else if (NewValue > OldValue)
    {
        PlayHealingEffects();
    }
    
    // Check for death
    if (NewValue <= 0.0f && OldValue > 0.0f)
    {
        HandleDeath();
    }
}
```

## Next Steps

With attribute sets in place, you can proceed to implementing:

1. [Gameplay Abilities](./6_3_gameplay_abilities.md) - Define specific abilities for your character
2. [Gameplay Effects](./6_4_gameplay_effects.md) - Create effects that modify attributes

## Common Issues and Solutions

- **Attributes Not Replicating**: Ensure you've added `DOREPLIFETIME_CONDITION_NOTIFY` for each attribute
- **Attribute Values Reset**: Make sure initialization is happening in the correct order
- **Clamp Values Not Working**: Verify that `PreAttributeChange` is implemented correctly
- **Changes Not Triggering Events**: Check delegate binding and ensure you're using the correct attribute reference

## Best Practices

- Group related attributes into dedicated attribute sets
- Use meta attributes (like Damage) for temporary calculations
- Keep attribute changes encapsulated within the attribute set classes
- Implement clean getter functions in your character class
- Use PreAttributeChange for validation, PostGameplayEffectExecute for game logic
- Initialize attributes using gameplay effects rather than direct assignment 