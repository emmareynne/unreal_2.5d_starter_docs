# Custom Collision Setup Supplements

This file contains additional content to enhance the main custom collision setup guide.

## Blueprint Implementation Examples

### Setting Up One-Way Platforms in Blueprint

![One-Way Platform Blueprint Setup](placeholder_image_one_way_platform.png)

```
[BLUEPRINT SCREENSHOT: Create a Blueprint graph showing how to implement one-way platform logic]
```

1. Create a new Blueprint class inheriting from `StaticMeshActor`
2. In the Class Defaults:
   - Set Collision Preset to `Platformer_ThinPlatform`
   - Enable "Generate Overlap Events"

3. Add this logic to the EventGraph:

```
[BLUEPRINT SCREENSHOT: Show the equivalent of the C++ drop-through code]
```

Blueprint logic:
- **Event Begin Play**: Initialize platform behavior
- **OnComponentBeginOverlap**: Check if overlap is from above
  - Get Hit Normal and compare Z component to threshold (e.g., > 0.7)
  - If from above AND not currently dropping through, allow standing on platform
  - If from below, ignore collision
- **Custom Event: DropThrough**:
  - Called when player presses down + jump
  - Temporarily ignore collision with player
  - Start timer to restore collision

### Attack Collision in Blueprint

```
[BLUEPRINT SCREENSHOT: Blueprint implementation of attack collision]
```

Blueprint implementation:
- Create a sphere collision component for attacks
- Set it to use the `Projectile` collision channel
- On attack input:
  - Activate collision briefly
  - Get overlapping actors filtering by `Enemy` channel
  - Apply damage to each overlapping enemy
  - Deactivate collision after attack animation

## Visual Setup Guide

### Project Settings Configuration

```
[SCREENSHOT: Project Settings > Engine > Collision showing custom channels]
```

Key areas to highlight:
- Trace Channel configuration panel
- Object Channel configuration panel
- Preset definitions section

### Collision Visualization Tools

```
[SCREENSHOT: Collision visualization in editor viewport]
```

To enable collision visualization:
1. In viewport, select "Show" > "Collision"
2. Or use console command: `show collision`
3. Different colors represent different collision channels:
   - Red: Blocking collision
   - Green: Overlapping collision
   - Yellow: Custom handling

## Troubleshooting Common 2.5D Collision Issues

### Problem: Character "sticks" to walls
**Symptoms:**
- Character gets caught on wall edges
- Movement feels sticky when sliding along walls

**Solutions:**
- Check capsule collision size - make sure it's appropriate for your character
- Add a small amount of "wall sliding" in your character movement:
```cpp
// In Character Movement Component
void UPlatformerCharacterMovement::PhysicsVolumeChanged(APhysicsVolume* NewVolume)
{
    Super::PhysicsVolumeChanged(NewVolume);
    
    // More slippery along walls
    GroundFriction = 8.0f;
    BrakingDecelerationWalking = 2048.0f;
    
    // Adjust wall sliding values
    WallFriction = 0.1f; // Lower value = more sliding
}
```

In Blueprint:
```
[BLUEPRINT SCREENSHOT: Setting wall friction values]
```

### Problem: Falling through thin platforms
**Symptoms:**
- Character occasionally falls through platforms
- Inconsistent platform detection

**Solutions:**
- Check the collision channel responses
- Ensure "PlatformerGroundTrace" is used for ground detection
- Add these trace debugging tools to verify proper detection:

```cpp
void APlatformerCharacter::DebugGroundDetection()
{
    FHitResult HitResult;
    FVector TraceStart = GetActorLocation();
    FVector TraceEnd = TraceStart - FVector(0, 0, 150.0f);
    
    FCollisionQueryParams QueryParams;
    QueryParams.AddIgnoredActor(this);
    
    FCollisionResponseParams ResponseParams;
    
    // Draw debug line to show ground detection
    DrawDebugLine(GetWorld(), TraceStart, TraceEnd, FColor::Green, false, 1.0f, 0, 1.0f);
    
    if (GetWorld()->LineTraceSingleByChannel(HitResult, TraceStart, TraceEnd, 
        COLLISION_PLATFORMER_GROUND, QueryParams, ResponseParams))
    {
        // Draw a point where we hit ground
        DrawDebugPoint(GetWorld(), HitResult.ImpactPoint, 10.0f, FColor::Red, false, 1.0f);
        DrawDebugString(GetWorld(), HitResult.ImpactPoint, 
            FString::Printf(TEXT("Hit: %s"), *HitResult.GetActor()->GetName()), 
            nullptr, FColor::White, 1.0f);
    }
}
```

In Blueprint:
```
[BLUEPRINT SCREENSHOT: Ground detection debugging]
```

### Problem: Projectiles passing through enemies
**Symptoms:**
- Projectiles fail to hit targets
- Inconsistent hit detection

