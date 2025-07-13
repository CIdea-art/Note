jcmd <JAVA进程id> VM.unlock_commercial_features  （解锁JFR记录功能权限

）

jcmd <JAVA进程id> JFR.start duration=300s name=hbv filename=hbv.jfr  （jfr分析结果会存在当前路径下的hbv.jfr 文件里, duration为分析时间）

duration配置时间或者jcmd 37578 JFR.stop手动停止

jmc工具下载地址

https://www.oracle.com/java/technologies/javase/products-jmc8-downloads.html