# 结课项目二(编译期多态实现任务管理系统)

### 项目简介

开发一个命令行界面的任务管理系统，允许用户添加、删除、查看和管理他们的待办事项。该系统将涵盖以下功能：



- **添加任务**：用户可以添加新的待办任务，包含任务描述、优先级和截止日期。

- **删除任务**：用户可以根据任务ID删除指定的任务。

- **查看任务**：用户可以查看所有任务，按优先级或截止日期排序。

- **更新任务**：用户可以更新任务的详细信息。

- **数据持久化**：任务数据将保存在本地文件中，以便在程序重启后仍能保留数据。

- **日志记录**：系统将记录用户的操作日志，使用单例模式实现日志系统。

- **扩展性**：采用CRTP模式实现基础命令接口，方便未来扩展更多命令。



### 项目结构

1. **主程序（**`main.cpp`**）**：处理用户输入和命令调用。

1. **任务管理模块（**`Task.h`**,** `Task.cpp`**,** `TaskManager.h`**,** `TaskManager.cpp`**）**：负责任务的创建、删除、更新和查询。

1. **日志系统（**`Logger.h`**,** `Logger.cpp`**）**：使用单例模式实现日志记录。

1. **命令接口（**`Command.h`**,** `Command.cpp`**）**：采用CRTP模式实现命令的基础接口和具体命令类。

1. **数据持久化（**`DataStore.h`**,** `DataStore.cpp`**）**：负责将任务数据保存到文件和从文件加载数据。



### 设计思路

#### 1. 单例模式：日志系统

日志系统是典型的单例模式应用场景，确保整个程序中只有一个日志实例，便于集中管理日志记录。



#### 2. CRTP模式：命令接口

采用CRTP模式实现命令的基础接口，使得每个具体命令类在编译时期就确定了具体的行为，避免了运行时多态的开销，并提升了性能。



#### 3. 面向对象设计

通过类的封装，将任务管理和数据持久化模块分离，增强代码的可维护性和可扩展性。



### CRTP vs 虚函数继承：区别与优劣分析

CRTP（奇异递归模板模式）和虚函数继承是C++中实现多态的两种不同方式，它们各有优缺点。



#### 基本区别

1. **多态实现机制**：

- - CRTP：静态多态（编译时确定）

- - 虚函数：动态多态（运行时确定）

1. **示例对比**：

```
// CRTP方式（代码中的实现）
template <typename Derived>
class Command {
public:
    void execute(const std::string& args) {
        static_cast<Derived*>(this)->executeImpl(args);
    }
};

// 虚函数方式
class Command {
public:
    virtual void execute(const std::string& args) = 0;
    virtual ~Command() {}
};
```

#### CRTP的优势

1. **性能优化**：

- - 无虚函数调用开销（无需查虚函数表）

- - 支持内联，允许更多编译器优化

- - 零运行时开销的抽象

1. **编译时检查**：

- - 类型安全，编译期捕获错误

- - 方法不存在时会产生编译错误

1. **适用场景**：

- - 高性能计算

- - 模板库设计

- - 嵌入式系统等资源受限环境



#### CRTP的劣势

1. **代码膨胀**：每个派生类都会实例化一套基类模板代码

1. **编译时间**：增加编译时间和编译复杂度

1. **调试困难**：模板错误提示往往难以理解

1. **接口不明确**：缺少明确的接口约束机制

1. **不支持动态加载**：无法支持运行时动态库加载等场景



#### 虚函数继承的优势

1. **运行时灵活性**：

- - 支持运行时确定的多态行为

- - 可以处理编译时未知的类型

1. **接口清晰**：

- - 明确定义接口，强制派生类实现

- - 符合大多数开发者的OOP直觉

1. **支持动态加载**：可与动态链接库配合使用

1. **适用场景**：

- - 插件系统

- - 框架设计

- - 需要运行时类型确定的场景



#### 虚函数继承的劣势

1. **性能开销**：

- - 虚函数调用的间接寻址开销

- - 每个对象额外的vptr（虚表指针）开销

1. **内联限制**：虚函数通常不能被内联

1. **内存开销**：虚函数表消耗额外内存



#### 本项目选择CRTP

