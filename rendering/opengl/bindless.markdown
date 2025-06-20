---
permalink: /rendering/opengl/bindless
title: Bindless Textures
---

![ogl](/assets/opengl.png)

This is part of a tutorial series teaching advanced modern OpenGL. To use all features to their fullest you will need to target OpenGL 4.6. These articles assume you have familiarity with OpenGL.

In this tutorial we will be looking at a method for eliminating the need to bind a texture to a uniform texture unit. If you haven't already make sure to check out the [Shader Storage Buffer Objects (SSBOs)](/rendering/opengl/ssbos) and [Programmable Vertex Pulling](/rendering/opengl/prog_vtx_pulling) tutorials since we will use both of these here.

## What Are Bindless Textures?

OpenGL 4.5 only requires a minimum of 16 texture units to be available to a shader for use, though many drivers support 32 or more. This is a very difficult limit to deal with if we are trying to do something like merge many different draw calls with different material textures, or access many shadow maps in a deferred lighting stage. There were two main ways to get around this in the past:

* Use array textures to combine multiple textures of the same dimensions
* Use texture atlases to allow one large texture to represent many smaller textures

Bindless textures are a 3rd alternative to this. Instead of explicitly binding a texture to a texture unit, bindless textures allow us to manually mark the texture as resident on the GPU and then pass its handle to a shader using a uniform buffer or shader storage buffer.

There is also a possible performance gain with this approach. If your renderer is currently spending a lot of time binding and unbinding textures between different draw calls, a bindless approach can reduce a lot of driver overhead. This is because your application can now figure out which textures are needed at the beginning of the frame, mark them all as resident at the sae time and then use those textures for any number of draw calls for the remainder of the frame.

## Downsides Of Bindless Textures

There are two main downsides of bindless textures:

1) Renderdoc does not support them at the time of writing so you will need to use a graphics debugger that does such as Nvidia NSight 

2) Bindless textures never made it into the core of OpenGL so it is still an extension even with 4.6. Nvidia and AMD both have a lot of hardware that is fully supportive of it, but outside of those two you will need to double check if the vendor supports it.

## A Look At The API

For this tutorial we will be focused on 3 main API functions which we will go through now.

**Retrieving Texture Handles**
{% highlight c++ %}
GLuint64 glGetTextureHandleARB(GLuint texture​);
{% endhighlight %}

The purpose of this function is to take a valid texture and return a 64-bit handle that represents that texture. This handle can be inserted directly into uniform buffers or shader storage buffers.

If you were to call this function multiple times with the same texture, it will always return the same handle. So what you can do is call this once after the texture is created and then use the handle for the entire lifetime of the texture without having to call this function again.

***Important!*** Khronos states the following on their wiki: *"Once a handle is created for a texture/sampler, none of its state can be changed. For Buffer Textures, this includes the buffer object that is currently attached to it (which also means that you cannot create a handle for a buffer texture that does not have a buffer attached). Not only that, in such cases the buffer object itself becomes immutable; it cannot be reallocated with glBufferData. Though just as with textures, its storage can still be mapped and have its data modified by other functions as normal."*

So the general guideline is to perform all initial setup that you will need for the texture or buffer texture, then once you are done call this. From then on you will only be able to change the texture data but no other state related to the texture.

**Making Texture Handles Resident**
{% highlight c++ %}
// handle is what is returned by glGetTextureHandleARB
void glMakeTextureHandleResidentARB(GLuint64 handle​);
{% endhighlight %}

Since we won't be binding to uniform texture units anymore, we need to be able to tell the driver which handles we plan to use before we use them. Otherwise it has no way of knowing.

Once you make a texture handle resident, it remains resident until you explicitly tell the driver to make it non-resident. This means that if you have a certain group of textures that will be around for the entire lifetime of your program, you could make them resident at the start and then leave them resident until the program is ending.

**Making Texture Handles Non-Resident**
{% highlight c++ %}
// handle is what is returned by glGetTextureHandleARB
void glMakeTextureHandleNonResidentARB(GLuint64 handle​);
{% endhighlight %}

This allows you to tell the driver that the handle is no longer being used by any shaders and can be taken off the residency list.

## Example: 1600 textures in a single shader

To show how to use this we are going to randomly generate 1600 textures (one for each of 1600 cube instances) and make them all resident for the shader to use. We will only be using a single instanced draw call to draw 1600 cubes.

