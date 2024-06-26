---
title: cgroup v1 中文文档
comments: true
categories:
  - Container
tags:
  - cgroup
date: 2020-03-01 15:24:15
---

## 目录：
=========

1. 控制组
   1.1 什么是控制组（cgroups）？
   1.2 为什么需要控制组？
   1.3 控制组是如何实现的？
   1.4 `notify_on_release` 是做什么的？
   1.5 `clone_children` 是做什么的？
   1.6 我该如何使用控制组？
2. 使用示例和语法
   2.1 基本用法
   2.2 附加进程
   2.3 按名称挂载层次结构
3. 内核 API
   3.1 概述
   3.2 同步
   3.3 子系统 API
4. 扩展属性的使用
5. 问题

## 1. 控制组
=================

### 1.1 什么是控制组（cgroups）？
----------------------

控制组提供了一种将任务及其所有未来的子任务聚合/分区成具有专门行为的分层组的机制。

定义：

一个 *cgroup* 将一组任务与一个或多个子系统的一组参数相关联。

一个 *子系统* 是一个利用控制组提供的任务分组功能以特定方式处理任务组的模块。子系统通常是一个“资源控制器”，负责调度资源或应用每个控制组的限制，但它也可以是任何希望对一组进程进行操作的模块，例如虚拟化子系统。

一个 *层次结构* 是一个按树形排列的控制组集合，这样系统中的每个任务都在该层次结构中的一个控制组中，每个子系统都有与该层次结构中的每个控制组相关的特定状态。每个层次结构都有一个与之关联的控制组虚拟文件系统实例。

在任何时候，可能有多个活动的任务控制组层次结构。每个层次结构都是系统中所有任务的一个分区。

用户级代码可以在控制组虚拟文件系统实例中按名称创建和销毁控制组，指定和查询任务分配到的控制组，并列出分配到控制组的任务 PID。这些创建和分配仅影响与该控制组文件系统实例关联的层次结构。

单独使用时，控制组仅用于简单的作业跟踪。目的是让其他子系统连接到通用控制组支持，以提供新的控制组属性，例如计算/限制控制组中进程可以访问的资源。例如，cpusets（请参见 Documentation/cgroup-v1/cpusets.txt）允许您将一组 CPU 和一组内存节点与每个控制组中的任务相关联。

### 1.2 为什么需要控制组？
----------------------------

在 Linux 内核中，有多个用于提供进程聚合的努力，主要是为了资源跟踪目的。这些努力包括 cpusets、CKRM/ResGroups、UserBeanCounters 和虚拟服务器命名空间。这些都需要进程分组/分区的基本概念，新分叉的进程最终与其父进程在同一个组（控制组）中。

内核控制组补丁提供了高效实现此类组所需的最基本的内核机制。它对系统快速路径的影响最小，并为特定子系统（如 cpusets）提供了挂钩，以便根据需要提供额外的行为。

提供了多层次支持，以便在任务划分为控制组的方式对不同的子系统有明显不同的情况下使用——并行层次结构允许每个层次结构成为任务的自然划分，而无需处理如果多个不相关的子系统需要被强制放入同一个控制组树中时会出现的复杂任务组合。

在一个极端情况下，每个资源控制器或子系统可以在一个单独的层次结构中；在另一个极端情况下，所有子系统都将附加到同一个层次结构中。

例如一个场景（最初由 vatsa@in.ibm.com 提出），可以从多个层次结构中受益，考虑一个大型大学服务器，拥有各种用户——学生、教授、系统任务等。该服务器的资源规划可以如下：
```bash

       CPU :          "顶级 cpuset"
                       /       \
               CPUSet1         CPUSet2
                  |               |
               (Professors)          (Students) 
             此外（system tasks）附加到顶级 cpuset（以便它们可以在任何地方运行），限制为 20%

       内存 : Professors (50%)，Students (30%)，system (20%)

       磁盘 : Professors (50%)，Students (30%)，system (20%)

       网络 : WWW browsing (20%)， Network File System (60%)，others (20%)
                           /  \
              Professors(15%)  Students (5%)
```


像 Firefox/Lynx 这样的浏览器进入 WWW 网络类，而 (k)nfsd 进入 NFS 网络类。

同时，Firefox/Lynx 将根据谁启动它（教授/学生）共享适当的 CPU/内存类。

