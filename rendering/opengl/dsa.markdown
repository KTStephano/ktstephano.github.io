---
permalink: /rendering/opengl/dsa
layout: post
---

![ogl](/assets/opengl.png)

This is part of a tutorial series teaching advanced modern OpenGL. To use all features to their fullest you will need to target OpenGL 4.6. These articles assume you have familiarity with OpenGL.

Modern OpenGL is all about improving performance and making the API much easier to use compared to older versions. This article is going to cover the simplest addition to the modern API: Direct State Access (DSA).

## What is Direct State Access (DSA)?

DSA is a replacement of the old [bind resource --> edit resource --> unbind resource] model. It meant that if you wanted to update a texture, frame buffer, or storage object, you first had to bind it to a target, then perform some edit that referenced the target, then unbind. Not only was this less than ideal for ease of use and understanding, it also meant a global GL state change was needed every time a resource needed to be edited. DSA gets rid of that.

## DSA Examples

This section will provide examples that walk through the old way of doing things, then show the new DSA way of doing things.

For example, here is the old way of copying one framebuffer to another:

{% highlight c++ %}
// OLD, no DSA

GLuint fboRead;
GLuint fboWrite;

glGenFramebuffers(1, &fboRead);
glGenFramebuffers(1, &fboWrite);

... code that populates fbo attachments ...

// Bind read+write buffers - global state change
glBindFramebuffer(GL_READ_FRAMEBUFFER, fboRead);
glBindFramebuffer(GL_DRAW_FRAMEBUFFER, fboWrite);

// Copy
glBlitFramebuffer(// read buffer bounds
                  readStartX, readStartY, readEndX, readEndY,
                  // write buffer bounds
                  writeStartX, writeStartY, writeEndX, writeEndY,
                  GL_COLOR_BUFFER_BIT, GL_LINEAR);

// Unbind read+write buffers - state change
glBindFramebuffer(GL_READ_FRAMEBUFFER, 0);
glBindFramebuffer(GL_DRAW_FRAMEBUFFER, 0);

{% endhighlight %}

With DSA, the above code can be simplified to the following:

{% highlight c++ %}
// NEW, DSA

GLuint fboRead;
GLuint fboWrite;

glGenFramebuffers(1, &fboRead);
glGenFramebuffers(1, &fboWrite);

... code that populates fbo attachments ...

// Copy - no state change required
glBlitNamedFramebuffer(fboRead, fboWrite, 
                       // read buffer bounds
                       readStartX, readStartY, readEndX, readEndY,
                       // write buffer bounds
                       writeStartX, writeStartY, writeEndX, writeEndY,
                       GL_COLOR_BUFFER_BIT, GL_LINEAR);

{% endhighlight %}

Notice that glBlitNamedFramebuffer takes handles to the read and write buffers so that we don't need to explicitly bind them to read and write points before performing the copy.

Here is an example of the old vs new method of transferring data to a shader storage buffer object (other buffer types supported as well):

### **Old:**
{% highlight c++ %}
GLuint buffer;
glCreateBuffers(1, &buffer);
// Bind - global state change
glBindBuffer(GL_SHADER_STORAGE_BUFFER, buffer);

// Pass data to buffer
glBufferStorage(GL_SHADER_STORAGE_BUFFER, 
                sizeof(float) * count, (const void *)floatData, 
                GL_DYNAMIC_STORAGE_BIT);

// Unbind - global state change
glBindBuffer(GL_SHADER_STORAGE_BUFFER, 0);
{% endhighlight %}

### **New:**
{% highlight c++ %}
GLuint buffer;
glCreateBuffers(1, &buffer);

// Pass data to buffer - no more bind + unbind!
glNamedBufferStorage(buffer, 
                     sizeof(float) * count, (const void *)floatData, 
                     GL_DYNAMIC_STORAGE_BIT);
{% endhighlight %}

There are also other parts of the API that have received new DSA variants of older functions. Below are a few more examples to show the DSA pattern.

### **Old:**
{% highlight c++ %}
// Global state change
glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_2D, texture);

glTexParameterf(GL_TEXTURE_2D, GL_TEXTURE_MAX_ANISOTROPY, 16.0f);

glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT);

// Global state change
glBindTexture(GL_TEXTURE_2D, 0);
{% endhighlight %}

### **New:**
{% highlight c++ %}
// Instead of taking a bound texture target, 
// now it takes the texture handle directly
glTextureParameterf(texture, GL_TEXTURE_MAX_ANISOTROPY, 16.0f);

glTextureParameteri(texture, GL_TEXTURE_WRAP_S, GL_REPEAT);
glTextureParameteri(texture, GL_TEXTURE_WRAP_T, GL_REPEAT);
{% endhighlight %}

### **Old:**
{% highlight c++ %}
// Global state change
glBindFramebuffer(GL_FRAMEBUFFER, fbo);

// Attach a depth texture
glFramebufferTexture(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, depthTexture, 0);

// Global state change
glBindFramebuffer(GL_FRAMEBUFFER, 0);
{% endhighlight %}

### **New:**
{% highlight c++ %}
// Now takes the framebuffer as a GLuint handle, so no need to bind+unbind
glNamedFramebufferTexture(fbo, GL_DEPTH_ATTACHMENT, depthTexture, 0);
{% endhighlight %}

## Conclusion

The general pattern is that anytime you find yourself using the older method of bind, edit, unbind, consider checking the documentation to see if a DSA version of the function is available so that you can avoid state changes and use the objects more directly instead.

## Learn OpenGL Fundamentals
* [https://learnopengl.com](https://learnopengl.com)

## References
* [https://www.khronos.org/opengl/wiki/Direct_State_Access](https://www.khronos.org/opengl/wiki/Direct_State_Access)
* [OpenGL SuperBible](https://www.amazon.com/OpenGL-Superbible-Comprehensive-Tutorial-Reference/dp/0672337479)