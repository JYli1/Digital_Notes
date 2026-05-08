# 主机发现
```bash
┌──(kali㉿kali)-[~]
└─$ nmap 10.216.75.0/24 -sn
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-08 12:58 CST
Nmap scan report for 10.216.75.183
Host is up (0.0070s latency).
MAC Address: 4E:20:31:25:4A:3C (Unknown)
Nmap scan report for 10.216.75.212
Host is up (0.0019s latency).
MAC Address: 30:E3:A4:48:AC:29 (Unknown)
Nmap scan report for 10.216.75.251
Host is up (0.00042s latency).
MAC Address: 08:00:27:A7:CC:59 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Nmap scan report for 10.216.75.81
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 10.80 seconds
```
靶机IP：`10.216.75.251`

# 信息收集
```md
┌──(kali㉿kali)-[~]
└─$ what

──────────────────────────────────────────────────
$ rustscan -a 10.216.75.251 --ulimit 5000 -- -A -sC -sV  (exit: 0)
──────────────────────────────────────────────────

结论

目标主机 10.216.75.251 是一台运行着 SSH、两个 HTTP 服务的 Linux 系统，其中 8080 端口标题暗示可能有备份系统或内部应用，值得深入探查。

关键发现


 端口      服务  版本/详情
 ────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
 22/tcp    SSH   OpenSSH 8.4p1 Debian 5+deb11u3，支持 RSA/ECDSA/ED25519 密钥
 80/tcp    HTTP  Apache 2.4.62 (Debian)，标题“CYBER MINESWEEPER - 赛博扫雷”，支持 GET/HEAD/POST/OPTIONS
 8080/tcp  HTTP  Apache 2.4.62 (Debian)，标题“Internal Backup System”，同样支持 GET/HEAD/POST/OPTIONS，且被检测到 http-open-proxy
                 风险（可能被重定向）


 • MAC 地址：08:00:27:A7:CC:59（Oracle VirtualBox），确认是虚拟靶机。
 • OS 猜测：Linux 4.15–5.19 / OpenWrt / RouterOS，但未完全确定。
 • HTTP 信息：80 端口标题为中文“赛博扫雷”，可能是一个游戏或挑战页面；8080 端口标题“Internal Backup
   System”暗示可能存在备份文件泄露或管理后台。



```
去web看看，这里`80`、`8080`都是web页面，扫了一下目录都没什么大收获
# 80端口
是三个扫雷游戏。
## LEVEL 01
很简单的`5*5`的扫雷，快速过了，得到
```
# pass1.zip
# 直接解压得到 pass1.txt

pass1：forget
```