**Solutions:**
- Check that projectiles are using the proper collision channel
- Ensure they have "Projectile" object type and "Block" response to "Enemy" channel
- For fast-moving projectiles, implement continuous collision detection:

```cpp
// In Projectile class
void APlatformerProjectile::BeginPlay()
{
    Super::BeginPlay();
    
    // Get projectile component
    UProjectileMovementComponent* ProjectileMovement = FindComponentByClass<UProjectileMovementComponent>();
    if (ProjectileMovement)
    {
        // Enable continuous collision detection for fast projectiles
        ProjectileMovement->bSweep = true;
        
        // For very fast projectiles, increase substep time
        ProjectileMovement->MaxSimulationTimeStep = 0.016f; // ~60fps simulation
        ProjectileMovement->MaxSimulationIterations = 8;
    }
    
    // Set collision component to use continuous collision detection
    UPrimitiveComponent* CollisionComp = FindComponentByClass<UPrimitiveComponent>();
    if (CollisionComp)
    {
        CollisionComp->bTraceComplexOnMove = true;
        CollisionComp->bReturnMaterialOnMove = true;
    }
}
```

In Blueprint:
```
[BLUEPRINT SCREENSHOT: Projectile collision setup]
```

### Problem: Z-fighting in visual elements
**Symptoms:**
- Flickering textures between background and gameplay elements
- Visual artifacts at object boundaries

**Solution:**
- Use depth bias in materials for background elements:
```
[BLUEPRINT SCREENSHOT: Material setup with depth bias]
```

- In the material editor:
  1. Add a "Depth Bias" node
  2. Connect it to the material output
  3. Use a small negative value (-0.5 to -2.0) for background elements
  4. Use a small positive value (0.5 to 2.0) for foreground elements

## Performance Profiling for Collision

### How to Profile Collision Performance

1. Enable the built-in profiler:
   - Press Ctrl+Shift+, (comma) in the editor
   - Or use console command: `stat game`

2. Look for these key metrics:
   - Physics time
   - Collision time
   - Tick time for physics actors

```
[SCREENSHOT: Profiler showing collision metrics]
```

### Quick Optimization Checklist

- [ ] Use simple collision shapes everywhere possible
- [ ] Disable collision completely for distant background objects
- [ ] Implement culling for physics objects outside play area
- [ ] Use the BackgroundObject collision preset for decorative elements
- [ ] Check Physics substepping settings in Project Settings
- [ ] Consider using the "Async Scene" for background physics

## Advanced Techniques

### Dynamic Collision Switching for Performance

For large levels with many collision objects, implement dynamic collision switching:

```cpp
void APlatformerLevelManager::UpdateActiveCollision()
{
    // Get player location
    APlatformerCharacter* Player = Cast<APlatformerCharacter>(UGameplayStatics::GetPlayerPawn(this, 0));
    if (!Player) return;
    
    FVector PlayerLocation = Player->GetActorLocation();
    
    // Update collision on all platforms based on distance from player
    for (TActorIterator<APlatform> It(GetWorld()); It; ++It)
    {
        APlatform* Platform = *It;
        float DistanceToPlayer = FVector::Distance(PlayerLocation, Platform->GetActorLocation());
        
        // Enable full collision for nearby platforms
        if (DistanceToPlayer < 1000.0f)
        {
            Platform->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
        }
        // Use simplified collision for mid-range platforms
        else if (DistanceToPlayer < 2000.0f)
        {
            Platform->SetCollisionEnabled(ECollisionEnabled::QueryOnly);
        }
        // Disable collision for distant platforms
        else
        {
            Platform->SetCollisionEnabled(ECollisionEnabled::NoCollision);
        }
    }
}
```

In Blueprint:
```
[BLUEPRINT SCREENSHOT: Dynamic collision switching]
```

## Integration with Animation System

### Collision During Animation Events

For accurate collision during attacks or special moves:

```
[BLUEPRINT SCREENSHOT: Animation Notify setup for collision]
```

Blueprint implementation:
1. Create Animation Notifies in your attack animations
2. On "AttackStart" notify:
   - Enable attack collision component
   - Set appropriate collision channels active
3. On "AttackEnd" notify:
   - Disable attack collision component
   - Reset collision responses

This ensures collision is only active during the appropriate animation frames.

---

## Diagram Templates

```
[Create or add diagrams for the following concepts]
```

1. **2.5D Collision Plane Visualization**
   - Show the main movement plane
   - Indicate depth axis vs. gameplay axes
   - Highlight collision boundaries

2. **Collision Channel Relationships**
   - Create a flowchart showing which channels interact
   - Use arrows to indicate Block/Overlap/Ignore relationships
   - Color-code by channel type

3. **One-Way Platform Mechanics**
   - Illustrate the drop-through sequence
   - Show collision states before/during/after drop-through

---

*Note: Replace all `[BLUEPRINT SCREENSHOT: description]` and `[SCREENSHOT: description]` placeholders with actual screenshots when implementing this guide.* 