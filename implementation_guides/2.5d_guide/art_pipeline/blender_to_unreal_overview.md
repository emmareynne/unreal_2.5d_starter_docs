# Working with Blender Artists: Overview

This guide covers the essentials of collaborating with artists using Blender for your 2.5D platformer project in Unreal Engine 5.5. Establishing a clear art pipeline from the beginning ensures smooth integration of assets and minimizes rework.

## Why a Clear Art Pipeline Matters

As a beginner working with artists, it's important to understand:

1. **Unreal has specific requirements** for imported assets to function correctly
2. **Setting up proper workflows early** prevents technical issues later
3. **Clear expectations save time** for both developers and artists
4. **Consistent asset structures** simplify integration and iteration
5. **Proper scale, orientation, and naming** avoid common import problems

## Asset Types for 2.5D Platformers

For a 2.5D platformer, you'll typically need these asset categories:

### 1. Character Assets
- Main character model and rig
- Enemy models and rigs
- NPC models and rigs

### 2. Environment Elements
- Platforms and terrain pieces
- Background and foreground elements
- Interactive objects (levers, doors, etc.)
- Collectible items

### 3. Props and Decorations
- Static props for scene dressing
- Animated props (moving machinery, etc.)
- Particle effect source meshes

## Collaboration Workflow Overview

The general workflow for collaborating with a Blender artist includes:

1. **Planning and Documentation**
   - Define asset requirements and specifications
   - Create asset lists with priorities
   - Establish naming conventions

2. **Asset Creation in Blender**
   - Modeling and UV mapping
   - Rigging and weight painting (for animated objects)
   - Animation creation
   - Materials setup

3. **Export Process**
   - FBX export with proper settings
   - Texture export in compatible formats
   - Animation export

4. **Import to Unreal**
   - Import with appropriate settings
   - Material setup in Unreal
   - Animation Blueprint creation
   - Asset testing and feedback

5. **Iteration Cycle**
   - Feedback on assets
   - Revisions in Blender
   - Re-export and update in Unreal

## Key Technical Requirements

Here are the foundational technical requirements to share with your Blender artist:

### Scale and Units
- Blender should use **Metric units** with a **1 unit = 1 cm** scale
- A typical human character should be around 180 units tall
- Unreal's scale is 1 unit = 1 cm (matches correctly when configured)

### Orientation
- Forward axis: **-Y** (negative Y)
- Up axis: **Z**
- Right axis: **X**

### Mesh Requirements
- Clean topology with proper edge flow
- Quad-based geometry (avoid n-gons)
- Proper UV unwrapping with no overlaps
- Texture resolution as powers of 2 (e.g., 512, 1024, 2048)
- Optimized polygon count (guideline: 1500-5000 for typical characters)

### Rigging Requirements
- Simple, properly weighted rigs
- Root bone at origin (0,0,0)
- Clear naming of bones
- Maximum 80-100 bones per character for good performance

## Next Steps

After reviewing this overview, check the following detailed guides:

1. [Blender Export Settings Guide](blender_export_settings.md)
2. [Asset Naming Conventions](asset_naming_conventions.md)
3. [Character Rigging Requirements](character_rigging_requirements.md)
4. [Environment Asset Guidelines](environment_asset_guidelines.md)
5. [Material Setup Pipeline](material_setup_pipeline.md)

## Conclusion

A successful collaboration with Blender artists relies on clear communication, established technical requirements, and consistent workflows. By establishing these guidelines early, you'll avoid common issues and create a more efficient development process for your 2.5D platformer.

Remember that this is a two-way process - be open to your artist's input and expertise, as they may have valuable insights on how to optimize assets and workflows. 