由于能够为不同的资源以不同的方式分类任务（通过将这些资源子系统放入不同的层次结构中），管理员可以轻松设置一个脚本，该脚本接收执行通知，并根据谁启动浏览器，他可以：
```bash
echo browser_pid > /sys/fs/cgroup/<restype>/<userclass>/tasks
```


只有一个层次结构时，他现在可能需要为每个启动的浏览器创建一个单独的控制组，并将其与适当的网络和其他资源类关联。这可能会导致此类控制组的激增。

另外，假设管理员想要暂时给一个学生的浏览器增强网络访问权限（因为是晚上，用户想要进行在线游戏 :)）或者给一个学生的仿真应用程序增强 CPU 功率。

通过能够将 PID 直接写入资源类，只需：
```bash
echo pid > /sys/fs/cgroup/network/<new_class>/tasks
#（一段时间后）
echo pid > /sys/fs/cgroup/network/<orig_class>/tasks
```
没有这种能力，管理员将不得不将控制组拆分成多个单独的组，然后将新控制组与新的资源类关联。

### 1.3 控制组是如何实现的？
---------------------------------

控制组通过以下方式扩展了内核：

- 系统中的每个任务都有一个指向 css_set 的引用计数指针。

- 一个 css_set 包含一组指向 cgroup_subsys_state 对象的引用计数指针，每个对象对应系统中注册的一个 cgroup 子系统。任务与其在每个层次结构中所属的 cgroup 之间没有直接链接，但可以通过 cgroup_subsys_state 对象的指针链确定。这是因为访问子系统状态通常在性能关键代码中频繁发生，而需要任务的实际 cgroup 分配（特别是在不同 cgroup 之间移动）的操作较少见。每个 task_struct 的 cg_list 字段通过 css_set 形成一个链表，锚定在 css_set->tasks 处。

- 可以挂载一个 cgroup 层次结构文件系统，以便从用户空间浏览和操作。

- 您可以列出附加到任何 cgroup 的所有任务（通过 PID）。

实现控制组需要在内核的几个简单挂钩，但不会影响性能关键路径：

- 在 init/main.c 中，在系统启动时初始化根 cgroup 和初始 css_set。

- 在 fork 和 exit 中，将任务附加和分离到其 css_set。

此外，可以挂载类型为 "cgroup" 的新文件系统，以启用浏览和修改内核当前已知的 cgroup。挂载 cgroup 文件系统时，可以在文件系统挂载选项中指定以逗号分隔的子系统列表。默认情况下，挂载 cgroup 文件系统会尝试挂载一个包含所有注册子系统的层次结构。

如果已经存在具有完全相同子系统集的活动层次结构，则将重用该层次结构进行新挂载。如果没有现有层次结构匹配，并且任何请求的子系统在现有层次结构中正在使用，则挂载将失败并显示 -EBUSY。否则，将激活一个新的层次结构，并与请求的子系统相关联。

目前不可能将新的子系统绑定到活动的 cgroup 层次结构，也不可能从活动的 cgroup 层次结构中解绑子系统。这可能在将来可能实现，但会带来复杂的错误恢复问题。

当卸载 cgroup 文件系统时，如果在顶级 cgroup 下创建了任何子 cgroup，则该层次结构将保持活动状态，即使已卸载；如果没有子 cgroup，则该层次结构将被停用。

没有为控制组添加新的系统调用——所有对控制组的查询和修改支持均通过此 cgroup 文件系统进行。

每个 /proc 下的任务都有一个额外的名为 'cgroup' 的文件，显示每个活动层次结构的子系统名称和 cgroup 名称，作为相对于 cgroup 文件系统根的路径。

每个 cgroup 在 cgroup 文件系统中由一个目录表示，包含描述该 cgroup 的以下文件：

- tasks: 列出附加到该 cgroup 的任务（通过 PID）。此列表无法保证排序。将线程 ID 写入此文件会将线程移动到该 cgroup。
- cgroup.procs: 列出 cgroup 中线程组 ID。此列表无法保证排序或不包含重复的 TGID，并且用户空间应对列表进行排序/去重，如果需要此属性。将线程组 ID 写入此文件会将该组中所有线程移动到该 cgroup。
- notify_on_release 标志：退出时是否运行释放代理？
- release_agent: 用于释放通知的路径（此文件仅存在于顶级 cgroup 中）

