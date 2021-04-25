两种调试方法：

remake -x

make package/smart_qos_manager/{clean,compile,install}  -j1  DEBUG=all



//打印出编译过程中的变量

 make  package/smart_qos_manager/compile V=s --print-data-base | grep "smart_qos"