# Debug System: Gameplay Debug Features

This guide covers additional gameplay-specific debug features for your 2.5D platformer.

## Enhanced Character Debugging

Building on our core debug component, let's add platformer-specific debug visualization and tools.

### Movement Path Visualization

Add this to your character class to visualize the movement path:

```cpp
// In your PlatformerCharacter.h
#include "PlatformerDebugComponent.h"
#include "GameFramework/GameModeBase.h"

// In PlatformerCharacter.cpp Tick()
void APlatformerCharacter::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);
    
    // Debug visualization
    #if !UE_BUILD_SHIPPING
    if (UWorld* World = GetWorld())
    {
        AGameModeBase* GameMode = World->GetAuthGameMode();
        if (GameMode)
        {
            UPlatformerDebugComponent* DebugComp = GameMode->FindComponentByClass<UPlatformerDebugComponent>();
            if (DebugComp && DebugComp->ShouldShowMovementPath())
            {
                static TArray<FVector> MovementHistory;
                
                // Add current position to history
                MovementHistory.Add(GetActorLocation());
                
                // Keep only last 100 positions
                if (MovementHistory.Num() > 100)
                {
                    MovementHistory.RemoveAt(0);
                }
                
                // Draw path
                for (int32 i = 1; i < MovementHistory.Num(); i++)
                {
                    DrawDebugLine(
                        World,
                        MovementHistory[i-1],
                        MovementHistory[i],
                        FColor::Green,
                        false,
                        -1.0f,
                        0,
                        2.0f
                    );
                }
                
                // Draw direction arrow
                if (MovementHistory.Num() > 1)
                {
                    FVector LastPos = MovementHistory.Last();
                    FVector Direction = GetVelocity().GetSafeNormal();
                    DrawDebugDirectionalArrow(
                        World,
                        LastPos,
                        LastPos + Direction * 100.0f,
                        50.0f,
                        FColor::Yellow,
                        false,
                        -1.0f,
                        0,
                        2.0f
                    );
                }
            }
        }
    }
    #endif
}
```

### Collision Debug Extension

Let's add a way to visualize the character's collision:

```cpp
// Add to PlatformerDebugComponent.h
UFUNCTION(BlueprintCallable, Category = "Debug")
void TogglePlayerCollisionVisibility();

// Add to PlatformerDebugComponent.cpp
void UPlatformerDebugComponent::TogglePlayerCollisionVisibility()
{
    ACharacter* Character = GetPlayerCharacter();
    if (Character)
    {
        UCapsuleComponent* Capsule = Character->GetCapsuleComponent();
        if (Capsule)
        {
            static bool bShowCollision = false;
            bShowCollision = !bShowCollision;
            
            if (bShowCollision)
            {
                Capsule->SetHiddenInGame(false);
                Capsule->SetVisibility(true);
            }
            else
            {
                Capsule->SetHiddenInGame(true);
            }
        }
    }
}

// Add a new button in DrawCharacterSection()
if (ImGui::Button("Toggle Collision Visibility"))
{
    TogglePlayerCollisionVisibility();
}
```

## Advanced Teleport System

Enhance the teleport system with additional features:

```cpp
// Add to PlatformerDebugComponent.h
UFUNCTION(BlueprintCallable, Category = "Debug")
void TeleportPlayerToStart();

UFUNCTION(BlueprintCallable, Category = "Debug")
void TeleportPlayerToCheckpoint();

// Add to PlatformerDebugComponent.cpp
void UPlatformerDebugComponent::TeleportPlayerToStart()
{
    UWorld* World = GetWorld();
    if (!World)
    {
        return;
    }
    
    // Find player start
    for (TActorIterator<APlayerStart> It(World); It; ++It)
    {
        APlayerStart* PlayerStart = *It;
        TeleportPlayerTo(PlayerStart->GetActorLocation());
        return;
    }
}

void UPlatformerDebugComponent::TeleportPlayerToCheckpoint()
{
    // Replace this with your game's checkpoint system
    // This is just an example
    ACharacter* Character = GetPlayerCharacter();
    if (!Character)
    {
        return;
    }
    
    // Example: Find actor with checkpoint tag
    UWorld* World = GetWorld();
    if (!World)
    {
        return;
    }
    
    for (TActorIterator<AActor> It(World); It; ++It)
    {
        AActor* Actor = *It;
        if (Actor->ActorHasTag(TEXT("Checkpoint")))
        {
            TeleportPlayerTo(Actor->GetActorLocation());
            return;
        }
    }
}

// Add these buttons to the DrawBookmarksSection()
if (ImGui::Button("Teleport to Start"))
{
    TeleportPlayerToStart();
}

ImGui::SameLine();

if (ImGui::Button("Teleport to Last Checkpoint"))
{
    TeleportPlayerToCheckpoint();
}
```

