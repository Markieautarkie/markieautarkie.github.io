---
title: "Building a Ray Tracer - Part 4: Powerful Polymorphism"
date: 2022-07-30 13:36:27 +0200
categories: [Projects, Manta Ray]
tags: [c++, ray tracing, intersections, abstraction]

math: true
---

This post is going to tie up the loose ends we left dangling around in the [previous post](https://markieautarkie.github.io/posts/building-a-ray-tracer-part-3-intersection-galore/). We put quite some effort into abstracting the setup for intersectable primitives, yet we still have to deal with conditionals for each individual object added to the scene. Plus, due to the way we execute these conditionals, we don't take into account whether the intersection we found is the closest one or not.

Let's fix these issues by creating another layer of abstraction. Each of the implemented primitives so far inherits from the abstract `intersectable` class, which describes a generic object that a ray can interact with. We can abuse this property by introducing a **list of intersectables**, which in turn also inherits from the `intersectable` class! This allows us to store *any object* which inherits from the abstract base class in the list, no matter if it is a sphere, triangle, plane, or whatever. Now we could loop over this list and let *C++* figure out what kind of object we are dealing with on its own; it will automagically grab the correct `intersect` method! This neat feature is called **polymorphism**, an absolute staple of object-oriented programming languages.

## Creating a list of intersectable objects
We'll jot down the class definition first. The `intersectable_list` class holds one single variable; the list of intersectable objects. We will use two *c++* specfic features to create this variable:

- `vector`: this is a generic collection type, comparable to an array. It can store any arbitrary type, but in our case it will store objects of class type `intersectable`. A nice feature of the `vector` collection is that it automatically grows (and shrinks) to fit objects we add or remove.
- `shared_ptr`: this is a special type of pointer, which has reference-counting properties (unlike normal *C++* pointers). Long story short, it helps us with memory management and it can even make the application more efficient in some cases.

> These features are introduced to deal with the manual pointer management present in *C++*. In most other object-oriented languages you don't need to worry about this; a normal list structure will often suffice!
{: .prompt-tip }

Without further ado, here's the implementation:

```c++
#pragma once

class intersectable_list : public intersectable
{
private:
    vector<shared_ptr<intersectable>> m_objects;

public:
    intersectable_list() {}
    intersectable_list(shared_ptr<intersectable> object) { add(object); }

    // methods to add/remove objects from the list
    void add(shared_ptr<intersectable> object) { m_objects.push_back(object); }
    void clear() { m_objects.clear(); }

    virtual bool intersect(const ray& r, intersect_data& data, float t_min, float t_max) const override;
};
```
{: file="intersectable_list.h" }

Now for the `intersect` method. Since we're dealing with a container for `intersectable` primitives/objects, the implementation will deal with looping over each object in the list. By taking into account the current closest found value for $t$, we can guarantee that the function will always return the closest intersection. The implementation is as follows:

```c++
#include "precomp.h"

bool intersectable_list::intersect(const ray& r, intersect_data& data, float t_min, float t_max) const
{
    // initially no intersection has taken place yet,
    // and set the closest intersection boundary to the provided maximum
    bool intersected = false;
    float t_closest = t_max;

    // temporary intersection data, will update whenever a closer intersection is found
    intersect_data temp_data;

    // loop over each object in the intersectable_list and check if the ray intersects with it
    for (const shared_ptr<intersectable>& object : m_objects)
        if (object->intersect(r, temp_data, t_min, t_closest))
        {
            // an intersection was found,
            // the closest value for t is the current intersection,
            // store the current intersection data
            intersected = true;
            t_closest = temp_data.t;
            data = temp_data;
        }

    // returns true if the ray intersected anything at all
    // if so, "data" will hold the information about the closest intersection
    return intersected;
}
```
{: file="intersectable_list.cpp" }

Perfect! The only thing left to do is update the `mantaray.cpp` file. Now we only have to instance whatever primitives we want, and the `intersectable_list` will take care of the rest:

```c++
intersectable_list scene;

// anything that happens only once at application start goes here
void mantaray::Init()
{
    // add a plane, sphere and a triangle with some parameters to the scene
    scene.add(make_shared<plane>(float3(0.0f, -1.0f, 0.0f), 1.0f));
    scene.add(make_shared<sphere>(float3(0.0f, 0.0f, 1.0f), 0.25f));
    scene.add(make_shared<triangle>(
        float3(-0.25f, 0.1f, 0.5f),
        float3(0.0f, 2.5f, 5.0f),
        float3(0.25f, -0.2f, 0.5f)));
}

color ray_color(const ray& r)
{
    // check for each ray whether it intersects with an object in the scene
    intersect_data data;
    if (scene.intersect(r, data, 0.0f, LARGE_FLOAT))
        // if an intersection is found, return a color based on the normal at that intersection
        return 0.5f * (data.normal + color(1.0f));
        
    // if there is no intersection, render the background as usual
    // background code here...
}
```
{: file="mantaray.cpp" .nolineno }

Running the application now gives the following result:

![Normal Mapped Scene Render](https://i.postimg.cc/gcQ7RXDV/normal-mapped-scene-render.png){: w="512" h="512" }
_A scene containing all the object types we've added so far. It only renders the closest intersection point (hence the clipping)._

And with that, we finished up the abstractions for now! The next post will be the most exciting chapter yet, as we will implement **Whitted-style ray tracing**. All the hard work we did will *finally* pay off, so I hope you are looking forward to it!