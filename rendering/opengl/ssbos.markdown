---
permalink: /rendering/opengl/ssbos
title: Test
---

![ogl](/assets/opengl.png)

This is part of a tutorial series teaching advanced modern OpenGL. To use all features to their fullest you will need to target OpenGL 4.6. These articles assume you have familiarity with OpenGL.

When writing graphics code that uses OpenGL, eventually we reach a stage where we want a flexible way to read from and write to GPU memory in ways that suit our application best. This is where the Shader Storage Buffer Object (SSBO) comes into play.

## What is the difference between a uniform buffer and a shader storage buffer?
1. A uniform buffer is read only. A shader storage buffer is both read and write. This includes *atomic* read+write operations for data synchronization between GPU cores.
2. Shader storage buffers can be much larger than uniform storage buffers, in practice up to the total size of all available GPU memory can be allocated for use.
3. Shader storage buffers have access to the std430 layout in GLSL which is an improvement over the old std140 layout.
4. Inside of a shader written in GLSL, an SSBO can be specified with variable length instead of having to know ahead of time what the size will be.

## Creating and using a Shader Storage Buffer

For this example, let's say we are performing instanced drawing using `glDrawArraysInstanced` and we want to pass in all of the model matrices for each instance as a shader storage buffer.

Creating the buffer:

{% highlight c++ %}
GLuint modelMatricesBuffer;
glCreateBuffers(1, &modelMatricesBuffer);
{% endhighlight %}

Inserting data into the buffer using Direct State Access (DSA):

{% highlight c++ %}
std::vector<glm::mat4> instancedModelMatrices;
... code to insert a mat4 per instance into the vector ...

glNamedBufferStorage(modelMatricesBuffer, 
                     sizeof(glm::mat4) * instancedModelMatrices.size(), 
                     (const void *)instancedModelMatrices.data(), 
                     GL_DYNAMIC_STORAGE_BIT);
{% endhighlight %}

It is important to keep in mind that once glNamedBufferStorage has been used, the size of the GPU memory region for the buffer is fixed for the rest of `modelMatricesBuffer's` life span. We are allowed to dynamically read and write data into that region, but we can't resize it.

The last parameter to glNamedBufferStorage is the usage flag. The following are the available options:

GL_DYNAMIC_STORAGE_BIT | Buffer contents can be directly updated using glBufferSubData
GL_MAP_READ_BIT        | Buffer memory will be mapped for reading by the CPU
GL_MAP_WRITE_BIT       | Buffer memory will be mapped for writing by the CPU
GL_MAP_PERSISTENT_BIT  | CPU may request that the memory be read from or written to by the GPU while the memory is still mapped. The CPU's pointer to the memory should remain valid even after the GPU writes to it.
GL_MAP_COHERENT_BIT    | When used with GL_MAP_PERSISTENT_BIT, reads and writes from CPU and GPU must be kept coherent
GL_CLIENT_STORAGE_BIT  | Whenever possible the driver should prefer backing the data with system memory rather than GPU memory

For more information, [see this page](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glBufferStorage.xhtml).

In order to use the buffer for instanced rendering in the vertex shader, we will set it up like this:

{% highlight glsl %}
#version 460 core

// Passed in like normal using a vertex array object
layout (location = 0) in vec3 position;
layout (location = 1) in vec2 texCoords;

uniform mat4 projection;
uniform mat4 view;

// SSBO containing the instanced model matrices
layout(binding = 2, std430) readonly buffer ssbo1 {
    mat4 modelMatrices[];
};

smooth out vec2 fsTexCoords;

void main() {
    gl_Position = 
        projection * view * modelMatrices[gl_InstanceID] * vec4(position, 1.0);
    fsTexCoords = texCoords;
}
{% endhighlight %}


## A breakdown of the vertex shader

{% highlight glsl %}
layout(binding = 2, std430) readonly buffer ssbo1 {
    mat4 modelMatrices[];
};
{% endhighlight %}

Here we are specifying the ssbo1 shader storage buffer binding point. We mark it as `readonly` so that the shader is not allowed to write to it. If we were to leave out "readonly" so that it read `layout(binding = 2, std430) buffer ssbo1`, the shader would be able to both read and write to the buffer (see below for more information).

We set the binding point to 2 so that when we get around to binding the buffer in C++ code, we will bind it to location 2.

We set the packing layout to std430 which determines how data is laid out in memory (see below for more information).

Finally, notice that `modelMatrices[]` is specified without an exact size. This is intentional since SSBOs do not require us to tell the shader the size of the array ahead of time. This means that an array of any length can be bound to binding point 2.

{% highlight glsl %}
void main() {
    gl_Position = 
        projection * view * modelMatrices[gl_InstanceID] * vec4(position, 1.0);
    fsTexCoords = texCoords;
}
{% endhighlight %}

