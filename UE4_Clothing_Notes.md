1. 如果在3dsmax bind pose模式下刷的布料，一定要有一个pose array。UE4导入FBX如果找不到bind pose，会默认使用动画第一帧进行布料的初始化。（极大的错误）  
UE会提示 can not find bind pose（warning）  
目前遇到的一例是导出的fbx中的bind pose丢失deformers信息，bind pose不可用。
2. clothing导入导出  
工具栏->PhysX->Physx Clothing::Import Clothing Template和Export Clothing Template
3. UE4绑定布料的时候，有mesh to mesh，它不允许退化的三角形，也就是两边的夹角太小。（两边的叉乘小于1e-8），原因未明确。
