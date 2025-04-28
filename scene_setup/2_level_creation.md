# Basic Level Geometry Creation

This document provides a step-by-step guide for creating the basic level geometry for our 2.5D platformer test environment in Unreal Engine 5.5.

## Overview

The test level serves as a controlled environment for developing and testing our component-based character system. We'll build a simple but effective layout with various platforms, ramps, and obstacles to test different movement mechanics.

## Creating a New Level

1. Launch Unreal Engine 5.5 and open your project
2. Go to **File → New Level**
3. Select **Empty Level** to start with a clean slate
4. Save the level with an appropriate name:
   - Go to **File → Save As...**
   - Navigate to `Content/Levels/`
   - Name it `TestLevel_01`

## Setting Up the World Grid

A proper grid setup is crucial for precise platform placement:

1. Open the **World Settings** panel:
   - From the toolbar, select **Settings → World Settings**
   - Or press **Shift+F8**

2. Configure the grid settings:
   - In the viewport, look for the dropdown arrow next to the "View Options" button in the top-right corner
   - Click it and select "Show → Grid" option to ensure the grid is visible
   - Go to **Edit → Editor Preferences**
   - In the search bar, type "Grid"
   - Under "Level Editor → Viewports → Viewport Interaction" you'll find all grid settings:
     - Set "Decimal Grid Sizes" to 16 units for precision
     - Enable "Editor Snap to Surface" and "Editor Snap Location" for accurate placement

## Creating the Floor Plane

1. Create a floor plane using the Geometry/BSP tools:
   - In the **Place Actors** panel (press **Ctrl+Shift+1** to open)
   - In the search bar, type "Cube" 
   - Drag a **Cube** actor into your level
   - With the cube selected, in the **Details** panel:
     - Set **Scale**: X=20.48, Y=20.48, Z=0.16 (creates a 2048×2048×16 cube)
     - Set **Location**: X=0, Y=0, Z=0
   - Right-click on the cube in the **Outliner** and select **Rename**
   - Name it "SM_Floor_01"

   Alternative method (for precise dimensions):
   - In the **Place Actors** panel, search for and add a **Cube (Static Mesh)** 
   - Right-click on it in the **Content Browser** and select **Asset Actions → Create Copy**
   - Name the copy "SM_Floor_01" and save it in Content/Meshes/Platforms
   - Open the Static Mesh Editor and use the modeling tools to set exact dimensions
   - Place this custom mesh in your level

2. Apply a grid material:
   - Create a new material named `M_GridFloor`
   - Open the material editor and implement a two-color grid with these steps:

```
// Creating a two-color grid in Material Editor:
// This will create distinct colors for horizontal and vertical grid lines

1. Add a "TextureCoordinate" node:
   - Right-click in empty space and search for "TextureCoordinate"
   - In the node settings, set UTiling and VTiling to 1.0
   
2. Create the horizontal grid lines (first color):
   - Add a "Component Mask" node:
     - Right-click and search for "Component Mask"
     - Connect the TextureCoordinate output to this node's input
     - In the node properties, check ONLY the R box (this isolates the U/horizontal component)
   - Add a "Multiply" node:
     - Connect the Component Mask output to input A
     - Set input B to 64 (matches grid size of 16 units × 4 for clear visibility)
   - Add a "Frac" node:
     - Connect the Multiply output to the Frac input
   - Add a "Step" node:
     - Right-click and search for "Step" (under Math)
     - Connect the Frac output to the X input
     - Set the Y input (threshold) to 0.95 (controls line thickness)
   
3. Create the vertical grid lines (second color):
   - Repeat the same process for vertical lines:
   - Add another "Component Mask" node:
     - Connect the TextureCoordinate output to this node
     - In node properties, check ONLY the G box (this isolates the V/vertical component)
   - Add another "Multiply" node (set to 10)
   - Add another "Frac" node
   - Add another "Step" node (threshold 0.95)

4. Add colors for each grid direction:
   - For horizontal lines:
     - Add a "Constant4Vector" node (first color)
     - Set it to green (0.0, 0.5, 0.0, 1.0)
     - Add a "Multiply" node
     - Connect the horizontal "Step" output to input A
     - Connect the green color node to input B
   - For vertical lines:
     - Add a "Constant4Vector" node (second color) 
     - Set it to red (0.5, 0.0, 0.0, 1.0)
     - Add a "Multiply" node
     - Connect the vertical "Step" output to input A
     - Connect the red color node to input B

5. Combine both colored grids:
   - Add an "Add" node
   - Connect both color multiplier outputs to this node

6. Connect the final output:
   - Connect the Add node output to the "Base Color" input on the Material Output node
   - Click "Apply" and "Save" in the toolbar

7. Verify the material in the preview:
   - You should see a black background with green horizontal lines and red vertical lines
   - If you want different colors, adjust the Constant4Vector node values
```

