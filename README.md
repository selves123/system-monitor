## 合肥新桥机场系统监控脚本
## System Monitor Scripts For Hefei Xinqiao International Airport

### 实现以下特性：
- Ping监控
- TCP协议监控
- HTTP协议监控
- DNS记录监控
- 后台服务监控（守护进程监控）
- 硬盘空间监控
- 负载监控


### 
### 简单运行
``` python monitor.py ```

### 命令行参数
```
-h, --help: display help Configuration  
-f CONFIG, --config=CONFIG: configuration file (monitor.ini)
-p PIDFILE, --pidfile=PIDFILE: Write PID into this file
-N, --no-network: Disable network listening socket (if enabled in config)
```