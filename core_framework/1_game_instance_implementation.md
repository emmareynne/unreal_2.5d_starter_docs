# Game Instance Implementation for 2.5D Platformer

This guide provides a detailed, step-by-step approach to implementing a custom GameInstance for your Platformer project in Unreal Engine 5.5. The GameInstance serves as the backbone of your game, persisting across level transitions and storing essential game data.

## What is a GameInstance?

The GameInstance is a singleton object that persists throughout your entire game session. Unlike other game framework classes (GameMode, PlayerController, etc.) which are destroyed and recreated when traveling between levels, the GameInstance remains in memory from game launch until game exit.

This makes it ideal for storing:
- Player progression data
- Global game state
- Settings
- Persistent audio
- Level transition handling
- Save/load functionality

## Prerequisites

- Unreal Engine 5.5 installed
- Platformer project created with the recommended plugins
- Basic understanding of C++ (though no prior Unreal experience is assumed)
- Visual Studio 2022 or other compatible IDE set up for Unreal Engine development

## Implementation Steps

### Step 1: Create the GameInstance C++ Class

1. Open your Platformer project in Unreal Editor
2. From the main menu, select **Tools → New C++ Class**
3. In the dialog, click **Show All Classes**
4. In the search field, type "GameInstance"
5. Select **GameInstance** from the results
6. Click **Next**
7. Name your class `PlatformerGameInstance`
8. Click **Create Class**

This will create a new C++ class that inherits from UGameInstance in your project's Source directory.

### Step 2: Set Up Project Build File

Before implementing our GameInstance, we need to make sure our project's build file includes the necessary modules. Open `Source/Platformer/Platformer.Build.cs` and update it:

```csharp
// Platformer.Build.cs
using UnrealBuildTool;

public class Platformer : ModuleRules
{
    public Platformer(ReadOnlyTargetRules Target) : base(Target)
    {
        PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;
    
        PublicDependencyModuleNames.AddRange(new string[] { 
            "Core", 
            "CoreUObject", 
            "Engine", 
            "InputCore" 
        });
        
        // We'll add these modules as we need them in later steps
        // "UMG", "Slate", "SlateCore" - For UI
        // "GameplayAbilities", "GameplayTags", "GameplayTasks" - For GAS
    }
}
```

### Step 3: Set Up Essential Header File

Open the generated header file (`PlatformerGameInstance.h`) and update it with the following code:

```cpp
// PlatformerGameInstance.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/GameInstance.h"
#include "PlatformerGameInstance.generated.h"

/**
 * Custom GameInstance for Platformer
 * Handles persistent game data and systems across level transitions
 */
UCLASS()
class PLATFORMER_API UPlatformerGameInstance : public UGameInstance
{
    GENERATED_BODY()
    
public:
    // Constructor
    UPlatformerGameInstance();
    
    // Virtual functions from UGameInstance
    virtual void Init() override;
    virtual void Shutdown() override;
    virtual void OnStart() override;
    
    // Static accessor function - makes it easy to access from anywhere
    UFUNCTION(BlueprintPure, Category="Game", meta=(WorldContext="WorldContextObject"))
    static UPlatformerGameInstance* GetInstance(const UObject* WorldContextObject);
    
    // Level management
    UFUNCTION(BlueprintCallable, Category="Game|Levels")
    void LoadLevel(const FName& LevelName);
    
    UFUNCTION(BlueprintCallable, Category="Game|Levels")
    void LoadLevelWithTransition(const FName& LevelName, float TransitionDuration = 1.0f);
    
    // Player progression data
    UPROPERTY(BlueprintReadWrite, Category="Game|Progress")
    int32 CollectedItems = 0;
    
    UPROPERTY(BlueprintReadWrite, Category="Game|Progress")
    int32 PlayerScore = 0;
    
    UPROPERTY(BlueprintReadWrite, Category="Game|Progress")
    int32 CurrentLevel = 0;
    
    // Save/Load functionality
    UFUNCTION(BlueprintCallable, Category="Game|SaveLoad")
    bool SaveGame();
    
    UFUNCTION(BlueprintCallable, Category="Game|SaveLoad")
    bool LoadGame();
    
    // Last checkpoint functionality
    UPROPERTY(BlueprintReadWrite, Category="Game|Checkpoint")
    FVector LastCheckpointLocation = FVector::ZeroVector;
    
    UFUNCTION(BlueprintCallable, Category="Game|Checkpoint")
    void SetCheckpoint(const FVector& Location);
    
    UFUNCTION(BlueprintPure, Category="Game|Checkpoint")
    FVector GetLastCheckpoint() const;
    
protected:
    // Audio management
    UPROPERTY()
    class UAudioComponent* BackgroundMusicComponent;
    
    // Loading screen management
    UPROPERTY(EditDefaultsOnly, Category="UI")
    TSubclassOf<class UUserWidget> LoadingScreenClass;
    
    UPROPERTY()
    class UUserWidget* LoadingScreenWidget;
    
    bool bIsLoadingLevel = false;
    
private:
    // Helper method to handle level loading completed
    void OnLevelLoadComplete(const float LoadTime, const FString& MapName);
};
```

