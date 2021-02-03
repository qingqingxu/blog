# 先说下ASAN的应用，很简单，坑很多

## ASAN介绍以及使用案例

## https://github.com/google/sanitizers/wiki/AddressSanitizer

## 总结起来就两点
1. **编译选项**：ASAN_CFLAGS += -fno-stack-protector -fno-omit-frame-pointer -fno-var-tracking -g -O0 -ggdb
2. **进程拉起前ASAN的预加载和环境变量导入**
   export LD_PRELOAD=libasan.so.4
   export ASAN_OPTIONS=halt_on_error=0:use_sigaltstack=0:detect_leaks=1:malloc_context_size=15:log_path=./asan.log

## gcc版本对ASAN相关检测的支持，当前使用版本gcc 4.8.5, 存在如下几个问题：

1. 不支持编译选项-fsanitize-recover=address，即ASAN检测出内存使用错误后进程会退出，无法继续运行，当前问题无法解决，将阻塞后续测试验证
2. ASAN检测出来错误信息打印的堆栈无符号信息，可定位性极差

## 采取措施

**升级编译环境gcc版本（gcc 7.3）**，升级gcc这想法以后都不敢轻易有了，升级gcc应该考虑当前编译环境已安装的软件对glibc等的依赖，比如：python依赖的库从低版本变成了高版本，直接导致python无法使用，当对于工程相对比较大的编译来说，可能对不同模块使用的一些接口也是有影响的
**什么场景可以升级gcc或者如果如何高效升级？**

①确认无其他软件对gcc相关库文件有依赖可以直接升级

②当然如果是gcc安装自定义路径，目录隔离也是可以的

③直接重装高版本系统

## 最终采取措施

直接换高版本的编译系统，整个工程的代码全量编译，运行环境使用高版本系统（毕竟只是用于内测，不对外发布，没必要浪费太多时间在适配上）

## 运行时的各种问题

1. 在依赖open-jdk的进程中对ASAN应用时出现系统命令相关段错误导致进程拉起失败

   网上搜到了相关解决方案：https://blog.gypsyengineer.com/en/security/running-java-with-addresssanitizer.html，主要是ASAN_OPTIONS配置项的设置，handle_segv=0，ASAN不接管系统产生的sigsegv信号，由操作系统或其他方式进行处理

2. 还是ASAN_OPTIONS：detect_leaks=0，先说下ASAN_OPTIONS，重点关注**Default value**，可根据个人实际需求配置

   https://github.com/google/sanitizers/wiki/SanitizerCommonFlags

   - handle_segv：接管系统产生的sigsegv信号，接管后ASAN打印对应的ASAN日志，不接管由系统处理，在可定位性方面如果有coredump配置，ASAN日志无法定位情况下不接管该信号，利用产生的coredump文件定位也是不错的选择
   - detect_leaks：泄露检测，ARM环境的进程如果涉及大量的shell命令操作就不要打开了，真的很卡很卡，检测结果大都非个人代码问题
   - detect_stack_use_after_return：检测还在使用已返回栈内存，这个检测配置打开其实还是检测出来了问题，但有个场景出现了cpu冲高导致业务心跳断链的问题，背景：进程中只有一个线程接收处理外部发送过来的消息，asan频繁申请大量的栈内存，导致消息处理很慢，至于为啥出现cpu冲高，这点目前还是没搞明白

3. ​	