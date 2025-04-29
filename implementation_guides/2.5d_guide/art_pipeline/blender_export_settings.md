# Blender Export Settings for Unreal Engine

This guide provides detailed export settings for Blender artists creating assets for your 2.5D platformer in Unreal Engine 5.5. Following these settings ensures your assets will import correctly with minimal adjustment needed in Unreal.

## Before Exporting: Essential Preparation

Before exporting any asset from Blender, ensure:

1. **Clean up your scene**
   - Delete unused objects, materials, and collections
   - Apply all modifiers (especially subdivision surface)
   - Remove any non-deforming shape keys

2. **Check scale and position**
   - Objects should use proper real-world scale (1 Blender unit = 1 cm)
   - Character meshes should be at origin, standing on the ground plane (Z=0)
   - Root bones for rigs should be at origin

3. **Finalize UVs and materials**
   - All meshes must have proper UV unwrapping
   - Check for UV overlaps and fix them
   - Ensure texture maps are named clearly

## FBX Export Settings for Static Meshes

When exporting static meshes (platforms, props, etc.), use these settings:

1. **Access the export dialog**
   - File > Export > FBX (.fbx)

2. **Main settings**
   ```
   Scale: 1.0
   Apply Scalings: FBX Units Scale
   Forward: -Y Forward
   Up: Z Up
   Apply Unit: Checked
   Use Space Transform: Checked
   ```

3. **Geometry settings**
   ```
   Smoothing: Face
   Apply Modifiers: Checked
   Loose Edges: Unchecked
   Tangent Space: Checked
   ```

4. **Armature settings (for static meshes)**
   ```
   Add Leaf Bones: Unchecked
   Primary Bone Axis: Y
   Secondary Bone Axis: X
   ```

5. **Bake Animation (for static meshes)**
   ```
   All animation options: Unchecked
   ```

## FBX Export Settings for Skeletal Meshes

For character models and other animated objects:

1. **Main settings**
   ```
   Scale: 1.0
   Apply Scalings: FBX Units Scale
   Forward: -Y Forward
   Up: Z Up
   Apply Unit: Checked
   Use Space Transform: Checked
   ```

2. **Geometry settings**
   ```
   Smoothing: Face
   Apply Modifiers: Checked
   Loose Edges: Unchecked
   Tangent Space: Checked
   ```

3. **Armature settings (CRITICAL for animated meshes)**
   ```
   Add Leaf Bones: Unchecked (important - Unreal doesn't need these)
   Primary Bone Axis: Y
   Secondary Bone Axis: X
   Armature FBX Node Type: Root (important for proper importing)
   Only Deform Bones: Unchecked (unless you have specific control bones)
   ```

4. **Bake Animation settings**
   If exporting animations with the mesh:
   ```
   Key All Bones: Checked
   NLA Strips: Unchecked (unless specifically using NLA)
   Force Start/End Keying: Checked
   Sampling Rate: 30 (or match your project's frame rate)
   Simplify: 0.0 (or a low value like 0.01 for optimization)
   ```

## Texture Export Guidelines

When exporting textures to use with your models:

1. **Recommended formats**
   - Diffuse/Albedo maps: PNG or TGA
   - Normal maps: TGA (no compression)
   - Roughness/Metallic/AO maps: PNG or TGA

2. **Resolution guidelines**
   - Character textures: 2048×2048 for main character, 1024×1024 for NPCs
   - Environment pieces: 1024×1024 or 2048×2048 depending on size
   - Small props: 512×512 or 1024×1024
   - Always use power-of-two resolutions (512, 1024, 2048, 4096)

3. **Naming conventions**
   ```
   [AssetName]_[TextureType].[extension]
   
   Examples:
   Hero_BaseColor.png
   Hero_Normal.tga
   Hero_ORM.png (Combined Occlusion, Roughness, Metallic)
   ```

## Animation Export Process

For exporting animations separately from the mesh:

1. **Prepare animations**
   - Use the Action Editor to organize animations
   - Name each action clearly (e.g., "Hero_Idle", "Hero_Run", "Hero_Jump")
   - Set proper start/end frames for each action

2. **Export settings**
   - Export only the armature and mesh
   - Enable "Selected Objects" to avoid exporting the entire scene
   - In the "Bake Animation" section:
     ```
     Key All Bones: Checked
     Force Start/End Keying: Checked
     Set Start/End frame to match your action
     ```

3. **One animation per file approach**
   - Export each animation as a separate FBX file
   - Use consistent naming: `[CharacterName]_[AnimationName].fbx`
   - This approach simplifies animation management in Unreal

## Common Export Issues and Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| Model imports at wrong scale | Incorrect scale settings | Set Scale to 1.0, Apply Unit checked |
| Rotated model in Unreal | Wrong axis settings | Use -Y Forward, Z Up |
| Missing bones | "Only Deform Bones" checked | Uncheck this option |
| Animation looks jittery | Low sampling rate | Increase sampling rate to 30 or 60 |
| Materials not importing | Material slots empty | Create basic materials for each slot |
| Inverted normals | Model has flipped faces | Recalculate normals in Blender |
| Mesh deforms incorrectly | Bad weights or bone orientation | Check weight painting and bone axes |

## Testing Your Export Process

Before committing to a full asset production pipeline, test the workflow with a simple asset:

1. Create a simple test character with basic rig and one animation
2. Export using these settings
3. Import into Unreal Engine
4. Verify scale, orientation, and animation playback
5. Adjust settings if needed

## Example Export Checklist

Use this checklist for each asset:

- [ ] Model is at origin (0,0,0)
- [ ] All modifiers are applied
- [ ] UVs are properly unwrapped
- [ ] Materials/texture slots are named clearly
- [ ] Scale is set to real-world values
- [ ] Forward: -Y, Up: Z settings used
- [ ] "Add Leaf Bones" is unchecked
- [ ] Textures exported at power-of-two resolutions
- [ ] FBX file named according to project conventions

## Conclusion

Consistent export settings are crucial for a smooth art pipeline between Blender and Unreal Engine. These settings create a reliable workflow that minimizes the need for adjustments after import, saving valuable development time.

For your 2.5D platformer, establish these standards with your artists from the beginning, and make sure to periodically review and test the pipeline with simple assets before proceeding to more complex ones. 