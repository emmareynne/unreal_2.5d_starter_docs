# Player Start Position Setup

This guide covers the proper setup of player starting positions in your 2.5D platformer, ensuring your character spawns correctly and consistently in your testing environment.

## Overview

The player start position determines where your character will spawn when the level begins or when the character respawns after death. Proper placement and configuration are essential for testing your character systems and ensuring a smooth gameplay experience.

## Implementation Steps

### Step 1: Add a PlayerStart Actor

1. In your level, search for "PlayerStart" in the Place Actors panel
2. Drag the PlayerStart actor into your level at the desired spawn location
3. Position it slightly above the ground plane to avoid collision issues

```
// Recommended position height
Z = [Floor Height] + [Character Capsule Half-Height] + 5.0f
```

### Step 2: Configure Rotation

For a 2.5D platformer, proper rotation is crucial:

1. Select the PlayerStart actor
2. Set its rotation to face along the gameplay plane
   - For side-scrolling: Typically facing along the X-axis (0, 0, 0) or (-180, 0, 0)
   - For depth-oriented gameplay: Typically facing along the Y-axis (0, 0, 90) or (0, 0, -90)
3. This orientation ensures the character starts facing in the correct direction

### Step 3: PlayerStart Tags and Properties

Configure additional properties for testing flexibility:

1. Select the PlayerStart actor
2. In the Details panel, locate the "Player Start" category
3. Set appropriate tags if needed:
   - PlayerStart Tag: For filtering spawn points (e.g., "Checkpoint1", "LevelStart")
4. Enable "Auto Possess Player" for the first player controller

### Step 4: Multiple Spawn Points

For testing different scenarios, add multiple player starts:

1. Place additional PlayerStart actors at key testing locations
2. Give each a unique tag (e.g., "LedgeTest", "CombatTest", "PlatformTest")
3. In your level blueprint or game mode, implement a system to choose specific spawn points:

```cpp
// Example code for selecting specific PlayerStart
APlayerStart* FindPlayerStart(const FString& TagToFind)
{
    TArray<AActor*> FoundActors;
    UGameplayStatics::GetAllActorsOfClass(GetWorld(), APlayerStart::StaticClass(), FoundActors);
    
    for (AActor* Actor : FoundActors)
    {
        APlayerStart* PlayerStart = Cast<APlayerStart>(Actor);
        if (PlayerStart && PlayerStart->PlayerStartTag == *TagToFind)
        {
            return PlayerStart;
        }
    }
    
    return nullptr;
}
```

### Step 5: Debug Visualization

Add visual indicators to easily identify spawn points:

1. Create a simple Blueprint that extends PlayerStart
2. Add a static mesh component (e.g., an arrow or beacon)
3. Add a text component showing the spawn point's tag
4. Make these components visible only in editor or when debugging is enabled

### Step 6: Checkpoint System Integration

To integrate with a checkpoint system:

1. Create a Blueprint class inheriting from PlayerStart
2. Add a trigger volume to detect when player reaches checkpoint
3. Add functionality to save the checkpoint as the new spawn location:

```cpp
// Example checkpoint activation
void ACheckpointPlayerStart::OnPlayerOverlap(AActor* OverlappedActor)
{
    APlatformerCharacter* Character = Cast<APlatformerCharacter>(OverlappedActor);
    if (Character)
    {
        // Save this checkpoint as active
        APlatformerGameMode* GameMode = Cast<APlatformerGameMode>(GetWorld()->GetAuthGameMode());
        if (GameMode)
        {
            GameMode->SetActiveCheckpoint(this);
        }
        
        // Visual/audio feedback
        ActivateCheckpointEffects();
    }
}
```

### Step 7: Safe Spawn Verification

Add safety checks to ensure the spawn position is valid:

1. Implement a function to verify spawn location safety:

```cpp
bool IsSpawnPositionSafe(const FVector& Location)
{
    // Check if location is inside geometry
    FCollisionQueryParams QueryParams;
    QueryParams.bTraceComplex = false;
    
    bool bIsBlocked = GetWorld()->OverlapBlockingTestByChannel(
        Location, 
        FQuat::Identity,
        ECC_Pawn,
        FCollisionShape::MakeSphere(50.0f),
        QueryParams
    );
    
    // Check if there's ground below
    FHitResult HitResult;
    FVector Start = Location;
    FVector End = Start - FVector(0, 0, 100.0f);
    
    bool bHasGroundBelow = GetWorld()->LineTraceSingleByChannel(
        HitResult,
        Start,
        End,
        ECC_Visibility
    );
    
    return !bIsBlocked && bHasGroundBelow;
}
```

## Best Practices

1. **Placement Height**: Always position the PlayerStart slightly above the ground
2. **Clear Area**: Ensure there's enough space around the spawn point to avoid collisions
3. **Forward Direction**: Align the PlayerStart's forward vector with the expected gameplay direction
4. **Testing Locations**: Place spawn points near specific test scenarios (platform sequences, combat areas)
5. **Visual Indicators**: Add visual markers around spawn points that are visible in editor
6. **Respawn Safety**: Implement checks to prevent spawning inside geometry
7. **Multiple Points**: Use multiple tagged spawn points for different testing scenarios

## Common Issues and Solutions

| Issue | Solution |
|-------|----------|
| Character spawns inside ground | Increase Z position of PlayerStart |
| Character immediately falls | Ensure PlayerStart is above solid ground |
| Character faces wrong direction | Adjust PlayerStart rotation |
| Collision on spawn | Clear area around PlayerStart or implement a spawn protection system |
| Can't find specific spawn point | Verify PlayerStartTag is set correctly |

## Testing

Verify your PlayerStart setup by:

1. Testing initial level load to ensure character spawns correctly
2. Testing respawn functionality if implemented
3. Verifying character orientation on spawn
4. Testing all auxiliary spawn points
5. Checking checkpoint functionality if implemented

## Next Steps

After setting up player start positions:

1. Implement respawn functionality
2. Create a checkpoint system
3. Add spawn effects or animations
4. Implement level transition spawn points 