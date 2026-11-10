# 从 Java / Python 到这个 React 项目

> **这份先别读。** 里面词多、话绕，是留着查阅的手册，不是入门材料。
>
> 想弄明白这个项目怎么跑 → 只读 `docs/run-through.md`（一份，白话，按时间顺序跟着那份名单走一遍）。那份最后附了一张「这些词其实就是这个意思」的对照表，遇到「挂载 / 槽位 / 重渲染」回去查一眼就行。
>
> 等 run-through 读顺了、想深挖某一个「为什么」，再回来按小节查这份。

## 先读这一页：一次运行，五个阶段

只跟一个主角：**那份员工名单**。从头到尾只有它。

| 阶段 | 名单此刻 | 想深挖再查 |
| --- | --- | --- |
| ① 打开网页 | 文件里写死的 10 条被造出来 | §1.7 / §3.4 |
| ② 登录，表格第一次出现 | 一开始用的就是那 10 条；随后表格每出现一次就读一次存档，可能换成另一份 | §1.6 / §3.3 |
| ③ 表格第一次画出来 | 每行编号被刷成 `1,2,3,…` | §3.2 |
| ④ 点 Add 提交 | 直接往那份里塞一条 → 存档写成一行字 → 表格重新造一遍，屏幕才多出一行 | §3.1 |
| ⑤ 按 F5 | 页面整个重来、② 重播；存档是空的时候 10 条种子会回来 | §3.3 / §3.4 |

一句话：**§3.1 / §3.2 / §3.3 / §3.4 不是四个话题，是同一份名单在 ④ ③ ⑤ ② 这四个时刻的样子。**

> 五步完整剧本 + 动手验证在 `docs/run-through.md`。

---

## 0. 先诊断：你不是"完全不懂"

把"懂一个项目"拆成三层，你的位置很清楚：

| 层 | 内容 | 你的状态 |
| --- | --- | --- |
| ① 功能层 | 有登录、有表格、能增删改、刷新后数据还在 | **懂了** |
| ② 结构层 | 文件怎么分、状态放在谁手里、两个 localStorage key | **大半懂了** |
| ③ 运行时层 | 谁触发重渲染、state 是引用还是值、render 什么时候跑、内存和存储是不是同一份数据 | **基本没建立** |

上一轮六道题，凡是考 ①② 的你都能搭上话（"只有点 Add/Edit 才会 push"、"setEmployees 触发了重渲染"其实也是在这两层的语感里猜的），凡是考 ③ 的全塌。**这不是能力问题，是缺一层模型。**

更麻烦的是 AI 生成的解析会放大这个错觉：它把因果写成完成的句子，读起来一路通顺，大脑很容易把"这句话我读懂了"登记成"这个机制我想通了"。判断有没有真通，只有一个标准——**能不能回答"如果把这行删掉会怎样"**。上一轮六道题都在问这个，不是故意刁难。

好消息：你缺的这层，Java 里全学过。`==` 比引用、`final List` 仍能 `add`、方法参数传的是引用——React 在这几点上跟你熟悉的东西几乎一模一样，只差有人把它点破。

**这份文档不做什么**：不重复逐文件走读、不列缺陷清单、不展开改进方向（那些留到这层模型建起来之后）。

---

## 1. 地基：八个 Java / Python 类比

### 1.1 组件 = 一个会被反复调用的函数

```java
// Java 心智里，"组件"很容易被想成一个对象/一个页面
class Table { void render() { ... } }
```

实际上它更接近一个**纯函数**：

```js
function Table(props) {
  // 算出一分"界面应该长什么样"的描述，返回它
  return <table>...</table>;
}
```

React 在需要的时候**调用这个函数**，拿到描述，再去改真实 DOM。关键点：它会被调用**很多次**，不是一次。

### 1.2 重渲染 = 把这个函数再跑一遍

"重渲染"这三个字最容易被误解成"刷新页面/重新加载"。都不是。它只是：

