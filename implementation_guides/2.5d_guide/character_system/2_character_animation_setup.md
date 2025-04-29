# Character Animation Setup for 2.5D Platformer

This guide covers setting up a robust animation system for your 2.5D platformer character in Unreal Engine 5.5, focusing on responsive feedback and smooth transitions between states.

## Overview

The animation system is critical for providing visual feedback and gameplay feel, consisting of:
- Animation blueprint with state machine
- Animation montages for specific actions
- Position adjustment during animations for environmental interactions
- Animation notifies for gameplay events
- Animation curves for fine-tuned control

By implementing a well-structured animation system, your character will respond fluidly to player input and environmental interactions.

## Prerequisites

Before setting up animations:
- Completed [Character Base Implementation](1_character_base_implementation.md)
- Imported character skeletal mesh with animations
- Set up basic movement functionality
- Configured camera for proper 2.5D view

## Implementation Steps

### Step 1: Create the Animation Blueprint

1. **Create Animation Blueprint**:
   - In Content Browser, right-click and select **Animation → Animation Blueprint**
   - Select your character's skeleton
   - Name it `ABP_PlatformerCharacter`

2. **Set Up State Machine**:
   - Double-click to open the Animation Blueprint
   - In AnimGraph, create a new State Machine named `Locomotion`
   - Add the following states:
     - Idle
     - Run
     - Jump
     - Fall
     - Land

3. **Create State Transitions**:
   - Idle → Run: `IsMoving` is true
   - Run → Idle: `IsMoving` is false
   - Idle/Run → Jump: `IsJumping` is true
   - Jump → Fall: `IsFalling` is true
   - Fall → Land: `ShouldLand` is true
   - Land → Idle/Run: Based on `IsMoving`

### Step 2: Create Variables and Functions

In the EventGraph of your Animation Blueprint:

1. **Create Variables**:
   ```
   bool IsMoving
   bool IsJumping
   bool IsFalling
   bool ShouldLand
   float MovementSpeed
   FVector Velocity
   ```

2. **Create AnimNotify Events**:
   - JumpStart_Notify
   - JumpApex_Notify
   - Footstep_Notify
   - Attack_Notify

3. **Initialize in AnimGraph**:
   ```cpp
   // Event Blueprint Update Animation
   void AnimBP_PlatformerCharacter::BlueprintUpdateAnimation(float DeltaTimeX)
   {
       // Get owner
       APlatformerCharacter* Character = Cast<APlatformerCharacter>(TryGetPawnOwner());
       if (!Character)
       {
           return;
       }
       
       // Get velocity
       Velocity = Character->GetVelocity();
       float Speed = Velocity.Size();
       
       // Update variables
       MovementSpeed = Speed;
       IsMoving = Speed > 10.0f;
       
       // Get movement component
       UCharacterMovementComponent* MovementComponent = Character->GetCharacterMovement();
       if (MovementComponent)
       {
           IsFalling = MovementComponent->IsFalling();
           IsJumping = IsFalling && Velocity.Z > 0;
           
           // Detect landing (was falling, now not falling)
           if (WasFalling && !IsFalling)
           {
               ShouldLand = true;
           }
           else
           {
               ShouldLand = false;
           }
           
           WasFalling = IsFalling;
       }
   }
   ```

### Step 3: Set Up Animation Montages

1. **Create Jump Montage**:
   - In Content Browser, find your jump animation
   - Right-click and select **Create → Animation Montage**
   - Name it `AM_Jump`
   - Set up sections: "Start", "Loop", "End"
   - Add animation notifies at key points:
     - JumpStart_Notify at start of jump
     - JumpApex_Notify at highest point
     - Land_Notify at landing frame

2. **Create Attack Montage**:
   - Create `AM_Attack` from attack animation
   - Add sections for combo if needed
   - Add Attack_Notify at hit frames
   - Configure blend in/out settings

3. **Create Ledge Grab Montage**:
   - Create `AM_LedgeGrab` from ledge grab animation
   - Add position adjustment notify
   - Configure root motion if needed

### Step 4: Implement Animation Blueprint Logic

Set up the state machine contents:

1. **Idle State**:
   - Play idle animation
   - Add idle variation blending for natural movement

