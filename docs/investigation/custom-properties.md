# Custom Properties Investigation

Date: 2025-11-06

## Question

Does the Blender addon code create custom properties on Blender objects?

## Answer

- `scripts/addons/animaide/__init__.py` registers `bpy.types.Scene.animaide = PointerProperty(type=AnimAideScene)`, so the add-on stores its state on the Scene datablock rather than per object.
- No modules in the add-on write ID properties on objects (e.g., no assignments like `obj["..."] = ...` or pointer properties on `bpy.types.Object`), and operations only read existing custom properties when present.

## Conclusion

The add-on extends the Scene with its own property group but does not create new custom properties on individual Blender objects.
