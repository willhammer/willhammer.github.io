# BGFX on Wayland (Linux): Fixing a Crash Caused by X11 Assumptions

## Overview
When running a game engine on Linux with Wayland, you may encounter a crash during BGFX initialization that looks like this:

'''
BX Attempting Xlib surface creation.
Signal: SIGSEGV (address not mapped to object)
'''

This happens even though your system is clearly running Wayland (for example, KDE Plasma on Wayland).
This article explains:
- Why this crash happens
- Why BGFX mistakenly tries to use X11
- The correct and minimal fix
- How to verify your setup

No prior knowledge of Wayland internals is required.

## Background: SDL, BGFX, and Wayland
SDL creates the application window and talks to the operating system (Wayland or X11).

BGFX renders graphics but does not create windows.
Instead, it requires native platform handles created by SDL.

On Linux, BGFX needs to know:
- Which window system is used (Wayland or X11)
- Which native handles belong to that system

## The Root Cause of the Crash

On Linux, if BGFX is not told explicitly which window system it is using, it tries to guess.
If the platform type is not set, BGFX often defaults to X11.

On a Wayland system:
- X11 handles are nullptr
- BGFX still attempts X11 Vulkan surface creation
- This leads to a null pointer dereference
- Result: SIGSEGV

This is why you may see:



XGetXCBConnection()



in the crash stack trace — even though you are running Wayland.

## The Correct Fix

You must explicitly tell BGFX that the window handle is a Wayland handle.

Step 1: Get native Wayland handles from SDL

~~~cpp
#include <wayland-client.h>
#include <SDL3/SDL.h>
#include <bgfx/bgfx.h>
#include <stdexcept>

void Renderer::init(SDL_Window* window)
{
    auto* props = SDL_GetWindowProperties(window);

    wl_display* display = static_cast<wl_display*>(
        SDL_GetPointerProperty(
            props,
            SDL_PROP_WINDOW_WAYLAND_DISPLAY_POINTER,
            nullptr));

    wl_surface* surface = static_cast<wl_surface*>(
        SDL_GetPointerProperty(
            props,
            SDL_PROP_WINDOW_WAYLAND_SURFACE_POINTER,
            nullptr));

    if (!display || !surface)
        throw std::runtime_error("Wayland display or surface not available");
~~~

Step 2: Tell BGFX explicitly that this is Wayland

~~~cpp
    bgfx::PlatformData pd{};
    pd.ndt = display;   // native display
    pd.nwh = surface;   // native window
    pd.type = bgfx::NativeWindowHandleType::Wayland; // IMPORTANT

    bgfx::setPlatformData(pd);
~~~

Step 3: Initialize BGFX normally

~~~cpp
    bgfx::Init init;
    init.type = bgfx::RendererType::Vulkan;
    init.platformData = pd;
    init.resolution.width = 640;
    init.resolution.height = 480;
    init.resolution.reset = BGFX_RESET_VSYNC;

    if (!bgfx::init(init))
        throw std::runtime_error("bgfx init failed");
}
~~~

## Why This Works

- NativeWindowHandleType::Wayland prevents BGFX from guessing
- BGFX uses the correct Vulkan surface creation path
- No X11 functions are called
- No crash occurs

## How to Verify You Are Running on Wayland

Before initializing BGFX:

~~~cpp
printf("SDL video driver: %s'''", SDL_GetCurrentVideoDriver());
~~~

Expected output:

~~~bash
wayland
~~~

If it prints x11, then SDL is not running on Wayland.

## Common Mistakes to Avoid

- Assuming BGFX auto-detects Wayland
- Passing Wayland handles without setting pd.type
- Mixing X11 and Wayland handles
- Relying on non-existent compile-time macros

## TL;DR

If BGFX crashes on Wayland and mentions X11:
- BGFX is guessing the wrong platform
- Explicitly set:

~~~cpp
pd.type = bgfx::NativeWindowHandleType::Wayland;
~~~

## P.S.:
I've initially prompted ChatGPT for a solution. It gave me the following answer:

~~~
Required build conditions
BGFX must be built with Wayland enabled

Check that one of these is true:
- BGFX_CONFIG_USE_WAYLAND=1
- BGFX_CONFIG_USE_X11=0 (optional but recommended)

If BGFX was built with only X11 enabled, it will always attempt Xlib surface creation, even on Wayland.
~~~

Upon searching for online documentation for these compile flags, I've found nothing. These flags don't exist.
I've then pasted the presumed BGFX_CONFIG_USE_WAYLAND flag as prompt to Gemini. Got the following response:

~~~
BGFX_CONFIG_USE_WAYLAND
is a configuration macro used in the bgfx graphics library to enable support for the Wayland display protocol on Linux systems.

- Function: When defined as 1 during the build process, it instructs bgfx to use Wayland-specific code paths for window creation and surface management instead of the traditional X11 protocol.

- Usage: It is typically passed as a compiler definition (e.g., -DBGFX_CONFIG_USE_WAYLAND=1) or toggled within the project's build configuration files (such as GENie or CMake).

- Requirements: Enabling this flag requires the presence of Wayland development headers and libraries on the host system. It is often used in conjunction with BGFX_CONFIG_RENDERER_VULKAN or BGFX_CONFIG_RENDERER_OPENGL, as these backends are commonly used with Wayland.

- Context: As modern Linux distributions shift toward Wayland by default, this flag allows developers to build native, high-performance applications that bypass the XWayland compatibility layer.

For detailed integration instructions, you can refer to the bgfx documentation or the project's GitHub repository.
~~~

They both went into the same kind of assumption hole! Which is puzzling to me. Is this due to similar structure? Are there algorithmic components in determining the chain of thought? 

When confronted about the error, ChatGPT has acknowledged that the flags do not exist. 
The reasoning error was explained thus:

~~~
Why I mentioned flags that do not exist
    This is the important self-reflection part.

What happened technically

I conflated two different generations / styles of BGFX platform handling:    
    1. Older or internal-style conditional compilation patterns
        Common in cross-platform C++ engines
        Frequently named *_USE_X11, *_USE_WAYLAND, etc.
    2. BGFX’s actual modern design
        Runtime selection via PlatformData::type
        Compile-time only guards for OS families, not window systems

BGFX deliberately moved away from “compile-time window system selection” toward runtime explicit declaration, precisely to avoid this class of ambiguity.

I incorrectly projected a plausible but incorrect internal model onto BGFX instead of strictly grounding the explanation in the repository’s current reality.
That is a reasoning error, not a missing fact lookup.

Why it sounded convincing (but was wrong)
    - The behavior (BGFX choosing X11) looked like a compile-time exclusion
    - Many engines do use such macros
    - The Vulkan backend code paths resemble that pattern superficially
    - So the explanation matched the symptoms, but not the actual mechanism
    - This is a classic failure mode: mechanistic plausibility overriding verification.
~~~

Is this a bias of general purpose models, to try to "intuit" various things, as a component of their reasoning? Is it safer to use code-oriented models when discovering or investigating in unkown territory?