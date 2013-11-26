﻿欢迎来到第十三课！今天讲法线贴图（normal mapping）。Welcome for our 13th tutorial ! Today we will talk about normal mapping.

第八课：基本光照模型之后，我们知道了如何用三角形法线得到不错的光照效果。需要注意的是，截至目前，每个顶点仅有一个法线：在三角形三个顶点间，法线是平滑过渡的；而颜色（纹理的采样）恰与此相反。Since <a title="Tutorial 8 : Basic shading" href="http://www.opengl-tutorial.org/beginners-tutorials/tutorial-8-basic-shading/">Tutorial 8 : Basic shading</a> , you know how to get decent shading using triangle normals. One caveat is that until now, we only had one normal per vertex : inside each triangle, they vary smoothly, on the opposite to the colour, which samples a texture. The basic idea of normal mapping is to give normals similar variations.
法线纹理Normal textures
法线纹理看起来像这样：A "normal texture" looks like this :

<a href="http://www.opengl-tutorial.org/wp-content/uploads/2011/05/normal.jpg"><img class="alignnone size-full wp-image-307" title="normal" src="http://www.opengl-tutorial.org/wp-content/uploads/2011/05/normal.jpg" alt="" width="341" height="336" /></a>

每个纹素的RGB值实际上表示的是XYZ向量：颜色的分量取值范围为0到1，而向量的分量取值范围是-1到1；可以建立从纹素到法线的简单映射：In each RGB texel is encoded a XYZ vector : each colour component is between 0 and 1, and each vector component is between -1 and 1, so this simple mapping goes from the texel to the normal :
normal = (2*color)-1 // on each component
法线纹理整体呈蓝色，因为法线基本是朝上的（上方即Z轴正向。OpenGL中Y轴=上，有所不同。这种不兼容很蠢，但没人想为此重写现有的工具，我们将就用吧。后面介绍详情。）The texture has a general blue tone because overall, the normal is up (so yeah, up is Z, which means that it's not our standard "Y=up" space. This is stupid but since nobody wants to recode all the tools, let's deal with it. More on this later.)

法线纹理的映射方式和颜色纹理相似。麻烦的是如何将法线从各三角形局部坐标系（切线坐标系tangent space，亦称图像坐标系image space）变换到模型坐标系（计算光照采用的坐标系）。This texture is mapped just like the diffuse one; The big problem is how to convert our normal, which is expressed in the space each individual triangle ( tangent space, also called image space), in model space (since this is what is used in our shading equation).
切线和双切线（Tangent and Bitangent）
想必大家对矩阵十分熟悉了；大家知道，定义一个坐标系（本例是切线坐标系）需要三个向量。现在朝上的向量已经有了，即法线：可用Blender计算，或做一个简单的叉乘。还差三角形所在平面上的两个向量，即切线和双切线向量。“切线”系数学术语，指曲线的方向；而"双切线"则是因为这里我们面对的不是曲线，而是曲面，所以需要两个切线，故称“双切线”。You are now so familiar with matrices that you know that in order to define a space (in our case, the tangent space), we need 3 vectors. We already have our UP vector : it's the normal, given by Blender or computed from the triangle by a simple cross product. We need two more, in the plane of the triangle. They are called the Tangent and Bitangent vectors. "Tangent" because it is the standard mathematical term for the vector that is in the direction of a curve, and "Bitangent" because here we don't have a curve, we have a surface, so we need two tangents, hence "bitangent".

我们想把切线的方向转换到纹理坐标系中。We want to orient our tangents in the same direction than our texture coordinates.

若把三角形的两条边记为deltaPos1和deltaPos2，deltaUV1和deltaUV2是对应的UV坐标下的差值；此问题可用以下方程表示：If we note deltaPos1 and deltaPos2 two edges of our triangle, and deltaUV1 and deltaUV2 the corresponding differences in UVs, we can express our problem with the following equation :
deltaPos1 = deltaUV1.x * T + deltaUV1.y * B
deltaPos2 = deltaUV2.x * T + deltaUV2.y * B
因此，已知T、B、N向量，可得下面这个漂亮的矩阵，完成从模型坐标系到切线坐标系的变换：So given our T, B, N vectors, we've got this nice matrix which enables us to go from Model Space to Tangent Space :