> 组件函数重新执行一次 → 得到新的 UI 描述 → React 和上一次的描述比对 → 只改有差异的 DOM。

所以重渲染时：**不会**重新读 localStorage、**不会**重新执行模块文件、**不会**重建模块级常量。它只是函数又跑了一遍。

顺带把三个词分清楚（中文译名容易混，英文其实是三个词）：**挂载 mount** = 组件第一次被创建并放进页面，此时 `useState` 拿初值、`useEffect(..., [])` 跑一次；**重渲染 re-render** = 同一个实例再算一遍界面，state 沿用，挂载时那个 effect 不会再跑；**卸载 unmount** = 从页面移除，内部 state 全部销毁，下次出现是全新实例。**「挂载」跟操作系统挂载磁盘没有任何关系**，只是同一个英文词被译成了同一个中文词。

### 1.3 state = 这个函数的记忆，而且只有 setState 才让函数重跑

```js
const [isAdding, setIsAdding] = useState(false);
```

`useState` 每次渲染返回"这一次的那个值"。你直接改变量没用，必须调 `setXxx(...)`，React 才会安排一次新的函数调用。

**类比**：像是给函数配了个外部小本本，函数自己不能改本本，得递张纸条（`setXxx`）给框架，框架改完本本再把函数叫回来重跑一次。

### 1.4 引用相等 = Java 的 `==` / Python 的 `is`（**最关键的一条**）

```java
List<Employee> a = employees;   // 同一个引用，不是拷贝
a.add(newEmployee);             // 内容变了，引用没变
setEmployees(a);                // 框架若用 == 判断"变没变" → 看不出变化
```

React 判断 state 有没有变，用的就是 `Object.is(prev, next)`。对数组/对象而言，**比较的是引用，不是内容**：

```js
employees.push(newEmployee);   // 内容变了，引用没变
setEmployees(employees);       // prev === next → React 认为没变 → bail out（跳过）
```

`bail out` 不是 bug，是性能优化：既然你觉得是同一个东西，我就不费劲重算了。**React 不会去看数组里多没多一个元素。**

> 自己推一遍：`const a = [1]; const b = a; b.push(2);` 此时 `Object.is(a, b)` 是 true 还是 false？
>
> **答案**：true。两个变量指向同一个数组对象。`push` 改的是对象内部，不是变量指向。整个项目一半的怪现象都来自这一句。

### 1.5 props = 传引用，不是传值拷贝

```java
void render(List<Employee> list) { list.add(x); }  // 调用方的 list 也变了
```

`<Table employees={employees} />` 同理：Table 拿到的就是 Dashboard 里那个数组**本体**，不是副本。所以 `Table.js:4-6` 在 Table 里改 `employee.id`，改的是 Dashboard state 里的对象，甚至改到模块常量（见 3.4）。

### 1.6 localStorage = 一个 `Map<String, String>`，不是数据库

```java
Map<String, String> storage;   // 只能存字符串
storage.put("employees_data", jsonString);
```

它没有"表"、没有"连接"、不会自动同步、改动不会通知谁。写进去就是**一份快照**，此后内存里的数组怎么变，跟这份字符串没有任何关系，直到你下一次 `getItem` 主动读回来。

还有一层最容易忽略：`JSON.stringify(array)` 存的是**文本拷贝**，不是对象本身；`JSON.parse` 读回来会**造一个全新的数组**，`Object.is` 跟写进去之前那个数组必然不等。相当于 Java 序列化到文件再反序列化——出来的是另一个对象。所以"我把数组存进去了"永远要翻译成"我把数组的一份文本快照存进去了"：对象和当前界面之间的引用链，在这一步被彻底切断。写库只保证**下一次表格出现时能读回来**（按 F5、或登出再登录），跟**这一次界面显示什么**没有任何关系。

顺带：`setItem('is_authenticated', true)` 里的 `true` 会被自动转成字符串 `"true"`，所以 `JSON.parse` 能还原成布尔。

### 1.7 `const` 只锁绑定，不锁内容

