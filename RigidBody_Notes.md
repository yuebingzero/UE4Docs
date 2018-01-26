参考Best Practices Guide
Jointed objects are unstable非常多的建议，值得尝试
1. 提升迭代次数，尤其是position iterations。position iterations和velocity iterations 4倍之后未见改善。
2. 将constrain创建多次，比如我们有ABCD四个constraint，创建顺序ABCDABCDABCDABCD比AAAABBBBCCCCDDDD好
3. joint projection。比较少数量的连接joint可以尝试
4. 用更小的time step，expensive但是effective。1次simulation call在dt时间N次迭代，用N次simulation calls在dt/N时间内1次迭代。
5. Consider tweaking inertia tensors. In particular, for ropes or chains of jointed objects, the PxJoint::setInvMassScale and PxJoint::setInvInertiaScale functions can be quite effective. An alternative is to compute the inertia tensor (e.g. using PxRigidBodyExt::setMassAndUpdateInertia) with an artificially increased mass, and then set the proper mass directly afterwards (using PxRigidBody::setMass).
7. Use spheres instead of capsules. A rope made of spheres will be more stable than a rope made of capsules. The positions of pivots can also affect stability. Placing the pivots at the spheres' centers is more stable than placing them on the spheres' surfaces.
用球形代替胶囊体，球形球心的位置尽量接近joints的位置（效果非常明显），但是碰撞处理方便么？（是不是能够考虑用胶囊体，但是把inertia xyz设成统一的来模拟球体？）
8. Use articulations. Perhaps not surprisingly, articulations are much better at simulating articulated objects. They can be used to model better ropes, bridges, vehicles, or ragdolls out-of-the-box, without the need for the above workarounds. Please refer to the Articulations chapter for details. They are more expensive than regular joints though.
工作量大，目前
