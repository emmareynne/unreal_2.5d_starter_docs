# Interface Definitions for 2.5D Platformer

This guide provides a detailed approach to creating reusable interfaces for your 2.5D Platformer project in Unreal Engine 5.5. Interfaces allow you to define common behaviors that can be implemented by different types of objects, promoting code reuse and flexibility.

## What are Interfaces?

In Unreal Engine, an interface is a collection of function declarations without implementations. They serve as a contract between the interface itself and the classes that implement it. Any class that implements an interface must provide implementations for all the functions declared in that interface.

Interfaces are especially useful in gameplay programming for:
- Allowing objects of different types to share common behaviors
- Creating systems that can interact with objects based on behavior rather than type
- Enabling Blueprint and C++ implementations of the same functionality
- Reducing dependencies between systems

For our 2.5D platformer, we'll create three core interfaces:
- **Damageable**: For entities that can receive damage
- **Interactable**: For objects players can interact with
- **MovementModifier**: For objects that modify character movement

## Prerequisites

- Unreal Engine 5.5 installed
- Platformer project created with the recommended plugins
- [GameInstance](./1_game_instance_implementation.md) implemented
- [GameMode](./2_game_mode_implementation.md) implemented
- Basic understanding of C++ (though no prior Unreal experience is assumed)

## Implementation Steps

### Step 1: Setting Up the Project Structure

First, let's create a directory for our interfaces:

1. Open your project in File Explorer
2. Navigate to the Source/Platformer/ folder
3. Create a new folder named "Interfaces"

This will be where we store all our interface files, keeping our project organized.

### Step 2: Creating the Damageable Interface

The Damageable interface will define functionality for any object that can take damage, such as the player character, enemies, and destructible objects.

In Unreal Engine 5.5, there are two ways to create interface classes:

#### Method 1: Start with a standard class and convert it

1. Open your Platformer project in Unreal Editor
2. From the main menu, select **Tools → New C++ Class**
3. Click **Show All Classes**
4. Select **Object** (not "Unreal Interface" as that option may not be available in UE 5.5)
5. Click **Next**
6. Name your class `DamageableInterface` (UE will automatically add 'U' prefix)
7. Click **Create Class**
8. When the header file opens, replace its contents with the interface code below

#### Method 2: Manually create the interface files

If you prefer, you can manually create the interface files in your project's Source directory:

1. Create a new C++ header file named `DamageableInterface.h` in your project's Source/[YourGameName]/Interfaces directory
2. Create a new C++ source file named `DamageableInterface.cpp` in the same directory
3. Add the code below to these files

Regardless of the method you choose, here is the code for your Damageable interface header file:

```cpp
// DamageableInterface.h
#pragma once

#include "CoreMinimal.h"
#include "UObject/Interface.h"
#include "DamageableInterface.generated.h"

// This class does not need to be modified.
UINTERFACE(MinimalAPI, Blueprintable, BlueprintType)
class UDamageableInterface : public UInterface
{
    GENERATED_BODY()
};

/**
 * Interface for objects that can receive damage
 */
class PLATFORMER_API IDamageableInterface
{
    GENERATED_BODY()

public:
    // Apply damage to this object
    UFUNCTION(BlueprintCallable, BlueprintNativeEvent, Category = "Damage")
    float TakeDamage(float DamageAmount, FDamageEvent const& DamageEvent, AController* EventInstigator, AActor* DamageCauser);
    
    // Check if the object is currently alive/active
    UFUNCTION(BlueprintCallable, BlueprintNativeEvent, Category = "Damage")
    bool IsAlive();
    
    // Get the current health percentage (0.0 - 1.0)
    UFUNCTION(BlueprintCallable, BlueprintNativeEvent, Category = "Damage")
    float GetHealthPercent();
};
```

And for the implementation file:

```cpp
// DamageableInterface.cpp
#include "DamageableInterface.h"

// Add default functionality here for any IDamageableInterface functions that are not pure virtual.
```

### Step 3: Creating the Interactable Interface

The Interactable interface will define functionality for objects that players can interact with, such as doors, switches, collectibles, and NPCs.

1. Create a new C++ Interface class named `InteractableInterface`
2. Open the header file and modify it with the following code:

```cpp
// InteractableInterface.h
#pragma once

#include "CoreMinimal.h"
#include "UObject/Interface.h"
#include "InteractableInterface.generated.h"

// This class does not need to be modified.
UINTERFACE(MinimalAPI, Blueprintable, BlueprintType)
class UInteractableInterface : public UInterface
{
    GENERATED_BODY()
};

/**
 * Interface for objects that can be interacted with
 */
class PLATFORMER_API IInteractableInterface
{
    GENERATED_BODY()

public:
    // Called when an actor interacts with this object
    UFUNCTION(BlueprintCallable, BlueprintNativeEvent, Category = "Interaction")
    void OnInteract(AActor* Interactor);
    
    // Get interaction prompt text to display
    UFUNCTION(BlueprintCallable, BlueprintNativeEvent, Category = "Interaction")
    FText GetInteractionText();
    
    // Check if this object can currently be interacted with
    UFUNCTION(BlueprintCallable, BlueprintNativeEvent, Category = "Interaction")
    bool CanInteract(AActor* Interactor);
    
    // Called when an actor begins focusing on this interactable
    UFUNCTION(BlueprintCallable, BlueprintNativeEvent, Category = "Interaction")
    void OnBeginFocus();
    
    // Called when an actor stops focusing on this interactable
    UFUNCTION(BlueprintCallable, BlueprintNativeEvent, Category = "Interaction")
    void OnEndFocus();
};
```

Update the implementation file:

```cpp
// InteractableInterface.cpp
#include "InteractableInterface.h"

// Add default functionality here for any IInteractableInterface functions that are not pure virtual.
```

### Step 4: Creating the MovementModifier Interface

The MovementModifier interface will define functionality for objects that can modify a character's movement, such as speed boosters, jump pads, and slippery surfaces.

1. Create a new C++ Interface class named `MovementModifierInterface`
2. Open the header file and modify it with the following code:

```cpp
// MovementModifierInterface.h
#pragma once

#include "CoreMinimal.h"
#include "UObject/Interface.h"
#include "MovementModifierInterface.generated.h"

// This class does not need to be modified.
UINTERFACE(MinimalAPI, Blueprintable, BlueprintType)
class UMovementModifierInterface : public UInterface
{
    GENERATED_BODY()
};

/**
 * Interface for objects that modify character movement
 */
class PLATFORMER_API IMovementModifierInterface
{
    GENERATED_BODY()

public:
    // Apply a movement modification to the character's movement component
    UFUNCTION(BlueprintCallable, BlueprintNativeEvent, Category = "Movement")
    void ModifyCharacterMovement(UCharacterMovementComponent* MovementComponent);
    
    // Remove a previously applied movement modification
    UFUNCTION(BlueprintCallable, BlueprintNativeEvent, Category = "Movement")
    void RemoveCharacterMovementModification(UCharacterMovementComponent* MovementComponent);
    
    // Get the type of movement modification
    UFUNCTION(BlueprintCallable, BlueprintNativeEvent, Category = "Movement")
    FName GetMovementModifierType();
};
```

Update the implementation file:

```cpp
// MovementModifierInterface.cpp
#include "MovementModifierInterface.h"

// Add default functionality here for any IMovementModifierInterface functions that are not pure virtual.
```

### Step 5: Implementing the Interfaces in C++

Now that we have defined our interfaces, let's implement one of them in a C++ class to demonstrate proper usage.

For example, to implement the Damageable interface in a custom actor:

```cpp
// BreakableBox.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "Interfaces/DamageableInterface.h"
#include "BreakableBox.generated.h"

UCLASS()
class PLATFORMER_API ABreakableBox : public AActor, public IDamageableInterface
{
    GENERATED_BODY()
    
public:    
    // Sets default values for this actor's properties
    ABreakableBox();

protected:
    // Called when the game starts or when spawned
    virtual void BeginPlay() override;

public:    
    // Called every frame
    virtual void Tick(float DeltaTime) override;

    // IDamageableInterface implementation
    virtual float TakeDamage_Implementation(float DamageAmount, FDamageEvent const& DamageEvent, AController* EventInstigator, AActor* DamageCauser) override;
    virtual bool IsAlive_Implementation() override;
    virtual float GetHealthPercent_Implementation() override;

private:
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Damage", meta = (AllowPrivateAccess = "true"))
    float MaxHealth = 100.0f;
    
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Damage", meta = (AllowPrivateAccess = "true"))
    float CurrentHealth = 100.0f;
    
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Damage", meta = (AllowPrivateAccess = "true"))
    TArray<UStaticMeshComponent*> DamageStateMeshes;
    
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "VFX", meta = (AllowPrivateAccess = "true"))
    UParticleSystem* BreakEffect;
};
```

And the implementation:

```cpp
// BreakableBox.cpp
#include "BreakableBox.h"
#include "Kismet/GameplayStatics.h"

ABreakableBox::ABreakableBox()
{
    PrimaryActorTick.bCanEverTick = true;
}

void ABreakableBox::BeginPlay()
{
    Super::BeginPlay();
    CurrentHealth = MaxHealth;
}

void ABreakableBox::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);
}

float ABreakableBox::TakeDamage_Implementation(float DamageAmount, FDamageEvent const& DamageEvent, AController* EventInstigator, AActor* DamageCauser)
{
    CurrentHealth = FMath::Max(0.0f, CurrentHealth - DamageAmount);
    
    // Update visual state based on health percentage
    float HealthPercent = GetHealthPercent_Implementation();
    int32 StateIndex = FMath::FloorToInt((1.0f - HealthPercent) * DamageStateMeshes.Num());
    StateIndex = FMath::Clamp(StateIndex, 0, DamageStateMeshes.Num() - 1);
    
    // Hide all meshes except the current state
    for (int32 i = 0; i < DamageStateMeshes.Num(); i++)
    {
        if (DamageStateMeshes[i])
        {
            DamageStateMeshes[i]->SetVisibility(i == StateIndex);
        }
    }
    
    // If health is zero, spawn break effect and destroy actor
    if (CurrentHealth <= 0)
    {
        if (BreakEffect)
        {
            UGameplayStatics::SpawnEmitterAtLocation(GetWorld(), BreakEffect, GetActorTransform());
        }
        
        // Delay destruction to allow effects to play
        SetLifeSpan(0.1f);
    }
    
    return DamageAmount;
}

bool ABreakableBox::IsAlive_Implementation()
{
    return CurrentHealth > 0;
}

float ABreakableBox::GetHealthPercent_Implementation()
{
    return (MaxHealth > 0) ? (CurrentHealth / MaxHealth) : 0.0f;
}
```

### Step 6: Using Interfaces in Blueprint

One of the benefits of interfaces is the ability to implement them in both C++ and Blueprint. Here's how to implement an interface in Blueprint:

1. Create a new Blueprint class
2. Open the Blueprint editor
3. In the "Class Settings" panel, go to the "Interfaces" section
4. Click the "Add" button and select one of our interfaces (e.g., `DamageableInterface`)
5. Once added, the interface functions will appear in the Functions panel under the category "Interface"
6. Right-click on a function (e.g., `TakeDamage`) and select "Implement Function"
7. This will create a new graph for that function where you can add Blueprint logic

If you don't see your interface in the list, make sure:
- Your interface is marked with `BlueprintType` in the UINTERFACE specifier
- You have compiled your C++ code
- You are looking in the "ALL CLASSES" tab, not just "COMMON" classes

### Step 7: Detecting and Using Interfaces

To detect if an actor implements an interface and call interface functions:

#### In C++:

```cpp
// NEVER use Cast<> for interfaces that might be implemented in Blueprint
// This will NOT work for Blueprint-only implementations:
// auto Interface = Cast<IDamageableInterface>(Actor); // DON'T DO THIS!

// Instead, always use this approach:
AActor* Target = /* some actor reference */;
if (Target && Target->Implements<UDamageableInterface>())
{
    // Call the interface function
    float Damage = IInteractableInterface::Execute_TakeDamage(Target, 10.0f, FDamageEvent(), nullptr, this);
    
    // Get health percentage 
    float HealthPercent = IDamageableInterface::Execute_GetHealthPercent(Target);
    // Use the result...
}
```

#### In Blueprint:

1. Get a reference to the target actor
2. Drag off from the actor reference and search for "Does Implement Interface"
3. Select the interface type (e.g., `DamageableInterface`)
4. Connect to a Branch node to check the result
5. From the actor reference, drag off and search for the interface function (e.g., "Take Damage")
6. Connect any required parameters

### Step 8: Storing Interface References

#### In C++:

When storing interface references in C++, avoid using `TScriptInterface<>` as it can be unreliable with Blueprint-only implementations. Instead, use a `UObject*` pointer:

```cpp
// Safer approach for mixed C++/Blueprint environments
UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Interfaces")
UObject* DamageableObject;

// When using it:
if (DamageableObject && DamageableObject->Implements<UDamageableInterface>())
{
    float Damage = IDamageableInterface::Execute_TakeDamage(DamageableObject, 10.0f, FDamageEvent(), nullptr, this);
}
```

#### In Blueprint:

In Blueprints, you can simply create a variable of your interface type, as long as your interface is marked with `BlueprintType`:

1. In your Blueprint, click the "+" button in the Variables section
2. Select your interface type (e.g., "Damageable Interface")
3. Name your variable and set its properties

### Step 9: Finding Blueprint Implementations

To find all Blueprints that implement a particular interface:

1. In the Content Browser, locate your interface asset
2. Right-click on it and select "Reference Viewer"
3. This will show all assets that reference this interface, including Blueprints that implement it
4. You can use the search box to filter the results

## Best Practices

1. **Keep Interfaces Focused**: Each interface should serve a specific purpose and contain only related functions.