4. Apply the material to the floor plane:
   - Select the floor plane (SM_Floor_01) in the level
   - In the **Details** panel, find the **Materials** section
   - Click the dropdown arrow next to "Element 0" 
   - Select the M_GridFloor material you created
   - The grid pattern should now appear on your floor plane
   
   Alternative method:
   - In the **Content Browser**, locate your M_GridFloor material
   - Drag and drop it directly onto the floor plane in the viewport
   - This will automatically apply it to the default material slot

## Creating Test Platforms

Now let's create various platforms to test different movement mechanics:

### 1. Basic Jumping Platforms

Create a series of platforms at increasing heights to test basic jumping:

1. Create multiple cube actors for platforms:
   - Follow the same process used for the floor plane
   - For each platform, set these dimensions:
     - Scale: X=2.56, Y=2.56, Z=0.16 (creates 256×256×16 platforms)

2. Position them at increasing heights:
   - Platform 1: (512, 0, 64)
   - Platform 2: (768, 0, 128)
   - Platform 3: (1024, 0, 192)

3. Create a new material named `M_GridPlatform`:
   - Follow the same grid pattern as the floor
   - Use a blue tint (0.0, 0.2, 0.8) instead of green

4. Apply the platform material to each platform mesh

### 2. Wall Jumping Test Area

Create a wall jumping test area with parallel walls:

1. Create two tall cube actors for walls:
   - Follow the same process as before
   - Set these dimensions:
     - Scale: X=0.16, Y=2.56, Z=3.84 (creates 16×256×384 walls)

2. Position them parallel to each other:
   - Wall 1: (-512, -128, 0)
   - Wall 2: (-512, 128, 0)

3. Create a new material named `M_GridWall`:
   - Follow the same grid pattern as before
   - Use a red tint (0.8, 0.2, 0.2)

4. Apply the wall material to both wall meshes

### 3. Ramps and Slopes

Create ramps to test movement on inclines using Blueprints:

1. Create a Blueprint for a test ramp:
   - In the **Content Browser**, right-click and select **Blueprint Class**
   - In the **Pick Parent Class** window, search for and select **Actor**
   - Name it "BP_TestRamp" and save it in Content/Blueprints/Environment
   - Double-click to open the Blueprint Editor

2. Add components in the Blueprint:
   - In the **Components** panel, click the **Add Component** button
   - Search for and add a **Static Mesh Component**
   - With the Static Mesh Component selected in the Components panel:
     - In the **Details** panel, find the **Static Mesh** property
     - Click the dropdown and select a basic wedge shape (or create one in the Static Mesh Editor)
     - Set the **Scale** to X=5.12, Y=2.56, Z=1.28 (creates a 512×256×128 ramp)
   
3. Add material control to the Blueprint:
   - In the **Blueprint Editor**, go to the **Class Defaults** tab 
   - In the **Variables** category, click the **+** button to add a new variable
   - Name it "RampMaterial"
   - Set its type to "Material" (from the dropdown)
   - Check "Editable" and "Expose on Spawn" to make it adjustable in the level editor
   - Add a Description: "Material to apply to the ramp"
   