First we will generate the data for the cube and insert its vertices into an SSBO.

{% highlight c++ %}
struct VertexData {
    float position[3];
    float uv[2];
    float normal[3];
};

// In main or some other function
std::vector<VertexData> vertices;

... code to insert a single cube data into vertices ... 

const int numVertices = data.size();
GLuint verticesBuffer;

glCreateBuffers(1, &verticesBuffer);
glNamedBufferStorage(
    verticesBuffer,
    sizeof(VertexData) * vertices.size(),
    (const void *)vertices.data(),
    GL_DYNAMIC_STORAGE_BIT
);

{% endhighlight %}

Next we will create 1600 different transforms, one for each cube instance. These will also be inserted into a SSBO.

{% highlight c++ %}
std::vector<glm::mat4> instancedMatrices;

// Each one will be 5 units apart from the others in the x/y direction
for (size_t x = 0; x < 200; x += 5) {
    for (size_t y = 0; y < 200; y += 5) {
        glm::mat4 mat(1.0f);
        mat[3].x = float(x);
        mat[3].y = float(y);
        mat[3].z = 0.0f;
        instancedMatrices.push_back(std::move(mat));
    }
}

const int numInstances = instancedMatrices.size();
GLuint modelMatricesBuffer;

glCreateBuffers(1, &modelMatricesBuffer);
glNamedBufferStorage(
    modelMatricesBuffer,
    sizeof(glm::mat4) * instancedMatrices.size(),
    (const void *)instancedMatrices.data(),
    GL_DYNAMIC_STORAGE_BIT
);
{% endhighlight %}

Now we will randomly generate 1600 textures.

**Setting Up Vectors**
{% highlight c++ %}
std::vector<GLuint> textures;
std::vector<GLuint64> textureHandles;

// Each texture has a width and height of 32x32, 
// with 3 channels of data (RGB) per pixel
const size_t textureSize = 32 * 32 * 3;
unsigned char textureData[textureSize];
{% endhighlight %}

We will have a vector for the textures and for the returned handles.

**Generating Textures And Retrieving Handles**
{% highlight c++ %}
for (int i = 0; i < numInstances; ++i) {
    const unsigned char limit = unsigned char(rand() % 231 + 25);
    // Randomly generate an unsigned char per RGB channel
    for (int j = 0; j < textureSize; ++j) {
        textureData[j] = unsigned char(rand() % limit);
    }

    GLuint texture;
    glCreateTextures(GL_TEXTURE_2D, 1, &texture);
    glTextureStorage2D(texture, 1, GL_RGB8, 32, 32);
    glTextureSubImage2D(
        texture, 
        // level, xoffset, yoffset, width, height
        0, 0, 0, 32, 32, 
        GL_RGB, GL_UNSIGNED_BYTE, 
        (const void *)&textureData[0]);
    glGenerateTextureMipmap(texture);

    // Retrieve the texture handle after we finish creating the texture
    const GLuint64 handle = glGetTextureHandleARB(texture);
    if (handle == 0) {
        std::cerr << "Error! Handle returned null" << std::endl;
        exit(-1);
    }

    textures.push_back(texture);
    textureHandles.push_back(handle);
}
{% endhighlight %}

There needs to be some sort of check to make sure the handle returned actually makes sense. If something bad happened it will return 0.

Next we need to pack all the texture handles into another SSBO.

**Packing Handles Into SSBO**
{% highlight c++ %}
GLuint textureBuffer;
glCreateBuffers(1, &textureBuffer);
glNamedBufferStorage(
    textureBuffer,
    sizeof(GLuint64) * textureHandles.size(),
    (const void *)textureHandles.data(),
    GL_DYNAMIC_STORAGE_BIT
);
{% endhighlight %}

Once we have all this we are ready to enter the rendering loop!

**Making Handles Resident And Drawing**
{% highlight c++ %}
glEnable(GL_DEPTH_TEST);
glClearColor(0.0f, 0.0f, 0.0f, 0.0f);
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

// Mark all as resident
for (GLuint64 handle : textureHandles) {
    glMakeTextureHandleResidentARB(handle);
}

// Bind our shader program
glUseProgram(shader);

// Set up matrices
glUniformMatrix4fv(
    glGetUniformLocation(shader, "projection"), 1, GL_FALSE, projectionMat
);
glUniformMatrix4fv(
    glGetUniformLocation(shader, "view"), 1, GL_FALSE, viewMat
);

