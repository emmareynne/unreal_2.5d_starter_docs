# Character Progression Systems with GAS (Part 1)

This guide covers implementing character progression systems using the Gameplay Ability System (GAS) for your 2.5D platformer in Unreal Engine 5.5.

## Overview

Character progression is a fundamental aspect of many games, allowing players to:
- Gain experience and level up
- Unlock and upgrade abilities
- Improve attributes
- Customize character builds

GAS provides a flexible foundation for implementing these systems through its attribute, effect, and ability frameworks.

## Experience and Leveling System

### 1. Experience Attribute Set

First, create an attribute set to manage experience and level:

```cpp
// PlatformerProgressionAttributeSet.h
#pragma once

#include "CoreMinimal.h"
#include "AttributeSet.h"
#include "AbilitySystemComponent.h"
#include "PlatformerProgressionAttributeSet.generated.h"

// Uses macros from AttributeSet.h
#define ATTRIBUTE_ACCESSORS(ClassName, PropertyName) \
    GAMEPLAYATTRIBUTE_PROPERTY_GETTER(ClassName, PropertyName) \
    GAMEPLAYATTRIBUTE_VALUE_GETTER(PropertyName) \
    GAMEPLAYATTRIBUTE_VALUE_SETTER(PropertyName) \
    GAMEPLAYATTRIBUTE_VALUE_INITTER(PropertyName)

/**
 * Attribute set for character progression.
 */
UCLASS()
class PLATFORMER_API UPlatformerProgressionAttributeSet : public UAttributeSet
{
    GENERATED_BODY()

public:
    UPlatformerProgressionAttributeSet();

    // Attribute replication
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    // Called after gameplay effect executes
    virtual void PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data) override;

    // Experience points
    UPROPERTY(BlueprintReadOnly, Category = "Progression", ReplicatedUsing = OnRep_Experience)
    FGameplayAttributeData Experience;
    ATTRIBUTE_ACCESSORS(UPlatformerProgressionAttributeSet, Experience)

    // Current level
    UPROPERTY(BlueprintReadOnly, Category = "Progression", ReplicatedUsing = OnRep_Level)
    FGameplayAttributeData Level;
    ATTRIBUTE_ACCESSORS(UPlatformerProgressionAttributeSet, Level)

    // Experience required for next level
    UPROPERTY(BlueprintReadOnly, Category = "Progression", ReplicatedUsing = OnRep_ExperienceToNextLevel)
    FGameplayAttributeData ExperienceToNextLevel;
    ATTRIBUTE_ACCESSORS(UPlatformerProgressionAttributeSet, ExperienceToNextLevel)

    // Skill points available to spend
    UPROPERTY(BlueprintReadOnly, Category = "Progression", ReplicatedUsing = OnRep_SkillPoints)
    FGameplayAttributeData SkillPoints;
    ATTRIBUTE_ACCESSORS(UPlatformerProgressionAttributeSet, SkillPoints)

protected:
    // Replication callbacks
    UFUNCTION()
    virtual void OnRep_Experience(const FGameplayAttributeData& OldExperience);

    UFUNCTION()
    virtual void OnRep_Level(const FGameplayAttributeData& OldLevel);

    UFUNCTION()
    virtual void OnRep_ExperienceToNextLevel(const FGameplayAttributeData& OldExperienceToNextLevel);

    UFUNCTION()
    virtual void OnRep_SkillPoints(const FGameplayAttributeData& OldSkillPoints);

    // Check if the character leveled up
    void CheckForLevelUp(const FGameplayEffectModCallbackData& Data);

    // Calculate experience required for a specific level
    float CalculateExperienceForLevel(int32 InLevel) const;
};
```