<a href="http://www.opengl-tutorial.org/wp-content/uploads/2011/05/TBN.png"><img class="alignnone size-full wp-image-308 whiteborder" title="TBN" src="http://www.opengl-tutorial.org/wp-content/uploads/2011/05/TBN.png" alt="" width="107" height="66" /></a>

但我们需要的是却是从切线坐标系（纹理定义于其中）到模型坐标系（在这里计算光照）的变换——这只需对以上矩阵求逆即可。对于这个矩阵（正交阵，即各向量相互正交，请看后面“延伸阅读”小节）来说，其逆矩阵就是转置矩阵，计算十分简单：And since we wanted it exactly the other way around ( from Tangent Space, in which our texture is expressed, to Model Space, in which we can do lighting ), we simply have to take its inverse, which in this case (an orthogonal matrix, i.e each vector is perpendicular to the others. See "going further" below) is also its transpose, much cheaper to compute :
invTBN = transpose(TBN)
亦即：, i.e. :
<a href="http://www.opengl-tutorial.org/wp-content/uploads/2011/05/transposeTBN.png"><img class="alignnone size-full wp-image-309 whiteborder" title="transposeTBN" src="http://www.opengl-tutorial.org/wp-content/uploads/2011/05/transposeTBN.png" alt="" width="262" height="70" /></a>