// Bind the SSBOs
glBindBufferBase(GL_SHADER_STORAGE_BUFFER, 0, verticesBuffer);
glBindBufferBase(GL_SHADER_STORAGE_BUFFER, 1, modelMatricesBuffer);
glBindBufferBase(GL_SHADER_STORAGE_BUFFER, 2, textureBuffer);

glDrawArraysInstanced(GL_TRIANGLES, 0, numVertices, numInstances);

glUseProgram(0);

// Mark all as non-resident - can be skipped if you know the same textures
// will all be used for the next frame
for (GLuint64 handle : textureHandles) {
    glMakeTextureHandleNonResidentARB(handle);
}
{% endhighlight %}

**A Look At The Shaders**

The vertex shader is almost identical to the one found in the Programmable Vertex Pulling tutorials except that now it writes gl_InstanceID as an output so that the fragment shader can use it. gl_InstanceID in this case will contain values from 0 to 1599 and the fragment shader will use this to determine which texture it should pull data from.

Another really important thing is to remember to use `#extension GL_ARB_bindless_texture : require` since the bindless textures never made it into core OpenGL. We will have to put this in both the vertex and the fragment shader.

*shader.vs*
{% highlight glsl %}
#version 460 core

#extension GL_ARB_bindless_texture : require

// This matches the C++ definition
struct VertexData {
    float position[3];
    float uv[2];
    float normal[3];
};

// readonly SSBO containing the data
layout(binding = 0, std430) readonly buffer ssbo1 {
    VertexData data[];
};

layout(binding = 1, std430) readonly buffer ssbo2 {
    mat4 modelTransforms[];
};

uniform mat4 projection;
uniform mat4 view;

// Helper functions to manually unpack the data into vectors given an index
vec3 getPosition(int index) {
    return vec3(
        data[index].position[0], 
        data[index].position[1], 
        data[index].position[2]
    );
}

vec2 getUV(int index) {
    return vec2(
        data[index].uv[0], 
        data[index].uv[1]
    );
}

vec3 getNormal(int index) {
    return vec3(
        data[index].normal[0], 
        data[index].normal[1], 
        data[index].normal[2]
    );
}

smooth out vec2 fsUv;
flat out vec3 fsNormal;
flat out int fsInstance;

void main()
{
    mat4 vp = projection * view;
    vec4 position = vec4(getPosition(gl_VertexID), 1.0);
    gl_Position = vp * modelTransforms[gl_InstanceID] * position;

    fsUv = getUV(gl_VertexID);
    fsNormal = getNormal(gl_VertexID);
    // Fragment shader needs this to select one of the 1600 available textures
    fsInstance = gl_InstanceID;
}
{% endhighlight %}

In the fragment shader below you'll notice that the readonly SSBO has type "sampler2D" inside of it. Depending on the texture type you are using this should work with sampler2D, sampler3D, samplerCube, etc. All of them only require that you pass in the 64-bit handles as the SSBO data.

*fragment.vs*
{% highlight glsl %}
#version 460 core

#extension GL_ARB_bindless_texture : require

// SSBO containing the textures
layout(binding = 2, std430) readonly buffer ssbo3 {
    sampler2D textures[];
};

smooth in vec2 fsUv;
flat in vec3 fsNormal;
flat in int fsInstance;

out vec4 color;

void main() {
    // Select texture based on instance
    sampler2D tex = textures[fsInstance];
    // Read from the texture with the normal texture() GLSL function
    color = vec4(texture(tex, fsUv).rgb, 1.0);
}
{% endhighlight %}

Running this program produces the following result on a GTX 1060:

![bindless](/assets/bindless.PNG)

## Note About Dynamically Uniform Expressions

