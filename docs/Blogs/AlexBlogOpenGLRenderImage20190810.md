
/*
    OpenGL 渲染图片
    着色器
    纹理渲染
*/


### 一、OpenGL渲染之着色器代码

1.1、片段着色器代码 


```
precision highp float;

uniform sampler2D Texture;
varying vec2 TextureCoordsVarying;

void main (void) {
    vec4 mask = texture2D(Texture, TextureCoordsVarying);
    gl_FragColor = vec4(mask.rgb, 1.0);
}
```

1.2、顶点着色器代码

```
attribute vec4 Position;
attribute vec2 TextureCoords;
varying vec2 TextureCoordsVarying;

void main (void) {
    gl_Position = Position;
    TextureCoordsVarying = TextureCoords;
}
```

1.3、字符串编译着色器的宏实现


```
#define STRINGIZE(x) #x
#define STRINGIZE2(x) STRINGIZE(x)
#define SHADER_STRING(text) @ STRINGIZE2(text)

NSString *const kGPUImageVertexShaderString = SHADER_STRING
(
	//write shader code here....
 );
```

### 二、OpenGL纹理渲染

2.1、初始化顶点信息

```
typedef struct {
    GLKVector3 positionCoord; // (X, Y, Z)
    GLKVector2 textureCoord; // (X, Y)
} VertexVector;

self.vertices = malloc(sizeof(VertexVector) * 4);
//sizeof(VertexVector) = max(sizeof(GLKVector3), sizeof(GLKVector2)) * 2
self.vertices[0] = (VertexVector){ {-1, 1, 0}, {0, 1} };
self.vertices[1] = (VertexVector){ {-1, -1, 0}, {0, 0} };
self.vertices[2] = (VertexVector){ {1, 1, 0}, {1, 1} };
self.vertices[3] = (VertexVector){ {1, -1, 0}, {1, 0} };
```

2.2、解压图片、获取图片的像素点数据。   

    bitsPerComponent : 颜色分量的字节位数
    bytesPerRow : 每行像素的字节数, (bytesPerRow = 像素点的字节数 * 像素宽度) RGBA的像素点字节数为4每位一个字节
    imageData : 解压后的像素数据BUFFER


```
CGImageRef cgImageRef = [image CGImage];
GLuint width = (GLuint)CGImageGetWidth(cgImageRef);
GLuint height = (GLuint)CGImageGetHeight(cgImageRef);
size_t bitsPerComponent = CGImageGetBitsPerComponent(cgImageRef);
size_t bytesPerRow = CGImageGetBytesPerRow(cgImageRef);
CGRect rect = CGRectMake(0, 0, width, height);

CGBitmapInfo bitmapInfo = CGImageGetBitmapInfo(cgImageRef);
CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
void *imageData = malloc(height * bytesPerRow);
CGContextRef context = CGBitmapContextCreate(imageData, width, height, bitsPerComponent, bytesPerRow, colorSpace, bitmapInfo);
CGContextTranslateCTM(context, 0, height);
CGContextScaleCTM(context, 1.0f, -1.0f);
CGColorSpaceRelease(colorSpace);
CGContextClearRect(context, rect);
CGContextDrawImage(context, rect, cgImageRef);

CGContextRelease(context);
free(imageData);
```

2.3、初始化TextureID(纹理对象),绑定图片数据到TextureID；textureID可重复使用。

    glGenTextures : 参数1 : 表示创建的textureID个数 参数2 : 创建的 textureID
    width : 像素宽度
    height : 像素高度


```
GLuint textureID;
glGenTextures(1, &textureID);//第一参数代表生成的textureID数量
glBindTexture(GL_TEXTURE_2D, textureID);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, width, height, 0, GL_RGBA, GL_UNSIGNED_BYTE, imageData);

glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);

glBindTexture(GL_TEXTURE_2D, 0);
```

2.4、初始化顶点缓存对象（VBO）

```
GLuint vertexBuffer;
glGenBuffers(1, &vertexBuffer);
glBindBuffer(GL_ARRAY_BUFFER, vertexBuffer);
GLsizeiptr bufferSizeBytes = sizeof(VertexVector) * 4;
glBufferData(GL_ARRAY_BUFFER, bufferSizeBytes, self.vertices, GL_STATIC_DRAW);
self.vertexBuffer = vertexBuffer;
```

2.5、通过OpenGL API获取对应的Context并设置为CunrrentContext

```
self.context = [[EAGLContext alloc] initWithAPI:kEAGLRenderingAPIOpenGLES2];
[EAGLContext setCurrentContext:self.context];
```
 
2.6、初始化EAGLLayer渲染layer对象

