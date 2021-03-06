---
layout: post
title: How components work
---

In this post, I'll try to explain how the general entity component architecture of malt works.

For starters, creating an entity and giving it a transform looks like this in malt:

```cpp
malt::entity e = malt::create_entity("foo");
malt::transform* t = e.add_component<malt::transform>();
t->translate({0, 3, 0});
```
This snippet creates a new entity called `foo`, gives it a transform, and translates it by a vector of `(0, 3, 0)`.

Now, we might want this `foo` object to be able to render some parts of the scene, so we attach a `camera` component to it as well:

```cpp
auto cam = e.add_component<malt::camera>();
cam->set_fov(90); // field of view, in degrees
```

After this frame, `foo` will render to the main display.

So basically, `entity::add_component` creates a new component of the type given to it, attaches to the entity given and returns
a pointer to the newly created component.

From a naive object oriented perspective, such a mechanism could be implemented using inheritance and a collection of 
base component pointers:

```cpp
class entity
{
    std::vector<malt::component*> m_comps;
public:
    template <class T>
    T* add_component()
    {
        T* comp = new T;
        m_comps.push_back(comp);
        return comp;
    };
};
```

This would of course work. Other than being created, one of the things a component does is being _updated_. Every frame, 
all components that wish to be updated have their update message handler called. A message handler is any function
that has the signature of `void Handle(MessageType, ArgumentTypes...)`. An update handler looks like `void Handle(malt::update)`.

Basically, the update handler of a camera looks like this:

```cpp
void camera::Handle(malt::update)
{
    // find all renderable objects in the scene
    // render them
}
```

Such functions are usually called 60 times per second per instance of each component type. That's a lot of function calls. In the 
case of our object oriented approach, it looks like this:

```cpp

class component
{
public:
    virtual void Handle(malt::update) {}
};

void entity::update()
{
    for (auto c : m_comps)
    {
        c->Handle(malt::update{});
    }
};

int main()
{
    //...
    while (game_is_running)
    {
        for (auto& entity : all_entities)
        {
            entity.update();
        }        
    }    
}

```

First of all, specifically in game development, calling a virtual function is considered as kicking children. 
And we are calling a virtual function for every component in the game. Now that's a bad design.

The problems of calling virtual functions like we did are as follows:

1. The function to be called is determined right before you call the function, which is at least 2 pointer indirections
2. When you find the function to be called, the code for that function may not be in the I-cache
3. After the I-cache is loaded with the instructions, the data for that component may not be in the D-cache

Now here are the facts: we need polymorphic behaviour for the components, so that a camera and an AI component does
different things when updated. We need dynamically added-removed components so that we can assign some entites as cameras
and some as monsters.

Polymorphic objects stored in a vector of pointers is just one way of implementing these requirements. Let's start with
fixing second and third problems, as they are related. The scond problem arises because we call the update handlers of
components in an _interleaved_ manner:

+ Update transform of entity 1
+ Update camera of entity 1
+ Update input of entity 1
+ Update skeleton of entity 1
+ Update renderer of entity 1
+ Update transform of entity 2
+ Update AI of entity 2
+ Update skeleton of entity 2
+ Update renderer of entity 2

When we jump from the handler of a component type to another component type like this, the chances that the code pages 
for update handler of say `transform` converges to 0 as the number of component types in our game increases. Instead,
if we could follow an update pattern of something like the following, we would minimize the I-cache misses to the optimal
level:

+ Update transform of entity 1
+ Update transform of entity 2
+ Update skeleton of entity 1
+ Update skeleton of entity 2
+ So on ...

In this case, the chances of retaining the code pages for a given component is quite higher than the previous one, and is
in fact the optimal one as you can't do anything more to avoid I-cache misses. However, since we iterate over the entities
instead of the component types, it's impossible to solve this problem with our current structure.