其他子系统（如 cpusets）可能会在每个 cgroup 目录中添加额外的文件。

新的 cgroups 可以使用 mkdir 系统调用或 shell 命令创建。通过写入该 cgroup 目录中相应的文件来修改 cgroup 的属性，如上所列。

嵌套 cgroup 的命名层次结构允许将大系统分割为嵌套的、动态可变的“软分区”。

每个任务的附加操作会自动在任务的任何子进程中继承，使其附加到一个 cgroup 允许将系统上的工作负载组织成相关的任务集。如果权限允许，任务可以重新附加到任何其他 cgroup。

当将任务从一个 cgroup 移动到另一个 cgroup 时，如果存在具有所需 cgroup 集合的现有 css_set，则该组将被重用，否则将分配一个新的 css_set。通过查找哈希表来定位适当的现有 css_set。

为了允许从 cgroup 访问 css_sets（因此访问任务），一组 cg_cgroup_link 对象形成一个格子；每个 cg_cgroup_link 链接到其 cgrp_link_list 字段中单个 cgroup 的 cg_cgroup_links 列表，并链接到其 cg_link_list 字段中单个 css_set 的 cg_cgroup_links 列表。

因此，通过迭代引用 cgroup 的每个 css_set，并子迭代每个 css_set 的任务集，可以列出 cgroup 中的任务集。

### 1.4 notify_on_release 是什么？

如果在一个 cgroup 中启用了 notify_on_release 标志（设置为1），那么每当该 cgroup 中的最后一个任务离开（退出或附加到其他 cgroup），并且该 cgroup 的最后一个子 cgroup 被移除时，内核会执行指定在该层次结构根目录中 "release_agent" 文件内容的命令，同时提供被弃用 cgroup 的路径名（相对于 cgroup 文件系统的挂载点）。这使得自动移除被弃用的 cgroups 成为可能。在系统启动时，根 cgroup 中 notify_on_release 的默认值是禁用（0）。其他 cgroup 在创建时的默认值是其父 cgroup 的 notify_on_release 设置的当前值。cgroup 层次结构的 release_agent 路径的默认值为空。

### 1.5 clone_children 是什么？

这个标志只影响 cpuset 控制器。如果在一个 cgroup 中启用了 clone_children 标志（设置为1），则在初始化期间，新的 cpuset cgroup 将从其父 cgroup 复制其配置。

### 1.6 我如何使用 cgroups？
要启动一个新的作业并将其包含在一个 cgroup 中，使用 "cpuset" cgroup 子系统的步骤大致如下：

1) 挂载 tmpfs 文件系统作为 cgroup 根目录：
```bash
mount -t tmpfs cgroup_root /sys/fs/cgroup
```

2) 创建 cpuset 目录：
```bash
mkdir /sys/fs/cgroup/cpuset
```

3) 挂载 cpuset cgroup 子系统到新创建的目录：
```bash
mount -t cgroup -o cpuset cpuset /sys/fs/cgroup/cpuset
```

4) 在 /sys/fs/cgroup/cpuset 虚拟文件系统中创建新的 cgroup，并通过写入（或使用 echo 命令）设置其配置。

5) 启动一个任务，该任务将作为新作业的 "创始者"。

6) 将该任务附加到新创建的 cgroup 中，通过将其 PID 写入该 cgroup 的 /sys/fs/cgroup/cpuset tasks 文件。

7) 从该创始者任务中 fork、exec 或 clone 作业任务。

例如，以下一系列命令将设置一个名为 "Charlie" 的 cgroup，其中包含 CPU 2 和 3，以及内存节点 1，并在该 cgroup 中启动一个子 shell 'sh'：

```bash
mount -t tmpfs cgroup_root /sys/fs/cgroup
mkdir /sys/fs/cgroup/cpuset
mount -t cgroup cpuset -o cpuset /sys/fs/cgroup/cpuset
cd /sys/fs/cgroup/cpuset
mkdir Charlie
cd Charlie
echo "2-3" > cpuset.cpus
echo "1" > cpuset.mems
echo "$$" > tasks
sh
# 子 shell 'sh' 现在运行在 cgroup Charlie 中
# 下一行应显示 '/Charlie'
cat /proc/self/cgroup
```

