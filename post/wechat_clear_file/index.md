>最近在做微信小游戏的项目，使用的是cocos creator的引擎做的开发，虽然引擎方有MD5码但是在清理本地缓存文件的功能上并不完善，在版本不断的迭代添加新的内容之后发现本地的缓存文件逐渐增长，有超出50M的趋势，经测试发现超出后就不能进入游戏了，
因此到了必须要解决的时候了。

<!--more-->
首先是理清解决问题的思路：
  1) 找到目标文件。
  2) 计算目标文件的大小。
  3) 判断目标文件的大小是否达到限制，是-->清理， 否-->进入游戏


整理好了思路接下来就是针对每一步的具体分析操作。

# 找目标文件

>微信提供了这方面的接口：

[FileSystemManager.rmdir(Object object)](https://developers.weixin.qq.com/minigame/dev/api/file/FileSystemManager.rmdir.html) 删除目录 

[FileSystemManager.readdir(Object object)](https://developers.weixin.qq.com/minigame/dev/api/file/FileSystemManager.readdir.html) 读取目录内文件列表

[FileSystemManager.getFileInfo(Object object)](https://developers.weixin.qq.com/minigame/dev/api/file/FileSystemManager.getFileInfo.html)  获取该小程序下的 本地临时文件 或 本地缓存文件 信息

这些就是清理操作所需要的的几个接口，具体的接口信息可以阅读[微信官方文档](https://developers.weixin.qq.com/minigame/dev/api/file/FileSystemManager.html)。

如何找到需要删除的文件？
---
<wx.env.USER_DATA_PATH + '/gamecaches'>  这是cocos creator 生成的存放游戏资源的文件地址，使用readdir接口就可以读取到这个文件价下面资源列表

如果这个文件下面还有游戏方自己下载的压缩包解压后的文件夹，可以使用读取的接口进行递归，然后获取列表信息（我在测试删除解压的文件的时候因为某些文件名带有中文导致下载到本地之后出现文件名乱码，导致最终微信的rmdir操作失败，即使删除游戏、删除微信都不能删除这些乱码文件的信息，但是实际上这些文件已经不存在与内存中，经测试并没有占用50M的内存空间，这个问题待后面找出解决办法再做分享）。

# 计算文件大小是否达到限制
我最开始想的是文件触发50M的限制之后进行删除操作，但是查了微信官方文档和cocos creator文档以及相关的论坛之后发现并不存在这样一个接口，而是触发之后直接报错，作为开发者是收不到警告的，因此这里就只有一个办法 ：自己找出文件计算大小。（这里必须要吐槽一下，微信既然做了这个限制，个人觉得就应该在触发限制后给出警告消息，不给警告也至少给一个目前存了多少东西给个统计也行啊，不然真的碰到的之后只能抓瞎。同时也要批评一下cocos creator 既然做了MD5 也知道问题的存在，咋就不改呢？）。

对于计算文件大小这里我最开始的做法是找出所有文件，然后获取文件大小，进行累加之后判断是否达到了50M，这会造成一个问题，如果本地存储达到了49.99M也不会触发清除操作，那么我的游戏再更新超过0.01M就会卡死。所以只能讲条件往前移，那么多少是合适的呢？我根据我们游戏现有的资源，以及将来可能的更新给出了一个25M的值，即达到25M就进行一次清理。但是这又有一个新的问题，如果每次进入游戏都进行计算那么每次都已有大概1-2秒的时间无法进入游戏，这个影响很大哦。最后决定每隔几个版本进行一次，具体的可以根据游戏的更新频率计算。

# 执行清理操作
这个方法就简单粗暴了，直接调用微信的rmdir清理获取的每个文件就可以了。

最后贴一下我最终的代码：
```
'use strict';

import { async } from "./utils/runtime";
let regeneratorRuntime = require('./utils/runtime');

const FileSize = 25; //清理资源的总量
const FilesName = ["new_download", "test_download", "img_download"]; //游戏方下载的压缩包解压后的文件夹

/**
 * 执行删除操作
 * filePath 需要删除的文件目录
 */
window.doDelete = function(filePath){
    let fs = wx.getFileSystemManager();
    return new Promise((resolve, reject) => {
        fs.rmdir({
            dirPath: filePath,
            recursive: true,
            success: () => {
                resolve();
            }
        });
    })
}

/**
 * 查找目标文件夹
 * filePath 文件目录
 */
window.findFile = function(filePath){
    let fs = wx.getFileSystemManager();
    let size = 0;
    let counts = 0;
    let num = 0;
    let now = new Date().getTime();
    let fileLength = 0;
    
    console.log("fs start time = ", now);
    console.log("readdir filePath = ", filePath);
    
    let deleteFile = async function(resolve){
        if ((size / 1024 / 1024) > FileSize && num < 1) {
            num++;
            let _now = new Date().getTime();
            let _time = (_now - now) / 1000;
            console.log("计算25M资源花费时间 time = ", _time.toFixed(2) + "秒", ", path = ", filePath);
            await window.doDelete(filePath);
            resolve(0);
        } else if(counts >= fileLength){
            let _now = new Date().getTime();
            let _time = (_now - now) / 1000;
            let _size = size / 1024 / 1024;
            console.log("计算花费时间 time = ", _time.toFixed(2) + "秒", ", 文件总量 size = ", _size.toFixed(2) + "M", ", path = ", filePath);
            resolve(size);
        }
    }

    return new Promise((resolve, reject) => {
        try {
            fs.readdir({
                dirPath: filePath,
                success: async (res) => {
                    console.log("readdir success fileList:", res);
                    let fileList = res.files;
                    fileLength = fileList.length;
                    for (let i = 0; i < fileLength; i++) {
                        let path = filePath + '/' + fileList[i];

                        //如果是文件夹进行递归查询
                        if(FilesName.indexOf(fileList[i]) >= 0){

                            let _size = await window.findFile(path);
                            size += _size;
                            counts++;
                            if(counts >= fileList.length){
                                deleteFile(resolve);
                            } else {
                                continue;
                            }
                        }

                        //获取文件的大小信息
                        fs.getFileInfo({
                            filePath: path,
                            success: (res) => {
                                size += res.size;
                                counts++;
                                deleteFile(resolve);
                            },
                            fail: async (res) => {
                                console.log("getFileInfo error :", res.errMsg);
                                resolve(0);
                            },
                        })
                    }
                },

                fail: (err) => {
                    console.log("readdir err:", err);
                    resolve(0);
                },

                complete: () => {
                    console.log("readdir complete");
                }
            })
        } catch (error) {
            console.log("readdir 获取该小程序下的 本地临时文件 或 本地缓存文件 信息失败 error：", error);
        }
    })
}
window.FileManager = async function () {
    
    let filePath = wx.env.USER_DATA_PATH + '/gamecaches';
    await window.findFile(filePath);
}
```