小提示：顺序是T、B、N，而非T、N、B。前者更符合人们的预期。因为一般认为Z轴=上（前面的讲纹理偏蓝的原因，还记得吗？）。Quick note : It's T,B,N instead of T,N,B as one would expect because normal textures assume that Z=up (hence the blue color, remember ?).
准备VBO（Vertex Buffer Object 顶点缓冲区对象）Preparing our VBO
计算切线和双切线Computing the tangents and bitangents
我们需要为整个模型计算切线、双切线和法线。用一个单独的函数完成这项工作：Since we need our tangents and bitangents on top of our normals, we have to compute them for the whole mesh. We'll do this in a separate function :
void computeTangentBasis(
    // inputs
    std::vector<glm::vec3> & vertices,
    std::vector<glm::vec2> & uvs,
    std::vector<glm::vec3> & normals,
    // outputs
    std::vector<glm::vec3> & tangents,
    std::vector<glm::vec3> & bitangents
){
为每个三角形计算边（deltaPos）和deltaUV For each triangle, we compute the edge (deltaPos) and the deltaUV
    for ( int i=0; i<vertices.size(); i+=3){

        // Shortcuts for vertices
        glm::vec3 & v0 = vertices[i+0];
        glm::vec3 & v1 = vertices[i+1];
        glm::vec3 & v2 = vertices[i+2];

        // Shortcuts for UVs
        glm::vec2 & uv0 = uvs[i+0];
        glm::vec2 & uv1 = uvs[i+1];
        glm::vec2 & uv2 = uvs[i+2];

        // Edges of the triangle : postion delta
        glm::vec3 deltaPos1 = v1-v0;
        glm::vec3 deltaPos2 = v2-v0;

        // UV delta
        glm::vec2 deltaUV1 = uv1-uv0;
        glm::vec2 deltaUV2 = uv2-uv0;
现在用公式来算切线和双切线：We can now use our formula to compute the tangent and the bitangent :
        float r = 1.0f / (deltaUV1.x * deltaUV2.y - deltaUV1.y * deltaUV2.x);
        glm::vec3 tangent = (deltaPos1 * deltaUV2.y   - deltaPos2 * deltaUV1.y)*r;
        glm::vec3 bitangent = (deltaPos2 * deltaUV1.x   - deltaPos1 * deltaUV2.x)*r;
最后，把这些切线和双切线缓存到数组。记住，还没为这些缓存的数据生成索引，因此每个顶点都有一份拷贝。Finally, we fill the <em>tangents </em>and <em>bitangents </em>buffers. Remember, these buffers are not indexed yet, so each vertex has its own copy.
        // Set the same tangent for all three vertices of the triangle.
        // They will be merged later, in vboindexer.cpp
        tangents.push_back(tangent);
        tangents.push_back(tangent);
        tangents.push_back(tangent);

        // Same thing for binormals
        bitangents.push_back(bitangent);
        bitangents.push_back(bitangent);
        bitangents.push_back(bitangent);

    }
生成索引Indexing
索引VBO的做法和之前类似，但有些许不同。Indexing our VBO is very similar to what we used to do, but there is a subtle difference.

若找到一个相似顶点（相同的坐标、法线、纹理坐标），我们不保存和使用它的切线、双切线；这两项我们取所有相似顶点的均值。因此，只需把老代码改动一下：If we find a similar vertex (same position, same normal, same texture coordinates), we don't want to use its tangent and binormal too ; on the contrary, we want to average them. So let's modify our old code a bit :
        // Try to find a similar vertex in out_XXXX
        unsigned int index;
        bool found = getSimilarVertexIndex(in_vertices[i], in_uvs[i], in_normals[i],     out_vertices, out_uvs, out_normals, index);

        if ( found ){ // A similar vertex is already in the VBO, use it instead !
            out_indices.push_back( index );

            // Average the tangents and the bitangents
            out_tangents[index] += in_tangents[i];
            out_bitangents[index] += in_bitangents[i];
        }else{ // If not, it needs to be added in the output data.
            // Do as usual
            [...]
        }
注意，这里没做归一化。这样做很妙，因为小三角形的切线、双切线向量也小；相对于大三角形（对最终形状影响较大），对最终结果的影响力也就小。Note that we don't normalize anything here. This is actually handy, because this way, small triangles, which have smaller tangent and bitangent vectors, will have a weaker effect on the final vectors than big triangles (which contribute more to the final shape).
着色器The shader
额外的缓冲区和uniform变量Additional buffers & uniforms
新加上两个缓冲区：分别存放切线和双切线：We need two new buffers : one for the tangents, and one for the bitangents :
    GLuint tangentbuffer;
    glGenBuffers(1, &tangentbuffer);
    glBindBuffer(GL_ARRAY_BUFFER, tangentbuffer);
    glBufferData(GL_ARRAY_BUFFER, indexed_tangents.size() * sizeof(glm::vec3), &indexed_tangents[0], GL_STATIC_DRAW);

    GLuint bitangentbuffer;
    glGenBuffers(1, &bitangentbuffer);
    glBindBuffer(GL_ARRAY_BUFFER, bitangentbuffer);
    glBufferData(GL_ARRAY_BUFFER, indexed_bitangents.size() * sizeof(glm::vec3), &indexed_bitangents[0], GL_STATIC_DRAW);
还需要一个uniform变量存储新的法线纹理：We also need a new uniform for our new normal texture :
    [...]
    GLuint NormalTexture = loadTGA_glfw("normal.tga");
    [...]
    GLuint NormalTextureID  = glGetUniformLocation(programID, "NormalTextureSampler");
另外一个uniform变量存储3x3的模型视图矩阵。严格地讲，这个矩阵不必要，但有它更方便；详见后文。由于仅仅计算旋转，不需要位移，因此只需矩阵左上角3x3的部分。And one for the 3x3 ModelView matrix. This is strictly speaking not necessary, but it's easier ; more about this later. We just need the 3x3 upper-left part because we will multiply directions, so we can drop the translation part.
    GLuint ModelView3x3MatrixID = glGetUniformLocation(programID, "MV3x3");
完整的绘制代码如下：So the full drawing code becomes :
        // Clear the screen
        glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

        // Use our shader
        glUseProgram(programID);

        // Compute the MVP matrix from keyboard and mouse input
        computeMatricesFromInputs();
        glm::mat4 ProjectionMatrix = getProjectionMatrix();
        glm::mat4 ViewMatrix = getViewMatrix();
        glm::mat4 ModelMatrix = glm::mat4(1.0);
        glm::mat4 ModelViewMatrix = ViewMatrix * ModelMatrix;
        glm::mat3 ModelView3x3Matrix = glm::mat3(ModelViewMatrix); // Take the upper-left part of ModelViewMatrix
        glm::mat4 MVP = ProjectionMatrix * ViewMatrix * ModelMatrix;

        // Send our transformation to the currently bound shader,
        // in the "MVP" uniform
        glUniformMatrix4fv(MatrixID, 1, GL_FALSE, &MVP[0][0]);
        glUniformMatrix4fv(ModelMatrixID, 1, GL_FALSE, &ModelMatrix[0][0]);
        glUniformMatrix4fv(ViewMatrixID, 1, GL_FALSE, &ViewMatrix[0][0]);
        glUniformMatrix4fv(ViewMatrixID, 1, GL_FALSE, &ViewMatrix[0][0]);
        glUniformMatrix3fv(ModelView3x3MatrixID, 1, GL_FALSE, &ModelView3x3Matrix[0][0]);

        glm::vec3 lightPos = glm::vec3(0,0,4);
        glUniform3f(LightID, lightPos.x, lightPos.y, lightPos.z);

        // Bind our diffuse texture in Texture Unit 0
        glActiveTexture(GL_TEXTURE0);
        glBindTexture(GL_TEXTURE_2D, DiffuseTexture);
        // Set our "DiffuseTextureSampler" sampler to user Texture Unit 0
        glUniform1i(DiffuseTextureID, 0);

        // Bind our normal texture in Texture Unit 1
        glActiveTexture(GL_TEXTURE1);
        glBindTexture(GL_TEXTURE_2D, NormalTexture);
        // Set our "Normal    TextureSampler" sampler to user Texture Unit 0
        glUniform1i(NormalTextureID, 1);

        // 1rst attribute buffer : vertices
        glEnableVertexAttribArray(0);
        glBindBuffer(GL_ARRAY_BUFFER, vertexbuffer);
        glVertexAttribPointer(
            0,                  // attribute
            3,                  // size
            GL_FLOAT,           // type
            GL_FALSE,           // normalized?
            0,                  // stride
            (void*)0            // array buffer offset
        );

        // 2nd attribute buffer : UVs
        glEnableVertexAttribArray(1);
        glBindBuffer(GL_ARRAY_BUFFER, uvbuffer);
        glVertexAttribPointer(
            1,                                // attribute
            2,                                // size
            GL_FLOAT,                         // type
            GL_FALSE,                         // normalized?
            0,                                // stride
            (void*)0                          // array buffer offset
        );

        // 3rd attribute buffer : normals
        glEnableVertexAttribArray(2);
        glBindBuffer(GL_ARRAY_BUFFER, normalbuffer);
        glVertexAttribPointer(
            2,                                // attribute
            3,                                // size
            GL_FLOAT,                         // type
            GL_FALSE,                         // normalized?
            0,                                // stride
            (void*)0                          // array buffer offset
        );

        // 4th attribute buffer : tangents
        glEnableVertexAttribArray(3);
        glBindBuffer(GL_ARRAY_BUFFER, tangentbuffer);
        glVertexAttribPointer(
            3,                                // attribute
            3,                                // size
            GL_FLOAT,                         // type
            GL_FALSE,                         // normalized?
            0,                                // stride
            (void*)0                          // array buffer offset
        );

        // 5th attribute buffer : bitangents
        glEnableVertexAttribArray(4);
        glBindBuffer(GL_ARRAY_BUFFER, bitangentbuffer);
        glVertexAttribPointer(
            4,                                // attribute
            3,                                // size
            GL_FLOAT,                         // type
            GL_FALSE,                         // normalized?
            0,                                // stride
            (void*)0                          // array buffer offset
        );

        // Index buffer
        glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, elementbuffer);

        // Draw the triangles !
        glDrawElements(
            GL_TRIANGLES,      // mode
            indices.size(),    // count
            GL_UNSIGNED_INT,   // type
            (void*)0           // element array buffer offset
        );

        glDisableVertexAttribArray(0);
        glDisableVertexAttribArray(1);
        glDisableVertexAttribArray(2);
        glDisableVertexAttribArray(3);
        glDisableVertexAttribArray(4);

        // Swap buffers
        glfwSwapBuffers();
