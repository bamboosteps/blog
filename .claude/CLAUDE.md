# 工作规则

- 修改完文件后,不要自动运行 `npm run start`(或 `npm run serve` 等本地预览命令)。
- 修改完文件后,不要自动执行 `git commit` / `git push`,等待用户确认后再提交。
- 文章里所有驱动代码和 app 代码,凡是超过 10 行的代码块,默认用 `<details><summary>Show code</summary>` 折叠起来(不加 `open` 属性),不要直接平铺展示。10 行以内的代码片段(比如宏展开、单个函数签名)不用折叠。summary 统一用英文提示词 `Show code`,不要把文件名写进 summary(文件名保留在代码块内部的注释里就行)。