代码使用CRTP模式实现了命令模式，适合于：



- 性能敏感的应用

- 命令类型在编译期已确定的系统

- 希望减少运行时开销的场景



如果您需要在运行时动态加载新命令或通过配置文件确定命令类型，则虚函数实现会更合适。



#### 选择建议

根据您的具体需求选择：



- 需要极致性能且类型在编译时已知：选CRTP

- 需要运行时灵活性和动态行为：选虚函数继承



## 源码实现

以下是项目的详细源码，包括各个模块的实现。



### 1. Logger（日志系统）

#### **Logger.h**

```cpp
#ifndef LOGGER_H
#define LOGGER_H

#include <string>
#include <fstream>
#include <mutex>

class Logger {
public:
    // 获取单例实例
    static Logger& getInstance();

    // 禁止拷贝和赋值
    Logger(const Logger&) = delete;
    Logger& operator=(const Logger&) = delete;

    // 记录日志
    void log(const std::string& message);

private:
    Logger(); // 私有构造函数
    ~Logger();

    std::ofstream logFile;
    std::mutex mtx;
};

#endif // LOGGER_H
```

#### **Logger.cpp**

```cpp
#include "Logger.h"
#include <iostream>
#include <chrono>
#include <ctime>

Logger& Logger::getInstance() {
    static Logger instance;
    return instance;
}

Logger::Logger() {
    logFile.open("log.txt", std::ios::app);
    if (!logFile.is_open()) {
        std::cerr << "无法打开日志文件。" << std::endl;
    }
}

Logger::~Logger() {
    if (logFile.is_open()) {
        logFile.close();
    }
}

void Logger::log(const std::string& message) {
    std::lock_guard<std::mutex> lock(mtx);
    if (logFile.is_open()) {
        // 获取当前时间
        auto now = std::chrono::system_clock::now();
        std::time_t now_time = std::chrono::system_clock::to_time_t(now);
        logFile << std::ctime(&now_time) << ": " << message << std::endl;
    }
}
```



### 2. Task（任务模块）

#### **Task.h**

```cpp
#ifndef TASK_H
#define TASK_H

#include <string>
#include <sstream>
#include <iomanip>

struct Task {
    int id;
    std::string description;
    int priority; // 1-高, 2-中, 3-低
    std::string dueDate; // 格式: YYYY-MM-DD

    std::string toString() const {
        std::ostringstream oss;
        oss << "ID: " << id 
            << ", 描述: " << description 
            << ", 优先级: " << priority 
            << ", 截止日期: " << dueDate;
        return oss.str();
    }
};

#endif // TASK_H
```

#### **TaskManager.h**

```cpp
#ifndef TASKMANAGER_H
#define TASKMANAGER_H

#include "Task.h"
#include <vector>
#include <string>
#include <fstream>
#include <algorithm>
#include <sstream>

class TaskManager {
public:
    TaskManager();

    void addTask(const std::string& description, int priority, const std::string& dueDate);
    void deleteTask(int id);
    void updateTask(int id, const std::string& description, int priority, const std::string& dueDate);
    void listTasks(int sortOption = 0) const; // 0-按ID, 1-按优先级, 2-按截止日期
    void loadTasks();
    void saveTasks() const;

private:
    std::vector<Task> tasks;
    int nextId;

    // 辅助排序函数
    static bool compareByPriority(const Task& a, const Task& b);
    static bool compareByDueDate(const Task& a, const Task& b);
};

#endif // TASKMANAGER_H
```

#### **TaskManager.cpp**