## LEVEL 02
`10*10`的扫雷，不需要受挫了，前端游戏，直接把源码发给ai，写一个js脚本自动计算
```js
(async function autoWin() {
    const ROWS = 10, COLS = 10;
    const api = (action, extra = '') => fetch('game_level2.php', {
        method: 'POST',
        headers: {'Content-Type': 'application/x-www-form-urlencoded'},
        body: `action=${action}${extra ? '&' + extra : ''}`
    }).then(r => r.json());

    async function init() {
        const data = await api('init');
        if (!data.success) throw new Error('init failed');
        return data;
    }

    async function getState() {
        const data = await api('getState');
        // data: { success, revealed, flagged, minesLeft, values? }
        return data;
    }

    async function reveal(r, c) {
        return await api('reveal', `r=${r}&c=${c}`);
    }

    async function flag(r, c) {
        return await api('flag', `r=${r}&c=${c}`);
    }

    async function solve() {
        // 初始化（可能游戏已在进行，我们可以重新init）
        await init();
        while (true) {
            const state = await getState();
            if (!state.success) break;
            if (state.win) { // getState 可能返回 win？原更新板调用getState，但getState返回的数据中是否有win? 看代码 updateBoard 调用了 getState，但没用 win 字段。胜利是在 handleClick 的 reveal 返回中检测的。所以 getState 可能不返回 win。故我们需要在 reveal 时检查win。
                // 也许不需要在这里检查
            }
            // 构建数字矩阵和已揭示、旗子
            const numbers = Array.from({length: ROWS}, () => Array(COLS).fill(null));
            if (state.values) {
                for (const v of state.values) {
                    numbers[v.r][v.c] = v.value;
                }
            }
            const revealed = state.revealed; // 二维布尔
            const flagged = state.flagged; // 二维布尔
            let madeMove = false;

            // 遍历每个已揭示的数字格子
            for (let r = 0; r < ROWS; r++) {
                for (let c = 0; c < COLS; c++) {
                    if (!revealed[r][c]) continue;
                    const val = numbers[r][c];
                    if (val === null || val === -1) continue; // -1是地雷，游戏失败。避免。
                    // 统计周围8格中的未知格和旗子数
                    let unknown = 0, flags = 0;
                    for (let dr = -1; dr <= 1; dr++) {
                        for (let dc = -1; dc <= 1; dc++) {
                            if (dr === 0 && dc === 0) continue;
                            const nr = r + dr, nc = c + dc;
                            if (nr >= 0 && nr < ROWS && nc >= 0 && nc < COLS) {
                                if (revealed[nr][nc]) continue; // 已揭示忽略
                                if (flagged[nr][nc]) flags++;
                                else unknown++;
                            }
                        }
                    }
                    if (val === flags && unknown > 0) {
                        // 周围所有未知格安全，揭示它们
                        for (let dr = -1; dr <= 1; dr++) {
                            for (let dc = -1; dc <= 1; dc++) {
                                if (dr === 0 && dc === 0) continue;
                                const nr = r + dr, nc = c + dc;
                                if (nr >= 0 && nr < ROWS && nc >= 0 && nc < COLS) {
                                    if (!revealed[nr][nc] && !flagged[nr][nc]) {
                                        const res = await reveal(nr, nc);
                                        if (res.hitMine) {
                                            // 踩雷了，应该不会，因为逻辑是安全的，但可能由于后端与状态不同步？如果发生，返回false表示失败
                                            return false;
                                        }
                                        if (res.win) {
                                            // 赢了！
                                            return { win: true, hint: res.hint, token: res.downloadToken };
                                        }
                                        madeMove = true;
                                        // 更新状态以便继续？由于状态已改变，我们可以退出循环重新获取状态，但需要跳出两层循环。简单做法：设置madeMove并break。
                                    }
                                }
                            }
                            if (madeMove) break;
                        }
                    } else if (val - flags === unknown && unknown > 0) {
                        // 所有未知格是雷，标旗
                        for (let dr = -1; dr <= 1; dr++) {
                            for (let dc = -1; dc <= 1; dc++) {
                                if (dr === 0 && dc === 0) continue;
                                const nr = r + dr, nc = c + dc;
                                if (nr >= 0 && nr < ROWS && nc >= 0 && nc < COLS) {
                                    if (!revealed[nr][nc] && !flagged[nr][nc]) {
                                        await flag(nr, nc);
                                        madeMove = true;
                                    }
                                }
                            }
                            if (madeMove) break;
                        }
                    }
                    if (madeMove) break;
                }
                if (madeMove) break;
            }

            // 如果没有可进行的移动，且游戏未胜利(需要检查是否全部非雷已揭示，简单的办法是检查剩余未揭示且未标旗的格子数量是否等于 minesLeft? 若 minesLeft=0 且 未揭示格子全是旗子？实际上胜利条件是所有非雷格子被揭示。我们可以检查 revealed 中 false 且不是 flagged 的格子数目是否大于 minesLeft? 不是，minesLeft 是剩余地雷数，如果未揭示且未标旗格子数等于 minesLeft，可能所有未揭示都是雷，但如果那样，标旗后应自动胜利？不一定。最好还是尝试随机猜测。
            if (!madeMove) {
                // 猜测：收集所有未揭示且未标旗的格子
                const candidates = [];
                for (let r = 0; r < ROWS; r++) {
                    for (let c = 0; c < COLS; c++) {
                        if (!revealed[r][c] && !flagged[r][c]) {
                            candidates.push({r, c});
                        }
                    }
                }
                if (candidates.length === 0) {
                    // 应该已经赢了，但可能getState没有win，我们强制检查？
                    // 尝试再次 reveal 一个已揭示格子？不能。退出。
                    break;
                }
                // 随机选一个
                const pick = candidates[Math.floor(Math.random() * candidates.length)];
                const res = await reveal(pick.r, pick.c);
                if (res.hitMine) {
                    return false; // 失败
                }
                if (res.win) {
                    return { win: true, hint: res.hint, token: res.downloadToken };
                }
                // 继续循环
            }
        }
        // 如果退出循环，可能无解或胜利未捕获
        return false;
    }

    // 主循环尝试直到胜利
    let result = false;
    let attempt = 0;
    while (!result) {
        attempt++;
        console.log(`第 ${attempt} 次尝试`);
        result = await solve();
        if (result === false) {
            // 等待游戏重置？solve内部init会重置，但可能需额外等待。
            await new Promise(resolve => setTimeout(resolve, 500));
        }
    }
    if (result.win) {
        console.log('🎉 胜利!', result.hint);
        // 可以触发下载
        if (result.token) {
            window.location.href = 'download.php?id=2&token=' + encodeURIComponent(result.token);
        }
    }
})();
```

