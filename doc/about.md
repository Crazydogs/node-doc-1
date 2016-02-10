本文档致力于全面细致地从概念和实践两方面介绍 Node.js 的 API 用法，文章整体分多个章节，每一章节针对一个模块或一个高级概念。

## 稳定性

在阅读文档的过程中，你会经常看到如下四种稳定性标识，用于说明当前模块的稳定程度。目前，Node.js 的 API 尚在完善之中，所以有的模块成熟稳定，有的模块尚处于实验阶段，有的模块正在重构。

<div class="s s0">
接口存在问题，不建议使用，无法有效保障向后兼容性。
</div>
<div class="s s1">
接口尚不稳定，未来可能会被修改或移除。
</div>
<div class="s s2">
接口表现成熟，除非绝对需要，否则不会修改，以 npm 开发环境的向后兼容性为优先原则。
</div>
<div class="s s3">
接口已锁定，不接受新的 API 建议，后续只会进行安全、性能或 Bug 方面的修改和完善。
</div>





























<style>
.s {
    margin: 1.5rem 0;
    padding: 10px 20px;
    color: white;
    border-radius: 5px;
}
.s:before {
    display: block;
    font-size: 2rem;
    font-weight: 900;
}
.s0 {
    background-color: #C04848;
}
.s0:before {
    content: "接口稳定性: 0 - 已过时";
}
.s1 {
    background-color: #F07241;
}
.s1:before {
    content: "接口稳定性: 1 - 实验中";
}
.s2 {
    background-color: #457D97;
}
.s2:before {
    content: "接口稳定性: 2 - 稳定";
}
.s3 {
    background-color: #14C3A2;
}
.s3:before {
    content: "接口稳定性: 3 - 已锁定";
}
</style>