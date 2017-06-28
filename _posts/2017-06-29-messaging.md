---
layout: post
title: Messaging and component communication
---

Malt is designed to be extremely modular, to the extent that the [core][malt_core] only contains fundamental features such as component management, serialization, reflection and message passing. 

Other features, such as rendering, main loop, transformations, asset management etc. are implemented in different modules. This separation decouples most of the code from other parts of the engine and keeps the code base in a more maintainable state.

This decoupling prevents modules from accessing modules that they aren't depending on. For example, transformations are implemented in `malt_basic` and it only depends on `malt_core`. Therefore, it's not possible for it to access, say `MeshRenderer` from `malt_render`. Obviously, this makes maintaining `malt_basic` easier since one only needs to know about `malt_core` and `malt_basic` to write code to it.

Another example is, `malt_render` is supposed to be able to render user defined objects as well. But it only provides a `MeshRenderer` yet. Say you want to be able to roll a quick geometry shader to tessellate spheres on the fly without using a mesh. If all `mesh_render` supported were meshes, then this would be impossible to do. Fortunately, that's not the case.

As a general design rule, specifically ones called every frame, we avoid virtual function usage in malt. Rendering is such a function and therefore it's not feasible to roll up a `Renderable` interface that you implement using runtime polymorphism and pass a pointer to the renderer.

Another way for a module to call functions on types that aren't even available at compile time is using malt messages. Simply, `malt_render` sets up a render context with each camera, and emits a `malt::render` message with each of the rendering context. So the mesh renderer's render handler looks like this:

```cpp
void mesh_renderer::Handle(malt::render, const render_ctx& ctx)
{
    // activate program
    // set variables such as model transformation and colors
    // draw using the mesh
}
```
And simply, you could roll your own custom sphere renderer like the following:

```cpp
void sphere_renderer::Handle(malt::render, const render_ctx& ctx)
{
	// activate program
	// pass variables such as radius and color
	// draw without any geometry
}
```
Then, without doing anything else, your sphere renderer will start receiving render messages each frame.

## Message syntax

