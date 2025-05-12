
### Most important implementation constraints
- high performance system iteration
- graph of systems enabling concurrency
- easily de/serializable

Component storage is a primary pivot point. We need to be able to quickly find, associate, access, create and destroy them. But most importantly, we need to be able to quickly *iterate them together*.

For quickly iterating them together, the first thought is to simply use sparse component arrays. Obviously the primary drawback to that is that systems have to iterate every entity. We want a query system that works around this issue.

### Archetypes
This is essentially the same pattern as in a JIT like V8 recognizing object type patterns. We allocate entities with the same components into packed SOA blocks that can then be efficiently iterated. This obviously has some minor complexity to it, but seems like a decent method. The worst case, where every entity has some unique combination of components, isn't much worse than some frameworks' default, and isn't really likely to happen. What's likely to happen is that this naturally groups entities into the a handful of tight blocks just how our systems want them.

We can then efficiently query these blocks using the query system, creating bundles of blocks for systems to process every entity within in sequence.

### Shared components
Components that can be referenced by multiple entities - Unity DOTS.  Great for things like materials where we often just want to render every object with that material at once?

### Chunks & chunk components
Within archetypes, divide memory into chunks, like a page or something.

After that, it is also possible to start reducing the data in chunks by combining the idea with shared components; all entities in a chunk share the same component data for some subset of components.

### Filtering
For example using shared component values, a system can choose to process only those entities that have a specific value for a particular shared component, effectively narrowing down the set of entities it needs to operate on.

Change detection is another great filtering mechanism that allows a system to only process entities whose components have been modified since the last time the system ran, avoiding unnecessary computations. ECS frameworks typically provide iterators of some kind that simplify the process of accessing the component data for the entities that match a system's query.

### Dependencies
Probably good to have some kind of way to define the bounds of *additional* queries from systems; such as an AI system that needs to do a spacial query for certain kinds of objects in the vicinity.


### References
How to efficiently reference a specific entity, across frames, given archetypes, packed soa, etc..?

I think we could use a slotmap for this. 



### Disabled flag
Tells systems to consider a component removed. Might be useful, since actually removing a component means moving the entire entity to a new archetype.

