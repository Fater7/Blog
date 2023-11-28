# Animation系统

- Animator
    - avatar：骨骼相关信息。
- AnimationClip，一个具体的动画实体。
- AnimationClipCurveData，AnimationClip中一条曲线的全部数据。
    - propertyName：该曲线影响的属性名称，如`transform.position.x`。
    - path：曲线所属的GameObject的路径。
    - curve：AnimationCurve对象。
- AnimationCurve，曲线描述实例。
    - keys：KeyFrame结构数组。
    - length：keys数量。
    - postWrapMode：时间结束后的行为。
    - preWrapMode：时间结束前的行为。
- AnimationController，一组动画。
    - layers：AnimatorControllerLayer数组，每一层独立持有一个动画状态机。
    - animationClips：该控制器使用的所有Clip。
- AnimatorControllerLayer，一层动画，只持有状态机数据，clip被controller持有。
    - stateMachine：该层的动画状态机，AnimatorStateMachine类型。

- AnimatorStateMachine，动画状态机