### Step 4: Implement the GameInstance Class

Now open the implementation file (`PlatformerGameInstance.cpp`) and update it with the following code:

```cpp
// PlatformerGameInstance.cpp
#include "PlatformerGameInstance.h"
#include "Kismet/GameplayStatics.h"
#include "GameFramework/GameUserSettings.h"
#include "Components/AudioComponent.h"
#include "Blueprint/UserWidget.h"
#include "Engine/Engine.h"
#include "Engine/World.h"

UPlatformerGameInstance::UPlatformerGameInstance()
{
    // Initialize default values
    CollectedItems = 0;
    PlayerScore = 0;
    CurrentLevel = 0;
    LastCheckpointLocation = FVector::ZeroVector;
    bIsLoadingLevel = false;
}

void UPlatformerGameInstance::Init()
{
    Super::Init();
    
    // Register delegate for level load completion
    FCoreUObjectDelegates::PostLoadMapWithWorld.AddUObject(this, &UPlatformerGameInstance::OnLevelLoadComplete);
    
    UE_LOG(LogTemp, Log, TEXT("Platformer Game Instance Initialized"));
}

void UPlatformerGameInstance::Shutdown()
{
    // Clean up any resources
    if (BackgroundMusicComponent)
    {
        BackgroundMusicComponent->Stop();
        BackgroundMusicComponent = nullptr;
    }
    
    // Unregister delegates
    FCoreUObjectDelegates::PostLoadMapWithWorld.RemoveAll(this);
    
    UE_LOG(LogTemp, Log, TEXT("Platformer Game Instance Shutdown"));
    
    Super::Shutdown();
}

void UPlatformerGameInstance::OnStart()
{
    Super::OnStart();
    
    // Any additional initialization when the game first starts
    UE_LOG(LogTemp, Log, TEXT("Platformer Game Instance OnStart"));
}

UPlatformerGameInstance* UPlatformerGameInstance::GetInstance(const UObject* WorldContextObject)
{
    if (WorldContextObject)
    {
        UWorld* World = WorldContextObject->GetWorld();
        if (World)
        {
            return Cast<UPlatformerGameInstance>(World->GetGameInstance());
        }
    }
    
    // Fallback for when we can't access the world context
    if (GEngine)
    {
        for (const FWorldContext& Context : GEngine->GetWorldContexts())
        {
            if (Context.WorldType == EWorldType::Game || Context.WorldType == EWorldType::PIE)
            {
                UWorld* World = Context.World();
                if (World)
                {
                    return Cast<UPlatformerGameInstance>(World->GetGameInstance());
                }
            }
        }
    }
    
    return nullptr;
}

void UPlatformerGameInstance::LoadLevel(const FName& LevelName)
{
    if (bIsLoadingLevel)
    {
        UE_LOG(LogTemp, Warning, TEXT("Already loading a level, cannot load %s now"), *LevelName.ToString());
        return;
    }
    
    bIsLoadingLevel = true;
    
    UGameplayStatics::OpenLevel(this, LevelName);
}

void UPlatformerGameInstance::LoadLevelWithTransition(const FName& LevelName, float TransitionDuration)
{
    if (bIsLoadingLevel)
    {
        UE_LOG(LogTemp, Warning, TEXT("Already loading a level, cannot load %s now"), *LevelName.ToString());
        return;
    }
    
    bIsLoadingLevel = true;
    
    // Create and display loading screen widget if needed
    if (LoadingScreenClass && GetWorld())
    {
        LoadingScreenWidget = CreateWidget<UUserWidget>(GetWorld(), LoadingScreenClass);
        if (LoadingScreenWidget)
        {
            LoadingScreenWidget->AddToViewport(100); // High Z-order to be on top
        }
    }
    
    // Use a timer to create a delay before actually loading the level
    // This allows the loading screen to be displayed first
    FTimerHandle TimerHandle;
    FTimerDelegate TimerDelegate;
    TimerDelegate.BindLambda([this, LevelName]() {
        UGameplayStatics::OpenLevel(this, LevelName);
    });
    
    GetWorld()->GetTimerManager().SetTimer(
        TimerHandle, 
        TimerDelegate, 
        0.1f, // Small delay to ensure loading screen is visible
        false
    );
}

void UPlatformerGameInstance::OnLevelLoadComplete(const float LoadTime, const FString& MapName)
{
    bIsLoadingLevel = false;
    
    // Remove loading screen if it was displayed
    if (LoadingScreenWidget)
    {
        LoadingScreenWidget->RemoveFromParent();
        LoadingScreenWidget = nullptr;
    }
    
    UE_LOG(LogTemp, Log, TEXT("Level %s loaded in %f seconds"), *MapName, LoadTime);
}

bool UPlatformerGameInstance::SaveGame()
{
    UPlatformerSaveGame* SaveGameInstance = Cast<UPlatformerSaveGame>(
        UGameplayStatics::CreateSaveGameObject(UPlatformerSaveGame::StaticClass()));
    
    if (SaveGameInstance)
    {
        // Copy data to save game object
        SaveGameInstance->CollectedItems = CollectedItems;
        SaveGameInstance->PlayerScore = PlayerScore;
        SaveGameInstance->CurrentLevel = CurrentLevel;
        SaveGameInstance->LastCheckpointLocation = LastCheckpointLocation;
        
        return UGameplayStatics::SaveGameToSlot(SaveGameInstance, "SaveSlot1", 0);
    }
    
    return false;
}

bool UPlatformerGameInstance::LoadGame()
{
    UPlatformerSaveGame* LoadedGame = Cast<UPlatformerSaveGame>(
        UGameplayStatics::LoadGameFromSlot("SaveSlot1", 0));
    
    if (LoadedGame)
    {
        // Copy data from save game object
        CollectedItems = LoadedGame->CollectedItems;
        PlayerScore = LoadedGame->PlayerScore;
        CurrentLevel = LoadedGame->CurrentLevel;
        LastCheckpointLocation = LoadedGame->LastCheckpointLocation;
        
        return true;
    }
    
    return false;
}

void UPlatformerGameInstance::SetCheckpoint(const FVector& Location)
{
    LastCheckpointLocation = Location;
    UE_LOG(LogTemp, Log, TEXT("Checkpoint set at: %s"), *Location.ToString());
}

FVector UPlatformerGameInstance::GetLastCheckpoint() const
{
    return LastCheckpointLocation;
}
```