![](file-20260508215345235.png)
放到控制台就得到了`hint`和`pass2.zip`
pass2.zip有密码，并且没爆破出来。我们先去做`LEVEL 03`

## LEVEL 03
依旧ai搓个脚本，这次还得考虑到有不可预测的情况，ai还改了两次代码，改成计算+预测概率的版本。
```js
(async function() {
    const _alert = window.alert;
    window.alert = function(){};

    const delay = ms => new Promise(res => setTimeout(res, ms));
    const STEP_DELAY = 150;
    const RETRY_DELAY = 500;

    function clickSafe(r, c) {
        if (revealed[r][c] || flagged[r][c]) return;
        revealed[r][c] = true;
        if (board[r][c] === 0) expandZeros(r, c);
        renderBoard();
    }

    function flagCell(r, c) {
        if (revealed[r][c] || flagged[r][c]) return;
        flagged[r][c] = true;
        minesLeft--;
        renderBoard();
    }

    function checkVictory() {
        for (let r = 0; r < ROWS; r++)
            for (let c = 0; c < COLS; c++)
                if (!revealed[r][c] && board[r][c] !== -1) return false;
        return true;
    }

    function isDead() {
        for (let r = 0; r < ROWS; r++)
            for (let c = 0; c < COLS; c++)
                if (revealed[r][c] && board[r][c] === -1) return true;
        return false;
    }

    // 获取某个格子周围的已翻开数字列表
    function getNeighborNumbers(r, c) {
        const nums = [];
        for (let dr = -1; dr <= 1; dr++) {
            for (let dc = -1; dc <= 1; dc++) {
                if (dr === 0 && dc === 0) continue;
                const nr = r + dr, nc = c + dc;
                if (nr>=0 && nr<ROWS && nc>=0 && nc<COLS && revealed[nr][nc] && board[nr][nc] > 0) {
                    nums.push({r: nr, c: nc, val: board[nr][nc]});
                }
            }
        }
        return nums;
    }

    // 计算某个未知格是雷的概率（基于相邻数字约束）
    function calcMineProb(r, c) {
        const neighbors = getNeighborNumbers(r, c);
        if (neighbors.length === 0) {
            // 孤立格子，使用全局剩余雷密度
            let totalUnknown = 0;
            for (let r = 0; r < ROWS; r++)
                for (let c = 0; c < COLS; c++)
                    if (!revealed[r][c] && !flagged[r][c]) totalUnknown++;
            return totalUnknown > 0 ? minesLeft / totalUnknown : 1;
        }

        let minProb = 1;
        for (const nb of neighbors) {
            let unknownCount = 0;
            let flagCount = 0;
            for (let dr = -1; dr <= 1; dr++) {
                for (let dc = -1; dc <= 1; dc++) {
                    if (dr === 0 && dc === 0) continue;
                    const nr = nb.r + dr, nc = nb.c + dc;
                    if (nr>=0 && nr<ROWS && nc>=0 && nc<COLS) {
                        if (!revealed[nr][nc]) {
                            if (flagged[nr][nc]) flagCount++;
                            else unknownCount++;
                        }
                    }
                }
            }
            const needMines = nb.val - flagCount;
            if (needMines < 0) return 1; // 不可能，但作为保护
            const prob = unknownCount > 0 ? needMines / unknownCount : 1;
            if (prob < minProb) minProb = prob;
        }
        return minProb;
    }

    // 智能猜测：选择概率最小的格子
    function smartGuess() {
        const cands = [];
        let minProb = Infinity;
        for (let r = 0; r < ROWS; r++) {
            for (let c = 0; c < COLS; c++) {
                if (!revealed[r][c] && !flagged[r][c]) {
                    const prob = calcMineProb(r, c);
                    if (prob < minProb) {
                        minProb = prob;
                        cands.length = 0;
                        cands.push({r, c, prob});
                    } else if (Math.abs(prob - minProb) < 1e-9) {
                        cands.push({r, c, prob});
                    }
                }
            }
        }
        if (cands.length === 0) return false;
        // 在最低概率的格子中随机选一个
        const pick = cands[Math.floor(Math.random() * cands.length)];
        console.log(`🧩 猜测格子 (${pick.r},${pick.c})，雷概率 ≈ ${(pick.prob*100).toFixed(1)}%`);
        clickSafe(pick.r, pick.c);
        return true;
    }

    // 标准推理一步（同前）
    function safeStep() {
        for (let r = 0; r < ROWS; r++) {
            for (let c = 0; c < COLS; c++) {
                if (!revealed[r][c] || board[r][c] <= 0) continue;
                const val = board[r][c];
                let unknown = [];
                let flags = 0;
                for (let dr = -1; dr <= 1; dr++) {
                    for (let dc = -1; dc <= 1; dc++) {
                        if (dr === 0 && dc === 0) continue;
                        const nr = r + dr, nc = c + dc;
                        if (nr>=0 && nr<ROWS && nc>=0 && nc<COLS && !revealed[nr][nc]) {
                            if (flagged[nr][nc]) flags++;
                            else unknown.push({r: nr, c: nc});
                        }
                    }
                }
                if (val === flags && unknown.length > 0) {
                    unknown.forEach(p => clickSafe(p.r, p.c));
                    return true;
                }
                if (val - flags === unknown.length && unknown.length > 0) {
                    unknown.forEach(p => flagCell(p.r, p.c));
                    return true;
                }
            }
        }
        return false;
    }

    async function solve() {
        while (true) {
            if (isDead()) {
                console.log('💥 踩雷，重新开始...');
                initGame();
                await delay(RETRY_DELAY);
                continue;
            }
            if (checkVictory()) {
                gameOver = true;
                try {
                    const resp = await fetch('set_completion.php?level=3');
                    const data = await resp.json();
                    showWinModal(data.success ? data.hint : '');
                } catch (e) { showWinModal(''); }
                console.log('🎉 通关成功！');
                break;
            }
            let moved = safeStep();
            if (!moved) {
                console.log('🤔 无推理步，计算概率...');
                smartGuess();
            }
            await delay(STEP_DELAY);
        }
    }

    await solve();
    window.alert = _alert;
    if (document.getElementById('winModal').classList.contains('active')) {
        console.log('⬇️ 正在下载 pass4.zip ...');
        downloadKeyLegit();
    }
})();
```
这个考虑到太卡了，给改慢了一点，会看到它一个个去做，但是也挺快的，几分钟就好了
得到`pass4.zip`
还有hint：
```md
快去8080端口看看吧，，有好东西在那里，你会用ftp吗？

pass2:真的有加密吗，不会是假的吧
```
## 8080端口
直接得到`pass5.zip`