```cpp
// PlatformerProgressionAttributeSet.cpp
#include "PlatformerProgressionAttributeSet.h"
#include "Net/UnrealNetwork.h"
#include "GameplayEffectExtension.h"
#include "GameplayEffect.h"
#include "GameplayEffectTypes.h"

UPlatformerProgressionAttributeSet::UPlatformerProgressionAttributeSet()
{
    // Set initial values
    ExperienceToNextLevel.SetBaseValue(100.0f);
    ExperienceToNextLevel.SetCurrentValue(100.0f);
    Experience.SetBaseValue(0.0f);
    Experience.SetCurrentValue(0.0f);
    Level.SetBaseValue(1.0f);
    Level.SetCurrentValue(1.0f);
    SkillPoints.SetBaseValue(0.0f);
    SkillPoints.SetCurrentValue(0.0f);
}

void UPlatformerProgressionAttributeSet::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME_CONDITION_NOTIFY(UPlatformerProgressionAttributeSet, Experience, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UPlatformerProgressionAttributeSet, Level, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UPlatformerProgressionAttributeSet, ExperienceToNextLevel, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UPlatformerProgressionAttributeSet, SkillPoints, COND_None, REPNOTIFY_Always);
}

void UPlatformerProgressionAttributeSet::PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data)
{
    Super::PostGameplayEffectExecute(Data);

    // Handle experience changes
    if (Data.EvaluatedData.Attribute == GetExperienceAttribute())
    {
        // Clamp experience to be >= 0
        SetExperience(FMath::Max(GetExperience(), 0.0f));
        
        // Check for level up after gaining experience
        CheckForLevelUp(Data);
    }
    // Handle level changes directly (though this is less common)
    else if (Data.EvaluatedData.Attribute == GetLevelAttribute())
    {
        // Ensure level is at least 1
        SetLevel(FMath::Max(GetLevel(), 1.0f));
        
        // Update experience required for next level
        SetExperienceToNextLevel(CalculateExperienceForLevel(FMath::RoundToInt(GetLevel() + 1)));
    }
    // Handle skill points changes
    else if (Data.EvaluatedData.Attribute == GetSkillPointsAttribute())
    {
        // Ensure skill points don't go negative
        SetSkillPoints(FMath::Max(GetSkillPoints(), 0.0f));
    }
}

void UPlatformerProgressionAttributeSet::OnRep_Experience(const FGameplayAttributeData& OldExperience)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UPlatformerProgressionAttributeSet, Experience, OldExperience);
}

void UPlatformerProgressionAttributeSet::OnRep_Level(const FGameplayAttributeData& OldLevel)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UPlatformerProgressionAttributeSet, Level, OldLevel);
}

void UPlatformerProgressionAttributeSet::OnRep_ExperienceToNextLevel(const FGameplayAttributeData& OldExperienceToNextLevel)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UPlatformerProgressionAttributeSet, ExperienceToNextLevel, OldExperienceToNextLevel);
}

void UPlatformerProgressionAttributeSet::OnRep_SkillPoints(const FGameplayAttributeData& OldSkillPoints)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UPlatformerProgressionAttributeSet, SkillPoints, OldSkillPoints);
}

void UPlatformerProgressionAttributeSet::CheckForLevelUp(const FGameplayEffectModCallbackData& Data)
{
    float CurrentXP = GetExperience();
    float XPToNextLevel = GetExperienceToNextLevel();
    int32 CurrentLevel = FMath::RoundToInt(GetLevel());
    
    // Check if we have enough XP to level up
    if (CurrentXP >= XPToNextLevel)
    {
        UAbilitySystemComponent* ASC = Data.Target.AbilityActorInfo->AbilitySystemComponent.Get();
        if (!ASC)
        {
            return;
        }
        
        // Increase level
        SetLevel(CurrentLevel + 1);
        
        // Award skill points (1 per level up)
        SetSkillPoints(GetSkillPoints() + 1.0f);
        
        // Calculate excess XP
        float ExcessXP = CurrentXP - XPToNextLevel;
        
        // Calculate XP for next level
        float NewXPToNextLevel = CalculateExperienceForLevel(CurrentLevel + 2);
        SetExperienceToNextLevel(NewXPToNextLevel);
        
        // Set current XP to excess XP
        SetExperience(ExcessXP);
        
        // Trigger level up gameplay cue
        FGameplayTag LevelUpTag = FGameplayTag::RequestGameplayTag(FName("GameplayCue.Character.LevelUp"));
        FGameplayCueParameters CueParams;
        CueParams.RawMagnitude = CurrentLevel + 1;
        ASC->ExecuteGameplayCue(LevelUpTag, CueParams);
        
        // Check for multiple level-ups in one go
        if (ExcessXP >= NewXPToNextLevel)
        {
            CheckForLevelUp(Data);
        }
    }
}

float UPlatformerProgressionAttributeSet::CalculateExperienceForLevel(int32 InLevel) const
{
    // Simple level scaling formula: 100 * Level^1.5
    return 100.0f * FMath::Pow(static_cast<float>(InLevel), 1.5f);
}
```

### 2. Experience Granting Effects

Create gameplay effects to award experience:

