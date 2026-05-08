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
