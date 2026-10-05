# 学生成绩管理系统 详细设计说明书

> 依据源码 `src/student_grade_system.py`（371 行）整理，仅描述设计，不修改任何代码。
> 技术约束：单文件 Python、命令行交互、仅标准库（`json`/`os`/`statistics`）、本地 JSON 持久化、不使用 Web 框架与数据库。

---

## 0 数据模型与全局常量

### 0.1 全局常量

| 名称 | 类型 | 值 | 说明 |
|---|---|---|---|
| `DATA_FILE` | str | `"grades_data.json"` | 数据文件名，位于进程当前工作目录 |
| `DEFAULT_DATA` | dict | 见 0.2 | 首次运行/数据损坏时使用的内置默认数据 |

### 0.2 内存数据结构（顶层字典 `data`）

| 键 | 类型 | 元素结构 | 说明 |
|---|---|---|---|
| `users` | list[dict] | `{username:str, password:str, role:str, student_id:str\|None}` | 登录账号，`role ∈ {admin, teacher, student}` |
| `students` | list[dict] | `{id:str, name:str, gender:str, class_name:str}` | 学生档案，`id` 唯一 |
| `courses` | list[dict] | `{id:str, name:str}` | 课程档案，`id` 唯一 |
| `scores` | list[dict] | `{student_id:str, course_id:str, score:float}` | 成绩记录，`(student_id, course_id)` 逻辑唯一 |

默认账号：`admin/admin123`、`teacher/teacher123`、`s001/123456`（学生绑定 `S001`）。
默认学生：`S001 张三 男 一班`、`S002 李四 男 一班`；默认课程：`C001 数学`、`C002 英语`。

### 0.3 数据约束

- `students.id`、`courses.id` 唯一；
- `scores` 中同一 `(student_id, course_id)` 组合只保留一条，重复录入视为更新；
- `score` 为 0~100 的浮点数（含整数与小数）。

---

## 1 函数清单

> 编号 F01~F20 按源码定义顺序排列。返回值 `None` 表示副作用型函数（打印/修改数据）。

| 编号 | 函数名 | 模块 | 参数 | 返回值 | 职责 |
|---|---|---|---|---|---|
| F01 | `save_data` | 持久化 | `data: dict` | `None` | 将内存数据写回 JSON 文件 |
| F02 | `load_data` | 持久化 | 无 | `dict` | 加载 JSON，缺失时创建默认数据 |
| F03 | `find_student` | 查找工具 | `data: dict, sid: str` | `dict \| None` | 按学号查学生 |
| F04 | `find_course` | 查找工具 | `data: dict, cid: str` | `dict \| None` | 按课程号查课程 |
| F05 | `find_user` | 查找工具 | `data: dict, username: str` | `dict \| None` | 按用户名查账号 |
| F06 | `login` | 认证 | `data: dict` | `dict \| None` | 三次机会校验账号密码 |
| F07 | `input_score` | 输入校验 | 无 | `float` | 循环读取合法成绩 |
| F08 | `list_students` | 学生管理 | `data: dict` | `None` | 打印学生列表（含性别、班级） |
| F09 | `list_courses` | 课程管理 | `data: dict` | `None` | 打印课程列表 |
| F10 | `list_scores` | 成绩管理 | `data: dict` | `None` | 打印全部成绩（关联姓名/课程名） |
| F11 | `add_student` | 学生管理 | `data: dict` | `None` | 新增学生 |
| F12 | `add_course` | 课程管理 | `data: dict` | `None` | 新增课程 |
| F13 | `set_score` | 成绩管理 | `data: dict` | `None` | 录入或更新成绩 |
| F14 | `query_student_scores` | 成绩查询 | `data: dict` | `None` | 按学号查询成绩与平均分 |
| F15 | `course_stats` | 统计 | `data: dict` | `None` | 单课程人数/平均/最高/最低 |
| F16 | `student_view_self` | 学生自助 | `data: dict, user: dict` | `None` | 学生查本人成绩与平均分 |
| F17 | `admin_menu` | 菜单 | `data: dict` | `None` | 管理员菜单循环 |
| F18 | `teacher_menu` | 菜单 | `data: dict` | `None` | 教师菜单循环 |
| F19 | `student_menu` | 菜单 | `data: dict, user: dict` | `None` | 学生菜单循环 |
| F20 | `main` | 入口 | 无 | `None` | 启动、加载、登录、角色分发 |