```cpp
// GE_AwardExperience.h
#pragma once

#include "CoreMinimal.h"
#include "GameplayEffect.h"
#include "GE_AwardExperience.generated.h"

/**
 * Gameplay effect for awarding experience points.
 */
UCLASS()
class PLATFORMER_API UGE_AwardExperience : public UGameplayEffect
{
    GENERATED_BODY()

public:
    UGE_AwardExperience();
};
```

```cpp
// GE_AwardExperience.cpp
#include "GE_AwardExperience.h"
#include "PlatformerProgressionAttributeSet.h"

UGE_AwardExperience::UGE_AwardExperience()
{
    DurationPolicy = EGameplayEffectDurationType::Instant;
    
    // Add modifier to increase experience
    FGameplayModifierInfo ModifierInfo;
    ModifierInfo.Attribute = UPlatformerProgressionAttributeSet::GetExperienceAttribute();
    ModifierInfo.ModifierOp = EGameplayModOp::Additive;
    ModifierInfo.ModifierMagnitude = FScalableFloat(10.0f); // Base XP reward
    Modifiers.Add(ModifierInfo);
}
```

### 3. Experience Source Component

Create a component to manage experience sources:

```cpp
// PlatformerExperienceComponent.h
#pragma once

#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "GameplayTagContainer.h"
#include "PlatformerExperienceComponent.generated.h"

class UGameplayEffect;
class UAbilitySystemComponent;

/**
 * Component for managing experience rewards and sources.
 */
UCLASS(ClassGroup=(Platformer), meta=(BlueprintSpawnableComponent))
class PLATFORMER_API UPlatformerExperienceComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    UPlatformerExperienceComponent();
    
    // Award experience to a target
    UFUNCTION(BlueprintCallable, Category = "Experience")
    void AwardExperienceToTarget(AActor* Target, float BaseXP, AActor* ExperienceSource = nullptr);
    
protected:
    // Experience award effect class
    UPROPERTY(EditDefaultsOnly, Category = "Experience")
    TSubclassOf<UGameplayEffect> ExperienceEffectClass;
    
    // Map of enemy types to experience multipliers
    UPROPERTY(EditDefaultsOnly, Category = "Experience")
    TMap<FGameplayTag, float> EnemyTypeXPMultipliers;
    
    // Get XP multiplier for an enemy type
    float GetXPMultiplierForEnemy(AActor* Enemy) const;
};
```

```cpp
// PlatformerExperienceComponent.cpp
#include "PlatformerExperienceComponent.h"
#include "AbilitySystemComponent.h"
#include "AbilitySystemBlueprintLibrary.h"
#include "GameplayEffect.h"

UPlatformerExperienceComponent::UPlatformerExperienceComponent()
{
    PrimaryComponentTick.bCanEverTick = false;
    
    // Default multipliers for different enemy types
    EnemyTypeXPMultipliers.Add(FGameplayTag::RequestGameplayTag("Enemy.Minion"), 1.0f);
    EnemyTypeXPMultipliers.Add(FGameplayTag::RequestGameplayTag("Enemy.Elite"), 2.0f);
    EnemyTypeXPMultipliers.Add(FGameplayTag::RequestGameplayTag("Enemy.Boss"), 5.0f);
}

void UPlatformerExperienceComponent::AwardExperienceToTarget(AActor* Target, float BaseXP, AActor* ExperienceSource)
{
    if (!Target || !ExperienceEffectClass)
    {
        return;
    }
    
    // Get ability system component from target
    UAbilitySystemComponent* TargetASC = UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent(Target);
    if (!TargetASC)
    {
        return;
    }
    
    // Create effect context
    FGameplayEffectContextHandle EffectContext = TargetASC->MakeEffectContext();
    EffectContext.AddSourceObject(ExperienceSource ? ExperienceSource : GetOwner());
    
    // Create effect spec
    FGameplayEffectSpecHandle SpecHandle = TargetASC->MakeOutgoingSpec(
        ExperienceEffectClass, 1, EffectContext);
    
    if (SpecHandle.IsValid())
    {
        // Calculate final XP based on source
        float FinalXP = BaseXP;
        if (ExperienceSource)
        {
            FinalXP *= GetXPMultiplierForEnemy(ExperienceSource);
        }
        
        // Set the XP amount 
        SpecHandle.Data.Get()->SetSetByCallerMagnitude(FGameplayTag::RequestGameplayTag("Data.XPAmount"), FinalXP);
        
        // Apply effect to target
        TargetASC->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get());
    }
}

float UPlatformerExperienceComponent::GetXPMultiplierForEnemy(AActor* Enemy) const
{
    if (!Enemy)
    {
        return 1.0f;
    }
    
    // Get the enemy's ability system
    UAbilitySystemComponent* EnemyASC = UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent(Enemy);
    if (!EnemyASC)
    {
        return 1.0f;
    }
    
    // Check for enemy type tags and return appropriate multiplier
    for (const auto& XPMultiplier : EnemyTypeXPMultipliers)
    {
        if (EnemyASC->HasMatchingGameplayTag(XPMultiplier.Key))
        {
            return XPMultiplier.Value;
        }
    }
    
    // Default multiplier
    return 1.0f;
}
```

