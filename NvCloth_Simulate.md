# Pre-sim works
更新physcial Mesh，更新Max Distance，更新碰撞体信息

# Simulation()
```c++
if(Solver->beginSimulation(DeltaTime))
{
    const int32 ChunkCount = Solver->GetSimulationChunkCount();
    for(int32 ChunkIdx = 0; ChunkIdx < ChunkCount; ++ChunkIdx)
    {
        Solver->simulationChunk(ChunkIdx);
    }
    Solver->endSimulation();
}
```
## > beginSimulation(DeltaTime)
1. 传入deltaTime 每一帧的tick时间，对于时间很长的帧，deltaTime会舍入到MaxPhysicsDeltaTime，默认是1/30s。也就是帧率低于30，布料运算会变慢。
2. beginFrame() Profile做了一个标记

## > simulateChunk(id)
实际的模拟过程
```c++
mSimulatedCloths[id].Simulate();
mSimulatedCloths[id].Destroy();
```
1. Simulate()大致分为四个阶段  
    1）InterationStateFactory输入为每一帧tick时间dt，根据dt和物理迭代频率计算并保存有每一次的迭代时间，上一次的迭代时间，平均迭代时间。  
    然后更新布料的线性速度和角速度  
    ```c++
    // update cloth
    float invFrameDt = 1.0f / frameDt;
    cloth.mLinearVelocity = invFrameDt * (cloth.mTargetMotion.p - cloth.mCurrentMotion.p);
    physx::PxQuat dq = cloth.mTargetMotion.q * cloth.mCurrentMotion.q.getConjugate();
    cloth.mAngularVelocity = log(dq) * invFrameDt;
    cloth.mPrevIterDt = mIterDt;
    cloth.mIterDtAvg.push(static_cast<uint32_t>(mNumIterations), mIterDt);
    cloth.mCurrentMotion = cloth.mTargetMotion;
    ```
    2）配置数据SwClothData  
    3）创建SwSolverKernel  
    4）更新cloth
