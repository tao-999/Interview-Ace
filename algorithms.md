# 前端算法

## 1) 两数之和（哈希表）

**问题**  
给定 `nums: number[]` 与 `target: number`，返回两数下标 `i, j`（`i<j`）使 `nums[i]+nums[j]=target`。若不存在返回 `null`。元素不可重复使用。

**原理与算法**  
- 一次遍历，维护哈希 `seen: Map<值, 下标>`。对每个 `x=nums[i]`，检查 `target-x` 是否已出现。  
- **不变量**：遍历到 `i` 时，`seen` 中存放的都是 `< i` 的元素，它们与 `nums[i]` 形成的候选对已被检查。  
- **正确性**：设答案为 `(p,q)` 且 `p<q`。当遍历到 `i=q` 时，`p<q` 已存入 `seen`，故能命中返回。  
- **复杂度**：时间 `O(n)`，空间 `O(n)`。

**实现（TypeScript）**
```ts
export function twoSum(nums: number[], target: number): [number, number] | null {
  const seen = new Map<number, number>(); // value -> index
  for (let i = 0; i < nums.length; i++) {
    const x = nums[i];
    const need = target - x;
    if (seen.has(need)) return [seen.get(need)!, i];
    // 若要求返回最左配对，可保持首次出现的下标：只有未见过时再 set
    if (!seen.has(x)) seen.set(x, i);
  }
  return null;
}
```

**易错点**  
- 不能用同一元素两次（如 `target=2x` 时，需确保另一个 `x` 已出现）。  
- 若题目要求返回**所有**不重复对，需要排序 + 去重或用集合记忆（与此题不同）。

---

## 2) 三数之和（排序 + 双指针，含去重）

**问题**  
给定整型数组，找出所有和为 `0` 的**不重复**三元组（`a≤b≤c` 语义下的集合去重）。

**原理与算法**  
- 先排序 `O(n log n)`。枚举首元素 `i`，目标变为在 `(i+1..n-1)` 中找两数和为 `-nums[i]`。  
- 双指针 `l=i+1, r=n-1`：若 `sum<0` 则 `l++`，若 `sum>0` 则 `r--`，等于 `0` 记录答案并**跳过重复值**。  
- **去重要点**：  
  - 外层：当 `nums[i]===nums[i-1]` 时跳过。  
  - 内层：命中一次后，`while(nums[l]==nums[l-1]) l++` 与 `while(nums[r]==nums[r+1]) r--`。  
- **剪枝**：若 `nums[i]>0`，之后不可能再有解；若 `nums[n-1]<0`，无解。  
- **复杂度**：排序 `O(n log n)` + 双指针 `O(n^2)`，总 `O(n^2)`。

**实现**
```ts
export function threeSum(nums: number[]): number[][] {
  nums.sort((a, b) => a - b);
  const n = nums.length, ans: number[][] = [];
  for (let i = 0; i < n; i++) {
    if (i > 0 && nums[i] === nums[i - 1]) continue; // 去重
    const a = nums[i];
    if (a > 0) break; // 剪枝
    let l = i + 1, r = n - 1;
    while (l < r) {
      const s = a + nums[l] + nums[r];
      if (s < 0) l++;
      else if (s > 0) r--;
      else {
        ans.push([a, nums[l], nums[r]]);
        do { l++; } while (l < r && nums[l] === nums[l - 1]);
        do { r--; } while (l < r && nums[r] === nums[r + 1]);
      }
    }
  }
  return ans;
}
```

**易错点**  
- 忘记三处去重（`i`、`l`、`r`）。  
- 全是 0 的情况应返回 `[0,0,0]` 一次。  
- JS 不溢出整型，但其他语言需要注意加法溢出（可用 `long`）。

---

## 3) 无重复字符的最长子串（滑动窗口）

**问题**  
返回字符串中不含重复字符的最长子串长度（或索引区间）。

**原理与算法**  
- 窗口 `[L..R]` 维护**无重复**不变量；遇到重复字符 `c`，将 `L` **跳到** `last[c]+1`（而不是 `L+1`）。  
- `last: Map<char, 最右出现位置>`；每步更新 `ans = max(ans, R-L+1)`。  
- **正确性**：`L` 单调不减，每个字符下标最多入/出窗口一次，总 `O(n)`。  
- **复杂度**：时间 `O(n)`，空间 `O(Σ)`（字符集大小）。

**实现（返回长度与区间）**
```ts
export function lengthOfLongestSubstring(s: string): { len: number; range: [number, number] } {
  const last = new Map<string, number>();
  let L = 0, bestLen = 0, best: [number, number] = [0, -1];
  for (let R = 0; R < s.length; R++) {
    const c = s[R];
    if (last.has(c)) L = Math.max(L, last.get(c)! + 1);
    last.set(c, R);
    if (R - L + 1 > bestLen) { bestLen = R - L + 1; best = [L, R]; }
  }
  return { len: bestLen, range: best };
}
```

**易错点与扩展**  
- **Unicode**：JS 字符串是 UTF-16 码元；如需按**用户感知字符（字素）**，需 `Intl.Segmenter` 或库处理。  
- **窗口推进**必须用 `Math.max`，保证 `L` 不回退。  
- 若需返回子串本身：`s.slice(best[0], best[1] + 1)`。

---

## 4) 最小覆盖子串（滑动窗口 + 计数）

**问题**  
给定字符串 `s` 与模式 `t`，求最短子串 `s[l..r]` 使其包含 `t` 中每个字符及其**需求次数**（区分大小写）。若不存在返回空串。

**原理与算法（典型“可行→收缩”模板）**  
- 预处理 `need[c] = 在 t 中需要的次数`，`needKinds = 需求的不同字符数`。  
- 扩张右端 `r`：若 `c∈need`，递增 `have[c]`，当 `have[c] == need[c]` 时 `formed++`。  
- 当 `formed == needKinds`（覆盖可行）时，尝试移动左端 `l` **收缩**并更新最优；若收缩使某字符不足，`formed--`，再继续扩张。  
- **正确性**：每个指针只前进不后退，总 `O(|s|+|t|)`。  
- **复杂度**：时间 `O(n)`，空间 `O(Σ)`。

**实现**
```ts
export function minWindow(s: string, t: string): string {
  if (t.length === 0 || s.length < t.length) return "";
  const need = new Map<string, number>();
  for (const c of t) need.set(c, (need.get(c) ?? 0) + 1);
  const have = new Map<string, number>();
  let formed = 0, needKinds = need.size;
  let bestLen = Infinity, best: [number, number] = [0, 0];

  for (let l = 0, r = 0; r < s.length; r++) {
    const cr = s[r];
    if (need.has(cr)) {
      const v = (have.get(cr) ?? 0) + 1;
      have.set(cr, v);
      if (v === need.get(cr)) formed++;
    }
    while (formed === needKinds) {
      if (r - l + 1 < bestLen) { bestLen = r - l + 1; best = [l, r]; }
      const cl = s[l++];
      if (need.has(cl)) {
        const v = (have.get(cl) ?? 0) - 1;
        have.set(cl, v);
        if (v < need.get(cl)!) formed--;
      }
    }
  }
  return bestLen === Infinity ? "" : s.slice(best[0], best[1] + 1);
}
```

**易错点**  
- `formed` 的定义是“**恰好达标**的种类数”，不是所有计数之和。  
- `have` 可能超过 `need`，只有从**超过**降到**不足**时才使 `formed--`。  
- 若 `t` 包含 `s` 中不存在的字符，直接返回空串。

---

## 5) 最长回文子串（中心扩展 / Manacher 对比）

**问题**  
返回字符串中最长的**回文子串**（连续）。

### 方案 A：中心扩展（易写，`O(n^2)`）

**原理**  
- 回文关于中心对称。中心可为单字符（奇数长度）或相邻两字符间（偶数长度）。  
- 对每个中心向两侧扩展直到字符不相等。记录最长区间。  
- **不变量**：扩展时始终满足 `s[l..r]` 是回文；每次比较失败后退出，故每个中心的扩展次数受回文长度限制。

**实现**
```ts
export function longestPalindromeCenter(s: string): string {
  if (s.length < 2) return s;
  let best: [number, number] = [0, 0];

  const expand = (L: number, R: number) => {
    while (L >= 0 && R < s.length && s[L] === s[R]) { L--; R++; }
    // 退出时 s[L+1..R-1] 为回文
    const l = L + 1, r = R - 1;
    if (r - l > best[1] - best[0]) best = [l, r];
  };

  for (let i = 0; i < s.length; i++) {
    expand(i, i);       // 奇数
    expand(i, i + 1);   // 偶数
  }
  return s.slice(best[0], best[1] + 1);
}
```

**复杂度**：最坏 `O(n^2)`，空间 `O(1)`。

**细节**  
- 若需索引，返回 `[l, r]` 即可。  
- UTF-16 字符同样问题：按码元比较；若需按字素，需额外分段。

### 方案 B：Manacher（线性 `O(n)`，更复杂）

**思想**  
- 通过在字符间插入分隔符（如 `#`）把奇偶回文统一为奇回文；对变换后的串 `T` 维护回文半径数组 `P[i]`。  
- 维护当前**最右回文右边界** `R` 及其中心 `C`。遍历位置 `i` 时，若 `i<R`，可利用对称点 `mirror = 2C - i` 的半径 `P[mirror]` 作为下界（`min(P[mirror], R-i)`），再向两侧扩展。更新 `C, R` 与最佳答案。  
- **不变量**：`R` 始终是已知回文的最右边界；`P[i]` 至少为镜像下界。

**实现（紧凑版）**
```ts
export function longestPalindromeManacher(s: string): string {
  if (s.length < 2) return s;
  // 1) 预处理：^ 和 $ 作为边界哨兵，避免越界判断
  const T = '^#' + s.split('').join('#') + '#$';
  const n = T.length, P = new Array<number>(n).fill(0);
  let C = 0, R = 0, bestCenter = 0;

  for (let i = 1; i < n - 1; i++) {
    const mir = 2 * C - i;
    if (i < R) P[i] = Math.min(R - i, P[mir]);
    // 尝试扩展
    while (T[i + 1 + P[i]] === T[i - 1 - P[i]]) P[i]++;
    if (i + P[i] > R) { C = i; R = i + P[i]; }
    if (P[i] > P[bestCenter]) bestCenter = i;
  }
  // 2) 从变换串坐标映射回原串坐标
  const rad = P[bestCenter];          // 在 T 上的半径
  const startInT = bestCenter - rad;  // T 上的左端
  const startInS = Math.floor((startInT - 1) / 2);
  return s.substring(startInS, startInS + rad);
}
```

**复杂度**：时间 `O(n)`，空间 `O(n)`。

**何时选哪种**  
- 绝大多数业务面试：中心扩展足够、易写且不易错。  
- 对极端长串/性能敏感：Manacher 更优，但实现成本更高。

---
## 6) 最长回文子序列（LPS，DP）

**问题**  
给定字符串 `s`，返回其最长**回文子序列**的长度（注意：子序列非连续）。

**原理（区间 DP）**  
- 定义 `dp[i][j]`：`s[i..j]` 的最长回文子序列长度。  
- 递推：
  - `s[i] === s[j]` → `dp[i][j] = dp[i+1][j-1] + 2`
  - 否则 `dp[i][j] = max(dp[i+1][j], dp[i][j-1])`
- 边界：`dp[i][i] = 1`；当 `i>j` 视为 0。按**子串长度从小到大**填表。  
- **正确性**：区间递推保证子问题严格小于原问题（`i+1..j-1`、`i+1..j`、`i..j-1`），不存在环依赖。  
- 复杂度：时间 `O(n^2)`，空间 `O(n^2)`；可滚动一维降到 `O(n)`（按右端 `j` 递增）。

**实现（TypeScript，返回长度）**
```ts
export function longestPalindromeSubseqLen(s: string): number {
  const n = s.length;
  if (n <= 1) return n;
  const dp = Array.from({ length: n }, () => new Array<number>(n).fill(0));
  for (let i = 0; i < n; i++) dp[i][i] = 1;
  for (let len = 2; len <= n; len++) {
    for (let i = 0; i + len - 1 < n; i++) {
      const j = i + len - 1;
      if (s[i] === s[j]) dp[i][j] = (len === 2) ? 2 : dp[i + 1][j - 1] + 2;
      else dp[i][j] = Math.max(dp[i + 1][j], dp[i][j - 1]);
    }
  }
  return dp[0][n - 1];
}
```

**重建一个解（可选）**  
- 从 `(i=0,j=n-1)` 回溯：  
  - 若 `s[i]===s[j]` 且 `dp[i][j]===dp[i+1][j-1]+2`，把 `s[i]` 放两端，`i++`,`j--`；  
  - 否则走向较大的那个子问题。  
- 注意存在多解，回溯返回任意一个即可。

**补充**  
- `LPS(s)` 的长度等于 `LCS(s, reverse(s))` 的长度，但**得到的序列可能不同**（依赖具体回溯路径）。


---

## 7) 编辑距离（Levenshtein，DP）

**问题**  
将字符串 `a` 转成 `b` 的最少编辑次数，允许操作：插入、删除、替换（代价均为 1）。

**原理（二维 DP）**  
- `dp[i][j]`：`a[0..i)` 到 `b[0..j)` 的最小代价。  
- 转移：
  - 删除 `a[i-1]`：`dp[i-1][j] + 1`
  - 插入 `b[j-1]`：`dp[i][j-1] + 1`
  - 替换/匹配：`dp[i-1][j-1] + (a[i-1]===b[j-1] ? 0 : 1)`
  - 取三者最小。  
- 初值：`dp[i][0]=i`，`dp[0][j]=j`。  
- 复杂度：时间 `O(mn)`，空间 `O(mn)`；可用一维滚动到 `O(min(m,n))`。

**实现（长度即可，TypeScript，一维滚动）**
```ts
export function editDistance(a: string, b: string): number {
  const m = a.length, n = b.length;
  if (m === 0) return n; if (n === 0) return m;
  const dp = new Array<number>(n + 1);
  for (let j = 0; j <= n; j++) dp[j] = j; // dp of i=0
  for (let i = 1; i <= m; i++) {
    let prevDiag = dp[0]; // dp[i-1][j-1]
    dp[0] = i;            // dp[i][0]
    for (let j = 1; j <= n; j++) {
      const tmp = dp[j];  // 保存 dp[i-1][j]
      const cost = a[i - 1] === b[j - 1] ? 0 : 1;
      dp[j] = Math.min(
        dp[j] + 1,         // 删除 a[i-1]
        dp[j - 1] + 1,     // 插入 b[j-1]
        prevDiag + cost    // 替换/匹配
      );
      prevDiag = tmp;
    }
  }
  return dp[n];
}
```

**扩展**  
- **Damerau–Levenshtein**：允许相邻转置，额外状态或哈希表记录最近出现位置。  
- **阈值剪枝**：若知上界 `k`，可只计算 `|i-j|≤k` 的带状 DP，超出返回 `>k`。  
- **路径重建**：保存方向指针或从 `dp[m][n]` 逆推。


---

## 8) 最长公共子序列 LCS（DP）

**问题**  
给定序列 `a`、`b`，求二者的最长公共子序列长度/序列。