顶点着色器Vertex shader
和前面讲的一样，所有计算都在观察坐标系中做，因为在这获取片断坐标更容易。这就是为什么要用模型视图矩阵乘T、B、N向量。As said before, we'll do everything in camera space, because it's simpler to get the fragment's position in this space. This is why we multiply our T,B,N vectors with the ModelView matrix.
    vertexNormal_cameraspace = MV3x3 * normalize(vertexNormal_modelspace);
    vertexTangent_cameraspace = MV3x3 * normalize(vertexTangent_modelspace);
    vertexBitangent_cameraspace = MV3x3 * normalize(vertexBitangent_modelspace);
这三个向量确定了TBN矩阵，其创建方式如下：These three vector define a the TBN matrix, which is constructed this way :
    mat3 TBN = transpose(mat3(
        vertexTangent_cameraspace,
        vertexBitangent_cameraspace,
        vertexNormal_cameraspace
    )); // You can use dot products instead of building this matrix and transposing it. See References for details.
此矩阵是从观察坐标系到切线坐标系的变换（若有一矩阵名为XXX_modelspace，则它执行的是从模型坐标系到切线坐标系的变换）。可以利用它计算切线坐标系中的光线方向和视线方向。This matrix goes from camera space to tangent space (The same matrix, but with XXX_modelspace instead, would go from model space to tangent space). We can use it to compute the light direction and the eye direction, in tangent space :
    LightDirection_tangentspace = TBN * LightDirection_cameraspace;
    EyeDirection_tangentspace =  TBN * EyeDirection_cameraspace;