For information on what dynamically uniform expressions are, see [https://www.khronos.org/opengl/wiki/Core_Language_(GLSL)#Dynamically_uniform_expression](https://www.khronos.org/opengl/wiki/Core_Language_(GLSL)#Dynamically_uniform_expression).

For a great writeup on this topic, see: [https://gist.github.com/JuanDiegoMontoya/55482fc04d70e83729bb9528ecdc1c61](https://gist.github.com/JuanDiegoMontoya/55482fc04d70e83729bb9528ecdc1c61)

In the above shader we are pulling the texture using the value inside of `flat in int fsInstance;`. Since we are drawing multiple instances with the same draw command, this will **not** be a dynamically uniform expression. Certain hardware supports this, for example the GTX 1060 I used supports it with no other extensions needed except bindless.

But on other hardware this could cause major issues since the code path leading to the texture access needs to be dynamically uniform. There are two main workarounds:

1) Switch to Multi-Draw Indirect (MDI) and make use of `gl_DrawID` which is guaranteed to be dynaically uniform. With this setup instead of having multiple instances of the cube, you would record 1600 draw commands so that `gl_DrawID` is set to a value of [0, 1600) depending on which command is being executed.


2) Add other extensions on top of bindless. For Nvidia (if required) you can add `#extension GL_NV_gpu_shader5 : require`. On AMD (if required) you can add `#extension GL_EXT_nonuniform_qualifier : require`.

(Special thanks to Jake Ryan over at [https://juandiegomontoya.github.io/modern_opengl.html](https://juandiegomontoya.github.io/modern_opengl.html) for pointing this out)

## uvec2 To sampler

For an example of this, see this shader: [https://github.com/JuanDiegoMontoya/GLest-Rendererer/blob/main/glRenderer/Resources/Shaders/gBufferBindless.fs](https://github.com/JuanDiegoMontoya/GLest-Rendererer/blob/main/glRenderer/Resources/Shaders/gBufferBindless.fs)

After making a 64-bit texture handle resident, it is possible to pass in that texture as an array of uvec2 instead of an explicit sampler2D. As a short example modified from the link above:

{% highlight glsl %}
#version 460 core
#extension GL_ARB_bindless_texture : enable

layout (location = 1) uniform uvec2 albedoHandle;

layout (location = 2) in vec2 fsTexCoord;

void main() {
    const bool hasAlbedo = (albedoHandle.x != 0 || albedoHandle.y != 0);
    vec4 color = vec4(0.1, 0.1, 0.1, 1);
    if (hasAlbedo) {
        // Notice the cast to sampler2D from the uvec2 handle
        color = texture(sampler2D(albedoHandle), fsTexCoord).rgba;
    }

    ... rest of shader ...
}
{% endhighlight %}

(Special thanks to Jake Ryan over at [https://juandiegomontoya.github.io/modern_opengl.html](https://juandiegomontoya.github.io/modern_opengl.html) for suggesting this example)

## Conclusion

Bindless textures are a very powerful tool to increase the number of textures that a shader can access. It supplements the old methods of using texture arrays or texture atlases, and for programs that spend large amounts of time binding/unbinding textures between draw calls it can offer a lot of performance improvements.

The main downsides relate to a) not being core in GL 4.6, b) not all graphics debuggers supporting it, and c) outside of Nvidia and AMD which have strong support for bindless, not all vendors (especially mobile) will support them.

## Learn OpenGL Fundamentals
* [https://learnopengl.com](https://learnopengl.com)

## References

* [https://www.khronos.org/opengl/wiki/Bindless_Texture](https://www.khronos.org/opengl/wiki/Bindless_Texture)
* [OpenGL SuperBible](https://www.amazon.com/OpenGL-Superbible-Comprehensive-Tutorial-Reference/dp/0672337479)
* [https://www.khronos.org/opengl/wiki/Core_Language_(GLSL)#Dynamically_uniform_expression](https://www.khronos.org/opengl/wiki/Core_Language_(GLSL)#Dynamically_uniform_expression)
* [https://stackoverflow.com/questions/40875564/opengl-bindless-textures-bind-to-uniform-sampler2d-array](https://stackoverflow.com/questions/40875564/opengl-bindless-textures-bind-to-uniform-sampler2d-array)
* [https://community.khronos.org/t/how-to-implement-bindless-textures-efficiently/76109/2](https://community.khronos.org/t/how-to-implement-bindless-textures-efficiently/76109/2)
* [https://juandiegomontoya.github.io/modern_opengl.html](https://juandiegomontoya.github.io/modern_opengl.html)
* [https://registry.khronos.org/OpenGL/extensions/NV/NV_gpu_shader5.txt](https://registry.khronos.org/OpenGL/extensions/NV/NV_gpu_shader5.txt)
* [https://github.com/KhronosGroup/GLSL/blob/master/extensions/ext/GL_EXT_nonuniform_qualifier.txt](https://github.com/KhronosGroup/GLSL/blob/master/extensions/ext/GL_EXT_nonuniform_qualifier.txt)