**原理（经典 DP）**  
- `dp[i][j]`：`a[0..i)` 与 `b[0..j)` 的 LCS 长度。  
- 转移：
  - 若 `a[i-1]===b[j-1]`：`dp[i][j]=dp[i-1][j-1]+1`
  - 否则：`dp[i][j]=max(dp[i-1][j], dp[i][j-1])`  
- 初值：`dp[0][*]=dp[*][0]=0`。  
- 复杂度：时间 `O(mn)`，空间 `O(mn)`（可滚动为两行 `O(min(m,n))`）。

**实现（返回长度 + 重建一个解）**
```ts
export function lcs(a: string, b: string): { len: number; seq: string } {
  const m = a.length, n = b.length;
  const dp = Array.from({ length: m + 1 }, () => new Array<number>(n + 1).fill(0));
  for (let i = 1; i <= m; i++) {
    for (let j = 1; j <= n; j++) {
      dp[i][j] = (a[i - 1] === b[j - 1])
        ? dp[i - 1][j - 1] + 1
        : Math.max(dp[i - 1][j], dp[i][j - 1]);
    }
  }
  // 回溯重建
  let i = m, j = n; const stack: string[] = [];
  while (i > 0 && j > 0) {
    if (a[i - 1] === b[j - 1]) { stack.push(a[i - 1]); i--; j--; }
    else if (dp[i - 1][j] >= dp[i][j - 1]) i--;
    else j--;
  }
  return { len: dp[m][n], seq: stack.reverse().join('') };
}
```

**补充**  
- 若 `b` 是 `a` 的**去重唯一排列**，可把 `b` 中元素映射到在 `a` 的位置序列，问题可降为 **LIS（O(n log n)**）。  
- LCS 与 `diff`/合并冲突解析密切相关（ Myers 算法是另一条路线）。


---

## 9) 乘积数组除自身（前后缀积，O(1) 额外空间）

**问题**  
给定 `nums`，返回 `ans`，其中 `ans[i] = ∏_{j≠i} nums[j]`。不允许使用除法，时间 `O(n)`，仅 `O(1)` 额外空间（不计输出）。

**原理**  
- `prefix[i] = nums[0]*...*nums[i-1]`；`suffix[i] = nums[i+1]*...*nums[n-1]`。  
- 单数组做法：  
  1）先写入 `ans[i]=prefix`（从左到右维护累积 `pre`）；  
  2）再从右到左维护 `suf`，令 `ans[i]*=suf`，并更新 `suf*=nums[i]`。  
- **正确性**：`ans[i]` 最终为左侧积×右侧积。  
- 复杂度：时间 `O(n)`，空间 `O(1)`。

**实现（TypeScript）**
```ts
export function productExceptSelf(nums: number[]): number[] {
  const n = nums.length, ans = new Array<number>(n);
  let pre = 1;
  for (let i = 0; i < n; i++) { ans[i] = pre; pre *= nums[i]; }
  let suf = 1;
  for (let i = n - 1; i >= 0; i--) { ans[i] *= suf; suf *= nums[i]; }
  return ans;
}
```

**零值处理（若额外要求精确）**  
- 若出现 0 的个数：  
  - `>1` → 全 0；  
  - `=1` → 只有 0 位置的结果为“非零元素之积”，其余为 0。  
- 上述双遍法**天然正确**，无需特判（乘到 0 的路径会被清零）。


---

## 10) 旋转数组中的搜索（二分与边界判定）

**问题**  
给定升序数组经某点旋转（无重复），在 `O(log n)` 时间判断是否存在 `target` 并返回下标，否则 `-1`。

**原理（有序半段判定 + 二分）**  
- 每次二分取 `mid`，总有一半区间是**有序**的。  
- 若 `nums[mid] >= nums[l]`，左半 `[l..mid]` 有序：  
  - 若 `nums[l] ≤ target < nums[mid]`，则在左半；否则在右半。  
- 否则右半 `[mid..r]` 有序：  
  - 若 `nums[mid] < target ≤ nums[r]`，则在右半；否则在左半。  
- **不变量**：每次缩小到必然包含答案的半边，直至 `l>r`。  
- 复杂度：`O(log n)`。

**实现（TypeScript）**
```ts
export function searchRotated(nums: number[], target: number): number {
  let l = 0, r = nums.length - 1;
  while (l <= r) {
    const m = (l + r) >> 1;
    if (nums[m] === target) return m;
    if (nums[m] >= nums[l]) { // 左半有序
      if (nums[l] <= target && target < nums[m]) r = m - 1;
      else l = m + 1;
    } else { // 右半有序
      if (nums[m] < target && target <= nums[r]) l = m + 1;
      else r = m - 1;
    }
  }
  return -1;
}
```

**变体（存在重复元素）**  
- 当 `nums[l]===nums[m]===nums[r]` 时无法判断哪边有序，可**两端缩一格**：`l++`, `r--`，最坏退化为 `O(n)`。  
- 若需找**旋转点**（最小值下标），可用另一套二分：比较 `nums[m]` 与 `nums[r]`，决定向右/左收缩。

---
## 11) 在有序数组中查找元素的第一个/最后一个位置（二分边界）

**问题**  
给定非降序数组 `nums` 与值 `target`，返回 `[firstIndex, lastIndex]`。不存在返回 `[-1,-1]`。

**原理（两次单调二分）**  
- **左边界（lower_bound）**：返回第一个 `nums[i] >= target` 的位置；若 `nums[i]===target` 则为首次出现。  
- **右边界（upper_bound）**：返回第一个 `nums[i] > target` 的位置；`lastIndex = upper_bound(target)-1`。  
- **不变量**：搜索区间采用半开区间 `[l, r)`，循环条件 `l<r` 保证收敛无死循环。

**实现（TypeScript）**
```ts
function lowerBound(nums: number[], x: number): number {
  let l = 0, r = nums.length;              // [l, r)
  while (l < r) {
    const m = (l + r) >> 1;
    if (nums[m] >= x) r = m; else l = m + 1;
  }
  return l;                                 // 第一个 >= x 的位置
}
function upperBound(nums: number[], x: number): number {
  let l = 0, r = nums.length;
  while (l < r) {
    const m = (l + r) >> 1;
    if (nums[m] > x) r = m; else l = m + 1;
  }
  return l;                                 // 第一个 > x 的位置
}
export function searchRange(nums: number[], target: number): [number, number] {
  const L = lowerBound(nums, target);
  if (L === nums.length || nums[L] !== target) return [-1, -1];
  const R = upperBound(nums, target) - 1;
  return [L, R];
}
```

**复杂度**：时间 `O(log n)`，空间 `O(1)`。  
**易错点**：比较符号写反、闭开区间混用；重复元素多时右边界=上界-1。

---

## 12) 寻找峰值（二分）

**问题**  
数组 `nums` 满足相邻不相等，定义**峰值**为 `nums[i] > nums[i-1] && nums[i] > nums[i+1]`（边界与虚拟 `-∞` 比较）。返回任意一个峰下标。

**原理（爬坡二分）**  
- 若 `nums[m] < nums[m+1]`，说明右侧存在上坡，峰值在右半；否则在左半（含 `m`）。  
- **直觉**：把数组看作分段上升/下降；总能向陡升方向走到某个峰。

**实现**
```ts
export function findPeakElement(nums: number[]): number {
  let l = 0, r = nums.length - 1;
  while (l < r) {
    const m = (l + r) >> 1;
    if (nums[m] < nums[m + 1]) l = m + 1; // 峰在右
    else r = m;                           // 峰在左（含 m）
  }
  return l; // or r
}
```

**复杂度**：时间 `O(log n)`，空间 `O(1)`。  
**注意**：存在多个峰时返回任意即可；边界以虚拟 `-∞` 视作更小，长度为 1 时直接返回 0。

---

## 13) 合并区间（排序 + 扫描线）

**问题**  
给定若干闭区间 `[l, r]`，合并所有**相交或相邻**（根据题意决定是否连通）区间。

**原理**  
- 先按起点升序、起点相同按终点升序排序。  
- 扫描：维护当前合并区间 `cur = [L, R]`。对下一个区间 `[l, r]`：  
  - 若 `l <= R`（**相交**；若允许相邻合并则用 `l <= R+1` 或题意“相邻也合并”即 `l <= R`），更新 `R = max(R, r)`；  
  - 否则输出 `cur` 并重置为 `[l, r]`。

**实现**
```ts
export function mergeIntervals(intervals: Array<[number, number]>): Array<[number, number]> {
  if (intervals.length === 0) return [];
  intervals.sort((a, b) => (a[0] - b[0]) || (a[1] - b[1]));
  const res: Array<[number, number]> = [];
  let [L, R] = intervals[0];
  for (let i = 1; i < intervals.length; i++) {
    const [l, r] = intervals[i];
    if (l <= R) { // 相交（若题目要求相邻也合并，这里也可写 <= R）
      R = Math.max(R, r);
    } else {
      res.push([L, R]); [L, R] = [l, r];
    }
  }
  res.push([L, R]);
  return res;
}
```

**复杂度**：排序 `O(n log n)`，扫描 `O(n)`。  
**边界**：区间开闭性需与题意一致；注意大整数排序比较器避免字符串排序。

---

## 14) 回溯总题型：子集 / 全排列 / 组合总和（剪枝与去重）

### 14.1 子集（可含重复元素，结果去重）
**原理**：排序后同层去重（对同一层的相同值只取第一分支），递归枚举“选/不选”。  
```ts
export function subsetsWithDup(nums: number[]): number[][] {
  nums.sort((a, b) => a - b);
  const res: number[][] = [], path: number[] = [];
  function dfs(start: number) {
    res.push(path.slice());
    for (let i = start; i < nums.length; i++) {
      if (i > start && nums[i] === nums[i - 1]) continue; // 同层去重
      path.push(nums[i]);
      dfs(i + 1);
      path.pop();
    }
  }
  dfs(0);
  return res;
}
```
**复杂度**：`O(2^n)`（输出敏感），空间 `O(n)` 递归深度。

### 14.2 全排列（含重复值的去重排列）
**原理**：排序 + 访问标记 `used[]`，在**同一层**跳过“前一个相同且未使用”的数，避免等价交换。  
```ts
export function permuteUnique(nums: number[]): number[][] {
  nums.sort((a, b) => a - b);
  const n = nums.length, used = new Array<boolean>(n).fill(false);
  const res: number[][] = [], path: number[] = [];
  function dfs() {
    if (path.length === n) { res.push(path.slice()); return; }
    for (let i = 0; i < n; i++) {
      if (used[i]) continue;
      if (i > 0 && nums[i] === nums[i - 1] && !used[i - 1]) continue; // 同层去重
      used[i] = true; path.push(nums[i]);
      dfs();
      path.pop(); used[i] = false;
    }
  }
  dfs();
  return res;
}
```

### 14.3 组合总和（I：可重复选取；II：每数最多一次）
**I（无限次选）**：同一分支可重复使用当前数 → 递归时 `dfs(i, remain - nums[i])`。排序后可**剪枝**：`nums[i] > remain` 直接 break。  
```ts
export function combinationSum(nums: number[], target: number): number[][] {
  nums.sort((a, b) => a - b);
  const res: number[][] = [], path: number[] = [];
  function dfs(start: number, remain: number) {
    if (remain === 0) { res.push(path.slice()); return; }
    for (let i = start; i < nums.length; i++) {
      const x = nums[i];
      if (x > remain) break;       // 剪枝
      path.push(x);
      dfs(i, remain - x);          // 允许重复
      path.pop();
    }
  }
  dfs(0, target);
  return res;
}
```
**II（每数一次且可能有重复值）**：同层去重 + 递归时 `i+1`。  
```ts
export function combinationSum2(nums: number[], target: number): number[][] {
  nums.sort((a, b) => a - b);
  const res: number[][] = [], path: number[] = [];
  function dfs(start: number, remain: number) {
    if (remain === 0) { res.push(path.slice()); return; }
    for (let i = start; i < nums.length; i++) {
      if (i > start && nums[i] === nums[i - 1]) continue; // 同层去重
      const x = nums[i];
      if (x > remain) break;
      path.push(x);
      dfs(i + 1, remain - x);      // 不能重复使用
      path.pop();
    }
  }
  dfs(0, target);
  return res;
}
```

**复杂度**：均为输出敏感；剪枝能显著降低搜索树规模。  
**关键**：排序 + “同层去重”与“分支复用/不复用”的边界清晰。

---

## 15) 接雨水（双指针 / 单调栈）

**问题**  
给定非负整数数组表示柱高，求降雨后能接的总水量。

### 方案 A：双指针（`O(n)` / `O(1)`）
**原理**  
- 对位置 `i`，可接水量由 `min(leftMax, rightMax) - height[i]` 决定。  
- 从两端向中间移动：若 `leftMax <= rightMax`，则**左侧水位已确定**，累加 `leftMax - h[l]` 后 `l++`；反之处理右侧。  
```ts
export function trapTwoPointers(h: number[]): number {
  let l = 0, r = h.length - 1, leftMax = 0, rightMax = 0, ans = 0;
  while (l < r) {
    if (h[l] <= h[r]) {
      leftMax = Math.max(leftMax, h[l]);
      ans += leftMax - h[l]; l++;
    } else {
      rightMax = Math.max(rightMax, h[r]);
      ans += rightMax - h[r]; r--;
    }
  }
  return ans;
}
```
**正确性**：当 `leftMax <= rightMax` 时，`i∈[l]` 的有效水位上限为 `leftMax`（右侧更高或相等，不会成为瓶颈）。

### 方案 B：单调栈（`O(n)` / `O(n)`）
**原理**  
- 维护**单调递减栈**存柱子下标。遇到更高的柱子 `i`，弹出 `mid` 作为“低洼”，若栈仍非空，左边界为 `stack.top()`，用“短板高度”计算一段水量：  
  `water += (min(h[left], h[i]) - h[mid]) * (i - left - 1)`。  
```ts
export function trapStack(h: number[]): number {
  const st: number[] = []; let ans = 0;
  for (let i = 0; i < h.length; i++) {
    while (st.length && h[i] > h[st[st.length - 1]]) {
      const mid = st.pop()!;
      if (!st.length) break;
      const left = st[st.length - 1];
      const bounded = Math.min(h[left], h[i]) - h[mid];
      const width = i - left - 1;
      ans += bounded * width;
    }
    st.push(i);
  }
  return ans;
}
```

**对比**  
- 双指针：常用、常数小；  
- 单调栈：更接近“凹槽”几何意义，易于扩展到“接雨水 II（二维）”的思考方式（但二维需优先队列/BFS）。

---
## 16) 每日温度（单调栈）

**问题**  
给定每日温度数组 `T[i]`，返回数组 `ans[i]` 表示从第 `i` 天开始你需要等待多少天才会遇到更高的温度；若不存在返回 `0`。

**原理（单调递减栈：存下标）**  
- 维护一个**严格递减**的温度下标栈 `st`（从栈底到栈顶对应温度严格递减）。  
- 遍历到 `i` 时，若 `T[i] > T[st.top]`，不断弹出 `j = st.top`，说明 `i` 就是 `j` 的**下一个更大元素**，`ans[j] = i - j`。  
- 将 `i` 入栈，继续。

**正确性**  
- 栈内下标的温度严格递减，因此遇到更高温时可以一次性解决一批之前未定的天数。每个下标最多入栈、出栈各一次，整体 `O(n)`。

