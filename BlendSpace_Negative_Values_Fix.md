# Unreal Engine Blend Space Negative Values Fix
## Forward Animation Working, Backward Animation Not Working

This is a common issue in Unreal Engine where the forward animation works correctly in a blend space, but the backward animation (negative values) doesn't function properly. Here are the most effective solutions:

## Root Causes
The issue typically stems from:
1. **Direction Value Flipping**: Values flip between -179.9° and 179.9° causing stuttering
2. **Orient Rotation to Movement**: Character movement component conflicts
3. **Interpolation Issues**: Blend space trying to interpolate the "long way" around
4. **Root Motion Conflicts**: Animation root motion fighting with character movement

## Solutions (Try in Order)

### 1. **Orient Rotation to Movement Fix** (Most Common Solution)
**Location**: Character Blueprint → Components → CharacterMovement (Inherited) → Details Panel

**Steps**:
1. Open your Character Blueprint
2. In the Components panel, select `CharacterMovement (Inherited)`
3. In the Details panel, find `Orient Rotation to Movement`
4. **Uncheck** this option

**Why this works**: This setting causes conflicts between the character movement component and the animation blend space direction calculations.

### 2. **Fix Direction Calculation in Animation Blueprint**
**Problem**: Direction values flip between -179° and 179° when walking backward

**Solution**: Modify the direction calculation logic:
```blueprint
// Instead of the default Epic calculation:
// If Direction > 180, subtract 360

// Use this modified approach:
// If Direction > 179, subtract 360
// If Direction < -179, add 360
```

**Implementation**:
- Open your Animation Blueprint
- Find the Direction calculation nodes
- Modify the comparison values from 180 to 179
- This prevents the constant flipping between negative and positive values

### 3. **Remove Direction Axis Interpolation**
**Location**: Blend Space → Sample Interpolation Settings

**Steps**:
1. Open your Blend Space
2. Go to Sample Interpolation settings
3. Set **Target Weight Interpolation Speed** for Direction axis to **0**
4. This removes interpolation that causes the "compass-like" behavior

### 4. **Use 2D Blend Space with Idle Bridge**
**For stubborn cases**:
1. Create a 2D Blend Space with Direction and Speed axes
2. Set Speed 0 with all directions pointing to your idle animation
3. This creates a "bridge" path: backward (-179°) → idle (0 speed) → forward (179°)
4. Set both Direction and Speed interpolation to 0 for instant transitions

### 5. **Root Motion Settings Check**
**Location**: Animation → Root Motion

**Steps**:
1. Open your animations in the Animation Editor
2. Check Root Motion settings in the Asset Details
3. Ensure consistent Root Motion settings across all animations
4. Try setting Root Motion to "Root Motion from Everything" or disable it entirely

### 6. **Character Movement Component Settings**
**Additional settings to check**:
- `Use Controller Rotation Yaw`: Usually should be **false** for third-person
- `Orient Movement to Rotation`: Set this in Event Graph if needed
- `Rotation Rate`: Adjust if character turns too slowly/quickly

## Debugging Steps

### 1. **Print Direction Values**
Add a Print String node to display the Direction value in real-time:
```blueprint
Direction → Print String
```
Look for values constantly flipping between -179 and 179.

### 2. **Check Blend Space Setup**
- Ensure your backward animation is placed at 180° (or -180°)
- Verify the blend space axis range is set to -180° to 180°
- Check that sample points don't overlap

### 3. **Animation Import Settings**
If animations were rotated during import:
- Try importing with -360° rotation instead of 180°
- Ensure all animations face the same direction before import
- Check that root bone rotations are consistent

## UE5 Specific Considerations

In Unreal Engine 5, blend spaces may automatically normalize values:
- If setting -180° to 180° gets normalized to -1 to 1, work with the normalized values
- The behavior is the same, just with different number ranges

## Quick Test
1. Uncheck "Orient Rotation to Movement"
2. Set Direction interpolation to 0
3. If still not working, implement the direction calculation fix

## Prevention for Future Projects
- Always use consistent animation facing directions
- Set up blend spaces before implementing character movement
- Test backward movement early in development
- Document your specific solution for team reference

Most users report success with the "Orient Rotation to Movement" fix alone. If that doesn't work, combine it with the interpolation removal for best results.