2. **Design with Both C++ and Blueprint in Mind**: Use `BlueprintNativeEvent` to ensure interfaces can be implemented and used in both C++ and Blueprint.

3. **Interface Function Specifiers**:
   - `BlueprintImplementableEvent`: Only implementable in Blueprint
   - `BlueprintNativeEvent`: Implementable in both C++ and Blueprint (generally the most versatile choice)

4. **Error Handling**: Always check if the target implements the interface before calling interface functions.

5. **Interface Detection**: Use `Actor->Implements<UInterfaceName>()` instead of `Cast<IInterfaceName>` as the latter won't work for Blueprint-only implementations.

6. **Interface Naming Conventions**: 
   - C++ interfaces should start with an 'I' prefix (e.g., `IDamageableInterface`)
   - The corresponding UInterface class starts with a 'U' prefix (e.g., `UDamageableInterface`)

7. **Interface Specifier Order**: When using both `NotBlueprintable` and `BlueprintType`, place `NotBlueprintable` first:
   ```cpp
   UINTERFACE(MinimalAPI, NotBlueprintable, BlueprintType)
   ```

8. **Function Implementation Naming**: In C++, use the `_Implementation` suffix for interface function implementations.

9. **Document Expected Behavior**: Clearly document what each interface function is expected to do, especially for complex behaviors.

## Practical Applications in a 2.5D Platformer

### Damageable Interface

- Player character, enemies, and destructible objects implement this interface
- A single damage system can interact with all damageable objects the same way
- Projectiles, traps, and attacks can cause damage without needing to know the specific type of target
- Visual damage indicators can be displayed using the GetHealthPercent function

### Interactable Interface

- Doors, levers, collectibles, and NPCs implement this interface
- Player's interaction system only needs to detect if an object is interactable
- Interaction prompts can be dynamically displayed using GetInteractionText
- Objects can determine if they are interactable based on game state or player state

### MovementModifier Interface

- Conveyor belts, bounce pads, speed boosters, and slippery surfaces implement this interface
- Character movement component can apply/remove modifiers when entering/exiting trigger volumes
- Movement modifiers can stack or override each other based on priority
- Temporary power-ups can apply movement modifications for a limited time

### Integration with UE5.5 Features

With UE5.5's MegaLights system, you could create a `LightInteractionInterface` for objects that:
- Modify nearby lights (dimming, color shifting, etc.)
- React to light intensity (plants that grow, creatures that hide from light)
- Emit dynamic light with shadow casting

This would leverage UE5.5's ability to efficiently handle many dynamic lights in a scene.

## Success Criteria

Your interface implementation is successful when:

- ✅ Multiple different classes can implement the same interface
- ✅ Code can interact with objects through interfaces without knowing their specific type
- ✅ Interfaces can be implemented in both C++ and Blueprint classes
- ✅ Interface functions are properly detected and called
- ✅ The system is extensible for adding new interface implementations

## Next Steps

After completing your interface definitions, you can move on to:

1. Implementing a [Debug Subsystem](./4_debug_subsystem.md) to help visualize and test game systems
2. Setting up [Custom Collision Channels](./5_custom_collision_setup.md) for your 2.5D platformer
3. Integrating with the [Gameplay Ability System](./6_gameplay_ability_integration.md)

## Troubleshooting

**Problem**: Cannot find interface implementation in Blueprint
**Solution**: Make sure the interface is marked with `BlueprintType` and functions are marked with `BlueprintNativeEvent` or `BlueprintImplementableEvent`

**Problem**: "Execute_" functions are not available
**Solution**: Ensure you're using the `Execute_` prefix with the exact function name and proper parameters

**Problem**: Interface not working in Blueprint classes
**Solution**: Check if the interface is properly added in Class Settings and functions are implemented

**Problem**: Getting compile errors when implementing interface functions
**Solution**: Make sure to use the `_Implementation` suffix for function implementations in C++

**Problem**: Cast<> to interface returning nullptr for Blueprint classes
**Solution**: Never use Cast<> for interfaces; always use Actor->Implements<UInterfaceName>() and Execute_ methods

**Problem**: Cannot see interface in "Add Interface" dropdown in Blueprint
**Solution**: Look in the "ALL CLASSES" tab instead of "COMMON" classes, and make sure your interface has the correct UINTERFACE specifiers

## Resources

- [Unreal Engine Interface Documentation](https://docs.unrealengine.com/5.5/en-US/interfaces-in-unreal-engine/)
- [Blueprint Interface Best Practices](https://docs.unrealengine.com/5.5/en-US/blueprint-interfaces-in-unreal-engine/)
- [C++ Programming with Unreal Engine](https://docs.unrealengine.com/5.5/en-US/programming-with-cplusplus-in-unreal-engine/) 