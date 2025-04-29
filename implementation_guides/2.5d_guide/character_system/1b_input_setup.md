# Enhanced Input System Setup

This guide covers setting up the input system for your 2.5D platformer in Unreal Engine 5.5, using the Enhanced Input System to create responsive, configurable controls for your character.

## Overview

The Enhanced Input System provides a flexible framework for handling player input, offering significant advantages over the legacy input system:
- Context-sensitive input mapping
- Complex input processing
- Runtime remapping support
- Input visualization and debugging

This guide will walk you through implementing the core input structure needed for a 2.5D platformer.

## Prerequisites

Before proceeding, ensure you have:
- Enhanced Input Plugin enabled in your project
- Basic character implementation started
- Project settings configured for Enhanced Input

## Implementation Steps

### Step 1: Project Configuration

First, configure your project to use Enhanced Input:

1. Open Project Settings (Edit > Project Settings)
2. Navigate to Engine > Input
3. Set "Default Input Component Class" to `EnhancedInputComponent`
4. Set "Default Player Input Class" to `EnhancedPlayerInput`
5. Under Plugins > Enhanced Input, ensure the plugin is enabled

### Step 2: Create Input Mapping Context

The Input Mapping Context defines which inputs are available in different game states:

1. Right-click in the Content Browser, select "Input > Input Mapping Context"
2. Name it "IMC_PlatformerDefault"
3. Open the new asset

### Step 3: Create Input Actions

Input Actions define the player actions that can be triggered:

1. Right-click in the Content Browser, select "Input > Input Action"
2. Create the following Input Actions:
   - `IA_Move` (Value Type: Vector2D)
   - `IA_Jump` (Value Type: Digital)
   - `IA_Interact` (Value Type: Digital)
   - `IA_Attack` (Value Type: Digital)
   - `IA_Pause` (Value Type: Digital)

3. For each action, configure appropriate properties:
   ```
   // Example configuration for Jump
   Trigger Behavior: Triggered
   Value Type: Digital (Boolean)
   
   // Example configuration for Move
   Trigger Behavior: Triggered
   Value Type: Vector2D
   ```

### Step 4: Configure Input Mappings

Now map physical input to your Input Actions:

1. Open the "IMC_PlatformerDefault" asset
2. Add mappings for each Input Action:

   a. For **IA_Move**:
      - Add "WASD" keyboard mapping:
        - W: Vector2D(0,1)
        - S: Vector2D(0,-1)
        - A: Vector2D(-1,0)
        - D: Vector2D(1,0)
      - Add left gamepad stick mapping
      
   b. For **IA_Jump**:
      - Add Space key mapping
      - Add gamepad face button (A/Cross) mapping
      
   c. For **IA_Interact**:
      - Add E key mapping
      - Add gamepad face button (B/Circle) mapping
      
   d. For **IA_Attack**:
      - Add Left Mouse Button mapping
      - Add gamepad right trigger mapping
      
   e. For **IA_Pause**:
      - Add Escape key mapping
      - Add gamepad start button mapping

### Step 5: Implement Input Component Setup

Update your character class to use the Enhanced Input system:

```cpp
// In PlatformerCharacter.h
protected:
    // Enhanced Input specific
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Input")
    class UInputMappingContext* DefaultMappingContext;
    
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Input")
    class UInputAction* MoveAction;
    
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Input")
    class UInputAction* JumpAction;
    
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Input")
    class UInputAction* InteractAction;
    
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Input")
    class UInputAction* AttackAction;
    
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Input")
    class UInputAction* PauseAction;
    
    // Input callback functions
    void HandleMoveInput(const struct FInputActionValue& Value);
    void HandleJumpInput(const struct FInputActionValue& Value);
    void HandleJumpReleased(const struct FInputActionValue& Value);
    void HandleInteractInput(const struct FInputActionValue& Value);
    void HandleAttackInput(const struct FInputActionValue& Value);
    void HandlePauseInput(const struct FInputActionValue& Value);
    
    // Input setup
    virtual void SetupPlayerInputComponent(class UInputComponent* PlayerInputComponent) override;
    
    // When player possession occurs
    virtual void PossessedBy(AController* NewController) override;
```

