# UI System Implementation

## Prerequisites
- Completed steps in combat_system.md
- Working character with movement and combat
- CommonUI plugin enabled

## Step 1: Create Basic HUD Class

1. Create new C++ class:
   - Tools → New C++ Class
   - Parent Class: HUD
   - Name: "PlatformerHUD"

2. Implement header:

```cpp
// PlatformerHUD.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/HUD.h"
#include "PlatformerHUD.generated.h"

UCLASS()
class YOURPROJECT_API APlatformerHUD : public AHUD
{
    GENERATED_BODY()
    
public:
    APlatformerHUD();
    
    // Called every frame
    virtual void DrawHUD() override;
    
    // Main HUD widget
    UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category="UI")
    TSubclassOf<class UUserWidget> MainHUDWidgetClass;
    
    // Pause menu widget
    UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category="UI")
    TSubclassOf<class UUserWidget> PauseMenuWidgetClass;
    
    // Game over widget
    UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category="UI")
    TSubclassOf<class UUserWidget> GameOverWidgetClass;
    
    // Show/hide UI
    UFUNCTION(BlueprintCallable, Category="UI")
    void ShowMainHUD();
    
    UFUNCTION(BlueprintCallable, Category="UI")
    void HideMainHUD();
    
    UFUNCTION(BlueprintCallable, Category="UI")
    void TogglePauseMenu();
    
    UFUNCTION(BlueprintCallable, Category="UI")
    void ShowGameOverScreen();
    
protected:
    // Called when the game starts or when spawned
    virtual void BeginPlay() override;
    
    // Widget instances
    UPROPERTY()
    class UUserWidget* MainHUDWidget;
    
    UPROPERTY()
    class UUserWidget* PauseMenuWidget;
    
    UPROPERTY()
    class UUserWidget* GameOverWidget;
    
    // Debug overlays
    void DrawDebugInfo();
    bool bShowDebugInfo = false;
};
```

3. Implement source file:

```cpp
// PlatformerHUD.cpp
#include "PlatformerHUD.h"
#include "Blueprint/UserWidget.h"
#include "Kismet/GameplayStatics.h"
#include "PlatformerCharacter.h"
#include "Engine/Canvas.h"

APlatformerHUD::APlatformerHUD()
{
    // Default ctor
}

void APlatformerHUD::BeginPlay()
{
    Super::BeginPlay();
    
    // Create main HUD widget
    if (MainHUDWidgetClass)
    {
        MainHUDWidget = CreateWidget<UUserWidget>(GetWorld(), MainHUDWidgetClass);
        if (MainHUDWidget)
        {
            MainHUDWidget->AddToViewport();
        }
    }
    
    // Create pause menu but don't add to viewport yet
    if (PauseMenuWidgetClass)
    {
        PauseMenuWidget = CreateWidget<UUserWidget>(GetWorld(), PauseMenuWidgetClass);
    }
    
    // Create game over screen but don't add to viewport yet
    if (GameOverWidgetClass)
    {
        GameOverWidget = CreateWidget<UUserWidget>(GetWorld(), GameOverWidgetClass);
    }
}

void APlatformerHUD::DrawHUD()
{
    Super::DrawHUD();
    
    // Draw debug info if enabled
    if (bShowDebugInfo)
    {
        DrawDebugInfo();
    }
}

void APlatformerHUD::ShowMainHUD()
{
    if (MainHUDWidget && !MainHUDWidget->IsInViewport())
    {
        MainHUDWidget->AddToViewport();
    }
}

void APlatformerHUD::HideMainHUD()
{
    if (MainHUDWidget && MainHUDWidget->IsInViewport())
    {
        MainHUDWidget->RemoveFromViewport();
    }
}

void APlatformerHUD::TogglePauseMenu()
{
    if (PauseMenuWidget)
    {
        if (PauseMenuWidget->IsInViewport())
        {
            PauseMenuWidget->RemoveFromViewport();
            
            // Resume game
            UGameplayStatics::SetGamePaused(GetWorld(), false);
            
            // Show mouse cursor
            if (APlayerController* PC = GetOwningPlayerController())
            {
                PC->SetShowMouseCursor(false);
                PC->SetInputMode(FInputModeGameOnly());
            }
        }
        else
        {
            PauseMenuWidget->AddToViewport();
            
            // Pause game
            UGameplayStatics::SetGamePaused(GetWorld(), true);
            
            // Show mouse cursor
            if (APlayerController* PC = GetOwningPlayerController())
            {
                PC->SetShowMouseCursor(true);
                PC->SetInputMode(FInputModeUIOnly());
            }
        }
    }
}

void APlatformerHUD::ShowGameOverScreen()
{
    if (GameOverWidget && !GameOverWidget->IsInViewport())
    {
        // Hide other widgets
        HideMainHUD();
        if (PauseMenuWidget && PauseMenuWidget->IsInViewport())
        {
            PauseMenuWidget->RemoveFromViewport();
        }
        
        // Show game over
        GameOverWidget->AddToViewport();
        
        // Show mouse cursor
        if (APlayerController* PC = GetOwningPlayerController())
        {
            PC->SetShowMouseCursor(true);
            PC->SetInputMode(FInputModeUIOnly());
        }
    }
}

void APlatformerHUD::DrawDebugInfo()
{
    // Get the player character
    APlatformerCharacter* Character = Cast<APlatformerCharacter>(UGameplayStatics::GetPlayerCharacter(GetWorld(), 0));
    if (!Character)
    {
        return;
    }
    
    // Draw basic debug info
    FString InfoText;
    
    // Position
    FVector Location = Character->GetActorLocation();
    InfoText = FString::Printf(TEXT("Position: X=%.1f Y=%.1f Z=%.1f"), Location.X, Location.Y, Location.Z);
    DrawText(InfoText, FLinearColor::White, 50, 50, nullptr, 1.0f);
    
    // Velocity
    FVector Velocity = Character->GetVelocity();
    float Speed = Velocity.Size();
    InfoText = FString::Printf(TEXT("Speed: %.1f"), Speed);
    DrawText(InfoText, FLinearColor::White, 50, 70, nullptr, 1.0f);
    
    // Is Grounded
    bool bIsGrounded = Character->GetCharacterMovement()->IsMovingOnGround();
    InfoText = FString::Printf(TEXT("Grounded: %s"), bIsGrounded ? TEXT("Yes") : TEXT("No"));
    DrawText(InfoText, FLinearColor::White, 50, 90, nullptr, 1.0f);
}
```