We could keep the component vectors sorted by the type and ask each entity to update a certain type of components to improve
I-cache coherency, however this time the cost of sorting the vector and finding the first and last component to update each 
frame for each entity is overwhelming. This is apparently not going to directly work with our structure.

## Global vectors to rescue

If we could have some global vectors of component pointers for each type, and iterate over these vectors each frame, we could
solve the I-cache issue:

```cpp
std::vector<transform*> transforms;
std::vector<skeleton*> skeletons;
...
std::vector<camera*> cameras;
std::vector<renderer*> renderers;

template <class T>
T* entity::add_component()
{
    T* comp = new T;
    auto& vector = get_vector_of<T>();
    vector.push_back(comp);
    m_comps.push_back(comp);
    return comp;
}

int main()
{
    ...
    while (game_is_running)
    {
        for (each component_vector)
        {
            for (auto comp in component_vector)
            {
                comp->Handle(malt::update{});
            }
        }
    }
}
```

This works nice, however it has a couple of drawbacks from the object oriented version:

1. A vector for each component type must be maintained. Adding a new component type or removing and existing one
requires some work.
2. A function template to get a reference to the vector of a specific component type must be implemented.

These problems can be eased using template metaprogramming tricks, and the only maintenance is adding or removing the
name of the component from a list. Forgetting to add or remove the name from the list results in a compile time error
and therefore is not much of a burden. Consider the snippet in an imaginary language:

```cpp
template <class... ComponentTypes>
class component_vectors
{
    template <class T>
    using vector_t = std::vector<T*>;

    for (each TYPE in ComponentTypes)
    {
        vector_t<TYPE> m_TYPEs; // these vectors are created inside the class
    }

public:
    template <class TYPE>
    vector_t<TYPE> get_vector_of()
    {
        return m_TYPEs; // the name of the template is pasted here, like a macro
    }

    template <class FunT>
    void for_each_vector(FunT& f)
    {
        for (each TYPE in ComponentTypes)
        {
            f(m_TYPEs); // call f with a reference to m_TYPEs
        }
    }
};

component_vectors<transform, camera, renderer, ..., skeleton> vectors;
```
Using some template magic and ugly looking workarounds, something like this could be implemented in standard C++.
The implementation details would involve a lot of out of topic template metaprogramming tricks so it's left as an
exercise to the reader.

Now that we have some central vectors to store pointers to components, you might be wondering why we aren't
directly storing the components in vectors rather than allocating them in the free store and storing pointers to them:

```cpp
template <class T>
component_vectors::vector_t = std::vector<T>; // no pointer, don't mind the wrong syntax

template <class T>
T* add_component()
{
    auto& v = get_vector_of<T>();
    v.emplace_back(); // creates a new object at the back of the vector
    return &v.back();
}
```
This solves both second and third problem as now the instances of a component type will live directly in continuous
memory. However, there is a drawback to this: pointers returned by `entity::add_component` may be invalidated as the
game runs.

This is clearly a win if we could circumvent the invalidation problem. We'll get back to that later as this has
just another benefit: this approach allows us to get rid of the virtual function all together.

If you're wondering how, take a look at our main function now:

```cpp
int main()
{
    ...
    while (game_is_running)
    {
        vectors.for_each_vector([](auto& vec)
        {
            for (auto& comp : vec)
            {
                comp.Handle(malt::update{}); // this line
            }
        });
    }
}
```

The lambda we're passing works like a template. That is, when the vectors object calls this lambda,
the full type of the vector will be known by this function and the type of the objects it stores.

As the actual types of the objects are known at compile time, the compiler will be able to devirtualize the
calls and the final program will only have statically dispatched function calls. Since we are calling the
functions with actual types rather than through a base class reference/pointer, we can remove the virtual
keyword from the function completely to guarantee zero overhead.

We could go further and completely remove the update handler from the base class. Since we have the actual
type of the objects when we're calling the handlers, we can determine whether the type has an update handler
at compile time and elect to not even try to call it. In an optimized build, compiler probably would inline
the call and realize it's an empty funtion and discard the call completely, we're not taking any chances and
not even trying to call the update handlers of component types that do not implement a handler.