### 1.1 各函数详细签名与返回语义

#### F01 `save_data(data) -> None`
- 参数：`data` 为完整顶层字典。
- 行为：以 `encoding="utf-8"`、`ensure_ascii=False`、`indent=2` 序列化写入 `DATA_FILE`。
- 返回：无。失败时仅打印 `[错误] 保存数据失败：{exc}`，**不抛出**、不向调用方回报成败。

#### F02 `load_data() -> dict`
- 参数：无。
- 返回：可用数据字典。三种来源：
  1. 文件不存在 → `save_data(DEFAULT_DATA)` 后返回 `DEFAULT_DATA` 的深拷贝；
  2. 文件存在且合法 → 返回解析结果，并对缺失的四个顶层键执行 `setdefault(key, [])`；
  3. 读取/解析异常 → 打印错误，返回 `DEFAULT_DATA` 的深拷贝。

#### F03/F04/F05 `find_student / find_course / find_user`
- 参数：`data` + 查找键（`sid`/`cid`/`username`）。
- 返回：首个命中元素（`dict`），未命中返回 `None`。
- 复杂度：线性 O(n)。

#### F06 `login(data) -> dict | None`
- 参数：`data`。
- 返回：认证通过的用户 `dict`；3 次失败后返回 `None`。
- 副作用：打印登录提示与剩余次数。

#### F07 `input_score() -> float`
- 参数：无。
- 返回：`[0, 100]` 内的 `float`（阻塞直到输入合法）。

#### F11 `add_student(data) -> None`
- 参数：`data`（原地 `append` 后调用 `save_data`）。
- 新增字段：`id`、`name`、`gender`（空则 `"未知"`）、`class_name`（空则 `"未分班"`）。

#### F12 `add_course(data) -> None`
- 参数：`data`。新增字段：`id`、`name`。

#### F13 `set_score(data) -> None`
- 参数：`data`。
- 行为：校验学生、课程、成绩后，命中已有记录则更新，否则追加。

#### F15 `course_stats(data) -> None`
- 统计字段：人数 `len`、平均 `statistics.mean`、最高 `max`、最低 `min`，均以 `:.2f` 打印。

#### F16 `student_view_self(data, user) -> None`
- 参数：`user` 为已登录用户；`sid = user.get("student_id")`。
- 边界：`sid` 为 `None` 或对应学生不存在时，提示“当前账号未绑定有效学号”。

#### F20 `main() -> None`
- 流程：打印横幅 → `load_data` → `login` → 失败则退出 → 按 `role` 调用 `admin_menu`/`teacher_menu`/`student_menu` → 未知角色打印退出。

---

## 2 关键流程伪代码

### 2.1 主流程 `main`

```
FUNCTION main():
    print 系统横幅
    data = load_data()
    user = login(data)
    IF user is None:
        print "登录失败，程序退出。"
        RETURN
    role = user.get("role")
    print "欢迎，" + user.username + "（" + role + "）"
    SWITCH role:
        CASE "admin":   admin_menu(data)
        CASE "teacher": teacher_menu(data)
        CASE "student": student_menu(data, user)
        DEFAULT:        print "未知角色，退出。"
```

### 2.2 持久化 `load_data` / `save_data`

