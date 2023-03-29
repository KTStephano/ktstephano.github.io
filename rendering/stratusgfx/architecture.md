---
permalink: /rendering/stratusgfx/architecture
---

This is a high level overview of the architecture of the engine as of version 0.9. The project itself is not finished and is still considered pre-release/beta-release. Expect bugs and instability. I have only been able to test it on one computer's hardware (Ryzen 5 1600, Nvidia GTX 1060).

## StratusGFX Core Design

Stratus is primarily a realtime 3D rendering engine with a few additional aspects that would be found in a more general purpose engine.

Things it does have:

* Compiles to libStratusEngine which can be linked against for easily setting up separate apps
* Realtime 3D renderer
* Resource management for textures and models
* Async asset loading
* Entity-Component System (ECS)
* Certain utilities such as pool allocators and multi threading
* A main engine class responsible for startup, main loop, shutdown

Things it does *not* have:

* General purpose editor
* Sound engine
* Physics engine
* Scripting
* Engine-specific file format (instead it accepts .obj, .fbx, .gltf)

So for a full-featured project the easiest path would probably be to either integrate Stratus into an existing engine or build around it (e.g. add Bullet Physics, etc.).

## License

Source code from tags/releases 0.1 to 0.9.4 have been released under MPL 2.0 now that the repo is public.

## External Libraries Used

-> [Assimp](https://github.com/assimp/assimp/tree/5a4f7c0a375fd2acc3bda9414d21efe711346ede) for model loading

-> [SDL](https://www.libsdl.org) for cross platform window and input

-> [GLM](https://github.com/g-truc/glm) for math

-> [GL3W](https://github.com/skaslev/gl3w/tree/3755745085ac2e865fd22270cfe9169c26640f70) for to load core GL profile and extensions

-> [Catch2](https://github.com/catchorg/Catch2/tree/432d03d1aab8472a0813c34a7f0e2e1a2c585d22) for unit testing

-> [CMake](https://cmake.org) to manage build system

-> [Meshoptimizer](https://github.com/zeux/meshoptimizer/tree/f734fd572aed5bf76e84d9ed62ca6f4f6c47d84e) for LOD generation

-> [STB](https://github.com/nothings/stb/tree/5736b15f7ea0ffb08dd38af21067c314d6a3aae9) for loading image files

## Code Split

The code is split into two spaces: engine and application.

Everything under Examples/ and Tests/ is considered application code.

Everything under Source/ is considered engine code. Source/Engine compiles to libStratusEngine.

**Application**

All code which falls under Examples/ and Tests/ is considered application code. These pieces of code link against libStratusEngine.

In order for a piece of code to integrate itself as application code, it implements the interface found in Source/Engine/StratusApplication.h.

As an example, here is the full source of ExampleEnv00:

{% highlight c++ %}
#include "StratusEngine.h"
#include "StratusLog.h"

// Create ExampleApp which implements the Application interface
class ExampleApp : public stratus::Application {
public:
    virtual ~ExampleApp() = default;

    const char * GetAppName() const override {
        // ExampleApp00 will show up as the name of the window
        return "ExampleApp00";
    }

    // Core engine design: by the time this function is called,
    // the application can safely use anything within the engine
    // without wondering if it has been initialized or not
    virtual bool Initialize() override {
        return true; // success
    }

    virtual stratus::SystemStatus Update(const double deltaSeconds) override {
        STRATUS_LOG 
            << "Successfully entered ExampleApp::Update! Delta seconds = " 
            << deltaSeconds 
            << std::endl;
        
        // Tells the engine to end after this frame
        return stratus::SystemStatus::SYSTEM_SHUTDOWN;
    }

    // Core engine design: this needs to be called first during shutdown 
    // so that any engine code the application calls won't crash
    virtual void Shutdown() override {
        STRATUS_LOG << "Successfully entered ExampleApp::ShutDown()" 
                    << std::endl;
    }
};

// Sets up the main loop and starts the engine
// with a focus on ExampleApp class
STRATUS_ENTRY_POINT(ExampleApp)
{% endhighlight %}

Aside from `STRATUS_ENTRY_POINT`, there is also `STRATUS_INLINE_ENTRY_POINT(ApplicationClass, numArgs, argList)` which takes numArgs and argList from the command line. The use case for this is if you already have your own main function which does other stuff, you can use INLINE_ENTRY which does not create its own main function. The integration tests under Tests/ make use of this.

**Engine**

Everything under Source/Engine/ compiles to libStratusEngine and is considered engine code.

The engine itself starts with Source/Engine/StratusEngine.h which is responsible for startup, shutdown and main loop execution.

Surrounding the engine is a series of classes that implement the SystemModule interface found in Source/Engine/StratusSystemModule.h. Code which is considered a SystemModule includes the resource manager, task (thread) manager, material manager, renderer frontend, entity component manager, and logging. All SystemModules are initialized, managed, and later shut down by the engine.

Most of the scene objects in the engine are part of the Entity-Component System. They are Entities and their data and behavior is determined by which components are added.

## Utilities

-> Async + Task System for multithreading

-> Thread class which wraps around C++ thread with added functionality

-> Concurrent hash map for multithreaded hash map read/write

-> Certain math utils built on top of GLM (StratusUtils.h and StratusMath.h)

-> Single and multi threaded pool allocators