2. **Run State**:
   - Play run animation
   - Scale playrate based on MovementSpeed

3. **Jump State**:
   - Play jump start animation
   - Trigger montage at transition

4. **Fall State**:
   - Play falling animation
   - Blend based on air time

5. **Land State**:
   - Play landing animation
   - Vary intensity based on fall height

### Step 5: Set Up Position Adjustment with Animation Notifies

1. **Create Custom Notify**:
   - Create a new Animation Notify class `AnimNotify_PositionAdjust`
   - Implement position adjustment parameter passing

```cpp
// AnimNotify_PositionAdjust.h
UCLASS()
class PLATFORMER_API UAnimNotify_PositionAdjust : public UAnimNotify
{
    GENERATED_BODY()
    
public:
    // The relative offset to apply
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Position Adjustment")
    FVector PositionOffset;
    
    // Duration of adjustment
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Position Adjustment")
    float AdjustmentDuration = 0.2f;
    
    // Notify event implementation
    virtual void Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation) override;
};

// AnimNotify_PositionAdjust.cpp
void UAnimNotify_PositionAdjust::Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation)
{
    if (!MeshComp || !MeshComp->GetOwner())
    {
        return;
    }
    
    APlatformerCharacter* Character = Cast<APlatformerCharacter>(MeshComp->GetOwner());
    if (!Character)
    {
        return;
    }
    
    // Calculate world space position
    FVector CurrentLocation = Character->GetActorLocation();
    FVector TargetLocation = CurrentLocation + Character->GetActorRotation().RotateVector(PositionOffset);
    
    // Start adjustment
    Character->StartPositionAdjustment(TargetLocation, AdjustmentDuration);
}
```

2. **Add to Montages**:
   - Add the PositionAdjust notify to ledge grab and other environmental interaction animations
   - Configure appropriate offset and duration values

### Step 6: Link Character Class to Animation System

1. **Update Character Class**:
   - Ensure animation montage references are set
   - Add methods to trigger montages

```cpp
// In PlatformerCharacter.h
UFUNCTION(BlueprintCallable, Category = "Animation")
void PlayJumpMontage();

UFUNCTION(BlueprintCallable, Category = "Animation")
void PlayLedgeGrabMontage();

UFUNCTION(BlueprintCallable, Category = "Animation")
void PlayAttackMontage();

UFUNCTION(BlueprintCallable, Category = "Animation")
void OnAnimNotify(FName NotifyName);

// In PlatformerCharacter.cpp
void APlatformerCharacter::PlayJumpMontage()
{
    if (JumpMontage && GetMesh() && GetMesh()->AnimScriptInstance)
    {
        GetMesh()->AnimScriptInstance->Montage_Play(JumpMontage);
    }
}

void APlatformerCharacter::PlayLedgeGrabMontage()
{
    if (LedgeGrabMontage && GetMesh() && GetMesh()->AnimScriptInstance)
    {
        GetMesh()->AnimScriptInstance->Montage_Play(LedgeGrabMontage);
    }
}

void APlatformerCharacter::PlayAttackMontage()
{
    if (CombatComponent)
    {
        CombatComponent->StartAttack();
    }
}

void APlatformerCharacter::OnAnimNotify(FName NotifyName)
{
    // Handle different anim notifies
    if (NotifyName == "JumpStart_Notify")
    {
        // Jump start logic
    }
    else if (NotifyName == "Attack_Notify")
    {
        // Apply attack damage
        if (CombatComponent)
        {
            CombatComponent->ApplyDamage();
        }
    }
    else if (NotifyName == "Footstep_Notify")
    {
        // Play footstep sound
        // Spawn footstep effect
    }
}
```

2. **Connect with Jump Component**:
   - Trigger animations from component actions

```cpp
// In PlatformerJumpComponent.cpp
void UPlatformerJumpComponent::StartJump()
{
    if (!CharacterOwner)
    {
        return;
    }
    
    // Handle jump count and physics
    if (JumpCount < MaxJumpCount)
    {
        JumpCount++;
        
        // Default jump implementation
        CharacterOwner->Jump();
        
        // Play animation
        APlatformerCharacter* PlatformerChar = Cast<APlatformerCharacter>(CharacterOwner);
        if (PlatformerChar)
        {
            PlatformerChar->PlayJumpMontage();
        }
    }
}
```