```
FUNCTION save_data(data):
    TRY:
        OPEN DATA_FILE FOR WRITE utf-8
        json.dump(data, ensure_ascii=False, indent=2)
    CATCH OSError as exc:
        print "[错误] 保存数据失败：" + exc
        # 吞掉异常，不影响后续内存操作

FUNCTION load_data():
    IF NOT os.path.exists(DATA_FILE):
        save_data(DEFAULT_DATA)
        RETURN deepcopy(DEFAULT_DATA)          # json round-trip 实现深拷贝
    TRY:
        data = json.load(OPEN(DATA_FILE, utf-8))
        FOR key IN ("users","students","courses","scores"):
            data.setdefault(key, [])           # 容错：补齐缺失键
        RETURN data
    CATCH (OSError, JSONDecodeError) as exc:
        print "[错误] 读取数据失败：" + exc
        print "将使用默认数据。"
        RETURN deepcopy(DEFAULT_DATA)
```

### 2.3 登录 `login`

```
FUNCTION login(data):
    print 登录提示与默认账号
    FOR i IN range(3):
        username = input("用户名：").strip()
        password = input("密码：").strip()
        user = find_user(data, username)
        IF user AND user.get("password") == password:
            RETURN user
        print "用户名或密码错误，剩余尝试次数：" + str(2 - i)
    RETURN None
```

### 2.4 成绩输入 `input_score`

```
FUNCTION input_score() -> float:
    WHILE True:
        raw = input("成绩（0-100，可小数）：").strip()
        IF raw == "":
            print "成绩不能为空"; CONTINUE
        TRY:
            score = float(raw)
        CATCH ValueError:
            print "成绩必须是数字"; CONTINUE
        IF NOT (0 <= score <= 100):
            print "成绩范围必须在 0 到 100 之间"; CONTINUE
        RETURN score
```

### 2.5 录入/更新成绩 `set_score`

```
FUNCTION set_score(data):
    sid = input("学号：").strip()
    IF find_student(data, sid) is None:
        print "学生不存在"; RETURN
    cid = input("课程号：").strip()
    IF find_course(data, cid) is None:
        print "课程不存在"; RETURN
    score = input_score()
    FOR item IN data["scores"]:
        IF item.student_id == sid AND item.course_id == cid:
            item.score = score
            save_data(data); print "成绩已更新"; RETURN
    data["scores"].append({student_id:sid, course_id:cid, score:score})
    save_data(data); print "成绩已录入"
```

### 2.6 菜单循环（以管理员为例）

```
FUNCTION admin_menu(data):
    WHILE True:
        print 管理员菜单
        choice = input("请选择：").strip()
        IF choice == "1": list_students(data)
        ... 分支 2~8 同理 ...
        ELIF choice == "0": BREAK
        ELSE: print "无效选项"
```

教师菜单调用 `list_students/list_courses/set_score/query_student_scores/course_stats`；
学生菜单选项 `1`、`2` **均调用** `student_view_self(data, user)`（该函数同时输出各科成绩与平均分），`0` 退出。

### 2.7 查询与统计

```
FUNCTION query_student_scores(data):
    sid = input("学号：").strip()
    IF find_student(data, sid) is None:
        print "学生不存在"; RETURN
    rows = [r FOR r IN data["scores"] IF r.student_id == sid]
    IF rows 为空:
        print "该生暂无成绩"; RETURN
    total = 0.0
    FOR r IN rows:
        cname = find_course(data, r.course_id).name OR "未知课程"
        print r.course_id + " " + cname + ": " + format(r.score, ".2f")
        total += r.score
    print "平均分：" + format(total / len(rows), ".2f")

FUNCTION course_stats(data):
    cid = input("课程号：").strip()
    course = find_course(data, cid)
    IF course is None:
        print "课程不存在"; RETURN
    scores = [r.score FOR r IN data["scores"] IF r.course_id == cid]
    IF scores 为空:
        print "该课程暂无成绩"; RETURN
    print 人数/平均/最高/最低（均值用 statistics.mean，均 .2f）
```

### 2.8 深拷贝语义

`load_data` 返回默认数据时使用 `json.loads(json.dumps(DEFAULT_DATA))`，避免后续 `add_*` 修改污染全局 `DEFAULT_DATA`。

---

## 3 异常分支汇总

