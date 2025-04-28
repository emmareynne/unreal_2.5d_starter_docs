# Initial Scene Setup Overview

This document outlines the process of creating an initial scene for our 2.5D platformer in Unreal Engine 5.5, establishing the foundation for implementing the core gameplay systems.

## What is the Initial Scene?

The initial scene serves as a test environment for our core systems, providing:
- A controlled space to test character movement and collision
- Basic platforms, walls, and obstacles for gameplay testing
- Camera setup specific to 2.5D perspective
- Proper lighting and visual setup for development
- Implementation of the core component-based architecture
- Reference points for level design

## Prerequisites

Before proceeding, ensure you have:
- [Installed and configured all required plugins](./1_prerequisites.md)
- Set up your project with the correct collision channels
- Checked your project settings for compatibility

## Implementation Steps

### 1. Project Configuration Check

Before creating the scene, ensure the project configuration is correct:

```
- Engine Version: Unreal Engine 5.5
- Project Type: C++ Project with Blueprint support
- Required Plugins:
  - Enhanced Input
  - GameplayAbilitySystem
  - CommonUI
  - Niagara
- Project Settings:
  - Input set to Enhanced Input System
  - Collision channels configured per framework docs
  - Default GameMode set to our PlatformerGameMode
```

### 2. Level Creation

Create a dedicated test level with a focused layout:

**Technical Approach:**
- Create a new level using Empty Level template
- Establish a grid-based layout for precise platform placement
- Set world bounds appropriate for a vertical slice of gameplay
- Configure sublevels for organized development (optional)

### 3. Environment Setup

Establish the basic environmental elements:

**Technical Approach:**
- Add a floor plane with proper collision for ground movement
- Create simple platform structures using BSP or Static Meshes
- Set up world boundaries to constrain player movement
- Implement [Lumen lighting](https://docs.unrealengine.com/5.3/en-US/lumen-global-illumination-and-reflections-in-unreal-engine/) for development visibility (replaces static lighting)
- Add visual reference points for scale and distance

### 4. Camera System

Configure a camera system appropriate for 2.5D gameplay:

**Technical Approach:**
- Create a camera manager blueprint extending from [PlayerCameraManager](https://docs.unrealengine.com/5.3/en-US/API/Runtime/Engine/Camera/APlayerCameraManager/)
- Configure orthographic or perspective camera based on visual style preference
- Implement basic tracking behavior (side-scrolling with depth perception)
- Add camera constraints to maintain gameplay visibility
- Set up camera collision to avoid obstruction

### 5. Character Placement

Add placeholder for character testing:

**Technical Approach:**
- Add PlayerStart actors at testing positions
- Configure the GameMode to use our PlatformerCharacter class
- Set spawn rotation to face proper gameplay direction
- Add debug visualization for player paths
- Create respawn points for testing death scenarios

### 6. Basic Art Setup

Establish visual clarity for testing:

**Technical Approach:**
- Use grid textures for platforms to visualize distance
- Create distinct materials for different surface types
- Configure [Virtual Shadow Maps](https://docs.unrealengine.com/5.3/en-US/virtual-shadow-maps-in-unreal-engine/) for better performance and quality
- Add reference objects for scale comparison
- Configure post-processing for development visibility

### 7. Component-Based Architecture Integration

Implement our component-based character framework:

**Technical Approach:**
- Set the level's GameMode override to use PlatformerGameMode
- Create base character class with properly configured components
- Attach specialized components for movement, jumping, and combat
- Configure component dependencies and communication
- Set up debug visualization for component states
- Create test instances with different component configurations

### 8. Enhanced Input Setup

Configure the Enhanced Input system for 2.5D controls:

**Technical Approach:**
- Create [Input Mapping Contexts](https://docs.unrealengine.com/5.3/en-US/enhanced-input-in-unreal-engine/) for different game states
- Set up Input Actions for movement, jumping, and interaction
- Configure appropriate input processors for 2.5D constraints
- Route input to the appropriate character components
- Test input visualization in the debug system

### 9. Animation Montage System Setup

Implement Animation Montages with custom positioning for character actions:

**Technical Approach:**
- Create an Animation Blueprint with a state machine for character states
- Design Animation Montages for key gameplay actions (jumping, ledge grabs)
- Set up animation notifies for precise timing of position adjustments
- Implement timeline-based position correction for environmental interactions
- Create debug visualization for animation transitions and position adjustments
- Test montage playback with different environmental scenarios

### 10. Performance Setup

Establish performance monitoring from the start:

**Technical Approach:**
- Add GPU and CPU visualization viewports
- Configure [Lumen quality settings](https://docs.unrealengine.com/5.3/en-US/lumen-technical-details-in-unreal-engine/) for development
- Set up memory budget monitoring
- Configure scalability settings for testing on different hardware

## Technical Considerations

### 2.5D Specific Requirements
- Constrain movement to a plane while maintaining visual depth
- Configure camera for clear visualization of the gameplay plane
- Set up lighting that enhances depth perception
- Use foreground/background elements to create parallax

### Collision Configuration
- Use simplified collision for fast iteration
- Ensure platforms have proper collision responses
- Configure trigger volumes for gameplay area detection
- Set up corner collision smoothing for movement

### Visual Feedback
- Implement debug drawing for movement constraints
- Add visualization for jump arcs and distances
- Use distinct colors for different interaction types
- Include scale reference objects for size perception

## Implementation Plan

### Step 1: Basic Level Geometry

1. Create a new level with **File > New Level > Empty Level**
2. Add a floor plane: (1000 x 1000 units) with grid material
3. Create basic testing platforms at various heights
4. Add walls for horizontal movement testing
5. Configure [Lumen global illumination](https://docs.unrealengine.com/5.3/en-US/lumen-global-illumination-and-reflections-in-unreal-engine/) with Directional Light and Sky Light:
   - Enable Lumen in Project Settings > Rendering > Global Illumination
   - Set Lumen quality to Medium for faster iteration
   - Add Sky Atmosphere for ambient lighting

### Step 2: Camera Setup

1. Create a new Blueprint extending [PlayerCameraManager](https://docs.unrealengine.com/5.3/en-US/API/Runtime/Engine/Camera/APlayerCameraManager/)
   - Class name: BP_PlatformerCameraManager
   - Location: Content/Blueprints/Camera
2. Configure camera properties:
   - Field of View: 60 degrees (or 30-40 for more orthographic feel)
   - Use perspective camera with fixed rotation
   - Set initial position offset (X: 0, Y: 300-500, Z: 100-200)
3. Add camera lag for smooth tracking
4. Create blend settings for transitions

### Step 3: Enhanced Input Setup

1. Create [Input Action](https://docs.unrealengine.com/5.3/en-US/enhanced-input-in-unreal-engine/) assets:
   - IA_Move (Vector2D value type)
   - IA_Jump (Digital value type)
   - IA_Interact (Digital value type)
2. Create an [Input Mapping Context](https://docs.unrealengine.com/5.3/en-US/enhanced-input-action-and-mapping-contexts-in-unreal-engine/) for gameplay
3. Assign input actions to keys/buttons with appropriate modifiers
4. Configure the Input Component in your character class

### Step 4: Player Setup

1. Place PlayerStart in a clear area on the ground plane
2. Set rotation to face the -Y or +Y axis (depending on your design)
3. Configure World Settings:
   - Default GameMode: BP_PlatformerGameMode or PlatformerGameMode C++ class
   - Default Pawn Class: BP_PlatformerCharacter or PlatformerCharacter C++ class
4. Add debug markers for spawn locations

### Step 5: Visual Environment

1. Create three material types:
   - Ground material: Grid with green accent
   - Platform material: Grid with blue accent
   - Wall material: Grid with red accent
2. Add a SkyAtmosphere component for background
3. Configure [Virtual Shadow Maps](https://docs.unrealengine.com/5.3/en-US/virtual-shadow-maps-in-unreal-engine/):
   - Enable in Project Settings > Engine > Rendering > Shadows
   - Set appropriate quality level for development
4. Configure basic post-processing:
   - Slight color grading for visual clarity
   - Ambient occlusion for depth
   - No motion blur for development

### Step 6: Component Architecture Setup

1. Create core character class with component structure:
   ```cpp
   // In PlatformerCharacter.h
   
   UCLASS()
   class APlatformerCharacter : public ACharacter
   {
       GENERATED_BODY()
       
   public:
       APlatformerCharacter();
   
   protected:
       // Core character components
       UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
       class UPlatformerMovementComponent* PlatformerMovement;
       
       UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
       class UPlatformerJumpComponent* JumpComponent;
       
       UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
       class UPlatformerCombatComponent* CombatComponent;
       
       UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
       class UPlatformerInteractionComponent* InteractionComponent;
       
       UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
       class UAbilitySystemComponent* AbilitySystemComponent;
       
       // Timeline for position adjustment during animations
       UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Animation")
       class UTimelineComponent* PositionAdjustmentTimeline;
   };
   ```

2. Initialize components in the constructor:
   ```cpp
   // In PlatformerCharacter.cpp
   
   APlatformerCharacter::APlatformerCharacter()
   {
       // Create movement component
       PlatformerMovement = CreateDefaultSubobject<UPlatformerMovementComponent>("PlatformerMovement");
       
       // Create jump component
       JumpComponent = CreateDefaultSubobject<UPlatformerJumpComponent>("JumpComponent");
       
       // Create combat component
       CombatComponent = CreateDefaultSubobject<UPlatformerCombatComponent>("CombatComponent");
       
       // Create interaction component
       InteractionComponent = CreateDefaultSubobject<UPlatformerInteractionComponent>("InteractionComponent");
       
       // Create ability system component
       AbilitySystemComponent = CreateDefaultSubobject<UAbilitySystemComponent>("AbilitySystemComponent");
       
       // Create position adjustment timeline component
       PositionAdjustmentTimeline = CreateDefaultSubobject<UTimelineComponent>("PositionAdjustmentTimeline");
   }
   ```

3. Create component classes with focused responsibilities:
   - UPlatformerMovementComponent: Handles ground movement and movement states
   - UPlatformerJumpComponent: Manages jump mechanics and air control
   - UPlatformerCombatComponent: Handles attacks and combat states
   - UPlatformerInteractionComponent: Manages interaction with environment

4. Set up component dependencies and communication through interfaces or delegates

### Step 7: Animation Montage Setup

1. Create a base Animation Blueprint for your character:
   - Create a new Animation Blueprint using your Skeletal Mesh
   - Set up a state machine with primary states (Idle, Run, Jump, Fall)
   - Create transitions based on movement parameters

2. Implement key Animation Montages:
   ```cpp
   // In PlatformerCharacter.h - Add montage references
   
   UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Animation")
   class UAnimMontage* JumpMontage;
   
   UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Animation")
   class UAnimMontage* LedgeGrabMontage;
   
   UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Animation")
   class UAnimMontage* LandingMontage;
   ```

3. Create position adjustment function:
   ```cpp
   // In PlatformerCharacter.h
   
   /** Start position adjustment over time */
   UFUNCTION(BlueprintCallable, Category = "Animation")
   void StartPositionAdjustment(FVector TargetPosition, float Duration = 0.2f);
   
   /** Timeline callback for position adjustment */
   UFUNCTION()
   void UpdatePositionAdjustment(float Alpha);
   
   /** Called when position adjustment completes */
   UFUNCTION()
   void FinishPositionAdjustment();
   
   private:
     // Stored positions for timeline interpolation
     FVector StartPosition;
     FVector EndPosition;
   ```

4. Set up the Timeline for smooth position adjustment:
   ```cpp
   // In PlatformerCharacter.cpp - BeginPlay
   
   void APlatformerCharacter::BeginPlay()
   {
       Super::BeginPlay();
       
       // Set up position adjustment timeline
       if (PositionAdjustmentTimeline)
       {
           FOnTimelineFloat ProgressUpdate;
           ProgressUpdate.BindUFunction(this, FName("UpdatePositionAdjustment"));
           
           FOnTimelineEvent FinishedEvent;
           FinishedEvent.BindUFunction(this, FName("FinishPositionAdjustment"));
           
           // Create curve if needed
           if (!PositionAdjustmentCurve)
           {
               PositionAdjustmentCurve = NewObject<UCurveFloat>(this, UCurveFloat::StaticClass());
               PositionAdjustmentCurve->FloatCurve.AddKey(0.0f, 0.0f);
               PositionAdjustmentCurve->FloatCurve.AddKey(1.0f, 1.0f);
           }
           
           PositionAdjustmentTimeline->AddInterpFloat(PositionAdjustmentCurve, ProgressUpdate);
           PositionAdjustmentTimeline->SetTimelineFinishedFunc(FinishedEvent);
       }
   }
   
   void APlatformerCharacter::StartPositionAdjustment(FVector TargetPosition, float Duration)
   {
       // Store start and end positions
       StartPosition = GetActorLocation();
       EndPosition = TargetPosition;
       
       // Set timeline duration and play
       PositionAdjustmentTimeline->SetPlayRate(1.0f / FMath::Max(Duration, 0.001f));
       PositionAdjustmentTimeline->PlayFromStart();
   }
   
   void APlatformerCharacter::UpdatePositionAdjustment(float Alpha)
   {
       // Interpolate position
       FVector NewPosition = FMath::Lerp(StartPosition, EndPosition, Alpha);
       SetActorLocation(NewPosition);
   }
   
   void APlatformerCharacter::FinishPositionAdjustment()
   {
       // Ensure final position is exact
       SetActorLocation(EndPosition);
   }
   ```

## Technical Deep Dive: 2.5D Movement Plane Setup

For 2.5D platformers, movement constraint is critical. Here's how to implement it in our component-based architecture:

```cpp
// In your platformer movement component:

// Extend UCharacterMovementComponent for specialized 2.5D movement
UCLASS()
class UPlatformerMovementComponent : public UCharacterMovementComponent
{
    GENERATED_BODY()
    
public:
    UPlatformerMovementComponent();
    
    // Override RequestDirectMove for 2.5D AI movement
    virtual void RequestDirectMove(const FVector& MoveVelocity, bool bForceMaxSpeed) override;
    
    // Override PhysicsVolume movement to constrain to 2.5D plane
    virtual void PhysWalking(float deltaTime, int32 Iterations) override;
    
protected:
    // 2.5D Constraint axis (which axis to zero out - typically Y for side-scrolling)
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "2.5D Movement")
    TEnumAsByte<EAxis::Type> ConstraintAxis = EAxis::Y;
};

// In PlatformerMovementComponent.cpp

// Override the RequestDirectMove method to constrain AI movement
void UPlatformerMovementComponent::RequestDirectMove(const FVector& MoveVelocity, bool bForceMaxSpeed)
{
    // Project the movement vector onto the 2D plane (assuming Y is depth)
    FVector ConstrainedMovement = MoveVelocity;
    
    if (ConstraintAxis == EAxis::X)
        ConstrainedMovement.X = 0.0f;
    else if (ConstraintAxis == EAxis::Y)
        ConstrainedMovement.Y = 0.0f;
    else if (ConstraintAxis == EAxis::Z)
        ConstrainedMovement.Z = 0.0f;
    
    // Normalize if needed
    if (!ConstrainedMovement.IsNearlyZero())
    {
        ConstrainedMovement = ConstrainedMovement.GetSafeNormal() * MoveVelocity.Size();
    }
    
    Super::RequestDirectMove(ConstrainedMovement, bForceMaxSpeed);
}
```

For Enhanced Input configuration with component handling:

```cpp
// In your player controller or pawn class:

void APlatformerCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    // Cast to Enhanced Input Component
    UEnhancedInputComponent* EnhancedInputComponent = Cast<UEnhancedInputComponent>(PlayerInputComponent);
    if (EnhancedInputComponent)
    {
        // Bind the move action - route to movement component
        EnhancedInputComponent->BindAction(MoveAction, ETriggerEvent::Triggered, this, &APlatformerCharacter::HandleMoveInput);
        
        // Bind the jump action - route to jump component
        EnhancedInputComponent->BindAction(JumpAction, ETriggerEvent::Started, this, &APlatformerCharacter::HandleJumpInput);
        EnhancedInputComponent->BindAction(JumpAction, ETriggerEvent::Completed, this, &APlatformerCharacter::HandleJumpEndInput);
        
        // Bind combat actions - route to combat component
        EnhancedInputComponent->BindAction(AttackAction, ETriggerEvent::Started, this, &APlatformerCharacter::HandleAttackInput);
    }
}

void APlatformerCharacter::HandleMoveInput(const FInputActionValue& Value)
{
    // Get input value as a 2D vector
    FVector2D MovementVector = Value.Get<FVector2D>();
    
    // Forward to movement component if valid
    if (PlatformerMovement)
    {
        PlatformerMovement->ProcessMoveInput(MovementVector);
    }
}

void APlatformerCharacter::HandleJumpInput(const FInputActionValue& Value)
{
    // Forward to jump component if valid
    if (JumpComponent)
    {
        JumpComponent->StartJump();
    }
}
```

For camera configuration in the Player Camera Manager:

```cpp
// In your camera manager class:

// Setup fixed perspective with slight angle
void APlatformerCameraManager::SetupCameraDefaults()
{
    // Set camera to fixed rotation looking at the 2D plane
    DefaultFOV = 60.0f;
    ViewPitchMin = ViewPitchMax = -10.0f; // Slight downward angle
    ViewYawMin = ViewYawMax = 90.0f;      // Fixed side view
    
    // Configure camera position
    ViewOffset = FVector(0.0f, 400.0f, 150.0f); // Offset from player
}
```

## Dependencies and Requirements

- Core character components implementation
- Custom collision setup
- Basic understanding of Unreal level design
- Understanding of camera setups in Unreal

## Next Steps After Implementation

After completing the initial scene:
1. Implement and test individual components (movement, jump, combat)
2. Test collision and movement mechanics
3. Add gameplay elements for testing (collectibles, hazards)
4. Implement camera follow behavior
5. Begin component interaction testing

## Success Criteria

The initial scene implementation is successful when:
- Scene loads without errors
- Player character spawns correctly with all components
- Components interact correctly (movement affects jumping, etc.)
- Camera is positioned for proper 2.5D visualization
- Movement is constrained to the 2D plane
- Lighting provides clear visibility of gameplay elements
- Component debug visualization shows correct state
- Performance is monitored and acceptable

## Development Timeline

Estimated implementation time: 4-6 hours depending on experience level.

Phase breakdown:
1. Basic level creation and layout (30-60 minutes)
2. Camera setup and configuration (30 minutes)
3. Enhanced Input setup (30 minutes)
4. Character component architecture implementation (60-90 minutes)
5. Character placement and GameMode setup (15 minutes)
6. Visual environment and materials (30-45 minutes)
7. Animation system setup (30-45 minutes)
8. Testing and debugging (30-60 minutes)

## Future Considerations

For future iterations of your scene, consider implementing these advanced UE5.5 features:

### Nanite Geometry
- Benefits: Allows for detailed platforms without performance hit
- When to add: When transitioning from blockout to final art
- Documentation: [Nanite Virtualized Geometry](https://docs.unrealengine.com/5.3/en-US/nanite-virtualized-geometry-in-unreal-engine/)

### World Partition
- Benefits: Better organization and streaming for larger levels
- When to add: When expanding beyond the test environment to full levels
- Documentation: [World Partition Overview](https://docs.unrealengine.com/5.3/en-US/world-partition-in-unreal-engine/) 