4. Create a Construction Script to apply the material:
   - In the Blueprint Editor, navigate to the **Graph** tab at the top
   - Make sure "Construction Script" is selected in the dropdown next to the Graph tab
   - You'll see a purple "Construction Script" node already in the graph
   - Note: This node is just a label and does NOT connect to anything directly
   
   - To get your component:
     - Right-click in empty space and search for your Static Mesh Component
     - You can either type "Get Static Mesh Component" or the specific name of your component
     - Or simply drag off the "self" reference and search for your component
   
   - Add the Set Material node:
     - Right-click in empty space and search for "Set Material"
     - Connect the output pin from your Static Mesh Component reference to the Target input of Set Material
   
   - Add your material variable:
     - In the **My Blueprint** panel on the left side, find your "RampMaterial" variable
     - Click and drag the RampMaterial variable onto the graph
     - Connect the output pin from RampMaterial to the Material input of Set Material
   
   - The Construction Script will automatically execute all nodes in the graph - no need to connect anything to the purple Construction Script node
   
   - Compile and Save:
     - Click the **Compile** button in the toolbar
     - Save the Blueprint with Ctrl+S or click the Save button

5. Place the ramps in your level:
   - Drag the BP_TestRamp Blueprint from the Content Browser into your level
   - Position them at strategic locations:
     - Ramp 1: (256, -512, 0), Rotation: (0, 0, 0)
     - Ramp 2: (512, -512, 0), Rotation: (0, 0, 90) 
     - Ramp 3: (768, -512, 0), Rotation: (0, 0, 180)
   - For each ramp instance, in the **Details** panel:
     - Find the "Ramp Material" property 
     - Set it to your grid material (can be the same as platforms or a different color)

6. Create a variation with a steeper angle (optional):
   - Duplicate the BP_TestRamp Blueprint (right-click → Duplicate)
   - Name it "BP_SteepRamp"
   - Open it and adjust the Static Mesh or add a Construction Script that:
     - Modifies the wedge's dimensions for a steeper slope
     - Adds additional collision for more precise testing
   - Place a few of these in your level for testing extreme slopes

This Blueprint approach allows you to:
- Easily place multiple ramps with consistent properties
- Change materials for all ramps at once by updating the Blueprint
- Add gameplay functionality later (like bouncy ramps or moving ramps)
- Create variations with different properties for testing

## Setting Up Lighting

Configure lighting using Unreal Engine 5.5's Lumen global illumination system:

