#os #linux #systemctl
`systemctl freeze` 会使用 cgroup 的 freeze 功能，让进程立刻冻结，但是保留进程运行时状态和上下文，调试完可以用 thaw 恢复，此外，`systemctl whoami` 可以立刻获取进程所属的单元（最近的那个单元，不是最远的层级）。