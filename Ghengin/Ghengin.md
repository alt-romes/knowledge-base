We create a singleton entity (in the style of singleton types -- one entity per component) for each render pipeline, material, and mesh.
Then, each render packet, stores only references to the singleton entities (one render pipeline reference, one material reference, and one mesh reference)

This way, we can edit a material and have the changes be reflected on all render packets using that material. Similarly, we can edit all render pipelines in a scene (and update, e.g., their camera render property) without requiring any built-in logic to update render pipelines or built-in camera logic.

It all boils down to render properties in pipelines and entities with these render pipelines (and similarly for material properties and materials, and mesh properties and meshes)