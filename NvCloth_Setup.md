# Setup
### NvCloth library
Uses callbacks for things like memory allocations, which need to be initialized.
### Factroy
```
#include <NvCloth/Factory.h>

...

nv::cloth::Factory* factory = NvClothCreateFactoryCPU();
if(factory==nullptr)
{
       //error
}

...

//At cleanup:
NvClothDestroyFactory(factory); //This works for all different factories.
```
### Fabric
```
#include <NvClothExt/ClothFabricCooker.h>

...

nv::cloth::ClothMeshDesc meshDesc;

//Fill meshDesc with data
meshDesc.setToDefault();
meshDesc.points.data = vertexArray;
meshDesc.points.stride = sizeof(vertexArray[0]);
meshDesc.points.count = vertexCount;
//meshDesc.triangels,meshDesc.invMasses
//etc. for quads, triangles and invMasses

//UE4 创建四边面，nvcloth对四边面更好
FClothingSystemRuntimeModule& ClothingModule = FModuleManager::Get().LoadModuleChecked<FClothingSystemRuntimeModule>("ClothingSystemRuntime");
nv::cloth::CLothMeshQuadifier* Quadifier = ClothingModule.GetQuadifier();
Quadifier->quadify(meshDesc);


physx::PxVec3 gravity(0.0f, -9.8f, 0.0f);
nv::cloth::Vector<int32_t>::Type phaseTypeInfo;
nv::cloth::Fabric* Fabric = NvClothCookFabricFromMesh(factory, meshDesc, gravity, &phaseTypeInfo, true);

...
//release
fabric->decRefCount();
```
### Cloth
```
// x y z staring frame positions;
// w invMass , 0 for static particle;
physx::PxVec4* particlePositions = ...;

nv::cloth::Cloth* Cloth = Factory->createCloth(nv::cloth::Range<physx::PxVec4>(particlePositions, particlePositions + particleCount), *Fabric);
// particlePositions can be freed here.

...

NV_CLOTH_DELETE(Cloth);

```
### Setup Phase(constrain) information

### Set up self collision indices (FIX ME)
```
Cloth->setSelfCollisionIndices(NvClothSupport::CreateRange(PhysMesh.SelfCollisionIndices));
```
### Set up motion constrains (max distance)

### Solver
```
nv::cloth::Solver* solver = factory->createSolver();

...

solver->addCloth(cloth);

...

solver->removeCloth(cloth);

...

NV_CLOTH_DELETE(solver);
```
