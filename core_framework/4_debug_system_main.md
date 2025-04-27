# Debug System: Main Setup & Component Implementation

This guide covers the core setup for the debug system in your 2.5D Platformer project using Unreal Engine 5.5's official ImGui plugin.

## Setting Up the ImGui Plugin

Unreal Engine 5.5 includes an official ImGui plugin that's properly integrated with the engine.

### Step 1: Enable the ImGui Plugin

1. Open your project in Unreal Editor
2. From the main menu, select **Edit → Plugins**
3. In the search bar, type "ImGui"
4. Find "ImGui" under the "Development Tools" category
5. Enable the plugin and restart the editor

![ImGui Plugin](https://docs.unrealengine.com/5.5/Images/developing-for-unreal/developer-tools/imgui/imgui-plugin.png)

### Step 2: Update Project Build Settings

1. Open your `[YourProjectName].Build.cs` file in your code editor
2. Add "ImGui" to your PublicDependencyModuleNames:

```csharp
PublicDependencyModuleNames.AddRange(new string[] { 
    "Core", 
    "CoreUObject", 
    "Engine", 
    "InputCore",
    "ImGui" 
});
```

## Creating the Debug Component

We'll create a simple Actor Component that can be attached to your GameMode:

### Step 1: Create the Component Class

Create a new C++ class:
1. From the main menu, select **Tools → New C++ Class**
2. Select **Actor Component** as the parent class
3. Name it `PlatformerDebugComponent`

Here's the header file (`PlatformerDebugComponent.h`):

```cpp
// PlatformerDebugComponent.h
#pragma once

#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "ImGuiModule.h"
#include "PlatformerDebugComponent.generated.h"

// Forward declarations
class APlayerController;
class ACharacter;
class UCharacterMovementComponent;

UENUM(BlueprintType)
enum class EDebugCategory : uint8
{
    Performance     UMETA(DisplayName = "Performance"),
    Character       UMETA(DisplayName = "Character"),
    Collision       UMETA(DisplayName = "Collision"),
    Combat          UMETA(DisplayName = "Combat"),
    AI              UMETA(DisplayName = "AI"),
    Camera          UMETA(DisplayName = "Camera")
};

USTRUCT(BlueprintType)
struct FPositionBookmark
{
    GENERATED_BODY()
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString Name;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FVector Location;
    
    FPositionBookmark() : Name(""), Location(FVector::ZeroVector) {}
    FPositionBookmark(const FString& InName, const FVector& InLocation) : Name(InName), Location(InLocation) {}
};

UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class PLATFORMER_API UPlatformerDebugComponent : public UActorComponent
{
    GENERATED_BODY()

public:    
    UPlatformerDebugComponent();
    
    // Public accessors
    UFUNCTION(BlueprintPure, Category = "Debug")
    bool IsPlayerInvincible() const { return bPlayerInvincible; }
    
    UFUNCTION(BlueprintPure, Category = "Debug")
    bool IsDebugCameraEnabled() const { return bDebugCameraEnabled; }
    
    UFUNCTION(BlueprintPure, Category = "Debug")
    bool IsCategoryEnabled(EDebugCategory Category) const;
    
    UFUNCTION(BlueprintPure, Category = "Debug")
    bool ShouldShowMovementPath() const { return bShowMovementPath; }
    
    // Teleport helpers
    UFUNCTION(BlueprintCallable, Category = "Debug")
    void TeleportPlayerTo(const FVector& Location);
    
    UFUNCTION(BlueprintCallable, Category = "Debug")
    void SaveCurrentPositionAsBookmark(const FString& BookmarkName);
    
    UFUNCTION(BlueprintCallable, Category = "Debug")
    TArray<FPositionBookmark> GetBookmarks() const { return PositionBookmarks; }

protected:
    virtual void BeginPlay() override;
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;
    
    // ImGui draw delegate and function
    void OnImGuiDraw();
    FImGuiDelegateHandle ImGuiDelegateHandle;
    
    // Debug state 
    bool bShowDebugWindow = false;
    bool bPlayerInvincible = false;
    bool bDebugCameraEnabled = false;
    bool bShowMovementPath = false;
    
    // Category enables
    TSet<EDebugCategory> EnabledCategories;
    
    // Position bookmarks for quick teleporting
    TArray<FPositionBookmark> PositionBookmarks;
    
    // Helper functions
    ACharacter* GetPlayerCharacter() const;
    APlayerController* GetPlayerController() const;
    UCharacterMovementComponent* GetCharacterMovement() const;
    
    // Main ImGui window draw functions
    void DrawPerformanceSection();
    void DrawCharacterSection();
    void DrawVisualizationSection();
    void DrawBookmarksSection();
    void DrawDebugCameraSection();
    
    // Console command handles
    TArray<IConsoleCommand*> RegisteredConsoleCommands;
    void RegisterConsoleCommands();
    void UnregisterConsoleCommands();
    
    // Console command callbacks
    static void ToggleDebugWindowCommand(const TArray<FString>& Args, UWorld* World);
    static void TeleportPlayerCommand(const TArray<FString>& Args, UWorld* World);
    static void ToggleCategoryCommand(const TArray<FString>& Args, UWorld* World);
};
```

### Step 2: Implement Core Component Functionality

Here's the core implementation file (`PlatformerDebugComponent.cpp`):

```cpp
// PlatformerDebugComponent.cpp
#include "PlatformerDebugComponent.h"
#include "Engine/Engine.h"
#include "Engine/World.h"
#include "GameFramework/Character.h"
#include "GameFramework/CharacterMovementComponent.h"
#include "GameFramework/PlayerController.h"
#include "DrawDebugHelpers.h"
#include "HAL/IConsoleManager.h"
#include "ImGuiModule.h"
#include "ImGuiDelegates.h"

UPlatformerDebugComponent::UPlatformerDebugComponent()
{
    PrimaryComponentTick.bCanEverTick = false;
}

void UPlatformerDebugComponent::BeginPlay()
{
    Super::BeginPlay();
    
    // Register ImGui callback
    ImGuiDelegateHandle = FImGuiModule::Get().OnImGuiCreated().AddUObject(this, &UPlatformerDebugComponent::OnImGuiDraw);
    
    // Register console commands
    RegisterConsoleCommands();
    
    UE_LOG(LogTemp, Log, TEXT("PlatformerDebugComponent initialized"));
}

void UPlatformerDebugComponent::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    // Unregister imgui delegate
    FImGuiModule::Get().OnImGuiCreated().Remove(ImGuiDelegateHandle);
    
    // Unregister console commands
    UnregisterConsoleCommands();
    
    Super::EndPlay(EndPlayReason);
}

bool UPlatformerDebugComponent::IsCategoryEnabled(EDebugCategory Category) const
{
    return EnabledCategories.Contains(Category);
}

void UPlatformerDebugComponent::TeleportPlayerTo(const FVector& Location)
{
    ACharacter* PlayerCharacter = GetPlayerCharacter();
    if (PlayerCharacter)
    {
        PlayerCharacter->TeleportTo(Location, PlayerCharacter->GetActorRotation());
        UE_LOG(LogTemp, Display, TEXT("Teleported player to %s"), *Location.ToString());
    }
}

void UPlatformerDebugComponent::SaveCurrentPositionAsBookmark(const FString& BookmarkName)
{
    ACharacter* PlayerCharacter = GetPlayerCharacter();
    if (PlayerCharacter)
    {
        FVector Location = PlayerCharacter->GetActorLocation();
        
        // Check if a bookmark with this name already exists
        for (int32 i = 0; i < PositionBookmarks.Num(); ++i)
        {
            if (PositionBookmarks[i].Name == BookmarkName)
            {
                PositionBookmarks[i].Location = Location;
                return;
            }
        }
        
        // Add new bookmark
        PositionBookmarks.Add(FPositionBookmark(BookmarkName, Location));
    }
}

ACharacter* UPlatformerDebugComponent::GetPlayerCharacter() const
{
    if (UWorld* World = GetWorld())
    {
        APlayerController* PC = World->GetFirstPlayerController();
        if (PC)
        {
            return Cast<ACharacter>(PC->GetPawn());
        }
    }
    return nullptr;
}

APlayerController* UPlatformerDebugComponent::GetPlayerController() const
{
    if (UWorld* World = GetWorld())
    {
        return World->GetFirstPlayerController();
    }
    return nullptr;
}

UCharacterMovementComponent* UPlatformerDebugComponent::GetCharacterMovement() const
{
    ACharacter* Character = GetPlayerCharacter();
    if (Character)
    {
        return Character->GetCharacterMovement();
    }
    return nullptr;
}

void UPlatformerDebugComponent::RegisterConsoleCommands()
{
    // Register toggle debug window command
    RegisteredConsoleCommands.Add(IConsoleManager::Get().RegisterConsoleCommand(
        TEXT("DebugWindow.Toggle"),
        TEXT("Toggle the ImGui debug window"),
        FConsoleCommandWithWorldArgsAndDebugDelegate::CreateStatic(&UPlatformerDebugComponent::ToggleDebugWindowCommand),
        ECVF_Default
    ));
    
    // Register teleport command
    RegisteredConsoleCommands.Add(IConsoleManager::Get().RegisterConsoleCommand(
        TEXT("Debug.TeleportPlayer"),
        TEXT("Teleport player to X Y Z coordinates: Debug.TeleportPlayer X Y Z"),
        FConsoleCommandWithWorldArgsAndDebugDelegate::CreateStatic(&UPlatformerDebugComponent::TeleportPlayerCommand),
        ECVF_Default
    ));
    
    // Register category toggle command
    RegisteredConsoleCommands.Add(IConsoleManager::Get().RegisterConsoleCommand(
        TEXT("Debug.ToggleCategory"),
        TEXT("Toggle a debug category: Debug.ToggleCategory CategoryName"),
        FConsoleCommandWithWorldArgsAndDebugDelegate::CreateStatic(&UPlatformerDebugComponent::ToggleCategoryCommand),
        ECVF_Default
    ));
}

void UPlatformerDebugComponent::UnregisterConsoleCommands()
{
    for (IConsoleCommand* Command : RegisteredConsoleCommands)
    {
        IConsoleManager::Get().UnregisterConsoleObject(Command);
    }
    RegisteredConsoleCommands.Empty();
}

void UPlatformerDebugComponent::ToggleDebugWindowCommand(const TArray<FString>& Args, UWorld* World)
{
    if (!World)
    {
        return;
    }
    
    // Find our component
    for (TObjectIterator<UPlatformerDebugComponent> It; It; ++It)
    {
        UPlatformerDebugComponent* DebugComponent = *It;
        if (DebugComponent && DebugComponent->GetWorld() == World)
        {
            DebugComponent->bShowDebugWindow = !DebugComponent->bShowDebugWindow;
            UE_LOG(LogTemp, Display, TEXT("Debug window %s"), 
                DebugComponent->bShowDebugWindow ? TEXT("shown") : TEXT("hidden"));
            return;
        }
    }
}

void UPlatformerDebugComponent::TeleportPlayerCommand(const TArray<FString>& Args, UWorld* World)
{
    if (!World || Args.Num() < 3)
    {
        UE_LOG(LogTemp, Warning, TEXT("Usage: Debug.TeleportPlayer X Y Z"));
        return;
    }
    
    // Find our component
    for (TObjectIterator<UPlatformerDebugComponent> It; It; ++It)
    {
        UPlatformerDebugComponent* DebugComponent = *It;
        if (DebugComponent && DebugComponent->GetWorld() == World)
        {
            float X = FCString::Atof(*Args[0]);
            float Y = FCString::Atof(*Args[1]);
            float Z = FCString::Atof(*Args[2]);
            
            DebugComponent->TeleportPlayerTo(FVector(X, Y, Z));
            return;
        }
    }
}

void UPlatformerDebugComponent::ToggleCategoryCommand(const TArray<FString>& Args, UWorld* World)
{
    if (!World || Args.Num() < 1)
    {
        UE_LOG(LogTemp, Warning, TEXT("Usage: Debug.ToggleCategory CategoryName"));
        return;
    }
    
    // Find our component
    for (TObjectIterator<UPlatformerDebugComponent> It; It; ++It)
    {
        UPlatformerDebugComponent* DebugComponent = *It;
        if (DebugComponent && DebugComponent->GetWorld() == World)
        {
            // Convert string to category enum
            const FString& CategoryName = Args[0];
            const UEnum* EnumPtr = FindObject<UEnum>(ANY_PACKAGE, TEXT("EDebugCategory"), true);
            if (EnumPtr)
            {
                for (int32 i = 0; i < EnumPtr->GetMaxEnumValue(); ++i)
                {
                    const FString EnumName = EnumPtr->GetNameStringByIndex(i);
                    if (EnumName.EndsWith(CategoryName, ESearchCase::IgnoreCase))
                    {
                        EDebugCategory Category = static_cast<EDebugCategory>(i);
                        
                        // Toggle category
                        if (DebugComponent->EnabledCategories.Contains(Category))
                        {
                            DebugComponent->EnabledCategories.Remove(Category);
                            UE_LOG(LogTemp, Display, TEXT("Debug category %s disabled"), *CategoryName);
                        }
                        else
                        {
                            DebugComponent->EnabledCategories.Add(Category);
                            UE_LOG(LogTemp, Display, TEXT("Debug category %s enabled"), *CategoryName);
                        }
                        
                        return;
                    }
                }
            }
            
            UE_LOG(LogTemp, Warning, TEXT("Unknown debug category: %s"), *CategoryName);
            return;
        }
    }
}

// ImGui drawing implementation
void UPlatformerDebugComponent::OnImGuiDraw()
{
    // Only draw if debug window is enabled
    if (!bShowDebugWindow)
    {
        return;
    }
    
    // Create main debug window with docking
    ImGuiWindowFlags window_flags = ImGuiWindowFlags_MenuBar;
    
    if (!ImGui::Begin("2.5D Platformer Debug", &bShowDebugWindow, window_flags))
    {
        ImGui::End();
        return;
    }
    
    // Menu bar
    if (ImGui::BeginMenuBar())
    {
        if (ImGui::BeginMenu("Options"))
        {
            if (ImGui::MenuItem("Save Layout"))
            {
                // TODO: Save ImGui layout
            }
            if (ImGui::MenuItem("Reset Layout"))
            {
                // TODO: Reset ImGui layout
            }
            ImGui::Separator();
            if (ImGui::MenuItem("Close", "Alt+D"))
            {
                bShowDebugWindow = false;
            }
            ImGui::EndMenu();
        }
        if (ImGui::BeginMenu("Categories"))
        {
            const UEnum* EnumPtr = FindObject<UEnum>(ANY_PACKAGE, TEXT("EDebugCategory"), true);
            if (EnumPtr)
            {
                for (int32 i = 0; i < EnumPtr->GetMaxEnumValue(); ++i)
                {
                    EDebugCategory Category = static_cast<EDebugCategory>(i);
                    bool bEnabled = EnabledCategories.Contains(Category);
                    
                    FString DisplayName = EnumPtr->GetDisplayNameTextByIndex(i).ToString();
                    if (ImGui::MenuItem(TCHAR_TO_ANSI(*DisplayName), NULL, bEnabled))
                    {
                        if (bEnabled)
                        {
                            EnabledCategories.Remove(Category);
                        }
                        else
                        {
                            EnabledCategories.Add(Category);
                        }
                    }
                }
            }
            ImGui::EndMenu();
        }
        ImGui::EndMenuBar();
    }
    
    // Tab bar for sections
    if (ImGui::BeginTabBar("DebugTabs", ImGuiTabBarFlags_None))
    {
        if (ImGui::BeginTabItem("Performance"))
        {
            DrawPerformanceSection();
            ImGui::EndTabItem();
        }
        
        if (ImGui::BeginTabItem("Character"))
        {
            DrawCharacterSection();
            ImGui::EndTabItem();
        }
        
        if (ImGui::BeginTabItem("Visualization"))
        {
            DrawVisualizationSection();
            ImGui::EndTabItem();
        }
        
        if (ImGui::BeginTabItem("Bookmarks"))
        {
            DrawBookmarksSection();
            ImGui::EndTabItem();
        }
        
        if (ImGui::BeginTabItem("Debug Camera"))
        {
            DrawDebugCameraSection();
            ImGui::EndTabItem();
        }
        
        ImGui::EndTabBar();
    }
    
    ImGui::End();
}

void UPlatformerDebugComponent::DrawPerformanceSection()
{
    ImGui::Text("Frame Time: %.3f ms (%.1f FPS)", 
                1000.0f / ImGui::GetIO().Framerate, 
                ImGui::GetIO().Framerate);
    
    // Frame time graph
    static float values[90] = {};
    static int values_offset = 0;
    
    // Store new value
    values[values_offset] = 1000.0f / ImGui::GetIO().Framerate;
    values_offset = (values_offset + 1) % IM_ARRAYSIZE(values);
    
    // Plot frame time graph
    ImGui::PlotLines("Frame Time (ms)", values, IM_ARRAYSIZE(values), values_offset, 
                      NULL, 0.0f, 50.0f, ImVec2(0, 80));
    
    // Memory stats
    FPlatformMemoryStats MemStats = FPlatformMemory::GetStats();
    ImGui::Text("Physical Memory: %.2f MB / %.2f MB", 
                MemStats.UsedPhysical / (1024.0f * 1024.0f),
                MemStats.TotalPhysical / (1024.0f * 1024.0f));
}

void UPlatformerDebugComponent::DrawCharacterSection()
{
    ACharacter* Character = GetPlayerCharacter();
    UCharacterMovementComponent* MovementComp = GetCharacterMovement();
    
    if (!Character || !MovementComp)
    {
        ImGui::TextColored(ImVec4(1.0f, 0.3f, 0.3f, 1.0f), "Character not found!");
        return;
    }
    
    // Character position
    FVector Location = Character->GetActorLocation();
    ImGui::Text("Position: X=%.2f Y=%.2f Z=%.2f", Location.X, Location.Y, Location.Z);
    
    // Character movement
    FVector Velocity = MovementComp->Velocity;
    float Speed = Velocity.Size();
    ImGui::Text("Velocity: X=%.2f Y=%.2f Z=%.2f (Speed: %.2f)", 
                Velocity.X, Velocity.Y, Velocity.Z, Speed);
    
    // Movement state
    ImGui::Text("Movement Mode: %s", 
                MovementComp->IsFalling() ? "Falling" : 
                MovementComp->IsSwimming() ? "Swimming" : 
                MovementComp->IsFlying() ? "Flying" : "Walking");
    
    ImGui::Separator();
    
    // Character tweaks
    ImGui::Checkbox("Invincible", &bPlayerInvincible);
    
    float MaxWalkSpeed = MovementComp->MaxWalkSpeed;
    if (ImGui::SliderFloat("Max Walk Speed", &MaxWalkSpeed, 100.0f, 2000.0f))
    {
        MovementComp->MaxWalkSpeed = MaxWalkSpeed;
    }
    
    float JumpZVelocity = MovementComp->JumpZVelocity;
    if (ImGui::SliderFloat("Jump Height", &JumpZVelocity, 100.0f, 2000.0f))
    {
        MovementComp->JumpZVelocity = JumpZVelocity;
    }
    
    float GravityScale = MovementComp->GravityScale;
    if (ImGui::SliderFloat("Gravity Scale", &GravityScale, 0.1f, 5.0f))
    {
        MovementComp->GravityScale = GravityScale;
    }
    
    ImGui::Separator();
    
    // Buttons for common actions
    if (ImGui::Button("Reset Movement Properties"))
    {
        // Reset to default values
        MovementComp->MaxWalkSpeed = 600.0f;
        MovementComp->JumpZVelocity = 600.0f;
        MovementComp->GravityScale = 1.0f;
    }
    
    if (ImGui::Button("Kill Velocity"))
    {
        MovementComp->StopMovementImmediately();
    }
}

void UPlatformerDebugComponent::DrawVisualizationSection()
{
    ImGui::Checkbox("Show Movement Path", &bShowMovementPath);
    
    // Debug rendering options
    static bool bShowCollision = false;
    if (ImGui::Checkbox("Show Collision", &bShowCollision))
    {
        if (UWorld* World = GetWorld())
        {
            if (bShowCollision)
            {
                World->Exec(World, TEXT("show collision"));
            }
            else
            {
                World->Exec(World, TEXT("show collision off"));
            }
        }
    }
    
    static bool bShowNavigation = false;
    if (ImGui::Checkbox("Show Navigation", &bShowNavigation))
    {
        if (UWorld* World = GetWorld())
        {
            if (bShowNavigation)
            {
                World->Exec(World, TEXT("show navigation"));
            }
            else
            {
                World->Exec(World, TEXT("show navigation off"));
            }
        }
    }
    
    static bool bShowDebugStats = false;
    if (ImGui::Checkbox("Show Debug Stats", &bShowDebugStats))
    {
        if (UWorld* World = GetWorld())
        {
            if (bShowDebugStats)
            {
                World->Exec(World, TEXT("stat fps"));
                World->Exec(World, TEXT("stat unit"));
            }
            else
            {
                World->Exec(World, TEXT("stat fps off"));
                World->Exec(World, TEXT("stat unit off"));
            }
        }
    }
}

void UPlatformerDebugComponent::DrawBookmarksSection()
{
    // Create new bookmark
    static char bookmarkName[128] = "";
    ImGui::InputText("Bookmark Name", bookmarkName, IM_ARRAYSIZE(bookmarkName));
    
    if (ImGui::Button("Save Current Position"))
    {
        if (bookmarkName[0] != '\0')
        {
            SaveCurrentPositionAsBookmark(FString(bookmarkName));
            bookmarkName[0] = '\0'; // Clear input
        }
    }
    
    ImGui::Separator();
    
    // List bookmarks
    ImGui::Text("Saved Positions:");
    
    if (PositionBookmarks.Num() == 0)
    {
        ImGui::TextColored(ImVec4(0.7f, 0.7f, 0.7f, 1.0f), "No bookmarks saved yet.");
    }
    
    for (int32 i = 0; i < PositionBookmarks.Num(); i++)
    {
        const FPositionBookmark& Bookmark = PositionBookmarks[i];
        FString ButtonLabel = FString::Printf(TEXT("Teleport##%d"), i);
        
        ImGui::PushID(i);
        
        if (ImGui::Button(TCHAR_TO_ANSI(*ButtonLabel)))
        {
            TeleportPlayerTo(Bookmark.Location);
        }
        
        ImGui::SameLine();
        ImGui::Text("%s (X=%.1f Y=%.1f Z=%.1f)", 
                    TCHAR_TO_ANSI(*Bookmark.Name), 
                    Bookmark.Location.X, 
                    Bookmark.Location.Y, 
                    Bookmark.Location.Z);
        
        ImGui::PopID();
    }
    
    if (PositionBookmarks.Num() > 0 && ImGui::Button("Clear All Bookmarks"))
    {
        PositionBookmarks.Empty();
    }
}

void UPlatformerDebugComponent::DrawDebugCameraSection()
{
    ImGui::Checkbox("Enable Debug Camera", &bDebugCameraEnabled);
    
    if (bDebugCameraEnabled)
    {
        APlayerController* PC = GetPlayerController();
        if (PC && !PC->IsViewTarget(PC->GetSpectatorPawn()))
        {
            // Switch to spectator cam
            GetWorld()->Exec(GetWorld(), TEXT("viewmode unlit"));
            PC->SetViewTargetWithBlend(PC->GetSpectatorPawn(), 0.5f);
        }
        
        // Camera speed settings
        static float CameraSpeed = 1.0f;
        if (ImGui::SliderFloat("Camera Speed", &CameraSpeed, 0.1f, 10.0f))
        {
            GetWorld()->Exec(GetWorld(), 
                             *FString::Printf(TEXT("spectatorfriction %.1f"), CameraSpeed));
        }
        
        ImGui::Text("Controls:");
        ImGui::BulletText("WASD - Move camera");
        ImGui::BulletText("Q/E - Up/Down");
        ImGui::BulletText("Right Mouse - Look around");
        ImGui::BulletText("Shift - Move faster");
    }
    else
    {
        APlayerController* PC = GetPlayerController();
        if (PC && PC->IsViewTarget(PC->GetSpectatorPawn()))
        {
            // Switch back to player
            GetWorld()->Exec(GetWorld(), TEXT("viewmode lit"));
            PC->SetViewTargetWithBlend(GetPlayerCharacter(), 0.5f);
        }
    }
}
```

## Adding the Component to Your Game

Add the debug component to your GameMode:

```cpp
// In your PlatformerGameMode.h
#include "PlatformerDebugComponent.h"

UPROPERTY(VisibleAnywhere, Category = "Debug")
UPlatformerDebugComponent* DebugComponent;

// In your PlatformerGameMode.cpp constructor
DebugComponent = CreateDefaultSubobject<UPlatformerDebugComponent>(TEXT("DebugComponent"));

// In your PlatformerGameMode.h
UFUNCTION(BlueprintPure, Category = "Debug")
bool IsPlayerInvincible() const 
{
    return DebugComponent ? DebugComponent->IsPlayerInvincible() : false;
}

UFUNCTION(BlueprintCallable, Category = "Debug")
UPlatformerDebugComponent* GetDebugComponent() const { return DebugComponent; }
```

## Setting Up Input for Debug Window Toggle

Create an input binding to toggle the debug window:

1. Open your project settings
2. Go to **Engine → Input → Action Mappings**
3. Add a new mapping named "ToggleDebugWindow" bound to Alt+D

Then in your player controller:

```cpp
// In your PlayerController.h
void ToggleDebugWindow();

// In PlayerController.cpp SetupInputComponent()
InputComponent->BindAction("ToggleDebugWindow", IE_Pressed, this, &APlatformerPlayerController::ToggleDebugWindow);

// In PlayerController.cpp
void APlatformerPlayerController::ToggleDebugWindow()
{
    // Use console command to toggle
    GetWorld()->Exec(GetWorld(), TEXT("DebugWindow.Toggle"));
}
```

Continue to [Gameplay Debug Helpers](./4_debug_system_gameplay.md) to implement additional gameplay debugging features. 