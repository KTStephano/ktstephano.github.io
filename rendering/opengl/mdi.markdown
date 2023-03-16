---
permalink: /rendering/opengl/mdi
---

![ogl](/assets/opengl.png)

This is part of a tutorial series teaching advanced modern OpenGL. To use all features to their fullest you will need to target OpenGL 4.6. These articles assume you have familiarity with OpenGL.

With older versions of OpenGL we had access to draw commands such as `glDrawArrays`, `glDrawElements`, or `glDrawElementsInstanced`. Improving performance would be about doing things like maximizing the use of instancing.

With the latest versions of OpenGL the API introduced the concept of Multi-Draw Indirect (MDI). MDI opens the door to a totally new approach to generating draw commands which supports parallel command creation (either with multiple CPU cores or even on the GPU) and at the same time reduces the amount of time the CPU has to spend submitting draw calls to the driver. It also allows us to reuse draw commands from previous frames if nothing has changed.

## How Multi-Draw Indirect (MDI) Differs From Old Methods

The primary difference is that instead of submitting each draw call manually (instanced or non-instanced, indexed or non-indexed), draw calls are instead packed into a GPU buffer which gets bound to the GL_DRAW_INDIRECT_BUFFER target before drawing. This means that instead of submitting multiple draw commands in a row, we instead submit every draw call that's currently in the bound GPU buffer at once.

This is where the option of parallelizing the draw command generation comes into play. Since OpenGL doesn't actually do anything with the draw command buffer until we bind it and submit it, multiple threads can write data to the buffer ahead of time before we synchronize and draw. The rendering thread will always submit the draw commands, but multiple threads can write to the draw command buffer before this happens.

Another important thing is that the GL_DRAW_INDIRECT_BUFFER is backed by GPU memory and can be used like a regular SSBO. This means you can pass it into a compute shader and have it manipulate the draw commands directly. The GPU can now generate its own work!

**Summary:**
* Multiple threads or the GPU can write to a draw command buffer
* The rendering thread synchronizes and submits the commands in the buffer with a single API call.

## A Look At The API

There are two functions which we will be dealing with when it comes to MDI:

{% highlight c++ %}
void glMultiDrawArraysIndirect(
    GLenum mode,
    const void *indirect,
    GLsizei drawcount,
    GLsizei stride
);

void glMultiDrawElementsIndirect(
    GLenum mode,
    GLenum type,
    const void *indirect,
    GLsizei drawcount,
    GLsizei stride
);
{% endhighlight %}

The difference between them is that the first performs non-indexed multi-draw while the second perforns indexed multi-draw.

**Parameters**

mode | GL_POINTS, GL_LINE_STRIP, GL_LINE_LOOP, GL_LINES, GL_LINE_STRIP_ADJACENCY, GL_LINES_ADJACENCY, GL_TRIANGLE_STRIP, GL_TRIANGLE_FAN, GL_TRIANGLES, GL_TRIANGLE_STRIP_ADJACENCY, GL_TRIANGLES_ADJACENCY, and GL_PATCHES
type | GL_UNSIGNED_BYTE, GL_UNSIGNED_SHORT, or GL_UNSIGNED_INT depending on the underlying type of the bound element buffer
indirect | Interpreted as a byte offset into the currently bound GL_DRAW_INDIRECT_BUFFER to start reading data
drawcount | Number of draw commands in the buffer currently bound to GL_DRAW_INDIRECT_BUFFER.
stride | Byte offset between the end of one draw command and the start of the next. If 0, the data in the draw command buffer is tightly packed. If greater than 0 then the structure contains extra data that the application plans to use but OpenGL needs to skip over.

## A Look At The Draw Command Structs

There are two draw command structs - one for MultiDrawArrays and one for MultiDrawElements.

{% highlight c++ %}
// Struct for MultiDrawArrays
typedef  struct {
    unsigned int  count;
    unsigned int  instanceCount;
    unsigned int  firstVertex;
    unsigned int  baseInstance;
    // Optional user-defined data goes here - if nothing, stride is 0
} DrawArraysIndirectCommand;
// sizeof(DrawArraysIndirectCommand) == 16

// Struct for MultiDrawElements
typedef  struct {
    unsigned int  count;
    unsigned int  instanceCount;
    unsigned int  firstIndex;
    int           baseVertex;
    unsigned int  baseInstance;
    // Optional user-defined data goes here - if nothing, stride is 0
} DrawElementsIndirectCommand;
// sizeof(DrawElementsIndirectCommand) == 20
{% endhighlight %}

