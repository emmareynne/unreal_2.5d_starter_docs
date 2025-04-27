# Custom Collision Setup for 2.5D Platformers

This guide covers the implementation of custom collision channels optimized for 2.5D platformers in Unreal Engine 5.5.

## Why Custom Collision Channels?

A 2.5D platformer has specific collision requirements that differ from both 2D and 3D games:

- Movement primarily occurs in a 2D plane while using 3D assets
- Visual depth needs to be distinct from gameplay collision
- Projectiles and attacks may need to pass through some objects but not others
- Performance optimization is critical since most objects don't need full 3D collision detection

By creating custom collision channels, we can:
- Simplify collision logic
- Improve performance
- Create more predictable gameplay
- Solve specific 2.5D gameplay challenges

## Setting Up Collision Channels

### Step 1: Define Channels in Project Settings

1. Open your project
2. Navigate to **Edit → Project Settings → Engine → Collision**
3. Under **Trace Channels**, add the following:

   | Name | Default Response | Description |
   |------|------------------|-------------|
   | Player | Block | Traces from the player character |
   | Enemy | Block | Traces from enemy characters |
   | Projectile | Block | Traces from projectiles |
   | Interaction | Ignore | Traces for interactive objects |
   | PlatformerGroundTrace | Block | Ground detection for platformer movement |
   | ThinPlatform | Overlap | One-way platforms the player can jump through |

4. Under **Object Channels**, add the following:

   | Name | Default Response | Description |
   |------|------------------|-------------|
   | Player | Block | Player character collisions |
   | Enemy | Block | Enemy character collisions |
   | Projectile | Block | Projectile collisions |
   | Pickup | Overlap | Collectible items |
   | ThinPlatform | Overlap | One-way platforms |
   | Hazard | Overlap | Damaging hazards |
   | BackgroundObject | Ignore | Visual-only background elements |

### Step 2: Create Custom Collision Presets

Configure these presets for commonly used object types:

1. **Platformer_Player**
   - Default: Block
   - Player: Ignore (don't collide with self)
   - Pickup: Overlap
   - Interaction: Overlap
   - BackgroundObject: Ignore
   - ThinPlatform: Custom (special handling)
   - Hazard: Overlap

2. **Platformer_Enemy**
   - Default: Block
   - Enemy: Block
   - Player: Block
   - Projectile: Block
   - Pickup: Ignore
   - BackgroundObject: Ignore
   - ThinPlatform: Block

3. **Platformer_Projectile**
   - Default: Block
   - Projectile: Ignore (don't collide with other projectiles)
   - Player: Custom (depends on friendly fire settings)
   - Enemy: Block
   - Pickup: Ignore
   - BackgroundObject: Ignore
   - ThinPlatform: Ignore (pass through thin platforms)

4. **Platformer_ThinPlatform**
   - Default: Ignore
   - Player: Custom (special handling)
   - Enemy: Block
   - Projectile: Ignore
   - ThinPlatform: Ignore (don't collide with other platforms)

5. **Platformer_Pickup**
   - Default: Ignore
   - Player: Overlap
   - Enemy: Ignore
   - Projectile: Ignore
   - BackgroundObject: Ignore

6. **Platformer_BackgroundObject**
   - Default: Ignore (purely visual)
   - BlockAllDynamic: Ignore
   - All custom channels: Ignore

## Implementing One-Way Platforms

One-way platforms are a staple of 2.5D platformers. Here's how to implement them:

```cpp
// In your character movement component
void UPlatformerCharacterMovement::PhysWalking(float deltaTime, int32 Iterations)
{
    // Check if we're trying to drop through a one-way platform
    bool bDropThrough = false;
    if (CharacterOwner->InputComponent && 
        CharacterOwner->InputComponent->GetAxisValue("MoveDown") < -0.5f && 
        IsJumpAllowed())
    {
        // Find if we're standing on a thin platform
        FHitResult FloorHit;
        if (CurrentFloor.IsWalkableFloor() && 
            CurrentFloor.GetHitResult(FloorHit) && 
            FloorHit.GetComponent() && 
            FloorHit.GetComponent()->GetCollisionObjectType() == ECC_GameTraceChannel5) // ThinPlatform channel
        {
            // Disable collision with thin platforms briefly
            TArray<AActor*> IgnoreActors;
            IgnoreActors.Add(CharacterOwner);
            UKismetSystemLibrary::SphereTraceMultiForObjects(
                GetWorld(),
                CharacterOwner->GetActorLocation(),
                CharacterOwner->GetActorLocation() - FVector(0, 0, 50.0f),
                CollisionShape.GetCapsuleRadius(),
                { UEngineTypes::ConvertToObjectType(ECC_GameTraceChannel5) }, // ThinPlatform
                false,
                IgnoreActors,
                EDrawDebugTrace::None,
                FloorHit,
                true
            );
            
            if (FloorHit.bBlockingHit)
            {
                // Set collision response to ignore for thin platforms
                GetCharacterOwner()->MeshComponent->SetCollisionResponseToChannel(
                    ECC_GameTraceChannel5, // ThinPlatform
                    ECR_Ignore
                );
                
                // Start timer to restore collision
                GetWorld()->GetTimerManager().SetTimer(
                    RestorePlatformCollisionTimer,
                    this,
                    &UPlatformerCharacterMovement::RestorePlatformCollision,
                    0.5f,
                    false
                );
                
                bDropThrough = true;
            }
        }
    }
    
    if (!bDropThrough)
    {
        Super::PhysWalking(deltaTime, Iterations);
    }
    else
    {
        // Force into falling state
        SetMovementMode(MOVE_Falling);
    }
}

void UPlatformerCharacterMovement::RestorePlatformCollision()
{
    // Restore collision with thin platforms
    GetCharacterOwner()->MeshComponent->SetCollisionResponseToChannel(
        ECC_GameTraceChannel5, // ThinPlatform
        ECR_Block
    );
}
```

## Special Handling for Player Attacks

For player attacks that need special collision behavior:

```cpp
// In your attack component
void UPlayerAttackComponent::ExecuteAttack()
{
    // Create attack collision
    FCollisionShape Shape = FCollisionShape::MakeSphere(AttackRadius);
    
    // Define collision parameters
    FCollisionQueryParams Params;
    Params.AddIgnoredActor(GetOwner()); // Don't hit self
    
    // Only check for enemy collisions
    TArray<FOverlapResult> Overlaps;
    FCollisionObjectQueryParams ObjectParams;
    ObjectParams.AddObjectTypesToQuery(ECC_GameTraceChannel2); // Enemy channel
    
    // Perform the overlap check
    if (GetWorld()->OverlapMultiByObjectType(
        Overlaps, 
        GetOwner()->GetActorLocation() + AttackOffset, 
        FQuat::Identity, 
        ObjectParams, 
        Shape, 
        Params))
    {
        for (FOverlapResult& Overlap : Overlaps)
        {
            if (Overlap.GetActor())
            {
                // Apply damage to enemy
                UGameplayStatics::ApplyDamage(
                    Overlap.GetActor(),
                    AttackDamage,
                    GetOwner()->GetInstigatorController(),
                    GetOwner(),
                    UDamageType::StaticClass()
                );
            }
        }
    }
}
```

## Optimizing Collision for 2.5D

Since 2.5D games constrain movement primarily to a plane, we can optimize:

### Collision Profile for Static Background Elements

For distant background elements:
1. Set collision preset to **Platformer_BackgroundObject**
2. Disable collision completely (NoCollision)
3. Disable "Generate Overlap Events"

### Limit Collision Range in Z-Axis

For gameplay elements, limit the collision detection range:

```cpp
// In your GameMode or Level Blueprint
void APlatformerGameMode::BeginPlay()
{
    Super::BeginPlay();
    
    // Set boundaries for collision detection
    AWorldSettings* WorldSettings = GetWorldSettings();
    if (WorldSettings)
    {
        // Only perform collision detection within a specific Z range
        // from slightly below the lowest platform to above the highest
        WorldSettings->bEnableWorldBoundsChecks = true;
        WorldSettings->KillZ = -1000.0f; // Below lowest point
        WorldSettings->WorldBounds = FBox(
            FVector(-HALF_WORLD_MAX, -HALF_WORLD_MAX, -1000.0f),
            FVector(HALF_WORLD_MAX, HALF_WORLD_MAX, 1000.0f)
        );
    }
}
```

### Use Hit Result Filtering

Filter hit results to consider only objects in the gameplay plane:

```cpp
bool APlatformerCharacter::ShouldConsiderHitResult(const FHitResult& Hit)
{
    // Get the gameplay plane's normal (usually along Y in a side-scroller)
    const FVector GameplayPlaneNormal = FVector(0, 1, 0);
    
    // Calculate how aligned the hit normal is with the gameplay plane
    float Alignment = FMath::Abs(FVector::DotProduct(Hit.Normal, GameplayPlaneNormal));
    
    // If the hit is nearly perpendicular to our gameplay plane, ignore it
    // This prevents collision with objects that are visually in the background/foreground
    const float AlignmentThreshold = 0.7f;
    return Alignment < AlignmentThreshold;
}
```

## Debugging Collision

Add these tools to your debug system for collision visualization:

```cpp
// In your debug component
void UPlatformerDebugComponent::DrawCollisionSection()
{
    // Toggle collision visualization
    static bool bShowCollision = false;
    if (ImGui::Checkbox("Show Collision", &bShowCollision))
    {
        if (UWorld* World = GetWorld())
        {
            if (bShowCollision)
            {
                World->Exec(World, TEXT("Show Collision"));
            }
            else
            {
                World->Exec(World, TEXT("Show Collision Off"));
            }
        }
    }
    
    // Visualize collision channels
    ImGui::Separator();
    ImGui::Text("Collision Channels");
    
    static bool bShowPlayerCollision = false;
    if (ImGui::Checkbox("Player Collisions", &bShowPlayerCollision))
    {
        ToggleCollisionDisplay(ECC_GameTraceChannel1, bShowPlayerCollision); // Player channel
    }
    
    static bool bShowEnemyCollision = false;
    if (ImGui::Checkbox("Enemy Collisions", &bShowEnemyCollision))
    {
        ToggleCollisionDisplay(ECC_GameTraceChannel2, bShowEnemyCollision); // Enemy channel
    }
    
    static bool bShowPlatformCollision = false;
    if (ImGui::Checkbox("Platform Collisions", &bShowPlatformCollision))
    {
        ToggleCollisionDisplay(ECC_GameTraceChannel5, bShowPlatformCollision); // ThinPlatform channel
    }
}

void UPlatformerDebugComponent::ToggleCollisionDisplay(ECollisionChannel Channel, bool bShow)
{
    // Find all actors using this collision channel
    UWorld* World = GetWorld();
    if (!World) return;
    
    for (TActorIterator<AActor> It(World); It; ++It)
    {
        AActor* Actor = *It;
        TArray<UPrimitiveComponent*> PrimitiveComponents;
        Actor->GetComponents<UPrimitiveComponent>(PrimitiveComponents);
        
        for (UPrimitiveComponent* Comp : PrimitiveComponents)
        {
            if (Comp->GetCollisionObjectType() == Channel || 
                Comp->GetCollisionResponseToChannel(Channel) != ECR_Ignore)
            {
                if (bShow)
                {
                    Comp->SetRenderCustomDepth(true);
                    Comp->SetCustomDepthStencilValue(1);
                }
                else
                {
                    Comp->SetRenderCustomDepth(false);
                }
            }
        }
    }
}
```

## C++ Configuration 

To define your collision channels in C++:

```cpp
// In YourGame.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/Engine.h"

// Custom collision channels
#define COLLISION_PLAYER               ECC_GameTraceChannel1
#define COLLISION_ENEMY                ECC_GameTraceChannel2
#define COLLISION_PROJECTILE           ECC_GameTraceChannel3
#define COLLISION_INTERACTION          ECC_GameTraceChannel4
#define COLLISION_THIN_PLATFORM        ECC_GameTraceChannel5
#define COLLISION_PLATFORMER_GROUND    ECC_GameTraceChannel6
#define COLLISION_PICKUP               ECC_GameTraceChannel7
#define COLLISION_HAZARD               ECC_GameTraceChannel8
#define COLLISION_BACKGROUND_OBJECT    ECC_GameTraceChannel9

// In YourGame.cpp
void ConfigureCollisionChannels()
{
    // Register custom collision channels
    // Note: This should match your Project Settings configuration
    UCollisionProfile::RegisterChannelConfig(
        "PlatformerCollision",
        ECollisionChannel::ECC_GameTraceChannel1,
        TEXT("Player"),
        true,   // Use for simulation
        true,   // Use for queries
        ECollisionResponse::ECR_Block
    );
    
    UCollisionProfile::RegisterChannelConfig(
        "PlatformerCollision",
        ECollisionChannel::ECC_GameTraceChannel2,
        TEXT("Enemy"),
        true,
        true,
        ECollisionResponse::ECR_Block
    );
    
    // Continue for other channels...
}
```

## Performance Considerations

For optimal collision performance in a 2.5D platformer:

1. **Use simple collision shapes** for physics calculations
   - Box, sphere, and capsule colliders are much faster than complex shapes
   - Keep your physics meshes simple, even when visual meshes are detailed

2. **Disable collision on decorative elements**
   - Background elements should have collision completely disabled
   - Apply the "Platformer_BackgroundObject" preset to decorative items

3. **Use overlap events sparingly**
   - Only enable "Generate Overlap Events" for objects that need them (pickups, hazards)
   - Overlap events are more expensive than blocking collisions

4. **Consider collision-based LOD**
   - Simplify collision shapes for distant objects
   - Distant enemies can use simpler collision or disable some collision checks

## Best Practices for Blueprint Users

For level designers and gameplay programmers using Blueprints:

1. **Always use the appropriate collision preset**
   - Use "Platformer_Player" for the player character
   - Use "Platformer_Enemy" for enemies
   - etc.

2. **Group similar objects**
   - Place platforms with similar collision behavior in the same folders
   - Create Blueprint classes for common object types with preset collision

3. **Test collision in isolation**
   - Use the debug visualization to verify collision is working as expected
   - Test edge cases like dropping through platforms, wall jumps, etc.

## Special 2.5D Scenarios

### Z-Locked Movement with 3D Collision

For characters that move only in 2D but need 3D collision:

```cpp
// In character movement component
void UPlatformerCharacterMovement::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
{
    Super::TickComponent(DeltaTime, TickType, ThisTickFunction);
    
    // Lock character to the 2D plane after physics calculation
    if (CharacterOwner)
    {
        FVector Location = CharacterOwner->GetActorLocation();
        
        // Lock to a specific Y value for side-scrollers
        // Or specific X value for top-down perspective
        Location.Y = GameplayPlaneY;
        
        CharacterOwner->SetActorLocation(Location);
    }
}
```

### Handling Depth-Based Collisions

For objects that should only collide when at similar depths:

```cpp
bool AShouldActorsCollide(AActor* ActorA, AActor* ActorB)
{
    // Get depths (Y position for side-scrollers)
    float DepthA = ActorA->GetActorLocation().Y;
    float DepthB = ActorB->GetActorLocation().Y;
    
    // Define the depth threshold for collision
    // Objects further apart than this won't collide
    const float DepthThreshold = 50.0f;
    
    return FMath::Abs(DepthA - DepthB) < DepthThreshold;
}
```

## Conclusion

A properly configured collision system is crucial for 2.5D platformers. By setting up custom channels, optimizing collision shapes, and implementing special handling for platformer-specific features like one-way platforms, you'll create a more responsive, performant, and predictable game experience.

Remember to use the debug visualization tools to verify your collision setup is working as expected, and always test edge cases to ensure a smooth player experience. 