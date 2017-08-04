### Android 默认的多语言支持
在 Android 工程的 res 目录下，通过定义对应的语言文件夹名称就可以实现多语言支持

``` java
// 手动切换语言
Resources resources = getResources();  
Configuration config = resources.getConfiguration();  
DisplayMetrics dm = resources.getDisplayMetrics();
config.locale = Locale.ENGLISH;
resources.updateConfiguration(config, dm);
```

### 支持应用内的语言自动切换（可以不依赖系统语言手动切换应用的语言包）
#### 1.extends Resources 
``` java
// Application 重写 getResources 方法
@Override
public Resources getResources() {
	return MyResources.getInstance(super.getResources());
}
```
``` java
// 重写 getString, getStringArray, getValue, getText 等方法
public class MyResources extends Resources {
	@Override
	public String getString(int id) {
	    // 从对应语言的 sparseArray 中取出文案
		String value = MyLang.getString(id); 
		if(value == null) {
			value = super.getString(id);
		} 
		return value;
	}

}
```

#### 2.使用 json 管理语言文案，通过脚本将 json 转换为 java 文件, 其中java 文件用 sparseArray 保存文案信息。
``` javascript
// 转换代码  
/* 
 * 优化 SparseArray 的使用性能：
 * 1.让 SparseArray 长度固定。
 * 2.SparseArray 是基于二分查找，让其保持顺序可以优化查找性能。 
*/

var fs = require("fs");


var translate = function (json, javaName) {
    var start = '/* AUTO-GENERATED FILE.  DO NOT MODIFY.*/\n\
package com.xxx.lang;\n\n\
import android.util.SparseArray;\n\
import com.xxx.R;\
\n\
\n\
class ' + javaName + ' {\n\
    public static SparseArray<Object> getStringMap() {\n';


    var keys = Object.keys(json);
    var sortedKeys = keys.sort(function (a, b) {
        return a.localeCompare(b);
    });
    var value = "";
    var len = 0;
    var mapString = "";
    var mapArrayString = "";
    var mapObjectString = "";
    sortedKeys.forEach(function (key) {
        len++;
        value = json[key];
        var type = objType(value);
        if (type == "string") {
            mapString += '        map.put(R.string.' + key + ', "' + stringify(value) + '");\r\n';
        } else if (type == "array") {
            mapArrayString += '        map.put(R.array.' + key + ', new String[]{' + arrayToString(value) + '});\r\n';
        } else if (type == "object") {
            mapObjectString += '        map.put(R.plurals.' + key + ', new String[]{' + objToString(value) + '});\r\n';
        } else {
            len--;
        }
    });


    var end = '        return map;\n\
    }\n\
}';


    return start + "        SparseArray<Object> map = new SparseArray<>(" + len + ");\n" + mapString + mapArrayString + mapObjectString + end;
};


var stringify = function (ori) {
    return ori.replace(/\\/g, '\\\\')
        .replace(/"/g, '\\"')
        .replace(/\\"/g, '\\"')
        .replace(/\n/g, '\\n')
        .replace(/\t/g, '\\t')
        ;
};

var arrayToString = function (arr) {
    var ret = "";
    var len = arr.length;
    for (var i = 0; i < len; i++) {
        ret += '"' + stringify(arr[i]) + '"';
        if (i != len - 1) {
            ret += ', ';
        }
    }

    return ret;
};

var objToString = function (obj) {
    var arr = [];
    for (var key in obj) {
        if (obj.hasOwnProperty(key)) {
            arr[key | 0] = obj[key];
        }
    }
    return arrayToString(arr);
};

var objType = function (obj) {
    return obj == null/*||isNaN(obj)*/ ? 
        String(obj) :
        (Object.prototype.toString.call(obj).match(/^\[object\s(.*)\]$/)[1] || "").toLowerCase();
};

var trans = function (path, javaName) {
    var source = "", ret = "";
    try {
        source = require(path);
        ret = translate(source, javaName);
    } catch (e) {
        console.error(e);
        return;
    }
    return ret;
};

module.exports = trans;
```

#### 3.让 gradle 支持 nodejs
[gradle-node-plugin](https://github.com/srs/gradle-node-plugin)
``` java
// 创建一个 gradle 文件
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath "com.moowork.gradle:gradle-node-plugin:0.14"
    }
}
apply plugin: "com.moowork.node"


task genLang(type: NodeTask) {
    script = file("scripts/gen.js")
    execOverrides {
        it.ignoreExitValue = true
        it.workingDir = "scripts/"
    }
}


node {
    version = "7.2.0"

    distBaseUrl = "https://nodejs.org/dist"
//    distBaseUrl = "http://npm.taobao.org/mirrors/node"

    // If true, it will download node using above parameters.
    // If false, it will try to use globally installed node.
    download = true

    workDir = file("${project.buildDir}/nodejs")

    npmWorkDir = file("${project.buildDir}/npm")

    nodeModulesDir = file("${project.projectDir}")
}
```



