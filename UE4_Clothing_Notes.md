1. 如果在3dsmax bind pose模式下刷的布料，一定要有一个pose array。UE4导入FBX如果找不到bind pose，会默认使用动画第一帧进行布料的初始化。（极大的错误）  
UE会提示 can not find bind pose（warning）  
目前遇到的一例是导出的fbx中的bind pose丢失deformers信息，bind pose不可用。
2. clothing导入导出  
工具栏->PhysX->Physx Clothing::Import Clothing Template和Export Clothing Template
3. UE4绑定布料的时候，有MeshToMesh，它不允许退化的三角形，也就是两边的夹角太小。（两边的叉乘小于1e-8），原因未明确。
4. **与环境物体进行碰撞** `USkeletalMeshCompoent::ProcesClothCollisionWithEnvironment()`逻辑有些不一样，这个是针对移动物体和场景的碰撞，碰撞类型目前只有静态物体（ECC_WorldStatic）和同为布料的物体(ECC_PhysicsBody)，而且仅在当前mesh移动的情况下才会更新。  
改变逻辑可以考虑。~~如果Capsule Collide在Skeletal Mesh之上，碰撞是以Capsule的碰撞类型为准。通过检测Overlap。Overlap如何检测到Capsule下的Skeletal Mesh~~ （原因是RigidBody没有参与场景的碰撞）。当前物体有一个碰撞范围，Bound。  
**布料的碰撞体** 并不在Physx的碰撞检测中，因为并没有在PxScene中创建Actor。PhysicsBody的碰撞必须要有物理胶囊体的支持。动画蓝图的RigidBody不行，应该是因为RigidBody中的碰撞体是创建在其他的PxScene中（待确定）  
提取其他布料的碰撞体，然后加到当前布料，问题非常大，因为碰撞体更新是需要骨骼的，UE4没考虑这一点。
5. 蓝图里bLocalSpaceSimulation已经弃用，布料模拟现在应该都是采用的Local Space Simulation。
6. 记住Ignore Backfacing和ignore Unpainted Regions的作用，方便美术操作。提醒美术灵活运用scale、add来刷数值。
7. 双层布料，布线问题，内外线能对应上？内外层留足够的距离。
8. 布料布线尽量垂直向下   
9. 布料起跑阶段，猛然弹起，是动画融合时间太短