### Step 5: Set Up Project Settings

Now that you've created your custom GameInstance class, you need to tell the engine to use it.

1. In the Unreal Editor, go to **Edit → Project Settings**
2. In the left panel, navigate to **Project → Maps & Modes**
3. Find the **Game Instance Class** setting
4. Click the dropdown and select your `PlatformerGameInstance`
5. Click **Save**

### Step 6: Test Your GameInstance

Create a simple test to verify your GameInstance works correctly:

1. Create a test level with a few basic platforms
2. Add a trigger volume or button that calls `LoadLevel` to another test level
3. In the second level, add a way to display the `CollectedItems` or `PlayerScore` values
4. Use Print String nodes to verify the data persists between levels

### Step 7: Expanded Features (Incremental Implementation)

After getting the basic GameInstance working, we can implement additional features incrementally. Here's how to proceed with each feature, with clean guidance on what new includes are needed.

#### A. Adding Level Transitions with Loading Screens

First, update the build file to include UI modules:

```csharp
// In Platformer.Build.cs
PublicDependencyModuleNames.AddRange(new string[] { 
    "Core", "CoreUObject", "Engine", "InputCore",
    "UMG", "Slate", "SlateCore" // Add UI modules
});
```

Then add to the header file:

```cpp
// Add to PlatformerGameInstance.h
// Add this in public section:
UFUNCTION(BlueprintCallable, Category="Game|Levels")
void LoadLevelWithTransition(const FName& LevelName, float TransitionDuration = 1.0f);

// Add this in protected section:
UPROPERTY(EditDefaultsOnly, Category="UI")
TSubclassOf<class UUserWidget> LoadingScreenClass;

UPROPERTY()
class UUserWidget* LoadingScreenWidget;
```