## Enemy & AI Debugging

If your game has enemies with AI, add debugging for them:

```cpp
// Add to PlatformerDebugComponent.h
void DrawEnemyDebugSection();

// Add this tab in OnImGuiDraw()
if (ImGui::BeginTabItem("Enemies & AI"))
{
    DrawEnemyDebugSection();
    ImGui::EndTabItem();
}

// Implement the new section
void UPlatformerDebugComponent::DrawEnemyDebugSection()
{
    UWorld* World = GetWorld();
    if (!World)
    {
        ImGui::TextColored(ImVec4(1.0f, 0.3f, 0.3f, 1.0f), "World not available!");
        return;
    }
    
    int32 EnemyCount = 0;
    
    // Toggle for AI debug
    static bool bShowAIDebug = false;
    if (ImGui::Checkbox("Show AI Debug", &bShowAIDebug))
    {
        // Toggle AI debug visualization
        if (bShowAIDebug)
        {
            World->Exec(World, TEXT("ai logall 1"));
        }
        else
        {
            World->Exec(World, TEXT("ai logall 0"));
        }
    }
    
    // Enemy controls
    if (ImGui::Button("Freeze All Enemies"))
    {
        // This assumes your enemies have a tag "Enemy"
        for (TActorIterator<AActor> It(World); It; ++It)
        {
            AActor* Actor = *It;
            if (Actor->ActorHasTag(TEXT("Enemy")))
            {
                EnemyCount++;
                // Freeze enemy
                Actor->CustomTimeDilation = 0.0f;
            }
        }
    }
    
    ImGui::SameLine();
    
    if (ImGui::Button("Resume All Enemies"))
    {
        for (TActorIterator<AActor> It(World); It; ++It)
        {
            AActor* Actor = *It;
            if (Actor->ActorHasTag(TEXT("Enemy")))
            {
                // Resume enemy
                Actor->CustomTimeDilation = 1.0f;
            }
        }
    }
    
    // Count enemies
    for (TActorIterator<AActor> It(World); It; ++It)
    {
        AActor* Actor = *It;
        if (Actor->ActorHasTag(TEXT("Enemy")))
        {
            EnemyCount++;
        }
    }
    
    ImGui::Text("Enemy Count: %d", EnemyCount);
    
    // Enemy spawning for testing
    static int32 SpawnCount = 1;
    ImGui::SliderInt("Spawn Count", &SpawnCount, 1, 10);
    
    static char enemyClass[128] = "Blueprint'/Game/Blueprints/Enemies/BP_Enemy.BP_Enemy'";
    ImGui::InputText("Enemy Class", enemyClass, IM_ARRAYSIZE(enemyClass));
    
    if (ImGui::Button("Spawn Enemies at Player"))
    {
        ACharacter* Character = GetPlayerCharacter();
        if (Character)
        {
            FVector SpawnLocation = Character->GetActorLocation();
            SpawnLocation.Z += 100.0f; // Spawn above player
            
            // Load enemy class
            UClass* EnemyClass = nullptr;
            FString ClassPath(enemyClass);
            EnemyClass = LoadObject<UClass>(nullptr, *ClassPath);
            
            if (EnemyClass)
            {
                // Spawn enemies in a circle around the player
                for (int32 i = 0; i < SpawnCount; i++)
                {
                    float Angle = (2.0f * PI * i) / SpawnCount;
                    FVector Offset(FMath::Cos(Angle) * 300.0f, FMath::Sin(Angle) * 300.0f, 0.0f);
                    FVector Location = SpawnLocation + Offset;
                    
                    World->SpawnActor<AActor>(EnemyClass, Location, FRotator::ZeroRotator);
                }
            }
            else
            {
                UE_LOG(LogTemp, Warning, TEXT("Failed to load enemy class: %s"), *ClassPath);
            }
        }
    }
}
```