# ZIP解密
目前得到的信息：
* 3个加密zip包+一个直接解开的pass
* 3个hint：
```md
	pass4:爆密码是爆不出的，看看文件有多大
	快去8080端口看看吧，，有好东西在那里，你会用ftp吗？
	pass2:真的有加密吗，不会是假的吧
```
首先我想到了pass2会不会是伪加密：
## pass2
十六进制编辑器打开`pass2.zip`查看：
```md
┌──(kali㉿kali)-[~/tmp]
└─$ xxd pass2.zip
00000000: 504b 0304 1400 0800 0800 6b7f 8c5c 0000  PK........k..\..
00000010: 0000 0000 0000 0000 0000 0900 2000 7061  ............ .pa
00000020: 7373 322e 7478 7475 780b 0001 0400 0000  ss2.txtux.......
00000030: 0004 0000 0000 5554 0d00 07da 50db 69db  ......UT....P.i.
00000040: 50db 69c3 50db 692b 482c 2e36 7abf 6756  P.i.P.i+H,.6z.gV
00000050: 4946 2a00 504b 0708 3fa8 515c 0d00 0000  IF*.PK..?.Q\....
00000060: 0b00 0000 504b 0102 1403 1400 0900 0800  ....PK..........
00000070: 6b7f 8c5c 3fa8 515c 0d00 0000 0b00 0000  k..\?.Q\........
00000080: 0900 1800 0000 0000 0000 0000 b681 0000  ................
00000090: 0000 7061 7373 322e 7478 7475 780b 0001  ..pass2.txtux...
000000a0: 0400 0000 0004 0000 0000 5554 0500 01da  ..........UT....
000000b0: 50db 6950 4b05 0600 0000 0001 0001 004f  P.iPK..........O
000000c0: 0000 0064 0000 0000 00                   ...d.....
```

