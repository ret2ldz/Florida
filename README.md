### 修复/掩盖的 frida 特征（共 12 个）：                                                                                                  
                                                                                                                                          
  1. "frida:rpc" 协议魔数字符串 → base64 解码随机名（0001）                                                                                 
  2. frida-agent-<arch>.so 库文件名 → 运行时 UUID 前缀（0002）                                                                              
  3. frida_agent_main 导出符号 → main（0003 + anti-anti-frida.py 重命名所有含 frida/FRIDA 的 ELF 符号）                                     
  4. gum-js-loop / gmain / gdbus / pool-spawner 线程名 → 随机字符（0004/0005/0007/0015）                                                    
  5. FridaScriptEngine / GLib-GIO / GDBusProxy / GumScript rodata 字符串 → 反转（0007）                                                     
  6. g_set_prgname("frida") 进程名 → ggbond（0008 + gum 0001）                                                                              
  7. memfd 名 frida-agent-<arch>.so → jit-cache（0009）                                                                                     
  8. u:object_r:frida_file:s0 / frida_memfd SELinux context → ggbond_*（0012）                                                              
  9. exit/_exit/abort 和 signal/sigaction 的 interceptor hook → 注释掉，去掉 trampoline（0013 + gum 0002）                                  
  10. "frida-" 临时文件前缀 → ggbond-（0014）                                                                                               
  11. 残留 frida\0 / FRIDA\0 在 .rodata 的 standalone 字符串 → 等长替换成 libgc\0 / XBNDL\0（0015）                                         
  12. /proc/net/unix 里 @/frida-zymbiote-<32hex> abstract socket → @/ggbnd-zymbiote-...，含 binary patcher 修预构建 ELF（0017）             
                                                                                                                                            
  可绕过的反调试：进程名扫描、/proc/self/maps 扫 libfrida-agent、/proc/self/task/*/comm 扫线程名、readlink /proc/self/fd/* 扫 memfd         
  名、dlsym 扫 frida_agent_main、getfilecon 扫 SELinux label、exit() 触发监听器检测、signal handler 被 swallow 检测、/data/local/tmp/frida-*
   文件扫描、/proc/net/unix 扫 frida、ELF .rodata 字符串扫描。                                                                              
                                                                                                                                          
  仍未覆盖：默认端口 27042、D-Bus 协议接口名 re.frida.HostSession17 等（改了会断 frida 客户端协议）、re.frida.Helper Java 包（预构建        
  helper.dex 不重建）、GType 名 FridaServer 等、TracerPid、agent dlopen 路径痕迹、QuickJS 常量。

### 建议运行方式

```
# ./florida -l 127.0.0.1:56789 &

> adb forward tcp:56789 tcp:56789
> frida -H 127.0.0.1:56789 -f com.app -l trace.js
```

### 写在最后

todo，

