# Asset Naming Conventions

This guide outlines recommended naming conventions for your 2.5D platformer project. Establishing consistent naming standards from the beginning helps maintain organization as your project grows and makes collaboration with artists more efficient.

## Why Naming Conventions Matter

As a beginner, it might seem unnecessary to worry about naming conventions early on, but proper naming:

1. **Prevents content chaos** as your project grows
2. **Makes assets easier to find** for everyone on the team
3. **Reduces errors** when referencing assets in Blueprints
4. **Simplifies batch operations** like search and replace
5. **Creates clear ownership boundaries** between team members

## Folder Structure Recommendations

Start with a clear folder structure in your Unreal project:

```
Content/
├── Characters/
│   ├── Player/
│   │   ├── Animations/
│   │   ├── Materials/
│   │   ├── Meshes/
│   │   └── Textures/
│   └── Enemies/
│       ├── Type1/
│       ├── Type2/
│       └── ...
├── Environment/
│   ├── Platforms/
│   ├── Background/
│   ├── Foreground/
│   ├── Interactive/
│   └── Props/
├── VFX/
│   ├── Particles/
│   └── Niagara/
├── UI/
│   ├── HUD/
│   ├── Menus/
│   └── Icons/
└── _Core/
    ├── Materials/
    ├── Blueprints/
    └── Dev/
```

## Prefix System

Use prefixes to identify asset types at a glance:

| Prefix | Asset Type | Example |
|--------|------------|---------|
| **SK_** | Skeletal Mesh | SK_Hero |
| **SM_** | Static Mesh | SM_Platform |
| **T_** | Texture | T_Hero_Diffuse |
| **M_** | Material | M_Metal_Shiny |
| **MI_** | Material Instance | MI_Metal_Rusty |
| **A_** | Animation | A_Hero_Jump |
| **AM_** | Animation Montage | AM_Hero_Attack |
| **BP_** | Blueprint | BP_MovingPlatform |
| **NS_** | Niagara System | NS_Dust |
| **S_** | Sound | S_Jump |
| **SC_** | Sound Cue | SC_Footsteps |
| **W_** | Widget | W_MainMenu |

## Naming Pattern Structure

Follow this general pattern for naming assets:

```
Prefix_CharacterOrLocation_SpecificElement_Variant
```

Examples:
- `SK_Hero_Body`
- `SM_Forest_Platform_Mossy`
- `T_Hero_Diffuse_Damaged`
- `A_Hero_Jump_High`

## Texture Naming Conventions

Use consistent suffixes for texture types:

| Suffix | Texture Type | Example |
|--------|------------|---------|
| **_BC** | Base Color/Diffuse | T_Hero_BC |
| **_N** | Normal Map | T_Hero_N |
| **_ORM** | Occlusion/Roughness/Metallic | T_Hero_ORM |
| **_E** | Emissive | T_Lamp_E |
| **_M** | Mask | T_Vines_M |
| **_DP** | Displacement | T_Rock_DP |

## Animation Naming

For animations, include the action and any variants:

```
A_Character_Action_Variant
```

Examples:
- `A_Hero_Idle_Normal`
- `A_Hero_Idle_Tired`
- `A_Hero_Jump_Start`
- `A_Hero_Jump_Loop`
- `A_Hero_Jump_Land`
- `A_Enemy_Attack_Melee`

## Blueprint Naming

For Blueprints, use descriptive names that indicate functionality:

```
BP_Category_SpecificFunction
```

Examples:
- `BP_Platform_Moving`
- `BP_Platform_Crumbling`
- `BP_Enemy_Patroller`
- `BP_Collectible_Coin`

## Material Naming

Materials should indicate their base type and any significant properties:

```
M_Surface_Variant_Property
```

Examples:
- `M_Metal_Polished`
- `M_Wood_Weathered`
- `M_Platform_Bouncy`
- `MI_Metal_Rusty_Wet`

## Special Considerations for 2.5D

For 2.5D platformers specifically:

1. **Layer indicators** for parallax backgrounds
   - `SM_BG_Layer1_Mountains`
   - `SM_BG_Layer2_Hills`
   - `SM_BG_Layer3_Trees`

2. **Interaction markers** for interactive objects
   - `SM_INT_Lever`
   - `SM_INT_Button`
   - `BP_INT_MovingPlatform`

3. **Direction variants** for character assets
   - `A_Hero_Run_Left`
   - `A_Hero_Run_Right` (if not using mirroring)

## Communication with Artists

When working with Blender artists:

1. **Share this naming convention document**
2. **Provide template names** for planned assets
3. **Create folder structure in advance**
4. **Review asset names** before final delivery
5. **Give feedback early** if names don't match conventions

## Working Files Naming

For source files (like Blender files), use a similar convention:

```
[ProjectName]_[AssetType]_[AssetName]_[Version].[extension]
```

Examples:
- `Platformer_Char_Hero_v01.blend`
- `Platformer_Prop_Crate_v02.blend`
- `Platformer_Env_Forest_v03.blend`

## Versioning

Include version numbers for iterative work:

1. **Major revisions**: `_v01`, `_v02`, etc.
2. **Minor tweaks**: `_v01a`, `_v01b`, etc.
3. **Final assets**: No version number or `_FINAL`

## Test Your Naming System

Before committing to a naming convention with your artists:

1. **Create sample assets** with your naming system
2. **Try searching for specific types** of assets
3. **Check for name collisions** or confusion
4. **Adjust rules** as needed before full implementation

## Tools to Help

Consider these tools to help maintain naming conventions:

1. **Unreal's validation rules** (Project Settings → Editor → Content → Content Validation Rules)
2. **Asset naming scripts** for bulk renaming
3. **Shared documentation** like this guide for reference
4. **Regular reviews** of asset names during development

## Conclusion

While it might seem like extra work to establish naming conventions early, they become increasingly valuable as your project grows. By setting clear standards from the beginning of your collaboration with Blender artists, you'll create a more organized and efficient development process.

Remember that the best naming convention is one that your team consistently follows. It's better to have simple rules that everyone uses than complex ones that get ignored. 