这里涉及到ZIP文件结构。直接问了ai，知道了就是存在伪加密
```bash
┌──(kali㉿kali)-[~/tmp]
└─$ what -q "这个zip文件结构有什么问题吗"

──────────────────────────────────────────────────
$ xxd pass2.zip  (exit: 0)
──────────────────────────────────────────────────

结论

该 ZIP 文件存在伪加密：本地文件头与中央目录中的加密标志位不一致，导致文件被标记为加密但实际可能未加密。

关键发现

 1 本地文件头（偏移 0x06）标志位为 0x0008 → bit3 置位（启用数据描述符），bit0 未置位（无加密）。
 2 中央目录条目（偏移 0x70）标志位为 0x0009 → bit0 置位（标记加密），bit3 也置位。
 3 伪加密通常通过修改中央目录或本地文件头的加密位实现，导致解压工具要求输入密码，但实际压缩数据并未加密。
 4 压缩方式为 0x0008（Deflate），压缩数据大小 13 字节，未压缩大小 11 字节，文件名 pass2.txt，结构其他部分正常。

下一步建议

 • 手动修复标志位：用十六进制编辑器将中央目录偏移 0x70 处的 09 00 改为 08 00，然后正常解压。
 • 直接解压绕过：尝试使用 unzip -P "" pass2.zip（密码为空）或 7z x pass2.zip（自动检测伪加密）。
 • 详细检查：使用 zipdetails 或 zipinfo -v pass2.zip 查看标志细节，确认是否有其他隐藏修
```
按照ai说的修改一下十六进制的加密位然后解压就可得到了。
我这里试一下他说的`7z`直接检测
```bash
┌──(kali㉿kali)-[~/tmp]
└─$ what

──────────────────────────────────────────────────
$ 7z x pass2.zip  (exit: 2)
──────────────────────────────────────────────────

一句话结论

pass2.zip 的头信息已损坏，但 7z 仍将其识别为 ZIP 并成功解压出 pass2.txt，需要注意的是 Headers Error
可能暗示文件被故意篡改或有隐写信息。

关键发现

 • 退出码 2：表示命令执行过程中出现严重错误（文件格式错误）。
 • Headers Error：Zip 文件局部或中心目录头校验失败，常见于文件截断、加密或手工构造的伪 zip。
 • 成功解压：尽管报错，7z 仍提取了 pass2.txt（11 字节），说明文件结构基本完整，仅头信息不合法。
 • 文件时间：Modified: 2026-04-12 15:59:22，时间在未来，可能为故意设置或线索指向某个日期。

下一步建议

 1 立即查看 pass2.txt 内容：可能直接包含密码或其他关键信息。
 2 检查 zip 文件是否隐藏其他数据：使用 binwalk 或 hexdump 分析末尾是否有附加数据（如伪加密或隐写）。
 3 尝试修复 zip 头：若文件内容不完整，可用 zip -F 或 zip -FF 恢复中心目录。

最短可执行修复命令


 # 查看解压出的文件内容
 cat pass2.txt
 # 同时检查原始 zip 文件的十六进制结构（末尾 100 字节）
 xxd pass2.zip | tail -20
 # 如果文件需修复，尝试重建中心目录（备份原文件后执行）
 zip -FF pass2.zip --out fixed.zip && 7z x fixed.zip

┌──(kali㉿kali)-[~/tmp]
└─$ cat pass2.txt
pass2：the
```
工具直接检测到头信息有错误，但是还是解压成功了，我们成功拿到`pass2`
## pass4 && pass5
我们注意到之前的hint：
```md
Hint: pass4:爆密码是爆不出的，看看文件有多大
```
我们看一看pass4的大小
```bash
-rw-rw-r-- 1 kali kali  168  5月 8日 13:43 pass4.zip
-rw-rw-r-- 1 kali kali  151  5月 8日 13:11 pass5.hash
-rw-rw-r-- 1 kali kali  172  4月10日 15:54 pass5.zip
-rw-rw-r-- 1 root root 3.0M 2023年 1月18日 pspy64
```
非常小，那也就是说内容应该就是一个txt文件里面几个字母，就像`pass1：forget`一样
那么这里有一个方法，也是爆破，但是不是爆破密码。
而是进行CRC碰撞
```python
import binascii
import itertools
import string
import time

target_crc = 0x66b9a733
length = 4

# 假设内容是字母和数字 (如果跑不出来，可以加上 string.punctuation 包含符号)
chars = string.ascii_letters + string.digits

# 计算总可能组合数
total_combinations = len(chars) ** length

print(f"[*] 目标 CRC32: {hex(target_crc)}")
print(f"[*] 字符集大小: {len(chars)} 个字符")
print(f"[*] 猜测长度: {length}")
print(f"[*] 总计需尝试: {total_combinations} 次")
print("[*] 开始爆破，请稍候...\n")

start_time = time.time()
count = 0

for p in itertools.product(chars, repeat=length):
    count += 1
    
    # 每 10 万次刷新一次屏幕，防止 IO 拖慢速度
    if count % 100000 == 0:
        percent = (count / total_combinations) * 100
        # \r 表示回到行首，end="" 表示不换行
        print(f"\r[~] 进度: {percent:>5.2f}% | 当前尝试: {''.join(p)} | 已跑: {count}/{total_combinations}", end="")

    text = ''.join(p).encode('ascii')
    
    if binascii.crc32(text) == target_crc:
        end_time = time.time()
        print(f"\n\n[+] ===========================================")
        print(f"[+] 成功！找到原文内容了: {text.decode('ascii')}")
        print(f"[+] 耗时: {end_time - start_time:.2f} 秒")
        print(f"[+] ===========================================\n")
        exit(0)

print("\n\n[-] 爆破结束，未找到匹配项。可能包含特殊字符？")
```