## Pointer invalidation

We've solved our initial problems but a new one popped out in the process. There is no silver bullet to this
problem. If we guarantee no pointer invalidation, we lose the cache coherent structure of storing everything
in a vector.

We also have an `entity::get_component` method that finds the component of an entity given the component type.
We could avoid storing any kinds of pointers to components and get the component each time we need it. This
solves the invalidation problem, but this time we have to linearly search through a vector to find the object
we need. With large number of components, this approach proves useless.

In malt, I'm using vectors to keep the cache coherency as it's invaluable in a game engine. But I also mend
the pointer invalidation problem a bit.

First, malt guarantees pointers to components stay valid during a frame. What this means is the following
code snippet is always safe to use, even in the case of concurrent updates:

```cpp
void some_component::Handle(malt::update{})
{
    auto comp = get_entity().add_component<another_comp>();
    // use comp, or store it to be used during this frame
}
```

This is possible since for each type of components, we store 2 containers: one contiguous vector and one
partially contiguous deque-like structure. Objects that are created in the current frame are added to the
deque-like container. This container does not move objects during expansion and therefore the pointers
are never invalidated while the container grows or shrinks. All the objects in this deque-like container
are moved to the vector container when the frame finishes. This guarantees that no component pointer gets
invalidated during a frame. As a consequence of this, pointers to all newly created components will get
invalidated before the next frame, and if there weren't enough space for the new components, pointers to
all the previously created components will get invaldeted as well.

However, this doesn't solve all the problems. You can't practically store any reference to any other
component in a component. But sometimes we need this, for example, transforms store references to their
respective parent and children. If every transform has to iterate over every transform to find each of
their children and parent, moving stuff around would be unbearably slow.

The solution to this problem is a smart pointer I've invented (at least I didn't find anything like it
with a quick google search). It's called `track_ptr`. Regular pointers don't actually point to
objects, but rather to memory locations. So when an object moves, the pointer is left pointing to the
place where an object used to live. This behaviour is sometimes what we need, but in this case, we need
something more powerful. `track_ptr`, as it's name suggests, points to an object and tracks that object
if it is moved, and is set to `nullptr` when the object it points to is deleted. I'll write more about
`track_ptr` and its performance but it's pretty much zero-overhead, or as fast and memory efficient as
possible.

Using `track_ptr`, it's possible to store the parent or children of a transform as the following, and
[this is precisely how `malt::transform` works][malt_transform]:
```cpp
class transform : public malt::component
{
    malt::track_ptr<transform> m_parent;
    std::vector<malt::track_ptr<transform>> m_children;
    void set_parent(transform* t) { m_parent = get_ptr(t); }
public:
    void add_child(transform* t) 
    { 
        m_children.emplace_back(get_ptr(t)); 
        t->set_parent(this); 
    }
};
```

## Conclusion

So there you have it. This is how components work in malt. The primary goals I try to achieve are:

1. Runtime performance: the most important aspect of such a fundamental piece of code is runtime performance.
Keeping everything in cache coherent structures, avoiding virtual functions and not calling unnecessary functions 
gets us as close as to the optimal solution.
2. Ease of use: If it doesn't impair runtime performance, ease of use is quite important. These methods are
the most used parts of the malt and they should be as easy to use correctly.

I believe this scheme satisfies both of the points and makes a good runtime component management core. There
are more to how the components are found, or organized in a dymanic way (components may be deleted). I'll write
more about these topics in the future.

All these information are implemented and is publicly available at the [`malt_core`][malt_core] github repository. 

Also, note most of the example codes are, sometimes overly, simplified and are quite different in the actual implementation.

[malt_transform]:https://github.com/malt3d/malt_basic/blob/master/include/malt_basic/components/transform.hpp#L21
[malt_core]:https://github.com/malt3d/malt_core