**实现（TypeScript）**
```ts
export function dailyTemperatures(T: number[]): number[] {
  const n = T.length, ans = new Array<number>(n).fill(0);
  const st: number[] = []; // 存下标，保持 T[st] 严格递减
  for (let i = 0; i < n; i++) {
    while (st.length && T[i] > T[st[st.length - 1]]) {
      const j = st.pop()!;
      ans[j] = i - j;
    }
    st.push(i);
  }
  return ans;
}
```

**复杂度**：时间 `O(n)`，空间 `O(n)`  
**易错点**：  
- 比较必须严格 `>`，否则“等温度”不算更热。  
- 栈里**存下标**而非温度，便于计算距离与失效判断。


---

## 17) 柱状图中最大的矩形（单调栈）

**问题**  
给定非负高度数组 `h[i]`，每个宽度为 1，求能形成的最大矩形面积。

**原理（单调非降栈 + 哨兵）**  
- 维护**单调非降**栈存下标。遍历到 `i` 时，若 `h[i] < h[st.top]`，不断弹出 `mid`，此时 `h[mid]` 作为矩形高度，其**左右第一小于它**的边界分别为：  
  - 右边界：`i-1`（因为当前 `h[i]` 小于它）  
  - 左边界：`st.top + 1`（弹栈后新的栈顶的右侧一位）  
  宽度 `w = i - 1 - (st.top + 1) + 1 = i - 1 - st.top`；若栈空则 `w = i`。  
- 在尾部追加一个高度 `0` 的**哨兵**，确保清空栈。

**实现（TypeScript）**
```ts
export function largestRectangleArea(h: number[]): number {
  const n = h.length;
  const heights = h.concat(0); // 哨兵
  const st: number[] = []; // 存下标，保持 heights[st] 非降
  let ans = 0;
  for (let i = 0; i < heights.length; i++) {
    while (st.length && heights[i] < heights[st[st.length - 1]]) {
      const mid = st.pop()!;
      const left = st.length ? st[st.length - 1] : -1;
      const width = i - left - 1;
      ans = Math.max(ans, heights[mid] * width);
    }
    st.push(i);
  }
  return ans;
}
```

**复杂度**：时间 `O(n)`，空间 `O(n)`  
**易错点**：  
- 栈单调性是**非降**（`<=` 也入栈），这样相等高度会被延后统一计算，避免重复统计。  
- 细节边界：栈空时左边界视为 `-1`。


---

## 18) 滑动窗口最大值（单调队列）

**问题**  
给定数组 `nums` 和窗口大小 `k`，求每个窗口的最大值。

**原理（双端队列，保持值单调递减，存下标）**  
- 队尾入队前，弹出所有**小于等于**新值的下标，使队列内对应值严格递减。  
- 队首若**过期**（`<= i - k`），弹出。  
- 从 `i >= k-1` 开始，窗口最大值即队首对应的值。

**实现（TypeScript）**
```ts
export function maxSlidingWindow(nums: number[], k: number): number[] {
  const n = nums.length, dq: number[] = [], res: number[] = [];
  for (let i = 0; i < n; i++) {
    // 入队：维护递减
    while (dq.length && nums[i] >= nums[dq[dq.length - 1]]) dq.pop();
    dq.push(i);
    // 出队：移除过期
    if (dq[0] <= i - k) dq.shift();
    // 记录答案
    if (i >= k - 1) res.push(nums[dq[0]]);
  }
  return res;
}
```

**复杂度**：时间 `O(n)`（每个下标最多入/出队一次），空间 `O(k)`  
**易错点**：  
- 比较用 `>=`，确保相等时**新元素覆盖旧元素**（更靠右，寿命更长）。  
- 队列存**下标**，便于判断是否过期。


---

## 19) K 个一组翻转链表（分段反转）

**问题**  
给定链表，将其每 `k` 个节点一组进行反转（`1 ≤ k ≤ n`），不足 `k` 的最后一组保持不变。返回新表头。

**原理**  
- 遍历链表，按 `k` 个为一段检测长度，够 `k` 则**原地反转**该段，并将上一段的尾结点连接到反转后的头结点。  
- 反转子过程：经典三指针 `prev/curr/next` 反转 `k` 次。

**实现（TypeScript，单链表）**
```ts
export class ListNode {
  constructor(public val = 0, public next: ListNode | null = null) {}
}

export function reverseKGroup(head: ListNode | null, k: number): ListNode | null {
  if (!head || k <= 1) return head;

  const dummy = new ListNode(0, head);
  let groupPrev: ListNode = dummy;

  while (true) {
    // 1) 找到这一组的第 k 个节点
    let kth: ListNode | null = groupPrev;
    for (let i = 0; i < k && kth; i++) kth = kth.next;
    if (!kth) break; // 不足 k

    const groupNext = kth.next;
    // 2) 反转 [groupPrev.next, kth]
    let prev: ListNode | null = groupNext;
    let cur: ListNode | null = groupPrev.next;
    while (cur !== groupNext) { // 反转到 groupNext 前
      const nxt = cur!.next;
      cur!.next = prev;
      prev = cur;
      cur = nxt;
    }
    // 3) 接回到主链
    const newGroupHead = prev!;
    const newGroupTail = groupPrev.next!;
    groupPrev.next = newGroupHead;
    groupPrev = newGroupTail;
  }

  return dummy.next;
}
```

**复杂度**：时间 `O(n)`，空间 `O(1)`  
**易错点**：  
- 反转时循环边界使用 `cur !== groupNext`，可避免额外计数。  
- 注意连接次序：上一段尾 → 新段头；新段尾（反转前的段头）→ `groupNext`。


---

## 20) 环形链表检测 & 链表相交（快慢指针 / 相遇点）

### 20.1 检测环与入口（Floyd 判圈）
**原理**  
- **相遇点**：快指针每次 2 步，慢指针每次 1 步；若存在环，必在环内相遇。  
- **入口证明**：设入环前长度为 `a`，环长 `b`。相遇后，让一指针回到表头，两指针都以 1 步前进，再次相遇即为**环入口**。  
  直觉：快指针比慢指针多走了 `k*b`，两者到入口所需步数相同。

**实现（返回入口或 `null`）**
```ts
export function detectCycle(head: ListNode | null): ListNode | null {
  let slow = head, fast = head;
  while (fast && fast.next) {
    slow = slow!.next;
    fast = fast.next.next;
    if (slow === fast) {
      let p1 = head, p2 = slow;
      while (p1 !== p2) { p1 = p1!.next; p2 = p2!.next; }
      return p1;
    }
  }
  return null;
}
```

### 20.2 链表相交（无环情形）
**原理（双指针切换头）**  
- 两指针 `pA`、`pB` 分别从 `A`、`B` 头出发，走到尾后**切换到对方表头**继续走。  
- 若相交，在第二轮某处相遇；若不相交，两者最终同时为 `null`。  
- **证明**：走过的长度均为 `lenA + lenB`，起点差异被抵消。

**实现**
```ts
export function getIntersectionNode(
  headA: ListNode | null, headB: ListNode | null
): ListNode | null {
  if (!headA || !headB) return null;
  let pA: ListNode | null = headA, pB: ListNode | null = headB;
  while (pA !== pB) {
    pA = pA ? pA.next : headB;
    pB = pB ? pB.next : headA;
  }
  return pA; // 相交点或 null
}
```

**进阶（有环相交）**  
- 先分别检测两表是否有环：  
  - 若一有环、一无环 → 不相交；  
  - 若都无环 → 用上法；  
  - 若都有环：  
    - 若入口相同 → 求入环前的相交点（或返回入口皆可）；  
    - 若入口不同 → 从其中一个入口沿环走一圈，若能遇到另一个入口 → 相交（在环上），否则不相交。

**复杂度**：均为时间 `O(n+m)`，空间 `O(1)`。

---
## 21) 二叉树遍历（层序 / 前中后序，递归与迭代）

**问题**  
实现二叉树的遍历：层序（BFS）、前序/中序/后序（DFS），给出递归与迭代写法，并说明核心不变量。

**模型与节点定义（TS）**
```ts
export class TreeNode {
  constructor(
    public val: number,
    public left: TreeNode | null = null,
    public right: TreeNode | null = null
  ) {}
}
```

### 21.1 层序遍历（BFS，队列）
**不变量**：队列中保存“下一批将被访问的节点”，逐层出队、入队子节点。
```ts
export function levelOrder(root: TreeNode | null): number[][] {
  if (!root) return [];
  const res: number[][] = [];
  const q: TreeNode[] = [root];
  while (q.length) {
    const n = q.length, level: number[] = [];
    for (let i = 0; i < n; i++) {
      const node = q.shift()!;
      level.push(node.val);
      if (node.left) q.push(node.left);
      if (node.right) q.push(node.right);
    }
    res.push(level);
  }
  return res;
}
```
时间 `O(n)`，空间 `O(w)`（`w` 为最大层宽）。

### 21.2 前中后序（递归）
```ts
export function preorderRec(root: TreeNode | null, out: number[] = []): number[] {
  if (!root) return out;
  out.push(root.val);
  preorderRec(root.left, out);
  preorderRec(root.right, out);
  return out;
}
export function inorderRec(root: TreeNode | null, out: number[] = []): number[] {
  if (!root) return out;
  inorderRec(root.left, out);
  out.push(root.val);
  inorderRec(root.right, out);
  return out;
}
export function postorderRec(root: TreeNode | null, out: number[] = []): number[] {
  if (!root) return out;
  postorderRec(root.left, out);
  postorderRec(root.right, out);
  out.push(root.val);
  return out;
}
```
时间 `O(n)`，空间最坏 `O(n)`（递归栈）。

### 21.3 前中后序（迭代）
**前序（根→左→右）**：栈出栈顺序与入栈顺序相反，先压右再压左。
```ts
export function preorderIter(root: TreeNode | null): number[] {
  if (!root) return [];
  const st: TreeNode[] = [root], out: number[] = [];
  while (st.length) {
    const node = st.pop()!;
    out.push(node.val);
    if (node.right) st.push(node.right);
    if (node.left) st.push(node.left);
  }
  return out;
}
```

**中序（左→根→右）**：指针走到最左一路入栈；回退访问根；转向右子树。
```ts
export function inorderIter(root: TreeNode | null): number[] {
  const out: number[] = [], st: TreeNode[] = [];
  let cur = root;
  while (cur || st.length) {
    while (cur) { st.push(cur); cur = cur.left; }
    const node = st.pop()!;
    out.push(node.val);
    cur = node.right;
  }
  return out;
}
```

**后序（左→右→根）**（两法）  
- 双栈法：前序“根右左”结果反转即可。  
- 单栈 + 访问标记：利用“最近一次访问的节点”为右子标记。
```ts
export function postorderIter(root: TreeNode | null): number[] {
  const res: number[] = [];
  const st: (TreeNode | null)[] = [];
  let cur = root, lastVisited: TreeNode | null = null;
  while (cur || st.length) {
    while (cur) { st.push(cur); cur = cur.left; }
    const peek = st[st.length - 1] as TreeNode;
    if (peek.right && lastVisited !== peek.right) {
      cur = peek.right; // 先处理右子树
    } else {
      res.push(peek.val);
      lastVisited = st.pop() as TreeNode;
    }
  }
  return res;
}
```

**进阶**：Morris 遍历可将前/中序做到 `O(1)` 额外空间（建立/拆除“线索”指针），适合极限内存场景。

---

## 22) 二叉树最近公共祖先 LCA（递归 / 倍增）

**问题**  
给定二叉树根 `root`，与两个节点 `p,q`，求最近公共祖先（LCA）。

### 22.1 通用递归（不依赖 BST）
**思想**  
- 递归返回值：在以 `root` 为根的子树中，若存在 `p` 或 `q`，返回该节点；若左右子树各返回一个非空，当前 `root` 即 LCA。  
- **不变量**：返回“存在性指示 + 该分支命中的节点”。

```ts
export function lowestCommonAncestor(root: TreeNode | null, p: TreeNode, q: TreeNode): TreeNode | null {
  if (!root || root === p || root === q) return root;
  const L = lowestCommonAncestor(root.left, p, q);
  const R = lowestCommonAncestor(root.right, p, q);
  if (L && R) return root;
  return L ?? R;
}
```
时间 `O(n)`，空间 `O(h)`（递归深度）。

### 22.2 倍增（Binary Lifting，适合多次查询）
**思想**  
- 预处理：DFS 记录每个节点的深度 `depth[u]` 与 `up[k][u]`（`u` 向上 2^k 个祖先）。  
- 查询：  
  1）把较深的节点提升到同深度；  
  2）自高向低尝试同时跳祖先，当 `up[k][u] !== up[k][v]` 时跳；  
  3）最后返回 `up[0][u]`。

**实现骨架（以邻接表形式维护树，需把二叉树转换或在构造时同时建立 parent）**
```ts
const LOG = 17; // 足以覆盖 1<<17 > 1e5
let up: number[][], depth: number[];
let id = new Map<TreeNode, number>(), nodes: TreeNode[] = [];

function index(root: TreeNode | null) { // 可选：为节点编号
  if (!root) return;
  const n = nodes.push(root) - 1;
  id.set(root, n);
  index(root.left); index(root.right);
}

function dfsBuild(u: number, parent: number) {
  up[0][u] = parent;
  for (let k = 1; k < LOG; k++) up[k][u] = up[k - 1][u] === -1 ? -1 : up[k - 1][ up[k - 1][u] ];
  const node = nodes[u];
  for (const v of [node.left, node.right]) {
    if (!v) continue;
    const vi = id.get(v)!;
    depth[vi] = depth[u] + 1;
    dfsBuild(vi, u);
  }
}
export function buildLCA(root: TreeNode | null) {
  nodes = []; id = new Map(); index(root);
  const n = nodes.length;
  up = Array.from({ length: LOG }, () => new Array<number>(n).fill(-1));
  depth = new Array<number>(n).fill(0);
  if (n) dfsBuild(0, -1);
}
export function lcaByIndex(a: TreeNode, b: TreeNode): TreeNode | null {
  if (!nodes.length) return null;
  let u = id.get(a)!, v = id.get(b)!;
  if (depth[u] < depth[v]) [u, v] = [v, u];
  let diff = depth[u] - depth[v];
  for (let k = 0; k < LOG; k++) if (diff & (1 << k)) u = up[k][u];
  if (u === v) return nodes[u];
  for (let k = LOG - 1; k >= 0; k--) {
    if (up[k][u] !== up[k][v]) { u = up[k][u]; v = up[k][v]; }
  }
  return up[0][u] === -1 ? null : nodes[ up[0][u] ];
}
```
预处理 `O(n log n)`，单次查询 `O(log n)`。适合**多次 LCA 查询**或在图/树题中作为基础设施。

**注**：若是 **BST** 的 LCA，可利用有序性质：当 `p.val<root.val<q.val`（或反之）时 `root` 即 LCA；否则向较大/较小方向递归，`O(h)`。

---

## 23) 二叉搜索树第 K 小元素（中序 + 计数 / 迭代 / 维护子树规模）

**问题**  
在 BST 中找第 `k` 小的键（1-based）。

### 23.1 中序 + 计数（迭代）
**不变量**：BST 的中序遍历是递增序列；用栈模拟中序，访问第 `k` 个即结果。
```ts
export function kthSmallest(root: TreeNode | null, k: number): number {
  const st: TreeNode[] = [];
  let cur = root, cnt = 0;
  while (cur || st.length) {
    while (cur) { st.push(cur); cur = cur.left; }
    cur = st.pop()!;
    if (++cnt === k) return cur.val;
    cur = cur.right;
  }
  throw new Error("k out of range");
}
```
时间 `O(h + k)`，最坏 `O(n)`；空间 `O(h)`。

