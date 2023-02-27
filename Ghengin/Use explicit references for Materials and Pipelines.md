#ghengin 

Commit: da91a3a44b8a7f687efe9d2ee20feac51aa90a27

Huge: Use explicit references for Materials and Pipelines

Previously, every render packet had its own Material and Pipeline. We
checked which materials were the same based on their unique ids and
identified them that way. To update a material, we would update the
properties of the material for each individual render packet.

Now, render packets have references to Materials and Pipelines that are
stored in with existential wrappers in "singleton entities" in the
entity component system. In this context, a singleton entity is an
entity that solely has that component, and the only entity with that
component is that exact one (similarly to singletons in dependent
types). An update to a material will be reflected on all packets that
reference that material.

This importantly unlocks the ability to run queries on "all render pipelines",
by querying, well, all render pipelines, which are now stored as
components in the entity component system -- rendering the query a
simple component query like all others.

This means we can implement logic for things like cameras lights and
timers outside of the Core engine by updating every frame, for all
(relevant) render pipelines, the camera properties, or the light
properties, which are only really render properties that are set once
per draw frame.