```cpp
#include "TaskManager.h"
#include "Logger.h"
#include <iostream>

TaskManager::TaskManager() : nextId(1) {
    loadTasks();
}

void TaskManager::addTask(const std::string& description, int priority, const std::string& dueDate) {
    Task task;
    task.id = nextId++;
    task.description = description;
    task.priority = priority;
    task.dueDate = dueDate;
    tasks.push_back(task);
    Logger::getInstance().log("添加任务: " + task.toString());
    saveTasks();
}

void TaskManager::deleteTask(int id) {
    auto it = std::find_if(tasks.begin(), tasks.end(), [id](const Task& task) {
        return task.id == id;
    });
    if (it != tasks.end()) {
        Logger::getInstance().log("删除任务: " + it->toString());
        tasks.erase(it);
        saveTasks();
    } else {
        std::cout << "未找到ID为 " << id << " 的任务。" << std::endl;
    }
}

void TaskManager::updateTask(int id, const std::string& description, int priority, const std::string& dueDate) {
    for (auto& task : tasks) {
        if (task.id == id) {
            Logger::getInstance().log("更新前任务: " + task.toString());
            task.description = description;
            task.priority = priority;
            task.dueDate = dueDate;
            Logger::getInstance().log("更新后任务: " + task.toString());
            saveTasks();
            return;
        }
    }
    std::cout << "未找到ID为 " << id << " 的任务。" << std::endl;
}

void TaskManager::listTasks(int sortOption) const {
    std::vector<Task> sortedTasks = tasks;
    switch (sortOption) {
        case 1:
            std::sort(sortedTasks.begin(), sortedTasks.end(), compareByPriority);
            break;
        case 2:
            std::sort(sortedTasks.begin(), sortedTasks.end(), compareByDueDate);
            break;
        default:
            break;
    }
    for (const auto& task : sortedTasks) {
        std::cout << task.toString() << std::endl;
    }
}

void TaskManager::loadTasks() {
    std::ifstream inFile("tasks.txt");
    if (!inFile.is_open()) {
        Logger::getInstance().log("任务文件不存在，开始新建。");
        return;
    }
    std::string line;
    while (std::getline(inFile, line)) {
        std::istringstream iss(line);
        Task task;
        char delimiter;
        iss >> task.id >> delimiter;
        std::getline(iss, task.description, ',');
        iss >> task.priority >> delimiter;
        iss >> task.dueDate;
        tasks.push_back(task);
        if (task.id >= nextId) {
            nextId = task.id + 1;
        }
    }
    inFile.close();
    Logger::getInstance().log("加载任务成功。");
}

void TaskManager::saveTasks() const {
    std::ofstream outFile("tasks.txt");
    if (!outFile.is_open()) {
        Logger::getInstance().log("无法打开任务文件进行保存。");
        return;
    }
    for (const auto& task : tasks) {
        outFile << task.id << "," << task.description << "," << task.priority << "," << task.dueDate << "\n";
    }
    outFile.close();
    Logger::getInstance().log("保存任务成功。");
}

bool TaskManager::compareByPriority(const Task& a, const Task& b) {
    return a.priority < b.priority;
}

bool TaskManager::compareByDueDate(const Task& a, const Task& b) {
    return a.dueDate < b.dueDate;
}
```

### 3. Command（命令接口）

#### **Command.h**

```cpp
#ifndef COMMAND_H
#define COMMAND_H

#include <string>

class CommandBase{
public:
    virtual void execute(const std::string& args) = 0;
};

// CRTP 基类模板
template <typename Derived>
class Command :public CommandBase{
public:
    void execute(const std::string& args) {
        static_cast<Derived*>(this)->executeImpl(args);
    }
};

// 具体命令类示例
#include "TaskManager.h"
#include "Logger.h"
#include <iostream>

// 添加任务命令
class AddCommand : public Command<AddCommand> {
public:
    AddCommand(TaskManager& manager) : taskManager(manager) {}
    void executeImpl(const std::string& args) {
        // 简单的参数解析：描述,优先级,截止日期
        size_t pos1 = args.find(',');
        size_t pos2 = args.find(',', pos1 + 1);
        if (pos1 == std::string::npos || pos2 == std::string::npos) {
            std::cout << "参数格式错误。请使用: add <描述>,<优先级>,<截止日期>" << std::endl;
            return;
        }
        std::string description = args.substr(0, pos1);
        int priority = std::stoi(args.substr(pos1 + 1, pos2 - pos1 - 1));
        std::string dueDate = args.substr(pos2 + 1);
        taskManager.addTask(description, priority, dueDate);
        std::cout << "任务添加成功。" << std::endl;
    }
private:
    TaskManager& taskManager;
};

// 删除任务命令
class DeleteCommand : public Command<DeleteCommand> {
public:
    DeleteCommand(TaskManager& manager) : taskManager(manager) {}
    void executeImpl(const std::string& args) {
        try{
            size_t pos;
            int id = std::stoi(args, &pos);
            if(pos != args.length()){
                std::cout << "参数格式错误。请使用: delete <ID>" << std::endl;
                return;
            }
            taskManager.deleteTask(id);
            std::cout << "任务删除成功。" << std::endl;
        }catch(const std::invalid_argument& e){
            std::cout << "参数格式错误。请使用: delete <ID>" << std::endl;
            return;
        }catch(const std::out_of_range& e){
            std::cout << "ID超出范围。请使用有效的任务ID。" << std::endl;
            return;
        }

    }
private:
    TaskManager& taskManager;
};

// 列出任务命令
class ListCommand : public Command<ListCommand> {
public:
    ListCommand(TaskManager& manager) : taskManager(manager) {}
    void executeImpl(const std::string& args) {
        int sortOption = 0;
        if (!args.empty()) {
            sortOption = std::stoi(args);
        }
        taskManager.listTasks(sortOption);
    }
private:
    TaskManager& taskManager;
};

// 更新任务命令
class UpdateCommand : public Command<UpdateCommand> {
public:
    UpdateCommand(TaskManager& manager) : taskManager(manager) {}
    void executeImpl(const std::string& args) {
        // 参数格式: ID,描述,优先级,截止日期
        size_t pos1 = args.find(',');
        size_t pos2 = args.find(',', pos1 + 1);
        size_t pos3 = args.find(',', pos2 + 1);
        if (pos1 == std::string::npos || pos2 == std::string::npos || pos3 == std::string::npos) {
            std::cout << "参数格式错误。请使用: update <ID>,<描述>,<优先级>,<截止日期>" << std::endl;
            return;
        }
        int id = std::stoi(args.substr(0, pos1));
        std::string description = args.substr(pos1 + 1, pos2 - pos1 - 1);
        int priority = std::stoi(args.substr(pos2 + 1, pos3 - pos2 - 1));
        std::string dueDate = args.substr(pos3 + 1);
        taskManager.updateTask(id, description, priority, dueDate);
        std::cout << "任务更新成功。" << std::endl;
    }
private:
    TaskManager& taskManager;
};

#endif // COMMAND_H
```

### 4. 主程序

**main.cpp**

```cpp
#include <iostream>
#include <string>
#include <unordered_map>
#include <memory>
#include "TaskManager.h"
#include "Command.h"

int main() {
    TaskManager taskManager;

    // 创建命令对象
    AddCommand addCmd(taskManager);
    DeleteCommand delCmd(taskManager);
    ListCommand listCmd(taskManager);
    UpdateCommand updateCmd(taskManager);

    // 命令映射
    std::unordered_map<std::string, std::unique_ptr<CommandBase>> commands;
    commands["add"] = std::make_unique<AddCommand>(taskManager);
    commands["delete"] = std::make_unique<DeleteCommand>(taskManager);
    commands["list"] = std::make_unique<ListCommand>(taskManager);
    commands["update"] = std::make_unique<UpdateCommand>(taskManager);

    std::cout << "欢迎使用任务管理系统！" << std::endl;
    std::cout << "可用命令: add, delete, list, update, exit" << std::endl;

    std::string input;
    while (true) {
        std::cout << "\n> ";
        std::getline(std::cin, input);
        if (input.empty()) continue;

        // 分离命令和参数
        size_t spacePos = input.find(' ');
        std::string cmd = input.substr(0, spacePos);
        std::string args;
        if (spacePos != std::string::npos) {
            args = input.substr(spacePos + 1);
        }

        if (cmd == "exit") {
            std::cout << "退出程序。" << std::endl;
            break;
        }

        auto it = commands.find(cmd);
        if (it != commands.end()) {
            it->second->execute(args);
        } else {
            std::cout << "未知命令：" << cmd << std::endl;
        }
    }

    return 0;
}
```

### 5.使用示例

