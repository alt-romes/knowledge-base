For transitioning to the new singleton render entities for meshes, materials, and pipelines, we can create the new entities, have the render packets keep the references, and keep the render queue *without changes*.

The render queue, currently, is created by going over all render packets and determining a somewhat optimal draw graph. We can keep doing it the exact same way, constructing the queue from the render packets, but we'll construct a queue containing references to the singleton entity, instead of a queue of the components such as materials directly. Then, we change the traverse render queue to pass the reference to the singleton entity as an argument instead of the MMP (mesh material pipeline), and fetch the corresponding MMP from the entity in order to render it, in the render function.

We can keep the current render queue logic but move to the singleton entities for MMPs idea!