### 23.2 维护子树规模（Order-Statistic Tree 思想，适合频繁查询）
在节点上维护 `size = left.size + right.size + 1`，查询时比较 `k` 与 `left.size+1` 来决定向左/右/命中：  
- `k === leftSize+1` → 命中；  
- `k ≤ leftSize` → 去左；  
- 否则 `k -= leftSize+1` 去右。  
插入/删除需维护 `size`，可结合平衡树（AVL/红黑树）保证 `O(log n)`。

---

## 24) 二叉树最大路径和（递归返回“贡献值”）

**问题**  
路径定义为从任意节点出发、到任意节点（不必过根），路径节点不重复；求所有路径的最大和。

**核心递归不变量**  
- `gain(node)`：从 `node` 出发向**下**走（只能选一侧子树）的最大贡献值；若贡献为负则视为 0（不取该子树）。  
- 在节点处尝试作为**路径拐点**：`pathSum = leftGain + node.val + rightGain`，更新全局最大值。  
- 返回给父节点的只能是“单边贡献”：`node.val + max(leftGain, rightGain)`（不能把左右都抬上去，否则路径会分叉）。

**实现（TS）**
```ts
export function maxPathSum(root: TreeNode | null): number {
  let best = -Infinity;
  function gain(node: TreeNode | null): number {
    if (!node) return 0;
    const left = Math.max(0, gain(node.left));
    const right = Math.max(0, gain(node.right));
    best = Math.max(best, left + node.val + right); // 以 node 为拐点
    return node.val + Math.max(left, right);        // 向上只取一边
  }
  gain(root);
  return best;
}
```
时间 `O(n)`，空间 `O(h)`。  
**注意**：节点值可能为负，因此初始 `best=-Infinity`；子树贡献为负时取 0 表示“不走该边”。

---

## 25) 课程表 / 能否完成所有课程（拓扑排序：Kahn / DFS）

**问题**  
给定课程数 `n` 与先修边 `prereq = [ [a, b] ]` 表示先修 `b` 才能学 `a`，判断是否可完成所有课程（是否有环）。扩展：输出任意一条可行的学习顺序（拓扑序）。

### 25.1 Kahn 算法（BFS 拓扑，入度为 0）
**不变量**：队列中是**当前入度为 0**的节点；每弹出一个，就“删除”它发出的边、相应邻居入度减 1。
```ts
export function canFinish(n: number, prereq: number[][]): boolean {
  const g: number[][] = Array.from({ length: n }, () => []);
  const indeg = new Array<number>(n).fill(0);
  for (const [a, b] of prereq) { g[b].push(a); indeg[a]++; }
  const q: number[] = [];
  for (let i = 0; i < n; i++) if (indeg[i] === 0) q.push(i);
  let seen = 0;
  while (q.length) {
    const u = q.shift()!; seen++;
    for (const v of g[u]) if (--indeg[v] === 0) q.push(v);
  }
  return seen === n; // 全部出队则无环
}
```
若需要**拓扑序**，将出队顺序收集到数组，最后长度为 `n` 即该序列有效。

时间 `O(n+m)`，空间 `O(n+m)`。

### 25.2 DFS 染色检测环（递归）
**颜色语义**：`0=未访问`，`1=访问中`（灰，递归栈中），`2=已完成`。  
出现回边到灰色结点即存在环。
```ts
export function canFinishDFS(n: number, prereq: number[][]): boolean {
  const g: number[][] = Array.from({ length: n }, () => []);
  for (const [a, b] of prereq) g[b].push(a);
  const color = new Array<number>(n).fill(0);
  let ok = true;
  function dfs(u: number) {
    color[u] = 1;
    for (const v of g[u]) {
      if (color[v] === 0) dfs(v);
      else if (color[v] === 1) { ok = false; return; } // 找到环
    }
    color[u] = 2;
  }
  for (let i = 0; i < n && ok; i++) if (color[i] === 0) dfs(i);
  return ok;
}
```

**Kahn vs DFS**  
- Kahn 更直观，顺带给出拓扑序；  
- DFS 易于嵌入“强连通/边分类”分析。两者时间/空间同阶。

**易错点**  
- 图方向：`b -> a`（学 b 才能学 a）；  
- 多个弱连通分量要从所有入度 0 的点出发；  
- DFS 写法要**在发现环后尽快剪枝**，避免不必要的递归。

---
## 26) 岛屿问题：数量 / 最大面积 / 封闭岛（DFS/BFS/并查集）

**模型**  
- 给定 `grid`（0/1 或 '0'/'1'），四联通（上/下/左/右）。“岛屿”指一组相邻的 1。  
- 变体：  
  - **数量**：统计连通分量数。  
  - **最大面积**：连通块内单元格数量最大值。  
  - **封闭岛**：值为 0 的“陆地”被 1（海洋）完全包围，且不与边界相连。

### DFS/BFS（最常用）
**不变量**：访问一次即标记（`visited` 或原地改值），同一连通块只会被计入一次。

```ts
type Grid = number[][];
const DIRS = [[1,0],[-1,0],[0,1],[0,-1]] as const;

export function numIslands(grid: Grid): number {
  const m = grid.length, n = grid[0]?.length ?? 0;
  if (!m || !n) return 0;
  let count = 0;
  const dfs = (r: number, c: number) => {
    if (r<0||r>=m||c<0||c>=n||grid[r][c] !== 1) return;
    grid[r][c] = -1; // 标记访问
    for (const [dr,dc] of DIRS) dfs(r+dr, c+dc);
  };
  for (let i=0;i<m;i++) for (let j=0;j<n;j++) {
    if (grid[i][j]===1){ count++; dfs(i,j); }
  }
  return count;
}

export function maxIslandArea(grid: Grid): number {
  const m = grid.length, n = grid[0].length;
  let best = 0;
  const dfs = (r: number, c: number): number => {
    if (r<0||r>=m||c<0||c>=n||grid[r][c] !== 1) return 0;
    grid[r][c] = -1;
    let area = 1;
    for (const [dr,dc] of DIRS) area += dfs(r+dr, c+dc);
    return area;
  };
  for (let i=0;i<m;i++) for (let j=0;j<n;j++) {
    if (grid[i][j]===1) best = Math.max(best, dfs(i,j));
  }
  return best;
}
```

**封闭岛**（如 `0` 为陆地，`1` 为水）  
- 思路：**先“淹没”边界可达的 0**（它们不可能封闭），再统计剩余 0 的连通分量数。
```ts
export function closedIslands(grid: Grid): number {
  const m = grid.length, n = grid[0].length;
  const dfs = (r: number, c: number) => {
    if (r<0||r>=m||c<0||c>=n||grid[r][c] !== 0) return;
    grid[r][c] = 2; // 标记为水
    for (const [dr,dc] of DIRS) dfs(r+dr, c+dc);
  };
  // 1) 淹没边界 0
  for (let i=0;i<m;i++) { dfs(i,0); dfs(i,n-1); }
  for (let j=0;j<n;j++) { dfs(0,j); dfs(m-1,j); }
  // 2) 统计剩余 0 的连通分量
  let ans = 0;
  for (let i=1;i<m-1;i++) for (let j=1;j<n-1;j++) {
    if (grid[i][j]===0){ ans++; dfs(i,j); }
  }
  return ans;
}
```

### 并查集（当需要频繁合并/查询或在线增量建图）
- 将 `(r,c)` 映射为一维 `id=r*n+c`，1 与邻居 1 `union`，数量 = 组件数。  
- **封闭岛**可增加一个“边界超级点”，凡是与边界连通的 0 都 union 到超级点，最终统计不与超级点连通的 0 的集合数。

**复杂度**：DFS/BFS/并查集构建均为 `O(mn)`；并查集近似常数的阿克曼反函数摊还。


---

## 27) 最短路径：无权图 BFS / 加权图 Dijkstra（优先队列）

**无权图**（边权统一为 1 或不计权）  
- **BFS** 一次层序即是起点到各点的最少边数；队列中的层序不变量保证首次到达即最短。
```ts
export function shortestPathUnweighted(n: number, edges: number[][], s: number): number[] {
  const g: number[][] = Array.from({length:n},()=>[]);
  for (const [u,v] of edges){ g[u].push(v); g[v].push(u); } // 无向
  const dist = new Array<number>(n).fill(Infinity);
  const q: number[] = [s]; dist[s]=0;
  for (let qi=0; qi<q.length; qi++){
    const u=q[qi];
    for (const v of g[u]) if (dist[v]===Infinity) {
      dist[v]=dist[u]+1; q.push(v);
    }
  }
  return dist;
}
```

**非负权图**  
- **Dijkstra + 最小堆**：维护 `dist[u]`，每次取当前未确定中 `dist` 最小的 `u` 松弛邻边。  
- **堆中多次入堆**是允许的：当弹出过时条目（`d>dist[u]`）直接跳过。

```ts
class MinHeap<T> {
  constructor(private arr: T[], private less: (a:T,b:T)=>boolean) {}
  static from<T>(less:(a:T,b:T)=>boolean){ return new MinHeap<T>([], less); }
  push(x:T){ const a=this.arr; a.push(x); this.up(a.length-1); }
  up(i:number){ const a=this.arr, less=this.less; for(let p;(i>0)&&less(a[i],a[p=(i-1)>>1]); i=p) [a[i],a[p]]=[a[p],a[i]]; }
  pop():T|undefined{ const a=this.arr; if(!a.length) return undefined; const top=a[0]; const x=a.pop()!; if(a.length){ a[0]=x; this.down(0); } return top; }
  down(i:number){ const a=this.arr, less=this.less; for(;;){ let l=i*2+1,r=l+1, m=i; if(l<a.length&&less(a[l],a[m])) m=l; if(r<a.length&&less(a[r],a[m])) m=r; if(m===i) break; [a[i],a[m]]=[a[m],a[i]]; i=m; } }
  get size(){ return this.arr.length; }
}

type Edge = [to:number, w:number];
export function dijkstra(n:number, graph: Edge[][], s:number): number[] {
  const dist = new Array<number>(n).fill(Infinity);
  dist[s]=0;
  const pq = MinHeap.from<[number,number]>((a,b)=>a[0]<b[0]); // [dist,u]
  pq.push([0,s]);
  while (pq.size){
    const [d,u] = pq.pop()!;
    if (d!==dist[u]) continue; // 过时条目
    for (const [v,w] of graph[u]){
      const nd = d + w;
      if (nd < dist[v]) { dist[v]=nd; pq.push([nd,v]); }
    }
  }
  return dist;
}
```

**正确性要点**  
- 非负权保证“从小到大出堆”的 `u` 已是最短（贪心成立）。  
- 存在**负权**：需 Bellman–Ford/SPFA 或 Johnson（重标定 + Dijkstra）。  
- 稀疏图优先堆实现 `O((V+E)logV)`；稠密图可用数组选最小 `O(V^2)`。


---

## 28) Trie 前缀树：插入 / 查询 / 前缀匹配与通配

**结构与不变量**  
- 每个节点存子边集合（`Map<char,child>` 或定长数组）与 `end` 标记/计数。  
- 查找/插入沿字符链走；**不存在即失败**或新建。

```ts
class TrieNode {
  children = new Map<string, TrieNode>();
  end = 0; // 该节点作为单词终止的次数
}
export class Trie {
  private root = new TrieNode();

  insert(word: string): void {
    let p = this.root;
    for (const ch of word) {
      if (!p.children.has(ch)) p.children.set(ch, new TrieNode());
      p = p.children.get(ch)!;
    }
    p.end++;
  }
  search(word: string): boolean {
    const node = this.walk(word);
    return !!node && node.end > 0;
  }
  startsWith(prefix: string): boolean {
    return !!this.walk(prefix);
  }
  private walk(s: string): TrieNode | null {
    let p = this.root;
    for (const ch of s) {
      const nxt = p.children.get(ch);
      if (!nxt) return null;
      p = nxt;
    }
    return p;
  }
}
```

**通配查询（如 `.` 匹配任意 1 个字符）**  
- 采用 DFS 回溯：遇到 `.` 则尝试所有子分支。
```ts
export function searchWithDot(root: TrieNode, pattern: string): boolean {
  const dfs = (node: TrieNode, i: number): boolean => {
    if (i === pattern.length) return node.end > 0;
    const ch = pattern[i];
    if (ch !== '.') {
      const nxt = node.children.get(ch);
      return !!nxt && dfs(nxt, i + 1);
    }
    for (const [, nxt] of node.children) if (dfs(nxt, i + 1)) return true;
    return false;
  };
  return dfs(root, 0);
}
```

**应用**  
- 前缀联想/自动补全、受限词过滤（搭配 Aho–Corasick 失配指针）、T9/拼写纠错（Trie + 编辑距离 DP 剪枝）。  
- 统计前缀出现次数：节点存 `pass` 计数（从根到该节点的路径经过次数）即可支持前缀计数查询。


---

## 29) LRU / LFU 缓存设计（哈希 + 双链表 / 频次桶）

### 29.1 LRU（Least Recently Used）
**不变量**：  
- 哈希 `key -> node`（O(1) 定位）。  
- 双向链表维护访问顺序：**最近用过**的在表头；淘汰表尾。  
- `get/put` 都把命中的节点移动到表头；满容时弹尾。

```ts
class DLinkedNode {
  constructor(
    public key: number, public val: number,
    public prev: DLinkedNode | null = null,
    public next: DLinkedNode | null = null
  ) {}
}
export class LRUCache {
  private map = new Map<number, DLinkedNode>();
  private head = new DLinkedNode(0,0);
  private tail = new DLinkedNode(0,0);
  constructor(private cap: number) { this.head.next = this.tail; this.tail.prev = this.head; }

  private addFirst(node: DLinkedNode){ node.next=this.head.next; node.prev=this.head; this.head.next!.prev=node; this.head.next=node; }
  private remove(node: DLinkedNode){ node.prev!.next=node.next; node.next!.prev=node.prev; }
  private moveToFirst(node:DLinkedNode){ this.remove(node); this.addFirst(node); }

  get(key: number): number {
    const node = this.map.get(key);
    if (!node) return -1;
    this.moveToFirst(node);
    return node.val;
  }
  put(key: number, value: number): void {
    const node = this.map.get(key);
    if (node) { node.val = value; this.moveToFirst(node); return; }
    if (this.map.size === this.cap) {
      const last = this.tail.prev!; this.remove(last); this.map.delete(last.key);
    }
    const x = new DLinkedNode(key, value); this.addFirst(x); this.map.set(key, x);
  }
}
```

### 29.2 LFU（Least Frequently Used）
**不变量**：  
- `key -> node{key,val,freq}`；  
- `freq -> 双向链表` 存该频次的 LRU 顺序（用于同频次内部再以时间淘汰）。  
- 维护当前最小频次 `minFreq`。  
- `get/put`：将节点从旧频次链移除，`freq++` 后插入新频次链头；必要时更新 `minFreq`。  
- 淘汰时从 `minFreq` 链表尾弹出。

