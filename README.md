# antiOllvm
anti ollvm like flat/bcf/sub

这是我个人做的一些关于去OLLVM混淆相关的进展。

利用符号执行的思路，目前在Android arm32的so代码上面能够有效的解决[孤挺花](https://github.com/GoSSIP-SJTU/Armariris)版本的平坦化混淆。

最重要的一点在于能够很好的patch指令，修复后的so代码能够直接利用ida的F5插件得到伪代码。

不过目前代码还在实验阶段，在识别真实块和还原控制流以及patch修复方面还有很长的适配、兼容工作需要完成。

Good Luck for me~