### 2. 使用示例和语法
============================

#### 2.1 基本用法
---------------

创建、修改和使用 cgroups 可以通过 cgroup 虚拟文件系统完成。

要挂载一个包含所有可用子系统的 cgroup 层次结构，请输入：
```bash
# mount -t cgroup xxx /sys/fs/cgroup
```
这里的 "xxx" 并不由 cgroup 代码解释，但会出现在 /proc/mounts 中，因此可以是任何有用的标识字符串。

注意：某些子系统在没有用户输入的情况下无法工作。例如，如果启用了 cpusets，用户必须在创建每个新的 cgroup 后，先填充 cpus 和 mems 文件，然后才能使用该组。

如在第 `1.2 Why are cgroups needed?' 部分所述，您应为每个单一资源或资源组创建不同的 cgroup 层次结构来进行控制。因此，您应在 /sys/fs/cgroup 上挂载一个 tmpfs，并为每个 cgroup 资源或资源组创建目录。
```bash
mount -t tmpfs cgroup_root /sys/fs/cgroup
mkdir /sys/fs/cgroup/rg1
```

要挂载仅包含 cpuset 和 memory 子系统的 cgroup 层次结构，请输入：
```bash
mount -t cgroup -o cpuset,memory hier1 /sys/fs/cgroup/rg1
```
目前虽然支持重新挂载 cgroups，但不建议使用。重新挂载允许更改绑定的子系统和 release_agent。重新绑定几乎没有用处，因为它只在层次结构为空时起作用，而 release_agent 本身应该替换为常规的 fsnotify。将来将删除重新挂载的支持。

要指定层次结构的 release_agent：
```bash
mount -t cgroup -o cpuset,release_agent="/sbin/cpuset_release_agent" \
xxx /sys/fs/cgroup/rg1
```
请注意，指定 'release_agent' 多次将返回失败。

当前仅在层次结构仅包含单个（根）cgroup 时支持更改子系统集合。支持从现有 cgroup 层次结构任意绑定/解绑子系统的能力，预计将在未来实现。

然后在 /sys/fs/cgroup/rg1 下可以找到对应系统 cgroups 树的树。例如，/sys/fs/cgroup/rg1 是包含整个系统的 cgroup。

如果要更改 release_agent 的值：
```bash
echo "/sbin/new_release_agent" > /sys/fs/cgroup/rg1/release_agent
```

也可以通过重新挂载进行更改。

如果要在 /sys/fs/cgroup/rg1 下创建一个新的 cgroup：
```bash
cd /sys/fs/cgroup/rg1
mkdir my_cgroup
```
现在你想在这个 cgroup 中做些什么。
```bash
cd my_cgroup
```
在这个目录中，你可以找到几个文件：
```bash 
ls 
cgroup.procs notify_on_release tasks
# 加上附加子系统添加的任何其他文件
```
现在将你的 shell 附加到这个 cgroup：
```bash
/bin/echo $$ > tasks
```
你还可以通过在这个目录中使用 mkdir 来在你的 cgroup 中创建子 cgroups。
```bash
mkdir my_sub_cs
```
要删除一个 cgroup，只需使用 rmdir：
```bash
rmdir my_sub_cs
```
如果 cgroup 正在使用中（内部有 cgroups，或者有附加的进程，或者被其他特定于子系统的引用所持有），这将失败。

2.2 挂载进程
-----------------------

```bash
/bin/echo PID > tasks
```
请注意，是 PID，不是 PIDs。你一次只能附加一个任务。
如果你有多个任务要附加，你需要一个接一个地执行：
```bash
/bin/echo PID1 > tasks
/bin/echo PID2 > tasks
	...
