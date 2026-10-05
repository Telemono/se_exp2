## 1 用户角色

| 角色 | 说明 | 权限 |
|---|---|---|
| 管理员 | 系统维护人员 | 学生管理、课程管理、成绩录入/修改、查询、统计 |
| 教师 | 任课教师 | 查看学生/课程、录入/修改成绩、查询学生成绩、课程统计 |
| 学生 | 学生本人 | 只能查看自己的成绩和平均分 |

默认演示账号：

- 管理员：`admin / admin123`
- 教师：`teacher / teacher123`
- 学生：`s001 / 123456`

---

## 2 功能需求列表

1. 登录与权限控制  
   - 用户输入用户名、密码登录。
   - 连续失败 3 次退出。
   - 不同角色进入不同菜单。

2. 学生管理  
   - 查看学生列表。
   - 添加学生，学号唯一，姓名不能为空。

3. 课程管理  
   - 查看课程列表。
   - 添加课程，课程号唯一，课程名不能为空。

4. 成绩管理  
   - 录入成绩。
   - 修改成绩。
   - 成绩范围 0~100，支持小数。
   - 同一学生同一课程重复录入时覆盖旧成绩。

5. 成绩查询  
   - 按学号查询某学生所有成绩。
   - 显示各科成绩和平均分。
   - 查看全部成绩列表。

6. 统计功能  
   - 按课程统计人数、平均分、最高分、最低分。

7. 数据持久化  
   - 使用本地 JSON 文件保存。
   - 不使用数据库。
   - 程序启动时自动加载，退出前保存。

8. 学生自助查询  
   - 学生登录后只能查看自己的成绩和平均分。

---

## 3 非功能需求

1. 单文件 Python 可运行  
   - 保存为一个 `.py` 文件，执行 `python student_grade_system.py` 即可。

2. 不使用 Web 框架  
   - 不涉及 Flask、Django、FastAPI 等。

3. 不使用数据库  
   - 不连接 MySQL、SQLite、MongoDB 等。
   - 使用本地 JSON 文件存储。

4. 仅依赖 Python 标准库  
   - 使用 `json`、`os`、`statistics` 等标准库。

5. 命令行交互  
   - 通过菜单和输入完成操作，适合课程设计或原型演示。

6. 可维护性  
   - 按功能拆分为函数，如登录、学生管理、成绩管理、统计等。

7. 性能要求  
   - 面向小规模数据，查询和统计采用线性遍历即可。

8. 兼容性  
   - 支持 Windows、Linux、macOS 上的 Python 3。

9. 安全性  
   - 简单角色权限隔离。
   - 演示版密码明文保存，生产环境应改为哈希存储。

10. 易用性  
   - 菜单清晰。
   - 输入错误时给出提示，不直接崩溃。

---

## 4 边界与异常

1. 输入为空  
   - 学号、课程号、姓名、课程名不能为空。

2. 重复数据  
   - 学号不能重复。
   - 课程号不能重复。

3. 成绩异常  
   - 成绩必须是数字。
   - 成绩必须在 0~100 之间。
   - 支持小数，显示时保留两位。

4. 关联对象不存在  
   - 录入成绩时，学生不存在或课程不存在应提示。
   - 查询学生成绩时，学生不存在应提示。

5. 无成绩统计  
   - 某课程没有成绩时，不计算平均分，避免除零。

6. 数据文件异常  
   - JSON 文件不存在时自动创建默认数据。
   - JSON 损坏时提示并使用默认数据。
   - 文件无读写权限时捕获 `OSError`。

7. 登录异常  
   - 用户名或密码错误提示。
   - 连续 3 次失败退出。

8. 学生账号未绑定学号  
   - 学生登录后若未绑定有效学号，提示无法查询。

9. 并发写入  
   - 单机 CLI 版本暂不处理多进程同时写入。

10. 删除级联  
   - 若后续实现删除学生或课程，需要同时处理关联成绩。

---

## 附：单文件 Python 参考实现

保存为 `student_grade_system.py`，运行：