```bash
┌──(kali㉿kali)-[~/tmp]
└─$ python3 zip.py
[*] 目标 CRC32: 0x66b9a733
[*] 字符集大小: 62 个字符
[*] 猜测长度: 4
[*] 总计需尝试: 14776336 次
[*] 开始爆破，请稍候...

[~] 进度: 59.55% | 当前尝试: K5rD | 已跑: 8800000/14776336

[+] ===========================================
[+] 成功！找到原文内容了: LiVe
[+] 耗时: 5.69 秒
[+] ===========================================
```
直接爆破到了，同时我们注意到pass5的长度也只有6字节，同样应该可以爆破，但是这个数量级大了很多，我们直接用c语言写
，这里注意到一点
```html
<h1>[ 内部备份服务器 - 仅限授权人员 ][a-z]</h1>
```
`8080`端口这里有一个【a-z】,是不是按时我们内容只包括小写字母，这里缩减了很多范围，加速爆破。我们试试。
```c
#include <stdio.h>
#include <string.h>
#include <zlib.h>

int main() {
    // 目标 CRC32
    unsigned long target = 0xea86a014;
    // 包含大小写和数字
    char chars[] = "abcdefghijklmnopqrstuvwxyz";
    int len = strlen(chars);
    char buf[7] = {0}; // 6个字符 + 1个结束符

    printf("[*] 开始 C 语言极限爆破...\n");

    for(int i=0; i<len; i++) {
        buf[0] = chars[i];
        printf("[~] 正在运算以 %c 开头的组合...\n", chars[i]);
        for(int j=0; j<len; j++) {
            buf[1] = chars[j];
            for(int k=0; k<len; k++) {
                buf[2] = chars[k];
                for(int l=0; l<len; l++) {
                    buf[3] = chars[l];
                    for(int m=0; m<len; m++) {
                        buf[4] = chars[m];
                        for(int n=0; n<len; n++) {
                            buf[5] = chars[n];

                            // 核心 CRC 计算
                            if (crc32(0L, (const unsigned char*)buf, 6) == target) {
                                printf("\n[+] ============================\n");
                                printf("[+] 成功找到内容: %s\n", buf);
                                printf("[+] ============================\n");
                                return 0;
                            }
                        }
                    }
                }
            }
        }
    }
    printf("[-] 未找到匹配项。\n");
    return 0;
}
```
编译运行
```bash
┌──(kali㉿kali)-[~/tmp]
└─$ gcc zip.c -O3 -lz -o zip
#-O3 开启最高性能优化，-lz 链接 zlib 库

┌──(kali㉿kali)-[~/tmp]
└─$ ./zip
[*] 开始 C 语言极限爆破...
[~] 正在运算以 a 开头的组合...
[~] 正在运算以 b 开头的组合...
[~] 正在运算以 c 开头的组合...
[~] 正在运算以 d 开头的组合...
[~] 正在运算以 e 开头的组合...
[~] 正在运算以 f 开头的组合...
[~] 正在运算以 g 开头的组合...
[~] 正在运算以 h 开头的组合...
[~] 正在运算以 i 开头的组合...
[~] 正在运算以 j 开头的组合...
[~] 正在运算以 k 开头的组合...
[~] 正在运算以 l 开头的组合...
[~] 正在运算以 m 开头的组合...
[~] 正在运算以 n 开头的组合...
[~] 正在运算以 o 开头的组合...
[~] 正在运算以 p 开头的组合...
[~] 正在运算以 q 开头的组合...
[~] 正在运算以 r 开头的组合...
[~] 正在运算以 s 开头的组合...
[~] 正在运算以 t 开头的组合...

[+] ============================
[+] 成功找到内容: thenow
[+] ============================
```
`这里如果没注意到小写字母的话是会爆破出另外的结果：eYOMhu 的`
，现在我们就有了`forget the LiVe thenow`，这里差一个pass3，但是没找到这个压缩包，但是可以知道应该是这样：
```md
`forget the {unkown}  LiVe thenow`
```
英文不太好，依旧ai
![](file-20260509001538673.png)
密码就是`forgetthepastLiVethenow`
我们去打开`宝箱.zip`
里面有一个`flag`文件
打开是：
```md
这里啥都没有，但是这个目录里不只一个文件
```
意识到有隐藏的文件了。
# flag.vhd解密