Below is a table detailing what each member is supposed to be interpreted as.

**Struct Members**

count | For `DrawArraysIndirect` this is interpreted as the number of vertices. For `DrawElementsIndirectCommand` this is interpreted as the number of indices.
instanceCount | Number of instances where 0 effectively disables the draw command. Setting instances to 0 is useful if you have an initial list of draw commands and want to disable them by the CPU or GPU during a frustum culling step for example.
firstVertex | For `DrawArraysIndirect` this is an index (not byte) offset into a bound vertex array to start reading vertex data.
firstIndex | For `DrawElementsIndirectCommand` this is an index (not byte) offset into the bound element array buffer to start reading index data.
baseVertex | For `DrawElementsIndirectCommand` this is interpreted as an addition to whatever index is read from the element array buffer.
baseInstance | If using instanced vertex attributes, this allows you to offset where the instanced buffer data is read from. The formula for the final instanced vertex attrib offset is `floor(instance / divisor) + baseInstance`. If you are **not** using instanced vertex attributes then you can use this member for whatever you want, for example storing a material index that you will manually read from in the shader.

Here is some pseudocode for how gl_VertexID could be set by the driver.

{% highlight c++ %}
// When using DrawArraysIndirectCommand
for each (DrawArraysIndirectCommand cmd : GL_DRAW_INDIRECT_BUFFER) {
    for (uint i = 0; i < cmd.count; ++i) {
        int gl_VertexID = cmd.firstVertex + i;
    }
}

// When using DrawElementsIndirectCommand when element array buffer is
// using unsigned ints
unsigned int * indices = (unsigned int *)ELEMENT_ARRAY_BUFFER;
for each (DrawElementsIndirectCommand cmd : GL_DRAW_INDIRECT_BUFFER) {
    for (uint i = 0; i < cmd.count; ++i) {
        int gl_VertexID = indices[cmd.firstIndex + i] + cmd.baseVertex;
    }
}
{% endhighlight %}

## Available Built-In Input Variables

The following table includes the relevant built-in inputs that are available for the vertex shader stage.

gl_VertexID | Vertex with firstVertex offset if `DrawArraysIndirectCommand`, vertex index with first index and base vertex offset if `DrawArraysIndirectCommand`
gl_InstanceID | Current instance whenever instanceCount > 1, else 0
gl_DrawID | The current draw command index we are on inside of the GL_DRAW_INDIRECT_BUFFER - so if you submitted 30 draw commands in the buffer, this value will range from 0 to 29. Useful for a situation such as needing to access a different transform matrix depending on the current draw command number.
gl_BaseVertex | Base vertex of current draw command
gl_BaseInstance | Base instance of current draw command (can use this to pass in any integer data you want if not using instanced vertex attributes)

## What First Vertex/Index and Base Vertex Are Useful For

Having these allows us to combine multiple mesh vertices and vertex indices into larger buffers that the draw commands can reference. For example, assume we have 3 meshes and we have merged their data into two large buffers (one for vertices, one for indices):

![png](/assets/vertices_indices.png)

We see here that the meshes now sit in adjacent memory and we will need to specify offsets to access the data. If using `DrawArraysIndirectCommand` we will specify that with the firstVertex offset so that we can correctly access the vertex data for each mesh per draw call.

Notice that the index buffer entries all reference the same indices 0 through 3, but the actual vertex buffer now goes from 0 through 9 since we put the vertices for all 3 meshes into it. If using `DrawElementsIndirectCommand` we specify the offset with a combination of firstIndex (where in the index buffer to start reading for the current draw call) and baseVertex (what the index should be offset by before using it to access the vertex in the vertex buffer for the current draw call).

**What mesh 1,2,3 draw commands would look like**
{% highlight c++ %}
DrawElementsIndirectCommand mesh1Cmd = {
    .count = 6,         // 6 indices total
    .instanceCount = 1,
    .firstIndex = 0,    // First in the index array
    .baseVertex = 0,    // First in the vertex array
    .baseInstance = 0
};

// For this command,
// initialIndex = indices[.firstIndex] + .baseVertex
//              = indices[6] + 3
//              = 1 + 3 = 4
DrawElementsIndirectCommand mesh2Cmd = {
    .count = 6,         // 6 indices total
    .instanceCount = 1,
    .firstIndex = 6,    // Starts at location 6 in index array
    .baseVertex = 3,    // Starts at location 3 in vertices array
    .baseInstance = 0
};