```bash
python student_grade_system.py
```

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""学生成绩管理系统：单文件 CLI，无 Web 框架、无数据库，数据存本地 JSON。"""

import json
import os
import statistics

DATA_FILE = "grades_data.json"

DEFAULT_DATA = {
    "users": [
        {"username": "admin", "password": "admin123", "role": "admin", "student_id": None},
        {"username": "teacher", "password": "teacher123", "role": "teacher", "student_id": None},
        {"username": "s001", "password": "123456", "role": "student", "student_id": "S001"},
    ],
    "students": [
        {"id": "S001", "name": "张三", "class_name": "一班"},
        {"id": "S002", "name": "李四", "class_name": "一班"},
    ],
    "courses": [
        {"id": "C001", "name": "数学"},
        {"id": "C002", "name": "英语"},
    ],
    "scores": [
        {"student_id": "S001", "course_id": "C001", "score": 92.5},
        {"student_id": "S001", "course_id": "C002", "score": 88.0},
        {"student_id": "S002", "course_id": "C001", "score": 76.0},
    ],
}


def save_data(data):
    try:
        with open(DATA_FILE, "w", encoding="utf-8") as f:
            json.dump(data, f, ensure_ascii=False, indent=2)
    except OSError as exc:
        print(f"[错误] 保存数据失败：{exc}")


def load_data():
    if not os.path.exists(DATA_FILE):
        save_data(DEFAULT_DATA)
        return json.loads(json.dumps(DEFAULT_DATA))

    try:
        with open(DATA_FILE, "r", encoding="utf-8") as f:
            data = json.load(f)
        for key in ("users", "students", "courses", "scores"):
            data.setdefault(key, [])
        return data
    except (OSError, json.JSONDecodeError) as exc:
        print(f"[错误] 读取数据失败：{exc}")
        print("将使用默认数据。")
        return json.loads(json.dumps(DEFAULT_DATA))


def find_student(data, sid):
    return next((s for s in data["students"] if s["id"] == sid), None)


def find_course(data, cid):
    return next((c for c in data["courses"] if c["id"] == cid), None)


def find_user(data, username):
    return next((u for u in data["users"] if u["username"] == username), None)


def login(data):
    print("\n请登录（默认：admin/admin123, teacher/teacher123, s001/123456）")
    for i in range(3):
        username = input("用户名：").strip()
        password = input("密码：").strip()
        user = find_user(data, username)
        if user and user.get("password") == password:
            return user
        print(f"用户名或密码错误，剩余尝试次数：{2 - i}")
    return None


def input_score():
    while True:
        raw = input("成绩（0-100，可小数）：").strip()
        if not raw:
            print("成绩不能为空")
            continue
        try:
            score = float(raw)
        except ValueError:
            print("成绩必须是数字")
            continue
        if not 0 <= score <= 100:
            print("成绩范围必须在 0 到 100 之间")
            continue
        return score


def list_students(data):
    print("\n--- 学生列表 ---")
    if not data["students"]:
        print("暂无学生")
        return
    for s in data["students"]:
        print(f"{s['id']:>8} | {s['name']:<8} | {s.get('class_name', '')}")


def list_courses(data):
    print("\n--- 课程列表 ---")
    if not data["courses"]:
        print("暂无课程")
        return
    for c in data["courses"]:
        print(f"{c['id']:>8} | {c['name']}")


def list_scores(data):
    print("\n--- 成绩列表 ---")
    if not data["scores"]:
        print("暂无成绩")
        return
    for r in data["scores"]:
        s = find_student(data, r["student_id"])
        c = find_course(data, r["course_id"])
        sname = s["name"] if s else "未知学生"
        cname = c["name"] if c else "未知课程"
        print(f"{r['student_id']} {sname} | {r['course_id']} {cname} | {r['score']}")


def add_student(data):
    sid = input("学号：").strip()
    if not sid:
        print("学号不能为空")
        return
    if find_student(data, sid):
        print("学号已存在")
        return

    name = input("姓名：").strip()
    if not name:
        print("姓名不能为空")
        return

    class_name = input("班级（可空）：").strip() or "未分班"
    data["students"].append({"id": sid, "name": name, "class_name": class_name})
    save_data(data)
    print("添加学生成功")


def add_course(data):
    cid = input("课程号：").strip()
    if not cid:
        print("课程号不能为空")
        return
    if find_course(data, cid):
        print("课程号已存在")
        return

    name = input("课程名：").strip()
    if not name:
        print("课程名不能为空")
        return

    data["courses"].append({"id": cid, "name": name})
    save_data(data)
    print("添加课程成功")


def set_score(data):
    sid = input("学号：").strip()
    student = find_student(data, sid)
    if not student:
        print("学生不存在")
        return

    cid = input("课程号：").strip()
    course = find_course(data, cid)
    if not course:
        print("课程不存在")
        return

    score = input_score()

    for item in data["scores"]:
        if item["student_id"] == sid and item["course_id"] == cid:
            item["score"] = score
            save_data(data)
            print("成绩已更新")
            return

    data["scores"].append({"student_id": sid, "course_id": cid, "score": score})
    save_data(data)
    print("成绩已录入")


def query_student_scores(data):
    sid = input("学号：").strip()
    student = find_student(data, sid)
    if not student:
        print("学生不存在")
        return

    rows = [r for r in data["scores"] if r["student_id"] == sid]
    if not rows:
        print("该生暂无成绩")
        return

    print(f"\n{student['id']} {student['name']} 的成绩：")
    total = 0.0
    for r in rows:
        course = find_course(data, r["course_id"])
        cname = course["name"] if course else "未知课程"
        print(f"  {r['course_id']} {cname}: {r['score']:.2f}")
        total += r["score"]

    print(f"平均分：{total / len(rows):.2f}")


def course_stats(data):
    cid = input("课程号：").strip()
    course = find_course(data, cid)
    if not course:
        print("课程不存在")
        return

    scores = [r["score"] for r in data["scores"] if r["course_id"] == cid]
    if not scores:
        print("该课程暂无成绩")
        return

    print(f"\n{course['id']} {course['name']} 统计：")
    print(f"  人数：{len(scores)}")
    print(f"  平均：{statistics.mean(scores):.2f}")
    print(f"  最高：{max(scores):.2f}")
    print(f"  最低：{min(scores):.2f}")


def student_view_self(data, user):
    sid = user.get("student_id")
    student = find_student(data, sid)
    if not student:
        print("当前账号未绑定有效学号")
        return

    rows = [r for r in data["scores"] if r["student_id"] == sid]
    if not rows:
        print("暂无成绩")
        return

    print(f"\n{student['id']} {student['name']} 的成绩：")
    total = 0.0
    for r in rows:
        course = find_course(data, r["course_id"])
        cname = course["name"] if course else "未知课程"
        print(f"  {r['course_id']} {cname}: {r['score']:.2f}")
        total += r["score"]

    print(f"平均分：{total / len(rows):.2f}")


def admin_menu(data):
    while True:
        print("""
