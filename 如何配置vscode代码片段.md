## 如何配置vscode代码片段

### 添加Python头部注释

**Code -> Preferences -> User Snappets -> python.json**  

    "pyheader": {   
    	"prefix": "pyheader",   
    	"body": [
    		"# -*- coding: UTF-8 -*-",
    		"'''",
    		"# @File    : $TM_FILENAME",
    		"# @Time    : $CURRENT_YEAR/$CURRENT_MONTH/$CURRENT_DATE $CURRENT_HOUR:$CURRENT_MINUTE:$CURRENT_SECOND",
    		"# @Author  : Zhang Hai",
    		"# @Version : 0.1",
    		"# @Contact : zhanghai04@maoyan.com",
    		"'''",
    		"",
    	]
    }   