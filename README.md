# Modern DirectX 12 — Bindless Rendering in Odin

> **Stack:** SDL3 · DX12 Bindless · Slang Shaders · Dear ImGui · Odin Language
>
> A ground-up tutorial for developers who already know Vulkan or OpenGL. No backward-compatibility — this is the modern **bindless, Shader Model 6.6+** workflow used by current engines.
> Written by cross-referencing the official [Odin d3d12 vendor bindings](https://github.com/odin-lang/Odin/blob/master/vendor/directx/d3d12/d3d12.odin).

---

## Table of Contents


**Foundations**

- [00 — Modern DirectX 12 BindlessSDL3 · Slang · ImGui · Odin](#modern-directx-12-bindlesssdl3-slang-imgui-odin)
- [01 — DX12 vs Vulkan: the Conceptual Map](#dx12-vs-vulkan-the-conceptual-map)
- [02 — Device & Command Queues](#device-command-queues)

**Bindless Core**

- [03 — Descriptor Heaps](#descriptor-heaps)
- [04 — Root Signature](#root-signature)
- [05 — Dynamic Resource Binding](#dynamic-resource-binding)

**Shaders — Slang**

- [06 — Slang — Bindless Resource Access](#slang-bindless-resource-access)
- [07 — Slang Modules & Interfaces](#slang-modules-interfaces)
- [08 — Compiling Slang → DXIL](#compiling-slang-dxil)

**Resources**

- [09 — Buffers & Textures](#buffers-textures)
- [10 — Upload & GPU Memory](#upload-gpu-memory)
- [11 — Barriers & Synchronisation](#barriers-synchronisation)

**Window & Rendering**

- [12 — SDL3 Window + DXGI Swapchain](#sdl3-window-dxgi-swapchain)
- [13 — Draw Commands](#draw-commands)
- [14 — Full Bindless Triangle — SDL3 + Slang](#full-bindless-triangle-sdl3-slang)

**Dear ImGui**

- [15 — ImGui DX12 Backend Setup](#imgui-dx12-backend-setup)
- [16 — ImGui SDL3 Integration](#imgui-sdl3-integration)
- [17 — ImGui Frame Loop & Render](#imgui-frame-loop-render)


---


## Modern DirectX 12 BindlessSDL3 · Slang · ImGui · Odin


This tutorial teaches you DirectX 12 from scratch, written for someone who already understands Vulkan and OpenGL. You will not learn the "safe" DX12 with descriptor tables, root descriptors, and backward-compatible patterns. You will learn the bindless, SM 6.6+ workflow that modern engines (Unreal 5, Godot 4.x, etc.) have converged on.


The stack: SDL3 for OS/window/input, DXGI + DX12 for rendering, Slang as the shader language (modern module system, not HLSL-compat mode), and Dear ImGui for debug UI. All code is idiomatic Odin.


  | | | |
|---|---|---|
|  | **** |  |
#### Bindless


Every resource lives in one giant descriptor heap. Shaders index into it directly with a uint. No rebinding, no descriptor sets.


      💎

#### Slang shaders


Modern module system with import, interfaces, generics, and automatic bindless lowering. Targets DXIL/SPIR-V/Metal.


      🪟

#### SDL3 + DXGI


SDL3 owns the window and input. DXGI grabs the HWND from SDL3 to create the swapchain. Clean separation of concerns.


      🖼️

#### ImGui DX12


ImGui's official imgui_impl_dx12 backend slots into the bindless heap and renders debug UI at the end of each frame.


      🦊

#### Odin idioms


COM objects as ^Interface with -> call syntax. Odin's windows package D3D12 bindings used directly.


      🚫

#### No compat layer


Requires Win10 1909+, D3D_FEATURE_LEVEL_12_1, Shader Model 6.6. GPU must support RESOURCE_BINDING_TIER_3.


> **Prerequisites:** You know what a command buffer / queue / fence / image layout is (Vulkan) or what a FBO / VAO / UBO is (OpenGL). You have odin installed, a Windows 10/11 machine with a DX12-capable GPU, and the Slang compiler (slangc) either as a standalone binary or via the Slang SDK.


---


## DX12 vs Vulkan: the Conceptual Map


You already know Vulkan, so here is the translation table. DX12 and Vulkan were designed around the same hardware realities at roughly the same time. The vocabulary differs; the ideas are the same.


| Vulkan | DirectX 12 | Notes |
| :--- | :--- | :--- |
| VkInstance | IDXGIFactory | Enumerate adapters, create device |
| VkPhysicalDevice | IDXGIAdapter | Physical GPU |
| VkDevice | ID3D12Device | Logical device — all resource creation |
| VkQueue (graphics) | ID3D12CommandQueue (DIRECT) | Submits command lists |
| VkQueue (compute) | ID3D12CommandQueue (COMPUTE) |  |
| VkCommandBuffer | ID3D12GraphicsCommandList | Records draw/dispatch/copy commands |
| VkCommandPool | ID3D12CommandAllocator | Backing memory for a command list |
| VkRenderPass / VkFramebuffer | Dynamic rendering (no equivalent) | DX12 never had render passes — you just set RTVs directly |
| VkDescriptorPool + VkDescriptorSet | ID3D12DescriptorHeap | One giant heap in bindless; index with uint |
| VkPipelineLayout | ID3D12RootSignature | Defines how shader accesses root data |
| VkPipeline | ID3D12PipelineState | Compiled shader + fixed-function state |
| VkBuffer | ID3D12Resource (buffer) | Same concept |
| VkImage | ID3D12Resource (texture) | Same concept |
| VkFence / VkSemaphore | ID3D12Fence | One type — value-based timeline semaphore |
| VkSwapchainKHR | IDXGISwapChain4 | DXGI's domain, not D3D12's |
| Image layout transitions (VkImageMemoryBarrier) | D3D12_RESOURCE_BARRIER (transition) | Same idea, same cost |
| SPIR-V | DXIL (compiled from Slang or HLSL via slangc/DXC) | We use Slang → DXIL. Slang also targets SPIR-V and Metal. |


### Key DX12 differences from Vulkan


No render passes. DX12 predates explicit render passes. You call OMSetRenderTargets and ClearRenderTargetView directly. This is actually simpler for a first tutorial.


Fences are timeline semaphores. ID3D12Fence holds a monotonically increasing 64-bit value. Signal on the GPU, wait on the CPU. Exactly Vulkan's VkSemaphore with VK_KHR_timeline_semaphore.


DXGI handles windows and swapchains. ID3D12Device never touches the window. That's IDXGIFactory's job. Coming from OpenGL or Vulkan + GLFW, this is the most surprising split.


---


## Device & Command Queues


In Odin the D3D12 bindings live in vendor:directx/d3d12 (package directx_d3d12) and DXGI in vendor:directx/dxgi (package directx_dxgi). Win32 primitives like HANDLE, HWND, and INFINITE come from core:sys/windows, which we alias as win32. We call COM vtable methods with Odin's -> syntax.


```odin
package renderer

import win32 "core:sys/windows"
import d3d12 "vendor:directx/d3d12"
import dxgi  "vendor:directx/dxgi"

// Mirrors a Vulkan "device context". Owns the logical device and queues.
GfxDevice :: struct {
    factory     : ^dxgi.IFactory6,
    adapter     : ^dxgi.IAdapter4,
    device      : ^d3d12.IDevice9,   // always use the latest version
    gfx_queue   : ^d3d12.ICommandQueue,
    copy_queue  : ^d3d12.ICommandQueue,
}

create_device :: proc() -> GfxDevice {
    dev : GfxDevice

    // Enable debug layer in debug builds only
    when ODIN_DEBUG {
        debug : ^d3d12.IDebug3
        d3d12.GetDebugInterface(d3d12.IDebug3_UUID, (^rawptr)(&debug))
        debug->EnableDebugLayer()
        debug->SetEnableGPUBasedValidation(true)
    }

    // Factory — equivalent to VkInstance
    flags : dxgi.CREATE_FACTORY = {}
    when ODIN_DEBUG { flags = {.DEBUG} }
    dxgi.CreateDXGIFactory2(flags, dxgi.IFactory6_UUID, (^rawptr)(&dev.factory))

    // Pick highest-performance adapter (discrete GPU preferred)
    dev.factory->EnumAdapterByGpuPreference(
        0,
        .HIGH_PERFORMANCE,
        dxgi.IAdapter4_UUID,
        (^rawptr)(&dev.adapter),
    )

    // Create logical device at feature level 12.1 minimum
    d3d12.CreateDevice(
        dev.adapter,
        ._12_1,           // D3D_FEATURE_LEVEL
        d3d12.IDevice9_UUID,
        (^rawptr)(&dev.device),
    )

    // Verify Resource Binding Tier 3 (required for bindless)
    opts : d3d12.FEATURE_DATA_OPTIONS
    dev.device->CheckFeatureSupport(.OPTIONS, &opts, u32(size_of(opts)))
    assert(opts.ResourceBindingTier >= ._3,
           "GPU does not support Resource Binding Tier 3 — bindless unavailable")

    // Graphics queue (= Vulkan VK_QUEUE_GRAPHICS_BIT)
    gfx_desc := d3d12.COMMAND_QUEUE_DESC{
        Type     = .DIRECT,
        Priority = 0,
        Flags    = {},
    }
    dev.device->CreateCommandQueue(&gfx_desc, d3d12.ICommandQueue_UUID, (^rawptr)(&dev.gfx_queue))

    // Dedicated copy queue for background uploads (optional but recommended)
    copy_desc := d3d12.COMMAND_QUEUE_DESC{ Type = .COPY }
    dev.device->CreateCommandQueue(©_desc, d3d12.ICommandQueue_UUID, (^rawptr)(&dev.copy_queue))

    return dev
}
```


> **Odin COM idiom:** DX12 objects are COM interfaces. In Odin, they are ^SomeInterface (pointer-to-struct-with-vtable). Call methods with obj->MethodName(args). This is exactly how you'd use them in C++ without any smart pointers.


---


## Descriptor Heaps


In Vulkan, bindless requires the VK_EXT_descriptor_indexing extension and VkDescriptorPool with UPDATE_AFTER_BIND flags. In DX12 with Tier 3 it is just… how it works. You create one heap with a million slots and forget about it.


There are two shader-visible heap types you care about:


- CBV_SRV_UAV heap — holds Constant Buffer Views, Shader Resource Views (textures/buffers read), and Unordered Access Views (read-write resources). This is your main bindless heap.
- SAMPLER heap — holds sampler states. Max 2048 slots, separate from the SRV heap.


> **One heap of each type bound at a time:** You can only have one CBV_SRV_UAV heap and one SAMPLER heap bound to a command list simultaneously. This is why in bindless you make them enormous and never swap them.


```odin
// Bindless descriptor heap — the heart of modern DX12.
// In Vulkan terms: one VkDescriptorPool for everything, forever.

BINDLESS_HEAP_CAPACITY :: 1_000_000  // DX12 max for CBV/SRV/UAV is 1,000,000
SAMPLER_HEAP_CAPACITY  :: 2_048      // Hard DX12 limit for sampler heaps

BindlessHeap :: struct {
    heap          : ^d3d12.IDescriptorHeap,
    sampler_heap  : ^d3d12.IDescriptorHeap,

    // CPU and GPU start handles (like VkDescriptorSet base address)
    cpu_base      : d3d12.CPU_DESCRIPTOR_HANDLE,
    gpu_base      : d3d12.GPU_DESCRIPTOR_HANDLE,
    srv_stride    : u32,   // hardware descriptor size in bytes

    // Simple linear allocator — in production use a free-list
    next_slot     : u32,
    sampler_slot  : u32,
}

create_bindless_heap :: proc(device: ^d3d12.IDevice9) -> BindlessHeap {
    h : BindlessHeap

    // Main bindless heap — CBV/SRV/UAV, shader-visible
    desc := d3d12.DESCRIPTOR_HEAP_DESC{
        Type           = .CBV_SRV_UAV,
        NumDescriptors = BINDLESS_HEAP_CAPACITY,
        Flags          = {.SHADER_VISIBLE},   // GPU can read this
    }
    device->CreateDescriptorHeap(&desc, d3d12.IDescriptorHeap_UUID, (^rawptr)(&h.heap))

    // Sampler heap
    samp_desc := d3d12.DESCRIPTOR_HEAP_DESC{
        Type           = .SAMPLER,
        NumDescriptors = SAMPLER_HEAP_CAPACITY,
        Flags          = {.SHADER_VISIBLE},
    }
    device->CreateDescriptorHeap(&samp_desc, d3d12.IDescriptorHeap_UUID, (^rawptr)(&h.sampler_heap))

    // Cache CPU/GPU handles and descriptor stride
    h.heap->GetCPUDescriptorHandleForHeapStart(&h.cpu_base)
    h.heap->GetGPUDescriptorHandleForHeapStart(&h.gpu_base)
    h.srv_stride = device->GetDescriptorHandleIncrementSize(.CBV_SRV_UAV)

    return h
}

// Allocate a slot in the bindless heap. Returns the index you pass to shaders.
bindless_alloc :: proc(h: ^BindlessHeap) -> u32 {
    idx := h.next_slot
    h.next_slot += 1
    assert(h.next_slot BINDLESS_HEAP_CAPACITY)
    return idx
}

// Get the CPU handle for slot i — used when writing descriptors
cpu_handle_at :: proc(h: BindlessHeap, idx: u32) -> d3d12.CPU_DESCRIPTOR_HANDLE {
    return { ptr = h.cpu_base.ptr + uintptr(idx * h.srv_stride) }
}

// Register an SRV (texture or buffer read) into a slot
register_srv :: proc(
    device  : ^d3d12.IDevice9,
    heap    : ^BindlessHeap,
    resource: ^d3d12.IResource,
    desc    : ^d3d12.SHADER_RESOURCE_VIEW_DESC,
) -> u32 {
    idx := bindless_alloc(heap)
    cpu := cpu_handle_at(heap^, idx)
    device->CreateShaderResourceView(resource, desc, cpu)
    return idx  // this uint is what you pass to the shader
}
```


### Binding the heap to a command list


Before recording any draw or dispatch, bind both heaps once. The GPU will use these for all subsequent ResourceDescriptorHeap[] accesses in HLSL.


```odin
// Called once per frame, at the top of your command list recording
bind_bindless_heaps :: proc(cmd: ^d3d12.IGraphicsCommandList7, heap: BindlessHeap) {
    heaps := [2]^d3d12.IDescriptorHeap{ heap.heap, heap.sampler_heap }
    cmd->SetDescriptorHeaps(2, &heaps[0])
}
```


---


## Root Signature


The Root Signature is DX12's equivalent of a Vulkan VkPipelineLayout. It defines what the shader can access from the root arguments. In bindless, the root signature is almost trivially small: you just pass a few uint32 constants (indices into the bindless heap) and one SRV or CBV pointing at a per-draw data buffer.


There are three types of root parameters:


| Type | GPU cost | Use for |
| :--- | :--- | :--- |
| Root Constants | Fastest — stored inline in root | Per-draw uint indices (bindless handles) |
| Root Descriptors (CBV/SRV/UAV) | 1 DWORD — GPU virtual address | Per-draw data buffer, structured buffer |
| Descriptor Tables | Indirect — GPU dereferences table | Legacy. Avoid in bindless. |


> **The bindless root signature pattern:** In a bindless renderer, your root signature typically has: (1) N root constants (usually 4–16 uint32s) that carry descriptor heap indices; (2) one root CBV pointing at a frame-constant buffer. That's it. No descriptor tables.


```odin
// Bindless root signature: 8 root constants + 1 root CBV for frame data.
//
// Root layout (visible in HLSL as register(b0, space0) etc.):
//   [0]    8 x uint32 root constants   → b0 space0
//   [1]    root CBV  (frame constants) → b1 space0

create_bindless_root_signature :: proc(device: ^d3d12.IDevice9) -> ^d3d12.IRootSignature {
    // Root parameter 0: 8 uint32 constants at b0
    constants_param := d3d12.ROOT_PARAMETER1{
        ParameterType = ._32BIT_CONSTANTS,
        Constants = {
            ShaderRegister = 0,
            RegisterSpace  = 0,
            Num32BitValues = 8,
        },
        ShaderVisibility = .ALL,
    }

    // Root parameter 1: CBV pointing at frame-constant buffer at b1
    frame_cbv_param := d3d12.ROOT_PARAMETER1{
        ParameterType = .CBV,
        Descriptor = {
            ShaderRegister = 1,
            RegisterSpace  = 0,
            Flags          = {.DATA_STATIC_WHILE_SET_AT_EXECUTE},
        },
        ShaderVisibility = .ALL,
    }

    params := [2]d3d12.ROOT_PARAMETER1{ constants_param, frame_cbv_param }

    // Describe flags: deny hull/domain/geometry shader access for performance
    flags :=
        d3d12.ROOT_SIGNATURE_FLAGS{
            .ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT,
            .DENY_HULL_SHADER_ROOT_ACCESS,
            .DENY_DOMAIN_SHADER_ROOT_ACCESS,
            .DENY_GEOMETRY_SHADER_ROOT_ACCESS,
            // SM 6.6 bindless flag — allows ResourceDescriptorHeap[] in HLSL
            .CBV_SRV_UAV_HEAP_DIRECTLY_INDEXED,
            .SAMPLER_HEAP_DIRECTLY_INDEXED,
        }

    desc := d3d12.VERSIONED_ROOT_SIGNATURE_DESC{
        Version = .1_1,
        Desc_1_1 = {
            NumParameters     = 2,
            pParameters       = ¶ms[0],
            NumStaticSamplers  = 0,
            Flags             = flags,
        },
    }

    blob       : ^d3d12.IBlob
    error_blob : ^d3d12.IBlob
    d3d12.SerializeVersionedRootSignature(&desc, &blob, &error_blob)

    root_sig : ^d3d12.IRootSignature
    device->CreateRootSignature(0, blob->GetBufferPointer(), blob->GetBufferSize(),
        d3d12.IRootSignature_UUID, (^rawptr)(&root_sig))

    return root_sig
}
```


---


## Dynamic Resource Binding


With the heap and root signature set up, binding a resource to a draw call is: write its index into a root constant. That is the entire binding model.


```odin
// Per-draw binding constants pushed via root constants.
// Maps to `ConstantBuffer<DrawConstants> g_draw : register(b0)` in HLSL.
DrawConstants :: struct {
    vertex_buffer_idx  : u32,  // heap index of vertex data SRV
    instance_data_idx  : u32,  // heap index of per-instance buffer SRV
    albedo_tex_idx     : u32,  // heap index of albedo texture SRV
    sampler_idx        : u32,  // sampler heap index
    transform_idx      : u32,  // index into transform buffer
    material_idx       : u32,  // index into material buffer
    _pad               : [2]u32,
}
#assert(size_of(DrawConstants) == 32)  // 8 x uint32 = 8 DWORDS = our root constant budget

draw_mesh :: proc(
    cmd       : ^d3d12.IGraphicsCommandList7,
    mesh      : Mesh,
    transform : u32,
    material  : Material,
) {
    dc := DrawConstants{
        vertex_buffer_idx = mesh.vertex_srv_idx,
        albedo_tex_idx    = material.albedo_idx,
        sampler_idx       = material.sampler_idx,
        transform_idx     = transform,
        material_idx      = material.material_idx,
    }

    // Root parameter 0 = our 8 constants. Push all at once.
    cmd->SetGraphicsRoot32BitConstants(
        0,                            // root parameter index
        8,                            // number of 32-bit values
        &dc,                          // source data
        0,                            // offset into root constants
    )

    cmd->DrawIndexedInstanced(mesh.index_count, 1, 0, 0, 0)
}
```


Notice: no vkCmdBindDescriptorSets, no descriptor set update, no pipeline layout reference. You just push 8 uint32s and draw. The shader receives them and uses them as indices into ResourceDescriptorHeap.


---


## Slang — Bindless Resource Access


Slang is a modern shading language developed by NVIDIA Research. It compiles to DXIL, SPIR-V, Metal, CUDA, and CPU. Crucially, it has a proper module system, interfaces, generics, and automatic bindless lowering — none of which exist in vanilla HLSL.


In Slang's modern mode you do not write register(t0) or register(b0) declarations at all. The compiler handles binding layout automatically, and for bindless targets it emits the correct ResourceDescriptorHeap / SamplerDescriptorHeap accesses in the output DXIL.


```odin
// Every binding declared manually
layout(set=0, binding=0) uniform sampler2D u_albedo;
layout(set=0, binding=1) readonly buffer VertBuf {
    Vertex v[];
} u_verts;

void main() {
    vec4 c = texture(u_albedo, uv);
    Vertex v = u_verts.v[gl_VertexIndex];
}
```


      Slang bindless (no register decls)

```odin
// No register() annotations anywhere.
// Indices arrive as push constants.
import "bindless";   // our shared module

[shader("fragment")]
float4 frag(VSOut i) : SV_TARGET {
    var tex  = get_texture2d(i.albedo_idx);
    var samp = get_sampler(i.sampler_idx);
    return tex.Sample(samp, i.uv);
}
```


### The bindless module — bindless.slang


Slang's __DynamicResource and descriptor heap indexing are wrapped in a shared module that all other shader modules import. This replaces the HLSL pattern of using ResourceDescriptorHeap[idx] directly in every file.


```hlsl
// Module: bindless
// Every shader imports this. Provides typed heap access helpers.
// No register() annotations — Slang resolves bindings for the target.

module bindless;

// Slang's built-in bindless heap accessors.
// On DX12 targets these lower to ResourceDescriptorHeap[idx] in DXIL.
// On Vulkan targets they lower to descriptor indexing extensions.
public Texture2D<float4> get_texture2d(uint idx) {
    return ResourceDescriptorHeap[idx];
}

public Texture2D<float> get_texture2d_f1(uint idx) {
    return ResourceDescriptorHeap[idx];
}

public StructuredBuffer<T> get_buffer<T>(uint idx) {
    return ResourceDescriptorHeap[idx];
}

public RWStructuredBuffer<T> get_rw_buffer<T>(uint idx) {
    return ResourceDescriptorHeap[idx];
}

public ByteAddressBuffer get_raw_buffer(uint idx) {
    return ResourceDescriptorHeap[idx];
}

public SamplerState get_sampler(uint idx) {
    return SamplerDescriptorHeap[idx];
}

public SamplerComparisonState get_cmp_sampler(uint idx) {
    return SamplerDescriptorHeap[idx];
}

// Per-draw push constants. Matches the Odin DrawConstants struct exactly.
// Use [[vk::push_constant]] on Vulkan; on DX12 this maps to root constants.
public struct DrawConstants {
    public uint vertex_buffer_idx;
    public uint transform_idx;
    public uint albedo_idx;
    public uint sampler_idx;
    public uint material_idx;
    public uint3 _pad;
}

// The push constant block — b0 on DX12, push_constant on Vulkan.
// Slang handles the difference automatically based on compile target.
[[vk::push_constant]]
ConstantBuffer<DrawConstants> g_draw;

public DrawConstants draw_constants() { return g_draw; }
```


### Vertex shader module — vertex.slang


```hlsl
module vertex;

import "bindless";

public struct Vertex {
    public float3 position;
    public float3 normal;
    public float2 uv;
}

public struct VSOut {
    public float4 sv_pos : SV_Position;
    public float2 uv     : TEXCOORD0;
    public float3 normal : NORMAL;
}

[shader("vertex")]
public VSOut vert_main(uint vid : SV_VertexID) {
    let dc = draw_constants();

    // Fetch vertex from the bindless heap — generic get_buffer<T>
    StructuredBuffer<Vertex> verts     = get_buffer<Vertex>(dc.vertex_buffer_idx);
    StructuredBuffer<float4x4> xforms  = get_buffer<float4x4>(dc.transform_idx);

    Vertex v   = verts[vid];
    float4x4 m = xforms[0];

    VSOut o;
    o.sv_pos = mul(m, float4(v.position, 1.0));
    o.uv     = v.uv;
    o.normal = v.normal;
    return o;
}
```


### Fragment shader module — material.slang


```hlsl
module material;

import "bindless";
import "vertex";         // re-uses VSOut

struct MaterialData {
    float4 base_color;
    float  roughness;
    float  metallic;
    float2 _pad;
}

[shader("fragment")]
float4 frag_main(VSOut i) : SV_TARGET {
    let dc = draw_constants();

    Texture2D<float4> albedo_tex = get_texture2d(dc.albedo_idx);
    SamplerState       samp       = get_sampler(dc.sampler_idx);

    // Material data from bindless buffer — typed generic access
    StructuredBuffer<MaterialData> mats = get_buffer<MaterialData>(dc.material_idx);
    MaterialData mat = mats[0];

    float4 base  = albedo_tex.Sample(samp, i.uv);
    return base * mat.base_color;
}
```


> **No vertex input layout:** Because the vertex shader fetches data via get_buffer(idx), the DX12 pipeline's InputLayout is null. SV_VertexID is the only system input. This is the standard bindless vertex pulling pattern in Slang.


---


## Slang Modules & Interfaces


Slang's module system works like a compiled library: each .slang file is a module, you import it, and the compiler only recompiles what changed. This is the primary reason to use Slang over raw HLSL for any serious project.


### Interfaces — shader polymorphism


Slang interfaces are compile-time contracts, similar to Rust traits or Go interfaces. They enable writing one generic pass that works with multiple material types — resolved at compile time, zero runtime cost.


```hlsl
module material_interface;

import "bindless";
import "vertex";

// Interface: any material type must implement evaluate().
// The compiler generates specialised code per concrete type — no vtable.
public interface IMaterial {
    float4 evaluate(VSOut surface, SamplerState samp);
}

// Concrete material: unlit textured
public struct UnlitMaterial : IMaterial {
    uint albedo_idx;

    public float4 evaluate(VSOut s, SamplerState samp) {
        return get_texture2d(albedo_idx).Sample(samp, s.uv);
    }
}

// Concrete material: simple PBR
public struct PBRMaterial : IMaterial {
    uint albedo_idx;
    uint normal_idx;
    uint orm_idx;   // occlusion/roughness/metallic packed

    public float4 evaluate(VSOut s, SamplerState samp) {
        float4 albedo = get_texture2d(albedo_idx).Sample(samp, s.uv);
        float4 orm    = get_texture2d(orm_idx).Sample(samp, s.uv);
        // ... full PBR calculation here
        return albedo;
    }
}

// Generic fragment shader — one function, two specialisations.
// slangc generates two separate DXIL blobs at compile time.
[shader("fragment")]
float4 frag_generic<M : IMaterial>(
    VSOut i,
    uniform M mat
) : SV_TARGET {
    let samp = get_sampler(draw_constants().sampler_idx);
    return mat.evaluate(i, samp);
}
```


### Generics on buffers


The get_buffer helper from the bindless module is itself generic. Slang resolves the type at compile time — no runtime overhead, no casts in the generated DXIL.


```hlsl
// Works for any struct T — Slang knows the stride at compile time
let joints = get_buffer<float4x4>(dc.joint_matrices_idx);
let lights  = get_buffer<PointLight>(dc.light_buffer_idx);
let count   = get_buffer<uint>(dc.draw_indirect_count_idx);

// RW access for compute shaders
let output = get_rw_buffer<ClusteredLight>(dc.output_idx);
```


### Slang vs HLSL — what you gain


| Feature | HLSL | Slang |
| :--- | :--- | :--- |
| Module system | No (only #include) | Yes — import, incremental compilation |
| Interfaces | No | Yes — compile-time polymorphism |
| Generics on types | No (template-like, limited) | Yes — full generic type parameters |
| Automatic binding layout | Manual register() required | Automatic, or explicit with [binding] |
| Multi-target (DX/VK/Metal) | DX12 only | DXIL, SPIR-V, MSL, CUDA, CPU |
| Reflection API | Limited (DXC reflection) | Rich typed reflection at runtime |
| HLSL compatibility | Native | Slang is a superset — HLSL compiles in Slang |


> **Slang is a superset of HLSL:** You can take any HLSL file, rename it .slang, and it will compile. Slang adds features on top. This means migrating existing HLSL codebase is incremental.


---


## Compiling Slang → DXIL


Slang ships its own compiler (slangc) and a runtime API (slang.h). For DX12 you target dxil (requires the DXC validator to be on PATH or alongside the binary). There are two modes: offline (build step) and runtime (load Slang DLL, compile in-process). Runtime is strongly recommended because Slang's reflection API can auto-generate root signatures.


### Offline compilation


```hlsl
# Compile vertex entry point — note -profile sm_6_6 not vs_6_6
slangc vertex.slang -entry vert_main -stage vertex \
    -target dxil -profile sm_6_6 \
    -o triangle_vs.dxil

# Compile fragment entry point
slangc material.slang -entry frag_main -stage fragment \
    -target dxil -profile sm_6_6 \
    -o triangle_ps.dxil

# Include module search path (-I) when modules are in subdirs
slangc -I shaders/ material.slang -entry frag_main -stage fragment \
    -target dxil -profile sm_6_6 -o triangle_ps.dxil

# Debug build: keep reflection metadata, line info
slangc material.slang -entry frag_main -stage fragment \
    -target dxil -profile sm_6_6 -g -O0 -o triangle_ps_dbg.dxil
```


### Runtime compilation from Odin


Runtime compilation lets you use Slang's reflection API to automatically read back parameter types, binding slots, and struct layouts — eliminating manual synchronisation between your Odin structs and your shader structs.


```hlsl
// Slang's C API is loaded via the shared library.
// Bindings live in vendor:slang or via a foreign import block.
// Below uses a simplified foreign import pattern.

import slang "vendor:slang"   // hypothetical Odin vendor binding

SlangCompiler :: struct {
    global_session : ^slang.IGlobalSession,
    session        : ^slang.ISession,
}

create_slang_compiler :: proc(shader_dir: string) -> SlangCompiler {
    sc : SlangCompiler
    slang.createGlobalSession(slang.API_VERSION, &sc.global_session)

    // Session = one compilation context. Share across your renderer lifetime.
    target_desc := slang.TargetDesc{
        format  = .DXIL,
        profile = sc.global_session->findProfile("sm_6_6"),
    }
    session_desc := slang.SessionDesc{
        targets           = &target_desc,
        targetCount       = 1,
        searchPaths       = &((cstring)(raw_data(shader_dir))),
        searchPathCount   = 1,
    }
    sc.global_session->createSession(session_desc, &sc.session)
    return sc
}

CompiledShader :: struct {
    vs_blob : []byte,
    ps_blob : []byte,
}

compile_shader :: proc(
    sc         : ^SlangCompiler,
    source_path: string,
    vs_entry   : string,
    ps_entry   : string,
) -> CompiledShader {
    module_ : ^slang.IModule
    sc.session->loadModule(strings.clone_to_cstring(source_path), &module_)

    // Find entry points by name
    vs_ep, ps_ep : ^slang.IEntryPoint
    module_->findEntryPointByName(strings.clone_to_cstring(vs_entry), &vs_ep)
    module_->findEntryPointByName(strings.clone_to_cstring(ps_entry), &ps_ep)

    // Compose into a program and link
    components := [3]^slang.IComponentType{ module_, vs_ep, ps_ep }
    composite  : ^slang.IComponentType
    sc.session->createCompositeComponentType(&components[0], 3, &composite, nil)

    linked : ^slang.IComponentType
    composite->link(&linked, nil)

    // Get the compiled DXIL blobs (entry point index 0 = vs, 1 = ps)
    vs_code, ps_code : ^slang.IBlob
    linked->getEntryPointCode(0, 0, &vs_code, nil)
    linked->getEntryPointCode(1, 0, &ps_code, nil)

    return CompiledShader{
        vs_blob = blob_to_bytes(vs_code),
        ps_blob = blob_to_bytes(ps_code),
    }
}

blob_to_bytes :: proc(blob: ^slang.IBlob) -> []byte {
    ptr  := blob->getBufferPointer()
    size := blob->getBufferSize()
    return ([^]byte)(ptr)[:size]
}

// Load a compiled PSO from a CompiledShader
create_graphics_pso :: proc(
    device   : ^d3d12.IDevice9,
    root_sig : ^d3d12.IRootSignature,
    shaders  : CompiledShader,
    rtv_fmt  : dxgi.FORMAT,
) -> ^d3d12.IPipelineState {
    pso_desc := d3d12.GRAPHICS_PIPELINE_STATE_DESC{
        pRootSignature        = root_sig,
        VS                    = { pShaderBytecode = &shaders.vs_blob[0], BytecodeLength = d3d12.SIZE_T(len(shaders.vs_blob)) },
        PS                    = { pShaderBytecode = &shaders.ps_blob[0], BytecodeLength = d3d12.SIZE_T(len(shaders.ps_blob)) },
        BlendState            = default_blend_state(),
        SampleMask            = 0xffff_ffff,
        RasterizerState       = default_rasterizer_state(),
        DepthStencilState     = default_depth_stencil_state(),
        InputLayout           = { nil, 0 },   // null — vertex pulling
        PrimitiveTopologyType = .TRIANGLE,
        NumRenderTargets      = 1,
        DSVFormat             = .D32_FLOAT,
    }
    pso_desc.RTVFormats[0]       = rtv_fmt
    pso_desc.SampleDesc.Count = 1

    pso : ^d3d12.IPipelineState
    device->CreateGraphicsPipelineState(&pso_desc, d3d12.IPipelineState_UUID, (^rawptr)(&pso))
    return pso
}
```


> **DXC validator requirement:** Slang targeting DXIL requires dxcompiler.dll and dxil.dll to be present (either in PATH or the same directory as your exe). They ship with the Windows SDK and separately as standalone DLLs from the DXC GitHub releases. The Slang SDK bundles them starting with Slang 2024.x.


---


## Buffers & Textures


DX12 has one resource type: ID3D12Resource. Whether it is a vertex buffer, constant buffer, texture, or render target depends on how you create it and what views you create for it — identical concept to Vulkan's VkBuffer/VkImage with views.


```odin
// Create a committed resource (= GPU allocation backed by an implicit heap).
// For bindless, most of your buffers live in DEFAULT heap (GPU-only).
// Upload heap is for CPU→GPU staging only.

create_buffer :: proc(
    device     : ^d3d12.IDevice9,
    size_bytes : u64,
    heap_type  : d3d12.HEAP_TYPE,     // .DEFAULT (GPU) or .UPLOAD (CPU writable)
    state      : d3d12.RESOURCE_STATES, // initial resource state
    flags      : d3d12.RESOURCE_FLAGS,  // .ALLOW_UNORDERED_ACCESS for UAV
) -> ^d3d12.IResource {
    heap_props := d3d12.HEAP_PROPERTIES{ Type = heap_type }
    res_desc   := d3d12.RESOURCE_DESC{
        Dimension        = .BUFFER,
        Alignment        = 0,
        Width            = size_bytes,
        Height           = 1,
        DepthOrArraySize = 1,
        MipLevels        = 1,
        Format           = .UNKNOWN,
        SampleDesc       = { Count = 1 },
        Layout           = .ROW_MAJOR,   // buffers always ROW_MAJOR
        Flags            = flags,
    }

    resource : ^d3d12.IResource
    device->CreateCommittedResource(
        &heap_props,
        {},                     // heap flags
        &res_desc,
        state,
        nil,                    // clear value (only for RT/DS)
        d3d12.IResource_UUID,
        (^rawptr)(&resource),
    )
    return resource
}

// Register a StructuredBuffer SRV into the bindless heap.
// Returns the index the shader will use.
register_structured_buffer_srv :: proc(
    device       : ^d3d12.IDevice9,
    heap         : ^BindlessHeap,
    resource     : ^d3d12.IResource,
    num_elements : u32,
    stride_bytes : u32,
) -> u32 {
    srv_desc := d3d12.SHADER_RESOURCE_VIEW_DESC{
        Format                  = .UNKNOWN,
        ViewDimension           = .BUFFER,
        Shader4ComponentMapping = d3d12.DEFAULT_SHADER_4_COMPONENT_MAPPING,
        Buffer = {
            FirstElement        = 0,
            NumElements         = num_elements,
            StructureByteStride = stride_bytes,
            Flags               = {},
        },
    }
    return register_srv(device, heap, resource, &srv_desc)
}

// Register a Texture2D SRV
register_texture2d_srv :: proc(
    device   : ^d3d12.IDevice9,
    heap     : ^BindlessHeap,
    resource : ^d3d12.IResource,
    format   : dxgi.FORMAT,
    mips     : u32,
) -> u32 {
    srv_desc := d3d12.SHADER_RESOURCE_VIEW_DESC{
        Format                  = format,
        ViewDimension           = .TEXTURE2D,
        Shader4ComponentMapping = d3d12.DEFAULT_SHADER_4_COMPONENT_MAPPING,
        Texture2D = {
            MostDetailedMip = 0,
            MipLevels       = mips,
        },
    }
    return register_srv(device, heap, resource, &srv_desc)
}
```


---


## Upload & GPU Memory


DX12 (and Vulkan) require you to explicitly stage data through an upload buffer. There is no glBufferData. The flow is identical to Vulkan's staging buffer pattern.


  **** __Upload Buffer
      HEAP_TYPE_UPLOAD, CPU-mapped


      Step 2
      CopyBufferRegion
      On copy/gfx cmdlist


      Step 3
      Resource Barrier
      COPY_DEST → SHADER_RESOURCE


      Step 4
      GPU Buffer
      HEAP_TYPE_DEFAULT, GPU-only


```odin
// One-shot synchronous upload. For production use a ring buffer
// of upload heaps and submit asynchronously on the copy queue.
upload_buffer_data :: proc(
    device     : ^d3d12.IDevice9,
    queue      : ^d3d12.ICommandQueue,
    dst_buffer : ^d3d12.IResource,
    data       : []byte,
) {
    size := u64(len(data))

    // Create upload staging buffer (CPU-writable UPLOAD heap)
    upload := create_buffer(device, size, .UPLOAD, d3d12.RESOURCE_STATE_GENERIC_READ, {})

    // Map and memcpy — UPLOAD heap stays persistently mapped (like VK_MEMORY_PROPERTY_HOST_VISIBLE)
    mapped : rawptr
    upload->Map(0, nil, &mapped)
    mem.copy(mapped, &data[0], len(data))
    upload->Unmap(0, nil)

    // Create a one-time command list for the copy
    alloc : ^d3d12.ICommandAllocator
    device->CreateCommandAllocator(.DIRECT, d3d12.ICommandAllocator_UUID, (^rawptr)(&alloc))
    cmd : ^d3d12.IGraphicsCommandList7
    device->CreateCommandList(0, .DIRECT, alloc, nil, d3d12.IGraphicsCommandList7_UUID, (^rawptr)(&cmd))

    cmd->CopyBufferRegion(dst_buffer, 0, upload, 0, size)

    // Transition destination from COPY_DEST to SHADER_RESOURCE
    barrier := d3d12.RESOURCE_BARRIER{
        Type  = .TRANSITION,
        Flags = {},
        Transition = {
            pResource   = dst_buffer,
            Subresource = d3d12.RESOURCE_BARRIER_ALL_SUBRESOURCES,
            StateBefore = {.COPY_DEST},
            StateAfter  = d3d12.RESOURCE_STATE_ALL_SHADER_RESOURCE,
        },
    }
    cmd->ResourceBarrier(1, &barrier)
    cmd->Close()

    // Submit and wait (synchronous upload — fine for init, bad for streaming)
    lists := [1]^d3d12.ICommandList{ (^d3d12.ICommandList)(cmd) }
    queue->ExecuteCommandLists(1, &lists[0])
    flush_queue(device, queue)  // waits for completion via fence
}
```


---


## Barriers & Synchronisation


DX12 has three barrier types — familiar from Vulkan's vkCmdPipelineBarrier:


- Transition barrier — change resource state (RENDER_TARGET → SHADER_RESOURCE etc.)
- Aliasing barrier — transition between two resources sharing memory
- UAV barrier — memory dependency on UAV read/write, no state change


DX12 Agility SDK 1.7+ adds Enhanced Barriers — a cleaner API matching Vulkan 1.3's VK_KHR_synchronization2. Use them if your Agility SDK is 1.710+.


```odin
transition_resource :: proc(
    cmd   : ^d3d12.IGraphicsCommandList7,
    res   : ^d3d12.IResource,
    before: d3d12.RESOURCE_STATES,
    after : d3d12.RESOURCE_STATES,
) {
    barrier := d3d12.RESOURCE_BARRIER{
        Type  = .TRANSITION,
        Transition = {
            pResource   = res,
            Subresource = d3d12.RESOURCE_BARRIER_ALL_SUBRESOURCES,
            StateBefore = before,
            StateAfter  = after,
        },
    }
    cmd->ResourceBarrier(1, &barrier)
}

// Flush a queue and wait: signal fence, CPU wait.
// Equivalent to vkQueueWaitIdle — use sparingly (stalls GPU pipeline).
flush_queue :: proc(device: ^d3d12.IDevice9, queue: ^d3d12.ICommandQueue) {
    fence      : ^d3d12.IFence
    fence_event : win32.HANDLE
    device->CreateFence(0, {}, d3d12.IFence_UUID, (^rawptr)(&fence))
    fence_event = win32.CreateEventW(nil, false, false, nil)

    queue->Signal(fence, 1)
    fence->SetEventOnCompletion(1, fence_event)
    win32.WaitForSingleObject(fence_event, win32.INFINITE)

    win32.CloseHandle(fence_event)
    fence->Release()
}
```


> **State tracking is your responsibility:** Unlike Vulkan's validation layer, the DX12 debug layer aggressively catches resource state errors. A resource in SHADER_RESOURCE state that you try to render to will generate a debug error and undefined behavior. Track states in a table parallel to your resource handles.


---


## SDL3 Window + DXGI Swapchain


SDL3 owns the OS window, the event loop, and input. DX12 never touches a window — DXGI creates the swapchain from an HWND, which SDL3 provides. This is a cleaner separation than GLFW: SDL3 handles input, DXGI handles presentation, they never conflict.


> **SDL3 vs SDL2:** SDL3 (released 2024) has a redesigned API. Window creation returns an SDL_Window* as before, but the event system uses SDL_GetEvent instead of SDL_PollEvent, and some types were renamed. Use SDL3; SDL2 is in maintenance mode.


```odin
package renderer

import sdl     "vendor:sdl3"
import win32 "core:sys/windows"
import dxgi   "vendor:directx/dxgi"
import d3d12  "vendor:directx/d3d12"

AppWindow :: struct {
    sdl_window : ^sdl.Window,
    hwnd       : win32.HWND,
    width      : u32,
    height     : u32,
}

create_window :: proc(title: cstring, w, h: i32) -> AppWindow {
    assert(sdl.Init({.VIDEO}))

    // SDL3: SDL_WINDOW_RESIZABLE instead of SDL2 flag integers
    win := sdl.CreateWindow(title, w, h, {.RESIZABLE})
    assert(win != nil)

    // Get HWND from SDL3 — SDL_GetPointerProperty with WM_INFO key
    props := sdl.GetWindowProperties(win)
    hwnd  := (win32.HWND)(sdl.GetPointerProperty(
        props,
        sdl.PROP_WINDOW_WIN32_HWND_POINTER,  // "SDL.window.win32.hwnd"
        nil,
    ))
    assert(hwnd != nil, "SDL3: failed to get HWND")

    return AppWindow{
        sdl_window = win,
        hwnd       = hwnd,
        width      = u32(w),
        height     = u32(h),
    }
}

// SDL3 event loop — returns false when the app should quit
process_events :: proc(win: ^AppWindow) -> (running: bool) {
    event : sdl.Event
    for sdl.PollEvent(&event) {
        switch event.type {
        case .QUIT:
            return false
        case .WINDOW_RESIZED:
            win.width  = u32(event.window.data1)
            win.height = u32(event.window.data2)
            // Trigger swapchain resize — see resize_swapchain() below
        case .KEY_DOWN:
            if event.key.key == .ESCAPE { return false }
        }
    }
    return true
}
```


### Creating the DXGI swapchain from the SDL3 HWND


The swapchain creation is identical to before — you just pass win.hwnd instead of a raw win32.HWND you would have constructed yourself.


```odin
SWAPCHAIN_BUFFER_COUNT :: 3

Swapchain :: struct {
    swapchain    : ^dxgi.ISwapChain4,
    back_buffers : [SWAPCHAIN_BUFFER_COUNT]^d3d12.IResource,
    rtv_heap     : ^d3d12.IDescriptorHeap,
    rtv_stride   : u32,
    frame_index  : u32,
    width        : u32,
    height       : u32,
    format       : dxgi.FORMAT,
}

create_swapchain :: proc(
    factory : ^dxgi.IFactory6,
    device  : ^d3d12.IDevice9,
    queue   : ^d3d12.ICommandQueue,
    win     : AppWindow,   // takes the SDL3-backed window
) -> Swapchain {
    sw : Swapchain
    sw.format = .R8G8B8A8_UNORM
    sw.width  = win.width
    sw.height = win.height

    desc := dxgi.SWAP_CHAIN_DESC1{
        Width       = win.width,
        Height      = win.height,
        Format      = sw.format,
        SampleDesc  = { Count = 1 },
        BufferUsage = {.RENDER_TARGET_OUTPUT},
        BufferCount = SWAPCHAIN_BUFFER_COUNT,
        SwapEffect  = .FLIP_DISCARD,
        Flags       = {.ALLOW_TEARING},
    }

    // win.hwnd comes directly from SDL3 — no other window setup needed
    sc1 : ^dxgi.ISwapChain1
    factory->CreateSwapChainForHwnd(queue, win.hwnd, &desc, nil, nil, &sc1)
    sc1->QueryInterface(dxgi.ISwapChain4_UUID, (^rawptr)(&sw.swapchain))

    // Disable DXGI's ALT+ENTER fullscreen toggle — SDL3 handles that
    factory->MakeWindowAssociation(win.hwnd, {.NO_ALT_ENTER})

    rtv_desc := d3d12.DESCRIPTOR_HEAP_DESC{
        Type           = .RTV,
        NumDescriptors = SWAPCHAIN_BUFFER_COUNT,
    }
    device->CreateDescriptorHeap(&rtv_desc, d3d12.IDescriptorHeap_UUID, (^rawptr)(&sw.rtv_heap))
    sw.rtv_stride = device->GetDescriptorHandleIncrementSize(.RTV)

    create_rtv_for_backbuffers(&sw, device)
    return sw
}

create_rtv_for_backbuffers :: proc(sw: ^Swapchain, device: ^d3d12.IDevice9) {
    rtv_base : d3d12.CPU_DESCRIPTOR_HANDLE
    sw.rtv_heap->GetCPUDescriptorHandleForHeapStart(&rtv_base)
    for i in 0..<SWAPCHAIN_BUFFER_COUNT {
        sw.swapchain->GetBuffer(u32(i), d3d12.IResource_UUID, (^rawptr)(&sw.back_buffers[i]))
        handle := d3d12.CPU_DESCRIPTOR_HANDLE{
            ptr = rtv_base.ptr + uintptr(u32(i) * sw.rtv_stride),
        }
        device->CreateRenderTargetView(sw.back_buffers[i], nil, handle)
    }
}

// Called when SDL3 fires WINDOW_RESIZED
resize_swapchain :: proc(sw: ^Swapchain, device: ^d3d12.IDevice9, w, h: u32) {
    // Release back-buffer references before resize
    for i in 0..<SWAPCHAIN_BUFFER_COUNT {
        if sw.back_buffers[i] != nil { sw.back_buffers[i]->Release() }
    }
    sw.swapchain->ResizeBuffers(SWAPCHAIN_BUFFER_COUNT, w, h, .UNKNOWN, {.ALLOW_TEARING})
    sw.width  = w
    sw.height = h
    create_rtv_for_backbuffers(sw, device)
    sw.frame_index = 0
}

present :: proc(sw: ^Swapchain) {
    sw.swapchain->Present(0, {.ALLOW_TEARING})
    sw.frame_index = sw.swapchain->GetCurrentBackBufferIndex()
}
```


> **SDL3 property key for HWND:** In SDL3 the HWND is retrieved via SDL_GetPointerProperty(props, SDL_PROP_WINDOW_WIN32_HWND_POINTER, nil). The old SDL2 SDL_SysWMinfo struct no longer exists in SDL3. This property-bag pattern is SDL3's replacement for all platform-specific window metadata.


> **RTVs live in a separate, non-shader-visible heap:** Render Target Views (RTVs) and Depth Stencil Views (DSVs) go in their own small heaps that do NOT have the SHADER_VISIBLE flag. Only your main CBV/SRV/UAV heap and the SAMPLER heap are shader-visible.


---


## Draw Commands


A DX12 frame loop: acquire → reset allocator → record → submit → present → wait for previous frame. Identical in structure to Vulkan.


```odin
FrameContext :: struct {
    cmd_alloc : ^d3d12.ICommandAllocator,
    cmd_list  : ^d3d12.IGraphicsCommandList7,
    fence_val : u64,
}

render_frame :: proc(
    gfx       : ^GfxDevice,
    sw        : ^Swapchain,
    heap      : BindlessHeap,
    root_sig  : ^d3d12.IRootSignature,
    pso       : ^d3d12.IPipelineState,
    frame     : ^FrameContext,
    fence     : ^d3d12.IFence,
    fence_val : ^u64,
    mesh      : Mesh,
) {
    idx := sw.frame_index

    // Reset allocator and re-open the command list (like vkBeginCommandBuffer)
    frame.cmd_alloc->Reset()
    frame.cmd_list->Reset(frame.cmd_alloc, nil)

    // Bind the bindless heaps — once per command list
    bind_bindless_heaps(frame.cmd_list, heap)

    // Transition backbuffer from PRESENT → RENDER_TARGET
    transition_resource(frame.cmd_list, sw.back_buffers[idx],
        d3d12.RESOURCE_STATE_PRESENT, {.RENDER_TARGET})

    // Set render target
    rtv_base : d3d12.CPU_DESCRIPTOR_HANDLE
    sw.rtv_heap->GetCPUDescriptorHandleForHeapStart(&rtv_base)
    rtv := d3d12.CPU_DESCRIPTOR_HANDLE{ ptr = rtv_base.ptr + uintptr(u32(idx) * sw.rtv_stride) }
    frame.cmd_list->OMSetRenderTargets(1, &rtv, false, nil)

    // Clear
    clear_color := [4]f32{ 0.05, 0.05, 0.08, 1.0 }
    frame.cmd_list->ClearRenderTargetView(rtv, &clear_color, 0, nil)

    // Set pipeline state and root signature
    frame.cmd_list->SetGraphicsRootSignature(root_sig)
    frame.cmd_list->SetPipelineState(pso)

    // Set viewport and scissor
    vp := d3d12.VIEWPORT{ Width = f32(sw.width), Height = f32(sw.height), MaxDepth = 1.0 }
    sr := d3d12.RECT{ right = i32(sw.width), bottom = i32(sw.height) }
    frame.cmd_list->RSSetViewports(1, &vp)
    frame.cmd_list->RSSetScissorRects(1, &sr)
    frame.cmd_list->IASetPrimitiveTopology(.TRIANGLELIST)

    // Draw — push bindless indices and call draw
    draw_mesh(frame.cmd_list, mesh, 0, mesh.material)

    // Transition backbuffer RENDER_TARGET → PRESENT
    transition_resource(frame.cmd_list, sw.back_buffers[idx],
        {.RENDER_TARGET}, d3d12.RESOURCE_STATE_PRESENT)

    frame.cmd_list->Close()

    // Submit command list
    lists := [1]^d3d12.ICommandList{ (^d3d12.ICommandList)(frame.cmd_list) }
    gfx.gfx_queue->ExecuteCommandLists(1, &lists[0])

    // Present
    present(sw)

    // Signal fence for this frame (for CPU-GPU sync on next use of this frame's allocator)
    fence_val^ += 1
    frame.fence_val = fence_val^
    gfx.gfx_queue->Signal(fence, fence_val^)
}

// At the start of frame N+SWAPCHAIN_BUFFER_COUNT, wait until frame N is done
wait_for_frame :: proc(fence: ^d3d12.IFence, frame: FrameContext) {
    if fence->GetCompletedValue() win32.CreateEventW(nil, false, false, nil)
        fence->SetEventOnCompletion(frame.fence_val, event)
        win32.WaitForSingleObject(event, win32.INFINITE)
        win32.CloseHandle(event)
    }
}
```


---


## Full Bindless Triangle — SDL3 + Slang


Putting the full stack together: SDL3 window, DX12 bindless rendering, Slang shaders compiled at runtime. The triangle vertex data lives in a GPU buffer and is fetched by SV_VertexID in the Slang vertex shader.


```hlsl
package triangle

import sdl    "vendor:sdl3"
import d3d12  "vendor:directx/d3d12"
import dxgi   "vendor:directx/dxgi"
import win32  "core:sys/windows"

Vertex :: struct { pos: [3]f32, uv: [2]f32 }

TRIANGLE_VERTS := [3]Vertex{
    { pos = {  0.0,  0.7, 0.0 }, uv = { 0.5, 0.0 } },
    { pos = {  0.7, -0.7, 0.0 }, uv = { 1.0, 1.0 } },
    { pos = { -0.7, -0.7, 0.0 }, uv = { 0.0, 1.0 } },
}

main :: proc() {
    // 1. SDL3 window — gets us HWND transparently
    app_win := create_window("DX12 Bindless Triangle", 1280, 720)

    // 2. DX12 device and queues
    gfx := create_device()

    // 3. Bindless heap
    heap := create_bindless_heap(gfx.device)

    // 4. Root signature
    root_sig := create_bindless_root_signature(gfx.device)

    // 5. Swapchain — passes the SDL3-backed AppWindow
    sw := create_swapchain(gfx.factory, gfx.device, gfx.gfx_queue, app_win)

    // 6. Upload vertex data to GPU
    vbuf := create_buffer(gfx.device, size_of(TRIANGLE_VERTS),
        .DEFAULT, {.COPY_DEST}, {})
    upload_buffer_data(gfx.device, gfx.copy_queue, vbuf,
        ([]byte)(TRIANGLE_VERTS[:3]))

    // 7. Register SRV into bindless heap
    vert_idx := register_structured_buffer_srv(
        gfx.device, &heap, vbuf, 3, size_of(Vertex))

    // 8. Compile shaders at runtime with Slang
    slang_ctx := create_slang_compiler("shaders/")
    shaders   := compile_shader(&slang_ctx, "triangle", "vert_main", "frag_main")
    pso       := create_graphics_pso(gfx.device, root_sig, shaders, .R8G8B8A8_UNORM)

    // 9. Per-frame resources
    frames    : [SWAPCHAIN_BUFFER_COUNT]FrameContext
    for &f in frames {
        gfx.device->CreateCommandAllocator(.DIRECT,
            d3d12.ICommandAllocator_UUID, (^rawptr)(&f.cmd_alloc))
        gfx.device->CreateCommandList(0, .DIRECT, f.cmd_alloc, nil,
            d3d12.IGraphicsCommandList7_UUID, (^rawptr)(&f.cmd_list))
        f.cmd_list->Close()
    }

    frame_fence : ^d3d12.IFence
    fence_val   : u64
    gfx.device->CreateFence(0, {}, d3d12.IFence_UUID, (^rawptr)(&frame_fence))

    // 10. Main loop — SDL3 event driven
    for process_events(&app_win) {
        fi := sw.frame_index
        wait_for_frame(frame_fence, frames[fi])

        mesh := Mesh{ vertex_srv_idx = vert_idx, index_count = 3 }
        render_frame(&gfx, &sw, heap, root_sig, pso,
            &frames[fi], frame_fence, &fence_val, mesh)
    }

    flush_queue(gfx.device, gfx.gfx_queue)
    sdl.Quit()
}
```


### The Slang shaders for this triangle


```hlsl
module triangle;

import "bindless";   // our shared module from section 06

struct Vertex { float3 pos; float2 uv; }

struct VSOut {
    float4 sv_pos : SV_Position;
    float2 uv     : TEXCOORD;
}

[shader("vertex")]
VSOut vert_main(uint vid : SV_VertexID) {
    let dc    = draw_constants();
    let verts = get_buffer<Vertex>(dc.vertex_buffer_idx);
    let v     = verts[vid];
    VSOut o;
    o.sv_pos = float4(v.pos, 1.0);
    o.uv     = v.uv;
    return o;
}

[shader("fragment")]
float4 frag_main(VSOut i) : SV_TARGET {
    // UV gradient — replace with texture sample once you have textures
    return float4(i.uv.x, i.uv.y, 0.5, 1.0);
}
```


> **What you built:** A fully bindless DX12 renderer with SDL3 window management and Slang shaders compiled at runtime. The vertex shader fetches data via the bindless module's get_buffer. All resource handles are uint root constants. No descriptor tables, no vertex input layout, no register() declarations.


---


## ImGui DX12 Backend Setup


Dear ImGui ships an official imgui_impl_dx12 backend. It manages its own font texture SRV, its own root signature and PSO, and issues draw calls at the end of your frame. The integration points are: one descriptor slot in your bindless heap for the font texture, and a call into ImGui's render function before you close and submit the command list.


> **ImGui descriptor heap integration:** The DX12 ImGui backend needs a SRV_HEAP and two descriptor handles: one CPU handle (to write the font SRV) and one GPU handle (to bind it in the shader). In bindless mode you allocate one slot in your existing bindless heap and hand ImGui those handles. ImGui does not create its own heap.


### Project setup


ImGui does not have official Odin bindings yet. The standard approach is to compile ImGui (the core + the DX12 + SDL3 backends) as a small C++ static library and foreign-import it from Odin. Create a imgui_impl/ folder in your project:


```bash
# From your project root — adjust paths to where you cloned imgui
cl /nologo /c /EHsc /std:c++17 \
    imgui/imgui.cpp \
    imgui/imgui_draw.cpp \
    imgui/imgui_widgets.cpp \
    imgui/imgui_tables.cpp \
    imgui/backends/imgui_impl_dx12.cpp \
    imgui/backends/imgui_impl_sdl3.cpp \
    /I imgui/ /I imgui/backends/ /I SDL3/include/ \
    /Fo imgui_impl\

lib /OUT:imgui_impl\imgui.lib imgui_impl\*.obj
```


```odin
package renderer

import  win32 "core:sys/windows"
import  d3d12 "vendor:directx/d3d12"

@(foreign_import)
imgui_lib "imgui_impl/imgui.lib"

foreign imgui_lib {
    // Core ImGui
    igCreateContext    :: proc() -> rawptr                        ---
    igDestroyContext   :: proc(ctx: rawptr)                       ---
    igGetIO            :: proc() -> ^ImGuiIO                      ---
    igNewFrame         :: proc()                                   ---
    igRender           :: proc()                                   ---
    igGetDrawData      :: proc() -> rawptr                        ---
    igBegin            :: proc(name: cstring, p_open: ^bool) -> bool ---
    igEnd              :: proc()                                   ---
    igText             :: proc(fmt: cstring, #c_vararg args: ..any) ---
    igSliderFloat      :: proc(label: cstring, v: ^f32, vmin, vmax: f32) -> bool ---

    // DX12 backend
    ImGui_ImplDX12_Init :: proc(
        device            : ^d3d12.IDevice9,
        num_frames        : i32,
        rtv_format        : u32,
        srv_heap          : ^d3d12.IDescriptorHeap,
        font_cpu_handle   : d3d12.CPU_DESCRIPTOR_HANDLE,
        font_gpu_handle   : d3d12.GPU_DESCRIPTOR_HANDLE,
    ) ---
    ImGui_ImplDX12_Shutdown   :: proc()                           ---
    ImGui_ImplDX12_NewFrame   :: proc()                           ---
    ImGui_ImplDX12_RenderDrawData :: proc(
        draw_data: rawptr,
        cmd_list : ^d3d12.IGraphicsCommandList7,
    ) ---
    ImGui_ImplDX12_CreateDeviceObjects :: proc() -> bool         ---
    ImGui_ImplDX12_InvalidateDeviceObjects :: proc()              ---

    // SDL3 backend
    ImGui_ImplSDL3_InitForD3D :: proc(window: rawptr)            ---
    ImGui_ImplSDL3_Shutdown   :: proc()                           ---
    ImGui_ImplSDL3_NewFrame   :: proc()                           ---
    ImGui_ImplSDL3_ProcessEvent :: proc(event: rawptr) -> bool  ---
}

// Minimal ImGuiIO mirror — only the fields we need to touch
ImGuiIO :: struct {
    config_flags   : i32,
    display_size   : [2]f32,
    delta_time     : f32,
    // ... many more fields; we don't need them all
}
```


---


## ImGui SDL3 Integration


ImGui needs two things from SDL3: the backend processes SDL events (mouse, keyboard, gamepad) and calls NewFrame each tick. Because SDL3 changed its event API from SDL2, use the SDL3-specific ImGui backend (imgui_impl_sdl3, not imgui_impl_sdl2).


```odin
import sdl "vendor:sdl3"

imgui_init :: proc(
    win    : AppWindow,
    device : ^d3d12.IDevice9,
    heap   : ^BindlessHeap,
) -> u32 /* returns font texture heap index */ {
    igCreateContext()

    io := igGetIO()
    io.delta_time = 1.0 / 60.0  // updated each frame from SDL timer

    // SDL3 backend — pass SDL_Window* as rawptr
    ImGui_ImplSDL3_InitForD3D(rawptr(win.sdl_window))

    // Allocate one slot in our bindless heap for ImGui's font texture
    font_slot    := bindless_alloc(heap)
    font_cpu     := cpu_handle_at(heap^, font_slot)
    font_gpu     := d3d12.GPU_DESCRIPTOR_HANDLE{
        ptr = heap.gpu_base.ptr + u64(font_slot * heap.srv_stride),
    }

    // DX12 backend — hand it our heap and the slot handles
    ImGui_ImplDX12_Init(
        device,
        SWAPCHAIN_BUFFER_COUNT,
        28,  // DXGI_FORMAT_R8G8B8A8_UNORM = 28
        heap.heap,
        font_cpu,
        font_gpu,
    )

    // Upload font atlas to GPU — needs a one-time command list execution
    ImGui_ImplDX12_CreateDeviceObjects()

    return font_slot
}

imgui_shutdown :: proc() {
    ImGui_ImplDX12_InvalidateDeviceObjects()
    ImGui_ImplDX12_Shutdown()
    ImGui_ImplSDL3_Shutdown()
    igDestroyContext(nil)
}
```


### Event forwarding


SDL3 events must be forwarded to ImGui before your own event handling. Update process_events to pass each event to the ImGui SDL3 backend first:


```odin
process_events :: proc(win: ^AppWindow) -> (running: bool) {
    event : sdl.Event
    for sdl.PollEvent(&event) {
        // Forward to ImGui FIRST — it may consume keyboard/mouse events
        ImGui_ImplSDL3_ProcessEvent(&event)

        switch event.type {
        case .QUIT:
            return false
        case .WINDOW_RESIZED:
            win.width  = u32(event.window.data1)
            win.height = u32(event.window.data2)
        case .KEY_DOWN:
            if event.key.key == .ESCAPE { return false }
        }
    }
    return true
}
```


---


## ImGui Frame Loop & Render


ImGui's render call must happen after your own draw calls but before you close the command list and transition the backbuffer to PRESENT. The DX12 backend issues its own SetGraphicsRootSignature, SetPipelineState, and draw calls — it does not affect your pipeline state after it returns.


```odin
render_frame :: proc(
    gfx       : ^GfxDevice,
    sw        : ^Swapchain,
    heap      : BindlessHeap,
    root_sig  : ^d3d12.IRootSignature,
    pso       : ^d3d12.IPipelineState,
    frame     : ^FrameContext,
    fence     : ^d3d12.IFence,
    fence_val : ^u64,
    mesh      : Mesh,
    delta_s   : f32,    // time since last frame (from SDL timer)
) {
    idx := sw.frame_index

    // ── ImGui: start new frame ──────────────────────────────────
    io := igGetIO()
    io.delta_time    = delta_s
    io.display_size  = { f32(sw.width), f32(sw.height) }
    ImGui_ImplDX12_NewFrame()
    ImGui_ImplSDL3_NewFrame()
    igNewFrame()

    // ── ImGui: build UI ─────────────────────────────────────────
    igBegin("Debug", nil)
    igText("Frame: %d", idx)
    igText("Delta: %.3f ms", delta_s * 1000)
    igEnd()

    // ── Finalize ImGui frame (does NOT render yet) ──────────────
    igRender()

    // ── DX12: record command list ───────────────────────────────
    frame.cmd_alloc->Reset()
    frame.cmd_list->Reset(frame.cmd_alloc, nil)

    // Bind bindless heaps — ImGui's backend also indexes into this heap
    bind_bindless_heaps(frame.cmd_list, heap)

    transition_resource(frame.cmd_list, sw.back_buffers[idx],
        d3d12.RESOURCE_STATE_PRESENT, {.RENDER_TARGET})

    rtv_base : d3d12.CPU_DESCRIPTOR_HANDLE
    sw.rtv_heap->GetCPUDescriptorHandleForHeapStart(&rtv_base)
    rtv := d3d12.CPU_DESCRIPTOR_HANDLE{
        ptr = rtv_base.ptr + uintptr(u32(idx) * sw.rtv_stride),
    }
    frame.cmd_list->OMSetRenderTargets(1, &rtv, false, nil)

    clear_color := [4]f32{ 0.05, 0.05, 0.08, 1.0 }
    frame.cmd_list->ClearRenderTargetView(rtv, &clear_color, 0, nil)

    frame.cmd_list->SetGraphicsRootSignature(root_sig)
    frame.cmd_list->SetPipelineState(pso)

    vp := d3d12.VIEWPORT{ Width = f32(sw.width), Height = f32(sw.height), MaxDepth = 1.0 }
    sr := d3d12.RECT{ right = i32(sw.width), bottom = i32(sw.height) }
    frame.cmd_list->RSSetViewports(1, &vp)
    frame.cmd_list->RSSetScissorRects(1, &sr)
    frame.cmd_list->IASetPrimitiveTopology(.TRIANGLELIST)

    // ── Your scene draw calls ────────────────────────────────────
    draw_mesh(frame.cmd_list, mesh, 0, mesh.material)

    // ── ImGui render — AFTER your draws, BEFORE close/present ───
    // The ImGui DX12 backend sets its own root sig and PSO here.
    // It uses the bindless heap we bound at the top (font texture slot).
    ImGui_ImplDX12_RenderDrawData(igGetDrawData(), frame.cmd_list)

    // ── Transition to PRESENT ────────────────────────────────────
    transition_resource(frame.cmd_list, sw.back_buffers[idx],
        {.RENDER_TARGET}, d3d12.RESOURCE_STATE_PRESENT)

    frame.cmd_list->Close()

    lists := [1]^d3d12.ICommandList{ (^d3d12.ICommandList)(frame.cmd_list) }
    gfx.gfx_queue->ExecuteCommandLists(1, &lists[0])

    present(sw)

    fence_val^ += 1
    frame.fence_val = fence_val^
    gfx.gfx_queue->Signal(fence, fence_val^)
}
```


### Delta time from SDL3


SDL3 provides high-resolution timers via SDL_GetTicksNS() (nanoseconds) or SDL_GetTicks() (milliseconds). Use them to compute the delta you pass to ImGui and your own simulation:


```odin
// In your main loop:
last_ns   : u64 = sdl.GetTicksNS()

for process_events(&app_win) {
    now_ns  := sdl.GetTicksNS()
    delta_s := f32(now_ns - last_ns) * 1e-9
    last_ns  = now_ns

    fi := sw.frame_index
    wait_for_frame(frame_fence, frames[fi])
    render_frame(&gfx, &sw, heap, root_sig, pso,
        &frames[fi], frame_fence, &fence_val, mesh, delta_s)
}
```


> **Re-bind heaps after ImGui if you need to resume bindless draws:** ImGui_ImplDX12_RenderDrawData calls SetDescriptorHeaps internally. If you have additional draws after ImGui in the same command list (unusual but possible), call bind_bindless_heaps again to restore your heap bindings.


> **Full stack summary:** You now have the complete modern DX12 stack: SDL3 creates the OS window and feeds events to both your app and ImGui. DXGI grabs the HWND for the swapchain. DX12 handles GPU resources with a bindless heap. Slang compiles shader modules with no register declarations. ImGui renders its debug UI into the same command list using a pre-allocated slot in your bindless heap. Every piece is cleanly separated — swap any layer independently.


### Further reading


From this foundation the natural next steps are: GPU-driven indirect draw with ExecuteIndirect, mesh shaders (Slang supports them natively), DX12 raytracing via the TLAS/BLAS API, and Work Graphs for GPU-generated work. All sit on top of the bindless heap and root signature pattern you already have.


---