```java
final List<Employee> employeesData = new ArrayList<>();
employeesData.add(x);   // 完全合法
```

JS 的 `const employeesData = [...]` 一模一样：`const` 不许你把变量指向另一个数组，但**随便 push / splice / 改对象字段**。所以"它是常量"和"它是只读种子"是两回事。

### 1.8 render 必须"纯"

因为函数会被调用很多次、时机由框架决定，React 要求：**算描述的过程中不要改入参、不要写存储、不要发请求**。改了就是"副作用"，结果会依赖"到底被调用了几次"——而这不是你能控制的。

`Table.js:4-6` 违反了这一条，这是全项目最刺眼的三行。

---

## 2. 这个项目的四行骨架

```
一个开关   isAuthenticated   →  显示 <Login/> 还是 <Dashboard/>
一份数组   employees         →  全部业务数据，Dashboard 独有
两个布尔   isAdding/isEditing →  表格页里显示 列表 / 新增表单 / 编辑表单（互斥三段）
两个 key   localStorage      →  is_authenticated、employees_data（持久化快照）
```

```
index.js  →  App (isAuthenticated)
               ├─ false/null →  Login (email, password)
               └─ true       →  Dashboard (employees, selectedEmployee, isAdding, isEditing)
                                  ├─ Header → Logout
                                  ├─ Table   （列表 + 行内 Edit / Delete）
                                  ├─ Add     （新增表单）
                                  └─ Edit    （编辑表单）
```

一次完整操作长什么样（点 Add 提交）：

1. 你在 `<Add/>` 里填了五个字段，每个字段是 **Add 自己的 state**。
2. 提交 → `handleAdd` 校验 → `const id = employees.length + 1` → 造一个新对象。
3. `employees.push(newEmployee)` —— **就地改了 Dashboard 的那份数组**（也顺手改了模块常量，见 3.4）。
4. `localStorage.setItem(...)` —— 把当前数组序列化成字符串，存一份快照。
5. `setEmployees(employees)` —— 同一个引用，**bail out，什么都不做**。
6. `setIsAdding(false)` —— **这个才让 Dashboard 重新跑一遍**：里面那个位置从 `<Add/>` 换成 `<Header/>` + `<Table/>`，Add 被拆掉，Table 是**重新造出来**的。
7. 新造的 Table 拿到的 props 还是那个数组（**同一个数组对象**，不是同一个 Table），它已经被第 3 步改过了，于是 map 出 11 行。
8. 弹一个 1.5 秒的成功框。

**整个过程，localStorage 只被写、没被读。**

> 自己推一遍：第 5 步删掉，界面会变吗？
>
> **答案**：不会。bail out 之后 Dashboard 没有任何理由重跑，界面还是原来那份 UI 描述。所以删掉 `setEmployees` 这一行，功能**完全正常**——这行代码是死代码。反过来，删掉第 6 步 `setIsAdding(false)`，界面会**停在新增表单纹丝不动**，但库已经写了，刷新后才看到 11 行。

---

## 3. 四个"为什么"——六道题的答案全在这一节

### 3.1 为什么 `setEmployees` 没生效，界面却更新了（对应 Q1）

因为"让界面更新"和"让 state 被认可"是**两件不同的事**：

| | 谁做的 | 效果 |
| --- | --- | --- |
| 数据变了 | `push` 就地改同一个对象 | 数组内容多了 1 条 |
| state 更新 | `setEmployees(employees)` | 引用相同 → **被跳过** |
| 界面更新 | `setIsAdding(false)` | Dashboard 重跑 → Add 被拆掉、Table 重新造出来 → 新 Table 读到已变的那份数组 |

用一句话记：**数据靠"改对象"生效，界面靠"另一个 state 变化"生效，两者碰巧凑在一起。**

**适用范围（别推过头）**：上面这条链专指 **Add / Edit**，它们原地 `push` / `splice`。**Delete 是另一条路**——`Dashboard/index.js:49-51` 先 `filter` 出一个新数组，再 `setEmployees(employeesCopy)`；新引用本身就触发了重渲染，不需要谁兜底。所以"引用从没变过"只对新增/编辑成立。

这也是为什么它脆：只要有人把"更新数据"和"切换界面"解耦（比如抽个 `saveEmployee()` 复用），界面立刻不动。

> 常见误答："新数据是从 `getItem` 读回来的。" —— 不对。`getItem` 在 `Dashboard` **每次第一次出现**时的 `useEffect(() => {...}, [])` 里跑一次；打开页面、登录、登出再登录、刷新各算一次，点 Add 时它早跑完了。写库是单向的，没有回流。

> **追问：后面反正有 `setIsAdding(false)` 会触发重渲染，那这行留着做什么，删掉行不行？**
>
> 能删，删完界面一模一样（两个 setState 在同一事件里被自动批处理，合并成一次渲染）。但"能删"和"该删"是两回事：
>
> - 它是**意图的痕迹，不是功能**。作者大概出于"改了数据就该 setState"的本能，写了个正确写法的空壳，里面装的却是原地修改，于是空壳失效。像 Java 里 `list.add(x); this.list = list;` —— 第二行什么都没干，看起来却很规范。
> - **删掉反而更危险**：代码里就只剩 `push` + 写库 + 切屏，"state 从头到尾没被更新过"这件事被藏得更深。
> - 真正该做的是**把它改成对的那一行**：不原地改、造新数组、把新数组交给 `setEmployees`。于是数据更新归它，切屏归 `setIsAdding`，各管一件事，不再互相兜底。
>
> "反正后面也会触发重渲染"这句话本身，就是这份代码最核心的隐式耦合：它把**"界面碰巧更新了"**当成了**"状态被正确登记了"**。React 里这是两件事。直接代价举例：将来加一句 `useEffect(() => { 更新标题 }, [employees])`，走 Add / Edit 时它**不会**跑 —— 这两处是原地改，`employees` 的引用没变，React 根本不知道数据动过。
>
> **但有个例外**：`handleDelete` 用 `filter` 造了新数组，那一次引用**真的变了**，这个 effect 反而会跑。同一份代码里，新增/编辑靠切屏兜底、删除靠新数组生效，两种机制混着用——这正是它难读的原因之一。

**再追问：就算前后是两个不同地址的新数组，重渲染时 React 自己去发现新地址不就行了？**

不行。**React 不检查内存。** `useState` 返回的不是"现在内存里最新的数组"，而是"上一次通过 setState 登记进 state 槽位的那个值"。重渲染只是把组件函数再跑一遍，函数从槽位取值；它不会扫描内存、不会读存储、不会发现有个新数组被造出来了。

```java
private List<Employee> employees = oldList;   // React 的 state 槽位
void handleAdd() {
    List<Employee> next = new ArrayList<>(employees);
    next.add(newEmployee);      // 新地址，11 条
    // 忘了 this.employees = next;
    switchView();               // 视图重建
}
void render() {
    List<Employee> list = this.employees;   // 还是 oldList，10 条
}
```

`next` 是局部变量，函数结束就无人引用，等着被回收。视图重建一万次也没用，因为字段没被改写。

| 写法 | 槽位指向 | 重渲染拿到 | `setEmployees` |
| --- | --- | --- | --- |
| 当前：原地 `push` | 那个数组本身，内容已变 | 同一数组 → 11 条 | **可省**（上一问） |
| 不可变，**漏掉** `setEmployees(next)` | 旧数组 | 旧数组 → 10 条 | 不能省，界面看不到新员工 |
| 不可变 + `setEmployees(next)` | 新数组 | 新数组 → 11 条 | **必需** |

所以"地址不同"不是可以省的理由，恰恰是它**从可省变成必需**的理由：地址相同时靠原地改蒙混过去了，地址一不同，不登记就彻底失联。`setEmployees` 的真正职责不是"申请重渲染"（那是附赠效果），而是**改写槽位的指向**；重渲染是**读取**已登记的值，不是**发现**新值。

### 3.2 为什么 id 会变（对应 Q3）

先分清两个东西：

- `Table.js:34` 的 `<td>{i + 1}</td>` 是**显示序号**，每次渲染按 map 下标算，天然 1、2、3…。
- `employee.id` 是**存在对象里的字段**，被 Edit / Delete / `key` 当作身份用。

正常项目里这两个互不相干。但 `Table.js:4-6` 把它们焊死了：

```js
employees.forEach((employee, i) => {
  employee.id = i + 1;      // 在 render 期间，原地重写 props 里的对象
});
```

**每次 Table 渲染，所有员工的 id 都被重刷成当前行号。** 所以：

- Dashboard 挂载 → Table 首帧把 10 条种子刷成 `1..10`；
- `useEffect` 从 localStorage 载入存档 → 再渲染一次 → 又把载入的数据刷成 `1..n`；
- 你看到列表时，id 必然已经是连续的 `1..n`。

于是 `Add.js:23` 的 `employees.length + 1` 才"恰好"不冲突——**不是因为 id 设计得好，是因为上一帧刚有人把它擦成了连续编号**。两边互为前提：重排保证连续，`length+1` 依赖连续。

**但写进库的 id 并不连续**。删掉第 3 行时：

```
当前 id（刚被重排过）: 1 2 3 4 5 6 7 8 9 10
filter 掉 id===3      → 1 2 4 5 6 7 8 9 10   （9 条，有缺口，最大值仍是 10）
↑ 这一份被 JSON.stringify 写进 localStorage
setEmployees 后 Table 再渲染 → 又被刷成 1 2 3 4 5 6 7 8 9
```

界面和库里的 id，在删除后**长期不一致**，全靠下次渲染重排掩盖。

**如果只把 `forEach` 删掉、不动 Add**（这正是改进清单第 1 条最容易做成的一半）：

刷新后 state 里就是 `1,2,4,…,10`，点 Add 得到 `length + 1 = 10` → **和已有最后一条撞 id**。下面三条都发生在「**已经删掉 `forEach`**」的前提下（保留它时 key 序列反而是稳定的，见 4.1）：

1. `key={employee.id}` 出现重复 key，React 告警，列表复用行为不可预期；
2. 点 `Edit(10)` → `employees.filter(e => e.id === 10)` 返回两条，解构只取第一条 → **编辑到旧记录**；
3. 点 `Delete(10)` → `filter(e => e.id !== 10)` 把两条都滤掉 → **一次删掉两个员工**。

> 自己推一遍：那"删除时把末尾那条删掉就行"为什么不对？
>
> **答案**：删除走的是 `handleDelete(id)` → `filter(e => e.id !== id)`，按 **id** 过滤，不是按数组下标。id 和位置被重排焊死之后，删到第 3 行会连带改变后面所有行的身份；一旦解除重排，id 就带着缺口，跟位置彻底脱钩。

### 3.3 为什么删光刷新会"复活"（对应 Q2）

两个条件叠在一起：

```js
const [employees, setEmployees] = useState(employeesData);   // 初值 = 10 条种子
useEffect(() => {
  const data = JSON.parse(localStorage.getItem('employees_data'));
  if (data !== null && Object.keys(data).length !== 0) setEmployees(data);
}, []);
```

1. state 初值**就是**种子，不是空数组；
2. 存档是 `[]` 时，`Object.keys([])` 得到空数组、`length === 0`，`!== 0` 为假 → 不覆盖 state。

结果：删光 10 条 → 库里 `"[]"` → 刷新 → 条件不成立 → state 仍是那 10 条种子 → **数据全部回来**。

**文档给的"修法"其实没修好**（这也是我上一轮出这道题的原因）：

```js
if (Array.isArray(data) && data.length) setEmployees(data);   // 6.1 / 9.2 原话
```

`data` 是 `[]` 时，`Array.isArray([])` 为 true，但 `[].length` 是 `0` → `true && 0` → **falsy** → 条件照样不成立 → 照样复活。它把 `Object.keys` 换成了 `length`，坑却原封不动：**"空表"仍然被当成"没数据"**。