/bin/echo PIDn > tasks
```
你可以通过将 echo 0 写入 tasks 文件来附加当前 shell 任务：
```bash
echo 0 > tasks
```
你可以使用 cgroup.procs 文件代替 tasks 文件一次性移动一个线程组中的所有线程。将任何任务的 PID 写入 cgroup.procs 将导致该线程组中的所有任务都被附加到 cgroup 中。将 0 写入 cgroup.procs 将移动写入任务所在线程组中的所有任务。

注意：由于每个任务在每个已挂载的层次结构中始终是一个 cgroup 的成员，要从当前 cgroup 中移除任务，必须将其移动到一个新的 cgroup（可能是根 cgroup）中，方法是将其写入新 cgroup 的 tasks 文件。

注意：由于某些 cgroup 子系统强制执行的一些限制，移动进程到另一个 cgroup 可能会失败。

2.3 通过名称挂载层次结构
--------------------------------

在挂载 cgroups 层次结构时，通过传递 name=<x> 选项将给定的名称与该层次结构关联起来。这在挂载预先存在的层次结构时很有用，可以通过名称而不是其一组活动子系统来引用它。每个层次结构要么没有名称，要么具有唯一名称。

名称应该匹配 [\w.-]+

当为新的层次结构传递 name=<x> 选项时，你需要手动指定子系统；当没有明确指定子系统时，不支持挂载所有子系统的传统行为。

子系统的名称将作为层次结构描述的一部分出现在 /proc/mounts 和 /proc/<pid>/cgroups 中。

3. 内核 API
   =============

3.1 概述
------------

每个希望钩入通用 cgroup 系统的内核子系统都需要创建一个 cgroup_subsys 对象。这包含各种方法，这些方法是从 cgroup 系统的回调，以及一个由 cgroup 系统分配的子系统 ID。

cgroup_subsys 对象中的其他字段包括：

- subsys_id：子系统的唯一数组索引，指示此子系统应管理 cgroup->subsys[] 中的哪个条目。

- name：应初始化为唯一的子系统名称。长度不应超过 MAX_CGROUP_TYPE_NAMELEN。

- early_init：指示子系统是否需要在系统启动时进行早期初始化。

系统创建的每个 cgroup 对象都有一个指针数组，由子系统 ID 索引；这个指针完全由子系统管理；通用 cgroup 代码永远不会触及这个指针。

3.2 同步
-------------------

cgroup 系统使用一个全局互斥锁 cgroup_mutex。任何想要修改 cgroup 的东西都应该获取此锁。也可以获取此锁以防止修改 cgroup，但在这种情况下，可能更适合使用更具体的锁。

更多详情请参见 kernel/cgroup.c。

子系统可以通过函数 cgroup_lock()/cgroup_unlock() 获取/释放 cgroup_mutex。

可以在以下方式中访问任务的 cgroup 指针：
- 在持有 cgroup_mutex 时
- 在持有任务的 alloc_lock 时（通过 task_lock()）
- 在 rcu_read_lock() 区段内通过 rcu_dereference()

3.3 子系统 API
-----------------

每个子系统应该：

- 在 linux/cgroup_subsys.h 中添加一个条目
- 定义一个名为 <name>_cgrp_subsys 的 cgroup_subsys 对象

每个子系统可以导出以下方法。唯一强制的方法是 css_alloc/free。其他任何为空的方法都被假定为成功的空操作。

struct cgroup_subsys_state *css_alloc(struct cgroup *cgrp)
（由调用者持有 cgroup_mutex）

用于为 cgroup 分配一个子系统状态对象。子系统应为传递的 cgroup 分配其子系统状态对象，成功时返回指向新对象的指针或 ERR_PTR() 值。在成功时，子系统指针应指向一个 cgroup_subsys_state 类型的结构（通常嵌入在一个更大的特定于子系统的对象中），该结构将由 cgroup 系统初始化。请注意，这将在初始化时调用，为此子系统创建根子系统状态；可以通过传递的 cgroup 对象具有 NULL 父对象（因为它是层次结构的根）来识别这种情况，并且可能是初始化代码的适当位置。

int css_online(struct cgroup *cgrp)
（由调用者持有 cgroup_mutex）

在 @cgrp 成功完成所有分配并对 cgroup_for_each_child/descendant_*() 迭代器可见后调用。子系统可以选择通过返回 -errno 失败创建。此回调可用于实现可靠的状态共享和沿层次结构的传播。有关详细信息，请参阅关于 cgroup_for_each_descendant_pre() 的注释。

void css_offline(struct cgroup *cgrp);
（由调用者持有 cgroup_mutex）

这是 css_online() 的对应物，仅当 css_online() 在 @cgrp 上成功时调用。这表示 @cgrp 的结束开始。@cgrp 正在被移除，子系统应开始放弃其在 @cgrp 上持有的所有引用。当所有引用都被放弃时，cgroup 移除将继续进行到下一步 - css_free()。在此回调之后，@cgrp 应被视为对子系统无效。

void css_free(struct cgroup *cgrp)
（由调用者持有 cgroup_mutex）

cgroup 系统即将释放 @cgrp；子系统应释放其子系统状态对象。在调用此方法时，@cgrp 完全未使用；@cgrp->parent 仍然有效。（注意 - 如果在新创建的 cgroup 的此子系统的 create() 方法调用后发生错误，则也可以调用此方法）。

int can_attach(struct cgroup *cgrp, struct cgroup_taskset *tset)
（由调用者持有 cgroup_mutex）

在将一个或多个任务移入 cgroup 之前调用；如果子系统返回错误，则将中止附加操作。@tset 包含要附加的任务，并且保证其中至少有一个任务。

如果任务集中有多个任务，则：
- 保证所有任务来自同一线程组
- @tset 包含线程组中所有任务，无论它们是否在切换 cgroup
- 第一个任务是领导者

每个 @tset 条目还包含任务的旧 cgroup，并且可以使用 cgroup_taskset_for_each() 迭代器轻松跳过不在切换 cgroup 的任务。请注意，这不会在 fork 上调用。如果此方法返回 0（成功），则在调用者持有 cgroup_mutex 的情况下应保持此有效，并确保将来将调用 attach() 或 cancel_attach()。

void css_reset(struct cgroup_subsys_state *css)
（由调用者持有 cgroup_mutex）

可选操作，应将 @css 的配置恢复到初始状态。目前仅在统一层次结构上禁用 cgroup.subtree_control 中的子系统时使用，但应保持启用状态，因为其他子系统依赖它。cgroup 核心通过删除关联的接口文件使此类 css 不可见，并调用此回调，以便隐藏的子系统可以返回到初始中性状态。这样可以防止隐藏的 css 引起意外的资源控制，并确保在稍后再次可见时配置处于初始状态。

void cancel_attach(struct cgroup *cgrp, struct cgroup_taskset *tset)
（由调用者持有 cgroup_mutex）

在 can_attach() 成功后，任务附加操作失败时调用。具有一些副作用的 can_attach() 的子系统应提供此功能，以便子系统可以实现回滚。如果没有，这不是必需的。仅会调用关于 can_attach() 操作已成功的子系统。参数与 can_attach() 相同。

void attach(struct cgroup *cgrp, struct cgroup_taskset *tset)
（由调用者持有 cgroup_mutex）

在任务已附加到 cgroup 后调用，以允许需要内存分配或阻塞的任何后附加活动。参数与 can_attach() 相同。

void fork(struct task_struct *task)

在任务被 fork 到 cgroup 时调用。

void exit(struct task_struct *task)

在任务退出时调用。

void free(struct task_struct *task)

在释放 task_struct 时调用。

void bind(struct cgroup *root)
（由调用者持有 cgroup_mutex）

在将 cgroup 子系统重新绑定到不同的层次结构和根 cgroup 时调用。当前这只会涉及在默认层次结构（永远不会有子 cgroup）和正在创建/销毁的层次结构之间的移动。

4. 扩展属性使用
   ===========================

cgroup 文件系统支持其目录和文件中的某些类型的扩展属性。
当前支持的类型包括：
- Trusted（XATTR_TRUSTED）
- Security（XATTR_SECURITY）

设置这些属性需要 CAP_SYS_ADMIN 权限。

与 tmpfs 中类似，cgroup 文件系统中的扩展属性使用内核内存存储，建议将使用量保持在最低。这也是为什么不支持用户定义扩展属性的原因，因为任何用户都可以这样做，并且值大小没有限制。

此功能的当前已知用户包括 SELinux 用于限制容器中的 cgroup 使用，以及 systemd 用于各种元数据

5. 问题
 ===========================

问：为什么要使用 '/bin/echo' 命令？
答：bash 的内置 'echo' 命令在调用 write() 时不会检查错误。如果在 cgroup 文件系统中使用它，你将无法确定命令是成功还是失败。

问：当我附加进程时，一行中只有第一个进程真正被附加了！
答：每次调用 write() 只能返回一个错误代码。因此，你应该每次只放置一个 PID。



# 原文参考
https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt
