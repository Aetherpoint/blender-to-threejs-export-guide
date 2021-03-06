# Blender to three.js export guide
📋 _This is a living document. The landscape for this topic is constantly changing and so this document will need to evolve with it. If anyone has anything to add/edit/remove, please create a pull request or start a discussion with an issue._

## It's best to export to glTF
glTF is the open standard for 3D models on the web. It is the format recommended in the three.js documentation. The three loaders for other formats (e.g OBJ, FBX) are not well maintained and it is likely many of them will end up very buggy or completely broken. This guide only explains how to export to glTF.

## Test with the glTF viewer
After exporting, it's best to test with the [glTF Viewer](https://gltf-viewer.donmccurdy.com/) by Dom McCurdy. This is a quick and easy way to check your model. If you test with your own code, there's a chance that any problems you experience are from your code and not the model.

## Which Blender version?
For the best results, although possibly unstable, use the very latest version of Blender (2.81 as of writing). Find it on the [daily builds](https://builder.blender.org/download/) page of Blender

### Armatures / Bones
- Don't try and export multiple armatures. Work with one armature per glTF file.
- Inverse Kinematics work, but animations will need to be "sampled" in export settings. Also, if you're using empties as targets, you'll need to make sure these are also exporting.
- Don't use bendy bones. These don't export for any of the standardised formats. One alternative could be to use normal bones and [Spline IK](https://docs.blender.org/manual/en/dev/rigging/constraints/tracking/spline_ik.html).
- Parenting objects outside of the mesh with a bone is buggy (e.g. weapons, hats). Feels like there might be a way to get this working (please share if you work this out).

### Shape keys / Morph Targets
Shape keys should convert to glTF "morph targets". However, if you're using modifiers such as "mirror" or "subdivision surface", you will not be able to apply these once you've created shape keys. This is a limitation in Blender, you can't apply modifiers to objects with shape keys. This is problematic, because this will need to happen on export (via an option). Note that armature modifiers seem to export fine without any extra effort. You have a few options here:
- Apply all your modifiers before adding shape keys. This is the simplest option, but is somewhat destructive.
- Keep your modifiers and do a [manual workaround](https://blender.stackexchange.com/questions/56795/shape-keys-and-applying-subdivision-surface-modifier) to apply modifiers. This could be quite time consuming.
- If you're using Blender 2.7, use the [Apply modifier to object with shape keys](https://github.com/przemir/ApplyModifierForObjectWithShapeKeys) addon. Note that using Blender 2.7 means you'll lose textures when exporting.
- Convert the above plugin to be compatible with Blender 2.8 (If you do this, please share on this repo via an issue or PR!!!)


### Exporting animations (NLA Editor tips)
If you want separate animations, you'll need to save them as "actions" in Blender and then make sure they are in the NLA editor. This part of the software is quite unintuitive, especially when it comes to exporting, it's highly advised to read through the Blender manual on the [NLA Editor](https://docs.blender.org/manual/en/latest/editors/nla/introduction.html). 

The below advice may still be useful for some users, so I'm keeping it in for now. However **there is a much simpler way to get your actions exporting**. Just make sure your action is "stashed", and it should export, no need to worry about the NLA Editor. Keep "NLA Strips" checked in the settings as this is still needed. Name your actions rather than the NLA strips.

Some relevant tips below:
- Add new actions with the "action editor". You add keyframes as you would with the normal timeline. These apply to one object each.
- Shape keys can also be animated here, under "Shape key editor"
- When you're happy with an action, make sure it's "pushed down" into the NLA editor. Actions that are pushed down are no longer related to the actions you can choose from in the action editor. If you change something in the library of actions, it won't affect the strips in the NLA editor.
- If you want to edit an action that has been pushed down (a strip), select the strip and press tab. It goes green (this is known as tweak mode) and your edits will be saved to that strip if you press tab again.
- Actions on different places on the timeline will be exported as seperate animation clips. Actions above and below each other, layered on the same place on the timeline, will be combined and exported as a single animation clip. Unfortunately this layering only seems to work for two actions at a time, having three will just export as two actions (Action 1 combined with 2, Action 2 combined with 3).
- Shape keys won't export as an action unless there is some other action layered with them, you can create an empty action (e.g. add some keyframes to an empty) to make them appear
- Make sure each action is set to "Nothing" under "Extrapolation" in the properties panel (sub group "Active Strip"). This prevents the last frame of one strip affecting strips further along the timeline. To see the properties panel, make sure you have a strip selected in the NLA editor and press "N". This prevents the strips from inteferring with each other further down the timeline.
- When animating an object, make sure its transforms are all set to zero. To do this, you need to "Apply" them. Press CTRL+A to do this when selecting the object in the 3D view. If you've done it correctly, you should see Location and Rotation set, scale set to 1 (view in the properties panel). This makes sure the object doesn't move to weird places when it is not being animated as part of a strip.
- When switching between actions, make sure you have the correct object selected in the 3D View, otherwise the wrong object will suddenly have the animation applied to it

### A note on mixing actions once in three.js
Because of the way actions are exported, blending animations doesn't work too well for anything other than transitions. Let's say you have a figure running, and also a figure standing and waving. In an ideal world, you'd be able to blend these two in your application and have a figure running and waving. Unfortunately, instead you'll have a figure halfway between the two states (e.g. walking and half waving). This is because Blender exports all static information as well as important stuff.

This even applies to animations with multiple meshes. For instance, if you had a character with a hat as a seperate object and you wanted a separate hat wobble animation, you won't be able to blend that animation with something else. One option here is export each mesh seperately and combine them in three.js.

This also applies to shape keys in animations. While they blend perfectly well as separate morph targets, when shape keys are part of an animation they blend in a similar way as explained above.

#### This mixing problem can be fixed with code!

Thanks to a [code snippet from CasualKyle](https://discourse.threejs.org/t/creating-a-dynamic-run-walk-animation/442/23) on the three.js forum, it is possible to strip out redundant animation data from each clip, so they blend properly (e.g running and waving). Unfortunately this won't work for two animations blending morph target information, but does work for bones and other objects.

```javascript
const loader = new THREE.GLTFLoader()

loader.load(url, (gltf) => {
  const scene = gltf.scene || gltf.scenes[ 0 ]
  const clips = gltf.animations || []

  const getInc = (trackName, scene) => {
    let inc

    const nameParts = trackName.split('.')
    const name = nameParts[ 0 ]
    const type = nameParts[ 1 ]

    switch (type) {
      case 'morphTargetInfluences':
        const mesh = scene.getObjectByName(name)
        inc = mesh.morphTargetInfluences.length
        break

      case 'quaternion':
        inc = 4
        break

      default:
        inc = 3
    }

    return inc
  }

  clips.forEach(function (clip) {
    for (let t = clip.tracks.length - 1; t >= 0; t--) {
      const track = clip.tracks[ t ]
      let isStatic = true

      const inc = getInc(track.name, scene)

      for (let i = 0; i < track.values.length - inc; i += inc) {
        for (let j = 0; j < inc; j++) {
          if (Math.abs(track.values[ i + j ] - track.values[ i + j + inc ]) > 0.000001) {
            isStatic = false
            break
          }
        }

        if (!isStatic) { break }
      }

      if (isStatic) {
        clip.tracks.splice(t, 1)
      }
    }
  })
})
```

_Please note this code hasn't been well tested._

## Important FBX export options
As mentioned above, for complex models, the best option is to export to FBX and then convert to glTF using [FBX2glTF](https://github.com/facebookincubator/FBX2glTF). Below are the important options for exporting to FBX from Blender.

### Main 
- Version: FBX 7.4 binary
- Exported features: "Empties", "Armature", "Mesh"

### Geometries 
- Apply Modifiers

### Animation
- Baked Animation
- NLA Strips (Not "All Actions". See NLA Editor tips above for explanation)
- Force Start/End Keying (Enable this if you have any single keyframe animations)

## Troubleshooting

### Parts of my mesh are missing when viewing in the browser. glTF error message "Accessor element at index 0 is NaN or Infinity."
This may be because you've added new parts to a mesh that is not associated with your armature. Try weight painting some of this mesh, or use the "weights > assign autmoatic from bones" in weight painting mode, with bone(s) selected.

### My object/bone position/rotation/scale is always exporting as 0
For some reason, if you don't have any values changing for an object from frame to frame, it will be set to 0. A simple fix for this is to make sure that there is some non-zero value change for the position/rotation/scale in your animation. Not sure why this is happening or at what point during the export process.

### My armature is scaled differently from the mesh. If I try to apply scale to the armature, it breaks the animation
This is an issue when importing from Mixamo. Below is a Blender python script that might help. Select your armature and then run the script.

```python
import bpy
for a in bpy.data.actions:
    for fc in a.fcurves:
        if "location" in fc.data_path:
            for kp in fc.keyframe_points:
                kp.co.y *= 0.01

ob = bpy.context.object
for pb in ob.pose.bones:
    pb.location = (0, 0, 0)

bpy.ops.object.transform_apply(scale=True)
```

The above script was modified from a [stack exchange answe](
https://blender.stackexchange.com/questions/143196/apply-scale-to-armature-works-on-rest-position-but-breaks-poses). More info there!

## Useful links
- [All exporters/converters and their features](https://github.com/KhronosGroup/glTF/issues/1271)