正确写法至少是：

```js
if (Array.isArray(data)) setEmployees(data);   // 空数组也必须覆盖
```

还得配套决定"什么时候用种子"：把种子从 state 初值挪成"**仅当 key 不存在时**写入并落库"的首次访问分支。否则 `data === null` 时界面显示种子、库里却什么都没有，语义依然含糊。

> 自己推一遍：用户手动删掉 `employees_data` 这个 key 后刷新，`JSON.parse(null)` 得到什么？条件成立吗？
>
> **答案**：得到 `null`（不抛异常），`Array.isArray(null)` 为 false → 不覆盖 → 显示种子。这正是"删库恢复种子"的原理。注意它和"库里是 `[]`"是**两种不同状态**，但旧代码把两者都当成了"没数据"。

### 3.4 为什么"种子数据"不是只读的（对应 Q4）

```js
const employeesData = [ {...}, {...}, ... ];   // 模块级数组，页面加载时创建一次
const [employees, setEmployees] = useState(employeesData);   // 存的是引用
```

Java 类比：`useState` 拿到的是 `employeesData` 这个 `List` 的**引用**，没有 clone、没有 freeze。所以：

- `employees.push(newEmployee)` 改的就是 `employeesData` **本体**；
- `Edit` 的 `splice` 同理；
- `Table` 的 `forEach` 改 `employee.id`，改的也是常量里那些对象（只是值恰好 `1..10`，看不出来）。

**什么时候改得到、什么时候改不到**，取决于 state 此刻还指不指向这个常量：

| 情形 | state 指向 | `push` 会不会污染模块常量 |
| --- | --- | --- |
| 首次访问，库里没有 `employees_data` | `employeesData` | **会** |
| `useEffect` 成功 `setEmployees(data)` 之后 | 从 JSON 解析出来的新数组 | 不会 |

所以"点 Add/Edit 才会改"你说对了，但结论反了：**不是"在 useState 里就改不到它"，恰恰是 useState 保存引用才改得到**。

顺带一个字面细节：源码里是 `push` 和 `splice`，**没有 `pop`**。`splice(i, 1, employee)` 是"替换第 i 个"，不是删除。

文档 8.3 说"删掉 `employees_data` → 刷新恢复 10 条种子"——**刷新**会重新执行模块文件、重建常量，所以这条是有效的。失效的是"不刷新、只让 Dashboard 重新挂载"（登出再登录）：模块常量此时已被污染成 11 条，而 `data === null` 又不会覆盖 state，于是你看到 11 条，不是 10 条。

---

## 4. 两处文档自己写偏的地方（对应 Q5 / Q6）

### 4.1 key 不是"抖动"，是"稳定地绑错对象"

文档 4.7 说重排导致 `key={employee.id}` "抖动、列表复用行为不可预测"。前半句不准确：

删掉中间一行 → `forEach` 立刻把剩下的 id 刷成 `1..n-1` → React 看到的 key 序列**始终是连续的 1,2,3,…，非常稳定，几乎不抖**。它会认为 key 3、4、5… 都还在，于是**复用原来的 DOM 节点、只改单元格文本**——同一个 `key=3` 那一行，上一帧是张三，这一帧变成李四。

真正的问题是：**key 绑的是"位置"，不是"这个是谁"**。key 的契约是"同一个 key 代表同一条数据"，这里被破坏了。目前表格里没有行内输入框 / 展开态 / 勾选态，所以看不出来；一旦加进去，删中间一行就会串位。文档括号里"本表没有输入框，所以没暴露"反倒说对了，只是结论方向写反了——不是"抖动"，是**稳定地绑错对象**。

### 4.2 ADR-4 那句"正确写法"，直接抄会翻倍

文档 5 节给的：

```js
setEmployees([...employees, newEmployee]);   // 说这是"正确写法"
```

如果**只替换 `Add.js:35`、保留第 33 行 `employees.push(newEmployee)`**：