| 编号 | 触发条件 | 所在函数 | 处理动作 | 是否终止当前流程 |
|---|---|---|---|---|
| E01 | JSON 文件不存在 | `load_data` | 写入默认数据并返回副本 | 否，继续 |
| E02 | JSON 解析失败 `JSONDecodeError` | `load_data` | 打印错误 + “将使用默认数据”，返回副本 | 否，继续 |
| E03 | 文件读取 `OSError` | `load_data` | 同上 | 否，继续 |
| E04 | 文件写入 `OSError` | `save_data` | 打印 `[错误] 保存数据失败` | 否，内存变更保留 |
| E05 | 用户名或密码错误 | `login` | 打印剩余次数，最多重试 3 次 | 3 次后返回 `None` 退出 |
| E06 | 登录三次失败 | `main` | 打印“登录失败，程序退出。” | 是 |
| E07 | 成绩为空 | `input_score` | 提示，重新输入 | 否 |
| E08 | 成绩非数字 `ValueError` | `input_score` | 提示“成绩必须是数字”，重新输入 | 否 |
| E09 | 成绩越界（<0 或 >100） | `input_score` | 提示范围，重新输入 | 否 |
| E10 | 学号为空 | `add_student` | 打印“学号不能为空” | 是（返回菜单） |
| E11 | 学号已存在 | `add_student` | 打印“学号已存在” | 是 |
| E12 | 姓名为空 | `add_student` | 打印“姓名不能为空” | 是 |
| E13 | 课程号为空 | `add_course` | 打印“课程号不能为空” | 是 |
| E14 | 课程号已存在 | `add_course` | 打印“课程号已存在” | 是 |
| E15 | 课程名为空 | `add_course` | 打印“课程名不能为空” | 是 |
| E16 | 学号不存在 | `set_score` / `query_student_scores` | 打印“学生不存在” | 是 |
| E17 | 课程号不存在 | `set_score` / `course_stats` | 打印“课程不存在” | 是 |
| E18 | 该生无成绩 | `query_student_scores` / `student_view_self` | 打印“该生暂无成绩”/“暂无成绩” | 是 |
| E19 | 该课程无成绩 | `course_stats` | 打印“该课程暂无成绩”，不计算均值 | 是（避免除零） |
| E20 | 学生账号未绑定有效学号 | `student_view_self` | 打印“当前账号未绑定有效学号” | 是 |
| E21 | 菜单输入无效选项 | `*_menu` | 打印“无效选项”，重新循环 | 否 |
| E22 | 成绩记录引用的学生/课程缺失 | `list_scores` / `query_*` | 显示“未知学生”/“未知课程” | 否 |
| E23 | 顶层键缺失 | `load_data` | `setdefault(key, [])` 补齐 | 否 |
| E24 | 未知角色 | `main` | 打印“未知角色，退出。” | 是 |
| E25 | 多进程并发写入 | 全局 | 未处理（单机单进程假设） | — |

补充说明：
- 所有 `input()` 均带 `.strip()`，空白输入按空字符串处理（`login` 中空用户名自然查不到用户）。
- `input_score` 仅捕获 `ValueError`；`float("nan")` / `float("inf")` 会通过比较被拒（`NaN` 比较恒 False，走越界提示）。
- `save_data` 失败不阻断业务，存在“内存已变、磁盘未落盘”的不一致窗口。

---

## 4 测试用例表

> 约定：均以 CLI 交互为测试手段；`输入` 列按交互顺序用 `→` 连接；“类型”中 正=正常路径，异=异常路径，边=边界值。

### 4.1 持久化与加载

| 用例 | 类型 | 前置条件 | 输入/操作 | 预期结果 |
|---|---|---|---|---|
| T01 | 正 | 无 `grades_data.json` | 启动程序 | 自动生成 `grades_data.json`，内容等于 `DEFAULT_DATA` |
| T02 | 正 | 文件合法 | 启动 → 查看学生 | 正常加载，学生列表与文件一致 |
| T03 | 异 | 文件内容非 JSON | 启动程序 | 打印“读取数据失败”“将使用默认数据”，进入登录 |
| T04 | 边 | 文件仅含 `{"users":[]}` | 启动程序 | `students/courses/scores` 补为 `[]`，不报 KeyError |
| T05 | 正 | 已新增学生 | 再次启动 | 新增数据仍在（持久化成功） |
| T06 | 异 | 文件只读/被占用（可模拟目录同名的 `grades_data.json`） | 新增学生 | 打印“保存数据失败”，程序不崩溃 |
| T07 | 边 | 数据文件存在但为 UTF-8 BOM | 启动 | 视情况解析；失败则走 E02 默认数据 |

### 4.2 登录与角色

| 用例 | 类型 | 前置条件 | 输入 | 预期结果 |
|---|---|---|---|---|
| T08 | 正 | 默认数据 | `admin` → `admin123` | 进入管理员菜单 |
| T09 | 正 | 默认数据 | `teacher` → `teacher123` | 进入教师菜单 |
| T10 | 正 | 默认数据 | `s001` → `123456` | 进入学生菜单 |
| T11 | 异 | — | `admin` → `wrong` | 提示剩余次数 2，重新输入 |
| T12 | 异 | — | 连续 3 次错误 | 第 3 次提示剩余 0，打印“登录失败，程序退出。” |
| T13 | 边 | — | 用户名含前后空格 `  admin  ` | `.strip()` 后登录成功 |
| T14 | 边 | — | 空用户名 + 空密码 | 认证失败，计数消耗 |

### 4.3 学生管理

| 用例 | 类型 | 输入序列 | 预期结果 |
|---|---|---|---|
| T15 | 正 | 管理员菜单 `2` → `S003` → `王五` → `女` → `二班` | 打印“添加学生成功”，列表含新学生 |
| T16 | 异 | `2` → 空学号 | 打印“学号不能为空”，返回菜单 |
| T17 | 异 | `2` → `S001` | 打印“学号已存在” |
| T18 | 异 | `2` → `S004` → 空姓名 | 打印“姓名不能为空”，不写入 |
| T19 | 边 | `2` → `S005` → `赵六` → 空性别 → 空班级 | 性别=`未知`，班级=`未分班` |
| T20 | 正 | `1` | 显示 `学号/姓名/性别/班级`，格式对齐 |
| T21 | 边 | 学生列表为空 | `1` | 打印“暂无学生” |

### 4.4 课程管理

| 用例 | 类型 | 输入序列 | 预期结果 |
|---|---|---|---|
| T22 | 正 | `4` → `C003` → `物理` | 打印“添加课程成功” |
| T23 | 异 | `4` → 空课程号 | 打印“课程号不能为空” |
| T24 | 异 | `4` → `C001` | 打印“课程号已存在” |
| T25 | 异 | `4` → `C004` → 空课程名 | 打印“课程名不能为空” |
| T26 | 正 | `3` | 显示课程列表 |
| T27 | 边 | 课程为空 | `3` | 打印“暂无课程” |

### 4.5 成绩录入/修改

| 用例 | 类型 | 输入序列 | 预期结果 |
|---|---|---|---|
| T28 | 正 | `6` → `S002` → `C002` → `85.5` | 打印“成绩已录入”，新增记录 |
| T29 | 正 | `6` → `S001` → `C001` → `95` | 打印“成绩已更新”，覆盖原 92.5 |
| T30 | 异 | `6` → `S999` | 打印“学生不存在” |
| T31 | 异 | `6` → `S001` → `C999` | 打印“课程不存在” |
| T32 | 异 | `6` → `S001` → `C001` → 空 | 提示“成绩不能为空”，重新输入 |
| T33 | 异 | ... → `abc` | 提示“成绩必须是数字”，重新输入 |
| T34 | 边 | ... → `-1` | 提示“成绩范围必须在 0 到 100 之间”，重新输入 |
| T35 | 边 | ... → `101` | 同上 |
| T36 | 边 | ... → `0` | 接受，记录 0.0 |
| T37 | 边 | ... → `100` | 接受，记录 100.0 |
| T38 | 边 | ... → `88.25` | 接受，小数保留 |

