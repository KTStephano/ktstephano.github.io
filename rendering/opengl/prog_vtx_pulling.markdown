---
permalink: /rendering/opengl/prog_vtx_pulling
title: Programmable Vertex Pulling
---

![ogl](/assets/opengl.png)

This is part of a tutorial series teaching advanced modern OpenGL. To use all features to their fullest you will need to target OpenGL 4.6. These articles assume you have familiarity with OpenGL.

With the old way of using OpenGL, it required that you send things like vertex, uv, and normal data in a format that the driver could understand and use. This meant using vertex buffer objects (VBOs) and vertex array objects (VAOs). With programmable vertex pulling we will be getting rid of VBOs entirely and using our own data format with manual data unpacking in the shader. This gives us a lot of flexibility with how we structure our data and will be very useful in future articles that will discuss other advanced techniques.

If you haven't already, make sure you check out the [Shader Storage Buffer Objects (SSBOs)](/rendering/opengl/ssbos) tutorial since we will be using those here.

## Old Method

Let's start by looking at an example of how we would pack up the vertices to send to the GPU in a stream with the old way of doing things.

*main.cpp*
{% highlight c++ %}
// Data is laid out as 3 positions, 2 uvs, 3 normals, repeat
std::vector<float> vertices;

... code to fill vertices with vertex data ...

GLuint vbo;
glGenBuffers(1, &vbo);

// Bind and upload data
glBindBuffer(GL_ARRAY_BUFFER, vbo);  
glBufferData(
    GL_ARRAY_BUFFER, 
    vertices.size(), 
    (const void *)vertices.data(), 
    GL_STATIC_DRAW
);

// Tell OpenGL how to interpret the data and where it should be linked to

// Positions
glVertexAttribPointer(
    0, 3, GL_FLOAT, GL_FALSE, 8 * sizeof(float), (void*)0
);
glEnableVertexAttribArray(0);

// UVs
glVertexAttribPointer(
    1, 2, GL_FLOAT, GL_FALSE, 8 * sizeof(float), (void*)(3 * sizeof(float))
);
glEnableVertexAttribArray(1);

// Normals
glVertexAttribPointer(
    2, 2, GL_FLOAT, GL_FALSE, 8 * sizeof(float), (void*)(5 * sizeof(float))
);
glEnableVertexAttribArray(2);

// Bind program and draw
glUseProgram(program);

glDrawArrays(GL_TRIANGLES, 0, vertices.size());

// Unbind
glUseProgram(0);
glBindBuffer(GL_ARRAY_BUFFER, 0);
{% endhighlight %}

Then the shader would look something like this:

*shader.vs*
{% highlight glsl %}
#version 330 core

layout (location = 0) in vec3 position;
layout (location = 1) in vec2 uv;
layout (location = 2) in vec3 normal;

uniform mat4 projection;
uniform mat4 view;

out vec2 fsUv;
out vec3 fsNormal;

void main()
{
    gl_Position = projection * view * vec4(position, 1.0);

    fsUv = uv;
    fsNormal = normal;
}
{% endhighlight %}

## New Method - Programmable Vertex Pulling with SSBOs

This will work for both indexed and non-indexed drawing and can be extended to further support multi-draw indirect commands. In the above example we put all the data into a vector of floats, then sent it to an OpenGL array buffer and told OpenGL how it was supposed to interpret the data and which locations it was supposed to send it to. With programmable vertex pulling we do away with that and instead manually unpack our data in the shader. 

The advantage of this is that we get OpenGL out of the way when it comes to interpreting our data and instead write the code to deal with our data directly a lot like we would do with C++, but now in GLSL. This offers us a lot of flexibility both with vertex data but other with other data that will be discussed in future tutorials.

Based on discussion in the comments, we will be using an empty VAO to avoid issues. More information can be found [here.](https://www.khronos.org/opengl/wiki/Vertex_Rendering/Rendering_Failure)

*main.cpp*
{% highlight c++ %}
// We use arrays of floats since they will be tightly packed with the
// layout std430 qualifier
struct VertexData {
    float position[3];
    float uv[2];
    float normal[3];
};

// (inside either main or a different function)
std::vector<VertexData> vertices;
... populate vertices with either hardcoded data or data you load from a file ...

// Create an empty VAO object to avoid errors
unsigned int emptyVAO;
glGenVertexArrays(1, &emptyVAO);

// Create and fill the mesh data buffer
GLuint vertexDataBuffer;
glCreateBuffers(1, &vertexDataBuffer);

glNamedBufferStorage(vertexDataBuffer, 
                     sizeof(VertexData) * vertices.size(),
                     (const void *)vertices.data(), 
                     GL_DYNAMIC_STORAGE_BIT);

// Bind the empty VAO
// See https://www.khronos.org/opengl/wiki/Vertex_Rendering/Rendering_Failure
glBindVertexArray(emptyVAO);

// Bind the buffer to location 0 - matches (binding = 0) for ssbo1 in the
// vertex shader listed below
glBindBufferBase(GL_SHADER_STORAGE_BUFFER, 0, vertexDataBuffer);

// Bind program and draw
glUseProgram(program);

glDrawArrays(GL_TRIANGLES, 0, vertices.size());

glUseProgram(0);

// Uncomment if you want to explicitly unbind the resouce
//glBindBufferBase(GL_SHADER_STORAGE_BUFFER, 0, 0);
{% endhighlight %}

Inside of the vertex shader we will be making use of a built-in input called `gl_VertexID`. When using non-indexed drawing such as `glDrawArrays`, gl_VertexID will be equal to the current vertex index. When using indexed drawing such as `glDrawElements`, gl_VertexID will be equal to the current index from the element array buffer.

*shader.vs*
{% highlight glsl %}
#version 460 core

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

out vec2 fsUv;
out vec3 fsNormal;

void main()
{
    gl_Position = projection * view * vec4(getPosition(gl_VertexID), 1.0);

    fsUv = getUV(gl_VertexID);
    fsNormal = getNormal(gl_VertexID);
}
{% endhighlight %}

Now we have access to a very convenient vertex processing method. Nice!

## Learn OpenGL Fundamentals
* [https://learnopengl.com](https://learnopengl.com)

## References
* [https://registry.khronos.org/OpenGL-Refpages/gl4/html/gl_VertexID.xhtml](https://registry.khronos.org/OpenGL-Refpages/gl4/html/gl_VertexID.xhtml)
* [3D Graphics Rendering Cookbook](https://www.amazon.com/Graphics-Rendering-Cookbook-comprehensive-algorithms/dp/1838986197)
* [OpenGL SuperBible](https://www.amazon.com/OpenGL-Superbible-Comprehensive-Tutorial-Reference/dp/0672337479)