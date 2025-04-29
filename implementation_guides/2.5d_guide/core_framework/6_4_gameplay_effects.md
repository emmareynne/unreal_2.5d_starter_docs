# Gameplay Ability System: Gameplay Effects

This guide covers implementing Gameplay Effects for your 2.5D platformer using Unreal Engine 5.5's Gameplay Ability System.

## What Are Gameplay Effects?

Gameplay Effects (GE) are modifiers that change attributes or apply status effects to characters. They are the primary mechanism for implementing:

- Damage and healing
- Buffs and debuffs
- Cooldowns and costs
- Temporary and permanent stat changes
- Status effects (stun, poison, etc.)

## Core Components of Gameplay Effects

### 1. Effect Structure and Duration

Gameplay Effects have three duration types:

- **Instant**: Applied once and immediately ends (damage, healing)
- **Duration**: Applied for a set time, then expires (temporary buffs)
- **Infinite**: Applied until explicitly removed (permanent upgrades)

### 2. Modifiers

Modifiers define how effects change attributes using these operations:

- **Add**: Directly adds a value to the attribute
- **Multiply**: Multiplies the attribute by a value
- **Divide**: Divides the attribute by a value
- **Override**: Sets the attribute to a specific value

### 3. Stacking

Effects can stack in various ways:

- **No stacking**: New applications replace the existing effect
- **Duration stacking**: Extends the duration when reapplied
- **Intensity stacking**: Increases the magnitude when reapplied

## Implementing Basic Gameplay Effects

### Creating a Damage Effect

```cpp
// PlatformerDamageEffect.h
#pragma once

#include "CoreMinimal.h"
#include "GameplayEffect.h"
#include "PlatformerDamageEffect.generated.h"

/**
 * Base Gameplay Effect for damage in the platformer.
 * Configurable damage amount that applies instantly.
 */
UCLASS()
class PLATFORMER_API UPlatformerDamageEffect : public UGameplayEffect
{
    GENERATED_BODY()

public:
    UPlatformerDamageEffect();
};
```

```cpp
// PlatformerDamageEffect.cpp
#include "PlatformerDamageEffect.h"
#include "PlatformerHealthAttributeSet.h"

UPlatformerDamageEffect::UPlatformerDamageEffect()
{
    // Set as an instant effect
    DurationPolicy = EGameplayEffectDurationType::Instant;
    
    // Set up damage modifier
    FGameplayModifierInfo DamageModifier;
    DamageModifier.Attribute = UPlatformerHealthAttributeSet::GetDamageAttribute();
    DamageModifier.ModifierOp = EGameplayModOp::Additive;
    DamageModifier.ModifierMagnitude = FScalableFloat(10.0f); // Default damage
    Modifiers.Add(DamageModifier);
}
```

### Creating a Speed Boost Effect

```cpp
// PlatformerSpeedBoostEffect.h
#pragma once

#include "CoreMinimal.h"
#include "GameplayEffect.h"
#include "PlatformerSpeedBoostEffect.generated.h"

/**
 * Gameplay Effect that temporarily increases movement speed.
 */
UCLASS()
class PLATFORMER_API UPlatformerSpeedBoostEffect : public UGameplayEffect
{
    GENERATED_BODY()

public:
    UPlatformerSpeedBoostEffect();
};
```

```cpp
// PlatformerSpeedBoostEffect.cpp
#include "PlatformerSpeedBoostEffect.h"
#include "PlatformerMovementAttributeSet.h"

UPlatformerSpeedBoostEffect::UPlatformerSpeedBoostEffect()
{
    // Set as duration-based effect
    DurationPolicy = EGameplayEffectDurationType::HasDuration;
    DurationMagnitude = FScalableFloat(5.0f); // 5 second duration
    
    // Set up speed boost modifier (50% increase)
    FGameplayModifierInfo SpeedModifier;
    SpeedModifier.Attribute = UPlatformerMovementAttributeSet::GetMoveSpeedAttribute();
    SpeedModifier.ModifierOp = EGameplayModOp::Multiplicative;
    SpeedModifier.ModifierMagnitude = FScalableFloat(1.5f); // 50% increase
    Modifiers.Add(SpeedModifier);
    
    // Add the "Boosted" tag while this effect is active
    InheritableGameplayEffectTags.AddTag(FGameplayTag::RequestGameplayTag("State.Boosted"));
}
```

### Creating a Cooldown Effect

```cpp
// PlatformerCooldownEffect.h
#pragma once

#include "CoreMinimal.h"
#include "GameplayEffect.h"
#include "PlatformerCooldownEffect.generated.h"

/**
 * Gameplay Effect for managing ability cooldowns.
 */
UCLASS()
class PLATFORMER_API UPlatformerCooldownEffect : public UGameplayEffect
{
    GENERATED_BODY()

public:
    UPlatformerCooldownEffect();
};
```