```ts
class LFUNode {
  constructor(
    public key:number, public val:number, public freq=1,
    public prev: LFUNode | null = null, public next: LFUNode | null = null
  ) {}
}
class LFUList {
  head = new LFUNode(0,0); tail = new LFUNode(0,0); size=0;
  constructor(){ this.head.next=this.tail; this.tail.prev=this.head; }
  addFirst(x:LFUNode){ x.next=this.head.next; x.prev=this.head; this.head.next!.prev=x; this.head.next=x; this.size++; }
  remove(x:LFUNode){ x.prev!.next=x.next; x.next!.prev=x.prev; this.size--; }
  removeLast(): LFUNode | null { if(!this.size) return null; const x=this.tail.prev!; this.remove(x); return x; }
}
export class LFUCache {
  private key2node = new Map<number, LFUNode>();
  private freq2list = new Map<number, LFUList>();
  private minFreq = 0;
  constructor(private cap:number){}

  private touch(node: LFUNode) {
    const oldFreq = node.freq;
    const lst = this.freq2list.get(oldFreq)!; lst.remove(node);
    if (oldFreq === this.minFreq && lst.size === 0) this.minFreq++;
    node.freq++;
    let nl = this.freq2list.get(node.freq);
    if (!nl) { nl = new LFUList(); this.freq2list.set(node.freq, nl); }
    nl.addFirst(node);
  }

  get(key:number): number {
    const node = this.key2node.get(key); if(!node) return -1;
    this.touch(node); return node.val;
  }
  put(key:number, value:number): void {
    if (this.cap===0) return;
    const node = this.key2node.get(key);
    if (node){ node.val=value; this.touch(node); return; }
    if (this.key2node.size === this.cap) {
      const lst = this.freq2list.get(this.minFreq)!;
      const rm = lst.removeLast()!; this.key2node.delete(rm.key);
    }
    const x = new LFUNode(key, value, 1);
    let l1 = this.freq2list.get(1);
    if (!l1){ l1 = new LFUList(); this.freq2list.set(1, l1); }
    l1.addFirst(x); this.key2node.set(key, x); this.minFreq = 1;
  }
}
```

**复杂度**：两者 `get/put` 均 `O(1)` 摊还；LFU 的常数略大、实现更复杂。  
**选择**：强时序局部性 → LRU；存在“冷数据被频繁触发一次后立刻淘汰”问题时考虑 LFU 或 LRU-K。


---

## 30) 前缀和与差分（含二维）

### 30.1 一维前缀和（区间求和）
**定义**：`pre[i] = a[0]+...+a[i-1]`（长度 `n+1`），区间 `[l,r]` 的和为 `pre[r+1]-pre[l]`。  
- 多次静态查询时能把每次查询降到 `O(1)`。

```ts
export class PrefixSum1D {
  pre: number[];
  constructor(a: number[]) {
    this.pre = new Array(a.length + 1).fill(0);
    for (let i=0;i<a.length;i++) this.pre[i+1] = this.pre[i] + a[i];
  }
  sum(l:number, r:number){ return this.pre[r+1]-this.pre[l]; }
}
```

### 30.2 一维差分（区间增量更新）
**思想**：区间 `[l,r]` 加 `val`，在差分数组 `diff` 上执行 `diff[l]+=val; diff[r+1]-=val`；最终 `a[i]=prefix(diff)[i]`。
```ts
export class Diff1D {
  diff: number[];
  constructor(n: number){ this.diff = new Array(n+1).fill(0); }
  rangeAdd(l:number, r:number, val:number){ this.diff[l]+=val; if (r+1<this.diff.length) this.diff[r+1]-=val; }
  materialize(): number[] {
    const n=this.diff.length-1, a=new Array<number>(n); let run=0;
    for (let i=0;i<n;i++){ run+=this.diff[i]; a[i]=run; }
    return a;
  }
}
```

### 30.3 二维前缀和（子矩形求和）
**定义**：`pre[i+1][j+1] = Σ_{0..i,0..j} a[x][y]`；  
查询子矩形 `[(r1,c1)..(r2,c2)]`：  
`sum = pre[r2+1][c2+1] - pre[r1][c2+1] - pre[r2+1][c1] + pre[r1][c1]`。

```ts
export class PrefixSum2D {
  pre: number[][];
  constructor(a: number[][]) {
    const m=a.length, n=a[0].length;
    this.pre = Array.from({length:m+1},()=>new Array<number>(n+1).fill(0));
    for (let i=0;i<m;i++) for (let j=0;j<n;j++)
      this.pre[i+1][j+1] = this.pre[i+1][j] + this.pre[i][j+1] - this.pre[i][j] + a[i][j];
  }
  sum(r1:number,c1:number,r2:number,c2:number): number {
    return this.pre[r2+1][c2+1]-this.pre[r1][c2+1]-this.pre[r2+1][c1]+this.pre[r1][c1];
  }
}
```

### 30.4 二维差分（矩形增量更新）
对矩形 `[(r1,c1)..(r2,c2)]` 加 `val`：在差分矩阵 `D` 上执行  
```
D[r1][c1]   += val
D[r1][c2+1] -= val
D[r2+1][c1] -= val
D[r2+1][c2+1] += val
```
最终通过**两次前缀**（先行后列或反之）还原。

```ts
export class Diff2D {
  D: number[][];
  constructor(public m:number, public n:number){
    this.D = Array.from({length:m+1},()=>new Array<number>(n+1).fill(0));
  }
  add(r1:number,c1:number,r2:number,c2:number,val:number){
    this.D[r1][c1]+=val;
    if (c2+1<=this.n) this.D[r1][c2+1]-=val;
    if (r2+1<=this.m) this.D[r2+1][c1]-=val;
    if (r2+1<=this.m && c2+1<=this.n) this.D[r2+1][c2+1]+=val;
  }
  materialize(): number[][] {
    const A = Array.from({length:this.m},()=>new Array<number>(this.n).fill(0));
    // 行前缀
    for (let i=0;i<=this.m;i++) for (let j=1;j<=this.n;j++) this.D[i][j]+=this.D[i][j-1];
    // 列前缀
    for (let j=0;j<=this.n;j++) for (let i=1;i<=this.m;i++) this.D[i][j]+=this.D[i-1][j];
    for (let i=0;i<this.m;i++) for (let j=0;j<this.n;j++) A[i][j]=this.D[i][j];
    return A;
  }
}
```

**选择与边界**  
- **多次静态求和** → 前缀和。  
- **多次区间增量 + 一次还原** → 差分。  
- 若需**同时**支持区间加与区间和在线查询，考虑**树状数组/线段树**（见第 50 题）。

---
## 31) 子数组和为 K 的个数（前缀和 + 哈希计数）

**问题**  
给定整数数组 `nums` 与整数 `k`，统计**连续子数组**的个数使其和为 `k`。允许负数与 0。

**原理**  
- 设前缀和 `pre[i] = nums[0] + ... + nums[i-1]`（`pre[0]=0`）。  
- 区间 `[l..r]` 的和为 `pre[r+1] - pre[l]`。欲使和为 `k`，等价于：  
  `pre[l] = pre[r+1] - k`。  
- 因此在遍历到前缀和 `S = pre[r+1]` 时，已有答案的增量就是**历史上** `pre` 等于 `S - k` 的次数。  
- 用计数哈希 `cnt: Map<sum, count>` 维护已出现的前缀和频次；初始化 `cnt(0)=1`（空前缀）。

**正确性**  
- 每个区间被计算一次，且只依赖其右端点前的历史前缀；不会重/漏计。  
- 允许负数，因为该方法不依赖单调性。

**复杂度**：时间 `O(n)`，空间 `O(n)`。

**实现（TypeScript）**
```ts
export function subarraySum(nums: number[], k: number): number {
  const cnt = new Map<number, number>();
  cnt.set(0, 1);
  let pre = 0, ans = 0;
  for (const x of nums) {
    pre += x;
    ans += cnt.get(pre - k) ?? 0;
    cnt.set(pre, (cnt.get(pre) ?? 0) + 1);
  }
  return ans;
}
```

**变体**  
- **恰好 K 个奇数的子数组数**：把奇偶映射为 0/1，再求“子数组和为 K”。  
- **二维**：构造行前缀后固定上下边界，将矩形和转为“一维子数组和为 K”，总复杂度 `O(n^2 m)`。


---

## 32) 第 K 大元素（小顶堆 / 随机化 Quickselect）

**问题**  
返回数组中第 `k` 大元素（`k≥1`）。

### 方案 A：小顶堆（`O(n log k)` 稳定易实现）
**思路**  
- 维护大小为 `k` 的**小顶堆**；遍历数组，若堆未满则入堆；否则比较堆顶与当前元素，若当前更大则替换堆顶。  
- 最终堆顶即第 `k` 大。

```ts
class MinHeap {
  private a: number[] = [];
  private up(i: number){ for(let p;(i>0)&&(this.a[i]<this.a[p=(i-1)>>1]); i=p) [this.a[i],this.a[p]]=[this.a[p],this.a[i]]; }
  private down(i:number){ for(;;){ let l=i*2+1,r=l+1,m=i; if(l<this.a.length&&this.a[l]<this.a[m]) m=l; if(r<this.a.length&&this.a[r]<this.a[m]) m=r; if(m===i) break; [this.a[i],this.a[m]]=[this.a[m],this.a[i]]; i=m; } }
  push(x:number){ this.a.push(x); this.up(this.a.length-1); }
  pop():number{ const t=this.a[0], x=this.a.pop()!; if(this.a.length){ this.a[0]=x; this.down(0);} return t; }
  top(){ return this.a[0]; } size(){ return this.a.length; }
}
export function kthLargest_heap(nums: number[], k: number): number {
  const h = new MinHeap();
  for (const x of nums) {
    if (h.size() < k) h.push(x);
    else if (x > h.top()) { h.pop(); h.push(x); }
  }
  return h.top();
}
```

### 方案 B：随机化 Quickselect（期望 `O(n)`）
**思路**  
- 选择一个枢轴，将数组按“大于枢轴 / 等于 / 小于”分区；第 `k` 大落在某一分区中，递归/迭代仅进入该分区。  
- 为避免最坏 `O(n^2)`，每轮随机选枢轴或三数取中。

```ts
export function kthLargest_quickselect(nums: number[], k: number): number {
  let l = 0, r = nums.length - 1, target = nums.length - k; // 转为第 target 小
  while (l <= r) {
    const p = l + Math.floor(Math.random() * (r - l + 1));
    const pivot = nums[p];
    // 三向切分 (Dutch National Flag)
    let i = l, lt = l, gt = r;
    while (i <= gt) {
      if (nums[i] < pivot) { [nums[i], nums[lt]] = [nums[lt], nums[i]]; i++; lt++; }
      else if (nums[i] > pivot) { [nums[i], nums[gt]] = [nums[gt], nums[i]]; gt--; }
      else i++;
    }
    if (target < lt) r = lt - 1;
    else if (target > gt) l = gt + 1;
    else return pivot; // target 落在等于区
  }
  throw new Error("unreachable");
}
```

**对比**  
- 堆：实现简单、适合**数据流**；  
- Quickselect：一次性查询更快、空间 `O(1)`，注意随机化与退化风险。


---

## 33) Top K 高频元素（小顶堆 / 桶排序）

**问题**  
给定数组，返回频率最高的 `k` 个元素。

### 方案 A：哈希 + 小顶堆（`O(n log k)`）
```ts
export function topKFrequent_heap(nums: number[], k: number): number[] {
  const freq = new Map<number, number>();
  for (const x of nums) freq.set(x, (freq.get(x) ?? 0) + 1);

  type P = [num: number, f: number];
  const heap: P[] = [];
  const less = (a: P, b: P) => a[1] < b[1];
  const up = (h: P[], i: number) => { for(let p;(i>0)&&less(h[i],h[p=(i-1)>>1]); i=p) [h[i],h[p]]=[h[p],h[i]]; };
  const down = (h: P[], i: number) => { for(;;){ let l=i*2+1,r=l+1,m=i; if(l<h.length&&less(h[l],h[m])) m=l; if(r<h.length&&less(h[r],h[m])) m=r; if(m===i) break; [h[i],h[m]]=[h[m],h[i]]; i=m; } };

  for (const [num, f] of freq) {
    if (heap.length < k) { heap.push([num, f]); up(heap, heap.length - 1); }
    else if (f > heap[0][1]) { heap[0] = [num, f]; down(heap, 0); }
  }
  return heap.sort((a,b)=>b[1]-a[1]).map(x=>x[0]); // 可选：按频次降序
}
```

### 方案 B：桶排序（`O(n)` 期望）
- 最大频次不超过 `n`，将元素按频次丢入 `bucket[f]`。从高频到低频收集，直到取满 `k`。
```ts
export function topKFrequent_bucket(nums: number[], k: number): number[] {
  const freq = new Map<number, number>();
  for (const x of nums) freq.set(x, (freq.get(x) ?? 0) + 1);
  const buckets: number[][] = Array.from({ length: nums.length + 1 }, () => []);
  for (const [num, f] of freq) buckets[f].push(num);
  const res: number[] = [];
  for (let f = buckets.length - 1; f >= 0 && res.length < k; f--) {
    for (const num of buckets[f]) {
      res.push(num);
      if (res.length === k) break;
    }
  }
  return res;
}
```

**选择**  
- `k` 远小于 `n` → 小顶堆；  
- 需要线性时间且可接受额外 `O(n)` 空间 → 桶。  
- 若需**稳定排序/次序**，额外携带键的插入序或次要排序规则。


---

## 34) 位运算技巧：`lowbit`、异或求唯一数、快速幂取模

### 34.1 `lowbit(x)`（最低位 1 的权值）
- **定义**：`x & -x`（二进制补码）得到 `x` 的最低非零位所代表的 2 的幂。  
- **用途**：树状数组（Fenwick）的索引跳跃；枚举子集时快速取出最低位；按位 DP。

```ts
export const lowbit = (x: number) => x & -x;
```

### 34.2 异或求唯一数
- **一个数只出现一次，其余出现两次**：全体异或即得唯一数（`a^a=0, a^0=a`）。
```ts
export const singleNumber = (nums: number[]) => nums.reduce((acc, x) => acc ^ x, 0);
```
- **两个数各出现一次，其余出现两次**：  
  设最终异或 `x = a ^ b`（`a≠b`，故 `x` 至少有一位为 1），取 `diff = lowbit(x)`，按该位将数组分成两组，分别异或得 `a`、`b`。
```ts
export function twoSingles(nums: number[]): [number, number] {
  const x = nums.reduce((acc, v) => acc ^ v, 0);
  const diff = x & -x;
  let a = 0, b = 0;
  for (const v of nums) (v & diff) ? (a ^= v) : (b ^= v);
  return [a, b];
}
```
- **其余出现三次**：逐位统计 1 的个数对 3 取模，还原唯一数（注意 JS 32 位整数范围）。
```ts
export function singleNumber3(nums: number[]): number {
  const cnt = new Array<number>(32).fill(0);
  for (const v of nums) for (let i = 0; i < 32; i++) cnt[i] += (v >>> i) & 1;
  let ans = 0;
  for (let i = 0; i < 32; i++) if (cnt[i] % 3) ans |= (1 << i);
  // 若存在负数，需处理符号位还原：JS 位运算是 32 位带符号
  return ans | 0;
}
```

### 34.3 快速幂取模（幂的二分法 / 平方-乘法）
- 递推：`a^b = (a^(b//2))^2 * (a^(b%2))`，每次把指数二分，乘法次序为 `O(log b)`。  
- 模下乘法需**先取模**避免溢出；在 JS 中可用 `BigInt` 做大数安全模乘。

```ts
export function modPow(a: number, b: number, mod: number): number {
  let base = ((a % mod) + mod) % mod;
  let exp = b >>> 0; // 非负
  let res = 1 % mod;
  while (exp > 0) {
    if (exp & 1) res = (res * base) % mod;
    base = (base * base) % mod;
    exp >>>= 1;
  }
  return res;
}
```

**扩展**  
- 需要**大模数**（> 2^53）：改用 `BigInt`；或用 Montgomery/Barrett 除法优化。  
- 需要**快速幂矩阵**：把乘法替换为矩阵乘法实现 Fibonacci 等线性递推 `O(log n)`。


---

## 35) 随机算法：Fisher–Yates 洗牌 / 蓄水池抽样 / 打乱数组

### 35.1 Fisher–Yates 洗牌（等概率打乱，原地，`O(n)`）
**原理**  
- 从尾到头（或从头到尾），在区间 `[0..i]` 上等概率随机一个 `j` 与 `i` 交换。  
- 归纳可证：每个排列出现概率均为 `1/n!`。  
- **注意**：`array.sort(() => Math.random() - 0.5)` 有偏且实现相关，不可用。

```ts
export function shuffle<T>(a: T[], rng = Math.random): T[] {
  for (let i = a.length - 1; i > 0; i--) {
    const j = Math.floor(rng() * (i + 1)); // 0..i
    [a[i], a[j]] = [a[j], a[i]];
  }
  return a;
}
```
**更稳健的随机源**（浏览器）：`crypto.getRandomValues` 生成 `Uint32` 后取模（处理偏差可用拒绝采样）。

### 35.2 蓄水池抽样（数据流等概率取样，未知/巨大 n）
**问题**  
从**长度未知**或过大无法全部载入内存的数据流中，等概率抽取 `k` 个元素（或 `k=1`）。

**算法（`k=1`）**  
- 读入第 `t` 个元素时，以概率 `1/t` 用它**替换**已有样本。最终每个元素被选中的概率均为 `1/n`。  

```ts
export function reservoirSample1<T>(stream: Iterable<T>): T | null {
  let cnt = 0, sample: T | null = null;
  for (const x of stream) {
    cnt++;
    if (Math.random() * cnt < 1) sample = x; // 概率 1/cnt
  }
  return sample;
}
```

**算法（一般 k）**  
- 先取前 `k` 个进入“蓄水池”；第 `t>k` 个元素以概率 `k/t` 被接纳，若接纳则等概率替换池内一个元素。  

```ts
export function reservoirSampleK<T>(stream: Iterable<T>, K: number): T[] {
  const pool: T[] = [];
  let t = 0;
  for (const x of stream) {
    t++;
    if (pool.length < K) pool.push(x);
    else if (Math.random() * t < K) {
      const j = Math.floor(Math.random() * K);
      pool[j] = x;
    }
  }
  return pool;
}
```

**正确性（素描证明）**  
- 对任一元素 `x_t`，被选中的概率：  
  进入池 `k/t` × 在后续 `(t+1..n)` 都不被替换（每一步留存概率 `1 - k/i`），连乘化简为 `k/n`；`k=1` 则为 `1/n`。

### 35.3 随机打乱 + 采样的工程注意
- **可测试性**：引入可注入 RNG 接口，测试固定种子；避免直接用全局随机源。  
- **偏差**：避免直接 `Math.floor(Math.random()*m)` 在**安全性要求高**的场景（使用 CSPRNG）。  
- **并行/分片**：可用 random shuffle 的**块交换**或在合并阶段进行基于权重的蓄水池以保证全局均匀性。

---
## 36) 最长递增子序列 LIS（贪心 + 二分，`O(n log n)`；带重建）

**问题**  
给定数组 `nums`，返回**严格递增**子序列的最长长度（可选：输出一个最优序列）。若要求**非降序**，只需把比较从 `<` 改为 `<=` 的相反界限。

**原理（耐心排序 / 贪心 + 二分）**  
- 维护数组 `tails[len-1]`：长度为 `len` 的所有递增子序列中，其**最小可能结尾值**。  
- 对每个 `x`，用二分在 `tails` 中找**第一个 ≥ x 的位置**并替换：  
  - 如果 `x` 比当前所有结尾都大 → `tails` 追加 `x`（长度 +1）；  
  - 否则用 `x` 改写该位置，使得同长度的结尾最小化 → 给未来留更大增长空间。  
- 证明要点：`tails` 单调不降，且其长度即为 LIS 长度。

**复杂度**：时间 `O(n log n)`，空间 `O(n)`（若需重建）。

**实现：长度 + 序列重建（TypeScript）**
```ts
// 严格递增版；若要非降序 LIS，把二分里的 >= 改成 >（即 lower_bound 改 upper_bound）
export function lis(nums: number[]): { len: number; seq: number[] } {
  const n = nums.length;
  if (!n) return { len: 0, seq: [] };

  // tailsIdx[k] = 结尾最小的、长度为 k+1 的 LIS 的“末元素下标”
  const tailsIdx: number[] = [];
  // prev[i] = 以 i 作为结尾时，前驱元素的下标（用于回溯重建）
  const prev = new Array<number>(n).fill(-1);

  for (let i = 0; i < n; i++) {
    const x = nums[i];
    // 在 tailsIdx 上二分，比较 nums[tailsIdx[m]] 与 x
    let l = 0, r = tailsIdx.length;
    while (l < r) {
      const m = (l + r) >> 1;
      if (nums[tailsIdx[m]] >= x) r = m; else l = m + 1;
    }
    if (l > 0) prev[i] = tailsIdx[l - 1];
    if (l === tailsIdx.length) tailsIdx.push(i);
    else tailsIdx[l] = i;
  }

  // 回溯重建
  const len = tailsIdx.length;
  const seq = new Array<number>(len);
  let k = tailsIdx[len - 1];
  for (let i = len - 1; i >= 0; i--) {
    seq[i] = nums[k];
    k = prev[k];
  }
  return { len, seq };
}
```

**补充**  
- 朴素 DP：`dp[i]=max(dp[j]+1)` 对所有 `j<i && nums[j]<nums[i]`，`O(n^2)`。  
- 多维限制（如“先按宽升序、宽相等按高降序，再对高求 LIS”）是常见变体；降序用于**去掉相等时的非法嵌套**。


---

## 37) 字符串匹配三板斧：KMP / Z-Algorithm / Rabin–Karp

**目标**：在文本 `T` 中查找模式 `P` 的所有出现位置。

### 37.1 KMP（前缀函数 / 失配表，`O(n+m)`）
**原理**  
- 构造 `pi[i]`：`P[0..i]` 的**最长真前缀**与**后缀**的最大重合长度。  
- 扫描 `T` 时，若 `T[i] ≠ P[j]`，用 `j = pi[j-1]` 回退，避免从头比较。  
- 不变量：`j` 始终为当前已匹配的 `P` 的前缀长度。

```ts
// 返回所有匹配起点
export function kmpSearch(T: string, P: string): number[] {
  if (P.length === 0) return Array.from({length: T.length}, (_, i) => i);
  const pi = new Array<number>(P.length).fill(0);
  for (let i = 1, j = 0; i < P.length; i++) {
    while (j > 0 && P[i] !== P[j]) j = pi[j - 1];
    if (P[i] === P[j]) j++;
    pi[i] = j;
  }
  const ans: number[] = [];
  for (let i = 0, j = 0; i < T.length; i++) {
    while (j > 0 && T[i] !== P[j]) j = pi[j - 1];
    if (T[i] === P[j]) j++;
    if (j === P.length) { ans.push(i - j + 1); j = pi[j - 1]; }
  }
  return ans;
}
```

### 37.2 Z-Algorithm（`Z[i]` 为 `S` 与 `S[i..]` 的最长公共前缀）
**思路**  
- 对 `S = P + '$' + T` 建 `Z` 数组；当 `Z[i] >= |P|` 即匹配。  
- 维护当前 Z-box `[L,R]`，若 `i ≤ R`，先用镜像 `Z[i-L]` 作为下界，再向右扩展。

```ts
export function zSearch(T: string, P: string): number[] {
  const S = P + "$" + T;
  const n = S.length, Z = new Array<number>(n).fill(0);
  for (let i = 1, L = 0, R = 0; i < n; i++) {
    if (i <= R) Z[i] = Math.min(R - i + 1, Z[i - L]);
    while (i + Z[i] < n && S[Z[i]] === S[i + Z[i]]) Z[i]++;
    if (i + Z[i] - 1 > R) { L = i; R = i + Z[i] - 1; }
  }
  const m = P.length, res: number[] = [];
  for (let i = m + 1; i < n; i++) if (Z[i] >= m) res.push(i - (m + 1));
  return res;
}
```

**KMP vs Z**：均线性；KMP只对 `P` 预处理，Z 一次性对连接串建表；工程上任选其一。

### 37.3 Rabin–Karp（滚动哈希，期望 `O(n+m)`）
**思路**  
- 计算窗口长度 `m=|P|` 的滚动哈希，与 `P` 的哈希比对；命中后**再做朴素比对**以防碰撞。  
- 单模会有极小概率碰撞；可用双模或 `BigInt` 提升稳健性。

```ts
// BigInt 版，避免浮点误差；字符映射简单起见用 codePoint
export function rabinKarp(T: string, P: string): number[] {
  const m = P.length, n = T.length;
  if (m === 0) return Array.from({length: n}, (_, i) => i);
  if (n < m) return [];
  const B = 131n, MOD = 1000000007n;
  let pow = 1n, hp = 0n, h = 0n;

  for (let i = 0; i < m; i++) {
    const a = BigInt(P.codePointAt(i)!);
    const b = BigInt(T.codePointAt(i)!);
    hp = (hp * B + a) % MOD;
    h  = (h  * B + b) % MOD;
    if (i) pow = (pow * B) % MOD;
  }

  const res: number[] = [];
  const eq = (i: number) => T.slice(i, i + m) === P; // 二次确认
  if (h === hp && eq(0)) res.push(0);

  for (let i = m; i < n; i++) {
    const add = BigInt(T.codePointAt(i)!);
    const rmv = BigInt(T.codePointAt(i - m)!);
    h = ( (h - rmv * pow % MOD + MOD) % MOD * B + add ) % MOD;
    if (h === hp && eq(i - m + 1)) res.push(i - m + 1);
  }
  return res;
}
```

**何时选 RK**：适合**多模式**（先哈希所有模式）、或在**二维/大块匹配**（如抄袭检测）场景；一般单模式直接用 KMP/Z 更简单。

---

## 38) 会议室问题：最少会议室数 / 能否参加所有会议（扫描线 / 小顶堆）

**问题**  
给定会议区间 `[start, end)`（半开区间），求需要的最少会议室数；若只有一个人参加是否可参加所有会议（即是否有重叠）。

### 方案 A：扫描线（事件 + 计数）
**原理**  
- 把每个区间拆成两个事件：`(+1, start)` 与 `(-1, end)`；按**时间升序，出场在前**排序。  
- 扫描累加计数的最大值即最少会议室数；若最大值 ≤ 1，则“能参加全部”。

```ts
export function minMeetingRooms_sweep(intervals: Array<[number, number]>): number {
  const ev: Array<[number, number]> = [];
  for (const [s, e] of intervals) { ev.push([s, +1]); ev.push([e, -1]); }
  ev.sort((a, b) => (a[0] - b[0]) || (a[1] - b[1])); // 同时刻先 -1 再 +1
  let cur = 0, best = 0;
  for (const [, d] of ev) { cur += d; best = Math.max(best, cur); }
  return best;
}
export const canAttendAll = (intervals: Array<[number, number]>) =>
  minMeetingRooms_sweep(intervals) <= 1;
```

### 方案 B：小顶堆（按会议**结束时间**）
**原理**  
- 按开始时间排序；用小顶堆维护**当前占用的会议室的结束时间**。  
- 对每个会议：若堆顶 `end ≤ start` → 复用这间（弹出再压入新的 `end`）；否则开新房（直接压入）。堆大小最大值即答案。

```ts
class MinHeap {
  private a: number[] = [];
  private up(i:number){ for(let p;(i>0)&&this.a[i]<this.a[p=(i-1)>>1]; i=p) [this.a[i],this.a[p]]=[this.a[p],this.a[i]]; }
  private down(i:number){ for(;;){ let l=i*2+1,r=l+1,m=i; if(l<this.a.length&&this.a[l]<this.a[m]) m=l; if(r<this.a.length&&this.a[r]<this.a[m]) m=r; if(m===i) break; [this.a[i],this.a[m]]=[this.a[m],this.a[i]]; i=m; } }
  push(x:number){ this.a.push(x); this.up(this.a.length-1); }
  top(){ return this.a[0]; }
  pop(){ const t=this.a[0], x=this.a.pop()!; if(this.a.length){ this.a[0]=x; this.down(0); } return t; }
  size(){ return this.a.length; }
}
export function minMeetingRooms_heap(intervals: Array<[number, number]>): number {
  intervals.sort((a,b)=>a[0]-b[0]);
  const h = new MinHeap();
  for (const [s,e] of intervals) {
    if (h.size() && h.top() <= s) h.pop(); // 复用
    h.push(e);
  }
  return h.size();
}
```

**两法等价**：扫描线更易解释“重叠数”；堆法更贴近资源复用语义。

---

## 39) 合并 K 个有序链表/数组（分治 / 小顶堆）

**问题**  
给定 `k` 个有序序列，合并成一个有序序列。链表或数组皆可。

### 方案 A：小顶堆（`O(N log k)`）
- 初始将每个序列的首元素入堆（携带“来自哪个序列及其下标/指针”）。  
- 每次弹出最小元素，接着把该序列的下一元素入堆。

**数组版**
```ts
type Cur = { val: number; i: number; j: number }; // 第 i 个数组的第 j 个
export function mergeKSortedArrays(arrs: number[][]): number[] {
  const res: number[] = [];
  const h: Cur[] = [];
  const less = (a:Cur,b:Cur)=>a.val<b.val;
  const up=(i:number)=>{ for(let p;(i>0)&&less(h[i],h[p=(i-1)>>1]); i=p) [h[i],h[p]]=[h[p],h[i]]; };
  const down=(i:number)=>{ for(;;){ let l=i*2+1,r=l+1,m=i; if(l<h.length&&less(h[l],h[m])) m=l; if(r<h.length&&less(h[r],h[m])) m=r; if(m===i) break; [h[i],h[m]]=[h[m],h[i]]; i=m; } };
  const push=(x:Cur)=>{ h.push(x); up(h.length-1); };
  const pop=():Cur=>{ const t=h[0], x=h.pop()!; if(h.length){ h[0]=x; down(0);} return t; };

  for (let i=0;i<arrs.length;i++) if (arrs[i].length) push({val:arrs[i][0], i, j:0});
  while (h.length) {
    const {val,i,j} = pop(); res.push(val);
    if (j+1 < arrs[i].length) push({val: arrs[i][j+1], i, j:j+1});
  }
  return res;
}
```

