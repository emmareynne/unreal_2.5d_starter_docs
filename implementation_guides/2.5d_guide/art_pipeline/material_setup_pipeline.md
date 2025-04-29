# Material Setup Pipeline for Blender to Unreal

This guide explains how to set up an efficient material workflow between Blender and Unreal Engine 5.5 for your 2.5D platformer. Following these guidelines will help ensure your materials look consistent and perform well.

## Understanding PBR Workflows

### What is PBR?

Physically Based Rendering (PBR) is the standard approach for creating realistic materials:

- **Based on real-world properties** of materials
- **Consistent across lighting conditions**
- **Uses specific texture maps** to define surface properties

### Key Material Properties

For 2.5D platformers, these are the essential material properties:

1. **Base Color** - The diffuse color of the surface
2. **Roughness** - How smooth or rough a surface appears
3. **Metallic** - Whether a surface is metal or non-metal
4. **Normal** - Surface detail without adding geometry
5. **Emissive** - Self-illumination (optional)
6. **Opacity** - Transparency (for glass, foliage, etc.)

## Blender Material Setup

### Creating Materials for Export

When creating materials in Blender:

1. **Use Principled BSDF shader**
   - Maps directly to Unreal's PBR workflow
   - Set Base Color, Metallic, Roughness properties
   - Connect Normal maps to Normal input

2. **Organize texture maps**
   - Name textures clearly ([AssetName]_[MapType])
   - Keep resolution consistent (1024, 2048, etc.)
   - Pack textures efficiently (ORM maps)

3. **UV mapping considerations**
   - Create clean, non-overlapping UVs
   - Consider secondary UVs for lightmaps
   - Pack UVs efficiently for texture space

### Baking Maps in Blender

For complex objects:

1. **High to low poly baking**
   - Create high-poly detail version
   - Create low-poly game version
   - Bake normal maps from high to low

2. **Texture baking settings**
   ```
   Margin: 4-8 pixels
   Selected to Active: Enabled (for high-to-low baking)
   Ray Distance: Set appropriate for your model scale
   ```

3. **Combined map baking**
   - Consider baking Occlusion, Roughness, and Metallic to single texture
   - Use RGB channels efficiently (R=Occlusion, G=Roughness, B=Metallic)
   - Saves texture memory and draw calls

## Exporting Textures from Blender

### Recommended Export Formats

| Map Type | Format | Compression | Notes |
|----------|--------|-------------|-------|
| Base Color | PNG | Medium | Keep color accuracy |
| Normal | TGA | None | Avoid compression artifacts |
| ORM | PNG | Low/None | Precision needed |
| Emissive | PNG | Medium | Only needed for glowing areas |
| Opacity Masks | PNG | None | Keep clean edges |

### Common Export Issues

1. **Gamma correction**
   - Color maps (Base Color, Emissive) are sRGB
   - Technical maps (Normal, ORM) are Linear
   - Set color space correctly during export

2. **Normal map format**
   - Blender uses OpenGL normal format
   - Unreal uses DirectX normal format
   - Convert or invert green channel when needed

## Unreal Material Setup

### Master Material Structure

Create flexible master materials to serve as templates:

1. **Basic master material**
   ```
   Parameters:
   - BaseColor (Texture2D)
   - NormalMap (Texture2D)
   - ORMMap (Texture2D)
   - EmissiveColor (Vector3)
   - EmissivePower (Scalar)
   ```

2. **Material functions for reuse**
   - Create functions for common operations
   - Detail texturing, weathering, edge highlighting
   - Parallax effects for added depth

3. **Layer-specific master materials**
   ```
   M_Background_Master - Optimized for distant objects
   M_Gameplay_Master - Full featured for gameplay objects
   M_Foreground_Master - Support for translucency
   ```

### Material Instance Workflow

Use material instances for efficiency:

1. **Setting up the hierarchy**
   ```
   Master Material
   ├── Location Material Instance (e.g., MI_Forest)
   │   └── Specific Asset Instance (e.g., MI_Forest_Tree_01)
   ```

2. **Parameter organization**
   - Group parameters by type
   - Use clear naming conventions
   - Set appropriate parameter defaults

3. **Instance advantages**
   - Much faster iteration time
   - Lower memory footprint
   - Consistent changes across assets

### 2.5D-Specific Material Techniques

Techniques particularly useful for 2.5D games:

1. **Depth-based effects**
   - Create materials that fade based on distance
   - Add fog to distant layers
   - Desaturate far background elements

2. **Edge highlighting for platforms**
   ```
   // Example node setup for edge highlight
   Fresnel (Exponent=3) → Multiply (Color) → Add to Emissive
   ```

3. **Parallax materials for added depth**
   - Use parallax occlusion mapping for details
   - Apply to brick walls, stone surfaces
   - Creates depth without extra geometry

### Performance Optimization

For smooth performance in 2.5D games:

1. **Material instancing**
   - Use instances whenever possible
   - Share textures between similar materials
   - Batch similar materials together