```
欢迎使用任务管理系统！
可用命令: add, delete, list, update, exit

> add 完成项目报告,1,2023-12-31
任务添加成功。

> add 购买日用品,2,2023-10-15
任务添加成功。

> list
ID: 1, 描述:  完成项目报告, 优先级: 1, 截止日期: 2023-12-31
ID: 2, 描述:  购买日用品, 优先级: 2, 截止日期: 2023-10-15

> update 2,购买更多日用品,1,2023-10-20
任务更新成功。

> list 1
ID: 1, 描述:  完成项目报告, 优先级: 1, 截止日期: 2023-12-31
ID: 2, 描述:  购买更多日用品, 优先级: 1, 截止日期: 2023-10-20

> delete 1
任务删除成功。

> list
ID: 2, 描述:  购买更多日用品, 优先级: 1, 截止日期: 2023-10-20

> exit
退出程序。
```

## 在CRTP模式下存储多种命令类型的解决方案

当使用CRTP模式时，存储不同类型的命令确实是个挑战，因为`AddCommand`、`DeleteCommand`等各自继承自不同的模板实例，没有共同的基类。以下是几种解决方案：



### 1. 使用非模板的公共基类

创建一个普通基类，然后让CRTP模板类继承自它：

```cpp
// 普通基类
class CommandBase {
public:
    virtual void execute(const std::string& args) = 0;
    virtual ~CommandBase() = default;
};

// 修改CRTP模板类
template <typename Derived>
class Command : public CommandBase {
public:
    void execute(const std::string& args) override {
        static_cast<Derived*>(this)->executeImpl(args);
    }
};

// 使用方式
std::unordered_map<std::string, std::unique_ptr<CommandBase>> commands;
commands["add"] = std::make_unique<AddCommand>(taskManager);
commands["delete"] = std::make_unique<DeleteCommand>(taskManager);
// 调用
commands["add"]->execute("任务描述,1,2023-12-31");
```

### 2. 使用类型擦除技术（std::function）

```cpp
//方式二、 类型擦除
auto add_command = std::make_shared<AddCommand>(taskManager);
auto delete_command = std::make_shared<DeleteCommand>(taskManager);
auto list_command = std::make_shared<ListCommand>(taskManager);
auto update_command = std::make_shared<UpdateCommand>(taskManager);
std::unordered_map<std::string, std::function<void(const std::string&)> > commands;
//放入map
commands["add"] = [add_command](const std::string& args)
      { add_command->execute(args); };
commands["delete"] = [delete_command](const std::string& args)
      { delete_command->execute(args); };
commands["list"] = [list_command](const std::string& args)
      { list_command->execute(args); };
commands["update"] = [update_command](const std::string& args)
      { update_command->execute(args); };

//调用方式
auto it = commands.find("add");
if (it != commands.end()) {
            it->second("喂狗,1,2025-03-12");
  } else {
        std::cout << "未知命令：" << cmd << std::endl;
   }

```

### 3. 使用std::any或std::variant (C++17)

```cpp
// 使用std::any
std::unordered_map<std::string, std::any> commands;
commands["add"] = AddCommand(taskManager);
commands["delete"] = DeleteCommand(taskManager);

// 调用（需要知道具体类型）
std::any_cast<AddCommand&>(commands["add"]).execute("任务描述,1,2023-12-31");

// 或使用std::variant
using CommandVariant = std::variant<AddCommand, DeleteCommand, ListCommand, UpdateCommand>;
std::unordered_map<std::string, CommandVariant> commands;
commands["add"] = AddCommand(taskManager);

// 调用
std::visit([&args](auto&& cmd) { cmd.execute(args); }, commands["add"]);
```

1. **定义**`CommandVariant`**类型**

```cpp
using CommandVariant = std::variant<AddCommand, DeleteCommand, ListCommand, UpdateCommand>;
```

- - `std::variant`: 是C++17引入的一个类型安全的联合体，能够持有预定义类型中的一种。这意味着`CommandVariant`可以保存`AddCommand`、`DeleteCommand`、`ListCommand`或`UpdateCommand`中的任意一个实例。

- - **用途**: 用于需要在运行时存储和管理多种不同类型对象的场景，同时保持类型安全。

2. **创建命令映射表**

`std::unordered_map<std::string, CommandVariant> commands;`

- - `std::unordered_map<std::string, CommandVariant>`: 创建一个哈希表，键是字符串（如"add"），值是`CommandVariant`类型。