## Level Information and World Debug

Add a level information section:

```cpp
// Add to PlatformerDebugComponent.h
void DrawLevelInfoSection();

// Add this tab in OnImGuiDraw()
if (ImGui::BeginTabItem("Level Info"))
{
    DrawLevelInfoSection();
    ImGui::EndTabItem();
}

// Implement the new section
void UPlatformerDebugComponent::DrawLevelInfoSection()
{
    UWorld* World = GetWorld();
    if (!World)
    {
        ImGui::TextColored(ImVec4(1.0f, 0.3f, 0.3f, 1.0f), "World not available!");
        return;
    }
    
    // Current level
    FString CurrentLevel = World->GetMapName();
    ImGui::Text("Current Level: %s", TCHAR_TO_ANSI(*CurrentLevel));
    
    // Actor counts
    int32 TotalActors = 0;
    int32 HiddenActors = 0;
    int32 StaticMeshActors = 0;
    int32 Pawns = 0;
    
    for (TActorIterator<AActor> It(World); It; ++It)
    {
        AActor* Actor = *It;
        TotalActors++;
        
        if (!Actor->IsVisible())
        {
            HiddenActors++;
        }
        
        if (Actor->IsA<AStaticMeshActor>())
        {
            StaticMeshActors++;
        }
        
        if (Actor->IsA<APawn>())
        {
            Pawns++;
        }
    }
    
    ImGui::Text("Total Actors: %d", TotalActors);
    ImGui::Text("Hidden Actors: %d", HiddenActors);
    ImGui::Text("Static Mesh Actors: %d", StaticMeshActors);
    ImGui::Text("Pawns: %d", Pawns);
    
    // Level loading
    if (ImGui::CollapsingHeader("Level Loading"))
    {
        static char levelName[128] = "/Game/Maps/TestMap";
        ImGui::InputText("Level Path", levelName, IM_ARRAYSIZE(levelName));
        
        if (ImGui::Button("Load Level"))
        {
            FString LevelNameStr(levelName);
            UGameplayStatics::OpenLevel(World, FName(*LevelNameStr));
        }
        
        if (ImGui::Button("Reload Current Level"))
        {
            UGameplayStatics::OpenLevel(World, FName(*World->GetName()));
        }
    }
    
    // World settings
    if (ImGui::CollapsingHeader("World Settings"))
    {
        AWorldSettings* WorldSettings = World->GetWorldSettings();
        if (WorldSettings)
        {
            float TimeScale = WorldSettings->GetEffectiveTimeDilation();
            if (ImGui::SliderFloat("Time Scale", &TimeScale, 0.1f, 5.0f))
            {
                WorldSettings->SetTimeDilation(TimeScale);
            }
            
            static bool bKinematicDebug = false;
            if (ImGui::Checkbox("Show Kinematic Bones", &bKinematicDebug))
            {
                if (bKinematicDebug)
                {
                    World->Exec(World, TEXT("ShowDebug KinematicBones"));
                }
                else
                {
                    World->Exec(World, TEXT("ShowDebug KinematicBones off"));
                }
            }
        }
    }
}
```

## Console Command Execution

Add a console command section for quick command execution:

```cpp
// Add to PlatformerDebugComponent.h
void DrawConsoleSection();

// Add this tab in OnImGuiDraw()
if (ImGui::BeginTabItem("Console"))
{
    DrawConsoleSection();
    ImGui::EndTabItem();
}

// Implement the new section
void UPlatformerDebugComponent::DrawConsoleSection()
{
    static char consoleCommand[256] = "";
    
    ImGui::Text("Enter Console Command:");
    bool commandEntered = ImGui::InputText("##ConsoleInput", consoleCommand, IM_ARRAYSIZE(consoleCommand), 
                                         ImGuiInputTextFlags_EnterReturnsTrue);
    
    ImGui::SameLine();
    
    bool executePressed = ImGui::Button("Execute");
    
    if (commandEntered || executePressed)
    {
        if (consoleCommand[0] != '\0')
        {
            UWorld* World = GetWorld();
            if (World)
            {
                World->Exec(World, UTF8_TO_TCHAR(consoleCommand));
                
                // Keep history of commands
                static std::vector<std::string> CommandHistory;
                CommandHistory.push_back(consoleCommand);
                
                // Clear input
                consoleCommand[0] = '\0';
            }
        }
    }
    
    // Common commands buttons
    if (ImGui::Button("stat fps"))
    {
        GetWorld()->Exec(GetWorld(), TEXT("stat fps"));
    }
    
    ImGui::SameLine();
    
    if (ImGui::Button("stat unit"))
    {
        GetWorld()->Exec(GetWorld(), TEXT("stat unit"));
    }
    
    ImGui::SameLine();
    
    if (ImGui::Button("stat unitgraph"))
    {
        GetWorld()->Exec(GetWorld(), TEXT("stat unitgraph"));
    }
    
    if (ImGui::Button("stat startfile"))
    {
        GetWorld()->Exec(GetWorld(), TEXT("stat startfile"));
    }
    
    ImGui::SameLine();
    
    if (ImGui::Button("stat stopfile"))
    {
        GetWorld()->Exec(GetWorld(), TEXT("stat stopfile"));
    }
    
    ImGui::Separator();
    
    if (ImGui::Button("r.SetRes 1920x1080"))
    {
        GetWorld()->Exec(GetWorld(), TEXT("r.SetRes 1920x1080"));
    }
    
    ImGui::SameLine();
    
    if (ImGui::Button("r.SetRes 1280x720"))
    {
        GetWorld()->Exec(GetWorld(), TEXT("r.SetRes 1280x720"));
    }
}
```

## Combat Debug Helpers

If your game has a combat system, add debugging for it:

```cpp
// Add to PlatformerDebugComponent.h
void DrawCombatSection();

// Add this tab in OnImGuiDraw()
if (ImGui::BeginTabItem("Combat"))
{
    DrawCombatSection();
    ImGui::EndTabItem();
}

// Implement the new section
void UPlatformerDebugComponent::DrawCombatSection()
{
    // Assuming your character has a health component
    ACharacter* Character = GetPlayerCharacter();
    if (!Character)
    {
        ImGui::TextColored(ImVec4(1.0f, 0.3f, 0.3f, 1.0f), "Character not found!");
        return;
    }
    
    // Health manipulation (this assumes you have a health component or interface)
    if (ImGui::Button("Heal Player"))
    {
        // Example: call Heal function
        // This is an example - replace with your actual health system
        UFunction* HealFunction = Character->FindFunction(FName("Heal"));
        if (HealFunction)
        {
            struct
            {
                float Amount = 100.0f;
            } Params;
            
            Character->ProcessEvent(HealFunction, &Params);
        }
    }
    
    ImGui::SameLine();
    
    if (ImGui::Button("Damage Player"))
    {
        // Example: call Damage function
        // This is an example - replace with your actual damage system
        UFunction* DamageFunction = Character->FindFunction(FName("ReceiveDamage"));
        if (DamageFunction)
        {
            struct
            {
                float Amount = 10.0f;
            } Params;
            
            Character->ProcessEvent(DamageFunction, &Params);
        }
    }
    
    // Weapons/abilities testing
    if (ImGui::CollapsingHeader("Weapons & Abilities"))
    {
        // This depends on your specific implementation
        // Here's a simple example for testing abilities
        
        if (ImGui::Button("Trigger Special Attack"))
        {
            UFunction* SpecialAttackFunction = Character->FindFunction(FName("TriggerSpecialAttack"));
            if (SpecialAttackFunction)
            {
                Character->ProcessEvent(SpecialAttackFunction, nullptr);
            }
        }
        
        if (ImGui::Button("Max Out Abilities"))
        {
            // Example: find a component or function to max abilities
            UFunction* MaxAbilitiesFunction = Character->FindFunction(FName("MaxOutAbilities"));
            if (MaxAbilitiesFunction)
            {
                Character->ProcessEvent(MaxAbilitiesFunction, nullptr);
            }
        }
    }
    
    // Hit effect visualization
    if (ImGui::CollapsingHeader("Hit Effects"))
    {
        static int32 SelectedEffect = 0;
        const char* Effects[] = { "Small Hit", "Medium Hit", "Large Hit", "Critical Hit" };
        
        ImGui::Combo("Effect Type", &SelectedEffect, Effects, IM_ARRAYSIZE(Effects));
        
        if (ImGui::Button("Play Hit Effect"))
        {
            // Example: call function to play hit effect
            UFunction* PlayHitEffectFunction = Character->FindFunction(FName("PlayHitEffect"));
            if (PlayHitEffectFunction)
            {
                struct
                {
                    int32 EffectType = SelectedEffect;
                } Params;
                
                Character->ProcessEvent(PlayHitEffectFunction, &Params);
            }
        }
    }
}
```