**链表版**
```ts
export class ListNode { constructor(public val:number, public next:ListNode|null=null){} }

export function mergeKLists(lists: Array<ListNode | null>): ListNode | null {
  type NodeWrap = { node: ListNode };
  const h: NodeWrap[] = [];
  const less = (a:NodeWrap,b:NodeWrap)=>a.node.val<b.node.val;
  const up=(i:number)=>{ for(let p;(i>0)&&less(h[i],h[p=(i-1)>>1]); i=p) [h[i],h[p]]=[h[p],h[i]]; };
  const down=(i:number)=>{ for(;;){ let l=i*2+1,r=l+1,m=i; if(l<h.length&&less(h[l],h[m])) m=l; if(r<h.length&&less(h[r],h[m])) m=r; if(m===i) break; [h[i],h[m]]=[h[m],h[i]]; i=m; } };
  const push=(x:NodeWrap)=>{ h.push(x); up(h.length-1); };
  const pop=():NodeWrap=>{ const t=h[0], x=h.pop()!; if(h.length){ h[0]=x; down(0);} return t; };

  for (const head of lists) if (head) push({ node: head });
  const dummy = new ListNode(0); let tail = dummy;
  while (h.length) {
    const { node } = pop();
    tail.next = node; tail = node;
    if (node.next) push({ node: node.next });
  }
  tail.next = null;
  return dummy.next;
}
```

### 方案 B：分治归并（pairwise merge，`O(N log k)`）
- 反复两两合并（像归并排序的自底向上），层数 `log k`，每层总处理 `N` 元素。链表版内存友好。

**选择**：堆法实现直观；分治常数更小、易并行化。

---

## 40) 数据流中位数 / 滑动中位数（双堆 + 延迟删除）

### 40.1 数据流中位数（在线维护）
**原理（双堆平衡）**  
- `lo`：最大堆，存较小一半；`hi`：最小堆，存较大一半；保持 `lo.size==hi.size` 或 `lo.size==hi.size+1`。  
- 插入时先进 `lo` 再把堆顶转移到 `hi`，随后如有不平衡再从 `hi` 转回到 `lo`。  
- 中位数：若总数奇数取 `lo.top()`；偶数取 `(lo.top()+hi.top())/2`。

```ts
class Heap<T> {
  constructor(private less:(a:T,b:T)=>boolean, private a:T[]=[]) {}
  push(x:T){ const h=this.a; h.push(x); for(let i=h.length-1,p;(i>0)&&this.less(h[i],h[p=(i-1)>>1]); i=p) [h[i],h[p]]=[h[p],h[i]]; }
  top(){ return this.a[0]; }
  pop():T{ const h=this.a, t=h[0], x=h.pop()!; if(h.length){ h[0]=x; for(let i=0;;){ let l=i*2+1,r=l+1,m=i; if(l<h.length&&this.less(h[l],h[m])) m=l; if(r<h.length&&this.less(h[r],h[m])) m=r; if(m===i) break; [h[i],h[m]]=[h[m],h[i]]; i=m; }} return t; }
  size(){ return this.a.length; }
}
export class MedianFinder {
  private lo = new Heap<number>((a,b)=>a>b); // max-heap
  private hi = new Heap<number>((a,b)=>a<b); // min-heap
  addNum(x:number){
    this.lo.push(x);
    this.hi.push(this.lo.pop());
    if (this.lo.size() < this.hi.size()) this.lo.push(this.hi.pop());
  }
  findMedian(): number {
    return this.lo.size() > this.hi.size()
      ? this.lo.top()
      : (this.lo.top() + this.hi.top()) / 2;
  }
}
```

### 40.2 滑动中位数（窗口大小 `k`）
**难点**：需要从堆中**删除窗口移出的元素**。二叉堆不支持 `O(log n)` 随机删除 → 用**延迟删除**（lazy deletion）+ 计数表：  
- 每次 `remove(x)` 只在计数表 `del[x]++`；当 `x` 恰好位于堆顶时，在 `pop` 前反复“**修剪**”堆顶：若堆顶元素在 `del` 中标为已删除则弹出并减少计数，直到顶端元素有效为止。  
- 插入/删除后按规则再平衡两堆尺寸。

```ts
class DualHeap {
  private lo = new Heap<number>((a,b)=>a>b); // max-heap
  private hi = new Heap<number>((a,b)=>a<b); // min-heap
  private delayed = new Map<number, number>();
  private loSize = 0; // 有效元素计数
  private hiSize = 0;

  constructor(private k: number) {}

  private incr(x:number){ this.delayed.set(x,(this.delayed.get(x)??0)+1); }
  private pruneLo(){
    while (this.lo.size() && (this.delayed.get(this.lo.top()) ?? 0) > 0) {
      const x = this.lo.pop(); this.delayed.set(x, (this.delayed.get(x) ?? 0) - 1);
    }
  }
  private pruneHi(){
    while (this.hi.size() && (this.delayed.get(this.hi.top()) ?? 0) > 0) {
      const x = this.hi.pop(); this.delayed.set(x, (this.delayed.get(x) ?? 0) - 1);
    }
  }
  private rebalance(){
    // 目标：loSize == hiSize 或 loSize == hiSize + 1
    if (this.loSize > this.hiSize + 1) {
      this.hi.push(this.lo.pop()); this.loSize--; this.hiSize++;
      this.pruneLo();
    } else if (this.loSize < this.hiSize) {
      this.lo.push(this.hi.pop()); this.loSize++; this.hiSize--;
      this.pruneHi();
    }
  }
  insert(x:number){
    if (!this.lo.size() || x <= this.lo.top()) { this.lo.push(x); this.loSize++; }
    else { this.hi.push(x); this.hiSize++; }
    this.rebalance();
  }
  erase(x:number){
    this.incr(x);
    if (this.lo.size() && x <= this.lo.top()) { this.loSize--; if (x === this.lo.top()) this.pruneLo(); }
    else { this.hiSize--; if (this.hi.size() && x === this.hi.top()) this.pruneHi(); }
    this.rebalance();
  }
  median(): number {
    return (this.k & 1) ? this.lo.top() : (this.lo.top() + this.hi.top()) / 2;
  }
}

export function medianSlidingWindow(nums:number[], k:number): number[] {
  const dh = new DualHeap(k), res: number[] = [];
  for (let i=0;i<nums.length;i++){
    dh.insert(nums[i]);
    if (i >= k-1) {
      res.push(dh.median());
      dh.erase(nums[i-k+1]);
    }
  }
  return res;
}
```

**复杂度**：插入/删除/修剪均摊 `O(log k)`；每元素至多经历一次入堆、一次延迟删除、一次被实际弹出。

**工程注意**  
- 若数据范围很大、存在大量重复值，延迟 map 规模可与窗口同级别；若值域较小可用计数堆/有序多重集（如平衡树）。  
- 需要**精确中位数**（整数/浮点）按题意返回；偶数窗口取两堆顶平均。

---
## 41) 两个有序数组的中位数（对数级二分，`O(log min(m,n))`）

**问题**  
给定两段升序数组 `A(m)`、`B(n)`，求合并后长度 `m+n` 的中位数（偶数长度取两中位的平均）。

**原理（划分法 / 二分较短数组）**  
- 设总左侧元素数 `left = ⌊(m+n+1)/2⌋`。在较短数组 `A` 上二分一个切分点 `i`，令 `j = left - i`，构成：
  ```
  A: [A0 ... A(i-1)] | [A(i) ...]
  B: [B0 ... B(j-1)] | [B(j) ...]
  ```
- 充分必要条件（“合法划分”）：`A[i-1] ≤ B[j]` 且 `B[j-1] ≤ A[i]`。  
  成立时：  
  - 若 `(m+n)` 为奇数，中位数 `= max(A[i-1], B[j-1])`；  
  - 若为偶数，中位数 `= (max(A[i-1], B[j-1]) + min(A[i], B[j])) / 2`。
- 若 `A[i-1] > B[j]`，说明 `i` 太大，向左缩；反之 `i` 太小，向右扩。
- 边界用哨兵：`A[-1]=B[-1]=-∞`，`A[m]=B[n]=+∞`。

**实现（TypeScript）**
```ts
export function findMedianSortedArrays(A: number[], B: number[]): number {
  if (A.length > B.length) return findMedianSortedArrays(B, A);
  const m = A.length, n = B.length;
  let l = 0, r = m;
  const NEG = -Infinity, POS = Infinity;
  const left = Math.floor((m + n + 1) / 2);

  while (l <= r) {
    const i = (l + r) >> 1;
    const j = left - i;

    const Aleft  = i > 0 ? A[i - 1] : NEG;
    const Aright = i < m ? A[i]     : POS;
    const Bleft  = j > 0 ? B[j - 1] : NEG;
    const Bright = j < n ? B[j]     : POS;

    if (Aleft <= Bright && Bleft <= Aright) {
      if ((m + n) % 2 === 1) return Math.max(Aleft, Bleft);
      return (Math.max(Aleft, Bleft) + Math.min(Aright, Bright)) / 2;
    } else if (Aleft > Bright) {
      r = i - 1;
    } else {
      l = i + 1;
    }
  }
  throw new Error("Invalid input");
}
```

**复杂度**：二分较短数组，`O(log min(m,n))`。  
**正确性要点**：合法划分 ⇔ 左侧最大 ≤ 右侧最小；将“合并后中位数问题”转化为“左右两边元素计数平衡”。

---

## 42) 单词拆分（`Word Break`，DP / 记忆化 / Trie 优化）

**问题**  
判断 `s` 能否被词典 `dict` 中的若干词**完全拼接**（可重复使用）。可延伸返回一种拆分方案。

**原理 A（DP，区间可行性）**  
- `dp[i]`：前缀 `s[0..i)` 可拆分。  
- 转移：若存在 `j<i`，且 `dp[j]=true` 且 `s[j..i)` 在词典中，则 `dp[i]=true`。  
- 为降复杂度，使用 `minLen/maxLen` 限制 `j` 的范围。

```ts
export function wordBreak(s: string, wordDict: string[]): boolean {
  const set = new Set(wordDict);
  const n = s.length;
  let minLen = Infinity, maxLen = 0;
  for (const w of set) { minLen = Math.min(minLen, w.length); maxLen = Math.max(maxLen, w.length); }
  const dp = new Array<boolean>(n + 1).fill(false);
  dp[0] = true;
  for (let i = 1; i <= n; i++) {
    for (let len = minLen; len <= maxLen; len++) {
      const j = i - len;
      if (j < 0) break;
      if (dp[j] && set.has(s.slice(j, i))) { dp[i] = true; break; }
    }
  }
  return dp[n];
}
```

**原理 B（记忆化 DFS）**  
- `can(i)`：能否拆分 `s[i..]`；对所有词 `w`，若 `s` 从 `i` 起以 `w` 开头，递归 `can(i+|w|)`。  
- 记忆化避免指数爆炸，适合**字典较小**而 `s` 较长。可结合 Trie 加速前缀匹配。

**返回一种拆分（回溯前驱）**
```ts
export function wordBreakOneSplit(s: string, dict: string[]): string[] | null {
  const set = new Set(dict), n = s.length;
  const prev = new Array<number>(n + 1).fill(-1);
  const dp = new Array<boolean>(n + 1).fill(false); dp[0] = true;
  let minLen = Infinity, maxLen = 0;
  for (const w of set) { minLen = Math.min(minLen, w.length); maxLen = Math.max(maxLen, w.length); }

  for (let i = 1; i <= n; i++) {
    for (let len = minLen; len <= maxLen; len++) {
      const j = i - len; if (j < 0) break;
      if (dp[j] && set.has(s.slice(j, i))) { dp[i] = true; prev[i] = j; break; }
    }
  }
  if (!dp[n]) return null;
  const parts: string[] = [];
  for (let i = n; i > 0; i = prev[i]) parts.push(s.slice(prev[i], i));
  return parts.reverse();
}
```

**复杂度**：DP `O(n * (#len in [minLen,maxLen]))`；记忆化依字典大小与切分点而定。  
**要点**：剪枝（长度边界）、`Set` 查找 `O(1)`、必要时用 Trie 降低字符拷贝与切片次数。

---

## 43) 单词接龙（`Word Ladder`，BFS / 双向 BFS）

**问题**  
给定 `beginWord`、`endWord` 与词表 `wordList`，每次可改变一个字母，要求每步都在词表中，求**最短变换序列长度**（含起点与终点）。

**原理 A（BFS，按层扩展）**  
- 将词当作图的节点，两词相差 1 字母连边；最短路即 BFS 层数。  
- 在线生成邻居：对每个位置尝试 26 个字母替换，若在词典中则为邻居。

**原理 B（双向 BFS）**  
- 同时从 `begin` 与 `end` 两端扩展，每次扩展较小的那一端；当两端有交集时返回深度和。  
- 复杂度显著降低，适合词表大、分支度高。

**实现（双向 BFS，TypeScript）**
```ts
export function ladderLength(beginWord: string, endWord: string, wordList: string[]): number {
  const dict = new Set(wordList);
  if (!dict.has(endWord)) return 0;
  let front = new Set<string>([beginWord]);
  let back = new Set<string>([endWord]);
  const L = beginWord.length;
  let steps = 1;

  const alpha = Array.from({length:26}, (_,i)=>String.fromCharCode(97+i));

  while (front.size && back.size) {
    // 始终扩展较小集合
    if (front.size > back.size) [front, back] = [back, front];
    const nxt = new Set<string>();
    for (const w of front) {
      const arr = w.split('');
      for (let i = 0; i < L; i++) {
        const old = arr[i];
        for (const ch of alpha) {
          if (ch === old) continue;
          arr[i] = ch;
          const v = arr.join('');
          if (back.has(v)) return steps + 1;
          if (dict.has(v)) { nxt.add(v); dict.delete(v); }
        }
        arr[i] = old;
      }
    }
    front = nxt;
    steps++;
  }
  return 0;
}
```

**复杂度**：最坏近似 `O(Σ * L * N)`，双向能将层数近似减半。  
**注意**：每个新入队单词要从 `dict` 删除，避免重复访问与环。

---

## 44) 网格最短路径（最多移除 `K` 个障碍，BFS 状态扩展）

**问题**  
`grid[m][n]` 中 `0` 为空地、`1` 为障碍，从 `(0,0)` 到 `(m-1,n-1)`，可至多移除 `k` 个障碍，求最短步数；若不可达返回 `-1`。

**原理（BFS on 状态 `(r,c,remK)`）**  
- 普通 BFS 加一维“剩余可移除数”作为状态。  
- **剪枝**：对每个格 `(r,c)` 只保留“到达时 `remK` 的最大值”，若后来以**更少**的 `remK` 再来便无意义。  
- **优化**：若 `k >= (m-1)+(n-1)`，可直接返回曼哈顿距离（最短路长度），因为最短路径上最多遇到 `m+n-2` 个障碍。

**实现（TypeScript）**
```ts
export function shortestPathEliminateObstacles(grid: number[][], k: number): number {
  const m = grid.length, n = grid[0].length;
  if (m === 1 && n === 1) return 0;
  const minSteps = (m - 1) + (n - 1);
  if (k >= minSteps) return minSteps;

  const best: number[][] = Array.from({ length: m }, () => new Array<number>(n).fill(-1));
  const q: Array<[number, number, number, number]> = []; // r,c,rem,dist
  q.push([0, 0, k, 0]); best[0][0] = k;

  const DIR = [[1,0],[-1,0],[0,1],[0,-1]] as const;
  for (let qi = 0; qi < q.length; qi++) {
    const [r, c, rem, d] = q[qi];
    for (const [dr, dc] of DIR) {
      const nr = r + dr, nc = c + dc;
      if (nr < 0 || nr >= m || nc < 0 || nc >= n) continue;
      let nrem = rem - (grid[nr][nc] === 1 ? 1 : 0);
      if (nrem < 0) continue;
      if (nr === m - 1 && nc === n - 1) return d + 1;
      if (nrem <= best[nr][nc]) continue; // 已有更优（剩余更大）的状态
      best[nr][nc] = nrem;
      q.push([nr, nc, nrem, d + 1]);
    }
  }
  return -1;
}
```