## Ability Unlocking and Upgrading

### 1. Ability Unlock System

Create classes to handle ability unlocking:

```cpp
// PlatformerUnlockableAbility.h
#pragma once

#include "CoreMinimal.h"
#include "PlatformerGameplayAbility.h"
#include "PlatformerUnlockableAbility.generated.h"

/**
 * Gameplay ability that can be unlocked and upgraded.
 */
UCLASS()
class PLATFORMER_API UPlatformerUnlockableAbility : public UPlatformerGameplayAbility
{
    GENERATED_BODY()

public:
    UPlatformerUnlockableAbility();
    
    // Get the current level of this ability
    UFUNCTION(BlueprintCallable, Category = "Ability")
    int32 GetAbilityLevel() const;
    
    // Get the maximum level for this ability
    UFUNCTION(BlueprintCallable, Category = "Ability")
    int32 GetMaxAbilityLevel() const;
    
    // Get the cost to unlock this ability
    UFUNCTION(BlueprintCallable, Category = "Ability")
    int32 GetUnlockCost() const;
    
    // Get the cost to upgrade this ability
    UFUNCTION(BlueprintCallable, Category = "Ability|Upgrade")
    int32 GetUpgradeCost(int32 CurrentLevel) const;
    
    // Check if the ability can be upgraded further
    UFUNCTION(BlueprintCallable, Category = "Ability|Upgrade")
    bool CanUpgrade(int32 CurrentLevel) const;
    
protected:
    // Maximum level this ability can reach
    UPROPERTY(EditDefaultsOnly, Category = "Ability|Progression")
    int32 MaxLevel = 3;
    
    // Cost in skill points to unlock the ability
    UPROPERTY(EditDefaultsOnly, Category = "Ability|Progression")
    int32 UnlockCost = 1;
    
    // Base cost to upgrade, increases with level
    UPROPERTY(EditDefaultsOnly, Category = "Ability|Progression")
    int32 BaseUpgradeCost = 1;
    
    // Scaling factor for upgrade costs
    UPROPERTY(EditDefaultsOnly, Category = "Ability|Progression")
    float UpgradeCostScaling = 1.5f;
    
    // Description of what changes with each level
    UPROPERTY(EditDefaultsOnly, Category = "Ability|Progression")
    TArray<FText> LevelDescriptions;
    
    // Prerequisites for unlocking
    UPROPERTY(EditDefaultsOnly, Category = "Ability|Progression")
    TArray<TSubclassOf<UPlatformerUnlockableAbility>> PrerequisiteAbilities;
    
    // Required character level to unlock
    UPROPERTY(EditDefaultsOnly, Category = "Ability|Progression")
    int32 RequiredCharacterLevel = 1;
};
```

```cpp
// PlatformerUnlockableAbility.cpp
#include "PlatformerUnlockableAbility.h"

UPlatformerUnlockableAbility::UPlatformerUnlockableAbility()
{
    // Default initialization
}

int32 UPlatformerUnlockableAbility::GetAbilityLevel() const
{
    return FMath::RoundToInt(GetAbilityLevel());
}

int32 UPlatformerUnlockableAbility::GetMaxAbilityLevel() const
{
    return MaxLevel;
}

int32 UPlatformerUnlockableAbility::GetUnlockCost() const
{
    return UnlockCost;
}

int32 UPlatformerUnlockableAbility::GetUpgradeCost(int32 CurrentLevel) const
{
    if (CurrentLevel <= 0 || CurrentLevel >= MaxLevel)
    {
        return 0;
    }
    
    // Scale cost with level
    return FMath::CeilToInt(BaseUpgradeCost * FMath::Pow(UpgradeCostScaling, CurrentLevel - 1));
}

bool UPlatformerUnlockableAbility::CanUpgrade(int32 CurrentLevel) const
{
    return CurrentLevel > 0 && CurrentLevel < MaxLevel;
}
```