2. **Texture atlasing**
   - Combine multiple textures into atlases
   - Reduces draw calls and material swaps
   - Good for environment sets (rocks, plants)

3. **LOD-specific materials**
   - Simpler materials for distant objects
   - Fewer texture samples for background
   - Consider dropping normal maps for far background

## Example Material Setups for Common Assets

### Character Materials

```
BaseColor: 2048×2048 (PNG)
Normal: 2048×2048 (TGA)
ORM: 2048×2048 (PNG)
Emissive: 1024×1024 (PNG, optional)

Material: Instance of M_Character_Master
- Enable two-sided for hair, cloth elements
- Set appropriate roughness values (skin ~0.3, fabric ~0.7)
```

### Platform/Environment Materials

```
BaseColor: 1024×1024 (PNG)
Normal: 1024×1024 (TGA)
ORM: 1024×1024 (PNG)

Material: Instance of M_Environment_Master
- Use vertex colors for blending variations
- Add edge highlighting for gameplay elements
- Set up different physics materials based on surface
```

### Background Materials

```
BaseColor: 1024×1024 or 512×512 (PNG)
Normal: Optional, 512×512 (TGA) if needed
Combined: 512×512 (PNG)

Material: Instance of M_Background_Master
- Simplify where possible
- Add depth fading
- Consider using imposter techniques for complex elements
```

## Importing Materials to Unreal

### Texture Import Settings

Configure appropriate import settings:

1. **Base Color maps**
   ```
   Compression Settings: Default (DXT1/BC1)
   sRGB: True
   MipGen Settings: Sharpen 0.2
   ```

2. **Normal maps**
   ```
   Compression Settings: Normalmap (BC5)
   sRGB: False
   MipGen Settings: Sharpen 0
   ```

3. **ORM maps (packed textures)**
   ```
   Compression Settings: Masks (BC7)
   sRGB: False
   MipGen Settings: Sharpen 0
   ```

### Material Assignment Workflow

1. **Importing FBX with materials**
   - Enable "Import Materials" and "Import Textures"
   - Choose "Create New Materials" for first import
   - Choose "Reuse Existing" for updates

2. **Post-import material setup**
   - Convert automatically created materials to instances
   - Apply master material parameters
   - Organize in proper folder structure

## Texture Optimization Techniques

### Texture Atlasing

1. **When to use atlases**
   - Similar objects that appear together
   - Environment sets (rocks, plants, debris)
   - Modular architecture pieces

2. **Atlas creation in Blender**
   - Create UV layouts that use portions of a shared texture
   - Pack UVs efficiently
   - Maintain consistent texel density

3. **Atlas handling in Unreal**
   - Use UV coordinates to sample from atlas
   - Share single material instance across objects
   - Reduces draw calls significantly

### Texture Compression

Appropriate compression settings save memory:

1. **Color textures (BaseColor)**
   - DXT1/BC1 for opaque
   - DXT5/BC3 for transparency
   - Consider DXT1/BC1 with separate opacity mask

2. **Normal maps**
   - BC5 (two-channel compression)
   - Preserves XY channels with high quality
   - Z is reconstructed mathematically

3. **Technical maps (ORM)**
   - BC7 for highest quality
   - BC4 if only using single channel

## Testing and Troubleshooting

### Material Testing Process

Validate your materials properly:

1. **Lighting tests**
   - Check materials under different lighting conditions
   - Test day/night transitions if applicable
   - Verify specular response looks appropriate

2. **Performance testing**
   - Monitor frame rate with profiling tools
   - Check GPU time spent on materials
   - Look for overdraw in complex scenes

### Common Material Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Normal map appears flat | Wrong texture settings | Set Compression to Normalmap, sRGB to False |
| Materials too shiny | Incorrect roughness value | Adjust roughness in material instance |
| Texture blurry up close | Insufficient resolution | Increase texture size or use detail textures |
| Materials drain performance | Too many unique materials | Convert to instances, reduce texture count |
| Z-fighting on surfaces | Overlapping geometry | Adjust Z-position slightly |
| Weird shading on models | Incorrect vertex normals | Recalculate normals in Blender |

## Communication with Blender Artists

When working with artists:

1. **Provide material examples**
   - Share screenshots of working materials
   - Create material reference sheets with parameters
   - Set clear expectations for texture sizes

2. **Establish texture naming conventions**
   ```
   [AssetName]_BC.png - Base Color
   [AssetName]_N.tga - Normal Map
   [AssetName]_ORM.png - Occlusion/Roughness/Metallic
   ```

3. **Define material standards document**
   - List required/optional maps
   - Specify texture resolutions by asset type
   - Provide target performance metrics

## Conclusion

A well-organized material pipeline between Blender and Unreal is essential for visual consistency and performance. By establishing clear standards and using material instances effectively, you can create a flexible system that allows for creative freedom while maintaining technical requirements.

Remember that for 2.5D platformers, material clarity and performance are often more important than extreme realism. Focus your detail where it matters - on gameplay elements that the player will see up close and interact with directly. 