### 4.6 查询与统计

| 用例 | 类型 | 输入序列 | 预期结果 |
|---|---|---|---|
| T39 | 正 | `7` → `S001` | 显示两门课成绩，平均分 `(92.5+88.0)/2=90.25` |
| T40 | 异 | `7` → `S999` | 打印“学生不存在” |
| T41 | 边 | `7` → 无成绩学生 | 打印“该生暂无成绩” |
| T42 | 正 | `8` → `C001` | 显示人数、平均、最高、最低（`.2f`） |
| T43 | 异 | `8` → `C999` | 打印“课程不存在” |
| T44 | 边 | `8` → 无成绩课程 | 打印“该课程暂无成绩”，不出现除零错误 |
| T45 | 正 | `5` | 显示全部成绩，含学生名与课程名 |
| T46 | 边 | 成绩为空 | `5` | 打印“暂无成绩” |
| T47 | 异 | 成绩引用了已删除学生/课程（手工改 JSON） | `5` | 显示“未知学生/未知课程”，不崩溃 |

### 4.7 学生自助

| 用例 | 类型 | 前置条件 | 输入序列 | 预期结果 |
|---|---|---|---|---|
| T48 | 正 | `s001` 登录 | 学生菜单 `1` | 显示 S001 各科成绩与平均分 |
| T49 | 正 | `s001` 登录 | 学生菜单 `2` | 与 T48 相同输出（同函数） |
| T50 | 异 | 手工把 `s001.student_id` 改为不存在的学号 | 学生菜单 `1` | 打印“当前账号未绑定有效学号” |
| T51 | 边 | 手工把 `s001.student_id` 置 `None` | 学生菜单 `1` | 同上 |
| T52 | 边 | `s001` 无成绩 | 学生菜单 `1` | 打印“暂无成绩” |

### 4.8 菜单与退出

| 用例 | 类型 | 输入序列 | 预期结果 |
|---|---|---|---|
| T53 | 正 | 管理员菜单 `0` | 退出菜单，程序结束 |
| T54 | 正 | 教师菜单 `0` | 退出 |
| T55 | 正 | 学生菜单 `0` | 退出 |
| T56 | 异 | 菜单输入 `9` / `abc` / 空 | 打印“无效选项”，返回菜单 |
| T57 | 边 | 连续调用同一功能多次后 `0` | 每次返回菜单，最终正常退出 |

### 4.9 边界与一致性

| 用例 | 类型 | 场景 | 预期结果 |
|---|---|---|---|
| T58 | 边 | 大量成绩（如 1 万条） | 线性查找仍可运行，无明显卡顿 |
| T59 | 边 | 同一学生同一课程连续录入 | 始终只有一条记录，值为最后一次 |
| T60 | 边 | `score` 输入 `nan`/`inf` | 因比较逻辑被判定越界并重输 |
| T61 | 异 | 保存失败后继续查询 | 内存中数据可见，磁盘未更新（存在不一致窗口） |
| T62 | 边 | 顶层键缺失 + 新增数据 | 补齐后正常追加并落盘 |

---

## 5 设计约束与已知局限

| 项 | 说明 |
|---|---|
| 单文件 | 全部逻辑位于 `student_grade_system.py`，无外部依赖 |
| 无数据库/框架 | 仅标准库 + JSON 文件 |
| 并发 | 不支持多进程/多用户并发写入，无文件锁 |
| 安全 | 密码明文存储，仅演示用；无哈希、无会话 |
| 错误处理 | 多数业务错误以打印提示 + 返回菜单处理；`save_data` 失败被吞掉 |
| 代码复用 | `list_scores`、`query_student_scores`、`student_view_self` 打印逻辑高度重复，后续可抽取 |
| 缺少删除 | 无删除学生/课程/成绩功能，故无级联清理 |
| 缺少测试 | 源码未附带自动化测试；本文件第 4 节可作为测试设计输入 |