In the implementation file:

```cpp
// Add include at the top of PlatformerGameInstance.cpp
#include "Blueprint/UserWidget.h"

// Then implement the method
void UPlatformerGameInstance::LoadLevelWithTransition(const FName& LevelName, float TransitionDuration)
{
    if (bIsLoadingLevel)
    {
        UE_LOG(LogTemp, Warning, TEXT("Already loading a level, cannot load %s now"), *LevelName.ToString());
        return;
    }
    
    bIsLoadingLevel = true;
    
    // Create and display loading screen widget if needed
    if (LoadingScreenClass && GetWorld())
    {
        LoadingScreenWidget = CreateWidget<UUserWidget>(GetWorld(), LoadingScreenClass);
        if (LoadingScreenWidget)
        {
            LoadingScreenWidget->AddToViewport(100); // High Z-order to be on top
        }
    }
    
    // Use a timer to create a delay before actually loading the level
    // This allows the loading screen to be displayed first
    FTimerHandle TimerHandle;
    FTimerDelegate TimerDelegate;
    TimerDelegate.BindLambda([this, LevelName]() {
        UGameplayStatics::OpenLevel(this, LevelName);
    });
    
    GetWorld()->GetTimerManager().SetTimer(
        TimerHandle, 
        TimerDelegate, 
        0.1f, // Small delay to ensure loading screen is visible
        false
    );
}

// Update OnLevelLoadComplete to handle the loading screen
void UPlatformerGameInstance::OnLevelLoadComplete(const float LoadTime, const FString& MapName)
{
    bIsLoadingLevel = false;
    
    // Remove loading screen if it was displayed
    if (LoadingScreenWidget)
    {
        LoadingScreenWidget->RemoveFromParent();
        LoadingScreenWidget = nullptr;
    }
    
    UE_LOG(LogTemp, Log, TEXT("Level %s loaded in %f seconds"), *MapName, LoadTime);
}
```

#### B. Adding Persistent Audio System

Add to the header file:

```cpp
// Add to PlatformerGameInstance.h in public section
UFUNCTION(BlueprintCallable, Category="Game|Audio")
void PlayBackgroundMusic(class USoundBase* MusicTrack, float FadeInDuration = 2.0f);

// Add to PlatformerGameInstance.h in protected section
UPROPERTY()
class UAudioComponent* BackgroundMusicComponent;
```

In the implementation file:

```cpp
// Add include at the top of PlatformerGameInstance.cpp
#include "Components/AudioComponent.h"

// In Shutdown method, add audio cleanup
void UPlatformerGameInstance::Shutdown()
{
    // Clean up any resources
    if (BackgroundMusicComponent)
    {
        BackgroundMusicComponent->Stop();
        BackgroundMusicComponent = nullptr;
    }
    
    // Unregister delegates
    FCoreUObjectDelegates::PostLoadMapWithWorld.RemoveAll(this);
    
    UE_LOG(LogTemp, Log, TEXT("Platformer Game Instance Shutdown"));
    
    Super::Shutdown();
}

// Implement the audio method
void UPlatformerGameInstance::PlayBackgroundMusic(USoundBase* MusicTrack, float FadeInDuration)
{
    if (!MusicTrack)
    {
        UE_LOG(LogTemp, Warning, TEXT("No music track provided"));
        return;
    }
    
    // Create audio component if needed
    if (!BackgroundMusicComponent)
    {
        BackgroundMusicComponent = UGameplayStatics::CreateSound2D(this, MusicTrack);
        BackgroundMusicComponent->bAutoDestroy = false;
    }
    else
    {
        // If music is already playing, fade it out
        if (BackgroundMusicComponent->IsPlaying())
        {
            BackgroundMusicComponent->FadeOut(FadeInDuration, 0.0f);
        }
        
        // Set new sound and fade in
        BackgroundMusicComponent->SetSound(MusicTrack);
    }
    
    BackgroundMusicComponent->FadeIn(FadeInDuration);
}
```

#### C. Adding Save/Load System

Add to the header file:

```cpp
// Add to PlatformerGameInstance.h in public section
UFUNCTION(BlueprintCallable, Category="Game|SaveLoad")
bool SaveGame();

UFUNCTION(BlueprintCallable, Category="Game|SaveLoad")
bool LoadGame();
```

First, create a separate SaveGame class by creating a new C++ class that inherits from USaveGame:

1. In the Unreal Editor, go to File → New C++ Class
2. Click "Show All Classes" 
3. Search for "SaveGame"
4. Select "SaveGame" from the list
5. Click Next
6. Name your class "PlatformerSaveGame" 
7. Click Create Class

Add the following code to the generated PlatformerSaveGame.h file:

```cpp
// PlatformerSaveGame.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/SaveGame.h"
#include "PlatformerSaveGame.generated.h"

UCLASS()
class PLATFORMER_API UPlatformerSaveGame : public USaveGame
{
    GENERATED_BODY()
    
public:
    UPROPERTY(SaveGame)
    int32 CollectedItems;
    
    UPROPERTY(SaveGame)
    int32 PlayerScore;
    
    UPROPERTY(SaveGame)
    int32 CurrentLevel;
    
    UPROPERTY(SaveGame)
    FVector LastCheckpointLocation;
};
```

Then implement the methods:

```cpp
// Include the save game header at the top of PlatformerGameInstance.cpp
#include "PlatformerSaveGame.h"

// IMPORTANT: You need to delete the placeholder implementations of SaveGame() and LoadGame()
// that look like this:
/*
bool UPlatformerGameInstance::SaveGame()
{
    // In a real implementation, you would:
    // 1. Create a USaveGame derived object
    // 2. Fill it with data from the GameInstance
    // 3. Use UGameplayStatics::SaveGameToSlot to save it
    
    UE_LOG(LogTemp, Log, TEXT("Game saved"));
    return true;
}

bool UPlatformerGameInstance::LoadGame()
{
    // In a real implementation, you would:
    // 1. Use UGameplayStatics::LoadGameFromSlot to load the save game
    // 2. Check if it was loaded successfully
    // 3. Extract the data into the GameInstance
    
    UE_LOG(LogTemp, Log, TEXT("Game loaded"));
    return true;
}
*/

// Replace them with these actual implementations:
bool UPlatformerGameInstance::SaveGame()
{
    UPlatformerSaveGame* SaveGameInstance = Cast<UPlatformerSaveGame>(
        UGameplayStatics::CreateSaveGameObject(UPlatformerSaveGame::StaticClass()));
    
    if (SaveGameInstance)
    {
        // Copy data to save game object
        SaveGameInstance->CollectedItems = CollectedItems;
        SaveGameInstance->PlayerScore = PlayerScore;
        SaveGameInstance->CurrentLevel = CurrentLevel;
        SaveGameInstance->LastCheckpointLocation = LastCheckpointLocation;
        
        return UGameplayStatics::SaveGameToSlot(SaveGameInstance, "SaveSlot1", 0);
    }
    
    return false;
}

bool UPlatformerGameInstance::LoadGame()
{
    UPlatformerSaveGame* LoadedGame = Cast<UPlatformerSaveGame>(
        UGameplayStatics::LoadGameFromSlot("SaveSlot1", 0));
    
    if (LoadedGame)
    {
        // Copy data from save game object
        CollectedItems = LoadedGame->CollectedItems;
        PlayerScore = LoadedGame->PlayerScore;
        CurrentLevel = LoadedGame->CurrentLevel;
        LastCheckpointLocation = LoadedGame->LastCheckpointLocation;
        
        return true;
    }
    
    return false;
}
```

