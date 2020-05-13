
一个简单地Shell脚本实现自动出包、输入Scheme名称即可快速实现framework打包、并自动合成真机和模拟器包。

针对IC2和VARecord的framework静态包制作及各基础依赖文件的关系图

```
基础包:
mongoose, libflv, libmov, AGSCom

VARecord Framework
    AGSCom, libmov, libflv

CC Framework
    http-parser, libevent-2.1.8-stable, AGSCom

DTC Framework
    http-parser, libevent-2.1.8-stable, MC

IC2 Framework
    mongoose, MC, MOSBase, DTC, AGSCom, CC

    1、输入需要编译的scheme名称
    2、根据scheme名称首先编译依赖的scheme、并将编译生成的静态库文件copy到scheme名称的工程内
    3、编译scheme功能、生成静态库文件、

    重复1、2、3、

```


以下是Shell脚本内容，新建文件将脚本内容复制到文件内、
1、新建文件
    touch yourShellName.sh
2、给脚本文件赋予可执行权限
    chmod u+x yourShellName.sh
3、执行Shell脚本 
    bash yourShellName.sh 或者  ./yourShellName.sh

================================= Shell Begin =================================
```

echo ""
echo ".......开始编译......."
echo ""

read -p "Enter your workspace dir, please: " WORKSPACE_DIR
read -p "Enter your sheme name, please: " SHEME_NAME

echo "workspace dir : ${WORKSPACE_DIR}"
echo "sheme name : ${SHEME_NAME}"

#构建的组态 Debug Or Release
UNIVERSAL_CONFIGURATION="Release"

#输出合成静态库文件目录
UNIVERSAL_OUTPUT_FOLDER="${WORKSPACE_DIR}/Product"
#构建的真机和模拟器库文件目录
UNIVERSAL_BUILD_FOLDER="${UNIVERSAL_OUTPUT_FOLDER}/Build"

function buildScheme() {
    # 分别编译真机和模拟器静态库文件
    echo "build scheme 参数 : $1"
    xcodebuild -scheme "$1" ONLY_ACTIVE_ARCH=NO -configuration "${UNIVERSAL_CONFIGURATION}" -sdk iphoneos BUILD_DIR="${UNIVERSAL_BUILD_FOLDER}" BUILD_ROOT="${UNIVERSAL_BUILD_FOLDER}" clean build
    xcodebuild -scheme "$1" ONLY_ACTIVE_ARCH=NO -configuration "${UNIVERSAL_CONFIGURATION}" -sdk iphonesimulator BUILD_DIR="${UNIVERSAL_BUILD_FOLDER}" BUILD_ROOT="${UNIVERSAL_BUILD_FOLDER}" clean build
        
    # 将生成的静态库头文件Copy到UNIVERSAL_OUTPUT_FOLDER/Product/scheme/include文件夹内
    OUTPUT_HEADER_FOLDER="${UNIVERSAL_OUTPUT_FOLDER}/$1/include"
    rm -rf "${OUTPUT_HEADER_FOLDER}"
    headerFolderIsExist "${OUTPUT_HEADER_FOLDER}"
    HEADER_FOLDER="${UNIVERSAL_BUILD_FOLDER}/${UNIVERSAL_CONFIGURATION}-iphoneos/usr/local/include/"
    if [[ -d "${HEADER_FOLDER}" ]]
    then
        HEADER_FOLDER="${UNIVERSAL_BUILD_FOLDER}/${UNIVERSAL_CONFIGURATION}-iphonesimulator/usr/local/include/"
    fi
    cp -af "${HEADER_FOLDER}" "${OUTPUT_HEADER_FOLDER}"

    #合成真机和模拟器.a静态包
    OUTPUT_LIB_FOLDER="${UNIVERSAL_OUTPUT_FOLDER}/$1/lib/"
    headerFolderIsExist "${OUTPUT_LIB_FOLDER}"
    lipo -create "${UNIVERSAL_BUILD_FOLDER}/${UNIVERSAL_CONFIGURATION}-iphonesimulator/lib$1.a" "${UNIVERSAL_BUILD_FOLDER}/${UNIVERSAL_CONFIGURATION}-iphoneos/lib$1.a" -output "${OUTPUT_LIB_FOLDER}/lib$1.a"
}

function headerFolderIsExist() {
    if [ -d "$1" ]
        mkdir -p "$1"
    then
        echo "dir: $1 已经存在"
    fi
}

#构建之前先移除旧的编译文件
rm -rf "${UNIVERSAL_BUILD_FOLDER}"
headerFolderIsExist "${UNIVERSAL_OUTPUT_FOLDER}"
headerFolderIsExist "${UNIVERSAL_BUILD_FOLDER}"

#调用函数对scheme进行编译
buildScheme "${SHEME_NAME}"

#echo "$PWD"
#UNIVERSAL_OUTPUT_FOLDER_PREFIX=$(dirname "$PWD")
#cd ${UNIVERSAL_OUTPUT_FOLDER_PREFIX}
#echo "${UNIVERSAL_OUTPUT_FOLDER_PREFIX}"

echo ""
echo ".......结束编译......."
echo ""

open "${UNIVERSAL_OUTPUT_FOLDER}/${SHEME_NAME}"

```
================================== Shell End ==================================