**复杂度**：`O(m*n*(k+1))` 状态，BFS 线性扩展。  
**要点**：状态支配关系（同一格保留“剩余 k 最大”的到达）确保剪枝正确性。

---

## 45) 不同子序列个数（`Distinct Subsequences`，DP，可能需大整数）

**问题**  
给定字符串 `S`、目标 `T`，求 `S` 的不同**子序列**中等于 `T` 的个数。计数可能很大。

**原理（前缀 DP，一维滚动）**  
- `dp[j]` 表示用 `S` 的当前前缀，构成 `T[0..j)` 的方案数。初始化 `dp[0]=1`（空串只有 1 种）。  
- 逐个扫描 `S` 的字符 `s`，对 `j` **从大到小**更新：  
  `if (s === T[j-1]) dp[j] += dp[j-1]`。  
- 逆序的原因：避免当前轮 `dp[j-1]` 已被本轮覆盖。

**实现（`BigInt` 版，避免溢出）**
```ts
export function numDistinct(S: string, T: string): bigint {
  const m = T.length;
  const dp: bigint[] = new Array(m + 1).fill(0n);
  dp[0] = 1n;
  for (let i = 0; i < S.length; i++) {
    const ch = S[i];
    for (let j = m; j >= 1; j--) {
      if (ch === T[j - 1]) dp[j] = dp[j] + dp[j - 1];
    }
  }
  return dp[m];
}
// 若题目要求返回 number，可在结果 < Number.MAX_SAFE_INTEGER 时转换，否则返回字符串 dp[m].toString()
```

**复杂度**：时间 `O(|S|·|T|)`，空间 `O(|T|)`。  
**优化**：当 `T` 中字符分布稀疏，可为 `T` 预建“字符到下标列表”的映射，遍历 `S` 时仅更新涉及的 `j`（仍需逆序）。

---
## 46) 最大子数组和 / 环形最大子数组和（Kadane + 取反求最小）

**问题**  
- 最大子数组和（连续）：求 `max_{l≤r} Σ_{i=l..r} a[i]`。  
- 环形版本：数组头尾可相连（形成一个环）的最大连续子数组和。

**原理 A：Kadane（线性）**  
- 维护以 `i` 结尾的**最佳前缀** `bestEnd`：`bestEnd = max(a[i], bestEnd + a[i])`。  
- 全局最优 `best = max(best, bestEnd)`。  
- 直觉：若 `bestEnd` 为负，延长只有害处，直接从 `a[i]` 重新开始。

**原理 B：环形最大**  
- 要么不跨环（直接 Kadane）；  
- 要么跨环：等价于**总和**减去**中间一段的最小子数组和**。  
  - 用 Kadane 求最小子数组和：对 `-a[i]` 求最大子数组和，再取相反数。  
- 特殊情况：若全为负数，则“总和 - 最小和”会把整个数组都剔除，答案应是**普通 Kadane**的最大单元素。

**实现（TypeScript）**
```ts
export function maxSubArray(nums: number[]): number {
  let best = -Infinity, end = 0;
  for (const x of nums) {
    end = Math.max(x, end + x);
    best = Math.max(best, end);
  }
  return best;
}

export function maxSubarraySumCircular(nums: number[]): number {
  const nonWrap = maxSubArray(nums);
  let total = 0, endMax = 0, bestMax = -Infinity; // Kadane 求最大
  let endMin = 0, bestMin = Infinity;             // Kadane 求最小
  for (const x of nums) {
    total += x;
    endMax = Math.max(x, endMax + x); bestMax = Math.max(bestMax, endMax);
    endMin = Math.min(x, endMin + x); bestMin = Math.min(bestMin, endMin);
  }
  // 若全负：bestMax === maximum element，且 bestMin === total
  const wrap = total - bestMin;
  return bestMin === total ? nonWrap : Math.max(nonWrap, wrap);
}
```

**复杂度**：均 `O(n)` 时间、`O(1)` 空间。  
**易错点**：环形题目必须处理“全负”特判，否则会错误返回 0（空区间）。

---

## 47) 零钱兑换 I/II（完全背包 DP）

**问题**  
- I（最少硬币数）：面额 `coins[]`，金额 `amount`，每种面额可无限次使用，求最少张数；无解返回 `-1`。  
- II（组合数量）：同样输入，求**组合数**（不计顺序）。

**原理**  
- 完全背包（物品可重复）：遍历**硬币外层**、金额**升序**，表示可以多次使用当前硬币。  
- I（最少张数）是**最优化**问题；II（组合数）是**计数**问题。  
- II 为“组合”，故**硬币在外层**以避免排列重复；若把金额外层/硬币内层，会统计到**排列数**。

**实现（TypeScript）**
```ts
// I: 最少张数
export function coinChangeMin(coins: number[], amount: number): number {
  const INF = amount + 1;
  const dp = new Array<number>(amount + 1).fill(INF);
  dp[0] = 0;
  for (const c of coins) {
    for (let v = c; v <= amount; v++) {
      dp[v] = Math.min(dp[v], dp[v - c] + 1);
    }
  }
  return dp[amount] === INF ? -1 : dp[amount];
}

// II: 组合数量（不计顺序）
export function coinChangeCount(coins: number[], amount: number): number {
  const dp = new Array<number>(amount + 1).fill(0);
  dp[0] = 1;
  for (const c of coins) {
    for (let v = c; v <= amount; v++) {
      dp[v] += dp[v - c];
    }
  }
  return dp[amount];
}
```

**复杂度**：两题均 `O(N·amount)`，`N=coins.length`；空间 `O(amount)`。  
**易错点**：I 题初始化需用不可达大值；II 题**硬币外层**保证组合不重复。

---

## 48) 0/1 背包（容量约束最大价值，滚动数组）

**问题**  
`n` 件物品，第 `i` 件重 `w[i]`、值 `v[i]`，背包容量 `W`，每件最多取 0/1 次，最大化总价值。

**原理（经典 DP）**  
- 二维：`dp[i][cap]` 表示考虑前 `i` 件、容量为 `cap` 的最优值。  
- 一维滚动：`dp[cap]` 表示当前已处理到某件时容量 `cap` 的最优值；**容量需从大到小**迭代，防止本轮结果被重复使用（区别于完全背包）。

**实现（TypeScript，一维滚动）**
```ts
export function knap01(weights: number[], values: number[], W: number): number {
  const n = weights.length;
  const dp = new Array<number>(W + 1).fill(0);
  for (let i = 0; i < n; i++) {
    const w = weights[i], val = values[i];
    for (let cap = W; cap >= w; cap--) {
      dp[cap] = Math.max(dp[cap], dp[cap - w] + val);
    }
  }
  return dp[W];
}
```

**重建方案（可选）**  
- 用二维 `dp[i][cap]` 或保留“选择”布尔表 `take[i][cap]`；从 `i=n..1` 回溯：若 `dp[i][cap] === dp[i-1][cap - w[i]] + v[i]` 则取之。

**复杂度**：`O(nW)` 时间，空间 `O(W)`（滚动）。  
**对比**：完全背包容量从小到大；0/1 背包需从大到小迭代。

---

## 49) Aho–Corasick 自动机（多模式匹配，Trie + 失败指针）

**问题**  
给定模式集合 `P1..Pk` 与文本 `T`，找出所有出现（或统计次数）。

**原理**  
1) 构造 **Trie**：每个模式插入，叶子/节点保存该节点终止的模式索引列表（可多模式共享尾部）。  
2) **失败指针（fail）**：类似 KMP 的“最长真后缀=前缀”。按层 BFS 建：  
   - 根的子节点 `fail=根`；  
   - 对边 `u --ch--> v`，`fail[v] = go(fail[u], ch)`（若无则继续沿 `fail` 链回退至根）。  
   - 同时把 `fail[v]` 的输出合并到 `v`（匹配到更短后缀的模式）。  
3) **匹配**：在文本上按字符 `ch` 转移 `state = go(state, ch)`（如无边则沿 `fail` 回退）；在每个状态输出其携带的模式命中。

**实现（TypeScript，计数每个模式出现次数）**
```ts
type Node = {
  next: Map<string, number>; // char -> child index
  fail: number;
  out: number[]; // 模式索引列表
};
export class AhoCorasick {
  private t: Node[] = [{ next: new Map(), fail: 0, out: [] }];

  add(word: string, id: number) {
    let p = 0;
    for (const ch of word) {
      if (!this.t[p].next.has(ch)) {
        this.t[p].next.set(ch, this.t.length);
        this.t.push({ next: new Map(), fail: 0, out: [] });
      }
      p = this.t[p].next.get(ch)!;
    }
    this.t[p].out.push(id);
  }

  build() {
    const q: number[] = [];
    // 根的第一层
    for (const [ch, v] of this.t[0].next) {
      this.t[v].fail = 0;
      q.push(v);
    }
    // BFS
    for (let qi = 0; qi < q.length; qi++) {
      const u = q[qi];
      for (const [ch, v] of this.t[u].next) {
        // 计算 v 的 fail
        let f = this.t[u].fail;
        while (f && !this.t[f].next.has(ch)) f = this.t[f].fail;
        if (this.t[f].next.has(ch)) f = this.t[f].next.get(ch)!;
        this.t[v].fail = f;
        // 合并输出（匹配更短后缀）
        this.t[v].out.push(...this.t[f].out);
        q.push(v);
      }
    }
  }

  // 返回每个模式的出现次数；patterns[] 用于确定数量
  search(text: string, patterns: string[]): number[] {
    const cnt = new Array<number>(patterns.length).fill(0);
    let s = 0;
    for (const ch of text) {
      while (s && !this.t[s].next.has(ch)) s = this.t[s].fail;
      if (this.t[s].next.has(ch)) s = this.t[s].next.get(ch)!;
      for (const id of this.t[s].out) cnt[id]++;
    }
    return cnt;
  }
}
```

**复杂度**：构建 `O(Σ|Pi|)`；匹配 `O(|T| + #匹配)`。  
**工程化**：  
- 字母表大时 `Map` 更灵活；定长字母表可用数组快些。  
- 若需输出所有**起始位置**，在匹配时记录索引并减去模式长度 + 1。

---

## 50) 线段树 / 树状数组（Fenwick）：区间查询与更新

**问题**  
维护序列上的**动态**聚合（如区间和/最值）与更新。  
- **树状数组（Fenwick）**：支持**前缀和**与**单点加**，`O(log n)`；通过两次前缀和得到区间和。  
- **线段树**：更通用；支持区间和/最值/最小值等，扩展**区间更新 + 懒标记**。

### 50.1 Fenwick（原理与实现）
**原理**  
- `idx & -idx`（`lowbit`）给出树状数组节点的覆盖长度；`BIT[i]` 维护区间 `(i - lowbit(i), i]` 的和。  
- `prefix(i)`：不断 `i -= lowbit(i)` 汇总；`add(i, delta)`：不断 `i += lowbit(i)` 传播。

```ts
export class Fenwick {
  private bit: number[];
  constructor(private n: number) { this.bit = new Array(n + 1).fill(0); }
  private lowbit(x: number) { return x & -x; }
  add(i: number, delta: number) {
    for (let x = i + 1; x <= this.n; x += this.lowbit(x)) this.bit[x] += delta;
  }
  prefix(r: number): number { // [0..r] 的和（r 基于 0）
    let s = 0;
    for (let x = r + 1; x > 0; x -= this.lowbit(x)) s += this.bit[x];
    return s;
  }
  range(l: number, r: number): number {
    if (l > r) return 0;
    return this.prefix(r) - (l ? this.prefix(l - 1) : 0);
  }
}
```

**扩展**  
- 要支持**区间加 / 区间和**：用两个 BIT 组合（差分思想），或在线段树做懒传播更直观。

### 50.2 线段树（区间和 + 区间加的懒标记）

**原理**  
- 每个节点维护其区间 `[L,R]` 的聚合（如和）；更新/查询若完全覆盖则直接使用节点值；**部分覆盖**则递归到左右子区间并合并。  
- **懒标记**：当对整段 `[L,R]` 进行区间加 `+v` 时，只在当前节点累加 `lazy += v`、`sum += v*(R-L+1)`，**延迟**到需要时向子节点下推（`pushDown`）。

```ts
export class SegTree {
  private n: number;
  private sum: number[];
  private lazy: number[];
  constructor(arr: number[]) {
    this.n = arr.length;
    this.sum = new Array(this.n * 4).fill(0);
    this.lazy = new Array(this.n * 4).fill(0);
    if (this.n) this.build(1, 0, this.n - 1, arr);
  }
  private build(o: number, L: number, R: number, a: number[]) {
    if (L === R) { this.sum[o] = a[L]; return; }
    const M = (L + R) >> 1, ls = o << 1, rs = ls | 1;
    this.build(ls, L, M, a); this.build(rs, M + 1, R, a);
    this.pull(o);
  }
  private pull(o: number) { this.sum[o] = this.sum[o << 1] + this.sum[(o << 1) | 1]; }
  private push(o: number, L: number, R: number, v: number) {
    this.sum[o] += v * (R - L + 1);
    this.lazy[o] += v;
  }
  private pushDown(o: number, L: number, R: number) {
    const tag = this.lazy[o];
    if (!tag) return;
    const M = (L + R) >> 1, ls = o << 1, rs = ls | 1;
    this.push(ls, L, M, tag);
    this.push(rs, M + 1, R, tag);
    this.lazy[o] = 0;
  }
  // 对 [l,r] 加 v
  update(l: number, r: number, v: number, o = 1, L = 0, R = this.n - 1) {
    if (l > R || r < L) return;
    if (l <= L && R <= r) { this.push(o, L, R, v); return; }
    this.pushDown(o, L, R);
    const M = (L + R) >> 1;
    this.update(l, r, v, o << 1, L, M);
    this.update(l, r, v, (o << 1) | 1, M + 1, R);
    this.pull(o);
  }
  // 查询 [l,r] 的和
  query(l: number, r: number, o = 1, L = 0, R = this.n - 1): number {
    if (l > R || r < L) return 0;
    if (l <= L && R <= r) return this.sum[o];
    this.pushDown(o, L, R);
    const M = (L + R) >> 1;
    return this.query(l, r, o << 1, L, M) + this.query(l, r, (o << 1) | 1, M + 1, R);
  }
}
```

**对比与选型**  
- **Fenwick**：实现简单、常数小；适合**前缀和/区间和 + 单点更新**，或通过两树实现**区间加/区间和**。  
- **线段树**：更灵活，天然支持**区间操作**（加、赋值、最大/最小、区间最大子段和等），可加**懒标记**处理 `O(log n)` 级区间更新。

**与前缀和/差分的边界**  
- **静态**多次区间和查询 → 前缀和；  
- **多次区间加，最后一次性还原** → 差分；  
- **在线更新 + 在线查询** → Fenwick/线段树（按操作类型择优）。

---