2. 创建SwSolverKernel和运行SwSolverKernel   
    - IterationStateFactory::create()  
    根据角速度判断当前cloth是否在旋转 isTurning  
    - 迭代模拟  **以下讨论** 都以迭代次数为4讨论
    ```c++
    while(mState.mRemainingIterations)//这一帧剩余模拟次数
	{
		iterateCloth();
		mState.update();//更新剩余模拟次数
	}
    ```
    - **intergrateParticles()**  （TODO Local Space Simulate）
    积分方法计算布料各顶点位置。如果pos->w == 0.0f，也就是无限质量，那么当前顶点位置保持不变。  
    x_next = x_cur + (x_cur - x_prev) \* dt_cur/dt_prev \* damping + g\*dt\*dt  
    如果正在旋转的话（isTurning）（TODO）  
    - **applyWind()**  根据drag和lift参数进行计算  
    - **constrainMotion()** 根据MaxDistance对顶点运动进行约束  
    mMotionConstrains.mTarget：当前帧的MaxDistance
    mMotionConstrains.mStart：上一帧的MaxDistance  
    迭代中根据第几次迭代对上述两个数据进行插值  
    迭代中 **每4个顶点** 进行计算，delta[i]是约束球球心到当前顶点位置的差值。radius[i]是每个顶点的约束半径（MaxDistance）  
    ```
    Simd4f sqrLength = gSimd4fEpsilon + deltaX * deltaX + deltaY * deltaY + deltaZ * deltaZ;
	Simd4f radius = max(gSimd4fZero, deltaW * scale + bias);
	Simd4f slack = gSimd4fOne - radius * rsqrt(sqrLength);
    ```  
    slack大于0的，会在当前位置补偿delta[i]\*slack的位移。  
    - **constrainTether()**  
    每个顶点会有一个或多个固定（MaxDistance为0）的顶点作为Anchor（锚），顶点到Anchor之间的距离会被提前记录，当顶点当前位置到Anchor位置的距离超出记录的数值，会将差值减去。如何创建每个顶点的Anchor数据，参看createTetherData()。（TODO）  
    - **solvFabric()** （TODO）  
    Fabric的迭代分为4个类型。Vertical，四边面垂直边的形变；Horizontal，四边面水平边的形变；Shear，四边面对角线的形变；Bend，相邻两个面的弯曲程度（相对于两个面连接的边，体现为平行于连接边的两个边的上下顶点距离）。每部分有4个参数，stiffness是抗形变能力，multiplier抗形变能力比例系数，compressionLimit形变允许的压缩范围，stretchLimit形变允许的拉伸范围。假设静止时约束边长为r，布料运算后的约束边长为e，约束边两个顶点为A、B，那么误差比例为 er = 1-r/e，在此处引入约束如下
    ```
    er = er - multipier * max(compressionLimit, min(er, stretchLimit));
    // 通常传入的参数，compressionLimit 在（1-r/r 到 1-r/(0r)，即(-Inf,0]，stretchLimit在（1-r/r 到 1-r/(3r)，即[0:2/3]）

    ```
    然后乘上stiffness
    ```
    ex = er * stiffness * (invMassA + invMassB)
    ```
    最后按照质量计算补偿，
    ```
    posA = posA + e * ex * invMassA;
    posB = posB - e * ex * invMassB；
    ```
    参数对应关系stiffnessFrequency，paramMultipier，paramStiffness是我们调用api时设置的参数。
    ```
    stiffExp = stiffnessFrequency * dt / iteraCount;
    multipier = 1 - (paramMultipier) ^ stiffExp;
    stiffness = 1 - (1-paramStiffness) ^ stiffExp;
    ```
    另外当paramStiffness为0的时候，该类型不计算约束。
    - **constrainSeparation()**  对应Backstop Offset和Backstop Radius  
    该方法和constrainMotion()原理一致，只是将顶点限制在约束球以外的范围。
    - **SwCollision()**   
    1)`collideConvexes(state)`  凸包碰撞  
    2)`collideTriangles(state)`  三角面碰撞  
    3)计算每一帧的顶点包围盒  
    4)生成球形、胶囊体形碰撞范围。（球形会进行插值，**代码中胶囊体支持两边的球形半径不一样**）  
    5)`buildAcceleration()`先确定所有球形组成的包围盒和顶点包围盒，如果需要CCD（连续碰撞），则包围盒由当前球形、顶点和上一帧球形、顶点共同确定。然后确定球形包围盒和顶点包围盒的重叠部分，修正一下重叠部分的数字误差。将重叠包围盒按x，y，z三个方向各划分8个格子，用一个32bits的mask标记某个球形是否在格子中（此处限制了球形最多支持32个），每个球形有两个first和last标记，first标记记录了某坐标轴该球最小的格子到第7个格子，last标记了某坐标轴该球最大的格子到第0个格子。对于球0，球0所占的格子就是从first[0]到last[0]。对于球0和球1组成的胶囊体，所占格子就是两个球的并集。  
    6)计算碰撞， **每四个连续顶点** 计算。（TODO）  
    先计算胶囊体碰撞  
    然后计算剔除胶囊体球形的球形碰撞  
    预计算摩擦力相关  
    7)计算摩擦力  
    8)计算碰撞质量变换
    - **SelfCollision()**  实质是点对点的碰撞，效果一般，总的来说所有的在intergrateParticles()后的方法都是补偿，如果intergrateParticles()已经模拟到了一个错误位置，但是在SelfCollision()看来是正确的没有碰撞的位置（是有可能的），那就无法起到碰撞的作用，就像有些提问说的，急需点和面的碰撞。  
    创建数据，理论上可以自定义顶点SelfConllisionIndices（实际是一个Vertex表）。UE4创建selfCollision数据时，剔除了MaxDistance非常小的点，剔除了两点间的距离小于SelfCollisionRadius的其中一个点。
    将布料的包围盒（AABB的）沿各个轴进行网格划分，最长轴划分为65533，其他划分为253个格子。将每个顶点分配到每个格子中，格子的坐标index作为顶点的key（低16位为最长周格子id，其他各8bits为其他格子id），将key进行基数排序（O(n+4\*256)，n是参与selfcollision的顶点数），从小到大。假设从最长轴依次的坐标轴顺序是i，j，k，i对应低16bits，j对应17~24bits，k对应25~32bits。排序之后，可以知道k轴的格子序号是从低到高排列的。按排序后的循序依次搜索顶点附近的格子，共搜索5次。假设Nc是最长轴方向CollisionDistance/GridSize+**2**，也就是最长轴方向碰撞距离跨越的格子数量。假设I，J，K是i，j，k轴的格子序号，[K,J,I]组成了32bits的key值。第一次搜索是[K,J,I]到[K,J,I+Nc]的格子。后四次搜索为[K,J+1,I-Nc]到[K,J+1,I+Nc]，[K+1,J-1,I-Nc]到[K+1,J-1,I+Nc]，[K+1,J,I-Nc]到[K+1,J,I+Nc]，[K+1,J+1,I-Nc]到[K+1,J+1,I+Nc]（搜索的方式，先找到大于[K+1,J+1,I-Nc]的第一个点，记为kFirst，然后找到大于[K+1,J+1,I-Nc]的第一个点，记为kLast，kFirst和kLast之间的点就是搜索到的点，\*不包括kLast，kFirst=kLast的情况就是一个点都没有）。将当前顶点与这5次搜索的顶点做碰撞检测，最坏是O(n^2)，大部分情况是O(m\*n)。
    计算两点碰撞并补偿。delta = pos0 - pos1，任意坐标轴的距离超出CollisionDistance，忽略碰撞，左右距离都小于CollisionDistance计算碰撞。补偿后的结果如下
    ```
    pos0 = pos0 + w0/(w0+w1) * (delta*(1-|r|/|delta|))
    pos1 = pos1 - w1/(w0+w1) * (delta*(1-|r|/|delta|))
    //其中 w0 w1是两定点的质量倒数，r是碰撞距离CollisionDistance
    ```
    - **updateSleepState()** （TODO）  
    - **mState.Upate()** 更新旋转导致的初始偏移的旋转，剩余迭代次数减一  

## > endSimulation
1. **interCollision()**  同一个Solver的不同布料之间的SelfCollision碰撞检测，需要设置mInterCollisionDistance和mInterCollisionFilter。相对于SelfCollision多出来的部分（TODO）  
2. endFrame() Profile标记结束