// For this command,
// initialIndex = indices[.firstIndex] + .baseVertex
//              = indices[12] + 6
//              = 0 + 6 = 6
DrawElementsIndirectCommand mesh3Cmd = {
    .count = 8,         // 8 indices total
    .instanceCount = 1,
    .firstIndex = 12,   // Starts at location 12 in index array
    .baseVertex = 6,    // Starts at location 6 in vertices array
    .baseInstance = 0
};
{% endhighlight %}

## Creating and Using Draw Command Buffers

This is done in the same way as an SSBO is created except now we use one of the two structs defined above. For example:

{% highlight c++ %}
std::vector<DrawElementsIndirectCommand> commands;

.. code that fills commands vector with DrawElementsIndirectCommand elements ..
// (for example)
commands.push_back(mesh1Cmd);
commands.push_back(mesh2Cmd);
commands.push_back(mesh3Cmd);

GLuint drawCmdBuffer;
glCreateBuffers(1, &drawCmdBuffer);

glNamedBufferStorage(drawCmdBuffer, 
                     sizeof(DrawElementsIndirectCommand) * commands.size(), 
                     (const void *)commands.data(), 
                     GL_DYNAMIC_STORAGE_BIT);

{% endhighlight %}

**Binding and Drawing**

{% highlight c++ %}

glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, indicesBuffer);
glBindBuffer(GL_DRAW_INDIRECT_BUFFER, drawCmdBuffer);

// This will submit all commands.size() draw commands in the currently
// bound buffer
glMultiDrawElementsIndirect(
    GL_TRIANGLES, 
    GL_UNSIGNED_INT, // Type of data in indicesBuffer
    (const void *)0, // No offset into draw command buffer
    commands.size(),
    0                // No stride as data is tightly packed
);

glBindBuffer(GL_DRAW_INDIRECT_BUFFER, 0);
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, 0);

{% endhighlight %}

## Accessing Draw Command Buffers On The GPU

Since the draw command buffer is just a regular GPU buffer, it can be bound to both the GL_DRAW_INDIRECT_BUFFER and the GL_SHADER_STORAGE_BUFFER targets.

*c++*
{% highlight c++ %}
// 3 matches (binding = 3) in compute shader below
glBindBufferBase(GL_SHADER_STORAGE_BUFFER, 3, drawCmdBuffer);
glUniform1ui(
    glGetUniformLocation(computeShader, "numDrawCommands"), commands.size()
);
{% endhighlight %}

*compute shader*
{% highlight glsl %}
#version 460 core
layout (local_size_x = 4, local_size_y = 4, local_size_z = 1) in;

// Matches the C++ definition
struct DrawElementsIndirectCommand {
    uint  count;
    uint  instanceCount;
    uint  firstIndex;
    int   baseVertex;
    uint  baseInstance;
};

// Buffer for both read and write access
layout (std430, binding = 3) buffer ssbo1 {
    DrawElementsIndirectCommand drawCommands[];
};

uniform uint numDrawCommands;

// Now the compute shader can read and write from the buffer however
// way that it needs to

{% endhighlight %}

## Reuse For Multiple Frames

It is possible that a scene could have large amounts of static geometry that is set up all at once and then left alone for many frames. In a case like this you could opt to generate a draw command buffer specifically for this static geometry and then reuse it over the course of many frames to save on performance. This would mean for the static portion of the scene the CPU would just bind the static draw command buffer without regenerating it, submit draw calls, and then move on and handle the dynamic parts of the scene separately.

## Learn OpenGL Fundamentals
* [https://learnopengl.com](https://learnopengl.com)

## References
* [https://registry.khronos.org/OpenGL-Refpages/gl4/html/glMultiDrawArraysIndirect.xhtml](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glMultiDrawArraysIndirect.xhtml)
* [https://registry.khronos.org/OpenGL-Refpages/gl4/html/glMultiDrawElementsIndirect.xhtml](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glMultiDrawElementsIndirect.xhtml)
* [https://www.khronos.org/opengl/wiki/Vertex_Shader/Defined_Inputs](https://www.khronos.org/opengl/wiki/Vertex_Shader/Defined_Inputs)
* [3D Graphics Rendering Cookbook](https://www.amazon.com/Graphics-Rendering-Cookbook-comprehensive-algorithms/dp/1838986197)
* [OpenGL SuperBible](https://www.amazon.com/OpenGL-Superbible-Comprehensive-Tutorial-Reference/dp/0672337479)