片断着色器Fragment shader
切线坐标系中的法线很容易获取：就在纹理中：Our normal, in tangent space, is really straightforward to get : it's our texture :
    // Local normal, in tangent space
    vec3 TextureNormal_tangentspace = normalize(texture2D( NormalTextureSampler, UV ).rgb*2.0 - 1.0);
?

一切准备就绪。漫反射光的值由切线坐标系中的n和l计算得来（在哪个坐标系中计算并不重要，重要的是n和l必须位于同一坐标系中），再用<em>clamp( dot( n,l ), 0,1 )</em>截断。镜面光用<em>clamp( dot( E,R ), 0,1 )</em>截断，E和R也必须位于同一坐标系中。搞定！So we've got everything we need now. Diffuse lighting uses <em>clamp( dot( n,l ), 0,1 )</em>, with n and l expressed in tangent space (it doesn't matter in which space you make your dot and cross products; the important thing is that n and l are both expressed in the same space). Specular lighting uses <em>clamp( dot( E,R ), 0,1 )</em>, again with E and R expressed in tangent space. Yay !
结果Results
这是目前得到的结果，可以看到：Here is our result so far. You can notice that :

	砖块看上去凹凸不平，这是因为法线变化丰富The bricks look bumpy because we have lots of variations in the normals
	水泥部分看上去很平整，这是因为这部分的法线纹理都是整齐的蓝色Cement looks flat because the? normal texture is uniformly blue

<a href="http://www.opengl-tutorial.org/wp-content/uploads/2011/05/normalmapping.png"><img class="alignnone size-large wp-image-315" title="normalmapping" src="http://www.opengl-tutorial.org/wp-content/uploads/2011/05/normalmapping-1024x793.png" alt="" width="640" height="495" /></a>
延伸阅读Going further
正交化Orthogonalization
顶点着色器中，为了计算得更快，我们没有用矩阵求逆，而是进行了转置。这只有当矩阵表示的坐标系是正交的时候才成立，而眼前这个矩阵还不是正交的。幸运的是这个问题很容易解决：只需在computeTangentBasis()末尾让切线与法线垂直。In our vertex shader we took the transpose instead of the inverse because it's faster. But it only works if the space that the matrix represents is orthogonal, which is not yet the case. Luckily, this is very easy to fix : we just have to make the tangent perpendicular to the normal at he end of computeTangentBasis() :
t = glm::normalize(t - n * glm::dot(n, t));
这个公式有点难理解，来看看图：This formula may be hard to grasp, so a little schema might help :

<a href="http://www.opengl-tutorial.org/wp-content/uploads/2011/05/gramshmidt.png"><img class="alignnone size-full wp-image-319 whiteborder" title="gramshmidt" src="http://www.opengl-tutorial.org/wp-content/uploads/2011/05/gramshmidt.png" alt="" width="300" height="157" /></a>

n和t差不多是相互垂直的，只要把t沿-n方向稍微“压”一下，这个幅度是dot(n,t)。n and t are almost perpendicular, so we "push" t in the direction of -n by a factor of dot(n,t)

<a href="http://www.cse.illinois.edu/iem/least_squares/gram_schmidt/">这里Here</a>有一个applet也讲得很清楚（仅含两个向量）'s a little applet that explains it too (Use only 2 vectors).
左手还是右手坐标系？Handedness
一般不必担心这个问题。但在某些情况下，比如使用对称模型时，UV坐标方向是错的，导致切线T方向错误。You usually don't have to worry about that, but in some cases, when you use symmetric models, UVs are oriented in the wrong way, and your T has the wrong orientation.