The main thing to notice here is that we are indexing the modelMatrices by the gl_InstanceID built-in shader input. gl_InstanceID will always store the current draw instance when using a command like `glDrawArraysInstanced`, or will be 0 if a non-instanced draw command was used.

An SSBO can be indexed by other integer variables including uniform integers that are passed in to the shader.

#### **some of the available memory qualifiers**

buffer | Shader can both read and write to the buffer memory
readonly buffer | Shader can only read from the buffer memory
writeonly buffer | Shader can only write to the buffer memory

For a full list of qualifiers, [see this page](https://www.khronos.org/opengl/wiki/Shader_Storage_Buffer_Object).

#### **std430 packing layout rules**

See [here](https://www.khronos.org/opengl/wiki/Interface_Block_(GLSL)) and [here](https://www.oreilly.com/library/view/opengl-programming-guide/9780132748445/app09lev1sec3.html) for more info.

The std430 packing layout is only available for shader storage buffers. It uses the following rules:

Scalar bool, uint, int, float, double | alignment is equal to standard machine size, i.e. sizeof(GLfloat)
Array of scalars | alignment is equal to length * standard machine size (not rouded up), i.e. length * sizeof(GLfloat)
vec2, ivec2 | alignment is equal to 2N, so 2 * sizeof(GLfloat) or 2 * sizeof(GLint)
vec3, ivec3 | alignment is equal to 4N (rounded up by 1), so 4 * sizeof(GLfloat) or 4 * sizeof(GLint)
vec4, ivec4 | alignment is equal to 4N
mat3 | alignment is equal to 3 vec4s (**not** vec3s), so 3 * 4 * N
mat4 | alignment is equal to 4 vec4s, so 4 * 4 * N

From this table we can see that scalars and arrays of scalars will be tightly packed, so `float array[3]` on the CPU side will equal `float array[3]` array on the GPU side for example.

A vec3 needs to be aligned to a 16 byte boundary. A vec4 is also a 16 byte boundary. A mat3 needs to be aligned to a 48 byte boundary. A mat4 needs to be aligned to a 64 byte boundary.

To make it easier for yourself, avoid using vec3 and mat3 in shader storage buffers.

## Binding and drawing

Now we are at the drawing stage which will look something like this:

{% highlight c++ %}
// Bind the storage buffer
// Note the 2 matches our binding = 2 in the vertex shader
glBindBufferBase(GL_SHADER_STORAGE_BUFFER, 2, modelMatricesBuffer);

// Bind our shader program
glUseProgram(shader);

// Set up matrices
glUniformMatrix4fv(
    glGetUniformLocation(shader, "projection"), 1, GL_FALSE, projectionMat
);
glUniformMatrix4fv(
    glGetUniformLocation(shader, "view"), 1, GL_FALSE, viewMat
);

// Bind VAO for positions and uv coordinates
glBindVertexArray(vao);

// Perform instanced drawing
glDrawArraysInstanced(
    GL_TRIANGLES, 0, numVertices, instancedModelMatrices.size()
);

glBindVertexArray(0);
glUseProgram(0);
{% endhighlight %}

## Unbinding? Using with multiple shaders?

There is no need to unbind a shader storage buffer after calling bind buffer base. You can overwrite the existing binding at any point by calling the function again with the same binding index but with a different shader storage buffer object.

If you want the same shader storage buffer to be used by multiple shaders, simply specify the exact same shader storage binding in each of those shaders. Then you can call glBindBufferBase once with the index and proceed to use it with multiple shader programs in a row.

## Can SSBOs be used to store vertex, uv, and normal data?

Yes! See the [Programmable Vertex Pulling](/rendering/opengl/prog_vtx_pulling) tutorial showing how to do this.

## Future articles

Aside from the upcoming programmable vertex pulling article, there will also be an article dedicated to compute shaders and showing how to use SSBOs to allow the GPU to perform highly parallel work and write the results to memory that the CPU will later be able to access.

## Learn OpenGL Fundamentals
* [https://learnopengl.com](https://learnopengl.com)

## References
* [https://registry.khronos.org/OpenGL-Refpages/gl4/html/glBufferStorage.xhtml](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glBufferStorage.xhtml)
* [https://www.khronos.org/opengl/wiki/Shader_Storage_Buffer_Object](https://www.khronos.org/opengl/wiki/Shader_Storage_Buffer_Object)
* [https://www.oreilly.com/library/view/opengl-programming-guide/9780132748445/app09lev1sec3.html](https://www.oreilly.com/library/view/opengl-programming-guide/9780132748445/app09lev1sec3.html)
* [https://www.khronos.org/opengl/wiki/Direct_State_Access](https://www.khronos.org/opengl/wiki/Direct_State_Access)
* [OpenGL SuperBible](https://www.amazon.com/OpenGL-Superbible-Comprehensive-Tutorial-Reference/dp/0672337479)