- - **用途**: 通过字符串键来索引和管理不同的命令对象，这使得根据用户输入或其他条件动态调用不同的命令变得容易。

3. **添加命令到映射表**

```
commands["add"] = AddCommand(taskManager);
```

- - `AddCommand(taskManager)`: 创建一个`AddCommand`对象，假设`taskManager`是其构造函数所需的参数。

- - **赋值给**`commands["add"]`: 将`AddCommand`实例存储在命令映射表中，键为"add"。

4. **调用命令**

`std::visit([&args](auto&& cmd) { cmd.execute(args); }, commands["add"]);`

**作用**: `std::visit`会自动检测`commands["add"]`中当前存储的具体类型（在这个例子中是`AddCommand`），然后调用lambda表达式，进而执行`AddCommand`的`execute`方法。

- - `std::visit`: 是一个用于访问`std::variant`中当前存储的值的函数。它接受一个访问者（通常是一个函数或lambda）和一个`std::variant`。

- - Lambda表达式:`[&args](auto&& cmd) { cmd.execute(args); }`

- - - **捕获列表**: `[&args]` 表示按引用捕获`args`，用于在lambda内部使用。

- - - **参数**: `auto&& cmd` 是一个通用引用，表示`std::variant`中存储的当前活跃类型（如`AddCommand`）。

- - - **操作**: 调用当前命令对象的`execute`方法，并传入`args`作为参数。

- - `commands["add"]`: 这是`std::variant`实例，当前存储的是`AddCommand`对象。

**为什么使用**`std::variant`**和**`std::visit`**？**



1. **类型安全**: 与传统的基类指针相比，`std::variant`在编译时就知道所有可能的类型，减少了运行时错误的风险。

1. **性能**: `std::variant`通常比基类多态更高效，因为它避免了虚函数调用的开销。

1. **简洁性**: 使用`std::variant`和`std::visit`可以避免复杂的类型检查和转换逻辑，使代码更简洁和易于维护。

### 4. 创建命令包装器

```cpp
class CommandWrapper {
    struct Concept {
        virtual void execute(const std::string& args) = 0;
        virtual ~Concept() = default;
    };
    
    template<typename T>
    struct Model : Concept {
        T command;
        Model(T cmd) : command(std::move(cmd)) {}
        void execute(const std::string& args) override {
            command.execute(args);
        }
    };
    
    std::unique_ptr<Concept> impl;
    
public:
    template<typename T>
    CommandWrapper(T cmd) : impl(std::make_unique<Model<T>>(std::move(cmd))) {}
    
    void execute(const std::string& args) {
        impl->execute(args);
    }
};

// 使用方式
std::unordered_map<std::string, CommandWrapper> commands;
commands.emplace("add", AddCommand(taskManager));
commands.emplace("delete", DeleteCommand(taskManager));

// 调用
commands["add"].execute("任务描述,1,2023-12-31");
```

## 推荐方案

方案1（非模板基类）和方案4（命令包装器）是最佳选择：



- 方案1简单直接，只需少量修改现有代码

- 方案4保持CRTP的性能优势，并提供类型安全的接口



如果性能是最优先考虑的，建议使用方案4，因为它避免了虚函数调用的开销。否则，方案1是最简单实用的解决方案。

## 功能扩展建议

1. **数据持久化增强**：目前的数据存储较为简单，可以考虑使用JSON格式存储任务，使用第三方库（如`nlohmann/json`）进行序列化和反序列化。

1. **用户认证**：添加用户登录功能，每个用户有独立的任务列表。

1. **图形界面**：使用图形库（如Qt或SFML）为任务管理系统添加图形用户界面。

1. **高级搜索和过滤**：支持根据关键字、日期范围、优先级等多条件搜索和过滤任务。

1. **任务分类**：为任务添加类别标签，便于分类管理。



## 代码优化与改进

- **命令注册改进**：当前命令注册比较硬编码，可以考虑使用工厂模式或宏来自动注册命令，增强灵活性。

- **错误处理**：增加更多的错误检查和处理机制，提升程序的健壮性。

- **多线程支持**：如果程序规模扩大，可以考虑将日志记录等耗时操作放到独立线程中，避免阻塞主线程。