```
CAEAGLLayer *layer = [[CAEAGLLayer alloc] init];
layer.frame = CGRectMake(0, 100, self.view.frame.size.width, self.view.frame.size.width);
layer.contentsScale = [[UIScreen mainScreen] scale];
```

2.7、初始化RenderBuffer、FrameBuffer,并把RenderBuffer附加到FrameBuffer上;之后将EAGLContext、RenderBuffer、EAGLLayer三者进行关联绑定，这样就可以通过EAGLContext上下文将RenderBuffer渲染到EAGLLayer上。

```
GLuint renderBuffer;
GLuint frameBuffer;

glGenRenderbuffers(1, &renderBuffer);
glBindRenderbuffer(GL_RENDERBUFFER, renderBuffer);
[self.context renderbufferStorage:GL_RENDERBUFFER fromDrawable:layer];

glGenFramebuffers(1, &frameBuffer);
glBindFramebuffer(GL_FRAMEBUFFER, frameBuffer);
glFramebufferRenderbuffer(GL_FRAMEBUFFER,
                          GL_COLOR_ATTACHMENT0,
                          GL_RENDERBUFFER,
                          renderBuffer);
```

2.8、设置渲染窗口的大小

```
glViewport(0, 0, self.drawableWidth, self.drawableHeight);
```

2.9、创建着色器、指定着色器代码源、编译着色器代码 
    
    vsh : 对应开始的顶点着色器代码
    fsh : 对应开始的片段着色器代码


```
NSString *shaderPath = [[NSBundle mainBundle] pathForResource:name ofType:shaderType == GL_VERTEX_SHADER ? @"vsh" : @"fsh"];
NSError *error;
NSString *shaderString = [NSString stringWithContentsOfFile:shaderPath encoding:NSUTF8StringEncoding error:&error];
if (!shaderString) {
    NSAssert(NO, @"shader不存在");
    exit(1);
}

GLuint shader = glCreateShader(shaderType);

const char *shaderStringUTF8 = [shaderString UTF8String];
int shaderStringLength = (int)[shaderString length];
glShaderSource(shader, 1, &shaderStringUTF8, &shaderStringLength);

glCompileShader(shader);

GLint compileSuccess;
glGetShaderiv(shader, GL_COMPILE_STATUS, &compileSuccess);
if (compileSuccess == GL_FALSE) {
    GLchar messages[256];
    glGetShaderInfoLog(shader, sizeof(messages), 0, &messages[0]);
    NSString *messageString = [NSString stringWithUTF8String:messages];
    NSAssert(NO, @"shader编译失败：%@", messageString);
    exit(1);
}
```

2.10、创建着色器程序、绑定编译好的顶点着色器和片段着色器、链接着色器程序、左后将此着色器指定为当前使用的着色器。
    
    vertexShader : 编译后的顶点着色器
    fragmentShader : 编译后的片段着色器
    program : 着色器程序BUFFER

```
GLuint program = glCreateProgram();
glAttachShader(program, vertexShader);
glAttachShader(program, fragmentShader);

glLinkProgram(program);

GLint linkSuccess;
glGetProgramiv(program, GL_LINK_STATUS, &linkSuccess);
if (linkSuccess == GL_FALSE) {
    GLchar messages[256];
    glGetProgramInfoLog(program, sizeof(messages), 0, &messages[0]);
    NSString *messageString = [NSString stringWithUTF8String:messages];
    NSAssert(NO, @"program链接失败：%@", messageString);
    exit(1);
}
glUseProgram(program);
```

2.11、设置着色器的参数解析方式

```
GLuint positionSlot = glGetAttribLocation(program, "Position");
GLuint textureSlot = glGetUniformLocation(program, "Texture");
GLuint textureCoordsSlot = glGetAttribLocation(program, "TextureCoords");

glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_2D, self.textureID);
glUniform1i(textureSlot, 0);

glEnableVertexAttribArray(positionSlot);
glVertexAttribPointer(positionSlot, 3, GL_FLOAT, GL_FALSE, sizeof(VertexVector), NULL + offsetof(VertexVector, positionCoord));

glEnableVertexAttribArray(textureCoordsSlot);
glVertexAttribPointer(textureCoordsSlot, 2, GL_FLOAT, GL_FALSE, sizeof(VertexVector), NULL + offsetof(VertexVector, textureCoord));
self.program = program;
```


### 三、总结

通过这次OpenGL图片渲染练习，对如何使用OpenGL绘制图片有了一定认知；也明白了渲染上下文、纹理对象、以及可视Layer之间的联系；当然也认识了着色器程序，学习了简单的着色器语言以及如何创建、编译、链接着色器程序，如何与着色器进行数据交互。

注：本文实现参考自GPUImage源码库,在此表示感谢！







