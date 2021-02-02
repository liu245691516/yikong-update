<template>
    <div id="app">

    </div>
</template>

<script>
    export default {}
    const path = require('path'); // 路径模块
    const fs = require('fs'); // 文件操作模块
    const os = require('os'); // 为了判断操作系统
    const request = require('request'); // 为了请求远程文件
    const md5 = require('md5-node');
    const __dirname = path.resolve();
    const win = nw.Window.get();
    const child_process = require('child_process');
    const Bagpipe = require('bagpipe');
    let bagpipe = new Bagpipe(10, {timeout: 5000});


    // 远端热更文件存放的url
    // @@以后把ip改成域名
    //上传文件时 fileType 1 linux 2 windows 3 json文件
    const servertUrl = 'http://123.206.83.233/largeScreen/';
    let remoteUrlPrefix = servertUrl + 'Windows/yikong-win/';
    let platformFileName = 'yikong-win';

    if(os.type() === 'Linux'){
        remoteUrlPrefix = servertUrl + 'Linux/yikong-linux/';
        platformFileName = 'yikong-linux';
    }
    let localFile = path.join(__dirname, '../filelist.json');    //本地json文件
    let remoteFileList = undefined; // 远程文件列表
    let localFileList = undefined; // 本地文件列表
    let networkState = true;

    let progressTimer; //设置一个定时器，用来模拟下载的进去条
    let inner = document.getElementById("xfj-progress");
    let downloading = false;
    let maxUpdateNum = 3;   //更新失败后 最大更新次数
    let updateNum = 0;      //已尝试更新次数
    let downloadTimer = undefined;

    // 检查当前网络状态
    if (navigator.onLine) {
        win.show(); 
        win.setAlwaysOnTop(false);
        readLocalFile();
        downloadUpdateJsonFile();
    } else {
        //没有网 直接启动播控
        launchScreenClient();
    }

    function downloadUpdateJsonFile(){
        let num = 60;
        downloadTimer = window.setInterval(() => {
            num -= 1;
            if(num <= 0){
                window.clearInterval(downloadTimer);
                launchScreenClient();
            }
        }, 1000)
    }

    //断开网络监听
    window.addEventListener('offline', function(){
        networkState = false;
        launchScreenClient();
    });

    //恢复网络监听
    window.addEventListener('online',  function(){
        networkState = true;
    });


    // function delayPromise(ms) {
    //     return new Promise(function(resolve) {
    //         setTimeout(resolve, ms);
    //     })
    // }

    // function timeoutPromise(promise, ms) {
    //     let timeout = delayPromise(ms).then(function() {
    //         throw new Error("time out after: " + ms + "ms");
    //     })

    //     return Promise.race([promise, timeout]);
    // }


    // 读取本地的文件列表r
    function readLocalFile(){
        fs.readFile(localFile, 'utf-8', function(err, data) {
            console.log(err)
            if (err) {
                console.log("读取本地文件列表失败，做完整更新。");
                localFileList = [];
                checkFileList();
            } else {
                localFileList = data ? JSON.parse(data) : [];
                checkFileList();
            }
        });
    }

    // 检查和创建目录，注意：如果传入的是目录最后必须以文件分隔符结束！！！例如：/home/ubuntu/client/
    function checkAndCreateDirectory(filename, myCallback) {
        let filePath = filename;
        let seperator = '';
        if (os.type() == 'Windows_NT') {
            seperator = '\\';
        } else {
            seperator = '/';
        }

        let fromIndex = 0;
        for (; ;) {
            fromIndex = filePath.indexOf(seperator, fromIndex) + 1;
            if (fromIndex == -1){
                break;
            }
            let checkPath = filePath.substring(0, fromIndex);
            if (checkPath.length == 0) {
                break;
            }
            if (!fs.existsSync(checkPath)) {
                fs.mkdirSync(checkPath)
            }
        }
        if(myCallback) myCallback();
    }


    // 下载一个远端文件到本地
    function downloadRemoteFile(filename) {
        if(!downloading && filename != 'filelist.json'){
            updatedStateText('更新中...');
            progress();
        }
        return new Promise((resolve, reject) => {
            let fileId = filename;
            let url = remoteUrlPrefix + filename;
            console.log(remoteUrlPrefix, filename)
            
            // 检测并创建必要的目录
            filename = path.join(__dirname, `../download/${platformFileName}/${filename}`);
            checkAndCreateDirectory(filename);
            
            console.log("开始下载： " + url);
            // 开始下载

            bagpipe.push(downloadFile, url, filename, fileId, resolve, function (err, res){
                // console.log(err, res)
            });
            
            // request(url, (error, res, body) => {
            //     if(!error && res.statusCode == 200){
            //         let stream = fs.createWriteStream(filename);
            //         request(url).pipe(stream).on("close", (err) =>{
            //             resolve({ err, filename: fileId });
            //         }).on("error", (err) => {
            //             console.log("文件：" + filename + " 下载失败！");
            //             console.log(err);
            //         });

            //     }else {
            //         console.log(error, url);
            //     }
            // })
        })
    }

    var downloadFile = function(src, dest, fileId, resolve){
        request(src).pipe(fs.createWriteStream(dest)).on('close', (err) => {
            resolve({ err, filename: fileId });
        })
    }

    // 下载远程的文件列表
    downloadRemoteFile('filelist.json').then(res => {
        window.clearInterval(downloadTimer);
        if (res.err) {
            console.log('远程文件列表下载失败，直接启动屏幕端。');
            // @@ 还需要停止本地文件列表的操作么？
            launchScreenClient();
        } else {
            console.log('远程文件列表下载成功');

            // 读取已下载文件并转成js对象
            fs.readFile(path.join(__dirname, `../download/${platformFileName}/${res.filename}`), 'utf-8', function(err, data) {
                if (err) {
                    console.log(err, "读取远程文件列表失败，直接启动屏幕端。");
                    launchScreenClient();
                } else {
                    remoteFileList = JSON.parse(data);
                    checkFileList();
                }
            });
        }
    });

    // 下载一个远端的实际文件完成的回调函数
    function remoteFileDownloadedCallback(err, filename) {
        if (err) {
            // @@ 先算失败，停止后续更新，并直接启动屏幕端
            // 以后可以改进成尝试重试若干次
            console.log('远程文件: ' + filename + ' 下载失败！这次更新算失败了');
        } else {
            console.log('远程文件: ' + filename + ' 下载成功！');
            for (let remoteFile of remoteFileList) {
                if (remoteFile.name == filename) {
                    remoteFile.processed = true;
                    break;
                }
            }
            // 检查是否所有文件都下载完成了
            checkUpdateCompleted();
        }
    }

    // 判断本地有没有下载过这个文件
    function checkAlreadyDownloaded(fileName, fileMd5) {
        let fullPath = path.join(__dirname, '/download/' + fileName);

        if (fs.existsSync(fullPath)) {
            let fileData = fs.readFileSync(fullPath);
            let localMd5 = md5(fileData);
            if (localMd5 == fileMd5) {
                console.log(fileName + ": 已经下载过了，直接跳过");

                for (let remoteFile of remoteFileList) {
                    if (remoteFile.name == fileName) {
                        remoteFile.processed = true;
                        break;
                    }
                }

                // 检查是否所有文件都下载完成了
                checkUpdateCompleted();

                return true;
            } else {
                return false;
            }
        } else {
            return false;
        }
    }

    // 逐一比对文件列表里的文件，并更新
    function checkFileList() {
        // 如果有一端的文件列表没有搞完，就不处理
        if (localFileList == undefined || remoteFileList == undefined) {
            return;
        }

        console.log('两端的文件列表都处理完毕，开始处理文件列表。');

        //是否有更改的文件
        let hasChangeFile = false;
        let files = [];
        
        // 先以远程文件列表为准，遍历一遍远程文件列表
        for (let remoteFile of remoteFileList) {
            let processed = false;
            // 遍历本地的文件列表
            for (let localFile of localFileList) {
                // 如果找到了同一个文件
                if (remoteFile.name == localFile.name) {
                    // 如果文件的md5码不一致
                    if (remoteFile.md5 != localFile.md5) {
                        if (!checkAlreadyDownloaded(remoteFile.name, remoteFile.md5)) {
                            // 那么这个文件更新了，需要从远端下载
                            hasChangeFile = true;
                            downloadRemoteFile(remoteFile.name).then(res => {
                                remoteFileDownloadedCallback(res.err, res.filename);
                            });
                            files.push(remoteFile.name);
                        }
                    } else { // md5码一致，说明本地文件最新，不用下载
                        // 那么标记一下这个文件已经处理完毕
                        remoteFile.processed = true;
                    }

                    // 标记处理完毕，并中断这次本地列表的循环
                    processed = true;
                    break;
                }
            }

            // 如果本地列表中没有这个文件
            if (processed == false) {
                if (!checkAlreadyDownloaded(remoteFile.name, remoteFile.md5)) {
                    // 那么是新增文件，直接从远端下载该文件
                    hasChangeFile = true;
                    files.push(remoteFile.name);
                    downloadRemoteFile(remoteFile.name).then(res => {
                        remoteFileDownloadedCallback(res.err, res.filename);
                    });
                }
            }
        }

        //如果所有文件都是最新，没有更新发生的情况下，会走到这里
        if(!hasChangeFile){
            launchScreenClient();
        }

        // @@ 没有检查本地多余的废弃文件，不影响实际运行。以后有时间可以完善这里。
    }

    // 检查是否所有的文件都已经下载完毕了
    function checkUpdateCompleted() {
        for (let remoteFile of remoteFileList) {
            // 如果还有文件没有处理完毕
            if (typeof(remoteFile.processed) == undefined || remoteFile.processed != true) {
                return;
            }
        }

        //下载完成  更新本地json文件
        fs.writeFileSync(localFile, JSON.stringify(remoteFileList), "utf8", function (err) {
            if (err) {
                console.log("写文件出错了，错误是：" + err);
            } else {
                console.log("ok");
            }
        });
        
        updatedStateText('更新完成!');
        inner.innerHTML = '100%';
        inner.style.width = '100%';
        clearInterval(progressTimer);

        // 全部文件都更新完毕了，这次更新完成
        // 把已经下载完成的所有文件都复制覆盖本地文件
        copyFiles(path.join(__dirname, `../download/${platformFileName}`), () => {
            rmdiFile(path.join(__dirname, '../download'), () => {
                console.log('删除临时文件成功')
                launchScreenClient();
            });
        });
    }

    // 更新完毕后，启动屏幕客户端
    function launchScreenClient() {
        console.log("启动屏幕客户端咯！");
        const yikong = path.join(__dirname, platformFileName == 'yikong-win' ? `../${platformFileName}/yikong.exe` : `../${platformFileName}/yikong`) 
        child_process.exec(yikong);
        setTimeout(() => {
            process.exit(0);
        }, 2000);
    }


    function copyFiles(_path, callback) {
        try {
            fs.readdir(_path, (err, files) => {
                function rename(index) {
                    if (index == files.length) return callback();
                    let newPath = path.join(_path, files[index]);
                    fs.stat(newPath, (err, stat) => {
                        if(err){
                            callback();
                        }else if (stat.isDirectory()) {
                            copyFiles(newPath, () => rename(index + 1));
                        } else {
                            let fileUrl = newPath;
                            checkAndCreateDirectory(fileUrl.replace('download\\', ''), () => {
                                try{
                                    fs.rename(
                                        fileUrl,
                                        fileUrl.replace('download\\', ''), 
                                        (err) => {
                                            rename(index + 1)
                                        }
                                    );
                                }catch(err){
                                    console.log(err)
                                }
                            })
                    
                        }
                    });
                }
                rename(0);
            });

            
        } catch (error) {
            console.log(error)
        }
    }


    //删除临时文件夹 及 文件
    function rmdiFile(dir, callback) {
        fs.readdir(dir, (err, files) => {
            function next(index) {
                if (index == files.length) return fs.rmdir(dir, callback);
                let newPath = path.join(dir, files[index]);

                fs.stat(newPath, (err, stat) => {
                    if(err){
                        callback();
                    }else if (stat.isDirectory()) {
                        rmdiFile(newPath, () => next(index + 1));
                    } else {
                        fs.unlink(newPath, (err) => {
                            next(index + 1)
                        });
                    }
                });
            }
            next(0);
        });
    }

    function progress(){
        downloading = true;
        document.getElementById("xfj-update").style.display = 'block';
        document.getElementById("app").innerHTML = '';
        var width = 1;
        var speed = 2;
        progressTimer = setInterval(addwidth, 300);
        function addwidth(){
            if(width >= 100){
                width = 100;
                updateNum += 1;
                if(updateNum <= maxUpdateNum) {
                    checkFileList();
                }else{
                    launchScreenClient();
                }
                clearInterval(progressTimer);
            }else{
                if(width >= 99) {
                    speed = 0.001;
                }else if (width >= 95) {
                    speed = 0.01;
                } 
                width += Math.random() * speed;
                inner.innerHTML = width.toFixed(2) + '%';
                inner.style.width = width + '%';
            }
        }
    }

    function updatedStateText(msg){
        document.getElementById("xfj-update").getElementsByTagName('h3')[0].innerHTML = msg;
    }


</script>