#### D. Adding Game Settings System

Add to the header file:

```cpp
// Add to PlatformerGameInstance.h in public section
UFUNCTION(BlueprintCallable, Category="Game|Settings")
void ApplyGraphicsSettings(int32 ResolutionQuality, int32 ViewDistanceQuality, 
                          int32 AntiAliasingQuality, bool bFullscreen);
```

In the implementation file:

```cpp
// Add include at the top of PlatformerGameInstance.cpp
#include "GameFramework/GameUserSettings.h"

// Implement the settings method
void UPlatformerGameInstance::ApplyGraphicsSettings(int32 ResolutionQuality, int32 ViewDistanceQuality, 
                                                   int32 AntiAliasingQuality, bool bFullscreen)
{
    UGameUserSettings* Settings = UGameUserSettings::GetGameUserSettings();
    if (Settings)
    {
        Settings->SetScreenResolution(FIntPoint(1920, 1080)); // Set appropriate resolution
        Settings->SetFullscreenMode(bFullscreen ? EWindowMode::Fullscreen : EWindowMode::Windowed);
        
        Settings->ScalabilityQuality.ResolutionQuality = ResolutionQuality;
        Settings->ScalabilityQuality.ViewDistanceQuality = ViewDistanceQuality;
        Settings->ScalabilityQuality.AntiAliasingQuality = AntiAliasingQuality;
        
        Settings->ApplySettings(false);
    }
}
```

## Best Practices

1. **Incremental Implementation**: Add features one at a time, only including the headers you need for each feature.

2. **Minimal Includes**: Only include headers that are necessary for the specific functionality you're implementing.

3. **Forward Declarations**: Use class forward declarations in headers where possible to avoid including full headers.

4. **Module Dependencies**: Only add module dependencies to your build file when you actually need them.

5. **Commenting and Documentation**: Clearly document what each method does and why each include is needed.

6. **Memory Management**: Be mindful of cleanup in the Shutdown method for any resources created during gameplay.

7. **Blueprint Exposure**: Make sure everything that might be needed from Blueprints is properly exposed with UFUNCTION/UPROPERTY.

8. **Error Handling**: Always include proper error checking for robust code.

## Success Criteria

Your GameInstance implementation is successful when:

- ✅ Data correctly persists when transitioning between levels
- ✅ The GameInstance can be easily accessed from any part of your game code
- ✅ Level loading works smoothly with proper transitions
- ✅ Save/load functionality correctly preserves game state
- ✅ The implementation is extensible for future needs

## Next Steps

After completing your GameInstance implementation, move on to:

1. Implementing your GameMode and GameState classes
2. Creating debugging tools using the Subsystem approach
3. Setting up your character movement system with 2.5D constraints

## Troubleshooting

**Problem**: GameInstance not persisting between level changes
**Solution**: Verify you've set the correct GameInstance class in Project Settings

**Problem**: Cannot access GameInstance from Blueprints
**Solution**: Ensure all needed functions are marked with UFUNCTION(BlueprintCallable) or UFUNCTION(BlueprintPure)

**Problem**: Game crashes when trying to access GameInstance
**Solution**: Always check for null pointers when using GetInstance()

**Problem**: Changes to GameInstance not compiled
**Solution**: Ensure you've saved all files and rebuilt the project