检查是否需要翻转这些方向很容易：TBN必须形成一个右手坐标系，即，向量cross(n,t)应该和b同向。。To check whether it must be inverted or not, the check is simple : TBN must form a right-handed coordinate system, i.e. cross(n,t) must have the same orientation than b.

在数学上，“向量A和向量B同向”就是“dot(A,B)>0”；故只需检查dot( cross(n,t) , b )是否大于0。In mathematics, "Vector A has the same orientation as Vector B" translates as dot(A,B)>0, so we need to check if dot( cross(n,t) , b ) > 0.

若dot( cross(n,t) , b ) < 0，就要翻转t：If it's false, just invert t :
if (glm::dot(glm::cross(n, t), b) < 0.0f){
???? t = t * -1.0f;
 }
在computeTangentBasis()末对每个顶点都做这个操作。This is also done for each vertex at the end of computeTangentBasis().
高光纹理Specular texture
出于乐趣，我在代码里加上了高光纹理；取代了原先简单的，作为高光颜色的“vec3(0.3,0.3,0.3)”灰，现在看起来像这样：Just for fun, I added a specular texture to the code. It looks like this :

<a href="http://www.opengl-tutorial.org/wp-content/uploads/2011/05/specular.jpg"><img class="alignnone size-full wp-image-317" title="specular" src="http://www.opengl-tutorial.org/wp-content/uploads/2011/05/specular.jpg" alt="" width="351" height="335" /></a>

and is used instead of the simple "vec3(0.3,0.3,0.3)" grey that we used as specular color.

<a href="http://www.opengl-tutorial.org/wp-content/uploads/2011/05/normalmappingwithspeculartexture.png"><img class="alignnone size-large wp-image-316" title="normalmappingwithspeculartexture" src="http://www.opengl-tutorial.org/wp-content/uploads/2011/05/normalmappingwithspeculartexture-1024x793.png" alt="" width="640" height="495" /></a>

注意，现在水泥部分始终是黑色的：因为高光纹理中，其高光分量为0。Notice that now, cement is always black : the texture says that it has no specular component.
用立即模式进行调试Debugging with the immediate mode
本站的初衷是让大家不再使用过时、缓慢、问题频出的立即模式。The real aim of this website is that you DON'T use immediate mode, which is deprecated, slow, and problematic in many aspects.

不过，用立即模式进行调试却十分方便：However, it also happens to be really handy for debugging :

<a href="http://www.opengl-tutorial.org/wp-content/uploads/2011/05/immediatemodedebugging.png"><img class="alignnone size-large wp-image-314" title="immediatemodedebugging" src="http://www.opengl-tutorial.org/wp-content/uploads/2011/05/immediatemodedebugging-1024x793.png" alt="" width="640" height="495" /></a>

这里，我们在立即模式下画了一些线条表示切线坐标系。Here we visualize our tangent space with lines drawn in immediate mode.

要进入立即模式，得关闭3.3内核配置：For this, you need to abandon the 3.3 core profile :
glfwOpenWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_COMPAT_PROFILE);
然后把矩阵传给旧式的OpenGL流水线（你也可以另写一个着色器，不过这样做更简单，反正都是在hacking）：then give our matrices to OpenGL's old-school pipeline (you can write another shader too, but it's simpler this way, and you're hacking anyway) :
glMatrixMode(GL_PROJECTION);
glLoadMatrixf((const GLfloat*)&ProjectionMatrix[0]);
glMatrixMode(GL_MODELVIEW);
glm::mat4 MV = ViewMatrix * ModelMatrix;
glLoadMatrixf((const GLfloat*)&MV[0]);
禁用着色器：Disable shaders :
glUseProgram(0);
然后画线条（本例中法线都已被归一化，乘了0.1，放到了对应顶点上）：And draw your lines (in this case, normals, normalized and multiplied by 0.1, and applied at the correct vertex) :
glColor3f(0,0,1);
glBegin(GL_LINES);
for (int i=0; i<indices.size(); i++){
??? glm::vec3 p = indexed_vertices[indices[i]];
??? glVertex3fv(&p.x);
??? glm::vec3 o = glm::normalize(indexed_normals[indices[i]]);
??? p+=o*0.1f;
??? glVertex3fv(&p.x);
}
glEnd();
记住：实际项目中别用立即模式！只在调试时用！别忘了之后恢复到内核配置，它可以保证不会启用立即模式！Remember : don't use immediate mode in real world ! Only for debugging ! And don't forget to re-enable the core profile afterwards, it will make sure that you don't do such things.
调试颜色Debugging with colors
调试时，将向量的值可视化很有用。最简单的方法是把向量都写到帧缓冲区。举个例子，我们把LightDirection_tangentspace可视化一下试试：When debugging, it can be useful to visualize the value of a vector. The easiest way to do this is to write it on the framebuffer instead of the actual colour. For instance, let's visualize LightDirection_tangentspace :
color.xyz = LightDirection_tangentspace;
<a href="http://www.opengl-tutorial.org/wp-content/uploads/2011/05/colordebugging.png"><img class="alignnone size-large wp-image-313" title="colordebugging" src="http://www.opengl-tutorial.org/wp-content/uploads/2011/05/colordebugging-1024x793.png" alt="" width="640" height="495" /></a>

