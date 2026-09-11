# 🔥超星学习通小助手 v4.4.1 结构分析（混淆/打包版）

> 分析对象：`D:\work\jiaoben copy to refresh.txt`
> 说明：任务描述称该文件 3238 行，**实测为 3300 行 / 195,286 字节**（`Get-Content` 与 read 工具一致）。本文件行号均以实测 3300 行为准。
> 全文唯二被截断的长行是 **L126**（`icon` 内联 SVG，3657 字符，已用 pwsh 切片确认其尾部仍为 SVG `</svg>`，无逻辑代码）。其余行均未被截断。
> 标注约定：所有结论后附行号；无直接证据的写法一律标「推断」或「未能确认」。

---

## 概览

| 项 | 值 | 行号 |
| --- | --- | --- |
| 脚本名 | 🔥超星学习通小助手-章节、作业、考试覆盖98%… | L2 |
| 版本 | `@version 4.4.1` | L4 |
| 作者 | `@author 爱吃蛋炒饭` | L6 |
| 命名空间 | `http://tampermonkey.net/` | L3 |
| 更新源 | `update.greasyfork.org/scripts/435357/...` | L53-L54 |
| 运行时机 | `@run-at document-end` | L7 |
| 匹配站点 | `*.chaoxing.com` / `*.edu.cn` / `*.nbdlib.cn` / `*.hnsyu.net` / `*.gdhkmooc.com` | L8-L12 |
| 声明特性 | `@antifeature payment 存在第三方付费接口` | L52 |
| 依赖库 | jQuery 3.6.0、layx 2.5.4、blueimp-md5 2.19.0（`@require`）；layx.css / layui.css（`@resource`） | L45-L50 |
| 授权 | `unsafeWindow`、`GM_info`、`GM_getResourceText`、`GM_addStyle`、`GM_setValue`、`GM_getValue`、`GM_xmlhttpRequest` | L38-L44 |
| 体积 | 3300 行 / 195 KB | — |

**一句话架构**：webpack 空壳引导（L57-L58）里挂了一个大 IIFE（L131-L3296），IIFE 内定义 4 个对象——`utils`（L143-L268）、`api`（L269-L447，超星站内接口）、`ServerApi`（L448-L587，第三方题库接口）、`page`（L588-L3281，UI+任务调度+答题 DOM 操作），最后 `page.init()`（L3290）用 `switch(true)` 按 `location.href` 分发到各页面逻辑。

**代码量分布（按行区间）**：

| 区段 | 行号 | 约占比 | 内容 |
| --- | --- | --- | --- |
| 元数据头 | L1-L55 | 1.7% | UserScript 头 |
| 全局常量与猴补丁 | L57-L142 | 2.6% | webpack 壳、`defaultConfig`、`otherApi`、`reqUrl`、`ttflist`、缓存读取、postMessage 补丁 |
| `utils` | L143-L268 | 3.8% | 工具函数 |
| `api` | L269-L447 | 5.4% | 超星站内 HTTP 封装 |
| `ServerApi` | L448-L587 | 4.2% | 第三方题库 HTTP 封装 |
| `page` | L588-L3281 | 81.6% | 路由、UI、任务调度、答题 |
| └ 其中 `decode` + 内联 Typr 字体解析 | L1497-L2485 | 30.0% | 字体反爬解码（第三方库 Typr 被内联，作者署名 wyn，L1494-L1495） |
| └ 答题 DOM 操作 `setAnswer`/`setWorkAnswer`/`setChapterAnswer*` | L2557-L2810 | 7.7% | — |
| └ 成绩/答案回收 `getScore`~`getScore4` | L2976-L3280 | 9.2% | — |
| 收尾 | L3282-L3300 | 0.6% | localStorage 清理、`page.init()` |

> 关键观察：**约 30% 的代码是内联的 Typr 字体解析库**（L1498-L2448），与业务无关；真正的业务逻辑集中在 L588-L1496 与 L2486-L3280。

---

## 运行入口与路由分发

入口：`page.init()`（L611-L882）。`GM_addStyle` 注入两个 CSS 资源（L612-L613），重置计数器（L614-L616），然后 `switch (true)`（L618）按 `location.href` 逐条匹配。

| # | URL 特征 | 触发条件 | 调用函数 | 行号 |
| --- | --- | --- | --- | --- |
| 1 | `/exam/test/reVersionTestStartNew` | 且 `newMooc=true` | `page.layx("ks")` → 注入「配置/自动切换」按钮 → `page.getQuestion("3")` → `page.startAsk()` | L619-L651 |
| 1b | 同上 | 且**不**含 `newMooc=true` | 改写 URL 追加/替换 `newMooc=true` → `location.href = url`（自跳转） | L652-L660 |
| 2 | `/mycourse/stu?` | 恒真 | `page.layx()`→`page.threadWatch()`→注入「配置」按钮→`ServerApi.checkKey(defaultConfig.token)`→`sleep(interval)`→`page.mainTask()` | L662-L680 |
| 3 | `/workHandle/handle` | 恒真 | `window.parent.postMessage(notify("error","作业已被删除-跳过"))` | L681-L683 |
| 4 | `/work/doHomeWorkNew` | body 含「此作业已被老师」 | postMessage 跳过 | L686-L689 |
| 4b | 同上 | body 含「您长时间没有操作」 | postMessage "遇到一个bug,后期修复" | L690-L693 |
| 4c | 同上 | URL 含 `reEdit=2` | `page.getScore()` → postMessage "作业待批阅" | L694-L699 |
| 4d | 同上 | URL 含 `mooc2=1` | 无逻辑（仅两行注释） | L700-L703 |
| 4e | 同上 | URL 含 `oldWorkId` | `page.layx("zj")`→配置按钮→`page.decode()`（失败则终止）→`defaultConfig.autoAnswer && page.startChapter()` | L704-L738 |
| 4f | 同上 | 其余 | `layx.msg('不支持旧版作业')` | L739-L741 |
| 5 | `/work/selectWorkQuestionYiPiYue` | `mooc2=1` → `getScoreNew()`，否则 `getScore()` | 然后 postMessage `"作业已完成"` | L743-L753 |
| 6 | `/mooc2/work/dowork` | 恒真 | `page.layx('zy')`→配置按钮→`autoAnswer && page.startWork()` | L754-L780 |
| 7 | `/visit/courses` | 恒真 | **空分支**（什么都不做） | L781-L782 |
| 8 | `/antispiderShowVerify.ac` 或 `/html/processVerify.ac` | 恒真 | `api.getVerifyCode("/processVerifyPng.ac?t=...")` → `ServerApi.getVerifyCode(imgData)`（服务端 OCR）→ 跳转 `/html/processVerify.ac?...&ucode=` | L783-L789 |
| 9 | `/mycourse/studentstudy` | 且 `defaultConfig.tutorial` | 注入右下角「快捷跳转」悬浮卡片（点卡片→`window.open(visit/stucoursemiddle...)`） | L790-L854 |
| 10 | `/ztnodedetailcontroller/visitnodedetail` | 恒真 | `layx.msg` 提示 + `$('html,body').animate` 自动滚到底并按 `readSpeed` 控速，结束后点 `.nodeItem.r i` | L855-L866 |
| 11 | `/mooc2/work/view` | 恒真 | `page.getScore3()` | L867-L869 |
| 12 | `/exam-ans/exam/test/reVersionPaperMarkContentNew` | 恒真 | `page.getScore4()` | L870-L872 |
| 13 | `default` | pathname 匹配 `/^\/(mooc2-ans\/course\|mooc2-ans\/zt\|mooc-ans\/course\|mooc-ans\/zt\|course\|zt)\/\d+\.html$/` | `setTimeout(() => _self.sendLogs && $('.course_section:eq(0) .chapterText').click(), interval)` | L873-L879 |

**注意**：
- 分支 1b 会无条件把考试 URL 改写并跳转（L652-L660），是考试流程的第一步。
- 分支 4d（`mooc2=1`）只有注释、没有可执行语句（L700-L703），是**死角**。
- `switch(true)` 是顺序匹配，靠 `break` 终止；分支 4 内部用 `break` 提前跳出（L688、L692、L698），逻辑正确但易漏 `break`。

---

## 配置系统

### `defaultConfig` 全字段（定义于 L59-L89）

| 字段 | 默认值 | 含义 | 定义行 | 配置面板行 | 代码消费点 |
| --- | --- | --- | --- | --- | --- |
| `ua` | `Dalvik/2.1.0 (… ChaoXingStudy_3_5.1.4_android_phone_614_74 …)` | 伪造的安卓客户端 UA | L60 | 无 | `api.defaultRequest` 默认参数 L299；L407/L412/L419/L424/L434 显式传 `navigator.userAgent` |
| `interval` | `3000` | 任务/请求间隔（毫秒） | L61 | L912-L917 | 全局 `sleep` 调用点 |
| `autoVideo` | `true` | 是否刷视频 | L62 | L942-L947 | L1219 |
| `autoRead` | `true` | 是否刷文档 | L63 | L960-L965 | L1236 |
| `autoAnswer` | `true` | 是否自动答题 | L64 | L966-L971 | L735、L775、L1246 |
| `videoSpeed` | `1` | 视频倍速（表单可填，逻辑上界 16） | L65 | L948-L953 | L1321、L1323 |
| `reviewMode` | `false` | 复习模式：已完成章节也重刷 | L66 | L984-L989 | L1144、L1153、L1157、L1318 |
| `matchRate` | `0.8` | 模糊匹配阈值 | L67 | L930-L935 | L252、L1091-L1093 |
| `autoSubmitRate` | `0.8` | 自动提交所需正确率 | L68 | L918-L923 | L1095-L1097、L2960 |
| `autoSubmit` | `true` | 是否自动提交 | L69 | L972-L977 | L2954-L2974 |
| `autoSwitch` | `false` | 考试页是否自动切下一题 | L70 | **无**（仅考试页按钮 L637-L642 切换 + L641 落盘） | L637-L640、L2829 |
| `tutorial` | `true` | 是否显示引导卡片 | L71 | L906-L911 | L853 |
| `randomAnswer` | `false` | 「无答案随机选择」 | L72 | L924-L929 | **全文件无消费点**（随机仍然无条件执行：L2684-L2688、L2748-L2752） |
| `threadWatch` | `true` | 进程守护（卡死自动刷新） | L73 | L990-L995 | L590 |
| `freeFirst` | `true` | 免费题库优先 | L74 | L996-L1001 | L1367、L2816、L2856、L2917 |
| `readSpeed` | `10` | 阅读滚动速度 | L75 | L954-L959 | L860 |
| `notice` | `本脚本仅供学习研究…` | 公告兜底文案 | L76 | 无 | L539 |
| `debugger` | `false` | 调试日志开关，**控制整个 `log()` 是否输出** | L77 | 无 | L130 |
| `types` | 8 种题型名→ID 映射 | `{单选题:'0',多选题:'1',填空题:'2',判断题:'3',简答题:'4',名词解释:'5',论述题:'6',计算题:'7'}` | L78-L87 | 无 | L3042、L3109、L3162、L3221、L3269 |
| `token` | `''` | 「付费题库token」（面板文案 L1004） | L88 | L1002-L1007 | L501（请求头）、L519（`data.key`）、L674（`checkKey`） |
| `aiAsk` | `false` | 「是否AI答题（正在开发中）」 | L89 | L1008-L1013 | **全文件无消费点** |

运行时动态附加到 `defaultConfig` 的键（不在初始定义中）：
- `workinx` / `succ` / `fail`：L614-L616 初始化为 0，L2863/L2875/L2928 累加，用于进度显示。
- `lastMsg`：L600-L607，只写内存、**不落盘**，供 `threadWatch` 比对悬浮窗文本。
- `hidden`：**从未定义**却被读取（L2537 `if (!defaultConfig.hidden)`），永远走真分支。

### 缓存读写点

| 操作 | 键 | 行号 | 说明 |
| --- | --- | --- | --- |
| 读 | `"20230524"`（`cache_key`，L115） | L127 | `let cacheData = GM_getValue(cache_key)` |
| 补字段 | 同上 | L128 | `if (cacheData && !cacheData.token) cacheData.token = defaultConfig.token`（**只补 token 一个字段**） |
| 覆盖 | 同上 | L129 | `defaultConfig = cacheData \|\| defaultConfig`（整对象替换，非合并） |
| 写 | 同上 | L641 | 考试页切 `autoSwitch` 时 |
| 写 | 同上 | L1099 | 配置面板点「保存」时 |
| 通用写 | 任意 key | L205 | `utils.cache(key,value)` → `GM_setValue(key,{value,time})`，**全文件无调用点** |
| 通用读 | `"noticetemp"` | L208-L216 / L896 | `utils.cacheExpired("noticetemp",600000)`，**从未被写入**，因此永远返回 `false` |

### `layx_config()` 表单映射（L900-L1119）

- 表单描述数组 `configForm`（L901-L1016）分 3 组：`基本设置/base`（L903-L937）、`章节配置/chapter`（L938-L979）、`其他配置/other`（L980-L1015）。
- 每个字段对象形如 `{type,label,value,desc}`，**键名即 `defaultConfig` 字段名**。
- 渲染：L1017-L1053 生成 radio tab + 三段 `tab-content`；控件类型见 L1026-L1037（`textarea`/`number`/`checkbox`，其它返回空串）。
- 提交回调（L1057-L1101）：
  1. 双重循环遍历 `configForm`（L1060-L1077），按 `name` 属性回读 DOM：checkbox→`.is(':checked')`（L1063-L1065）、textarea→`.val()`（L1066-L1068）、number→`.val()`（L1069-L1071），写回 `defaultConfig[key]`（L1074）。
  2. 校验与纠偏：`interval<1000`→3000（L1078-L1081）；`videoSpeed>16`→1（L1082-L1084）；`>1` 仅告警（L1085-L1086）；`<1`→1（L1087-L1090）；`matchRate` 越界→0.8（L1091-L1094）；`autoSubmitRate` 越界→0.8（L1095-L1098）。
  3. `GM_setValue(cache_key, defaultConfig)`（L1099）→ `layx.destroy(id)`（L1100）。
- 表单样式与 tab 切换纯 CSS 实现：L1117（`#base:checked~#base-content{display:block}` 等）。
- **注意**：`number` 类型回读的是**字符串**（L1070），因此 `defaultConfig.interval`、`videoSpeed`、`readSpeed` 保存后类型从 `number` 变为 `string`，仅靠数值比较的隐式转换维持运行（详见「风险与脆弱点」）。

---

## 模块地图

### `utils`（L143-L268）

| 方法 | 签名 | 职责 | 返回值 | 行号 |
| --- | --- | --- | --- | --- |
| `randomStr` | `(len=32)` | 生成小写字母数字随机串 | `string` | L144-L151 |
| `notify` | `(level,msg)` | 生成 postMessage 用的 JSON 字符串 | `string` | L152-L158 |
| `sortData` | `(data)` | 按 `parentnodeid` 把平铺章节数组构建成树 | 根节点数组 | L159-L166 |
| `toOneArray` | `(arr)` | 树 → 一维数组（递归） | `array` | L167-L169 |
| `sleep` | `(time)` | Promise 延时 | `Promise` | L170-L172 |
| `getUrlParam` | `(name)` | 从 `location.search` 取参（`unescape`） | `string\|null` | L173-L177 |
| `toQueryString` | `(obj)` | 键排序、数组展开、`encodeURIComponent` | `string` | L178-L183 |
| `getInputParam` | `(name)` | 取 `document.getElementsByName(name)[0].value` | `string\|null` | L184-L187 |
| `getVideoEnc` | `(clazzid,uid,jobid,objectId,playingTime,duration)` | **视频进度签名**：`md5("[clazzid][uid][jobid][objectId][playingTime*1000][d_yHJ!$pdA~5][duration*1000][0_duration]")` | `string` | L188-L190 |
| `getTimestamp` | `()` | `Date.getTime()` | `number` | L191-L193 |
| `removeHtml` | `(html)` | 去标签（保留 `img/sub/sup/br`）、`&nbsp;`→空格、折叠空白、`<br>`→`\n`；非字符串原样返回 | `string` | L194-L203 |
| `cache` | `(key,value)` | 写 GM 缓存（带时间戳） | `value` | L204-L207 |
| `cacheExpired` | `(key,time)` | 读 GM 缓存，未过期返回值，否则 `false` | `any\|false` | L208-L216 |
| `matchIndex` | `(options,answer)` | 精确匹配，返回选项下标数组（O(n²)，**允许重复下标**） | `number[]` | L217-L227 |
| `similarity` | `(s,t)` | Levenshtein 归一化相似度 | `number` | L228-L240 |
| `fuzzyMatchIndex` | `(options,answer)` | 逐答案取最相似选项，`>defaultConfig.matchRate` 才收 | `number[]` | L241-L255 |
| `processTtfList` | `async (ttflist)` | 依次尝试 3 个 ttf 映射表源，`ServerApi.request(item,'GET',{})` + `JSON.parse`，全部失败返回 `null` | `object\|null` | L256-L267 |

### `api`（L269-L447，超星站内接口）

| 方法 | 签名 | 职责 | 返回值 | 行号 |
| --- | --- | --- | --- | --- |
| `monitorVerify` | `(responseText,url,method,data,ua)` | 验证码兜底：解析响应 → `layx.prompt` 贴图；解析失败则 `layx.iframe` 打开 `antispiderShowVerify.ac` 并每 5s 轮询重试 | `Promise<response>` | L270-L298 |
| `defaultRequest` | `async (url,method,data={},ua=defaultConfig.ua,verify=false)` | **超星请求统一出口**：GET 直接请求；POST 走 `utils.toQueryString`；命中「输入验证码」自动走 `monitorVerify` 后递归重试；捕获 "connect list" 报错并提示补 `@connect` | `Promise<response>` | L299-L351 |
| `getVerifyCode` | `async (url)` | GM 请求图片（`responseType:"blob"`）→ FileReader → dataURL | `Promise<string>` | L354-L370 |
| `getCourseChapter` | `async (courseId,classId,pz)` | `/gas/clazz`，取课程+章节树元数据 | `object` | L371-L375 |
| `getChapterList` | `async (courseid,clazzid,nodes,userid,cpi)` | POST `/job/myjobsnodesmap`，取各章完成情况 | `object` | L376-L388 |
| `getChapterInfo` | `async (id,courseid)` | `/gas/knowledge`，取章内卡片列表 | `object` | L389-L399 |
| `getChapterDetail` | `async (courseid,clazzid,knowledgeid,num,cpi)` | `/knowledge/cards`，返回含 `mArg` 的 HTML 文本（**URL 里写死 `cpi=229749849`**，L401） | `string` | L400-L404 |
| `uploadStudyLog` | `async (courseid,clazzid,knowledgeid,cpi)` | 先从 `studentcourse` 页正则抓 `encode=`（L408），再请求 `/log/setlog` 上报学习记录 | `string\|false` | L405-L416 |
| `docStudy` | `async (jobid,knowledgeid,courseid,clazzid,jtoken)` | `/ananas/job/document` 完成文档任务 | `object` | L417-L421 |
| `videoStudy` | `async (data,dtoken,taskDefaultConfig)` | `taskDefaultConfig.reportUrl + "/" + dtoken + "?" + query` 上报视频进度 | `object` | L422-L426 |
| `getVideoConfig` | `async (objectId)` | `/ananas/status/{objectId}` 取视频元数据（duration/objectid/dtoken） | `object` | L427-L431 |
| `unlockChapter` | `async (courseid,clazzid,knowledgeid,userid,cpi)` | `/job/submitstudy` 解锁闯关章节 | `number`(status) | L432-L436 |
| `initdatawithviewer` | `async (mid,cpi,classid,taskDefaultConfig)` | `taskDefaultConfig.initdataUrl` 取视频内嵌题 | `array` | L437-L441 |
| `submitdatawithviewer` | `async (classid,cpi,objectid,eventid,memberinfo,answer)` | `/question/quiz-validation` 提交视频题（答案直接取自 `isRight==true` 的选项，L1273） | `object` | L442-L446 |

### `ServerApi`（L448-L587，第三方题库接口）

| 方法 | 签名 | 职责 | 返回值 | 行号 |
| --- | --- | --- | --- | --- |
| `request` | `async (url,method,data,headers={})` | 裸 `GM_xmlhttpRequest`，超时 **5000ms**；GET 不带 body，非 GET 用 `JSON.stringify` | `Promise<response>` | L449-L489 |
| `defaultRequest` | `async (url,method,data,headers={})` | 注入公共头（`Content-Type/v/referer/t/token/u`，L496-L503），**遍历 `reqUrl` 逐个尝试，只接受 `status===200`；全部失败时无 return（隐式 `undefined`）** | `Promise<response>\|undefined` | L490-L517 |
| `search` | `async (data,status=true)` | 题库搜索：`data.key = status && defaultConfig.token \|\| ''`（L519），`data.source='xy_'+version`（L526），`url='search?'+query`，POST `data` | `Promise<response>` | L518-L529 |
| `configRequest` | `async (url)` | `defaultRequest(url,'get')` 的薄封装 | — | L530-L531（**无调用点**） |
| `get_msg` | `async ()` | `def/autoMsg` 取公告，解析失败回落 `defaultConfig.notice` | `string` | L532-L540 |
| `searchOther` | `(data,item)` | 按 `otherApi` 条目发请求（免费题库通道），超时 5000ms | `Promise<response>` | L541-L559 |
| `checkKey` | `async (key)` | POST `key` 验证付费 token；成功时把 `num/usenum` 挂到 `reqUrl`（L572-L573）；**key 为空只告警不 return（L561-L563）** | `void` | L560-L578 |
| `getVerifyCode` | `async (img)` | POST `code`（base64 去前缀）→ 服务端 OCR 返回验证码 | `string` | L579-L586 |

### `page`（L588-L3281）

| 方法 | 签名 | 职责 | 返回值 | 行号 |
| --- | --- | --- | --- | --- |
| `threadWatch` | `async ()` | 每 320s 检查 `#layx_status_msg` 是否消失（消失→`alert`+停表）；若文本含「每60秒更新一次进度」且与上次一致→`location.reload()` | `void` | L589-L610 |
| `init` | `async ()` | 路由分发入口 | `Promise` | L611-L882 |
| `layx` | `async (id="abcde",option={})` | 创建悬浮窗：默认样式 L894，公告栏 L896（`cacheExpired("noticetemp")`→`ServerApi.get_msg()`），标题含版本号 L898 | `void` | L883-L899 |
| `layx_config` | `function ()` | 配置面板（见「配置系统」） | `void` | L900-L1119 |
| `layx_log` | `function (msg,level="info")` | 追加日志行；超出可视行数则删首行 | `void` | L1120-L1129 |
| `layx_status_msg` | `function (msg)` | 写 `#layx_status_msg` | `void` | L1130-L1132 |
| `mainTask` | `async ()` | **主任务调度**（见流程章节） | `void` | L1133-L1209 |
| `finishTask` | `async (task,item3,pz,taskDefaultConfig)` | 单任务分发：`video`/`document`/`workid` | `Promise<void>` | L1210-L1265 |
| `finishVideoAnswer` | `async (task,pz,videoData,taskDefaultConfig)` | 视频内嵌题自动作答 | `Promise<void>` | L1266-L1286 |
| `finishVideo` | `async (task,videoData,pz,taskDefaultConfig)` | **视频进度上报循环** | `Promise<boolean>` | L1287-L1332 |
| `finishWork` | `async ()` | 监听 `message` 事件等待作业 iframe 汇报结果 | `Promise<void>` | L1333-L1352 |
| `requestMerge` | `async (data,num=0)` | **题库聚合与回退**（见流程章节） | `Promise<array>` | L1359-L1449 |
| `clear` | `function ()` | 清空全页答案（含 UE 富文本编辑器） | `void` | L1450-L1459 |
| `clearCurrent` | `function (item)` | 清空指定题目的答案（旧版 DOM） | `void` | L1460-L1475 |
| `clearCurrentNew` | `function (item)` | 清空指定题目的答案（mooc2 DOM） | `void` | L1476-L1491 |
| `decode` | `async ()` | **字体反爬解码**（见流程章节） | `Promise<void>` | L1497-L2485 |
| `base64ToUint8Array` | `(base64)` | base64 → Uint8Array | `Uint8Array` | L2486-L2493 |
| `getQuestion` | `function (type,html='')` | 按 `'1'`(章节作业)`'2'`(作业)`'3'`(考试) 三种 DOM 结构抽题 | `{question,options,type,questionData,workType}` | L2494-L2556 |
| `answerFormat` | `function (answer)` | 数组去 null + 去标签；字符串走 `.cl()` | `array\|string` | L2557-L2571 |
| `setAnswer` | `function (type,options,answer)` | 考试页填答（单选/多选/判断/填空等） | `boolean\|array` | L2572-L2615 |
| `setWorkAnswer` | `function (type,options,answer,inx)` | 作业页填答 | `boolean\|array` | L2616-L2660 |
| `setChapterAnswer` | `function (type,options,answer,inx)` | 章节作业填答（**`mooc2=1` 时转派 `setChapterAnswerNew`**，L2662-L2664） | `boolean\|array` | L2661-L2727 |
| `setChapterAnswerNew` | `function (type,options,answer,inx)` | mooc2 版章节填答 | `boolean\|array` | L2728-L2794 |
| `randomChapterAnswer` | `function (type,options,inx)` | 随机作答（**无调用点**） | `boolean` | L2795-L2810 |
| `startAsk` | `async (data)` | 考试页答题主循环（单题 + 可选自动切题） | `Promise<void>` | L2812-L2831 |
| `startWork` | `async ()` | 作业页逐题作答（输出进度表） | `Promise<void>` | L2832-L2891 |
| `startChapter` | `async ()` | 章节作业逐题作答 + 自动提交 | `Promise<void>` | L2892-L2975 |
| `getScore` | `async ()` | 从「已批阅」页回收正确答案 → 上传 `save1` | `Promise<void>` | L2976-L3056 |
| `getScoreNew` | `async ()` | mooc2 版成绩回收 | `Promise<void>` | L3057-L3124 |
| `getScore2` | `async (data)` | 按服务端下发的 `data.url` 抓页面 → 解析单选/多选 → POST 到 `data.url1` | `Promise<void>` | L3125-L3181 |
| `getScore3` | `async ()` | `/mooc2/work/view` 页成绩回收 | `Promise<void>` | L3182-L3231 |
| `getScore4` | `async ()` | 考试卷面成绩回收 | `Promise<void>` | L3232-L3280 |

### 调用依赖关系

```
page.init ──┬─ page.layx ── ServerApi.get_msg ── ServerApi.defaultRequest ── reqUrl[]  (付费题库域名)
            ├─ page.layx_config ── GM_setValue(cache_key)
            ├─ ServerApi.checkKey ── ServerApi.defaultRequest
            ├─ page.threadWatch
            ├─ page.decode ── utils.processTtfList ── ServerApi.request (ttflist 三个第三方 ttf 源)
            ├─ page.mainTask ── api.* ── api.defaultRequest ── GM_xmlhttpRequest (超星站点)
            │        └─ page.finishTask ─┬─ page.finishVideoAnswer ── api.initdatawithviewer/submitdatawithviewer
            │                            ├─ page.finishVideo ── utils.getVideoEnc + api.videoStudy
            │                            ├─ api.docStudy
            │                            └─ layx.iframe + page.finishWork (message 事件)
            ├─ page.startAsk / startWork / startChapter ── page.requestMerge
            │        └─ ServerApi.searchOther (免费 otherApi) / ServerApi.search (付费 reqUrl)
            └─ page.getScore* ── ServerApi.defaultRequest("save1")
```

- `utils` 被 `api`、`ServerApi`、`page` 三方共用（无反向依赖）。
- `api` 只依赖 `utils`、`page.layx_log`（L282、L332、L345、L347）、`layx`、`GM_xmlhttpRequest`。
- `ServerApi` 依赖 `utils`、`reqUrl`、`otherApi`、`script_info`、`_self`、`page.layx_log`（L562、L575、L577）。
- `page` 依赖全部三者，并大量直接操作 `_self`（`unsafeWindow`）与 jQuery。
- **`page.decode` 是 `page` 中唯一依赖 `ServerApi` 的业务函数**（L2464），其余 `ServerApi` 调用集中在 `layx`/`requestMerge`/`getScore*`/`checkKey`。

---

## 关键业务流程

### 1. 主任务调度 `mainTask`（L1133-L1209）

1. 组装上下文 `pz = {courseid, clazzid, cpi, userid}`，来源为 URL 参数 / `#userId` 输入框 / `_self.uid`（L1134-L1139）。
2. `api.getCourseChapter` 取课程与章节树（L1140）→ `utils.sortData`+`toOneArray` 拍平（L1142）。
3. 判定模式：任一章 `status==="task"` → 闯关模式（L1143、L1145）；`reviewMode` → 复习模式（L1144）。
4. `api.getChapterList` 取各章未完成数，累加为 `unfinishcount`（L1147-L1148）。
5. 内层 `doTask(item)`（L1151-L1189）：
   - 闯关模式且 `jobcount==0` → `api.unlockChapter` 解锁（L1152）。
   - `unfinishcount===0 && !reviewMode` → 直接结束（L1153-L1155）。
   - 单章已完成的跳过（L1157-L1160）。
   - `api.uploadStudyLog` 上报学习记录（L1161）。
   - `api.getChapterInfo` 取卡片列表（L1163）；先请求一次 `transfer` 页（L1164-L1165）。
   - 对每张卡片 `api.getChapterDetail` 取 HTML，若含「章节未开放」→ 返回 `"unlock"`（L1169-L1171）。
   - 正则 `/mArg\s*=\s*({.*?});/` 抽取任务配置（L1172-L1177），遍历 `mArg.attachments`，逐个 `page.finishTask`（L1178-L1184），每步 `sleep(interval)`。
6. 外层循环（L1190-L1206）：记录 `lastItem`；`back=="unlock"` 时解锁上一章并 `unlockChapterNum++`（L1201）；`back===false` 则 `break`（L1202-L1204）。
7. 结束语（L1207-L1208）。

**逻辑瑕疵**：`if (unlockChapterNum > 0)`（L1193）在自增**之前**判断，因此第一条「章节未开放」提示要等到第二次触发才出现；闯关模式下会直接 `return` 停刷（L1195-L1198）。

### 2. 章节学习（`finishTask` L1210-L1265）

| 任务类型 | 分支行 | 步骤 |
| --- | --- | --- |
| `video` | L1218-L1234 | ① `autoVideo` 关闭则跳过（L1219-L1223）② `api.getVideoConfig(objectId)`（L1224）③ `finishVideoAnswer` 先做内嵌题（L1225）④ `finishVideo` 跑进度（L1227） |
| `document` | L1235-L1244 | `api.docStudy(task.property.jobid, item3.knowledgeid, courseid, clazzid, task.jtoken)`（L1241） |
| `workid` | L1245-L1258 | ① 打印付费题库剩余次数（L1251）② 拼 `/api/work?...` URL（L1252）③ `layx.iframe('workiframe','作业',url)`（L1254）④ `finishWork()` 等 iframe 的 postMessage（L1255）⑤ 关闭 iframe（L1256） |
| 其它 | L1259-L1262 | 记「暂不支持该任务类型」 |

**视频进度上报 `finishVideo`（L1287-L1332）分步**：
1. 组装上报体 `data`：`clazzId/playingTime/duration/clipTime/objectId/otherInfo(去掉 &cour 之后)/courseId/jobid/userid/isdrag/view=pc/enc/rt(默认0.9)/dtype(Video或Audio)/_t`（L1289-L1305）。
2. `duration` 为空 → `resolve(false)`（L1306-L1308）**但未 return**（继续执行，见风险）。
3. `while(true)`（L1311-L1329）：
   - `isdrag = time<duration ? 3 : 4`（L1312）；`playingTime` 取 min（L1313）。
   - 更新悬浮窗进度文案「每60秒更新一次进度」（L1314-L1315）。
   - `data.enc = utils.getVideoEnc(...)`（L1316）→ `api.videoStudy(data, videoData.dtoken, taskDefaultConfig)`（L1317）。
   - 终止条件：`time>=duration` 或（`result.isPassed && !reviewMode`）（L1318-L1320）。
   - `time += 60 * videoSpeed`（L1321）；若超出时长则补 `sleep(waitTime*1000)`（L1322-L1325），否则 `sleep(60000)`（L1327）。
4. `resolve(result.isPassed)`（L1330）。

**文档/阅读时长**：
- 文档任务走 `api.docStudy`（L417-L421、L1241）。
- 独立阅读时长路由：`/ztnodedetailcontroller/visitnodedetail`（L855-L866）用 `animate` 滚到底、时长 = `滚动距离 × (readSpeed||10)` 毫秒（L860），结束后点 `.nodeItem.r i`（L862）。
- 章节卡片级学习记录走 `api.uploadStudyLog`（L405-L416、L1161）。

### 3. 作业自动答题（`startWork` L2832-L2891）

1. 建结果表格 `#qlist` 写入 `#layx_content`（L2833-L2845）。
2. 对每个 `.questionLi` 循环 `startWorkTask(i)`（L2847-L2884）：
   - `page.getQuestion("2", questionList[i])` 抽题（L2848）。
   - 生成三列表格行（题号/题目/答案）（L2849-L2853）。
   - `page.requestMerge(data)` 取候选答案（L2854）→ 找第一个「数组且 `setWorkAnswer` 成功」的答案（L2855）。
   - 若失败且 `freeFirst` → `page.requestMerge(data,1)` 再试（L2856-L2859）。
   - 成功：绿色 + `succ++`（L2860-L2863）；失败：红色 + `fail++` + 注入「重试」链接（L2864-L2877）。
   - 更新进度文案与表格行（L2878-L2883）。
3. 循环间 `sleep(interval)`（L2888），最后汇总（L2890）。
> 注意：作业页**没有自动提交**逻辑，提交由 iframe 内页面自行处理（L754-L780 分支只调用 `startWork`）。

### 4. 章节作业答题 + 自动提交（`startChapter` L2892-L2975）

1. 目标选择器为 `.TiMu`（L2893），`getQuestion("1", ...)`（L2907）。
2. 取答案/回退逻辑同 `startWork`（L2913-L2923），但调用 `page.setChapterAnswer`（L2915、L2922）。
3. 汇总文案（L2954）→ 计算正确率 `z = succ / questionList.length`（L2955）。
4. 若 `autoSubmit`（L2956-L2974）：
   - 先注册一个 200s 兜底定时器，超时向父窗口发「提交超时，已暂时关闭」（L2957-L2959）——**该 `setInterval` 不会在成功路径清除**。
   - `z >= autoSubmitRate` → `btnBlueSubmit()`（全局函数，来自页面）+ `submitCheckTimes()`（L2960-L2965）。
   - 否则 `noSubmit()` + 覆盖 `window.alert` 为只打日志（L2967-L2972）。

### 5. 考试答题（`startAsk` L2812-L2831 + 路由分支 1 L619-L651）

1. 进入 `reVersionTestStartNew` 且 `newMooc=true` 时创建 `ks` 悬浮窗（L621-L626），隐藏原生标题（L627），注入「自动切换」「配置」按钮（L629-L647）。
2. `page.getQuestion("3")` 从考试 DOM 抽题（L649、L2494-L2547），并把题干/选项渲染到 `#layx_content`（L2537-L2546）。
3. `page.startAsk(reqData)`（L651、L2812-L2831）：
   - `requestMerge(data)` → 找第一个可成功 `setAnswer` 的答案（L2814-L2815）。
   - 失败且 `freeFirst` → `requestMerge(data,1)` 重试（L2816-L2819）。
   - 成功显示绿色答案，失败显示服务端返回的提示串（L2820-L2827）。
   - `autoSwitch` 时点击「下一题」（L2829）；找不到按钮则提示「答题已完成」（L2830）。
4. 非 `newMooc` 的考试 URL 会被强制改写成 `newMooc=true` 后跳转（L652-L660）。

### 6. 字体反爬解码 `decode`（L1497-L2485）

1. 内联第三方库 **Typr**（L1498-L2448，作者 wyn，出处见注释 L1493-L1496）：解析 TTF/OTF 的 `cmap/head/hhea/maxp/hmtx/name/OS-2/post/loca/kern/glyf/CFF/SVG` 表，支持 `shape`/`glyphToPath`/`pathToSVG`，并可选用 WebAssembly HarfBuzz 做复杂整形（`initHB`，L2399-L2447，需外部 wasm 地址，本脚本未调用）。
2. 取字体：`$('style:contains(font-cxsecret)')`（L2449）；**不存在则直接 `return`**（L2450）。
3. 正则提取 `base64,...`（L2451）→ `Typr.parse(base64ToUint8Array(font))[0]`（L2452）。
4. 取映射表：`await utils.processTtfList(ttflist)`（L2464），即依次尝试 L123-L125 三个第三方 JSON；全部失败 → `throw new Error('未找到字体文件')`（L2465-L2467）。
5. 遍历 CJK 基本区 `19968..40869`（L2469-L2475）：`codeToGlyph` → `glyphToPath` → `md5(JSON.stringify(path)).slice(24)` 作为键，取 `table[hash]` 得到真实码点。
6. 替换 `.font-cxsecret` 内文本（L2476-L2484）并移除 class。
- `base64ToUint8Array` 见 L2486-L2493。
- 调用点：**仅 L726**（作业页 `oldWorkId` 分支），失败会 `layx.msg('字体解码失败…')` 并 `return`（L727-L732）。

### 7. 验证码兜底

| 场景 | 步骤 | 行号 |
| --- | --- | --- |
| 请求头返回含「输入验证码」 | `api.defaultRequest` 检测 → `layx_log` 提示 → `api.monitorVerify` → 递归重试 | L331-L338 |
| `monitorVerify`：响应是 JSON | 解析 `verify_png_path` / `verify_path`，`layx.prompt` 显示图片与输入框，确认后 `window.open(verify_path + "&ucode=" + value)` | L273-L278 |
| `monitorVerify`：响应非 JSON | 打开 `https://{domain}/antispiderShowVerify.ac` 的 iframe，每 5s 递归调用 `defaultRequest(...,verify=true)` 探测，成功后销毁 iframe 并 `resolve` | L279-L296 |
| 浏览到验证码页面本身 | 抓 `/processVerifyPng.ac` → `api.getVerifyCode`（blob→dataURL）→ `ServerApi.getVerifyCode`（**上传到付费题库做 OCR**）→ 跳转 `/html/processVerify.ac?...&ucode=` | L783-L789 |
| `@connect` 未加白名单 | 从错误串提取域名，提示「请复制以下代码至脚本中的第19行位置」（**行号提示已过期**，当前 `@connect` 在 L13-L34） | L342-L345 |

### 8. 题库搜索与多源回退（`requestMerge` L1359-L1449）

1. 写入 `data.id = _self.uid || getCookie('UID') || getCookie('_uid') || 0`（L1362，try/catch 兜底为 0）。
2. **免费优先分支**（`defaultConfig.freeFirst && num === 0`，L1367-L1391）：
   - `otherApi.forEach` → `ServerApi.searchOther(data,item)` → `item.getanswer(response)`（L1369-L1378），每个条目 `.catch(()=>[])`。
   - 同时**空跑一次付费搜索**：`ServerApi.search(data,false)` 的结果被丢弃（L1381-L1385）。
   - 返回 `await Promise.all(promiseArr)`（L1390）——**只含免费题库结果**。
3. **付费分支**（L1392-L1446）：`ServerApi.search(data)`，按 `result.code` 分派 200/401/403/404/500/default（L1396-L1415）。`default` 分支会回调 `page.getScore2(result.data)`（L1413）。
4. 失败重试：`res === "error-_-"` 且 `num < 3` → 递归 `requestMerge(data, num+1)`（L1437-L1438）；`num>=3` → 返回 `[]`（L1439-L1440）。
5. 调用方在「无有效答案」时对免费优先模式补一次 `num=1`（L2816-L2819、L2856-L2859、L2917-L2923），此时 `num!==0` 会走付费分支，形成「免费→付费」回退。

> 详见下一节《题库链路取证》。

### 9. 成绩/答案回收（`getScore`~`getScore4`）

| 函数 | 触发路由 | 抽取内容 | 上传 | 行号 |
| --- | --- | --- | --- | --- |
| `getScore` | `/work/selectWorkQuestionYiPiYue`（非 mooc2）、`reEdit=2` | 判断题/填空题的**正确答案**（含「我的答案」反推，L2997-L3007） | `ServerApi.defaultRequest("save1","post",{questionList,url:location.href})` | L2976-L3056（上传 L3050-L3055） |
| `getScoreNew` | 同上（mooc2=1） | 判断题/填空题正确答案（用 `.marking_dui/.marking_cuo` 判定对错，L3070-L3084） | 同上 | L3057-L3124（上传 L3117-L3122） |
| `getScore2` | 付费题库 `search` 的 `default` 分支下发 `data.url` | 抓取该 URL，解析 `.Py-mian1` 下的单选/多选 | POST 到服务端下发的 `data.url1` | L3125-L3181（L3130-L3131、L3176-L3178） |
| `getScore3` | `/mooc2/work/view` | 判断题/填空题 | 同上 | L3182-L3231 |
| `getScore4` | 考试卷面页 | 判断题/填空题 | 同上 | L3232-L3280 |

> 事实：这 5 个函数都会把 **题目 + 正确答案 + 当前 URL** 回传到第三方题库服务器（`save1`）。这是一条明确的「答案回填」链路。

### 10. 配置面板刷新

- 打开入口：`/mycourse/stu?`（L670）、考试页（L645）、作业页（L720、L769）。
- 打开前会 `log(defaultConfig)` 打印当前配置（L644、L719、L768）。
- 保存后仅写 GM（L1099）并关闭窗口（L1100）；**不刷新页面、不重启任务**，新配置对「当前已进入的循环」生效与否取决于各 `await` 的读取时机（多数在每轮循环里重新读 `defaultConfig.x`，因此大多即时生效）。
- `layx_config` 独立于 `layx` 主窗口，使用固定 id `'Domsd'`（L1054）。

---

## 题库链路取证

> 本节所有结论基于代码文本；凡带「推断」标记的为解释性结论，未经外部证据验证。外部核实未能完成（本机无外网，`web_fetch` 返回 `fetch failed`），因此**无法确认 greasyfork 脚本 435357 的当前内容与原作者身份**。

### 1. `otherApi`（一之题库，`cx.icodef.com`）完整调用链

**定义**：L90-L114，数组，只有 1 个元素（L91-L114）。

```js
// L90-L114
}, otherApi = [
    {
        desc: "接口来源:https://cx.icodef.com/query.html",        // L92
        url: 'http://cx.icodef.com/wyn-nb?v=4',                    // L93
        headers: {
            'Content-Type': 'application/x-www-form-urlencoded;charset=utf-8',
            'Authorization': '8l395BS2NoxMdDGH'/** 一之题库token填写 */   // L96
        },
        method: 'post',                                            // L98
        getdata: (data) => `question=${encodeURIComponent(data.question)}`,  // L99-L101
        getanswer: (response) => { ... }                           // L102-L113
    }
],
```

**调用链（唯一路径）**：

```
page.startAsk / startWork / startChapter
  └─ page.requestMerge(data, num)                       L2814 / L2854 / L2913
       └─ if (defaultConfig.freeFirst && num === 0)     L1367
            ├─ otherApi.forEach(item =>                 L1369
            │     ServerApi.searchOther(data, item)      L1371  → L541-L559 (GM_xmlhttpRequest, timeout 5000)
            │       .then(response => item.getanswer(response))   L1373 → L102-L113
            │       .catch(() => []))                    L1376
            ├─ ServerApi.search(data, false)  ← 结果被丢弃（空跑）  L1381-L1385
            └─ return await Promise.all(promiseArr)      L1390
```

**触发条件（事实）**：
- 免费接口**只在 `defaultConfig.freeFirst === true` 且 `num === 0` 时被调用**（L1367）。
- `defaultConfig.freeFirst` 默认 `true`（L74）。
- 若 `freeFirst` 被设为 `false`，第一次就进入 `else` 分支（L1392）只走付费接口，而调用方补调的 `requestMerge(data,1)`（L2817/L2857/L2919）也因为 `num!==0` 落到 `else`——因此 **`freeFirst=false` 时 `otherApi` 全程不会被调用**。
- 免费优先分支**不是「失败后回退」到免费，而是「免费优先、失败后回退付费」**：免费结果由 L1390 返回；无有效答案时调用方传 `num=1` 再进付费分支（L2816-L2819、L2856-L2859、L2917-L2923）。

**硬编码 token 位置（逐项）**：

| 内容 | 行号 | 说明 |
| --- | --- | --- |
| `'Authorization': '8l395BS2NoxMdDGH'` | **L96** | 写死的 token 值 |
| `/** 一之题库token填写 */` | **L96**（同一行，行尾注释） | 注释语义是「此处填写一之题库 token」→ 说明该字段**原设计留给使用者填写** |

**关键事实**：`defaultConfig` 中唯一与 token 相关的字段是 `token`（L88，面板标签「付费题库token」L1004），它被用于**付费**链路（L501、L519、L674）。**没有任何配置项能覆盖 L96 的 `Authorization`** —— 免费题库的 token 是硬编码的。

### 2. `reqUrl`（付费题库 `api.tikuhai.com`）完整链路

**定义**：L115-L120。

```js
], _self = unsafeWindow, top = (null && (_self)), script_info = GM_info.script,
cache_key = "20230524", reqUrl = [                    // L115
    {
        "api": "https://api.tikuhai.com/",            // L117
        "headers": {}                                  // L118
    },
    ],                                                 // L120
```

**链路**：

```
ServerApi.defaultRequest(url, method, data, headers)   L490
  ├─ 注入公共头 v/referer/t/token/u                    L496-L503
  │    'token': defaultConfig.token || ''              L501  ← 配置面板「付费题库token」
  │    'u': _self.uid || getCookie('UID') || getCookie('_uid')  L502
  └─ for (i in reqUrl) { api.api + url }  只接受 status===200   L504-L516
       └─ 全部失败时函数无 return → 返回 undefined     L516-L517

使用方：
  ServerApi.search        → 'search?'+query   POST      L518-L529  ← 取答案主干
  ServerApi.checkKey      → 'key'             POST      L560-L578  ← 初始化校验 token
  ServerApi.getVerifyCode → 'code'            POST      L579-L586  ← 验证码服务端 OCR
  ServerApi.get_msg       → 'def/autoMsg'     GET       L532-L540  ← 悬浮窗公告
  ServerApi.configRequest → 任意 url          GET       L530-L531  （无调用点）
  page.getScore*          → 'save1'           POST      L3050/L3117/L3225/L3273 ← 答案回传
```

**强依赖点（删掉会破坏什么）**：

| 依赖点 | 行号 | 删除后的后果 |
| --- | --- | --- |
| `ServerApi.search` | L518-L529 | `page.requestMerge` 付费分支失效；`search(...)` 返回 `undefined` → `.then(response => JSON.parse(response.responseText))`（L1394-L1395）抛 `TypeError` → 被 `.catch`（L1416-L1436）吞掉，状态栏显示「请求错误,等待重试」，重试 3 次后返回 `[]` → 全部题目显示「暂无答案」 |
| `ServerApi.checkKey` | L560-L578；调用点 L674 | 课程首页初始化时多一次无效请求；L674 未 `await`，失败不影响主流程 |
| 公共头 `token` | L501 | 若无 key，服务端预计返回 401 → L1401-L1402 `return result.msg`（字符串），不构成数组 → 该候选被跳过 |
| `data.key` | L519 | 同上，付费检索失去鉴权 |
| `ServerApi.getVerifyCode` | L579-L586；调用点 L786 | 路由分支 8（L783-L789）验证码自动填写失效；`JSON.parse(res.responseText).data.code`（L585）会抛错，`page.init()` 的 catch（L3292）只调 `log()`，而 `log` 在 `debugger:false` 时是空函数（L130）→ **静默失败** |
| `ServerApi.get_msg` | L532-L540；调用点 L896 | 悬浮窗公告栏回落 `defaultConfig.notice`（L539），UI 不破 |
| `save1` 回传 | L3050、L3117、L3225、L3273 | 成绩回收功能失效（不再上传答案），但不影响答题本身 |
| `reqUrl.num/usenum` | 写入 L572-L573、L1398-L1399；展示 L1251、L2823、L2878、L2890、L2942、L2954 | 无处获取，UI 一直显示「暂未获取/暂未使用」 |

### 3. L102-L113 答案过滤：作用路径与完整代码

```js
// L102-L113
getanswer: (response) => {
    const res = JSON.parse(response.responseText);
    if (res.code === 1) {
        let data = res.data.replace(/javascript:void\(0\);/g, '').trim().replace(/\n/g, '');
        if (data.includes('叛逆') || data.includes('公众号') || data.includes('李恒雅') || data.includes('一之')) {   // L106
            return false;
        } else {
            return data.split("#");
        }
    }
    return false;
}
```

| 问题 | 事实 | 行号 |
| --- | --- | --- |
| 作用在哪个返回路径 | 它是 `otherApi[0]` 的属性，唯一调用点是 L1373 `item.getanswer(response)`，即**仅作用于 `cx.icodef.com`（一之题库）的响应** | L102-L113、L1373 |
| 是否影响付费题库 | **不影响**。付费走 `ServerApi.search` → L1394-L1415 直接 `JSON.parse(response.responseText)` 取 `result.data.answer`，不经过 `getanswer` | L1394-L1415 |
| 命中后的行为 | 返回 `false` → L1374 `res === false ? [] : res` → 该候选变为空数组，被 L2815/2855/2915 的 `item.length > 0` 过滤掉 | L1374 |
| 是否「连带」影响一之题库答案 | 是。过滤对象正是一之题库返回的答案文本；若一之返回内容中出现「一之」二字，**整条答案被丢弃**。**推断（强）**：把原作者品牌词（`一之`）放进作用于其自身响应的黑名单，在原始免费版中缺少合理性动机，符合「去除原作者标识」的改动意图；`叛逆/公众号/李恒雅` 更像广告水印词。**另一种同样成立的解读（推断，中）**：该过滤只是统一的「去广告」策略，`一之` 是因为其答案尾部附加了公众号/作者水印。代码中没有任何注释说明，**无法确认**动因。 |

### 4. 关键词逐处清单

| 关键词 | 行号 | 上下文 | 作用 |
| --- | --- | --- | --- |
| `icodef` | L13 | `// @connect      icodef.com` | 允许 GM 跨域访问一之题库域名 |
| | L92 | `desc: "接口来源:https://cx.icodef.com/query.html"` | 标注免费接口来源页 |
| | L93 | `url: 'http://cx.icodef.com/wyn-nb?v=4'` | 免费题库实际请求地址（**明文 http**） |
| `tikuhai` | L14 | `// @connect      tikuhai.com` | 允许访问付费题库域名 |
| | L117 | `"api": "https://api.tikuhai.com/"` | 付费题库 base URL |
| `一之` | L96 | `/** 一之题库token填写 */` | 说明该字段属一之题库 |
| | L106 | `data.includes('一之')` | 过滤含此词的答案 |
| | L92 | `接口来源:https://cx.icodef.com/query.html`（未直接写「一之」） | 间接指向一之 |
| `token` | L88 | `token: ''` | 默认空 |
| | L96 | `'Authorization': '8l395BS2NoxMdDGH'` | 免费题库硬编码 token |
| | L128 | 缓存回填 token | 兼容旧缓存 |
| | L501 | 请求头 `token` | 付费鉴权 |
| | L519 | `data.key = status && defaultConfig.token \|\| ''` | 付费检索鉴权键 |
| | L1002-L1007 | 面板 textarea「付费题库token」 | UI 入口 |
| | L674 | `ServerApi.checkKey(defaultConfig.token)` | 启动校验 |
| | L1241/L1252/L1317 | `jtoken` / `ktoken` / `dtoken` | **与题库无关**，是超星自身的任务令牌 |
| `key` | L564-L567 | `let url='key'; data={key}` | 付费 token 校验接口 |
| | L519 | `data.key=` | 检索参数 |
| | L325/L119 | 变量名 `key` | 无关 |
| | L2477-L2481 | 字体替换循环变量 | 无关 |
| | L3283-L3286 | localStorage key 遍历 | 无关 |
| `search` | L518-L529 | `ServerApi.search` | 付费检索 |
| | L527 | `'search?' + query` | 付费接口路径 |
| | L541-L559 | `ServerApi.searchOther` | 免费检索 |
| | L1369-L1394 | 调用点 | 聚合 |
| | L1767、L1734 等 | Typr 内部 `searchRange`/`search` 字段 | 无关 |
| `秘钥` | L562 | `page.layx_log("秘钥为空停止检测", "notice")` | `checkKey` 空 key 提示 |
| | L577 | `page.layx_log("秘钥验证失败", "notice")` | 校验失败提示 |
| `付费` | L52 | `@antifeature payment 存在第三方付费接口` | 元数据声明 |
| | L1000 | `desc: '开启后免费题库无答案后才会搜索付费题库'` | 面板文案 |
| | L1004 | `label: '付费题库token'` | 面板文案 |
| | L1251 | `付费题库剩余次数:${reqUrl.num}}` | 日志（多一个 `}`） |
| `antifeature` | L52 | `// @antifeature  payment  存在第三方付费接口` | Greasy Fork 元数据标记 |
| `@connect` | L13-L34 | 共 22 条 | L13 `icodef.com`、L14 `tikuhai.com`、L15-L31 超星各域、L32 `forestpolice.org`、L33 `muketool.com`、L34 `ocsjs.com`；**注意 L26 `chaoxing.comm` 为笔误** |
| | L344 | 报错提示「请复制以下代码至脚本中的**第19行**位置」 | 行号已过期 |
| `updateURL` | L54 | `https://update.greasyfork.org/scripts/435357/...meta.js` | 更新源 |
| `downloadURL` | L53 | `https://update.greasyfork.org/scripts/435357/...user.js` | 下载源 |

### 5. 代码一致性取证（第三方改动痕迹）

| # | 证据（行号） | 事实 | 推断 | 强度 |
| --- | --- | --- | --- | --- |
| 1 | L96 | `'Authorization': '8l395BS2NoxMdDGH'/** 一之题库token填写 */` —— 注释明确说此处置空待填，实际是具体 token | 该字段被第三方填入了自己的（或转售的）一之 token，使免费接口以「改动者的身份/额度」计费 | **强** |
| 2 | L92 vs L117 | `desc` 指向 `cx.icodef.com`（免费/原始接口），但真正兜底的是 `api.tikuhai.com` | 付费接口是**后插入**的；`otherApi` 保留的是原始结构 | **强** |
| 3 | L52 | `@antifeature payment` | 打包者自己声明了付费接口，属主动披露 | **强** |
| 4 | L13-L14 | 两个题库域名同时存在于 `@connect` | 双题库并存 | **强** |
| 5 | L1000 + L1004 | 面板明确区分「免费题库」与「付费题库token」 | 改动者知道两套链路的存在并为其加了 UI | 中强 |
| 6 | L1367-L1391 | 免费优先分支里 `ServerApi.search(data,false)` 的结果被直接丢弃（L1381-L1385 的 `.then(()=>[])`） | **推断**：这是「先记账/预热付费接口」的写法，或改动后遗留的空跑；无论动机如何，都说明付费链路被特意保留在免费分支中 | 中 |
| 7 | L106 | 过滤黑名单包含 `一之` | 见 §3，两解；但黑名单作用于被过滤源头自身的品牌词，至少说明**该行是人为定制**的 | 中强 |
| 8 | L115 | `cache_key = "20230524"`（日期型常量）与 L4 `@version 4.4.1` 无绑定 | 缓存键不随版本变化；若第三方在 2023-05-24 之后改过配置结构，老用户的旧配置会**整对象覆盖**新默认值（L129），新字段变 `undefined`，只有 `token` 被回填（L128） | 中 |
| 9 | L3053、L3120、L3228、L3276 | 传 `reqUrl.headers`，但 `reqUrl` 是**数组**且从未定义 `headers`（定义 L115-L120、L118 只在元素上） | **推断**：早期版本 `reqUrl` 曾是对象结构，改动后遗留的引用；当前值为 `undefined`，靠默认参数兜住 | 中 |
| 10 | L1251 | 模板串 `付费题库剩余次数:${reqUrl.num}}` 多一个 `}` | 手改痕迹 | 弱 |
| 11 | L26 | `// @connect chaoxing.comm` | 手写元数据笔误 | 中 |
| 12 | L72 + L924 / L89 + L1008 | `randomAnswer`、`aiAsk` 有默认值与面板控件，但全文件无任何消费点 | **推断**：原有逻辑被删除或从未实现，UI 与行为脱节（`randomAnswer:false` 时随机选答依然发生，L2684-L2688、L2748-L2752） | 中 |
| 13 | L700-L703 | `if (location.href.includes('mooc2=1')) { /* 仅两行注释 */ }` | 分支被清空但保留外壳 | 弱 |
| 14 | L2537 | `if (!defaultConfig.hidden)` —— `hidden` 从未定义 | **推断**：原版有「隐藏题目面板」开关，现被移除/未定义 | 中 |
| 15 | L530-L531 | `ServerApi.configRequest` 无调用点；L1359 `randomChapterAnswer` 无调用点；L204-L207 `utils.cache` 无调用点；L896 的 `noticetemp` 从未写入 | 多处死代码，说明代码被改写/裁剪过 | 中 |
| 16 | L53-L54 | `@downloadURL`/`@updateURL` 指向 greasyfork **script 435357**，与本文件内容无法离线比对 | **未能确认**该 ID 当前指向的是原作者发布页还是二次打包页 | — |

### 6. 补充判断：本次改动的性质

**事实层面**，免费（一之 `cx.icodef.com`）与付费（`api.tikuhai.com`）两套题库并存，免费链路的 token 被硬编码（L96）且**缺少任何用户可配置入口**，付费链路则拥有完整 UI（L1000-L1007）、启动校验（L674）、余额展示（L1251 等）与答案回传通道（L3050 等）。

**推断（中强）**：付费链路是后加的「变现层」，免费链路是被保留但被「锁死 token + 降级为二级优先」的原始功能。因此用户所说的「删掉付费接口、还原到向原作者申请 token 的免费接口」在代码上对应两处核心动作：
1. 把 L96 的 `Authorization` 改回可配置（或填入用户自己的 token）；
2. 让 `requestMerge` 只走免费链路（把 L1367 的条件改为不再依赖 `freeFirst`，或直接删掉 L1392-L1446 付费分支）。

---

## 术后影响面

> 场景：删除 `reqUrl` 链路、`ServerApi.checkKey`、秘钥相关 UI 与逻辑。以下逐条列出受影响的函数与行号。

| 受影响的点 | 行号 | 现状 | 删除后 | 必须同步改造 |
| --- | --- | --- | --- | --- |
| `reqUrl` 定义 | L115-L120 | 定义付费 base URL | 删除后 `ServerApi.defaultRequest` 的 `for` 循环（L504）立刻抛 `TypeError: reqUrl is not defined`（若连变量一起删） | 要么保留空数组 `reqUrl = []`，要么重写 L490 整体 |
| `ServerApi.defaultRequest` | L490-L517 | 付费题库请求出口 | `reqUrl` 为空时循环不执行 → **隐式返回 `undefined`** | 必须显式 `return null/undefined` 并让所有调用方判空 |
| `ServerApi.search` | L518-L529 | 用 `data.key=defaultConfig.token` 检索 | 删除后此函数是死代码 | 删除，或改为调用 `searchOther` 以复用免费链路 |
| `ServerApi.checkKey` | L560-L578 | 校验付费 token、写 `reqUrl.num/usenum` | 删除后无调用者 | 同时删调用点 L674 |
| `ServerApi.getVerifyCode` | L579-L586 | 付费服务端 OCR | 若只删 `reqUrl`：`defaultRequest` 返回 `undefined`，在 **L585** `res.responseText` 抛 `TypeError`；若连本函数一起删：L786 抛 `ReferenceError` | 改造 L783-L789：改为本地展示验证码让用户手填，或整段删除（必须同时处理 L785-L786 两次 `await`） |
| `ServerApi.get_msg` | L532-L540 | 悬浮窗公告 | 若只删 `reqUrl`：`res.responseText` 抛错但被本函数的 `try/catch`（L535-L540）接住 → 回落 `defaultConfig.notice`，**UI 不受影响**；若连本函数一起删：L896 抛 `ReferenceError` | 倾向保留该函数（零风险降级）；若确要删，需把 L896 改为直接使用 `defaultConfig.notice`（L76），否则所有 `page.layx()` 调用点（L621、L663、L705、L755）都会进入 L3292 的空 `log` 静默失败 |
| `ServerApi.configRequest` | L530-L531 | 死代码 | 可直接删 | 无 |
| `save1` 回传 | L3050-L3055、L3117-L3122、L3225-L3230、L3273-L3278 | 5 处答案回传 | 删除后 `getScore*` 只剩 DOM 解析 | 建议连 `getScore*` 的上传段一起删，否则会因 `defaultRequest` 返回 undefined 触发 `.then` 前的 TypeError（在 `.then` 链上，未包 try/catch，会产生**未捕获的 Promise 拒绝**） |
| `page.getScore2` | L3125-L3181 | 抓服务端下发的 `data.url` | 付费链路删除后此函数不再被触发（唯一调用点 L1413 在付费 `default` 分支） | 可整体删除（约 57 行） |
| 配置面板「付费题库token」 | L1002-L1007 | textarea → `defaultConfig.token` | 删除后 `defaultConfig.token` 成为无 UI 的孤儿字段 | 若改为「一之题库 token」，需同时改 L96 让 `Authorization` 读取 `defaultConfig.token`，并处理 L501/L519/L674 三个旧引用 |
| 配置面板「免费题库优先」 | L996-L1001 | `freeFirst` 开关 | 逻辑上不再需要（只剩一条题库链路） | 保留亦可（恒真），但 `desc` 文案（L1000）会失实，建议改文案 |
| 启动校验调用 | L674 | `ServerApi.checkKey(defaultConfig.token)` | 删除 | 若要保留「token 是否有效」的校验，需新增一个免费接口的校验请求 |
| 余额展示 | L1251、L2823、L2878、L2890、L2942、L2954 | 6 处 `reqUrl.num` | 删除后 `reqUrl` 未定义 | 逐处删除 `${reqUrl.num || '…'}` 片段，或统一改为 `''` |
| 搜索回退逻辑 | L1367-L1446 | 两分支 | 删除付费分支后 `requestMerge` 可简化为「调 `searchOther` 并 `Promise.all`」 | 同步删除 L2816-L2819、L2856-L2859、L2917-L2923 的 `requestMerge(data,1)` 二次调用，否则 `num!==0` 时会走进已删分支 |
| 请求头 | L496-L503 | `token`/`u`/`v`/`referer`/`t` | 若 `ServerApi.defaultRequest` 整体删除，这些头也没了 | 免费链路 `searchOther`（L541-L559）用的是 `item.headers`（L547），**不走公共头**，因此互不影响 |
| `@connect tikuhai.com` | L14 | 白名单 | 删除后 GM 无权限访问该域 | 同时删除，减少权限面 |
| `@antifeature payment` | L52 | 付费声明 | 去付费化后应删除 | 若保留，安装时会继续被打上「存在付费接口」标记 |
| `cache_key` | L115、L127-L129、L641、L1099 | 旧配置整对象覆盖 | 改配置结构（删 `token`/`freeFirst`）后，老缓存仍会把这两个字段带回来 | **必须同步 bump `cache_key`**，否则 L129 会把旧字段重新注入 `defaultConfig` |

**不留死代码 / 不报错的最小自查清单**：
1. 全局搜 `ServerApi.`、`reqUrl`、`defaultConfig.token`、`checkKey`、`get_msg`、`getVerifyCode`，逐一确认无残留调用。
2. 确认 `page.layx`（L896）的公告来源已替换。
3. 确认根路由 L783-L789 的验证码分支已被处理。
4. 确认 `requestMerge` 的所有 4 个调用方（L2814、L2854、L2913 + 补调 3 处）签名一致。
5. 确认 `String.prototype.cl`（L140-L142）未被依赖丢失（`answerFormat` L2568 仍在用）。

---

## 数据存储与外部依赖

### GM 存储

| 键 | 类型 | 读 | 写 | 说明 |
| --- | --- | --- | --- | --- |
| `"20230524"`（`cache_key`） | 对象（整个 `defaultConfig`） | L127 | L641、L1099 | 唯一持久化配置；无版本迁移逻辑，只回填 `token`（L128） |
| `"noticetemp"` | `{value,time}` | L896（经 L208-L216） | **从未写入** | 声明的 10 分钟公告缓存实际从未生效，每次 `layx()` 都会请求一次 `def/autoMsg` |
| 任意键（`utils.cache`） | `{value,time}` | L208-L216 | L204-L207 | 全文件无调用点（死代码） |

### 浏览器存储

| 存储 | 操作 | 行号 | 说明 |
| --- | --- | --- | --- |
| `localStorage`（`_self.localStorage`，即页面上下文） | 遍历并删除所有以 `layx_` 开头的键 | L3282-L3288 | 每次脚本加载（所有匹配页面）都执行，用于清理 layx 库的残留状态 |
| `sessionStorage` | **未使用** | — | 全文件无匹配 |
| `document.cookie` | 未直接读；通过 `_self.getCookie`（若页面未定义则注入空实现，L491-L495）读取 `UID`/`_uid` | L491-L495、L502、L524、L1362 | 通过 `unsafeWindow` 借用宿主页面的 cookie 读取能力 |

### 外部 HTTP 域名与接口

| 类别 | 目标 | 用途 | 行号 |
| --- | --- | --- | --- |
| 免费题库 | `http://cx.icodef.com/wyn-nb?v=4` | POST 检索答案 | L93（调用 L541-L559） |
| 免费题库来源页 | `https://cx.icodef.com/query.html` | 仅注释 | L92 |
| 付费题库 | `https://api.tikuhai.com/` | `search?` / `key` / `code` / `def/autoMsg` / `save1` | L117（路径 L527、L564、L580、L533、L3050 等） |
| 字体映射表 | `https://www.forestpolice.org/ttf/2.0/table.json` | 字体反爬映射 | L123 |
| 字体映射表 | `https://cdn.ocsjs.com/resources/font/table.json` | 同上 | L124 |
| 字体映射表 | `https://static.muketool.com/scripts/cx/v2/fonts/cxsecret.json` | 同上 | L125 |
| 超星（动态） | `_self.ServerHost.mooc1Domain` / `moocTJDomain` / `moocDomain` | 站内 API | L372、L386、L396、L401、L411、L418、L428、L433、L443、L810、L1252 |
| 超星（写死） | `https://mooc1.chaoxing.com/mycourse/transfer` | 章节跳转预热 | L1164 |
| 超星（写死） | `https://mooc1.chaoxing.com/mooc2/images/chapter_wancheng.png` | 引导卡片图标 | L842 |
| 超星（相对） | `location.origin + /mooc2-ans/mycourse/studentcourse`、`/ananas/...`、`/gas/...`、`/job/...`、`/knowledge/...`、`/question/...`、`/log/setlog` | 站内接口 | L406、L372-L443、L411 |
| 超星（写死参数） | `cpi=229749849`（出现在 `/knowledge/cards` URL 中） | 硬编码的课程 cpi | L401 |
| CDN 依赖 | `https://lib.baomitu.com/jquery/3.6.0/jquery.min.js` | jQuery | L45 |
| CDN 依赖 | `https://mirrors.sustech.edu.cn/cdnjs/ajax/libs/layx/2.5.4/layx.min.js` | layx UI 库 | L46 |
| CDN 依赖 | `https://mirrors.sustech.edu.cn/cdnjs/ajax/libs/blueimp-md5/2.19.0/js/md5.min.js` | md5（视频签名 + 字体哈希） | L47 |
| CDN 资源 | `.../layx/2.5.4/layx.min.css`、`.../layui/2.12.0/css/layui.min.css` | 样式（`GM_getResourceText` 注入，L612-L613） | L49-L50 |
| 更新源 | `https://update.greasyfork.org/scripts/435357/...` | `@downloadURL` / `@updateURL` | L53-L54 |
| 白名单外未声明 | `_self` 未列出的域名（如自定义题库） | 会触发 `connect list` 报错 | L342-L345 |

### 硬编码凭据 / 密钥 / 常量

| 内容 | 行号 | 类型 |
| --- | --- | --- |
| `'Authorization': '8l395BS2NoxMdDGH'` | L96 | **第三方题库 token（明文）** |
| 伪造 UA `ChaoXingStudy_3_5.1.4_android_phone_614_74 …` | L60 | 固定标识 |
| 视频签名盐 `d_yHJ!$pdA~5` | L189 | 算法常量（超星侧约定） |
| `cache_key = "20230524"` | L115 | 写死缓存键 |
| `cpi=229749849` | L401 | 写死请求参数 |
| 脚本 ID `435357` | L53-L54 | Greasy Fork 脚本 ID |

---

## 风险与脆弱点

| # | 风险 | 行号 | 为什么是风险 |
| --- | --- | --- | --- |
| 1 | **硬编码第三方 token** | L96 | 明文凭据随脚本分发；无法更换、无法撤销；一旦上游作废，免费链路整体失效；也把「谁在付费」暴露给任意读者 |
| 2 | **写死缓存键 + 整对象覆盖** | L115、L127-L129、L641、L1099 | `defaultConfig = cacheData \|\| defaultConfig` 会**丢弃所有新增默认值**；只有 `token` 被回填（L128）。改动配置结构后老用户读到脏配置，新字段为 `undefined`（如 `randomAnswer`、`aiAsk` 未定义时行为与默认相反） |
| 3 | **`unsafeWindow.postMessage` 猴子补丁** | L133-L139 | ① 改写宿主页面全局 API，与业务强耦合；② `msg.includes(...)`（L135）在页面用**对象**调用 `postMessage` 时抛 `TypeError`，可能破坏超星自身功能；③ 副作用不可见、卸载困难 |
| 4 | **`String.prototype.cl` 原型污染** | L140-L142、L2501、L2514、L2528、L2568 | 通过赋值给 `String.prototype` 添加**可枚举**属性，任何字符串对象的 `for...in`/`Object.keys` 都会多出 `cl`；与页面其它脚本/库冲突；且该方法是答案清洗的关键路径，被覆盖即静默出错 |
| 5 | **`ServerApi.defaultRequest` 失败时隐式返回 `undefined`** | L490-L517 | 所有调用方（L529、L540、L568、L584、L3050 等）默认拿到 response 对象；空值会在 `res.responseText` 处抛 `TypeError`，且 `page.init()` 顶层 catch 只调 `log()`（L3292），而 `log` 在 `debugger:false` 时是空函数（L130）→ **静默失败** |
| 6 | **`log()` 默认是空函数** | L130 | `defaultConfig.debugger` 默认 `false`（L77），导致 `page.init().catch(e => log(e))`（L3292-L3294）不产生任何可见输出；`log(defaultConfig)`（L644、L719、L768）等调试语句同理 |
| 7 | **`new Promise(async (resolve,reject)=>{...})` 反模式** | L542、L1211、L1288、L1334 | 异步执行器内的 `await` 抛错不会 reject 外层 Promise，而是产生**未处理的拒绝**并使外层 Promise **永不 settle** → `mainTask` 挂死、`threadWatch` 只能靠文本比对兜底 |
| 8 | **`finishVideo` 的 `while(true)` 无退出兜底** | L1311-L1329 | 终止只依赖 `time >= duration` 或 `result.isPassed`；若 `api.videoStudy` 一直返回 `isPassed:false` 且 `duration` 异常大（或 `videoSpeed` 被表单存成字符串 `""`），循环长时间运行；`videoSpeed` 已被 L1087-L1090 兜到 1，但其它字段无兜底 |
| 9 | **`duration` 为空时未 `return`** | L1306-L1308 | 只 `resolve(false)` 却继续执行后续循环，构造出 `duration: undefined` 的上报体 |
| 10 | **表单把 number 存成字符串** | L1069-L1071 | `defaultConfig.interval`/`videoSpeed`/`readSpeed`/`matchRate`/`autoSubmitRate` 保存后类型变化；`matchRate` 被清空为 `""` 时 L1091-L1094 的越界判断（`"" > 1`、`"" < 0` 均为 false）**不会纠偏**，随后 L252 的 `maxSim > ""`（即 `> 0`）使**任意选项都被判定匹配** |
| 11 | **答案回传第三方** | L3050-L3055、L3117-L3122、L3225-L3230、L3273-L3278 | 题目、正确答案、当前 URL 全部上传；隐私与合规风险；且未提供关闭开关 |
| 12 | **服务端可指定抓取 URL** | L3127-L3131、L3172-L3178 | `getScore2` 直接 `GM_xmlhttpRequest` 服务端返回的 `data.url`，并把结果 POST 到 `data.url1`；等于把「请求目标」交给远端（SSRF 面 + 可被用来做流量/数据中转） |
| 13 | **答案过滤黑名单作用于自家题库响应** | L106、L1373 | 见《题库链路取证》§3；任何含「一之」的答案被整条丢弃，可能造成**免费链路可用性下降**而无任何提示 |
| 14 | **题库单点依赖 + 静默降级** | L93、L117、L541-L559 | 免费链路只有一个地址（明文 http，L93），付费链路只有一个 base（L117）；两者任一不可用都只在状态栏显示文本，不重试、不告警 |
| 15 | **字体映射表依赖 3 个第三方站点** | L122-L125、L2464-L2467 | 全部失败则 `throw`，作业页直接放弃（L727-L732）；且映射表与字体版本解耦，**表过期时不会报错，只会静默替换成错误字符**（L2476-L2484 无校验） |
| 16 | **字体解码 O(20902) 次 md5** | L2469-L2475 | 每个 CJK 码点都要 `glyphToPath` + `JSON.stringify` + `md5`，在低端机上可能造成明显卡顿 |
| 17 | **`match[i]` 可能为 `undefined`** | L2474、L2480 | 映射表缺键时 `String.fromCharCode(undefined)` → 写入 `\u0000`，静默破坏题目文本 |
| 18 | **`_self.ServerHost` 未做存在性检查** | L372、L386、L396、L401、L411、L418、L428、L433、L443 | 依赖宿主页面注入的全局对象；页面结构变更（超星改版）会直接 `TypeError`，并被 L3292 的空 `log` 吞掉 |
| 19 | **`_self.UE` 未做存在性检查** | L1455、L1465、L1471、L1481、L2607、L2652、L2719、L2786 | 富文本编辑器全局对象缺失即抛错，答题中断 |
| 20 | **`window.parent.postMessage` 与 `_self.addEventListener` 的隐式配对** | L682/L687/L697/L752/L2958/L2972 与 L1335-L1350 | `finishWork` 靠**首条 message 事件**结束等待（L1336-L1348），任何无关的 message（含页面自身的对象消息）都可能被 `JSON.parse` 失败或提前 `resolve`；`JSON.parse(event.data)`（L1336）非字符串 data 会抛错 |
| 21 | **验证码流程可能永久挂起** | L275-L278、L284-L295 | `layx.prompt` 分支中 `resolve` 只在 `monitorVerify` 里被间接调用（L290），若用户关闭弹窗则 `api.defaultRequest` 的 `await` 永不返回；`api.getVerifyCode`（L354-L370）**没有 `onerror`**，请求失败时 Promise 永不 settle |
| 22 | **`checkKey` 空 key 只告警不返回** | L561-L563 | 注释语义是「空则停止检测」，实际仍会发请求（L568），逻辑与文案不符 |
| 23 | **`@connect` 未覆盖全部目标 + 提示行号过期** | L13-L34、L342-L345 | 报错文案让用户复制代码到「第19行」，而当前 `@connect` 段在 L13-L34，误导手工修复 |
| 24 | **`threadWatch` 直接 `location.reload()`** | L594-L609 | 判据是「悬浮窗文本未变且含特定串」，误判会导致刷新循环；`alert()`（L597）会阻塞页面 |
| 25 | **全局覆盖 `window.alert`** | L2969-L2971 | 正确率不足时把 `window.alert` 改成只打日志（而 `log` 默认静默，L130），页面后续所有 alert 被吞 |
| 26 | **200s 兜底 `setInterval` 不清除** | L2957-L2959 | 自动提交成功后该定时器仍存活，200s 后会向父窗口发「提交超时，已暂时关闭」，产生**误导性提示** |
| 27 | **`reqUrl.headers` 引用不存在** | L3053、L3120、L3228、L3276 | 传 `undefined`（靠默认参数兜住），但暴露了未完成的重构 |
| 28 | **配置校验不完整** | L1078-L1098 | 只校验 5 个字段的边界；`readSpeed` 可为任意值（L860 用于 `animate` 时长），极端值会让阅读任务瞬间结束或极慢 |
| 29 | **`localStorage` 无条件清理宿主页面键** | L3282-L3288 | 在所有匹配页面上删除 `layx_*` 键；若宿主页面恰好用同前缀存储，会被误删 |
| 30 | **`_self.getCookie` 注入宿主全局** | L491-L495 | 向 `unsafeWindow` 写入函数，污染宿主命名空间 |
| 31 | **`top` 被赋值为 `undefined`** | L115 | webpack 压缩产物：`top = (null && (_self))`，本意是屏蔽全局 `top` 引用；留在源码里只会误导读者 |
| 32 | **依赖 CDN `@require` 无完整性校验** | L45-L50 | 三个 CDN（含国内镜像 `mirrors.sustech.edu.cn`）任一被劫持/下线，脚本直接不可用 |
| 33 | **`@grant unsafeWindow` + 原型污染组合** | L38、L133-L142 | 补丁作用于宿主页面上下文（而非沙箱），影响面超出脚本本身 |
| 34 | **明文 HTTP 题库地址** | L93 | `http://cx.icodef.com/...` 可被中间人篡改（返回内容直接变成答案与 DOM 文本） |
| 35 | **`getScore2` 强依赖服务端下发结构** | L3125-L3131、L3176 | `data.url`/`data.url1`/`data.headers` 完全由服务端决定；服务端改协议即静默失效 |
| 36 | **`switch(true)` 长链** | L618-L881 | 分支多、条件互相包含（`/work/doHomeWorkNew` 与 `/mooc2/work/dowork` 等），新增路由易被前面的 `break` 遮蔽 |
| 37 | **`String.prototype.cl` 依赖前缀格式** | L140-L142 | 只能剥离 `【…】` 前缀与 `（x.x分）` 后缀；超星改文案格式即导致题干残留/误剥离，且无日志 |

---

## 修改入口建议

> 以下为「改哪里」的定位建议，不含具体改动代码。

### A. 只想改「默认配置」

| 目标 | 位置 |
| --- | --- |
| 字段默认值 | `defaultConfig` 定义块 **L59-L89** |
| 缓存键（**改任何配置结构都必须同时改这里**） | `cache_key = "20230524"` **L115** |
| 旧缓存字段回填 | **L128**（当前只回填 `token`，新增字段需在此补） |
| 面板默认显示值 | L901-L1016（`configForm` 各字段的 `value`） |
| 保存时校验/纠偏 | **L1078-L1098** |
| 面板字段 → 配置键的映射规则 | L1060-L1077（按 `name` 属性回写） |

### B. 只想改「界面」

| 目标 | 位置 |
| --- | --- |
| 悬浮窗默认外观（尺寸/位置/皮肤/注入 CSS） | L884-L895（`layx()` 的 `configs`），CSS 在 **L894** |
| 悬浮窗标题与状态栏/日志区 DOM 结构 | **L893、L897** |
| 配置面板全部 HTML 与 tab 样式 | L901-L1053（生成）、**L1117**（CSS） |
| 配置面板按钮（保存/取消） | L1056-L1109 |
| 日志渲染与行数上限 | L1120-L1129 |
| 状态栏文案 | L1130-L1132；各调用点如 L648、L736、L1207、L1285、L2890 |
| 考试页注入按钮 | L629-L647 |
| 作业/课程页「配置」按钮 | L665-L672、L714-L723、L763-L774 |
| 章节页引导卡片（文案/样式） | **L791-L853**（HTML 在 L817-L851） |
| 内联 SVG 图标 | **L126**（单行 3657 字符，勿手改） |
| 面板公告栏内容来源 | L896（`ServerApi.get_msg()`）与兜底 **L539**、默认文案 **L76** |

### C. 只想改「题库源」

| 目标 | 位置 |
| --- | --- |
| 免费题库条目（URL / 请求头 / 请求体 / 答案解析） | **L90-L114**（`otherApi`），token 在 **L96** |
| 付费题库 base URL | **L115-L120**（`reqUrl`） |
| 免费请求发送函数 | L541-L559（`ServerApi.searchOther`） |
| 付费检索与鉴权 | L518-L529（`search`）、L560-L578（`checkKey`） |
| 公共请求头（token/u/v/referer/t） | **L496-L503** |
| 多源聚合与回退顺序 | **L1359-L1449**（`requestMerge`），免费优先开关判断在 **L1367** |
| 调用方补调（免费失败→付费） | L2816-L2819、L2856-L2859、L2917-L2923 |
| 免费优先开关与 UI 文案 | 默认值 L74；面板 L996-L1001 |
| token 输入框 | L1002-L1007 |
| 答案黑名单过滤 | **L106** |
| 启动时的题库校验 | **L674** |
| 余额展示 | L1251、L2823、L2878、L2890、L2942、L2954 |
| 验证码 OCR 通道 | L579-L586 与调用处 L786 |
| 新增题库域名时补白名单 | `@connect` **L13-L34** |

### D. 只想改「核心学习流程」

| 目标 | 位置 |
| --- | --- |
| 章节遍历/闯关解锁/结束条件 | **L1133-L1209**（`mainTask`），闯关判定 L1143/L1145，未完成数 L1148，单章跳过 L1157-L1160 |
| 任务类型分发（video/document/workid） | **L1210-L1265**（`finishTask`），三个分支 L1218 / L1235 / L1245 |
| 视频进度上报循环与节奏 | **L1287-L1332**（`finishVideo`），循环 L1311-L1329，步长 L1321 |
| 视频签名算法 | **L188-L190**（`utils.getVideoEnc`，盐在 L189） |
| 视频元数据获取 | L427-L431（`getVideoConfig`） |
| 视频内嵌题 | L1266-L1286（`finishVideoAnswer`），答案取 `isRight` 于 L1273 |
| 文档任务 | L1235-L1244 + L417-L421（`docStudy`） |
| 学习记录上报 | L405-L416（`uploadStudyLog`），`encode` 抓取 L408 |
| 阅读时长页专项 | **L855-L866**（滚动动画与 `readSpeed`） |
| 作业答题 | L2832-L2891（`startWork`）与 L2616-L2660（`setWorkAnswer`） |
| 章节作业答题与自动提交 | **L2892-L2975**（`startChapter`），提交判定 L2956-L2974 |
| 考试答题 | L2812-L2831（`startAsk`）与 L2572-L2615（`setAnswer`） |
| 题库下发 URL 抓取回填 | L3125-L3181（`getScore2`） |
| 成绩回收 | L2976-L3056 / L3057-L3124 / L3182-L3231 / L3232-L3280 |
| 字体反爬解码 | **L1497-L2485**，核心在 L2449-L2484；映射表源 **L122-L125**；获取实现 L256-L267 |
| 路由分发（新增页面逻辑） | **L611-L882** |
| 题面抽取（DOM 选择器最易失效处） | **L2494-L2556**（`getQuestion` 三种结构） |
| 清空答案（切换题目/重试前） | L1450-L1491 |
