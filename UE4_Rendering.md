# Render
FViewInfo  
FSceneViewState  
FDeferredShadingSceneRenderer::Render(FRHICommandListImmediate& RHICmdList)
## PrepareViewRectsForRendering
获取视口的分辨率，根据画面质量等级设置分辨率缩放
BaseScalability.ini  
屏幕百分比PerfIndexValues_ResolutionQuaity = "50 71 87 100 100"  
对应像素数缩放 "25% 50% 75% 100% 100%"   
控制台参数 CVarScreenPercentage(static 编辑器生命周期生效)
## Render init
## InitViews
Find the visible primitives
### -PreVisbilityFrameSetup
* DoLazyStaticMeshUpdate  
NumRemovedPerFrame = 10 每帧只更新10个StaticMesh  
* Scene->FXSystem->PreInitViews()  
GPU粒子Frame步进，交换当前和之前的Textures(double buffer)  
重置排序的粒子模拟
* Setup montin blur parameters (also check for cameramovement thresholds)  
* 决定是否要遮挡查询和其屏幕范围（Occlusion query）  
* SetupDistanceFieldTemporalOffset  
* 处理TemporalAA Jitter
* **TO BE CONTINUED**  

### -ComputeViewVisibility
* GetPrecomputedVisibilityData() 获取预计算的可见性数据（如果有的话）  
* FrustumCulling FrustumCull&lt;false, false&gt;() 视锥体剔除 bool 是否自定义Cull, bool 是否用球形检测   
 [参考链接](https://udn.unrealengine.com/questions/252385/performance-of-frustumcull.html)  
* 



## PrePass
EarlyZPassMode:  
DDM_NonMaskedOnly  only opaque materials  
DDM_AllOccluders  opaque and masked materials 不包含非遮光板  
DDM_AllOpaque
### -BeginRendringPrePass
