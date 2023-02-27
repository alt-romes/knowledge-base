#ghengin 

Can a transform be just a mesh property, and a scene graph just a function called in the update function to traverse the meshes and update those properties? But the transform shouldn't be associated to the mesh, but rather to the render packet. So perhaps, meshes shouldn't be global and render packets shouldn't reference meshes? How can we have the more general case?

Previously we were iterating over all render packets every frame and constructing the render queue from that. Now, we wouldn't have to because the render queue is created once and modified directly every time an edit or addition needs to be done?

A quite relevant resource on rendering queues is https://realtimecollisiondetection.net/blog/?p=86 which was used on God of War 3 and Heavenly Sword for PS3

The following quote hints to a solution similar to the one I'm proposing, in which we have predefined buckets into which we insert the specific draw calls. We're currently not using the depth sort nor do we need any flexible bucketing system yet, so we might have this solution for now.

> In the past, however, we had slightly different criteria and used a different solution. On the PS2 we were not willing to sacrifice speed for flexibility (and the needs for flexibility were lesser too) so we simply had predefined buckets for all the different combinations of layers, viewports, translucency, etc. and just inserted draw calls directly into the appropriate bin. On drawing, we would just process each bucket in whichever order we wanted to draw them, no sorting necessary! Obviously, this system is much faster with no sorting required, but it is also much less flexible, and doesn’t deal well with e.g. including depth sorting as part of the bucketing. (A very similar bucketing system was used on the PS1 as well.)

Great rendering resources:
* https://realtimecollisiondetection.net/blog/?p=55
* http://real.mtak.hu/28740/1/szecsi_gg14_statechange.pdf

> The objective is to determine how and when to upload or   bind data to the GPU in order to minimize CPU-GPU data   transfer and communication latency. Several entities need to   be rendered using the same resources, shader programs and   state blocks, and thus, re-binding—or even re-uploading—   these could needlessly stall operation. For simple scenar-   ios, this could be addressed manually in the content creation   phase, using a material system that orders materials so that   similar ones follow each other. Then, entities can be sorted   by material ID in a render queue, in run-time, before ren-   dering. However, content-creation-phase ordering might not   be optimal for the actual set of entities that are present at   run-time. Furthermore, if rendering behavior is not merely   defined by a material, but by an ever extendable set of com-   ponents, then the render queue approach is insufficient. For   example, a future gameplay decision may dictate that the   player is given a gun that changes the texture of entities fired   upon. This can easily be solved by creating a new render   component class that imposes a texture setting when visited,   overloading the material default. However, this could break   the material system and render queue ordering, calling for   a custom workaround to be implemented. In order to avoid   development overheads of this kind, we aim to determine the   optimal render ordering at run-time.

A recent engine that claims to have a GPU-driven renderer: https://www.ambient.run/
> **Powerful renderer**: The Ambient renderer is GPU-driven, with both culling and level-of-detail switching being handled entirely by the GPU. By default, it uses [PBR](https://en.wikipedia.org/wiki/Physically_based_rendering). It also supports cascading shadow maps and instances everything that can be instanced.

Resources on GPU-driven rendering:
* https://on-demand.gputechconf.com/gtc/2013/presentations/S3032-Advanced-Scenegraph-Rendering-Pipeline.pdf
* http://advances.realtimerendering.com/s2017/2017_Sig_Improved_Culling_final.pdf
* http://advances.realtimerendering.com/s2022/index.html
* https://www.gdcvault.com/play/1023109/Optimizing-the-Graphics-Pipeline-With
* https://www.advances.realtimerendering.com/s2015/aaltonenhaar_siggraph2015_combined_final_footer_220dpi.pdf
* https://www.gdcvault.com/play/1022990/Rendering-Rainbow-Six-Siege
* https://advances.realtimerendering.com/s2020/RenderingDoomEternal.pdf