## Saving & Loading Debug Helpers

```cpp
// Add to PlatformerDebugComponent.h
void DrawSaveGameSection();

// Add this tab in OnImGuiDraw()
if (ImGui::BeginTabItem("Save & Load"))
{
    DrawSaveGameSection();
    ImGui::EndTabItem();
}

// Implement the new section
void UPlatformerDebugComponent::DrawSaveGameSection()
{
    static char saveSlotName[64] = "DebugSaveSlot";
    ImGui::InputText("Save Slot Name", saveSlotName, IM_ARRAYSIZE(saveSlotName));
    
    if (ImGui::Button("Quick Save"))
    {
        // Example implementation - replace with your actual save game system
        UGameplayStatics::SaveGameToSlot(UGameplayStatics::CreateSaveGameObject(USaveGame::StaticClass()), 
                                         saveSlotName, 0);
    }
    
    ImGui::SameLine();
    
    if (ImGui::Button("Quick Load"))
    {
        // Example implementation - replace with your actual save game system
        UGameplayStatics::LoadGameFromSlot(saveSlotName, 0);
    }
    
    if (ImGui::Button("New Game"))
    {
        // Example implementation - replace with your actual new game logic
        UGameplayStatics::OpenLevel(GetWorld(), FName("StartLevel"));
    }
    
    // List existing save games
    ImGui::Separator();
    ImGui::Text("Existing Save Games:");
    
    TArray<FString> SaveSlots;
    TArray<int32> UserIndices;
    UGameplayStatics::GetSaveGameSlotNames(SaveSlots, UserIndices);
    
    if (SaveSlots.Num() == 0)
    {
        ImGui::TextColored(ImVec4(0.7f, 0.7f, 0.7f, 1.0f), "No save games found.");
    }
    
    for (int32 i = 0; i < SaveSlots.Num(); i++)
    {
        const FString& SlotName = SaveSlots[i];
        
        ImGui::PushID(i);
        
        if (ImGui::Button("Load"))
        {
            UGameplayStatics::LoadGameFromSlot(SlotName, UserIndices[i]);
        }
        
        ImGui::SameLine();
        
        if (ImGui::Button("Delete"))
        {
            UGameplayStatics::DeleteGameInSlot(SlotName, UserIndices[i]);
        }
        
        ImGui::SameLine();
        ImGui::Text("%s", TCHAR_TO_ANSI(*SlotName));
        
        ImGui::PopID();
    }
}
```

## Connecting to Your Gameplay Systems

This debug system is designed to be extensible. As you develop your platformer, you can add more sections and debug helpers:

1. For your combat system, extend the combat tab to show player stats, enemy stats, and damage calculations
2. For your movement system, add visualization of physics forces and jumping trajectories
3. For your progression system, add debug options to unlock levels, abilities, or items

Remember to include safeguards in your actual gameplay code:

```cpp
// Example: In your damage calculation
float CalculateDamage(float BaseDamage, float TargetDefense)
{
    // Check if player is invincible from debug menu
    APlatformerGameMode* GameMode = Cast<APlatformerGameMode>(GetWorld()->GetAuthGameMode());
    if (GameMode && GameMode->IsPlayerInvincible())
    {
        return 0.0f; // No damage if player is invincible
    }
    
    // Normal damage calculation
    return BaseDamage * (100.0f / (100.0f + TargetDefense));
}
```

## Debug Build Configuration

For shipping builds, it's important to disable debug features. Add a preprocessor directive to your debug component:

```cpp
#if !UE_BUILD_SHIPPING
// All debug rendering and functionality
#else
// Minimal functionality that's safe for shipping builds
#endif
```

This ensures that your debug tools don't affect performance or expose development features in your released game. 