```
push 之后           → 数组已有 11 条（含新对象 1 次）
[...employees, newEmployee] → 展开这 11 条，再追加同一个对象 → 12 条
```

结果：**新员工出现两次**，两条 id 都是 11，直接重复 key，Edit / Delete 跟着串。

而 9.3 说的"改动 4 行"若指 `setEmployees([...employees])`（只浅拷贝、**不再追加**），就不会翻倍——它解决的是"引用相等导致 bail out"，让 React 能察觉变化。但**原地修改本身还在**：模块常量和 props 照样被污染，只是现在能被检测到了。这是"能跑"和"正确"的区别。

根本解是删掉 `push` / `splice`，改成不可变更新：

```js
const next = [...employees, newEmployee];
localStorage.setItem('employees_data', JSON.stringify(next));
setEmployees(next);
setIsAdding(false);
```

这样数据变更、写库、state 更新用的是**同一份新数组**，不再依赖另一个 state 兜底。

> 自己推一遍：为什么"传新数组"能同时解决 3.1 和 3.4 的一部分？
>
> **答案**：`next` 是新引用 → `Object.is` 不等 → 不再 bail out，界面更新不再依赖 `setIsAdding`；原地修改没了，模块常量和 props 也不再被污染。但 id 重排（3.2）和判空（3.3）是另外两个独立问题，**不会**因此自动消失。

---

## 5. 你上一轮那几句话，逐句对照

| 你说的 | 实际情况 | 差在哪一层 |
| --- | --- | --- |
| "setEmployees 触发了重渲染" | 引用相同，bail out；真正触发的是 `setIsAdding(false)` | ③ 引用相等 + 重渲染触发条件 |
| "新数据从 `getItem` 读回来的" | 只在表格第一次出现时读一次；这里是同一个内存对象被就地改了 | ③ 内存 vs 存储是两份东西 |
| "删光 10 条刷新，数据应该不会复活" | 会复活，这正是文档 6.1 的 bug 本身 | ② 现象没记住 / ③ 判空逻辑没推 |
| "id 本来就是一个一个往后加，为什么要重排" | `Table.js:4-6` 每次渲染都把它刷成行号 | ③ render 期副作用 |
| "在 useState 里面不会改到常量" | useState 存引用，恰恰改得到 | ③ 引用语义 |
| "push 和 pop" | 是 `push` 和 `splice` | 字面细节，不影响理解 |

**规律很明显**：六句里有五句，问题都出在同一层——**"内存里到底有几份数据、谁和谁是同一个对象、函数什么时候被重新调用"**。补上这一层，其余全是顺推。

---

## 6. 五分钟动手验证（强烈建议做一遍）

你自述"看着代码像样子就过了"——那正好，别再靠读，用运行事实把它钉死。`npm start`，登录后打开 DevTools Console：

| 操作 | 预期看到 | 验证的是 |
| --- | --- | --- |
| 清空浏览器里这两个 key、刷新后，还没做任何增删改 → `JSON.parse(localStorage.getItem('employees_data'))` | `null` —— **首次访问根本没写过名单这个 key**，种子只是内存里的初值 | 1.6 / 3.3 |
| 在表格里删掉第 3 行 → 再执行上一句 | 9 条，id 是 `1,2,4,5,…,10` —— **有缺口、最大值 10**；`salary` 是字符串 | 1.6 / 3.2 |
| 紧接着 `JSON.parse(localStorage.getItem('employees_data')).map(e => e.id)` 和界面 No. 列对比 | No. 是 1..9，库里是 1,2,4..10 | 3.2 显示序号 ≠ 实体 id |
| 删光所有行 → 刷新 | 10 条种子回来 | 3.3 复活 |
| `localStorage.setItem('employees_data','[]'); location.reload()` | 依然 10 条 | 3.3 `[]` 被当成"没数据" |
| 清掉两个 key → **刷新一次** → 登录 → 新增一条员工 → `removeItem('employees_data')` → **不刷新**，点 Logout 再 Login | 看到 11 条而不是 10 条 | 3.4 模块常量被污染 |

