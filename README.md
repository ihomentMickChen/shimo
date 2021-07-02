# 石墨文档批量导出脚本使用教程

## 1、安装油猴脚本

1.1 打开chrome浏览器 地址栏输入：https://chrome.google.com/webstore/search/tampermonkey?hl=zh-CN进入Chrome网上应用店

1.2 在Chrome网上应用店搜索tampermonkey

![image-20210702132812632](https://github.com/ihomentMickChen/shimo/blob/master/images/image-20210702132812632.png)

安装tampermonkey

![image-20210702133106692](https://github.com/ihomentMickChen/shimo/blob/master/images/image-20210702133106692.png)

安装成功后地址栏后面会出现一个插件图标，点击**管理扩展程序**

![image-20210702133444821](https://github.com/ihomentMickChen/shimo/blob/master/images/image-20210702133444821.png)

确保油猴脚本和开发者模式是打开状态

![image-20210702133747106](https://github.com/ihomentMickChen/shimo/blob/master/images/image-20210702133747106.png)

点击油猴插件弹出页面选择进入管理面板

![image-20210702133916580](https://github.com/ihomentMickChen/shimo/blob/master/images/image-20210702133916580.png)

![image-20210702134004477](https://github.com/ihomentMickChen/shimo/blob/master/images/image-20210702134004477.png)

## 2、添加脚本

点击加号添加脚本

![image-20210702134159160](https://github.com/ihomentMickChen/shimo/blob/master/images/image-20210702134159160.png)

**把下面的代码粘贴到编辑器界面**添加完后按键盘ctrl（command) + s保存

![image-20210702134546723](https://github.com/ihomentMickChen/shimo/blob/master/images/image-20210702134546723.png)

```typescript
"use strict";
// ==UserScript==
// @name 石墨文档批量导出
// @namespace https://walkedby.com/
// @supportURL https://github.com/gordonwalkedby/shimo-export
// @homepage https://github.com/gordonwalkedby/shimo-export
// @version 1.2
// @description 请先阅读 https://github.com/gordonwalkedby/shimo-export
// @author Gordon Walkedby
// @include https://shimo.im/*
// @connect shimo.im
// @connect smcdn.cn
// @connect shimonote.com
// @grant GM_xmlhttpRequest
// @grant GM_setValue
// @grant GM_notification
// @grant GM_registerMenuCommand
// @grant GM_getValue
// @grant GM_openInTab
// @noframes
// @require https://cdn.bootcdn.net/ajax/libs/jszip/3.5.0/jszip.min.js
// @require https://cdn.bootcdn.net/ajax/libs/FileSaver.js/2.0.5/FileSaver.min.js
// ==/UserScript==
/// <reference path = "header.ts" />
// 石墨文档里的文件类型
let ShimoItemType;
(function (ShimoItemType) {
    ShimoItemType[ShimoItemType["folder"] = 0] = "folder";
    ShimoItemType[ShimoItemType["docs"] = 1] = "docs";
    ShimoItemType[ShimoItemType["sheets"] = 2] = "sheets";
    ShimoItemType[ShimoItemType["mindmaps"] = 3] = "mindmaps";
    ShimoItemType[ShimoItemType["forms"] = 4] = "forms";
    ShimoItemType[ShimoItemType["boards"] = 5] = "boards";
    ShimoItemType[ShimoItemType["docx"] = 6] = "docx";
    ShimoItemType[ShimoItemType["presentation"] = 7] = "presentation";
})(ShimoItemType || (ShimoItemType = {}));
const ShimoItemTypeNames = new Map();
ShimoItemTypeNames.set(ShimoItemType.folder, "文件夹");
ShimoItemTypeNames.set(ShimoItemType.docs, "普通文档");
ShimoItemTypeNames.set(ShimoItemType.sheets, "表格");
ShimoItemTypeNames.set(ShimoItemType.mindmaps, "思维导图");
ShimoItemTypeNames.set(ShimoItemType.forms, "表单（不可导出）");
ShimoItemTypeNames.set(ShimoItemType.boards, "白板（不可导出）");
ShimoItemTypeNames.set(ShimoItemType.docx, "传统文档");
ShimoItemTypeNames.set(ShimoItemType.presentation, "幻灯片");
/// <reference path = "header.ts" />
/// <reference path = "interfaces.ts" />
// 把石墨文档的链接转换为信息，这只处理链接字符串，返回的name默认是空的
function URL2ShimoItem(url) {
    const reg = new RegExp("(folder|docs|sheets|mindmaps|forms|boards|docx|presentation)/([a-z|A-Z|0-9]+)", "gim");
    let rs = reg.exec(url);
    if (rs == null) {
        WriteLog("无法解析链接：" + url);
        return null;
    }
    let id = rs[2];
    let tp = ShimoItemType[rs[1]];
    let t = {
        type: tp,
        name: "",
        id: id,
        path: []
    };
    return t;
}
// 日志输出面板
const logPanel = document.createElement("div");
logPanel.style.width = "500px";
logPanel.style.height = "600px";
logPanel.style.position = "fixed";
logPanel.style.backgroundColor = "black";
logPanel.style.right = "100px";
logPanel.style.padding = "10px";
logPanel.style.color = "white";
logPanel.style.display = "none";
logPanel.style.zIndex = "99999";
logPanel.style.wordWrap = "anywhere";
logPanel.style.overflow = "auto";
document.body.insertBefore(logPanel, document.body.firstChild);
let logHistory = "";
function WriteLog(str) {
    let now = new Date;
    str = now.toLocaleTimeString() + " | " + str;
    console.log(str);
    logHistory += str + "\n";
    logPanel.style.display = "block";
    let span = document.createElement("span");
    span.innerText = str;
    span.style.display = "block";
    logPanel.appendChild(span);
    logPanel.scrollBy(0, logPanel.scrollHeight);
}
// 复制数组
function CloneArray(a) {
    return [...a];
}
// 组件路径的字符串
function BuildFolderPathName(t, addOwn) {
    let pathname = "";
    t.path.forEach(function (v) {
        if (pathname.length > 0) {
            pathname += "/";
        }
        pathname += v;
    });
    if (addOwn) {
        if (pathname.length > 0) {
            pathname += "/";
        }
        pathname += t.name;
    }
    return pathname;
}

// 异步里的暂停工作
function Sleep(ms) {
    let t = new Promise(function (resolve, reject) {
        setTimeout(function () {
            resolve();
        }, ms);
    });
    return t;
}
/// <reference path = "helpers.ts" />
const ExportFormats = new Map();
// 格式选择菜单
const fm = document.createElement("div");
const fmOptions = new Map();
(function () {
    let defaultFormats = new Map();
    defaultFormats.set("docs", "docx");
    defaultFormats.set("sheets", "xlsx");
    defaultFormats.set("mindmaps", "xmind");
    defaultFormats.set("docx", "docx");
    defaultFormats.set("presentation", "pptx");
    defaultFormats.forEach(function (v, k) {
        ExportFormats.set(k, GM_getValue("fm_" + k, v));
    });
    document.body.insertBefore(fm, document.body.firstChild);
    fm.style.width = "400px";
    fm.style.height = "600px";
    fm.style.position = "fixed";
    fm.style.backgroundColor = "black";
    fm.style.right = "100px";
    fm.style.padding = "10px";
    fm.style.color = "white";
    fm.style.display = "none";
    fm.style.zIndex = "99999";
    fm.style.wordWrap = "anywhere";
    fm.style.overflow = "auto";
    let h1 = document.createElement("h1");
    h1.innerText = "修改导出的格式";
    fm.appendChild(h1);
    let helps = document.createElement("p");
    helps.innerText = "下面所做的修改会自动保存。\n石墨不支持导出白板，我就没办法了。\n表单无法导出，表单的收集数据在另外的在线表格里。\n尽量不要选择导出图片，可能会因为文档过大导出失败。\n\n";
    fm.appendChild(helps);
    let addSelect = function (id, text, options) {
        let p = document.createElement("p");
        p.style.margin = "4px";
        fm.appendChild(p);
        let span = document.createElement("span");
        span.innerText = text;
        p.appendChild(span);
        let s = document.createElement("select");
        p.appendChild(s);
        fmOptions.set(id, s);
        options.forEach(function (v) {
            let o = document.createElement("option");
            o.innerText = v;
            o.value = v;
            s.appendChild(o);
            if (v == ExportFormats.get(id)) {
                o.selected = true;
            }
        });
        s.addEventListener("change", function () {
            let v = this.value;
            GM_setValue("fm_" + id, v);
            ExportFormats.set(id, v);
        });
    };
    addSelect("docs", "普通文档：", ["pdf", "docx", "jpg", "md"]);
    addSelect("docx", "文档传统版：", ["pdf", "docx", "wps"]);
    addSelect("sheets", "表格：", ["xlsx"]);
    addSelect("presentation", "幻灯片：", ["pdf", "pptx"]);
    addSelect("mindmaps", "思维导图：", ["jpg", "xmind"]);
})();
function OpenFormatMenu() {
    fm.style.display = "block";
}
function GetFormatByType(v) {
    let kind = ShimoItemType[v];
    let format = ExportFormats.get(kind);
    if (format == null || format.length < 1) {
        return null;
    }
    return format;
}
/// <reference path = "header.ts" />
/// <reference path = "helpers.ts" />
const fails = [];
const AllFolders = new Map();
// 扫描全部的文件夹
async function ScanFolders() {
    while (WaitingCollectFolderIDs.length > 0) {
        let fd = WaitingCollectFolderIDs[0];
        let name = BuildFolderPathName(fd, true);
        WriteLog("开始扫描文件夹：" + name);
        let ok = false;
        for (let retry = 0; retry < 4; retry++) {
            if (retry > 0) {
                WriteLog("重试：" + retry);
            }
            await Sleep(1000);
            try {
                await ScanFolder(fd);
                ok = true;
                if (retry > 0) {
                    WriteLog("重试成功：" + name);
                }
                break;
            }
            catch (error) {
            }
        }
        if (!ok) {
            fails.push(name);
            WriteLog("放弃文件夹扫描：" + name);
        }
        WaitingCollectFolderIDs.splice(0, 1);
    }
}
// 扫描单个文件夹
async function ScanFolder(folder) {
    let id = folder.id;
    let pathname = BuildFolderPathName(folder, true);
    FolderIDs.set(pathname, id);
    let p = new Promise(function (resolve, reject) {
        GM_xmlhttpRequest({
            url: "https://shimo.im/lizard-api/files?collaboratorCount=true&folder=" + id,
            headers: {
                accept: "application/vnd.shimo.v2+json",
                Referer: "https://shimo.im/folder/" + id
            },
            timeout: 180000,
            method: "GET",
            onload: function (rep) {
                if (rep.status == 200) {
                    try {
                        let array = JSON.parse(rep.responseText);
                        let path = CloneArray(folder.path);
                        path.push(folder.name);
                        array.forEach(function (v) {
                            let t = URL2ShimoItem(v.url);
                            if (t != null) {
                                t.name = v.name;
                                t.path = CloneArray(path);
                                if (t.type == ShimoItemType.folder) {
                                    WaitingCollectFolderIDs.push(t);
                                    let path2 = BuildFolderPathName(t, true);
                                    AllFolders.set(path2, t);
                                    console.log(" fd", t);
                                }
                                else {
                                    GotItems.push(t);
                                    console.log(" item", t);
                                }
                            }
                        });
                        resolve();
                        return;
                    }
                    catch (error) {
                        WriteLog("API返回JSON解析失败：" + error + "，id:" + id);
                        reject();
                    }
                }
                else {
                    WriteLog("检查子文件夹出错！返回" + rep.status.toFixed() + "，id:" + id);
                    reject();
                }
            },
            ontimeout: function () {
                WriteLog("检查子文件夹超时！ id:" + id);
                reject();
            },
            onerror: function (rep) {
                WriteLog("检查子文件夹出错！信息：" + rep.error + "，id:" + id);
                reject();
            }
        });
    });
    return p;
}
// 导出所有已知的文件
async function ExportItems() {
    WriteLog("扫描结束，统计信息：");
    let len = GotItems.length;
    WriteLog("总文件数量：" + len.toFixed());
    const maxDownloadOnce = 5;
    if (len > maxDownloadOnce) {
        let guessTime = len * 15 + 60;
        let dd = new Date;
        dd.setSeconds(dd.getSeconds() + guessTime);
        WriteLog("石墨文档有速度限制，1分钟最多只能导出 " + maxDownloadOnce.toFixed() + " 个文件，相当于 " + (60 / maxDownloadOnce).toFixed() + " 秒一个，所以说预计完成导出时间是：" + dd.toLocaleString());
    }
    let map = new Map();
    let goodCount = 0;
    GotItems.forEach(function (v) {
        let key = v.type;
        if (!map.has(key)) {
            map.set(key, 1);
        }
        else {
            let olds = map.get(key);
            map.set(key, olds + 1);
        }
        if (GetFormatByType(key) != null) {
            goodCount += 1;
        }
    });
    WriteLog("实际可导出文件数量：" + goodCount.toFixed());
    map.forEach(function (v, k) {
        let typeName = ShimoItemTypeNames.get(k) || "未知格式：";
        WriteLog(typeName + "：" + v);
    });
    WriteLog("下面开始下载要导出的文件");
    let zip = new JSZip;
    for (let index = 0; index < len; index++) {
        let v = GotItems[index];
        let pathname = BuildFolderPathName(v, true);
        let format = GetFormatByType(v.type);
        if (format == null) {
            console.log("无法导出，跳过：", pathname);
            continue;
        }
        WriteLog("进度：" + (index / len * 100).toFixed(1) + "% " + pathname + "." + format);
        let ok = false;
        let retryCount = 6;
        for (let retry = 0; retry < retryCount; retry++) {
            if (retry > 0) {
                WriteLog("重试：" + retry.toFixed());
            }
            await Sleep(3000);
            try {
                let blob = await ExportItem(v, format);
                let zp = zip;
                if (v.path.length > 0) {
                    v.path.forEach(function (fd) {
                        fd = fd.trim();
                        if (fd.length < 1) {
                            fd = "unNamed_" + Math.random().toString();
                        }
                        let zp2 = zp.folder(fd);
                        if (zp2 == null) {
                            WriteLog("异常！zip folder 出错！ " + fd);
                            throw fd;
                        }
                        zp = zp2;
                    });
                }
                zp.file(v.name + "." + format, blob);
                ok = true;
                if (retry > 0) {
                    WriteLog("重试成功：" + pathname);
                }
                break;
            }
            catch (error) {
            }
        }
        if (!ok) {
            WriteLog("失败，跳过：" + pathname);
            fails.push(pathname);
        }
        else {
            let path2 = BuildFolderPathName(v, false);
            let toRemove = null;
            for (let g = 0; g < AllFolders.size; g++) {
                let fd = AllFolders.keys().next().value;
                if (fd == path2) {
                    toRemove = fd;
                    break;
                }
            }
            if (toRemove != null) {
                AllFolders.delete(toRemove);
            }
        }
    }
    AllFolders.forEach(function (v) {
        let zp = zip;
        v.path.push(v.name);
        v.path.forEach(function (fd) {
            let zp2 = zp.folder(fd);
            if (zp2 == null) {
                WriteLog("异常！zip folder 出错！ " + fd);
                throw fd;
            }
            zp = zp2;
        });
        zp.file("空白文件夹.txt", "Nothing here");
    });
    WriteLog("导出工作结束，失败数：" + fails.length);
    if (fails.length > 0) {
        let str = "";
        fails.forEach(function (v) {
            str += v + "\n";
        });
        WriteLog(str);
    }
    let dt = new Date;
    let pass = (dt.getTime() - StartTime) / 1000 / 60;
    WriteLog("总用时：" + pass.toFixed(1) + " 分钟。");
    WriteLog("正在生成 zip 文件");
    zip.file("导出日志.txt", logHistory);
    zip.generateAsync({ type: "blob" }).then(function (content) {
        WriteLog("zip 文件生成成功");
        downloadZip(content,dt.getTime().toFixed() + ".zip");
        let but = document.createElement("button");
        but.innerText = "点我下载";
        but.style.padding = "5px";
        but.style.fontSize = "2em";
        but.onclick = function () {
            downloadZip(content,dt.getTime().toFixed() + ".zip");
        };
        logPanel.appendChild(but);
        GM_notification({ text: "您的石墨文档导出已经完成！可以下载了！", title: "下载完成！", timeout: 0 });
        WriteLog("可以下载了！");
    },function (reason) {
        WriteLog("生成 zip 文件失败"+reason);
    })
}

function downloadZip(content, fileName) {
    WriteLog("正在下载zip文件,请查看浏览器");
    saveAs(content,fileName);
}
// 下载一个石墨文件，返回 base64 字符串
function ExportItem(t, format) {
    let url = "https://xxport.shimo.im/files/" + t.id + "/export?type=" + format + "&file=" + t.id + "&returnJson=1&name=" + t.name;
    console.log(url);
    let refr = BuildFolderPathName(t, false);
    if (FolderIDs.has(refr)) {
        refr = FolderIDs.get(refr);
        refr = "https://shimo.im/folder/" + refr;
    }
    else {
        refr = "https://shimo.im/desktop";
    }
    let p = new Promise(function (resolve, reject) {
        let downloads = function (u2) {
            console.log("去下载：", u2);
            GM_xmlhttpRequest({
                url: u2,
                method: "GET",
                timeout: 180000,
                responseType: "blob",
                headers: {
                    Accept: "*/*",
                    Referer: refr
                },
                onload: function (rep) {
                    if (rep.status == 200) {
                        let obj = rep.response;
                        resolve(obj);
                    }
                    else {
                        WriteLog("下载异常：" + rep.status.toFixed() + " " + rep.responseText);
                        reject();
                    }
                },
                onerror: function (rep) {
                    WriteLog("下载异常：" + rep.error);
                    reject();
                }, ontimeout: function () {
                    WriteLog("下载超时！");
                    reject();
                }
            });
        };
        GM_xmlhttpRequest({
            url: url,
            headers: {
                Accept: "*/*",
                Referer: refr
            },
            method: "GET",
            timeout: 180000,
            onload: async function (rep) {
                if (rep.status == 200) {
                    try {
                        let info = JSON.parse(rep.responseText);
                        let u = info.redirectUrl;
                        if (u.length > 15) {
                            downloads(u);
                        }
                    }
                    catch (error) {
                        WriteLog("API返回JSON解析失败：" + error);
                        reject();
                    }
                }
                else {
                    let str = rep.responseText;
                    WriteLog("返回异常：" + rep.status.toFixed() + " " + str);
                    //{"requestId":"e5a1795135de65730960400ca6d9f160","error":"Rate limit exceeded, retry in 22 seconds","errorCode":0}
                    const reg = new RegExp("retry in ([0-9]+) seconds", "gim");
                    let rs = reg.exec(str);
                    if (rs != null) {
                        let num = parseFloat(rs[1]) + 1;
                        let now = new Date;
                        now.setSeconds(now.getSeconds() + num);
                        WriteLog("石墨限制，等待到这之后再继续：" + now.toLocaleTimeString());
                        await Sleep(num * 1000);

                    }
                    reject();
                }
            },
            onerror: function (rep) {
                WriteLog("返回出错：" + rep.error);
                reject();
            },
            ontimeout: function () {
                WriteLog("返回超时！");
                reject();
            }
        });
    });
    return p;
}
/// <reference path = "header.ts" />
/// <reference path = "scan&export.ts" />
//  文件夹的ID表
const FolderIDs = new Map();
// 已经收集到的文件列表
const GotItems = [];
// 等待扫描的文件夹列表
const WaitingCollectFolderIDs = [];
let StartTime = 0;
GM_registerMenuCommand("设置导出的格式", function () {
    if (StartTime > 123456) {
        alert("我已经启动过了！要调整导出格式，请刷新后重试。");
        return;
    }
    OpenFormatMenu();
});
GM_registerMenuCommand("点我开始导出", function () {
    if (StartTime > 123456) {
        alert("我已经启动过了！请刷新本页面后再试！");
        return;
    }
    let url = location.href;
    if (url.includes("desktop")) {
        WaitingCollectFolderIDs.push({
            name: "起点", id: "", type: ShimoItemType.folder, path: []
        });
    }
    else {
        let fd = URL2ShimoItem(url);
        if (fd == null || fd.type != ShimoItemType.folder) {
            alert("对不起，请在桌面页面或者在文件夹页面使用我。");
            return;
        }
        fd.name = "起点";
        WaitingCollectFolderIDs.push(fd);
    }
    fm.remove();
    WriteLog("这里是戈登走過去编写的石墨文档批量导出工具。希望可以帮助到你。祝愿你每天开心。少加班，多休息。\n QWQ");
    let now = new Date;
    StartTime = now.getTime();
    WriteLog("今天是：" + now.toLocaleDateString());
    WriteLog("开始导出工作");
    WriteLog("扫描子文件夹和子文件中...");
    ScanFolders().then(function () {
        ExportItems();
    });
});
GM_registerMenuCommand("查看源码、反馈BUG", function () {
    GM_openInTab("https://github.com/gordonwalkedby/shimo-export", { active: true, insert: true });
});

```

保存成功后页面如下所示：

![image-20210702134637905](https://github.com/ihomentMickChen/shimo/blob/master/images/image-20210702134637905.png)

## 3、使用脚本下载石墨文档

3.1 登录石墨文档，进入你要批量导出的**文件夹**或**我的桌面**

![image-20210702135106241](https://github.com/ihomentMickChen/shimo/blob/master/images/image-20210702135106241.png)

3.2 点击开始导出脚本开始进行批量导出工作：

![image-20210702135401965](https://github.com/ihomentMickChen/shimo/blob/master/images/image-20210702135401965.png)

**脚本工作过程中不要关闭此页面或刷新页面，当导出结束后，会自动将导出的文件压缩成.zip格式压缩包进行下载，您也可以点击【点我下载】按钮进行手动下载**

![image-20210702135633995](https://github.com/ihomentMickChen/shimo/blob/master/images/image-20210702135633995.png)