**SUCCESS CHECK:** HUD class compiles with no errors

## Step 2: Create Main HUD Widget with CommonUI

1. Create Common UI Base Classes:
   - Create `PlatformerCommonButtonBase` from `CommonButtonBase`
   - Create `PlatformerCommonTextBlock` from `CommonTextBlock`
   - Create `PlatformerCommonActivatableWidget` from `CommonActivatableWidget`

2. Create Main HUD Widget:
   - In Content Browser, right-click → User Interface → Widget Blueprint
   - Name it "WBP_MainHUD"
   - Use the following structure:

```
- Root Canvas Panel
  |- Health Bar (Progress Bar)
  |- Score Counter (CommonTextBlock)
  |- Collectibles Counter (CommonTextBlock)
  |- Ability Icons (Horizontal Box)
  |- Lives Indicator (Horizontal Box)
```

3. Bind Widget to Character Data:
   - Create `UpdateHealth` function
   - Create `UpdateScore` function 
   - Create `UpdateCollectibles` function
   - Implement event binding in widget's Construct event

**SUCCESS CHECK:** HUD displays and updates correctly with player data

## Step 3: Create Pause Menu

1. Create Pause Menu Widget:
   - In Content Browser, right-click → User Interface → Widget Blueprint
   - Name it "WBP_PauseMenu"
   - Use CommonUI components

```
- Root Canvas Panel
  |- Background Overlay
  |- Title Text (PlatformerCommonTextBlock)
  |- Buttons Container (Vertical Box)
     |- Resume Button (PlatformerCommonButtonBase)
     |- Settings Button (PlatformerCommonButtonBase)
     |- Quit Button (PlatformerCommonButtonBase)
```

2. Implement Button Functionality:
   - Resume: Close pause menu
   - Settings: Open settings panel
   - Quit: Return to main menu

3. Add Pause Input to Character:
   - Update character input bindings for pause action
   - Connect to HUD's TogglePauseMenu function

**SUCCESS CHECK:** Pause menu shows/hides correctly and buttons work

## Step 4: Create Game Over Screen

1. Create Game Over Widget:
   - In Content Browser, right-click → User Interface → Widget Blueprint
   - Name it "WBP_GameOver"
   - Use the following structure:

```
- Root Canvas Panel
  |- Background Overlay
  |- "Game Over" Text (PlatformerCommonTextBlock)
  |- Score Display (PlatformerCommonTextBlock)
  |- Buttons Container (Vertical Box)
     |- Retry Button (PlatformerCommonButtonBase)
     |- Quit Button (PlatformerCommonButtonBase)
```

2. Implement Button Functionality:
   - Retry: Reload current level
   - Quit: Return to main menu

**SUCCESS CHECK:** Game over screen displays correctly after player death

## Step 5: Create UI Feedback Elements

1. Implement Damage Indicator:
   - Create screen flash effect for player damage
   - Add animation for health bar changes

2. Create Pickup Notifications:
   - Design pop-up notification for collectibles
   - Implement fade in/out animations

3. Add Score Popup:
   - Create temporary score increase display
   - Add animation for score changes

**SUCCESS CHECK:** UI provides clear feedback for game events

## Step 6: Create Collectible Counter

1. Update Character to Track Collectibles:
   - Add collectible counter variable
   - Create function to increment counter

2. Create Collectible UI Element:
   - Design counter display with icon
   - Bind to character's collectible count

3. Implement Collection Animation:
   - Add animation for counter updates
   - Create particle effect for collection

**SUCCESS CHECK:** Collectible counter updates correctly when items are collected

## Step 7: Implement Game State UI Connection

1. Create Game State Class to Track Global Info:
   - Create `PlatformerGameState` class
   - Add score, collectibles, level tracking

2. Connect HUD to Game State:
   - Update HUD to read from game state
   - Implement game state change callbacks

3. Create Level Complete UI:
   - Design level complete overlay
   - Show score, collectibles, time

**SUCCESS CHECK:** Game state information displays correctly in UI

## Step 8: Add Input Hints

1. Create Input Hint Widget:
   - Design input prompt display
   - Show context-sensitive controls

2. Implement Context Detection:
   - Show interact prompts near interactable objects
   - Display combat controls when enemies are nearby

**SUCCESS CHECK:** Input hints display correctly based on context

## Next Steps
With your UI system in place, proceed to level design in `level_design.md`. 