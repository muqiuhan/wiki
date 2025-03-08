这本书描述了 100 个最常见和最关键的 TypeScript 使用错误的场景，但目前还没写完，像是一个经验性的小册子。

当前有的章节主要内容有:
- 常见的基础错误
	- 过度使用 any
	- 不使用 strict mode，不正确使用变量以及滥用 [optional chaining](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-7.html)
	- 过度使用 [nullish](https://mariusschulz.com/blog/nullish-coalescing-the-operator-in-typescript)
	- 滥用或不恰当得使用 module export
	- 混淆 == 和 ===
	- 忽略类型推导
- 错误使用类型，别名和接口
	- 混淆类型别名和接口
	- 错误理解或管理 [Type widening](https://www.typescriptlang.org/play/?#example/type-widening-and-narrowing)
	- 如何适当的使用 [Type guards](https://app.immersivetranslate.com/pdf/)
	- 如何正确使用 readonly 修饰
	- 如何正确使用 keyof 和 [Extract](https://www.typescriptlang.org/docs/handbook/utility-types.html#extracttype-union) 等实用类型
- 规范地编写函数和方法
	- 如何正确编写重载函数的签名来增强类型安全
	- 明确函数的返回类型
	- 在函数中正确使用 [Rest 参数](https://www.tutorialsteacher.com/typescript/rest-parameters)
	- 正确使用 this 和 globalThis 以及 bind, apply, call, StrictBindingCallApply 
	- 适当的为函数使用 ReturnType, Paramaters, Partial, ThisParamaterType, OmitThisParamater 等实用类型。
- 规范地编写类和构造函数
	- 实现接口和抽象类
	- 管理静态成员和访问修饰符
	- 合理地初始化类中的属性
	- 组织类的 getter setter
	- 在类中使用装饰器
	- 确保安全的 overrides