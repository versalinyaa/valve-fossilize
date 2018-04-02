# Fossilize

[![Build Status](https://travis-ci.org/Themaister/Fossilize.svg?branch=master)](https://travis-ci.org/Themaister/Fossilize)

**NOTE: The repo is under construction, do not use this yet.**

Fossilize is a simple library for serializing various persistent Vulkan objects which typically end up
in hashmaps. CreateInfo structs for these Vulkan objects can be recorded and replayed.

- VkSampler (immutable samplers in set layouts)
- VkDescriptorSetLayout
- VkPipelineLayout
- VkRenderPass
- VkShaderModule
- VkPipeline (compute/graphics)

The goal for this project is to cover some main use cases:

### High priority

- For internal engine use. Extend the notion of VkPipelineCache to also include these persistent objects,
so they can be automatically created in load time rather than manually declaring everything up front.
Ideally, this serialized cache could be shipped, and applications can assume all persistent objects are already created.
- Create a Vulkan layer which can capture this cache for repro purposes.
A paranoid mode would serialize the cache before every pipeline creation, to deal with crashing drivers.
- Easy way of sending shader compilation repros to conformance. Capture internally or via a Vulkan layer and send it off.
Normally, this is very difficult with normal debuggers because they generally rely on capturing frames or similar,
which doesn't work if compilation segfaults the driver. Shader compilation in Vulkan requires a lot of state,
which requires sending more complete repro applications.

### Mid priority

- Some convenience tools to modify/disassemble/spirv-opt parts of the cache.

### Low priority

- Serialize state in application once, replay on N devices to build up VkPipelineCache objects without having to run application.
- Benchmark a pipeline offline by feeding it fake input.

## Build

If rapidjson is not already bundled in your project, you need to check out the submodule.

```
git submodule update --init
```

otherwise, you can set `FOSSILIZE_RAPIDJSON_INCLUDE_PATH` if building this library as part of your project.
It is also possible to use `FOSSILIZE_VULKAN_INCLUDE_PATH` to override Vulkan header include paths.

Standalone build:
```
mkdir build
cd build
cmake ..
cmake --build .
```

Link as part of other project:
```
add_subdirectory(fossilize)
target_link_library(your-target fossilize)
```

## Serialization format

Simple JSON format which represents the various `Vk*CreateInfo` structures.
`pNext` is currently not supported.
When referring to other VK handle types like `pImmutableSamplers` in `VkDescriptorSetLayout`, or `VkRenderPass` in `VkPipeline`,
a 1-indexed format is used. 0 represents `VK_NULL_HANDLE` and 1+, represents an array index into the respective array (off-by-one).
Data blobs (specialization constant data, SPIR-V) are encoded in base64, but I'll likely need something smarter to deal with large applications which have half a trillion SPIR-V files.
When recording or replaying, a mapping from and to real Vk object handles must be provided by the application so the offset-based indexing scheme can be resolved to real handles.

## Sample API usage

### Recording state
```
// Note that fossilize.hpp will include Vulkan headers, so make sure you include vulkan.h before
// this one if you care about which Vulkan headers you use.
#include "fossilize.hpp"

void create_state()
{
    try
    {
        Fossilize::StateRecorder recorder;
        // TODO here: Add way to capture which extensions/physical device features were used to deal with exotic things
        // which require extensions when making repro cases.

        VkDescriptorSetLayoutCreateInfo info = {
            VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO
        };

        // Fill in stuff.

        Fossilize::Hash hash = Fossilize::Hashing::compute_hash_descriptor_set_layout(recorder, info);
        // Or use your own hash. This 64-bit hash will be provided back to application when replaying the state.
        // The builtin hashing functions are mostly useful for a Vulkan capturing layer, testing or similar.
        unsigned set_index = recorder.register_descriptor_set_layout(hash, info);

        vkCreateDescriptorSetLayout(..., &layout);

        // Register the true handle of the set layout so that the recorder can map
        // layout to an internal index when using register_pipeline_layout for example.
        // Setting handles are only required when other Vk*CreateInfo calls refer to any other Vulkan handles.
        recorder.set_descriptor_set_layout_handle(set_index, layout);

        // Do the same for render passes, pipelines, shader modules, samplers (if using immutable samplers) as necessary.

        std::string serialized = recorder.serialize();
        save_to_disk(serialized);
    }
    catch (const std::exception &e)
    {
        // Can throw exception on API misuse.
    }
}
```

### Replaying state
```
// Note that fossilize.hpp will include Vulkan headers, so make sure you include vulkan.h before
// this one if you care about which Vulkan headers you use.
#include "fossilize.hpp"

struct Device : Fossilize::StateCreatorInterface
{
    // See header for other functions.
    bool set_num_descriptor_set_layouts(unsigned count) override
    {
        // Know a-head of time how many set layouts there will be, useful for multi-threading.
        return true;
    }

    bool enqueue_create_descriptor_set_layout(Hash hash, unsigned index,
                                              const VkDescriptorSetLayoutCreateInfo *create_info,
                                              VkDescriptorSetLayout *layout) override
    {
        // Can queue this up for threaded creation (useful for pipelines).
        // create_info persists as long as Fossilize::Replayer exists.
        VkDescriptorSetLayout set_layout = populate_internal_hash_map(hash, create_info);

        // Let the replayer know how to fill in VkDescriptorSetLayout in upcoming pipeline creation calls.
        // Can use dummy values here if we don't care about using the create_info structs verbatim.
        *layout = set_layout;
        return true;
    }

    void wait_enqueue() override
    {
        // If using multi-threaded creation, join all queued tasks here.
    }
};

void replay_state(Device &device)
{
    try
    {
        Fossilize::Replayer replayer;
        replayer.parse(device, serialized_state, serialized_state_size);
        // Now internal hashmaps are warmed up, and all pipelines have been created.
    }
    catch (const std::exception &e)
    {
        // Can throw exception on API misuse.
    }
}
```

## Vulkan layer capture

TBD. Will try Vulkan Layer Framework for this.

## Submit shader failure repro cases

TBD