这说明：This means :

	在圆柱体的右侧，光线（如白色线条所示）是朝上（在切线坐标系中）的。也就是说，光线和三角形的法线同向。On the right part of the cylinder, the light (represented by the small white line) is UP (in tangent space). In other words, the light is in the direction of the normal of the triangles.
	O在圆柱体的中间部分，光线和切线方向（指向+X）同向。n the middle part of the cylinder, the light is in the direction of the tangent (towards +X)

友情提示：A few tips :

	可视化前，变量是否需要规范化？这取决于具体情况。Depending on what you're trying to visualize, you may want to normalize it.
	如果结果不好看懂，就逐分量地可视化。比如，只观察红色，而将绿色和蓝色分量强制设为0。If you can't make sense of what you're seeing, visualize all components separately by forcing for instance green and blue to 0.
	别折腾alpha值，太复杂了:)Avoid messing with alpha, it's too complicated :)
	若想将一个负值可视化，可以采用和处理法线纹理一样的技巧：转而把(v+1.0)/2.0可视化，于是黑色就代表-1，而白色代表+1。只不过这样做有点绕弯子。If you want to visualize negative value, you can use the same trick that our normal textures use : visualize (v+1.0)/2.0 instead, so that black means -1 and full color means +1. It's hard to understand what you see, though.

?
调试变量名Debugging with variable names
前面已经讲过了，搞清楚向量所处的坐标系至关重要。千万别把一个观察坐标系里的向量和一个模型坐标系里的向量做点乘。As already stated before, it's crucial to exactly know in which space your vectors are. Don't take the dot product of a vector in camera space and a vector in model space.

在向量名称后缀“_modelspace”可以有效地避免这类计算错误。Appending the space of each vector in their names ("..._modelspace") helps fixing math bugs tremendously.
练习Exercises

	在indexVBO_TBN函数中，在做加法前把向量归一化，看看结果。Normalize the vectors in indexVBO_TBN before the addition and see what it does.
	用颜色可视化其他向量（如instance、EyeDirection_tangentspace），试着解释你看到的结果。Visualize other vectors (for instance, EyeDirection_tangentspace) in color mode, and try to make sense of what you see

工具和链接Tools & Links

	<a href="http://www.crazybump.com/">Crazybump</a> 制作法线纹理的好工具，收费。, a great tool to make normal maps. Not free.
	<a href="http://developer.nvidia.com/nvidia-texture-tools-adobe-photoshop">Nvidia's photoshop plugin</a>免费，不过Photoshop不免费……. Free, but photoshop isn't...
	<a href="http://www.zarria.net/nrmphoto/nrmphoto.html">Make your own normal maps out of several photos</a>
	<a href="http://www.katsbits.com/tutorials/textures/making-normal-maps-from-photographs.php">Make your own normal maps out of one photo</a>
	关于矩阵转置的详细资料Some more info on <a href="http://www.katjaas.nl/transpose/transpose.html">matrix transpose</a>

参考文献References

	<a href="http://www.terathon.com/code/tangent.html">Lengyel, Eric. “Computing Tangent Space Basis Vectors for an Arbitrary Mesh”. Terathon Software 3D Graphics Library, 2001.</a>
	<a href="http://www.amazon.com/dp/1568814240">Real Time Rendering, third edition</a>
	<a href="http://www.amazon.com/dp/1584504250">ShaderX4</a>

?