最后一条最反直觉，也最能证明"内存里的数组"和"localStorage 的字符串"是两回事。

---

## 7. 现在回头看第 9 节那五条改进

只列"每条在修哪一节的哪个机制"，不展开方案：

| 改进 | 对应本文 | 修的是什么 |
| --- | --- | --- |
| 1. 稳定 id | 3.2 / 4.1 | render 期改入参、id 与位置焊死、key 错绑 |
| 2. 修判空 | 3.3 | 空表被当成没数据；注意文档给的写法本身没修好 |
| 3. setState 传新数组 | 3.1 / 4.2 | 引用相等 bail out、靠另一个 state 兜底 |
| 4. 抽数据层 | 1.6 / 3.1 | 5 处读写散落；把"内存"与"快照"的边界显式化 |
| 5. 补四条测试 | 全部 | 上述缺陷目前一个都测不出来（现有唯一测试只断言"能渲染不崩"） |

**这也解释了上一轮为什么我没直接谈改进**：第 1～3 条恰好就是你没答上的 Q3、Q2、Q6。机制没通的时候谈改进，只会得到一张背下来的清单。

---

## 8. 合上文档，再答一次那六题（答案要点）

1. `setEmployees` 同引用 → bail out；`setIsAdding(false)` 让 Dashboard 重跑、Add 被拆掉、Table 重新造出来，新 Table 读到已被 `push` 改过的同一个数组。数据没被读回。删掉 `setIsAdding(false)`：界面停在表单不动，刷新才见 11 行。（Delete 走 `filter` 新数组，是另一条路。）
2. **会复活**。库里 `[]` → `Object.keys([]).length === 0` → 不覆盖 → 保持种子。文档给的 `Array.isArray(data) && data.length` 对 `[]` 仍为假，**照样复活**；应改成 `Array.isArray(data)` 并把种子挪到"key 不存在"分支。
3. `Table.js:4-6` 每次渲染把 id 刷成行号；挂载 → 首帧重排 → 载入存档 → 再重排，所以 `length+1` 才成立。删除后写库的是 `1,2,4,…,10`。只删重排：新增 id=10 与旧记录撞车 → 重复 key、Edit 改错人、Delete 一次删两条。
4. `useState` 存引用，不拷贝；首次访问时 `push` 改的就是模块常量 `employeesData`，`useEffect` 载入存档后 state 换成新数组才改不到。`Table` 的 forEach 也在改它。刷新会重建模块所以 8.3 有效；不刷新只重挂载则失效。
5. 不准确。key 序列始终 `1..n`，不抖；React 复用 DOM 只改文本，问题在于 **key 绑位置不绑实体**，行内一旦有本地状态就串位。
6. 保留 `push` 再写 `[...employees, newEmployee]` → 12 条、新员工两次。`[...employees]`（只浅拷贝）不翻倍但没解决原地修改；根本解是删掉 `push`/`splice`、用不可变更新。

---

## 附：一页速查卡

1. 组件 = 被反复调用的函数；重渲染 = 函数再跑一遍，**不刷新页面、不读存储**。
2. `Object.is` 对数组比**引用**，跟 Java `==` / Python `is` 一样；内容变了不等于 state 变了。
3. props 传引用，子组件改 props = 改父组件的数据。
4. `const` 锁绑定不锁内容，等价于 Java `final List` 仍可 `add`。
5. `setState` 是唯一让组件函数重跑的入口；两个 setState 在同一事件里，一个可能白调、另一个兜底。
6. localStorage 是 `Map<String,String>`，写进去就是快照，**不会回流**。
7. render 里不要改入参；`Table.js:4-6` 改了，于是 id 成了行号。
8. `[]` 和 `null` 是两种状态；用 `.length` 判空会把"空表"当成"没数据"。

> 判断自己有没有真懂，只问一句：**"把这行删掉会怎样？"** 能答出来，才算过。