```cpp
// PlatformerCooldownEffect.cpp
#include "PlatformerCooldownEffect.h"

UPlatformerCooldownEffect::UPlatformerCooldownEffect()
{
    // Set as duration-based effect
    DurationPolicy = EGameplayEffectDurationType::HasDuration;
    DurationMagnitude = FScalableFloat(3.0f); // 3 second cooldown by default
    
    // Add cooldown tags
    InheritableGameplayEffectTags.AddTag(FGameplayTag::RequestGameplayTag("Cooldown"));
    // This allows us to make specific cooldown tags for different abilities
    // e.g., Cooldown.Ability.Jump, Cooldown.Ability.Dash
}
```

## Using Gameplay Effects in Abilities

### Applying a Damage Effect

```cpp
// Inside an attack ability
void UPlatformerAttackAbility::ActivateAbility(...)
{
    // Existing code...
    
    // Create damage effect spec
    UGameplayEffect* DamageEffectClass = UPlatformerDamageEffect::StaticClass()->GetDefaultObject<UGameplayEffect>();
    FGameplayEffectSpecHandle DamageEffectSpecHandle = MakeOutgoingGameplayEffectSpec(DamageEffectClass, GetAbilityLevel());
    
    // Set the damage value (can be based on character stats, ability level, etc.)
    float DamageAmount = 25.0f;
    DamageEffectSpecHandle.Data->SetSetByCallerMagnitude(FGameplayTag::RequestGameplayTag("Data.Damage"), DamageAmount);
    
    // Apply to the target
    if (TargetActor)
    {
        UAbilitySystemComponent* TargetASC = UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent(TargetActor);
        if (TargetASC)
        {
            ApplyGameplayEffectSpecToTarget(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, DamageEffectSpecHandle, TargetASC);
        }
    }
}
```

### Applying a Buff to Self

```cpp
// Inside a buff ability
void UPlatformerBuffAbility::ActivateAbility(...)
{
    // Existing code...
    
    // Create speed boost effect spec
    UGameplayEffect* SpeedBoostEffectClass = UPlatformerSpeedBoostEffect::StaticClass()->GetDefaultObject<UGameplayEffect>();
    FGameplayEffectSpecHandle BoostEffectSpecHandle = MakeOutgoingGameplayEffectSpec(SpeedBoostEffectClass, GetAbilityLevel());
    
    // Apply to self
    UAbilitySystemComponent* SourceASC = GetAbilitySystemComponentFromActorInfo();
    if (SourceASC)
    {
        ActiveEffectHandle = ApplyGameplayEffectSpecToOwner(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, BoostEffectSpecHandle);
    }
}
```

### Applying a Cooldown

```cpp
// Inside any ability that needs a cooldown
void UPlatformerDashAbility::ActivateAbility(...)
{
    // Apply the ability's effects
    // ...
    
    // Apply cooldown
    if (CooldownGameplayEffectClass)
    {
        FGameplayEffectSpecHandle CooldownSpecHandle = MakeOutgoingGameplayEffectSpec(CooldownGameplayEffectClass, GetAbilityLevel());
        
        // You can customize the cooldown duration here
        // CooldownSpecHandle.Data->SetDuration(CustomDuration, true);
        
        // Add gameplay tag for this specific ability's cooldown
        FGameplayTagContainer CooldownTags;
        CooldownTags.AddTag(FGameplayTag::RequestGameplayTag("Cooldown.Ability.Dash"));
        
        // Apply the cooldown tags
        for (const FGameplayTag& CooldownTag : CooldownTags)
        {
            CooldownSpecHandle.Data->DynamicGrantedTags.AddTag(CooldownTag);
        }
        
        // Apply the cooldown to self
        ApplyGameplayEffectSpecToOwner(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, CooldownSpecHandle);
    }
}
```

## Advanced Gameplay Effect Techniques

### 1. Using SetByCaller for Dynamic Values

SetByCaller allows you to determine effect magnitudes at runtime:

```cpp
// When creating the effect
FGameplayEffectSpecHandle EffectSpecHandle = MakeOutgoingGameplayEffectSpec(EffectClass, Level);
EffectSpecHandle.Data->SetSetByCallerMagnitude(FGameplayTag::RequestGameplayTag("Data.Damage"), 50.0f);

// Inside the GameplayEffect constructor
FGameplayModifierInfo ModifierInfo;
ModifierInfo.Attribute = UPlatformerHealthAttributeSet::GetDamageAttribute();
ModifierInfo.ModifierOp = EGameplayModOp::Additive;
ModifierInfo.ModifierMagnitude = FScalableFloat(10.0f);
// Use SetByCaller if available
ModifierInfo.ModifierMagnitude.SetValue(EGameplayModifierEvaluationType::SetByCaller);
ModifierInfo.ModifierMagnitude.SetSetByCallerTag(FGameplayTag::RequestGameplayTag("Data.Damage"));
Modifiers.Add(ModifierInfo);
```

### 2. Creating Stacking Effects

For effects that stack in intensity or duration:

```cpp
UPlatformerStackingDamageOverTimeEffect::UPlatformerStackingDamageOverTimeEffect()
{
    DurationPolicy = EGameplayEffectDurationType::HasDuration;
    DurationMagnitude = FScalableFloat(5.0f);
    
    // Allow this effect to stack
    StackingType = EGameplayEffectStackingType::AggregateBySource;
    StackLimitCount = 5; // Max 5 stacks
    
    // For stacking duration (instead of intensity)
    // StackDurationRefreshPolicy = EGameplayEffectStackingDurationPolicy::RefreshOnSuccessfulApplication;
    
    // For stacking intensity
    StackExpirationPolicy = EGameplayEffectStackingExpirationPolicy::ClearEntireStack;
    
    // Damage modifier (increases with each stack)
    FGameplayModifierInfo DamageModifier;
    DamageModifier.Attribute = UPlatformerHealthAttributeSet::GetDamageAttribute();
    DamageModifier.ModifierOp = EGameplayModOp::Additive;
    DamageModifier.ModifierMagnitude = FScalableFloat(5.0f); // Per stack
    Modifiers.Add(DamageModifier);
}
```

### 3. Conditional Gameplay Effects

Effects that only apply under certain conditions:

```cpp
UPlatformerConditionalDamageEffect::UPlatformerConditionalDamageEffect()
{
    DurationPolicy = EGameplayEffectDurationType::Instant;
    
    // Set up conditional damage
    FGameplayModifierInfo DamageModifier;
    DamageModifier.Attribute = UPlatformerHealthAttributeSet::GetDamageAttribute();
    DamageModifier.ModifierOp = EGameplayModOp::Additive;
    DamageModifier.ModifierMagnitude = FScalableFloat(20.0f);
    
    // Apply 50% more damage if target is in air
    FGameplayEffectExecutionDefinition ExecutionDefinition;
    // Add custom execution calculation class here
    
    // Or use application requirements
    ApplicationRequirements.Add(FGameplayTagRequirement(
        FGameplayTag::RequestGameplayTag("State.InAir"),
        EGameplayTagRequirementType::RequirementExists, 
        true
    ));
    
    Modifiers.Add(DamageModifier);
}
```

## Gameplay Effect Data Assets

For designer-friendly configuration, create Blueprint-based Gameplay Effect data assets:

1. Create a Blueprint derived from your Gameplay Effect class
2. Configure the effect's properties in the Blueprint editor
3. Use the Blueprint asset in your code

```cpp
// In your character or ability class header
UPROPERTY(EditDefaultsOnly, Category = "Effects")
TSubclassOf<UGameplayEffect> DamageEffectClass;

UPROPERTY(EditDefaultsOnly, Category = "Effects")
TSubclassOf<UGameplayEffect> SpeedBoostEffectClass;

UPROPERTY(EditDefaultsOnly, Category = "Effects")
TSubclassOf<UGameplayEffect> CooldownEffectClass;
```

## Testing Gameplay Effects

To test your Gameplay Effects:

1. Create a simple test map with targets
2. Add debug visualization for active effects
3. Add UI to display current attribute values
4. Add commands to apply effects manually

```cpp
// Debug function to apply an effect
void APlatformerCharacter::DebugApplyEffect(TSubclassOf<UGameplayEffect> EffectClass, float Level)
{
    if (EffectClass && AbilitySystemComponent)
    {
        FGameplayEffectContextHandle EffectContext = AbilitySystemComponent->MakeEffectContext();
        EffectContext.AddSourceObject(this);
        
        FGameplayEffectSpecHandle SpecHandle = AbilitySystemComponent->MakeOutgoingSpec(
            EffectClass, Level, EffectContext);
        
        if (SpecHandle.IsValid())
        {
            FActiveGameplayEffectHandle ActiveGEHandle = 
                AbilitySystemComponent->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get());
                
            UE_LOG(LogTemp, Log, TEXT("Applied effect %s, handle: %s"), 
                *GetNameSafe(EffectClass), *ActiveGEHandle.ToString());
        }
    }
}
```

## Common Issues and Solutions

- **Effect Not Applied**: Check that the Gameplay Effect's tags don't conflict with immunity tags on the target
- **Incorrect Magnitude**: Verify that the attribute and operation type match what you're trying to modify
- **Stacking Not Working**: Ensure stacking rules are properly configured
- **Conditional Effect Not Triggering**: Check that the target has the required tags or meets the conditions

## Best Practices

- **Create Base Effect Classes**: Make parent classes for common effect types
- **Use SetByCaller**: For values determined at runtime
- **Tag Everything**: Use a consistent tag system for effects and conditions
- **Blueprint Exposure**: Expose configuration options to designers
- **Handle Edge Cases**: Consider what happens when effects stack, expire, or are removed prematurely

## Next Steps

With Gameplay Effects configured, you can now move on to:

1. [Gameplay Cues](./6_5_gameplay_cues.md) - Add visual and audio feedback for your effects
2. [Integrating with Other Systems](./6_6_gas_integration.md) - Connect GAS with UI, animation, etc.

## Resources

- [Unreal Engine GAS Documentation](https://docs.unrealengine.com/5.5/en-US/gameplay-ability-system-for-unreal-engine/)
- [Community GAS Documentation](https://github.com/tranek/GASDocumentation) 