### 2. Ability Tree Manager

Create a component to manage ability trees and progression:

```cpp
// PlatformerAbilityTreeComponent.h
#pragma once

#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "GameplayTagContainer.h"
#include "Abilities/GameplayAbility.h"
#include "PlatformerAbilityTreeComponent.generated.h"

class UPlatformerUnlockableAbility;
class UAbilitySystemComponent;

// Struct to track unlocked ability state
USTRUCT(BlueprintType)
struct FUnlockedAbilityInfo
{
    GENERATED_BODY()
    
    // The ability class
    UPROPERTY(BlueprintReadOnly)
    TSubclassOf<UGameplayAbility> AbilityClass;
    
    // Current level of the ability
    UPROPERTY(BlueprintReadOnly)
    int32 AbilityLevel = 0;
    
    // The granted ability spec handle (if active)
    UPROPERTY()
    FGameplayAbilitySpecHandle AbilitySpecHandle;
};

/**
 * Component for managing ability trees and progression.
 */
UCLASS(ClassGroup=(Platformer), meta=(BlueprintSpawnableComponent))
class PLATFORMER_API UPlatformerAbilityTreeComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    UPlatformerAbilityTreeComponent();
    
    // Initialize component with ability system
    UFUNCTION(BlueprintCallable, Category = "Ability Tree")
    void InitializeWithAbilitySystem(UAbilitySystemComponent* InAbilitySystem);
    
    // Unlock a new ability
    UFUNCTION(BlueprintCallable, Category = "Ability Tree")
    bool UnlockAbility(TSubclassOf<UPlatformerUnlockableAbility> AbilityClass);
    
    // Upgrade an existing ability
    UFUNCTION(BlueprintCallable, Category = "Ability Tree")
    bool UpgradeAbility(TSubclassOf<UPlatformerUnlockableAbility> AbilityClass);
    
    // Get info about an unlocked ability
    UFUNCTION(BlueprintCallable, Category = "Ability Tree")
    bool GetAbilityInfo(TSubclassOf<UPlatformerUnlockableAbility> AbilityClass, FUnlockedAbilityInfo& OutInfo) const;
    
    // Check if an ability is unlocked
    UFUNCTION(BlueprintCallable, Category = "Ability Tree")
    bool IsAbilityUnlocked(TSubclassOf<UPlatformerUnlockableAbility> AbilityClass) const;
    
    // Check if an ability can be unlocked
    UFUNCTION(BlueprintCallable, Category = "Ability Tree")
    bool CanUnlockAbility(TSubclassOf<UPlatformerUnlockableAbility> AbilityClass) const;
    
    // Check if an ability can be upgraded
    UFUNCTION(BlueprintCallable, Category = "Ability Tree")
    bool CanUpgradeAbility(TSubclassOf<UPlatformerUnlockableAbility> AbilityClass) const;
    
    // Get all available abilities
    UFUNCTION(BlueprintCallable, Category = "Ability Tree")
    void GetAvailableAbilities(TArray<TSubclassOf<UPlatformerUnlockableAbility>>& OutAbilities) const;
    
    // Get all unlocked abilities
    UFUNCTION(BlueprintCallable, Category = "Ability Tree")
    void GetUnlockedAbilities(TArray<FUnlockedAbilityInfo>& OutAbilities) const;

protected:
    // Reference to the owner's ability system
    UPROPERTY()
    UAbilitySystemComponent* AbilitySystemComponent;
    
    // Map of unlocked abilities and their current level
    UPROPERTY()
    TMap<TSubclassOf<UGameplayAbility>, FUnlockedAbilityInfo> UnlockedAbilities;
    
    // All available ability classes that can be unlocked
    UPROPERTY(EditDefaultsOnly, Category = "Ability Tree")
    TArray<TSubclassOf<UPlatformerUnlockableAbility>> AvailableAbilities;
    
    // Check if prerequisites are met for an ability
    bool ArePrerequisitesMet(UPlatformerUnlockableAbility* Ability) const;
    
    // Check if character level requirement is met
    bool IsCharacterLevelSufficient(UPlatformerUnlockableAbility* Ability) const;
    
    // Get current skill points
    int32 GetCurrentSkillPoints() const;
    
    // Spend skill points
    bool SpendSkillPoints(int32 Amount);
};
```

We'll need to continue in a second file since this is getting large: 