1. Add a **Directional Light (Sun)**:
   - Open the **Place Actors** panel (press **Ctrl+Shift+1**)
   - Search for "Directional Light" and drag it into your scene
   - Position it at coordinates X=0, Y=0, Z=1000 for optimal coverage
   - Rotate it to approximately X=-90, Y=0, Z=0 for a true overhead noon position
   - Note: The scale of the directional light doesn't affect its illumination (it's an infinite light source)
   - With the light selected, in the **Details** panel:
     - Set **Intensity** to 15.0 for bright midday sunlight (adjust to taste)
     - Set **Light Color** to a pure white (#FFFFFF) for direct sunlight
     - Under the **Light** category, check **Cast Shadows**
     - Look for any option about connecting to atmosphere/sky (exact name varies by UE5.5 version)
       - This might be called "Atmosphere Sun Light", "Primary Sky Light Source", or similar
       - Enabling this option designates this light as the main sun in your scene
     - Set **Dynamic Shadow Distance MovableLight** to 10000
   - You can press **Ctrl+L** to quickly adjust the sun direction in the viewport
   - This will be your "main" directional light because:
     - It's connected to the Sky Atmosphere (most important designation)
     - It casts shadows (only one directional light should typically cast shadows)
     - It has the highest intensity value

2. Add a **Fill Light** for better depth:
   - Return to the **Place Actors** panel and add another **Directional Light**
   - Position it at coordinates X=0, Y=0, Z=1000
   - Rotate it to approximately X=-45, Y=0, Z=0 (coming at an angle)
   - In the **Details** panel:
     - Set **Intensity** to 3.0 (for noon, fill light needs to be stronger)
     - Set **Light Color** to a very slight blue tint (#E6EEFF) for subtle cool fill
     - Under the **Light** category, uncheck **Cast Shadows**
     - Set **Source Angle** to 5.0 for softer edges
   - This fill light helps soften the harsh noon shadows and provides depth
   - Without this, noon lighting can appear flat and uninteresting

3. Add a **Sky Light**:
   - Return to the **Place Actors** panel
   - Search for "Sky Light" and drag it into your scene
   - Position doesn't matter for Sky Lights as they affect the entire scene equally
   - Typically placed at origin (0,0,0) or grouped with other lighting actors for organization
   - In the **Details** panel:
     - Set **Intensity** to 1.0
     - Make sure "Mobility" is set to "Movable" (not Static)
     - If available, look for options related to real-time updates or Lumen integration

4. Add a **Sky Atmosphere**:
   - From the **Place Actors** panel, search for "Sky Atmosphere"
   - Drag it into your scene
   - The Sky Atmosphere should automatically connect to your Directional Light
   - Keep default settings for initial testing
   - If your sky appears black, ensure Lumen is enabled (see step 5)

5. Configure **Lumen** in Project Settings:
   - Go to **Edit → Project Settings**
   - Navigate to **Engine → Rendering → Global Illumination**
   - Ensure **Dynamic Global Illumination Method** is set to **Lumen**
   - Look for Lumen quality settings and set to Medium for development

6. Add **Exponential Height Fog** (optional but recommended):
   - From the **Place Actors** panel, search for "Exponential Height Fog"
   - Drag it into your scene
   - In the **Details** panel:
     - Adjust **Fog Density** to around 0.02 for subtle atmospheric depth
     - Set **Start Distance** to 1000 to keep nearby objects clear
     - Enable **Volumetric Fog** for enhanced atmosphere

7. **Post-Process Volume** for final adjustments:
   - From the **Place Actors** panel, search for "Post Process Volume"
   - Drag it into your scene
   - In the **Details** panel, check **Infinite Extent (Unbound)**
   - Under **Exposure**:
     - Set **Method** to **Auto Exposure (Histogram)**
     - Adjust **Exposure Compensation** to fine-tune brightness
   - Under **Bloom**:
     - Set **Intensity** around 0.3-0.5 for a subtle glow
   - **Placement Note**: Position this volume at the center of your level (around X=0, Y=0, Z=0) for consistent effect application

**Important Notes for UE5.5:**
- If your sky appears black, make sure your Directional Light is properly connected to the Sky Atmosphere
- Lumen requires the Deferred Rendering Path (not Forward)
- For best results, keep your Directional Light, Sky Light and Sky Atmosphere all in the default layer
- UI options may vary slightly between different UE5.5 versions - look for similar options with slightly different names
- In World Partition maps, make sure these actors are set to "Always Loaded" in their World Partition settings

## Refining Your Lighting Setup

After setting up the basic lighting, you may need to make adjustments to achieve the ideal look for your 2.5D platformer. Here's how to address common issues and enhance your lighting:

### Adjusting Overall Brightness

If your scene is too dark or too bright:

1. **Using Post-Process (Recommended)**:
   - Adjust using the Post Process Volume's "Exposure" settings
   - Increase "Exposure Compensation" to +1.0 or +1.5 for a brighter scene
   - This is more physically accurate than using extremely bright lights

2. **Adjusting Light Intensities**:
   - For realistic lighting, use physically-based values:
     - Sun (Directional Light): 10-20 (simulates real sunlight)
     - Sky Light: 1.0-2.0
     - Fill lights: 0.5-1.5
   - Avoid extremely high values that create unrealistic results

### Creating Professional Lighting Depth

For more professional-looking lighting:

1. **Implement 3-Point Lighting**:
   - **Key Light**: Your main Directional Light (sun)
   - **Fill Light**: Add a second Directional Light with:
     - Lower intensity (0.3-0.8)
     - Slight blue tint (#B4C7E0)
     - Positioned from the opposite direction of your key light
     - "Cast Shadows" disabled
   - **Rim/Back Light**: Add a third light pointing upward for edge definition

2. **Enhance Material Response**:
   - Adjust material roughness for better light response
   - Use subtle Emissive values on key objects (0.1-0.3)
   - Enable "Clear Coat" on reflective surfaces

### Fixing Common Lighting Issues

**Black Sky Issues**:
1. Ensure your Directional Light is properly connected to the Sky Atmosphere
   - Look for any option similar to "Atmosphere/Sky Light Source" and enable it
2. Check Project Settings → Rendering → Support Sky Atmosphere is enabled
3. If still black, add a BP_SkySphere from the Content Browser
   - In its details, assign your Directional Light to it

**Too Many Shadows**:
1. Adjust Cascade Shadow Maps in your Directional Light:
   - Increase "Dynamic Shadow Distance StationaryLight"
   - Reduce "Shadow Bias" slightly
2. Increase Sky Light intensity to fill shadowed areas
3. Add small fill lights in very dark corners

**Flickering or Strange Artifacts**:
1. Ensure "Static Lighting Level Scale" is set to 1.0 in World Settings
2. Check that all objects have proper UV mapping
3. Rebuild lighting if using any static lighting components

### Performance Optimization

Balance quality and performance:

1. **For Development**:
   - Set Lumen Quality to "Medium"
   - Disable extra features like volumetric fog while working

2. **For Final Builds**:
   - Selectively enable higher quality settings
   - Bake lighting for static elements when possible
   - Use the "Scalability" settings to create presets for different hardware

### 2.5D-Specific Considerations

For the best 2.5D look:

1. **Consistent Side Lighting**:
   - Position your key light to cast shadows perpendicular to the player's movement
   - This enhances the side-scrolling depth perception

2. **Subtle Depth Enhancements**:
   - Add very subtle fog (density 0.01-0.02)
   - Use a slight depth-of-field effect for distant backgrounds
   - Consider a vignette effect (intensity 0.2-0.3) for focus

3. **Stylized Adjustments**:
   - Use Color Grading in the Post Process Volume to enhance your game's mood
   - Consider subtle chromatic aberration (0.1-0.2) for a more cinematic look

Remember that lighting is iterative - make small adjustments, test in different areas of your level, and refine until you achieve the desired atmosphere.

## Adding Reference Objects

Add reference objects to help visualize scale and distances:

1. Import a basic character mesh:
   - You can use the Mannequin from the Engine Content
   - Right-click in the Content Browser and select **Add → Add Engine Content → Mannequin** 

2. Place the mannequin at strategic locations:
   - Near jumping platforms to visualize jump heights
   - Between walls to show wall jump distance
   - At the base of ramps

3. Add measurement markers:
   - Create a simple text mesh using **Text3D** actor
   - Label key distances like "1-Unit Jump", "2-Unit Jump"
   - Position them near relevant test areas

## Setting World Bounds

Define the playable area with boundaries:

1. Create invisible wall static meshes:
   - Create thin box static meshes (16 units thick)
   - Position them around the perimeter of your level
   - Make them tall enough to prevent jumping over (512+ units)
   - Apply a semi-transparent material to them for visibility during development

2. Add a **Kill Z** height in World Settings:
   - In the **World Settings** panel
   - Set **Kill Z** to -1000 units
   - This will destroy actors that fall below this height

## Testing the Layout

Before proceeding to the next steps, test your level layout:

1. Save your level
2. In the viewport, use **Simulate** mode to navigate around
3. Check for:
   - Appropriate platform spacing for different jump heights
   - Sufficient space between walls for wall jumping
   - Clear visibility with the lighting setup
   - Proper scale compared to the character reference

## Optimizing for Performance

Apply these optimizations to ensure smooth development:

1. Configure **Nanite** for platforms:
   - Select all platform static meshes
   - In the **Details** panel, enable **Use Nanite**

2. Set appropriate **LOD** settings:
   - For non-Nanite meshes, configure LOD distances
   - Right-click meshes and select **Asset Actions → Bulk Edit via Property Matrix**
   - Adjust LOD settings as needed

3. Organize actors into folders:
   - In the **World Outliner**, right-click and select **New Folder**
   - Create folders for Platforms, Walls, Lights, and Reference Objects
   - Drag relevant actors into appropriate folders

## Next Steps

With the basic level geometry in place, you're now ready to proceed to [Camera Setup](./3_camera_setup.md) to configure the 2.5D camera system. 