```cpp
// In PlatformerCharacter.cpp
#include "EnhancedInputComponent.h"
#include "EnhancedInputSubsystems.h"
#include "InputActionValue.h"

void APlatformerCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    Super::SetupPlayerInputComponent(PlayerInputComponent);
    
    // Get the local player enhanced input subsystem
    APlayerController* PC = Cast<APlayerController>(GetController());
    if (PC)
    {
        UEnhancedInputLocalPlayerSubsystem* Subsystem = ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(PC->GetLocalPlayer());
        if (Subsystem)
        {
            // Clear existing mappings and add our mapping context
            Subsystem->ClearMappingContext(DefaultMappingContext);
            Subsystem->AddMappingContext(DefaultMappingContext, 0);
        }
    }
    
    // Set up action bindings
    UEnhancedInputComponent* EnhancedInputComponent = Cast<UEnhancedInputComponent>(PlayerInputComponent);
    if (EnhancedInputComponent)
    {
        // Moving
        EnhancedInputComponent->BindAction(MoveAction, ETriggerEvent::Triggered, this, &APlatformerCharacter::HandleMoveInput);
        
        // Jumping
        EnhancedInputComponent->BindAction(JumpAction, ETriggerEvent::Started, this, &APlatformerCharacter::HandleJumpInput);
        EnhancedInputComponent->BindAction(JumpAction, ETriggerEvent::Completed, this, &APlatformerCharacter::HandleJumpReleased);
        
        // Interacting
        EnhancedInputComponent->BindAction(InteractAction, ETriggerEvent::Started, this, &APlatformerCharacter::HandleInteractInput);
        
        // Attacking
        EnhancedInputComponent->BindAction(AttackAction, ETriggerEvent::Started, this, &APlatformerCharacter::HandleAttackInput);
        
        // Pausing
        EnhancedInputComponent->BindAction(PauseAction, ETriggerEvent::Started, this, &APlatformerCharacter::HandlePauseInput);
    }
}

void APlatformerCharacter::PossessedBy(AController* NewController)
{
    Super::PossessedBy(NewController);
    
    // Set up input context when possessed
    APlayerController* PC = Cast<APlayerController>(NewController);
    if (PC)
    {
        UEnhancedInputLocalPlayerSubsystem* Subsystem = ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(PC->GetLocalPlayer());
        if (Subsystem)
        {
            Subsystem->ClearMappingContext(DefaultMappingContext);
            Subsystem->AddMappingContext(DefaultMappingContext, 0);
        }
    }
}

void APlatformerCharacter::HandleMoveInput(const FInputActionValue& Value)
{
    // Input is a Vector2D
    FVector2D MovementVector = Value.Get<FVector2D>();
    
    if (Controller != nullptr)
    {
        // Find out which way is forward
        const FRotator Rotation = Controller->GetControlRotation();
        const FRotator YawRotation(0, Rotation.Yaw, 0);
        
        // Get forward and right vectors
        const FVector ForwardDirection = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::X);
        const FVector RightDirection = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::Y);
        
        // For 2.5D, we typically use just one axis for movement
        // For side-scrolling, use X axis (forward/backward)
        // For depth-based, use Y axis (left/right)
        
        // Side-scrolling example (use X-axis for movement):
        AddMovementInput(ForwardDirection, MovementVector.X);
        
        // Uncomment for depth movement if needed:
        // AddMovementInput(RightDirection, MovementVector.Y);
    }
}

void APlatformerCharacter::HandleJumpInput(const FInputActionValue& Value)
{
    // Example implementation for jump component
    if (JumpComponent)
    {
        JumpComponent->StartJump();
    }
    else
    {
        // Fallback to built-in jump
        Jump();
    }
}

void APlatformerCharacter::HandleJumpReleased(const FInputActionValue& Value)
{
    // Example implementation for jump component
    if (JumpComponent)
    {
        JumpComponent->StopJump();
    }
    else
    {
        // Fallback to built-in jump
        StopJumping();
    }
}

void APlatformerCharacter::HandleInteractInput(const FInputActionValue& Value)
{
    // Example implementation for interaction component
    if (InteractionComponent)
    {
        InteractionComponent->TryInteract();
    }
}

void APlatformerCharacter::HandleAttackInput(const FInputActionValue& Value)
{
    // Example implementation for combat component
    if (CombatComponent)
    {
        CombatComponent->StartAttack();
    }
}

void APlatformerCharacter::HandlePauseInput(const FInputActionValue& Value)
{
    // Example pause implementation
    APlayerController* PC = Cast<APlayerController>(GetController());
    if (PC)
    {
        // Toggle pause state
        if (PC->IsPaused())
        {
            PC->SetPause(false);
        }
        else
        {
            PC->SetPause(true);
        }
        
        // Notify game mode or HUD about pause state change
        // This could show/hide pause menu
    }
}
```

### Step 6: Configure Multiple Input Contexts

For different gameplay states (e.g., normal gameplay, menu navigation), create additional Input Mapping Contexts:

1. Create "IMC_PlatformerMenu" for menu navigation
2. Create "IMC_PlatformerDialog" for dialog interactions
3. Implement context switching in your code:

```cpp
// Add function to switch contexts
void APlatformerCharacter::SetInputContext(UInputMappingContext* NewContext, int32 Priority = 0)
{
    APlayerController* PC = Cast<APlayerController>(GetController());
    if (PC)
    {
        UEnhancedInputLocalPlayerSubsystem* Subsystem = ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(PC->GetLocalPlayer());
        if (Subsystem)
        {
            // Clear existing mapping context if needed
            if (CurrentContext)
            {
                Subsystem->RemoveMappingContext(CurrentContext);
            }
            
            // Add new mapping context
            Subsystem->AddMappingContext(NewContext, Priority);
            CurrentContext = NewContext;
        }
    }
}
```

### Step 7: Input Modifiers and Triggers

Use Input Modifiers to refine how input is processed:

1. Open one of your Input Actions (e.g., IA_Move)
2. Add modifiers like:
   - Dead Zone: Filter out small stick movements
   - Smoothing: Smooth rapid input changes
   - Scale: Adjust input sensitivity

For example, to add a dead zone to analog movement:
1. Select the IA_Move Input Action
2. Click "Add Modifier" in the Modifiers section
3. Select "Dead Zone"
4. Configure values (e.g., 0.1 Lower Threshold, 0.9 Upper Threshold)

## 2.5D Specific Input Considerations

For 2.5D platformers, consider these specific input configurations:

1. **Movement Constraint**
   - Use a swizzle modifier on IA_Move to zero out the Y (or X) component
   - Example: `SwizzleInputAxisValues(X=0, Y=1, Z=0)` to only use Y axis

2. **Jump Buffering**
   - Implement a small buffer window for jump inputs
   - This helps with player-perceived responsiveness

```cpp
// Example jump buffering implementation
void APlatformerJumpComponent::StartJump()
{
    if (CanJump())
    {
        // Jump immediately
        PerformJump();
    }
    else
    {
        // Buffer jump input for a short time
        bJumpBuffered = true;
        GetWorld()->GetTimerManager().SetTimer(JumpBufferTimerHandle, this, &APlatformerJumpComponent::ClearJumpBuffer, JumpBufferTime, false);
    }
}

void APlatformerJumpComponent::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
{
    Super::TickComponent(DeltaTime, TickType, ThisTickFunction);
    
    // Check if we have a buffered jump and can now jump
    if (bJumpBuffered && CanJump())
    {
        PerformJump();
        ClearJumpBuffer();
    }
}

void APlatformerJumpComponent::ClearJumpBuffer()
{
    bJumpBuffered = false;
}
```

3. **Coyote Time**
   - Allow jumping shortly after walking off platforms
   - Improves feel and forgiveness

```cpp
// Example coyote time implementation
void APlatformerJumpComponent::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
{
    Super::TickComponent(DeltaTime, TickType, ThisTickFunction);
    
    // Check if character just left the ground
    bool bIsGroundedNow = IsGrounded();
    
    if (bWasGrounded && !bIsGroundedNow)
    {
        // Just left the ground, start coyote time
        bInCoyoteTime = true;
        GetWorld()->GetTimerManager().SetTimer(CoyoteTimeHandle, this, &APlatformerJumpComponent::EndCoyoteTime, CoyoteTimeDuration, false);
    }
    
    bWasGrounded = bIsGroundedNow;
}

bool APlatformerJumpComponent::CanJump() const
{
    // Can jump if grounded or in coyote time
    return IsGrounded() || bInCoyoteTime;
}

void APlatformerJumpComponent::EndCoyoteTime()
{
    bInCoyoteTime = false;
}
```

## Testing Input Setup

Test your input configuration with these steps:

1. Create a simple test level (or use your existing test environment)
2. Place your character with the Enhanced Input setup
3. Verify basic movement, jumping, and interaction inputs
4. Test edge cases like:
   - Rapid input alternation
   - Input during different animation states
   - Context switching between gameplay and menus
5. Try different input devices (keyboard, gamepad)
6. Test input visualization if implemented

## Best Practices

1. **Input Encapsulation**: Route inputs to specific components rather than handling everything in the character class
2. **Consistent Naming**: Use clear, consistent naming for input actions (IA_ActionName)
3. **Input Priorities**: Use context priorities to handle overlapping input contexts
4. **Controller Support**: Always test with both keyboard/mouse and gamepad
5. **Input Feedback**: Provide visual/audio feedback for input actions
6. **Debug Visualization**: Implement input visualization for debugging

## Common Issues and Solutions

| Issue | Solution |
|-------|----------|
| Input not responding | Verify mapping context is added to the subsystem |
| Wrong axis movement | Check axis mappings and modifiers in the Input Action |
| Inconsistent input | Add smoothing modifiers to relevant inputs |
| Dead zones too large/small | Adjust dead zone values in modifiers |
| Input feels delayed | Check for input processing in Tick vs. direct handling |

## Next Steps

After setting up the basic input system:

1. Implement input visualization for debugging
2. Set up menu navigation input contexts
3. Create advanced input combinations for special moves
4. Implement input rebinding interface for player customization
5. Add accessibility options for input sensitivity and alternative mappings 