### Step 7: Configure Root Motion

For certain animations like ledge grabs, root motion provides more precise control:

1. **Enable Root Motion**:
   - Open the animation asset
   - Enable "Force Root Lock" or "Root Motion" settings based on animation type
   
2. **Process Root Motion**:
   - Set animation blueprint to use root motion
   - Configure the Locomotion State Machine to process root motion when needed

3. **Handle Edge Cases**:
   - Add checks for movement constraints during root motion
   - Create transition rules to handle interrupted root motion

## Advanced Animation Techniques

### 1. Animation Curves for Dynamic Movement

Use animation curves to drive character parameters:

```cpp
// In animation blueprint update
// Get curve value from animation
float LeanAmount = GetCurveValue("Lean");

// Apply to character movement or visuals
Character->SetLeanAmount(LeanAmount);
```

### 2. Additive Animation Blending

For responsive movement feedback:

1. Create additive animations for:
   - Leaning during direction changes
   - Impact reactions
   - Looking at targets

2. In Animation Blueprint:
   - Use "Apply Additive" nodes
   - Blend based on movement parameters
   - Layer on top of base animations

### 3. Inverse Kinematics (IK)

For realistic foot placement on uneven terrain:

1. **Foot IK Setup**:
   - Add foot trace in Animation Blueprint
   - Apply IK adjustments based on surface height
   - Adjust pelvis height to compensate

```cpp
// In AnimGraph
// Perform line traces for each foot
FHitResult LeftFootHit, RightFootHit;
FVector LeftFootLocation = GetBoneLocation("foot_l");
FVector RightFootLocation = GetBoneLocation("foot_r");

// Trace down from each foot
FVector TraceStart = LeftFootLocation;
FVector TraceEnd = TraceStart - FVector(0, 0, IKTraceDistance);
bool bLeftHit = GetWorld()->LineTraceSingleByChannel(LeftFootHit, TraceStart, TraceEnd, ECC_Visibility);

// Similar for right foot...

// Apply IK adjustment based on hit results
```

2. **Hand IK for Environmental Interaction**:
   - Add hand IK for ledge grabs
   - Dynamically adjust hand position based on environment

## Technical Considerations

### Performance Optimization

1. **Animation Blueprint Complexity**:
   - Minimize expensive operations in per-frame updates
   - Cache frequently used values
   - Reduce unnecessary state transitions

2. **LOD Systems**:
   - Set up Animation LODs for distance-based simplification
   - Create simpler animations for distant characters

### Animation Debugging

1. **Visual Debugging**:
   - Add debug visualization for animation states
   - Implement animation logging for troubleshooting

```cpp
// In animation blueprint update
if (bDebugAnimation)
{
    // Draw current state on screen
    GEngine->AddOnScreenDebugMessage(-1, 0.0f, FColor::Yellow, 
        FString::Printf(TEXT("Animation State: %s"), *CurrentStateName));
    
    // Visualize foot IK
    DrawDebugLine(GetWorld(), LeftFootLocation, LeftFootHit.Location, FColor::Green, false, -1.0f, 0, 2.0f);
}
```

2. **State Monitoring**:
   - Add animation state monitoring in PIE
   - Create editor tools for animation testing

## Testing Your Animation System

Create a test suite for animation validation:

1. Test state transitions with different movement inputs
2. Verify animation responsiveness during rapid input changes
3. Test ledge grab and position adjustment functionality
4. Validate animation blending during movement direction changes
5. Check root motion precision for specific actions

## Success Criteria

Your animation system is complete when:
- Character animation smoothly transitions between states
- Animations properly reflect character movement speed and direction
- Position adjustments during animations work correctly
- Animation notifies trigger appropriate gameplay actions
- Montages play and blend correctly for special actions
- Character visually responds to environmental interactions
- Animation system performs well with minimal hitches

## Next Steps

After completing animation setup:
1. Implement advanced animation-driven gameplay features
2. Create additional montages for special abilities
3. Refine animation blending for smoother transitions
4. Add facial animations and character expression
5. Implement camera animation integration

Proceed to [Character Interaction System](3_character_interaction_system.md) for implementing environment interaction capabilities. 