--- 管理员菜单 ---
1. 查看学生
2. 添加学生
3. 查看课程
4. 添加课程
5. 查看全部成绩
6. 录入/修改成绩
7. 查询学生成绩
8. 课程统计
0. 退出
""")
        choice = input("请选择：").strip()
        if choice == "1":
            list_students(data)
        elif choice == "2":
            add_student(data)
        elif choice == "3":
            list_courses(data)
        elif choice == "4":
            add_course(data)
        elif choice == "5":
            list_scores(data)
        elif choice == "6":
            set_score(data)
        elif choice == "7":
            query_student_scores(data)
        elif choice == "8":
            course_stats(data)
        elif choice == "0":
            break
        else:
            print("无效选项")


def teacher_menu(data):
    while True:
        print("""
--- 教师菜单 ---
1. 查看学生
2. 查看课程
3. 录入/修改成绩
4. 查询学生成绩
5. 课程统计
0. 退出
""")
        choice = input("请选择：").strip()
        if choice == "1":
            list_students(data)
        elif choice == "2":
            list_courses(data)
        elif choice == "3":
            set_score(data)
        elif choice == "4":
            query_student_scores(data)
        elif choice == "5":
            course_stats(data)
        elif choice == "0":
            break
        else:
            print("无效选项")


def student_menu(data, user):
    while True:
        print("""
--- 学生菜单 ---
1. 查看我的成绩
2. 查看我的平均分
0. 退出
""")
        choice = input("请选择：").strip()
        if choice == "1":
            student_view_self(data, user)
        elif choice == "2":
            student_view_self(data, user)
        elif choice == "0":
            break
        else:
            print("无效选项")


def main():
    print("=" * 40)
    print("学生成绩管理系统（单文件 CLI）")
    print("=" * 40)

    data = load_data()
    user = login(data)
    if not user:
        print("登录失败，程序退出。")
        return

    role = user.get("role")
    print(f"\n欢迎，{user.get('username')}（{role}）")

    if role == "admin":
        admin_menu(data)
    elif role == "teacher":
        teacher_menu(data)
    elif role == "student":
        student_menu(data, user)
    else:
        print("未知角色，退出。")


if __name__ == "__main__":
    main()
```