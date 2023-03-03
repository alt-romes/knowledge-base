#ghengin #core 


Ghengin is completely defined by its Core, and other Ghengin features such as built-in Camera support and directional lights (eventually) are implemented on top of Core, and could have just as easily be defined and created by the engine user.

At the Core of every game in Ghengin are `RenderPacket`s (naturally, `RenderPacket`s are a concept that defines Ghengin Core)

A render packet is defined by three further fundamental concepts
* A mesh
* A material
* And a render pipeline

A render pipeline is defined by a <u><strong>shader</strong></u> and a set of <u><strong>render properties</strong></u>
A material is defined by a set of <u><strong>render properties</strong></u>
A mesh is defined by its <u><strong>vertices</strong></u> and a set of <u><strong>render properties</strong></u>

The draw function at the Core of Ghengin works roughly as:

```
∀ pipeline ∈ render pipelines:
	bind pipeline render properties at descriptor set #0
	∀ material ∈ materials assigned to pipeline:
		 bind material render properties at descriptor set #1
		 ∀ mesh that uses material:
			 bind mesh render properties at descriptor set #2
			 draw render packet (that is constituted by this mesh, material, pipeline)
```

A render property can be either one of
* A dynamic binding, which gets written to a uniform buffer every frame
* A static binding, which is only written when manually updated, and whose value is bound to a uniform buffer to make it available every frame (note that the buffer is not written every frame, only *used*)
* A texture property, which is closely related to shader's Texture2Ds

***

Core by having meshes, materials, render pipelines *and* shaders be described in the type system, can enforce that when a render packet is created, its consitutients (the meshes, materials, and pipelines (and the shaders in the pipeline)) are *compatible*. That is:
* The pipeline render properties are compatible with the shader descriptor set #0 bindings
* The material render properties are compatible with the shader descriptor set #1 bindings
* The mesh render properties are compatible with the shader descriptor set #2 bindings
* The mesh vertices are compatible with the shader inputs


