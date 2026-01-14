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

'''cpp
XGetXCBConnection()
'''

in the crash stack trace — even though you are running Wayland.

## The Correct Fix

You must explicitly tell BGFX that the window handle is a Wayland handle.

Step 1: Get native Wayland handles from SDL

'''cpp
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

'''

Step 2: Tell BGFX explicitly that this is Wayland

'''cpp
    bgfx::PlatformData pd{};
    pd.ndt = display;   // native display
    pd.nwh = surface;   // native window
    pd.type = bgfx::NativeWindowHandleType::Wayland; // IMPORTANT

    bgfx::setPlatformData(pd);
'''

Step 3: Initialize BGFX normally

'''cpp
    bgfx::Init init;
    init.type = bgfx::RendererType::Vulkan;
    init.platformData = pd;
    init.resolution.width = 640;
    init.resolution.height = 480;
    init.resolution.reset = BGFX_RESET_VSYNC;

    if (!bgfx::init(init))
        throw std::runtime_error("bgfx init failed");
}
'''

## Why This Works

- NativeWindowHandleType::Wayland prevents BGFX from guessing
- BGFX uses the correct Vulkan surface creation path
- No X11 functions are called
- No crash occurs

## How to Verify You Are Running on Wayland

Before initializing BGFX:

'''cpp
printf("SDL video driver: %s'''", SDL_GetCurrentVideoDriver());
'''

Expected output:

'''
wayland
'''

If it prints x11, then SDL is not running on Wayland.

## Common Mistakes to Avoid

- Assuming BGFX auto-detects Wayland
- Passing Wayland handles without setting pd.type
- Mixing X11 and Wayland handles
- Relying on non-existent compile-time macros

# TL;DR

If BGFX crashes on Wayland and mentions X11:
- BGFX is guessing the wrong platform
- Explicitly set:
'''cpp
pd.type = bgfx::NativeWindowHandleType::Wayland;
'''